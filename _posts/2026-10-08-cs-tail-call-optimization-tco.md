---
layout: post
title: "테일 콜 최적화(TCO) 완전 정복: 재귀 호출을 스택 없이 실행하는 컴파일러 마법"
date: 2026-10-08
categories: [cs, computer-science]
tags: [tail-call, TCO, recursion, compiler, stack, scheme, functional-programming, trampoline]
---

재귀 함수를 깊게 호출하면 **스택 오버플로(Stack Overflow)**가 발생합니다. 수백만 번의 재귀 호출이 필요한 알고리즘에서는 치명적입니다. 그런데 Scheme 프로그래머는 재귀로 무한 루프를 구현해도 스택 오버플로가 절대 발생하지 않는다고 말합니다. 이를 가능하게 하는 기법이 바로 **테일 콜 최적화(Tail Call Optimization, TCO)**입니다. 이 글에서는 TCO의 원리를 어셈블리 수준에서 이해하고, 언어별 지원 현황과 우회 기법을 살펴봅니다.

---

## 개념 설명: 테일 포지션과 테일 콜

### 스택 프레임의 생애주기

함수를 호출할 때마다 **스택 프레임(Stack Frame)**이 생성됩니다. 스택 프레임에는 지역 변수, 반환 주소, 매개변수가 저장됩니다. 호출이 반환되면 프레임이 해제됩니다.

```
main()
  ↓ 호출
factorial(5)   ← 스택 프레임 1
  ↓ 호출
factorial(4)   ← 스택 프레임 2
  ↓ 호출
factorial(3)   ← 스택 프레임 3
  ...
factorial(1)   ← 스택 프레임 N  (최대 N개 동시 존재)
```

N이 클수록 스택이 깊어지고 결국 스택 영역이 고갈됩니다.

### 테일 포지션(Tail Position)

**테일 포지션**이란 함수에서 가장 마지막으로 실행되는 위치를 말합니다. 이 위치의 함수 호출을 **테일 콜(Tail Call)**이라고 합니다.

```python
# 테일 콜이 아닌 재귀 (일반 재귀)
def factorial_naive(n):
    if n == 0:
        return 1
    return n * factorial_naive(n - 1)  # 반환 후 '곱셈'이 남아있음 → 테일 콜 아님
    #         ^^^^^^^^^^^^^^^^^^^^^^^^
    # 이 호출의 결과에 n을 곱해야 하므로 현재 프레임을 유지해야 함

# 테일 콜 재귀 (누산기 패턴)
def factorial_tail(n, acc=1):
    if n == 0:
        return acc
    return factorial_tail(n - 1, acc * n)  # 이 호출이 마지막 작업 → 테일 콜
    #      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    # 이 호출의 결과를 그대로 반환 → 현재 프레임 불필요
```

두 번째 함수에서 재귀 호출은 **함수의 마지막 작업**이므로, 호출 후 현재 스택 프레임에서 할 일이 전혀 없습니다. 따라서 현재 프레임을 유지할 이유가 없습니다.

---

## 왜 필요한가: 함수형 프로그래밍과 재귀

명령형 언어에서 반복문(`for`, `while`)은 스택을 사용하지 않습니다. 그러나 **순수 함수형 언어(Haskell, Erlang, Scheme)**에서는 변경 가능한 루프 변수가 없어 반복을 반드시 재귀로 표현합니다.

```scheme
;; Scheme: 꼬리 재귀로 구현한 무한 이벤트 루프
;; TCO가 있어야 스택 오버플로 없이 영원히 실행 가능

(define (event-loop state)
  (let ((new-state (process-event state)))
    (event-loop new-state)))  ; 꼬리 호출 → 무한 실행 가능

(event-loop initial-state)
```

R7RS(Scheme 표준)는 "올바르게 꼬리 재귀적(Properly Tail-Recursive)"일 것을 언어 규격으로 요구합니다. TCO가 없으면 Scheme으로 실용적인 프로그램을 작성할 수 없습니다.

---

## 실제 구현 예제 1: 어셈블리 수준의 TCO

컴파일러가 테일 콜을 최적화하는 방식을 x86-64 어셈블리로 살펴봅니다.

```c
/* C로 작성한 꼬리 재귀 팩토리얼 */
long factorial(long n, long acc) {
    if (n <= 1) return acc;
    return factorial(n - 1, acc * n);  // 테일 콜
}
```

TCO 없이 컴파일 (`-O0`):
```asm
factorial:
    push    rbp              ; 스택 프레임 설정
    mov     rbp, rsp
    sub     rsp, 16
    cmp     rdi, 1           ; n <= 1?
    jle     .base_case
    imul    rsi, rdi         ; acc * n
    dec     rdi              ; n - 1
    call    factorial        ; ← 새 스택 프레임 생성 (재귀 깊이만큼 쌓임)
    jmp     .done
.base_case:
    mov     rax, rsi
.done:
    leave
    ret
```

TCO 적용 후 컴파일 (`-O2` 이상):
```asm
factorial:
    cmp     rdi, 1           ; n <= 1?
    jle     .base_case
    imul    rsi, rdi         ; acc = acc * n
    dec     rdi              ; n = n - 1
    jmp     factorial        ; ← CALL 대신 JMP! 스택 프레임 재사용
    ; 'ret'이 없어도 됨: 같은 함수의 시작으로 점프
.base_case:
    mov     rax, rsi
    ret
```

핵심: `call factorial`(새 프레임 생성) → `jmp factorial`(현재 프레임 재사용). 이것이 TCO입니다. 재귀 깊이가 100만이어도 스택에는 프레임 1개만 존재합니다.

```c
#include <stdio.h>

/* GCC에게 꼬리 재귀 최적화를 강제하는 예제 */
/* gcc -O2 로 컴파일 시 TCO 적용됨 */

static long sum_tail(long n, long acc) {
    if (n == 0) return acc;
    return sum_tail(n - 1, acc + n);  // 테일 콜
}

/* 비교: 일반 재귀 (TCO 불가) */
static long sum_naive(long n) {
    if (n == 0) return 0;
    return n + sum_naive(n - 1);  // 반환 후 덧셈이 남아 있음
}

int main(void) {
    /* TCO 덕분에 스택 오버플로 없음 */
    long result = sum_tail(1000000L, 0L);
    printf("Sum 1..1000000 = %ld\n", result);  // 500000500000

    /* 이 호출은 스택 오버플로 발생 (대부분 시스템에서) */
    /* long fail = sum_naive(1000000L); */

    return 0;
}
```

---

## 실제 구현 예제 2: 트램폴린(Trampoline) 기법

TCO를 언어/런타임이 지원하지 않을 때 **트램폴린(Trampoline)**을 사용하면 스택 오버플로 없이 깊은 재귀를 구현할 수 있습니다.

트램폴린의 아이디어: 함수가 결과를 반환하는 대신 **"다음에 호출할 함수"를 반환**합니다. 외부 루프(트램폴린)가 함수를 연속적으로 호출합니다.

```python
from typing import Callable, Any, Union

# Thunk: 인자 없는 함수 (지연 계산)
Thunk = Callable[[], Any]

def trampoline(f: Callable) -> Callable:
    """
    트램폴린 데코레이터.
    함수가 Thunk를 반환하면 계속 호출, 그 외 값이면 결과로 반환.
    """
    def wrapper(*args, **kwargs):
        result = f(*args, **kwargs)
        while callable(result):
            result = result()  # Thunk 실행
        return result
    return wrapper


# 트램폴린 방식의 팩토리얼
def _factorial_tramp(n: int, acc: int):
    if n == 0:
        return acc
    # 결과 대신 Thunk(클로저) 반환 → 스택에 쌓이지 않음
    return lambda: _factorial_tramp(n - 1, acc * n)

@trampoline
def factorial(n: int) -> int:
    return _factorial_tramp(n, 1)


# 트램폴린 방식의 홀수/짝수 판별 (상호 재귀)
def _is_even(n: int):
    if n == 0:
        return True
    return lambda: _is_odd(n - 1)

def _is_odd(n: int):
    if n == 0:
        return False
    return lambda: _is_even(n - 1)

is_even = trampoline(_is_even)


# 테스트
print(factorial(10))       # 3628800
print(factorial(100000))   # 아주 큰 수 (스택 오버플로 없음)
print(is_even(999999))     # False
print(is_even(1000000))    # True (재귀 깊이 100만, 스택 오버플로 없음)
```

트램폴린의 동작 원리:
```
is_even(1000000)
  → lambda: _is_odd(999999)    ← Thunk 반환 (스택 프레임 없음)
    → lambda: _is_even(999998)
      → lambda: _is_odd(999997)
        ...
          → True                ← 최종 값 반환
```

각 단계에서 스택 프레임은 트램폴린 루프 1개만 유지됩니다. Python, Java, JavaScript처럼 TCO를 지원하지 않는 언어에서 깊은 재귀가 필요할 때 유용합니다.

---

## 주의사항 및 팁

### 1. 언어별 TCO 지원 현황

| 언어 | TCO 지원 | 비고 |
|------|---------|------|
| Scheme | ✅ 표준 명세 | R7RS에서 필수 요구사항 |
| Haskell | ✅ 지연 평가로 일부 대체 | 재귀보다 무한 리스트 선호 |
| Erlang | ✅ BEAM VM 지원 | 꼬리 재귀 필수 |
| Kotlin | ✅ `tailrec` 키워드 | 컴파일러가 루프로 변환 |
| C/C++ | ✅ 조건부 | `-O2` 이상에서 컴파일러 재량 |
| Python | ❌ 의도적 미지원 | Guido가 스택 트레이스 가독성 이유로 거부 |
| JavaScript | ⚠️ ES6 명세에 있으나 | V8 등 대부분 미구현 |
| Java (JVM) | ❌ JVM 제약 | `invokedynamic` 해결책 논의 중 |

### 2. Kotlin `tailrec` 키워드

```kotlin
tailrec fun fibonacci(n: Int, a: Long = 0, b: Long = 1): Long {
    if (n == 0) return a
    return fibonacci(n - 1, b, a + b)  // tailrec이므로 루프로 컴파일
}
// 컴파일러가 꼬리 재귀가 아니면 경고 발생: "A function is marked as tail-recursive but no tail calls are found"
```

### 3. 꼬리 재귀 변환 패턴: 누산기(Accumulator)

일반 재귀를 꼬리 재귀로 바꾸는 핵심 기법은 **누산기 매개변수 추가**입니다:

```
일반 재귀:    result = f(x) OP recursive_call(...)
꼬리 재귀:    result = recursive_call(..., acc OP f(x))
```

연산을 "나중에 적용"하는 대신 "지금 누산기에 반영"합니다.

### 4. TCO가 항상 가능한 것은 아닙니다

상호 재귀(mutual recursion), 비결정적 분기(트리 탐색), 일부 분할 정복 알고리즘은 테일 포지션에서 두 개 이상의 재귀 호출을 해야 하므로 TCO 적용이 불가능합니다. 이 경우 명시적 스택, 트램폴린, 또는 반복적 접근으로 대체합니다.

---

## 참고 자료
- [Wikipedia: Tail call](https://en.wikipedia.org/wiki/Tail_call)
- [EPFL CS420: Tail Calls](https://cs420.epfl.ch/archive/26/c/09_tail-calls.html)
- [What is Tail Call Optimization? - DesignGurus](https://designgurus.io/answers/detail/what-is-tail-call-optimization)
