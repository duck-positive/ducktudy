---
layout: post
title: "단조 스택과 단조 큐 완전 정복: Next Greater Element부터 슬라이딩 윈도우 최댓값까지 O(N)에 해결하기"
date: 2026-10-20
categories: [cs, computer-science]
tags: [monotonic-stack, monotonic-queue, algorithm, data-structure, sliding-window, stack, deque]
---

## 단조 스택과 단조 큐란 무엇인가?

**단조 스택(Monotonic Stack)**과 **단조 큐(Monotonic Deque)**는 기본적으로는 흔한 스택·덱 자료구조이지만, 원소를 삽입할 때 **단조 증가 또는 단조 감소 순서를 항상 유지**한다는 하나의 불변식을 갖는다. 이 단순해 보이는 제약이 배열 내 '다음으로 크거나 작은 원소', '구간 최댓값·최솟값' 같은 문제를 브루트 포스 O(N²)에서 O(N)으로 끌어내리는 마법을 부린다.

단조 스택·큐는 특별한 라이브러리 없이 스택이나 덱 하나로 구현되므로, 개념을 이해하면 어떤 언어에서든 즉시 사용할 수 있다.

### 단조 감소 스택 (Monotonically Decreasing Stack)

스택 내부 원소가 아래에서 위로 **감소(또는 비증가)** 순서를 이룬다. 새 원소가 스택 top보다 크거나 같으면 top을 팝(pop)한다. 이 방식으로 '아직 답을 찾지 못한 인덱스'를 관리하다가, 팝이 일어나는 순간 그 인덱스의 **다음 큰 원소**가 결정된다.

반대로 **단조 증가 스택**은 top보다 작거나 같으면 팝하며, '다음 작은 원소'를 구할 때 쓴다.

---

## 왜 필요한가? — O(N²)의 벽을 깨는 단 하나의 불변식

배열에서 각 원소의 '다음으로 큰 원소'를 찾는 문제를 생각해보자. 브루트 포스로는 모든 (i, j) 쌍을 확인해야 하므로 O(N²)이 걸린다. 하지만 단조 스택은 각 원소가 스택에 **한 번 push되고 한 번 pop**된다는 사실을 보장한다. 따라서 전체 push·pop 횟수는 N번을 넘지 않아 **전체 시간 복잡도 O(N)**이 된다.

이 패턴이 등장하는 대표 문제들:

| 문제 | 활용 자료구조 |
|------|------------|
| Next Greater Element | 단조 감소 스택 |
| Previous Smaller Element | 단조 증가 스택 |
| 히스토그램 최대 직사각형 | 단조 증가 스택 |
| Sliding Window Maximum | 단조 감소 덱 |
| 주식 Span 문제 | 단조 감소 스택 |
| Trapping Rain Water | 단조 감소 스택 |

---

## 실제 구현 예제

### 예제 1: Next Greater Element — 각 원소의 다음 큰 원소 찾기

```python
from typing import List

def next_greater_element(nums: List[int]) -> List[int]:
    """
    각 원소에 대해 오른쪽에서 처음으로 자신보다 큰 원소를 반환한다.
    없으면 -1.
    시간 복잡도: O(N), 공간 복잡도: O(N)
    """
    n = len(nums)
    result = [-1] * n
    stack = []  # 단조 감소 스택: 인덱스 저장

    for i in range(n):
        # 현재 원소가 스택 top 인덱스의 값보다 크면,
        # top 인덱스의 '다음 큰 원소'가 nums[i]로 결정된다.
        while stack and nums[stack[-1]] < nums[i]:
            idx = stack.pop()
            result[idx] = nums[i]
        stack.append(i)
    # stack에 남은 인덱스들은 오른쪽에 큰 원소가 없으므로 -1 유지

    return result


# 테스트
nums = [2, 1, 5, 6, 2, 3]
print(next_greater_element(nums))
# 출력: [5, 5, 6, -1, 3, -1]
# 설명:
#   2 → 오른쪽에서 처음으로 큰 수: 5
#   1 → 5
#   5 → 6
#   6 → 없음 (-1)
#   2 → 3
#   3 → 없음 (-1)
```

**핵심 로직 추적**

| i | nums[i] | 스택 상태 (인덱스) | result |
|---|---------|------------------|--------|
| 0 | 2 | [0] | [-1,-1,-1,-1,-1,-1] |
| 1 | 1 | [0,1] | [-1,-1,-1,-1,-1,-1] |
| 2 | 5 | [2] | [5,5,-1,-1,-1,-1] |
| 3 | 6 | [3] | [5,5,6,-1,-1,-1] |
| 4 | 2 | [3,4] | [5,5,6,-1,-1,-1] |
| 5 | 3 | [3,5] | [5,5,6,-1,-1,3] |

순환 배열(circular array) 변형에서는 배열을 두 배로 늘리거나 인덱스를 `% n`으로 처리하면 된다.

---

### 예제 2: 슬라이딩 윈도우 최댓값 — 단조 감소 덱

크기 k의 슬라이딩 윈도우가 배열 위를 오른쪽으로 이동할 때 각 위치에서의 최댓값을 O(N)에 구한다.

```python
from collections import deque
from typing import List

def max_sliding_window(nums: List[int], k: int) -> List[int]:
    """
    크기 k의 윈도우가 왼쪽에서 오른쪽으로 이동할 때 각 윈도우의 최댓값.
    덱에는 인덱스를 저장하며, 값 기준 단조 감소를 유지한다.
    시간 복잡도: O(N), 공간 복잡도: O(k)
    """
    dq = deque()   # 단조 감소 덱: 인덱스 저장 (앞이 현재 윈도우의 최댓값)
    result = []

    for i in range(len(nums)):
        # 1. 덱 앞이 윈도우 범위를 벗어났으면 제거
        if dq and dq[0] < i - k + 1:
            dq.popleft()

        # 2. 새 원소보다 작거나 같은 덱 뒤쪽 원소들은 불필요 → 제거
        while dq and nums[dq[-1]] <= nums[i]:
            dq.pop()

        dq.append(i)

        # 3. 윈도우가 k개 채워진 시점부터 결과 수집
        if i >= k - 1:
            result.append(nums[dq[0]])

    return result


# 테스트
nums = [1, 3, -1, -3, 5, 3, 6, 7]
k = 3
print(max_sliding_window(nums, k))
# 출력: [3, 3, 5, 5, 6, 7]
# 윈도우별:
#   [1,3,-1]  → 3
#   [3,-1,-3] → 3
#   [-1,-3,5] → 5
#   [-3,5,3]  → 5
#   [5,3,6]   → 6
#   [3,6,7]   → 7
```

덱의 앞은 현재 윈도우 최댓값의 인덱스를 가리킨다. 뒤에서 새 원소보다 작은 원소를 제거하므로, 덱은 항상 단조 감소 상태를 유지한다. 각 원소가 덱에 최대 1번 추가되고 1번 제거되므로 총 연산은 O(N)이다.

---

### 예제 3: 히스토그램 최대 직사각형

각 막대의 너비가 1인 히스토그램에서 가장 넓은 직사각형 넓이를 구하는 문제다. 단조 증가 스택으로 '현재 막대가 오른쪽 경계가 되는 직사각형'의 왼쪽 경계를 O(N)에 찾는다.

```python
from typing import List

def largest_rectangle_in_histogram(heights: List[int]) -> int:
    """
    단조 증가 스택 활용.
    스택 팝이 일어나는 순간, 팝된 인덱스의 막대를 높이로 하는
    최대 직사각형의 너비를 계산한다.
    시간 복잡도: O(N)
    """
    stack = []   # 단조 증가: 높이 기준
    max_area = 0
    heights = heights + [0]  # 센티널: 마지막에 스택을 모두 비움

    for i, h in enumerate(heights):
        while stack and heights[stack[-1]] > h:
            height = heights[stack.pop()]
            width = i if not stack else i - stack[-1] - 1
            max_area = max(max_area, height * width)
        stack.append(i)

    return max_area


# 테스트
heights = [2, 1, 5, 6, 2, 3]
print(largest_rectangle_in_histogram(heights))  # 출력: 10
# 높이 5, 6 두 막대로 너비 2짜리 직사각형 = 10

heights2 = [2, 4]
print(largest_rectangle_in_histogram(heights2))  # 출력: 4
```

---

## 주의사항 및 팁

### 1. 인덱스 vs 값 — 스택에 무엇을 저장할까?

단조 스택에는 **인덱스**를 저장하는 것이 일반적이다. 값만 저장하면 '너비 계산' 등 위치 정보가 필요한 문제에서 막힌다. 인덱스를 저장하면 값은 `nums[idx]`로 언제든 꺼낼 수 있다.

### 2. 등호 처리 — `<` vs `<=`

팝 조건에서 등호를 포함할지 여부에 따라 결과가 달라진다. "다음으로 **크거나 같은** 원소"를 원하면 `<` 대신 `<=`를 사용한다. 같은 값이 여러 개 있을 때 중복 처리 여부를 문제 조건에 맞게 설정해야 한다.

### 3. 슬라이딩 윈도우 덱: 앞쪽 범위 체크를 먼저

덱에서 앞쪽 범위 이탈 체크를 먼저 하고, 뒤쪽 단조성 유지를 나중에 해야 한다. 순서가 바뀌면 유효하지 않은 인덱스로 값을 조회하는 버그가 발생한다.

### 4. 센티널(Sentinel) 기법

히스토그램 문제처럼 배열 끝에 `0` 같은 극솟값을 추가하면, 배열 순회가 끝난 후 스택에 남아있는 원소들을 일괄 처리할 수 있어 코드가 단순해진다.

### 5. 연습 문제 로드맵

1. LeetCode 496 — Next Greater Element I (기본)
2. LeetCode 503 — Next Greater Element II (원형 배열)
3. LeetCode 84 — Largest Rectangle in Histogram (단조 증가 스택)
4. LeetCode 239 — Sliding Window Maximum (단조 감소 덱)
5. LeetCode 42 — Trapping Rain Water (단조 감소 스택 변형)
6. LeetCode 901 — Online Stock Span (단조 감소 스택 응용)

단조 스택 문제의 공통 패턴은 "배열을 한 번 순회하면서, 아직 답을 모르는 원소들을 스택에 보관하고, 새 원소가 조건을 충족하는 순간 스택에서 꺼내어 답을 확정한다"는 것이다. 이 감각을 익히면 처음 보는 문제도 패턴으로 분해할 수 있다.

---

## 시간·공간 복잡도 정리

| 연산 | 시간 복잡도 | 공간 복잡도 |
|------|-----------|----------|
| Next Greater Element (단조 스택) | O(N) | O(N) |
| Sliding Window Maximum (단조 덱) | O(N) | O(k) |
| 히스토그램 최대 직사각형 | O(N) | O(N) |

---

## 참고 자료

- [Monotonic Stack — Baeldung on Computer Science](https://www.baeldung.com/cs/monotonic-stack)
- [Monotonic Stack Code Template — labuladong](https://labuladong.online/algo/en/data-structure/monotonic-stack/)
- [A comprehensive guide and template for monotonic stack based problems — LeetCode Discussion](https://leetcode.com/discuss/post/2347639)
