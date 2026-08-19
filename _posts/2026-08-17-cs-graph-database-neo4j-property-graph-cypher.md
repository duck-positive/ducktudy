---
layout: post
title: "그래프 데이터베이스 완전 정복: Neo4j, 프로퍼티 그래프 모델, Cypher 쿼리 내부 구조"
date: 2026-08-17
categories: [cs, computer-science]
tags: [graph-database, neo4j, cypher, property-graph, nosql, database, graph-theory]
---

소셜 네트워크에서 "친구의 친구가 좋아하는 식당"을 찾는 쿼리를 관계형 DB로 표현하면 어떻게 될까? `users`, `friendships`, `likes`, `restaurants` 테이블을 3~4번 JOIN해야 한다. 데이터가 수백만 건이라면 이 JOIN은 수 초, 수십 초가 걸린다.

그래프 데이터베이스는 이 문제를 근본적으로 다르게 접근한다. **연결 자체가 일급 시민(first-class citizen)**이다. 관계는 별도 테이블이 아닌, 노드 사이의 포인터로 직접 저장된다. 덕분에 관계를 따라가는 탐색이 JOIN 없이 이루어진다.

Neo4j는 세계에서 가장 널리 쓰이는 그래프 데이터베이스다. Cypher는 Neo4j의 선언적 쿼리 언어로, SQL처럼 "무엇을 원하는가"를 기술하면 엔진이 최적 경로를 찾는다.

---

## 왜 그래프 데이터베이스인가

### 관계형 DB의 JOIN 비용

관계형 모델에서 관계는 외래 키와 JOIN으로 표현된다. 관계가 깊어질수록 JOIN이 중첩된다:

```sql
-- 친구의 친구가 좋아하는 식당 (관계형 DB)
SELECT DISTINCT r.name
FROM users u1
JOIN friendships f1 ON u1.id = f1.user_id
JOIN users u2 ON f1.friend_id = u2.id
JOIN friendships f2 ON u2.id = f2.user_id
JOIN likes l ON f2.friend_id = l.user_id
JOIN restaurants r ON l.restaurant_id = r.id
WHERE u1.id = 1001;
```

4단계 JOIN이다. 각 단계에서 중간 결과셋이 생성되고, 해시 조인 또는 인덱스 스캔이 일어난다. 그래프가 깊어질수록 성능은 기하급수적으로 저하된다.

### 그래프 DB의 인덱스-프리 인접성 (Index-Free Adjacency)

그래프 DB는 각 노드가 인접 노드에 대한 **직접 포인터**를 저장한다. 탐색은 인덱스 조회 없이 포인터를 따라가는 것이므로, 홉(hop) 수에 비례하는 O(depth × fanout) 복잡도다. 전체 데이터 크기와 무관하다.

**인덱스-프리 인접성**: 노드 A → 노드 B의 관계를 따라가는 데 인덱스 조회가 필요 없다. 물리적으로 B의 주소가 A에 저장되어 있다. 관계형 DB의 인덱스 스캔이 `O(log N)` (B-트리)이라면, 그래프 DB의 인접 노드 접근은 `O(1)`에 가깝다.

---

## 프로퍼티 그래프 모델

Neo4j는 **프로퍼티 그래프 모델(Property Graph Model)**을 사용한다. 네 가지 요소로 구성된다:

### 1. 노드 (Node)
그래프의 개체(entity). 사람, 상품, 도시 등이 노드가 된다.

### 2. 레이블 (Label)
노드의 유형을 분류한다. `(:Person)`, `(:Movie)`, `(:City)`. 한 노드가 여러 레이블을 가질 수 있다 (`(:Person:Director)`).

### 3. 관계 (Relationship)
두 노드를 연결하는 방향 있는 간선. 반드시 **하나의 타입**과 **방향**을 가진다. `(a)-[:FRIENDS_WITH]->(b)`. 관계에도 프로퍼티를 붙일 수 있다 (예: 친구가 된 날짜).

### 4. 프로퍼티 (Property)
노드와 관계에 붙는 키-값 쌍. `{name: "Alice", age: 30}`.

---

## Cypher 쿼리 언어

Cypher는 패턴 매칭 기반 선언적 언어다. 핵심 문법은 ASCII 아트처럼 생겼다:

- `(n)`: 노드
- `(n:Label)`: 레이블이 붙은 노드
- `(n {key: value})`: 프로퍼티 조건
- `-[:TYPE]->`: 방향 있는 관계
- `-[:TYPE]-`: 방향 무관 관계

### 데이터 생성 예제

```cypher
// 노드 생성
CREATE (alice:Person {name: "Alice", age: 30, city: "Seoul"})
CREATE (bob:Person {name: "Bob", age: 28, city: "Busan"})
CREATE (carol:Person {name: "Carol", age: 32, city: "Seoul"})

CREATE (r1:Restaurant {name: "삼청동 수제비", category: "한식", rating: 4.8})
CREATE (r2:Restaurant {name: "망원동 파스타", category: "이탈리안", rating: 4.5})
CREATE (r3:Restaurant {name: "홍대 라멘", category: "일식", rating: 4.7})

// 관계 생성
MATCH (a:Person {name: "Alice"}), (b:Person {name: "Bob"})
CREATE (a)-[:FRIENDS_WITH {since: 2022}]->(b)

MATCH (a:Person {name: "Alice"}), (c:Person {name: "Carol"})
CREATE (a)-[:FRIENDS_WITH {since: 2021}]->(c)

MATCH (b:Person {name: "Bob"}), (r:Restaurant {name: "홍대 라멘"})
CREATE (b)-[:LIKES {visited: 5}]->(r)

MATCH (c:Person {name: "Carol"}), (r:Restaurant {name: "삼청동 수제비"})
CREATE (c)-[:LIKES {visited: 3}]->(r)

MATCH (c:Person {name: "Carol"}), (r:Restaurant {name: "망원동 파스타"})
CREATE (c)-[:LIKES {visited: 7}]->(r)
```

### 친구의 친구가 좋아하는 식당 추천

```cypher
// Alice의 친구들이 좋아하는 식당 (Alice가 아직 가지 않은 곳)
MATCH (me:Person {name: "Alice"})-[:FRIENDS_WITH]->(friend)-[:LIKES]->(restaurant:Restaurant)
WHERE NOT (me)-[:LIKES]->(restaurant)
RETURN restaurant.name AS 추천식당,
       restaurant.rating AS 평점,
       collect(friend.name) AS 추천인,
       count(*) AS 추천수
ORDER BY 추천수 DESC, 평점 DESC
LIMIT 10;
```

동등한 SQL 4-JOIN이 수 초 걸릴 수 있는 쿼리가, 그래프 DB에서는 포인터 추적으로 밀리초 단위에 처리된다.

### 최단 경로 탐색

```cypher
// Alice에서 특정 식당까지의 최단 소셜 경로
MATCH path = shortestPath(
  (alice:Person {name: "Alice"})-[*..6]-(target:Restaurant {name: "홍대 라멘"})
)
RETURN path, length(path) AS hops;
```

`[*..6]`은 최대 6홉 이내의 모든 관계를 탐색하라는 의미다. Neo4j는 내부적으로 BFS를 사용해 최단 경로를 찾는다.

### 패턴 기반 탐색 — 공통 친구

```cypher
// Alice와 Bob의 공통 친구
MATCH (alice:Person {name: "Alice"})-[:FRIENDS_WITH]-(common)-[:FRIENDS_WITH]-(bob:Person {name: "Bob"})
WHERE alice <> bob
RETURN common.name AS 공통친구;
```

---

## Neo4j 내부 구조

### 저장 레이아웃

Neo4j는 노드, 관계, 프로퍼티를 각각 다른 고정 크기 파일에 저장한다.

- **neostore.nodestore.db**: 노드 레코드. 9바이트 고정 크기. 첫 번째 관계 ID와 첫 번째 프로퍼티 ID를 포함.
- **neostore.relationshipstore.db**: 관계 레코드. 33바이트. 시작 노드, 끝 노드, 이전/다음 관계 ID를 포함하는 이중 연결 리스트.
- **neostore.propertystore.db**: 프로퍼티 레코드. 키-값 쌍.

노드 레코드가 관계 ID를 직접 포함하므로, 특정 노드의 관계 목록을 가져오는 것이 인덱스 탐색 없이 포인터 따라가기로 가능하다.

### 트랜잭션과 일관성

Neo4j는 ACID 트랜잭션을 지원한다. WAL(Write-Ahead Log) 기반 durability를 보장한다. 쓰기 시 먼저 로그에 기록한 뒤 실제 저장소를 업데이트한다.

```cypher
// 트랜잭션 예시 (드라이버 레벨)
BEGIN
MATCH (a:Account {id: "A1"}), (b:Account {id: "B1"})
SET a.balance = a.balance - 1000
SET b.balance = b.balance + 1000
COMMIT
```

---

## 실전 활용 패턴

### MERGE로 중복 방지

```cypher
// 없으면 생성, 있으면 매칭 (upsert 개념)
MERGE (u:Person {email: "alice@example.com"})
ON CREATE SET u.name = "Alice", u.created_at = datetime()
ON MATCH SET u.last_seen = datetime()
RETURN u;
```

`MERGE`는 `MATCH + CREATE`를 원자적으로 결합한다. 중복 노드 생성을 방지하는 핵심 패턴이다.

### 페이지랭크로 영향력 있는 노드 찾기

GDS(Graph Data Science) 라이브러리를 활용하면 PageRank, Betweenness Centrality 같은 그래프 알고리즘을 직접 실행할 수 있다:

```cypher
// GDS로 PageRank 실행
CALL gds.pageRank.stream('my-graph')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS name, score
ORDER BY score DESC
LIMIT 10;
```

---

## 그래프 DB 적합 사례 vs 부적합 사례

**적합한 사례:**
- 소셜 네트워크 (친구 추천, 영향력 분석)
- 추천 엔진 (협업 필터링)
- 사기 탐지 (비정상 거래 패턴 그래프 분석)
- 지식 그래프 (Google Knowledge Graph)
- 네트워크/인프라 의존성 분석
- 접근 제어 그래프 (RBAC, ABAC)

**부적합한 사례:**
- 단순 CRUD, 집계 보고서 중심 → 관계형 DB가 더 적합
- 대용량 시계열 데이터 → 시계열 DB가 적합
- 문서 중심 데이터 → 문서 DB가 적합
- 관계의 깊이가 1~2홉을 넘지 않는 데이터 → JOIN도 충분

---

## 주의사항 및 팁

**슈퍼 노드(Super Node) 문제**: 수백만 개의 관계를 가진 노드(예: 팔로워 수백만 명의 셀럽)는 그 노드를 경유하는 탐색에서 병목이 된다. 관계 타입을 더 세분화하거나, 시간/카테고리로 파티셔닝하라.

**인덱스는 진입점에만**: 그래프 탐색 자체는 인덱스가 불필요하지만, 탐색의 **시작 노드**를 찾는 데는 인덱스가 필요하다. `CREATE INDEX ON :Person(email)`처럼 쿼리 진입점 속성에 반드시 인덱스를 만들어라.

**관계 방향 설계**: 관계 방향은 의미론적으로 자연스럽게 설계해야 한다. Cypher는 `--`로 방향 무관 탐색을 지원하지만, 내부적으로는 양방향 탐색이라 성능이 절반으로 줄어든다.

**Neo4j 5.x Aura와 클러스터**: 운영 환경에서는 Causal Cluster를 사용해 읽기 부하를 레플리카로 분산하라. 쓰기는 항상 리더로, 읽기는 팔로워로 라우팅하는 bookmarks 메커니즘을 이해하면 일관성 문제를 피할 수 있다.

## 참고 자료
- [Neo4j Official Documentation](https://neo4j.com/docs/)
- [Cypher Manual — Introduction](https://neo4j.com/docs/cypher-manual/current/introduction/)
- [Graph database concepts — Neo4j Getting Started](https://neo4j.com/docs/getting-started/appendix/graphdb-concepts/)
- [Cypher (query language) — Wikipedia](https://en.wikipedia.org/wiki/Cypher_(query_language))
