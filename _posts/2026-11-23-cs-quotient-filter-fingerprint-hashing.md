---
layout: post
title: "쿼션트 필터(Quotient Filter) 완전 정복: 삭제 가능한 확률적 멤버십 테스트 자료구조"
date: 2026-11-23
categories: [cs, computer-science]
tags: [quotient-filter, bloom-filter, probabilistic, data-structures, hashing, fingerprint]
---

## 개념 설명

쿼션트 필터(Quotient Filter, QF)는 2011년 Michael Bender 등이 발표한 확률적 멤버십 테스트 자료구조입니다. 블룸 필터(Bloom Filter)와 마찬가지로 "원소가 집합에 존재하는가?"를 빠르게 답하되, 블룸 필터가 해결하지 못한 **삭제(Deletion)** 를 지원합니다.

### 핵심 아이디어: 지문(Fingerprint) 분해

원소 `x`를 해시하여 `p`비트 지문(fingerprint) `f`를 얻은 뒤, 이를 두 부분으로 나눕니다.

```
f = [  quotient q (상위 q비트)  |  remainder r (하위 r비트)  ]
```

- **Quotient `q`**: 슬롯 테이블에서 원소가 저장될 **캐노니컬 슬롯(canonical slot)**의 인덱스. `q`비트이므로 테이블 크기는 `2^q`개 슬롯.
- **Remainder `r`**: 실제로 슬롯에 저장되는 값. 같은 캐노니컬 슬롯을 공유하는 원소들끼리의 구분자.

### 슬롯 메타데이터: 3개 비트

각 슬롯은 `r`비트 나머지 값 외에 3개의 메타데이터 비트를 가집니다.

| 비트 | 이름 | 의미 |
|------|------|------|
| `is_occupied` | 점유 여부 | 이 슬롯이 어떤 원소의 캐노니컬 슬롯인가 |
| `is_continuation` | 연속 여부 | 이 슬롯의 값이 이전 슬롯과 같은 `run`에 속하는가 |
| `is_shifted` | 이동 여부 | 이 슬롯의 값이 본래 캐노니컬 슬롯에서 오른쪽으로 밀려났는가 |

**Run**: 같은 캐노니컬 슬롯을 공유하는 나머지 값들의 연속된 묶음.  
**Cluster**: 하나 이상의 Run이 연속해서 이어진 구역.

이 세 비트를 활용하면 슬롯을 뒤섞어도 어떤 나머지가 어느 캐노니컬 슬롯에서 왔는지 역추적할 수 있습니다.

---

## 왜 필요한가

### 블룸 필터의 한계

블룸 필터는 각 원소를 `k`개의 해시 함수로 비트 배열에 표시합니다. 조회는 `O(k)`, 삽입도 `O(k)`로 빠르지만 **삭제를 지원하지 않습니다**. 비트가 여러 원소에 의해 공유될 수 있어, 비트를 지우면 다른 원소에 대한 정보도 훼손됩니다.

**카운팅 블룸 필터(Counting Bloom Filter)** 는 카운터를 써서 삭제를 지원하지만, 각 비트 대신 4~8비트 카운터가 필요해 메모리가 4~8배 증가합니다.

### 쿼션트 필터의 장점

1. **삭제 지원**: 나머지를 슬롯에서 제거하고 오른쪽 원소를 왼쪽으로 당기면 됩니다.
2. **캐시 친화적**: 연속 메모리 블록 사용. 블룸 필터는 무작위 메모리 접근이 많아 캐시 미스가 빈번합니다.
3. **머징 가능**: 두 개의 쿼션트 필터를 하나로 합칠 수 있습니다 (블룸 필터는 불가).
4. **공간 효율**: false positive rate 1%에서 블룸 필터와 비슷한 메모리, 하지만 삭제와 머징까지 가능.

---

## 실제 구현 예제

### 예제 1: Python으로 구현하는 간단한 쿼션트 필터

```python
import math
import hashlib

class QuotientFilter:
    def __init__(self, capacity: int, error_rate: float = 0.01):
        # 필요한 비트 수 계산
        self.p = math.ceil(math.log2(capacity / error_rate))
        self.r = math.ceil(math.log2(1 / error_rate))
        self.q = self.p - self.r
        self.size = 1 << self.q  # 2^q 슬롯

        # 슬롯 배열: 각 원소는 (is_occupied, is_continuation, is_shifted, remainder)
        self.slots = [None] * self.size
        self.is_occupied = [False] * self.size
        self.is_continuation = [False] * self.size
        self.is_shifted = [False] * self.size

    def _fingerprint(self, item: str) -> int:
        h = int(hashlib.sha256(item.encode()).hexdigest(), 16)
        return h & ((1 << self.p) - 1)  # 하위 p비트만 사용

    def _quotient(self, fp: int) -> int:
        return fp >> self.r  # 상위 q비트

    def _remainder(self, fp: int) -> int:
        return fp & ((1 << self.r) - 1)  # 하위 r비트

    def _find_run_start(self, canonical: int) -> int:
        """canonical slot의 run이 시작하는 실제 슬롯 위치 반환"""
        # occupied 비트가 세팅된 슬롯 수를 세어 run 시작 위치 추정
        b = canonical
        while self.is_shifted[b]:
            b = (b - 1) % self.size
        
        s = b
        while b != canonical:
            # continuation이 아닌 슬롯(=새로운 run 시작)을 세면서 앞으로 이동
            while self.is_continuation[s]:
                s = (s + 1) % self.size
            b += 1
            while not self.is_occupied[b % self.size]:
                b += 1
            b = b % self.size
        return s

    def insert(self, item: str) -> None:
        fp = self._fingerprint(item)
        q = self._quotient(fp)
        r = self._remainder(fp)

        if self.slots[q] is None:
            # 빈 슬롯에 직접 삽입
            self.slots[q] = r
            self.is_occupied[q] = True
            return

        self.is_occupied[q] = True
        s = self._find_run_start(q)

        # run 내 적절한 위치 찾기 (정렬 삽입)
        while self.slots[s] is not None and self.slots[s] < r:
            if not self.is_continuation[(s + 1) % self.size]:
                break
            s = (s + 1) % self.size

        # 나머지 원소들을 오른쪽으로 한 칸씩 밀기
        insert_pos = (s + 1) % self.size
        self._shift_right(insert_pos, r)

    def _shift_right(self, pos: int, value: int) -> None:
        curr_val = value
        curr_cont = True
        curr_shift = True
        while curr_val is not None:
            prev_val = self.slots[pos]
            prev_cont = self.is_continuation[pos]
            prev_shift = self.is_shifted[pos]
            self.slots[pos] = curr_val
            self.is_continuation[pos] = curr_cont
            self.is_shifted[pos] = curr_shift
            curr_val = prev_val
            curr_cont = prev_cont
            curr_shift = prev_shift
            pos = (pos + 1) % self.size

    def lookup(self, item: str) -> bool:
        fp = self._fingerprint(item)
        q = self._quotient(fp)
        r = self._remainder(fp)

        if not self.is_occupied[q]:
            return False

        s = self._find_run_start(q)
        # run 내에서 remainder 탐색
        while True:
            if self.slots[s] == r:
                return True
            if not self.is_continuation[(s + 1) % self.size]:
                break
            s = (s + 1) % self.size
        return False


# 사용 예시
if __name__ == "__main__":
    qf = QuotientFilter(capacity=1000, error_rate=0.01)
    
    words = ["apple", "banana", "cherry", "date", "elderberry"]
    for w in words:
        qf.insert(w)
    
    for w in words:
        print(f"lookup('{w}'): {qf.lookup(w)}")  # True
    
    print(f"lookup('fig'): {qf.lookup('fig')}")     # 높은 확률로 False
    print(f"lookup('grape'): {qf.lookup('grape')}") # 높은 확률로 False
```

### 예제 2: Counting Quotient Filter (CQF)로 중복 원소 카운팅

Counting Quotient Filter는 나머지(remainder) 슬롯을 연장하여 삽입 횟수를 함께 저장합니다. 이를 통해 멀티셋(multiset)의 멤버십 테스트와 근사 카운팅을 동시에 지원합니다.

```python
from collections import defaultdict

class SimpleCountingQF:
    """
    CQF의 단순화 버전: 해시맵으로 (q, r) → count 매핑을 유지.
    실제 CQF는 단일 연속 배열을 사용하지만, 개념 이해를 위한 구현.
    """
    def __init__(self, q_bits: int = 10, r_bits: int = 7):
        self.q_bits = q_bits
        self.r_bits = r_bits
        self.p = q_bits + r_bits
        self.table: dict = defaultdict(int)  # (q, r) -> count

    def _fingerprint(self, item: str) -> tuple[int, int]:
        h = int(hashlib.sha256(item.encode()).hexdigest(), 16)
        fp = h & ((1 << self.p) - 1)
        q = fp >> self.r_bits
        r = fp & ((1 << self.r_bits) - 1)
        return q, r

    def insert(self, item: str, count: int = 1) -> None:
        key = self._fingerprint(item)
        self.table[key] += count

    def delete(self, item: str, count: int = 1) -> bool:
        key = self._fingerprint(item)
        if self.table[key] == 0:
            return False
        self.table[key] = max(0, self.table[key] - count)
        return True

    def count(self, item: str) -> int:
        key = self._fingerprint(item)
        return self.table[key]

    def lookup(self, item: str) -> bool:
        return self.count(item) > 0


# DNA k-mer 카운팅 예제 (바이오인포매틱스의 실제 활용 사례)
def count_kmers(sequence: str, k: int = 21) -> SimpleCountingQF:
    cqf = SimpleCountingQF(q_bits=20, r_bits=10)
    for i in range(len(sequence) - k + 1):
        kmer = sequence[i:i+k]
        cqf.insert(kmer)
    return cqf


if __name__ == "__main__":
    dna = "ATCGATCGATCGATCGATCG" * 100
    cqf = count_kmers(dna, k=10)
    
    test_kmer = "ATCGATCGAT"
    print(f"k-mer '{test_kmer}' 등장 횟수 (근사): {cqf.count(test_kmer)}")
    
    # 삭제 테스트
    cqf.delete(test_kmer, count=5)
    print(f"5회 삭제 후 카운트: {cqf.count(test_kmer)}")
    
    print(f"존재하지 않는 k-mer 카운트: {cqf.count('ZZZZZZZZZZ')}")
```

---

## 주의사항과 팁

### 1. 슬롯 로드 팩터(Load Factor)

쿼션트 필터의 false positive율은 로드 팩터 `α = n / 2^q`에 크게 영향을 받습니다.

| 로드 팩터 | 성능 |
|-----------|------|
| α ≤ 50% | 쿼리당 평균 1.5 슬롯 접근 |
| α ≤ 75% | 쿼리당 평균 3 슬롯 접근 |
| α ≥ 90% | 성능 급격히 저하, 클러스터가 길어짐 |

**권장**: 로드 팩터를 70~75% 이하로 유지하세요. 블룸 필터처럼 100%에 가깝게 채우면 삽입/조회 성능이 선형에 가까워집니다.

### 2. `q`와 `r` 비트 선택

- 원하는 false positive rate `ε`와 최대 원소 수 `n`이 있을 때:
  - `r = ceil(log2(1/ε))` → FP rate 제어
  - `q = ceil(log2(n)) + 여유 비트` → 충분한 슬롯 확보

예) n=100만, ε=1% → r=7, q=22 → 슬롯 수 4M → 총 메모리 ≈ 4M × (7+3) bit = 5MB

### 3. 블룸 필터와 비교

| 항목 | 블룸 필터 | 쿼션트 필터 |
|------|-----------|-------------|
| 삭제 지원 | ✗ | ✓ |
| 캐시 효율 | 낮음 (무작위 접근) | 높음 (연속 메모리) |
| 머징 | ✗ | ✓ |
| 구현 복잡도 | 낮음 | 높음 |
| 메모리 (1% FP) | ~9.6 bits/element | ~10 bits/element |

### 4. CQF(Counting Quotient Filter) 활용

실제 프로덕션에서는 `splatlab/cqf`(GitHub)처럼 SIMD 명령어를 활용한 구현을 사용하는 것이 좋습니다. 특히 DNA 서열 분석, 네트워크 트래픽 중복 탐지, 대규모 중복 URL 필터링에서 많이 활용됩니다.

---

## 참고 자료

- [splatlab/cqf: A General-Purpose Counting Filter](https://github.com/splatlab/cqf)
- [vedantk/quotient-filter: a quotient filter written in C](https://github.com/vedantk/quotient-filter)
