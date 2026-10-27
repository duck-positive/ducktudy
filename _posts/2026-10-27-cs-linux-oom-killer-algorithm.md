---
layout: post
title: "Linux OOM Killer 알고리즘: 메모리 압박 상황의 프로세스 선택 원리"
date: 2026-10-27
categories: [cs, computer-science]
tags: [linux, kernel, oom-killer, memory-management, cgroups, oom_score]
---

서버 운영 중 갑자기 프로세스가 강제 종료되고 커널 로그에 `Killed process 12345 (java) total-vm:4194304kB`와 같은 메시지가 남는다면, Linux OOM(Out-Of-Memory) Killer가 작동한 것이다. 이 글에서는 OOM Killer가 어떤 알고리즘으로 어떤 프로세스를 선택해 종료하는지, 그리고 실무에서 어떻게 이를 제어할 수 있는지 깊이 있게 살펴본다.

## OOM 상황이 발생하는 조건

Linux 커널은 메모리 할당 요청을 처리할 때 실제로 물리 메모리가 부족해지면 OOM 상태로 진입한다. 단순히 메모리가 없을 때만 발생하는 것이 아니라, 다음 조건들이 복합적으로 작용한다.

1. **가상 메모리 과할당(overcommit)**: Linux는 기본적으로 요청된 메모리보다 더 많은 가상 주소 공간을 허용한다(`vm.overcommit_memory = 0 또는 1`). 실제 물리 메모리에 접근하는 시점(page fault)에야 실제 메모리가 할당되므로, 메모리를 `malloc`으로 할당해도 곧바로 물리 메모리를 차지하지 않는다.

2. **스왑 공간 고갈**: 스왑이 없거나 스왑도 모두 사용된 상태에서 추가 할당이 요청되면 OOM이 발생한다.

3. **cgroup 메모리 한도 초과**: cgroup v1/v2의 `memory.limit_in_bytes`(v1) 또는 `memory.max`(v2)를 초과할 경우 해당 cgroup 내에서 OOM이 발생한다.

## OOM Killer의 동작 흐름

커널 소스(`mm/oom_kill.c`)에서 OOM이 발생하면 다음 순서로 동작한다.

```
페이지 할당 실패 (__alloc_pages_nodemask)
        ↓
out_of_memory() 호출
        ↓
oom_kill_process() 호출 여부 결정
        ↓
select_bad_process() — 죽일 프로세스 선택
        ↓
oom_badness() — 각 프로세스 점수 계산 (0~1000)
        ↓
가장 높은 점수의 프로세스에 SIGKILL 전송
        ↓
oom_kill_process() 실행, 메모리 해제 대기
```

### oom_badness() 계산 방식

핵심은 `oom_badness()` 함수다. Linux 2.6.36 이후 단순화된 알고리즘을 사용한다.

```
점수(0~1000) = (프로세스의 RSS + 스왑 사용량) / 총 사용 가능 메모리 * 1000
             + oom_score_adj (조정값: -1000 ~ +1000)
```

- **RSS(Resident Set Size)**: 현재 물리 메모리에 올라와 있는 페이지 수
- **총 사용 가능 메모리**: 전체 RAM + 스왑 + 루트 사용자 예약분(3%)
- **oom_score_adj**: 관리자가 직접 조정하는 값, -1000이면 항상 0점 (절대 안 죽임), +1000이면 항상 1000점 (최우선 종료 대상)

점수가 0인 프로세스(init, 커널 스레드 등)는 절대 종료되지 않는다.

## 코드 예제 1: OOM Score 모니터링 (Python)

```python
import os
import glob
from dataclasses import dataclass
from typing import Optional

@dataclass
class ProcessOOMInfo:
    pid: int
    name: str
    oom_score: int
    oom_score_adj: int
    rss_kb: int

def read_file_safe(path: str) -> Optional[str]:
    try:
        with open(path, 'r') as f:
            return f.read().strip()
    except (IOError, PermissionError):
        return None

def get_process_oom_info(pid: int) -> Optional[ProcessOOMInfo]:
    base = f"/proc/{pid}"
    
    name = read_file_safe(f"{base}/comm")
    oom_score = read_file_safe(f"{base}/oom_score")
    oom_score_adj = read_file_safe(f"{base}/oom_score_adj")
    status = read_file_safe(f"{base}/status")
    
    if not all([name, oom_score, oom_score_adj, status]):
        return None
    
    # VmRSS 추출
    rss_kb = 0
    for line in status.split('\n'):
        if line.startswith('VmRSS:'):
            parts = line.split()
            if len(parts) >= 2:
                rss_kb = int(parts[1])
            break
    
    return ProcessOOMInfo(
        pid=pid,
        name=name,
        oom_score=int(oom_score),
        oom_score_adj=int(oom_score_adj),
        rss_kb=rss_kb,
    )

def list_top_oom_candidates(top_n: int = 20) -> list[ProcessOOMInfo]:
    results = []
    for proc_dir in glob.glob("/proc/[0-9]*/"):
        try:
            pid = int(proc_dir.split('/')[2])
        except (ValueError, IndexError):
            continue
        
        info = get_process_oom_info(pid)
        if info is not None:
            results.append(info)
    
    return sorted(results, key=lambda x: x.oom_score, reverse=True)[:top_n]

def print_oom_report():
    candidates = list_top_oom_candidates()
    print(f"{'PID':>8} {'이름':<20} {'OOM점수':>8} {'adj':>6} {'RSS(MB)':>10}")
    print("-" * 60)
    for p in candidates:
        print(
            f"{p.pid:>8} {p.name:<20} {p.oom_score:>8} "
            f"{p.oom_score_adj:>+6} {p.rss_kb // 1024:>9}MB"
        )

if __name__ == "__main__":
    print_oom_report()
```

실행 결과 예시:
```
     PID 이름                  OOM점수    adj    RSS(MB)
------------------------------------------------------------
    1234 java                      842     +0       2048MB
    5678 python3                   631     +0        512MB
    9012 nginx                      45  -300         32MB
```

## 코드 예제 2: oom_score_adj 설정 및 cgroup 메모리 제한 (Bash + Python)

```bash
#!/bin/bash
# 특정 프로세스의 OOM 우선순위 조정

PID=$1
ADJ_VALUE=$2  # -1000 (절대 보호) ~ +1000 (최우선 종료)

if [ -z "$PID" ] || [ -z "$ADJ_VALUE" ]; then
    echo "Usage: $0 <pid> <adj_value>"
    exit 1
fi

if [ ! -d "/proc/$PID" ]; then
    echo "프로세스 $PID 를 찾을 수 없습니다"
    exit 1
fi

PROC_NAME=$(cat /proc/$PID/comm 2>/dev/null)
CURRENT_ADJ=$(cat /proc/$PID/oom_score_adj 2>/dev/null)
CURRENT_SCORE=$(cat /proc/$PID/oom_score 2>/dev/null)

echo "프로세스: $PROC_NAME (PID=$PID)"
echo "현재 oom_score_adj: $CURRENT_ADJ"
echo "현재 oom_score: $CURRENT_SCORE"
echo "→ oom_score_adj를 $ADJ_VALUE 로 변경"

echo "$ADJ_VALUE" > /proc/$PID/oom_score_adj
if [ $? -eq 0 ]; then
    NEW_SCORE=$(cat /proc/$PID/oom_score)
    echo "✓ 변경 완료. 새 oom_score: $NEW_SCORE"
else
    echo "✗ 변경 실패 (권한 확인 필요)"
fi
```

```python
# cgroup v2 메모리 제한 설정 예시
import os
import subprocess

CGROUP_ROOT = "/sys/fs/cgroup"

def create_memory_limited_cgroup(name: str, limit_mb: int) -> str:
    cgroup_path = os.path.join(CGROUP_ROOT, name)
    os.makedirs(cgroup_path, exist_ok=True)
    
    # 메모리 상한 설정 (bytes)
    limit_bytes = limit_mb * 1024 * 1024
    with open(os.path.join(cgroup_path, "memory.max"), "w") as f:
        f.write(str(limit_bytes))
    
    # 스왑 포함 총 메모리 상한 설정
    with open(os.path.join(cgroup_path, "memory.swap.max"), "w") as f:
        f.write("0")  # 스왑 사용 금지
    
    print(f"cgroup '{name}' 생성: 메모리 한도 {limit_mb}MB, 스왑 없음")
    return cgroup_path

def add_process_to_cgroup(cgroup_path: str, pid: int) -> None:
    with open(os.path.join(cgroup_path, "cgroup.procs"), "w") as f:
        f.write(str(pid))
    print(f"PID {pid} → cgroup {cgroup_path} 등록 완료")

def get_cgroup_memory_stats(cgroup_path: str) -> dict:
    stats = {}
    stat_file = os.path.join(cgroup_path, "memory.stat")
    try:
        with open(stat_file) as f:
            for line in f:
                key, _, value = line.partition(' ')
                stats[key] = int(value.strip())
    except IOError:
        pass
    return stats

# 사용 예:
# cg = create_memory_limited_cgroup("my-app", 512)
# add_process_to_cgroup(cg, os.getpid())
```

## OOM Killer의 실제 로그 분석

OOM 발생 시 `/var/log/kern.log` 또는 `dmesg`에서 상세 로그를 확인할 수 있다.

```
[12345.678] oom-kill:constraint=CONSTRAINT_NONE,nodemask=(null),cpuset=/,mems_allowed=0,global_oom,task_memcg=/system.slice/myapp.service,task=java,pid=1234,uid=1000
[12345.678] Out of memory: Killed process 1234 (java) total-vm:8388608kB, anon-rss:4194304kB, file-rss:65536kB, shmem-rss:0kB, UID:1000 pgtables:16384kB oom_score_adj:0
```

- **total-vm**: 가상 메모리 전체 크기
- **anon-rss**: 익명 메모리(힙, 스택) RSS
- **file-rss**: 파일 매핑 RSS (공유 라이브러리 등)
- **pgtables**: 페이지 테이블 크기
- **oom_score_adj**: 종료 당시의 adj 값

## 주의사항과 실무 팁

### 1. Java 애플리케이션과 OOM

JVM은 `-Xmx`로 힙을 제한해도 네이티브 메모리(Metaspace, 코드 캐시, 스레드 스택, NIO 버퍼)를 추가로 사용한다. JVM 프로세스의 실제 RSS는 `-Xmx` 값보다 훨씬 클 수 있다.

```bash
# JVM 네이티브 메모리 추적
java -XX:NativeMemoryTracking=summary -Xmx1g -jar app.jar

# 메모리 사용 현황 출력
jcmd <pid> VM.native_memory summary
```

### 2. 컨테이너(Docker/Kubernetes) 환경

컨테이너 환경에서는 cgroup 메모리 한도가 OOM Killer를 트리거한다. `resources.limits.memory` 설정이 핵심이다.

```yaml
# Kubernetes Pod 메모리 제한
resources:
  requests:
    memory: "512Mi"
  limits:
    memory: "1Gi"  # 이 값 초과 시 cgroup OOM → Pod 재시작
```

컨테이너가 재시작되는 이유가 `OOMKilled`인지 확인하려면:
```bash
kubectl describe pod <pod-name> | grep -A5 "Last State"
# Last State: Terminated
#   Reason: OOMKilled
```

### 3. oom_score_adj 활용 전략

| 값 범위 | 용도 |
|---------|------|
| -1000 | 절대 종료 불가 (init, 커널 데몬) |
| -500 ~ -100 | 중요 시스템 서비스 (DB, 모니터링 에이전트) |
| 0 (기본) | 일반 사용자 프로세스 |
| +500 ~ +999 | 재시작이 쉬운 작업자 프로세스, 배치 작업 |
| +1000 | 항상 가장 먼저 종료 (테스트 프로세스) |

### 4. vm.overcommit_memory 설정

```bash
# 0: 기본 휴리스틱 (과할당 허용, 명백히 불가한 경우만 거부)
# 1: 항상 과할당 허용 (OOM 위험 증가)
# 2: 엄격 모드 (RAM * overcommit_ratio + swap 이상 할당 거부)
cat /proc/sys/vm/overcommit_memory  # 현재 설정 확인
sysctl vm.overcommit_memory=2        # 엄격 모드 적용
```

### 5. OOM 발생 전에 감지하기

Prometheus + node_exporter를 사용한다면 다음 메트릭을 모니터링하라:
- `node_memory_MemAvailable_bytes` < 전체 RAM의 10% → 경고
- `node_vmstat_oom_kill` 증가 → OOM 발생 감지

## 참고 자료
- [Linux Kernel OOM Killer Source - mm/oom_kill.c](https://github.com/torvalds/linux/blob/master/mm/oom_kill.c)
- [proc_pid_oom_score_adj(5) - Linux Manual Page](https://man7.org/linux/man-pages/man5/proc_pid_oom_score_adj.5.html)
- [Linux cgroup v2 Memory Controller 공식 문서](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
- [Kubernetes OOM Killer 문서](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
