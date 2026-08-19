---
layout: post
title: "epoll과 이벤트 기반 I/O 멀티플렉싱 완전 정복: Nginx·Node.js가 수십만 연결을 처리하는 비밀"
date: 2026-08-14
categories: [cs, computer-science]
tags: [epoll, io-multiplexing, event-driven, linux, nginx, nodejs, select, poll, network, c]
---

## I/O 멀티플렉싱이란

서버 프로그램은 수천 개의 클라이언트를 동시에 처리해야 합니다. 가장 단순한 방법은 각 연결마다 스레드 하나씩을 할당하는 방식인데, 이는 스레드마다 기본 수 MB의 스택 메모리가 필요하며 운영체제 스케줄러가 끊임없이 컨텍스트 스위칭을 수행해야 합니다. 1만 개의 동시 연결을 이런 방식으로 처리하면 수 GB의 메모리가 소진되고, 실제 유용한 작업보다 스위칭 오버헤드가 더 커지는 이른바 **C10K 문제**에 봉착합니다.

**I/O 멀티플렉싱(I/O Multiplexing)**은 하나의 스레드로 여러 파일 디스크립터(File Descriptor, FD)를 동시에 감시하다가, I/O 이벤트가 발생한 FD에만 처리를 수행하는 방식입니다. CPU 시간을 I/O 대기에 낭비하지 않고 실제 작업에만 사용할 수 있습니다.

Linux에서는 세 가지 메커니즘이 역사적으로 발전했습니다: `select` → `poll` → `epoll`. 각각의 설계를 이해하면 epoll이 왜 강력한지 자연스럽게 알 수 있습니다.

## select와 poll의 구조와 한계

### select(2): 1983년의 유산

`select(2)`는 BSD 4.2에서 도입된 오래된 API입니다. 비트마스크로 감시할 FD 집합을 커널에 전달하면, 이벤트가 발생할 때까지 블록했다가 깨어납니다:

```c
#include <sys/select.h>

int select(int nfds,
           fd_set *readfds,   // 읽기 가능 여부 감시할 FD 집합
           fd_set *writefds,  // 쓰기 가능 여부 감시할 FD 집합
           fd_set *exceptfds, // 예외 상황 감시할 FD 집합
           struct timeval *timeout);
```

**구조적 한계:**

1. **FD_SETSIZE 제한**: `fd_set`은 고정 크기 비트배열(보통 1024비트). FD 번호 1024 이상은 감시 불가
2. **O(n) 탐색**: 깨어난 후 어떤 FD에 이벤트가 발생했는지 전체 비트셋을 순회해야 함
3. **매 호출마다 재전달 + 덮어쓰기**: 커널이 상태를 저장하지 않아 매번 전체 FD 집합을 재구성해서 넘겨야 하고, 반환 후 원본 집합이 수정되어 원본 보존을 위한 복사가 필요

### poll(2): FD 개수 제한 해소

`poll(2)`은 `pollfd` 구조체 배열을 사용해 FD 개수 제한을 없앴습니다:

```c
#include <poll.h>

struct pollfd {
    int   fd;       /* 감시할 파일 디스크립터 */
    short events;   /* 감시 원하는 이벤트 (POLLIN, POLLOUT, ...) */
    short revents;  /* 반환: 실제 발생한 이벤트 */
};

int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

FD 개수 제한은 없어졌지만 핵심 문제는 그대로입니다: **매 호출마다 전체 FD 배열을 사용자 공간 → 커널로 복사**해야 하고, 커널은 이를 처음부터 끝까지 순회해 이벤트를 확인합니다.

연결이 10만 개이고 이벤트가 1개만 발생해도 매번 10만 개의 `pollfd` 구조체를 복사하고 순회해야 합니다. O(n) 비용이 이벤트 발생 빈도와 무관하게 지속적으로 발생합니다.

## epoll: 이벤트 발생 수에 비례하는 O(1) 설계

Linux 2.5.44(2002년)에서 도입된 `epoll`은 관심 FD를 **커널에 지속 등록**해두고, 이벤트가 발생한 FD만 골라 반환하는 구조입니다.

### 세 가지 핵심 시스템 콜

```c
#include <sys/epoll.h>

/* 1. epoll 인스턴스(커널 자료구조) 생성, epfd 반환 */
int epoll_create1(int flags);
/* flags: 0 또는 EPOLL_CLOEXEC (exec 시 FD 자동 닫기) */

/* 2. FD를 관심 목록에 추가/수정/삭제 */
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
/* op: EPOLL_CTL_ADD, EPOLL_CTL_MOD, EPOLL_CTL_DEL */

struct epoll_event {
    uint32_t events;   /* EPOLLIN, EPOLLOUT, EPOLLERR, EPOLLHUP, EPOLLET 등 */
    epoll_data_t data; /* 사용자 데이터: fd, ptr, u32, u64 중 하나 */
};

/* 3. 이벤트 발생 대기 */
int epoll_wait(int epfd, struct epoll_event *events,
               int maxevents, int timeout);
/* 반환값: 준비된 FD 수 (timeout이면 0, 오류면 -1) */
```

### 내부 자료구조

epoll 인스턴스는 커널 내부에 두 개의 자료구조를 유지합니다:

- **Interest List(관심 목록)**: 감시 중인 FD와 이벤트 마스크의 집합. **레드-블랙 트리**로 구현되어 O(log n) 삽입/수정/삭제 지원
- **Ready List(준비 목록)**: 이벤트가 발생한 FD의 목록. **이중 연결 리스트**로 구현

FD에 이벤트가 발생하면 커널의 소켓/파일 드라이버가 epoll의 Ready List에 직접 추가합니다. `epoll_wait()`는 Ready List를 사용자 공간으로 복사할 뿐이므로 **O(발생한 이벤트 수)**의 복잡도입니다. 감시 FD가 10만 개여도 이벤트가 1개면 거의 즉시 반환됩니다.

| | select | poll | epoll |
|---|---|---|---|
| 최대 FD 수 | 1024 (기본) | 무제한 | 무제한 |
| 등록 방식 | 매 호출마다 | 매 호출마다 | 한 번 등록 후 유지 |
| 커널→유저 복사 | 전체 FD 집합 | 전체 FD 배열 | 이벤트 발생한 FD만 |
| 이벤트 탐색 | O(n) | O(n) | O(1) |
| 지원 OS | 범용 | 범용 | Linux 전용 |

### 에지 트리거(ET) vs 레벨 트리거(LT)

epoll은 두 가지 트리거 모드를 지원합니다:

**레벨 트리거(LT, Level-Triggered)**: 기본 동작. FD에 읽지 않은 데이터가 남아있는 한 `epoll_wait()`를 호출할 때마다 계속 이벤트를 반환합니다. `select`/`poll`과 동일한 동작으로, 구현이 단순합니다.

**에지 트리거(ET, Edge-Triggered)**: `EPOLLET` 플래그로 활성화. FD 상태가 **변경되는 순간에만** 이벤트를 반환합니다. 더 효율적이지만, 이벤트를 받은 후 `read()`/`recv()`를 `EAGAIN`이 반환될 때까지 반복해 모든 데이터를 읽어야 합니다.

## 실제 구현: epoll 기반 에코 서버

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <netinet/in.h>
#include <errno.h>

#define MAX_EVENTS 128
#define BUF_SIZE   4096
#define PORT       8080

static void set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags == -1) { perror("fcntl F_GETFL"); exit(1); }
    if (fcntl(fd, F_SETFL, flags | O_NONBLOCK) == -1) {
        perror("fcntl F_SETFL"); exit(1);
    }
}

int main(void) {
    int server_fd = socket(AF_INET, SOCK_STREAM | SOCK_CLOEXEC, 0);
    if (server_fd == -1) { perror("socket"); exit(1); }

    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in addr = {
        .sin_family      = AF_INET,
        .sin_addr.s_addr = INADDR_ANY,
        .sin_port        = htons(PORT)
    };
    if (bind(server_fd, (struct sockaddr *)&addr, sizeof(addr)) == -1) {
        perror("bind"); exit(1);
    }
    listen(server_fd, SOMAXCONN);
    set_nonblocking(server_fd);

    int epfd = epoll_create1(EPOLL_CLOEXEC);
    if (epfd == -1) { perror("epoll_create1"); exit(1); }

    struct epoll_event ev = {
        .events  = EPOLLIN | EPOLLET,
        .data.fd = server_fd
    };
    epoll_ctl(epfd, EPOLL_CTL_ADD, server_fd, &ev);

    struct epoll_event events[MAX_EVENTS];
    char buf[BUF_SIZE];
    printf("[epoll 에코 서버] 포트 %d에서 수신 대기 중...\n", PORT);

    for (;;) {
        int n = epoll_wait(epfd, events, MAX_EVENTS, -1);
        if (n == -1) {
            if (errno == EINTR) continue;
            perror("epoll_wait"); break;
        }

        for (int i = 0; i < n; i++) {
            int fd     = events[i].data.fd;
            uint32_t e = events[i].events;

            if (fd == server_fd) {
                for (;;) {
                    int cfd = accept4(server_fd, NULL, NULL,
                                      SOCK_NONBLOCK | SOCK_CLOEXEC);
                    if (cfd == -1) {
                        if (errno == EAGAIN || errno == EWOULDBLOCK) break;
                        perror("accept4"); break;
                    }
                    ev.events  = EPOLLIN | EPOLLET | EPOLLRDHUP;
                    ev.data.fd = cfd;
                    epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &ev);
                    printf("[연결] fd=%d\n", cfd);
                }
            } else if (e & (EPOLLRDHUP | EPOLLHUP | EPOLLERR)) {
                epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                close(fd);
                printf("[종료] fd=%d\n", fd);
            } else if (e & EPOLLIN) {
                for (;;) {
                    ssize_t r = read(fd, buf, sizeof(buf));
                    if (r == -1) {
                        if (errno == EAGAIN || errno == EWOULDBLOCK) break;
                        perror("read");
                        close(fd); break;
                    }
                    if (r == 0) {
                        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                        close(fd); break;
                    }
                    write(fd, buf, (size_t)r);
                }
            }
        }
    }

    close(epfd);
    close(server_fd);
    return 0;
}
```

```bash
gcc -O2 -Wall -o echo_server echo_server.c
./echo_server &
echo "Hello, epoll!" | nc 127.0.0.1 8080
```

## Nginx와 Node.js의 이벤트 루프

### Nginx의 epoll 활용

Nginx는 마스터 프로세스와 여러 워커 프로세스로 구성됩니다. 각 워커 프로세스는 **독립적인 epoll 인스턴스 하나**를 유지하며 단일 스레드 이벤트 루프로 모든 연결을 처리합니다:

```nginx
worker_processes auto;

events {
    worker_connections 65536;
    use epoll;
    multi_accept on;
    accept_mutex off;
}

http {
    keepalive_timeout 65;
    keepalive_requests 1000;
}
```

### Node.js의 libuv와 이벤트 루프

Node.js는 **libuv** 라이브러리를 통해 epoll을 활용합니다. V8 엔진의 JavaScript 실행과 I/O 이벤트를 단일 스레드 이벤트 루프로 통합합니다:

```javascript
const net = require('net');

const server = net.createServer((socket) => {
    socket.on('data', (data) => {
        socket.write(data);
    });
    socket.on('end', () => socket.destroy());
    socket.on('error', (err) => {
        console.error('소켓 오류:', err.message);
        socket.destroy();
    });
});

server.listen(8080, '0.0.0.0', () => {
    console.log('Node.js 에코 서버 시작 (포트 8080)');
});

let activeConnections = 0;
server.on('connection', (socket) => {
    activeConnections++;
    socket.on('close', () => activeConnections--);
});

setInterval(() => {
    const memMB = (process.memoryUsage().rss / 1024 / 1024).toFixed(1);
    console.log(`활성 연결: ${activeConnections}, 메모리: ${memMB}MB`);
}, 5000);
```

Node.js의 이벤트 루프 페이즈:

```
   ┌─────────────────────────────────────────┐
   │               이벤트 루프                │
   │                                         │
   │  timers → pending callbacks → idle      │
   │      → poll (epoll_wait) → check        │
   │      → close callbacks → (반복)         │
   └─────────────────────────────────────────┘
```

**poll 페이즈**에서 libuv가 `epoll_wait()`를 호출해 I/O 이벤트를 기다립니다.

## 주의사항과 팁

### 1. EPOLLONESHOT: 멀티스레드 환경의 경쟁 조건 방지

멀티스레드 서버에서 여러 스레드가 같은 epoll 인스턴스를 공유하면 경쟁 조건(race condition)이 발생할 수 있습니다. `EPOLLONESHOT` 플래그를 사용하면 이벤트 발생 후 해당 FD를 자동으로 비활성화합니다:

```c
ev.events = EPOLLIN | EPOLLET | EPOLLONESHOT;
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);

// 처리 완료 후 EPOLL_CTL_MOD로 재활성화
ev.events = EPOLLIN | EPOLLET | EPOLLONESHOT;
epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev);
```

### 2. Thundering Herd 문제와 SO_REUSEPORT

여러 워커 프로세스가 같은 서버 소켓을 epoll로 감시하면, 새 연결 시 모든 워커가 깨어나지만 하나만 `accept()`에 성공합니다. Linux 3.9+의 `SO_REUSEPORT`로 해결합니다:

```c
int opt = 1;
setsockopt(server_fd, SOL_SOCKET, SO_REUSEPORT, &opt, sizeof(opt));
// 이제 커널이 새 연결을 라운드 로빈으로 워커에 분배
```

### 3. epoll vs io_uring 선택 기준

Linux 5.1+에서 도입된 `io_uring`은 시스템 콜을 배치 처리하고 커널-사용자 공간 메모리를 공유 링 버퍼로 연결해 일부 워크로드에서 더 낮은 레이턴시를 보입니다. 단 커널 5.10+를 요구하고 코드 복잡도가 높습니다. 현재 대부분의 프로덕션 시스템에서는 epoll을 여전히 사용합니다.

## 참고 자료
- [epoll(7) - Ubuntu Manpages](https://manpages.ubuntu.com/manpages/focal/man7/epoll.7.html)
- [epoll_create(2) - Ubuntu Manpages](https://manpages.ubuntu.com/manpages/focal/man2/epoll_create.2.html)
- [epoll_ctl(2) - Ubuntu Manpages](https://manpages.ubuntu.com/manpages/focal/man2/epoll_ctl.2.html)
- [select(2) - Ubuntu Manpages](https://manpages.ubuntu.com/manpages/focal/man2/select.2.html)
