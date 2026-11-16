---
layout: post
title: "작업-범위 모델(Work-Span Model) 완전 정복: 병렬 알고리즘의 성능을 수학적으로 분석하는 방법"
date: 2026-11-16
categories: [cs, computer-science]
tags: [parallel-algorithms, work-span-model, cilk, parallel-prefix-sum, brent-theorem, concurrency, fork-join]
---

## 개념 설명

현대 컴퓨터는 수십~수백 개의 코어를 가집니다. 그러나 더 많은 코어를 사용한다고 해서 항상 그에 비례하는 성능 향상이 이루어지지는 않습니다. 왜 P개의 프로세서를 사용해도 P배의 속도 향상을 얻기 어려울까요? 이를 이해하고 예측하기 위한 수학적 도구가 **작업-범위 모델(Work-Span Model)**입니다.

작업-범위 모델(또는 Work-Depth 모델)은 1990년대 Guy Blelloch와 Robert Harper가 정립한 병렬 알고리즘 분석 프레임워크입니다. PRAM(Parallel Random Access Machine) 모델의 단순함을 계승하면서도, Cilk·Java ForkJoin·OpenMP 등 실제 병렬 런타임의 성능을 정확히 예측합니다.

**핵심 측정값 두 가지:**

- **작업(Work) W(n)**: 계산 그래프의 전체 연산 수. 1개의 프로세서로 실행했을 때의 순차 시간과 같습니다.
- **범위(Span) S(n)** (또는 Depth, Critical Path Length): 계산 그래프에서 가장 긴 의존성 경로의 길이. 무한히 많은 프로세서를 사용해도 이보다 빠를 수 없는 하한값입니다.

**병렬성(Parallelism)**: 이상적인 최대 속도 향상 가능량.

```
병렬성 = W(n) / S(n)
```

병렬성이 P보다 훨씬 크면 P개의 프로세서를 효율적으로 활용할 수 있습니다. 병렬성이 P보다 작으면 일부 프로세서는 대기하게 됩니다.

---

## 왜 필요한가

### Amdahl의 법칙의 한계

전통적인 **Amdahl의 법칙**은 "순차 부분이 s%인 프로그램은 아무리 많은 프로세서를 사용해도 1/s 이상 빠르게 할 수 없다"고 말합니다. 그러나 이 법칙은 단순히 순차 대 병렬 비율만을 고려하며, 알고리즘의 세부 구조를 무시합니다.

작업-범위 모델은 더 정확합니다.

```
예시: 순차 합산 vs 병렬 트리 합산 (n=16 elements)

순차 합산:
  Work = O(n), Span = O(n), 병렬성 = 1

병렬 트리 합산:
  Work = O(n), Span = O(log n), 병렬성 = n/log n

16개 요소의 경우: 병렬성 ≈ 16/4 = 4
→ 4개의 프로세서를 100% 활용 가능
```

### Brent의 스케줄링 정리

**Brent의 정리(Brent's Theorem)**는 작업-범위 모델의 핵심 결과입니다.

P개의 프로세서로 그리디 스케줄러를 사용할 때, 실행 시간 T_P는:

```
T_P ≤ W/P + S
```

여기서 W는 작업, S는 범위, P는 프로세서 수입니다.

Cilk의 작업 훔치기(Work-Stealing) 스케줄러는 이 경계를 **기대값으로** 달성합니다.

```
E[T_P] = W/P + O(S)
```

이 정리의 중요성: 알고리즘의 W와 S를 알면, 임의의 P에 대한 실행 시간을 예측할 수 있습니다.

---

## 계산 그래프(DAG)와 병렬 프리미티브

병렬 계산은 **방향 비순환 그래프(DAG)**로 표현됩니다. 각 노드는 하나의 연산이고, 에지는 의존성(앞 연산의 결과가 뒤 연산의 입력)을 나타냅니다.

**병렬 프리미티브:**

- **spawn**: 새 병렬 태스크를 생성 (Cilk의 `cilk_spawn`)
- **sync**: 모든 자식 태스크가 완료될 때까지 대기 (`cilk_sync`)
- **parallel-for**: 반복을 병렬로 실행

```
fork-join 패턴의 작업-범위:

              [시작]
             /      \
      [왼쪽 절반]  [오른쪽 절반]
             \      /
              [결합]

Work  = W(n/2) + W(n/2) + O(1) → T(n) = 2T(n/2) + O(1) → W = O(n)
Span  = S(n/2) + O(1)          → T(n) = T(n/2) + O(1)  → S = O(log n)
병렬성 = O(n/log n)
```

---

## 핵심 병렬 알고리즘 분석

### 병렬 프리픽스 합(Parallel Prefix Sum / Scan)

프리픽스 합은 `result[i] = a[0] + a[1] + ... + a[i]`를 계산하는 연산입니다. 병렬 스캔 알고리즘은 GPU, CUDA, 병렬 정렬 등 수많은 곳에서 핵심 프리미티브로 사용됩니다.

**업-스윕(Up-Sweep) + 다운-스윕(Down-Sweep)** 2단계 알고리즘:

```
업-스윕 단계 (부분 합 트리 구성):
레벨 0: [1, 2, 3, 4, 5, 6, 7, 8]
레벨 1: [3,  7,  11, 15]
레벨 2: [10,     26]
레벨 3: [36]              ← 전체 합

다운-스윕 단계 (프리픽스 전파):
레벨 3: [0]
레벨 2: [0,  10]
레벨 1: [0, 3, 10, 21]
레벨 0: [0, 1, 3, 6, 10, 15, 21, 28]  ← 프리픽스 합

Work = O(n)      (각 레벨마다 n/2 연산, log n 레벨)
Span = O(log n)  (각 레벨을 병렬로 처리)
병렬성 = O(n/log n)
```

---

## 실제 구현 예제

### 예제 1: 순차 vs 병렬 프리픽스 합 (Python)

```python
import concurrent.futures
import time

def sequential_prefix_sum(arr):
    """순차 프리픽스 합 — Work: O(n), Span: O(n)"""
    result = [0] * len(arr)
    running = 0
    for i, x in enumerate(arr):
        running += x
        result[i] = running
    return result

def parallel_prefix_sum(arr):
    """
    병렬 프리픽스 합 (업-스윕 + 다운-스윕)
    Work: O(n), Span: O(log n)
    """
    n = len(arr)
    # 2의 제곱수로 패딩
    size = 1
    while size < n:
        size <<= 1
    data = arr[:] + [0] * (size - n)
    
    # 업-스윕: 부분 합 트리 구성
    # 각 레벨을 독립적으로 병렬 처리 가능
    step = 1
    while step < size:
        with concurrent.futures.ThreadPoolExecutor() as executor:
            futures = []
            for i in range(step - 1, size, step * 2):
                right = i + step
                if right < size:
                    left_val = data[i]
                    right_val = data[right]
                    futures.append(executor.submit(lambda l, r: l + r, left_val, right_val))
            
            idx = 0
            for i in range(step - 1, size, step * 2):
                right = i + step
                if right < size and idx < len(futures):
                    data[right] = futures[idx].result()
                    idx += 1
        step <<= 1
    
    # 다운-스윕: 프리픽스 전파
    data[size - 1] = 0
    step = size >> 1
    while step > 0:
        with concurrent.futures.ThreadPoolExecutor() as executor:
            new_data = data[:]
            futures = []
            indices = []
            for i in range(step - 1, size, step * 2):
                right = i + step
                if right < size:
                    left_val = data[i]
                    right_val = data[right]
                    futures.append((i, right, left_val, right_val))
            
            for (li, ri, lv, rv) in futures:
                new_data[li] = rv           # 왼쪽 ← 오른쪽 이전 값
                new_data[ri] = lv + rv      # 오른쪽 ← 왼쪽 + 오른쪽
            data = new_data
        step >>= 1
    
    # 원소 하나씩 더하기 (exclusive → inclusive 변환)
    result = [data[i] + arr[i] for i in range(n)]
    return result

# 성능 측정
import random
arr = [random.randint(1, 10) for _ in range(1024)]

t0 = time.time()
seq_result = sequential_prefix_sum(arr)
t1 = time.time()
par_result = parallel_prefix_sum(arr)
t2 = time.time()

print(f"배열 크기: {len(arr)}")
print(f"순차 프리픽스 합: {t1-t0:.6f}초")
print(f"병렬 프리픽스 합: {t2-t1:.6f}초")
print(f"결과 일치: {seq_result == par_result}")
print(f"\n처음 10개 결과 (순차): {seq_result[:10]}")
print(f"처음 10개 결과 (병렬): {par_result[:10]}")
```

### 예제 2: 작업-범위 분석 도구와 병렬 병합 정렬

```python
import concurrent.futures
import threading

# 작업과 범위를 추적하는 계측(instrumentation) 클래스
class WorkSpanTracker:
    def __init__(self):
        self._lock = threading.Lock()
        self.work = 0      # 총 연산 수
        self.max_span = 0  # 임계 경로 길이 (범위)
    
    def add_work(self, amount=1):
        with self._lock:
            self.work += amount
    
    def update_span(self, path_length):
        with self._lock:
            self.max_span = max(self.max_span, path_length)

tracker = WorkSpanTracker()

def parallel_merge_sort(arr, depth=0):
    """
    병렬 병합 정렬
    Work:  O(n log n)
    Span:  O(log² n)  — 각 레벨 병렬화 + 병합 O(log n)
    병렬성: O(n / log n)
    """
    n = len(arr)
    tracker.add_work(1)  # 분할 비용

    if n <= 1:
        tracker.update_span(depth)
        return arr[:]
    
    mid = n // 2
    left_arr = arr[:mid]
    right_arr = arr[mid:]
    
    if n > 32:  # 작은 배열은 순차 처리 (오버헤드 방지)
        with concurrent.futures.ThreadPoolExecutor(max_workers=2) as executor:
            future_left  = executor.submit(parallel_merge_sort, left_arr, depth + 1)
            future_right = executor.submit(parallel_merge_sort, right_arr, depth + 1)
            left  = future_left.result()
            right = future_right.result()
    else:
        left  = parallel_merge_sort(left_arr, depth + 1)
        right = parallel_merge_sort(right_arr, depth + 1)
    
    # 병합 (O(n) work, O(log n) span with parallel merge)
    merged = []
    i, j = 0, 0
    while i < len(left) and j < len(right):
        tracker.add_work(1)
        if left[i] <= right[j]:
            merged.append(left[i]); i += 1
        else:
            merged.append(right[j]); j += 1
    merged.extend(left[i:])
    merged.extend(right[j:])
    tracker.update_span(depth + len(merged))
    return merged

# 테스트
import random, math
arr = [random.randint(1, 1000) for _ in range(256)]
tracker = WorkSpanTracker()

sorted_arr = parallel_merge_sort(arr)
print(f"입력 크기: {len(arr)}")
print(f"정렬 결과 올바름: {sorted_arr == sorted(arr)}")
print(f"측정된 작업(Work): {tracker.work}")
print(f"이론적 Work O(n log n): {len(arr) * int(math.log2(len(arr)))}")
print()

# 작업-범위 모델 요약
algorithms = [
    ("순차 프리픽스 합",  "O(n)",      "O(n)",      "O(1)"),
    ("병렬 프리픽스 합",  "O(n)",      "O(log n)",  "O(n/log n)"),
    ("순차 병합 정렬",    "O(n log n)","O(n log n)","O(1)"),
    ("병렬 병합 정렬",    "O(n log n)","O(log² n)", "O(n/log n)"),
    ("순차 BFS",         "O(V+E)",    "O(V+E)",    "O(1)"),
    ("병렬 BFS",         "O(V+E)",    "O(D log V)","O((V+E)/(D log V))"),
    ("행렬 곱셈 (순차)", "O(n³)",     "O(n³)",     "O(1)"),
    ("행렬 곱셈 (병렬)", "O(n³)",     "O(log n)",  "O(n³/log n)"),
]

print(f"{'알고리즘':<20} {'Work':<15} {'Span':<15} {'병렬성':<20}")
print("-" * 70)
for name, work, span, par in algorithms:
    print(f"{name:<20} {work:<15} {span:<15} {par:<20}")
```

---

## 실전 응용: Cilk와 OpenMP

### Cilk 스타일 병렬 프로그래밍 (C/C++)

```c
#include <cilk/cilk.h>
#include <stdio.h>

// 병렬 피보나치 — Work: O(φⁿ), Span: O(n), 병렬성: O(φⁿ/n)
long fib(int n) {
    if (n < 2) return n;
    
    long x = cilk_spawn fib(n - 1);  // 별도 태스크로 실행
    long y = fib(n - 2);              // 현재 태스크에서 실행
    cilk_sync;                        // 모든 spawn 완료 대기
    return x + y;
}

// 병렬 프리픽스 합 (Cilk)
// Span: O(log n), 최적 Work: O(n)
void parallel_scan(int* arr, int* out, int n) {
    if (n == 1) { out[0] = arr[0]; return; }
    
    int* temp = malloc(n/2 * sizeof(int));
    // 병렬 짝수 위치 합산
    cilk_for (int i = 0; i < n/2; i++)
        temp[i] = arr[2*i] + arr[2*i+1];
    
    parallel_scan(temp, temp, n/2);  // 재귀
    
    // 병렬 결과 전파
    cilk_for (int i = 0; i < n; i++) {
        if (i == 0) out[0] = arr[0];
        else if (i % 2 == 1) out[i] = temp[i/2];
        else out[i] = temp[i/2 - 1] + arr[i];
    }
    free(temp);
}
```

---

## 작업-범위 분석 방법론

알고리즘을 설계할 때 작업-범위를 체계적으로 분석하는 방법:

**재귀 관계(Recurrence Relations)로 분석:**

```
W(n) = a·W(n/b) + f(n)   → 마스터 정리로 해결
S(n) = S(n/b) + g(n)     → 보통 O(log^k n) 형태
```

**병렬 for-loop:**
- `parallel-for i = 1 to n: O(1) work per iter`
- Work = O(n), Span = O(log n) (바이너리 트리로 스케줄링)

**spawn-sync 패턴:**
- spawn/sync가 k회 → Span += O(k)
- 각 spawn이 S(n/k) span → 전체 Span = max(S(n/k)) + O(k)

---

## 주의사항과 팁

**1. 작업 효율(Work Efficiency)**: 병렬 알고리즘의 Work가 최적 순차 알고리즘보다 크지 않아야 합니다. Work가 순차보다 Θ(log n) 배 더 크면, 프로세서 수가 충분해도 순차보다 느릴 수 있습니다.

**2. 세분화(Granularity) 조절**: spawn/fork의 오버헤드는 수십~수백 사이클입니다. 너무 작은 태스크를 spawn하면 오버헤드가 작업 비용을 초과합니다. 경험적으로 태스크당 최소 수천 사이클의 작업이 필요합니다.

**3. 작업 훔치기(Work Stealing)**: Cilk·Java ForkJoinPool·Go의 스케줄러는 작업 훔치기 방식으로 Brent의 경계 `W/P + O(S)`를 기대값으로 달성합니다. 이 보장이 성립하려면 태스크 간 의존성이 DAG 형태여야 합니다.

**4. 범위(Span)가 병목**: 범위 S(n)이 O(n)이면 프로세서를 아무리 추가해도 순차 실행과 같은 최악 케이스 가능성이 있습니다. 알고리즘을 재설계하여 Span을 O(log^k n)으로 줄이는 것이 핵심입니다.

**5. 메모리 접근 패턴**: 병렬로 실행되는 여러 스레드가 동일한 캐시 라인을 쓰면 **거짓 공유(False Sharing)** 문제가 발생합니다. 각 스레드가 독립적인 메모리 영역을 사용하도록 데이터를 분리하세요.

**6. 병렬성 프로파일링**: Intel VTune, perf, Cilkview(OpenCilk)로 실제 병렬성과 이론적 병렬성의 차이를 측정하고, 범위 임계 경로를 식별하여 최적화 대상을 찾으세요.

---

## 참고 자료

- [OpenCilk Project — 고성능 병렬 프로그래밍 런타임](https://github.com/OpenCilk/opencilk-project)
- [KTH Algorithm Competition Template Library — 병렬 알고리즘 구현 모음](https://github.com/kth-competitive-programming/kactl)
