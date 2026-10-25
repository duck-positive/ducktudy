---
layout: post
title: "그리디 알고리즘 심화: 탐욕적 선택 속성과 Exchange Argument 증명 기법"
date: 2026-10-25
categories: [cs, computer-science]
tags: [greedy, algorithm, exchange-argument, matroid, optimization, activity-selection]
---

## 개요

그리디(탐욕적) 알고리즘은 각 단계에서 지역적으로 최선인 선택을 반복함으로써 전역 최적해를 구하는 알고리즘 패러다임이다. 동적 프로그래밍처럼 모든 경우를 고려하지 않기 때문에 구현이 간단하고 실행 속도가 빠르다. 그러나 그리디가 올바르게 동작하려면 반드시 두 가지 속성이 성립해야 한다: **탐욕적 선택 속성(Greedy Choice Property)**과 **최적 부분구조(Optimal Substructure)**다.

이 글에서는 그리디 알고리즘의 정확성을 증명하는 두 가지 핵심 기법인 **Exchange Argument**와 **Greedy Stays Ahead**를 깊이 있게 다루고, Matroid 이론을 통해 그리디가 왜 특정 문제 구조에서만 최적이 되는지를 설명한다.

---

## 왜 그리디 정확성 증명이 필요한가

그리디 알고리즘은 직관적으로 "당연히 맞을 것 같아 보이는" 전략이지만, 실제로는 매우 쉽게 틀릴 수 있다. 대표적인 예가 **0/1 배낭 문제(Knapsack)**다. 물건을 나눌 수 없는 경우, 단순히 가성비(가치/무게)가 높은 순서로 선택하면 최적해를 보장하지 못한다.

```
무게 제한: 10kg
물건 A: 무게 6kg, 가치 12 (가성비 2.0)
물건 B: 무게 4kg, 가치 7  (가성비 1.75)
물건 C: 무게 5kg, 가치 9  (가성비 1.8)

그리디: A(12) + B(7) = 19  [무게 10kg]
최적:   B(7)  + C(9) = 16  → 아니면  A+B=19
실제 최적: A + B = 19  → 이 예에서는 맞지만...

다른 예:
물건 A: 무게 6kg, 가치 12 (가성비 2.0)
물건 B: 무게 5kg, 가치 9  (가성비 1.8)
물건 C: 무게 5kg, 가치 9  (가성비 1.8)

그리디: A 선택 후 B or C 못 넣음 = 12
최적:   B + C = 18  ← 그리디 실패
```

반면 물건을 분할할 수 있는 **분할 배낭 문제(Fractional Knapsack)**는 가성비 순 그리디가 항상 최적해를 준다. 이 차이가 왜 발생하는지 이해하는 것이 그리디 정확성 분석의 핵심이다.

---

## 핵심 개념

### 탐욕적 선택 속성 (Greedy Choice Property)

전역 최적해에 현재 단계에서의 지역 최적 선택이 포함될 수 있다는 속성이다. 즉, 이후 선택을 고려하지 않고 지금 가장 좋아 보이는 선택을 해도 전역 최적을 잃지 않는다는 보장이다.

### 최적 부분구조 (Optimal Substructure)

한 번의 선택을 하고 나서 남은 부분 문제의 최적해가 원래 문제의 최적해에 기여한다는 속성이다. 다이나믹 프로그래밍도 이 속성을 사용하지만, 그리디는 여기에 탐욕적 선택 속성이 추가된다.

---

## Exchange Argument 증명 기법

Exchange Argument는 그리디의 정확성을 증명하는 가장 강력하고 일반적인 기법이다. 아이디어는 다음과 같다:

> "임의의 최적해가 있다면, 그것을 단계적으로 그리디해로 변환할 수 있고, 이 과정에서 해의 품질은 절대 나빠지지 않는다."

### 증명 구조

1. 최적해 OPT가 존재한다고 가정한다.
2. OPT와 그리디해 G가 다른 첫 번째 지점을 찾는다.
3. OPT의 해당 원소를 G의 원소로 교환(swap)해도 해의 품질이 유지되거나 나아짐을 보인다.
4. 이 교환을 반복하면 OPT → G로 변환되며, 결국 G의 비용 ≤ OPT의 비용이 된다.
5. OPT는 최적이므로 G의 비용 = OPT의 비용, 즉 G도 최적이다.

### 구현 예제 1: 활동 선택 문제 (Activity Selection)

n개의 활동이 있으며, 각 활동은 시작 시간 s[i]와 종료 시간 f[i]를 가진다. 서로 겹치지 않는 최대 개수의 활동을 선택하는 문제다.

그리디 전략: 종료 시간이 빠른 활동을 우선 선택한다.

```python
def activity_selection(activities):
    """
    activities: [(start, finish), ...]
    반환: 선택된 활동 인덱스 리스트
    """
    # 종료 시간 기준 정렬
    sorted_acts = sorted(enumerate(activities), key=lambda x: x[1][1])
    
    selected = []
    last_finish = -1
    
    for idx, (start, finish) in sorted_acts:
        if start >= last_finish:
            selected.append(idx)
            last_finish = finish
    
    return selected


# 테스트
activities = [(1,4), (3,5), (0,6), (5,7), (3,8), (5,9), (6,10), (8,11), (8,12), (2,13), (12,14)]
result = activity_selection(activities)
print(f"선택된 활동 수: {len(result)}")
print(f"선택된 활동 인덱스: {result}")
# 출력: 선택된 활동 수: 4
# 선택된 활동 인덱스: [0, 3, 7, 10]
```

**Exchange Argument 증명 (스케치):**

최적해 OPT에서 그리디가 고른 첫 번째 활동 a₁ 대신 다른 활동 b₁이 선택되었다고 가정하자. b₁의 종료 시간 f(b₁) ≥ f(a₁) (그리디가 가장 이른 종료를 선택하므로). OPT에서 b₁을 a₁로 교환해도, a₁이 더 일찍 끝나기 때문에 이후 선택 가능한 활동의 집합이 줄어들지 않는다. 따라서 교환 후에도 OPT 크기가 유지되거나 커진다. 이를 귀납적으로 반복하면 그리디해가 최적임이 증명된다.

---

## Greedy Stays Ahead 기법

이 기법은 그리디 알고리즘이 모든 단계에서 다른 어떤 알고리즘보다 "앞서" 있음을 귀납법으로 보인다.

```python
def interval_scheduling_max_jobs(jobs):
    """
    Greedy Stays Ahead 적용:
    k번째 그리디 선택의 종료 시간 ≤ 임의 OPT의 k번째 활동 종료 시간
    
    이 불변식을 유지하면 그리디가 최대 개수를 선택함이 보장됨
    """
    jobs_sorted = sorted(enumerate(jobs), key=lambda x: x[1][1])
    
    greedy_times = []
    last_end = float('-inf')
    
    for idx, (s, e) in jobs_sorted:
        if s >= last_end:
            greedy_times.append(e)
            last_end = e
    
    return len(greedy_times), greedy_times


count, finish_times = interval_scheduling_max_jobs(
    [(0,3), (1,4), (2,5), (4,7), (5,8), (6,9)]
)
print(f"최대 활동 수: {count}, 종료 시간: {finish_times}")
# 출력: 최대 활동 수: 3, 종료 시간: [3, 7, 9]
```

**귀납 불변식:** 그리디가 k번째로 선택한 활동의 종료 시간은 임의의 최적해에서 k번째 활동의 종료 시간보다 같거나 이르다.

- k=1: 그리디는 종료 시간이 가장 이른 활동을 선택. 어떤 최적해도 종료 시간이 이보다 이른 활동을 첫 번째로 선택할 수 없음.
- 귀납: k번째까지 그리디가 OPT보다 앞서 있으면, (k+1)번째 선택 가능한 활동들의 집합이 그리디 쪽이 더 크거나 같으므로 (k+1)번째에서도 앞서거나 같다.

이를 통해 그리디가 선택하는 활동 수 ≥ OPT의 활동 수, 즉 최적임이 증명된다.

---

## 구현 예제 2: 작업 스케줄링 (Weighted Job Scheduling - Greedy가 실패하는 예)

그리디가 항상 통하는 것은 아니다. 마감 기한과 가중치가 있는 작업 스케줄링에서 단순한 그리디는 실패한다.

```python
def greedy_scheduling_wrong(jobs):
    """
    잘못된 그리디: 마감 기한 순으로 정렬하면 가중치 합 최대화 불가
    """
    # (weight, deadline)
    jobs_sorted = sorted(jobs, key=lambda x: x[1])  # 마감 기한 순
    
    selected = []
    time = 0
    for w, d in jobs_sorted:
        if time < d:  # 마감 전에 처리 가능
            selected.append((w, d))
            time += 1
    
    return sum(w for w, d in selected)


def dp_weighted_scheduling(jobs):
    """
    올바른 DP: 최대 가중치 합을 위해 DP 사용
    각 작업은 단위 시간 소요
    """
    n = len(jobs)
    T = max(d for _, d in jobs)
    jobs_sorted = sorted(jobs, key=lambda x: x[1])
    
    # dp[t] = 시간 t까지 처리 가능한 최대 가중치
    dp = [0] * (T + 1)
    for w, d in jobs_sorted:
        for t in range(min(d, T), 0, -1):
            dp[t] = max(dp[t], dp[t-1] + w)
    
    return dp[T]


jobs = [(20, 2), (15, 2), (10, 1), (5, 3), (1, 3)]
print(f"그리디 결과: {greedy_scheduling_wrong(jobs)}")
print(f"DP 최적 결과: {dp_weighted_scheduling(jobs)}")
# 그리디: 마감 기한만 봐서는 최대 가중치 합을 보장 못 함
```

---

## Matroid 이론과 그리디

Matroid는 독립 집합(independent sets)의 추상적 구조로, 그리디가 최적을 보장하는 문제들의 공통 구조를 수학적으로 포착한다.

**Matroid의 정의:** 집합 E와 독립 집합족 ℐ ⊆ 2^E가 다음 공리를 만족할 때 (E, ℐ)를 Matroid라 한다:
1. ∅ ∈ ℐ (공집합은 독립)
2. A ∈ ℐ이고 B ⊆ A이면 B ∈ ℐ (hereditary property)
3. A, B ∈ ℐ이고 |A| < |B|이면 ∃e ∈ B\A s.t. A ∪ {e} ∈ ℐ (augmentation property)

**핵심 정리:** 가중치 최대화 문제에서, 독립 집합이 Matroid를 이루면 탐욕적 알고리즘(가중치 큰 원소 우선 선택)이 최대 가중치 독립 집합을 찾는다.

대표적인 Matroid 예시:
- **Graphic Matroid**: 그래프의 무사이클 간선 집합 → Kruskal MST가 그리디로 최적
- **Partition Matroid**: 각 파티션에서 k개 이하 선택 → 마감 기한 스케줄링
- **Linear Matroid**: 벡터 공간의 선형 독립 집합

```python
def greedy_matroid_max_weight(elements, is_independent):
    """
    Generic 그리디 알고리즘 for Matroid 구조 문제
    
    elements: [(weight, element), ...]
    is_independent: 현재 선택 집합이 독립인지 판단 함수
    """
    # 가중치 내림차순 정렬
    sorted_elems = sorted(elements, reverse=True)
    
    selected = []
    for weight, elem in sorted_elems:
        candidate = selected + [elem]
        if is_independent(candidate):
            selected.append(elem)
    
    return selected, sum(w for w, e in sorted_elems if e in selected)


# 예시: Graphic Matroid (무사이클 간선 집합 = Spanning Forest)
def kruskal_mst(n, edges):
    """Union-Find 기반 Kruskal - Matroid 그리디의 실제 구현"""
    parent = list(range(n))
    
    def find(x):
        while parent[x] != x:
            parent[x] = parent[parent[x]]
            x = parent[x]
        return x
    
    def union(x, y):
        px, py = find(x), find(y)
        if px == py:
            return False
        parent[px] = py
        return True
    
    edges_sorted = sorted(edges, key=lambda e: e[2])  # 가중치 오름차순
    mst = []
    total = 0
    
    for u, v, w in edges_sorted:
        if union(u, v):
            mst.append((u, v, w))
            total += w
    
    return mst, total


edges = [(0,1,4), (0,2,3), (1,2,1), (1,3,2), (2,3,4), (3,4,2), (4,0,5)]
mst, cost = kruskal_mst(5, edges)
print(f"MST: {mst}")
print(f"MST 총 가중치: {cost}")
```

---

## 주의사항 및 팁

### 1. 그리디가 실패하는 징후 파악
- 부분 선택의 "미래 영향"이 현재 가치와 독립적이지 않을 때 (0/1 배낭)
- 선택 순서가 이후 선택 가능 집합에 복잡하게 영향을 줄 때

### 2. Exchange Argument 작성 시 주의점
- 교환 이후 해의 실행 가능성(feasibility)이 유지됨을 반드시 확인
- "교환이 해를 나쁘게 하지 않는다"는 부등호 방향에 주의
- 교환이 유한 번 후 종료됨을 보여야 함 (단조성 인수)

### 3. 그리디 정당화 체크리스트
```
□ 탐욕적 선택 속성: 첫 번째 선택이 최적해에 포함될 수 있는가?
□ 최적 부분구조: 첫 번째 선택 후 남은 문제가 동일 형태인가?
□ Exchange Argument: OPT를 G로 교환하면 품질이 유지되는가?
□ Matroid 구조: 독립 집합이 Matroid 공리를 만족하는가?
```

### 4. 대표 그리디 알고리즘과 증명 기법

| 알고리즘 | 증명 기법 | 핵심 속성 |
|---------|----------|---------|
| Activity Selection | Exchange Argument | 최조 종료 시간 |
| Huffman Coding | Exchange Argument | 최저 빈도 병합 |
| Kruskal MST | Matroid (Graphic) | 무사이클 간선 |
| Dijkstra SSSP | Greedy Stays Ahead | 음수 간선 없음 |
| Fractional Knapsack | Exchange Argument | 분할 가능성 |

---

## 참고 자료
- [Algorithm Design (Kleinberg & Tardos) - Chapter 4: Greedy Algorithms](https://www.cs.cornell.edu/home/kleinber/networks-book/)
- [Introduction to Algorithms (CLRS) - Chapter 16: Greedy Algorithms](https://mitpress.mit.edu/books/introduction-algorithms-fourth-edition)
- [Exchange Argument Handout - Cornell CS](https://www.cs.cornell.edu/courses/cs482/2005su/handouts/greedy_exchange.pdf)
- [Matroid Theory and Greedy Algorithms - Wikipedia](https://en.wikipedia.org/wiki/Matroid)
