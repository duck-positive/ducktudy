---
layout: post
title: "Datalog와 선언형 데이터베이스 완전 정복: 규칙 기반 추론 엔진이 재귀 쿼리와 그래프 분석을 푸는 방법"
date: 2026-09-24
categories: [cs, computer-science]
tags: [datalog, declarative-programming, recursive-queries, deductive-database, logic-programming, datomic, soufflé]
---

SQL은 강력하지만, 재귀적 관계(계층 구조, 그래프 경로, 전이 폐포)를 표현할 때는 불편하다. 계층 쿼리(`WITH RECURSIVE`)는 장황하고, 고정 깊이 조인은 한계가 있다. **Datalog**는 이 문제를 선언적 규칙(rule)으로 우아하게 해결한다. Prolog의 사촌이면서도 순수 선언형 특성 덕분에 정적 분석 도구(Doop, Soufflé), 분산 데이터 시스템(Datomic, Cascalog), 네트워크 설정 관리 등에 폭넓게 쓰인다. 이 아티클에서는 Datalog의 문법, 평가 모델, 재귀 고정점 계산, 그리고 실제 구현까지 완전히 다룬다.

---

## 1. Datalog란 무엇인가?

Datalog는 1970년대에 관계형 데이터베이스 이론에서 파생된 **논리 프로그래밍 기반 쿼리 언어**다. SQL처럼 데이터를 질의하지만, 핵심 차이는:

- **재귀 규칙**: 규칙이 자기 자신을 참조할 수 있다 → 전이 폐포(transitive closure) 자연스럽게 표현
- **단조성(Monotonicity)**: 새 사실을 추가하면 파생 사실은 늘어나기만 한다 → 고정점 계산이 항상 종료
- **바텀업 평가**: Prolog의 탑다운(깊이 우선 탐색) 대신 바텀업(반복적 고정점)으로 평가 → 무한 루프 없음

### Datalog 문법 기본

```prolog
% Fact (사실): 부모-자식 관계
parent(alice, bob).
parent(alice, carol).
parent(bob, dave).
parent(carol, eve).

% Rule (규칙): head :- body
% "X는 Y의 조상이다"
ancestor(X, Y) :- parent(X, Y).
ancestor(X, Y) :- parent(X, Z), ancestor(Z, Y).

% 쿼리: alice의 모든 자손은?
% ?- ancestor(alice, Who).
% → Who = bob, carol, dave, eve
```

---

## 2. 왜 Datalog가 필요한가?

### SQL의 재귀 쿼리 vs Datalog 비교

같은 질문을 SQL과 Datalog로 표현해보자: "노드 A에서 도달 가능한 모든 노드를 찾아라."

**SQL (WITH RECURSIVE)**:
```sql
WITH RECURSIVE reachable(node) AS (
    SELECT 'A' AS node  -- 기저 케이스
    UNION
    SELECT e.to_node    -- 재귀 케이스
    FROM edges e
    JOIN reachable r ON e.from_node = r.node
)
SELECT * FROM reachable;
```

**Datalog**:
```prolog
reachable(X) :- start(X).
reachable(Y) :- reachable(X), edge(X, Y).
```

Datalog 버전이 훨씬 간결하며, 복잡한 그래프 분석 규칙이 수십 개 추가되어도 조합이 자연스럽다.

---

## 3. 고정점 계산: Datalog 평가 모델

Datalog 엔진의 핵심은 **최소 고정점(Least Fixed Point)** 계산이다.

### 나이브 평가(Naïve Evaluation)

가장 단순한 평가 알고리즘:

```
T⁰ = EDB (기저 사실)
T^(i+1) = T^i ∪ immediate_consequence(T^i)
반복, T^(i+1) = T^i 가 될 때까지 (고정점 도달)
```

이를 Python으로 직접 구현해보자:

```python
from typing import Set, Dict, Tuple, List

# 사실(EDB: Extensional Database) - 기저 사실
EDB: Dict[str, Set[Tuple]] = {
    'parent': {
        ('alice', 'bob'),
        ('alice', 'carol'),
        ('bob', 'dave'),
        ('carol', 'eve'),
        ('dave', 'frank'),
    },
    'ancestor': set(),  # IDB: 파생될 사실
}

def evaluate_rules(db: Dict[str, Set[Tuple]]) -> Dict[str, Set[Tuple]]:
    """
    규칙:
      ancestor(X, Y) :- parent(X, Y).
      ancestor(X, Y) :- parent(X, Z), ancestor(Z, Y).
    """
    new_facts = set()

    # 규칙 1: ancestor(X, Y) :- parent(X, Y)
    for (x, y) in db['parent']:
        new_facts.add((x, y))

    # 규칙 2: ancestor(X, Y) :- parent(X, Z), ancestor(Z, Y)
    for (x, z) in db['parent']:
        for (z2, y) in db['ancestor']:
            if z == z2:
                new_facts.add((x, y))

    return new_facts

def naive_fixpoint(db: Dict[str, Set[Tuple]]) -> Set[Tuple]:
    """나이브 고정점 평가"""
    iteration = 0
    while True:
        old_ancestors = frozenset(db['ancestor'])
        new_facts = evaluate_rules(db)
        db['ancestor'] = db['ancestor'] | new_facts

        print(f"Iteration {iteration}: {len(db['ancestor'])} ancestors")
        iteration += 1

        if frozenset(db['ancestor']) == old_ancestors:
            break  # 고정점 도달

    return db['ancestor']

result = naive_fixpoint(EDB)
print("\n모든 조상 관계:")
for (x, y) in sorted(result):
    print(f"  ancestor({x}, {y})")
```

출력:
```
Iteration 0: 5 ancestors
Iteration 1: 8 ancestors
Iteration 2: 9 ancestors
Iteration 3: 9 ancestors  ← 고정점

모든 조상 관계:
  ancestor(alice, bob)
  ancestor(alice, carol)
  ancestor(alice, dave)
  ancestor(alice, eve)
  ancestor(alice, frank)
  ancestor(bob, dave)
  ancestor(bob, frank)
  ancestor(carol, eve)
  ancestor(dave, frank)
```

### 반증명식 세미-나이브 평가(Semi-Naïve Evaluation)

나이브 평가는 매 반복마다 전체 사실을 재검사하므로 비효율적이다. **세미-나이브 평가**는 새로 추가된 사실(`Δ`)만 처리하여 중복 계산을 제거한다.

```python
def semi_naive_fixpoint(parent_facts: Set[Tuple]) -> Set[Tuple]:
    """
    세미-나이브 고정점 평가
    Δancestor: 이번 이터레이션에 새로 추가된 사실만 추적
    """
    ancestor: Set[Tuple] = set()

    # 초기: 규칙 1로 Δ 초기화
    delta: Set[Tuple] = set(parent_facts)  # ancestor(X,Y) :- parent(X,Y)
    ancestor |= delta

    iteration = 0
    while delta:
        new_delta: Set[Tuple] = set()

        # 규칙 2: parent(X,Z), Δancestor(Z,Y) → ancestor(X,Y)
        # Δ에 있는 사실만 조인 파트너로 사용
        for (x, z) in parent_facts:
            for (z2, y) in delta:
                if z == z2 and (x, y) not in ancestor:
                    new_delta.add((x, y))

        ancestor |= new_delta
        delta = new_delta

        print(f"Iteration {iteration}: Δ크기={len(delta)}, 총={len(ancestor)}")
        iteration += 1

    return ancestor

parent_facts = EDB['parent']
result = semi_naive_fixpoint(parent_facts)
print(f"\n총 ancestor 사실 수: {len(result)}")
```

세미-나이브는 재귀 깊이가 깊은 그래프에서 나이브보다 수십~수백 배 빠르다.

---

## 4. Soufflé: 고성능 Datalog 컴파일러

**Soufflé**는 LLVM 기반의 오픈소스 Datalog 컴파일러로, 정적 분석 도구(Facebook Infer, Doop 등)에서 사용된다. Datalog 프로그램을 C++로 컴파일하여 병렬 실행한다.

```prolog
// Soufflé 문법으로 작성한 프로그램 분석기

// 타입 선언
.decl assign(v: symbol, e: symbol)    // v = e
.decl load(v: symbol, p: symbol)      // v = *p
.decl store(p: symbol, v: symbol)     // *p = v
.decl alias(p: symbol, q: symbol)     // p와 q는 같은 메모리를 가리킬 수 있다

// 데이터: C 프로그램의 포인터 연산
.input assign
.input load
.input store

// 앤더슨의 점-기반 포인터 분석 (Anderson's Points-To Analysis)
// 규칙 1: 직접 대입  p = q → p가 q와 alias
alias(P, Q) :- assign(P, Q).

// 규칙 2: 전이성  p=q, q alias r → p alias r
alias(P, R) :- alias(P, Q), alias(Q, R).

// 규칙 3: 역방향  p alias q → q alias p (대칭성)
alias(Q, P) :- alias(P, Q).

// 규칙 4: 로드-스토어  *p = v, p alias q, w = *q → w alias v
alias(W, V) :- store(P, V), alias(P, Q), load(W, Q).

.output alias
```

Soufflé 실행:
```bash
# 컴파일 후 실행 (병렬 처리)
souffle --jobs=8 --output-dir=./out pointer_analysis.dl

# 또는 인터프리터 모드
souffle -F input_dir -D output_dir pointer_analysis.dl
```

---

## 5. Datomic: Datalog를 데이터베이스 쿼리 언어로

**Datomic**은 Rich Hickey(Clojure 창시자)가 만든 불변 시계열 데이터베이스로, **Datalog를 쿼리 언어**로 채택한다.

```clojure
;; Datomic 데이터 모델: 사실(Fact)은 [엔티티 속성 값 트랜잭션] 튜플
;; Eav(Entity-Attribute-Value) 모델

;; 데이터 트랜잭션
(d/transact conn
  [{:db/id -1
    :person/name "Alice"
    :person/age  30
    :person/friends #{[:person/name "Bob"]
                      [:person/name "Carol"]}}
   {:db/id -2
    :person/name "Bob"
    :person/age  25}])

;; Datalog 쿼리: 30세 이상인 모든 사람의 이름과 친구 목록
(d/q '[:find ?name ?friend-name
       :where
       [?e :person/age ?age]
       [(>= ?age 30)]
       [?e :person/name ?name]
       [?e :person/friends ?f]
       [?f :person/name ?friend-name]]
     (d/db conn))

;; 시점 쿼리: 어제의 데이터베이스 상태로 쿼리
(let [yesterday (d/as-of (d/db conn) #inst "2026-09-23")]
  (d/q '[:find ?name
         :where [?e :person/name ?name]]
       yesterday))

;; 재귀 규칙: 친구의 친구까지 (전이 폐포)
(d/q '[:find ?name
       :in $ % ?start
       :where (friends-of ?start ?person)
              [?person :person/name ?name]]
     (d/db conn)
     ;; 재귀 규칙 정의
     '[[(friends-of ?a ?b)
        [?a :person/friends ?b]]
       [(friends-of ?a ?b)
        [?a :person/friends ?mid]
        (friends-of ?mid ?b)]]
     alice-entity-id)
```

---

## 6. Datalog의 제한과 확장

### 기본 Datalog의 제한
- **부정 없음**: 기본 Datalog는 부정을 지원하지 않는다 (단조성 유지). 확장판인 **Datalog¬**에서 계층적 부정(stratified negation) 지원.
- **집계 없음**: COUNT, SUM 등 집계를 기본 지원하지 않는다. 확장 버전에서 `count`, `min`, `max` 제공.
- **함수 적용 제한**: 일반적으로 순수 관계 연산만 지원.

### 확장 Datalog 예시 (Soufflé의 집계)

```prolog
// 각 노드의 도달 가능한 노드 수 계산
.decl edge(from: number, to: number)
.decl reachable(src: number, dst: number)
.decl reach_count(src: number, cnt: unsigned)

.input edge

reachable(X, Y) :- edge(X, Y).
reachable(X, Z) :- reachable(X, Y), edge(Y, Z).

// 집계: 각 노드에서 도달 가능한 노드 수
reach_count(X, count : { reachable(X, _) }).

.output reach_count
```

---

## 7. 주의사항과 팁

### 종료 보장
Datalog는 단조성(Monotonicity) 덕분에 평가가 항상 종료한다. 하지만 **데이터가 무한하면** 종료하지 않을 수 있으므로, 실제 구현에서는 EDB가 유한해야 한다.

### 성능 최적화
- **Magic Sets 변환**: 쿼리에 바인딩된 변수를 이용해 관련 없는 사실을 조기에 제거하는 최적화. 탑다운과 바텀업의 장점을 결합한다.
- **인덱싱**: 조인 연산에 자주 쓰이는 속성에 B-트리 또는 해시 인덱스를 자동 생성한다.
- **병렬 평가**: Soufflé는 각 이터레이션을 병렬로 평가해 멀티코어 활용도를 높인다.

### 적합한 사용 사례
- **정적 프로그램 분석**: 포인터 분석, 타입 추론, 보안 취약점 탐지
- **그래프 데이터베이스 쿼리**: 경로 탐색, 커뮤니티 감지
- **네트워크 설정 검증**: 라우팅 가능성, 방화벽 규칙 일관성
- **지식 그래프 추론**: OWL 온톨로지, 의미 웹

---

## 참고 자료

- [Datalog - Wikipedia](https://en.wikipedia.org/wiki/Datalog)
- [Datalog and Recursive Query Processing (Todd J. Green et al.)](http://blogs.evergreen.edu/sosw/files/2014/04/Green-Vol5-DBS-017.pdf)
- [An Introduction to Datalog - Michelin IT Engineering Blog](https://blogit.michelin.io/an-introduction-to-datalog/)
- [ZodiacEdge: a Datalog Engine With Incremental Rule Set Maintenance (arXiv)](https://arxiv.org/pdf/2312.14530)
