---
layout: post
title: "의사 난수 생성기(PRNG) 완전 정복: Mersenne Twister, PCG, Xoshiro의 내부 구조와 선택 기준"
date: 2026-11-04
categories: [cs, computer-science]
tags: [prng, random-number-generator, mersenne-twister, pcg, xoshiro, algorithms, simulation, statistics]
---

## 개요

난수(Random Number)는 시뮬레이션, 암호학, 게임, 머신러닝, 통계적 샘플링 등 컴퓨터 과학의 핵심 여러 분야에서 필수적이다. 그러나 디지털 컴퓨터는 결정론적(deterministic) 기계이므로 진정한 난수를 생성할 수 없다. 대신 초기 **시드(seed)** 값에서 출발해 수학적 연산으로 난수처럼 보이는 수열을 만드는 **의사 난수 생성기(PRNG: Pseudo-Random Number Generator)**를 사용한다.

좋은 PRNG는 다음 조건을 만족해야 한다:
- **긴 주기**: 같은 수열이 반복되기 전까지의 길이가 충분히 길어야 한다
- **균등 분포**: 생성된 수가 출력 범위에 걸쳐 균등하게 분포해야 한다
- **고차원 균등분포**: 연속된 k개의 출력을 k차원 공간에 뿌렸을 때 균등하게 분포해야 한다
- **빠른 속도**: 대용량 시뮬레이션에서 병목이 되지 않아야 한다
- **재현성**: 동일 시드로 동일한 수열을 재현할 수 있어야 한다

이 글에서는 역사적으로 가장 많이 쓰인 **Mersenne Twister**와 현대적 대안인 **PCG**, **Xoshiro** 패밀리를 비교 분석한다.

---

## 선형 합동 생성기 (LCG): 역사적 출발점

가장 단순한 PRNG는 1948년 D.H. Lehmer가 제안한 **선형 합동 생성기(LCG)**다:

```
Xₙ₊₁ = (a · Xₙ + c) mod m
```

구현은 단 한 줄이지만 품질이 나쁘다. `java.util.Random`이 사용하는 LCG의 경우, 연속된 두 출력값의 쌍 `(Xₙ, Xₙ₊₁)`을 평면에 뿌리면 뚜렷한 줄무늬(lattice structure)가 나타난다. 주기가 m(보통 2³²~2⁴⁸)으로 제한되며, 하위 비트의 품질이 특히 나쁘다.

---

## Mersenne Twister (MT19937): 오랫동안의 표준

### 개요

1998년 Matsumoto와 Nishimura가 발표한 **MT19937**은 메르센 소수 2¹⁹⁹³⁷-1을 주기로 사용하는 PRNG다. Python의 `random` 모듈, R의 기본 PRNG, Ruby, PHP 등 수많은 언어와 프레임워크가 MT19937을 기본으로 채택했다.

**핵심 특성**:
- 주기: 2¹⁹⁹³⁷ - 1 (천문학적으로 긴 주기)
- 상태 크기: 624개의 32비트 정수 = 2496 바이트
- 균등분포 차원: 623차원까지 균등분포 보장 (k-distribution)
- 출력: 32비트 또는 64비트 정수

### 내부 구조

MT는 선형 되먹임 시프트 레지스터(LFSR)를 기반으로 하며, 세 단계로 작동한다:

1. **Twist (비틀기)**: 624개의 상태 단어를 행렬 선형 점화식으로 갱신
2. **Temper (조절)**: 하나의 상태 단어에 비트 연산을 적용해 출력값의 균등분포를 개선
3. **Generate**: 32비트 정수를 반환

```
# Twist 단계 (의사 코드)
for i in range(624):
    y = (MT[i] & 0x80000000) + (MT[(i+1) % 624] & 0x7fffffff)
    MT[i] = MT[(i + 397) % 624] ^ (y >> 1)
    if y % 2 != 0:
        MT[i] ^= 0x9908b0df

# Temper 단계
def temper(y):
    y ^= (y >> 11)
    y ^= (y << 7) & 0x9d2c5680
    y ^= (y << 15) & 0xefc60000
    y ^= (y >> 18)
    return y
```

### MT의 약점

MT는 긴 주기와 고차원 균등분포를 보장하지만 현대 기준에서 여러 단점이 있다:

- **큰 상태**: 2496 바이트는 L1 캐시 미스를 유발할 수 있다
- **통계적 약점**: TestU01 BigCrush 테스트에서 여러 항목 실패
- **예측 가능성**: 출력값 624개만 관측하면 내부 상태를 완전히 복원 가능 (암호학적으로 완전히 부적합)
- **느린 초기화**: 복잡한 초기화 과정

---

## PCG (Permuted Congruential Generator)

### 개요

2014년 Melissa E. O'Neill이 제안한 **PCG**는 "LCG의 단순함 + 출력 함수의 고품질화"라는 아이디어에서 출발한다. LCG는 구조가 단순해 빠르지만 품질이 나쁘다. PCG는 LCG를 내부 상태로 사용하되, 출력 단계에서 **Permutation 함수**를 적용해 품질을 극적으로 향상시킨다.

**핵심 특성**:
- 상태 크기: PCG32는 64비트 상태 (8 바이트), PCG64는 128비트 상태
- 주기: 2⁶⁴ (PCG32) 또는 2¹²⁸ (PCG64)
- 복수 스트림: 같은 시드에서 다른 `increment` 값으로 독립적인 스트림 생성 가능
- 통계 품질: TestU01 BigCrush 완전 통과

### PCG32 구현

```c
#include <stdint.h>
#include <stdio.h>

typedef struct {
    uint64_t state;
    uint64_t inc;    // 반드시 홀수여야 함
} pcg32_random_t;

// PCG32 초기화
void pcg32_srandom(pcg32_random_t *rng, uint64_t init_state, uint64_t init_seq) {
    rng->state = 0U;
    rng->inc = (init_seq << 1u) | 1u;  // inc는 항상 홀수
    pcg32_random_r(rng);               // 더미 호출
    rng->state += init_state;
    pcg32_random_r(rng);               // 더미 호출
}

// PCG32 난수 생성 (핵심 함수)
uint32_t pcg32_random_r(pcg32_random_t *rng) {
    uint64_t oldstate = rng->state;
    
    // LCG 상태 전진
    rng->state = oldstate * 6364136223846793005ULL + rng->inc;
    
    // XSH-RR 출력 함수: 상위 비트를 하위 비트에 XOR하고 회전
    uint32_t xorshifted = ((oldstate >> 18u) ^ oldstate) >> 27u;
    uint32_t rot = oldstate >> 59u;
    return (xorshifted >> rot) | (xorshifted << ((-rot) & 31));
}

int main() {
    pcg32_random_t rng;
    pcg32_srandom(&rng, 42u, 54u);
    
    printf("PCG32 난수 10개:\n");
    for (int i = 0; i < 10; i++) {
        printf("%u\n", pcg32_random_r(&rng));
    }
    return 0;
}
```

**출력 함수 XSH-RR 설명**:
- **XSH(XorShift-High)**: 상위 비트를 사용해 하위 비트를 XorShift. LCG의 약한 하위 비트를 개선
- **RR(Random Rotation)**: 최상위 비트를 사용해 회전 거리를 결정. 출력 비트 모두가 균등하게 활용됨

### PCG의 장점

PCG32는 PCG64 등 여러 변형이 있으며, `increment`를 다르게 설정하면 2⁶² 개의 독립적인 스트림을 생성할 수 있다. 이는 병렬 시뮬레이션에서 각 스레드/프로세스가 독립적인 난수 스트림을 갖는 데 유용하다.

---

## Xoshiro / Xorshift 패밀리

### 개요

Sebastiano Vigna와 David Blackman이 개발한 **Xoshiro(XOR/Shift/Rotate)** 시리즈는 최소한의 연산으로 최고의 통계 품질을 추구한다. NumPy, Julia, Rust의 기본 PRNG로 채택되었다.

대표적인 변형:
- **xoshiro256****: 256비트 상태, 64비트 출력 → 범용 목적
- **xoshiro128****: 128비트 상태, 32비트 출력 → 임베디드/게임
- **xoroshiro128+**: 128비트 상태, 부동소수점 생성에 최적
- **splitmix64**: 초기화 용도로 사용

### xoshiro256** 구현 (Python)

```python
class Xoshiro256StarStar:
    """
    xoshiro256** PRNG
    상태: 4개의 64비트 정수
    주기: 2^256 - 1
    출력: 64비트 정수
    """
    def __init__(self, seed: int):
        # splitmix64로 초기 상태 생성
        self.s = [0, 0, 0, 0]
        state = seed & 0xFFFFFFFFFFFFFFFF
        for i in range(4):
            state += 0x9e3779b97f4a7c15
            state &= 0xFFFFFFFFFFFFFFFF
            z = state
            z = (z ^ (z >> 30)) * 0xbf58476d1ce4e5b9 & 0xFFFFFFFFFFFFFFFF
            z = (z ^ (z >> 27)) * 0x94d049bb133111eb & 0xFFFFFFFFFFFFFFFF
            self.s[i] = z ^ (z >> 31)

    def _rotl(self, x: int, k: int) -> int:
        return ((x << k) | (x >> (64 - k))) & 0xFFFFFFFFFFFFFFFF

    def next_u64(self) -> int:
        s = self.s
        result = self._rotl(s[1] * 5 & 0xFFFFFFFFFFFFFFFF, 7) * 9 & 0xFFFFFFFFFFFFFFFF

        t = (s[1] << 17) & 0xFFFFFFFFFFFFFFFF
        s[2] ^= s[0]
        s[3] ^= s[1]
        s[1] ^= s[2]
        s[0] ^= s[3]
        s[2] ^= t
        s[3] = self._rotl(s[3], 45)

        return result

    def next_float(self) -> float:
        """[0, 1) 범위의 부동소수점 생성"""
        return (self.next_u64() >> 11) * (1.0 / (1 << 53))


# --- 테스트 ---
rng = Xoshiro256StarStar(seed=42)

print("xoshiro256** 난수 10개 (정수):")
for _ in range(5):
    print(f"  {rng.next_u64()}")

rng2 = Xoshiro256StarStar(seed=42)
print("\n[0,1) 부동소수점 5개:")
for _ in range(5):
    print(f"  {rng2.next_float():.6f}")
```

**Jump-Ahead 기능**: xoshiro는 2^128 또는 2^192 단계를 O(log(jump_size)) 연산으로 건너뛸 수 있다. 이를 활용하면 각 스레드가 독립적인 서브시퀀스를 사용하는 병렬 PRNG를 쉽게 구성할 수 있다.

---

## 세 PRNG 비교 분석

| 특성 | MT19937 | PCG32 | xoshiro256** |
|------|---------|-------|--------------|
| 상태 크기 | 2496 B | 8 B | 32 B |
| 주기 | 2¹⁹⁹³⁷-1 | 2⁶⁴ | 2²⁵⁶-1 |
| 속도 | 보통 | 빠름 | 매우 빠름 |
| BigCrush 통과 | 실패(일부) | 완전 통과 | 완전 통과 |
| 복수 스트림 | 어려움 | 쉬움(inc 변경) | Jump 필요 |
| Jump-Ahead | 없음 | 없음 | 있음 |
| 암호학 적합 | 완전 부적합 | 완전 부적합 | 완전 부적합 |
| 사용처 | 레거시 호환 | 범용, 게임 | 과학 계산, ML |

---

## PRNG 선택 가이드

1. **보안이 필요한 경우**: PRNG는 사용하면 안 된다. `os.urandom()`, `/dev/urandom`, AES-CTR-DRBG 등 CSPRNG를 사용한다.

2. **재현 가능한 과학 시뮬레이션**: xoshiro256** 또는 PCG64. NumPy 1.17+는 `numpy.random.default_rng()`로 xoshiro256**을 사용한다.

3. **병렬 시뮬레이션**: PCG(복수 스트림)나 xoshiro(Jump)를 각 워커에 독립적으로 할당한다.

4. **레거시 호환성**: Python `random`, R `set.seed()` 등은 MT19937 기반이므로 재현성 목적으로 유지한다.

5. **임베디드/게임**: xoshiro128** 또는 Splitmix64 (초경량 상태).

---

## 주의사항

- **시드 관리**: 같은 시드는 항상 같은 수열을 생성한다. 보안과 무관한 목적이라도 시드를 로그에 기록하면 나중에 재현할 수 있다.
- **공유 상태**: 멀티스레드 환경에서 하나의 PRNG 인스턴스를 공유하면 경쟁 조건이 발생한다. 스레드당 독립 인스턴스를 사용한다.
- **모듈로 편향(modulo bias)**: `rand() % N`은 N이 2의 거듭제곱이 아닐 때 불균등 분포를 만든다. 거부 샘플링(rejection sampling)으로 해결한다.

---

## 참고 자료
- [PCG 공식 사이트: pcg-random.org](https://www.pcg-random.org/)
- [xoshiro/xorshift 공식 페이지 (Vigna)](https://prng.di.unimi.it/)
- [Matsumoto & Nishimura (1998): Mersenne Twister — ACM TOMACS](https://dl.acm.org/doi/10.1145/272991.272995)
- [NumPy: Random Generator 공식 문서](https://numpy.org/doc/stable/reference/random/generator.html)
