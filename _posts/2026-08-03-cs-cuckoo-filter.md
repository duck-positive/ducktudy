---
layout: post
title: "Cuckoo Filter 완전 정복: 삭제를 지원하는 차세대 확률적 멤버십 필터"
date: 2026-08-03
categories: [cs, computer-science]
tags: [cuckoo-filter, bloom-filter, probabilistic, data-structure, hashing, membership-test]
---

## 개요

"이 사용자 ID가 이미 존재하는가?", "이 URL을 이미 크롤링했는가?", "이 IP 주소가 블랙리스트에 있는가?" — 수십억 개의 원소로 구성된 집합에서 이런 멤버십 질의(Membership Query)를 수행하는 것은 메모리 효율적인 자료구조를 요구한다.

**블룸 필터(Bloom Filter)**는 이 문제를 해결한 확률적 자료구조의 고전이지만, 한 가지 치명적인 약점이 있다: **원소를 삭제할 수 없다**. 2014년 Fan, Andersen, Kaminsky, Mitzenmacher가 발표한 **"Cuckoo Filter: Practically Better Than Bloom"** 논문은 이 문제를 해결하면서 공간 효율성까지 향상시킨 새로운 자료구조를 제안했다.

---

## 블룸 필터의 한계

블룸 필터는 k개의 해시 함수와 비트 배열을 사용한다.
- **삽입**: k개의 해시 함수로 위치를 계산하고 비트를 1로 설정
- **조회**: k개의 위치가 모두 1이면 "있을 수 있음(Possibly Present)", 하나라도 0이면 "없음(Definitely Absent)"
- **삭제 불가**: 비트를 0으로 되돌리면 해당 위치를 공유하는 다른 원소도 삭제되어 버림

삭제를 지원하는 **카운팅 블룸 필터(Counting Bloom Filter)**는 각 비트 대신 카운터를 사용하지만, 메모리가 3~4배 증가한다.

---

## Cuckoo Filter의 핵심 아이디어

Cuckoo Filter는 이름처럼 **뻐꾸기 해싱(Cuckoo Hashing)**에서 영감을 받았다. 뻐꾸기 해싱에서 새 원소를 삽입할 때 기존 원소를 다른 위치로 "쫓아내는" 것처럼, Cuckoo Filter도 같은 원리를 사용한다.

### 핵심 구성 요소

1. **핑거프린트(Fingerprint)**: 원소의 해시값 일부 (예: 8비트 또는 16비트)
2. **버킷 배열**: 각 버킷은 b개의 핑거프린트를 저장할 수 있는 슬롯 집합
3. **2개의 후보 위치**: 각 원소는 두 버킷 중 하나에 저장됨

### 두 위치 계산 방법

```
f = fingerprint(x)
i₁ = hash(x) mod m
i₂ = i₁ XOR hash(f) mod m
```

이 공식의 핵심은 **대칭성(Symmetry)**이다: `i₁ = i₂ XOR hash(f)` 도 성립한다. 따라서 핑거프린트만 알고 있으면 현재 위치에서 다른 위치를 계산할 수 있다. 원본 원소를 저장하지 않아도 된다!

---

## 삽입, 조회, 삭제 알고리즘

### 삽입(Insert)

```python
def insert(self, item):
    f = self.fingerprint(item)
    i1 = self.hash(item) % self.num_buckets
    i2 = (i1 ^ self.hash(f)) % self.num_buckets

    # 두 후보 버킷 중 빈 슬롯에 삽입
    if self.buckets[i1].has_empty_slot():
        self.buckets[i1].add(f)
        return True
    if self.buckets[i2].has_empty_slot():
        self.buckets[i2].add(f)
        return True

    # 뻐꾸기 이동: 기존 원소를 쫓아내고 삽입
    i = random.choice([i1, i2])
    for _ in range(self.max_kicks):
        # 현재 버킷에서 랜덤하게 핑거프린트를 꺼냄
        evicted_f = self.buckets[i].remove_random()
        self.buckets[i].add(f)
        f = evicted_f
        # 쫓겨난 핑거프린트의 다른 후보 버킷으로 이동
        i = (i ^ self.hash(f)) % self.num_buckets
        if self.buckets[i].has_empty_slot():
            self.buckets[i].add(f)
            return True

    # max_kicks 초과 → 필터가 가득 참 (삽입 실패)
    return False
```

### 조회(Lookup)

```python
def lookup(self, item):
    f = self.fingerprint(item)
    i1 = self.hash(item) % self.num_buckets
    i2 = (i1 ^ self.hash(f)) % self.num_buckets

    return f in self.buckets[i1] or f in self.buckets[i2]
```

### 삭제(Delete)

```python
def delete(self, item):
    f = self.fingerprint(item)
    i1 = self.hash(item) % self.num_buckets
    i2 = (i1 ^ self.hash(f)) % self.num_buckets

    if f in self.buckets[i1]:
        self.buckets[i1].remove(f)
        return True
    if f in self.buckets[i2]:
        self.buckets[i2].remove(f)
        return True
    return False  # 원소가 없음
```

삭제는 단순히 핑거프린트를 제거하면 된다. 블룸 필터의 비트 공유 문제가 없기 때문이다.

---

## 완전한 Python 구현

```python
import hashlib
import random
from typing import Optional

class Bucket:
    def __init__(self, size: int):
        self.size = size
        self.slots = [None] * size

    def has_empty_slot(self) -> bool:
        return None in self.slots

    def add(self, fp: int) -> bool:
        for i, slot in enumerate(self.slots):
            if slot is None:
                self.slots[i] = fp
                return True
        return False

    def remove(self, fp: int) -> bool:
        for i, slot in enumerate(self.slots):
            if slot == fp:
                self.slots[i] = None
                return True
        return False

    def remove_random(self) -> Optional[int]:
        occupied = [i for i, s in enumerate(self.slots) if s is not None]
        if not occupied:
            return None
        idx = random.choice(occupied)
        fp = self.slots[idx]
        self.slots[idx] = None
        return fp

    def __contains__(self, fp: int) -> bool:
        return fp in self.slots


class CuckooFilter:
    def __init__(self, capacity: int, fp_size: int = 8, bucket_size: int = 4,
                 max_kicks: int = 500):
        self.capacity = capacity
        self.fp_size = fp_size           # 비트 단위 핑거프린트 크기
        self.fp_mask = (1 << fp_size) - 1
        self.bucket_size = bucket_size   # 버킷당 슬롯 수
        self.max_kicks = max_kicks
        self.num_buckets = capacity // bucket_size
        self.buckets = [Bucket(bucket_size) for _ in range(self.num_buckets)]
        self.count = 0

    def _hash(self, data) -> int:
        if isinstance(data, int):
            data = data.to_bytes(4, 'big')
        elif isinstance(data, str):
            data = data.encode()
        return int(hashlib.sha256(data).hexdigest(), 16)

    def _fingerprint(self, item) -> int:
        fp = self._hash(item) & self.fp_mask
        return fp if fp != 0 else 1  # 핑거프린트는 0이 아니어야 함

    def insert(self, item) -> bool:
        f = self._fingerprint(item)
        i1 = self._hash(item) % self.num_buckets
        i2 = (i1 ^ self._hash(f)) % self.num_buckets

        if self.buckets[i1].add(f):
            self.count += 1
            return True
        if self.buckets[i2].add(f):
            self.count += 1
            return True

        i = random.choice([i1, i2])
        for _ in range(self.max_kicks):
            evicted_f = self.buckets[i].remove_random()
            self.buckets[i].add(f)
            f = evicted_f
            i = (i ^ self._hash(f)) % self.num_buckets
            if self.buckets[i].add(f):
                self.count += 1
                return True

        return False  # 삽입 실패

    def lookup(self, item) -> bool:
        f = self._fingerprint(item)
        i1 = self._hash(item) % self.num_buckets
        i2 = (i1 ^ self._hash(f)) % self.num_buckets
        return f in self.buckets[i1] or f in self.buckets[i2]

    def delete(self, item) -> bool:
        f = self._fingerprint(item)
        i1 = self._hash(item) % self.num_buckets
        i2 = (i1 ^ self._hash(f)) % self.num_buckets

        if self.buckets[i1].remove(f):
            self.count -= 1
            return True
        if self.buckets[i2].remove(f):
            self.count -= 1
            return True
        return False

    @property
    def load_factor(self) -> float:
        return self.count / (self.num_buckets * self.bucket_size)


# 사용 예시
if __name__ == "__main__":
    cf = CuckooFilter(capacity=1000, fp_size=8, bucket_size=4)

    # 삽입
    items = [f"user_{i}" for i in range(800)]
    for item in items:
        cf.insert(item)

    print(f"부하율: {cf.load_factor:.2%}")  # 약 80%

    # 조회
    print(cf.lookup("user_1"))    # True
    print(cf.lookup("user_999"))  # False (삽입 안 됨) 또는 False Positive

    # 삭제 후 조회
    cf.delete("user_1")
    print(cf.lookup("user_1"))    # False (블룸 필터와 달리 삭제 가능!)

    # 거짓 양성률 측정
    false_positives = sum(1 for i in range(1000, 2000) if cf.lookup(f"user_{i}"))
    print(f"거짓 양성률: {false_positives / 1000:.2%}")
```

---

## 성능 분석: Bloom Filter vs Cuckoo Filter

### 공간 효율성

핑거프린트 크기 f 비트, 버킷 크기 b = 4일 때 Cuckoo Filter의 비트 비용:

```
bits/item = f / load_factor ≈ f / 0.955 ≈ 1.05 × f
```

블룸 필터의 최적 비트 비용:
```
bits/item = -log₂(ε) / ln(2) ≈ 1.44 × log₂(1/ε)
```

- **거짓 양성률 ε = 1%**: 블룸 필터 ≈ 9.6 bits/item, Cuckoo Filter (8-bit fp) ≈ 8.4 bits/item
- **거짓 양성률 ε = 0.1%**: 블룸 필터 ≈ 14.4 bits/item, Cuckoo Filter (12-bit fp) ≈ 12.6 bits/item

Cuckoo Filter가 약 **10~30% 더 공간 효율적**이다.

### 시간 복잡도

| 연산 | 블룸 필터 | Cuckoo Filter |
|------|-----------|---------------|
| 삽입 | O(k) | O(1) 평균 |
| 조회 | O(k) | O(1) |
| 삭제 | 불가 | O(1) |

### 최대 부하율

버킷 크기 b = 4일 때 Cuckoo Filter는 최대 **95.5%**의 부하율을 달성하며, 이 이상에서는 삽입이 실패하기 시작한다.

---

## 실제 사용 사례

### 1. Apache Kafka / Apache Cassandra
Cassandra의 SSTable에서 존재하지 않는 키 조회를 빠르게 필터링하는 데 활용한다. 블룸 필터와 달리 Cuckoo Filter는 키 만료 시 삭제가 가능하다.

### 2. DNS/방화벽 IP 차단
악성 IP 목록은 계속 업데이트된다. 블룸 필터는 새 IP를 추가할 수 있지만 차단 해제는 불가능하다. Cuckoo Filter는 IP 추가/제거 모두 O(1)에 지원한다.

### 3. CDN 중복 요청 필터링
동일 URL의 반복 요청을 감지하되, 캐시 만료 후에는 새 항목으로 처리해야 한다. Cuckoo Filter의 삭제 기능이 이 시나리오에 적합하다.

---

## 주의사항 및 한계

1. **동일 원소의 중복 삽입**: 같은 원소를 2번 이상 삽입하면 같은 핑거프린트가 여러 버킷에 저장된다. 한 번 삭제해도 나머지가 남아 false positive를 유발한다. 중복 삽입 횟수를 추적하거나 외부에서 관리해야 한다.

2. **핑거프린트 충돌**: 서로 다른 두 원소가 같은 핑거프린트를 가질 수 있다. fp_size를 늘리면 false positive 확률이 줄지만 공간이 증가한다.

3. **삽입 실패**: 부하율이 95% 이상이면 뻐꾸기 사이클(Cuckoo Cycle)이 발생해 삽입이 실패할 수 있다. 필터를 미리 적절한 크기로 설계해야 한다.

4. **안정적인 삭제 보장**: 삽입된 적 없는 원소를 삭제하면 false negative가 생길 수 있다. 반드시 삽입 여부를 확인하고 삭제해야 한다.

---

## 참고 자료

- [Cuckoo Filter: Practically Better Than Bloom - Fan et al., CMU (PDF)](https://www.cs.cmu.edu/~dga/papers/cuckoo-conext2014.pdf)
- [Cuckoo Filter - Brilliant.org](https://brilliant.org/wiki/cuckoo-filter/)
- [Cuckoo Hashing Wikipedia](https://en.wikipedia.org/wiki/Cuckoo_hashing)
- [libcuckoo - C++ High-Performance Cuckoo Hash Table](https://github.com/efficient/libcuckoo)
