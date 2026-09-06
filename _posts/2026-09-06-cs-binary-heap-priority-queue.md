---
layout: post
title: "이진 힙(Binary Heap) 완전 정복: Priority Queue 구현과 Heapsort의 수학적 원리"
date: 2026-09-06
categories: [cs, computer-science]
tags: [binary-heap, priority-queue, heapsort, data-structures, algorithms]
---

## 이진 힙이란 무엇인가

이진 힙(Binary Heap)은 완전 이진 트리(Complete Binary Tree)를 배열로 표현한 자료구조로, **힙 속성(Heap Property)**을 만족해야 합니다. 힙 속성에는 두 가지가 있습니다.

- **최대 힙(Max-Heap)**: 부모 노드의 값이 자식 노드의 값보다 항상 크거나 같다.
- **최소 힙(Min-Heap)**: 부모 노드의 값이 자식 노드의 값보다 항상 작거나 같다.

완전 이진 트리를 배열에 매핑하면 부모·자식 인덱스를 O(1)에 계산할 수 있습니다.

```
인덱스 i의 노드:
  왼쪽 자식: 2*i + 1
  오른쪽 자식: 2*i + 2
  부모: (i - 1) / 2
```

이 단순한 매핑 덕분에 포인터 없이도 트리 연산을 배열에서 직접 수행할 수 있습니다.

## 왜 이진 힙이 필요한가

우선순위 큐(Priority Queue)는 "현재 가장 높은(또는 낮은) 우선순위 원소를 O(log N)에 삽입하고 꺼낸다"는 요구사항을 충족해야 합니다. 여러 구현 방식을 비교하면 다음과 같습니다.

| 자료구조 | 삽입 | 최솟값 삭제 | 최솟값 조회 |
|--------|------|------------|------------|
| 정렬된 배열 | O(N) | O(1) | O(1) |
| 비정렬 배열 | O(1) | O(N) | O(N) |
| 이진 힙 | O(log N) | O(log N) | O(1) |
| 피보나치 힙 | O(1)* | O(log N)* | O(1) |

이진 힙은 피보나치 힙보다 이론적 복잡도는 열위이지만 캐시 친화적 배열 기반 구조 덕분에 **실용적 성능**에서 앞서는 경우가 많습니다. Dijkstra, Prim, Huffman 인코딩, 운영체제 스케줄러 등 수많은 알고리즘의 핵심 엔진으로 사용됩니다.

## 핵심 연산 구현

### Sift-Up과 Sift-Down

힙의 두 핵심 보조 연산은 삽입 후 위로 버블링하는 **Sift-Up**과, 루트 삭제 후 아래로 내려보내는 **Sift-Down**입니다.

```python
class MinHeap:
    def __init__(self):
        self._data = []

    def push(self, val):
        self._data.append(val)
        self._sift_up(len(self._data) - 1)

    def pop(self):
        if not self._data:
            raise IndexError("Heap is empty")
        # 루트와 마지막 원소 교환 후 마지막 제거
        self._data[0], self._data[-1] = self._data[-1], self._data[0]
        min_val = self._data.pop()
        if self._data:
            self._sift_down(0)
        return min_val

    def peek(self):
        return self._data[0] if self._data else None

    def _sift_up(self, i):
        parent = (i - 1) // 2
        while i > 0 and self._data[i] < self._data[parent]:
            self._data[i], self._data[parent] = self._data[parent], self._data[i]
            i = parent
            parent = (i - 1) // 2

    def _sift_down(self, i):
        n = len(self._data)
        while True:
            smallest = i
            left, right = 2 * i + 1, 2 * i + 2
            if left < n and self._data[left] < self._data[smallest]:
                smallest = left
            if right < n and self._data[right] < self._data[smallest]:
                smallest = right
            if smallest == i:
                break
            self._data[i], self._data[smallest] = self._data[smallest], self._data[i]
            i = smallest

# 사용 예시
heap = MinHeap()
for v in [5, 3, 8, 1, 9, 2]:
    heap.push(v)

result = []
while heap.peek() is not None:
    result.append(heap.pop())
print(result)  # [1, 2, 3, 5, 8, 9]
```

**Sift-Up 복잡도 분석**: 트리 높이 h = ⌊log₂ N⌋이므로 O(log N)

**Sift-Down 복잡도 분석**: 마찬가지로 O(log N), 단 상수 인자가 더 크다(비교가 2회 발생)

### Heapify — 배열을 힙으로 O(N) 변환

N개 원소를 일일이 push하면 O(N log N)이지만, **Floyd's Heapify** 알고리즘은 O(N)에 배열을 힙으로 만듭니다.

```python
def heapify(arr):
    """제자리(in-place) 최소 힙 구성 — O(N)"""
    n = len(arr)
    # 마지막 내부 노드(리프가 아닌 노드)부터 루트까지 Sift-Down
    for i in range((n - 2) // 2, -1, -1):
        _sift_down(arr, i, n)

def _sift_down(arr, i, n):
    while True:
        smallest = i
        left, right = 2 * i + 1, 2 * i + 2
        if left < n and arr[left] < arr[smallest]:
            smallest = left
        if right < n and arr[right] < arr[smallest]:
            smallest = right
        if smallest == i:
            break
        arr[i], arr[smallest] = arr[smallest], arr[i]
        i = smallest

# 왜 O(N)인가?
# 높이 h인 노드에서 Sift-Down 비용은 O(h)
# 높이 h의 노드 수는 최대 ⌈N/2^(h+1)⌉
# 총 비용 = Σ h * N/2^(h+1) = N * Σ h/2^(h+1) = N * 2 = O(N)
```

이 O(N) Heapify가 Heapsort의 핵심입니다.

## Heapsort 완전 구현

```python
def heapsort(arr):
    """
    1단계: 배열을 최대 힙으로 변환 — O(N)
    2단계: N번 루트(최댓값)를 끝으로 보내고 힙 크기 축소 — O(N log N)
    전체: O(N log N), 제자리, 불안정 정렬
    """
    n = len(arr)

    # 1단계: max-heapify (내림차순 정렬을 위해 최대 힙 사용)
    for i in range((n - 2) // 2, -1, -1):
        _max_sift_down(arr, i, n)

    # 2단계: 루트를 끝으로 보내며 정렬
    for end in range(n - 1, 0, -1):
        arr[0], arr[end] = arr[end], arr[0]  # 최댓값을 맨 뒤로
        _max_sift_down(arr, 0, end)          # 줄어든 힙에서 복원

def _max_sift_down(arr, i, n):
    while True:
        largest = i
        left, right = 2 * i + 1, 2 * i + 2
        if left < n and arr[left] > arr[largest]:
            largest = left
        if right < n and arr[right] > arr[largest]:
            largest = right
        if largest == i:
            break
        arr[i], arr[largest] = arr[largest], arr[i]
        i = largest

# 테스트
data = [64, 34, 25, 12, 22, 11, 90]
heapsort(data)
print(data)  # [11, 12, 22, 25, 34, 64, 90]
```

### Heapsort와 Quicksort 비교

| 항목 | Heapsort | Quicksort |
|------|----------|-----------|
| 최선 | O(N log N) | O(N log N) |
| 평균 | O(N log N) | O(N log N) |
| 최악 | **O(N log N)** | O(N²) |
| 제자리 | O(1) | O(log N) 스택 |
| 안정성 | 불안정 | 불안정 |
| 캐시 효율 | 낮음 | 높음 |

Heapsort는 최악 경우 보장이 필요한 실시간 시스템에서 유리하지만, 캐시 미스가 많아 평균적으로 Quicksort보다 느립니다.

## 실전 응용: k번째 최솟값과 Merge K Sorted Lists

### k번째 최솟값 — O(N log k)

```python
import heapq

def kth_smallest(nums, k):
    """
    최대 힙으로 크기 k를 유지: 힙 상단이 k번째 최솟값
    시간: O(N log k), 공간: O(k)
    """
    max_heap = []
    for num in nums:
        heapq.heappush(max_heap, -num)  # 파이썬은 최소 힙이므로 음수 사용
        if len(max_heap) > k:
            heapq.heappop(max_heap)
    return -max_heap[0]

print(kth_smallest([7, 10, 4, 3, 20, 15], 3))  # 7

def merge_k_sorted(lists):
    """
    K개 정렬 리스트 병합 — O(N log K)
    각 리스트의 첫 원소를 힙에 유지
    """
    heap = []
    result = []
    
    for i, lst in enumerate(lists):
        if lst:
            heapq.heappush(heap, (lst[0], i, 0))  # (값, 리스트인덱스, 원소인덱스)

    while heap:
        val, list_idx, elem_idx = heapq.heappop(heap)
        result.append(val)
        next_idx = elem_idx + 1
        if next_idx < len(lists[list_idx]):
            heapq.heappush(heap, (lists[list_idx][next_idx], list_idx, next_idx))

    return result

lists = [[1, 4, 7], [2, 5, 8], [3, 6, 9]]
print(merge_k_sorted(lists))  # [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

## 주의사항과 고급 팁

### 1. 인덱스 변형에 주의

0-indexed와 1-indexed 두 가지 방식이 있습니다. 1-indexed에서는 왼쪽 자식이 `2i`, 부모가 `i//2`로 더 깔끔하지만 배열 크기를 N+1로 할당해야 합니다.

### 2. Decrease-Key 연산

다익스트라 알고리즘에서는 이미 힙에 있는 원소의 우선순위를 낮추는 **Decrease-Key**가 필요합니다. 표준 이진 힙은 이를 O(log N)에 지원하려면 **위치 인덱스 배열**을 추가로 관리해야 합니다.

```python
class IndexedMinHeap:
    """위치 추적 최소 힙 — Dijkstra에서 decrease-key O(log N) 지원"""
    def __init__(self, capacity):
        self.n = 0
        self.heap = []           # (key, node_id) 쌍
        self.pos = {}            # node_id → 힙 내 위치

    def push(self, key, node_id):
        self.heap.append((key, node_id))
        self.pos[node_id] = len(self.heap) - 1
        self._sift_up(len(self.heap) - 1)

    def decrease_key(self, node_id, new_key):
        i = self.pos[node_id]
        self.heap[i] = (new_key, node_id)
        self._sift_up(i)  # 키가 작아졌으므로 위로 올라갈 수 있음

    def _sift_up(self, i):
        while i > 0:
            parent = (i - 1) // 2
            if self.heap[i][0] < self.heap[parent][0]:
                self.heap[i], self.heap[parent] = self.heap[parent], self.heap[i]
                self.pos[self.heap[i][1]] = i
                self.pos[self.heap[parent][1]] = parent
                i = parent
            else:
                break
```

### 3. 힙이 적합하지 않은 경우

- **임의 원소 삭제**: 위치를 알더라도 O(log N)이 필요하며 인덱스 추적이 복잡합니다.
- **범위 쿼리**: "k번째부터 m번째까지 원소" 쿼리는 세그먼트 트리나 정렬된 배열이 적합합니다.
- **단조 증가 삽입**: 삽입 순서가 이미 정렬되어 있다면 Sift-Up이 불필요합니다.

### 4. D-ary Heap으로 캐시 효율 향상

이진 힙 대신 자식이 d개인 d진 힙을 사용하면 Sift-Up 횟수는 줄고(log_d N) Sift-Down 비교는 늘어납니다(d log_d N). 삽입이 잦은 Dijkstra에서는 d=4 힙이 실용적으로 더 빠릅니다.

## 마무리

이진 힙은 단순한 배열 표현과 O(log N) 연산을 결합해 우선순위 큐를 실용적으로 구현하는 자료구조입니다. Floyd's Heapify로 O(N)에 배열을 힙으로 변환하고, 이를 바탕으로 O(N log N) 최악 보장 Heapsort를 구현할 수 있습니다. Dijkstra, Prim, Huffman 등 핵심 알고리즘의 하부 엔진으로서, 이진 힙의 내부 동작을 깊이 이해하면 성능 병목을 정확히 진단하고 최적화할 수 있습니다.

## 참고 자료
- [Python heapq 공식 문서](https://docs.python.org/3/library/heapq.html)
- [CLRS — Binary Heaps (MIT OpenCourseWare)](https://ocw.mit.edu/courses/6-006-introduction-to-algorithms-fall-2011/pages/lecture-videos/)
- [Wikipedia — Binary Heap](https://en.wikipedia.org/wiki/Binary_heap)
- [Visualgo — Heap Visualization](https://visualgo.net/en/heap)
