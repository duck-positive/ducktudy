---
layout: post
title: "MinHash와 SimHash: 웹 규모 문서 유사도 탐지의 수학적 원리"
date: 2026-10-21
categories: [cs, computer-science]
tags: [minhash, simhash, lsh, near-duplicate, jaccard-similarity, hashing, probabilistic]
---

# MinHash와 SimHash: 웹 규모 문서 유사도 탐지의 수학적 원리

수십억 개의 웹 페이지 중 중복 콘텐츠를 찾는 것, 수백만 개의 코드 저장소에서 유사 코드를 탐지하는 것, 실시간으로 뉴스 기사의 중복 여부를 판단하는 것 — 이 모든 문제는 **대규모 문서 유사도 탐지**로 귀결된다. 

두 문서를 문자 단위로 비교하면 O(N) 비용이 들고, 수십억 쌍을 비교하면 불가능해진다. Google은 이 문제를 해결하기 위해 **MinHash**와 **SimHash**를 사용한다. 두 알고리즘은 서로 다른 수학적 토대 위에서 작동하지만, 공통된 목표를 가진다: **소형 스케치(sketch)**로 문서를 압축하여 유사도를 빠르게 근사 추정한다.

## 왜 필요한가?

### Jaccard 유사도의 직접 계산의 한계

두 집합 $A$, $B$의 **Jaccard 유사도**는 다음과 같이 정의된다:

$$J(A, B) = \frac{|A \cap B|}{|A \cup B|}$$

이 값이 1에 가까울수록 두 집합은 유사하다. 문서를 **shingle(n-gram)** 집합으로 변환하면 Jaccard 유사도로 문서 유사성을 측정할 수 있다.

예: "the cat sat on the mat"
- 2-gram shingles: {"the cat", "cat sat", "sat on", "on the", "the mat"}

그러나 수십억 문서 쌍에 대해 교집합·합집합을 직접 계산하는 건 불가능하다. MinHash는 이 비용을 획기적으로 줄인다.

### 웹 크롤링에서의 실전 문제

- 웹에는 동일 콘텐츠가 수천 개 URL에 존재한다(미러, 신디케이션)
- 검색 엔진은 이를 필터링해야 한다
- 하루 수십억 개의 새 URL이 등장한다

## MinHash: Jaccard 유사도의 확률적 추정

### 핵심 원리

MinHash의 천재적 발상은 다음 등가 관계에서 출발한다:

**두 집합 $A$, $B$에서 랜덤 해시 함수 $h$를 적용할 때, $\min_{x \in A}h(x) = \min_{x \in B}h(x)$일 확률은 정확히 $J(A, B)$이다.**

증명: 합집합 $A \cup B$의 원소 중 최솟값을 가진 원소 $m$에 대해, $m \in A \cap B$이면 두 최솟값이 같고, $m \notin A \cap B$이면 다르다. 따라서 확률 = $|A \cap B| / |A \cup B| = J(A, B)$.

이 원리를 **$k$개**의 독립적인 해시 함수에 적용하면, 각 함수마다 최솟값을 구해 $k$개의 서명(signature)을 만들고, 두 서명이 일치하는 비율로 Jaccard 유사도를 추정할 수 있다.

### LSH(Locality-Sensitive Hashing)와의 결합

$k$개의 해시 함수를 $b$개의 밴드로 나눠 각 밴드 내의 서명이 모두 일치하는 쌍을 후보 쌍으로 선택하는 기법이 **MinHash LSH**다.

- **밴드 수 $b$**, **밴드당 해시 수 $r = k/b$**
- 두 문서의 Jaccard 유사도가 $s$일 때, 적어도 한 밴드에서 일치할 확률:
$$P(\text{후보}) = 1 - (1 - s^r)^b$$

이 S-커브를 이용해 임계값을 설정한다.

## 실제 구현 예제: MinHash

### 예제 1: Python MinHash 구현

```python
import hashlib
import random
from collections import defaultdict

def get_shingles(text: str, k: int = 3) -> set:
    """k-gram 쉰글 집합 생성"""
    words = text.lower().split()
    return {' '.join(words[i:i+k]) for i in range(len(words) - k + 1)}

class MinHasher:
    def __init__(self, num_hashes: int = 128):
        self.num_hashes = num_hashes
        # 랜덤 계수 a, b 생성 (해시 함수: h(x) = (a*x + b) % p)
        _prime = (1 << 61) - 1  # 메르센 소수
        self.a = [random.randint(1, _prime - 1) for _ in range(num_hashes)]
        self.b = [random.randint(0, _prime - 1) for _ in range(num_hashes)]
        self._p = _prime

    def _hash(self, value: int, i: int) -> int:
        return (self.a[i] * value + self.b[i]) % self._p

    def signature(self, shingles: set) -> list:
        """문서의 MinHash 서명 계산"""
        sig = [float('inf')] * self.num_hashes
        for shingle in shingles:
            # 쉰글을 정수로 변환
            h = int(hashlib.md5(shingle.encode()).hexdigest(), 16)
            for i in range(self.num_hashes):
                val = self._hash(h, i)
                if val < sig[i]:
                    sig[i] = val
        return sig

    def jaccard_estimate(self, sig1: list, sig2: list) -> float:
        """두 서명의 Jaccard 유사도 추정"""
        matches = sum(1 for a, b in zip(sig1, sig2) if a == b)
        return matches / self.num_hashes

class MinHashLSH:
    def __init__(self, num_hashes: int = 128, bands: int = 16):
        self.hasher = MinHasher(num_hashes)
        self.bands = bands
        self.rows = num_hashes // bands
        self.buckets: dict = defaultdict(list)

    def add_document(self, doc_id: str, text: str):
        shingles = get_shingles(text)
        sig = self.hasher.signature(shingles)
        # 각 밴드에서 버킷 해시
        for band_idx in range(self.bands):
            start = band_idx * self.rows
            band = tuple(sig[start:start + self.rows])
            bucket_key = (band_idx, hash(band))
            self.buckets[bucket_key].append((doc_id, sig))

    def find_candidates(self, text: str) -> list:
        shingles = get_shingles(text)
        sig = self.hasher.signature(shingles)
        candidates = set()
        for band_idx in range(self.bands):
            start = band_idx * self.rows
            band = tuple(sig[start:start + self.rows])
            bucket_key = (band_idx, hash(band))
            for (doc_id, _) in self.buckets[bucket_key]:
                candidates.add(doc_id)
        return list(candidates)

# 사용 예시
lsh = MinHashLSH(num_hashes=128, bands=16)
docs = [
    ("doc1", "the quick brown fox jumps over the lazy dog"),
    ("doc2", "the quick brown fox jumped over the lazy dog"),  # 거의 동일
    ("doc3", "machine learning is a subset of artificial intelligence"),
    ("doc4", "deep learning is a type of machine learning"),
]
for doc_id, text in docs:
    lsh.add_document(doc_id, text)

query = "the quick brown fox leaps over the lazy dog"
candidates = lsh.find_candidates(query)
print("유사 문서 후보:", candidates)  # doc1, doc2 발견

# 정확한 유사도 측정
hasher = MinHasher(128)
query_sig = hasher.signature(get_shingles(query))
for doc_id, text in docs:
    sig = hasher.signature(get_shingles(text))
    est = hasher.jaccard_estimate(query_sig, sig)
    print(f"{doc_id}: 추정 Jaccard = {est:.3f}")
```

## SimHash: 해밍 거리로 측정하는 유사도

### 핵심 원리

**SimHash**(Charikar, 2002)는 MinHash와는 다른 접근법을 취한다. 문서의 **가중 특징 벡터**를 **$b$비트 해시값 하나**로 압축한다.

**SimHash 생성 알고리즘**:
1. 문서의 각 특징(단어 또는 쉰글) $f$와 가중치 $w(f)$에 대해
2. 특징의 해시값 $h(f)$를 $b$비트 이진수로 변환
3. 각 비트 위치 $i$에 대해: $h(f)$의 $i$번째 비트가 1이면 $V[i] += w(f)$, 0이면 $V[i] -= w(f)$
4. 최종 SimHash의 $i$번째 비트: $V[i] > 0$이면 1, 아니면 0

두 SimHash 간의 **해밍 거리**가 가까울수록 두 문서는 유사하다.

### 예제 2: SimHash 구현

```python
import hashlib
from collections import Counter

def simhash(text: str, num_bits: int = 64) -> int:
    """텍스트의 SimHash 계산"""
    # TF 가중치를 사용한 단어 빈도
    words = text.lower().split()
    weights = Counter(words)
    
    v = [0.0] * num_bits
    
    for word, weight in weights.items():
        # 단어를 num_bits 해시로 변환
        h = int(hashlib.md5(word.encode()).hexdigest(), 16)
        # 하위 num_bits 비트만 사용
        for i in range(num_bits):
            bit = (h >> i) & 1
            if bit:
                v[i] += weight
            else:
                v[i] -= weight
    
    # 부호에 따라 0 또는 1
    fingerprint = 0
    for i in range(num_bits):
        if v[i] > 0:
            fingerprint |= (1 << i)
    return fingerprint

def hamming_distance(h1: int, h2: int) -> int:
    """두 정수의 해밍 거리 (XOR의 비트 수)"""
    xor = h1 ^ h2
    return bin(xor).count('1')

def simhash_similarity(h1: int, h2: int, num_bits: int = 64) -> float:
    """SimHash 기반 유사도 [0, 1]"""
    dist = hamming_distance(h1, h2)
    return 1 - dist / num_bits

# 대규모 중복 탐지 시스템
class SimHashDeduplicator:
    def __init__(self, max_hamming: int = 3, num_bits: int = 64):
        self.max_hamming = max_hamming
        self.num_bits = num_bits
        self.fingerprints: list = []  # (fingerprint, doc_id)

    def add_document(self, doc_id: str, text: str) -> bool:
        """문서 추가. 중복이면 False 반환."""
        fp = simhash(text, self.num_bits)
        for existing_fp, existing_id in self.fingerprints:
            dist = hamming_distance(fp, existing_fp)
            if dist <= self.max_hamming:
                print(f"중복 감지: '{doc_id}' ≈ '{existing_id}' (해밍 거리={dist})")
                return False
        self.fingerprints.append((fp, doc_id))
        return True

# 사용 예시
dedup = SimHashDeduplicator(max_hamming=5)
texts = [
    ("news1", "Apple announces new iPhone with improved camera and battery life"),
    ("news2", "Apple unveils new iPhone featuring better camera and longer battery"),  # 거의 동일
    ("news3", "Samsung releases Galaxy S with OLED display and fast charging"),
    ("news4", "Apple reveals latest iPhone with enhanced photography and battery"),  # 유사
]
for doc_id, text in texts:
    is_new = dedup.add_document(doc_id, text)
    print(f"{'추가됨' if is_new else '중복 필터'}: {doc_id}")

# 두 해시의 유사도 비교
h1 = simhash("machine learning for natural language processing")
h2 = simhash("deep learning techniques in natural language understanding")
print(f"유사도: {simhash_similarity(h1, h2):.3f}")
print(f"해밍 거리: {hamming_distance(h1, h2)}")
```

## MinHash vs SimHash: 언제 무엇을 쓸까?

| 특성 | MinHash | SimHash |
|------|---------|---------|
| 측정 기준 | Jaccard 유사도 | 코사인 유사도(근사) |
| 서명 크기 | $k \times 32$비트 | 단 64비트 |
| 적합한 유사도 임계값 | 낮은~중간 (50~70%) | 높은 유사도 (90%+) |
| 특징 가중치 지원 | 제한적 | 기본 지원 |
| LSH 통합 | 밴드/행 기법 | 비트 샘플링 기법 |
| 대표 적용 | 웹 중복 페이지, 표절 검사 | 뉴스 중복, 코드 클론 |

**Google의 선택**: 웹 크롤 중복 탐지에는 SimHash(64비트)를 사용한다. 수십억 문서의 지문을 저장하고 Hamming 거리 3 이하이면 중복으로 판정한다.

## 주의사항과 팁

### MinHash 구현의 함정

- **해시 함수 독립성**: 서명 품질은 해시 함수의 독립성에 의존한다. 실무에서는 LCG 해시보다 MurmurHash나 xxHash를 권장한다.
- **쉰글 크기**: k가 너무 작으면 우연한 일치가 늘고, 너무 크면 희소해진다. 보통 3~5 단어 쉰글이 적절하다.

### SimHash의 한계

- **짧은 문서**: 텍스트가 10단어 미만이면 SimHash의 신뢰도가 떨어진다.
- **언어 다양성**: 동일 내용의 다국어 버전은 SimHash로 탐지 불가. MinHash + 번역 파이프라인 필요.

### 오탐(False Positive) 제어

MinHash LSH에서 밴드 수와 행 수를 조정하면 오탐률과 미탐률 사이의 트레이드오프를 제어할 수 있다. 정밀도보다 재현율이 중요한 시스템(예: 표절 탐지)에서는 밴드 수를 늘리는 게 유리하다.

### 실무 최적화

- **온라인 업데이트**: SimHash는 새 문서를 O(단어 수)에 추가할 수 있다.
- **샤딩**: 수십억 개의 지문은 해시 기반 샤딩으로 분산 저장한다.
- **근사 해밍 검색**: 64비트 SimHash에서 해밍 거리 $k$ 이하 검색은 비트 분할 인덱싱으로 O(1)~O(k)에 가능하다.

## 참고 자료
- [Wikipedia: MinHash](https://en.wikipedia.org/wiki/MinHash)
- [Wikipedia: SimHash](https://en.wikipedia.org/wiki/SimHash)
- [Wikipedia: Locality-Sensitive Hashing](https://en.wikipedia.org/wiki/Locality-sensitive_hashing)
- [Spot Intelligence: SimHash Guide](https://spotintelligence.com/2023/01/02/simhash/)
