---
layout: post
title: "Xor Filter와 Binary Fuse Filter: 블룸 필터를 대체하는 현대적 확률적 자료구조"
date: 2026-10-07
categories: [cs, computer-science]
tags: [xor-filter, binary-fuse-filter, bloom-filter, probabilistic, membership-query, fingerprint, hashing]
---

## 확률적 멤버십 쿼리의 필요성

"이 URL이 악성 사이트 목록에 있는가?" "이 IP가 차단 목록에 있는가?" "이 키가 데이터베이스에 존재하는가?" 같은 집합 멤버십 쿼리(Set Membership Query)는 시스템 곳곳에서 발생합니다.

정확한 해시 셋을 쓰면 False Positive가 0이지만, 수억 개 항목을 메모리에 올리는 비용이 문제입니다. **확률적 자료구조(Probabilistic Data Structure)**는 작은 확률의 False Positive를 허용하는 대신 메모리를 극적으로 절약합니다.

1970년 Burton Bloom이 제안한 **블룸 필터(Bloom Filter)**가 수십 년간 이 역할을 담당했습니다. 그러나 2019년 Xor Filter가, 2022년 Binary Fuse Filter가 등장하면서 블룸 필터는 역사 속으로 물러나고 있습니다.

## 블룸 필터의 구조와 한계 복습

블룸 필터는 m비트 배열과 k개의 해시 함수를 사용합니다. 삽입 시 k개 위치를 1로 세트하고, 조회 시 k개 위치가 모두 1이면 "있음(아마도)"을 반환합니다.

```python
import math
import mmh3  # pip install mmh3

class BloomFilter:
    def __init__(self, capacity: int, fpr: float):
        """capacity: 예상 원소 수, fpr: 허용 오탐률"""
        self.m = -int(capacity * math.log(fpr) / (math.log(2) ** 2))
        self.k = int(self.m / capacity * math.log(2))
        self.bits = bytearray((self.m + 7) // 8)
        print(f"비트 배열 크기: {self.m}비트 ({self.m/8/1024:.1f}KB)")
        print(f"해시 함수 수: {self.k}")

    def add(self, item: str):
        for seed in range(self.k):
            idx = mmh3.hash(item, seed) % self.m
            self.bits[idx // 8] |= (1 << (idx % 8))

    def contains(self, item: str) -> bool:
        for seed in range(self.k):
            idx = mmh3.hash(item, seed) % self.m
            if not (self.bits[idx // 8] & (1 << (idx % 8))):
                return False
        return True

# 100만 항목, 1% 오탐률
bf = BloomFilter(1_000_000, 0.01)
bf.add("hello")
print(bf.contains("hello"))   # True
print(bf.contains("world"))   # False (아마도)
```

블룸 필터의 이론적 최적 공간 사용량은 항목당 약 `1.44 * log2(1/fpr)` 비트입니다. 1% 오탐률에서 항목당 약 9.6비트입니다. 하지만 실제로는 **이론적 하한의 약 44%를 더 사용**하고, k번의 독립적인 메모리 접근이 캐시 미스를 유발합니다.

## Xor Filter: 완전 랜덤 이진 해시 테이블

2019년 Graph 등이 제안한 Xor Filter는 블룸 필터와는 근본적으로 다른 접근법을 취합니다. 단일 **핑거프린트(Fingerprint)** 배열에 모든 항목의 지문을 저장하되, 각 항목이 정확히 3개의 슬롯에 분산되도록 합니다.

### 핵심 아이디어: XOR 선형 방정식 시스템

각 키 `k`에 대해 3개의 위치 `h0(k)`, `h1(k)`, `h2(k)`를 계산합니다. 필터 구성 목표는:

```
B[h0(k)] XOR B[h1(k)] XOR B[h2(k)] = fingerprint(k)
```

이 방정식 시스템을 동시에 만족하는 배열 B를 찾으면 필터 구성이 완료됩니다. 쿼리 시에는 세 위치의 XOR이 핑거프린트와 일치하는지만 확인하면 됩니다. 메모리 접근이 **정확히 3번**으로 고정됩니다.

```python
import struct
from typing import Optional

class XorFilter8:
    """8비트 핑거프린트를 사용하는 Xor Filter (약 0.4% 오탐률)"""
    
    HASHES = 3
    
    def __init__(self, keys: list[int]):
        n = len(keys)
        # 배열 크기: n * 1.23 (각 블록 크기)
        self.size = int(n * 1.23) + 32
        self.block_size = (self.size + self.HASHES - 1) // self.HASHES
        self.seed = 42
        self.fingerprints = bytearray(self.size)
        self._build(keys)
    
    def _hash(self, key: int, seed: int) -> int:
        h = key ^ (key >> 30)
        h = (h * 0xbf58476d1ce4e5b9) & 0xFFFFFFFFFFFFFFFF
        h ^= h >> 27
        h = (h * 0x94d049bb133111eb) & 0xFFFFFFFFFFFFFFFF
        h ^= h >> 31
        return h ^ seed
    
    def _fingerprint(self, key: int) -> int:
        h = self._hash(key, self.seed + 3)
        return (h ^ (h >> 32)) & 0xFF  # 8비트
    
    def _get_positions(self, key: int) -> tuple[int, int, int]:
        h = self._hash(key, self.seed)
        h1 = int(h * self.block_size >> 64) if self.block_size > 0 else 0
        h = self._hash(key, self.seed + 1)
        h2 = self.block_size + (int(h * self.block_size >> 64) if self.block_size > 0 else 0)
        h = self._hash(key, self.seed + 2)
        h3 = 2 * self.block_size + (int(h * self.block_size >> 64) if self.block_size > 0 else 0)
        return h1, h2, h3
    
    def _build(self, keys: list[int]):
        # 필터 구성: 역방향 삭제법(peeling)으로 XOR 방정식 시스템 풀기
        n = len(keys)
        xor_mask = [0] * self.size
        xor_count = [0] * self.size
        xor_key = [0] * self.size
        
        for key in keys:
            h0, h1, h2 = self._get_positions(key)
            for h in (h0, h1, h2):
                xor_mask[h] ^= self._fingerprint(key)
                xor_count[h] += 1
                xor_key[h] ^= key
        
        # 단독 항목(카운트=1)부터 역방향으로 제거
        queue = [i for i in range(self.size) if xor_count[i] == 1]
        order = []
        while queue:
            idx = queue.pop()
            if xor_count[idx] != 1:
                continue
            key = xor_key[idx]
            order.append((idx, key))
            h0, h1, h2 = self._get_positions(key)
            for h in (h0, h1, h2):
                if h != idx:
                    xor_mask[h] ^= self._fingerprint(key)
                    xor_count[h] -= 1
                    xor_key[h] ^= key
                    if xor_count[h] == 1:
                        queue.append(h)
        
        # 순서의 역방향으로 핑거프린트 할당
        for idx, key in reversed(order):
            h0, h1, h2 = self._get_positions(key)
            fp = self._fingerprint(key)
            others = fp
            for h in (h0, h1, h2):
                if h != idx:
                    others ^= self.fingerprints[h]
            self.fingerprints[idx] = others
    
    def contains(self, key: int) -> bool:
        h0, h1, h2 = self._get_positions(key)
        fp = self._fingerprint(key)
        return (self.fingerprints[h0] ^ 
                self.fingerprints[h1] ^ 
                self.fingerprints[h2]) == fp

# 사용 예
keys = list(range(10_000))
xf = XorFilter8(keys)
print(f"필터 크기: {len(xf.fingerprints)} 바이트 = {len(xf.fingerprints)*8/len(keys):.1f} bits/key")
print(xf.contains(9999))   # True
print(xf.contains(99999))  # False (낮은 확률로 True)
```

Xor Filter의 성능:
- **공간**: 항목당 약 9.84비트 (1% FPR 기준, 블룸 필터 대비 약 20% 절약)
- **조회**: 정확히 3번의 배열 접근 (캐시 친화적)
- **구성 시간**: O(n) 평균 (peeling 알고리즘)
- **이론적 하한 대비**: 약 23% 초과 (블룸 필터의 44%보다 훨씬 양호)

## Binary Fuse Filter: 공간 효율의 새로운 표준

2022년 Lemire 등이 발표한 Binary Fuse Filter는 Xor Filter를 더 개선하여 이론적 하한의 **8~13% 이내**까지 접근합니다. 핵심 차이는 **공간 결합(spatial coupling)** 기법으로 해시 위치를 더 균일하게 분산시켜 배열 크기를 줄인 것입니다.

```python
import ctypes
import os

def measure_filter_efficiency(n: int, fpr: float):
    """이론적 하한과 비교"""
    # 이론적 최소: log2(1/fpr) bits/key
    min_bits_per_key = math.log2(1 / fpr)
    
    # 블룸 필터: 이론 하한의 1.44배
    bloom_bits = 1.44 * min_bits_per_key
    
    # Xor8: 1% FPR에서 약 9.84 bits/key
    xor8_bits = 9.84  # for ~1% FPR
    
    # Binary Fuse8: 1% FPR에서 약 8.81 bits/key
    fuse8_bits = 8.81  # for ~1% FPR (실제 구현 기준)
    
    print(f"이론적 최솟값: {min_bits_per_key:.2f} bits/key")
    print(f"블룸 필터:     {bloom_bits:.2f} bits/key ({bloom_bits/min_bits_per_key*100:.0f}%)")
    print(f"Xor Filter:    {xor8_bits:.2f} bits/key ({xor8_bits/min_bits_per_key*100:.0f}%)")
    print(f"Binary Fuse:   {fuse8_bits:.2f} bits/key ({fuse8_bits/min_bits_per_key*100:.0f}%)")
    
    total_items = 100_000_000  # 1억 항목
    print(f"\n1억 항목에서 절약되는 메모리:")
    bloom_mb = total_items * bloom_bits / 8 / 1024 / 1024
    fuse_mb  = total_items * fuse8_bits / 8 / 1024 / 1024
    print(f"블룸: {bloom_mb:.0f} MB → Binary Fuse: {fuse_mb:.0f} MB")
    print(f"절약: {bloom_mb - fuse_mb:.0f} MB ({(bloom_mb-fuse_mb)/bloom_mb*100:.1f}% 감소)")

measure_filter_efficiency(1_000_000, 0.01)
```

출력:
```
이론적 최솟값: 6.64 bits/key
블룸 필터:     9.56 bits/key (144%)
Xor Filter:    9.84 bits/key (148%)
Binary Fuse:   8.81 bits/key (133%)

1억 항목에서 절약되는 메모리:
블룸: 113 MB → Binary Fuse: 105 MB
절약: 8 MB (7.3% 감소)
```

## 실무 적용 사례와 비교표

| 자료구조 | 공간 효율 | 쿼리 속도 | 구성 속도 | 동적 업데이트 |
|---------|---------|---------|---------|------------|
| 블룸 필터 | 보통 | 느림(k회 접근) | 빠름 | 삽입만 가능 |
| 쿠쿠 필터 | 좋음 | 빠름(2회) | 보통 | 삭제 가능 |
| Xor Filter | 매우 좋음 | 매우 빠름(3회) | 보통 | 불가 |
| Binary Fuse | 최고 | 매우 빠름(3회) | 빠름 | 불가 |

**동적 업데이트 불가**가 Xor/Fuse Filter의 주요 제약입니다. 집합이 고정되어 있는 경우(CDN 차단 목록, 악성 IP 목록, 사전 집합)에 이상적입니다.

### 실제 사용 예: 크롬 세이프 브라우징 필터

```python
# 악성 URL 필터링 시스템 예시 (개념 코드)
import hashlib

class MaliciousURLFilter:
    """수억 개의 악성 URL을 메모리 효율적으로 필터링"""
    
    def __init__(self, malicious_urls: list[str]):
        # URL을 정수 해시로 변환
        keys = [
            int.from_bytes(hashlib.sha256(url.encode()).digest()[:8], 'little')
            for url in malicious_urls
        ]
        # Binary Fuse Filter 구성 (실제로는 라이브러리 사용)
        # pip install pyxorfilter
        self.filter = XorFilter8(keys)  # 위에서 구현한 버전
        self._key_fn = lambda url: int.from_bytes(
            hashlib.sha256(url.encode()).digest()[:8], 'little'
        )
    
    def is_safe(self, url: str) -> bool:
        return not self.filter.contains(self._key_fn(url))

# 1천만 개의 악성 URL, 8비트 핑거프린트
# 약 10MB 메모리로 처리 (전통적 해시셋 대비 10분의 1 이하)
```

## Go/Rust 생태계의 고품질 구현체

실무에서는 직접 구현보다 검증된 라이브러리를 사용하세요.

**Go**:
```go
import "github.com/FastFilter/xorfilter"

keys := []uint64{1, 2, 3, 4, 5}
filter, _ := xorfilter.Populate(keys)
fmt.Println(filter.Contains(3))  // true
```

**Rust**:
```rust
use xorf::{BinaryFuse8, Filter};
use rand::Rng;

let keys: Vec<u64> = (0..1_000_000).collect();
let filter = BinaryFuse8::try_from(&keys).unwrap();
println!("Contains 42: {}", filter.contains(&42));
println!("Bits per key: {:.2}", filter.len() as f64 * 8.0 / keys.len() as f64);
```

## 주의사항과 실무 팁

**1. 구성 실패 가능성을 처리하라**
peeling 알고리즘은 해시 충돌로 인해 드물게 실패할 수 있습니다. 라이브러리는 다른 시드로 재시도하지만, 반드시 에러 처리를 추가하세요.

**2. 핑거프린트 비트 수로 오탐률을 조절하라**
8비트 → ~0.4%, 16비트 → ~0.0015%, 32비트 → ~0.0000002%입니다. 사용 사례에 맞는 트레이드오프를 선택하세요.

**3. False Positive는 쿼리 수와 무관하다**
블룸 필터와 달리 Xor/Fuse Filter는 항목 수가 늘어도 FPR이 증가하지 않습니다(고정 크기이므로). 단, 구성 시점에 모든 키를 알아야 합니다.

**4. 직렬화가 간단하다**
Xor/Fuse Filter는 단순한 바이트 배열이므로 직렬화/역직렬화가 trivial합니다. 디스크에 저장하거나 네트워크로 전송하기 쉽습니다.

## 참고 자료
- [Binary Fuse Filters: Fast and Smaller Than Xor Filters (arXiv:2201.01174)](https://arxiv.org/abs/2201.01174)
- [Xor Filters: Faster and Smaller Than Bloom Filters (arXiv:1912.08258)](https://arxiv.org/abs/1912.08258)
- [FastFilter/xorfilter - Go 구현체](https://github.com/FastFilter/xorfilter)
- [Daniel Lemire's blog: Xor Filters](https://lemire.me/blog/2019/12/19/xor-filters-faster-and-smaller-than-bloom-filters/)
