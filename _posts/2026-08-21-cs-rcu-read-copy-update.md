---
layout: post
title: "RCU(Read-Copy-Update) 완전 정복: 리눅스 커널이 락 없이 독자를 처리하는 동시성 기법"
date: 2026-08-21
categories: [cs, computer-science]
tags: [concurrency, RCU, Linux-kernel, synchronization, lock-free, 동시성, 커널]
---

## 개념 설명: 읽기를 위한 락은 너무 비싸다

멀티코어 시스템에서 공유 데이터를 안전하게 읽으려면 어떻게 해야 할까? 가장 단순한 답은 **뮤텍스(mutex)** 다. 하지만 뮤텍스는 읽기 스레드가 아무리 많아도 하나씩 통과시킨다. 이를 개선한 **읽기-쓰기 락(rwlock)** 은 다수의 독자를 동시에 허용하지만, 여전히 `rw_lock_rdlock()` 호출 시 원자적 카운터 증가(atomic increment)가 필요하고, 쓰기 락 대기 중에는 독자도 블록된다.

Linux 커널은 2002년 또 다른 접근을 채택했다. **RCU(Read-Copy-Update)** 다. RCU는 읽기 측에서 **아무런 원자 연산도, 메모리 배리어도, 락도 없이** 데이터를 읽을 수 있게 한다. 대신 쓰기 측이 모든 복잡성을 감당한다.

이름에 알고리즘의 핵심이 담겨 있다:
- **Read**: 독자는 현재 버전의 데이터를 락 없이 읽는다.
- **Copy**: 쓰는 측은 데이터를 복사해 새 버전을 준비한다.
- **Update**: 포인터를 원자적으로 교체해 새 버전을 공개하고, 이전 버전은 모든 독자가 빠져나간 뒤에 해제한다.

---

## 왜 필요한가?

### 읽기 집약적 워크로드의 현실

Linux 커널의 많은 데이터 구조는 읽기가 쓰기보다 압도적으로 많다.

| 데이터 구조 | 읽기:쓰기 비율 |
|-------------|----------------|
| 라우팅 테이블 | ~1,000,000:1 |
| 파일 시스템 inode 캐시 | ~100,000:1 |
| 네트워크 디바이스 리스트 | ~10,000:1 |
| 모듈 리스트 | ~1,000:1 |

이런 환경에서 rwlock을 쓰면 독자가 매번 카운터를 원자적으로 업데이트하므로 캐시 라인을 계속 무효화(cacheline invalidation)한다. 코어가 수백 개인 서버에서는 이 오버헤드만으로도 심각한 성능 병목이 된다.

### RCU vs rwlock 성능 비교

읽기 측 임계 구역 진입 비용:

| 동기화 기법 | 아키텍처 | 비용 |
|-------------|----------|------|
| rwlock rdlock | x86 | ~60 사이클 |
| RCU read_lock | x86 (선점 없는 커널) | **0 사이클** |
| RCU read_lock | x86 (선점 있는 커널) | ~5 사이클 |

"0 사이클"은 과장이 아니다. `CONFIG_PREEMPTION=n` 커널에서 `rcu_read_lock()`은 문자 그대로 코드를 생성하지 않는다.

---

## RCU의 핵심 메커니즘

### 1. Grace Period (유예 기간)

RCU가 쓰기 측에서 보장해야 하는 핵심 불변식은 **"이전 버전 데이터를 참조하는 독자가 모두 임계 구역을 빠져나갈 때까지 해제를 미룬다"** 는 것이다. 이 대기 기간을 **Grace Period** 라고 부른다.

```
시간 →

독자 A: ──[rcu_read_lock]───────────────[rcu_read_unlock]──
독자 B:           ──[rcu_read_lock]──[rcu_read_unlock]──
쓰기:   ──[데이터 복사·수정]──[포인터 교체]──[synchronize_rcu()]──[old 해제]
                                           │←── Grace Period ──→│
```

`synchronize_rcu()`는 현재 진행 중인 모든 RCU 임계 구역이 완료될 때까지 블록한다. 이후 안전하게 구버전 데이터를 해제할 수 있다.

### 2. 정족수 상태(Quiescent State)

각 CPU가 RCU 임계 구역 밖에 있는 상태를 **정족수 상태(Quiescent State)** 라고 한다. 모든 CPU가 한 번씩 정족수 상태를 통과하면 Grace Period가 끝난다.

선점 없는 커널에서는 컨텍스트 스위치(context switch)가 정족수 상태의 증거다. CPU가 스케줄러를 실행한다는 것은 그 CPU가 RCU 임계 구역 밖에 있다는 뜻이기 때문이다.

### 3. call_rcu(): 비동기 콜백

`synchronize_rcu()`가 블로킹 방식이라면, `call_rcu()`는 콜백을 등록하고 즉시 반환하는 비동기 방식이다. Grace Period 이후 커널이 자동으로 콜백을 호출한다.

```c
// 쓰기 측: 비동기 방식으로 구버전 해제
call_rcu(&old_node->rcu_head, free_node_callback);
```

---

## 실제 구현 예시

### 구현 1: 커널 스타일 RCU 패턴 (C 유사 의사 코드)

실제 Linux 커널 코드와 유사한 패턴으로 RCU 보호 연결 리스트에서 노드를 조회·교체하는 예를 보인다.

```c
#include <linux/rcupdate.h>
#include <linux/slab.h>

/* RCU로 보호되는 전역 포인터 */
struct my_data {
    int value;
    struct rcu_head rcu;  // RCU 콜백 헤드
};

static struct my_data __rcu *global_data;

/* ── 읽기 측 ─────────────────────────────────────────── */
void reader_function(void)
{
    struct my_data *data;

    rcu_read_lock();              // 선점 금지 (선점 커널) 또는 no-op
    data = rcu_dereference(global_data);  // 메모리 배리어 포함 역참조
    if (data)
        pr_info("value: %d\n", data->value);
    rcu_read_unlock();            // 독자 임계 구역 종료
    // data 사용 금지! rcu_read_unlock 이후 해제될 수 있음
}

/* ── 쓰기 측 ─────────────────────────────────────────── */
static void free_old_data(struct rcu_head *head)
{
    struct my_data *data = container_of(head, struct my_data, rcu);
    kfree(data);  // Grace Period 이후 안전하게 해제
}

void writer_function(int new_value)
{
    struct my_data *new_data, *old_data;

    /* 1. 새 버전 할당 및 초기화 (락 없이 가능) */
    new_data = kmalloc(sizeof(*new_data), GFP_KERNEL);
    new_data->value = new_value;

    /* 2. 구버전 포인터 저장 */
    old_data = rcu_dereference_protected(global_data,
                                          lockdep_is_held(&my_mutex));

    /* 3. 포인터 원자적 교체 — 이 순간부터 독자는 새 버전을 본다 */
    rcu_assign_pointer(global_data, new_data);

    /* 4. Grace Period 대기 후 구버전 해제 (비동기 방식) */
    if (old_data)
        call_rcu(&old_data->rcu, free_old_data);
}
```

핵심 API:
- `rcu_read_lock()` / `rcu_read_unlock()`: 독자 임계 구역 표시
- `rcu_dereference()`: 포인터 안전 역참조 (메모리 배리어 포함)
- `rcu_assign_pointer()`: 포인터 안전 교체 (쓰기 메모리 배리어 포함)
- `synchronize_rcu()` / `call_rcu()`: Grace Period 대기 또는 비동기 콜백

### 구현 2: 사용자 공간에서 RCU 개념 시뮬레이션 (C++)

커널 없이 사용자 공간에서 RCU의 핵심 아이디어를 재현할 수 있다. 에폭(epoch) 기반 메모리 회수가 대표적인 방법이다.

```cpp
#include <atomic>
#include <memory>
#include <vector>
#include <thread>
#include <iostream>

// ─── 사용자 공간 RCU 시뮬레이션: 에폭(Epoch) 기반 ──────────────

template <typename T>
class RCUProtected {
    std::atomic<T*> ptr_;
    std::vector<T*> retired_;   // Grace Period 이후 해제할 구버전 목록
    std::atomic<int> readers_{0};

public:
    explicit RCUProtected(T* initial) : ptr_(initial) {}

    // ── 독자 측 ──────────────────────────────────────────────
    class ReadGuard {
        RCUProtected& rcu_;
    public:
        explicit ReadGuard(RCUProtected& rcu) : rcu_(rcu) {
            rcu_.readers_.fetch_add(1, std::memory_order_acquire);
        }
        ~ReadGuard() {
            rcu_.readers_.fetch_sub(1, std::memory_order_release);
        }
        T* get() {
            // memory_order_consume/acquire로 포인터 로드
            return rcu_.ptr_.load(std::memory_order_acquire);
        }
    };

    ReadGuard read_lock() { return ReadGuard(*this); }

    // ── 쓰기 측 ──────────────────────────────────────────────
    void update(T* new_val) {
        T* old = ptr_.exchange(new_val, std::memory_order_acq_rel);
        // 간이 Grace Period: 모든 독자가 빠져나갈 때까지 스핀
        // 실제 커널은 컨텍스트 스위치 기반으로 더 효율적으로 처리
        while (readers_.load(std::memory_order_acquire) > 0)
            std::this_thread::yield();
        delete old;  // Grace Period 완료 → 안전 해제
    }

    ~RCUProtected() {
        delete ptr_.load();
    }
};

// ── 사용 예시 ────────────────────────────────────────────────────
struct Config {
    int timeout;
    std::string endpoint;
};

int main() {
    RCUProtected<Config> rcu_config(new Config{30, "api.example.com"});

    // 독자 스레드 (수백 개 동시 실행 가능, 락 경합 없음)
    auto reader = [&]() {
        for (int i = 0; i < 100; ++i) {
            auto guard = rcu_config.read_lock();
            Config* cfg = guard.get();
            // cfg를 안전하게 읽는다 — 이 블록 안에서만 유효
            std::cout << "timeout=" << cfg->timeout << "\n";
        }
    };

    // 쓰기 스레드 (일반적으로 드물게 실행)
    auto writer = [&]() {
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
        rcu_config.update(new Config{60, "api-v2.example.com"});
        std::cout << "Config updated\n";
    };

    std::thread t1(reader), t2(reader), t3(writer);
    t1.join(); t2.join(); t3.join();
    return 0;
}
```

이 구현은 설명 목적이며 실제 생산 환경에서는 [liburcu](https://liburcu.org) 같은 검증된 사용자 공간 RCU 라이브러리를 사용하라.

---

## RCU의 변형들

### Sleepable RCU (SRCU)
기본 RCU는 독자 임계 구역 내에서 슬립(sleep)이 불가능하다. SRCU는 독자 임계 구역 내 슬립을 허용하는 대신 더 많은 비용을 지불한다.

```c
DEFINE_SRCU(my_srcu);

// 독자
int idx = srcu_read_lock(&my_srcu);
/* 슬립 가능 */
srcu_read_unlock(&my_srcu, idx);

// 쓰기
synchronize_srcu(&my_srcu);
```

### Tasks RCU
`task_struct` 순회를 위한 특수 RCU. 사용자 공간 태스크의 컨텍스트 스위치를 정족수 상태로 활용한다.

### Tiny RCU
임베디드 단일 CPU 환경을 위한 최소 구현. Grace Period가 단순히 `schedule()` 호출 한 번으로 완료된다.

---

## 주의사항 및 팁

### 1. 독자 임계 구역 내 포인터 유출 금지
`rcu_read_unlock()` 이후에는 RCU 보호 포인터를 절대 역참조하면 안 된다. 이미 해제되었을 수 있다.

```c
// 잘못된 사용
struct data *d;
rcu_read_lock();
d = rcu_dereference(global);
rcu_read_unlock();
printk("%d\n", d->value);  // 위험! d가 이미 kfree 되었을 수 있음
```

### 2. rcu_dereference를 반드시 사용
컴파일러 최적화로 인해 포인터 로드가 분리될 수 있다. `rcu_dereference()`는 내부적으로 `READ_ONCE()`와 메모리 배리어를 조합해 이를 방지한다.

### 3. 쓰기 측 직렬화는 별도로
RCU는 읽기-쓰기 경쟁만 해결한다. 복수의 쓰기 스레드 간 경쟁은 별도의 뮤텍스로 직렬화해야 한다.

### 4. Grace Period 비용 과소평가 금지
`synchronize_rcu()`는 밀리초 단위의 지연을 일으킬 수 있다. 고빈도 업데이트가 필요하다면 `call_rcu()` + 배칭(batching)으로 Grace Period 횟수를 줄여야 한다.

### 5. rwlock과 비교해 언제 RCU를 선택할까?
- 읽기가 쓰기보다 훨씬 많다: **RCU 적합**
- 임계 구역이 짧다: **RCU 적합**
- 쓰기 지연이 허용된다: **RCU 적합**
- 메모리가 극도로 부족하다: 구버전 보관 비용 때문에 **rwlock 고려**
- 읽기와 쓰기 비율이 균등하다: **rwlock 고려**

---

## 참고 자료

- [What is RCU? – "Read, Copy, Update" — The Linux Kernel documentation](https://docs.kernel.org/RCU/whatisRCU.html)
- [RCU Concepts — The Linux Kernel documentation](https://docs.kernel.org/RCU/rcu.html)
- [What is RCU, Fundamentally? — LWN.net](https://lwn.net/Articles/262464/)
- [A Tour Through RCU's Requirements — The Linux Kernel documentation](https://docs.kernel.org/6.5/RCU/Design/Requirements/Requirements.html)
