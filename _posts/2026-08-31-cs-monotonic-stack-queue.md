---
layout: post
title: "단조 스택(Monotonic Stack)과 단조 큐(Monotonic Deque) 완전 정복: 히스토그램 최대 직사각형부터 슬라이딩 윈도우 최솟값까지"
date: 2026-08-31
categories: [cs, computer-science]
tags: [monotonic-stack, monotonic-deque, sliding-window, stack, deque, algorithm, competitive-programming]
---

## 개념 설명

**단조 스택(Monotonic Stack)**은 스택 내 원소들이 항상 단조 증가(또는 단조 감소) 순서를 유지하도록 관리되는 스택입니다. 새 원소를 삽입할 때, 단조성을 위반하는 기존 원소들을 모두 꺼낸 뒤 삽입합니다. 각 원소는 최대 한 번 push되고 최대 한 번 pop되므로, 전체 시간 복잡도는 **O(n)**입니다.

**단조 큐(Monotonic Deque / Monotonic Queue)**는 양방향 큐(deque)를 이용해 단조성을 유지합니다. 단조 스택과 달리 **앞(front)**에서도 원소를 제거할 수 있어 **슬라이딩 윈도우** 제약을 처리할 수 있습니다. 슬라이딩 윈도우 최솟값/최댓값 문제의 핵심 자료구조입니다.

---

## 왜 필요한가?

단조 스택/큐가 없다면 다음 문제들을 O(n²) 혹은 그 이상의 복잡도로밖에 풀 수 없습니다:

| 문제 유형                              | 나이브 복잡도 | 단조 스택/큐 |
|---------------------------------------|--------------|-------------|
| 각 원소의 오른쪽 첫 번째 작은 수 찾기   | O(n²)        | O(n)        |
| 히스토그램 최대 직사각형               | O(n²)        | O(n)        |
| 슬라이딩 윈도우 최솟값/최댓값          | O(nk)        | O(n)        |
| 빗물 트래핑(Trapping Rain Water)      | O(n²)        | O(n)        |
| 주식 스팬(Stock Span Problem)         | O(n²)        | O(n)        |

단조 스택은 "현재 원소보다 작거나 큰 **이전/다음** 원소를 효율적으로 찾아야 하는" 모든 상황에서 등장합니다. 면접과 경쟁 프로그래밍에서 빠질 수 없는 핵심 테크닉입니다.

---

## 실제 구현 예제

### 예제 1: 히스토그램에서 가장 큰 직사각형 (Python)

LeetCode 84번 문제입니다. 단조 증가 스택을 유지하며 각 막대를 오른쪽 경계로 삼았을 때 최대 넓이를 계산합니다.

```python
def largest_rectangle_in_histogram(heights: list[int]) -> int:
    """단조 증가 스택으로 O(n) 해결"""
    n = len(heights)
    stack = []   # 단조 증가 스택 (인덱스 저장)
    max_area = 0

    for i in range(n + 1):
        # 현재 높이 (범위 초과 시 0으로 처리해 남은 원소 모두 pop)
        h = heights[i] if i < n else 0

        while stack and heights[stack[-1]] > h:
            height = heights[stack.pop()]
            # 오른쪽 경계: i, 왼쪽 경계: stack[-1] + 1 (없으면 0)
            width  = i if not stack else i - stack[-1] - 1
            max_area = max(max_area, height * width)

        stack.append(i)

    return max_area


# 테스트
heights = [2, 1, 5, 6, 2, 3]
print(largest_rectangle_in_histogram(heights))  # 10

heights2 = [2, 4]
print(largest_rectangle_in_histogram(heights2))  # 4


# 각 원소의 '다음으로 작은 원소' 인덱스도 동일 패턴으로 구할 수 있음
def next_smaller(arr: list[int]) -> list[int]:
    """각 원소에 대해 오른쪽 첫 번째로 작은 원소의 인덱스 반환"""
    n = len(arr)
    result = [n] * n        # 없으면 n(센티넬)
    stack  = []             # 단조 증가 스택 (인덱스)

    for i, v in enumerate(arr):
        while stack and arr[stack[-1]] > v:
            result[stack.pop()] = i
        stack.append(i)
    return result


arr = [4, 5, 2, 10, 8]
print(next_smaller(arr))   # [2, 2, 5, 4, 5]
# arr[0]=4의 다음 작은 원소는 인덱스 2(값 2)
# arr[1]=5의 다음 작은 원소는 인덱스 2(값 2)
```

**핵심 패턴 분석**:
- **단조 증가 스택**: `arr[i] >= arr[stack[-1]]`일 때 pop — "이전에 더 큰 원소들을 현재 원소로 처리"
- **단조 감소 스택**: `arr[i] <= arr[stack[-1]]`일 때 pop — "다음으로 큰 원소 탐색에 사용"

---

### 예제 2: 슬라이딩 윈도우 최솟값 (Java)

크기 k의 슬라이딩 윈도우가 배열을 이동할 때 각 윈도우의 최솟값을 O(n)에 구합니다. LeetCode 239(최댓값) 변형입니다.

```java
import java.util.ArrayDeque;
import java.util.Deque;

public class SlidingWindowMinimum {

    /**
     * 단조 증가 덱으로 슬라이딩 윈도우 최솟값 O(n) 계산
     * deque에는 인덱스가 저장되며, 대응하는 값이 단조 증가 순서 유지
     */
    public static int[] minSlidingWindow(int[] nums, int k) {
        int n = nums.length;
        int[] result = new int[n - k + 1];
        Deque<Integer> deque = new ArrayDeque<>();  // 단조 증가 덱 (인덱스)

        for (int i = 0; i < n; i++) {
            // 1. 윈도우 밖으로 나간 인덱스 제거 (front에서)
            while (!deque.isEmpty() && deque.peekFirst() <= i - k) {
                deque.pollFirst();
            }

            // 2. 현재 값보다 크거나 같은 원소 제거 (단조성 유지, back에서)
            while (!deque.isEmpty() && nums[deque.peekLast()] >= nums[i]) {
                deque.pollLast();
            }

            deque.offerLast(i);

            // 3. 윈도우가 완전히 채워지면 결과 기록 (front = 현재 최솟값)
            if (i >= k - 1) {
                result[i - k + 1] = nums[deque.peekFirst()];
            }
        }
        return result;
    }

    public static void main(String[] args) {
        int[] nums = {1, 3, -1, -3, 5, 3, 6, 7};
        int k = 3;
        int[] ans = minSlidingWindow(nums, k);
        // 각 윈도우: [1,3,-1], [3,-1,-3], [-1,-3,5], [-3,5,3], [5,3,6], [3,6,7]
        // 최솟값:    -1,      -3,        -3,        -3,       3,       3
        for (int v : ans) System.out.print(v + " ");
        // 출력: -1 -3 -3 -3 3 3
        System.out.println();

        // 최댓값 버전: ">=" 대신 "<="로 바꾸면 단조 감소 덱이 됨
    }
}
```

**단조 큐 동작 흐름 요약**:
```
i=0 (1):  deque=[0]
i=1 (3):  3>=1? NO → deque=[0,1]
i=2 (-1): -1>=3? YES pop; -1>=1? YES pop → deque=[2]   결과[0]=-1
i=3 (-3): -3>=-1? YES pop → deque=[3]                  결과[1]=-3
i=4 (5):  5>=-3? NO → deque=[3,4]                      결과[2]=-3
...
```

---

## 주의사항 및 팁

### 1. 스택에 값 vs 인덱스 저장
대부분의 경우 **인덱스를 저장**해야 합니다. 값을 저장하면 "어디까지 범위인가"를 계산할 수 없습니다. 슬라이딩 윈도우 문제에서는 `i - k`로 만료 인덱스를 체크해야 하므로 인덱스 저장이 필수입니다.

### 2. 등호 처리 (`>` vs `>=`)
pop 조건에서 등호를 포함하면 **중복 값의 최후 인덱스**만 남고, 포함하지 않으면 **최초 인덱스**가 남습니다. 문제 요구사항에 따라 정확히 설정해야 합니다.

### 3. 단조 스택 방향 선택 기준
- **다음으로 작은 원소(Next Smaller Element)** → 단조 **증가** 스택 (더 큰 원소를 pop)
- **다음으로 큰 원소(Next Greater Element)** → 단조 **감소** 스택 (더 작은 원소를 pop)
- **이전으로 작은 원소(Previous Smaller Element)** → 단조 증가 스택, 왼쪽에서 오른쪽으로 탐색

### 4. DP + 단조 큐 조합
점화식이 `dp[i] = min(dp[j]) + cost(j, i)` 형태에서 j의 범위가 슬라이딩 윈도우라면 단조 큐로 O(n)에 최적화할 수 있습니다. 이 패턴은 **큐 최적화 DP(Queue Optimization DP)**라고 불립니다.

### 5. 실전 문제 목록
- Next Greater/Smaller Element (기본)
- Largest Rectangle in Histogram (LeetCode 84)
- Trapping Rain Water (LeetCode 42)
- Sliding Window Maximum (LeetCode 239)
- Sum of Subarray Minimums (LeetCode 907)
- 큐 최적화 DP: 1D 배낭 문제 변형

---

## 참고 자료

- [Introduction to Monotonic Stack - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/introduction-to-monotonic-stack-2/)
- [Introduction to Monotonic Queues - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/introduction-to-monotonic-queues/)
- [Monotonic Stack/Deque - AlgoMonster](https://algo.monster/problems/mono_stack_intro)
- [Monotonic Queue Explained with LeetCode Problems - Medium](https://medium.com/algorithms-and-leetcode/monotonic-queue-explained-with-leetcode-problems-7db7c530c1d6)
