---
layout: post
title: "io_uring 완전 정복: 리눅스 비동기 I/O의 혁명과 제로 카피 시스템콜"
date: 2026-06-23
categories: [cs, computer-science]
tags: [io_uring, linux, async-io, kernel, performance, systems-programming, epoll]
---

## 개념 설명: io_uring이란 무엇인가

리눅스 커널 5.1(2019년)에 도입된 `io_uring`은 Jens Axboe가 설계한 비동기 I/O 인터페이스로, 기존 비동기 I/O의 모든 단점을 근본적으로 해결한 혁신적인 메커니즘이다. 이름 그대로 **I/O 작업을 위한 두 개의 링(ring) 자료구조**를 핵심으로 사용한다.

기존 비동기 I/O 방식에는 크게 세 가지 계열이 있었다.

- **POSIX AIO**: POSIX 표준이지만 실제 커널 레벨 비동기가 아닌 스레드 풀로 구현되어 컨텍스트 스위칭 비용이 크다.
- **Linux AIO (libaio)**: 커널 레벨 비동기지만 `O_DIRECT` 플래그가 있는 파일과 특정 소켓에만 동작하는 심각한 제약이 있다.
- **epoll + non-blocking I/O**: 실제로 읽기/쓰기가 준비된 시점을 알려주지만, `read()`/`write()` 호출 자체는 여전히 시스템콜을 발생시킨다.

`io_uring`은 이 모든 문제를 **유저 공간과 커널 공간이 공유하는 링 버퍼**를 통해 해결한다. I/O 요청 제출과 완료 알림 모두 메모리 공유로 이루어지므로, 이상적인 경우 시스템콜을 **0번** 호출하고도 수천 개의 I/O 작업을 처리할 수 있다.

### 두 개의 링: SQ와 CQ

io_uring의 핵심 구조는 두 개의 링 버퍼다.

```
유저 공간                 커널 공간
┌─────────────────┐       ┌─────────────────┐
│  Submission     │ mmap  │                 │
│  Queue (SQ)     │──────▶│   커널 I/O      │
│  [요청 추가]    │       │   처리 엔진     │
└─────────────────┘       │                 │
                          │                 │
┌─────────────────┐ mmap  │                 │
│  Completion     │◀──────│                 │
│  Queue (CQ)     │       └─────────────────┘
│  [결과 수신]    │
└─────────────────┘
```

**Submission Queue (SQ)**: 유저가 수행하고 싶은 I/O 작업을 기술한 `sqe`(submission queue entry) 구조체를 추가하는 링이다. `io_uring_enter()` 시스템콜 한 번으로 수백 개의 작업을 배치(batch) 제출할 수 있다.

**Completion Queue (CQ)**: 커널이 I/O 완료 결과를 `cqe`(completion queue entry) 형태로 추가하는 링이다. 유저는 별도의 시스템콜 없이 CQ를 폴링하거나 `io_uring_wait_cqe()`로 블로킹 대기하면 된다.

---

## 왜 io_uring이 필요한가

### 1. 시스템콜 오버헤드 제거

현대 서버는 초당 수십만 건의 I/O를 처리해야 한다. `read()` 하나를 호출할 때마다 발생하는 비용을 살펴보면:

1. 유저 → 커널 모드 전환 (컨텍스트 스위치)
2. 커널에서 인자 복사 (유저 공간 메모리 검증 포함)
3. 커널 → 유저 모드 복귀

이 과정이 수십 나노초에서 수백 나노초를 소비한다. NVMe SSD의 레이턴시가 10~100μs, 네트워크 RTT가 수십μs인 상황에서 시스템콜 오버헤드는 무시할 수 없는 병목이다.

io_uring의 **SQPOLL 모드**를 사용하면 커널이 별도의 커널 스레드를 띄워 SQ를 폴링하므로, 애플리케이션이 시스템콜을 단 한 번도 호출하지 않고 I/O를 제출할 수 있다.

### 2. 프로그래밍 모델의 단순화

epoll 기반 비동기 코드는 "준비 상태 알림 → 실제 I/O → 콜백 처리"의 복잡한 상태 머신을 요구한다. io_uring은 "요청 제출 → 완료 수신"의 단순한 완료 기반(completion-based) 모델을 제공하여 코드 구조를 획기적으로 단순화한다.

### 3. 지원 연산의 확장

기존 Linux AIO가 파일 I/O에만 국한됐던 것과 달리, io_uring은 다음을 모두 지원한다:
- 파일 읽기/쓰기 (`read`, `write`, `readv`, `writev`)
- 네트워크 소켓 (`recv`, `send`, `accept`, `connect`)
- 타이머 (`timeout`, `link_timeout`)
- 파일 시스템 연산 (`fsync`, `fallocate`, `statx`)
- 고정 버퍼 (`IORING_OP_READ_FIXED`) 를 통한 제로 카피

---

## 실제 구현 예제

### 예제 1: liburing으로 파일 비동기 읽기 (C)

`liburing`은 io_uring을 쉽게 사용하기 위한 헬퍼 라이브러리다. 먼저 설치가 필요하다:

```bash
# Ubuntu/Debian
sudo apt install liburing-dev
```

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <string.h>
#include <liburing.h>

#define QUEUE_DEPTH 1
#define BLOCK_SIZE  4096

int main(int argc, char *argv[]) {
    struct io_uring ring;
    struct io_uring_sqe *sqe;
    struct io_uring_cqe *cqe;
    struct iovec iov;
    char *buf;
    int fd, ret;

    if (argc < 2) {
        fprintf(stderr, "Usage: %s <filename>\n", argv[0]);
        return 1;
    }

    // 1. io_uring 인스턴스 초기화 (SQ/CQ 크기 = QUEUE_DEPTH)
    ret = io_uring_queue_init(QUEUE_DEPTH, &ring, 0);
    if (ret < 0) {
        fprintf(stderr, "io_uring_queue_init: %s\n", strerror(-ret));
        return 1;
    }

    // 2. 파일 열기
    fd = open(argv[1], O_RDONLY);
    if (fd < 0) {
        perror("open");
        io_uring_queue_exit(&ring);
        return 1;
    }

    // 3. 읽기 버퍼 준비
    buf = malloc(BLOCK_SIZE);
    iov.iov_base = buf;
    iov.iov_len  = BLOCK_SIZE;

    // 4. SQE(제출 항목) 가져오기 및 작업 기술
    sqe = io_uring_get_sqe(&ring);
    io_uring_prep_readv(sqe, fd, &iov, 1, 0);  // offset=0 부터 readv
    io_uring_sqe_set_data(sqe, buf);             // 사용자 데이터 태깅

    // 5. 제출 — 이 한 번의 syscall로 배치 제출 가능
    ret = io_uring_submit(&ring);
    if (ret < 0) {
        fprintf(stderr, "io_uring_submit: %s\n", strerror(-ret));
        goto cleanup;
    }

    // 6. 완료 대기 (CQE 수신)
    ret = io_uring_wait_cqe(&ring, &cqe);
    if (ret < 0) {
        fprintf(stderr, "io_uring_wait_cqe: %s\n", strerror(-ret));
        goto cleanup;
    }

    if (cqe->res < 0) {
        fprintf(stderr, "Async readv failed: %s\n", strerror(-cqe->res));
    } else {
        printf("Read %d bytes:\n%.*s\n", cqe->res, cqe->res, buf);
    }

    // 7. CQE 소비 완료 표시 (CQ 링 헤드 전진)
    io_uring_cqe_seen(&ring, cqe);

cleanup:
    free(buf);
    close(fd);
    io_uring_queue_exit(&ring);
    return ret < 0 ? 1 : 0;
}
```

컴파일 및 실행:
```bash
gcc -o uring_read uring_read.c -luring
./uring_read /etc/hostname
```

### 예제 2: 다중 I/O 배치 처리로 성능 극대화 (C)

io_uring의 진정한 강점은 **단일 시스템콜로 수십~수백 개의 I/O 작업을 배치 제출**하는 것이다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <string.h>
#include <liburing.h>

#define QUEUE_DEPTH  64
#define BLOCK_SIZE   4096
#define NUM_FILES     8

typedef struct {
    int    fd;
    char  *buf;
} FileCtx;

int main(void) {
    struct io_uring     ring;
    struct io_uring_sqe *sqe;
    struct io_uring_cqe *cqe;
    FileCtx              files[NUM_FILES];
    struct iovec         iovs[NUM_FILES];
    int                  ret, i, completed = 0;

    // 테스트용 임시 파일 생성
    for (i = 0; i < NUM_FILES; i++) {
        char path[32];
        snprintf(path, sizeof(path), "/tmp/uring_test_%d", i);
        FILE *f = fopen(path, "w");
        fprintf(f, "Hello from file %d! io_uring batch demo.", i);
        fclose(f);

        files[i].fd  = open(path, O_RDONLY);
        files[i].buf = calloc(1, BLOCK_SIZE);
        iovs[i].iov_base = files[i].buf;
        iovs[i].iov_len  = BLOCK_SIZE;
    }

    // io_uring 초기화
    io_uring_queue_init(QUEUE_DEPTH, &ring, 0);

    // 모든 파일에 대한 readv를 한꺼번에 SQ에 추가
    for (i = 0; i < NUM_FILES; i++) {
        sqe = io_uring_get_sqe(&ring);
        io_uring_prep_readv(sqe, files[i].fd, &iovs[i], 1, 0);
        // user_data로 인덱스를 태깅해 완료 시 파악
        io_uring_sqe_set_data64(sqe, (uint64_t)i);
    }

    // 단 한 번의 syscall로 NUM_FILES개 I/O 제출!
    ret = io_uring_submit(&ring);
    printf("Submitted %d I/O operations with 1 syscall\n", ret);

    // 완료 처리
    while (completed < NUM_FILES) {
        ret = io_uring_wait_cqe(&ring, &cqe);
        if (ret == 0 && cqe->res > 0) {
            int idx = (int)io_uring_cqe_get_data64(cqe);
            printf("[File %d] Read %d bytes: %s\n",
                   idx, cqe->res, files[idx].buf);
        }
        io_uring_cqe_seen(&ring, cqe);
        completed++;
    }

    // 정리
    for (i = 0; i < NUM_FILES; i++) {
        close(files[i].fd);
        free(files[i].buf);
        char path[32];
        snprintf(path, sizeof(path), "/tmp/uring_test_%d", i);
        remove(path);
    }
    io_uring_queue_exit(&ring);
    return 0;
}
```

출력 예시:
```
Submitted 8 I/O operations with 1 syscall
[File 3] Read 42 bytes: Hello from file 3! io_uring batch demo.
[File 0] Read 42 bytes: Hello from file 0! io_uring batch demo.
...
```

완료 순서는 I/O 완료 순서에 따라 달라질 수 있다. `io_uring_sqe_set_data64()`로 태깅한 인덱스로 어떤 파일의 결과인지 구분한다.

---

## 고급 기능: 고정 버퍼와 SQPOLL

### 고정 버퍼 (Fixed Buffers)

매번 I/O마다 유저 버퍼의 유효성 검사를 커널이 수행하는 비용을 제거하기 위해 버퍼를 미리 등록할 수 있다.

```c
// 버퍼를 커널에 미리 등록
struct iovec fixed_iovs[1];
fixed_iovs[0].iov_base = buf;
fixed_iovs[0].iov_len  = BLOCK_SIZE;
io_uring_register_buffers(&ring, fixed_iovs, 1);

// 이후 io_uring_prep_read_fixed() 사용 — 버퍼 검증 비용 0
sqe = io_uring_get_sqe(&ring);
io_uring_prep_read_fixed(sqe, fd, buf, BLOCK_SIZE, 0, 0);
```

### SQPOLL 모드

`IORING_SETUP_SQPOLL` 플래그로 초기화하면 커널 스레드가 SQ를 지속적으로 폴링하여 `io_uring_submit()` 호출 없이도 제출이 이루어진다. 극한의 레이턴시가 필요한 고성능 서버에서 사용된다.

---

## 주의사항 및 팁

### 1. 커널 버전 확인

io_uring의 기능은 커널 버전별로 크게 다르다:
- **5.1**: 기본 파일/소켓 I/O
- **5.6**: `send`, `recv`, `accept`, `connect` 지원
- **5.11**: 고정 파일 + 버퍼 링 (완전한 제로 카피)
- **6.0+**: io_uring 기반 네트워크 스택 성능 대폭 향상

프로덕션 환경에서는 `uname -r`로 커널 버전을 반드시 확인하자.

### 2. 보안 고려사항

io_uring은 강력한 만큼 보안 취약점의 대상이 되기도 했다. 2022년 구글 안드로이드 팀은 보안 위험을 이유로 Android에서 io_uring 사용을 비활성화했다. 컨테이너 환경(Docker, Kubernetes)에서는 seccomp 프로파일이 `io_uring_enter`/`io_uring_setup` syscall을 차단할 수 있다.

```bash
# 컨테이너에서 io_uring 허용 여부 확인
grep -i io_uring /proc/self/status 2>/dev/null || echo "io_uring status check"
```

### 3. 에러 처리 패턴

CQE의 `res` 필드가 음수이면 errno 값을 나타낸다. `-EAGAIN`은 재시도가 필요한 경우, `-ECANCELED`는 작업이 취소된 경우다. 모든 CQE를 `io_uring_cqe_seen()`으로 소비하지 않으면 CQ 링이 꽉 차서 이후 제출이 실패할 수 있다.

### 4. liburing vs 직접 syscall

프로덕션 코드에서는 반드시 `liburing`을 사용하자. 직접 `io_uring_setup(2)`, `io_uring_enter(2)` 시스템콜을 호출하는 것은 내부 동작 이해를 위한 학습 목적에만 적합하다.

### 5. Tokio, Glommio — 언어 생태계의 io_uring

Rust 비동기 런타임 생태계에서 io_uring은 적극적으로 활용된다:
- **Tokio-uring**: Tokio의 io_uring 백엔드
- **Glommio**: DataDog이 공개한 io_uring 네이티브 비동기 런타임
- **Monoio**: 바이트댄스의 thread-per-core io_uring 런타임

Java 진영에서는 Netty 5가 io_uring 지원을 추가했다.

---

## 정리

`io_uring`은 단순한 성능 개선을 넘어 리눅스 I/O 프로그래밍 패러다임의 전환점이다. 유저-커널 공유 링 버퍼, 배치 제출, 제로 카피 고정 버퍼, SQPOLL 모드를 통해 수만 IOPS의 차세대 스토리지와 네트워크를 효율적으로 다룰 수 있다. 아직 생태계가 성숙 중이지만, 고성능 서버 프로그래밍을 목표로 한다면 반드시 익혀야 할 기술이다.

## 참고 자료
- [Efficient I/O with io_uring — Jens Axboe (kernel.dk)](https://kernel.dk/io_uring.pdf)
- [io_uring(7) Linux Manual Page — man7.org](https://man7.org/linux/man-pages/man7/io_uring.7.html)
- [Lord of the io_uring — 완전 튜토리얼](https://unixism.net/loti/what_is_io_uring.html)
- [Why you should use io_uring for network I/O — Red Hat Developer](https://developers.redhat.com/articles/2023/04/12/why-you-should-use-iouring-network-io)
