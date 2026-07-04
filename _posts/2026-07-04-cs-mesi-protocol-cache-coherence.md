---
layout: post
title: "MESI 프로토콜과 캐시 일관성 완전 정복: 멀티코어 CPU가 데이터를 동기화하는 방법"
date: 2026-07-04
categories: [cs, computer-science]
tags: [cache-coherence, mesi, multicore, cpu, memory, hardware, concurrency, false-sharing]
---

## 개요: 캐시 일관성 문제란 무엇인가

현대 CPU는 메인 메모리(RAM)보다 수십~수백 배 빠른 **캐시(Cache)** 를 여러 계층으로 가지고 있습니다. 단일 코어 시대에는 이 캐시가 성능 향상만 가져왔지만, **멀티코어 시대**에는 전혀 새로운 문제가 발생합니다. 코어 0이 메모리 주소 `0x1000`의 값을 자신의 L1 캐시에 캐싱한 뒤 수정했는데, 코어 1도 같은 주소의 구버전 데이터를 자신의 캐시에 가지고 있다면 어떻게 될까요?

이것이 바로 **캐시 일관성(Cache Coherence)** 문제입니다. 여러 프로세서가 동일한 메모리 위치에 대해 서로 다른 값을 캐시하게 되면, 어느 코어가 '올바른' 데이터를 가지고 있는지 알 수 없게 됩니다. 이 문제를 해결하기 위해 하드웨어 수준에서 고안된 프로토콜이 바로 **MESI 프로토콜**입니다.

---

## 왜 MESI 프로토콜이 필요한가

### 초기 접근: 쓰기 무효화(Write-Invalidate) vs 쓰기 갱신(Write-Update)

캐시 일관성을 유지하는 방법에는 크게 두 가지 전략이 있습니다:

1. **Write-Invalidate**: 한 코어가 캐시 라인을 수정하면, 다른 모든 코어의 해당 캐시 라인을 '무효(Invalid)' 상태로 만든다. 다른 코어가 다음에 그 값을 읽으면 최신 값을 다시 가져온다.
2. **Write-Update (Write-Broadcast)**: 한 코어가 캐시 라인을 수정하면, 변경된 값을 즉시 모든 코어에 브로드캐스트해 동기화한다.

MESI는 Write-Invalidate 방식을 채택합니다. Write-Update는 브로드캐스트 트래픽이 과도하게 발생해 버스 대역폭을 낭비하기 때문입니다. 특히 연속으로 수정이 발생하는 경우 쓸모없는 중간 값들이 모두 전파되는 비효율이 생깁니다.

### 버스 스누핑(Bus Snooping)

MESI 프로토콜의 핵심 동작 메커니즘은 **버스 스누핑**입니다. 각 코어의 캐시 컨트롤러가 공유 버스를 '엿보면서(snoop)' 다른 코어의 메모리 접근 이벤트를 감지합니다. 어떤 코어가 특정 주소에 읽기나 쓰기를 수행하면, 그 트랜잭션이 버스에 브로드캐스트되고, 해당 주소를 캐싱하고 있는 다른 모든 코어가 자신의 캐시 상태를 업데이트합니다.

---

## MESI의 4가지 캐시 라인 상태

MESI는 각 캐시 라인(일반적으로 64바이트 블록)에 다음 네 가지 상태 중 하나를 부여합니다:

| 상태 | 약어 | 의미 | 메인 메모리와의 일치 | 다른 캐시 존재 여부 |
|------|------|------|---------------------|---------------------|
| **Modified** | M | 이 캐시가 유일하게 수정된 최신 복사본을 보유 | 불일치 (dirty) | 없음 |
| **Exclusive** | E | 이 캐시만 보유하는 클린 복사본 | 일치 (clean) | 없음 |
| **Shared** | S | 여러 캐시가 동일 값을 보유 | 일치 (clean) | 있음 |
| **Invalid** | I | 이 캐시 라인은 유효하지 않음 | 해당 없음 | 무관 |

### 상태 전이 다이어그램

```
                     [다른 코어가 같은 주소 읽기]
          M ─────────────────────────────────────→ S
          │                                         │
[라인 수정]│      [자신이 읽기 (캐시 미스)]           │[다른 코어가 쓰기]
          ↓            ↓                            ↓
          M ←── E ────→ S ──────────────────────→ I
               ↑    [다른 코어가 읽기]
               │
          [캐시 미스 후 자신만 읽기]
```

각 상태 전이 예시:
- **I → E**: 캐시 미스. 메모리에서 읽어오고 다른 캐시에 복사본이 없으면 Exclusive 상태로 진입.
- **E → M**: Exclusive 상태의 라인에 쓰기 수행. 버스 트랜잭션 없이 즉시 Modified로.
- **M → S**: 다른 코어가 같은 주소를 읽으려 할 때, Modified 코어가 메모리에 write-back 후 Shared로.
- **S → I**: 다른 코어가 쓰기(Read For Ownership, RFO) 수행 시, 나머지 캐시는 Invalid로.

---

## 실제 구현 예제

### 예제 1: False Sharing 문제와 해결 (Java)

**False Sharing**은 서로 다른 변수가 같은 캐시 라인에 들어 있을 때 발생하는 MESI 관련 성능 저하입니다. 코어 0이 변수 A를 수정하면 코어 1의 변수 B가 같은 캐시 라인에 있더라도 해당 라인 전체가 Invalid가 됩니다.

```java
import java.util.concurrent.*;

/**
 * False Sharing 성능 비교
 * 같은 캐시 라인(64바이트)에 서로 다른 카운터 변수가 위치하면
 * 코어 간 MESI 상태 전이가 과도하게 발생해 성능이 저하된다.
 */
public class FalseSharingDemo {

    // 나쁜 예: 두 카운터가 같은 캐시 라인에 위치할 가능성이 높음
    static long counterA = 0;
    static long counterB = 0;

    // 좋은 예: @Contended 어노테이션으로 캐시 라인 패딩 적용
    // (JVM 플래그: -XX:-RestrictContended 필요)
    @jdk.internal.vm.annotation.Contended
    static long paddedCounterA = 0;
    @jdk.internal.vm.annotation.Contended
    static long paddedCounterB = 0;

    static final long ITERATIONS = 100_000_000L;

    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== False Sharing 시연 ===");

        // 테스트 1: False Sharing 발생 가능
        long start = System.nanoTime();
        Thread t1 = new Thread(() -> {
            for (long i = 0; i < ITERATIONS; i++) counterA++;
        });
        Thread t2 = new Thread(() -> {
            for (long i = 0; i < ITERATIONS; i++) counterB++;
        });
        t1.start(); t2.start();
        t1.join(); t2.join();
        long withFalseSharing = System.nanoTime() - start;

        // 테스트 2: 패딩으로 False Sharing 방지
        start = System.nanoTime();
        Thread t3 = new Thread(() -> {
            for (long i = 0; i < ITERATIONS; i++) paddedCounterA++;
        });
        Thread t4 = new Thread(() -> {
            for (long i = 0; i < ITERATIONS; i++) paddedCounterB++;
        });
        t3.start(); t4.start();
        t3.join(); t4.join();
        long withoutFalseSharing = System.nanoTime() - start;

        System.out.printf("False Sharing 있음: %,d ms%n", withFalseSharing / 1_000_000);
        System.out.printf("False Sharing 없음: %,d ms%n", withoutFalseSharing / 1_000_000);
        System.out.printf("성능 차이: %.2fx%n",
                (double) withFalseSharing / withoutFalseSharing);
    }
}

// 수동 패딩 방식 (레거시 코드나 어노테이션 미사용 시)
class PaddedLong {
    // 총 8 + 56 = 64바이트 → 캐시 라인 1개 점유
    public volatile long value = 0L;
    public long p1, p2, p3, p4, p5, p6, p7; // 56바이트 패딩
}
```

### 예제 2: MESI 상태 시뮬레이터 (Python)

```python
from enum import Enum
from dataclasses import dataclass, field
from typing import Dict, List, Optional

class State(Enum):
    MODIFIED  = 'M'
    EXCLUSIVE = 'E'
    SHARED    = 'S'
    INVALID   = 'I'

@dataclass
class CacheLine:
    address: int
    data: Optional[int]
    state: State = State.INVALID

class Cache:
    def __init__(self, core_id: int, bus: 'Bus'):
        self.core_id = core_id
        self.bus = bus
        self.lines: Dict[int, CacheLine] = {}

    def _get_line(self, addr: int) -> CacheLine:
        if addr not in self.lines:
            self.lines[addr] = CacheLine(address=addr, data=None)
        return self.lines[addr]

    def read(self, addr: int) -> int:
        line = self._get_line(addr)

        if line.state in (State.MODIFIED, State.EXCLUSIVE, State.SHARED):
            print(f"  Core{self.core_id}: READ HIT  addr={addr:#x} state={line.state.value}")
            return line.data

        # 캐시 미스: 버스에 Read 트랜잭션 발행
        print(f"  Core{self.core_id}: READ MISS addr={addr:#x} — 버스 트랜잭션 발행")
        data, others_have_copy = self.bus.read(addr, self.core_id)
        line.data = data
        # 다른 캐시가 해당 라인을 가지면 Shared, 아니면 Exclusive
        line.state = State.SHARED if others_have_copy else State.EXCLUSIVE
        print(f"  Core{self.core_id}: 상태 → {line.state.value}, data={data}")
        return data

    def write(self, addr: int, data: int):
        line = self._get_line(addr)

        if line.state == State.MODIFIED:
            # 이미 독점 수정 상태, 버스 트랜잭션 불필요
            line.data = data
            print(f"  Core{self.core_id}: WRITE (M→M) addr={addr:#x} data={data}")
            return

        if line.state == State.EXCLUSIVE:
            # 독점 클린 상태 → 수정 상태로, 버스 트랜잭션 불필요
            line.data = data
            line.state = State.MODIFIED
            print(f"  Core{self.core_id}: WRITE (E→M) addr={addr:#x} data={data}")
            return

        # Shared 또는 Invalid: Read For Ownership(RFO) 발행
        print(f"  Core{self.core_id}: WRITE RFO  addr={addr:#x} — 다른 캐시 무효화 요청")
        self.bus.read_for_ownership(addr, self.core_id)
        line.data = data
        line.state = State.MODIFIED
        print(f"  Core{self.core_id}: 상태 → M, data={data}")

    def snoop_read(self, addr: int, requester_id: int) -> Optional[int]:
        """다른 코어의 Read 트랜잭션을 감지하고 상태 전이"""
        line = self._get_line(addr)
        if line.state == State.MODIFIED:
            print(f"  Core{self.core_id}: SNOOP READ 감지 — M→S, write-back 수행")
            # 메모리에 write-back 후 Shared로 전이
            self.bus.memory[addr] = line.data
            line.state = State.SHARED
            return line.data
        elif line.state == State.EXCLUSIVE:
            print(f"  Core{self.core_id}: SNOOP READ 감지 — E→S")
            line.state = State.SHARED
        return None

    def snoop_rfo(self, addr: int, requester_id: int):
        """다른 코어의 RFO를 감지하고 자신의 라인 무효화"""
        line = self._get_line(addr)
        if line.state != State.INVALID:
            if line.state == State.MODIFIED:
                self.bus.memory[addr] = line.data  # write-back
            print(f"  Core{self.core_id}: SNOOP RFO 감지 — {line.state.value}→I")
            line.state = State.INVALID

class Bus:
    def __init__(self, num_cores: int):
        self.memory: Dict[int, int] = {}
        self.caches: List[Cache] = []

    def attach(self, cache: Cache):
        self.caches.append(cache)

    def read(self, addr: int, requester_id: int):
        """버스 Read 트랜잭션: 다른 캐시에 스누핑"""
        data_from_cache = None
        others_have_copy = False
        for cache in self.caches:
            if cache.core_id == requester_id:
                continue
            result = cache.snoop_read(addr, requester_id)
            if result is not None:
                data_from_cache = result
            if cache.lines.get(addr, CacheLine(addr, None)).state != State.INVALID:
                others_have_copy = True
        # 메모리 또는 다른 캐시에서 데이터 반환
        data = data_from_cache if data_from_cache is not None else self.memory.get(addr, 0)
        return data, others_have_copy

    def read_for_ownership(self, addr: int, requester_id: int):
        """RFO 트랜잭션: 다른 캐시 전부 무효화"""
        for cache in self.caches:
            if cache.core_id != requester_id:
                cache.snoop_rfo(addr, requester_id)


# 시뮬레이션 실행
if __name__ == "__main__":
    bus = Bus(num_cores=3)
    cores = [Cache(i, bus) for i in range(3)]
    for c in cores:
        bus.attach(c)

    bus.memory[0x100] = 42  # 메인 메모리 초기값

    print("=== MESI 프로토콜 시뮬레이션 ===\n")

    print("[1] Core0이 0x100 읽기")
    cores[0].read(0x100)  # → Exclusive

    print("\n[2] Core1이 0x100 읽기")
    cores[1].read(0x100)  # Core0: E→S, Core1: → S

    print("\n[3] Core0이 0x100에 99 쓰기")
    cores[0].write(0x100, 99)  # RFO: Core1 I, Core0: S→M

    print("\n[4] Core2가 0x100 읽기")
    cores[2].read(0x100)  # Core0: M→S + write-back, Core2: → S
```

---

## 주요 파생 프로토콜

MESI의 한계를 보완하기 위해 여러 변형 프로토콜이 등장했습니다:

- **MESIF (Intel)**: Forward 상태 추가. Shared 상태에서 특정 캐시가 '대표'로 다른 캐시의 Read에 직접 응답 → 메모리 대역폭 절감.
- **MOESI (AMD)**: Owned 상태 추가. Modified 상태에서 다른 코어가 읽을 때, 메인 메모리에 write-back하지 않고도 다른 캐시와 공유 가능 → 메모리 트래픽 감소.
- **MESI + Directory**: 대규모 NUMA 시스템에서 버스 스누핑 대신 디렉터리(Directory)를 통해 어떤 코어가 어떤 캐시 라인을 보유하는지 추적.

---

## 주의사항과 실무 팁

### 1. False Sharing을 경계하라
스레드 로컬처럼 보이는 변수가 실제로는 같은 캐시 라인에 있을 수 있습니다. 배열에서 인접 인덱스를 서로 다른 스레드가 수정하는 패턴이 대표적입니다. Linux `perf` 도구의 `perf c2c` 서브커맨드로 False Sharing 핫스팟을 찾을 수 있습니다.

### 2. 메모리 가시성(Memory Visibility)과의 관계
MESI는 하드웨어 수준의 캐시 일관성을 보장하지만, 소프트웨어 수준의 **메모리 순서(Memory Ordering)** 는 별개입니다. CPU가 성능을 위해 명령어를 재배치(reorder)할 수 있으므로, Java의 `volatile`, C++의 `std::atomic`, 또는 메모리 펜스(Memory Fence) 명령어가 필요합니다.

### 3. 캐시 라인 크기를 항상 의식하라
x86 CPU의 캐시 라인은 일반적으로 **64바이트**입니다. 자주 함께 접근되는 데이터는 같은 캐시 라인에, 스레드마다 독립적으로 수정되는 데이터는 서로 다른 캐시 라인에 위치하도록 데이터 구조를 설계하는 것이 성능의 핵심입니다.

### 4. NUMA 환경에서는 더 복잡하다
멀티소켓 서버(NUMA 아키텍처)에서는 소켓 간 캐시 일관성 비용이 소켓 내부보다 훨씬 큽니다. 프로세스/스레드를 동일 NUMA 노드에 바인딩하는 것이 중요합니다.

---

## 정리

MESI 프로토콜은 멀티코어 CPU가 메모리 일관성을 유지하는 핵심 하드웨어 메커니즘입니다. Modified, Exclusive, Shared, Invalid 네 가지 상태와 버스 스누핑을 통해 어떤 코어가 최신 데이터를 가지고 있는지 하드웨어가 자동으로 추적합니다. 개발자는 이 원리를 이해함으로써 False Sharing 같은 미묘한 성능 저하 원인을 찾아내고, 캐시 친화적인 데이터 구조를 설계할 수 있습니다.

## 참고 자료
- [java17-mesi-false-sharing-processor-optimisations-workshop (GitHub)](https://github.com/mtumilowicz/java17-mesi-false-sharing-processor-optimisations-workshop)
- [CacheCoherenceProtocols: MESI, MOSI, MOESI 구현 (GitHub)](https://github.com/sahilgupta5/CacheCoherenceProtocols)
