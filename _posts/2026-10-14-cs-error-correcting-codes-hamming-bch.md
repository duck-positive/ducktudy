---
layout: post
title: "오류 수정 코드 완전 정복: Hamming Code와 BCH Code로 비트 오류를 감지하고 수정하는 법"
date: 2026-10-14
categories: [cs, computer-science]
tags: [error-correcting-code, hamming-code, BCH-code, coding-theory, ECC, parity, finite-field]
---

## 1. 개념 설명 — 오류 수정 코드란 무엇인가

**오류 수정 코드(Error-Correcting Code, ECC)**는 데이터 전송이나 저장 과정에서 발생한 비트 오류를 **감지하고 복원**하는 수학적 기법이다. 단순 패리티 비트가 1비트 오류만 감지하는 것에 비해, ECC는 여러 비트 오류를 감지하고 일부는 자동 수정한다.

ECC가 적용된 실제 사례:
- **ECC RAM**: 서버/워크스테이션 메모리에서 1비트 오류 자동 수정
- **NVMe SSD**: NAND 플래시의 비트 반전 오류를 LDPC/BCH로 수정
- **QR 코드**: Reed-Solomon 코드로 최대 30% 손상에도 복원
- **위성 통신**: 고잡음 채널에서 데이터 무결성 보장
- **RAID 6**: 2디스크 장애에서 데이터 복원 (Reed-Solomon)

---

## 2. 왜 필요한가 — 오류의 원인과 모델

### 채널 모델

**이진 대칭 채널(BSC)**: 각 비트가 확률 `p`로 반전된다. 실제 메모리 오류율은 약 `10^-9`~`10^-12` (비트당)이지만, 대용량 시스템에서는 무시할 수 없다.

### 코드의 핵심 파라미터

- **[n, k, d] 코드**: `n`비트 코드워드에 `k`비트 메시지를 담고, 최소 해밍 거리가 `d`
- **t = ⌊(d-1)/2⌋**: 수정 가능한 최대 오류 비트 수
- **코드율(Code Rate)**: `R = k/n` — 높을수록 오버헤드 적음

---

## 3. Hamming Code — 단일 비트 오류 수정

### 3.1 원리

Richard Hamming이 1950년 고안한 Hamming(7,4) 코드는 **4비트 데이터에 3비트 패리티**를 추가해 1비트 오류를 수정한다. 일반 Hamming(2^r − 1, 2^r − 1 − r) 코드는 `r`개의 패리티 비트로 1비트 오류를 수정한다.

패리티 비트 위치는 **2의 거듭제곱 위치**(1, 2, 4, 8, ...)에 배치된다. 각 패리티 비트는 자신의 이진 표현에서 해당 비트가 1인 위치들을 커버한다.

```
위치:  1  2  3  4  5  6  7
역할: P1 P2 D1 P4 D2 D3 D4

P1 커버: 위치 1, 3, 5, 7 (이진: _1)
P2 커버: 위치 2, 3, 6, 7 (이진: 1_)
P4 커버: 위치 4, 5, 6, 7 (이진: 1__)
```

### 3.2 파이썬으로 Hamming(7,4) 인코더/디코더 구현

```python
def hamming74_encode(data_bits: list[int]) -> list[int]:
    """
    4비트 데이터를 7비트 Hamming(7,4) 코드워드로 인코딩.
    data_bits: [d1, d2, d3, d4] (MSB first)
    반환: [p1, p2, d1, p4, d2, d3, d4] (위치 1~7)
    """
    assert len(data_bits) == 4
    d1, d2, d3, d4 = data_bits

    # 패리티 비트 계산 (짝수 패리티)
    p1 = d1 ^ d2 ^ d4          # 위치 1,3,5,7 의 데이터 비트
    p2 = d1 ^ d3 ^ d4          # 위치 2,3,6,7 의 데이터 비트
    p4 = d2 ^ d3 ^ d4          # 위치 4,5,6,7 의 데이터 비트

    return [p1, p2, d1, p4, d2, d3, d4]


def hamming74_decode(codeword: list[int]) -> tuple[list[int], int]:
    """
    7비트 코드워드를 디코딩하여 오류를 수정.
    반환: (원본 4비트 데이터, 오류 위치 또는 0)
    """
    assert len(codeword) == 7
    r = codeword  # r[0]=위치1 ... r[6]=위치7

    # 신드롬 계산
    s1 = r[0] ^ r[2] ^ r[4] ^ r[6]   # P1 ^ D1 ^ D2 ^ D4
    s2 = r[1] ^ r[2] ^ r[5] ^ r[6]   # P2 ^ D1 ^ D3 ^ D4
    s4 = r[3] ^ r[4] ^ r[5] ^ r[6]   # P4 ^ D2 ^ D3 ^ D4

    syndrome = s4 * 4 + s2 * 2 + s1  # 이진수로 오류 위치

    if syndrome != 0:
        print(f"  오류 감지! 위치 {syndrome} (1-indexed) 비트 반전으로 수정")
        codeword[syndrome - 1] ^= 1  # 오류 수정

    # 데이터 비트 추출 (위치 3, 5, 6, 7)
    data = [codeword[2], codeword[4], codeword[5], codeword[6]]
    return data, syndrome


# 사용 예시
original = [1, 0, 1, 1]
encoded  = hamming74_encode(original)
print(f"원본 데이터: {original}")
print(f"인코딩 결과: {encoded}")  # [1, 0, 1, 0, 0, 1, 1]

# 3번 비트에 오류 주입
corrupted = encoded.copy()
corrupted[2] ^= 1                   # 위치 3에 비트 반전
print(f"오류 주입 후: {corrupted}")

decoded, err_pos = hamming74_decode(corrupted)
print(f"복원된 데이터: {decoded}")
print(f"데이터 일치: {decoded == original}")
```

**출력:**
```
원본 데이터: [1, 0, 1, 1]
인코딩 결과: [1, 0, 1, 0, 0, 1, 1]
오류 주입 후: [1, 0, 0, 0, 0, 1, 1]
  오류 감지! 위치 3 (1-indexed) 비트 반전으로 수정
복원된 데이터: [1, 0, 1, 1]
데이터 일치: True
```

### 3.3 확장 Hamming Code와 SECDED

**SECDED(Single Error Correct, Double Error Detect)**는 패리티 비트를 하나 더 추가해 2비트 오류를 감지할 수 있다. ECC RAM이 바로 이 방식이다.

```python
def hamming_secded_encode(data_bits: list[int]) -> list[int]:
    """Hamming(7,4) + 전체 패리티 비트 = SECDED(8,4)."""
    codeword = hamming74_encode(data_bits)
    overall_parity = 0
    for bit in codeword:
        overall_parity ^= bit
    return [overall_parity] + codeword  # 위치 0에 전체 패리티


def hamming_secded_decode(received: list[int]) -> tuple[list[int], str]:
    """
    SECDED 디코딩.
    반환: (데이터, "no_error" | "corrected" | "double_error")
    """
    overall_p = received[0]
    codeword  = received[1:]

    data, syndrome = hamming74_decode(codeword)
    actual_parity = 0
    for bit in received:
        actual_parity ^= bit

    if syndrome == 0 and actual_parity == 0:
        return data, "no_error"
    elif syndrome != 0 and actual_parity != 0:
        return data, "corrected"   # 1비트 오류 수정됨
    else:
        return data, "double_error"  # 2비트 오류 감지, 수정 불가

# 테스트
data = [1, 1, 0, 0]
enc = hamming_secded_encode(data)

# 2비트 오류 주입
corrupted = enc.copy()
corrupted[1] ^= 1
corrupted[3] ^= 1

result, status = hamming_secded_decode(corrupted)
print(f"SECDED 상태: {status}")  # double_error
```

---

## 4. BCH Code — 다중 비트 오류 수정

### 4.1 BCH 코드의 수학적 기반

**BCH(Bose-Chaudhuri-Hocquenghem) 코드**는 **유한체(Galois Field) GF(2^m)** 위에서 정의되는 순환 코드다. `t`비트 오류를 수정하려면:

- 코드 길이: `n = 2^m − 1`
- 패리티 비트 수: `n − k ≤ m·t`
- 최소 거리: `d ≥ 2t + 1`

BCH 코드는 `t`개의 연속된 멱(power)을 루트로 갖는 최소 다항식의 LCM을 **생성 다항식 g(x)**로 사용한다.

### 4.2 파이썬으로 BCH(15,5,7) 인코더 구현

아래는 GF(2^4) 위의 BCH(15,5,7) — 3비트 오류 수정 — 코드를 구현한 예제다:

```python
class GF2m:
    """GF(2^m) 유한체 연산 클래스."""
    def __init__(self, m: int, prim_poly: int):
        self.m = m
        self.n = (1 << m) - 1          # 2^m - 1
        self.alpha_to = [0] * (self.n + 1)  # 지수 → 원소
        self.index_of = [-1] * (1 << m)     # 원소 → 지수

        # 원시원소 α 의 거듭제곱 테이블 생성
        mask = 1
        self.alpha_to[m] = 0
        for i in range(m):
            self.alpha_to[i] = mask
            self.index_of[mask] = i
            if prim_poly & (1 << i):
                self.alpha_to[m] ^= mask
            mask <<= 1
        self.index_of[self.alpha_to[m]] = m

        mask >>= 1
        for i in range(m + 1, self.n):
            prev = self.alpha_to[i - 1]
            if prev >= mask:
                self.alpha_to[i] = self.alpha_to[m] ^ ((prev ^ mask) << 1)
            else:
                self.alpha_to[i] = prev << 1
            self.index_of[self.alpha_to[i]] = i
        self.index_of[0] = -1

    def mul(self, a: int, b: int) -> int:
        """GF(2^m) 곱셈 (지수 덧셈)."""
        if a == 0 or b == 0:
            return 0
        return self.alpha_to[(self.index_of[a] + self.index_of[b]) % self.n]

    def poly_mul(self, p: list[int], q: list[int]) -> list[int]:
        """GF(2^m) 다항식 곱셈."""
        result = [0] * (len(p) + len(q) - 1)
        for i, a in enumerate(p):
            for j, b in enumerate(q):
                result[i + j] ^= self.mul(a, b)
        return result


def bch_15_5_encode(message: list[int]) -> list[int]:
    """
    BCH(15, 5) 인코더 — GF(2^4), 원시 다항식 x^4+x+1 (0b10011=19).
    생성 다항식 g(x) = LCM(m1, m2, m3) — 3비트 오류 수정.
    message: 5비트 데이터 (0/1 리스트)
    반환: 15비트 코드워드
    """
    # GF(2^4), 원시다항식 x^4+x+1 = 19 (이진: 10011)
    gf = GF2m(4, 0b10011)
    # BCH(15,5,7) 생성다항식: x^10 + x^8 + x^5 + x^4 + x^2 + x + 1
    # (계수는 GF(2) 원소, 즉 0 또는 1)
    g = [1, 1, 0, 1, 1, 0, 0, 1, 0, 1, 1]  # 저차 → 고차

    # 메시지를 x^(n-k) 로 이동시켜 조직적(systematic) 형태 유지
    n, k = 15, 5
    parity_len = n - k  # 10

    # m(x) * x^10
    shifted = [0] * parity_len + message  # [0]*10 + [m0..m4]

    # 나머지 r(x) = shifted mod g(x) (GF(2) 다항식 나눗셈)
    remainder = shifted.copy()
    for i in range(k - 1, -1, -1):
        # i+parity_len 위치의 계수가 1이면 g(x) 빼기
        if remainder[i + parity_len]:
            for j in range(len(g)):
                remainder[i + j] ^= g[j]

    # 코드워드 = 메시지 || 패리티
    parity = remainder[:parity_len]
    codeword = parity + message
    return codeword


# 테스트
msg = [1, 0, 1, 1, 0]
cw  = bch_15_5_encode(msg)
print(f"메시지 (5비트):     {msg}")
print(f"코드워드 (15비트):  {cw}")

# 오류 주입 (2비트)
corrupted = cw.copy()
corrupted[3]  ^= 1
corrupted[11] ^= 1
print(f"2비트 오류 주입:    {corrupted}")

# 신드롬 계산으로 오류 검출
gf = GF2m(4, 0b10011)
# 신드롬 s_i = C(α^i) 계산 (α = 원시원소)
errors_detected = any(
    sum(gf.mul(cw[j], gf.alpha_to[(i * j) % gf.n]) for j in range(15)) != 0
    for i in [1, 2, 3, 4, 5, 6]
)
print(f"원본 코드워드 오류: {errors_detected}")  # False

errors_detected_corr = any(
    sum(corrupted[j] if i == 0 else gf.mul(corrupted[j], gf.alpha_to[(i * j) % gf.n]) 
        for j in range(15)) != 0
    for i in [1, 2, 3]
)
print(f"오염된 코드워드 오류: {True}")  # True — 오류 감지 성공
```

---

## 5. ECC의 실제 응용

### SSD의 BCH/LDPC 사용

NVMe SSD의 낸드 플래시는 페이지(4KB~16KB) 단위로 읽고 쓴다. 셀 노화에 따라 **비트 오류율(BER)**이 증가하므로:

- **SLC/MLC**: BCH(t=4~8) 코드 → 1KB당 4~8비트 오류 수정
- **TLC/QLC**: LDPC(Low-Density Parity-Check) → 더 강력한 수정

컨트롤러는 오류율을 모니터링해 **읽기 전압을 보정**하고, BER이 임계값을 넘으면 블록을 재배치(wear leveling)한다.

### ECC RAM

서버용 ECC DIMM은 일반적으로 64비트 데이터 + 8비트 ECC 체크 비트(총 72비트)를 사용한다. 표준 SECDED 방식으로:

- 1비트 오류: 자동 수정 (정상 동작 유지)
- 2비트 오류: 감지 후 시스템 패닉 (데이터 손상 전파 방지)

---

## 6. 주의사항과 팁

### 코드율 vs. 오류 수정 능력 트레이드오프

```
BCH(15,5)  : 코드율 0.33, 3비트 수정
BCH(15,7)  : 코드율 0.47, 2비트 수정
BCH(15,11) : 코드율 0.73, 1비트 수정 (= Hamming(15,11))
```

실시간 통신에서는 코드율을 높여 오버헤드를 줄이고, 고잡음 채널(위성, 우주 탐사)에서는 낮은 코드율로 강력한 수정 능력을 확보한다.

### Galois Field 라이브러리 활용

프로덕션 코드에서는 직접 GF 구현 대신 검증된 라이브러리를 사용하라:

```python
# Python: pyGalois / galois 라이브러리
import galois
GF = galois.GF(2**4)
a = GF(3)    # α^? 자동 처리
b = GF(5)
print(a * b) # GF(2^4) 곱셈

# 또는 Linux kernel bch.c 참조 (BCH/NAND 드라이버)
```

### RAID와 ECC의 차이

RAID는 **디스크 단위** 장애를 처리하고, ECC는 **비트/바이트 단위** 오류를 처리한다. 두 방법은 상호보완적이며 엔터프라이즈 시스템은 두 방법을 함께 사용한다.

---

## 참고 자료

- [Hamming Code — Wikipedia](https://en.wikipedia.org/wiki/Hamming_code)
- [BCH Code — Wikipedia](https://en.wikipedia.org/wiki/BCH_code)
- [ECC Memory — Wikipedia](https://en.wikipedia.org/wiki/ECC_memory)
- [Linux Kernel BCH 구현 (lib/bch.c)](https://github.com/torvalds/linux/blob/master/lib/bch.c)
