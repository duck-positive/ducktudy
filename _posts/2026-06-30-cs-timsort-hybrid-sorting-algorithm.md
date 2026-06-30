---
layout: post
title: "TimSort 완전 정복: Python과 Java가 선택한 하이브리드 정렬 알고리즘의 원리"
date: 2026-06-30
categories: [cs, computer-science]
tags: [algorithms, sorting, timsort, python, java, merge-sort, insertion-sort]
---

## 개념 설명

TimSort는 2002년 Tim Peters가 Python의 `list.sort()`를 위해 설계한 하이브리드 정렬 알고리즘이다. 병합 정렬(Merge Sort)과 삽입 정렬(Insertion Sort)의 장점을 결합하여, **실세계 데이터에서 이미 정렬된 부분 시퀀스가 자주 존재한다**는 관찰을 핵심 전제로 삼는다.

Python 2.3부터 Python 3.10까지의 기본 정렬 알고리즘으로 사용되었고, Java 7의 `Arrays.sort(Object[])`와 `Collections.sort()`에도 채택되었다. Android 플랫폼과 Swift의 표준 라이브러리도 TimSort 계열 알고리즘을 사용한다.

### 핵심 용어

| 용어 | 설명 |
|------|------|
| **Run** | 이미 (오름차순 또는 내림차순으로) 정렬된 연속 원소의 부분 배열 |
| **minrun** | Run의 최소 길이. 32~64 사이의 값으로 계산됨 |
| **Galloping mode** | 한 쪽 배열에서 연속적인 원소들이 다른 쪽 원소보다 작을 때 이진 탐색으로 전환하는 최적화 |

---

## 왜 필요한가

### 기존 알고리즘의 한계

순수 병합 정렬의 시간 복잡도는 O(n log n)이다. 하지만 이미 정렬된 데이터나 부분적으로 정렬된 데이터에서도 동일하게 O(n log n)의 비교를 수행한다.

반면 삽입 정렬은 이미 정렬된 배열에서 O(n)에 동작하지만, 역순 정렬된 대규모 배열에서는 O(n²)으로 느려진다.

TimSort는 다음 두 가지 관찰을 통해 이 문제를 해결한다:

1. **실세계 데이터는 부분적으로 정렬되어 있다.** 로그 파일, 타임스탬프, 사용자 입력 데이터 등은 완전 무작위가 아니다.
2. **작은 배열은 삽입 정렬이 캐시 친화적이다.** 배열 크기가 32~64 미만이면 삽입 정렬의 상수 계수가 더 유리하다.

### 시간 복잡도 비교

| 알고리즘 | 최선 | 평균 | 최악 | 공간 |
|---------|------|------|------|------|
| TimSort | O(n) | O(n log n) | O(n log n) | O(n) |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) |

TimSort의 최선 케이스가 O(n)인 이유는, 배열 전체가 이미 정렬된 하나의 Run이라면 비교 횟수가 n-1번에 그치기 때문이다.

---

## 실제 구현 예제

### 예제 1: TimSort 핵심 로직을 Python으로 구현

아래는 TimSort의 핵심 단계를 직접 구현한 코드이다. `minrun` 계산, 삽입 정렬로 Run 생성, 병합 정렬로 Run 합치기의 세 단계로 구성된다.

```python
def calc_minrun(n: int) -> int:
    """
    TimSort의 minrun을 계산한다.
    n을 32~64 범위의 값으로 줄이면서 오른쪽으로 시프트한 비트 수를
    남은 값에 더한다. 결과적으로 n/minrun이 2의 거듭제곱에 가까워진다.
    """
    r = 0
    while n >= 64:
        r |= n & 1
        n >>= 1
    return n + r


def insertion_sort(arr: list, left: int, right: int) -> None:
    """삽입 정렬로 arr[left:right+1] 구간을 정렬한다."""
    for i in range(left + 1, right + 1):
        key = arr[i]
        j = i - 1
        while j >= left and arr[j] > key:
            arr[j + 1] = arr[j]
            j -= 1
        arr[j + 1] = key


def merge(arr: list, left: int, mid: int, right: int) -> None:
    """arr[left:mid+1]과 arr[mid+1:right+1]을 병합한다."""
    left_part = arr[left:mid + 1]
    right_part = arr[mid + 1:right + 1]

    i = j = 0
    k = left

    while i < len(left_part) and j < len(right_part):
        # 안정성(stability) 보장: 같은 값이면 왼쪽 배열 원소가 먼저
        if left_part[i] <= right_part[j]:
            arr[k] = left_part[i]
            i += 1
        else:
            arr[k] = right_part[j]
            j += 1
        k += 1

    while i < len(left_part):
        arr[k] = left_part[i]
        i += 1
        k += 1

    while j < len(right_part):
        arr[k] = right_part[j]
        j += 1
        k += 1


def timsort(arr: list) -> list:
    n = len(arr)
    minrun = calc_minrun(n)

    # 1단계: 각 Run을 삽입 정렬로 정렬
    for start in range(0, n, minrun):
        end = min(start + minrun - 1, n - 1)
        insertion_sort(arr, start, end)

    # 2단계: 정렬된 Run들을 병합
    size = minrun
    while size < n:
        for left in range(0, n, 2 * size):
            mid = min(left + size - 1, n - 1)
            right = min(left + 2 * size - 1, n - 1)
            if mid < right:
                merge(arr, left, mid, right)
        size *= 2

    return arr


# 검증
import random
data = [random.randint(0, 1000) for _ in range(500)]
sorted_data = timsort(data[:])
assert sorted_data == sorted(data), "정렬 결과가 다릅니다!"
print("TimSort 검증 성공!")

# 이미 정렬된 데이터에서의 성능 확인
already_sorted = list(range(10000))
import time
start = time.perf_counter()
timsort(already_sorted[:])
elapsed = time.perf_counter() - start
print(f"이미 정렬된 10,000개 원소: {elapsed*1000:.2f}ms")
```

### 예제 2: Galloping Mode 시뮬레이션

Galloping(갤로핑)은 TimSort의 핵심 최적화 중 하나로, 병합 시 한 쪽 배열에서 연속적으로 이기는 원소가 `MIN_GALLOP`(기본값 7)개 이상 나타나면, 이진 탐색으로 전환하여 비교 횟수를 줄이는 기법이다.

```python
import bisect

MIN_GALLOP = 7

def gallop_right(key, arr: list, start: int, end: int) -> int:
    """
    arr[start:end] 에서 key가 들어갈 위치를 갤로핑으로 탐색한다.
    1, 2, 4, 8, ... 간격으로 확장하다가 범위를 좁힌 뒤 이진 탐색한다.
    """
    # 갤로핑 단계: 간격을 2배씩 늘리며 상한 탐색
    offset = 1
    last_offset = 0
    while offset < (end - start) and arr[start + offset] < key:
        last_offset = offset
        offset = (offset << 1) + 1  # 2^k - 1

    # 범위를 실제 배열 크기로 클리핑
    offset = min(offset, end - start)

    # 이진 탐색으로 정확한 삽입 위치 결정
    lo = start + last_offset
    hi = start + offset
    return bisect.bisect_left(arr, key, lo, hi)


def merge_with_galloping(left: list, right: list) -> list:
    """갤로핑 최적화를 적용한 병합"""
    result = []
    i = j = 0
    left_wins = right_wins = 0

    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            result.append(left[i])
            i += 1
            left_wins += 1
            right_wins = 0
        else:
            result.append(right[j])
            j += 1
            right_wins += 1
            left_wins = 0

        # 한 쪽이 MIN_GALLOP번 연속으로 이기면 갤로핑 모드 진입
        if left_wins >= MIN_GALLOP:
            # right[j] 이상인 left의 원소들을 한 번에 복사
            pos = bisect.bisect_left(left, right[j], i)
            result.extend(left[i:pos])
            i = pos
            left_wins = 0
        elif right_wins >= MIN_GALLOP:
            pos = bisect.bisect_left(right, left[i], j)
            result.extend(right[j:pos])
            j = pos
            right_wins = 0

    result.extend(left[i:])
    result.extend(right[j:])
    return result


# 갤로핑이 유리한 케이스: [1,2,...,1000] + [1001,1002,...,2000]
left  = list(range(1, 1001))
right = list(range(1001, 2001))
merged = merge_with_galloping(left, right)
assert merged == list(range(1, 2001))
print(f"갤로핑 병합 결과: {merged[:5]} ... {merged[-5:]}")

# 일반 병합과 비교: 갤로핑은 첫 번째 비교 후 1000개를 한 번에 복사
print("이미 정렬된 두 배열 병합 시 갤로핑이 O(1) 수준으로 처리")
```

---

## 주의사항 및 팁

### 1. 안정성(Stability)이 보장된다

TimSort는 안정 정렬(Stable Sort)이다. 같은 값을 가진 원소의 상대적 순서가 보존되므로, 이미 `last_name`으로 정렬된 사람 목록을 `first_name`으로 다시 정렬해도 `last_name` 순서가 유지된다. Python의 `sorted()`가 이를 보장하기 때문에 다음과 같은 다중 키 정렬이 가능하다:

```python
# 먼저 last_name으로 정렬한 뒤 first_name으로 재정렬해도
# last_name 순서가 유지됨 (안정 정렬 보장)
people = sorted(people, key=lambda x: x.last_name)
people = sorted(people, key=lambda x: x.first_name)
```

### 2. Python 3.11부터 Powersort로 교체됨

Python 3.11부터 TimSort는 **Powersort**로 교체되었다. Powersort는 Run 병합 순서를 더 이론적으로 최적화하여 worst-case 비교 횟수를 줄인다. 그러나 실용적인 차이는 대부분의 경우 미미하다.

### 3. 커스텀 비교자 사용 시 주의

Java에서 `Comparator`를 잘못 구현하면 TimSort가 `IllegalArgumentException`을 발생시킨다. 비교 함수는 반드시 **추이성(transitivity)**, **반대칭성(antisymmetry)**, **전반성(totality)**을 만족해야 한다.

```java
// 잘못된 비교자 — 추이성 위반
Comparator<Integer> broken = (a, b) -> {
    if (a > b) return 1;
    return -1; // 같을 때 -1 반환 → 비교 순서에 따라 다른 결과
};

// 올바른 비교자
Comparator<Integer> correct = Integer::compareTo;
Arrays.sort(arr, correct);
```

### 4. `minrun`은 왜 32~64인가

`minrun`이 이 범위인 이유는 두 가지다:
- **삽입 정렬의 효율**: 배열 크기 64 미만에서는 삽입 정렬이 O(n²)이어도 캐시 지역성 덕분에 실제로 빠르다.
- **2의 거듭제곱에 가까운 분할**: Run 수가 2의 거듭제곱에 가까우면 병합 트리가 균형을 이루어 최종 병합 단계에서 낭비가 없다.

---

## 요약

TimSort는 "실세계 데이터는 완전 무작위가 아니다"는 통찰에서 출발한 실용적인 정렬 알고리즘이다. 이론적 worst-case 복잡도는 O(n log n)이지만, 부분 정렬된 데이터에서는 선형에 가까운 성능을 보인다. 20년 이상 Python과 Java의 기본 정렬로 쓰인 이유는 단순히 빠르기 때문이 아니라, **안정성·예측 가능성·실세계 데이터 특성을 모두 만족하는 균형**에 있다.

---

## 참고 자료

- [CPython listsort.txt — Tim Peters의 원본 설계 문서](https://github.com/python/cpython/blob/main/Objects/listsort.txt)
- [Timsort — Wikipedia](https://en.wikipedia.org/wiki/Timsort)
- [TimSort — GeeksforGeeks](https://www.geeksforgeeks.org/dsa/timsort/)
- [Java Arrays.sort() uses TimSort — OpenJDK Source](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/TimSort.java)
