---
layout: post
title: "정렬 네트워크(Sorting Networks) 완전 정복: 비교기 회로와 병렬 정렬의 수학"
date: 2026-09-22
categories: [cs, computer-science]
tags: [sorting-networks, bitonic-sort, parallel-computing, algorithms, comparator, gpu, simd]
---

## 개념 설명

일반적인 정렬 알고리즘은 데이터에 따라 비교 순서가 달라집니다. QuickSort는 피벗 선택에 따라, MergeSort는 재귀 분할에 따라 동적으로 결정됩니다. 이런 알고리즘은 **데이터 의존적(data-dependent)**입니다.

**정렬 네트워크(Sorting Network)**는 완전히 다른 접근법입니다. 비교할 원소 쌍의 순서가 **입력 데이터와 무관하게** 미리 고정되어 있습니다. 회로도처럼 비교기(comparator)들이 배선으로 연결된 고정 구조이며, 어떤 입력이 들어와도 동일한 순서로 비교를 수행합니다.

### 비교기(Comparator)

정렬 네트워크의 기본 단위는 **비교기(Comparator)**입니다. 두 입력 a, b를 받아 min(a,b)와 max(a,b)를 출력합니다.

```
  a ──┬── min(a,b)
      ⊗
  b ──┴── max(a,b)
```

비교기는 두 원소를 비교해 필요하면 교환(swap)합니다. 정렬 네트워크는 이런 비교기를 여러 층(layer)으로 쌓아 올립니다. 같은 층의 비교기들은 서로 독립적이므로 **병렬로 실행**됩니다.

## 왜 정렬 네트워크인가

### 예측 가능한 실행 시간

정렬 네트워크는 입력값과 무관하게 항상 동일한 수의 비교를 수행합니다. 최악/평균/최선이 동일합니다. 실시간 시스템, 하드웨어 회로, 보안 소프트웨어(타이밍 사이드채널 방지)에서 이 특성이 중요합니다.

### SIMD 및 GPU 친화적

현대 CPU의 SIMD(AVX2, AVX-512)와 GPU는 같은 연산을 여러 데이터에 동시에 적용하는 데 특화되어 있습니다. 정렬 네트워크는 비교 순서가 고정되어 있어 SIMD 레지스터에 직접 매핑할 수 있습니다. CUDA GPU에서 빠른 정렬을 구현할 때 Bitonic Sort가 표준처럼 사용됩니다.

### 하드웨어 구현

FPGA나 ASIC에서 정렬 회로를 직접 구현할 때 비교기 배열로 정렬 네트워크를 합성합니다. 소규모 고정 크기 배열(예: 16개 센서값 실시간 정렬)에는 전용 하드웨어 정렬 네트워크가 이상적입니다.

## Bitonic Sort (바이토닉 정렬)

1968년 Ken Batcher가 고안한 Bitonic Sort는 가장 널리 사용되는 정렬 네트워크입니다.

### 바이토닉 수열(Bitonic Sequence)

바이토닉 수열은 **단조 증가 후 단조 감소(또는 그 반대)**하는 수열입니다.
- `[1, 3, 5, 8, 6, 4, 2]` — 증가 후 감소
- `[9, 6, 3, 1, 2, 5, 7]` — 감소 후 증가

### 핵심 성질: Bitonic Split

길이 2n의 바이토닉 수열의 앞 n개와 뒤 n개를 각 위치별로 min/max 비교하면, 앞 n개는 모두 뒤 n개보다 작은 바이토닉 수열이 됩니다. 이를 재귀적으로 반복하면 정렬이 완성됩니다.

### 구현

```python
def bitonic_sort(arr: list, ascending: bool = True) -> list:
    """
    Bitonic Sort — O(n log²n) 비교, O(log²n) 병렬 단계
    """
    arr = arr[:]
    _bitonic_sort_recursive(arr, 0, len(arr), ascending)
    return arr

def _bitonic_sort_recursive(arr, lo, cnt, ascending):
    if cnt <= 1:
        return
    mid = cnt // 2
    # 앞 절반은 오름차순, 뒤 절반은 내림차순으로 정렬 → 바이토닉 수열 생성
    _bitonic_sort_recursive(arr, lo, mid, ascending=True)
    _bitonic_sort_recursive(arr, lo + mid, cnt - mid, ascending=False)
    # 전체를 병합
    _bitonic_merge(arr, lo, cnt, ascending)

def _bitonic_merge(arr, lo, cnt, ascending):
    if cnt <= 1:
        return
    mid = _greatest_power_of_two_less_than(cnt)
    for i in range(lo, lo + cnt - mid):
        if (arr[i] > arr[i + mid]) == ascending:
            arr[i], arr[i + mid] = arr[i + mid], arr[i]
    _bitonic_merge(arr, lo, mid, ascending)
    _bitonic_merge(arr, lo + mid, cnt - mid, ascending)

def _greatest_power_of_two_less_than(n: int) -> int:
    k = 1
    while k < n:
        k <<= 1
    return k >> 1


# 테스트
import random
data = random.sample(range(100), 16)
print("입력:", data)
result = bitonic_sort(data)
print("출력:", result)
assert result == sorted(data), "정렬 실패!"
print("검증 통과!")
```

### 복잡도 분석

크기 n에 대해 Bitonic Sort의 구조:
- **비교기 수**: O(n log²n)
- **병렬 단계 수**: O(log²n)  
  (단계 i에서 전체 단계는 log(n)×(log(n)+1)/2)
- **시간 복잡도** (단일 프로세서): O(n log²n)
- **병렬 시간 복잡도**: O(log²n) — 같은 단계의 비교기는 동시 실행

n=16에 대한 구체적인 비교기 수: 6단계 × 8비교 = 48개 비교기

```
단계 구조 (n=8의 경우, ↑는 오름차순 비교기):
Layer 1:  [0↑1] [2↓3] [4↑5] [6↓7]           → 4 비교기, 병렬
Layer 2:  [0↑2] [1↑3] [4↓6] [5↓7]           → 4 비교기, 병렬
Layer 3:  [0↑1] [2↑3] [4↓5] [6↓7]           → 4 비교기, 병렬
Layer 4:  [0↑4] [1↑5] [2↑6] [3↑7]           → 4 비교기, 병렬
Layer 5:  [0↑2] [1↑3] [4↑6] [5↑7]           → 4 비교기, 병렬
Layer 6:  [0↑1] [2↑3] [4↑5] [6↑7]           → 4 비교기, 병렬
```

## Batcher의 Odd-Even Merge Sort

Batcher는 Bitonic Sort와 함께 **Odd-Even Merge Sort**도 고안했습니다. 홀수 인덱스와 짝수 인덱스를 따로 병합하는 방식으로, 임의 크기(2의 거듭제곱이 아닌)도 처리할 수 있습니다.

```python
def odd_even_merge_sort(arr: list) -> list:
    arr = arr[:]
    n = len(arr)
    _oe_merge_sort(arr, 0, n)
    return arr

def _oe_merge_sort(arr, lo, n):
    if n <= 1:
        return
    mid = n // 2
    _oe_merge_sort(arr, lo, mid)
    _oe_merge_sort(arr, lo + mid, n - mid)
    _oe_merge(arr, lo, n, 1)

def _oe_merge(arr, lo, n, step):
    if n <= 1:
        return
    if n == 2:
        if arr[lo] > arr[lo + step]:
            arr[lo], arr[lo + step] = arr[lo + step], arr[lo]
        return
    # 홀수 인덱스와 짝수 인덱스를 따로 병합
    _oe_merge(arr, lo, n // 2, step * 2)         # 짝수 인덱스
    _oe_merge(arr, lo + step, n // 2, step * 2)  # 홀수 인덱스
    # 인접한 쌍 비교
    for i in range(lo + step, lo + step * (n - 1), step * 2):
        if arr[i] > arr[i + step]:
            arr[i], arr[i + step] = arr[i + step], arr[i]


# 비교 테스트
data = [64, 25, 12, 22, 11, 90, 45, 33]
print("Bitonic Sort:  ", bitonic_sort(data))
print("OE Merge Sort: ", odd_even_merge_sort(data))
print("Python sorted: ", sorted(data))
```

## 고정 크기 최적 정렬 네트워크

소규모 배열에서는 **최소 비교기 수**를 사용하는 정렬 네트워크가 알려져 있습니다. 이를 **최적 정렬 네트워크**라고 합니다.

```python
def sort4_optimal(a, b, c, d):
    """
    4원소 최적 정렬 네트워크 — 5개 비교기 사용
    (이론적 최솟값)
    """
    # Layer 1 (병렬)
    if a > c: a, c = c, a
    if b > d: b, d = d, b
    # Layer 2 (병렬)
    if a > b: a, b = b, a
    if c > d: c, d = d, c
    # Layer 3
    if b > c: b, c = c, b
    return a, b, c, d

def sort8_network(arr):
    """
    8원소 정렬 네트워크 — 19개 비교기
    (Knuth TAOCP Vol. 3에서 알려진 최적 네트워크)
    """
    a = arr[:]
    def cmp(i, j):
        if a[i] > a[j]: a[i], a[j] = a[j], a[i]
    
    # Layer 1
    cmp(0,1); cmp(2,3); cmp(4,5); cmp(6,7)
    # Layer 2
    cmp(0,2); cmp(1,3); cmp(4,6); cmp(5,7)
    # Layer 3
    cmp(1,2); cmp(5,6)
    # Layer 4
    cmp(0,4); cmp(1,5); cmp(2,6); cmp(3,7)
    # Layer 5
    cmp(2,4); cmp(3,5)
    # Layer 6
    cmp(1,2); cmp(3,4); cmp(5,6)
    
    return a

# 검증
import random
for _ in range(1000):
    test = random.sample(range(1000), 8)
    assert sort8_network(test) == sorted(test), "네트워크 정렬 오류!"
print("sort8_network: 1000회 랜덤 테스트 통과")
```

## 정렬 네트워크의 이론적 한계

### AKS 네트워크

1983년 Ajtai, Komlós, Szemerédi는 O(n log n)개의 비교기만 사용하는 정렬 네트워크의 존재를 증명했습니다(AKS 네트워크). 이는 이론적으로 최적입니다.

그러나 AKS 네트워크의 상수 인수가 매우 크기 때문에 실용적이지 않습니다. 실무에서는 Bitonic Sort나 Odd-Even Merge Sort가 여전히 주류입니다.

### 0-1 원리(Zero-One Principle)

정렬 네트워크 설계에서 중요한 검증 정리입니다: **0과 1로 이루어진 모든 입력을 올바르게 정렬하면, 임의의 값도 올바르게 정렬합니다.** n비트 입력에 대해 2^n가지를 테스트하면 정렬 네트워크의 정확성을 완전히 검증할 수 있습니다.

```python
def verify_sorting_network(network_fn, n: int) -> bool:
    """
    0-1 원리로 정렬 네트워크 검증
    n개 원소 → 2^n가지 0/1 입력 테스트
    """
    from itertools import product
    for bits in product([0, 1], repeat=n):
        inp = list(bits)
        out = network_fn(inp)
        if out != sorted(out):
            print(f"실패: 입력 {inp} → 출력 {out}")
            return False
    print(f"0-1 원리 검증 통과: 2^{n} = {2**n}가지 입력 모두 정렬 성공")
    return True

verify_sorting_network(sort8_network, 8)
```

## 실전 활용 — GPU 정렬

CUDA에서 Bitonic Sort는 공유 메모리를 활용해 워프(warp) 단위로 병렬화합니다.

```python
# GPU Bitonic Sort 의사코드 (CUDA 스타일)
# 실제 CUDA 코드를 Python으로 표현

def gpu_bitonic_sort_simulation(arr):
    """
    GPU 스타일 Bitonic Sort 시뮬레이션
    각 스레드가 하나의 비교기 담당
    """
    n = len(arr)
    arr = arr[:]
    
    # k: 현재 정렬할 부분 크기 (2의 거듭제곱)
    k = 2
    while k <= n:
        # j: 비교 간격
        j = k // 2
        while j >= 1:
            # 각 (i, j) 쌍을 병렬로 처리 (GPU에서는 각 스레드가 하나씩)
            for i in range(n):
                l = i ^ j  # XOR로 비교 상대 계산
                if l > i:
                    # i가 어떤 블록의 오름차순 부분인지 확인
                    if (i & k) == 0:
                        if arr[i] > arr[l]:
                            arr[i], arr[l] = arr[l], arr[i]
                    else:
                        if arr[i] < arr[l]:
                            arr[i], arr[l] = arr[l], arr[i]
            j //= 2
        k *= 2
    return arr

# 테스트
data = [4, 2, 7, 1, 9, 3, 8, 5, 6, 0, 11, 10, 14, 12, 15, 13]
result = gpu_bitonic_sort_simulation(data)
assert result == sorted(data)
print("GPU Bitonic Sort 검증 통과:", result)
```

## 주의사항과 팁

### 1. 반드시 2의 거듭제곱 크기

Bitonic Sort의 기본 구현은 원소 수가 2의 거듭제곱이어야 합니다. 그렇지 않은 경우 더미 원소(최대값)로 패딩하고 정렬 후 제거합니다. Odd-Even Merge Sort는 임의 크기를 지원합니다.

### 2. 캐시 지역성 고려

GPU에서 공유 메모리를 활용하지 않으면 글로벌 메모리 접근이 발산(divergent)될 수 있습니다. Bitonic Sort의 마지막 단계들은 strided 접근 패턴이 되어 캐시 미스가 증가합니다. 워프 단위로 공유 메모리에 로드 후 처리하는 것이 최적입니다.

### 3. 작은 크기에서의 실용성

n ≤ 32에서는 최적 정렬 네트워크가 Quicksort보다 빠릅니다. 실제로 많은 라이브러리에서 작은 배열은 하드코딩된 정렬 네트워크를 사용합니다. C++ STL의 `std::sort`는 내부적으로 작은 크기에 대해 삽입 정렬 또는 네트워크 정렬로 전환합니다.

### 4. SIMD와의 결합

x86 AVX2를 사용하면 256비트 레지스터로 8개의 32비트 정수를 동시에 비교할 수 있습니다. Bitonic Sort의 각 비교기를 `_mm256_min_epi32` / `_mm256_max_epi32` 명령어로 구현하면 단일 코어에서도 극도로 빠른 정렬이 가능합니다.

## 참고 자료

- [Sorting Networks - Wikipedia](https://en.wikipedia.org/wiki/Sorting_network)
- [Bitonic Sorter - Wikipedia](https://en.wikipedia.org/wiki/Bitonic_sorter)
- [Bitonic Sorting Network - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/bitonic-sorting-network-using-parallel-computing/)
- [Comparison of Parallel Sorting Algorithms (arXiv:1511.03404)](https://arxiv.org/abs/1511.03404)
