---
layout: post
title: "Work-Span 모델과 병렬 알고리즘 설계: Brent 정리와 병렬 프리픽스 합"
date: 2026-08-22
categories: [cs, computer-science]
tags: [parallel-algorithms, work-span, brent-theorem, prefix-sum, parallel-computing, dag, multicore]
---

멀티코어 CPU가 보편화된 오늘날, 알고리즘을 단순히 "빠른 순차 알고리즘"으로만 평가하는 시대는 지났습니다. 병렬 환경에서 알고리즘의 성능을 분석하고 최적화하려면 **Work-Span 모델**을 이해해야 합니다. 이 모델은 병렬 알고리즘의 이론적 한계를 정형화하고, **Brent의 정리**를 통해 실제 p개의 프로세서에서 달성 가능한 실행 시간을 예측합니다.

---

## Work-Span 모델이란?

병렬 알고리즘의 실행을 **DAG(Directed Acyclic Graph)** 로 모델링합니다. 각 노드는 기본 연산 단위(O(1) 시간), 간선은 데이터 의존성을 나타냅니다.

이 DAG에서 두 가지 핵심 지표를 정의합니다:

### Work (작업량, W)

DAG의 모든 노드 수, 즉 알고리즘이 수행하는 총 연산 횟수입니다. 프로세서가 1개일 때의 순차 실행 시간에 해당합니다.

```
W = (DAG의 전체 노드 수)
```

### Span (스팬, D 또는 깊이)

DAG에서 소스부터 싱크까지의 **가장 긴 경로**의 길이입니다. 이는 **임계 경로(critical path)** 라고도 불리며, 아무리 많은 프로세서를 사용해도 줄일 수 없는 최소 실행 시간입니다.

```
D = (DAG의 longest path)
```

### 직관적 이해

정수 배열 `[a, b, c, d]`의 합을 구하는 병렬 트리 방식을 생각해봅시다:

```
레벨 0: a     b     c     d
레벨 1:   a+b         c+d
레벨 2:       (a+b)+(c+d)
```

- Work = 3 (덧셈 3번, O(n))
- Span = 2 (로그 깊이, O(log n))
- 순차 합산: Work = 3, Span = 3

---

## Brent의 정리 (Brent's Theorem)

**정리**: W개의 연산, D의 스팬을 가지는 병렬 알고리즘을 p개의 프로세서로 실행할 때 실행 시간 T_p는:

```
T_p ≤ W/p + D
```

**해석**:
- `W/p`: 완벽한 부하 분산이 이루어질 때의 이상적 시간
- `D`: 아무리 병렬화해도 줄일 수 없는 최소 시간(임계 경로)
- 실제 시간은 두 항의 합 이하임을 보장

**스피드업(Speedup)**:
```
Speedup = T_1 / T_p ≤ p   (Amdahl의 법칙과 유사)
```

**효율성(Efficiency)**:
```
E = Speedup / p = T_1 / (p × T_p)
```

이상적으로는 E = 1 (100% 효율)이지만, 스팬 D가 클수록 효율이 떨어집니다.

### Work-효율 알고리즘 (Work-Efficient)

순차 알고리즘의 Work와 동일한 Work를 갖는 병렬 알고리즘을 **Work-효율적**이라고 합니다. 예를 들어:
- 배열 합산: 순차 O(n) vs 병렬 O(n) work, O(log n) span → Work-효율적
- 일부 나이브한 병렬 알고리즘은 O(n log n) work를 쓰기도 함 → Work-비효율적

---

## 병렬 프리픽스 합 (Parallel Prefix Sum / Scan)

병렬 알고리즘에서 가장 중요한 기본 연산 중 하나입니다.

**문제**: 배열 `A[0..n-1]`에 대해 `P[i] = A[0] ⊕ A[1] ⊕ ... ⊕ A[i]` 를 계산 (⊕는 결합 연산).

순차 알고리즘은 O(n) work, O(n) span입니다. 병렬 알고리즘 목표: O(n) work, O(log n) span.

### Brent-Kung 알고리즘

두 단계로 구성됩니다:

**Up-sweep (Reduce) 단계**: 트리를 아래에서 위로 올라가며 부분합 계산

```
초기:    A[0]  A[1]  A[2]  A[3]  A[4]  A[5]  A[6]  A[7]
레벨1:   A[0]  A[0:1]  A[2]  A[2:3]  A[4]  A[4:5]  A[6]  A[6:7]
레벨2:   A[0]  A[0:1]  A[2]  A[0:3]  A[4]  A[4:5]  A[6]  A[4:7]
레벨3:   A[0]  A[0:1]  A[2]  A[0:3]  A[4]  A[4:5]  A[6]  A[0:7]
```

**Down-sweep 단계**: 루트에서 아래로 내려가며 접두사 합 전파

Work = O(n), Span = O(log n)

---

## 코드 예제 1: Python으로 구현하는 병렬 프리픽스 합

```python
import multiprocessing as mp
from typing import Callable

def parallel_prefix_sum(arr: list[int], p: int = None) -> list[int]:
    """
    Work-효율적인 병렬 프리픽스 합 구현
    Work: O(n), Span: O(log n)
    """
    n = len(arr)
    if n == 0:
        return []
    if p is None:
        p = mp.cpu_count()
    
    result = arr[:]
    
    # Up-sweep (Reduce) phase
    step = 1
    while step < n:
        new_result = result[:]
        indices = list(range(step * 2 - 1, n, step * 2))
        
        # 병렬로 처리 (멀티프로세싱 대신 의미를 명확히 하기 위해 순차 시뮬레이션)
        for i in indices:
            new_result[i] = result[i - step] + result[i]
        result = new_result
        step *= 2
    
    # Down-sweep phase
    result[n - 1] = 0  # exclusive scan의 경우 0으로 초기화
    step = n // 2
    while step >= 1:
        new_result = result[:]
        indices = list(range(step * 2 - 1, n, step * 2))
        for i in indices:
            t = result[i - step]
            new_result[i - step] = result[i]
            new_result[i] = result[i] + t
        result = new_result
        step //= 2
    
    # exclusive -> inclusive scan 변환
    return [result[i + 1] if i + 1 < n else (result[n - 1] + arr[n - 1]) 
            for i in range(n)]


def parallel_prefix_generic(arr: list, op: Callable, identity, p: int = None) -> list:
    """일반 결합 연산에 대한 병렬 프리픽스 (inclusive scan)"""
    n = len(arr)
    if n == 1:
        return [arr[0]]
    
    # Divide: 홀수 인덱스끼리 pairwise 연산
    pairs = [op(arr[2*i], arr[2*i+1]) for i in range(n // 2)]
    if n % 2 == 1:
        pairs.append(arr[-1])
    
    # Conquer: 재귀 호출
    prefix_pairs = parallel_prefix_generic(pairs, op, identity, p)
    
    # Combine: 결과 합치기
    result = [None] * n
    result[0] = arr[0]
    for i in range(1, n):
        if i % 2 == 1:  # 홀수 위치: prefix_pairs에서 직접 읽기
            result[i] = prefix_pairs[i // 2]
        else:  # 짝수 위치: 이전 홀수 prefix에 현재 값 적용
            result[i] = op(prefix_pairs[i // 2 - 1], arr[i])
    return result


# 테스트
if __name__ == "__main__":
    import time
    
    # 기본 프리픽스 합
    arr = [1, 2, 3, 4, 5, 6, 7, 8]
    result = parallel_prefix_sum(arr)
    print(f"입력:  {arr}")
    print(f"프리픽스 합: {result}")
    # 기대: [1, 3, 6, 10, 15, 21, 28, 36]
    
    # 일반 결합 연산 (최대값 프리픽스)
    max_prefix = parallel_prefix_generic(arr, max, float('-inf'))
    print(f"최대값 프리픽스: {max_prefix}")
    # 기대: [1, 2, 3, 4, 5, 6, 7, 8]
    
    # 곱 프리픽스
    import operator
    prod_prefix = parallel_prefix_generic([1, 2, 3, 4, 5], operator.mul, 1)
    print(f"곱 프리픽스: {prod_prefix}")
    # 기대: [1, 2, 6, 24, 120]
    
    # 성능 비교 (n=10^6)
    n = 1_000_000
    big_arr = list(range(n))
    
    t0 = time.perf_counter()
    seq_result = [sum(big_arr[:i+1]) for i in range(min(100, n))]
    t1 = time.perf_counter()
    print(f"\n순차 프리픽스 합 (처음 100개): {t1-t0:.4f}s")
```

---

## 코드 예제 2: Work-Span 분석 도구 (C++ 의사코드)

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <cmath>

struct AlgorithmMetrics {
    long long work;   // W
    long long span;   // D
    int processors;   // p
    
    // Brent's theorem 상한
    double brent_upper_bound() const {
        return (double)work / processors + span;
    }
    
    // 이상적 스피드업
    double ideal_speedup() const {
        return (double)work / brent_upper_bound();
    }
    
    // 효율성 (0~1)
    double efficiency() const {
        return ideal_speedup() / processors;
    }
    
    // 병렬성 (parallelism = W/D, 의미 있게 활용할 수 있는 최대 프로세서 수)
    double parallelism() const {
        return (double)work / span;
    }
    
    void print() const {
        std::cout << "=== Work-Span 분석 ===\n";
        std::cout << "  Work (W)     : " << work << "\n";
        std::cout << "  Span (D)     : " << span << "\n";
        std::cout << "  Processors   : " << processors << "\n";
        std::cout << "  T_p ≤         : " << brent_upper_bound() << "\n";
        std::cout << "  Speedup ≤    : " << ideal_speedup() << "x\n";
        std::cout << "  Efficiency   : " << efficiency() * 100 << "%\n";
        std::cout << "  Parallelism  : " << parallelism() << "\n";
        
        if (processors > parallelism()) {
            std::cout << "  ⚠ 경고: 프로세서 수(" << processors 
                      << ")가 병렬성(" << parallelism() 
                      << ")을 초과. 스케일 이득 없음.\n";
        }
    }
};

// 병렬 머지소트의 Work-Span 분석
// Work: O(n log n) — 순차와 동일 (Work-효율적)
// Span: O(log² n) — 각 레벨이 O(log n) span을 가진 병렬 머지
AlgorithmMetrics analyze_parallel_mergesort(int n) {
    return AlgorithmMetrics{
        .work = (long long)(n * std::log2(n)),   // O(n log n)
        .span = (long long)(std::log2(n) * std::log2(n)),  // O(log² n)
        .processors = 8
    };
}

// 병렬 BFS (PBFS)
// Work: O(V + E), Span: O(D * log n) where D = BFS depth
AlgorithmMetrics analyze_parallel_bfs(int V, int E, int depth) {
    return AlgorithmMetrics{
        .work = V + E,
        .span = (long long)(depth * std::log2(V)),
        .processors = 16
    };
}

int main() {
    std::cout << "[병렬 머지소트 n=10^6]\n";
    auto ms = analyze_parallel_mergesort(1'000'000);
    ms.print();
    
    std::cout << "\n[병렬 BFS V=10^5, E=10^6, depth=20]\n";
    auto bfs = analyze_parallel_bfs(100'000, 1'000'000, 20);
    bfs.print();
    
    // Work-효율성 비교
    std::cout << "\n=== 알고리즘 비교 ===\n";
    struct Case { std::string name; long long work, span; };
    std::vector<Case> cases = {
        {"순차 합산",        1000, 1000},
        {"나이브 병렬 합산", 1000 * 10, 10},  // work가 10배 증가 (비효율)
        {"Brent-Kung 합산",  2000, 20},        // 2n work, log n span (효율적)
    };
    
    for (auto& c : cases) {
        double parallelism = (double)c.work / c.span;
        std::cout << c.name << ": W=" << c.work << " D=" << c.span 
                  << " 병렬성=" << parallelism << "\n";
    }
    return 0;
}
```

---

## 주요 병렬 알고리즘의 Work-Span 정리

| 알고리즘 | Work | Span | Work-효율적? |
|---------|------|------|-------------|
| 병렬 합산 (트리) | O(n) | O(log n) | ✓ |
| 병렬 프리픽스 합 | O(n) | O(log n) | ✓ |
| 병렬 머지소트 | O(n log n) | O(log² n) | ✓ |
| 병렬 퀵소트 (균형) | O(n log n) | O(log² n) | ✓ |
| 나이브 행렬 곱 | O(n³) | O(log n) | ✓ |
| 병렬 BFS | O(V+E) | O(D·log V) | ✓ |
| 나이브 병렬 합산 | O(n log n) | O(log n) | ✗ |

---

## 주의사항 및 팁

### Span이 Work보다 더 중요한 경우
프로세서 수 p가 매우 많을 때는 `W/p` 항이 작아지고 D가 병목이 됩니다. 이 경우 Work를 약간 늘리더라도 Span을 줄이는 것이 유리합니다. Brent 정리의 두 항 중 어느 쪽이 병목인지를 분석해야 합니다.

### 캐시 효율성
Work-Span 모델은 메모리 계층을 무시합니다. 실제 멀티코어 환경에서는 캐시 지역성(cache locality)이 이론적 예측과 실제 성능의 격차를 만들 수 있습니다. **Cache-oblivious 병렬 알고리즘**은 이를 해결하는 한 방법입니다.

### 실무 병렬 프레임워크
- **C++**: `std::execution::par_unseq` (TBB 기반), Intel TBB
- **Java**: ForkJoinPool (Work-Stealing 스케줄러)
- **Python**: `multiprocessing`, `concurrent.futures`
- **GPU**: CUDA에서 parallel scan은 cuDNN 등의 핵심 primitive

### Work-Stealing 스케줄러와의 연결
Brent의 정리에서 T_p ≤ W/p + D를 실제로 달성하려면 작업이 프로세서에 균등 분배되어야 합니다. Cilk의 **Work-Stealing** 스케줄러는 이를 확률론적으로 보장하며, 기대 실행 시간이 O(W/p + D)임이 증명되어 있습니다.

---

## 참고 자료

- [Parallel Algorithms — Guy E. Blelloch and Bruce M. Maggs (CMU)](https://www.cs.cmu.edu/~guyb/papers/BM04.pdf)
- [Overview, Models of Computation, Brent's Theorem — Stanford CME323](https://stanford.edu/~rezab/dao/notes/lecture01/cme323_lec1.pdf)
- [Parallel Prefix Sum Algorithm Overview — Emergent Mind](https://www.emergentmind.com/topics/parallel-prefix-sum-algorithm)
- [Parallel Algorithms — MIT 6.006 / 6.046](https://ocw.mit.edu/courses/6-046j-design-and-analysis-of-algorithms-spring-2015/)
