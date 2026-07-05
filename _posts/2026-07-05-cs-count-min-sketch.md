---
layout: post
title: "Count-Min Sketch 완전 정복: 수십억 이벤트의 빈도를 수 KB로 추정하는 확률적 자료구조"
date: 2026-07-05
categories: [cs, computer-science]
tags: [count-min-sketch, probabilistic, data-structure, streaming, frequency-estimation, heavy-hitters, sketch]
---

## 개요

실시간 스트리밍 환경에서 "이 URL이 지난 1분간 몇 번 호출됐나?", "가장 빈번하게 등장하는 IP는 무엇인가?" 같은 질문에 답해야 할 때가 있다. 이벤트 수가 수십억 개라면 정확한 카운터를 HashMap에 담는 것은 메모리상 불가능하다. Count-Min Sketch(이하 CMS)는 이 문제를 **오버카운팅 오차를 허용하는 대신 상수 수준의 메모리**로 해결하는 확률적 자료구조다.

HyperLogLog가 "고유 값이 몇 개인가(카디널리티)"를 세는 자료구조라면, CMS는 "특정 값이 얼마나 자주 등장하는가(빈도)"를 추정하는 자료구조다. 두 가지는 목적이 완전히 다르며, 스트리밍 데이터 처리 시스템에서 상호 보완적으로 쓰인다.

---

## 왜 필요한가

### 정확한 카운팅의 비용

URL 클릭 수를 정확히 추적한다고 가정하자. 1억 개의 고유 URL이 있다면 HashMap은 키 1개당 최소 수십 바이트를 요구하므로 수 GB가 필요하다. 반면 CMS는 파라미터 설정에 따라 **수십 KB~수 MB** 안에 담을 수 있다.

### 스트리밍 알고리즘의 제약

스트리밍 알고리즘은 다음 세 가지 제약 아래 동작한다:
- **단일 패스(Single Pass)**: 데이터를 한 번만 읽는다.
- **서브리니어 공간(Sublinear Space)**: 입력 크기보다 훨씬 작은 메모리를 사용한다.
- **빠른 업데이트**: 이벤트 하나당 O(d) 시간(d는 해시 함수 개수, 보통 5~10).

CMS는 세 조건을 모두 만족한다.

---

## 자료구조 구조와 동작 원리

### 구조

CMS는 `d × w` 크기의 2차원 정수 배열 `C`와 d개의 독립적인 해시 함수 `h_1, h_2, ..., h_d`로 구성된다.

```
        w (너비)
   ┌──────────────────────┐
d  │ h1: [ 3  0  5  1  2 ]│
(깊│ h2: [ 1  4  2  3  0 ]│
이)│ h3: [ 0  2  8  1  0 ]│
   └──────────────────────┘
```

- `w`: 각 행의 버킷 수, 클수록 충돌 확률 감소
- `d`: 행(해시 함수) 수, 클수록 오차 확률 감소

### Update (카운터 증가)

아이템 `x`가 도착하면 d개의 해시 함수로 각 행의 열 인덱스를 계산하고 카운터를 1 증가시킨다:

```
for i in 1..d:
    C[i][h_i(x)] += 1
```

### Query (빈도 추정)

아이템 `x`의 빈도를 추정할 때는 d개의 행에서 읽은 값의 **최솟값**을 반환한다:

```
estimate(x) = min over i in 1..d: C[i][h_i(x)]
```

최솟값을 쓰는 이유는 해시 충돌로 인해 카운터가 **과대 계수(overcounting)** 될 수는 있어도 **과소 계수(undercounting)** 는 절대 발생하지 않기 때문이다. 따라서 min이 실제 빈도에 가장 가까운 값이다.

---

## 오차 보장 (Error Guarantee)

CMS는 다음 확률적 보장을 제공한다:

```
P[ estimate(x) ≤ freq(x) + ε·N ] ≥ 1 - δ
```

- `freq(x)`: x의 실제 빈도
- `N`: 총 이벤트 수
- `ε = e / w` (e ≈ 2.718)
- `δ = (1/2)^d` (또는 더 엄밀하게는 e^(-d/2) 근사)

즉 `w = e/ε`, `d = ln(1/δ)`로 파라미터를 설정하면 원하는 오차·확률 보장을 얻을 수 있다.

예: ε=0.01, δ=0.01을 원하면:
- `w = e/0.01 ≈ 272` 버킷
- `d = ln(100) ≈ 5` 행
- 총 메모리: `272 × 5 × 4 byte = 5.44 KB`

---

## 실제 구현 예제

### Python 구현

```python
import hashlib
import math

class CountMinSketch:
    def __init__(self, epsilon: float, delta: float):
        """
        epsilon: 상대 오차 (e.g. 0.01 = 1%)
        delta:   오차 확률 (e.g. 0.01 = 1%)
        """
        self.w = math.ceil(math.e / epsilon)   # 너비
        self.d = math.ceil(math.log(1 / delta)) # 깊이
        self.table = [[0] * self.w for _ in range(self.d)]
        self.seeds = list(range(self.d))        # 해시 시드

    def _hash(self, item: str, seed: int) -> int:
        h = hashlib.md5(f"{seed}:{item}".encode()).hexdigest()
        return int(h, 16) % self.w

    def update(self, item: str, count: int = 1):
        for i in range(self.d):
            col = self._hash(item, self.seeds[i])
            self.table[i][col] += count

    def query(self, item: str) -> int:
        return min(
            self.table[i][self._hash(item, self.seeds[i])]
            for i in range(self.d)
        )

    def memory_bytes(self) -> int:
        return self.w * self.d * 4  # 4 bytes per int32


# 사용 예시
cms = CountMinSketch(epsilon=0.01, delta=0.01)
events = ["login", "search", "login", "purchase", "login", "search", "login"]
for e in events:
    cms.update(e)

print(f"login 빈도 추정: {cms.query('login')}")    # 실제 4
print(f"search 빈도 추정: {cms.query('search')}")  # 실제 2
print(f"메모리 사용량: {cms.memory_bytes()} bytes")
```

### Heavy Hitters 탐지 (Top-K 아이템)

CMS만으로는 "가장 빈번한 K개"를 바로 찾을 수 없다. **MinHeap을 함께 사용**하면 Heavy Hitters를 실시간으로 추적할 수 있다.

```python
import heapq

class HeavyHitterTracker:
    """CMS + MinHeap으로 Top-K 아이템 실시간 추적"""

    def __init__(self, epsilon: float, delta: float, k: int):
        self.cms = CountMinSketch(epsilon, delta)
        self.k = k
        self.heap = []          # (freq_estimate, item)
        self.in_heap = {}       # item -> 현재 heap내 freq

    def update(self, item: str):
        self.cms.update(item)
        freq = self.cms.query(item)

        if item in self.in_heap:
            # 힙 직접 업데이트 (lazy deletion 방식)
            self.in_heap[item] = freq
        elif len(self.heap) < self.k:
            heapq.heappush(self.heap, (freq, item))
            self.in_heap[item] = freq
        elif freq > self.heap[0][0]:
            _, removed = heapq.heapreplace(self.heap, (freq, item))
            self.in_heap.pop(removed, None)
            self.in_heap[item] = freq

    def top_k(self):
        return sorted(self.in_heap.items(), key=lambda x: -x[1])


tracker = HeavyHitterTracker(epsilon=0.01, delta=0.01, k=3)
stream = ["a"]*50 + ["b"]*30 + ["c"]*20 + ["d"]*5 + ["e"]*3
for item in stream:
    tracker.update(item)

print("Top-3 Heavy Hitters:", tracker.top_k())
# Top-3 Heavy Hitters: [('a', 50), ('b', 30), ('c', 20)]
```

---

## Redis의 Count-Min Sketch 지원

Redis 모듈 `RedisBloom`에는 CMS가 내장되어 있다:

```bash
# 초기화: 너비 1000, 깊이 5
CMS.INITBYDIM cms_key 1000 5

# 오차율/확률로 자동 계산
CMS.INITBYPROB cms_key 0.001 0.01

# 카운터 증가
CMS.INCRBY cms_key item1 10 item2 20

# 빈도 조회
CMS.QUERY cms_key item1 item2
```

프로덕션에서는 Redis Cluster 위에 CMS를 올려 분산 빈도 추적을 구현할 수 있다.

---

## CMS vs 유사 자료구조 비교

| 자료구조 | 추정 대상 | 오차 유형 | 메모리 |
|---|---|---|---|
| HashMap | 정확한 빈도 | 없음 | O(n) |
| **Count-Min Sketch** | **빈도 추정** | **오버카운팅만** | **O(1/ε · log 1/δ)** |
| HyperLogLog | 카디널리티 추정 | 양방향 오차 | O(log log n) |
| Bloom Filter | 멤버십 테스트 | 거짓 양성만 | O(n) |
| Count Sketch | 빈도 추정 | 양방향 오차 | O(1/ε² · log 1/δ) |

Count Sketch는 CMS보다 양방향 오차를 내지만, 음수 카운터를 지원하기 때문에 값 감소(decrement) 연산이 필요한 경우에 유용하다.

---

## 주의사항 및 팁

### 1. 파라미터 선택
ε는 항상 `상대 오차 × 총 이벤트 수`임을 명심하자. 총 이벤트가 10억 개라면 ε=0.001이어도 최대 오차는 100만 회다. 목표 오차를 절댓값으로 먼저 결정한 뒤 ε을 역산하는 것이 좋다.

### 2. 해시 함수 독립성
d개의 해시 함수는 **쌍독립(pairwise independent)** 이어야 오차 보장이 유효하다. MurmurHash3나 xxHash로 시드(seed)를 달리해 생성하는 것이 일반적이다.

### 3. 삭제 지원 안 됨
표준 CMS는 카운터 감소가 불가능하다. 이벤트 삭제(취소)가 필요하면 Conservative Update CMS나 CM-CU 변형을 사용한다.

### 4. Conservative Update 최적화
Conservative Update는 `C[i][h_i(x)]`가 현재 추정값보다 큰 경우에만 업데이트하여 오버카운팅을 줄인다:

```python
def update_conservative(self, item: str):
    est = self.query(item)  # 현재 추정값
    for i in range(self.d):
        col = self._hash(item, self.seeds[i])
        if self.table[i][col] <= est:  # 필요한 경우만 업데이트
            self.table[i][col] = est + 1
```

---

## 실전 활용 사례

- **Netflix/YouTube**: 실시간 시청 이벤트의 콘텐츠별 조회수 추정
- **CloudFlare**: DDoS 감지를 위한 IP별 요청 빈도 추적
- **Kafka Streams**: 토픽별 메시지 빈도를 상태 저장소 없이 추정
- **Twitter**: 트렌딩 해시태그 탐지 (Heavy Hitter + CMS)
- **데이터베이스 쿼리 최적화기**: 컬럼 값 빈도 히스토그램 근사

---

## 정리

Count-Min Sketch는 스트리밍 환경에서 빈도 추정 문제를 공간-정확도 트레이드오프로 우아하게 해결한다. 오차는 오버카운팅 방향으로만 발생하며, 파라미터 `(ε, δ)` 로 오차 범위와 확률을 수학적으로 보장할 수 있다. HyperLogLog(카디널리티), Bloom Filter(멤버십)와 함께 **확률적 자료구조 3대장**으로 불리며, 대규모 실시간 분석 시스템의 핵심 구성 요소다.

## 참고 자료
- [Count-Min Sketch 원논문: Cormode & Muthukrishnan (2005)](https://cs.uwaterloo.ca/~ali/courses/854-W10/count-min.pdf)
- [Wikipedia: Count–min sketch](https://en.wikipedia.org/wiki/Count%E2%80%93min_sketch)
- [Redis Count-Min Sketch 공식 문서](https://redis.io/docs/latest/develop/data-types/probabilistic/count-min-sketch/)
- [k-way merge algorithm Wikipedia](https://en.wikipedia.org/wiki/K-way_merge_algorithm)
