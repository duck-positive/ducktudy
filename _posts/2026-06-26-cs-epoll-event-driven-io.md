---
layout: post
title: "epoll과 이벤트 주도 I/O 완전 정복: Node.js와 Nginx가 수백만 연결을 처리하는 방법"
date: 2026-06-26
categories: [cs, computer-science]
tags: [epoll, event-loop, io-multiplexing, linux, networking, c10k, async-io, select, poll]
---

## 개요

Nginx는 단일 워커 프로세스로 수만 개의 동시 연결을 처리한다. Node.js는 싱글 스레드임에도 높은 I/O 처리량을 자랑한다. 이 마법의 핵심에는 **epoll**이라는 리눅스 커널 시스템 콜이 있다. 이 글에서는 epoll의 내부 동작 원리, 기존 I/O 모델과의 차이, 그리고 실제 코드로 이벤트 루프를 직접 구현하는 방법까지 깊이 있게 다룬다.

---

## 1. 문제의 시작: C10K 문제

2001년 댄 케겔(Dan Kegel)은 "C10K 문제"를 제기했다. 웹 서버 하나가 동시에 1만(10K) 개의 클라이언트 연결을 처리할 수 있어야 한다는 도전이었다. 당시 표준적인 접근 방식은 두 가지였다.

### 1.1 스레드 기반 모델 (Thread-per-connection)

연결 하나당 OS 스레드를 하나씩 할당하는 방식이다. Apache 웹 서버가 초창기에 사용한 방법이다. 문제는 스레드 자체가 비싸다는 것이다.

- **메모리**: 스레드 하나당 스택 메모리 기본 8MB (Linux 기준)
- **컨텍스트 스위칭**: 1만 개 스레드가 CPU를 나눠 쓰면 스케줄링 오버헤드가 폭발적으로 증가
- **동기화 비용**: 공유 자원 접근 시 락 경합(Lock Contention)

1만 개 연결이라면 이론상 80GB 스택 메모리가 필요하다. 현실적으로 불가능하다.

### 1.2 select/poll 기반 모델

이를 해결하기 위해 등장한 것이 I/O 멀티플렉싱이다. 단일 스레드에서 여러 소켓을 동시에 감시하는 방법이다.

```c
// select 기반 I/O 멀티플렉싱 — O(n) 문제를 보여주는 예시
#include <sys/select.h>
#include <unistd.h>
#include <stdio.h>

void select_example(int* fds, int n) {
    fd_set read_fds;
    int max_fd = 0;

    FD_ZERO(&read_fds);
    for (int i = 0; i < n; i++) {
        FD_SET(fds[i], &read_fds);
        if (fds[i] > max_fd) max_fd = fds[i];
    }

    // select는 매 호출마다 전체 fd_set을 커널에 복사하고
    // 커널이 O(n)으로 모든 fd를 스캔한다.
    struct timeval timeout = {5, 0};  // 5초 타임아웃
    int ready = select(max_fd + 1, &read_fds, NULL, NULL, &timeout);

    if (ready > 0) {
        for (int i = 0; i < n; i++) {
            if (FD_ISSET(fds[i], &read_fds)) {
                // fds[i]가 읽기 가능한 상태
                printf("fd %d is ready\n", fds[i]);
            }
        }
    }
}
```

`select`와 `poll`의 근본적인 한계는 **O(n) 복잡도**다. 감시하는 파일 디스크립터가 10만 개라면 매번 10만 개를 순회해야 한다. 또한 `select`는 `FD_SETSIZE`(보통 1024) 제한이 있어 대규모 연결 처리 자체가 불가능하다.

---

## 2. epoll의 등장: O(1) 이벤트 통지

epoll은 Linux 커널 2.5.44(2002년)에서 처음 도입된 I/O 이벤트 알림 시스템이다. `select`/`poll`의 O(n) 문제를 해결하여 **O(1)** 또는 **O(이벤트 수)** 복잡도를 달성한다.

### 2.1 epoll의 세 가지 시스템 콜

epoll API는 세 개의 시스템 콜로 구성된다.

| 시스템 콜 | 역할 |
|-----------|------|
| `epoll_create1(flags)` | epoll 인스턴스(커널 내부 자료구조) 생성, fd 반환 |
| `epoll_ctl(epfd, op, fd, event)` | 관심 fd 등록(EPOLL_CTL_ADD)/수정/삭제 |
| `epoll_wait(epfd, events, maxevents, timeout)` | 이벤트가 발생한 fd 목록 수신, 블로킹 대기 |

핵심은 **fd 등록(epoll_ctl)** 과 **이벤트 대기(epoll_wait)** 가 분리되어 있다는 점이다. 커널은 등록된 fd를 **레드-블랙 트리(Red-Black Tree)** 로 관리하고, 이벤트 발생 시 **이중 연결 리스트(Ready List)** 에 추가한다. `epoll_wait`는 이 Ready List만 반환하므로 O(이벤트 수) 복잡도를 가진다.

### 2.2 수준 트리거 vs 엣지 트리거

epoll은 두 가지 동작 모드를 지원한다.

- **LT (Level-Triggered, 기본값)**: fd가 읽기 가능한 상태인 동안 계속 이벤트를 통지한다. `select`/`poll`과 동일한 의미론. 데이터를 일부만 읽어도 다음 `epoll_wait`에서 다시 통지된다.
- **ET (Edge-Triggered, `EPOLLET` 플래그)**: 상태 변화가 일어난 순간에만 한 번 통지한다. 더 효율적이지만, **반드시 루프를 돌며 `EAGAIN`이 올 때까지 모든 데이터를 읽어야** 한다. 그렇지 않으면 다음 이벤트를 영영 받지 못한다.

---

## 3. 실제 구현: C로 만드는 에코 서버

다음은 epoll을 사용한 비동기 에코 서버의 완전한 구현이다.

```c
// epoll_echo_server.c — epoll 기반 비동기 에코 서버
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <fcntl.h>
#include <netinet/in.h>
#include <sys/epoll.h>
#include <sys/socket.h>

#define PORT        8080
#define MAX_EVENTS  64
#define BUF_SIZE    4096

// fd를 논블로킹 모드로 설정
static void set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main(void) {
    // 1. 리슨 소켓 생성
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    set_nonblocking(listen_fd);

    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port   = htons(PORT),
        .sin_addr.s_addr = INADDR_ANY,
    };
    bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(listen_fd, SOMAXCONN);

    // 2. epoll 인스턴스 생성
    int epfd = epoll_create1(EPOLL_CLOEXEC);

    // 3. 리슨 소켓을 epoll에 등록 (읽기 이벤트 + 엣지 트리거)
    struct epoll_event ev = {
        .events  = EPOLLIN | EPOLLET,
        .data.fd = listen_fd,
    };
    epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);

    struct epoll_event events[MAX_EVENTS];
    char buf[BUF_SIZE];

    printf("Echo server listening on port %d\n", PORT);

    // 4. 이벤트 루프
    for (;;) {
        // epoll_wait: 이벤트 발생까지 블로킹 (-1 = 무한 대기)
        int n = epoll_wait(epfd, events, MAX_EVENTS, -1);

        for (int i = 0; i < n; i++) {
            int fd = events[i].data.fd;

            if (fd == listen_fd) {
                // 새 연결 수락 (ET 모드: 루프로 모든 연결 수락)
                for (;;) {
                    int conn_fd = accept(listen_fd, NULL, NULL);
                    if (conn_fd == -1) {
                        if (errno == EAGAIN || errno == EWOULDBLOCK) break;
                        perror("accept"); break;
                    }
                    set_nonblocking(conn_fd);

                    ev.events  = EPOLLIN | EPOLLET;
                    ev.data.fd = conn_fd;
                    epoll_ctl(epfd, EPOLL_CTL_ADD, conn_fd, &ev);
                    printf("New connection: fd=%d\n", conn_fd);
                }
            } else if (events[i].events & EPOLLIN) {
                // 클라이언트 데이터 수신 (ET 모드: EAGAIN까지 루프)
                for (;;) {
                    ssize_t nread = read(fd, buf, sizeof(buf));
                    if (nread == 0) {
                        // 연결 종료
                        printf("Connection closed: fd=%d\n", fd);
                        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                        close(fd);
                        break;
                    }
                    if (nread == -1) {
                        if (errno == EAGAIN) break;  // 다 읽음
                        perror("read"); break;
                    }
                    // 에코: 받은 데이터 그대로 전송
                    write(fd, buf, nread);
                }
            }
        }
    }

    close(listen_fd);
    close(epfd);
    return 0;
}
```

이 서버는 단일 스레드로 수천 개의 연결을 동시에 처리한다. 핵심은 `epoll_wait`가 I/O 이벤트가 발생한 fd만 돌려준다는 것이다. 10만 개의 연결 중 실제로 데이터를 보낸 100개의 fd만 처리하면 된다.

---

## 4. Python asyncio: epoll 위의 고수준 추상화

Python의 `asyncio`는 내부적으로 epoll(Linux), kqueue(macOS), IOCP(Windows)를 사용한다.

```python
# asyncio_echo_server.py — asyncio 기반 에코 서버
# 내부적으로 epoll이 동작한다.
import asyncio

HOST = '127.0.0.1'
PORT = 8888

async def handle_client(reader: asyncio.StreamReader,
                         writer: asyncio.StreamWriter) -> None:
    addr = writer.get_extra_info('peername')
    print(f"Connected from {addr}")

    try:
        while True:
            # await 지점에서 이벤트 루프가 다른 코루틴에 제어를 넘긴다.
            # 커널 수준에서는 epoll_wait가 이 소켓의 EPOLLIN 이벤트를 기다린다.
            data = await reader.read(4096)
            if not data:
                break

            message = data.decode(errors='replace')
            print(f"Received {len(data)} bytes from {addr}")

            writer.write(data)          # 버퍼에 넣기
            await writer.drain()        # 실제 전송 완료 대기

    except asyncio.CancelledError:
        pass
    except ConnectionResetError:
        pass
    finally:
        writer.close()
        await writer.wait_closed()
        print(f"Disconnected from {addr}")

async def main() -> None:
    # asyncio.start_server 내부에서 소켓 생성, O_NONBLOCK 설정,
    # epoll에 EPOLLIN 등록이 모두 처리된다.
    server = await asyncio.start_server(
        handle_client, HOST, PORT,
        backlog=1000,
    )
    addrs = ', '.join(str(s.getsockname()) for s in server.sockets)
    print(f"Serving on {addrs}")

    async with server:
        await server.serve_forever()

if __name__ == '__main__':
    # uvloop을 설치하면 libuv(epoll 래퍼)를 사용해 더 빠르게 동작한다:
    # import uvloop; asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())
    asyncio.run(main())
```

asyncio의 이벤트 루프는 본질적으로 epoll의 파이썬 래퍼다. `await reader.read()`는 커널에 "이 fd에 데이터가 오면 깨워줘"를 등록하고 다른 코루틴을 실행하는 것과 같다.

---

## 5. 주의사항 및 실전 팁

### 5.1 엣지 트리거(ET) 사용 시 필수 규칙

ET 모드에서는 데이터를 남기지 않는 것이 핵심이다.

```c
// WRONG: ET 모드에서 부분 읽기 — 나머지 데이터를 영영 놓친다
ssize_t nread = read(fd, buf, 128);  // 실제 데이터가 4096바이트라면?

// CORRECT: EAGAIN이 올 때까지 루프
for (;;) {
    ssize_t nread = read(fd, buf, sizeof(buf));
    if (nread <= 0) {
        if (nread == -1 && errno == EAGAIN) break;  // 모두 읽음
        // nread == 0: 연결 종료, nread == -1 + 기타 errno: 오류
        close(fd); break;
    }
    process(buf, nread);
}
```

### 5.2 EPOLLONESHOT으로 다중 스레드 안전성 확보

멀티 스레드 서버에서 같은 fd의 이벤트가 여러 스레드에서 동시에 처리되면 경쟁 조건이 발생한다. `EPOLLONESHOT`을 사용하면 한 번 이벤트를 받은 후 해당 fd가 자동으로 비활성화된다.

```c
// 멀티스레드 환경에서 안전한 epoll 등록
struct epoll_event ev = {
    .events  = EPOLLIN | EPOLLET | EPOLLONESHOT,
    .data.fd = conn_fd,
};
epoll_ctl(epfd, EPOLL_CTL_ADD, conn_fd, &ev);

// 처리 완료 후 다시 활성화 (다른 스레드에서)
ev.events = EPOLLIN | EPOLLET | EPOLLONESHOT;
epoll_ctl(epfd, EPOLL_CTL_MOD, conn_fd, &ev);
```

### 5.3 EPOLLRDHUP으로 연결 종료 감지

`EPOLLRDHUP`(Linux 2.6.17+)는 상대방이 연결을 닫았을 때(half-close 포함) 이벤트를 발생시킨다. `read`가 0을 반환하는 것을 기다리는 것보다 더 빠르게 연결 종료를 감지할 수 있다.

### 5.4 timerfd, eventfd와 결합

epoll은 소켓뿐 아니라 리눅스의 모든 "파일 디스크립터"를 감시할 수 있다. `timerfd_create`로 타이머를, `eventfd`로 스레드 간 시그널링을 epoll에 통합하면 완전한 이벤트 주도 시스템을 구축할 수 있다.

### 5.5 `MAX_EVENTS` 조정

`epoll_wait`의 `maxevents` 파라미터는 한 번의 호출로 처리할 최대 이벤트 수다. 이 값이 너무 작으면 이벤트가 누적되고, 너무 크면 스택 메모리 낭비가 발생한다. 보통 64~256 정도가 균형 잡힌 값이다.

---

## 6. epoll vs kqueue vs IOCP 비교

| | epoll (Linux) | kqueue (BSD/macOS) | IOCP (Windows) |
|---|---|---|---|
| 모델 | 준비 기반(Readiness) | 준비 기반 | 완료 기반(Completion) |
| 연결 수 | 수백만 | 수백만 | 수백만 |
| 멀티 이벤트 타입 | fd, timerfd, eventfd, signalfd | fd, timer, signal, proc, vnode | 소켓, 파일, pipe |
| 사용 라이브러리 | libuv, libev | libuv, libev | libuv, Boost.Asio |

Node.js가 사용하는 **libuv**는 플랫폼에 따라 자동으로 epoll/kqueue/IOCP를 선택한다. 그래서 Node.js 코드는 OS에 무관하게 고성능 비동기 I/O를 사용할 수 있다.

---

## 7. 정리

epoll은 C10K 문제를 해결한 핵심 기술이다. fd 등록(epoll_ctl)과 이벤트 대기(epoll_wait)를 분리하고, 커널 내부에 레드-블랙 트리와 Ready List를 유지함으로써 O(1) 이벤트 통지를 달성한다. 엣지 트리거 모드에서는 반드시 EAGAIN까지 데이터를 완전히 소비해야 하며, EPOLLONESHOT을 통해 멀티스레드 환경에서도 안전하게 사용할 수 있다.

오늘날의 고성능 서버(Nginx, Redis, Node.js, Envoy 등)는 모두 이 epoll 위에 구축된 이벤트 루프를 기반으로 동작한다. 비동기 I/O의 본질을 이해하면 이 서버들의 성능 특성과 한계를 정확하게 파악할 수 있다.

## 참고 자료
- [epoll(7) Linux Manual Page](https://linux.die.net/man/7/epoll)
- [epoll_ctl(2) Linux Manual Page](https://linux.die.net/man/2/epoll_ctl)
- [The C10K Problem — Dan Kegel](http://www.kegel.com/c10k.html)
- [A look at dynamic linking — LWN.net](https://lwn.net/Articles/961117/)
