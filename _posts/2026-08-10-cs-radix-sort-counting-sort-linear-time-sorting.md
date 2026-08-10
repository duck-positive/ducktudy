---
layout: post
title: "기수 정렬(Radix Sort)과 계수 정렬(Counting Sort) 완전 정복: 비교 없이 선형 시간에 정렬하는 원리"
date: 2026-08-10
categories: [cs, computer-science]
tags: [radix-sort, counting-sort, bucket-sort, sorting-algorithms, algorithm, linear-time]
---

정렬 알고리즘을 배우다 보면 반드시 만나는 결론이 있습니다. "비교 기반 정렬의 하한은 Ω(n log n)이다." 하지만 이 하한은 어디까지나 **비교 기반** 알고리즘에만 적용됩니다. 계수 정렬(Counting Sort)과 기수 정렬(Radix Sort)은 원소 간 비교를 전혀 하지 않고, 자릿수와 빈도 정보만을 활용해 **O(n)** 혹은 **O(nk)** 시간에 정렬을 완료하는 비교 비기반(non-comparison-based) 알고리즘입니다. 이 글에서는 두 알고리즘의 원리, 구현, 그리고 실전에서의 활용 방법을 깊이 살펴봅니다.

---

## 1. 왜 비교 기반 정렬의 한계를 극복해야 하는가

머지 소트, 퀵 소트, 힙 소트는 두 원소를 비교해 순서를 결정합니다. n개 원소의 모든 순열 수는 n!이므로, 결정 트리(decision tree)의 리프 노드가 n! 이상이어야 합니다. 이진 결정 트리의 높이는 최소 ⌈log₂(n!)⌉ ≈ n log n이므로, **최악의 경우 반드시 Ω(n log n)번의 비교가 필요**합니다.

반면, 입력 원소가 특정 범위의 정수라는 사실을 안다면 완전히 다른 전략을 사용할 수 있습니다. "각 값이 몇 번 등장하는가"만 세어도 정렬된 결과를 즉시 만들어낼 수 있습니다. 이것이 계수 정렬의 핵심 아이디어입니다.

---

## 2. 계수 정렬(Counting Sort)

### 2.1 개념

계수 정렬은 입력 배열의 각 원소 값을 **인덱스**로 사용하는 보조 배열(count array)에 빈도를 기록한 뒤, 누적 합(prefix sum)으로 각 원소의 최종 위치를 결정합니다.

알고리즘의 3단계:
1. **Count**: 각 값의 등장 횟수를 count 배열에 기록
2. **Cumulate**: count 배열을 누적 합 배열로 변환 → 각 값의 마지막 위치 + 1
3. **Place**: 원본 배열을 역방향으로 순회하며 output 배열의 올바른 위치에 삽입 (안정성 보장)

### 2.2 시간·공간 복잡도

- **시간**: O(n + k), 여기서 k는 값의 범위 (max - min + 1)
- **공간**: O(n + k) (보조 배열)
- **안정성**: 역방향 순회 덕분에 동일 키 원소들의 상대 순서가 보존됨 (stable)

k가 n에 비해 매우 크면 오히려 비효율적이므로, **k = O(n)일 때 가장 유리**합니다.

### 2.3 Python 구현

```python
def counting_sort(arr: list[int], max_val: int) -> list[int]:
    """
    arr: 정렬할 배열 (0 이상 max_val 이하 정수)
    반환: 정렬된 새 배열
    """
    count = [0] * (max_val + 1)

    # 1단계: 빈도 기록
    for x in arr:
        count[x] += 1

    # 2단계: 누적 합 (각 값의 마지막 위치 + 1)
    for i in range(1, max_val + 1):
        count[i] += count[i - 1]

    # 3단계: 역방향 순회하며 올바른 위치에 삽입 (안정 정렬 보장)
    output = [0] * len(arr)
    for x in reversed(arr):
        count[x] -= 1
        output[count[x]] = x

    return output


# 테스트
data = [4, 2, 2, 8, 3, 3, 1]
print(counting_sort(data, max_val=8))
# [1, 2, 2, 3, 3, 4, 8]
```

계수 정렬의 핵심은 `reversed(arr)` 로 역순 삽입입니다. 앞에서부터 순회하면 동일 값에 대해 순서가 뒤집혀 안정성이 깨집니다.

---

## 3. 기수 정렬(Radix Sort)

### 3.1 개념

기수 정렬은 **자리 수별로 반복 정렬**합니다. 예를 들어 10진수 3자리 정수라면 1의 자리 → 10의 자리 → 100의 자리 순으로 세 번의 계수 정렬을 수행합니다. 각 자리 정렬에서 **안정 정렬**을 사용하면, 상위 자리가 같을 때 하위 자리의 순서가 그대로 유지되어 최종 결과가 올바릅니다.

두 가지 방식:
- **LSD(Least Significant Digit)**: 가장 낮은 자리부터 → 대부분의 실용 구현
- **MSD(Most Significant Digit)**: 가장 높은 자리부터 → 사전식 정렬에 적합

### 3.2 시간·공간 복잡도

- **시간**: O(d × (n + b)), d는 자릿수, b는 진법(base)
- d = log_b(max_val)이므로 → O((n + b) × log_b(max_val))
- b = n으로 설정하면 d = O(log_n(max_val)), 전체 O(n × log_n(max_val))
- max_val = n^c (상수)이면 d = O(c) → **O(n)**

### 3.3 Python 구현 (LSD, 10진수)

```python
def radix_sort(arr: list[int]) -> list[int]:
    """LSD 기수 정렬 (10진수, 비음수 정수)"""
    if not arr:
        return arr

    max_val = max(arr)
    exp = 1  # 현재 자릿수의 지수 (1, 10, 100, ...)

    result = arr[:]
    while max_val // exp > 0:
        result = _counting_sort_by_digit(result, exp)
        exp *= 10

    return result


def _counting_sort_by_digit(arr: list[int], exp: int) -> list[int]:
    """특정 자릿수(exp)를 키로 사용하는 안정 계수 정렬"""
    n = len(arr)
    count = [0] * 10
    output = [0] * n

    # 해당 자릿수의 빈도 계산
    for x in arr:
        digit = (x // exp) % 10
        count[digit] += 1

    # 누적 합
    for i in range(1, 10):
        count[i] += count[i - 1]

    # 역방향 순회로 안정성 보장
    for x in reversed(arr):
        digit = (x // exp) % 10
        count[digit] -= 1
        output[count[digit]] = x

    return output


# 테스트
data = [170, 45, 75, 90, 802, 24, 2, 66]
print(radix_sort(data))
# [2, 24, 45, 66, 75, 90, 170, 802]
```

위 구현의 핵심은 `_counting_sort_by_digit`이 **안정 정렬**이라는 점입니다. 만약 이 단계에서 안정성이 깨지면 이전 자리의 정렬 결과가 덮어씌워져 최종 결과가 틀립니다.

---

## 4. 버킷 정렬(Bucket Sort)

버킷 정렬은 입력이 균등 분포를 따를 때 평균 O(n)에 동작하는 알고리즘입니다. [0, 1) 범위의 실수 배열을 n개의 버킷으로 분할하고, 각 버킷 내부를 삽입 정렬로 마무리합니다.

```python
def bucket_sort(arr: list[float]) -> list[float]:
    """
    arr: [0.0, 1.0) 범위 실수 배열
    평균 O(n), 균등 분포 가정
    """
    n = len(arr)
    if n == 0:
        return arr

    # n개의 빈 버킷 생성
    buckets: list[list[float]] = [[] for _ in range(n)]

    for x in arr:
        idx = int(x * n)          # [0, n) 범위의 버킷 인덱스
        idx = min(idx, n - 1)     # 경계 처리 (x == 1.0 방어)
        buckets[idx].append(x)

    # 각 버킷 내부 삽입 정렬 후 병합
    result: list[float] = []
    for bucket in buckets:
        bucket.sort()             # 소규모이므로 삽입 정렬과 동등
        result.extend(bucket)

    return result


data = [0.897, 0.565, 0.656, 0.123, 0.665, 0.343]
print(bucket_sort(data))
# [0.123, 0.343, 0.565, 0.656, 0.665, 0.897]
```

---

## 5. 실전 활용 가이드

### 5.1 언제 기수/계수 정렬을 선택해야 하는가

| 상황 | 추천 알고리즘 |
|---|---|
| 키가 제한된 정수 범위, k = O(n) | 계수 정렬 |
| 정수 또는 고정 자릿수 문자열, d 작음 | LSD 기수 정렬 |
| 실수, 균등 분포 | 버킷 정렬 |
| 범용, 제약 없음 | 퀵소트 / 머지소트 |

### 5.2 기수 정렬의 진법 선택

진법 b를 크게 잡을수록 자릿수 d가 줄어들지만 계수 정렬의 k(= b)가 커집니다. 최적은 b ≈ n으로, 이 경우 d = O(log_n(max_val)) = O(1) (max_val = n^c 가정)이 됩니다. 실무에서는 b = 256(8비트 단위)이 흔히 사용됩니다.

### 5.3 주의사항

1. **메모리 사용량**: 계수 배열과 출력 배열이 추가로 필요합니다. 인플레이스(in-place) 불가.
2. **음수 처리**: 음수가 있다면 오프셋(offset)을 더해 0 이상으로 변환하거나, 부호 비트를 마지막 자리로 분리하는 추가 처리가 필요합니다.
3. **안정성 필수**: 기수 정렬은 각 단계가 **반드시 안정 정렬**이어야 합니다. 불안정한 서브 정렬을 쓰면 결과가 틀립니다.
4. **키 타입 제한**: 부동 소수점 등 비정수 키에 직접 적용하기 어렵습니다 (비트 패턴 변환 필요).

---

## 6. 정리

계수 정렬과 기수 정렬은 비교 기반 알고리즘의 Ω(n log n) 하한을 "비교를 하지 않는다"는 발상으로 우회합니다. 입력 도메인에 대한 추가 가정(범위, 자릿수)이 성립할 때 이 알고리즘들은 큰 n에서 압도적으로 빠릅니다. 실제로 데이터베이스 해시 조인, 네트워크 패킷 분류, 정수 키 대규모 정렬 등에서 기수 정렬이 널리 사용됩니다. 알고리즘을 선택할 때는 "입력에 대해 내가 무엇을 알고 있는가"를 먼저 따져보는 습관이 중요합니다.

## 참고 자료
- [Radix Sort - VisuAlgo](https://visualgo.net/en/sorting)
- [Radix Sort - Brilliant Math & Science Wiki](https://brilliant.org/wiki/radix-sort/)
- [11.2 Counting Sort and Radix Sort - Open Data Structures](https://opendatastructures.org/ods-java/11_2_Counting_Sort_Radix_So.html)
- [Radix Sort - OI Wiki](https://en.oi-wiki.org/basic/radix-sort/)
