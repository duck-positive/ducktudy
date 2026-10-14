---
layout: post
title: "비동기 이벤트 루프 심화: libuv, asyncio, Tokio의 내부 구조 비교"
date: 2026-10-14
categories: [cs, computer-science]
tags: [event-loop, async, libuv, asyncio, tokio, non-blocking-io, concurrency]
---

## 1. 개념 설명 — 이벤트 루프란 무엇인가

**이벤트 루프(Event Loop)**는 단일 스레드(또는 소수의 스레드)에서 수많은 I/O 작업을 동시에 처리하기 위한 실행 모델이다. OS가 제공하는 비동기 I/O 알림 메커니즘(epoll, kqueue, IOCP)과 조합해 수만 개의 연결을 효율적으로 관리한다.

전통적인 **스레드 per 연결** 모델은 스레드 생성·컨텍스트 스위치 오버헤드가 크고, C10K(동시 10,000 연결) 문제를 유발한다. 이벤트 루프는 이 문제를 해결하는 핵심 패러다임이다.

### 이벤트 루프의 기본 구조

```
while true:
    I/O 이벤트 폴링 (epoll_wait / kevent / GetQueuedCompletionStatus)
    타이머 만료 콜백 처리
    I/O 준비 콜백 처리
    setImmediate / 다음 틱 처리
```

---

## 2. 왜 필요한가 — 동시성 vs. 병렬성

| 모델 | 동시성 | CPU 활용 | 복잡도 |
|------|--------|----------|--------|
| Thread-per-connection | OS 스케줄링 | 멀티코어 활용 | 락/경쟁 조건 |
| Event Loop (단일 스레드) | 이벤트 기반 | 1코어 | 콜백 지옥 |
| Event Loop + 워커 풀 | 이벤트 + 스레드 | 멀티코어 활용 | 균형적 |

이벤트 루프는 I/O-bound 작업(웹 서버, 데이터베이스 클라이언트)에 최적이지만, CPU-bound 작업(암호화, 이미지 처리)은 별도 워커 스레드/프로세스로 오프로드해야 한다.

---

## 3. libuv — Node.js의 심장

**libuv**는 C로 작성된 크로스 플랫폼 비동기 I/O 라이브러리로, Node.js의 런타임 기반이다.

### 3.1 libuv 이벤트 루프의 단계

```
   ┌─────────────────────────────────┐
   │           timers                │  ← setTimeout, setInterval 콜백
   └──────────────┬──────────────────┘
                  │
   ┌──────────────▼──────────────────┐
   │     pending callbacks           │  ← 이전 루프의 I/O 오류 콜백
   └──────────────┬──────────────────┘
                  │
   ┌──────────────▼──────────────────┐
   │       idle, prepare             │  ← 내부 사용
   └──────────────┬──────────────────┘
                  │
   ┌──────────────▼──────────────────┐
   │            poll                 │  ← I/O 이벤트 대기 및 처리
   └──────────────┬──────────────────┘
                  │
   ┌──────────────▼──────────────────┐
   │            check                │  ← setImmediate 콜백
   └──────────────┬──────────────────┘
                  │
   ┌──────────────▼──────────────────┐
   │       close callbacks           │  ← socket.on('close') 등
   └─────────────────────────────────┘
```

### 3.2 libuv 스레드 풀 (UV_THREADPOOL_SIZE)

파일시스템 I/O, DNS 조회(getaddrinfo), 암호화(crypto)는 libuv가 관리하는 **기본 4개** 스레드 풀에서 실행된다. 블로킹 작업을 메인 루프 스레드에서 격리한다.

```c
/* C로 libuv 타이머와 파일 읽기 예제 */
#include <uv.h>
#include <stdio.h>
#include <stdlib.h>

static uv_loop_t *loop;

void on_timer(uv_timer_t *handle) {
    printf("타이머 콜백: 1초마다 호출\n");
}

void on_read(uv_fs_t *req) {
    if (req->result < 0) {
        fprintf(stderr, "파일 읽기 오류: %s\n", uv_strerror((int)req->result));
    } else {
        printf("파일 읽기 완료: %zd 바이트\n", req->result);
    }
    uv_fs_req_cleanup(req);
    free(req);
}

int main(void) {
    loop = uv_default_loop();

    /* 타이머 등록 */
    uv_timer_t timer;
    uv_timer_init(loop, &timer);
    uv_timer_start(&timer, on_timer, 0, 1000);  /* 즉시 시작, 1초 반복 */

    /* 비동기 파일 읽기 */
    uv_fs_t *read_req = malloc(sizeof(uv_fs_t));
    uv_buf_t buf = uv_buf_init(malloc(4096), 4096);
    uv_fs_open(loop, read_req, "/etc/hostname", O_RDONLY, 0, NULL);
    /* uv_fs_read(loop, ...) 로 이어지는 비동기 체인 */

    printf("이벤트 루프 시작\n");
    return uv_run(loop, UV_RUN_DEFAULT);
}
```

---

## 4. asyncio — Python의 이벤트 루프

Python asyncio는 **selector 기반**(기본) 또는 **uvloop**(libuv 기반, 선택적) 이벤트 루프를 제공한다.

### 4.1 asyncio 내부 구조

```
asyncio 이벤트 루프
├── _ready 큐    ← 즉시 실행할 콜백
├── _scheduled   ← 지연 실행할 콜백 (타이머, 힙 정렬)
└── _selector    ← selectors.DefaultSelector (epoll/kqueue/select)
```

코루틴이 `await` 에서 중단되면 루프는 다른 태스크를 실행한다. 내부적으로 `Task` 객체가 `__step()` 메서드를 통해 코루틴을 재개한다.

### 4.2 asyncio 심화 예제 — 커스텀 이벤트 루프 훅

```python
import asyncio
import time
import selectors

class InstrumentedLoop(asyncio.SelectorEventLoop):
    """이벤트 루프 각 단계를 계측하는 커스텀 루프."""

    def _run_once(self):
        # 폴링 직전 ready 큐 크기 확인
        n_ready = len(self._ready)
        n_sched = len(self._scheduled)

        t0 = time.perf_counter()
        super()._run_once()
        elapsed = time.perf_counter() - t0

        if elapsed > 0.05:  # 50ms 이상 블로킹 감지
            print(f"[경고] 루프 블로킹 {elapsed*1000:.1f}ms | "
                  f"ready={n_ready} sched={n_sched}")


async def cpu_hog():
    """이벤트 루프를 블로킹하는 나쁜 예."""
    await asyncio.sleep(0)     # 다른 태스크에 양보
    time.sleep(0.1)            # CPU-bound 작업 (루프 블로킹!)
    return "done"


async def io_task(name, delay):
    """비동기 I/O 작업 시뮬레이션."""
    await asyncio.sleep(delay)
    print(f"[{name}] 완료 (delay={delay}s)")


async def main():
    # 병렬 태스크 실행
    tasks = [
        asyncio.create_task(io_task("A", 0.1)),
        asyncio.create_task(io_task("B", 0.2)),
        asyncio.create_task(io_task("C", 0.05)),
    ]
    await asyncio.gather(*tasks)

    # 블로킹 감지 테스트
    await cpu_hog()
    print("모든 태스크 완료")


# 커스텀 루프 사용
loop = InstrumentedLoop()
asyncio.set_event_loop(loop)
loop.run_until_complete(main())
loop.close()
```

### 4.3 asyncio 내부: Task 스텝 메커니즘

```python
import asyncio
import inspect

# 코루틴이 어떻게 Task 로 감싸지는지 확인
async def my_coro():
    await asyncio.sleep(0.01)
    return 42

async def inspect_task():
    coro = my_coro()
    task = asyncio.ensure_future(coro)
    
    print(f"Task 상태: {task._state}")  # PENDING
    await task
    print(f"Task 상태: {task._state}")  # FINISHED
    print(f"결과: {task.result()}")     # 42
    
    # Task 내부: _Task__step 콜백이 루프 _ready 에 등록됨
    print(f"Task 래핑: {type(task)}")

asyncio.run(inspect_task())
```

---

## 5. Tokio — Rust의 비동기 런타임

**Tokio**는 Rust의 `async/await`를 구동하는 비동기 런타임으로, libuv나 asyncio와 달리 **멀티스레드 작업 훔치기(work-stealing) 스케줄러**를 기본으로 사용한다.

### 5.1 Tokio 아키텍처

```
Tokio Runtime
├── Reactor (Mio → epoll/kqueue/IOCP)
│   └── I/O 이벤트를 Future의 Waker 에 연결
├── Scheduler (work-stealing threadpool)
│   ├── Worker 0 (run queue + 훔치기)
│   ├── Worker 1 (run queue + 훔치기)
│   └── ...Worker N
└── Timer Wheel (계층적 타이밍 휠)
```

Tokio의 `Future`는 **폴링(poll)** 모델이다. 런타임이 `Future::poll()` 을 호출하고, `Poll::Pending` 이 반환되면 `Waker`를 등록해 이후 I/O 완료 시 재폴링을 예약한다.

### 5.2 Tokio 심화 예제 — 커스텀 Future 직접 구현

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Waker};
use std::thread;
use std::time::Duration;

/// 지정 시간 후 완료되는 커스텀 Future
struct TimerFuture {
    shared: Arc<Mutex<SharedState>>,
}

struct SharedState {
    completed: bool,
    waker: Option<Waker>,
}

impl Future for TimerFuture {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let mut state = self.shared.lock().unwrap();
        if state.completed {
            Poll::Ready(())
        } else {
            // Waker 저장 — 완료 시 재폴링 요청
            state.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}

impl TimerFuture {
    fn new(duration: Duration) -> Self {
        let shared = Arc::new(Mutex::new(SharedState {
            completed: false,
            waker: None,
        }));

        let thread_shared = Arc::clone(&shared);
        thread::spawn(move || {
            thread::sleep(duration);
            let mut state = thread_shared.lock().unwrap();
            state.completed = true;
            if let Some(waker) = state.waker.take() {
                waker.wake();  // Tokio 스케줄러에 재폴링 요청
            }
        });

        TimerFuture { shared }
    }
}

#[tokio::main]
async fn main() {
    println!("타이머 시작");
    TimerFuture::new(Duration::from_millis(100)).await;
    println!("100ms 후 완료");

    // work-stealing 확인: 여러 태스크를 생성해 스레드 분산
    let handles: Vec<_> = (0..8)
        .map(|i| {
            tokio::spawn(async move {
                tokio::time::sleep(Duration::from_millis(10 * i)).await;
                println!("태스크 {} 완료 (스레드: {:?})", i, thread::current().id());
            })
        })
        .collect();

    for h in handles {
        h.await.unwrap();
    }
}
```

---

## 6. 세 런타임 비교

| 항목 | libuv | asyncio | Tokio |
|------|-------|---------|-------|
| 언어 | C | Python | Rust |
| 스레드 모델 | 싱글 + 워커 풀 | 싱글 (기본) | 멀티 work-stealing |
| I/O 백엔드 | epoll/kqueue/IOCP | selectors / uvloop | mio (epoll/kqueue/IOCP) |
| 타이머 | 힙 | 힙 | 계층적 타이밍 휠 |
| CPU-bound 해결 | workerData(worker_threads) | run_in_executor | spawn_blocking |
| 메모리 안전 | 수동 | GIL + GC | 소유권 시스템 |

---

## 7. 주의사항과 팁

### 이벤트 루프 블로킹 방지

모든 런타임에서 메인 루프 스레드를 블로킹하면 전체 처리량이 저하된다:

- **Node.js**: `fs.readFileSync` 대신 `fs.promises.readFile` 사용
- **asyncio**: `time.sleep()` 대신 `await asyncio.sleep()`, CPU 작업은 `loop.run_in_executor()`
- **Tokio**: `std::thread::sleep` 대신 `tokio::time::sleep`, `spawn_blocking`으로 블로킹 작업 격리

### Tokio 런타임 플레이버 선택

```rust
// 단일 스레드 (임베디드 등 리소스 제한 환경)
#[tokio::main(flavor = "current_thread")]
async fn main() { ... }

// 멀티 스레드 (기본, 서버 환경)
#[tokio::main(flavor = "multi_thread", worker_threads = 4)]
async fn main() { ... }
```

---

## 참고 자료

- [libuv 공식 문서 — Design Overview](https://docs.libuv.org/en/v1.x/design.html)
- [Python asyncio — Event Loop 내부](https://docs.python.org/3/library/asyncio-eventloop.html)
- [Tokio 공식 튜토리얼](https://tokio.rs/tokio/tutorial)
- [이벤트 루프 — Wikipedia](https://en.wikipedia.org/wiki/Event_loop)
