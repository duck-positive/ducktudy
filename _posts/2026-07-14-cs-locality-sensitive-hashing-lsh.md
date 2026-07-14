---
layout: post
title: "LSH(Locality-Sensitive Hashing) 완전 정복: 고차원 유사도 검색의 핵심 알고리즘"
date: 2026-07-14
categories: [cs, computer-science]
tags: [LSH, locality-sensitive-hashing, 유사도검색, 해시, ANN, 벡터검색, MinHash, SimHash]
---

## 개요: 고차원 공간의 저주와 유사도 검색

10억 개의 이미지가 있다고 생각해보자. 새로운 이미지가 주어졌을 때, 이와 가장 유사한 이미지를 찾으려면 어떻게 해야 할까? 모든 이미지와 하나하나 비교하면 O(N)이 되고, N이 수십억이라면 실시간 응답은 불가능하다. 여기에 차원이 512차원인 벡터라면 유클리드 거리 계산 비용도 엄청나다.

이것이 바로 **차원의 저주(Curse of Dimensionality)**다. 고차원 공간에서는 모든 점들이 서로 비슷한 거리에 위치하게 되어, 전통적인 트리 기반 자료구조(KD-트리 등)가 제 성능을 내지 못한다.

**LSH(Locality-Sensitive Hashing)**은 이 문제를 근본적으로 다른 방식으로 풀어낸다: **유사한 항목은 같은 버킷에, 다른 항목은 다른 버킷에 해시할 확률을 높이는 해시 함수 패밀리를 설계하는 것이다.**

## 왜 LSH가 필요한가

### 기존 접근법의 한계

- **브루트 포스**: O(N·d) — N=10억, d=512차원이면 실시간 불가능
- **KD-트리**: 고차원(보통 20차원 이상)에서 성능 급락, O(N^(d-1)/d)에 근접
- **VP-트리, Ball-tree**: 마찬가지로 고차원 데이터에 부적합

### LSH의 핵심 아이디어

LSH는 **근사 최근접 이웃(Approximate Nearest Neighbor, ANN)** 문제를 풀기 위해 설계되었다. 정확한 답을 포기하는 대신, 확률적으로 가까운 이웃을 매우 빠르게 찾는다.

핵심 성질은 다음과 같다:
- 거리 `d(p, q) ≤ r1`인 두 점에 대해, 같은 해시값을 가질 확률 ≥ P1
- 거리 `d(p, q) ≥ r2`인 두 점에 대해, 같은 해시값을 가질 확률 ≤ P2

여기서 P1 > P2, r1 < r2가 되도록 설계한다.

## 핵심 개념: 해시 함수 패밀리

### (r, cr, P1, P2)-민감 해시 패밀리

해시 함수 패밀리 H가 다음 조건을 만족하면 **locality-sensitive**하다고 한다:

```
For p, q ∈ U:
  - If d(p,q) ≤ r  → Pr[h(p) = h(q)] ≥ P1
  - If d(p,q) ≥ cr → Pr[h(p) = h(q)] ≤ P2
```

여기서 c > 1은 근사 비율(approximation ratio)이다.

### AND-OR 증폭 기법

단일 해시 함수는 오탐(false positive)/미탐(false negative) 비율이 높을 수 있다. 이를 해결하기 위해:

- **AND 구성 (밴드 내부)**: k개의 해시 함수를 AND 조합 → 오탐 감소, P1/P2 간격 확대
- **OR 구성 (밴드 간)**: L개의 밴드를 OR 조합 → 미탐 감소, 재현율 향상

```
g_j(q) = [h_j1(q), h_j2(q), ..., h_jk(q)]  (j번째 밴드, k개 해시 AND)
q와 p가 후보 쌍이 되려면: 적어도 하나의 밴드 j에서 g_j(p) = g_j(q)
```

## 구체적인 LSH 변형들

### 1. MinHash (Jaccard 유사도, 집합 유사도)

MinHash는 두 집합의 **Jaccard 유사도**를 보존하는 LSH다.

**Jaccard 유사도**: J(A, B) = |A ∩ B| / |A ∪ B|

핵심 아이디어: 랜덤 순열 π에 대해 h_π(A) = min_{x∈A} π(x)라 할 때,
```
Pr[h_π(A) = h_π(B)] = J(A, B)
```

이 성질을 이용해 k개의 MinHash 함수를 AND로 조합하고, L개의 밴드를 OR로 조합한다.

```python
import hashlib
import numpy as np
from collections import defaultdict

class MinHashLSH:
    def __init__(self, num_hashes=100, bands=20):
        self.num_hashes = num_hashes
        self.bands = bands
        self.rows_per_band = num_hashes // bands
        # 랜덤 해시 파라미터 (a*x + b mod p)
        rng = np.random.RandomState(42)
        self.a = rng.randint(1, 2**31 - 1, num_hashes)
        self.b = rng.randint(0, 2**31 - 1, num_hashes)
        self.p = (1 << 31) - 1  # Mersenne prime
        self.buckets = defaultdict(list)

    def _minhash_signature(self, shingle_set):
        """집합에 대한 MinHash 서명 벡터 계산"""
        sig = np.full(self.num_hashes, np.inf)
        for elem in shingle_set:
            # 원소를 정수로 변환
            h = int(hashlib.md5(str(elem).encode()).hexdigest(), 16) % self.p
            # 각 해시 함수 적용
            hashes = (self.a * h + self.b) % self.p
            sig = np.minimum(sig, hashes)
        return sig.astype(int)

    def add(self, doc_id, shingle_set):
        sig = self._minhash_signature(shingle_set)
        for band_idx in range(self.bands):
            start = band_idx * self.rows_per_band
            end = start + self.rows_per_band
            band_key = (band_idx, tuple(sig[start:end]))
            self.buckets[band_key].append(doc_id)

    def query(self, shingle_set):
        sig = self._minhash_signature(shingle_set)
        candidates = set()
        for band_idx in range(self.bands):
            start = band_idx * self.rows_per_band
            end = start + self.rows_per_band
            band_key = (band_idx, tuple(sig[start:end]))
            candidates.update(self.buckets.get(band_key, []))
        return candidates

def make_shingles(text, k=3):
    """k-shingle 집합 생성"""
    return set(text[i:i+k] for i in range(len(text) - k + 1))

# 사용 예제
lsh = MinHashLSH(num_hashes=100, bands=20)

documents = {
    "doc1": "the quick brown fox jumps over the lazy dog",
    "doc2": "the quick brown fox leaps over the lazy cat",  # 유사
    "doc3": "machine learning is a subset of artificial intelligence",  # 다름
    "doc4": "the fast brown fox jumps over the sleepy dog",  # 유사
}

for doc_id, text in documents.items():
    shingles = make_shingles(text)
    lsh.add(doc_id, shingles)

query_text = "the quick brown fox jumps over the lazy dog"
candidates = lsh.query(make_shingles(query_text))
print(f"후보 문서: {candidates}")
# 출력: {'doc1', 'doc2', 'doc4'} (doc3는 너무 달라서 제외)
```

### 2. SimHash (코사인 유사도, 실수 벡터)

SimHash는 고차원 **실수 벡터**의 코사인 유사도를 보존하는 LSH다. Charikar(2002)가 제안했으며, Google이 웹 중복 문서 탐지에 사용했다.

핵심: 랜덤 단위 벡터 r에 대해 h_r(v) = sign(v · r)

```
Pr[h_r(p) = h_r(q)] = 1 - θ(p,q)/π
```

여기서 θ는 두 벡터 사이의 각도다. 즉, 유사한 방향의 벡터일수록 같은 해시 확률이 높다.

```python
import numpy as np
from collections import defaultdict

class SimHashLSH:
    def __init__(self, dim, num_hashes=64, bands=8):
        self.dim = dim
        self.num_hashes = num_hashes
        self.bands = bands
        self.rows = num_hashes // bands
        rng = np.random.RandomState(42)
        # 랜덤 하이퍼플레인 법선 벡터들
        self.planes = rng.randn(num_hashes, dim)
        self.buckets = defaultdict(list)
        self.data = {}

    def _hash_vector(self, vec):
        """벡터를 이진 서명으로 변환"""
        projections = self.planes @ vec  # shape: (num_hashes,)
        return (projections > 0).astype(int)

    def add(self, item_id, vec):
        sig = self._hash_vector(vec)
        self.data[item_id] = vec
        for band_idx in range(self.bands):
            start = band_idx * self.rows
            end = start + self.rows
            band_key = (band_idx, tuple(sig[start:end]))
            self.buckets[band_key].append(item_id)

    def query(self, vec, top_k=5):
        sig = self._hash_vector(vec)
        candidates = set()
        for band_idx in range(self.bands):
            start = band_idx * self.rows
            end = start + self.rows
            band_key = (band_idx, tuple(sig[start:end]))
            candidates.update(self.buckets.get(band_key, []))

        # 후보들 중에서 실제 코사인 유사도로 재랭킹
        results = []
        for item_id in candidates:
            candidate_vec = self.data[item_id]
            cos_sim = np.dot(vec, candidate_vec) / (
                np.linalg.norm(vec) * np.linalg.norm(candidate_vec) + 1e-10
            )
            results.append((item_id, cos_sim))

        results.sort(key=lambda x: -x[1])
        return results[:top_k]

# 사용 예제
dim = 128
lsh = SimHashLSH(dim=dim, num_hashes=64, bands=8)

rng = np.random.RandomState(0)
base_vec = rng.randn(dim)

# 유사 벡터 추가
for i in range(1000):
    noise = rng.randn(dim) * 0.1
    lsh.add(f"item_{i}", base_vec + noise)

# 완전히 다른 벡터들도 추가
for i in range(1000, 2000):
    lsh.add(f"item_{i}", rng.randn(dim))

query_vec = base_vec + rng.randn(dim) * 0.05
results = lsh.query(query_vec, top_k=5)
print("유사도 검색 결과:")
for item_id, score in results:
    print(f"  {item_id}: 코사인 유사도 = {score:.4f}")
```

## LSH의 오차 분석과 파라미터 튜닝

### False Positive / False Negative 트레이드오프

```
P_collision(밴드 k개, L개) = 1 - (1 - s^k)^L
```

- s: 두 항목의 실제 유사도 (Jaccard 등)
- k가 클수록: 오탐 감소, 미탐 증가
- L이 클수록: 미탐 감소, 오탐 증가, 메모리 증가

**S-커브(S-curve)**: 이 함수는 S자형 임계 함수(threshold function)를 형성한다. 적절한 k, L 선택으로 임계점을 원하는 유사도에 설정할 수 있다.

```python
import numpy as np
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt

def lsh_probability(s, k, L):
    """유사도 s인 두 항목이 후보 쌍이 될 확률"""
    return 1 - (1 - s**k)**L

s_values = np.linspace(0, 1, 100)

# 다양한 (k, L) 설정 비교
configs = [(3, 10), (5, 20), (10, 10), (2, 30)]

for k, L in configs:
    probs = [lsh_probability(s, k, L) for s in s_values]
    threshold = (1/L)**(1/k)
    print(f"k={k}, L={L}: 임계 유사도 ≈ {threshold:.3f}")
```

### 실전 파라미터 선택 가이드

| 목표 | 권장 설정 |
|------|-----------|
| 높은 정밀도(낮은 오탐) | k 크게, L 작게 |
| 높은 재현율(낮은 미탐) | k 작게, L 크게 |
| 균형 | k=5~10, L=10~20 |

## 실전 활용과 주요 라이브러리

### Python datasketch

```python
from datasketch import MinHash, MinHashLSH

# MinHash LSH with Jaccard threshold=0.5
lsh = MinHashLSH(threshold=0.5, num_perm=128)

def text_to_minhash(text, num_perm=128):
    m = MinHash(num_perm=num_perm)
    for word in text.lower().split():
        m.update(word.encode('utf8'))
    return m

texts = {
    "doc1": "deep learning neural network machine learning",
    "doc2": "neural network deep learning artificial intelligence",
    "doc3": "database sql query optimization index",
}

for doc_id, text in texts.items():
    m = text_to_minhash(text)
    lsh.insert(doc_id, m)

query = text_to_minhash("deep neural network learning")
result = lsh.query(query)
print(f"유사 문서: {result}")  # ['doc1', 'doc2']
```

### Faiss와 LSH 통합

Meta의 Faiss는 GPU 가속 LSH를 지원한다:

```python
import faiss
import numpy as np

d = 128      # 차원
n = 100000   # 데이터 수
nbits = 256  # LSH 비트 수

# LSH 인덱스 생성
index = faiss.IndexLSH(d, nbits)

# 데이터 추가
rng = np.random.RandomState(1)
data = rng.randn(n, d).astype('float32')
index.add(data)

# 쿼리
query = rng.randn(5, d).astype('float32')
k = 10
distances, indices = index.search(query, k)
print(f"상위 {k}개 이웃 인덱스:\n{indices}")
```

## 주의사항과 실전 팁

### 1. LSH vs HNSW 선택 기준

| 항목 | LSH | HNSW |
|------|-----|------|
| 메모리 | 낮음 (O(n)) | 높음 (그래프 저장) |
| 쿼리 속도 | 빠름 | 매우 빠름 |
| 정확도 | 파라미터 의존 | 높음 |
| 동적 삽입 | 쉬움 | 지원하지만 복잡 |
| 고차원 성능 | 좋음 | 매우 좋음 |

### 2. 거리 척도에 맞는 LSH 선택

- **Jaccard 유사도**: MinHash
- **코사인 유사도**: SimHash (랜덤 하이퍼플레인)
- **유클리드 거리**: Random Projection LSH (p-stable distribution)
- **Hamming 거리**: Bit-sampling LSH

### 3. 메모리와 속도 트레이드오프

```python
# 메모리를 아끼려면: 밴드 수(L) 줄이고 각 밴드 크기(k) 키우기
# → 정밀도 향상, 재현율 감소

# 속도를 높이려면: k 줄이기
# → 후보 수 증가하지만 개별 해시가 빠름

# 실전 권장: 먼저 L=20, k=5로 시작하고
# 정밀도가 낮으면 k 증가, 재현율이 낮으면 L 증가
```

### 4. 스트리밍 데이터 처리

LSH는 삽입이 O(L)로 매우 효율적이라 실시간 스트리밍 데이터에 적합하다. 단, 삭제 시에는 버킷에서 제거 로직이 필요하다.

## 마무리

LSH는 수십억 개의 고차원 벡터에서 유사도 검색을 실용적으로 만들어주는 핵심 알고리즘이다. MinHash는 문서/집합 유사도에, SimHash는 실수 벡터 유사도에 각각 강점을 가진다. 정확한 최근접 이웃이 필요하지 않고 속도가 중요한 상황, 메모리 제약이 있는 환경, 또는 동적으로 데이터가 추가되는 경우에 HNSW보다 LSH가 더 적합할 수 있다.

현대 벡터 데이터베이스(Pinecone, Weaviate, Faiss)는 LSH를 하나의 인덱싱 옵션으로 제공하며, 특히 수십억 규모의 데이터셋에서 메모리 효율적인 ANN 검색에 활발히 사용된다.

## 참고 자료

- [Locality-Sensitive Hashing — Wikipedia](https://en.wikipedia.org/wiki/Locality-sensitive_hashing)
- [MIT LSH Homepage — Andoni & Indyk](https://www.mit.edu/~andoni/LSH/)
- [Mastering Locality Sensitive Hashing — Zilliz Learn](https://zilliz.com/learn/mastering-locality-sensitive-hashing-a-comprehensive-tutorial)
- [datasketch: Big Data Looks Small — GitHub](https://github.com/ekzhu/datasketch)
