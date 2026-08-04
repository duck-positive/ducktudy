---
layout: post
title: "Linux 시그널(Signal) 처리 완전 정복: POSIX 비동기 이벤트 메커니즘과 Graceful Shutdown 구현"
date: 2026-08-04
categories: [cs, computer-science]
tags: [linux, signal, posix, sigaction, os, system-programming, c]
---

시그널(Signal)은 Unix/Linux 계열 운영체제에서 프로세스 간, 혹은 커널과 프로세스 사이에 비동기적으로 이벤트를 전달하는 소프트웨어 인터럽트 메커니즘이다. Ctrl+C를 눌렀을 때 프로세스가 종료되는 것, `kill` 명령어로 프로세스를 종료하는 것, 자식 프로세스가 죽었을 때 부모가 이를 알아채는 것 — 이 모든 것이 시그널을 통해 이루어진다. 시그널은 운영체제 시스템 프로그래밍에서 가장 섬세하게 다뤄야 하는 메커니즘 중 하나다.

## 시그널이란 무엇인가

시그널은 정수값으로 식별되며, 각각 특정 이벤트를 나타낸다. POSIX 표준은 수십 가지 시그널을 정의한다. 주요 시그널 목록은 다음과 같다.

| 번호 | 이름     | 기본 동작 | 설명 |
|------|----------|-----------|------|
| 1    | SIGHUP   | 종료      | 터미널 연결 종료, 데몬 재로드에 활용 |
| 2    | SIGINT   | 종료      | Ctrl+C, 인터럽트 요청 |
| 3    | SIGQUIT  | 코어 덤프 | Ctrl+\, 종료 + 코어 덤프 |
| 9    | SIGKILL  | 종료      | 강제 종료 (핸들러 등록 불가) |
| 11   | SIGSEGV  | 코어 덤프 | 잘못된 메모리 접근 |
| 13   | SIGPIPE  | 종료      | 읽는 쪽이 닫힌 파이프에 쓰기 |
| 14   | SIGALRM  | 종료      | 타이머 만료 |
| 15   | SIGTERM  | 종료      | 정상 종료 요청 (kill의 기본값) |
| 17   | SIGCHLD  | 무시      | 자식 프로세스 상태 변경 |
| 10   | SIGUSR1  | 종료      | 사용자 정의 시그널 1 |
| 12   | SIGUSR2  | 종료      | 사용자 정의 시그널 2 |

시그널은 세 가지 방식으로 처리할 수 있다. 첫째, 기본 동작(default action)을 그대로 수행한다. 둘째, 시그널을 무시(ignore)한다. 셋째, 사용자 정의 핸들러(handler)를 등록해 원하는 코드를 실행한다. SIGKILL과 SIGSTOP은 핸들러 등록이나 무시가 불가능하며, 커널이 강제로 처리한다.

## 왜 시그널 처리가 중요한가

### Graceful Shutdown

운영 환경의 서버 프로세스는 SIGTERM을 받아도 즉시 종료해서는 안 된다. 진행 중인 요청이 있다면 완료 후 종료해야 하고, 데이터베이스 연결을 정상적으로 닫아야 하며, 임시 파일을 정리해야 한다. 이를 Graceful Shutdown이라 한다.

### 데몬 구성 파일 재로드

nginx, apache 같은 데몬은 SIGHUP을 받으면 재시작 없이 설정 파일을 다시 읽는다. `nginx -s reload`가 내부적으로 `kill -HUP $(cat nginx.pid)`를 실행하는 이유다.

### 좀비 프로세스 방지

자식 프로세스가 종료되면 부모가 `wait()` 계열 함수로 종료 상태를 수거하기 전까지 좀비(zombie) 상태로 남는다. 좀비는 PID 테이블을 점유하므로 오래 지속되면 PID 고갈이 발생한다. SIGCHLD 핸들러에서 `waitpid()`를 호출해 이를 방지한다.

### 타임아웃 구현

`alarm()` 시스템콜과 SIGALRM을 조합하면 특정 연산의 최대 수행 시간을 제한할 수 있다.

## signal() vs sigaction()

구식 `signal()` 함수는 플랫폼마다 동작이 달라 사용을 피해야 한다. 예를 들어, 일부 구현에서는 핸들러가 한 번 실행된 후 기본 동작으로 리셋되어 경쟁 조건(race condition)이 발생한다. `sigaction()`은 POSIX 표준이며 이러한 문제를 해결한 더 강력한 인터페이스다.

## 코드 예제 1: sigaction으로 Graceful Shutdown 구현

```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <string.h>
#include <unistd.h>

/* volatile sig_atomic_t: 시그널 핸들러에서 안전하게 읽고 쓸 수 있는 유일한 타입 */
static volatile sig_atomic_t g_shutdown = 0;

static void handle_shutdown(int signo) {
    g_shutdown = 1;
    /* async-signal-safe: write()는 안전하지만 printf()는 위험 */
    const char msg[] = "시그널 수신, shutdown 플래그 설정\n";
    write(STDERR_FILENO, msg, sizeof(msg) - 1);
}

int main(void) {
    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.sa_handler = handle_shutdown;
    sigemptyset(&sa.sa_mask);

    /* SIGTERM 처리 중 SIGINT도 블록 (동시 수신 방지) */
    sigaddset(&sa.sa_mask, SIGINT);

    /*
     * SA_RESTART: read/write/accept 같은 느린 시스템콜이
     * 시그널에 의해 중단된 후 자동 재시작
     */
    sa.sa_flags = SA_RESTART;

    if (sigaction(SIGTERM, &sa, NULL) == -1) {
        perror("sigaction SIGTERM");
        return 1;
    }

    /* SIGINT(Ctrl+C)도 같은 핸들러로 처리 */
    sigemptyset(&sa.sa_mask);
    sigaddset(&sa.sa_mask, SIGTERM);
    if (sigaction(SIGINT, &sa, NULL) == -1) {
        perror("sigaction SIGINT");
        return 1;
    }

    /* SIGPIPE 무시: 소켓/파이프 오류를 errno로 처리 */
    signal(SIGPIPE, SIG_IGN);

    printf("PID %d: 서버 시작. 'kill %d' 또는 Ctrl+C로 종료\n",
           getpid(), getpid());

    int request_count = 0;
    while (!g_shutdown) {
        /* 실제 요청 처리 로직 시뮬레이션 */
        sleep(1);
        printf("요청 처리 중... (%d번째)\n", ++request_count);
    }

    /* Graceful Shutdown: 진행 중인 요청 완료 대기 로직 */
    printf("Graceful Shutdown 시작...\n");
    /* 실제에서는 여기서 active connection 대기, DB 연결 종료 등 */
    printf("서버 정상 종료 완료 (처리 요청: %d건)\n", request_count);
    return 0;
}
```

이 예제의 핵심 포인트:
- `volatile sig_atomic_t`는 원자적 읽기/쓰기가 보장되는 타입으로, 시그널 핸들러와 메인 스레드 사이의 플래그 공유에 사용한다.
- `SA_RESTART`는 네트워크 서버에서 필수다. 이 플래그 없이는 `accept()`가 EINTR 에러로 실패할 수 있다.
- 핸들러 내에서 `printf()`는 unsafe하다. `write()`를 직접 사용해야 한다.

## 코드 예제 2: 자식 프로세스 좀비 방지 + 시그널 마스킹

```c
#include <stdio.h>
#include <signal.h>
#include <sys/wait.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>

/*
 * SIGCHLD 핸들러: errno를 저장/복원하는 것이 중요하다.
 * waitpid() 호출이 errno를 변경할 수 있기 때문이다.
 */
static void handle_sigchld(int signo) {
    int saved_errno = errno;
    pid_t pid;
    int status;

    /*
     * WNOHANG: 종료된 자식이 없으면 즉시 반환.
     * 루프: 여러 자식이 동시에 종료됐을 때 모두 처리.
     */
    while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
        if (WIFEXITED(status)) {
            char buf[64];
            int len = snprintf(buf, sizeof(buf),
                "[핸들러] 자식 PID %d 정상 종료 (exit: %d)\n",
                (int)pid, WEXITSTATUS(status));
            write(STDOUT_FILENO, buf, len);
        }
    }
    errno = saved_errno;
}

int main(void) {
    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.sa_handler = handle_sigchld;
    sigemptyset(&sa.sa_mask);

    /*
     * SA_NOCLDSTOP: 자식이 SIGSTOP으로 중지될 때는 SIGCHLD 발생 안 함.
     * 종료될 때만 SIGCHLD 발생.
     */
    sa.sa_flags = SA_RESTART | SA_NOCLDSTOP;
    sigaction(SIGCHLD, &sa, NULL);

    printf("부모 PID: %d\n", getpid());

    for (int i = 1; i <= 3; i++) {
        pid_t pid = fork();
        if (pid == 0) {
            /* 자식 프로세스 */
            printf("자식 %d (PID %d) 시작\n", i, getpid());
            sleep(i); /* 각 자식이 다른 시점에 종료 */
            printf("자식 %d (PID %d) 작업 완료\n", i, getpid());
            _exit(i * 10); /* _exit 사용: stdio 버퍼 플러시 없이 종료 */
        } else if (pid < 0) {
            perror("fork");
        }
    }

    /* 부모는 계속 실행: SIGCHLD 핸들러가 자동으로 자식을 reap */
    for (int i = 0; i < 5; i++) {
        printf("부모: 메인 작업 %d\n", i + 1);
        sleep(1);
    }

    printf("부모: 모든 자식 처리 완료, 좀비 없음\n");
    return 0;
}
```

## 시그널 마스킹(Signal Masking)과 임계 구역 보호

특정 코드 영역에서 시그널을 일시적으로 블록해야 할 때 `sigprocmask()`를 사용한다.

```c
sigset_t mask, oldmask;
sigemptyset(&mask);
sigaddset(&mask, SIGTERM);
sigaddset(&mask, SIGINT);

/* 임계 구역 진입 전 시그널 블록 */
sigprocmask(SIG_BLOCK, &mask, &oldmask);

/* --- 임계 구역 시작 --- */
/* 데이터 구조 일관성이 필요한 코드 */
/* --- 임계 구역 끝 --- */

/* 이전 마스크로 복원: 블록된 시그널이 있으면 이 시점에 전달 */
sigprocmask(SIG_SETMASK, &oldmask, NULL);
```

## self-pipe trick과 signalfd

이벤트 루프(`select`, `poll`, `epoll`)와 시그널을 함께 사용할 때는 **self-pipe 패턴**이 필수다. 시그널 핸들러에서 파이프에 1바이트를 쓰고, 이벤트 루프는 파이프의 읽기 fd를 모니터링한다. 이렇게 하면 `select()`가 블록된 상태에서도 시그널을 안전하게 감지할 수 있다.

Linux에서는 이를 위한 전용 시스템콜 `signalfd()`가 제공된다. 시그널을 파일 디스크립터로 읽을 수 있어 이벤트 루프에 자연스럽게 통합된다.

```c
#include <sys/signalfd.h>

sigset_t mask;
sigemptyset(&mask);
sigaddset(&mask, SIGTERM);
sigaddset(&mask, SIGINT);

/* 이 프로세스에서 해당 시그널의 기본 처리 블록 */
sigprocmask(SIG_BLOCK, &mask, NULL);

/* 시그널을 fd로 읽기 */
int sfd = signalfd(-1, &mask, SFD_NONBLOCK | SFD_CLOEXEC);

/* epoll에 sfd 추가 후, sfd가 readable할 때 */
struct signalfd_siginfo fdsi;
read(sfd, &fdsi, sizeof(fdsi));
printf("시그널 수신: %u\n", fdsi.ssi_signo);
```

## 멀티스레드 환경에서의 시그널 처리

멀티스레드 프로그램에서 시그널은 프로세스 전체에 전달되지만, 어느 스레드가 처리할지는 불확실하다. 권장 패턴은 다음과 같다.

1. 모든 작업 스레드에서 `pthread_sigmask()`로 관심 시그널을 블록한다.
2. 전용 시그널 처리 스레드를 하나 만들어 `sigwait()` 또는 `sigwaitinfo()`로 동기적으로 처리한다.

```c
void* signal_thread(void* arg) {
    sigset_t mask = *(sigset_t*)arg;
    int signo;
    while (1) {
        sigwait(&mask, &signo); /* 시그널을 동기적으로 수신 */
        if (signo == SIGTERM || signo == SIGINT) {
            printf("종료 시그널 수신\n");
            /* 안전하게 shutdown 처리 */
            break;
        }
    }
    return NULL;
}
```

## 주의사항 및 실전 팁

**async-signal-safe 함수만 핸들러에서 호출하라.** `printf()`, `malloc()`, `free()`, `syslog()` 같은 함수는 내부적으로 잠금(lock)을 사용하므로, 핸들러에서 호출하면 메인 스레드가 같은 잠금을 보유 중일 때 데드락이 발생한다. `write()`, `_exit()`, `getpid()`, `sigprocmask()`, `waitpid()` 등 POSIX가 명시한 async-signal-safe 함수만 사용해야 한다.

**핸들러에서 errno를 보존하라.** 핸들러 진입 시 `int saved = errno;`로 저장하고 종료 전 `errno = saved;`로 복원한다. 핸들러 내에서 시스템콜이 실패하면 errno가 변경되어 메인 코드의 오류 처리가 오염된다.

**SIGKILL/SIGSTOP은 절대 블록하거나 무시할 수 없다.** `sigprocmask()`로 이 시그널을 블록 목록에 추가해도 커널이 조용히 무시한다.

**SIGPIPE는 기본적으로 무시하라.** 네트워크 서버에서 클라이언트가 연결을 끊은 상태에서 `write()`하면 SIGPIPE가 발생하고 프로세스가 종료된다. `signal(SIGPIPE, SIG_IGN)`으로 무시하고 `write()`의 EPIPE 에러를 처리하는 것이 일반적이다.

**`kill -l`로 전체 시그널 목록을 확인하라.** 실시간 시그널(SIGRTMIN~SIGRTMAX)은 큐에 쌓이므로 여러 번 발생해도 누락되지 않는다. 일반 시그널은 큐잉되지 않아 핸들러 실행 중 같은 시그널이 도착하면 무시될 수 있다.

## 참고 자료
- [signal(7) - Linux Manual Page](https://man7.org/linux/man-pages/man7/signal.7.html)
- [sigaction(2) - Linux Manual Page](https://man7.org/linux/man-pages/man2/sigaction.2.html)
- [signal-safety(7) - Linux Manual Page](https://man7.org/linux/man-pages/man7/signal-safety.7.html)
- [signalfd(2) - Linux Manual Page](https://man7.org/linux/man-pages/man2/signalfd.2.html)
