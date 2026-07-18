---
layout: post
title: "Continuation Passing Style과 Coroutine 내부 구현 원리: async/await는 어떻게 동작하는가"
date: 2026-07-18
categories: [cs, computer-science]
tags: [continuation-passing-style, cps, coroutine, async-await, stackful, stackless, state-machine, trampoline, compiler]
---

Python의 `async/await`, Kotlin의 `suspend`, JavaScript의 `async function`을 사용하면 비동기 코드를 마치 동기 코드처럼 작성할 수 있다. 그런데 컴파일러와 런타임은 이 코드를 실제로 어떻게 처리할까? 핵심에는 **Continuation Passing Style(CPS)**과 **상태 기계(State Machine) 변환**이 있다. 이 글에서는 이 두 개념의 이론적 배경부터 실제 구현까지 단계적으로 살펴본다.

---

## 개념 설명

### Continuation이란

**Continuation**은 "현재 계산이 완료된 후 나머지 프로그램의 실행 흐름 전체"를 일급 값(First-class value)으로 표현한 것이다. 쉽게 말해 "이 작업 이후에 무엇을 해야 하는가"를 함수로 캡슐화한 것이다.

일반적인 함수 호출은 반환값을 호출 지점으로 돌려보낸다.

```python
def add(x, y):
    return x + y  # 결과를 호출자에게 반환

result = add(3, 4)  # 7
print(result)
```

**CPS(Continuation Passing Style)**에서는 함수가 결과를 직접 반환하지 않고, "다음에 무엇을 할지"를 나타내는 콜백(continuation)에 결과를 전달한다.

```python
def add_cps(x, y, k):
    k(x + y)  # 결과를 continuation k에 전달

add_cps(3, 4, lambda result: print(result))  # 7 출력
```

### Stackful vs Stackless Coroutine

코루틴은 실행을 일시 중단(suspend)하고 나중에 재개(resume)할 수 있는 서브루틴이다. 구현 방식에 따라 두 종류로 나뉜다.

**Stackful Coroutine** (예: goroutine, libco, Kotlin 코루틴의 일부)
- 각 코루틴이 자체 스택을 가짐
- 콜 스택의 임의 깊이에서 suspend 가능
- 스택 메모리 오버헤드 존재 (초기 수 KB ~ 수십 KB)
- 컨텍스트 스위치: 레지스터 + 스택 포인터 교체

**Stackless Coroutine** (예: Python asyncio, JavaScript async/await, Rust async, C++20 coroutine)
- 별도 스택 없이 상태 기계로 컴파일됨
- suspend 지점이 컴파일 타임에 확정 (await/yield 위치)
- 상태 프레임만 힙에 저장 → 메모리 효율적 (수십 바이트)
- 중첩 호출에서 await가 가능하려면 전체 체인이 async여야 함

---

## 왜 필요한가

### 콜백 지옥의 해결

네트워크 요청, 파일 I/O, 타이머 등 비동기 작업을 처리할 때 콜백 중첩은 필연적이었다.

```javascript
// 콜백 지옥 (Callback Hell)
fetchUser(userId, function(user) {
    fetchProfile(user.id, function(profile) {
        fetchPosts(profile.id, function(posts) {
            render(user, profile, posts);
        }, handleError);
    }, handleError);
}, handleError);
```

CPS를 기반으로 한 async/await 변환은 이를 다음처럼 바꾼다.

```javascript
async function loadUserData(userId) {
    const user = await fetchUser(userId);
    const profile = await fetchProfile(user.id);
    const posts = await fetchPosts(profile.id);
    render(user, profile, posts);
}
```

코드는 동기처럼 보이지만, 컴파일러가 이를 상태 기계로 변환해 스레드를 블로킹하지 않고 실행한다.

### I/O 집약적 서비스의 스케일링

1만 개의 동시 HTTP 요청을 처리할 때, 스레드 기반 모델은 1만 개의 OS 스레드(= 수십 GB RAM)가 필요하다. 코루틴 기반 모델은 이벤트 루프 위에서 수십만 개의 코루틴을 수 MB의 메모리로 처리할 수 있다.

---

## 실제 구현 예제

### 예제 1: CPS 변환 수동 구현 (Python)

컴파일러가 자동으로 하는 CPS 변환을 수동으로 보여준다.

```python
# ── 직접 스타일 (Direct Style) ──────────────────────────────────
def factorial(n):
    if n == 0:
        return 1
    return n * factorial(n - 1)

print(factorial(5))  # 120


# ── CPS 스타일 ────────────────────────────────────────────────
# 모든 함수가 결과를 직접 반환하지 않고 continuation에 전달
def factorial_cps(n, k):
    if n == 0:
        k(1)  # base case: continuation에 1 전달
    else:
        # n-1의 팩토리얼을 구한 뒤, 그 결과에 n을 곱해 k에 전달
        factorial_cps(n - 1, lambda result: k(n * result))

factorial_cps(5, print)  # 120


# ── Trampoline으로 스택 오버플로 방지 ────────────────────────────
# CPS는 재귀 깊이에 따라 스택이 쌓이는 문제가 있다.
# Trampoline: continuation을 즉시 호출하는 대신 thunk(0인자 함수)로 반환
class Thunk:
    """지연 평가될 계산을 캡슐화"""
    def __init__(self, func, *args):
        self.func = func
        self.args = args
    
    def __call__(self):
        return self.func(*self.args)

def trampoline(f):
    """Thunk가 나오는 동안 반복 실행 (스택 불필요)"""
    result = f
    while callable(result):
        result = result()
    return result

def factorial_trampoline(n, acc=1):
    """꼬리 재귀 팩토리얼 → Thunk로 감싸서 Trampoline에 전달"""
    if n == 0:
        return acc
    # 즉시 재귀하지 않고 Thunk 반환
    return Thunk(factorial_trampoline, n - 1, n * acc)

# Python의 재귀 한도를 우회하여 큰 수도 처리 가능
result = trampoline(Thunk(factorial_trampoline, 10000))
print(f"10000! 마지막 5자리: {result % 100000}")


# ── CPS로 비동기 연산 시뮬레이션 ──────────────────────────────
import time
from threading import Thread

def async_fetch(url, on_success, on_error):
    """비동기 HTTP 요청 시뮬레이션 (CPS 스타일)"""
    def worker():
        time.sleep(0.1)  # 네트워크 지연 시뮬레이션
        if "invalid" in url:
            on_error(Exception(f"Invalid URL: {url}"))
        else:
            on_success({"url": url, "data": "response data"})
    Thread(target=worker, daemon=True).start()

def fetch_and_process(user_id):
    """콜백 체인으로 비동기 작업 연결 (CPS 패턴)"""
    def on_user(user):
        print(f"유저 로드 완료: {user}")
        async_fetch(
            f"http://api/profile/{user['id']}",
            on_profile,
            lambda e: print(f"프로필 오류: {e}")
        )
    
    def on_profile(profile):
        print(f"프로필 로드 완료: {profile}")
    
    async_fetch(
        f"http://api/user/{user_id}",
        on_user,
        lambda e: print(f"유저 오류: {e}")
    )

fetch_and_process(42)
time.sleep(0.5)  # 비동기 작업 완료 대기
```

### 예제 2: Stackless Coroutine의 상태 기계 변환 직접 구현 (Python)

컴파일러가 `async/await`를 어떻게 상태 기계로 변환하는지 수동으로 구현한다.

```python
# ── 컴파일러가 변환할 원본 async 코드 ─────────────────────────
# async def fetch_user_data(user_id):
#     user = await db_query(f"SELECT * FROM users WHERE id={user_id}")
#     profile = await http_get(f"/profile/{user['id']}")
#     return {"user": user, "profile": profile}

# ── 위 코드를 상태 기계로 수동 변환 ────────────────────────────
class FetchUserDataCoroutine:
    """
    컴파일러가 async def를 변환한 상태 기계 클래스
    각 await 지점이 하나의 상태(state)에 대응
    """
    STATE_START = 0
    STATE_AWAIT_DB = 1
    STATE_AWAIT_HTTP = 2
    STATE_DONE = 3
    STATE_ERROR = 4
    
    def __init__(self, user_id):
        self.user_id = user_id
        self.state = self.STATE_START
        self.result = None
        self.exception = None
        # 지역 변수들: suspend/resume 사이에 보존해야 함
        self._user = None
        self._profile = None
        # 완료 콜백 (continuation)
        self._continuation = None
    
    def send(self, value=None):
        """
        코루틴에 값을 주입하고 다음 yield 지점까지 실행.
        이것이 바로 Python 제너레이터의 send() 메서드와 동일한 동작.
        """
        try:
            return self._step(value)
        except StopIteration:
            raise
        except Exception as e:
            self.exception = e
            self.state = self.STATE_ERROR
            raise
    
    def _step(self, incoming_value):
        """상태 기계의 한 스텝 실행"""
        if self.state == self.STATE_START:
            # await db_query(...) 호출 → 비동기 작업을 이벤트 루프에 등록하고 yield
            self.state = self.STATE_AWAIT_DB
            # 이벤트 루프에 "이 작업이 완료되면 나를 깨워라"라는 Future를 yield
            return ("DB_QUERY", f"SELECT * FROM users WHERE id={self.user_id}")
        
        elif self.state == self.STATE_AWAIT_DB:
            # DB 쿼리 결과가 incoming_value로 들어옴
            self._user = incoming_value  # user 지역변수 저장
            # await http_get(...) 호출
            self.state = self.STATE_AWAIT_HTTP
            return ("HTTP_GET", f"/profile/{self._user['id']}")
        
        elif self.state == self.STATE_AWAIT_HTTP:
            # HTTP 응답이 incoming_value로 들어옴
            self._profile = incoming_value
            # 모든 await 완료 → return 값 설정
            self.result = {"user": self._user, "profile": self._profile}
            self.state = self.STATE_DONE
            raise StopIteration(self.result)  # Python 제너레이터 종료 관례
        
        elif self.state == self.STATE_DONE:
            raise RuntimeError("Coroutine is already done")


class SimpleEventLoop:
    """간단한 이벤트 루프 (실제 asyncio의 축소판)"""
    
    def __init__(self):
        self.pending = []  # (coroutine, value) 큐
    
    def run(self, coro):
        """코루틴을 이벤트 루프에서 실행"""
        # 첫 번째 스텝 실행
        try:
            event_type, event_arg = coro.send(None)
            self._handle_event(coro, event_type, event_arg)
        except StopIteration as e:
            return e.value
        return self._process_pending()
    
    def _handle_event(self, coro, event_type, event_arg):
        """이벤트 처리 후 코루틴 재개"""
        # 실제 I/O 작업 시뮬레이션
        if event_type == "DB_QUERY":
            # 실제로는 여기서 비동기 DB 드라이버에 쿼리 등록
            mock_result = {"id": 42, "name": "Alice"}
            print(f"[EventLoop] DB 쿼리 완료: {mock_result}")
            self.pending.append((coro, mock_result))
        
        elif event_type == "HTTP_GET":
            mock_result = {"avatar": "alice.png", "bio": "Engineer"}
            print(f"[EventLoop] HTTP 요청 완료: {mock_result}")
            self.pending.append((coro, mock_result))
    
    def _process_pending(self):
        """대기 중인 코루틴들 재개"""
        while self.pending:
            coro, value = self.pending.pop(0)
            try:
                event_type, event_arg = coro.send(value)
                self._handle_event(coro, event_type, event_arg)
            except StopIteration as e:
                return e.value
        return None


# 실행
loop = SimpleEventLoop()
coro = FetchUserDataCoroutine(user_id=42)
result = loop.run(coro)
print(f"\n최종 결과: {result}")


# ── Python의 실제 async/await와 비교 ───────────────────────────
import asyncio

async def fetch_user_data_real(user_id):
    # 이것이 컴파일러가 위의 상태 기계로 변환하는 코드
    await asyncio.sleep(0)  # DB 쿼리 시뮬레이션
    user = {"id": user_id, "name": "Alice"}
    await asyncio.sleep(0)  # HTTP 요청 시뮬레이션  
    profile = {"avatar": "alice.png", "bio": "Engineer"}
    return {"user": user, "profile": profile}

async def main():
    result = await fetch_user_data_real(42)
    print(f"async/await 결과: {result}")

asyncio.run(main())
```

---

## 컴파일러의 CPS 변환 과정

실제 컴파일러(Kotlin, Rust, C#)가 `async/await`를 처리하는 과정을 단계별로 이해할 수 있다.

```
1단계: 파싱
   async def foo():
       x = await bar()
       return x + 1

2단계: CPS 변환 (내부 표현)
   def foo_cps(k_return, k_exception):
       def after_bar(x):
           try:
               k_return(x + 1)
           except Exception as e:
               k_exception(e)
       bar_cps(after_bar, k_exception)

3단계: 상태 기계 생성
   class foo_StateMachine:
       state = 0
       local_x = None
       
       def resume(self, value):
           if self.state == 0:
               self.state = 1
               return bar()  # await 포인트
           elif self.state == 1:
               self.local_x = value
               return Done(self.local_x + 1)

4단계: 힙 할당 최적화
   - 필요한 지역 변수만 힙에 저장
   - await 포인트를 넘나들지 않는 변수는 스택에 유지
```

---

## Kotlin Coroutine과의 연결

Kotlin의 `suspend` 함수는 컴파일 시 **CPS 변환**된다.

```kotlin
// 개발자가 작성하는 코드
suspend fun fetchUser(id: Int): User {
    val data = httpGet("/user/$id")  // suspend point
    return parseUser(data)
}

// 컴파일러가 변환한 실제 시그니처 (Continuation 주입)
fun fetchUser(id: Int, continuation: Continuation<User>): Any {
    // state machine...
}
```

`Continuation<T>` 인터페이스는 바로 CPS의 `k` 함수에 해당한다. Kotlin 컴파일러는 모든 `suspend` 함수에 이 매개변수를 추가하고, 함수 본문을 상태 기계로 변환한다.

---

## 주의사항 및 팁

### 구조적 동시성 (Structured Concurrency)

async/await만 있으면 예외 처리와 취소(cancellation) 전파가 어렵다. Kotlin의 `CoroutineScope`, Python의 `asyncio.TaskGroup`, Swift의 `async let` 등 구조적 동시성 도구를 함께 사용해야 한다.

```python
# Python 3.11+ TaskGroup을 사용한 구조적 동시성
async def fetch_all():
    async with asyncio.TaskGroup() as tg:
        task1 = tg.create_task(fetch_user(1))
        task2 = tg.create_task(fetch_profile(1))
    # 여기서 두 태스크가 모두 완료됨. 하나가 실패하면 나머지도 취소됨
    return task1.result(), task2.result()
```

### 블로킹 코드의 위험

`async` 컨텍스트에서 블로킹 I/O나 CPU 집약적 작업을 호출하면 이벤트 루프 전체가 블로킹된다.

```python
# 잘못된 예 - 이벤트 루프를 블로킹
async def bad():
    time.sleep(1)  # 이벤트 루프 전체 블로킹!

# 올바른 예 - 스레드풀로 오프로드
async def good():
    await asyncio.to_thread(time.sleep, 1)  # 별도 스레드에서 실행
```

### 상태 기계의 메모리 최적화

Rust의 `async fn`은 모든 지역 변수를 포함한 상태를 하나의 열거형(Enum)으로 묶어 힙에 저장한다. 컴파일러 최적화 덕분에 실제 메모리 사용량은 수십 바이트에 불과하다. 이것이 OS 스레드(수 KB 스택)보다 수백 배 더 많은 동시 코루틴을 유지할 수 있는 이유다.

### Colored Function 문제

`async` 함수는 동기 코드에서 직접 `await`할 수 없다. 이 제약을 "함수에 색깔이 있다"고 표현하기도 한다. 라이브러리 설계 시 동기/비동기 API를 별도로 제공하거나, `asyncio.run()`으로 명시적 진입점을 두는 방식으로 해결한다.

---

## 참고 자료
- [Continuation-passing style - Wikipedia](https://en.wikipedia.org/wiki/Continuation-passing_style)
- [Haskell/Continuation passing style - Wikibooks](https://en.wikibooks.org/wiki/Haskell/Continuation_passing_style)
- [Implementing Coroutines - abhinavsarkar.net](https://abhinavsarkar.net/posts/implementing-co-3/)
- [Kotlin Coroutines: Design and Implementation - GitHub](https://github.com/Kotlin/kotlin-coroutines/blob/master/kotlin-coroutines-informal.md)
