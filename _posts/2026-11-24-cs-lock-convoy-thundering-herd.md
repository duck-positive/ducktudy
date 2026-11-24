---
layout: post
title: "락 콘보이(Lock Convoy)와 Thundering Herd: 동시성 성능 함정 완전 분석"
date: 2026-11-24
categories: [cs, computer-science]
tags: [concurrency, lock-convoy, thundering-herd, mutex, epoll, linux, performance, threading, synchronization]
---

멀티스레드 프로그램에서 락(lock)을 올바르게 사용하고 있음에도 성능이 예상보다 훨씬 나쁜 경우가 있습니다. 락 콘보이(Lock Convoy)와 Thundering Herd는 그런 상황을 만드는 대표적인 동시성 함정입니다. 이 글에서는 두 문제의 메커니즘, 감지 방법, 그리고 실전 해결책을 다룹니다.

## 개념 설명

### 락 콘보이(Lock Convoy)

락 콘보이는 1979년 "The Convoy Phenomenon" 논문에서 처음 기술된 고전적인 성능 문제입니다. 여러 스레드가 같은 뮤텍스를 반복적으로 획득하려 할 때, 과도한 컨텍스트 스위치로 인해 처리량이 급격히 떨어지는 현상입니다.

#### 메커니즘

```
초기 상태: 스레드 A가 Lock L을 보유 중

1. 스레드 B가 Lock L 요청 → 블록 → OS가 B를 대기 큐에 삽입
2. 스레드 A가 Lock L 해제 → OS가 B를 깨움 → 컨텍스트 스위치 발생
3. 스레드 A가 즉시 Lock L을 다시 요청
   → B가 이미 큐에 있으므로 B가 우선 획득 (공정 스케줄러)
   → A가 대기 큐에 삽입 → 또 컨텍스트 스위치
4. B가 Lock L 보유 중 → C 스레드 도착 → 큐에 대기 → 컨텍스트 스위치
5. ∞ 반복: 스레드들이 "호위 행렬(convoy)"을 형성
```

핵심 문제: 락 해제 직후 같은 스레드가 즉시 재획득할 수 없어서, 불필요한 컨텍스트 스위치가 연속으로 발생합니다. 컨텍스트 스위치 비용(수 마이크로초)이 임계 구역 실행 시간보다 크면 처리량이 급격히 감소합니다.

### Thundering Herd (사나운 무리 문제)

다수의 스레드/프로세스가 하나의 이벤트를 기다리다가, 그 이벤트 발생 시 모두 한꺼번에 깨어나지만 오직 하나만 처리할 수 있어 나머지는 헛되이 CPU를 낭비하는 현상입니다.

```
상황: N개 워커 스레드가 accept() / epoll_wait()으로 클라이언트 연결 대기

이벤트 발생: 클라이언트 1개 연결 →
  → 스레드 1 깨어남 (연결 처리)
  → 스레드 2 깨어남 (accept 실패, EAGAIN → 다시 대기)
  → 스레드 3 깨어남 (accept 실패 → 다시 대기)
  ...
  → 스레드 N 깨어남 (accept 실패 → 다시 대기)
  
  비용: N개 스레드 모두 스케줄링 → 컨텍스트 스위치 N회
  이득: 연결 처리 1회
```

N이 커질수록 낭비가 선형으로 증가합니다.

## 왜 문제인가

### 성능 측면

- **락 콘보이**: 임계 구역이 짧을수록 더 치명적입니다. 임계 구역 10μs, 컨텍스트 스위치 5μs라면 50%가 낭비입니다.
- **Thundering Herd**: 워커 수가 많을수록 더 치명적입니다. 1,000 워커에 연결 1개 → 999회의 헛된 wakeup.

### 감지 방법

```bash
# 락 콘보이 감지: perf로 mutex 경쟁 측정
perf stat -e lock:contention_begin,lock:contention_end ./my_app

# context switch 비율 확인
pidstat -w 1 -p <pid>

# Thundering Herd: strace로 불필요한 accept 실패 카운트
strace -c -e trace=accept4 ./my_app 2>&1 | grep accept4
```

## 실제 구현 예제

### 예제 1: 락 콘보이 재현과 해결 (Go)

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"sync/atomic"
	"time"
)

const (
	numWorkers  = 8
	testDuration = 200 * time.Millisecond
)

// 패턴 1: 락 콘보이 유발
// 임계 구역이 매우 짧고, 해제 직후 즉시 재획득 시도
func convoyProne(result *int64) {
	var mu sync.Mutex
	var counter int64
	done := make(chan struct{})
	var wg sync.WaitGroup

	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for {
				select {
				case <-done:
					return
				default:
					mu.Lock()
					counter++ // 아주 짧은 임계 구역
					mu.Unlock()
					// 즉시 재시도 → 콘보이 발생
				}
			}
		}()
	}

	time.Sleep(testDuration)
	close(done)
	wg.Wait()
	atomic.StoreInt64(result, counter)
}

// 패턴 2: runtime.Gosched()로 콘보이 완화
// 락 해제 후 스케줄러에 양보 → 다른 스레드가 충분히 실행될 기회 부여
func convoyMitigated(result *int64) {
	var mu sync.Mutex
	var counter int64
	done := make(chan struct{})
	var wg sync.WaitGroup

	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for {
				select {
				case <-done:
					return
				default:
					mu.Lock()
					counter++
					mu.Unlock()
					runtime.Gosched() // 스케줄러에 양보
				}
			}
		}()
	}

	time.Sleep(testDuration)
	close(done)
	wg.Wait()
	atomic.StoreInt64(result, counter)
}

// 패턴 3: 락-프리 아토믹 (락 자체 제거)
func lockFreeApproach(result *int64) {
	var counter int64
	done := make(chan struct{})
	var wg sync.WaitGroup

	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for {
				select {
				case <-done:
					return
				default:
					atomic.AddInt64(&counter, 1)
				}
			}
		}()
	}

	time.Sleep(testDuration)
	close(done)
	wg.Wait()
	atomic.StoreInt64(result, counter)
}

// 패턴 4: 배치 처리로 락 획득 빈도 줄이기
// 작업을 모아서 한 번에 처리 → 락 획득 횟수 대폭 감소
func batchedLock(result *int64) {
	var mu sync.Mutex
	var counter int64
	done := make(chan struct{})
	var wg sync.WaitGroup

	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			localBatch := int64(0)
			const batchSize = 1000
			for {
				select {
				case <-done:
					// 남은 배치 커밋
					mu.Lock()
					counter += localBatch
					mu.Unlock()
					return
				default:
					localBatch++
					if localBatch >= batchSize {
						mu.Lock()
						counter += localBatch
						mu.Unlock()
						localBatch = 0
					}
				}
			}
		}()
	}

	time.Sleep(testDuration)
	close(done)
	wg.Wait()
	atomic.StoreInt64(result, counter)
}

func main() {
	var r1, r2, r3, r4 int64
	convoyProne(&r1)
	convoyMitigated(&r2)
	lockFreeApproach(&r3)
	batchedLock(&r4)

	fmt.Printf("콘보이 발생:         %12d ops\n", r1)
	fmt.Printf("Gosched 양보:        %12d ops\n", r2)
	fmt.Printf("락-프리 아토믹:      %12d ops\n", r3)
	fmt.Printf("배치 처리:           %12d ops\n", r4)
}
```

### 예제 2: Thundering Herd 방지 (C, Linux)

```c
#include <errno.h>
#include <netinet/in.h>
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <unistd.h>

#define PORT        8080
#define NUM_THREADS 4
#define MAX_EVENTS  32

/* ── 방법 1: EPOLLEXCLUSIVE (Linux 4.5+) ────────────────────────────────
 * 각 워커 스레드가 별도 epoll_fd를 생성하고,
 * EPOLLEXCLUSIVE 플래그로 같은 listen_fd를 등록.
 * 이벤트 발생 시 대기 중인 epoll 중 하나만 깨어남 (exclusive wakeup).
 */
static int listen_fd;

void *epollexclusive_worker(void *arg) {
    int worker_id = *(int *)arg;
    int epfd = epoll_create1(0);
    if (epfd < 0) { perror("epoll_create1"); return NULL; }

    struct epoll_event ev = {
        .events = EPOLLIN | EPOLLEXCLUSIVE,  /* ← 핵심 플래그 */
        .data.fd = listen_fd
    };
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev) < 0) {
        perror("epoll_ctl");
        close(epfd);
        return NULL;
    }

    struct epoll_event events[MAX_EVENTS];
    printf("[Worker %d] 준비 완료 (EPOLLEXCLUSIVE)\n", worker_id);

    while (1) {
        int n = epoll_wait(epfd, events, MAX_EVENTS, -1);
        for (int i = 0; i < n; i++) {
            int client_fd = accept4(listen_fd, NULL, NULL, SOCK_NONBLOCK);
            if (client_fd < 0) {
                if (errno == EAGAIN || errno == EWOULDBLOCK) continue;
                perror("accept4");
                continue;
            }
            /* 클라이언트 처리 */
            const char *resp = "HTTP/1.1 200 OK\r\nContent-Length: 2\r\n\r\nOK";
            write(client_fd, resp, strlen(resp));
            close(client_fd);
        }
    }
    close(epfd);
    return NULL;
}

/* ── 방법 2: SO_REUSEPORT (Linux 3.9+) ──────────────────────────────────
 * 각 워커 스레드가 같은 포트에 자신만의 소켓을 바인딩.
 * 커널의 소켓 레이어가 직접 로드밸런싱 → 한 번에 하나의 소켓에만 이벤트 발생.
 * EPOLLEXCLUSIVE보다 균등한 분배를 보장.
 */
int create_reuseport_socket(int port) {
    int fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);
    if (fd < 0) { perror("socket"); return -1; }

    int yes = 1;
    setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes));
    setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, &yes, sizeof(yes)); /* ← 핵심 */

    struct sockaddr_in addr = {
        .sin_family      = AF_INET,
        .sin_port        = htons(port),
        .sin_addr.s_addr = INADDR_ANY
    };
    if (bind(fd, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        perror("bind"); close(fd); return -1;
    }
    if (listen(fd, SOMAXCONN) < 0) {
        perror("listen"); close(fd); return -1;
    }
    return fd;
}

void *reuseport_worker(void *arg) {
    int worker_id = *(int *)arg;
    int my_fd = create_reuseport_socket(PORT);  /* 각 스레드가 독립 소켓 */
    if (my_fd < 0) return NULL;

    int epfd = epoll_create1(0);
    struct epoll_event ev = { .events = EPOLLIN, .data.fd = my_fd };
    epoll_ctl(epfd, EPOLL_CTL_ADD, my_fd, &ev);

    printf("[Worker %d] 준비 완료 (SO_REUSEPORT)\n", worker_id);
    struct epoll_event events[MAX_EVENTS];
    while (1) {
        int n = epoll_wait(epfd, events, MAX_EVENTS, -1);
        for (int i = 0; i < n; i++) {
            int client_fd = accept4(my_fd, NULL, NULL, SOCK_NONBLOCK);
            if (client_fd < 0) continue;
            const char *resp = "HTTP/1.1 200 OK\r\nContent-Length: 2\r\n\r\nOK";
            write(client_fd, resp, strlen(resp));
            close(client_fd);
        }
    }
    close(epfd);
    close(my_fd);
    return NULL;
}

/* ── 방법 3: accept_mutex 패턴 (Nginx 고전적 해법) ──────────────────────
 * 한 번에 하나의 프로세스/스레드만 accept를 대기하도록 뮤텍스로 제어.
 * 현재는 SO_REUSEPORT가 더 효율적이어서 Nginx도 기본값을 off로 변경.
 */
static pthread_mutex_t accept_mutex = PTHREAD_MUTEX_INITIALIZER;

void *accept_mutex_worker(void *arg) {
    int worker_id = *(int *)arg;
    printf("[Worker %d] 준비 완료 (accept_mutex)\n", worker_id);
    while (1) {
        pthread_mutex_lock(&accept_mutex);
        int client_fd = accept(listen_fd, NULL, NULL);
        pthread_mutex_unlock(&accept_mutex);
        if (client_fd < 0) continue;
        const char *resp = "HTTP/1.1 200 OK\r\nContent-Length: 2\r\n\r\nOK";
        write(client_fd, resp, strlen(resp));
        close(client_fd);
    }
    return NULL;
}

int main(void) {
    /* listen_fd 생성 */
    listen_fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);
    int yes = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes));
    struct sockaddr_in addr = {
        .sin_family = AF_INET, .sin_port = htons(PORT),
        .sin_addr.s_addr = INADDR_ANY
    };
    bind(listen_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(listen_fd, SOMAXCONN);

    pthread_t threads[NUM_THREADS];
    int ids[NUM_THREADS];
    for (int i = 0; i < NUM_THREADS; i++) {
        ids[i] = i;
        pthread_create(&threads[i], NULL, epollexclusive_worker, &ids[i]);
    }
    for (int i = 0; i < NUM_THREADS; i++)
        pthread_join(threads[i], NULL);
    return 0;
}
```

## 주의사항/팁

### 락 콘보이 감지와 해결

**감지**: `perf lock` 도구나 `mutrace`로 뮤텍스 경쟁 비율을 측정합니다. mutex wait time이 전체 CPU 시간의 10% 이상이면 콘보이 의심.

**해결책 비교**:

| 접근법 | 효과 | 적용 시나리오 |
|--------|------|--------------|
| `runtime.Gosched()` / `sched_yield()` | 중간 | 임시 완화, 다른 스레드에 양보 |
| 배치 처리 | 높음 | 카운터, 로그 등 집계 가능한 작업 |
| 락-프리 (CAS/아토믹) | 매우 높음 | 단순 카운터, 큐, 스택 |
| 쓰레드 로컬 버퍼 | 높음 | 집계 후 주기적 플러시 |
| 세분화된 락 (Sharding) | 높음 | 해시맵 등 분할 가능한 자료구조 |

### 공정 락(Fair Lock)과 콘보이

Java의 `ReentrantLock(fair=true)`는 FIFO 대기 순서를 보장하여 기아(starvation)를 막지만, 콘보이에는 더 취약합니다. 짧고 빈번한 임계 구역에는 불공정 락(unfair lock, 기본값)이 훨씬 나은 처리량을 보입니다.

```java
// 불공정 락 (기본값, 콘보이에 덜 취약)
ReentrantLock unfairLock = new ReentrantLock();      // fair=false

// 공정 락 (기아 방지, 콘보이에 취약)
ReentrantLock fairLock = new ReentrantLock(true);   // fair=true
```

### Cache Stampede (캐시 스탬피드)와의 비교

Thundering Herd의 변형으로 **Cache Stampede**(캐시 스탬피드)가 있습니다. 캐시 항목이 만료될 때 여러 요청이 동시에 DB로 몰리는 현상입니다.

```python
import threading
import time

_cache = {}
_lock = threading.Lock()

# 나쁜 패턴: Double-Checked Locking 없는 캐시 → Stampede 발생
def get_user_bad(user_id: int):
    if user_id not in _cache:
        # 여러 스레드가 동시에 여기 도달 → DB 중복 조회
        data = db_query(user_id)
        _cache[user_id] = data
    return _cache[user_id]

# 좋은 패턴: 키별 락 + 더블 체크
_per_key_locks: dict[int, threading.Lock] = {}
_per_key_meta = threading.Lock()

def get_user_good(user_id: int):
    if user_id in _cache:
        return _cache[user_id]

    with _per_key_meta:
        if user_id not in _per_key_locks:
            _per_key_locks[user_id] = threading.Lock()
        key_lock = _per_key_locks[user_id]

    with key_lock:
        if user_id not in _cache:  # 더블 체크
            _cache[user_id] = db_query(user_id)
    return _cache[user_id]
```

### Thundering Herd 플랫폼별 해결책

| 플랫폼 | 해결책 |
|--------|--------|
| Linux 4.5+ | `EPOLLEXCLUSIVE` 플래그 |
| Linux 3.9+ | `SO_REUSEPORT` (커널 로드밸런싱) |
| Windows | I/O Completion Ports (IOCP) — 기본적으로 1개만 wakeup |
| Node.js (cluster) | 마스터-워커 모델, `cluster.schedulingPolicy` |
| Nginx | `accept_mutex off` + `SO_REUSEPORT` |

### 스핀락과 콘보이

스핀락은 블록되지 않으므로 컨텍스트 스위치가 없어 콘보이에 덜 취약합니다. 하지만 대기 중 CPU를 계속 소비하므로, 임계 구역이 짧고 경쟁이 낮을 때만 뮤텍스보다 유리합니다. Linux 커널은 `spinlock_t`를 짧은 임계 구역에 사용하고, 긴 대기가 예상되면 `mutex`를 사용합니다.

## 참고 자료

- [Wikipedia: Lock convoy](https://en.wikipedia.org/wiki/Lock_convoy)
- [Wikipedia: Thundering herd problem](https://en.wikipedia.org/wiki/Thundering_herd_problem)
- [OpenGenus: Lock Convoy Problem](https://iq.opengenus.org/lock-convoy-problem)
- [Concurrency notes gist - dpaez](https://gist.github.com/dpaez/5d92ba6b9f82f3efe1ab3d9ced86a6ab)
