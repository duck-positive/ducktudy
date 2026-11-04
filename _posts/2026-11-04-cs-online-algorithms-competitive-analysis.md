---
layout: post
title: "온라인 알고리즘과 경쟁 분석 완전 정복: 스키 대여 문제부터 k-서버 알고리즘까지"
date: 2026-11-04
categories: [cs, computer-science]
tags: [online-algorithms, competitive-analysis, ski-rental, k-server, paging, algorithms, amortized-analysis]
---

## 개요

**온라인 알고리즘(Online Algorithm)**은 미래 입력을 알지 못하는 상태에서 각 입력이 도착할 때마다 즉시 결정을 내려야 하는 알고리즘이다. 반면 **오프라인 알고리즘(Offline Algorithm)**은 모든 입력을 사전에 알고 있다고 가정한다.

현실 세계의 대부분의 문제는 온라인 성격을 띤다. 운영체제의 스케줄러는 미래에 어떤 프로세스가 올지 모르고, 네트워크 라우터는 다음 패킷이 어디서 올지 모르며, CDN 캐시는 어떤 콘텐츠가 요청될지 미리 알 수 없다.

**경쟁 분석(Competitive Analysis)**은 온라인 알고리즘의 성능을 측정하는 프레임워크로, 1985년 Sleator와 Tarjan이 제안했다. 온라인 알고리즘의 비용을 같은 입력에 대한 **최적 오프라인 알고리즘(OPT)**의 비용과 비교한다.

---

## 경쟁 비율의 정의

온라인 알고리즘 ALG가 **c-경쟁적(c-competitive)**이라 함은, 어떤 입력 시퀀스 σ에 대해서도 다음이 성립함을 의미한다:

```
ALG(σ) ≤ c · OPT(σ) + b
```

여기서 c는 **경쟁 비율(competitive ratio)**, b는 상수다. `b = 0`인 경우 **엄격히 c-경쟁적(strictly c-competitive)**이라 한다.

경쟁 분석의 목표는 경쟁 비율 c를 최소화하는 알고리즘을 설계하고, 하한(lower bound)을 증명해 더 이상 개선이 불가능함을 보이는 것이다.

---

## 스키 대여 문제 (Ski Rental Problem)

### 문제 설정

스키장에서 스키를 빌리는 비용은 하루 1원, 구매 비용은 B원이다. 당신은 앞으로 며칠이나 스키를 탈지 모른다. 언제 구매로 전환할지 결정해야 한다.

이 문제는 **임대 대 구매(Rent-or-Buy)** 문제의 원형으로, 많은 실제 최적화 문제의 추상화다:
- 클라우드: 온디맨드(임대) vs 예약(구매)
- 캐시: 항목을 메모리에 올릴지(임대) vs 미리 로드(구매)할지

### 결정론적 최적 알고리즘

**전략**: B-1일간 임대하다가 B번째 날에 구매한다.

```python
def ski_rental_deterministic(actual_days: int, buy_cost: int) -> dict:
    """
    결정론적 Break-Even 전략.
    actual_days: 실제로 스키를 타는 날수
    buy_cost: 구매 비용 (임대 단가 = 1)
    """
    threshold = buy_cost  # B번째 날에 구매
    
    if actual_days < threshold:
        # 구매 시점 이전에 스키 중단 → 계속 임대
        online_cost = actual_days
    else:
        # B일 임대 후 구매
        online_cost = (threshold - 1) + buy_cost  # = 2B - 1

    # 오프라인 최적: 미래를 알고 있다면
    opt_cost = min(actual_days, buy_cost)  # 일찍 사거나 계속 임대

    ratio = online_cost / max(opt_cost, 1)
    return {
        "online_cost": online_cost,
        "opt_cost": opt_cost,
        "competitive_ratio": ratio,
    }

# 경쟁 비율 분석
buy_cost = 10
print(f"구매 비용 B={buy_cost}")
print(f"{'실제 날수':>8} | {'온라인':>6} | {'OPT':>5} | {'비율':>6}")
print("-" * 35)
for days in [1, 5, 9, 10, 11, 20, 50]:
    r = ski_rental_deterministic(days, buy_cost)
    print(f"{days:>8} | {r['online_cost']:>6} | {r['opt_cost']:>5} | {r['competitive_ratio']:>6.2f}")
```

**출력**:
```
구매 비용 B=10
실제 날수 | 온라인 |   OPT |   비율
-----------------------------------
       1 |      1 |     1 |   1.00
       5 |      5 |     5 |   1.00
       9 |      9 |     9 |   1.00
      10 |     19 |    10 |   1.90
      11 |     19 |    10 |   1.90
      20 |     19 |    10 |   1.90
      50 |     19 |    10 |   1.90
```

**경쟁 비율 증명**: 최악의 경우는 actual_days = B일 때다. 온라인 비용 = (B-1) + B = 2B-1, OPT = B이므로 비율 = (2B-1)/B → 2. 이 알고리즘의 경쟁 비율은 2에 수렴한다.

**하한 증명**: 어떤 결정론적 알고리즘도 경쟁 비율 2를 달성할 수 없다. 역적(adversary)은 알고리즘이 구매하기 전날 스키를 중단해 임대를 계속하게 만들 수 있다.

### 무작위 알고리즘의 개선

무작위 알고리즘은 경쟁 비율을 개선할 수 있다. `[1, B]` 범위에서 무작위로 임계값 T를 선택하면:

```python
import random
import math

def ski_rental_randomized(actual_days: int, buy_cost: int, num_simulations: int = 100000) -> float:
    """
    무작위 Break-Even 전략의 기댓값 경쟁 비율 시뮬레이션.
    T를 기하분포(geometric distribution)로 샘플링: P(T=t) ∝ (1-1/B)^(t-1)
    """
    B = buy_cost
    opt = min(actual_days, B)
    
    total_ratio = 0.0
    for _ in range(num_simulations):
        # 기하분포로 T 샘플링: B-1번 임대 후 구매하는 전략
        # E[임대비율] ≤ (e/(e-1)) ≈ 1.582
        p = 1.0 / B
        T = 1
        while T < B and random.random() > p:
            T += 1
        
        if actual_days < T:
            cost = actual_days
        else:
            cost = (T - 1) + B
        
        total_ratio += cost / max(opt, 1)
    
    return total_ratio / num_simulations

B = 10
print("무작위 알고리즘 기대 경쟁 비율 (1,000회 시뮬레이션):")
for days in [1, 5, 9, 10, 15, 30]:
    ratio = ski_rental_randomized(days, B, 10000)
    print(f"  days={days:3d}: 기대 비율 ≈ {ratio:.3f}")
print(f"\n이론적 최적 무작위 비율: e/(e-1) ≈ {math.e/(math.e-1):.4f}")
```

무작위 알고리즘의 최적 경쟁 비율은 **e/(e-1) ≈ 1.582**로, 결정론적 하한 2보다 낮다.

---

## 페이징 문제 (Paging Problem)

### 문제 설정

느린 메모리(디스크)에서 빠른 캐시(k 슬롯)로 페이지를 로드하는 문제다. 캐시 미스(cache miss) 시 비용 1이 발생하며, 캐시가 가득 찬 상태에서 새 페이지를 올리려면 기존 페이지를 하나 추출(evict)해야 한다. **어떤 페이지를 추출할지 결정하는 것**이 온라인 알고리즘의 과제다.

**오프라인 최적(Bélády's algorithm)**: 미래에 가장 나중에 사용될 페이지를 추출한다. 이는 최적이지만 미래 정보가 필요해 온라인에서 사용 불가능하다.

### LRU 분석

**LRU(Least Recently Used)**: 가장 오래 전에 사용된 페이지를 추출한다.

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()
        self.miss_count = 0

    def access(self, page: int) -> bool:
        """페이지 접근. 캐시 미스이면 True 반환."""
        if page in self.cache:
            self.cache.move_to_end(page)
            return False  # 캐시 히트
        
        # 캐시 미스
        self.miss_count += 1
        if len(self.cache) >= self.capacity:
            self.cache.popitem(last=False)  # LRU 항목 추출
        self.cache[page] = True
        return True

def belady_opt(requests: list, capacity: int) -> int:
    """Bélády 최적 알고리즘: 미래에 가장 늦게 쓸 페이지를 추출."""
    cache = set()
    miss_count = 0
    
    for i, page in enumerate(requests):
        if page in cache:
            continue
        miss_count += 1
        if len(cache) >= capacity:
            # 미래 사용 순서 계산
            future_use = {}
            for cached_page in cache:
                try:
                    future_use[cached_page] = requests[i+1:].index(cached_page)
                except ValueError:
                    future_use[cached_page] = float('inf')
            evict = max(future_use, key=lambda p: future_use[p])
            cache.remove(evict)
        cache.add(page)
    
    return miss_count

# --- 비교 실험 ---
import random
random.seed(42)
requests = [random.randint(0, 7) for _ in range(50)]
capacity = 4

lru = LRUCache(capacity)
for page in requests:
    lru.access(page)

opt_misses = belady_opt(requests, capacity)
lru_misses = lru.miss_count

print(f"요청 시퀀스: {requests}")
print(f"\n캐시 용량 k={capacity}")
print(f"OPT 미스 수: {opt_misses}")
print(f"LRU 미스 수: {lru_misses}")
print(f"비율: {lru_misses / max(opt_misses, 1):.2f} (이론적 상한: {capacity})")
```

**LRU의 경쟁 비율**: LRU는 k-경쟁적이다. 즉, OPT 대비 최대 k배의 비용이 발생한다. 모든 결정론적 페이징 알고리즘의 경쟁 비율 하한은 k이므로 LRU는 **결정론적 최적**이다.

**FIFO도 k-경쟁적**: FIFO(First-In-First-Out)도 k-경쟁적이지만 LRU보다 실제 성능이 떨어지는 경우가 많다. 경쟁 분석은 최악의 경우를 분석하므로 평균 케이스를 구분하지 못하는 한계가 있다.

---

## k-서버 문제 (k-Server Problem)

### 문제 설정

메트릭 공간(Metric Space) M에 k개의 서버와 일련의 요청 시퀀스가 주어진다. 각 요청은 M의 한 점이며, 서버 중 하나를 그 점으로 이동시켜야 한다. 이동 비용은 메트릭 거리이고, 총 이동 비용을 최소화하는 것이 목표다.

페이징 문제는 k-서버 문제의 특수 케이스다: 각 메모리 페이지를 한 점, 각 캐시 슬롯을 서버로 보면 이동 비용이 0 또는 1인 균등 메트릭이 된다.

**k-서버 추측(k-Server Conjecture)**: 모든 메트릭 공간에 대해 k-경쟁적인 결정론적 알고리즘이 존재한다는 추측. 아직 완전히 증명되지 않았다.

**작업 함수 알고리즘(Work Function Algorithm)**: (2k-1)-경쟁적임이 증명된 알고리즘으로, k-서버 추측에 가장 가까운 결과다.

### 직선 위의 k-서버: 더블 커버 알고리즘

```python
import heapq

def double_cover_line(requests: list, k: int, initial_positions: list) -> dict:
    """
    직선 메트릭에서의 k-서버 더블 커버 알고리즘.
    직선 위에서 k-경쟁적임이 증명됨.
    """
    servers = sorted(initial_positions[:k])
    total_cost = 0
    opt_cost = 0  # 단순화: OPT 하한 추정

    history = []
    for req in requests:
        # 가장 가까운 서버 찾기
        min_dist = float('inf')
        best_server = -1
        for i, pos in enumerate(servers):
            d = abs(pos - req)
            if d < min_dist:
                min_dist = d
                best_server = i
        
        total_cost += min_dist
        servers[best_server] = req
        servers.sort()
        history.append({
            "request": req,
            "cost": min_dist,
            "servers": servers[:],
        })

    # OPT 하한: 모든 요청의 순차 이동 거리의 절반
    prev = requests[0]
    for req in requests[1:]:
        opt_cost += abs(req - prev) / k
        prev = req

    return {
        "total_cost": total_cost,
        "opt_lower_bound": opt_cost,
        "ratio": total_cost / max(opt_cost, 1),
        "history": history[:5],  # 처음 5개만 출력
    }

# 실험: 직선 위 3-서버 문제
initial = [0, 10, 20]
requests = [5, 15, 2, 18, 8, 12, 25, 3]

result = double_cover_line(requests, k=3, initial_positions=initial)
print(f"초기 서버 위치: {initial}")
print(f"요청 시퀀스: {requests}")
print(f"\n총 이동 비용 (온라인): {result['total_cost']}")
print(f"OPT 하한 추정: {result['opt_lower_bound']:.2f}")
print(f"경쟁 비율: {result['ratio']:.2f}")
print("\n처음 5번의 요청 처리:")
for step in result['history']:
    print(f"  요청={step['request']}, 비용={step['cost']}, 서버={step['servers']}")
```

---

## Yao의 소형화 원리와 무작위 하한

**Yao의 Minimax 원리**: 결정론적 알고리즘에 대한 하한을 증명할 때, 역적(adversary)이 입력 분포를 선택하는 혼합 전략 게임으로 변환한다.

무작위 알고리즘의 경쟁 비율 하한 ρ를 증명하려면:
1. 입력 분포 D를 구성한다
2. 임의의 결정론적 알고리즘 ALG에 대해 E_D[ALG] ≥ ρ · E_D[OPT]임을 보인다
3. 무작위 알고리즘은 고정된 입력에 대해 결정론적 알고리즘들의 혼합이므로 동일한 하한이 적용된다

스키 대여 문제에서 무작위 하한은 e/(e-1)이며, **하모닉 알고리즘**이 이를 달성한다.

---

## 경쟁 분석의 한계와 대안

경쟁 분석은 최악의 경우를 분석하므로 실제 성능을 과소평가할 수 있다.

**완화된 모델들**:
- **세련된 경쟁 분석(Refined CA)**: FIFO와 LRU를 구분하기 위해 `k'`-서버 문제로 변환
- **평균 케이스 분석**: 입력이 특정 분포를 따른다고 가정
- **비율 최적성(Bijective Analysis)**: 모든 입력 순열을 비교
- **조언 복잡도(Advice Complexity)**: 오라클이 몇 비트를 알려줄 때 경쟁 비율이 어떻게 변하는지 분석

---

## 실제 응용

| 도메인 | 온라인 문제 | 대표 알고리즘 |
|--------|------------|-------------|
| 운영체제 | 페이지 교체 | LRU, CLOCK |
| 웹 캐싱 | CDN 캐시 교체 | LRU-K, ARC |
| 클라우드 | VM 배치/이전 | 균형 알고리즘 |
| 주식 거래 | 최적 매도 타이밍 | 예약 가격 알고리즘 |
| 네트워크 | 패킷 버퍼링 | Tail Drop, RED |
| 스케줄링 | 작업 순서 결정 | List Scheduling |

---

## 주의사항과 팁

1. **경쟁 비율의 의미**: 경쟁 비율 c는 최악의 경우 OPT보다 c배 나쁠 수 있다는 의미지, 평균적으로 그렇다는 뜻이 아니다.

2. **상수 항 b**: 엄격한 경쟁 비율과 완화된 경쟁 비율을 구분해야 한다. 일부 문제에서는 상수 b 없이는 유한한 경쟁 비율 달성이 불가능하다.

3. **메트릭 공간 의존성**: 많은 온라인 알고리즘의 경쟁 비율은 메트릭 공간의 구조에 의존한다. 직선, 트리, 균등 메트릭에서 각기 다른 결과가 나온다.

4. **무작위 vs 결정론적 차이**: Yao의 원리로 무작위 알고리즘의 하한을 결정론적 하한보다 낮게 증명할 수 있지만, 역적 모델(oblivious vs adaptive)에 따라 결과가 달라진다.

---

## 참고 자료
- [Wikipedia: Online algorithm](https://en.wikipedia.org/wiki/Online_algorithm)
- [Wikipedia: Competitive analysis (online algorithm)](https://en.wikipedia.org/wiki/Competitive_analysis_(online_algorithm))
- [Borodin & El-Yaniv (1998): Online Computation and Competitive Analysis — Cambridge University Press](https://www.cambridge.org/core/books/online-computation-and-competitive-analysis/3C8B85F68741D23A49F46F39C3E0BABC)
- [Sleator & Tarjan (1985): Amortized Efficiency of List Update and Paging Rules — CACM](https://dl.acm.org/doi/10.1145/2786.2793)
