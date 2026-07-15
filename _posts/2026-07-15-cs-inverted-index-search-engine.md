---
layout: post
title: "역색인(Inverted Index) 완전 정복: 검색 엔진이 0.1초 만에 수십억 문서를 찾는 비밀"
date: 2026-07-15
categories: [cs, computer-science]
tags: [inverted-index, search-engine, lucene, elasticsearch, information-retrieval, tfidf, bm25]
---

## 개념 설명

역색인(Inverted Index)은 정보 검색 시스템의 핵심 자료구조로, 문서에서 단어를 찾는 대신 **단어에서 문서를 찾는** 방향으로 색인을 구축한다. 책 뒷면의 '찾아보기' 페이지를 상상하면 이해하기 쉽다. "운영체제"라는 단어가 42쪽, 87쪽, 203쪽에 나온다는 정보를 저장해두고, 검색 시에는 그 목록을 바로 조회하는 것이다.

역색인은 두 가지 핵심 구성 요소로 이루어진다:

1. **단어 사전(Term Dictionary)**: 색인에 존재하는 모든 고유 단어(Term)의 목록. B-Tree 또는 FST(Finite State Transducer)로 구성해 빠른 검색을 지원한다.
2. **포스팅 목록(Posting List)**: 각 단어가 등장하는 문서 ID 목록. 추가 정보(위치, 빈도 등)도 함께 저장할 수 있다.

```
문서 1: "데이터베이스 인덱스 성능 최적화"
문서 2: "데이터베이스 트랜잭션 격리 수준"
문서 3: "인덱스 자료구조 B-Tree와 LSM-Tree"

역색인 구축 결과:
┌─────────────────┬──────────────────────────────────────┐
│ Term            │ Posting List                         │
├─────────────────┼──────────────────────────────────────┤
│ 데이터베이스    │ [doc1(tf=1), doc2(tf=1)]             │
│ 인덱스          │ [doc1(tf=1,pos=2), doc3(tf=1,pos=1)] │
│ 성능            │ [doc1(tf=1)]                         │
│ 최적화          │ [doc1(tf=1)]                         │
│ 트랜잭션        │ [doc2(tf=1)]                         │
│ 격리            │ [doc2(tf=1)]                         │
│ 자료구조        │ [doc3(tf=1)]                         │
│ B-Tree          │ [doc3(tf=1)]                         │
│ LSM-Tree        │ [doc3(tf=1)]                         │
└─────────────────┴──────────────────────────────────────┘
```

## 왜 필요한가

전방 색인(Forward Index)은 문서별로 포함된 단어 목록을 저장한다. 특정 단어를 검색하려면 모든 문서를 순차 스캔해야 하므로 O(N) 시간이 걸린다. 문서 수가 수십억 개라면 이 방식은 현실적으로 불가능하다.

역색인을 사용하면 단어를 키로 직접 조회(O(log N) 또는 O(1))해 후보 문서 목록을 즉시 가져올 수 있다. "데이터베이스 AND 인덱스"를 검색한다면:

```
데이터베이스 → [doc1, doc2]
인덱스       → [doc1, doc3]
교집합 연산  → [doc1]
```

두 포스팅 목록의 교집합만 구하면 되므로, 전체 문서를 읽지 않고도 O(k) 시간(k: 후보 문서 수)에 결과를 반환한다.

## 실제 구현 예제

### 예제 1: Python으로 역색인 직접 구현

```python
import re
import math
from collections import defaultdict
from typing import Dict, List, Tuple

class InvertedIndex:
    def __init__(self):
        # term -> [(doc_id, tf, [positions])]
        self.index: Dict[str, List[Tuple]] = defaultdict(list)
        self.doc_store: Dict[int, str] = {}
        self.doc_lengths: Dict[int, int] = {}
        self.doc_count = 0

    def _tokenize(self, text: str) -> List[str]:
        """간단한 토크나이저: 소문자 변환 + 단어 분리"""
        return re.findall(r'\b[a-z가-힣]+\b', text.lower())

    def add_document(self, doc_id: int, text: str):
        self.doc_store[doc_id] = text
        tokens = self._tokenize(text)
        self.doc_lengths[doc_id] = len(tokens)
        self.doc_count += 1

        # 단어별 위치와 빈도 계산
        term_positions: Dict[str, List[int]] = defaultdict(list)
        for pos, token in enumerate(tokens):
            term_positions[token].append(pos)

        for term, positions in term_positions.items():
            tf = len(positions) / len(tokens)  # 정규화된 TF
            self.index[term].append((doc_id, tf, positions))

    def search_and(self, *terms: str) -> List[int]:
        """AND 검색: 모든 단어가 포함된 문서 반환"""
        if not terms:
            return []

        # 포스팅 목록을 크기 순으로 정렬 (작은 것부터 교집합 처리)
        postings = sorted(
            [set(doc_id for doc_id, _, _ in self.index.get(t, []))
             for t in terms],
            key=len
        )
        if not postings:
            return []

        result = postings[0]
        for posting in postings[1:]:
            result = result & posting
        return sorted(result)

    def bm25_score(self, doc_id: int, term: str,
                   k1: float = 1.5, b: float = 0.75) -> float:
        """BM25 관련성 점수 계산"""
        postings = self.index.get(term, [])
        posting_map = {d: (tf, pos) for d, tf, pos in postings}

        if doc_id not in posting_map:
            return 0.0

        # IDF: 해당 단어가 포함된 문서가 적을수록 희귀하고 중요함
        df = len(postings)  # Document Frequency
        idf = math.log((self.doc_count - df + 0.5) / (df + 0.5) + 1)

        # TF 정규화: 문서 길이에 따른 보정
        tf_raw = posting_map[doc_id][0] * self.doc_lengths[doc_id]
        avg_dl = sum(self.doc_lengths.values()) / max(len(self.doc_lengths), 1)
        dl = self.doc_lengths[doc_id]
        tf_norm = (tf_raw * (k1 + 1)) / (tf_raw + k1 * (1 - b + b * dl / avg_dl))

        return idf * tf_norm

    def ranked_search(self, query: str) -> List[Tuple[int, float]]:
        """BM25 점수 기반 랭킹 검색"""
        terms = self._tokenize(query)
        scores: Dict[int, float] = defaultdict(float)

        for term in terms:
            for doc_id, _, _ in self.index.get(term, []):
                scores[doc_id] += self.bm25_score(doc_id, term)

        return sorted(scores.items(), key=lambda x: x[1], reverse=True)


# 사용 예시
idx = InvertedIndex()
idx.add_document(1, "데이터베이스 인덱스 성능 최적화 방법")
idx.add_document(2, "데이터베이스 트랜잭션 격리 수준 이해")
idx.add_document(3, "인덱스 자료구조 B-Tree와 LSM-Tree 비교")
idx.add_document(4, "검색 엔진 인덱스 최적화와 성능 튜닝")

# AND 검색
result = idx.search_and("인덱스", "성능")
print(f"'인덱스 AND 성능' 결과: {result}")  # [1, 4]

# BM25 랭킹 검색
ranked = idx.ranked_search("인덱스 최적화")
print("BM25 랭킹 결과:")
for doc_id, score in ranked:
    print(f"  doc{doc_id}: {score:.3f} - {idx.doc_store[doc_id][:20]}...")
```

### 예제 2: Lucene 스타일의 세그먼트 기반 역색인 구조

Apache Lucene은 역색인을 **불변 세그먼트(Immutable Segment)** 단위로 관리하며, 백그라운드 병합(Merge)으로 세그먼트를 통합한다. 이 구조를 간단히 시뮬레이션해보자.

```python
import heapq
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class Segment:
    """불변 세그먼트: 한 번 생성 후 수정 불가"""
    segment_id: int
    index: Dict[str, List[int]]  # term -> sorted doc_id list
    deleted: set = field(default_factory=set)  # 삭제된 문서 ID (라이브 필터링)
    doc_count: int = 0

    def search(self, term: str) -> List[int]:
        return [d for d in self.index.get(term, []) if d not in self.deleted]

    def delete(self, doc_id: int):
        """삭제는 즉시 인덱스에서 제거하지 않고, 삭제 마킹만 한다"""
        self.deleted.add(doc_id)

    @property
    def live_count(self):
        return self.doc_count - len(self.deleted)


class LuceneStyleIndex:
    """Lucene 스타일의 세그먼트 기반 역색인"""
    def __init__(self, segment_buffer_size: int = 3):
        self.segments: List[Segment] = []
        self.buffer: Dict[str, List[int]] = defaultdict(list)
        self.buffer_doc_count = 0
        self.buffer_size = segment_buffer_size
        self._seg_id = 0
        self._doc_id = 0

    def add_document(self, text: str) -> int:
        doc_id = self._doc_id
        self._doc_id += 1
        tokens = set(re.findall(r'\b[a-z가-힣]+\b', text.lower()))

        for token in tokens:
            self.buffer[token].append(doc_id)
        self.buffer_doc_count += 1

        # 버퍼가 가득 차면 새 세그먼트로 플러시
        if self.buffer_doc_count >= self.buffer_size:
            self._flush()
        return doc_id

    def _flush(self):
        """인메모리 버퍼를 불변 세그먼트로 변환"""
        if not self.buffer:
            return
        seg = Segment(
            segment_id=self._seg_id,
            index=dict(self.buffer),
            doc_count=self.buffer_doc_count
        )
        self.segments.append(seg)
        self._seg_id += 1
        self.buffer = defaultdict(list)
        self.buffer_doc_count = 0
        print(f"  [Flush] 세그먼트 {seg.segment_id} 생성 (docs={seg.doc_count})")

        # 세그먼트가 많아지면 병합 (실제 Lucene은 티어드 병합 정책 사용)
        if len(self.segments) >= 4:
            self._merge_all()

    def _merge_all(self):
        """모든 세그먼트를 하나로 병합 (삭제된 문서 제거)"""
        merged_index: Dict[str, List[int]] = defaultdict(list)
        total_docs = 0
        for seg in self.segments:
            for term, doc_ids in seg.index.items():
                live_ids = [d for d in doc_ids if d not in seg.deleted]
                merged_index[term].extend(live_ids)
            total_docs += seg.live_count

        for term in merged_index:
            merged_index[term] = sorted(set(merged_index[term]))

        merged_seg = Segment(
            segment_id=self._seg_id,
            index=dict(merged_index),
            doc_count=total_docs
        )
        self.segments = [merged_seg]
        self._seg_id += 1
        print(f"  [Merge] 세그먼트 병합 완료 → 세그먼트 {merged_seg.segment_id} (docs={total_docs})")

    def search(self, term: str) -> List[int]:
        """모든 세그먼트에서 결과를 수집해 통합"""
        self._flush()  # 버퍼의 미플러시 데이터 포함
        result = set()
        for seg in self.segments:
            result.update(seg.search(term))
        # 버퍼도 검색
        if term in self.buffer:
            result.update(self.buffer[term])
        return sorted(result)

    def delete(self, doc_id: int):
        for seg in self.segments:
            if any(doc_id in ids for ids in seg.index.values()):
                seg.delete(doc_id)


# 사용 예시
lsi = LuceneStyleIndex(segment_buffer_size=2)
d1 = lsi.add_document("elasticsearch inverted index segment merge")
d2 = lsi.add_document("lucene segment immutable flush")
d3 = lsi.add_document("inverted index posting list compression")
d4 = lsi.add_document("search engine ranking bm25 tfidf")
d5 = lsi.add_document("lucene query parser boolean search")

print(f"\n'inverted' 검색 결과: {lsi.search('inverted')}")
print(f"'lucene' 검색 결과: {lsi.search('lucene')}")

# 삭제 후 재검색
lsi.delete(d1)
print(f"\nd1 삭제 후 'inverted' 검색 결과: {lsi.search('inverted')}")
```

## Lucene의 고급 최적화 기법

### FST (Finite State Transducer) 기반 단어 사전

Lucene은 단어 사전을 FST로 구현한다. Trie에서 파생된 FST는 공통 접두사와 접미사를 공유해 메모리를 극적으로 줄인다. 수백만 단어로 구성된 사전을 수 MB에 저장할 수 있는 이유가 바로 FST다.

```
단어 집합: {"cat", "cats", "dog", "dogs"}

일반 Trie:
root → c → a → t → [EOF]
                  → s → [EOF]
       d → o → g → [EOF]
                  → s → [EOF]

FST (접미사 공유):
공통 접미사 's'를 공유하는 상태로 압축
→ 메모리 절약, 탐색 속도 유지
```

### SIMD를 활용한 포스팅 목록 교집합

대형 포스팅 목록의 교집합을 구할 때 Lucene은 SIMD 명령어를 활용한 벡터화된 비트셋 연산을 수행한다. 앞서 살펴본 Roaring Bitmap이 여기서 활용된다.

### 단어 위치(Phrase Query)와 TF 정보 저장

포스팅 목록에 단어 위치를 저장하면 구문 검색(Phrase Query)이 가능해진다. `"데이터베이스 인덱스"` 검색 시 두 단어가 연속으로 등장하는 문서만 반환할 수 있다.

## 주의사항 및 팁

### 1. 분석기(Analyzer) 선택의 중요성
역색인 품질은 토크나이저와 필터 파이프라인에 달려 있다. 한국어의 경우 형태소 분석기(Nori, Arirang 등)를 사용해야 "먹었다" → "먹" 같은 원형 복원이 이루어진다. 검색 시 쿼리도 동일한 분석기를 거쳐야 인덱스와 일치한다.

### 2. 세그먼트 수와 검색 성능 트레이드오프
세그먼트가 많을수록 검색 시 여러 세그먼트를 병렬 탐색하므로 메모리 사용이 증가한다. `forceMerge(1)`로 단일 세그먼트로 통합하면 검색이 빨라지지만, 병합 비용이 크므로 쓰기가 없는 시점(새벽 배치)에 수행하는 것이 좋다.

### 3. `_source` vs 역색인 저장
Elasticsearch의 `_source`는 원본 문서를 JSON으로 저장하지만 검색에는 사용되지 않는다. 실제 검색은 역색인을 통해 이루어진다. `_source: false`를 설정하면 저장 공간을 줄일 수 있지만, 도큐먼트 내용을 반환받을 수 없어진다.

### 4. Near Real-Time(NRT) 검색
Lucene의 기본 세그먼트 플러시 주기는 1초다. 플러시 전 데이터는 인메모리 세그먼트에 존재해 검색에 포함되므로, 정확한 의미의 실시간(즉시)은 아니지만 '근 실시간(NRT)' 검색이 가능하다.

## 참고 자료
- [apache/lucene - GitHub](https://github.com/apache/lucene)
- [elastic/elasticsearch - GitHub](https://github.com/elastic/elasticsearch)
- [CMU 15-445/645 Database Systems — Information Retrieval](https://15445.courses.cs.cmu.edu/fall2023/)
- [Introduction to Information Retrieval (Stanford textbook)](https://nlp.stanford.edu/IR-book/html/htmledition/irbook.html)
