---
layout: post
title: "HyperLogLog 완전 정복: 수십억 개의 고유 값을 12KB로 세는 확률적 알고리즘"
date: 2026-07-02
categories: [cs, computer-science]
tags: [hyperloglog, probabilistic, cardinality, redis, streaming, data-structure, algorithm]
---

## HyperLogLog란 무엇인가?

HyperLogLog(HLL)는 **집합 내 고유 원소의 수(카디널리티)를 매우 적은 메모리로 추정**하는 확률적 알고리즘입니다. Philippe Flajolet, Éric Fusy, Olivier Gandouet, Frédéric Meunier가 2007년 논문 *"HyperLogLog: the analysis of a near-optimal cardinality estimation algorithm"*에서 발표했으며, Redis, Apache Flink, BigQuery, Presto 등 수많은 대규모 시스템에서 핵심 컴포넌트로 채택되었습니다.

핵심 가치는 **공간 효율성**에 있습니다. 10억 개의 고유 사용자 ID(각 8바이트)를 정확히 세려면 약 8GB 메모리가 필요합니다. HyperLogLog는 단 **12KB**의 고정 메모리로 0.81% 표준 오차 이내에서 동일한 작업을 수행합니다. 정확도와 공간을 트레이드오프하는 대신, 실용적으로는 이 오차율이 충분히 작은 경우가 대부분입니다.

## 왜 HyperLogLog가 필요한가?

### 정확한 카디널리티의 비용

정확한 카디널리티 계산은 **모든 원소를 메모리에 보관**해야 합니다. 해시셋으로 구현하면 삽입은 O(1)이지만 메모리는 원소 수에 비례합니다. 스트리밍 환경에서는 이 방식이 불가능합니다:

- 하루 1억 번의 고유 방문자를 정확히 세려면 수 GB 메모리
- 네트워크 트래픽에서 고유 소스 IP 수를 실시간으로 추적하려면 무한한 메모리
- 광고 플랫폼에서 고유 노출 수를 집계하면 수십억 개의 이벤트

이런 환경에서는 **정확도를 약간 희생하고 공간을 획기적으로 줄이는** 접근이 필수적입니다. HyperLogLog가 그 해답입니다.

### 선행 알고리즘: Flajolet-Martin

HyperLogLog의 전신은 1985년 Flajolet과 Martin이 제안한 **Probabilistic Counting** 알고리즘입니다. 아이디어는 단순합니다: 무작위 해시 값에서 **선두 0비트의 최대값(ρ_max)**을 관찰하면 카디널리티를 추정할 수 있다는 것입니다.

균일하게 분포된 해시 값에서 k개의 연속된 0비트로 시작하는 값이 나올 확률은 2^(-k)입니다. 따라서 최대 선두 0비트 수가 k라면, 대략 2^k개의 원소를 처리했다고 추론할 수 있습니다. HyperLogLog는 이 아이디어를 **스토캐스틱 평균(stochastic averaging)**과 **조화 평균(harmonic mean)**으로 개선하여 정밀도를 극적으로 높였습니다.

## 핵심 알고리즘 원리

### 1단계: 해싱과 레지스터 분할

원소를 32비트 또는 64비트 해시로 변환합니다. 해시 값의 **상위 b비트**로 m = 2^b개의 버킷(레지스터) 중 하나를 선택하고, **나머지 비트**의 선두 0비트 수를 해당 레지스터에 저장합니다.

```
hash(element) = [b비트 인덱스][나머지 비트 ...]
                    ↓                ↓
              레지스터 선택      선두 0비트 수 계산
```

Redis의 기본 구현은 b = 14이므로 m = 16,384개의 레지스터를 사용합니다. 각 레지스터는 최대 6비트 값(0~63)을 저장하므로 총 16,384 × 6 / 8 = **12,288 바이트 ≈ 12KB**입니다.

### 2단계: 조화 평균으로 카디널리티 추정

각 레지스터 i의 최대 선두 0비트 수를 M[i]라 하면, 카디널리티 추정값은 다음과 같습니다:

```
E = α_m × m² × (Σ 2^(-M[i]))^(-1)
```

여기서 α_m은 편향 보정 상수, m은 레지스터 수, 합산은 i = 1 ~ m에 대해 수행합니다. **조화 평균**을 사용하는 이유는 산술 평균보다 이상치(극단적으로 큰 M[i])의 영향을 덜 받기 때문입니다.

### 3단계: 소수/대수 보정

- **카디널리티가 작을 때**: LinearCounting 알고리즘으로 전환 (빈 레지스터 수 활용)
- **카디널리티가 클 때**: 32비트 해시의 충돌 보정 적용
- **중간 범위**: 위의 기본 공식 사용

## 실제 구현 예제

### 예제 1: Python으로 HyperLogLog 직접 구현

```python
import hashlib
import math
import struct

class HyperLogLog:
    def __init__(self, b: int = 14):
        """
        b: 레지스터 수의 로그2 값 (4 ~ 16)
        b=14이면 16,384개 레지스터, 표준오차 ≈ 0.81%
        """
        if not 4 <= b <= 16:
            raise ValueError("b must be between 4 and 16")
        self.b = b
        self.m = 1 << b  # 2^b 개의 레지스터
        self.registers = [0] * self.m
        self.alpha = self._get_alpha(self.m)

    def _get_alpha(self, m: int) -> float:
        if m == 16:   return 0.673
        if m == 32:   return 0.697
        if m == 64:   return 0.709
        return 0.7213 / (1 + 1.079 / m)

    def _hash(self, element) -> int:
        h = hashlib.sha1(str(element).encode()).digest()
        return struct.unpack(">I", h[:4])[0]  # 32비트 해시

    def _leading_zeros(self, value: int, max_bits: int) -> int:
        """나머지 비트에서 선두 0비트 수 + 1 계산"""
        if value == 0:
            return max_bits + 1
        count = 1
        while not (value & (1 << (max_bits - 1))):
            count += 1
            value <<= 1
        return count

    def add(self, element) -> None:
        h = self._hash(element)
        # 상위 b비트로 레지스터 인덱스 결정
        j = h >> (32 - self.b)
        # 나머지 비트에서 선두 0비트 계산
        w = h & ((1 << (32 - self.b)) - 1)
        leading = self._leading_zeros(w, 32 - self.b)
        self.registers[j] = max(self.registers[j], leading)

    def count(self) -> int:
        """카디널리티 추정값 반환"""
        # 조화 평균으로 원시 추정값 계산
        raw = self.alpha * (self.m ** 2) / sum(2 ** (-r) for r in self.registers)

        # 소수 보정: LinearCounting
        if raw <= 2.5 * self.m:
            zeros = self.registers.count(0)
            if zeros > 0:
                return round(self.m * math.log(self.m / zeros))

        # 대수 보정: 32비트 해시 충돌
        if raw <= (1 << 32) / 30:
            return round(raw)
        return round(-(1 << 32) * math.log(1 - raw / (1 << 32)))

    def merge(self, other: "HyperLogLog") -> None:
        """두 HLL을 병합 (레지스터별 최댓값)"""
        if self.b != other.b:
            raise ValueError("Cannot merge HLL with different precision")
        for i in range(self.m):
            self.registers[i] = max(self.registers[i], other.registers[i])


# 사용 예시
hll = HyperLogLog(b=14)
import random

ACTUAL_COUNT = 1_000_000
unique_ids = set()
for _ in range(ACTUAL_COUNT):
    uid = random.randint(1, 10_000_000)
    unique_ids.add(uid)
    hll.add(uid)

actual = len(unique_ids)
estimated = hll.count()
error_rate = abs(estimated - actual) / actual * 100
print(f"실제 카디널리티:  {actual:,}")
print(f"HLL 추정값:       {estimated:,}")
print(f"오차율:           {error_rate:.2f}%")
print(f"메모리 사용량:    {hll.m * 1} bytes (레지스터 배열)")
# 예시 출력:
# 실제 카디널리티:  999,823
# HLL 추정값:       1,003,456
# 오차율:           0.37%
# 메모리 사용량:    16384 bytes
```

### 예제 2: Redis HyperLogLog 명령어와 실전 사용 패턴

Redis는 `PFADD`, `PFCOUNT`, `PFMERGE` 명령어로 HLL을 기본 지원합니다. 아래는 웹 서비스의 DAU(일간 활성 사용자) 집계 패턴입니다.

```python
import redis
from datetime import date, timedelta

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def track_user_visit(user_id: str, visit_date: date = None) -> None:
    """사용자 방문 이벤트를 HLL에 기록"""
    if visit_date is None:
        visit_date = date.today()
    key = f"dau:{visit_date.isoformat()}"
    r.pfadd(key, user_id)
    r.expire(key, 90 * 86400)  # 90일 TTL

def get_dau(target_date: date) -> int:
    """특정 날짜의 DAU 추정값 반환"""
    key = f"dau:{target_date.isoformat()}"
    return r.pfcount(key)

def get_wau(end_date: date) -> int:
    """주간 활성 사용자(WAU) — 7일치 HLL 병합"""
    keys = [
        f"dau:{(end_date - timedelta(days=i)).isoformat()}"
        for i in range(7)
    ]
    dest_key = f"wau:{end_date.isoformat()}"
    r.pfmerge(dest_key, *keys)
    r.expire(dest_key, 7 * 86400)
    return r.pfcount(dest_key)

def get_mau(end_date: date) -> int:
    """월간 활성 사용자(MAU) — 30일치 HLL 병합"""
    keys = [
        f"dau:{(end_date - timedelta(days=i)).isoformat()}"
        for i in range(30)
    ]
    dest_key = f"mau:{end_date.isoformat()}"
    r.pfmerge(dest_key, *keys)
    r.expire(dest_key, 7 * 86400)
    return r.pfcount(dest_key)


# 시뮬레이션: 3일간 사용자 방문 기록
import random
today = date.today()

for day_offset in range(3):
    visit_date = today - timedelta(days=day_offset)
    # 각 날짜에 50,000 ~ 80,000명 방문
    n_visitors = random.randint(50_000, 80_000)
    for _ in range(n_visitors):
        user_id = str(random.randint(1, 500_000))  # 일부 재방문자 포함
        track_user_visit(user_id, visit_date)

dau = get_dau(today)
wau = get_wau(today)
print(f"오늘의 DAU: {dau:,}")
print(f"주간 WAU:   {wau:,}")
print(f"월간 MAU:   {get_mau(today):,}")
# PFMERGE가 자동으로 중복을 제거하므로 WAU ≤ DAU × 7
```

`PFMERGE`의 핵심 특성은 **레지스터별 최댓값(max)**을 취한다는 점입니다. 이는 합집합(union) 연산에 해당하며, 덕분에 날짜별 HLL을 독립적으로 유지하면서 임의 기간의 고유 사용자 수를 계산할 수 있습니다.

## 주의사항과 실전 팁

### 1. 교집합(Intersection) 추정의 한계

HyperLogLog는 **합집합 카디널리티**만 직접 계산할 수 있습니다. 교집합은 포함-배제 원리로 간접 추정해야 하며, 집합이 많아질수록 오차가 누적됩니다.

```
|A ∩ B| ≈ |A| + |B| - |A ∪ B|
```

집합이 거의 겹치지 않거나 매우 많이 겹치는 극단적인 경우에는 오차가 커질 수 있습니다.

### 2. 비결정론적 특성 주의

같은 원소 집합이라도 삽입 순서나 해시 충돌에 따라 추정값이 미세하게 달라질 수 있습니다. 단위 테스트에서 정확한 값을 기대하면 안 됩니다. 오차 범위(`error_rate < 2%`)로 검증하세요.

### 3. b 값 선택 기준

| b 값 | 레지스터 수 | 메모리 | 표준 오차 |
|------|------------|--------|-----------|
| 10   | 1,024      | 768B   | 2.60%     |
| 12   | 4,096      | 3KB    | 1.30%     |
| 14   | 16,384     | 12KB   | 0.81%     |
| 16   | 65,536     | 48KB   | 0.40%     |

대부분의 프로덕션 환경에서는 b=14 (Redis 기본값)가 최적입니다.

### 4. 해시 함수 품질이 중요

해시 함수가 균일 분포를 보장해야 알고리즘이 올바르게 작동합니다. MD5나 SHA-1처럼 암호학적 해시가 필요하진 않지만, MurmurHash3, xxHash처럼 **통계적으로 균일한 비암호화 해시**가 적합합니다.

### 5. 메모리 절약의 실제 규모

100개 서비스에서 각 30일치 DAU HLL을 보관한다면:
- 정확한 해시셋 방식: 수억 명 × 8바이트 × 100 × 30 = 테라바이트 급
- HyperLogLog: 12KB × 100 × 30 = **36MB** (수만 배 절약)

## 참고 자료

- [Redis HyperLogLog 공식 문서](https://redis.io/docs/latest/develop/data-types/probabilistic/hyperloglogs/)
- [HyperLogLog Wikipedia](https://en.wikipedia.org/wiki/HyperLogLog)
- [원본 논문: HyperLogLog - the analysis of a near-optimal cardinality estimation algorithm](https://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf)
- [HyperLogLog in Practice (Google 개선 버전)](https://research.google/pubs/hyperloglog-in-practice-algorithmic-engineering-of-a-state-of-the-art-cardinality-estimation-algorithm/)
