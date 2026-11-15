---
layout: post
title: "ANS(비대칭 수 체계) 완전 정복: Zstandard·LZFSE가 선택한 차세대 엔트로피 코딩의 수학"
date: 2026-11-15
categories: [cs, computer-science]
tags: [compression, entropy-coding, ANS, rANS, tANS, FSE, zstandard, information-theory]
---

현대 데이터 압축의 핵심은 **엔트로피 코딩(Entropy Coding)**입니다. 허프만 코딩이 수십 년간 표준이었지만, 2009년 Jarek Duda가 발표한 **ANS(Asymmetric Numeral Systems, 비대칭 수 체계)**는 패러다임을 바꾸었습니다. ANS는 산술 코딩에 필적하는 압축률을 허프만 코딩에 근접한 속도로 달성합니다. Facebook의 Zstandard, Apple의 LZFSE, Linux 커널, Google Chrome, Android 등 현대 소프트웨어 생태계가 ANS를 채택한 이유를 깊이 살펴봅니다.

## 개념 설명

### 엔트로피 코딩의 목표

정보 이론에 따르면, 심볼 `s`의 발생 확률이 `p(s)`일 때 최적 코드 길이는 `-log₂(p(s))` 비트입니다. 예를 들어 `A`가 75% 확률로 발생한다면 최적 코드 길이는 `-log₂(0.75) ≈ 0.415` 비트입니다.

| 방법 | 압축률 | 속도 | 특성 |
|------|--------|------|------|
| 허프만 코딩 | 정수 비트 단위 (손실) | 매우 빠름 | 구현 단순 |
| 산술 코딩 | 이론적 최적에 근접 | 느림 | 나눗셈, carry 전파 |
| **ANS** | **이론적 최적에 근접** | **빠름** | **산술 코딩 + 허프만 속도** |

### ANS의 핵심 직관: 상태 기계로서의 압축

ANS는 정수 `x`를 **상태(state)**로 간주합니다. 심볼 하나를 인코딩할 때마다 상태가 갱신됩니다. 최종 상태가 압축된 메시지입니다.

**핵심 아이디어**: 심볼 `s` (확률 `p(s) = freq(s) / M`)를 인코딩할 때:
- 상태 `x`를 `x'`로 갱신: 상태가 `1/p(s)` 배 증가 → 약 `-log₂(p(s))` 비트만큼 정보 추가
- 상태가 너무 커지면 하위 비트를 출력 스트림에 저장 (정규화, renormalization)

**디코딩**은 역방향으로 진행: 상태로부터 심볼과 이전 상태를 복원합니다.

### ANS의 두 변종

**rANS (Range ANS)**:
- 나눗셈/나머지 연산으로 상태 갱신
- 메모리 효율적, 동적 확률 분포 적합
- 인코딩: `x' = (x ÷ freq[s]) × M + cumfreq[s] + (x mod freq[s])`
- 디코딩: `x = freq[s] × (x' ÷ M) + (x' mod M) - cumfreq[s]`

**tANS (Table ANS / Finite State Entropy)**:
- 모든 상태 전이를 사전 계산된 룩업 테이블에 저장
- 나눗셈 없이 테이블 참조만으로 동작 → SIMD에 유리
- Zstandard가 사용하는 방식 (FSE: Finite State Entropy)
- 단점: 확률 분포 변경 시 테이블 재구성 필요

## 왜 필요한가?

### 허프만 코딩의 한계

```
A(p=0.75): 이상적 코드 길이 = 0.415 bits
허프만 코딩 할당: 1 bit (2.4배 비효율)

B(p=0.5): 이상적 = 1 bit → 허프만 = 1 bit (완벽)
C(p=0.25): 이상적 = 2 bits → 허프만 = 2 bits (완벽)
D(p=0.125): 이상적 = 3 bits → 허프만 = 3 bits (완벽)
```

확률이 2의 거듭제곱이 아닌 경우 허프만은 필연적으로 비효율적입니다. `A(75%)`, `B(25%)` 두 심볼만 있다면:
- 허프만: 1 비트/심볼
- 이론 최적: 0.75×0.415 + 0.25×2 = 0.811 bits/심볼
- 허프만 오버헤드: 23%

ANS는 이 차이를 거의 0에 수렴시킵니다.

### 산술 코딩보다 빠른 이유

산술 코딩은 `[low, high)` 구간을 나눗셈으로 갱신하고 carry 전파를 처리해야 합니다. 반면 rANS는 나눗셈이 한 번이고 carry가 없으며, tANS는 나눗셈도 없습니다.

## 실제 구현 예제

### 예제 1: rANS 완전 구현 (Python)

```python
"""
rANS (Range ANS) 완전 구현
상태 범위: [L, bL) where L = 2^23, b = 256 (바이트)
"""
import math
from typing import List, Tuple

class RANS:
    def __init__(self, freqs: List[int]):
        """
        freqs: 각 심볼의 빈도 (합이 M = 2의 거듭제곱이어야 함)
        """
        self.M = sum(freqs)
        assert self.M & (self.M - 1) == 0, "M은 2의 거듭제곱이어야 합니다"

        self.freqs = freqs
        # 누적 빈도 (cumulative frequency)
        self.cumfreqs = [0]
        for f in freqs:
            self.cumfreqs.append(self.cumfreqs[-1] + f)

        # 디코딩 룩업 테이블: state % M -> (심볼, 오프셋 i)
        # i는 "이 심볼의 i번째 출현"을 의미
        self.decode_table = [None] * self.M
        for s, f in enumerate(freqs):
            for i in range(f):
                self.decode_table[self.cumfreqs[s] + i] = (s, i)

        self.L = 1 << 23  # 하한 L = 2^23
        self.b = 256       # 출력 단위 (1 바이트 = 256)
        self.LOG_B = 8     # log2(b)

    def encode(self, symbols: List[int]) -> Tuple[int, bytes]:
        """
        rANS 인코딩
        반환: (최종 상태, 비트스트림)
        주의: 역순으로 인코딩 (스택 구조)
        """
        state = self.L
        bitstream = bytearray()

        for s in reversed(symbols):
            f = self.freqs[s]

            # 1단계: 정규화 - 상태를 [L/M*b*f, L/M*b*b*f) 범위로 맞춤
            # 너무 크면 하위 바이트를 출력
            upper_bound = ((self.L // self.M) * self.b * f)
            while state >= upper_bound:
                bitstream.append(state & 0xFF)  # 하위 바이트 출력
                state >>= self.LOG_B            # 오른쪽으로 1바이트 시프트

            # 2단계: 인코딩 - 상태 갱신
            # x' = (x // freq[s]) * M + cumfreq[s] + (x % freq[s])
            q, r = divmod(state, f)
            state = q * self.M + self.cumfreqs[s] + r

        return state, bytes(reversed(bitstream))

    def decode(self, final_state: int, bitstream: bytes, n_symbols: int) -> List[int]:
        """
        rANS 디코딩
        주의: 비트스트림을 순방향으로 소비 (인코딩의 역순)
        """
        symbols = []
        state = final_state
        bs = bytearray(bitstream)
        pos = 0

        for _ in range(n_symbols):
            # 1단계: 심볼 결정
            slot = state % self.M
            s, i = self.decode_table[slot]
            symbols.append(s)

            # 2단계: 이전 상태 복원
            # x = freq[s] * (x' // M) + i
            state = self.freqs[s] * (state // self.M) + i

            # 3단계: 역정규화 - 비트스트림에서 바이트를 읽어 상태 복원
            while state < self.L and pos < len(bs):
                state = (state << self.LOG_B) | bs[pos]
                pos += 1

        return symbols

    def compression_stats(self, symbols: List[int]) -> dict:
        """압축 통계 계산"""
        total = len(symbols)
        # 이론적 엔트로피
        entropy = 0.0
        for s, f in enumerate(self.freqs):
            if f > 0:
                p = f / self.M
                count = symbols.count(s)
                entropy += count * (-math.log2(p))

        state, bs = self.encode(symbols)
        actual_bits = len(bs) * 8 + state.bit_length()

        return {
            'n_symbols': total,
            'theoretical_bits': entropy,
            'actual_bits': actual_bits,
            'overhead_percent': (actual_bits / entropy - 1) * 100 if entropy > 0 else 0,
            'bits_per_symbol': actual_bits / total,
        }

# --- 실험: 여러 확률 분포에서 압축 효율 비교 ---
def run_experiment():
    import random

    print("=" * 60)
    print("rANS 압축 실험")
    print("=" * 60)

    # 실험 1: 비균등 분포 (허프만이 비효율적인 경우)
    # A=12, B=4 → 비율 3:1 (M=16)
    freqs_1 = [12, 4]  # A: 75%, B: 25%
    rans_1 = RANS(freqs_1)

    # 실제 분포에 맞는 심볼 생성
    random.seed(42)
    symbols_1 = random.choices([0, 1], weights=[12, 4], k=10000)

    stats = rans_1.compression_stats(symbols_1)
    print(f"\n실험 1: A(75%) B(25%), {stats['n_symbols']}개 심볼")
    print(f"  이론 최적 비트: {stats['theoretical_bits']:.1f}")
    print(f"  실제 사용 비트: {stats['actual_bits']}")
    print(f"  오버헤드:       {stats['overhead_percent']:.4f}%")
    print(f"  심볼당 비트:    {stats['bits_per_symbol']:.4f}")

    p_A = 12/16
    huffman_bps = p_A * 1 + (1-p_A) * 1  # 허프만은 둘 다 1비트
    print(f"  허프만 심볼당:  {huffman_bps:.4f}")

    # 실험 2: 거의 균등 분포
    freqs_2 = [4, 4, 4, 4]  # A,B,C,D 각 25% (M=16)
    rans_2 = RANS(freqs_2)
    symbols_2 = random.choices([0, 1, 2, 3], k=10000)
    stats2 = rans_2.compression_stats(symbols_2)
    print(f"\n실험 2: 균등 분포 4심볼, {stats2['n_symbols']}개 심볼")
    print(f"  이론 최적 비트: {stats2['theoretical_bits']:.1f}")
    print(f"  실제 사용 비트: {stats2['actual_bits']}")
    print(f"  오버헤드:       {stats2['overhead_percent']:.4f}%")

    # 왕복 테스트 (encode → decode 검증)
    print("\n--- 왕복 검증 ---")
    test_symbols = [0, 0, 1, 0, 0, 1, 0, 0, 0, 1]
    state, bs = rans_1.encode(test_symbols)
    recovered = rans_1.decode(state, bs, len(test_symbols))
    print(f"원본:   {test_symbols}")
    print(f"복원:   {recovered}")
    print(f"일치:   {test_symbols == recovered}")

run_experiment()
```

### 예제 2: tANS 테이블 구성과 FSE 원리 (Python)

```python
"""
tANS / FSE (Finite State Entropy) 테이블 구성 원리
Zstandard가 사용하는 방식의 단순화 버전
"""
import math
from typing import List, Dict, Tuple

def spread_symbols(freqs: List[int], table_log: int) -> List[int]:
    """
    심볼을 테이블에 균등하게 분산 배치
    Zstandard의 spread 함수 구현
    
    핵심: step = (table_size >> 1) + (table_size >> 3) + 3
    이 값은 table_size와 서로소(gcd=1)를 보장하여 모든 슬롯을 채움
    """
    table_size = 1 << table_log
    step = (table_size >> 1) + (table_size >> 3) + 3
    spread = [None] * table_size

    pos = 0
    for s, f in enumerate(freqs):
        for _ in range(f):
            while spread[pos] is not None:
                pos = (pos + 1) & (table_size - 1)
            spread[pos] = s
            pos = (pos + step) & (table_size - 1)

    return spread

def build_decode_table(freqs: List[int], table_log: int) -> List[Dict]:
    """
    tANS 디코딩 테이블 구성
    각 슬롯 i에 대해: {symbol, next_state, nb_bits}
    
    상태 범위: [table_size, 2*table_size)
    슬롯 i (0 <= i < table_size) → 상태 (table_size + i)
    """
    table_size = 1 << table_log
    spread = spread_symbols(freqs, table_log)

    # 각 심볼의 순번 추적
    symbol_next = list(freqs)  # 첫 번째 출현 위치 = freq[s]

    decode_table = [None] * table_size
    for i in range(table_size):
        s = spread[i]
        # 이 슬롯에서 s를 디코딩할 때의 다음 상태
        # next_state = symbol_next[s]; symbol_next[s]++
        x = symbol_next[s]
        symbol_next[s] += 1

        # 다음 상태 x에서 몇 비트를 읽어야 정규화 범위 [L, 2L)로 돌아올지 계산
        # nb_bits = table_log - floor(log2(x))
        nb_bits = table_log - x.bit_length() + 1

        decode_table[i] = {
            'symbol': s,
            'next_state': x,       # 역정규화 전 베이스 상태
            'nb_bits': nb_bits,    # 읽어야 할 비트 수
        }

    return decode_table

def build_encode_table(freqs: List[int], table_log: int) -> List[Dict]:
    """
    tANS 인코딩 테이블 구성
    상태 x에서 심볼 s를 인코딩할 때 출력할 비트 수와 새 슬롯 계산
    """
    table_size = 1 << table_log
    decode_table = build_decode_table(freqs, table_log)

    # 디코딩 테이블로부터 역방향으로 인코딩 테이블 구성
    encode_table = [[None] * table_size for _ in range(len(freqs))]

    for slot, entry in enumerate(decode_table):
        s = entry['symbol']
        base_x = entry['next_state']
        nb_bits = entry['nb_bits']

        # 상태 범위: [base_x * 2^nb_bits, base_x * 2^(nb_bits+1))
        for extra in range(1 << nb_bits):
            x = (base_x << nb_bits) | extra
            if freqs[s] <= x < 2 * freqs[s]:
                encode_table[s][x - freqs[s]] = {
                    'nb_bits': nb_bits,
                    'next_slot': slot,  # 다음 상태 = table_size + slot
                }

    return encode_table

def visualize_tans(freqs: List[int], table_log: int):
    """tANS 테이블 시각화"""
    table_size = 1 << table_log
    spread = spread_symbols(freqs, table_log)
    decode_table = build_decode_table(freqs, table_log)

    print(f"테이블 크기: {table_size} (2^{table_log})")
    print(f"심볼 빈도:  {freqs}")
    print(f"총 M = {sum(freqs)}")
    print()

    # 심볼 배치 시각화
    names = "ABCDEFGH"
    print("심볼 배치 (spread):")
    print("  슬롯:  " + " ".join(f"{i:2d}" for i in range(table_size)))
    print("  심볼:  " + " ".join(f" {names[s]}" if s is not None else " ?" for s in spread))
    print()

    # 각 심볼의 정보 이론적 코드 길이
    total = sum(freqs)
    print("심볼별 정보:")
    for s, f in enumerate(freqs):
        p = f / total
        ideal = -math.log2(p) if p > 0 else float('inf')
        # 실제 tANS 코드 길이 (평균)
        actual_bits = []
        for entry in decode_table:
            if entry['symbol'] == s:
                actual_bits.append(entry['nb_bits'])
        avg_bits = sum(actual_bits) / len(actual_bits) if actual_bits else 0
        print(f"  심볼 {names[s]}: p={p:.3f}, 이상적={ideal:.3f}비트, "
              f"tANS 평균={avg_bits:.3f}비트")

    # 엔트로피
    entropy = -sum((f/total) * math.log2(f/total) for f in freqs if f > 0)
    print(f"\n엔트로피: {entropy:.4f} bits/symbol")

# 시연
print("=" * 55)
print("실험 1: 비균등 분포 (Zstandard 전형적 사례)")
print("=" * 55)
# A=12, B=4 (합=16=2^4)
visualize_tans([12, 4], table_log=4)

print("\n" + "=" * 55)
print("실험 2: 4개 심볼 (합=16=2^4)")
print("=" * 55)
# A=8, B=4, C=3, D=1 (합=16=2^4)
visualize_tans([8, 4, 3, 1], table_log=4)
```

## 주의사항과 팁

### 1. 정규화(Renormalization) 범위 설계

rANS에서 상태를 허용 범위 `[L, bL)`에 유지하는 것이 핵심입니다:
- **너무 작은 L**: 출력 비트 수가 늘어 비효율
- **너무 큰 L**: 큰 초기 상태가 필요해 오버헤드 증가
- Zstandard는 `L = 1 << 23`, `b = 256`을 사용합니다

### 2. tANS의 역방향 디코딩과 인터리빙

tANS는 본질적으로 **LIFO(스택)** 구조입니다. 인코딩 순서의 역순으로 디코딩됩니다. Zstandard는 이 문제를 **인터리빙 스트림(2개 또는 4개)**으로 해결합니다:

```
스트림 1: 짝수 번째 심볼 인코딩
스트림 2: 홀수 번째 심볼 인코딩
→ 디코딩 시 두 스트림을 교대로 처리
→ 순서 복원 + 현대 CPU의 비순차 실행 활용
```

### 3. 빈도 정규화의 함정

실제 빈도를 `M = 2^table_log`에 맞게 정규화할 때:
- **빈도가 0이 되는 심볼**: 그 심볼은 절대 디코딩 불가 → 반드시 최소 빈도 1 보장
- **반올림 오차 누적**: 정규화 후 빈도의 합이 정확히 M이어야 함
- Zstandard는 큰 빈도부터 처리하고 나머지를 마지막 심볼에 할당합니다

### 4. SIMD 최적화

tANS는 나눗셈이 없어 SIMD로 여러 심볼을 병렬 처리하기 좋습니다:
```
4개 인터리빙 스트림 × 4 심볼/사이클 = 초당 수 GB 처리
(현대 CPU 기준: 단일 코어 ~1-5 GB/s 디코딩 속도)
```

### 5. 실제 사용 사례와 선택 기준

| 상황 | 권장 |
|------|------|
| 정적 확률 분포 (LZ77 뒤단) | tANS (Zstandard FSE) |
| 적응적(adaptive) 압축 | rANS |
| 초고속 트레이드오프 필요 | tANS + SIMD |
| 작은 메모리 환경 | rANS (테이블 불필요) |
| 범용 파일 압축 | Zstandard (내부적으로 tANS) |

### 6. ANS와 Huffman의 공존

현실에서는 ANS가 허프만을 완전히 대체하지 않습니다. 매우 짧은 메시지나 고정 테이블이 필요한 경우 허프만이 더 단순하고 효율적일 수 있습니다. Zstandard 자체도 특정 조건(매우 짧은 블록)에서는 허프만을 선택합니다.

## 참고 자료

- [Asymmetric Numeral Systems - Wikipedia](https://en.wikipedia.org/wiki/Asymmetric_numeral_systems)
- [Jarek Duda의 ANS 원본 논문 (arXiv:0902.0271)](https://arxiv.org/abs/0902.0271)
- [Zstandard 공식 문서](https://facebook.github.io/zstd/)
- [FSE (Finite State Entropy) 구현 - GitHub](https://github.com/Cyan4973/FiniteStateEntropy)
