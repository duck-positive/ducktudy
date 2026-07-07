---
layout: post
title: "단조 스택(Monotonic Stack)과 단조 큐(Monotonic Queue) 완전 정복: 선형 시간 범위 쿼리의 핵심"
date: 2026-07-07
categories: [cs, computer-science]
tags: [algorithm, stack, queue, sliding-window, next-greater-element, monotonic, data-structure]
---

단조 스택(Monotonic Stack)과 단조 큐(Monotonic Queue)는 이름부터 낯설게 느껴질 수 있지만, 코딩 인터뷰와 실제 시스템 구현에서 반복적으로 등장하는 강력한 기법이다. "다음으로 큰 원소 찾기", "슬라이딩 윈도우 최댓값" 같은 문제를 O(n²)에서 O(n)으로 줄여주는 핵심 도구다. 이 글에서는 두 자료구조의 원리, 구현, 그리고 실전 활용까지 완전히 파헤친다.

## 왜 필요한가: 순진한 O(n²) 접근의 한계

배열이 있을 때 각 원소에 대해 "오른쪽에서 처음으로 자신보다 큰 원소"를 찾는 문제를 생각하자. 가장 직관적인 방법은 이중 루프다.

```python
def next_greater_naive(arr):
    n = len(arr)
    result = [-1] * n
    for i in range(n):
        for j in range(i + 1, n):
            if arr[j] > arr[i]:
                result[i] = arr[j]
                break
    return result
```

이 코드는 O(n²)다. n = 10만이라면 100억 번의 비교가 필요하다. 단조 스택을 사용하면 이를 O(n)으로 줄일 수 있다.

슬라이딩 윈도우 최댓값 문제도 마찬가지다. 크기 k인 윈도우를 배열 위로 이동하면서 각 위치에서의 최댓값을 구할 때, 나이브 구현은 O(nk)지만 단조 큐를 사용하면 O(n)이 된다. 이 두 기법을 이해하는 것은 범위 쿼리(Range Query) 문제를 선형 시간에 해결하는 첫걸음이다.

## 단조 스택(Monotonic Stack)

### 핵심 아이디어

단조 스택은 **항상 단조 증가 또는 단조 감소 상태를 유지하는 스택**이다. 새 원소를 삽입할 때, 현재 원소가 스택의 top과 단조성을 깨뜨리면 단조성이 회복될 때까지 top을 pop한다.

핵심 통찰: pop 당하는 순간이 바로 "다음으로 큰/작은 원소를 만나는 순간"이다.

### 단조 감소 스택으로 Next Greater Element 구현

```python
from collections import deque

def next_greater(arr):
    """
    각 원소의 오른쪽에서 처음으로 큰 원소를 O(n)에 구한다.
    스택에는 아직 NGE를 찾지 못한 인덱스를 저장한다(단조 감소).
    """
    n = len(arr)
    result = [-1] * n
    stack = []  # 인덱스를 저장하는 단조 감소 스택

    for i in range(n):
        # 현재 원소가 스택 top의 원소보다 크면 → top의 NGE는 arr[i]
        while stack and arr[i] > arr[stack[-1]]:
            idx = stack.pop()
            result[idx] = arr[i]
        stack.append(i)

    # 스택에 남은 원소들은 NGE가 없음 → result는 이미 -1로 초기화됨
    return result


def largest_rectangle_histogram(heights):
    """
    히스토그램에서 가장 큰 직사각형 넓이 (LeetCode 84번).
    단조 증가 스택을 사용한다.
    """
    stack = []
    max_area = 0
    heights.append(0)  # 센티넬: 남은 스택을 모두 flush

    for i, h in enumerate(heights):
        while stack and heights[stack[-1]] > h:
            height = heights[stack.pop()]
            width = i if not stack else i - stack[-1] - 1
            max_area = max(max_area, height * width)
        stack.append(i)

    heights.pop()  # 센티넬 제거
    return max_area


# 테스트
arr = [4, 6, 3, 2, 8, 1]
print(next_greater(arr))          # [6, 8, 8, 8, -1, -1]

heights = [2, 1, 5, 6, 2, 3]
print(largest_rectangle_histogram(heights))  # 10
```

**각 원소는 스택에 최대 1번 push되고 최대 1번 pop된다.** 따라서 전체 시간복잡도는 O(n)이다.

### 단조 스택의 변형: 이전으로 작은 원소(Previous Smaller Element)

방향을 바꾸면 PSE(Previous Smaller Element)를 구할 수 있다. 오른쪽에서 왼쪽으로 순회하거나, 단조 조건을 반전하면 된다.

```python
def previous_smaller(arr):
    """각 원소의 왼쪽에서 처음으로 작은 원소를 O(n)에 구한다."""
    n = len(arr)
    result = [-1] * n
    stack = []  # 단조 증가 스택

    for i in range(n):
        while stack and arr[stack[-1]] >= arr[i]:
            stack.pop()
        result[i] = arr[stack[-1]] if stack else -1
        stack.append(i)

    return result
```

### 응용: 빗물 트래핑(Trapping Rain Water)

```python
def trap_rain_water(height):
    """
    각 위치에 고인 물의 양을 계산한다.
    단조 감소 스택: 오목한 구간을 찾아 물의 양을 계산.
    """
    stack = []
    water = 0

    for i, h in enumerate(height):
        while stack and h > height[stack[-1]]:
            bottom_idx = stack.pop()
            if not stack:
                break
            left_idx = stack[-1]
            width = i - left_idx - 1
            bounded_height = min(height[left_idx], h) - height[bottom_idx]
            water += width * bounded_height

        stack.append(i)

    return water


print(trap_rain_water([0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1]))  # 6
```

## 단조 큐(Monotonic Deque)

### 핵심 아이디어

단조 큐는 **덱(deque, 양방향 큐)을 사용하여 슬라이딩 윈도우의 최댓값/최솟값을 O(1)에 반환**하는 자료구조다. 앞에서는 만료된 원소를 제거하고, 뒤에서는 단조성을 깨뜨리는 원소를 제거한다.

- **뒤(rear)에서 pop**: 새 원소가 들어올 때 자신보다 작거나 같은 원소를 제거 (최대 단조 큐)
- **앞(front)에서 pop**: 윈도우 범위를 벗어난 만료 인덱스 제거

### 슬라이딩 윈도우 최댓값 구현

```python
from collections import deque

def sliding_window_maximum(nums, k):
    """
    크기 k인 슬라이딩 윈도우의 최댓값 배열을 O(n)에 구한다.
    덱에는 인덱스를 저장하며, 대응되는 값은 단조 감소로 유지된다.
    """
    dq = deque()  # 인덱스를 저장하는 단조 감소 덱
    result = []

    for i, num in enumerate(nums):
        # 1. 윈도우를 벗어난 인덱스 제거 (앞에서)
        while dq and dq[0] < i - k + 1:
            dq.popleft()

        # 2. 현재 값보다 작거나 같은 원소 제거 (뒤에서)
        #    이 원소들은 현재 원소가 윈도우에 있는 한 절대 최대가 될 수 없다.
        while dq and nums[dq[-1]] <= num:
            dq.pop()

        dq.append(i)

        # 윈도우가 완성된 시점부터 결과 추가
        if i >= k - 1:
            result.append(nums[dq[0]])

    return result


def sliding_window_minimum(nums, k):
    """슬라이딩 윈도우 최솟값: 단조 증가 덱."""
    dq = deque()
    result = []

    for i, num in enumerate(nums):
        while dq and dq[0] < i - k + 1:
            dq.popleft()
        while dq and nums[dq[-1]] >= num:
            dq.pop()
        dq.append(i)
        if i >= k - 1:
            result.append(nums[dq[0]])

    return result


# 테스트
nums = [1, 3, -1, -3, 5, 3, 6, 7]
print(sliding_window_maximum(nums, 3))  # [3, 3, 5, 5, 6, 7]
print(sliding_window_minimum(nums, 3))  # [-1, -3, -3, -3, 3, 3]
```

**각 원소는 덱에 최대 1번 삽입되고 최대 1번 제거**되므로 O(n) 시간복잡도를 달성한다.

### 심화 응용: 가장 큰 직사각형 슬라이딩 윈도우

단조 큐는 DP 최적화에도 사용된다. 점화식 `dp[i] = min(dp[j]) + cost` 형태에서 j의 범위가 슬라이딩 윈도우이면 단조 큐로 O(n)에 해결할 수 있다.

```python
def jump_game_dp_optimized(nums, k):
    """
    각 위치 i에서 점프할 수 있는 범위가 [i-k, i-1]일 때,
    최소 비용으로 끝까지 가는 경로 계산 (단순화된 예제).
    dp[i] = min(dp[i-k..i-1]) + nums[i]
    """
    n = len(nums)
    dp = [float('inf')] * n
    dp[0] = nums[0]
    dq = deque([0])  # 최솟값 단조 증가 덱 (인덱스)

    for i in range(1, n):
        # 범위 밖 인덱스 제거
        while dq and dq[0] < i - k:
            dq.popleft()

        dp[i] = dp[dq[0]] + nums[i]  # 윈도우 내 최솟값

        # 현재 dp[i]보다 크거나 같은 원소 제거 (미래에 쓸모없음)
        while dq and dp[dq[-1]] >= dp[i]:
            dq.pop()
        dq.append(i)

    return dp[-1]
```

## 두 자료구조 비교

| 특성 | 단조 스택 | 단조 큐 |
|------|-----------|---------|
| 기반 자료구조 | 스택 (LIFO) | 덱 (양방향) |
| 주요 용도 | NGE/PSE, 히스토그램 | 슬라이딩 윈도우 최댓/최솟값 |
| 원소 만료 처리 | 불필요 | front에서 범위 초과 제거 |
| 단조 방향 | 증가 또는 감소 | 증가 또는 감소 |
| 시간복잡도 | O(n) | O(n) |
| 공간복잡도 | O(n) | O(k) |

## 주의사항과 팁

**1. 인덱스 vs 값 저장**: 대부분의 경우 인덱스를 저장하는 것이 유리하다. 인덱스를 통해 값과 위치를 모두 파악할 수 있고, 단조 큐의 만료 처리도 편리하다.

**2. 등호 처리**: `arr[j] >= arr[i]` vs `arr[j] > arr[i]` 중 무엇을 선택하느냐에 따라 "현재 원소와 같은 값"의 처리가 달라진다. 문제의 요구사항에 따라 신중하게 선택해야 한다.

**3. 센티넬(Sentinel) 값 활용**: `largest_rectangle_histogram` 예제처럼 배열 끝에 0을 추가하면 남은 스택 원소를 별도 처리 없이 flush할 수 있다.

**4. 단조 방향 선택 기준**:
- **최댓값 쿼리** → 단조 감소 (top이 항상 최대)
- **최솟값 쿼리** → 단조 증가 (top이 항상 최소)
- **NGE(다음 큰 원소)** → 단조 감소 스택
- **NSE(다음 작은 원소)** → 단조 증가 스택

**5. 원형 배열 처리**: 원형 배열에서 NGE를 구할 때는 배열을 두 번 순회하거나(`for i in range(2*n)` with `i%n`) 배열을 두 배로 복사하여 처리한다.

**6. 실전 활용 범위**: 단순한 코딩 문제를 넘어, 스트리밍 데이터 처리 시스템에서 슬라이딩 윈도우 통계(최댓값, 최솟값, 분위수)를 효율적으로 계산하는 데도 사용된다. 모니터링 시스템이나 실시간 대시보드 구현 시 유용하다.

단조 스택과 단조 큐는 습득하면 활용처가 무궁무진한 기법이다. 핵심은 "어떤 원소가 미래에 절대 답이 될 수 없는가"를 파악하고 즉시 제거하는 것이다. 이 직관을 가지면 관련 문제들이 훨씬 명확하게 보이기 시작한다.

## 참고 자료
- [Introduction to Monotonic Stack - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/introduction-to-monotonic-stack-2/)
- [Introduction to Monotonic Queues - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/introduction-to-monotonic-queues/)
- [Monotonic Stack & Queue — Algorithms-LeetCode](https://x-czh.github.io/Algorithms-LeetCode/Topics/Monotonic-Stack-&-Queue.html)
- [Monotonic Queue Explained with LeetCode Problems - Medium](https://medium.com/algorithms-and-leetcode/monotonic-queue-explained-with-leetcode-problems-7db7c530c1d6)
