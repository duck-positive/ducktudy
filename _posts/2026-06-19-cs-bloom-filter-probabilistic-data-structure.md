---
layout: post
title: "블룸 필터(Bloom Filter) 완전 정복: 공간 효율적인 확률적 자료구조"
date: 2026-06-19
categories: [cs, computer-science]
tags: [bloom-filter, data-structure, probabilistic, hashing, redis, cassandra]
---

## 블룸 필터란 무엇인가?

블룸 필터(Bloom Filter)는 1970년 Burton Howard Bloom이 제안한 **확률적(probabilistic) 자료구조**입니다. 특정 원소가 집합에 속하는지 여부를 빠르고 공간 효율적으로 테스트하는 데 특화되어 있습니다.

블룸 필터는 두 가지 대답만 할 수 있습니다:
- **"확실히 없음(Definitely NOT in set)"**: 원소가 집합에 절대 없다고 보장
- **"아마 있음(Possibly in set)"**: 원소가 있을 수도 있음 (거짓 양성 가능)

즉, **False Negative는 절대 발생하지 않지만**, **False Positive는 발생할 수 있습니다**. 이 특성 덕분에 블룸 필터는 "없다"는 결과는 100% 신뢰할 수 있어 캐시 미스 방지, URL 중복 체크, 스팸 필터링 등에 매우 유용합니다.

---

## 왜 블룸 필터가 필요한가?

### 전통적인 집합 자료구조의 한계

해시셋(HashSet)은 멤버십 테스트를 O(1)에 수행하지만, 원소 하나당 수십~수백 바이트를 사용합니다. 수억 개의 URL을 저장하는 웹 크롤러를 생각해 보세요. 10억 개의 URL을 HashSet에 저장하면 수십~수백 GB의 메모리가 필요합니다.

### 블룸 필터의 공간 효율

블룸 필터는 원소 자체를 저장하지 않습니다. 대신 **비트 배열(bit array)**과 **k개의 해시 함수**만 사용합니다. 10억 개의 URL에 대해 1% 거짓 양성률을 허용한다면, 약 **9.6비트/원소** = 약 1.2GB로 충분합니다. HashSet 대비 수십~수백 배 공간을 절약합니다.

### 실제 사용 사례

- **Apache Cassandra**: SSTable의 읽기 요청 전, 해당 키가 파일에 없으면 디스크 I/O 자체를 생략
- **Redis**: `BF.ADD`, `BF.EXISTS` 명령어로 블룸 필터 기본 제공
- **Google Chrome**: 악성 URL 목록을 클라이언트에 블룸 필터로 저장해 빠른 차단
- **HBase/BigTable**: 행 키가 존재하지 않는 블록 스캔을 건너뜀
- **Medium**: 이미 읽은 아티클 추천 제외

---

## 내부 동작 원리

### 구조

블룸 필터는 다음 두 가지로 구성됩니다:
1. **m비트의 비트 배열** (초기값 모두 0)
2. **k개의 독립적인 해시 함수** (각각 0 ~ m-1 범위의 인덱스를 반환)

### 삽입(Insert)

원소를 삽입할 때 k개의 해시 함수를 적용하여 k개의 인덱스를 구하고, 해당 비트를 모두 1로 설정합니다.

### 조회(Lookup)

원소를 조회할 때 동일하게 k개의 해시 함수를 적용합니다.
- k개의 비트가 **모두 1이면**: "아마 있음" (거짓 양성 가능)
- 하나라도 **0이면**: "확실히 없음" (거짓 음성 없음)

---

## 실제 구현 예제

### 예제 1: Python으로 블룸 필터 직접 구현

```python
import math
import hashlib


class BloomFilter:
    def __init__(self, capacity: int, false_positive_rate: float = 0.01):
        """
        capacity: 예상 원소 개수
        false_positive_rate: 허용할 거짓 양성 비율 (0~1)
        """
        self.capacity = capacity
        self.fp_rate = false_positive_rate

        # 최적 비트 배열 크기: m = -(n * ln(p)) / (ln(2)^2)
        self.bit_size = self._optimal_bit_size(capacity, false_positive_rate)

        # 최적 해시 함수 개수: k = (m/n) * ln(2)
        self.hash_count = self._optimal_hash_count(self.bit_size, capacity)

        # 비트 배열 초기화 (bytearray 사용으로 메모리 효율화)
        self.bit_array = bytearray(math.ceil(self.bit_size / 8))

        print(f"[BloomFilter] bit_size={self.bit_size}, hash_count={self.hash_count}")
        print(f"[BloomFilter] 메모리 사용량: {len(self.bit_array)} bytes "
              f"({len(self.bit_array) / 1024:.2f} KB)")

    @staticmethod
    def _optimal_bit_size(n: int, p: float) -> int:
        return int(-(n * math.log(p)) / (math.log(2) ** 2))

    @staticmethod
    def _optimal_hash_count(m: int, n: int) -> int:
        return max(1, int((m / n) * math.log(2)))

    def _get_hash_positions(self, item: str) -> list[int]:
        """
        SHA-256과 MD5를 시드로 삼아 k개의 독립적인 해시 위치를 생성합니다.
        double hashing 기법: h_i(x) = (h1(x) + i * h2(x)) mod m
        """
        h1 = int(hashlib.sha256(item.encode()).hexdigest(), 16)
        h2 = int(hashlib.md5(item.encode()).hexdigest(), 16)
        return [(h1 + i * h2) % self.bit_size for i in range(self.hash_count)]

    def _set_bit(self, pos: int):
        byte_idx = pos // 8
        bit_idx = pos % 8
        self.bit_array[byte_idx] |= (1 << bit_idx)

    def _get_bit(self, pos: int) -> bool:
        byte_idx = pos // 8
        bit_idx = pos % 8
        return bool(self.bit_array[byte_idx] & (1 << bit_idx))

    def add(self, item: str):
        for pos in self._get_hash_positions(item):
            self._set_bit(pos)

    def contains(self, item: str) -> bool:
        return all(self._get_bit(pos) for pos in self._get_hash_positions(item))


# --- 사용 예시 ---
if __name__ == "__main__":
    bf = BloomFilter(capacity=10_000, false_positive_rate=0.01)

    # 삽입
    visited_urls = [
        "https://example.com/page/1",
        "https://example.com/page/2",
        "https://example.com/page/3",
    ]
    for url in visited_urls:
        bf.add(url)

    # 조회
    print(bf.contains("https://example.com/page/1"))   # True (삽입됨)
    print(bf.contains("https://example.com/page/99"))  # False (삽입 안 됨, 확실히 없음)

    # 거짓 양성률 실험
    import random, string

    false_positives = 0
    test_count = 10_000
    for _ in range(test_count):
        fake = ''.join(random.choices(string.ascii_lowercase, k=20))
        if bf.contains(fake):
            false_positives += 1

    print(f"실측 거짓 양성률: {false_positives / test_count:.4f} (목표: 0.01)")
```

위 코드를 실행하면 설정한 `false_positive_rate`에 근사한 실측 거짓 양성률을 확인할 수 있습니다.

---

### 예제 2: Redis 블룸 필터 활용 (RedisBloom 모듈)

Redis는 `RedisBloom` 모듈을 통해 블룸 필터를 네이티브로 지원합니다. 아래는 Python `redis-py` 클라이언트를 사용하는 예시입니다.

```python
import redis

# RedisBloom이 활성화된 Redis 인스턴스에 연결
r = redis.Redis(host="localhost", port=6379, decode_responses=True)

BLOOM_KEY = "visited:urls"

def crawler_visit(url: str) -> bool:
    """
    URL을 크롤링하기 전 블룸 필터로 중복 방문 여부를 확인합니다.
    반환값: True = 이미 방문(skip), False = 처음 방문(crawl)
    """
    # BF.ADD: 원소를 추가하고, 이미 있었으면 0, 새로 추가됐으면 1 반환
    is_new = r.execute_command("BF.ADD", BLOOM_KEY, url)

    if is_new == 1:
        print(f"[NEW] 크롤링 시작: {url}")
        return False  # 크롤링 필요
    else:
        print(f"[SKIP] 이미 방문: {url}")
        return True   # 크롤링 불필요


def check_exists(url: str) -> bool:
    """BF.EXISTS로 URL 존재 여부만 확인 (삽입 없음)"""
    result = r.execute_command("BF.EXISTS", BLOOM_KEY, url)
    return result == 1


def setup_bloom_filter(expected_items: int = 1_000_000, error_rate: float = 0.001):
    """
    BF.RESERVE로 커스텀 블룸 필터를 사전 생성합니다.
    error_rate: 0.001 = 0.1% 거짓 양성률
    """
    try:
        r.execute_command("BF.RESERVE", BLOOM_KEY, error_rate, expected_items)
        print(f"[INIT] 블룸 필터 생성 완료: capacity={expected_items}, fp_rate={error_rate}")
    except redis.exceptions.ResponseError:
        print("[INIT] 블룸 필터가 이미 존재합니다.")


if __name__ == "__main__":
    setup_bloom_filter(expected_items=1_000_000, error_rate=0.001)

    urls = [
        "https://news.example.com/article/101",
        "https://news.example.com/article/102",
        "https://news.example.com/article/101",  # 중복
    ]

    for url in urls:
        crawler_visit(url)

    # 직접 존재 여부 확인
    print(check_exists("https://news.example.com/article/101"))  # True
    print(check_exists("https://news.example.com/article/999"))  # False (확실히 없음)
```

---

## 거짓 양성률(False Positive Rate) 수식

$$P_{fp} \approx \left(1 - e^{-kn/m}\right)^k$$

- **n**: 삽입된 원소 수
- **m**: 비트 배열 크기
- **k**: 해시 함수 개수

**최적 k값**: $k = \frac{m}{n} \ln 2$

이 수식을 활용하면 허용 거짓 양성률에 맞는 최적 비트 배열 크기와 해시 함수 개수를 미리 계산할 수 있습니다.

| 원소 수(n) | FP Rate | 비트 크기(m) | 해시 수(k) | 메모리 |
|---|---|---|---|---|
| 1,000,000 | 1% | 9,585,059 | 7 | ~1.2 MB |
| 1,000,000 | 0.1% | 14,377,589 | 10 | ~1.8 MB |
| 10,000,000 | 1% | 95,850,584 | 7 | ~11.7 MB |

---

## 주의사항과 팁

### 1. 삭제가 불가능하다

표준 블룸 필터는 비트를 0으로 되돌릴 수 없어 **삭제를 지원하지 않습니다**. 여러 원소가 같은 비트를 공유하기 때문에 하나를 지우면 다른 원소의 정보도 손상됩니다.

**해결책**: `Counting Bloom Filter`는 비트 대신 카운터를 사용해 삽입/삭제를 모두 지원하지만, 메모리 사용량이 4~8배 증가합니다.

### 2. 크기는 처음에 결정된다

표준 블룸 필터는 동적으로 크기를 늘릴 수 없습니다. 예상 원소 수보다 훨씬 많은 원소가 삽입되면 거짓 양성률이 급격히 올라갑니다.

**해결책**: `Scalable Bloom Filter`는 내부적으로 여러 개의 필터를 계층으로 쌓아 동적 확장을 지원합니다.

### 3. 해시 함수의 품질이 중요하다

해시 함수가 충돌이 많거나 편향이 있으면 이론보다 거짓 양성률이 높아집니다. **MurmurHash3** 또는 **xxHash** 처럼 비암호화 해시이지만 분포가 균일하고 빠른 함수를 권장합니다.

### 4. 실전 설계 가이드

- 예상 원소 수를 **넉넉하게(1.5~2배)** 잡는 것이 안전합니다
- FP Rate은 애플리케이션 요구사항에 맞게 설정: 크롤러는 0.1~1%, 보안 필터는 0.001%
- 주기적으로 블룸 필터를 **재구성(rebuild)** 하는 루틴을 마련하세요

---

## 참고 자료

- [Bloom Filters by Example - llimllib.github.io](https://llimllib.github.io/bloomfilter-tutorial/)
- [Introduction to Bloom Filter — Baeldung CS](https://www.baeldung.com/cs/bloom-filter)
- [Bloom Filters Explained — System Design](https://systemdesign.one/bloom-filters-explained/)
- [Bloom Filters Introduction and Python Implementation — GeeksforGeeks](https://www.geeksforgeeks.org/python/bloom-filters-introduction-and-python-implementation/)
