---
layout: post
title: "Bloom Filter 완전 정복: 확률적 멤버십 테스트로 메모리를 극단적으로 절약하는 법"
date: 2026-11-22
categories: [cs, computer-science]
tags: [bloom-filter, data-structure, algorithm, probabilistic, cuckoo-filter, counting-bloom-filter]
---

## 개요

분산 시스템과 대규모 데이터 처리에서 "이 원소가 집합에 존재하는가?"라는 질문은 끊임없이 등장합니다. 단순하게 해시셋(HashSet)을 쓰면 정확하지만, 수억 개의 URL을 메모리에 올릴 수는 없습니다. 이때 등장하는 것이 **Bloom Filter**입니다. Bloom Filter는 1970년 Burton Howard Bloom이 제안한 확률적 자료구조(probabilistic data structure)로, 아주 적은 메모리로 멤버십 테스트를 수행합니다. 단, **거짓 양성(false positive)**이 발생할 수 있고 **삭제가 불가능**하다는 트레이드오프가 존재합니다.

이 글에서는 Bloom Filter의 수학적 원리, Python 구현, 그리고 Counting Bloom Filter와 Cuckoo Filter까지 확장하여 살펴봅니다.

---

## 왜 Bloom Filter가 필요한가?

### 실제 사용 사례

- **웹 크롤러 중복 URL 필터링**: Google, Bing 같은 크롤러는 이미 방문한 URL을 수십억 개 추적합니다. HashSet은 URL 하나당 수십 바이트가 필요하지만, Bloom Filter는 원소 하나당 약 10비트 미만으로 처리할 수 있습니다.
- **데이터베이스 캐시 미스 방지**: HBase, Cassandra, RocksDB는 디스크 I/O를 줄이기 위해 Bloom Filter로 존재하지 않는 키 조회를 걸러냅니다.
- **패스워드 블랙리스트**: 유출된 패스워드 수억 개를 메모리에 올려 실시간으로 차단합니다.
- **네트워크 라우터**: 패킷이 특정 집합에 속하는지를 나노초 단위로 판별합니다.

### 메모리 효율의 이점

원소 100만 개를 1% 오류율로 저장할 경우:
- **HashSet**: 약 32MB (문자열 평균 32바이트 × 100만)
- **Bloom Filter**: 약 1.14MB (약 9.6비트/원소)

약 **28배의 메모리 절약**이 가능합니다.

---

## Bloom Filter의 작동 원리

### 구조

Bloom Filter는 두 가지로 구성됩니다:
1. **m 비트 배열**: 초기에 모두 0으로 초기화
2. **k개의 독립적인 해시 함수**: 각 원소를 비트 배열의 k개 위치로 매핑

### 삽입(Insert)

원소 x를 삽입할 때 k개의 해시 함수 h₁(x), h₂(x), ..., hₖ(x) 각각이 반환하는 인덱스의 비트를 1로 설정합니다.

### 조회(Lookup)

원소 x를 조회할 때 k개의 해시 함수가 가리키는 모든 비트가 1이면 "존재할 수도 있음(possibly present)", 하나라도 0이면 "확실히 없음(definitely absent)"을 반환합니다.

### 오류율 수식

m 비트, k 해시 함수, n개 원소를 삽입했을 때 거짓 양성 확률 p는 다음과 같습니다:

```
p ≈ (1 - e^(-kn/m))^k
```

최적의 k는 `k = (m/n) * ln(2)` 이며, 이를 대입하면 `m = -n * ln(p) / (ln 2)²` 입니다.

---

## 구현 예제 1: Python으로 Bloom Filter 구현

```python
import math
import hashlib
from bitarray import bitarray  # pip install bitarray

class BloomFilter:
    def __init__(self, n: int, p: float = 0.01):
        """
        n: 예상 원소 수
        p: 허용 거짓 양성 확률 (0 < p < 1)
        """
        self.n = n
        self.p = p
        # 최적 비트 배열 크기
        self.m = math.ceil(-n * math.log(p) / (math.log(2) ** 2))
        # 최적 해시 함수 수
        self.k = math.ceil((self.m / n) * math.log(2))
        self.bits = bitarray(self.m)
        self.bits.setall(False)
        print(f"Bloom Filter: m={self.m} bits ({self.m/8/1024:.2f} KB), k={self.k} hash functions")

    def _hash_positions(self, item: str):
        positions = []
        for seed in range(self.k):
            h = hashlib.md5(f"{seed}:{item}".encode()).hexdigest()
            pos = int(h, 16) % self.m
            positions.append(pos)
        return positions

    def add(self, item: str):
        for pos in self._hash_positions(item):
            self.bits[pos] = True

    def __contains__(self, item: str) -> bool:
        return all(self.bits[pos] for pos in self._hash_positions(item))


# 사용 예시
bf = BloomFilter(n=1_000_000, p=0.01)

# 삽입
urls = [f"https://example.com/page/{i}" for i in range(1_000_000)]
for url in urls:
    bf.add(url)

# 정확한 조회 (삽입된 원소)
print("https://example.com/page/42 존재?", "https://example.com/page/42" in bf)  # True

# 거짓 양성 테스트 (삽입되지 않은 원소)
test_urls = [f"https://other.com/page/{i}" for i in range(10_000)]
false_positives = sum(1 for url in test_urls if url in bf)
print(f"거짓 양성률: {false_positives / len(test_urls):.4f} (목표: {bf.p})")
```

**실행 결과 예시:**
```
Bloom Filter: m=9585059 bits (1163.69 KB), k=7 hash functions
https://example.com/page/42 존재? True
거짓 양성률: 0.0098 (목표: 0.01)
```

거짓 음성(false negative)은 절대 발생하지 않습니다. 삽입된 원소를 "없다"고 하는 경우는 없습니다.

---

## 구현 예제 2: Counting Bloom Filter와 Cuckoo Filter 비교

표준 Bloom Filter는 삭제가 불가능합니다. 비트를 0으로 되돌리면 다른 원소의 정보가 손상될 수 있기 때문입니다.

### Counting Bloom Filter

비트 배열 대신 카운터 배열을 사용합니다. 삭제 시 카운터를 감소시킵니다. 단, 메모리가 3~4배 증가합니다.

```python
class CountingBloomFilter:
    def __init__(self, n: int, p: float = 0.01, counter_bits: int = 4):
        self.m = math.ceil(-n * math.log(p) / (math.log(2) ** 2))
        self.k = math.ceil((self.m / n) * math.log(2))
        self.max_count = (1 << counter_bits) - 1  # 15 for 4-bit
        self.counters = [0] * self.m

    def _hash_positions(self, item: str):
        positions = []
        for seed in range(self.k):
            h = hashlib.md5(f"{seed}:{item}".encode()).hexdigest()
            positions.append(int(h, 16) % self.m)
        return positions

    def add(self, item: str):
        for pos in self._hash_positions(item):
            if self.counters[pos] < self.max_count:
                self.counters[pos] += 1

    def remove(self, item: str):
        # 먼저 존재 여부 확인
        if item not in self:
            return
        for pos in self._hash_positions(item):
            if self.counters[pos] > 0:
                self.counters[pos] -= 1

    def __contains__(self, item: str) -> bool:
        return all(self.counters[pos] > 0 for pos in self._hash_positions(item))


# 사용 예시 — 삭제 가능
cbf = CountingBloomFilter(n=10_000, p=0.01)
cbf.add("apple")
cbf.add("banana")
print("apple 존재?", "apple" in cbf)   # True
cbf.remove("apple")
print("apple 삭제 후 존재?", "apple" in cbf)  # False
print("banana 존재?", "banana" in cbf)  # True (영향 없음)
```

### Cuckoo Filter (쿠쿠 필터)

2014년 CMU에서 발표된 Cuckoo Filter는 쿠쿠 해싱을 기반으로 하며, 삭제를 지원하면서도 Bloom Filter보다 공간 효율이 높습니다 (거짓 양성률 < 3% 범위에서).

원리:
1. 각 원소의 **핑거프린트(fingerprint)** 를 2바이트 등 소형으로 계산
2. 두 개의 후보 버킷(bucket₁, bucket₂)을 계산
3. 삽입 시 비어 있는 버킷에 핑거프린트 저장
4. 가득 찼으면 기존 핑거프린트를 쿠쿠처럼 쫓아내어 대체

```python
import struct

class CuckooFilter:
    def __init__(self, capacity: int, bucket_size: int = 4, fingerprint_bits: int = 8):
        self.capacity = capacity
        self.bucket_size = bucket_size
        self.fp_mask = (1 << fingerprint_bits) - 1
        self.buckets = [[None] * bucket_size for _ in range(capacity)]
        self.max_kicks = 500

    def _fingerprint(self, item: str) -> int:
        h = int(hashlib.sha256(item.encode()).hexdigest(), 16)
        fp = h & self.fp_mask
        return fp or 1  # 0은 사용 불가 (빈 슬롯 표시)

    def _bucket_index(self, item: str) -> int:
        h = int(hashlib.md5(item.encode()).hexdigest(), 16)
        return h % self.capacity

    def _alt_index(self, index: int, fp: int) -> int:
        # i2 = i1 XOR hash(fingerprint)
        fp_hash = int(hashlib.md5(str(fp).encode()).hexdigest(), 16)
        return (index ^ fp_hash) % self.capacity

    def add(self, item: str) -> bool:
        fp = self._fingerprint(item)
        i1 = self._bucket_index(item)
        i2 = self._alt_index(i1, fp)

        for i in [i1, i2]:
            if None in self.buckets[i]:
                self.buckets[i][self.buckets[i].index(None)] = fp
                return True

        # 쿠쿠킹: 무작위로 쫓아내기
        import random
        i = random.choice([i1, i2])
        for _ in range(self.max_kicks):
            slot = random.randrange(self.bucket_size)
            fp, self.buckets[i][slot] = self.buckets[i][slot], fp
            i = self._alt_index(i, fp)
            if None in self.buckets[i]:
                self.buckets[i][self.buckets[i].index(None)] = fp
                return True
        return False  # 필터 가득 참

    def __contains__(self, item: str) -> bool:
        fp = self._fingerprint(item)
        i1 = self._bucket_index(item)
        i2 = self._alt_index(i1, fp)
        return fp in self.buckets[i1] or fp in self.buckets[i2]

    def remove(self, item: str) -> bool:
        fp = self._fingerprint(item)
        i1 = self._bucket_index(item)
        i2 = self._alt_index(i1, fp)
        for i in [i1, i2]:
            if fp in self.buckets[i]:
                self.buckets[i][self.buckets[i].index(fp)] = None
                return True
        return False


cf = CuckooFilter(capacity=1024)
cf.add("hello")
cf.add("world")
print("hello 존재?", "hello" in cf)   # True
cf.remove("hello")
print("hello 삭제 후?", "hello" in cf)  # False
```

---

## 자료구조 비교 요약

| 특성 | Bloom Filter | Counting BF | Cuckoo Filter | HashSet |
|---|---|---|---|---|
| 메모리 (백만 원소, 1% FPR) | ~1.1 MB | ~4.4 MB | ~1 MB | ~32 MB |
| 거짓 양성 | 가능 | 가능 | 가능 | 없음 |
| 거짓 음성 | 없음 | 없음 | 없음 | 없음 |
| 삭제 | 불가 | 가능 | 가능 | 가능 |
| 조회 속도 | O(k) | O(k) | O(1) | O(1) 평균 |

---

## 주의사항과 팁

1. **해시 함수 선택**: 암호학적 해시(MD5, SHA)는 느립니다. 실제 프로덕션에서는 MurmurHash, xxHash 등 비암호학적 해시를 사용하세요.

2. **최적 파라미터 계산**: `m = -n * ln(p) / (ln 2)²` 공식으로 비트 수를 미리 계산하세요. 원소 수를 과소추정하면 오류율이 급격히 올라갑니다.

3. **동시성**: 표준 Bloom Filter는 비트 배열을 공유하므로, 멀티스레드 환경에서는 아토믹 연산 또는 락이 필요합니다.

4. **직렬화**: 비트 배열을 디스크에 저장하면 재시작 후에도 필터를 복원할 수 있습니다. Redis의 `BF.ADD` 명령이 이를 지원합니다.

5. **Cuckoo Filter 용량 한계**: 로드 팩터가 95% 이상이면 삽입 실패가 급증합니다. 여유 있게 `capacity`를 설정하세요.

---

## 참고 자료

- [Wikipedia: Bloom filter](https://en.wikipedia.org/wiki/Bloom_filter)
- [Cuckoo Filter 원논문 (CMU, 2014)](https://www.cs.cmu.edu/~binfan/papers/conext14_cuckoofilter.pdf)
- [Redis Bloom Filter 모듈 공식 문서](https://redis.io/docs/data-types/probabilistic/bloom-filter/)
- [Brilliant.org: Cuckoo Filter](https://brilliant.org/wiki/cuckoo-filter/)
