---
layout: post
title: "가상화와 컨테이너 심화: Hypervisor, cgroups, Linux Namespace로 이해하는 격리의 원리"
date: 2026-06-26
categories: [cs, computer-science]
tags: [cgroups, namespaces, container, docker, virtualization, hypervisor, linux-kernel, isolation, seccomp]
---

## 개요

"Docker는 VM이 아니라 프로세스다"라는 말을 들어본 적이 있을 것이다. 그렇다면 단순한 프로세스가 어떻게 독립된 파일 시스템, 독립된 네트워크 스택, 제한된 CPU/메모리를 가질 수 있을까? 답은 리눅스 커널의 두 가지 기술, **Linux Namespaces** 와 **cgroups(Control Groups)** 에 있다. 이 글에서는 하이퍼바이저 기반 가상화와 컨테이너의 차이를 먼저 이해하고, 리눅스 커널 수준에서 컨테이너가 어떻게 구현되는지 직접 코드와 명령으로 확인한다.

---

## 1. 가상화의 두 계층: VM vs 컨테이너

### 1.1 Type 1 / Type 2 하이퍼바이저

**하이퍼바이저(Hypervisor)**는 물리 하드웨어 위에서 여러 개의 가상 머신(VM)을 실행하는 소프트웨어다.

| 구분 | Type 1 (Bare-metal) | Type 2 (Hosted) |
|------|---------------------|-----------------|
| 실행 위치 | 하드웨어 위에 직접 | 호스트 OS 위 |
| 예시 | VMware ESXi, Xen, KVM | VirtualBox, VMware Workstation |
| 성능 | 높음 | 상대적으로 낮음 |
| 격리 강도 | 매우 강함 (완전히 독립된 커널) | 강함 |

VM은 각자 **독립된 OS 커널**을 가진다. 완전한 격리이지만 부팅에 수십 초, 수백 MB의 메모리 오버헤드가 따른다.

### 1.2 컨테이너: 커널을 공유하는 격리

컨테이너는 **호스트 OS의 커널을 공유**하면서 프로세스를 논리적으로 격리한다. 격리를 구현하는 핵심 기술이 Namespace와 cgroups다.

```
VM 구조:                     컨테이너 구조:

┌──────────┐ ┌──────────┐   ┌──────────┐ ┌──────────┐
│   App A  │ │   App B  │   │   App A  │ │   App B  │
├──────────┤ ├──────────┤   ├──────────┤ ├──────────┤
│ Guest OS │ │ Guest OS │   │  Libs A  │ │  Libs B  │
├──────────┤ ├──────────┤   └──────────┘ └──────────┘
│ VMM/HW   │ │ VMM/HW   │         ↓           ↓
└──────────┘ └──────────┘   ┌────────────────────────┐
      ↓           ↓         │    Host OS Kernel      │
┌───────────────────────┐   │  (Namespace + cgroups) │
│    Type 1 Hypervisor  │   └────────────────────────┘
└───────────────────────┘         ↓
┌───────────────────────┐   ┌────────────────────────┐
│    Physical Hardware  │   │    Physical Hardware    │
└───────────────────────┘   └────────────────────────┘
```

컨테이너는 커널 공유 덕분에 밀리초 단위로 시작하고, 메모리 오버헤드는 프로세스 수준이다.

---

## 2. Linux Namespaces: 격리의 구현

Namespace는 **프로세스가 보는 시스템 자원의 뷰를 격리**하는 커널 기능이다. 현재 리눅스는 8종류의 Namespace를 지원한다.

| Namespace | 격리 대상 | 관련 플래그 |
|-----------|-----------|-------------|
| PID | 프로세스 ID 트리 | CLONE_NEWPID |
| Network | 네트워크 인터페이스, 라우팅 | CLONE_NEWNET |
| Mount | 파일 시스템 마운트 포인트 | CLONE_NEWNS |
| UTS | 호스트명, 도메인명 | CLONE_NEWUTS |
| IPC | System V IPC, POSIX 메시지 큐 | CLONE_NEWIPC |
| User | UID/GID 매핑 | CLONE_NEWUSER |
| Cgroup | cgroup 루트 뷰 | CLONE_NEWCGROUP |
| Time | 시스템 클럭 (Linux 5.6+) | CLONE_NEWTIME |

### 2.1 unshare로 직접 Namespace 체험

```bash
# 새 UTS namespace에서 호스트명 변경 (호스트에는 영향 없음)
sudo unshare --uts /bin/bash -c '
    hostname my-container
    hostname   # "my-container" 출력
'
hostname   # 호스트: 변경 없음

# PID namespace: 격리된 프로세스 트리에서 PID 1이 되어보기
sudo unshare --pid --fork --mount-proc /bin/bash
ps aux   # 이 bash가 PID 1로 보임

# 모든 namespace를 격리한 미니 컨테이너
sudo unshare \
    --pid --fork \
    --mount-proc \
    --net \
    --uts \
    --ipc \
    /bin/bash -c '
        hostname sandbox
        echo "My PID: $$"     # 1
        ip link show           # lo만 보임 (격리된 네트워크)
        ps aux                 # 이 bash만 보임
    '
```

### 2.2 clone() 시스템 콜로 컨테이너 구현

Docker는 내부적으로 `clone()`(또는 `unshare()`) 시스템 콜을 사용한다. C로 직접 구현해보자.

```c
// mini_container.c — namespace를 사용한 최소 컨테이너 구현
#define _GNU_SOURCE
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <sys/mount.h>

#define STACK_SIZE (1024 * 1024)  // 자식 프로세스 스택 1MB

// 자식 프로세스(컨테이너 내부)에서 실행될 함수
static int container_main(void* arg) {
    // UTS namespace: 호스트명 설정
    if (sethostname("my-container", 12) < 0) {
        perror("sethostname");
        return 1;
    }

    // Mount namespace: /proc 다시 마운트
    // (PID namespace가 격리되어 있으므로 새 /proc 필요)
    mount("proc", "/proc", "proc", 0, NULL);

    printf("[container] hostname: my-container\n");
    printf("[container] PID: %d (항상 1)\n", getpid());

    // 쉘 실행 (실제 컨테이너처럼)
    char* const args[] = {"/bin/sh", NULL};
    execvp("/bin/sh", args);

    // 정리: /proc 언마운트
    umount("/proc");
    return 0;
}

int main(void) {
    // 자식 프로세스용 스택 할당
    char* stack = malloc(STACK_SIZE);
    if (!stack) {
        perror("malloc");
        return 1;
    }

    // clone() — fork()와 달리 격리 플래그 지정 가능
    // CLONE_NEWPID: PID namespace 격리
    // CLONE_NEWUTS: UTS(hostname) namespace 격리
    // CLONE_NEWNS:  Mount namespace 격리
    // CLONE_NEWNET: Network namespace 격리
    int flags = CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS | SIGCHLD;

    pid_t pid = clone(
        container_main,
        stack + STACK_SIZE,  // 스택은 위에서 아래로 자람
        flags,
        NULL
    );

    if (pid < 0) {
        perror("clone");
        free(stack);
        return 1;
    }

    printf("[host] Container started with PID: %d\n", pid);

    // 컨테이너 프로세스 종료 대기
    int status;
    waitpid(pid, &status, 0);
    printf("[host] Container exited with status: %d\n",
           WEXITSTATUS(status));

    free(stack);
    return 0;
}
```

```bash
# 빌드 및 실행 (루트 권한 필요)
gcc mini_container.c -o mini_container
sudo ./mini_container

# 컨테이너 내부에서:
# $ hostname     → my-container
# $ ps aux       → 이 sh만 보임 (PID 1)
# $ ip link      → lo만 있음 (별도 net namespace)
```

---

## 3. cgroups: 자원을 제한하고 계측한다

**cgroups(Control Groups)**는 프로세스 그룹의 CPU, 메모리, 디스크 I/O, 네트워크 대역폭 등 자원 사용량을 **제한(Limit)**, **우선순위 부여(Prioritize)**, **계측(Account)**, **격리(Isolate)** 하는 커널 기능이다.

### 3.1 cgroup v1 vs cgroup v2

현재 리눅스에는 두 버전이 공존한다.

- **cgroup v1**: 컨트롤러별로 독립적인 계층 구조. `/sys/fs/cgroup/cpu/`, `/sys/fs/cgroup/memory/` 등 여러 마운트 포인트.
- **cgroup v2 (통합 계층)**: 단일 계층 구조. `/sys/fs/cgroup/` 하나. 리눅스 4.5+, systemd 232+ 기본. Docker 20.10+에서 기본 지원.

```bash
# 현재 cgroup 버전 확인
stat -f /sys/fs/cgroup | grep Type
# cgroup2 → v2, tmpfs → v1

# v2: 단일 통합 계층
ls /sys/fs/cgroup/
# cgroup.controllers  cgroup.stat  memory.stat  system.slice/  user.slice/ ...
```

### 3.2 cgroups v2로 직접 CPU/메모리 제한하기

```bash
# 1. 새 cgroup 생성
sudo mkdir /sys/fs/cgroup/my-container

# 2. 사용할 컨트롤러 활성화
echo "+cpu +memory +pids" | sudo tee /sys/fs/cgroup/my-container/cgroup.subtree_control

# 3. 서브 cgroup 생성
sudo mkdir /sys/fs/cgroup/my-container/app

# 4. CPU 제한: 전체 CPU의 50% (쿼터/피리어드 방식)
#    cpu.max = <quota us> <period us>
#    500000 / 1000000 = 50%
echo "500000 1000000" | sudo tee /sys/fs/cgroup/my-container/app/cpu.max

# 5. 메모리 제한: 최대 256MB
echo "268435456" | sudo tee /sys/fs/cgroup/my-container/app/memory.max

# 6. 프로세스 수 제한: 최대 50개
echo "50" | sudo tee /sys/fs/cgroup/my-container/app/pids.max

# 7. 현재 bash를 이 cgroup에 추가
echo $$ | sudo tee /sys/fs/cgroup/my-container/app/cgroup.procs

# 8. CPU 집약적 작업 실행 (50% CPU 제한이 걸림)
stress --cpu 4 --timeout 10 &

# 9. 실제 사용량 확인
cat /sys/fs/cgroup/my-container/app/cpu.stat
# usage_usec 12456789  → 총 CPU 사용 마이크로초
# throttled_usec 6234567  → 제한으로 인한 지연

cat /sys/fs/cgroup/my-container/app/memory.current
# 현재 사용 중인 메모리 (바이트)

# 10. 정리
sudo rmdir /sys/fs/cgroup/my-container/app
sudo rmdir /sys/fs/cgroup/my-container
```

### 3.3 Python으로 cgroup 통계 모니터링

```python
#!/usr/bin/env python3
# cgroup_monitor.py — cgroup v2 통계 실시간 모니터링
import time
import os
from pathlib import Path

CGROUP_PATH = Path("/sys/fs/cgroup")


def read_stat(cgroup: str, file: str) -> str:
    """cgroup 파일 읽기"""
    p = CGROUP_PATH / cgroup / file
    if p.exists():
        return p.read_text().strip()
    return "N/A"


def parse_cpu_stat(stat_text: str) -> dict:
    """cpu.stat 파싱"""
    result = {}
    for line in stat_text.splitlines():
        parts = line.split()
        if len(parts) == 2:
            result[parts[0]] = int(parts[1])
    return result


def monitor_container_cgroup(cgroup_name: str, interval: float = 1.0):
    """컨테이너 cgroup 자원 사용 현황 출력"""
    cgroup = f"my-container/{cgroup_name}"

    print(f"Monitoring cgroup: {cgroup}")
    print(f"{'Time':>10} {'CPU(ms)':>10} {'Throttle(ms)':>14} {'Mem(MB)':>10} {'PIDs':>6}")
    print("-" * 55)

    prev_usage = 0
    while True:
        # CPU 통계
        cpu_stat = read_stat(cgroup, "cpu.stat")
        if cpu_stat != "N/A":
            stat = parse_cpu_stat(cpu_stat)
            usage_us = stat.get("usage_usec", 0)
            throttled_us = stat.get("throttled_usec", 0)
            delta_ms = (usage_us - prev_usage) / 1000
            prev_usage = usage_us
        else:
            delta_ms = throttled_us = 0

        # 메모리 통계
        mem_current = read_stat(cgroup, "memory.current")
        mem_mb = int(mem_current) / (1024 * 1024) if mem_current != "N/A" else 0

        # PID 수
        pids = read_stat(cgroup, "pids.current")

        print(f"{time.strftime('%H:%M:%S'):>10} {delta_ms:>10.1f} "
              f"{throttled_us/1000:>14.1f} {mem_mb:>10.1f} {pids:>6}")

        time.sleep(interval)


if __name__ == "__main__":
    monitor_container_cgroup("app", interval=1.0)
```

---

## 4. 완전한 컨테이너: Namespace + cgroups + rootfs

실제 Docker가 하는 일은 다음 세 가지 기술의 조합이다.

1. **Namespaces**: PID, Net, Mount, UTS, IPC, User 격리
2. **cgroups**: CPU, 메모리, I/O, PID 수 제한
3. **overlayfs + chroot**: 컨테이너 파일 시스템 격리

```bash
# Docker inspect로 실제 cgroup 경로 확인
docker run -d --name test-container --cpus=0.5 --memory=256m nginx

# Docker가 생성한 cgroup 확인
CONTAINER_ID=$(docker inspect --format '{{.Id}}' test-container)
ls /sys/fs/cgroup/system.slice/docker-${CONTAINER_ID}.scope/

# Docker 컨테이너의 Namespace 확인
PID=$(docker inspect --format '{{.State.Pid}}' test-container)
ls -la /proc/$PID/ns/
# lrwxrwxrwx cgroup -> cgroup:[4026532xxx]
# lrwxrwxrwx ipc    -> ipc:[4026532xxx]
# lrwxrwxrwx mnt    -> mnt:[4026532xxx]
# lrwxrwxrwx net    -> net:[4026532xxx]
# lrwxrwxrwx pid    -> pid:[4026532xxx]
# lrwxrwxrwx uts    -> uts:[4026532xxx]

# 컨테이너의 namespace에서 명령 실행 (nsenter)
sudo nsenter --target $PID --pid --net --mount -- /bin/sh
# 이제 컨테이너 내부에서 실행하는 것과 동일
```

### 4.1 overlayfs: 컨테이너 이미지 레이어

```
Docker 이미지 레이어 구조 (overlayfs):

┌─────────────────────────────────┐
│ Container Layer (읽기/쓰기)     │  ← 컨테이너 실행 중 변경 사항
├─────────────────────────────────┤
│ Image Layer 3 (읽기 전용)       │  ← RUN apt-get install nginx
├─────────────────────────────────┤
│ Image Layer 2 (읽기 전용)       │  ← COPY app.conf
├─────────────────────────────────┤
│ Image Layer 1 (읽기 전용)       │  ← FROM ubuntu:22.04
└─────────────────────────────────┘

overlayfs가 이 레이어들을 합쳐 단일 파일 시스템 뷰로 제공한다.
컨테이너가 파일을 수정하면 Copy-on-Write로 Container Layer에 복사본이 생긴다.
```

---

## 5. 주의사항 및 보안 강화

### 5.1 Namespace는 커널 취약점에 취약하다

VM과 달리 컨테이너는 커널을 공유한다. 커널 취약점을 통해 컨테이너 탈출(Container Escape)이 가능하다. CVE-2019-5736(runc 취약점)이 대표적인 사례다.

**완화 방법:**
- **seccomp 프로파일**: 시스템 콜 필터링으로 공격 표면 축소
- **AppArmor/SELinux**: 강제 접근 제어(MAC)
- **User Namespace**: 컨테이너 내 root를 호스트의 비권한 UID로 매핑
- **rootless 컨테이너**: Docker rootless mode, Podman 기본값

```bash
# Docker 컨테이너에 seccomp 프로파일 적용
docker run --security-opt seccomp=/etc/docker/seccomp.json nginx

# User namespace: 컨테이너 내 UID 0이 호스트에서 UID 100000으로 매핑
docker run --userns-remap=default nginx

# 기본 seccomp로 차단된 시스템 콜 확인
docker run --rm alpine strace -e trace=ptrace sleep 1
# ptrace: Operation not permitted (seccomp에 의해 차단)
```

### 5.2 cgroup v2 메모리 OOM 동작 이해

메모리 제한 초과 시 커널 OOM Killer가 가장 많은 메모리를 사용하는 프로세스를 종료한다. 컨테이너에서는 보통 PID 1(엔트리포인트)이 종료되어 컨테이너 전체가 내려간다.

```bash
# OOM 이벤트 감시 (cgroup v2)
cat /sys/fs/cgroup/my-container/app/memory.events
# oom 3              → OOM이 3번 발생
# oom_kill 2         → OOM killer가 2번 프로세스를 죽임
# oom_group_kill 1   → 그룹 전체를 종료한 횟수

# memory.oom.group=1 설정 시 OOM 발생하면 전체 cgroup 종료
echo 1 | sudo tee /sys/fs/cgroup/my-container/app/memory.oom.group
```

### 5.3 cgroup v1과 v2의 혼용 주의

일부 레거시 시스템에서는 v1과 v2가 혼용된다. Docker와 시스템의 cgroup 버전이 다를 경우 예상치 못한 제한이 적용되지 않을 수 있다.

```bash
# systemd-cgls로 전체 cgroup 트리 확인
systemd-cgls

# cgroup v2에서 사용 가능한 컨트롤러 확인
cat /sys/fs/cgroup/cgroup.controllers
# cpuset cpu io memory hugetlb pids rdma misc
```

---

## 6. 가상화 스펙트럼 정리

| 기술 | 격리 강도 | 시작 시간 | 메모리 오버헤드 | 커널 공유 |
|------|-----------|-----------|----------------|-----------|
| Type 1 VM (KVM/ESXi) | 최고 | 수십 초 | 수백 MB | 아니오 |
| Type 2 VM (VirtualBox) | 높음 | 수십 초~분 | 수백 MB | 아니오 |
| gVisor (샌드박스 컨테이너) | 높음 | 수백 ms | ~수십 MB | 부분 (UserSpace 커널) |
| 일반 컨테이너 (Docker) | 중간 | 수십 ms | 수 MB | 예 |
| rootless 컨테이너 (Podman) | 중간+ | 수십 ms | 수 MB | 예 (User NS) |

---

## 7. 정리

컨테이너는 Namespace로 프로세스의 **보이는 것(뷰)**을 격리하고, cgroups로 사용할 수 있는 **자원 양**을 제한한다. 여기에 overlayfs를 통한 파일 시스템 격리가 더해져 완전한 컨테이너가 탄생한다.

Docker는 이 세 기술을 편리한 API로 포장한 것이다. `docker run --cpus=0.5 --memory=256m nginx` 명령 하나 뒤에서 커널은 새 PID Namespace를 생성하고, cgroup에 CPU 쿼터를 설정하고, overlayfs 레이어를 합치고, 네트워크 인터페이스를 생성한다. 이 내부 동작을 이해하면 컨테이너 성능 튜닝, 보안 강화, 그리고 "컨테이너 탈출"이 왜 발생하는지를 정확히 파악할 수 있다.

## 참고 자료
- [cgroup_namespaces(7) — Linux Manual Page](https://linux.die.net/man/7/namespaces)
- [cgroups — Wikipedia](https://en.wikipedia.org/wiki/Cgroups)
- [What Are Namespaces and cgroups? — NGINX Blog](https://blog.nginx.org/blog/what-are-namespaces-cgroups-how-do-they-work)
- [How Docker Containers Work — namespaces and cgroups](https://earthly.dev/blog/namespaces-and-cgroups-docker/)
