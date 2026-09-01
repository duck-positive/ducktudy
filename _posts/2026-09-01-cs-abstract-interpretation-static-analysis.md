---
layout: post
title: "추상 해석(Abstract Interpretation)과 정적 분석 완전 정복: 컴파일러가 버그를 자동으로 찾는 수학적 원리"
date: 2026-09-01
categories: [cs, computer-science]
tags: [static-analysis, abstract-interpretation, program-verification, lattice-theory, compiler, Clang, sanitizers]
---

## 개요

**추상 해석(Abstract Interpretation)**은 1977년 Patrick Cousot과 Radhia Cousot 부부가 발표한 이론으로, 프로그램을 실제로 실행하지 않고도 그 동작을 수학적으로 분석하는 프레임워크다. 오늘날 Clang Static Analyzer, Facebook Infer, Google AddressSanitizer(컴파일 타임 계측), NASA의 항공기 소프트웨어 검증 등 다양한 정적 분석 도구의 이론적 토대다.

핵심 아이디어는 단순하다: **정확한 실행값 대신, 값들의 집합을 표현하는 추상 값(abstract value)으로 프로그램을 해석한다.** 예를 들어 "이 변수가 양수인지 음수인지 0인지"만 추적하는 **부호 분석(Sign Analysis)**, 또는 "이 변수가 [a, b] 범위에 있다"는 **구간 분석(Interval Analysis)**이 추상 해석의 구체적인 사례다.

---

## 왜 추상 해석이 필요한가

### 테스팅의 한계

단위 테스트는 특정 입력에 대한 올바름만 확인한다. 모든 가능한 입력을 테스트하는 것은 불가능하다. 정수 하나만 해도 약 40억 가지 값이 존재한다.

### 정적 분석의 두 가지 목표

- **건전성(Soundness)**: 분석이 "안전하다"고 말하면 실제로도 안전해야 한다. 즉, **거짓 음성(False Negative)이 없어야 한다.** 버그를 놓치면 안 된다.
- **완전성(Completeness)**: 분석이 "위험하다"고 말하면 실제로도 위험해야 한다. 즉, **거짓 양성(False Positive)이 없어야 한다.** 실제 버그가 아닌데 경고를 내면 안 된다.

라이스(Rice) 정리에 의해 **비자명(non-trivial) 프로그램 성질을 100% 정확하게 판단하는 알고리즘은 존재하지 않는다.** 따라서 실용적인 도구는 건전성이나 완전성 중 하나를 희생한다.

- AddressSanitizer, Valgrind → 실제 실행에서만 탐지, 완전성 희생
- Clang Static Analyzer, Facebook Infer → **건전성 우선**, 거짓 양성 허용
- 추상 해석 기반 검증 도구(Astrée 등) → **완전 건전**, 거짓 음성 0 보장

---

## 수학적 기반: 격자와 갈루아 연결

### 완전 격자(Complete Lattice)

추상 도메인은 완전 격자 `(D, ⊑, ⊔, ⊓, ⊥, ⊤)` 으로 표현된다.

- `D`: 추상 값들의 집합 (예: `{음수, 영, 양수, TOP, BOT}`)
- `⊑`: 정밀도 순서 (더 구체적인 정보를 아래에 배치)
- `⊔`: 상한(join, least upper bound) — 두 추상 값을 합칠 때 사용
- `⊥`: 최소 원소 (불가능한 상태, "dead code")
- `⊤`: 최대 원소 (아무것도 모른다, "unknown")

### 갈루아 연결(Galois Connection)

구체 도메인(Concrete Domain) `C`와 추상 도메인(Abstract Domain) `A` 사이의 갈루아 연결은 두 함수 쌍 `(α, γ)` 으로 정의된다.

- `α: P(C) → A`: **추상화 함수(Abstraction)** — 구체 값 집합을 추상 값으로 변환
- `γ: A → P(C)`: **구체화 함수(Concretization)** — 추상 값이 나타내는 구체 값 집합 반환

조건: `∀ S ∈ P(C), ∀ a ∈ A: α(S) ⊑ a ⟺ S ⊆ γ(a)`

즉, `α`로 추상화하면 실제 값을 항상 **과대 근사(over-approximation)**한다. 버그를 놓치지 않기 위해 불확실한 경우에는 "위험할 수도 있다"는 쪽으로 판단한다.

---

## 실전 구현: 부호 분석(Sign Analysis)

간단한 부호 도메인을 Python으로 구현해보자.

```python
from enum import Enum, auto
from typing import Optional

class Sign(Enum):
    BOT  = auto()   # 도달 불가능 (bottom)
    NEG  = auto()   # 음수  (< 0)
    ZERO = auto()   # 영    (= 0)
    POS  = auto()   # 양수  (> 0)
    TOP  = auto()   # 알 수 없음 (top)

# 추상화 함수: 정수 → 부호
def alpha(n: int) -> Sign:
    if n < 0: return Sign.NEG
    if n == 0: return Sign.ZERO
    return Sign.POS

# join (상한): 두 추상 값을 합쳐 더 많은 값을 표현
JOIN_TABLE = {
    (Sign.BOT,  Sign.BOT):  Sign.BOT,
    (Sign.BOT,  Sign.NEG):  Sign.NEG,
    (Sign.BOT,  Sign.ZERO): Sign.ZERO,
    (Sign.BOT,  Sign.POS):  Sign.POS,
    (Sign.NEG,  Sign.NEG):  Sign.NEG,
    (Sign.NEG,  Sign.ZERO): Sign.TOP,  # 음수 or 0 → 알 수 없음
    (Sign.NEG,  Sign.POS):  Sign.TOP,
    (Sign.ZERO, Sign.ZERO): Sign.ZERO,
    (Sign.ZERO, Sign.POS):  Sign.TOP,
    (Sign.POS,  Sign.POS):  Sign.POS,
}

def join(a: Sign, b: Sign) -> Sign:
    if a == Sign.TOP or b == Sign.TOP: return Sign.TOP
    key = (min(a, b, key=lambda x: x.value),
           max(a, b, key=lambda x: x.value))
    return JOIN_TABLE.get(key, Sign.TOP)

# 추상 덧셈: α({x + y | x ∈ γ(a), y ∈ γ(b)})
def abstract_add(a: Sign, b: Sign) -> Sign:
    if Sign.BOT in (a, b): return Sign.BOT
    if a == Sign.TOP or b == Sign.TOP: return Sign.TOP
    if a == Sign.ZERO: return b
    if b == Sign.ZERO: return a
    if a == b: return a                     # POS+POS=POS, NEG+NEG=NEG
    return Sign.TOP                          # NEG+POS = 알 수 없음

# 추상 나눗셈: 0으로 나누기 감지
def abstract_div(a: Sign, b: Sign) -> Optional[Sign]:
    if Sign.BOT in (a, b): return Sign.BOT
    if b == Sign.ZERO:
        raise ZeroDivisionError("[추상 해석] b가 반드시 0 — 나눗셈 오류 확정!")
    if b == Sign.TOP:
        print("[경고] b가 0일 수 있음 — 잠재적 나눗셈 오류")
        return Sign.TOP
    # b가 NEG 또는 POS일 때
    if a == Sign.TOP or a == Sign.ZERO: return Sign.TOP
    # POS/POS=POS, NEG/NEG=POS, POS/NEG=NEG, NEG/POS=NEG
    if a == b: return Sign.POS
    return Sign.NEG

# --- 간단한 프로그램 분석 예제 ---
# 프로그램: result = (x + 1) / y
# 입력 가정: x > 0, y는 알 수 없음

x_sign = Sign.POS
y_sign = Sign.TOP   # 사용자 입력 등 알 수 없는 값

tmp = abstract_add(x_sign, alpha(1))  # x + 1: POS+POS = POS
print(f"x + 1 의 부호: {tmp}")       # POS

result = abstract_div(tmp, y_sign)    # 경고: y가 0일 수 있음
print(f"(x+1)/y 의 부호: {result}")
```

실행 결과:
```
x + 1 의 부호: Sign.POS
[경고] b가 0일 수 있음 — 잠재적 나눗셈 오류
(x+1)/y 의 부호: Sign.TOP
```

`y_sign = Sign.ZERO`로 설정하면 `ZeroDivisionError`가 발생한다. 이것이 추상 해석의 핵심이다: 실제 실행 없이, 가능한 값들의 집합을 추상 도메인에서 계산하여 오류 가능성을 탐지한다.

---

## 실전 구현: 구간 분석(Interval Analysis)

더 정밀한 분석을 위해 값의 범위를 추적하는 구간 도메인을 구현한다.

```python
from __future__ import annotations
import math

INF = math.inf

class Interval:
    """[lo, hi] 형태의 정수 구간을 추상 값으로 사용"""
    def __init__(self, lo: float, hi: float):
        assert lo <= hi or (lo == INF and hi == -INF), "빈 구간"
        self.lo = lo
        self.hi = hi

    @classmethod
    def bottom(cls) -> Interval:
        return cls(INF, -INF)  # 빈 구간 (도달 불가)

    @classmethod
    def top(cls) -> Interval:
        return cls(-INF, INF)

    def is_bottom(self) -> bool:
        return self.lo > self.hi

    def join(self, other: Interval) -> Interval:
        if self.is_bottom(): return other
        if other.is_bottom(): return self
        return Interval(min(self.lo, other.lo), max(self.hi, other.hi))

    def __add__(self, other: Interval) -> Interval:
        if self.is_bottom() or other.is_bottom(): return Interval.bottom()
        return Interval(self.lo + other.lo, self.hi + other.hi)

    def __mul__(self, other: Interval) -> Interval:
        if self.is_bottom() or other.is_bottom(): return Interval.bottom()
        products = [self.lo*other.lo, self.lo*other.hi,
                    self.hi*other.lo, self.hi*other.hi]
        return Interval(min(products), max(products))

    def contains_zero(self) -> bool:
        return self.lo <= 0 <= self.hi

    def check_array_access(self, size: int) -> str:
        if self.lo < 0:
            return f"[오류] 음수 인덱스 가능: [{self.lo}, {self.hi}]"
        if self.hi >= size:
            return f"[오류] 범위 초과 가능: [{self.lo}, {self.hi}] (크기 {size})"
        return f"[안전] 인덱스 [{self.lo}, {self.hi}] (크기 {size})"

    def __repr__(self) -> str:
        if self.is_bottom(): return "⊥"
        lo = "-∞" if self.lo == -INF else str(int(self.lo))
        hi = "+∞" if self.hi == INF else str(int(self.hi))
        return f"[{lo}, {hi}]"

# --- for 루프 분석 (Widening 없는 단순 버전) ---
# 코드: for i in range(n): arr[i] = i * 2
# n의 범위: [0, 100]

n = Interval(0, 100)
i = Interval(0, 0)  # 초기값

# 루프 내 변수 범위 계산 (단순화: 불동점까지 반복)
prev = Interval.bottom()
while True:
    new_i = i.join(Interval(0, 0)).join(
        Interval(i.lo, i.hi - 1) if not i.is_bottom() else Interval.bottom()
    )
    # 실제로는 n-1이 상한
    loop_i = Interval(0, n.hi - 1) if n.hi > 0 else Interval.bottom()
    result = loop_i * Interval(2, 2)
    break

print(f"루프 인덱스 i 범위: {loop_i}")
print(f"arr[i] 쓰기 값 범위: {result}")
print(loop_i.check_array_access(100))
print(loop_i.check_array_access(50))   # 범위 초과 경고
```

실행 결과:
```
루프 인덱스 i 범위: [0, 99]
arr[i] 쓰기 값 범위: [0, 198]
[안전] 인덱스 [0, 99] (크기 100)
[오류] 범위 초과 가능: [0, 99] (크기 50)
```

---

## 실제 도구에서의 적용

### Clang Static Analyzer

Clang에 내장된 정적 분석기는 추상 해석 기반으로 C/C++/Objective-C 코드를 분석한다.

```bash
# 정적 분석 실행
clang --analyze -Xanalyzer -analyzer-output=html -o report/ mycode.c

# 또는 scan-build 래퍼 사용
scan-build make
```

주요 탐지 패턴:
- Null 포인터 역참조
- 해제된 메모리 사용 (use-after-free)
- 배열 범위 초과
- 자원 누수 (파일 디스크립터, 메모리)

### Facebook Infer

Facebook이 개발한 Infer는 Java/C/C++/Objective-C를 지원하며, **Bi-Abduction**이라는 고급 추상 해석 기법으로 **프로시저 간(Interprocedural) 분석**을 수행한다.

```bash
# Java 프로젝트 분석
infer run -- javac HelloWorld.java

# Android 프로젝트
infer run -- gradle build
```

---

## Widening과 Narrowing: 루프 처리의 핵심

루프가 있는 프로그램에서 구간 분석을 정확히 적용하면 무한 루프에 빠질 수 있다. 이를 해결하기 위해:

- **Widening(∇)**: 수렴을 강제로 빠르게 끝내는 연산. 예: 구간이 커지는 방향으로 `±∞`로 즉시 확장.
- **Narrowing(△)**: Widening 후 과도하게 확장된 구간을 다시 좁히는 연산.

```python
def widen(a: Interval, b: Interval) -> Interval:
    """Widening: 위로 발산하는 경계는 즉시 +∞로, 아래로는 -∞로"""
    if a.is_bottom(): return b
    lo = b.lo if b.lo >= a.lo else -INF
    hi = b.hi if b.hi <= a.hi else INF
    return Interval(lo, hi)
```

Widening을 적용하면 루프 분석이 유한 단계 내에 종료되는 것이 보장된다(단, 정밀도는 다소 희생된다).

---

## 주의사항과 팁

### 1. 거짓 양성(False Positive) 관리

건전한 분석기는 "위험할 수도 있다"는 경고를 과도하게 낸다. Facebook Infer의 연구에 따르면 실제 배포된 코드에서 거짓 양성률이 수십 %에 달한다. 이를 줄이기 위해:
- 더 정밀한 추상 도메인 사용 (구간 → 옥토곤 → 폴리헤드라)
- 경로 민감(path-sensitive) 분석 적용
- 사용자 어노테이션(`@Nullable`, `@NonNull`)으로 힌트 제공

### 2. 추상 도메인 선택의 트레이드오프

| 도메인 | 표현력 | 분석 속도 | 사례 |
|--------|--------|-----------|------|
| 부호 | 낮음 | 빠름 | 오버플로우 탐지 |
| 구간 | 중간 | 중간 | 배열 범위 검사 |
| 옥토곤 | 높음 | 느림 | 루프 경계 증명 |
| 폴리헤드라 | 매우 높음 | 매우 느림 | 정밀 검증 |

### 3. 현장 적용 전략

- CI/CD 파이프라인에 Clang Static Analyzer 또는 Infer를 통합하여 **지속적 분석** 수행
- 새 코드에 대해서만 분석 결과를 리뷰하는 **diff 기반 모드** 활용
- 보안 취약점(버퍼 오버플로우, SQL injection 경로)에 집중하는 **특수 목적 체커** 작성

---

## 정리

추상 해석은 "실행하지 않고도 버그를 찾는" 수학적으로 건전한 프레임워크다. 격자 이론과 갈루아 연결을 기반으로, 구체적인 실행 상태 대신 추상 도메인에서 프로그램 의미론을 계산한다. Clang, Infer 등 실용적인 도구들이 이 이론을 기반으로 수백만 라인의 산업 코드를 검증하고 있다. 완벽한 분석은 이론적으로 불가능하지만, 잘 설계된 추상 도메인과 Widening 전략을 통해 실용적인 수준의 건전한 분석이 가능하다.

## 참고 자료
- [google/sanitizers: AddressSanitizer·ThreadSanitizer GitHub 저장소](https://github.com/google/sanitizers)
- [llvm/llvm-project: Clang Static Analyzer 소스 코드](https://github.com/llvm/llvm-project)
- [brettwooldridge/HikariCP](https://github.com/brettwooldridge/HikariCP)
