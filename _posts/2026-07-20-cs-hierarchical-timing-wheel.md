---
layout: post
title: "계층적 타이밍 휠(Hierarchical Timing Wheel) 완전 정복: Kafka·Netty·Linux 커널의 O(1) 타이머 구현"
date: 2026-07-20
categories: [cs, computer-science]
tags: [timing-wheel, timer, data-structure, kafka, netty, linux-kernel, scheduler, O1]
---

수백만 개의 타이머를 동시에 관리해야 하는 시스템이 있다면 어떻게 할까요? 소켓 타임아웃, 재전송 타이머, 세션 만료, 딜레이 큐 — 실시간 서버에서 타이머는 수십만 개가 동시에 살아있을 수 있습니다. 단순한 우선순위 큐(힙)는 O(log N)의 삽입/삭제 비용 때문에 N이 수백만에 달하면 병목이 됩니다. **계층적 타이밍 휠(Hierarchical Timing Wheel)** 은 이 문제를 **O(1) 삽입, O(1) 삭제**로 해결합니다. 1987년 Varghese와 Lauck이 제안한 이 자료구조는 Kafka 퍼거토리(Purgatory), Netty의 `HashedWheelTimer`, Linux 커널의 `timer_list`, 그리고 Go의 runtime timer에 이르기까지 현대 시스템의 핵심 구성 요소입니다.

## 개념 설명

### 단순 타이밍 휠 (Simple Timing Wheel)

시계의 초침을 상상해 보세요. 원형 배열(circular array)로 표현된 슬롯이 있고, 현재 시각을 가리키는 포인터가 매 틱(tick)마다 한 칸씩 이동합니다. 타이머를 등록할 때는 `(현재 슬롯 + 지연 시간) % 슬롯 수` 위치에 타이머를 연결 리스트로 이어붙입니다. 포인터가 슬롯에 도착할 때 해당 슬롯의 타이머들을 실행합니다.

```
슬롯:  [0] [1] [2] [3] [4] [5] [6] [7]
             ↑ 현재 포인터 (tick=1)

"3틱 후" 타이머 → 슬롯 (1+3)%8 = 4에 삽입
"10틱 후" 타이머 → 슬롯 (1+10)%8 = 3에 삽입, round=1 표시
```

문제는 **라운드 수** 처리입니다. 슬롯이 8개일 때 "10틱 후" 타이머는 슬롯 3에 들어가지만, 포인터가 3에 도착했을 때 아직 1라운드 남았다면 실행하면 안 됩니다. 라운드 카운터를 함께 저장하면 해결되지만, 포인터가 슬롯에 도착할 때마다 연결 리스트를 순회해야 하므로 최악의 경우 O(N)이 됩니다.

### 계층적 타이밍 휠 (Hierarchical Timing Wheel)

시, 분, 초처럼 **여러 단계의 휠을 계층화**합니다. 각 레벨의 해상도(resolution)와 범위(range)가 다릅니다.

```
레벨 1 (초): 슬롯 60개, 해상도 1초    → 범위 0~59초
레벨 2 (분): 슬롯 60개, 해상도 60초   → 범위 0~3599초
레벨 3 (시): 슬롯 24개, 해상도 3600초 → 범위 0~86399초
```

타이머를 삽입할 때:
- 만료까지 1분 미만이면 → 레벨 1의 해당 슬롯에 삽입
- 만료까지 1시간 미만이면 → 레벨 2의 해당 슬롯에 삽입
- 만료까지 1일 미만이면 → 레벨 3의 해당 슬롯에 삽입

레벨 2의 포인터가 한 칸 전진하면 그 슬롯의 타이머들을 레벨 1으로 **재삽입(cascade)** 합니다. 이렇게 계층을 타고 내려오다가 레벨 1에서 실제 실행됩니다.

삽입 비용은 O(m) — m은 레벨 수 (보통 2~5). 삭제는 연결 리스트에서 제거이므로 포인터를 저장하면 O(1). 만료 처리는 슬롯 당 평균 O(1) (균등 분포 가정).

### Kafka의 DelayQueue 통합

Kafka의 타이밍 휠은 약간 다른 전략을 씁니다. 각 레벨의 휠에서 **사용 중인 슬롯만** Java의 `DelayQueue`에 등록합니다. 별도 스레드가 DelayQueue를 모니터링하다가 가장 임박한 슬롯의 만료 시각에 깨어나 포인터를 전진시킵니다. 이 방식의 장점은 **아무 타이머도 만료되지 않는 구간에는 CPU를 전혀 쓰지 않는다**는 것입니다 — 일반 타이밍 휠의 고정 interval 폴링과 다릅니다.

---

## 왜 필요한가

### 힙 기반 타이머의 한계

Java의 `ScheduledExecutorService`나 Python의 `heapq`를 이용한 타이머 큐는 힙(heap)을 사용합니다.

| 연산       | 힙         | 타이밍 휠   |
|-----------|-----------|-----------|
| 삽입       | O(log N)  | O(1)      |
| 삭제       | O(log N)  | O(1)*     |
| 만료 처리  | O(log N)  | O(1) 평균 |

N = 100만 타이머, 초당 100만 삽입/삭제가 발생하면 힙은 초당 약 2천만 번의 비교 연산이 필요합니다. 타이밍 휠은 이를 O(1)에 처리합니다.

### Kafka Purgatory: 수백만 In-Flight 요청 타임아웃

Kafka 브로커에서 `acks=all`로 설정된 ProducerRequest는 모든 레플리카가 복제를 완료할 때까지 퍼거토리(Purgatory, 연옥)에서 대기합니다. 초당 수십만 건의 요청이 수십 밀리초에서 수 초 사이에 타임아웃될 수 있으므로, O(log N) 힙으로는 감당이 어렵습니다.

---

## 실제 구현 예제

### 예제 1 — Python: 단순 단계 타이밍 휠 구현

```python
from collections import defaultdict
from typing import Callable, Optional
import time

class TimerTask:
    def __init__(self, expiration: int, callback: Callable):
        self.expiration = expiration  # 절대 틱 시각
        self.callback = callback
        self.cancelled = False
        # 연결 리스트 포인터
        self._prev: Optional['TimerTask'] = None
        self._next: Optional['TimerTask'] = None
        self._bucket: Optional['TimerBucket'] = None

    def cancel(self) -> bool:
        if self._bucket:
            self._bucket.remove(self)
            self.cancelled = True
            return True
        return False

class TimerBucket:
    """타이밍 휠의 슬롯 하나 — 이중 연결 리스트"""
    def __init__(self):
        self.expiration = -1L if hasattr(int, 'bit_length') else -1
        self._root = TimerTask(-1, lambda: None)  # 더미 헤드
        self._root._next = self._root
        self._root._prev = self._root
        self.count = 0

    def add(self, task: TimerTask):
        last = self._root._prev
        task._prev = last
        task._next = self._root
        last._next = task
        self._root._prev = task
        task._bucket = self
        self.count += 1

    def remove(self, task: TimerTask):
        if task._bucket is not self:
            return
        task._prev._next = task._next
        task._next._prev = task._prev
        task._prev = task._next = None
        task._bucket = None
        self.count -= 1

    def flush(self, reinsert: Callable[['TimerTask'], None]):
        """슬롯의 모든 태스크를 만료/재삽입 처리"""
        task = self._root._next
        while task is not self._root:
            nxt = task._next
            self.remove(task)
            reinsert(task)
            task = nxt
        self.expiration = -1

class TimingWheel:
    """단일 레벨 타이밍 휠"""
    def __init__(self, tick_ms: int, wheel_size: int, start_ms: int,
                 overflow_wheel: Optional['TimingWheel'] = None):
        self.tick_ms = tick_ms
        self.wheel_size = wheel_size
        self.interval = tick_ms * wheel_size  # 이 레벨이 커버하는 총 시간
        self.current_time = (start_ms // tick_ms) * tick_ms  # 틱 정렬
        self.buckets = [TimerBucket() for _ in range(wheel_size)]
        self.overflow_wheel = overflow_wheel  # 상위 레벨

    def add(self, task: TimerTask) -> bool:
        expiration = task.expiration
        if expiration < self.current_time + self.tick_ms:
            # 이미 만료됐거나 즉시 만료 — 실행 대상
            return False
        elif expiration < self.current_time + self.interval:
            # 이 레벨에서 처리 가능
            ticks = expiration // self.tick_ms
            bucket = self.buckets[ticks % self.wheel_size]
            bucket.add(task)
            if bucket.expiration != (ticks * self.tick_ms):
                bucket.expiration = ticks * self.tick_ms
                # DelayQueue에 이 bucket의 만료 시각을 등록 (Kafka 방식 시뮬레이션)
            return True
        else:
            # 이 레벨 범위 초과 — 상위 레벨로 위임
            if self.overflow_wheel is None:
                raise OverflowError(f"타이머 범위 초과: {expiration}")
            return self.overflow_wheel.add(task)

    def advance_clock(self, time_ms: int):
        if time_ms >= self.current_time + self.tick_ms:
            self.current_time = (time_ms // self.tick_ms) * self.tick_ms
            if self.overflow_wheel:
                self.overflow_wheel.advance_clock(self.current_time)


class HierarchicalTimer:
    """계층적 타이밍 휠 매니저 (3레벨)"""
    def __init__(self, tick_ms: int = 1, wheel_size: int = 20):
        now = int(time.time() * 1000)
        # 레벨 1: 1ms 해상도, 20슬롯 → 20ms 커버
        # 레벨 2: 20ms 해상도, 20슬롯 → 400ms 커버
        # 레벨 3: 400ms 해상도, 20슬롯 → 8000ms 커버
        w3 = TimingWheel(tick_ms * wheel_size * wheel_size, wheel_size, now)
        w2 = TimingWheel(tick_ms * wheel_size, wheel_size, now, overflow_wheel=w3)
        self.w1 = TimingWheel(tick_ms, wheel_size, now, overflow_wheel=w2)
        self._expired: list = []
        self._tick_ms = tick_ms

    def schedule(self, delay_ms: int, callback: Callable) -> TimerTask:
        expiration = int(time.time() * 1000) + delay_ms
        task = TimerTask(expiration, callback)
        if not self.w1.add(task):
            # 즉시 만료 — 직접 실행 큐에 추가
            self._expired.append(task)
        return task

    def tick(self):
        now = int(time.time() * 1000)
        self.w1.advance_clock(now)
        # 만료된 버킷의 태스크를 재삽입하거나 실행
        def reinsert_or_run(task: TimerTask):
            if not task.cancelled:
                if not self.w1.add(task):
                    self._expired.append(task)
        # 실제 구현은 DelayQueue에서 만료된 버킷을 폴링하는 방식 사용
        for task in self._expired:
            if not task.cancelled:
                task.callback()
        self._expired.clear()


# 사용 예시
timer = HierarchicalTimer(tick_ms=1, wheel_size=20)

results = []
timer.schedule(5,  lambda: results.append("5ms 타이머"))
timer.schedule(15, lambda: results.append("15ms 타이머"))
t = timer.schedule(25, lambda: results.append("25ms 타이머 (취소됨)"))
t.cancel()

print("타이머 등록 완료. 틱 처리 중...")
for i in range(30):
    time.sleep(0.001)  # 1ms
    timer.tick()

print("실행된 타이머:", results)
```

### 예제 2 — Java: Netty `HashedWheelTimer` 실전 활용

Netty는 `HashedWheelTimer`를 제공합니다. 내부적으로 단일 레벨 해시 휠이지만, 라운드 수를 저장하여 긴 타임아웃도 처리합니다.

```java
import io.netty.util.HashedWheelTimer;
import io.netty.util.Timeout;
import io.netty.util.TimerTask;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

public class TimingWheelDemo {

    public static void main(String[] args) throws InterruptedException {
        // tickDuration: 1틱의 시간 단위
        // ticksPerWheel: 휠의 슬롯 수 (2의 거듭제곱 권장)
        HashedWheelTimer timer = new HashedWheelTimer(
            1, TimeUnit.MILLISECONDS,  // 1ms per tick
            512                        // 512 슬롯 → 512ms = 1바퀴
        );

        AtomicInteger counter = new AtomicInteger(0);
        long startNs = System.nanoTime();

        // 타이머 등록: 500ms, 1000ms, 1500ms 후 실행
        Timeout t1 = timer.newTimeout(
            timeout -> System.out.printf("[%.1fms] 태스크 1 실행%n",
                (System.nanoTime() - startNs) / 1e6),
            500, TimeUnit.MILLISECONDS
        );

        Timeout t2 = timer.newTimeout(
            timeout -> System.out.printf("[%.1fms] 태스크 2 실행%n",
                (System.nanoTime() - startNs) / 1e6),
            1000, TimeUnit.MILLISECONDS
        );

        Timeout t3 = timer.newTimeout(
            timeout -> System.out.printf("[%.1fms] 태스크 3 실행 (취소되어야 함)%n",
                (System.nanoTime() - startNs) / 1e6),
            1500, TimeUnit.MILLISECONDS
        );

        // 200ms 후 t3 취소
        timer.newTimeout(
            timeout -> {
                boolean cancelled = t3.cancel();
                System.out.printf("[%.1fms] t3 취소 시도: %b%n",
                    (System.nanoTime() - startNs) / 1e6, cancelled);
            },
            200, TimeUnit.MILLISECONDS
        );

        // 대량 타이머 등록 성능 테스트
        long insertStart = System.nanoTime();
        Timeout[] tasks = new Timeout[100_000];
        for (int i = 0; i < tasks.length; i++) {
            final int idx = i;
            tasks[idx] = timer.newTimeout(
                t -> counter.incrementAndGet(),
                1000 + (idx % 1000),  // 1000~1999ms 분산
                TimeUnit.MILLISECONDS
            );
        }
        long insertMs = (System.nanoTime() - insertStart) / 1_000_000;
        System.out.printf("타이머 100,000개 등록 시간: %dms (평균 %.2fμs/개)%n",
            insertMs, insertMs * 1000.0 / 100_000);

        // 절반 취소
        for (int i = 0; i < tasks.length; i += 2) {
            tasks[i].cancel();
        }

        Thread.sleep(2500);
        System.out.printf("실행된 타이머 수: %d / %d%n", counter.get(), tasks.length / 2);

        timer.stop();
        System.out.println("타이머 종료");
    }
}
```

`HashedWheelTimer`의 슬롯 계산: `(현재 tick + 지연 tick) & (슬롯 수 - 1)`. 슬롯 수가 2의 거듭제곱일 때 모듈로 연산을 비트마스크로 대체할 수 있어 성능이 향상됩니다.

---

## 주의사항 및 팁

### 1. 타이머 정확도와 틱 해상도의 트레이드오프

틱 주기가 작을수록 정확도가 높아지지만, 더 자주 틱 스레드가 깨어나 CPU를 소비합니다. Netty의 권장값은 100ms~1s입니다. 정밀한 타이머가 필요하다면 OS 레벨의 고해상도 타이머(`clock_gettime(CLOCK_MONOTONIC)`)를 참고하고, 소프트 타이머의 한계(스레드 스케줄링 지연)를 감안해야 합니다.

### 2. 타이머 콜백의 실행 컨텍스트

타이밍 휠의 틱 스레드에서 직접 콜백을 실행하면 무거운 콜백이 틱 스레드를 블로킹해 다른 타이머들이 지연됩니다. Netty는 Worker 스레드 풀로 콜백을 위임합니다. Kafka는 만료된 태스크를 별도 ExecutorService에 제출합니다.

### 3. 시간 역행(Clock Skew) 방어

NTP 동기화나 가상화 환경에서 시스템 시계가 뒤로 갈 수 있습니다. 모노토닉 클록(`System.nanoTime()`, `CLOCK_MONOTONIC`)을 사용하고, `currentTime`은 절대로 감소하지 않도록 구현해야 합니다.

### 4. 대규모 재삽입(Cascade) 폭풍

계층 타이밍 휠에서 상위 레벨 슬롯이 만료되면 수많은 타이머가 동시에 하위 레벨로 재삽입됩니다. 이를 **cascade 폭풍**이라 합니다. Kafka는 DelayQueue 통합으로 이를 완화합니다 — 상위 레벨 슬롯 만료가 즉각적으로 알려지므로, 만료 직전에 미리 cascade를 시작할 수 있습니다.

### 5. 슬롯 수는 2의 거듭제곱으로

모듈로 연산 `% slots`를 비트마스크 `& (slots - 1)`로 대체할 수 있어 성능이 약 2배 향상됩니다. Netty의 기본값 512 = 2^9가 좋은 예입니다.

### 6. Linux 커널의 계층적 타이밍 휠

Linux 5.x 커널의 `kernel/time/timer.c`는 9개 레벨의 타이밍 휠을 사용합니다 (LVL_BITS=6, LVL_SIZE=64). 최대 타이머 범위는 약 24일이며, 각 레벨은 이전 레벨의 64배 해상도를 갖습니다. `add_timer()`, `del_timer()` 모두 O(1)에 동작합니다.

타이밍 휠은 "시계"라는 일상적 개념을 알고리즘으로 구현한 우아한 자료구조입니다. O(log N) 장벽을 O(1)로 넘어서면서, 수백만 타이머를 관리하는 현대 분산 시스템의 필수 부품으로 자리잡았습니다.

## 참고 자료

- [Apache Kafka, Purgatory, and Hierarchical Timing Wheels — Confluent Blog](https://www.confluent.io/blog/apache-kafka-purgatory-hierarchical-timing-wheels/)
- [Hashed and Hierarchical Timing Wheels — the morning paper](https://blog.acolyer.org/2015/11/23/hashed-and-hierarchical-timing-wheels/)
- [Netty HashedWheelTimer API](https://netty.io/4.1/api/io/netty/util/HashedWheelTimer.html)
- [Linux kernel timer implementation — kernel.org](https://www.kernel.org/doc/html/latest/timers/timers-howto.html)
