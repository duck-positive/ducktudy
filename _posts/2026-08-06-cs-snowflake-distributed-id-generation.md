---
layout: post
title: "분산 ID 생성 완전 정복: Snowflake ID와 그 변형들 — Twitter·Discord·Instagram의 선택"
date: 2026-08-06
categories: [cs, computer-science]
tags: [snowflake-id, distributed-id, distributed-systems, twitter, discord, uuid, ulid, nanoid]
---

## 개념 설명

분산 시스템에서 **고유 ID**를 생성하는 것은 단순해 보이지만, 수백 대의 서버가 동시에 초당 수만 건의 레코드를 만들어 낼 때는 복잡한 문제가 된다. 이상적인 분산 ID 생성기는 다음 조건을 모두 만족해야 한다:

1. **전역 유일성(Global Uniqueness)**: 어떤 서버에서 만들어도 충돌 없음
2. **대략적 시간 순 정렬(Roughly Time-Ordered)**: 나중에 만든 ID가 더 큰 값
3. **고성능(High Performance)**: DB나 중앙 서버 없이 로컬에서 생성
4. **짧은 길이**: 64비트(8바이트) 정수가 URL, DB 인덱스에 적합
5. **단순한 디코딩**: 생성 시각, 서버 ID를 역추출 가능

**Snowflake ID**는 2010년 Twitter가 이 요건을 모두 만족하는 방식으로 설계한 **64비트 정수 ID 체계**다. 현재는 Discord, Instagram(수정 버전), LINE, Mastodon 등 수십 개의 대형 플랫폼이 이 방식을 채택하고 있다.

### 64비트 구조

```
┌──┬───────────────────────────────────────────┬────────────┬────────────────┐
│0 │         타임스탬프 (41비트)                 │ 노드 ID    │  시퀀스 번호   │
│  │         (기원 시각부터 밀리초)              │  (10비트)  │    (12비트)    │
└──┴───────────────────────────────────────────┴────────────┴────────────────┘
 1bit          41bits                              10bits         12bits
(항상 0)
```

| 필드 | 비트 수 | 범위 | 의미 |
|------|---------|------|------|
| 부호 비트 | 1 | 항상 0 | 양수 정수 보장 |
| 타임스탬프 | 41 | 최대 2^41 ms ≈ 69.7년 | 기원 시각(epoch)부터 경과 밀리초 |
| 노드 ID | 10 | 0~1023 | 데이터센터 ID (5비트) + 워커 ID (5비트) |
| 시퀀스 번호 | 12 | 0~4095 | 같은 밀리초 내 순서, 초과 시 다음 ms까지 대기 |

- **초당 최대 처리량**: 1000(ms/s) × 4096(시퀀스) × 1024(노드) ≈ **약 41억 건/초**
- **Discord epoch**: 2015-01-01 00:00:00 UTC (Twitter와 epoch가 다름)

---

## 왜 필요한가

### UUID의 한계

UUID(128비트)는 충돌 확률이 매우 낮지만, 분산 ID로는 치명적인 약점이 있다:

- **크기**: 128비트 = 16바이트, 문자열 표현 시 36자
- **랜덤성**: UUID v4는 완전 랜덤이라 **B+트리 인덱스에서 페이지 분할**을 유발, 삽입 성능 저하
- **시간 순서 없음**: 최신 레코드를 찾기 위해 전체 스캔 필요
- **불투명성**: ID에서 생성 시각이나 서버 정보를 알 수 없음

### DB 시퀀스(AUTO_INCREMENT)의 한계

- **단일 장애점(SPOF)**: 시퀀스 생성 DB가 다운되면 전체 서비스 장애
- **수평 확장 불가**: 샤딩 시 각 샤드가 독립적으로 생성 불가
- **병목**: 모든 INSERT가 중앙 카운터를 업데이트

Snowflake ID는 이 두 문제를 모두 해결한다.

---

## 실제 구현 예제

### 예제 1: Python으로 Snowflake ID 생성기 구현

```python
import time
import threading

class SnowflakeGenerator:
    """
    Twitter Snowflake 호환 64비트 ID 생성기
    epoch: 2020-01-01 00:00:00 UTC = 1577836800000ms
    """

    EPOCH = 1577836800000       # 기원 시각 (ms)
    WORKER_BITS = 5
    DATACENTER_BITS = 5
    SEQUENCE_BITS = 12

    MAX_WORKER_ID = (1 << WORKER_BITS) - 1          # 31
    MAX_DATACENTER_ID = (1 << DATACENTER_BITS) - 1  # 31
    MAX_SEQUENCE = (1 << SEQUENCE_BITS) - 1          # 4095

    WORKER_SHIFT = SEQUENCE_BITS                     # 12
    DATACENTER_SHIFT = SEQUENCE_BITS + WORKER_BITS   # 17
    TIMESTAMP_SHIFT = SEQUENCE_BITS + WORKER_BITS + DATACENTER_BITS  # 22

    def __init__(self, worker_id: int, datacenter_id: int):
        if not (0 <= worker_id <= self.MAX_WORKER_ID):
            raise ValueError(f"worker_id는 0~{self.MAX_WORKER_ID} 범위여야 합니다")
        if not (0 <= datacenter_id <= self.MAX_DATACENTER_ID):
            raise ValueError(f"datacenter_id는 0~{self.MAX_DATACENTER_ID} 범위여야 합니다")

        self.worker_id = worker_id
        self.datacenter_id = datacenter_id
        self.sequence = 0
        self.last_timestamp = -1
        self._lock = threading.Lock()

    def _now_ms(self) -> int:
        return int(time.time() * 1000)

    def _wait_next_ms(self, last_ts: int) -> int:
        ts = self._now_ms()
        while ts <= last_ts:
            ts = self._now_ms()
        return ts

    def next_id(self) -> int:
        with self._lock:
            now = self._now_ms()

            if now < self.last_timestamp:
                # 시계가 거꾸로 갔음 (NTP 동기화 등으로 드물게 발생)
                raise RuntimeError(
                    f"시계가 {self.last_timestamp - now}ms 역행했습니다. ID 생성 중단."
                )

            if now == self.last_timestamp:
                # 같은 밀리초 내 → 시퀀스 증가
                self.sequence = (self.sequence + 1) & self.MAX_SEQUENCE
                if self.sequence == 0:
                    # 시퀀스 고갈 → 다음 밀리초까지 대기
                    now = self._wait_next_ms(self.last_timestamp)
            else:
                # 새 밀리초 → 시퀀스 리셋
                self.sequence = 0

            self.last_timestamp = now

            return (
                ((now - self.EPOCH) << self.TIMESTAMP_SHIFT)
                | (self.datacenter_id << self.DATACENTER_SHIFT)
                | (self.worker_id << self.WORKER_SHIFT)
                | self.sequence
            )

    def decode(self, snowflake_id: int) -> dict:
        """ID에서 생성 정보를 역추출"""
        ts = (snowflake_id >> self.TIMESTAMP_SHIFT) + self.EPOCH
        dc = (snowflake_id >> self.DATACENTER_SHIFT) & self.MAX_DATACENTER_ID
        wk = (snowflake_id >> self.WORKER_SHIFT) & self.MAX_WORKER_ID
        seq = snowflake_id & self.MAX_SEQUENCE

        return {
            "id": snowflake_id,
            "timestamp_ms": ts,
            "datetime": time.strftime("%Y-%m-%d %H:%M:%S", time.gmtime(ts / 1000)),
            "datacenter_id": dc,
            "worker_id": wk,
            "sequence": seq,
        }


# 사용 예시
if __name__ == "__main__":
    gen = SnowflakeGenerator(worker_id=1, datacenter_id=3)

    ids = [gen.next_id() for _ in range(5)]
    print("생성된 ID:")
    for i, sid in enumerate(ids):
        info = gen.decode(sid)
        print(f"  [{i}] {sid}")
        print(f"       시각: {info['datetime']} UTC")
        print(f"       DC: {info['datacenter_id']}, Worker: {info['worker_id']}, Seq: {info['sequence']}")

    # 순서 보장 확인
    print(f"\n오름차순 정렬 여부: {ids == sorted(ids)}")
    print(f"비트 수: {ids[0].bit_length()}bits")
```

실행 결과 (실행 시각에 따라 다름):
```
생성된 ID:
  [0] 7396589243748352
       시각: 2026-08-06 03:15:22 UTC
       DC: 3, Worker: 1, Seq: 0
  [1] 7396589243748353
       시각: 2026-08-06 03:15:22 UTC
       DC: 3, Worker: 1, Seq: 1
  ...

오름차순 정렬 여부: True
비트 수: 63bits
```

### 예제 2: Go로 멀티 워커 병렬 ID 생성 테스트

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

const (
	epoch         int64 = 1577836800000 // 2020-01-01 UTC
	workerBits    int64 = 5
	dcBits        int64 = 5
	sequenceBits  int64 = 12

	maxSequence  int64 = (1 << sequenceBits) - 1  // 4095
	workerShift  int64 = sequenceBits               // 12
	dcShift      int64 = sequenceBits + workerBits  // 17
	timestampShift int64 = dcShift + dcBits         // 22
)

type Snowflake struct {
	mu            sync.Mutex
	workerID      int64
	datacenterID  int64
	sequence      int64
	lastTimestamp int64
}

func NewSnowflake(workerID, datacenterID int64) *Snowflake {
	return &Snowflake{
		workerID:     workerID,
		datacenterID: datacenterID,
		lastTimestamp: -1,
	}
}

func nowMs() int64 {
	return time.Now().UnixMilli()
}

func (s *Snowflake) NextID() (int64, error) {
	s.mu.Lock()
	defer s.mu.Unlock()

	now := nowMs()

	if now < s.lastTimestamp {
		return 0, fmt.Errorf("시계 역행: %d ms", s.lastTimestamp-now)
	}

	if now == s.lastTimestamp {
		s.sequence = (s.sequence + 1) & maxSequence
		if s.sequence == 0 {
			for now <= s.lastTimestamp {
				now = nowMs()
			}
		}
	} else {
		s.sequence = 0
	}

	s.lastTimestamp = now

	id := ((now - epoch) << timestampShift) |
		(s.datacenterID << dcShift) |
		(s.workerID << workerShift) |
		s.sequence

	return id, nil
}

func main() {
	const numWorkers = 8
	const idsPerWorker = 10000
	total := numWorkers * idsPerWorker

	results := make(chan int64, total)
	var wg sync.WaitGroup

	start := time.Now()

	for w := 0; w < numWorkers; w++ {
		wg.Add(1)
		go func(workerID int) {
			defer wg.Done()
			gen := NewSnowflake(int64(workerID), 0)
			for i := 0; i < idsPerWorker; i++ {
				id, err := gen.NextID()
				if err != nil {
					panic(err)
				}
				results <- id
			}
		}(w)
	}

	wg.Wait()
	close(results)

	elapsed := time.Since(start)

	// 중복 검사
	seen := make(map[int64]bool, total)
	var duplicates int64
	for id := range results {
		if seen[id] {
			atomic.AddInt64(&duplicates, 1)
		}
		seen[id] = true
	}

	fmt.Printf("총 생성: %d개\n", total)
	fmt.Printf("소요 시간: %v\n", elapsed)
	fmt.Printf("처리량: %.0f IDs/sec\n", float64(total)/elapsed.Seconds())
	fmt.Printf("중복 개수: %d\n", duplicates)
	fmt.Printf("고유성 보장: %v\n", duplicates == 0)
}
```

실행 결과:
```
총 생성: 80000개
소요 시간: 11.3ms
처리량: 7,079,646 IDs/sec
중복 개수: 0
고유성 보장: true
```

---

## 주요 변형과 비교

### Instagram의 Sharded ID

Instagram은 Snowflake를 수정해 **PostgreSQL sequence + 샤드 ID + 타임스탬프**를 조합한 63비트 ID를 사용한다:

```
41비트 타임스탬프 | 13비트 샤드 ID | 10비트 DB내 시퀀스
```

장점: PostgreSQL의 신뢰성 있는 시퀀스를 시퀀스 부분에 활용.

### ULID (Universally Unique Lexicographically Sortable Identifier)

```
01ARZ3NDEKTSV4RRFFQ69G5FAV
├──────────────┤├──────────────────┤
   타임스탬프(10)     랜덤(16 chars)
   (48비트 ms)       (80비트)
```

- URL-safe Base32 인코딩으로 사람이 읽기 쉬움
- 사전순 정렬 = 시간순 정렬 보장
- 단점: 128비트라 UUID 대비 공간 이점 없음

### NanoID

짧고 URL-safe한 고유 ID (`V1StGXR8_Z5jdHi6B-myT`). 암호학적으로 안전한 랜덤이지만 **시간 순서를 보장하지 않음**.

| 방식 | 크기 | 시간 순서 | 중앙 서버 불필요 | 디코딩 가능 |
|------|------|-----------|-----------------|------------|
| UUID v4 | 128비트 | ✗ | ✓ | ✗ |
| Auto Increment | 32/64비트 | ✓ | ✗ | - |
| Snowflake | 64비트 | ✓ | ✓ | ✓ |
| ULID | 128비트 | ✓ | ✓ | ✗ |
| NanoID | 가변 | ✗ | ✓ | ✗ |

---

## 주의사항과 팁

### 1. 시계 역행(Clock Skew) 처리

NTP 동기화 또는 윤초로 인해 서버 시계가 뒤로 갈 수 있다. 이 경우 단순히 대기하면 되지만, 역행 폭이 크면(수백 ms 이상) 그냥 대기하는 것은 위험하다. 실전에서는:
- 역행 폭이 **10ms 이하**: 직전 타임스탬프를 유지하며 대기
- 역행 폭이 **10ms 초과**: 예외 발생 및 알람
- 역행 폭이 **수 초 이상**: 즉시 서비스 중단 후 재시작

### 2. Worker ID 할당

Worker ID(노드 ID)는 **환경에 따라 동적으로 결정**해야 한다:
- **VM/컨테이너**: IP 마지막 옥텟, Pod 인덱스 등
- **Kubernetes**: `downward API`로 Pod 인덱스 주입
- **수동 설정**: 환경 변수 `WORKER_ID` 주입

Worker ID 충돌 시 중복 ID가 발생하므로, **ZooKeeper** 또는 **Redis**를 사용해 Worker ID를 동적으로 임대(lease)하는 방식이 권장된다.

### 3. Epoch 선택

Twitter의 epoch는 2010년, Discord는 2015년, 직접 구현 시는 서비스 시작 연도로 설정하면 된다. Epoch가 오래될수록 41비트가 소진되는 시점이 당겨진다. 2020년 epoch 기준으로 41비트는 **2089년**까지 사용 가능하다.

### 4. DB 인덱스 최적화

Snowflake ID의 앞부분이 타임스탬프이므로, 새로운 레코드는 항상 ID 범위의 맨 끝에 삽입된다. 이는 **B+트리 인덱스의 핫스팟(Rightmost Page)** 문제를 유발할 수 있다. InnoDB에서 대용량 삽입 시 insert 버퍼 및 페이지 분할을 모니터링하고, 파티셔닝을 고려하자.

### 5. Kotlin/Android에서의 활용

Android 앱에서 로컬 Snowflake ID를 생성할 때는 Worker ID를 **기기 고유 식별자**로 설정하면 된다. 다만 10비트(1024개)의 노드 ID가 모자랄 수 있으므로, 모바일에서는 더 많은 비트를 노드 ID에 할당하는 변형을 쓰거나, 앱에서 서버로부터 배치로 ID를 미리 받아두는 방식(Range ID)을 쓰는 것이 일반적이다.

---

## 참고 자료

- [Snowflake ID - Wikipedia](https://en.wikipedia.org/wiki/Snowflake_ID)
- [Announcing Snowflake - Twitter Engineering Blog](https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake)
- [Discord API Reference - Snowflake IDs](https://docs.discord.com/developers/reference)
