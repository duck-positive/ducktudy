---
layout: post
title: "파라메트릭 서치(Parametric Search) 완전 정복: 결정 문제로 변환하는 이분 탐색의 고급 활용"
date: 2026-08-15
categories: [cs, computer-science]
tags: [parametric-search, binary-search, algorithm, optimization, competitive-programming]
---

이분 탐색(Binary Search)은 정렬된 배열에서 원소를 찾는 알고리즘으로 잘 알려져 있습니다. 그러나 이분 탐색의 진정한 힘은 여기서 그치지 않습니다. **파라메트릭 서치(Parametric Search)**, 혹은 **이분 탐색으로 답 구하기(Binary Search on Answer)**는 최적화 문제를 결정 문제(Decision Problem)로 변환하여 이분 탐색을 적용하는 강력한 기법입니다. 이 기법을 이해하면 직접적으로 계산하기 어려운 최솟값·최댓값 문제를 매우 우아하게 풀 수 있습니다.

## 파라메트릭 서치란 무엇인가?

파라메트릭 서치의 핵심 아이디어는 단순합니다.

> **"정답이 어떤 범위 안에서 단조로운 성질을 가질 때, 특정 값이 정답이 될 수 있는지를 판별하는 함수로 이분 탐색을 수행한다."**

일반적인 이분 탐색은 정렬된 배열의 인덱스를 탐색합니다. 파라메트릭 서치는 배열이 아닌 **정답 후보 공간** 위에서 이분 탐색을 수행합니다. 이때 다음 두 조건이 충족되어야 합니다.

1. **단조성(Monotonicity)**: 어떤 값 `x`가 가능하다면, `x`보다 더 완화된 조건의 값도 가능해야 합니다. 즉, 가능/불가능이 명확하게 나뉘는 임계점이 존재해야 합니다.
2. **효율적인 결정 함수**: `check(mid)`가 참인지 거짓인지를 다항 시간 안에 판별할 수 있어야 합니다.

## 왜 파라메트릭 서치가 필요한가?

많은 최적화 문제는 최적값을 직접 구하기가 매우 어렵습니다. 예를 들어:

- 어떤 자원을 `k`명에게 배분할 때, 최솟값을 최대화하라.
- 특정 연산을 `t`번 이내에 완료할 때, `t`의 최솟값을 구하라.
- 연속된 구간의 합 중 최댓값을 최소화하라.

이런 문제들은 직접적인 공식이 없는 경우가 많습니다. 하지만 "정답이 `x` 이상/이하일 때 조건을 만족하는가?"라는 **결정 문제**로 바꾸면, 그리디·DP·그래프 탐색 등의 방법으로 `O(N)` 또는 `O(N log N)` 안에 판별할 수 있는 경우가 매우 많습니다. 결정 함수가 단조성을 가지면 이분 탐색 `O(log(범위))` 와 결합해 전체 시간 복잡도를 크게 낮출 수 있습니다.

## 기본 구조와 구현 예제

### 예제 1: 블루레이 디스크에 영상 나누기

N개의 영상을 M개의 블루레이에 순서를 유지하며 담으려 합니다. 블루레이의 크기는 모두 같으며, 그 크기를 최소화하려 합니다. 각 영상의 길이 배열 `lengths`가 주어질 때, 블루레이 크기의 최솟값을 구하세요.

**결정 문제로 변환**: "블루레이 크기가 `mid`일 때 M개 이하로 담을 수 있는가?"

```python
def can_fit(lengths, mid, m):
    """블루레이 크기가 mid일 때 m개 이하의 블루레이로 담을 수 있는지 판별"""
    count = 1
    current = 0
    for length in lengths:
        if length > mid:
            return False  # 단일 영상이 mid를 초과하면 불가능
        if current + length > mid:
            count += 1
            current = 0
        current += length
    return count <= m


def solve_bluray(lengths, m):
    """파라메트릭 서치로 블루레이 최소 크기 탐색"""
    lo = max(lengths)         # 최소 가능 크기: 가장 긴 영상 길이
    hi = sum(lengths)         # 최대 가능 크기: 전체 합산

    answer = hi
    while lo <= hi:
        mid = (lo + hi) // 2
        if can_fit(lengths, mid, m):
            answer = mid
            hi = mid - 1      # 더 작은 크기로 시도
        else:
            lo = mid + 1      # 크기를 늘려야 함

    return answer


# 예시: 9개 영상, 3개의 블루레이
lengths = [1, 2, 3, 4, 5, 6, 7, 8, 9]
m = 3
print(f"블루레이 최소 크기: {solve_bluray(lengths, m)}")
# 출력: 블루레이 최소 크기: 17
```

**핵심 분석**:
- `lo`는 단일 영상의 최대 길이(이 이상이어야 담을 수 있음)
- `hi`는 전체 영상 합산(1개에 전부 담는 경우)
- `check` 함수의 단조성: 크기가 충분하면 항상 가능, 너무 작으면 불가능 → 임계점 존재

### 예제 2: 연속 구간 합의 최솟값을 최대화 (나무 자르기)

N개의 통나무를 `k`개의 인부가 나눠 자를 때, 각 인부가 자르는 구간 합의 최댓값을 최소화하는 문제입니다. 이 역시 "각 인부의 부담이 `mid` 이하일 때 `k`명 이하로 해결 가능한가?"로 변환됩니다.

```java
public class ParametricSearch {

    // 각 구간의 합이 limit 이하가 되도록 나눌 때 필요한 최소 인부 수
    static int countWorkers(int[] logs, int limit) {
        int workers = 1;
        int current = 0;
        for (int log : logs) {
            if (log > limit) return Integer.MAX_VALUE; // 불가능
            if (current + log > limit) {
                workers++;
                current = 0;
            }
            current += log;
        }
        return workers;
    }

    static int solve(int[] logs, int k) {
        long lo = 0;
        long hi = 0;
        for (int log : logs) {
            lo = Math.max(lo, log);  // 단일 원소 최대값
            hi += log;               // 전체 합
        }

        long answer = hi;
        while (lo <= hi) {
            long mid = (lo + hi) / 2;
            if (countWorkers(logs, (int) mid) <= k) {
                answer = mid;
                hi = mid - 1;        // 더 작게 시도
            } else {
                lo = mid + 1;
            }
        }
        return (int) answer;
    }

    public static void main(String[] args) {
        int[] logs = {1, 2, 3, 4, 5, 6, 7, 8, 9};
        int k = 3;
        System.out.println("최대 부담의 최솟값: " + solve(logs, k));
        // 출력: 최대 부담의 최솟값: 17
    }
}
```

## 파라메트릭 서치 적용 패턴

파라메트릭 서치가 적용되는 문제 유형을 패턴별로 정리하면 다음과 같습니다.

| 패턴 | 결정 함수 | 대표 문제 |
|------|---------|---------|
| 최솟값 최대화 | `check(x)`: x 이상 가능한가? | 자원 배분, 거리 최소화 |
| 최댓값 최소화 | `check(x)`: x 이하 가능한가? | 구간 나누기, 부담 최소화 |
| k번째 원소 찾기 | `count(x)`: x 이하 원소 개수 ≥ k? | 행렬에서 k번째 원소 |
| 연속 조건 만족 | `check(x)`: 연속 길이 x 이상 가능? | 연속 부분 수열 |

## 실수 범위에서의 이분 탐색

정수 범위뿐 아니라 실수 범위에서도 파라메트릭 서치를 적용할 수 있습니다. 이 경우 반복 횟수를 고정하거나, 정밀도 기반으로 종료합니다.

```python
def parametric_real(lo: float, hi: float, check, iterations=100) -> float:
    """실수 범위 파라메트릭 서치 — 100회 반복으로 충분한 정밀도 확보"""
    for _ in range(iterations):
        mid = (lo + hi) / 2
        if check(mid):
            hi = mid
        else:
            lo = mid
    return (lo + hi) / 2


# 예: 원의 넓이가 target 이상이 되는 최소 반지름
import math

target_area = 100.0
result = parametric_real(
    lo=0,
    hi=100,
    check=lambda r: math.pi * r * r >= target_area
)
print(f"최소 반지름: {result:.6f}")  # ≈ 5.641896
```

## 주의사항과 팁

**1. 단조성 검증이 선결 조건**
결정 함수 `check(x)`가 단조증가 또는 단조감소해야 이분 탐색이 올바르게 동작합니다. 단조성이 없으면 파라메트릭 서치를 적용할 수 없습니다. 문제를 풀기 전에 반드시 단조성을 확인하세요.

**2. 경계값 설정**
`lo`와 `hi`는 가능한 정답 범위를 완전히 포함해야 합니다. `lo`를 너무 크게 설정하거나 `hi`를 너무 작게 설정하면 올바른 답을 찾지 못합니다. 일반적으로 `lo`는 가능한 최솟값, `hi`는 가능한 최댓값으로 설정합니다.

**3. 오버플로 주의**
정수 범위의 `lo + hi`는 오버플로가 발생할 수 있습니다. `mid = lo + (hi - lo) / 2` 방식을 사용하거나 `long` 자료형을 활용하세요.

**4. check 함수 효율성**
파라메트릭 서치의 총 시간 복잡도는 `O(check 복잡도 × log(범위))`입니다. check 함수 자체가 느리면 전체가 느려집니다. 대부분의 경우 `O(N)` 또는 `O(N log N)` 안에 동작하도록 설계합니다.

**5. 정답 방향 혼동 주의**
"최솟값 최대화"와 "최댓값 최소화"는 이분 탐색 방향이 다릅니다. check가 참일 때 `hi`를 줄일지 `lo`를 늘릴지를 문제 조건에 맞게 정확하게 설정하세요. 코드를 작성할 때 구체적인 예시 하나를 직접 손으로 추적해 보는 것이 실수를 방지하는 가장 좋은 방법입니다.

**6. 정수 이분 탐색 종료 조건**
`while lo <= hi`를 사용할 때 `lo`, `hi`, `mid`의 관계를 주의하세요. 잘못된 종료 조건은 무한 루프를 유발합니다. 특히 `mid = lo`가 될 수 있는 경우를 주의하세요.

파라메트릭 서치는 익숙해지면 직접 공식을 유도하기 어려운 수많은 최적화 문제를 단숨에 해결하는 강력한 도구가 됩니다. "이 값이 가능한가?"라는 결정 문제로 변환할 수 있는지를 항상 먼저 떠올리는 습관을 기르세요.

## 참고 자료
- [CP-Algorithms: Binary Search (cp-algorithms 공식 저장소)](https://raw.githubusercontent.com/cp-algorithms/cp-algorithms/main/src/num_methods/binary_search.md)
- [AhmadElsagheer: Competitive Programming Library — Binary Search Curriculum](https://github.com/AhmadElsagheer/Competitive-programming-library/blob/master/curriculum/outlines/solving_techniques/binary_search.md)
- [williamfiset/Algorithms — Binary Search Implementation (Java)](https://github.com/williamfiset/Algorithms)
