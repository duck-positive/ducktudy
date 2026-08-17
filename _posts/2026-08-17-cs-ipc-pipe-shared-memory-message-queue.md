---
layout: post
title: "IPC 완전 정복: 파이프·공유 메모리·메시지 큐·소켓으로 이해하는 프로세스 간 통신"
date: 2026-08-17
categories: [cs, computer-science]
tags: [ipc, pipe, shared-memory, message-queue, unix-socket, linux, os, systems-programming]
---

운영체제에서 각 프로세스는 독립된 가상 주소 공간을 가진다. 이는 메모리 보호를 제공하는 대신, 프로세스끼리 직접 메모리를 공유할 수 없다는 제약을 만든다. **IPC(Inter-Process Communication)**는 이 장벽을 넘어 프로세스들이 데이터를 교환하고 동기화하는 메커니즘의 총칭이다. 웹 서버가 백엔드 프로세스와 통신하거나, 쉘이 파이프로 명령어를 연결하거나, 데이터베이스가 캐시 프로세스와 협력하는 모든 상황에 IPC가 존재한다.

리눅스는 Unix 전통에서 내려온 System V IPC와 현대적인 POSIX IPC 두 계열을 지원한다. 각각의 메커니즘은 성능, 사용 편의성, 신뢰성에서 다른 트레이드오프를 가진다.

---

## 왜 IPC가 필요한가

단일 프로세스로 해결하면 되지 않냐는 물음이 생긴다. 그러나 현대 소프트웨어는 세 가지 이유로 멀티프로세스 구조를 택한다.

**격리(Isolation)**: 웹 브라우저가 각 탭을 별도 프로세스로 실행하는 이유가 여기 있다. 한 탭이 죽어도 다른 탭은 멀쩡하다. 공유 메모리 없이 프로세스 경계만으로 버그 전파를 막는다.

**권한 분리(Privilege Separation)**: sshd는 루트 권한으로 소켓을 열고, 실제 세션은 일반 권한 프로세스에 넘긴다. 보안 버그가 루트로 이어지는 공격 경로를 차단한다.

**언어·런타임 혼용**: Python 서버가 C++ 이미지 처리 모듈을 별도 프로세스로 띄우고 소켓으로 통신한다. 각 언어의 강점을 활용하면서도 협력한다.

---

## 메커니즘 1: 파이프(Pipe)

파이프는 가장 오래된 IPC다. `ls | grep .md` 처럼 쉘에서 매일 쓰는 바로 그것이다.

### 익명 파이프(Anonymous Pipe)

`pipe()` 시스템 콜은 두 개의 파일 디스크립터를 반환한다. `fd[0]`은 읽기 전용, `fd[1]`은 쓰기 전용이다. 데이터는 **단방향**으로만 흐른다. 부모-자식 프로세스처럼 fork를 통해 관계를 맺은 프로세스 사이에서만 사용할 수 있다.

```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>

int main(void) {
    int fd[2];
    pid_t pid;
    char buf[256];

    if (pipe(fd) == -1) { perror("pipe"); return 1; }

    pid = fork();
    if (pid == 0) {
        /* 자식: 쓰기 전용 fd 사용 */
        close(fd[0]);
        const char *msg = "안녕하세요, 부모 프로세스!";
        write(fd[1], msg, strlen(msg) + 1);
        close(fd[1]);
        return 0;
    } else {
        /* 부모: 읽기 전용 fd 사용 */
        close(fd[1]);
        ssize_t n = read(fd[0], buf, sizeof(buf));
        printf("부모가 받음: %s (bytes: %zd)\n", buf, n);
        close(fd[0]);
        wait(NULL);
    }
    return 0;
}
```

커널은 파이프를 링 버퍼(ring buffer)로 구현한다. 리눅스에서 기본 크기는 65536 바이트(64KB)이며, `fcntl(F_SETPIPE_SZ)`로 변경 가능하다. 버퍼가 가득 차면 `write()`는 블로킹된다. 읽는 쪽이 없으면 `SIGPIPE` 신호를 받는다.

### 이름 있는 파이프(Named Pipe, FIFO)

관계 없는 프로세스 간에는 **FIFO**를 쓴다. 파일 시스템에 이름을 갖는 특수 파일로 존재하므로, 프로세스 관계 없이 경로만 알면 연결할 수 있다.

```bash
# 터미널 1
mkfifo /tmp/myfifo
cat /tmp/myfifo          # 블로킹 대기

# 터미널 2
echo "FIFO 테스트 메시지" > /tmp/myfifo
```

FIFO는 양방향 통신이 불가능하다. 양방향이 필요하면 FIFO 두 개를 사용하거나 소켓으로 전환한다.

---

## 메커니즘 2: 공유 메모리(Shared Memory)

공유 메모리는 IPC 중 **가장 빠르다**. 데이터를 복사할 필요 없이 두 프로세스가 동일한 물리 메모리 페이지를 각자의 가상 주소 공간에 매핑한다. 커널을 경유하는 복사 오버헤드가 없으므로, 대용량 데이터를 주고받을 때 파이프보다 훨씬 빠르다.

POSIX API는 `shm_open()` + `mmap()` 조합을 쓴다.

```c
/* 생산자(producer.c) */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <unistd.h>

#define SHM_NAME "/my_shm"
#define SHM_SIZE 4096

int main(void) {
    /* 공유 메모리 객체 생성 */
    int fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);
    if (fd == -1) { perror("shm_open"); exit(1); }

    ftruncate(fd, SHM_SIZE);

    /* 프로세스 주소 공간에 매핑 */
    void *ptr = mmap(NULL, SHM_SIZE, PROT_READ | PROT_WRITE,
                     MAP_SHARED, fd, 0);
    if (ptr == MAP_FAILED) { perror("mmap"); exit(1); }
    close(fd);

    /* 데이터 기록 */
    const char *msg = "공유 메모리로 전달된 메시지";
    memcpy(ptr, msg, strlen(msg) + 1);
    printf("생산자: 데이터 기록 완료\n");

    sleep(3);  /* 소비자가 읽을 시간 */

    munmap(ptr, SHM_SIZE);
    shm_unlink(SHM_NAME);
    return 0;
}
```

```c
/* 소비자(consumer.c) */
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <unistd.h>

#define SHM_NAME "/my_shm"
#define SHM_SIZE 4096

int main(void) {
    int fd = shm_open(SHM_NAME, O_RDONLY, 0666);
    if (fd == -1) { perror("shm_open"); exit(1); }

    void *ptr = mmap(NULL, SHM_SIZE, PROT_READ, MAP_SHARED, fd, 0);
    if (ptr == MAP_FAILED) { perror("mmap"); exit(1); }
    close(fd);

    printf("소비자: '%s'\n", (char *)ptr);

    munmap(ptr, SHM_SIZE);
    return 0;
}
```

컴파일: `gcc producer.c -o producer -lrt`, `gcc consumer.c -o consumer -lrt`

### 공유 메모리의 함정: 동기화

공유 메모리는 **동기화를 전혀 제공하지 않는다**. 생산자가 쓰는 중에 소비자가 읽으면 불완전한 데이터를 볼 수 있다. 세마포어(`sem_open`)나 뮤텍스(`pthread_mutex`)를 공유 메모리에 함께 올려 동기화해야 한다.

---

## 메커니즘 3: POSIX 메시지 큐(Message Queue)

메시지 큐는 파이프와 달리 **경계가 있는(message-boundary)** 통신을 지원한다. 파이프는 바이트 스트림이므로 메시지 경계를 애플리케이션이 직접 관리해야 한다. 메시지 큐는 쓴 단위 그대로 읽힌다.

POSIX 메시지 큐는 추가로 **우선순위**를 지원한다. 높은 우선순위 메시지는 먼저 꺼낸다.

```c
/* mq_send.c */
#include <stdio.h>
#include <stdlib.h>
#include <mqueue.h>
#include <string.h>

#define MQ_NAME "/my_mq"

int main(void) {
    struct mq_attr attr = {
        .mq_flags   = 0,
        .mq_maxmsg  = 10,    /* 큐에 최대 10개 메시지 */
        .mq_msgsize = 256,   /* 메시지 최대 크기 256바이트 */
        .mq_curmsgs = 0
    };

    mqd_t mqd = mq_open(MQ_NAME, O_CREAT | O_WRONLY, 0666, &attr);
    if (mqd == (mqd_t)-1) { perror("mq_open"); exit(1); }

    const char *msgs[] = {
        "일반 작업 1",
        "긴급 작업 (우선순위 높음)",
        "일반 작업 2"
    };
    unsigned int prios[] = {5, 10, 5};  /* 10이 더 높은 우선순위 */

    for (int i = 0; i < 3; i++) {
        mq_send(mqd, msgs[i], strlen(msgs[i]) + 1, prios[i]);
        printf("전송: '%s' (우선순위=%u)\n", msgs[i], prios[i]);
    }

    mq_close(mqd);
    return 0;
}
```

```c
/* mq_recv.c */
#include <stdio.h>
#include <stdlib.h>
#include <mqueue.h>

#define MQ_NAME "/my_mq"

int main(void) {
    mqd_t mqd = mq_open(MQ_NAME, O_RDONLY);
    if (mqd == (mqd_t)-1) { perror("mq_open"); exit(1); }

    struct mq_attr attr;
    mq_getattr(mqd, &attr);

    char buf[256];
    unsigned int prio;

    for (int i = 0; i < 3; i++) {
        ssize_t n = mq_receive(mqd, buf, sizeof(buf), &prio);
        if (n > 0)
            printf("수신: '%s' (우선순위=%u)\n", buf, prio);
    }

    mq_close(mqd);
    mq_unlink(MQ_NAME);
    return 0;
}
```

컴파일: `gcc mq_send.c -o mq_send -lrt`

수신 순서는 우선순위 기준이므로 "긴급 작업"이 먼저 출력된다.

---

## 메커니즘 4: Unix 도메인 소켓(Unix Domain Socket)

Unix 소켓은 **파일 시스템 경로를 주소로 사용하는 소켓**이다. 네트워크 소켓과 동일한 API지만 루프백 주소 없이 커널 내부에서 데이터를 전달해 TCP보다 훨씬 빠르다. 양방향 전이중(full-duplex) 통신을 기본 지원하며, 파일 디스크립터 전달(`SCM_RIGHTS`) 같은 고급 기능도 가능하다.

nginx가 PHP-FPM과 통신하거나, Docker 데몬이 CLI와 통신하는 방식이 Unix 소켓이다.

```c
/* uds_server.c — SOCK_STREAM */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <sys/un.h>

#define SOCK_PATH "/tmp/my.sock"

int main(void) {
    int srv = socket(AF_UNIX, SOCK_STREAM, 0);
    struct sockaddr_un addr = { .sun_family = AF_UNIX };
    strncpy(addr.sun_path, SOCK_PATH, sizeof(addr.sun_path) - 1);
    unlink(SOCK_PATH);

    bind(srv, (struct sockaddr *)&addr, sizeof(addr));
    listen(srv, 5);
    printf("서버 대기 중: %s\n", SOCK_PATH);

    int cli = accept(srv, NULL, NULL);
    char buf[256];
    ssize_t n = read(cli, buf, sizeof(buf));
    buf[n] = '\0';
    printf("수신: %s\n", buf);

    const char *resp = "서버 응답: 처리 완료";
    write(cli, resp, strlen(resp));

    close(cli); close(srv); unlink(SOCK_PATH);
    return 0;
}
```

---

## 메커니즘 비교 요약

| 메커니즘 | 방향 | 관계 | 속도 | 경계 | 영속성 |
|---------|------|------|------|------|--------|
| 익명 파이프 | 단방향 | 부모-자식 | 빠름 | 없음 | 프로세스 수명 |
| Named FIFO | 단방향 | 무관계 | 빠름 | 없음 | 파일시스템 |
| 공유 메모리 | 양방향 | 무관계 | 가장 빠름 | 없음 | 명시적 해제 |
| 메시지 큐 | 단방향 | 무관계 | 중간 | 있음 | 명시적 해제 |
| Unix 소켓 | 양방향 | 무관계 | 빠름 | 선택 | 파일시스템 |

---

## 주의사항 및 팁

**공유 메모리는 항상 세마포어와 함께**: 동기화 없는 공유 메모리는 레이스 컨디션의 온상이다. POSIX 세마포어(`sem_open`)를 공유 메모리에 함께 올리거나, `pthread_mutex`를 `PTHREAD_PROCESS_SHARED` 속성으로 초기화해 사용하라.

**파이프 버퍼 한계를 고려하라**: 65KB 이상 데이터를 파이프로 한 번에 쓰면 블로킹된다. `O_NONBLOCK`과 `select()`/`poll()`을 조합하거나, 대용량은 공유 메모리로 전환하라.

**메시지 큐는 커널 자원이다**: `mq_unlink()`를 호출하지 않으면 시스템이 재부팅될 때까지 `/dev/mqueue`에 남는다. `atexit()` 핸들러로 정리 루틴을 등록하라.

**Unix 소켓 권한**: `/tmp/my.sock` 파일의 권한이 접근 제어에 영향을 미친다. 필요한 최소 권한으로 설정하고, 소유자/그룹으로 접근 범위를 제한하라.

**fd 전달로 프로세스 간 커널 오브젝트 공유**: Unix 소켓의 `sendmsg(SCM_RIGHTS)`는 파일 디스크립터 자체를 다른 프로세스에 전달한다. 파일, 소켓, epoll 핸들 등 모든 fd를 전달할 수 있어 특권 분리 아키텍처에 핵심적으로 쓰인다.

## 참고 자료
- [Linux IPC Syscalls Reference — linasm.sourceforge.net](https://linasm.sourceforge.net/docs/syscalls/ipc.php)
- [Inter-process communication in Linux: Using pipes and message queues — Opensource.com](https://opensource.com/article/19/4/interprocess-communication-linux-channels)
- [sysvipc(7) — Linux Manual Page](https://man7.org/linux/man-pages/man7/sysvipc.7.html)
- [Inter-Process Communication (IPC) in Linux: A Comprehensive Guide — linuxvox.com](https://linuxvox.com/blog/ipc-linux/)
