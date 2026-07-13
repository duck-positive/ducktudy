---
layout: post
title: "HNSW 완전 정복: 벡터 데이터베이스의 심장, 근사 최근접 이웃 탐색 알고리즘"
date: 2026-07-13
categories: [cs, computer-science]
tags: [hnsw, vector-search, ann, nearest-neighbor, graph, embedding, vector-database]
---

벡터 데이터베이스가 AI 인프라의 핵심이 된 지금, 그 내부에서 수십억 개의 고차원 벡터를 밀리초 단위로 검색하게 해주는 알고리즘이 있다. 바로 **HNSW(Hierarchical Navigable Small World)**다. Pinecone, Weaviate, pgvector, Milvus 등 주요 벡터 데이터베이스가 기본 인덱스로 채택한 알고리즘으로, 2016년 Yu. A. Malkov와 D. A. Yashunin이 발표한 논문에서 시작됐다.

## 왜 ANN(근사 최근접 이웃)이 필요한가

임베딩 벡터로 표현된 텍스트나 이미지의 유사도를 찾는 가장 단순한 방법은 **전수 탐색(Brute-force)**이다. 쿼리 벡터와 데이터베이스 내 모든 벡터의 거리를 계산하고 가장 가까운 것을 반환한다. 1,000만 개의 1,536차원 벡터(OpenAI `text-embedding-3-small`)를 저장한다면, 쿼리 하나당 약 1,536 × 10,000,000 = **153억 번의 부동소수점 연산**이 필요하다. 레이턴시와 비용 모두 현실적이지 않다.

ANN(Approximate Nearest Neighbor) 알고리즘은 **"완벽하게 가장 가까운 벡터"가 아니라 "충분히 가까운 벡터"를 매우 빠르게 찾는** 방식으로 이 문제를 해결한다. Recall(재현율) 99%를 유지하면서도 브루트포스 대비 수십~수백 배 빠른 검색이 가능하다.

주요 ANN 알고리즘을 비교하면:
- **LSH (Locality-Sensitive Hashing)**: 해시 함수로 유사 벡터를 같은 버킷에 매핑. 차원의 저주에 취약.
- **FAISS IVF (Inverted File Index)**: k-means 클러스터링으로 공간을 분할, 후보 클러스터만 탐색. 클러스터 수 조정이 까다롭다.
- **HNSW**: 계층적 그래프 기반. 정확도, 속도, 인덱싱 시간 균형이 가장 우수해 현재 사실상 표준.

## HNSW의 핵심 아이디어: NSW + 계층 구조

### NSW (Navigable Small World)

HNSW의 기반인 NSW는 **소세계 현상(Small-world phenomenon)**에서 영감을 얻었다. 페이스북 이용자 중 임의의 두 명은 평균 3.57명의 지인을 거치면 연결된다는 개념이다. NSW 그래프는 이와 유사하게 구성된다.

- 각 노드(벡터)는 가까운 이웃들과 엣지로 연결된다.
- 장거리 연결(long-range links)이 존재해 그래프 전체를 빠르게 횡단할 수 있다.
- 쿼리 탐색 시 현재 노드에서 쿼리와 가장 가까운 이웃으로 탐욕적(greedy)으로 이동한다.

문제는 NSW의 탐색 복잡도가 **O(log N · log N)**으로, 데이터 규모가 커질수록 탐색 경로가 길어지고 초기 진입점(entry point)의 품질에 따라 정확도가 크게 흔들린다는 점이다.

### HNSW: 계층 구조로 NSW를 개선

HNSW는 NSW를 **다층 계층(hierarchy)**으로 쌓아 이 문제를 해결한다.

```
Layer 2 (최상위):  ●---------●              (소수의 노드, 장거리 링크)
Layer 1         :  ●---●---●---●            (중간 밀도)
Layer 0 (최하위):  ●-●-●-●-●-●-●-●-●-●    (모든 노드, 정밀 링크)
```

각 노드는 레이어 0에 반드시 포함되며, 확률 `exp(-level / mL)` 에 따라 상위 레이어에도 포함될 수 있다. 이 지수 분포 덕분에 상위 레이어로 갈수록 노드 수가 기하급수적으로 줄어든다. 이는 마치 지도의 축척처럼 동작한다 — 상위 레이어는 광역 지도, 하위 레이어는 상세 지도.

### 탐색 알고리즘

1. 가장 높은 레이어의 진입점(entry point)에서 시작
2. 현재 레이어에서 쿼리에 가장 가까운 노드를 탐욕적으로 탐색 (greedy search)
3. 더 이상 가까워질 수 없으면 한 레이어 아래로 내려감
4. 레이어 0에 도달할 때까지 반복, 레이어 0에서 최종 후보 `ef` 개를 수집
5. 수집된 후보 중 쿼리에 가장 가까운 `k` 개를 반환

탐색 복잡도는 **O(log N)**으로, 전수 탐색의 O(N)과 비교해 압도적이다.

## 핵심 파라미터

| 파라미터 | 의미 | 기본값 | 영향 |
|---|---|---|---|
| `M` | 각 노드가 유지하는 최대 연결 수 | 16 | 높을수록 정확도↑, 메모리↑ |
| `ef_construction` | 인덱스 구축 시 탐색 범위 | 200 | 높을수록 품질↑, 구축 시간↑ |
| `ef_search` | 쿼리 시 탐색 범위 | 50 | 높을수록 recall↑, 레이턴시↑ |
| `M0` | 레이어 0의 최대 연결 수 | `2 * M` | 레이어 0은 더 촘촘하게 |

## 실제 구현 예제

### 예제 1: hnswlib로 벡터 인덱스 구축 및 검색 (Python)

```python
import hnswlib
import numpy as np

DIM = 128       # 벡터 차원
N = 100_000     # 데이터셋 크기
K = 10          # 찾고자 하는 이웃 수

# 랜덤 벡터 데이터셋 생성 (실제로는 임베딩 모델 출력)
np.random.seed(42)
data = np.random.rand(N, DIM).astype(np.float32)

# 1. 인덱스 생성
index = hnswlib.Index(space='cosine', dim=DIM)
index.init_index(max_elements=N, ef_construction=200, M=16)

# 2. 벡터 삽입 (레이블은 0부터 N-1)
index.add_items(data, list(range(N)))

# 3. 탐색 파라미터 설정 (정확도 vs 속도 트레이드오프)
index.set_ef(50)  # ef_search

# 4. 쿼리 벡터로 k-NN 탐색
query = np.random.rand(1, DIM).astype(np.float32)
labels, distances = index.knn_query(query, k=K)

print(f"쿼리 결과 (코사인 유사도 기준 top-{K}):")
for i, (label, dist) in enumerate(zip(labels[0], distances[0])):
    similarity = 1 - dist  # cosine space는 distance = 1 - similarity
    print(f"  #{i+1}: 벡터 ID={label}, 유사도={similarity:.4f}")

# 5. 인덱스 저장 및 로드
index.save_index("hnsw_index.bin")

index2 = hnswlib.Index(space='cosine', dim=DIM)
index2.load_index("hnsw_index.bin", max_elements=N)
print(f"\n저장 후 로드된 인덱스 크기: {index2.element_count}개 벡터")
```

### 예제 2: HNSW 그래프 레이어 구조 시각화 (Python)

```python
import numpy as np
import math
from collections import defaultdict

class SimpleHNSW:
    """HNSW 핵심 로직을 직접 구현한 교육용 예제"""

    def __init__(self, M=4, ef_construction=20, m_L=None):
        self.M = M                         # 레이어 1+ 최대 연결 수
        self.M0 = 2 * M                    # 레이어 0 최대 연결 수
        self.ef_construction = ef_construction
        self.m_L = m_L or (1 / math.log(M))  # 레이어 정규화 인자
        self.graphs = defaultdict(dict)    # graphs[layer][node_id] = {neighbor_id: dist}
        self.data = []                     # 벡터 저장소
        self.entry_point = None
        self.max_layer = 0

    def _dist(self, a, b):
        """유클리드 거리"""
        return math.sqrt(sum((x - y) ** 2 for x, y in zip(a, b)))

    def _get_random_layer(self):
        """지수 분포로 삽입 레이어 결정"""
        return int(-math.log(np.random.uniform()) * self.m_L)

    def insert(self, vec):
        node_id = len(self.data)
        self.data.append(vec)
        insert_layer = self._get_random_layer()

        print(f"노드 {node_id} 삽입 → 레이어 0~{insert_layer}에 배치")

        if self.entry_point is None:
            self.entry_point = node_id
            self.max_layer = insert_layer
            for layer in range(insert_layer + 1):
                self.graphs[layer][node_id] = {}
            return

        # 상위 레이어에서 삽입 레이어까지 그리디 탐색으로 진입점 개선
        ep = self.entry_point
        for layer in range(self.max_layer, insert_layer, -1):
            ep = self._greedy_search(vec, ep, layer)

        # 삽입 레이어부터 0까지 연결 추가
        for layer in range(min(insert_layer, self.max_layer), -1, -1):
            M_at_layer = self.M0 if layer == 0 else self.M
            candidates = self._search_layer(vec, ep, self.ef_construction, layer)
            neighbors = self._select_neighbors(node_id, candidates, M_at_layer, layer)
            self.graphs[layer][node_id] = {}
            for nb in neighbors:
                dist = self._dist(vec, self.data[nb])
                self.graphs[layer][node_id][nb] = dist
                self.graphs[layer].setdefault(nb, {})[node_id] = dist
            if candidates:
                ep = min(candidates, key=lambda n: self._dist(vec, self.data[n]))

        if insert_layer > self.max_layer:
            self.max_layer = insert_layer
            self.entry_point = node_id
            for layer in range(self.max_layer, insert_layer + 1):
                self.graphs[layer][node_id] = {}

    def _greedy_search(self, query, ep, layer):
        best = ep
        best_dist = self._dist(query, self.data[ep])
        changed = True
        while changed:
            changed = False
            for nb in self.graphs[layer].get(best, {}):
                d = self._dist(query, self.data[nb])
                if d < best_dist:
                    best_dist, best, changed = d, nb, True
        return best

    def _search_layer(self, query, ep, ef, layer):
        visited = {ep}
        candidates = [ep]
        result = [ep]
        while candidates:
            c = min(candidates, key=lambda n: self._dist(query, self.data[n]))
            candidates.remove(c)
            f = max(result, key=lambda n: self._dist(query, self.data[n]))
            if self._dist(query, self.data[c]) > self._dist(query, self.data[f]):
                break
            for nb in self.graphs[layer].get(c, {}):
                if nb not in visited:
                    visited.add(nb)
                    candidates.append(nb)
                    result.append(nb)
                    if len(result) > ef:
                        worst = max(result, key=lambda n: self._dist(query, self.data[n]))
                        result.remove(worst)
        return result

    def _select_neighbors(self, node_id, candidates, M, layer):
        return sorted(candidates, key=lambda n: self._dist(self.data[node_id], self.data[n]))[:M]

    def print_structure(self):
        print(f"\n=== HNSW 그래프 구조 (최대 레이어: {self.max_layer}) ===")
        for layer in range(self.max_layer, -1, -1):
            nodes = list(self.graphs[layer].keys())
            print(f"Layer {layer}: {len(nodes)}개 노드 {nodes}")

# 사용 예시
np.random.seed(0)
hnsw = SimpleHNSW(M=2, ef_construction=5)
for i in range(8):
    vec = [np.random.uniform(0, 10), np.random.uniform(0, 10)]
    hnsw.insert(vec)

hnsw.print_structure()
```

## HNSW의 시간/공간 복잡도

| 연산 | 복잡도 |
|---|---|
| 삽입 | O(M · log N) |
| 탐색 | O(log N) |
| 공간 | O(N · M) |

공간 복잡도가 선형이지만, 전수 탐색의 O(N · D) 공간에 비해 D(차원) 대신 M(연결 수, 보통 16~64)이라 고차원에서 효율적이다.

## 주의사항 및 실전 팁

**파라미터 튜닝**
- `M`을 높이면 recall이 올라가지만 메모리 사용량이 선형 증가한다. 고정밀 환경에서는 M=32~64, 메모리 제약 환경에서는 M=8~16.
- `ef_search`는 런타임에 동적으로 조절 가능하다. 정밀도가 중요한 쿼리는 높게, 속도가 중요한 쿼리는 낮게 설정할 수 있다.
- `ef_construction`은 인덱스 빌드 품질에 영향. 한 번 결정하면 재구축 없이 변경 불가.

**삭제(Delete)의 어려움**
HNSW는 그래프 기반이라 노드 삭제가 복잡하다. 대부분의 구현체는 **soft delete** 방식(삭제 플래그만 표시)을 사용한다. 삭제가 많으면 성능이 저하되므로, 주기적인 인덱스 재구축(rebuild)이 권장된다.

**동시성 문제**
hnswlib는 멀티스레드 읽기를 지원하지만, 쓰기는 별도 동기화가 필요하다. 프로덕션에서 실시간 삽입이 필요하면 Weaviate나 Milvus 같은 벡터 DB의 동시성 관리 기능을 활용하는 것이 안전하다.

**필터링과의 조합**
메타데이터 필터(예: `category = 'AI' AND created_at > 2025-01-01`)와 ANN을 조합할 때 주의가 필요하다. 전체 결과를 받은 뒤 필터링(post-filtering)하면 top-k가 부족할 수 있다. HNSW 탐색 중 필터를 적용하는 **pre-filtering** 방식이 정확하지만 구현이 복잡하다. 대부분의 벡터 DB는 두 방식을 혼합 사용한다.

## 참고 자료
- [Hierarchical Navigable Small World - Wikipedia](https://en.wikipedia.org/wiki/Hierarchical_navigable_small_world)
- [Pinecone: HNSW 상세 해설](https://www.pinecone.io/learn/series/faiss/hnsw/)
- [Milvus HNSW Index Documentation](https://milvus.io/docs/hnsw.md)
- [원본 논문: Efficient and robust approximate nearest neighbor search using HNSW (arXiv:1603.09320)](https://arxiv.org/abs/1603.09320)
