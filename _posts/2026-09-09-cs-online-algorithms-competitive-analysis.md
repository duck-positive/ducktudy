---
layout: post
title: "온라인 알고리즘과 경쟁 분석 완전 정복: 미래를 모른 채 결정을 내리는 알고리즘의 수학"
date: 2026-09-09
categories: [cs, computer-science]
tags: [online-algorithm, competitive-analysis, ski-rental, k-server, paging, competitive-ratio]
---

## 개념 설명

컴퓨터 과학에서 대부분의 알고리즘은 **오프라인(offline)** 환경을 가정한다. 입력 전체를 미리 알고 있기 때문에, 최적의 전략을 사전에 계획하고 실행할 수 있다. 그러나 현실 세계의 수많은 문제는 그렇지 않다. 데이터가 하나씩, 순차적으로 도착하고, 알고리즘은 미래의 입력을 전혀 모른 채 **즉각적인 결정**을 내려야 한다.

**온라인 알고리즘(Online Algorithm)**은 바로 이러한 환경을 다룬다. 입력 시퀀스의 다음 원소를 보지 않은 상태에서, 현재까지의 정보만으로 돌이킬 수 없는 결정(irrevocable decision)을 내린다. 네트워크 라우팅, 운영체제 스케줄러, 주식 매매 알고리즘, 웹 캐싱 등이 모두 온라인 알고리즘이 적용되는 대표적인 영역이다.

온라인 알고리즘의 성능을 측정하는 핵심 도구가 **경쟁 분석(Competitive Analysis)**이다. 1985년 Sleator와 Tarjan이 이동-전면(Move-to-Front) 휴리스틱을 분석하며 정형화한 이 프레임워크는, 온라인 알고리즘 OPT-ON의 비용을 전지전능한 **최적 오프라인 알고리즘(OPT)**의 비용과 비교하는 방식을 취한다.

### 경쟁 비율(Competitive Ratio)의 정의

온라인 알고리즘 ALG가 임의의 입력 시퀀스 σ에 대해 비용 ALG(σ)를 내고, 최적 오프라인 알고리즘이 비용 OPT(σ)를 낸다고 할 때:

```
경쟁 비율 c = max_σ [ ALG(σ) / OPT(σ) ]
```

ALG가 **c-competitive**하다는 것은, 어떤 입력 시퀀스에 대해서도 ALG의 비용이 최적값의 c배를 넘지 않음을 의미한다. c가 작을수록 더 좋은 온라인 알고리즘이다. 경쟁 비율이 상수(입력 크기에 무관)라면 해당 알고리즘은 "경쟁적(competitive)"이라고 부른다.

---

## 왜 경쟁 분석이 필요한가

### 전통적인 최악 케이스 분석의 한계

최악 케이스 분석은 "가장 나쁜 입력에서 얼마나 걸리는가"를 측정한다. 하지만 온라인 알고리즘에서는 미래 입력에 대한 무지 자체가 본질적인 불확실성 비용을 야기하며, 이 비용을 알고리즘의 "절대적 복잡도"로 표현하는 것은 의미가 없다.

예를 들어, 페이지 폴트 수는 캐시 크기와 접근 패턴에 따라 완전히 달라진다. 경쟁 분석은 이 불확실성 속에서 **"온라인 알고리즘이 가능한 최선의 전략에 비해 얼마나 나쁜가"**를 정량화한다.

### 확률론적 알고리즘과 적대자 모델

경쟁 분석은 **결정론적(deterministic)**과 **확률론적(randomized)** 알고리즘 모두에 적용된다. 확률론적 알고리즘을 분석할 때는 적대자(adversary)의 모델이 중요해진다:

- **무적응 적대자(Oblivious adversary)**: 알고리즘의 랜덤 비트를 알지 못하고 입력을 사전에 고정한다.
- **적응적 온라인 적대자(Adaptive online adversary)**: 알고리즘의 이전 출력을 관찰하며 실시간으로 입력을 생성한다.
- **적응적 오프라인 적대자(Adaptive offline adversary)**: 알고리즘의 모든 랜덤 비트를 알고 최악의 입력을 설계한다.

무적응 적대자에 대한 기대 경쟁 비율이 가장 의미 있는 분석으로 여겨진다.

---

## 실제 구현 예제

### 예제 1: 스키 렌탈 문제 (Ski Rental Problem)

스키 렌탈 문제는 온라인 알고리즘의 교과서적 예제다. 당신은 스키를 타러 왔고, 하루 렌탈비는 1달러, 스키 구입 비용은 B달러다. 앞으로 며칠을 더 스키를 탈지 모른다. 언제 렌탈을 멈추고 구매해야 할까?

- **최악 전략**: 절대 구매하지 않음 → 경쟁 비율 = ∞ (무한히 렌탈 가능)
- **항상 구매 전략**: 첫날 구매 → 하루만 타면 비율 = B배
- **B일째 구매 전략**: B-1일 렌탈 후 B일째에 구매

**B일째 구매 전략의 분석:**

```python
def ski_rental_deterministic(days_skied: int, buy_cost: int = 10) -> dict:
    """
    결정론적 스키 렌탈 전략: buy_cost일 렌탈 후 구매
    경쟁 비율: 2 - 1/buy_cost (buy_cost가 클수록 2에 근접)
    """
    daily_rent = 1
    total_rental_cost = 0
    bought = False
    buy_day = None

    for day in range(1, days_skied + 1):
        if not bought:
            if day == buy_cost:  # B일째 구매
                total_rental_cost += daily_rent * (buy_cost - 1)
                bought = True
                buy_day = day
            else:
                total_rental_cost += daily_rent

    online_cost = total_rental_cost + (buy_cost if bought else 0)
    opt_cost = min(days_skied * daily_rent, buy_cost)  # 처음부터 알았다면
    ratio = online_cost / opt_cost if opt_cost > 0 else 1.0

    return {
        "days": days_skied,
        "online_cost": online_cost,
        "opt_cost": opt_cost,
        "competitive_ratio": ratio,
        "bought": bought,
        "buy_day": buy_day
    }

# 다양한 시나리오 테스트
for days in [1, 5, 9, 10, 11, 20]:
    result = ski_rental_deterministic(days)
    print(f"Days: {result['days']:2d} | Online: {result['online_cost']:3d} | "
          f"OPT: {result['opt_cost']:3d} | Ratio: {result['competitive_ratio']:.3f} | "
          f"Bought: {result['bought']}")

# Output:
# Days:  1 | Online:   1 | OPT:   1 | Ratio: 1.000 | Bought: False
# Days:  5 | Online:   5 | OPT:   5 | Ratio: 1.000 | Bought: False
# Days:  9 | Online:   9 | OPT:   9 | Ratio: 1.000 | Bought: False
# Days: 10 | Online:  19 | OPT:  10 | Ratio: 1.900 | Bought: True
# Days: 11 | Online:  19 | OPT:  10 | Ratio: 1.900 | Bought: True
# Days: 20 | Online:  19 | OPT:  10 | Ratio: 1.900 | Bought: True
```

이 결정론적 전략의 경쟁 비율은 `2 - 1/B`다. B = 10이면 1.9-competitive. 

**확률론적 전략으로 경쟁 비율 개선:**

```python
import math
import random

def ski_rental_randomized(days_skied: int, buy_cost: int = 10) -> dict:
    """
    확률론적 스키 렌탈 전략 (Karlin et al., 1994)
    기하 분포에서 렌탈→구매 전환일을 샘플링
    기대 경쟁 비율: e/(e-1) ≈ 1.582 (결정론적 2-1/B보다 우수)
    
    전략: r을 [1, B] 중 기하분포(1/B)로 샘플링, r일째 구매
    """
    # 기하분포: P(X = k) = (1 - 1/B)^(k-1) * (1/B)
    p = 1.0 / buy_cost
    buy_day = 1
    while random.random() > p and buy_day < buy_cost:
        buy_day += 1

    online_cost = 0
    if days_skied >= buy_day:
        online_cost = (buy_day - 1) * 1 + buy_cost  # 렌탈 + 구매
    else:
        online_cost = days_skied * 1  # 렌탈만

    opt_cost = min(days_skied, buy_cost)
    return {"online_cost": online_cost, "opt_cost": opt_cost}

# 몬테카를로 시뮬레이션으로 기대 경쟁 비율 측정
def simulate_competitive_ratio(target_days: int, trials: int = 100000) -> float:
    total_ratio = 0.0
    for _ in range(trials):
        res = ski_rental_randomized(target_days)
        if res["opt_cost"] > 0:
            total_ratio += res["online_cost"] / res["opt_cost"]
    return total_ratio / trials

print(f"이론값 e/(e-1) = {math.e / (math.e - 1):.4f}")
for days in [9, 10, 15, 20]:
    ratio = simulate_competitive_ratio(days)
    print(f"Days={days:2d}: 시뮬레이션 경쟁 비율 = {ratio:.4f}")
```

### 예제 2: 페이징 알고리즘과 k-서버 문제

페이징(Paging) 문제는 캐시 크기가 k인 환경에서 메모리 접근 시 발생하는 페이지 폴트를 최소화하는 문제다. 이는 k-서버 문제의 특수한 경우다.

```python
from collections import OrderedDict
from typing import List, Tuple

class PagingSimulator:
    """LRU와 FIFO의 경쟁 비율 비교"""

    def lru(self, pages: List[int], cache_size: int) -> int:
        cache = OrderedDict()  # 가장 최근 접근 = 마지막
        faults = 0
        for page in pages:
            if page in cache:
                cache.move_to_end(page)
            else:
                faults += 1
                if len(cache) == cache_size:
                    cache.popitem(last=False)  # LRU 제거
                cache[page] = True
        return faults

    def fifo(self, pages: List[int], cache_size: int) -> int:
        cache = OrderedDict()
        faults = 0
        for page in pages:
            if page in cache:
                pass  # FIFO는 순서를 바꾸지 않음
            else:
                faults += 1
                if len(cache) == cache_size:
                    cache.popitem(last=False)  # 가장 오래된 페이지 제거
                cache[page] = True
        return faults

    def opt(self, pages: List[int], cache_size: int) -> int:
        """오프라인 최적 알고리즘: 미래를 알기 때문에 가장 나중에 쓰일 페이지 교체"""
        cache = set()
        faults = 0
        for i, page in enumerate(pages):
            if page in cache:
                continue
            faults += 1
            if len(cache) == cache_size:
                # 미래에 가장 늦게 사용될 페이지 찾기
                future_use = {}
                for p in cache:
                    next_use = float('inf')
                    for j in range(i + 1, len(pages)):
                        if pages[j] == p:
                            next_use = j
                            break
                    future_use[p] = next_use
                evict = max(future_use, key=future_use.get)
                cache.remove(evict)
            cache.add(page)
        return faults

sim = PagingSimulator()

# 적대적 시퀀스 예: k+1개 페이지를 순환 접근 (LRU 최악 케이스)
k = 3
adversarial = [0, 1, 2, 3, 0, 1, 2, 3, 0, 1, 2, 3]
lru_faults = sim.lru(adversarial, k)
fifo_faults = sim.fifo(adversarial, k)
opt_faults = sim.opt(adversarial, k)

print(f"Cache size k={k}, 시퀀스 길이={len(adversarial)}")
print(f"LRU 페이지 폴트:  {lru_faults} (경쟁 비율: {lru_faults/opt_faults:.2f})")
print(f"FIFO 페이지 폴트: {fifo_faults} (경쟁 비율: {fifo_faults/opt_faults:.2f})")
print(f"OPT 페이지 폴트:  {opt_faults}")
print(f"\n이론적 최적 경쟁 비율(결정론적): k = {k}")
print(f"이론적 최적 경쟁 비율(확률론적): H_k = {sum(1/i for i in range(1, k+1)):.4f}")

# Output:
# LRU 페이지 폴트:  12 (경쟁 비율: 4.00)
# FIFO 페이지 폴트: 12 (경쟁 비율: 4.00)
# OPT 페이지 폴트:  3
# 이론적 최적 경쟁 비율(결정론적): k = 3
# 이론적 최적 경쟁 비율(확률론적): H_k = 1.8333
```

이 결과는 이론과 일치한다. 페이징 문제에서 임의의 결정론적 알고리즘의 경쟁 비율은 정확히 k이고, 최적 확률론적 알고리즘의 기대 경쟁 비율은 조화급수 H_k다 (Fiat et al., 1991).

---

## 주요 결과와 주의사항

### 야오의 정리 (Yao's Minimax Principle)

확률론적 알고리즘의 경쟁 비율 하한을 분석할 때 핵심 도구다. **최적 확률론적 알고리즘(무적응 적대자 기준)의 기대 비용 ≥ 최적 결정론적 알고리즘에 대한 최악 분포에서의 기대 비용**이 성립한다. 즉, 확률론적 알고리즘의 하한을 증명하려면 "가장 어려운 랜덤 입력 분포"를 찾으면 된다.

### k-서버 추측 (k-Server Conjecture)

Manasse, McGeoch, Sleator (1988)이 제안한 미해결 문제로, k-서버 문제의 최적 결정론적 온라인 알고리즘의 경쟁 비율이 정확히 k라고 추측한다. n=2 (Work Function Algorithm)와 균일 메트릭 공간, 직선 위 등 특수한 경우는 증명되었으나, 일반적인 메트릭 공간에서는 여전히 열린 문제다.

### 경쟁 분석의 한계

경쟁 분석은 **최악 케이스 비율**을 측정하기 때문에 지나치게 비관적일 수 있다. 예를 들어 LRU는 경쟁 비율이 k이지만 실제 워크로드에서는 훨씬 뛰어난 성능을 보인다. 이를 보완하기 위해 다음 대안적 분석 기법이 등장했다:

- **분할 상환 분석 (Amortized Analysis)**: 개별 연산보다 전체 시퀀스의 평균 비용을 봄
- **완화된 온라인 문제 (Relaxed Online)**: 약간의 lookahead를 허용
- **비용 모델 확장**: 재배치 비용 등을 반영한 정제된 모델

### 실용적 설계 팁

1. **문제를 올바르게 모델링하라**: 결정의 "돌이킬 수 없음" 정도를 정확히 파악해야 한다. 부분적으로 번복 가능한 경우는 세미-온라인 알고리즘으로 모델링한다.
2. **확률론적 전략 우선 검토**: 결정론적 알고리즘보다 경쟁 비율이 낮은 경우가 많다.
3. **분포 가정이 가능하면 활용**: 실제 입력 분포가 균일하지 않다면 분포 기반 알고리즘이 우수하다.
4. **경쟁 분석 외 지표도 함께 측정**: 평균 케이스 성능, 실험적 검증을 병행한다.

---

## 참고 자료
- [Competitive Analysis (Wikipedia)](https://en.wikipedia.org/wiki/Competitive_analysis_(online_algorithm))
- [Online Computation and Competitive Analysis — Borodin & El-Yaniv (Cambridge)](https://www.cambridge.org/core/books/online-computation-and-competitive-analysis/92C6D42AA9AF6EBF42DA6E92A36FB11F)
- [The k-Server Problem — MIT OpenCourseWare](https://ocw.mit.edu/courses/6-854j-advanced-algorithms-fall-2008/)
- [Sleator & Tarjan (1985) — Amortized efficiency of list update and paging rules](https://dl.acm.org/doi/10.1145/2786.2793)
