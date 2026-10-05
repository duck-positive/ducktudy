---
layout: post
title: "이레이저 코딩과 Reed-Solomon 코드 완전 정복: 분산 스토리지가 데이터 손실을 복구하는 방법"
date: 2026-10-05
categories: [cs, computer-science]
tags: [erasure-coding, reed-solomon, distributed-storage, galois-field, fault-tolerance, ceph, hdfs]
---

## 개요

하드 디스크는 언젠가 반드시 고장난다. 구글의 연구에 따르면 첫 해 1.7%, 3년 후 8.6%의 HDD가 고장난다. 수십만 대의 디스크를 운용하는 AWS S3나 Google GCS 같은 대규모 클라우드 스토리지에서는 매일 수십 대의 디스크가 장애를 일으킨다.

이런 환경에서 데이터를 어떻게 안전하게 보존할까?

가장 단순한 방법은 **복제(replication)**다. 데이터를 3곳에 동일하게 저장하면 2곳이 동시에 고장나도 데이터를 복구할 수 있다. 하지만 저장 공간이 3배 필요하다.

**이레이저 코딩(Erasure Coding)**은 이보다 훨씬 저장 효율적인 방법이다. k개의 데이터 조각을 n개(n > k)의 코딩된 조각으로 변환하여, 임의의 k개의 조각만 있으면 원본을 복구할 수 있다. 중복도는 n/k배이고, n-k개의 조각 손실까지 허용한다.

AWS S3 Glacier는 (14, 10) Reed-Solomon 코드를 사용한다. 14개 조각 중 10개만 있으면 복구 가능하고, 4개 조각 손실을 허용하며, 저장 공간은 1.4배만 사용한다. 3배 복제 대비 저장 효율이 2.1배 더 좋다.

---

## 왜 필요한가

### RAID와의 비교

| 방식 | 허용 고장 수 | 저장 오버헤드 | 사용처 |
|------|------------|--------------|--------|
| RAID-1 (미러링) | n/2 개 | 200% | 로컬 디스크 |
| RAID-5 | 1개 | 1/n 패리티 | 서버 스토리지 |
| RAID-6 | 2개 | 2/n 패리티 | 서버 스토리지 |
| 3-way 복제 | 2개 | 300% | 분산 스토리지 |
| **(k, n) 이레이저 코딩** | **n-k개** | **n/k** | **클라우드 스토리지** |

### 실제 시스템 사용 현황

- **Facebook f4**: (14, 10) RS 코드 — 데이터 오버헤드 40%, 연간 수 엑사바이트 절약
- **Backblaze Vaults**: (20, 17) RS 코드
- **Google Colossus**: 다양한 EC 설정
- **Ceph**: Jerasure 라이브러리 사용, (K+M, K) 설정 지원
- **HDFS-EC**: 오픈소스 하둡에 내장, 여러 RS 코드 지원

---

## 핵심 원리: 어떻게 복구하는가

### XOR 기반 단순 이레이저 코드

가장 단순한 형태는 XOR 패리티다. k=3, n=4 예시:

```
d0 = 10110100
d1 = 01101011
d2 = 11001001
p0 = d0 XOR d1 XOR d2 = 00010110  ← 패리티
```

d1이 손실되면: `d1 = d0 XOR d2 XOR p0`로 복구 가능.
단, 2개 이상의 데이터 손실은 복구 불가. 이것이 RAID-5의 원리다.

### Reed-Solomon 코드: 다항식 기반 접근

Reed-Solomon 코드는 XOR의 한계를 뛰어넘어 임의의 k개 조각으로 복구할 수 있다.

**핵심 아이디어**: k개의 데이터를 k-1차 다항식의 계수로 표현하고, 이 다항식을 n개의 서로 다른 점에서 평가한다. n개의 점 중 어느 k개로도 k-1차 다항식을 유일하게 복원할 수 있다(라그랑주 보간법).

```
데이터: d = [d0, d1, d2]  (k=3)
다항식: P(x) = d0 + d1*x + d2*x²

인코딩 (n=5 점):
  s0 = P(1) = d0 + d1 + d2
  s1 = P(2) = d0 + 2*d1 + 4*d2
  s2 = P(3) = d0 + 3*d1 + 9*d2
  s3 = P(4) = d0 + 4*d1 + 16*d2
  s4 = P(5) = d0 + 5*d1 + 25*d2

5개 중 임의의 3개로 P(x)를 복원 → d0, d1, d2 복구
```

현실에서는 정수 산술 대신 **유한 체(Galois Field)**를 사용한다. 정수 덧셈/곱셈은 오버플로가 발생하지만, GF(2^8) (8비트 유한 체)는 항상 8비트 결과를 보장하고 분수 없이 나눗셈이 가능하다.

---

## 실제 구현 예제

### 예제 1: XOR 기반 단순 이레이저 코드 (Python)

```python
class SimpleErasureCode:
    """
    XOR 기반 (k, k+1) 이레이저 코드.
    k개 데이터 조각 + 1개 패리티.
    1개 조각 손실 복구 가능.
    """

    def encode(self, data_shards: list[bytes]) -> list[bytes]:
        """k개의 데이터 조각을 받아 1개의 패리티 조각 생성"""
        if not data_shards:
            return []
        size = max(len(s) for s in data_shards)
        # 길이를 맞추기 위해 패딩
        padded = [s.ljust(size, b'\x00') for s in data_shards]

        # 패리티 = 모든 데이터 XOR
        parity = bytearray(size)
        for shard in padded:
            for i in range(size):
                parity[i] ^= shard[i]

        return padded + [bytes(parity)]

    def decode(self, shards: list[bytes | None], k: int) -> list[bytes]:
        """
        shards: 조각 목록, 손실된 조각은 None
        k: 원본 데이터 조각 수
        """
        assert len(shards) == k + 1, "shards 개수가 맞지 않음"
        none_indices = [i for i, s in enumerate(shards) if s is None]

        if len(none_indices) == 0:
            return list(shards[:k])
        if len(none_indices) > 1:
            raise ValueError("XOR 코드는 1개 손실만 복구 가능")

        lost_idx = none_indices[0]
        size = max(len(s) for s in shards if s is not None)

        # 손실된 조각 복구: 나머지 조각들의 XOR
        recovered = bytearray(size)
        for i, shard in enumerate(shards):
            if i == lost_idx:
                continue
            s = shard.ljust(size, b'\x00')
            for j in range(size):
                recovered[j] ^= s[j]

        result = list(shards[:k])
        result[lost_idx] = bytes(recovered)
        return result


# 사용 예시
ec = SimpleErasureCode()
data = [b"Hello", b"World", b"!"]
encoded = ec.encode(data)
print("인코딩된 조각:", [s.hex() for s in encoded])

# 두 번째 조각(index 1) 손실 시뮬레이션
lost_shards = [encoded[0], None, encoded[2], encoded[3]]
recovered = ec.decode(lost_shards, k=3)
print("복구된 데이터:", [s.rstrip(b'\x00') for s in recovered])
# → [b'Hello', b'World', b'!']
```

### 예제 2: Reed-Solomon 코드 사용 (reedsolo 라이브러리)

```python
# pip install reedsolo
import reedsolo

class ReedSolomonStorage:
    """
    Reed-Solomon (n, k) 이레이저 코드.
    n = 전체 조각 수, k = 데이터 조각 수
    (n-k)개 조각 손실까지 복구 가능.
    """

    def __init__(self, n: int, k: int):
        assert n > k, "n은 k보다 커야 함"
        self.n = n
        self.k = k
        self.parity_count = n - k
        self.rs = reedsolo.RSCodec(self.parity_count)

    def encode(self, data: bytes) -> list[bytes]:
        """
        데이터를 k개의 청크로 나누고 인코딩하여 n개 조각 반환.
        각 조각은 동일한 크기.
        """
        # 청크 크기 계산 (k 배수로 패딩)
        chunk_size = (len(data) + self.k - 1) // self.k
        padded = data.ljust(chunk_size * self.k, b'\x00')

        shards = []
        for i in range(self.k):
            shards.append(padded[i * chunk_size:(i + 1) * chunk_size])

        # reedsolo는 바이트 단위로 동작
        # 각 청크를 독립적으로 인코딩 (단순화된 예시)
        encoded_data = self.rs.encode(data)
        return [bytes(encoded_data[i::self.n]) for i in range(self.n)]

    def decode_simple(self, shards: list[bytes | None]) -> bytes:
        """손실된 조각(None)이 포함된 목록에서 원본 데이터 복구"""
        available = sum(1 for s in shards if s is not None)
        if available < self.k:
            raise ValueError(f"복구 불가: {available}개 조각 가용, {self.k}개 필요")

        # 실제 RS 디코딩 (erasure 위치 명시)
        erasures = [i for i, s in enumerate(shards) if s is None]
        combined = bytearray()
        for s in shards:
            if s is not None:
                combined.extend(s)

        try:
            decoded, _, _ = self.rs.decode(combined, erase_pos=erasures)
            return bytes(decoded)
        except reedsolo.ReedSolomonError as e:
            raise ValueError(f"RS 디코딩 실패: {e}")


# (6, 4) RS 코드 예시: 6개 조각 중 4개만 있으면 복구
rs = ReedSolomonStorage(n=6, k=4)

original_data = b"Hello, Erasure Coding World!"
print(f"원본 데이터: {original_data}")
print(f"원본 크기: {len(original_data)} bytes")

encoded_shards = rs.encode(original_data)
print(f"\n인코딩 결과: {len(encoded_shards)}개 조각")
for i, shard in enumerate(encoded_shards):
    print(f"  조각[{i}]: {shard.hex()[:20]}...")

# 2개 조각 손실 시뮬레이션 (인덱스 1, 3 손실)
damaged = list(encoded_shards)
damaged[1] = None
damaged[3] = None
print(f"\n조각 1, 3 손실 후: {[s is not None for s in damaged]}")

try:
    recovered = rs.decode_simple(damaged)
    print(f"복구 성공: {recovered}")
except ValueError as e:
    print(f"복구 실패: {e}")
```

---

## Galois Field(GF) 산술: 왜 필요한가

일반 정수로 Reed-Solomon을 구현하면 다항식 계수가 기하급수적으로 커진다. 또한, k개 점으로 다항식을 보간하려면 나눗셈이 필요한데, 정수 나눗셈은 분수를 만든다.

**GF(2^8)**는 256개의 원소(0~255)로 이루어진 유한 체로:
- 덧셈 = XOR (올림 없음, 항상 8비트)
- 곱셈 = GF 다항식 곱셈 mod 생성 다항식
- 나눗셈 = 역원 테이블 조회 (log/exp 테이블로 O(1) 구현)

```python
# GF(2^8) 산술 핵심 구현
GF_EXP = [0] * 512
GF_LOG = [0] * 256

def _init_gf_tables(primitive=0x11d):
    """GF(2^8) log/exp 테이블 초기화 (생성다항식 x^8+x^4+x^3+x^2+1 = 0x11d)"""
    x = 1
    for i in range(255):
        GF_EXP[i] = x
        GF_LOG[x] = i
        x <<= 1
        if x & 0x100:
            x ^= primitive
        x &= 0xFF
    for i in range(255, 512):
        GF_EXP[i] = GF_EXP[i - 255]

def gf_mul(x: int, y: int) -> int:
    """GF(2^8) 곱셈"""
    if x == 0 or y == 0:
        return 0
    return GF_EXP[(GF_LOG[x] + GF_LOG[y]) % 255]

def gf_div(x: int, y: int) -> int:
    """GF(2^8) 나눗셈"""
    if y == 0:
        raise ZeroDivisionError
    if x == 0:
        return 0
    return GF_EXP[(GF_LOG[x] - GF_LOG[y]) % 255]

_init_gf_tables()

# 예시
print(gf_mul(3, 7))   # → 9 (GF(2^8)에서의 3 × 7)
print(gf_div(9, 3))   # → 7 (역연산)
```

---

## 시스템 설계 시 고려사항

### 1. 복구 비용
이레이저 코딩의 복구는 복제 방식보다 훨씬 비싸다. (14, 10) RS 코드에서 하나의 조각을 복구하려면 10개의 조각을 읽어야 한다. 반면 3-way 복제는 다른 노드 1개만 읽으면 된다. **읽기 비율이 높은 데이터**에는 복제가, **쓰기 후 거의 읽지 않는 콜드 데이터**에는 EC가 유리하다.

### 2. 작은 파일 문제
RS 코드는 파일을 k개로 나누므로 매우 작은 파일은 오버헤드가 크다. 페이스북 f4는 작은 파일들을 하나의 큰 파일로 묶어(blob) EC를 적용한다.

### 3. 스트라이핑 vs 전체 파일 인코딩
대용량 파일은 고정 크기 스트라이프로 나누어 각 스트라이프마다 독립적으로 EC를 적용한다. 이렇게 하면 임의 읽기 시 불필요한 데이터를 읽지 않아도 된다.

### 4. 로컬 재구성 코드 (Locally Repairable Codes, LRC)
표준 RS 코드의 복구 비용을 줄이기 위해 조각들을 그룹으로 나누어 그룹 내 로컬 패리티를 추가하는 방식. Microsoft Azure Storage와 Facebook f4에서 사용. 단일 조각 복구 시 전체 k개가 아닌 로컬 그룹 내 r개만 읽으면 된다.

---

## 마무리

이레이저 코딩은 수학적으로 최적에 가까운 방법으로 데이터 내구성과 저장 효율을 동시에 달성한다. 단순 XOR부터 Reed-Solomon, 로컬 재구성 코드까지 다양한 변형이 실제 스토리지 시스템에 적용되고 있다. 데이터 손실에 대비하는 분산 시스템을 설계할 때, 데이터 접근 패턴과 복구 비용을 함께 고려하여 적절한 코딩 스킴을 선택하는 것이 핵심이다.

## 참고 자료
- [Plank, J. S. (2013). Erasure Coding for Storage Applications. FAST Tutorial.](https://web.eecs.utk.edu/~jplank/plank/papers/FAST-2013-Tutorial.html)
- [Sathiamoorthy, M. et al. (2013). XORing Elephants: Novel Erasure Codes for Big Data. VLDB.](https://arxiv.org/abs/1301.3791)
- [Wikipedia: Erasure code](https://en.wikipedia.org/wiki/Erasure_code)
- [Backblaze: Erasure Coding — Why 3x Replication is Out](https://www.backblaze.com/blog/reed-solomon/)
