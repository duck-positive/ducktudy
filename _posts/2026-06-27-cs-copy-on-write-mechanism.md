---
layout: post
title: "Copy-on-Write(CoW) 완전 정복: Linux fork와 Redis BGSAVE가 메모리를 공유하는 방법"
date: 2026-06-27
categories: [cs, computer-science]
tags: [copy-on-write, linux, kernel, fork, redis, memory-management, operating-system, cow]
---

Copy-on-Write(CoW)는 운영체제와 데이터베이스 시스템에서 가장 우아하게 쓰이는 메모리 최적화 기법 중 하나다. "쓸 때만 복사한다"는 단순한 원칙 하나가 Linux `fork()` 시스템 콜, Redis BGSAVE, Git 내부 오브젝트 저장소, 심지어 프로그래밍 언어의 문자열 구현까지 폭넓게 관통한다. 이 글에서는 CoW의 하드웨어·커널 레벨 동작 원리부터 Redis 운영 시 주의해야 할 메모리 급증 현상까지 깊이 파고든다.

---

## 개념 설명

### CoW의 핵심 아이디어

Copy-on-Write는 여러 주체가 동일한 리소스를 **읽기 전용으로 공유**하다가, 그 중 어느 하나가 **쓰기를 시도하는 순간**에만 별도의 복사본을 만드는 지연 복사(lazy copy) 전략이다.

이를 물리적 메모리 관점으로 구체화하면 다음과 같다:

```
[프로세스 A] → 가상 주소 0x1000 ─┐
                                    ├─→ 물리 페이지 P (읽기 전용)
[프로세스 B] → 가상 주소 0x2000 ─┘

프로세스 A가 0x1000에 쓰기 시도:
  1. 페이지 폴트(Page Fault) 발생
  2. 커널이 물리 페이지 P를 복사 → 새 페이지 P'
  3. 프로세스 A의 PTE(Page Table Entry)를 P'로 교체
  4. P'에 쓰기 허용, 쓰기 재실행
  5. 프로세스 B는 여전히 원본 P를 사용
```

이 과정에서 "비용"은 쓰기가 실제로 일어날 때만 발생한다. 두 프로세스가 데이터를 읽기만 한다면 물리 메모리는 단 한 번도 복사되지 않는다.

### 하드웨어 지원: MMU와 페이지 폴트

CoW는 CPU의 MMU(Memory Management Unit)가 없으면 구현할 수 없다. MMU는 가상 주소를 물리 주소로 변환하는 과정에서 각 페이지의 **접근 권한 비트**를 검사한다. CoW로 공유된 페이지는 PTE에서 `W`(Write) 비트가 0으로 세팅되어 있다. 쓰기가 시도되면 CPU는 **보호 페이지 폴트**를 발생시키고, 커널의 폴트 핸들러가 위 복사 과정을 수행한 뒤 쓰기를 재실행한다.

---

## 왜 필요한가

### Linux fork()의 O(1) 자식 생성

전통적인 UNIX `fork()` 구현은 부모 프로세스의 메모리 전체를 자식에게 복사했다. 부모가 1GB 메모리를 사용하고 있다면 자식 생성에만 1GB를 복사해야 했다. 많은 경우 자식은 `exec()` 를 즉시 호출해서 기존 메모리를 완전히 버리는 패턴(fork-exec idiom)을 사용하므로, 복사 비용이 순전한 낭비였다.

CoW를 적용하면 `fork()` 시점에 발생하는 실제 복사는 **페이지 테이블 복사와 PTE 권한 변경**뿐이다. 수백만 페이지를 물리적으로 복사하는 대신 수천 개의 PTE만 변경하면 되므로, fork 비용이 메모리 크기와 무관하게 거의 일정해진다.

### Redis BGSAVE: 스냅샷 중에도 클라이언트 요청 처리

Redis는 단일 스레드 이벤트 루프로 동작한다. RDB 스냅샷을 메인 스레드에서 직렬로 수행하면 그 동안 모든 클라이언트 요청이 블록된다. `BGSAVE` 명령은 이 문제를 CoW로 우아하게 해결한다:

1. `BGSAVE` 명령 수신
2. `fork()` 호출 → 자식 프로세스 생성 (CoW로 메모리 공유)
3. **자식**: 공유된 메모리 스냅샷을 순차적으로 읽어 `.rdb` 파일 기록
4. **부모(메인)**: 클라이언트 요청을 계속 처리하며 변경된 페이지만 CoW로 분리

자식이 보는 메모리는 `fork()` 시점의 일관된 스냅샷이고, 부모가 쓰기를 수행해도 자식의 뷰에는 영향이 없다.

---

## 실제 구현 예제

### 예제 1: Linux fork()와 CoW 동작 확인 (C)

아래 코드는 부모·자식 프로세스가 같은 변수의 주소를 공유하다가 쓰기 시점에 분기되는 것을 보여준다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    int value = 42;

    printf("[부모] fork 전: value = %d, 주소 = %p\n", value, (void*)&value);

    pid_t pid = fork();

    if (pid == 0) {
        // 자식 프로세스
        printf("[자식] fork 직후: value = %d, 주소 = %p\n", value, (void*)&value);

        // 쓰기 발생 → CoW 트리거
        value = 99;
        printf("[자식] 쓰기 후: value = %d, 주소 = %p\n", value, (void*)&value);
        exit(0);
    } else {
        // 부모 프로세스
        wait(NULL);
        // 자식의 쓰기가 부모에는 영향 없음
        printf("[부모] 자식 종료 후: value = %d, 주소 = %p\n", value, (void*)&value);
    }

    return 0;
}
```

**실행 결과:**
```
[부모] fork 전: value = 42, 주소 = 0x7ffe12345678
[자식] fork 직후: value = 42, 주소 = 0x7ffe12345678   ← 같은 가상 주소
[자식] 쓰기 후: value = 99, 주소 = 0x7ffe12345678    ← 같은 가상 주소, 다른 물리 페이지
[부모] 자식 종료 후: value = 42, 주소 = 0x7ffe12345678  ← 부모는 영향 없음
```

가상 주소는 동일하지만 물리 페이지는 쓰기 시점에 분기된다. 이것이 CoW의 핵심이다.

### 예제 2: Redis BGSAVE 메모리 사용량 모니터링 (Python + redis-py)

실제 운영 환경에서 BGSAVE 중 CoW로 인한 메모리 증가량을 모니터링하는 스크립트다.

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def monitor_cow_memory():
    """BGSAVE 전후 CoW 메모리 사용량 비교"""
    info_before = r.info('memory')
    used_before = info_before['used_memory_human']
    print(f"BGSAVE 전 메모리: {used_before}")

    # BGSAVE 시작
    r.bgsave()
    print("BGSAVE 시작됨 (자식 프로세스 fork)")

    # BGSAVE 진행 중 메모리 모니터링
    for _ in range(10):
        time.sleep(1)
        info = r.info('all')

        rdb_in_progress = info.get('rdb_bgsave_in_progress', 0)
        if not rdb_in_progress:
            break

        # CoW로 인해 복사된 바이트 수
        cow_size = info.get('rdb_last_cow_size', 0)
        used_memory = info['used_memory_human']
        child_mem = info.get('used_memory_rss_human', 'N/A')

        print(f"  진행 중 | 메모리: {used_memory} | "
              f"RSS: {child_mem} | "
              f"CoW 복사량: {cow_size / 1024:.1f} KB")

    info_after = r.info('memory')
    cow_last = info_after.get('rdb_last_cow_size', 0)
    print(f"\nBGSAVE 완료")
    print(f"최종 CoW 복사량: {cow_last / 1024 / 1024:.2f} MB")
    print(f"쓰기가 많을수록 CoW 복사량 증가 → 메모리 이중화 위험")

if __name__ == '__main__':
    monitor_cow_memory()
```

`rdb_last_cow_size` 값이 크다는 것은 BGSAVE 중 부모 프로세스가 많은 페이지를 수정했음을 의미한다. 이 값이 전체 메모리의 50% 이상이면 메모리 부족(OOM)이 발생할 수 있다.

---

## 주의사항 및 팁

### THP(Transparent Huge Pages)와 CoW 폭탄

Linux 커널의 기본 페이지 크기는 4KB이지만, THP(Transparent Huge Pages)가 활성화되면 2MB 거대 페이지가 사용된다. CoW는 페이지 단위로 복사하므로, **1바이트만 수정해도 2MB 전체가 복사**된다. Redis 공식 문서에서 THP 비활성화를 강력히 권고하는 이유다.

```bash
# THP 확인
cat /sys/kernel/mm/transparent_hugepage/enabled

# THP 비활성화 (런타임)
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# 영구 적용: /etc/rc.local 또는 systemd 서비스에 추가
```

### vm.overcommit_memory 설정

Linux는 기본적으로 메모리 할당 시 **실제 가용 메모리**를 확인한다. CoW 특성상 fork 시 최악의 경우 메모리가 2배 필요할 수 있는데, 커널이 이를 보수적으로 판단해 `fork()` 자체를 실패시킬 수 있다.

```bash
# 현재 설정 확인
cat /proc/sys/vm/overcommit_memory
# 0: 휴리스틱 (기본), 1: 항상 허용, 2: 제한적 허용

# Redis 권장 설정 (항상 허용)
echo 1 > /proc/sys/vm/overcommit_memory
sysctl vm.overcommit_memory=1
```

### CoW가 도움이 되지 않는 경우

- **쓰기 집약적 BGSAVE**: BGSAVE 중 모든 키를 갱신하면 CoW 이득이 없고 메모리가 2배 필요
- **대규모 데이터 구조 변경**: Hash/List 수정 시 해당 페이지 전체가 복사됨
- **만료(TTL) 키 대량 처리**: 만료 시 키가 삭제되며 페이지가 지속적으로 CoW됨

**운영 팁**: Redis에서 BGSAVE 주기를 낮 트래픽 시간대로 설정하고, 메모리 여유분을 최소 50% 이상 확보해 두는 것이 핵심이다.

---

## 참고 자료
- [Redis Persistence — 공식 문서](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [Zalando Engineering: Understanding Redis Background Memory Usage](https://engineering.zalando.com/posts/2019/05/understanding-redis-background-memory-usage.html)
- [Linux man page: fork(2)](https://man7.org/linux/man-pages/man2/fork.2.html)
- [Linux Kernel: Copy-on-Write and Page Tables](https://www.kernel.org/doc/html/latest/mm/page_tables.html)
