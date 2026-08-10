---
layout: post
title: "캐시 비인식 알고리즘(Cache-Oblivious Algorithms) 완전 정복: 캐시 크기를 몰라도 모든 계층을 최적으로 활용하는 설계 원리"
date: 2026-08-10
categories: [cs, computer-science]
tags: [cache-oblivious, cache-efficient, algorithms, memory-hierarchy, divide-and-conquer, performance]
---

현대 CPU와 메모리 사이에는 L1, L2, L3 캐시라는 여러 층의 고속 메모리가 존재합니다. 프로그램의 성능은 알고리즘의 이론적 복잡도 못지않게 **캐시를 얼마나 잘 활용하는가**에 의해 결정됩니다. 

캐시를 의식한 알고리즘(cache-aware)은 특정 캐시 크기 M과 캐시 라인 크기 B를 알고 이를 이용해 블록 크기를 최적화합니다. 하지만 캐시 파라미터는 CPU 모델마다 다르고, 멀티레벨 계층에서는 한 레벨에 최적화하면 다른 레벨이 손해를 볼 수 있습니다.

**캐시 비인식 알고리즘(Cache-Oblivious Algorithms)**은 캐시 크기 M이나 라인 크기 B를 전혀 알지 못해도, **재귀적 분할 정복 구조** 자체로 모든 메모리 계층에서 점근적으로 최적인 캐시 동작을 달성합니다. 1999년 Frigo, Leiserson, Prokop, Ramachandran이 제안한 이 패러다임은 데이터베이스, 행렬 연산 라이브러리, 파일 시스템에서 폭넓게 활용됩니다.

---

## 1. 이상적인 캐시 모델(Ideal Cache Model)

캐시 비인식 알고리즘의 분석은 **이상적인 캐시 모델(Ideal Cache Model, ICM)**을 기반으로 합니다:

- 2-레벨 메모리: 크기 M인 캐시(빠름)와 무한 크기의 주 메모리(느림)
- 캐시 라인 크기 B (단위 전송 블록)
- **완전 연관(fully associative)**: 어떤 메모리 블록도 어느 캐시 라인에든 들어갈 수 있음
- **최적 교체 정책(OPT)**: 향후 가장 오래 사용되지 않을 블록을 교체

이 모델에서 알고리즘의 **캐시 복잡도(I/O complexity)**는 "전체 메모리에서 캐시로 블록을 몇 번 전송하는가"입니다.

핵심 정리: 이상적인 캐시 모델에서 최적인 캐시 비인식 알고리즘은, 상수 인수의 오차 내에서 실제 임의의 캐시 계층에서도 최적으로 동작합니다.

---

## 2. 일반 배열 접근 vs 캐시 비인식: 행렬 전치 예제

### 2.1 순진한 행렬 전치 (캐시 비친화적)

n×n 행렬의 전치를 단순 이중 루프로 구현하면:

```python
def naive_transpose(A: list[list[float]], n: int) -> list[list[float]]:
    """순진한 행렬 전치 - 캐시 비친화적"""
    B = [[0.0] * n for _ in range(n)]
    for i in range(n):
        for j in range(n):
            B[j][i] = A[i][j]
    return B
```

행 우선 저장(row-major)에서 `A[i][j]`는 캐시 친화적이지만(같은 행의 원소가 연속 메모리), `B[j][i]`는 열 방향으로 접근하므로 **캐시 미스가 O(n²/B)**번 발생합니다. n이 크면 캐시 라인 1개에 1개 원소만 사용하는 최악의 패턴이 됩니다.

### 2.2 캐시 비인식 행렬 전치 (재귀적 분할)

재귀적으로 행렬을 분할하면, 부분 행렬이 캐시에 들어갈 만큼 작아질 때 자동으로 지역성(locality)이 발생합니다:

```python
def cache_oblivious_transpose(
    A: list[list[float]], B: list[list[float]],
    r0: int, c0: int, r1: int, c1: int,  # 원본 A의 부분 행렬 [r0:r1, c0:c1]
    br0: int, bc0: int                    # B에서의 시작 위치 (전치된 좌표)
) -> None:
    """
    재귀적 분할로 캐시 비인식 전치 수행.
    B[bc0 : bc0+(c1-c0), br0 : br0+(r1-r0)] = A[r0:r1, c0:c1].T
    """
    rows = r1 - r0
    cols = c1 - c0

    if rows == 0 or cols == 0:
        return

    # 기저 사례: 블록이 충분히 작으면 직접 처리
    if rows <= 16 and cols <= 16:
        for i in range(rows):
            for j in range(cols):
                B[bc0 + j][br0 + i] = A[r0 + i][c0 + j]
        return

    # 더 긴 차원을 반으로 분할 (정사각형에 가깝게 유지)
    if rows >= cols:
        mid = rows // 2
        cache_oblivious_transpose(A, B, r0, c0, r0 + mid, c1, br0, bc0)
        cache_oblivious_transpose(A, B, r0 + mid, c0, r1, c1, br0 + mid, bc0)
    else:
        mid = cols // 2
        cache_oblivious_transpose(A, B, r0, c0, r1, c0 + mid, br0, bc0)
        cache_oblivious_transpose(A, B, r0, c0 + mid, r1, c1, br0, bc0 + mid)


def make_matrix(rows: int, cols: int, val_fn=lambda i, j: float(i * 10 + j)):
    return [[val_fn(i, j) for j in range(cols)] for i in range(rows)]


n = 6
A = make_matrix(n, n)
B = [[0.0] * n for _ in range(n)]
cache_oblivious_transpose(A, B, 0, 0, n, n, 0, 0)

# 검증
for i in range(n):
    for j in range(n):
        assert B[i][j] == A[j][i], f"B[{i}][{j}] = {B[i][j]} != A[{j}][{i}] = {A[j][i]}"
print("전치 검증 성공")
```

이 알고리즘의 캐시 복잡도는 **Θ(n²/B)**로, 이는 모든 n²개 원소를 적어도 한 번씩 읽고 써야 하므로 최적입니다. 핵심은 블록 크기를 명시적으로 설정하지 않아도 재귀 분할이 자연스럽게 캐시에 맞는 크기로 수렴한다는 것입니다.

---

## 3. 캐시 비인식 병합 정렬

표준 병합 정렬은 이미 재귀 구조를 가지므로 기본적으로 캐시 비인식입니다. 하지만 임시 배열 생성 패턴을 최적화하면 더욱 캐시 효율적으로 만들 수 있습니다.

```python
def cache_oblivious_mergesort(arr: list[int], aux: list[int], lo: int, hi: int) -> None:
    """
    캐시 비인식 병합 정렬 (in-place aux 배열 사용).
    arr[lo:hi]를 정렬합니다.
    """
    if hi - lo <= 1:
        return

    mid = (lo + hi) // 2

    # 두 절반을 재귀 정렬
    cache_oblivious_mergesort(arr, aux, lo, mid)
    cache_oblivious_mergesort(arr, aux, mid, hi)

    # 두 정렬된 절반을 병합
    _merge(arr, aux, lo, mid, hi)


def _merge(arr: list[int], aux: list[int], lo: int, mid: int, hi: int) -> None:
    """arr[lo:mid]와 arr[mid:hi]를 병합하여 arr[lo:hi]에 저장"""
    # aux에 복사
    for k in range(lo, hi):
        aux[k] = arr[k]

    i, j, k = lo, mid, lo
    while i < mid and j < hi:
        if aux[i] <= aux[j]:
            arr[k] = aux[i]; i += 1
        else:
            arr[k] = aux[j]; j += 1
        k += 1

    while i < mid:
        arr[k] = aux[i]; i += 1; k += 1
    while j < hi:
        arr[k] = aux[j]; j += 1; k += 1


# 테스트
import random
data = list(range(20))
random.shuffle(data)
print("Before:", data[:10], "...")
aux = [0] * len(data)
cache_oblivious_mergesort(data, aux, 0, len(data))
print("After: ", data[:10], "...")
assert data == sorted(data), "정렬 실패"
print("병합 정렬 검증 성공")
```

병합 정렬의 캐시 복잡도:
- **비교 횟수**: O(n log n) — 비교 기반 최적
- **캐시 미스 수**: Θ(n/B × log(n/M) + n/B) = **Θ((n/B) log_{M/B}(n/B))** — 외부 정렬 하한과 일치하는 최적값

이것이 캐시 비인식 알고리즘의 위력입니다. M이나 B를 코드에 하드코딩하지 않고도 임의의 캐시 파라미터에서 최적 I/O 복잡도를 달성합니다.

---

## 4. B-트리 vs 캐시 비인식 트리

### 4.1 기존 B-트리의 한계

B-트리는 블록 크기 B를 알고 노드 크기를 B로 설정해 캐시 효율을 극대화합니다. 하지만 B가 다른 시스템이나 다른 캐시 레벨에서는 최적이 아닙니다.

### 4.2 van Emde Boas 레이아웃 (캐시 비인식 정적 탐색 트리)

완전 이진 트리를 **van Emde Boas(vEB) 레이아웃**으로 메모리에 배치하면 캐시 비인식하게 최적 탐색 성능을 달성합니다.

```python
class CacheObliviousStaticTree:
    """
    van Emde Boas 레이아웃을 사용한 캐시 비인식 정적 탐색 트리.
    정렬된 배열을 vEB 레이아웃으로 변환하여 저장.
    """
    def __init__(self, sorted_arr: list[int]):
        self.n = len(sorted_arr)
        self.data = [0] * self.n
        if self.n > 0:
            self._build(sorted_arr, 0, self.n, 0)

    def _build(self, arr: list[int], lo: int, hi: int, pos: int) -> None:
        """
        arr[lo:hi]를 vEB 레이아웃으로 data에 재귀적으로 배치.
        상위 절반 트리를 먼저, 하위 절반 트리들을 이어서 배치.
        """
        n = hi - lo
        if n == 0:
            return
        if n == 1:
            self.data[pos] = arr[lo]
            return

        # 상위 서브트리 크기: 전체 n개 중 √n 크기의 상단 트리
        top_size = 1
        while top_size * top_size < n:
            top_size *= 2
        top_size //= 2
        if top_size == 0:
            top_size = 1

        # 상단 트리에 들어갈 원소 인덱스 (중앙 원소들)
        step = n // (top_size + 1) if top_size + 1 <= n else 1
        top_keys = [arr[lo + (i + 1) * step - 1] for i in range(top_size) if lo + (i + 1) * step - 1 < hi]

        # 단순화: 실제 vEB 레이아웃은 복잡하므로 여기서는 원리만 시연
        # 실용 구현은 Brodal et al. (2002) 참고
        mid = lo + n // 2
        self.data[pos] = arr[mid]  # 현재 노드는 중앙값
        left_size = mid - lo
        if pos * 2 + 1 < self.n:
            self._build(arr, lo, mid, pos * 2 + 1)
        if pos * 2 + 2 < self.n:
            self._build(arr, mid + 1, hi, pos * 2 + 2)

    def search(self, key: int) -> bool:
        """트리에서 key 탐색"""
        pos = 0
        while pos < self.n:
            if self.data[pos] == key:
                return True
            elif key < self.data[pos]:
                pos = pos * 2 + 1
            else:
                pos = pos * 2 + 2
        return False


tree = CacheObliviousStaticTree(list(range(1, 16, 2)))  # [1,3,5,7,9,11,13,15]
print("탐색 7:", tree.search(7))   # True
print("탐색 4:", tree.search(4))   # False
print("탐색 15:", tree.search(15)) # True
```

vEB 레이아웃의 캐시 복잡도: O(log_B n)의 캐시 미스로 탐색 — 이는 B를 알고 설계한 B-트리와 동일한 최적값입니다.

---

## 5. 캐시 비인식 알고리즘의 복잡도 요약

| 알고리즘 | I/O 복잡도 | 최적 여부 |
|---|---|---|
| 행렬 전치 (캐시 비인식) | Θ(n²/B) | 최적 |
| 병합 정렬 (캐시 비인식) | Θ((n/B) log_{M/B}(n/B)) | 최적 |
| 행렬 곱 (Strassen 변형) | Θ(n³ / (B√M)) | 최적 |
| vEB 정적 탐색 트리 | O(log_B n) | 최적 |
| 순진한 이중 루프 전치 | Θ(n²) | 비최적 |

---

## 6. 주의사항 및 실전 팁

1. **재귀 오버헤드**: 재귀 분할의 함수 호출 오버헤드가 작은 n에서 문제가 될 수 있습니다. 기저 사례(16×16 등)를 충분히 크게 설정해 완화합니다.

2. **이상적인 캐시 모델의 한계**: 실제 캐시는 fully-associative가 아닌 set-associative이고 LRU가 OPT보다 상수 인수만큼 나쁩니다. 하지만 정리상 상수 인수 내에서 최적이므로 점근적으로는 동일합니다.

3. **실용적 구현 복잡성**: vEB 레이아웃, 캐시 비인식 행렬 곱 등은 구현이 상당히 복잡합니다. 실제로는 FFTW, ATLAS, Intel MKL 등의 라이브러리가 내부적으로 이 기법을 사용하므로, 직접 구현보다는 이런 라이브러리를 활용하는 것이 현실적입니다.

4. **적용 적합 분야**: 대규모 행렬 연산, 외부 정렬, 데이터베이스 인덱싱, 파일 시스템 B-트리 등 I/O 바운드 워크로드에서 효과가 큽니다. CPU 바운드 계산에는 효과가 제한됩니다.

5. **언어/컴파일러 지원**: C/C++에서는 재귀 인라이닝과 함께 효과적입니다. GCC의 `-O3` 최적화와 결합하면 실용적인 성능을 얻을 수 있습니다.

---

## 7. 정리

캐시 비인식 알고리즘은 "캐시 크기를 모른다"는 것을 약점이 아닌 강점으로 전환합니다. 재귀적 분할 정복 구조 자체가 모든 메모리 계층을 자동으로 최적화하기 때문입니다. M과 B를 하드코딩한 캐시 인식 알고리즘보다 이식성이 뛰어나고, 멀티레벨 캐시 계층 전체에서 동시에 최적 성능을 냅니다. "데이터를 재귀적으로 반으로 나눈다"는 간단한 원칙이 복잡한 메모리 계층에 자동으로 적응하는 강력한 설계임을 기억하세요.

## 참고 자료
- [Cache-Oblivious Algorithms and Data Structures - Erik Demaine (MIT)](https://erikdemaine.org/papers/BRICS2002/)
- [MIT 6.046J Lecture 24: Cache-Oblivious Algorithms - MIT OpenCourseWare](https://ocw.mit.edu/courses/6-046j-design-and-analysis-of-algorithms-spring-2015/resources/lecture-24-cache-oblivious-algorithms-searching-sorting/)
- [Cache Oblivious Algorithm - GeeksforGeeks](https://www.geeksforgeeks.org/operating-systems/cache-oblivious-algorithm/)
- [MIT Introduction to Algorithms - Cache Oblivious Algorithms (catonmat.net)](https://catonmat.net/mit-introduction-to-algorithms-part-fourteen)
