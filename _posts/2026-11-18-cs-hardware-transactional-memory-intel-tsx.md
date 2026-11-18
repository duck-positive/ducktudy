---
layout: post
title: "하드웨어 트랜잭셔널 메모리(HTM) 완전 정복: Intel TSX와 낙관적 동시성의 하드웨어 지원"
date: 2026-11-18
categories: [cs, computer-science]
tags: [concurrency, hardware, transactional-memory, intel-tsx, cpu-architecture, synchronization]
---

멀티코어 프로세서 시대에 공유 메모리 동시성은 소프트웨어 개발의 가장 어려운 문제 중 하나입니다. 뮤텍스 락은 정확하지만 비관적(pessimistic)이라 실제 충돌이 없어도 직렬화합니다. **하드웨어 트랜잭셔널 메모리(Hardware Transactional Memory, HTM)**는 이 딜레마를 하드웨어 레벨에서 해결하려는 시도입니다. 프로세서가 직접 낙관적 동시성 제어를 수행하여, 충돌이 없으면 락 없이 커밋하고 충돌 시에만 롤백합니다.

## 트랜잭셔널 메모리의 개념

### 소프트웨어 트랜잭셔널 메모리(STM)의 선행

트랜잭셔널 메모리 아이디어는 1993년 Herlihy와 Moss의 논문에서 처음 제안되었습니다. 데이터베이스 트랜잭션과 동일한 ACID 의미를 메모리 접근에 적용하자는 것입니다.

```
트랜잭션 시작 (xbegin)
  → 메모리 읽기/쓰기를 투기적으로 실행
  → 충돌 없음? → 커밋 (xend) — 원자적으로 모든 변경 적용
  → 충돌 있음? → 중단 (abort) — 모든 변경 폐기, 롤백
```

**낙관적 가정**: 대부분의 경우 스레드들이 같은 메모리 위치에 동시에 접근하지 않는다는 가정 하에, 미리 직렬화하지 않고 실행 후 충돌 여부를 판단합니다.

### HTM vs STM

| 측면 | 소프트웨어 TM (STM) | 하드웨어 TM (HTM) |
|---|---|---|
| 구현 위치 | 런타임/컴파일러 | 프로세서 하드웨어 |
| 오버헤드 | 상당히 큼 (write barrier 등) | 거의 없음 (하드웨어 처리) |
| 트랜잭션 크기 | 이론상 무제한 | L1 캐시 크기 제약 |
| 중단 이유 | 프로그램 제어 가능 | 하드웨어 이유로 비결정적 중단 가능 |
| 활용 | Haskell STM, Clojure ref | Intel TSX, ARM TME, IBM POWER |

## Intel TSX: x86의 HTM 구현

Intel은 Haswell 아키텍처(2013년)에서 **Transactional Synchronization eXtensions(TSX)**를 도입했습니다. TSX는 두 가지 인터페이스를 제공합니다:

### 1. Restricted Transactional Memory (RTM)

RTM은 명시적 트랜잭션 경계를 제공하는 프로그래머 친화적 인터페이스입니다:

```c
// RTM 명령어:
// XBEGIN <abort_handler>  — 트랜잭션 시작, 중단 시 abort_handler로 점프
// XEND                     — 트랜잭션 커밋 시도
// XABORT <code>            — 명시적 중단 (8비트 코드)
// XTEST                    — 트랜잭션 실행 중 여부 확인
```

**내부 메커니즘**: 트랜잭션 실행 중 프로세서는 읽기/쓰기한 캐시 라인들을 `read-set`과 `write-set`으로 추적합니다. 다른 프로세서가 write-set의 캐시 라인을 수정하거나, read-set의 캐시 라인을 무효화하면 충돌로 감지되어 트랜잭션이 중단됩니다.

### 2. Hardware Lock Elision (HLE)

HLE는 기존 `lock` 접두사를 그대로 사용하면서 투기적으로 락을 제거합니다. 기존 코드와 완전 하위 호환됩니다:

```asm
; 기존 코드: lock xchg [lock_addr], rax
; HLE로 변환: xacquire lock xchg [lock_addr], rax
;            (내부적으로 트랜잭션 실행 시도, 실패 시 실제 락 획득)
```

## 실전 코드 예제

### 예제 1: RTM으로 카운터 동시성 최적화 (C/C++)

```cpp
#include <immintrin.h>  // Intel Intrinsics (TSX)
#include <atomic>
#include <iostream>

// TSX RTM을 사용한 낙관적 카운터 증가
class TransactionalCounter {
    long long count = 0;
    int lock = 0;  // 폴백 락

    static constexpr int MAX_RETRIES = 10;

public:
    void increment() {
        for (int attempt = 0; attempt < MAX_RETRIES; attempt++) {
            unsigned int status = _xbegin();

            if (status == _XBEGIN_STARTED) {
                // === 트랜잭션 영역 시작 ===
                if (lock != 0) {
                    // 다른 스레드가 폴백 락을 보유 중 → 명시적 중단
                    _xabort(0xFF);
                }
                count++;  // 투기적 실행 (실제론 아직 커밋 안 됨)
                _xend();  // 커밋 시도
                return;   // 성공
                // === 트랜잭션 영역 끝 ===
            }

            // 트랜잭션 중단됨
            // status 비트 플래그로 중단 이유 분석:
            if (status & _XABORT_EXPLICIT) {
                // _xabort()로 명시적 중단 → 폴백으로 직행
                break;
            }
            if (status & _XABORT_RETRY) {
                // 재시도 권장 → 다음 루프
                continue;
            }
            // 용량 초과, 디버그 등 → 폴백
            break;
        }

        // 폴백 경로: 일반 mutex 사용
        while (__atomic_test_and_set(&lock, __ATOMIC_ACQUIRE)) {
            // spin wait
        }
        count++;  // 비트랜잭셔널 실행
        __atomic_clear(&lock, __ATOMIC_RELEASE);
    }

    long long get() const { return count; }
};

// TSX 지원 여부 확인
bool tsx_supported() {
    unsigned int eax, ebx, ecx, edx;
    __cpuid_count(7, 0, eax, ebx, ecx, edx);
    return (ebx >> 11) & 1;  // RTM: bit 11 of EBX
}

int main() {
    if (!tsx_supported()) {
        std::cout << "TSX 미지원 환경" << std::endl;
        return 1;
    }

    TransactionalCounter counter;

    // 간단한 단일 스레드 검증
    for (int i = 0; i < 1000; i++) {
        counter.increment();
    }
    std::cout << "최종 카운터: " << counter.get() << std::endl;
    // 출력: 최종 카운터: 1000
    return 0;
}
// 컴파일: g++ -O2 -mrtm -o tsx_demo tsx_demo.cpp
```

### 예제 2: Java에서의 개념적 HTM 시뮬레이션과 STM 비교

Java에서 HTM을 직접 사용할 수는 없지만, JVM 내부적으로 `synchronized` 블록에 HTM을 활용하는 최적화(Restricted Transactional Memory 기반 biased locking 대체)가 구현되어 있습니다. 아래는 STM 방식으로 HTM 의미를 표현한 예제입니다:

```java
import java.util.concurrent.atomic.AtomicReference;
import java.util.function.UnaryOperator;

// 소프트웨어로 HTM 의미를 흉내 낸 간단한 STM 구현
// (실제 HTM은 하드웨어가 이 모든 과정을 O(1)에 처리)
public class SimpleSTMDemo {

    // STM 트랜잭션 가능한 Ref
    static class TRef<T> {
        private final AtomicReference<T> value;
        private volatile long version;

        TRef(T initial) {
            this.value = new AtomicReference<>(initial);
            this.version = 0;
        }

        // 읽기: 현재 값과 버전 번호 캡처
        T read(Transaction tx) {
            tx.readSet.put(this, version);
            return value.get();
        }

        // 쓰기: write-set에 기록 (커밋 전까지는 실제로 쓰지 않음)
        @SuppressWarnings("unchecked")
        void write(Transaction tx, T newVal) {
            tx.writeSet.put(this, newVal);
        }
    }

    static class Transaction {
        final java.util.Map<TRef<?>, Long> readSet = new java.util.HashMap<>();
        final java.util.Map<TRef<?>, Object> writeSet = new java.util.HashMap<>();

        // 커밋: read-set 충돌 검사 + write-set 원자적 적용
        @SuppressWarnings("unchecked")
        boolean commit() {
            // 1단계: read-set 검증 (버전 충돌 확인)
            for (var entry : readSet.entrySet()) {
                if (entry.getKey().version != entry.getValue()) {
                    return false;  // 충돌 → 중단
                }
            }
            // 2단계: write-set 적용 (원자적이지 않음 — 실제 STM은 더 복잡)
            for (var entry : writeSet.entrySet()) {
                TRef<Object> ref = (TRef<Object>) entry.getKey();
                ref.value.set(entry.getValue());
                ref.version++;
            }
            return true;  // 커밋 성공
        }
    }

    // 트랜잭션 실행 헬퍼
    static <T> T atomically(java.util.function.Function<Transaction, T> body) {
        while (true) {
            Transaction tx = new Transaction();
            try {
                T result = body.apply(tx);
                if (tx.commit()) return result;
                // 충돌 → 재시도 (HTM은 하드웨어가 자동 재시도)
            } catch (Exception e) {
                // 중단 → 재시도
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        TRef<Integer> balance1 = new TRef<>(1000);
        TRef<Integer> balance2 = new TRef<>(500);

        // 계좌 이체: balance1에서 balance2로 300 이체
        // 두 업데이트가 원자적으로 발생
        atomically(tx -> {
            int from = balance1.read(tx);
            int to   = balance2.read(tx);
            balance1.write(tx, from - 300);
            balance2.write(tx, to   + 300);
            return null;
        });

        // 결과 확인
        System.out.println("balance1: " + balance1.value.get()); // 700
        System.out.println("balance2: " + balance2.value.get()); // 800
    }
}
```

## HTM의 중단 원인과 한계

### 중단(Abort)이 발생하는 주요 원인

1. **캐시 충돌**: 두 스레드의 read/write-set이 겹치는 경우
2. **L1 캐시 용량 초과**: 트랜잭션이 접근하는 데이터가 L1 캐시를 초과하면 하드웨어가 강제 중단
3. **I/O 명령어**: 시스템 콜, I/O 작업은 트랜잭션 내에서 불가
4. **캐시 라인 거짓 공유(False Sharing)**: 논리적으로 무관한 데이터가 같은 캐시 라인에 있으면 불필요한 충돌 발생
5. **디버그/프로파일링 도구**: 하드웨어 브레이크포인트 사용 시 트랜잭션 중단
6. **TSX 보안 취약점(TSX Asynchronous Abort)**: 2019년에 발견된 보안 취약점(CVE-2019-11135)으로, 많은 시스템에서 마이크로코드 업데이트로 TSX가 비활성화됨

### 현재 상황: TSX의 미래

Intel은 2021년 여러 Xeon/Core 프로세서에서 TSX Asynchronous Abort(TAA) 취약점 완화를 위해 마이크로코드로 RTM을 비활성화했습니다. 현재 많은 서버 환경에서 TSX를 사용할 수 없는 상태입니다.

대안:
- **ARM TME (Transactional Memory Extension)**: ARMv9에 포함된 TM 확장, 현재 활성화됨
- **IBM POWER10**: HTM을 지원하는 서버 아키텍처
- **소프트웨어 TM**: Haskell의 STM, Clojure의 ref/transaction

```c
// TSX 사용 가능 여부 런타임 확인
#include <cpuid.h>

bool is_rtm_available() {
    unsigned int eax, ebx, ecx, edx;
    if (__get_cpuid_count(7, 0, &eax, &ebx, &ecx, &edx)) {
        bool rtm_supported = (ebx >> 11) & 1;
        // /proc/cpuinfo 또는 CPUID leaf 7의 RTM_ALWAYS_ABORT 비트도 확인해야 함
        return rtm_supported;
    }
    return false;
}
```

## 마이크로아키텍처 수준의 구현

HTM은 L1 캐시와 완전히 통합되어 동작합니다:

- **Read-set 추적**: 읽은 캐시 라인에 "TX-read" 태그 → 다른 코어가 이 라인을 쓰려 하면 충돌
- **Write-set 버퍼링**: 쓴 내용은 캐시에만 있고 메모리에 커밋 안 됨 → 중단 시 단순히 태그만 제거
- **커밋 원자성**: `XEND` 실행 시 단일 사이클에 write-set을 메모리에 반영 (캐시 라인 단위)

HTM은 "충돌이 적은" 자료구조(해시 맵, 연결 리스트 등)에서 뮤텍스 대비 큰 성능 향상을 보입니다. 하지만 충돌이 많거나 트랜잭션이 큰 경우 오히려 오버헤드가 증가합니다.

## 참고 자료
- [Transactional memory - Wikipedia](https://en.wikipedia.org/wiki/Transactional_memory)
- [Intel TSX Programming Considerations - Intel Developer Guide](https://www.intel.com/content/www/us/en/docs/cpp-compiler/developer-guide-reference/2021-8/tsx-programming-considerations-ov.html)
- [Transactional Synchronization Extensions Overview - Intel](https://www.physics.ntua.gr/~konstant/HetCluster/intel2021.7/HPC/cpp_compiler/cpp_compiler_classic_dev_guide/GUID-FB2F2539-18F5-4D5A-B814-F29FD0C32326.html)
- [Making Sense of Intel Haswell TSX - AnandTech](https://www.anandtech.com/show/6290/making-sense-of-intel-haswell-transactional-synchronization-extensions)
