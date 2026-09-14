---
layout: post
title: "Succinct 자료구조 완전 정복: Rank/Select 연산으로 최소 공간에서 최대 성능 달성하기"
date: 2026-09-14
categories: [cs, computer-science]
tags: [succinct, rank-select, bitvector, wavelet-tree, data-structures, compression]
---

## 개념 설명: Succinct 자료구조란 무엇인가?

컴퓨터 과학에서 자료구조를 설계할 때 우리는 보통 두 가지 목표를 추구한다. 빠른 연산과 적은 메모리 사용. 그런데 이 두 목표는 종종 충돌한다. 더 빠른 연산을 위해 보조 인덱스를 추가하면 메모리가 늘어나고, 메모리를 아끼려면 압축을 하지만 연산이 느려진다.

**Succinct 자료구조(Succinct Data Structures)**는 이 딜레마를 근본적으로 해결하는 접근법이다. "정보 이론적 하한(information-theoretic lower bound)"에 근접한 공간을 사용하면서도, 원래 자료구조가 지원하는 연산을 효율적으로 수행할 수 있도록 설계된다.

구체적으로, n개의 원소로 이루어진 오브젝트를 표현하는 데 최소한 필요한 비트 수를 Z라 할 때, Succinct 자료구조는 Z + o(Z) 비트를 사용한다. 여기서 o(Z)는 Z보다 점근적으로 훨씬 작은 오버헤드를 의미한다. 압축(Compressed) 자료구조는 Z 비트 이하를 사용하지만 연산이 느리고, 일반 자료구조는 연산은 빠르지만 훨씬 많은 공간을 쓴다.

### Rank와 Select: 핵심 연산

Succinct 자료구조의 기초는 **비트 벡터(Bit Vector)**다. n비트 배열 B가 있을 때 두 가지 핵심 연산이 정의된다.

- **rank(i, c)**: B[0..i-1] 범위에서 비트 값 c(0 또는 1)의 개수
- **select(j, c)**: c 값의 j번째 등장 위치

예를 들어 B = `0110100110`이라면:
- rank(6, 1) = 3 (B[0..5]에서 1의 개수)
- select(2, 1) = 2 (두 번째 1이 인덱스 2에 위치)

이 두 연산이 O(1) 또는 O(log n)에 동작한다면, 이를 기반으로 트리, 문자열 인덱스, 그래프 등 다양한 자료구조를 매우 적은 공간으로 구현할 수 있다.

---

## 왜 Succinct 자료구조가 필요한가?

### 실제 데이터 규모의 문제

현대 시스템에서 다루는 데이터는 방대하다.

- **전체 텍스트 인덱스**: Wikipedia 전체(약 20GB)를 메모리에 올려 빠른 검색을 지원하려면?
- **DNA 서열**: 인간 게놈 하나가 약 3GB. 수백만 명의 게놈 데이터를 인덱싱하려면?
- **웹 그래프**: 수십억 개의 웹 페이지 링크 구조를 메모리에 표현하려면?

이런 규모에서 기존 자료구조는 한계에 부딪힌다. FM-Index 같은 Succinct 문자열 인덱스는 텍스트 자체보다 작은 공간에서 부분 문자열 검색을 O(m log n) 시간에 수행한다.

### 캐시 효율

Succinct 자료구조는 작은 공간에 데이터를 밀집시키기 때문에 CPU 캐시 적중률이 높아진다. 포인터 기반의 트리보다 비트 패킹된 표현이 L1/L2 캐시에 더 잘 들어맞는다.

---

## 실제 구현 예제

### 예제 1: Rank/Select 지원 비트 벡터 (Python)

```python
class BitVector:
    """O(1) rank와 O(log n) select를 지원하는 Succinct 비트 벡터"""
    
    BLOCK_SIZE = 64      # 슈퍼블록 크기
    SMALL_BLOCK = 8      # 소블록 크기
    
    def __init__(self, bits: list[int]):
        self.n = len(bits)
        self.bits = bits[:]
        self._build_index()
    
    def _build_index(self):
        n = self.n
        block = self.BLOCK_SIZE
        small = self.SMALL_BLOCK
        
        # 슈퍼블록: 각 블록 시작까지의 누적 rank
        self.super_blocks = []
        cumulative = 0
        for i in range(0, n, block):
            self.super_blocks.append(cumulative)
            cumulative += sum(self.bits[i:i+block])
        
        # 소블록: 슈퍼블록 내에서의 상대적 rank
        self.small_blocks = []
        for sb_start in range(0, n, block):
            relative = 0
            for j in range(sb_start, min(sb_start + block, n), small):
                self.small_blocks.append(relative)
                relative += sum(self.bits[j:j+small])
        
        # 팝카운트 룩업 테이블 (8비트)
        self.popcount = [bin(i).count('1') for i in range(256)]
    
    def rank1(self, i: int) -> int:
        """B[0..i-1]에서 1의 개수 (O(1))"""
        if i <= 0:
            return 0
        i = min(i, self.n)
        
        sb_idx = (i - 1) // self.BLOCK_SIZE
        sb_count = self.super_blocks[sb_idx]
        
        # 소블록 내 위치
        sb_start = sb_idx * self.BLOCK_SIZE
        small_idx = (i - 1 - sb_start) // self.SMALL_BLOCK
        
        # 소블록 인덱스 계산
        global_small_idx = sb_idx * (self.BLOCK_SIZE // self.SMALL_BLOCK) + small_idx
        small_count = self.small_blocks[global_small_idx]
        
        # 나머지 비트 직접 계산
        start = sb_start + small_idx * self.SMALL_BLOCK
        remainder = sum(self.bits[start:i])
        
        return sb_count + small_count + remainder
    
    def rank0(self, i: int) -> int:
        """B[0..i-1]에서 0의 개수"""
        return i - self.rank1(i)
    
    def select1(self, j: int) -> int:
        """j번째 1의 위치 (1-indexed, O(log n))"""
        lo, hi = 0, self.n
        while lo < hi:
            mid = (lo + hi) // 2
            if self.rank1(mid) >= j:
                hi = mid
            else:
                lo = mid + 1
        return lo - 1  # 0-indexed

# 사용 예시
bits = [0, 1, 1, 0, 1, 0, 0, 1, 1, 0]
bv = BitVector(bits)

print(f"비트 벡터: {bits}")
print(f"rank1(6) = {bv.rank1(6)}")   # B[0..5]에서 1의 수
print(f"rank1(9) = {bv.rank1(9)}")   # B[0..8]에서 1의 수
print(f"select1(2) = {bv.select1(2)}")  # 두 번째 1의 위치
print(f"select1(4) = {bv.select1(4)}")  # 네 번째 1의 위치
```

출력:
```
비트 벡터: [0, 1, 1, 0, 1, 0, 0, 1, 1, 0]
rank1(6) = 3
rank1(9) = 4
select1(2) = 2
select1(4) = 7
```

### 예제 2: Wavelet Tree — 임의 알파벳에서의 Rank/Select (C++)

Wavelet Tree는 비트 벡터의 rank/select를 임의의 알파벳으로 확장한 자료구조다. 알파벳 σ에 대해 O(n log σ) 비트를 사용하며, 임의 문자에 대한 rank/select를 O(log σ) 시간에 처리한다.

```cpp
#include <bits/stdc++.h>
using namespace std;

struct WaveletTree {
    int lo, hi;                          // 알파벳 범위 [lo, hi]
    WaveletTree *left = nullptr, *right = nullptr;
    vector<int> b;                       // 왼쪽으로 간 원소들의 누적 합 (rank용)
    
    // [l, r) 범위의 arr[lo..hi] 알파벳으로 Wavelet Tree 구축
    void build(int* from, int* to, int lo, int hi) {
        this->lo = lo; this->hi = hi;
        if (lo == hi || from >= to) return;
        
        int mid = (lo + hi) / 2;
        
        // 각 위치에서 왼쪽(<=mid)으로 간 원소 수를 누적
        b.reserve(to - from + 1);
        b.push_back(0);
        for (auto it = from; it != to; ++it)
            b.push_back(b.back() + (*it <= mid ? 1 : 0));
        
        // 안정 파티션: <=mid는 왼쪽, >mid는 오른쪽
        auto pivot = stable_partition(from, to, [mid](int x){ return x <= mid; });
        
        left  = new WaveletTree(); left->build(from, pivot, lo, mid);
        right = new WaveletTree(); right->build(pivot, to, mid+1, hi);
    }
    
    // [l, r) 범위에서 c의 등장 횟수
    int rank(int l, int r, int c) {
        if (lo == hi) return r - l;
        int mid = (lo + hi) / 2;
        if (c <= mid) {
            return left->rank(b[l], b[r], c);
        } else {
            int lb = l - b[l], rb = r - b[r];  // 오른쪽 자식 인덱스
            return right->rank(lb, rb, c);
        }
    }
    
    // [l, r) 범위에서 k번째로 작은 원소 (1-indexed)
    int kth(int l, int r, int k) {
        if (lo == hi) return lo;
        int mid = (lo + hi) / 2;
        int lb = b[r] - b[l];  // 왼쪽으로 간 원소 수
        if (k <= lb) {
            return left->kth(b[l], b[r], k);
        } else {
            return right->kth(l - b[l], r - b[r], k - lb);
        }
    }
};

int main() {
    // 예시: 문자열 "banana"를 정수 배열로
    // a=0, b=1, n=2
    int arr[] = {1, 0, 2, 0, 2, 0};  // b a n a n a
    int n = 6;
    
    WaveletTree* wt = new WaveletTree();
    int tmp[6];
    copy(arr, arr + n, tmp);
    wt->build(tmp, tmp + n, 0, 2);
    
    // "banana"[0..5]에서 'a'(=0)의 rank
    cout << "rank of 'a' in [0,6): " << wt->rank(0, 6, 0) << endl; // 3
    
    // "banana"[0..5]에서 'n'(=2)의 rank
    cout << "rank of 'n' in [0,6): " << wt->rank(0, 6, 2) << endl; // 2
    
    // [0..5]에서 3번째로 작은 원소
    cout << "3rd smallest in [0,6): " << wt->kth(0, 6, 3) << endl;  // 0 ('a')
    
    // [1..4]에서 2번째로 작은 원소
    cout << "2nd smallest in [1,4): " << wt->kth(1, 4, 2) << endl;  // 2 ('n')
    
    return 0;
}
```

Wavelet Tree는 구간 내 k번째 원소, 구간 rank/select, 구간 중앙값 등 다양한 쿼리를 모두 O(log σ)에 처리할 수 있어 FM-Index, BWT, CSA(Compressed Suffix Array) 등의 구현에 핵심적으로 사용된다.

---

## 더 깊은 활용: FM-Index와 전체 텍스트 검색

Succinct 자료구조의 가장 강력한 응용 중 하나는 **FM-Index**다. 텍스트 T의 BWT(Burrows-Wheeler Transform)를 Wavelet Tree로 압축하면, 텍스트 자체보다 작은 공간에서 패턴 검색을 O(m log σ) 시간에 수행할 수 있다.

핵심 아이디어는 **LF Mapping**이다. BWT 배열 L에서의 rank 연산으로 SA(Suffix Array) 탐색을 수행하며, 메모리에 텍스트를 완전히 올릴 필요 없이 압축된 상태에서 직접 탐색한다.

```
패턴 검색 의사 코드 (FM-Index backward search):

function search(pattern P, FM-Index F):
    lo, hi = 0, n
    for i from len(P)-1 to 0:
        c = P[i]
        lo = C[c] + rank(BWT, lo, c)
        hi = C[c] + rank(BWT, hi, c)
        if lo >= hi: return 0  # 패턴 없음
    return hi - lo  # 등장 횟수
```

여기서 `rank(BWT, i, c)`는 BWT[0..i-1]에서 문자 c의 등장 수이며, Wavelet Tree로 O(log σ)에 계산한다.

---

## 주의사항과 팁

### 1. 실용적 구현의 복잡성

이론적으로는 O(1) rank가 가능하지만, 실제 구현에서는 슈퍼블록-소블록-룩업 테이블의 3단계 구조가 필요하며 캐시 적중률을 고려한 블록 크기 선택이 중요하다. 블록 크기가 너무 작으면 오버헤드가 크고, 너무 크면 캐시 미스가 증가한다. 일반적으로 64비트 워드 크기에 맞추는 것이 효율적이다.

### 2. SDSL-lite 라이브러리 활용

직접 구현보다 **SDSL-lite(Succinct Data Structure Library)**를 활용하는 것이 실무에서 더 현실적이다. 40여 편의 논문을 구현한 C++ 라이브러리로, `sdsl::bit_vector`, `sdsl::rank_support_v`, `sdsl::select_support_mcl` 등을 바로 사용할 수 있다.

```cpp
#include <sdsl/bit_vectors.hpp>
using namespace sdsl;

bit_vector bv(10, 0);
bv[1] = bv[2] = bv[4] = bv[7] = bv[8] = 1;

rank_support_v<1> rs(&bv);
select_support_mcl<1> ss(&bv);

cout << rs(6) << endl;   // rank1(6) = 3
cout << ss(2) << endl;   // select1(2) = 2 (0-indexed)
```

### 3. 공간과 속도 트레이드오프 이해

| 자료구조 | 공간 | rank/select 속도 |
|----------|------|-----------------|
| 단순 배열 | n 비트 | O(n) |
| 포인터 트리 | O(n log n) 비트 | O(log n) |
| Jacobson bitvector | n + O(n log log n / log n) | O(1) |
| Wavelet Tree | n log σ + o(n log σ) | O(log σ) |

실제 사용 시 σ가 작은 경우(DNA: σ=4, ASCII: σ=128)에는 Wavelet Tree가 매우 효율적이다.

### 4. 동적 vs 정적

기본적인 Succinct 자료구조는 **정적(Static)**이다. 즉, 구축 후 수정이 어렵다. 동적 업데이트가 필요한 경우 Dynamic Bit Vector 연구(Makinen & Navarro의 Dynamic Compressed Text)를 참고하되, 구현 복잡도가 크게 증가함을 인지해야 한다.

---

## 정리

Succinct 자료구조는 빅데이터 시대의 핵심 기술이다. 정보 이론의 한계에 근접한 공간을 사용하면서도 효율적인 쿼리를 지원함으로써, 메모리의 벽을 넘어서는 새로운 가능성을 열어준다. FM-Index를 통한 전체 텍스트 검색, 압축 접미사 배열, Succinct 트리 등은 모두 rank/select라는 단순한 원시 연산 위에 쌓아 올린 정교한 건축물이다. 메모리가 병목인 대규모 데이터 처리를 다루는 엔지니어라면 반드시 이해해야 할 분야다.

## 참고 자료
- [GitHub - simongog/sdsl-lite: Succinct Data Structure Library 2.0](https://github.com/simongog/sdsl-lite)
- [GitHub - eopXD/SuccinctDS: Succinct Wavelet Tree C++ 구현](https://github.com/eopXD/SuccinctDS)
- [GitHub - nectarios-ef/Wavelet-Tree: Python Wavelet Tree 구현](https://github.com/nectarios-ef/Wavelet-Tree)
