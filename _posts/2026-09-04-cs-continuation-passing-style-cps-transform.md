---
layout: post
title: "Continuation Passing Style(CPS) 완전 정복: async/await와 코루틴의 수학적 토대"
date: 2026-09-04
categories: [cs, computer-science]
tags: [cps, continuation, async-await, compiler, functional-programming, coroutine, tail-call]
---

## Continuation이란 무엇인가

프로그램을 실행하는 도중 어떤 표현식을 계산할 때, **"이 표현식의 값으로 나머지 프로그램이 무엇을 할 것인가"** 라는 질문에 대한 답이 바로 **continuation(계속)**입니다.

예를 들어 `f(x) + g(y)`를 계산한다고 하면:
- `f(x)`의 continuation: "그 결과와 g(y)를 더한 뒤 전체 표현식의 결과로 반환한다"
- `g(y)`의 continuation: "f(x)의 결과와 그것을 더한 뒤 전체 표현식의 결과로 반환한다"

이 continuation을 **명시적인 함수 인자**로 변환하는 것이 **Continuation Passing Style(CPS)** 입니다.

```python
# 직접 스타일 (Direct Style)
def add(x, y):
    return x + y

result = add(3, 4)  # 7

# CPS 변환 후
def add_cps(x, y, k):  # k = continuation (나머지 계산)
    k(x + y)           # 결과를 k에 넘김, 절대 직접 반환하지 않음

add_cps(3, 4, print)   # 7 출력
```

CPS 함수는 **절대 값을 반환하지 않습니다.** 대신 결과를 continuation 함수에 넘깁니다. 프로그램의 제어 흐름이 완전히 명시적이고 선형화됩니다.

---

## 왜 CPS가 중요한가

### 컴파일러 중간 표현으로서의 CPS

CPS는 1970년대부터 함수형 언어 컴파일러의 **중간 표현(Intermediate Representation, IR)** 으로 사용되어 왔습니다. Appel의 책 "Compiling with Continuations"이 이를 체계화했습니다.

CPS IR의 장점:
- **모든 연산이 꼬리 호출(tail call)**: CPS 변환 후에는 모든 함수 호출이 꼬리 위치에 있으므로, 스택 프레임 없이 점프(jump)로 컴파일 가능
- **제어 흐름의 명시화**: 조건 분기, 예외, 함수 호출 모두 함수 호출로 표현되어 최적화가 균일해짐
- **클로저와 자유 변수 분석 용이**: continuation이 클로저이므로 람다 리프팅(lambda lifting) 등의 최적화가 자연스러움

### async/await와 코루틴의 구현 원리

JavaScript의 `async/await`, Python의 `asyncio`, Kotlin의 coroutine, Rust의 `Future`는 모두 내부적으로 CPS 변환(또는 그 변형)을 사용합니다.

```python
# Python asyncio 개념
async def fetch(url):
    data = await http_get(url)
    return process(data)

# 컴파일러 내부에서 CPS와 유사한 변환
def fetch_cps(url, k):
    def after_get(data):
        k(process(data))
    http_get_cps(url, after_get)
```

`await`는 사실 "여기서 continuation을 저장하고, 비동기 작업이 완료되면 그 continuation을 호출하라"는 명령입니다.

### 예외 처리와 비지역 탈출

CPS는 예외, early return, goto 등 **비지역 제어 흐름(non-local control flow)** 을 순수 함수 호출로 표현할 수 있게 합니다. `call/cc`(call with current continuation)는 이 능력을 최대화한 연산입니다.

---

## 실제 구현 예제

### 예제 1: 재귀 함수의 CPS 변환과 꼬리 재귀 최적화

```python
import sys
sys.setrecursionlimit(10000)

# ── 직접 스타일 ──
def factorial(n):
    if n == 0:
        return 1
    return n * factorial(n - 1)
# 문제: 스택이 n 깊이만큼 쌓임

# ── CPS 변환 ──
def factorial_cps(n, k):
    """k: 결과를 받을 continuation"""
    if n == 0:
        k(1)
    else:
        factorial_cps(n - 1, lambda result: k(n * result))
# 여전히 문제: lambda가 클로저를 캡처해 스택처럼 쌓임

# ── CPS + 트램폴린(Trampolining) ──
# CPS 후에도 재귀가 남는 경우를 반복문으로 처리
def trampoline(f):
    """반환값이 callable이면 계속 호출, 아니면 최종값 반환"""
    while callable(f):
        f = f()
    return f

def factorial_trampoline(n, k=None):
    if k is None:
        k = lambda x: x
    if n == 0:
        return lambda: k(1)
    return lambda: factorial_trampoline(n - 1, lambda result: k(n * result))

# 트램폴린으로 실행: 스택 오버플로 없이 큰 수 계산 가능
print(trampoline(factorial_trampoline(1000)))  # 1000!

# ── 더 나은 방법: CPS + 누산기 ──
def factorial_acc_cps(n, acc, k):
    """tail-recursive CPS: acc로 중간 결과 누적"""
    if n == 0:
        return lambda: k(acc)     # 꼬리 위치의 k 호출
    return lambda: factorial_acc_cps(n - 1, n * acc, k)

result = trampoline(factorial_acc_cps(10, 1, lambda x: x))
print(f"10! = {result}")  # 3628800


# ── CPS 변환 검증: 상호 재귀 ──
def is_even_cps(n, k):
    if n == 0:
        return lambda: k(True)
    return lambda: is_odd_cps(n - 1, k)

def is_odd_cps(n, k):
    if n == 0:
        return lambda: k(False)
    return lambda: is_even_cps(n - 1, k)

result = trampoline(is_even_cps(100, lambda x: x))
print(f"100은 짝수: {result}")  # True
result = trampoline(is_odd_cps(99, lambda x: x))
print(f"99는 홀수: {result}")   # True
```

### 예제 2: call/cc 시뮬레이션과 비지역 탈출

```python
from contextlib import contextmanager

# ── Python으로 call/cc 흉내 내기 ──
# Python에는 진짜 call/cc가 없지만, 예외로 비지역 탈출 구현

class Escape(Exception):
    def __init__(self, value):
        self.value = value

def call_with_escape(func):
    """
    func(escape): escape(v)를 호출하면 즉시 call_with_escape(v)를 반환
    """
    try:
        result = [None]
        def escape(v):
            raise Escape(v)
        result[0] = func(escape)
        return result[0]
    except Escape as e:
        return e.value


# 예시 1: 리스트에서 음수를 만나면 즉시 False 반환 (early exit)
def all_positive(lst):
    return call_with_escape(lambda exit_: [
        exit_(False) if x < 0 else None
        for x in lst
    ] or True)

print(all_positive([1, 2, 3]))        # True
print(all_positive([1, -2, 3]))       # False (음수 만나는 순간 종료)


# 예시 2: 트리에서 특정 값 찾기 — 발견 즉시 탈출
def make_tree(val, left=None, right=None):
    return {"val": val, "left": left, "right": right}

def find_in_tree(tree, target):
    def search(node, exit_):
        if node is None:
            return
        if node["val"] == target:
            exit_(True)  # 발견! 즉시 전체 탐색 종료
        search(node["left"], exit_)
        search(node["right"], exit_)
        return False

    return call_with_escape(lambda e: search(tree, e) or False)

tree = make_tree(1,
    make_tree(2, make_tree(4), make_tree(5)),
    make_tree(3, make_tree(6), make_tree(7))
)
print(find_in_tree(tree, 6))   # True (6을 찾는 순간 탐색 종료)
print(find_in_tree(tree, 99))  # False


# ── async/await를 CPS로 직접 구현 ──
class Promise:
    """미래에 완료될 값을 나타내는 단순 Promise"""
    def __init__(self):
        self._callbacks = []
        self._value = None
        self._resolved = False

    def resolve(self, value):
        self._value = value
        self._resolved = True
        for cb in self._callbacks:
            cb(value)

    def then(self, callback):
        """CPS의 continuation 등록"""
        p = Promise()
        if self._resolved:
            callback(self._value)
        else:
            def wrapped(value):
                result = callback(value)
                if isinstance(result, Promise):
                    result.then(p.resolve)
                else:
                    p.resolve(result)
            self._callbacks.append(wrapped)
        return p


# CPS 스타일의 비동기 체이닝
def async_double(x):
    p = Promise()
    # 즉시 resolve (실제는 I/O 대기)
    p.resolve(x * 2)
    return p

def async_add_one(x):
    p = Promise()
    p.resolve(x + 1)
    return p

# CPS 체이닝: (5 * 2) + 1 = 11
async_double(5) \
    .then(async_add_one) \
    .then(lambda result: print(f"비동기 결과: {result}"))  # 11
```

---

## CPS 변환 알고리즘

임의의 직접 스타일(direct style) 표현식을 기계적으로 CPS로 변환하는 알고리즘이 있습니다. Fischer와 Plotkin이 정형화한 방법으로, 표현식의 구조에 따라 재귀적으로 적용합니다.

**상수와 변수**: 값을 continuation에 직접 전달
```
CPS[v](k) = k(v)
```

**함수 정의**: continuation을 추가 인자로 받는 CPS 함수로 변환
```
CPS[λx.e](k) = k(λx.λκ. CPS[e](κ))
```

**함수 적용**: 함수와 인자를 먼저 계산한 뒤, 결과를 continuation에 전달
```
CPS[f(a)](k) = CPS[f](λfv. CPS[a](λav. fv(av)(k)))
```

이 변환의 결과는 항상 **꼬리 재귀 형태**입니다. 즉, CPS 변환된 프로그램의 모든 함수 호출은 꼬리 위치에 있어서 스택 프레임 없이 최적화됩니다.

---

## 주의사항 및 실전 팁

### 클로저 폭발 (Closure Explosion)

나이브한 CPS 변환은 각 중간 단계마다 클로저를 생성합니다. `f(g(h(x)))` 변환 시 3개의 클로저가 생기며, 깊은 표현식에서는 수백 개의 클로저가 중첩됩니다. 이를 완화하기 위해:

- **관리형 CPS**: 꼬리 위치의 연산은 클로저 없이 직접 변환
- **Administrative Reduction**: 생성 즉시 적용 가능한 클로저를 인라인 처리

### 스택과 힙 트레이드오프

CPS는 콜스택을 클로저(힙)로 바꿉니다. 스택 오버플로는 없어지지만 GC 압박이 늘어납니다. 트램폴린 패턴을 사용하면 클로저 생성을 줄일 수 있지만, 간접 호출 오버헤드가 생깁니다.

### 디버깅의 어려움

CPS 변환된 코드는 스택 트레이스가 의미 없어집니다. `continuation 3 → continuation 5 → continuation 1`처럼 원래 코드와의 대응 관계를 잃습니다. 실용적인 언어(JavaScript, Kotlin)는 소스 맵이나 디버그 정보로 이 문제를 완화합니다.

### 현대 언어에서의 실용적 적용

- **Kotlin coroutines**: 컴파일러가 suspend 함수를 Continuation 인터페이스 기반 상태 머신으로 변환 (CPS 변환의 변형)
- **JavaScript async/await**: V8, SpiderMonkey가 async 함수를 Promise 기반 CPS로 변환
- **Rust async/await**: Future trait이 CPS의 poll 메커니즘을 구현
- **GHC Haskell**: Continuation monad(`ContT`)로 CPS를 라이브러리 수준에서 제공

---

## 참고 자료

- [Continuation-Passing Style - Wikipedia](https://en.wikipedia.org/wiki/Continuation-passing_style)
- [How to Compile with Continuations - Matt Might](https://matt.might.net/articles/cps-conversion/)
- [CPS Lecture Notes - CS6110 Cornell](https://www.cs.cornell.edu/courses/cs6110/2013sp/lectures/lec13-sp13.pdf)
- [Into CPS, Never to Return - Max Bernstein](https://bernsteinbear.com/blog/cps/)
