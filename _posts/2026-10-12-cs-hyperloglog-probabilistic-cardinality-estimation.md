---
layout: post
title: "HyperLogLog 완전 정복: 12KB로 수십억 개의 원소 수를 세는 확률적 마법"
date: 2026-10-12
categories: [cs, computer-science]
tags: [hyperloglog, probabilistic, cardinality, redis, hashing, big-data, sketch]
---

## HyperLogLog란 무엇인가?

웹 서비스를 운영하다 보면 "오늘 순 방문자(unique visitor) 수는 몇 명인가?" 또는 "이 쿼리로 반환되는 고유 값의 수는 얼마인가?"와 같은 질문에 자주 직면한다. 정확한 집합의 크기(cardinality)를 구하는 가장 직관적인 방법은 HashSet이나 BitSet을 사용하는 것이다. 그러나 하루에 수억 건의 요청이 들어오는 서비스에서 HashSet을 유지하면 GB 단위의 메모리가 필요하다.

**HyperLogLog(HLL)**는 이 문제를 혁신적으로 해결한다. 2007년 Philippe Flajolet 등이 발표한 이 알고리즘은 **단 12KB의 메모리**로 수십억 개의 원소를 가진 집합의 크기를 **오차율 0.81% 이내**로 추정할 수 있다. "LogLog"라는 이름은 알고리즘이 사용하는 메모리가 `O(log log n)`에 비례한다는 사실에서 유래한다.

HyperLogLog는 Apache Spark의 `approx_count_distinct`, PostgreSQL의 `hll` 확장, Redis의 내장 자료형 등 수많은 프로덕션 시스템에서 사용된다. 이 글에서는 HLL의 수학적 원리부터 직접 구현, 그리고 Redis를 통한 실전 활용까지 깊이 있게 살펴본다.

---

## 왜 HyperLogLog가 필요한가?

### 정확한 집합 크기 계산의 비용

집합에 원소를 추가하고 정확한 크기를 알려면 기존에 삽입한 원소를 모두 기억해야 한다. `n`개의 원소를 저장하면 O(n) 메모리가 필요하다. 예를 들어 1억 명의 고유 사용자 ID(각 64비트)를 HashSet으로 유지하면:

```
1억 × 8바이트 = 800MB
```

하루 10억 명의 순 방문자가 있다면 8GB가 필요하다. 게다가 해시 충돌 방지를 위한 오버헤드까지 더하면 실제 메모리 사용량은 몇 배 더 늘어난다.

### HLL의 혁신: 확률적 근사(Probabilistic Approximation)

HLL은 정확성 대신 메모리를 극단적으로 줄이는 대가로, 작은 오차(~1%)를 허용한다. 이 트레이드오프는 많은 실무 상황에서 완전히 합리적이다:

- **웹 분석**: "오늘 UV가 1,000,234명인지 1,000,301명인지"는 중요하지 않다.
- **데이터베이스 쿼리 플래너**: 정확한 카디널리티보다 대략적인 값이면 충분하다.
- **네트워크 트래픽 분석**: 고유 IP 수를 실시간으로 추정할 때 메모리가 제한된다.

---

## 수학적 원리: 해시와 Leading Zeros

### 핵심 직관: 동전 던지기

HLL의 핵심 직관은 **희귀한 패턴의 관찰 빈도**에서 집합의 크기를 추정한다는 것이다. 공정한 동전을 던진다고 생각해 보자:

- 앞면이 1번 연속 나올 확률: 1/2
- 앞면이 2번 연속 나올 확률: 1/4
- 앞면이 k번 연속 나올 확률: 1/2^k

만약 실험 중에 앞면이 최대 10번 연속 나왔다면, 시도 횟수는 대략 2^10 = 1024번이라고 추정할 수 있다.

HLL은 이 직관을 해시 함수에 적용한다. 각 원소를 해시하면 균일한 이진 문자열이 생성된다. 이 문자열에서 **Leading Zeros의 최대값(ρ_max)**을 추적하면, 원소 수 n을 `2^ρ_max`로 추정할 수 있다.

### 분산 감소: Bucket(Register) 분할

단순한 최대 Leading Zeros 추정은 분산이 너무 크다. HLL은 이를 해결하기 위해 **m개의 버킷(레지스터)**으로 나눈다:

1. 해시 값의 처음 `b` 비트로 버킷 인덱스(j)를 결정한다.
2. 나머지 비트에서 Leading Zeros의 수(ρ)를 계산한다.
3. 각 버킷의 최댓값을 `M[j] = max(M[j], ρ)`로 갱신한다.
4. 최종 추정: 조화 평균을 이용한 공식 적용

```
E = α_m × m² × (Σ 2^(-M[j]))^(-1)
```

여기서 α_m은 편향 보정 상수, m은 버킷 수다. Redis는 m = 16384(2^14)개의 6비트 레지스터를 사용하여 총 `16384 × 6 / 8 = 12,288 바이트 ≈ 12KB`를 사용한다.

---

## 실제 구현 예제

### 예제 1: Python으로 HyperLogLog 구현

```python
import hashlib
import math
import struct

class HyperLogLog:
    def __init__(self, p: int = 14):
        """
        p: 버킷 비트 수. m = 2^p 개의 레지스터를 사용.
        p=14 → m=16384, 오차율 ≈ 0.81% (Redis 기본값)
        """
        self.p = p
        self.m = 1 << p  # 2^p
        self.registers = [0] * self.m
        self.alpha = self._get_alpha(self.m)

    def _get_alpha(self, m: int) -> float:
        if m == 16:
            return 0.673
        elif m == 32:
            return 0.697
        elif m == 64:
            return 0.709
        else:
            return 0.7213 / (1 + 1.079 / m)

    def _hash(self, value: str) -> int:
        h = hashlib.md5(value.encode()).digest()
        return struct.unpack('<Q', h[:8])[0]  # 64비트 해시

    def _rho(self, w: int, max_bits: int) -> int:
        """Leading zeros + 1 계산 (w의 상위 max_bits 비트 기준)"""
        for i in range(max_bits, 0, -1):
            if w & (1 << (i - 1)):
                return max_bits - i + 1
        return max_bits + 1

    def add(self, value: str):
        x = self._hash(value)
        # 상위 p 비트 → 레지스터 인덱스
        j = x >> (64 - self.p)
        # 나머지 비트에서 leading zeros 계산
        w = x & ((1 << (64 - self.p)) - 1)
        rho = self._rho(w, 64 - self.p)
        self.registers[j] = max(self.registers[j], rho)

    def count(self) -> int:
        Z = sum(2.0 ** (-m) for m in self.registers)
        E = self.alpha * self.m * self.m / Z

        # 소범위 보정 (small range correction)
        if E <= 2.5 * self.m:
            V = self.registers.count(0)
            if V > 0:
                E = self.m * math.log(self.m / V)

        # 대범위 보정 (large range correction)
        elif E > (1 << 32) / 30:
            E = -(1 << 32) * math.log(1 - E / (1 << 32))

        return int(E)

    def merge(self, other: 'HyperLogLog') -> 'HyperLogLog':
        """두 HLL 스케치 병합 (PFMERGE에 해당)"""
        assert self.p == other.p
        merged = HyperLogLog(self.p)
        merged.registers = [max(a, b) for a, b in zip(self.registers, other.registers)]
        return merged


# 실험: 실제 원소 수 vs HLL 추정값 비교
import random
import string

def random_string(length=10):
    return ''.join(random.choices(string.ascii_letters, k=length))

hll = HyperLogLog(p=14)
actual_set = set()

for n in [1_000, 10_000, 100_000, 1_000_000]:
    while len(actual_set) < n:
        s = random_string()
        actual_set.add(s)
        hll.add(s)

    estimated = hll.count()
    error = abs(estimated - n) / n * 100
    print(f"실제: {n:>10,} | 추정: {estimated:>10,} | 오차: {error:.2f}%")
```

실행 결과 (예시):
```
실제:      1,000 | 추정:      1,023 | 오차: 2.30%
실제:     10,000 | 추정:      9,974 | 오차: 0.26%
실제:    100,000 | 추정:    100,789 | 오차: 0.79%
실제:  1,000,000 | 추정:    998,341 | 오차: 0.17%
```

원소 수가 많아질수록 중심극한정리에 의해 오차율이 이론값인 `1.04/√m ≈ 0.81%`에 수렴한다.

---

### 예제 2: Redis HyperLogLog로 실시간 UV 카운터 구축

Redis는 `PFADD`, `PFCOUNT`, `PFMERGE` 세 가지 명령어로 HLL을 지원한다. 아래는 실시간 UV 집계 시스템의 예다:

```python
import redis
from datetime import datetime, timedelta

class UVCounter:
    """Redis HyperLogLog를 이용한 일별/월별 UV 카운터"""

    def __init__(self, host='localhost', port=6379):
        self.r = redis.Redis(host=host, port=port, decode_responses=True)

    def _daily_key(self, date: datetime = None) -> str:
        if date is None:
            date = datetime.now()
        return f"uv:daily:{date.strftime('%Y-%m-%d')}"

    def _monthly_key(self, year: int, month: int) -> str:
        return f"uv:monthly:{year}-{month:02d}"

    def track_visit(self, user_id: str, timestamp: datetime = None):
        """방문 기록 — 최대 12KB 사용으로 수억 명 UV 추적 가능"""
        if timestamp is None:
            timestamp = datetime.now()

        daily_key = self._daily_key(timestamp)
        self.r.pfadd(daily_key, user_id)
        # 키 만료: 90일 후 자동 삭제
        self.r.expire(daily_key, 60 * 60 * 24 * 90)

    def get_daily_uv(self, date: datetime = None) -> int:
        return self.r.pfcount(self._daily_key(date))

    def get_monthly_uv(self, year: int, month: int) -> int:
        """해당 월의 모든 일별 키를 PFMERGE로 합산"""
        daily_keys = []
        start = datetime(year, month, 1)
        end = datetime(year, month + 1, 1) if month < 12 else datetime(year + 1, 1, 1)
        current = start

        while current < end:
            key = self._daily_key(current)
            if self.r.exists(key):
                daily_keys.append(key)
            current += timedelta(days=1)

        if not daily_keys:
            return 0

        # PFMERGE: 중복 제거 후 월별 UV 계산 (추가 메모리: 12KB 단 하나)
        monthly_key = self._monthly_key(year, month)
        self.r.pfmerge(monthly_key, *daily_keys)
        self.r.expire(monthly_key, 3600)  # 1시간 캐시
        return self.r.pfcount(monthly_key)

    def compare_memory_usage(self, n_users: int):
        """HashSet vs HLL 메모리 비교"""
        hashset_bytes = n_users * 16  # UUID 평균 16바이트
        hll_bytes = 12 * 1024         # Redis HLL: 항상 12KB
        print(f"사용자 수: {n_users:,}")
        print(f"HashSet 메모리: {hashset_bytes / 1024 / 1024:.1f} MB")
        print(f"HLL 메모리:     {hll_bytes / 1024:.1f} KB (항상 고정)")
        print(f"절약 비율: {hashset_bytes / hll_bytes:.0f}x")


# 사용 예시
counter = UVCounter()
today = datetime.now()

# 방문 기록
for user_id in ['user_001', 'user_002', 'user_003', 'user_001']:  # user_001 중복
    counter.track_visit(user_id, today)

print(f"오늘 UV: {counter.get_daily_uv(today)}")  # 3 (중복 제거)

counter.compare_memory_usage(100_000_000)
# 사용자 수: 100,000,000
# HashSet 메모리: 1,525.9 MB
# HLL 메모리:     12.0 KB (항상 고정)
# 절약 비율: 130,048x
```

`PFMERGE`의 강력한 점은 여러 날의 HLL 스케치를 병합할 때도 각각 12KB이므로, **월별 30개의 스케치를 합쳐도 정확도를 유지하면서 단 12KB**로 결과를 산출한다는 것이다.

---

## 주의사항 및 실전 팁

### 1. 오차율은 원소 수와 무관하다
HLL의 오차율은 레지스터 수 `m`에 의해서만 결정된다(`1.04/√m`). 원소 수가 1천만이든 10억이든 동일한 오차율을 갖는다. 메모리를 4배 늘리면(m을 4배 증가) 오차율이 절반으로 줄어든다.

### 2. HLL은 삭제를 지원하지 않는다
한번 추가된 원소는 제거할 수 없다. 삭제가 필요한 경우에는 슬라이딩 윈도우 방식으로 여러 HLL 스케치를 사용하거나, Count-Min Sketch + HLL 조합을 검토하라.

### 3. 병합(Merge)의 무손실성
같은 파라미터(p값)의 두 HLL은 레지스터별 최댓값을 취하는 방식으로 완벽하게 병합된다. 이 특성 덕분에 분산 환경에서 각 노드가 독립적으로 HLL을 유지하고 나중에 합산할 수 있다.

### 4. 작은 집합에 대한 보정
원소 수가 `2.5m`보다 작을 때는 빈 레지스터의 수를 이용한 LinearCounting으로 전환해야 한다. 구현 시 이 분기 처리를 빠뜨리면 소규모 집합에서 오차가 급격히 커진다.

### 5. Redis HLL의 Sparse/Dense 표현
Redis는 원소 수가 적을 때 압축된 Sparse 포맷을 사용하고, 일정 임계치를 넘으면 Dense 포맷(12KB)으로 전환한다. 따라서 소규모 집합에서는 12KB보다 훨씬 적은 메모리를 사용한다.

---

## 마무리

HyperLogLog는 "정확도를 약간 희생하면 메모리를 수천 배 절약할 수 있다"는 트레이드오프의 교과서적인 사례다. 수십억 원소의 집합 크기를 12KB로 추정하는 이 알고리즘은 확률론, 정보 이론, 해시 함수가 결합된 걸작이다.

실무에서는 Redis의 `PFADD`/`PFCOUNT`/`PFMERGE`를 사용하면 HLL을 즉시 적용할 수 있으며, 정확한 카운터가 필요하지 않은 UV, 고유 쿼리 수, 네트워크 유량 분석 등의 시나리오에서 큰 효과를 발휘한다. 다음에 "수억 개의 고유 값을 세어야 해"라는 요구사항을 만난다면, HyperLogLog를 가장 먼저 고려하라.

---

## 참고 자료
- [HyperLogLog — Wikipedia](https://en.wikipedia.org/wiki/HyperLogLog)
- [Redis HyperLogLog 공식 문서](https://redis.io/docs/data-types/hyperloglogs)
- [HyperLogLog in Practice (Google, 2013)](https://research.google/pubs/hyperloglog-in-practice-algorithmic-engineering-of-a-state-of-the-art-cardinality-estimation-algorithm/)
- [Antirez: Redis new data structure — HyperLogLog](http://antirez.com/news/75)
