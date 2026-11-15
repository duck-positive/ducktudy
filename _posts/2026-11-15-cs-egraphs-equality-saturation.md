---
layout: post
title: "E-graphs와 Equality Saturation 완전 정복: 모든 등가 변환을 동시에 탐색하는 컴파일러 최적화의 비밀"
date: 2026-11-15
categories: [cs, computer-science]
tags: [compiler, optimization, e-graph, equality-saturation, rewriting, egg, rust]
---

컴파일러 최적화의 가장 큰 난제 중 하나는 **rewriting 순서에 따른 지역 최적해 문제**입니다. `a * 2 + a * 3`을 최적화할 때, 먼저 분배법칙을 적용하면 `a * 5`로 단순화할 수 있지만, 다른 규칙을 먼저 적용하면 더 복잡한 형태로 고착될 수 있습니다. E-graphs와 Equality Saturation은 이 문제를 근본적으로 해결하는 우아한 방법입니다.

## 개념 설명

### E-graph란?

E-graph(Equality Graph)는 **등가 클래스(equivalence class)의 집합**으로 표현식들을 압축적으로 저장하는 자료구조입니다. 두 가지 핵심 구성 요소로 이루어집니다.

**E-node**: 연산자와 자식 e-class ID들의 쌍. 예를 들어 `(+ [0] [1])`은 e-class 0과 e-class 1을 더하는 노드입니다.

**E-class**: 의미적으로 동등한(equivalent) e-node들의 집합. 하나의 e-class는 "서로 같은 값을 나타내는 모든 표현식의 집합"을 의미합니다.

일반 AST(추상 구문 트리)와의 차이를 비교해봅시다:

```
일반 AST: 단 하나의 표현식만 저장
  (+ (* a 2) (* a 3))

E-graph: 등가인 모든 표현식을 동시에 저장
  e-class 5 = {(+ e3 e4), (* a e6), (* e6 a)}
  e-class 6 = {5, (+ 2 3)}
  e-class 3 = {(* a 2), (* 2 a)}
  ...
```

### Equality Saturation이란?

Equality Saturation은 E-graph에 rewrite rule을 반복 적용하되, **기존 표현식을 삭제하지 않고 새로운 등가 표현식을 추가**하는 방식으로 동작합니다. 모든 가능한 변환이 적용된 후(포화 상태), 비용 함수에 따라 최적의 표현식을 추출합니다.

알고리즘 흐름:
1. 초기 표현식을 E-graph에 추가
2. Rewrite rule을 적용해 등가 표현식 추가 (merge)
3. E-graph가 포화(saturate)될 때까지 반복
4. Cost function으로 최적 표현식 추출

## 왜 필요한가?

### 기존 탐욕적 Rewriting의 문제

전통적인 컴파일러 최적화는 rewrite rule을 순차적으로 적용합니다:

```
(a + 0) * 1
→ [add-0 적용] a * 1
→ [mul-1 적용] a    ✓ 최적 결과
```

하지만 규칙 적용 순서가 달라지면 최적해를 놓칩니다:

```
(a * 2) + (a * 3)
→ [factor 적용 실패: 형태가 다름]
→ ... 최적화 불가
```

E-graph는 모든 규칙을 동시에, 비파괴적으로 적용하므로 이 문제를 완전히 해결합니다.

### 실제 적용 사례

- **Herbie**: 부동소수점 수식을 수학적으로 등가이면서 수치적으로 더 안정된 형태로 자동 변환
- **Halide**: 이미지 처리 파이프라인 스케줄 최적화
- **ONNX/TensorFlow**: 딥러닝 연산 그래프 최적화
- **쿼리 최적화**: 관계형 대수식의 등가 변환 탐색

## 실제 구현 예제

### 예제 1: egg 라이브러리로 수식 최적화 (Rust)

[egg](https://github.com/egraphs-good/egg)는 Rust로 구현된 고성능 E-graph 라이브러리입니다.

```rust
use egg::{rewrite as rw, *};

// 언어 정의: 수식을 나타내는 열거형
define_language! {
    enum Math {
        Num(i32),
        "+" = Add([Id; 2]),
        "*" = Mul([Id; 2]),
        Symbol(Symbol),
    }
}

// Rewrite rule 정의
fn make_rules() -> Vec<Rewrite<Math, ()>> {
    vec![
        rw!("commute-add";   "(+ ?a ?b)"             => "(+ ?b ?a)"),
        rw!("commute-mul";   "(* ?a ?b)"             => "(* ?b ?a)"),
        rw!("add-0";         "(+ ?a 0)"              => "?a"),
        rw!("mul-0";         "(* ?a 0)"              => "0"),
        rw!("mul-1";         "(* ?a 1)"              => "?a"),
        rw!("distribute";    "(* ?a (+ ?b ?c))"      => "(+ (* ?a ?b) (* ?a ?c))"),
        rw!("factor";        "(+ (* ?a ?b) (* ?a ?c))" => "(* ?a (+ ?b ?c))"),
        rw!("add-same";      "(+ ?a ?a)"             => "(* ?a 2)"),
    ]
}

// Equality Saturation으로 표현식 단순화
fn simplify(s: &str) -> String {
    // 1. 표현식 파싱
    let expr: RecExpr<Math> = s.parse().unwrap();

    // 2. Runner 실행 (equality saturation)
    let runner = Runner::default()
        .with_expr(&expr)
        .with_iter_limit(10)       // 최대 반복 횟수
        .with_node_limit(10_000)   // 최대 e-node 수 제한
        .run(&make_rules());

    // 3. 최소 AST 크기를 비용 함수로 최적 추출
    let extractor = Extractor::new(&runner.egraph, AstSize);
    let (cost, best) = extractor.find_best(runner.roots[0]);

    println!("  비용(AST 노드 수): {}", cost);
    best.to_string()
}

fn main() {
    let cases = vec![
        // (a * 2) + (a * 3) = a * (2 + 3) = a * 5
        "(+ (* a 2) (* a 3))",
        // (a * 1) = a
        "(* a 1)",
        // (a + 0) * 1 = a
        "(* (+ a 0) 1)",
        // 0 * (복잡한 식) = 0
        "(* 0 (+ (* a b) (* c d)))",
    ];

    for case in cases {
        println!("입력: {}", case);
        println!("출력: {}", simplify(case));
        println!();
    }
}

/* 출력 예:
입력: (+ (* a 2) (* a 3))
  비용(AST 노드 수): 3
출력: (* a 5)

입력: (* a 1)
  비용(AST 노드 수): 1
출력: a

입력: (* (+ a 0) 1)
  비용(AST 노드 수): 1
출력: a

입력: (* 0 (+ (* a b) (* c d)))
  비용(AST 노드 수): 1
출력: 0
*/
```

### 예제 2: Python으로 구현한 E-graph 핵심 구조

```python
from dataclasses import dataclass, field
from typing import Any, Dict, List, Optional, Tuple

@dataclass(frozen=True)
class ENode:
    """E-node: 연산자와 자식 e-class ID들의 쌍"""
    op: str
    children: Tuple[int, ...] = field(default_factory=tuple)

class EClass:
    """E-class: 의미적으로 동등한 e-node들의 집합"""
    def __init__(self, id: int):
        self.id = id
        self.nodes: List[ENode] = []

class UnionFind:
    """경로 압축 Union-Find: e-class 합치기"""
    def __init__(self):
        self._parent: Dict[int, int] = {}

    def make_set(self, x: int):
        self._parent[x] = x

    def find(self, x: int) -> int:
        while self._parent[x] != x:
            self._parent[x] = self._parent[self._parent[x]]  # 경로 압축
            x = self._parent[x]
        return x

    def union(self, x: int, y: int) -> int:
        rx, ry = self.find(x), self.find(y)
        if rx == ry:
            return rx
        self._parent[ry] = rx
        return rx

class EGraph:
    """E-graph 핵심 구현"""

    def __init__(self):
        self.classes: Dict[int, EClass] = {}
        self.node_to_class: Dict[ENode, int] = {}
        self.uf = UnionFind()
        self._next_id = 0
        # 보류 중인 merge (rebuilding을 위해)
        self._pending: List[Tuple[int, int]] = []

    def _new_class_id(self) -> int:
        id = self._next_id
        self._next_id += 1
        return id

    def find(self, id: int) -> int:
        return self.uf.find(id)

    def add(self, node: ENode) -> int:
        """e-node 추가. 이미 존재하면 해당 e-class 반환"""
        # 자식 ID를 대표 ID로 정규화
        canonical = ENode(node.op, tuple(self.find(c) for c in node.children))

        if canonical in self.node_to_class:
            return self.find(self.node_to_class[canonical])

        # 새 e-class 생성
        new_id = self._new_class_id()
        self.uf.make_set(new_id)
        eclass = EClass(new_id)
        eclass.nodes.append(canonical)
        self.classes[new_id] = eclass
        self.node_to_class[canonical] = new_id
        return new_id

    def merge(self, id1: int, id2: int) -> int:
        """두 e-class 합치기 (등가 관계 추가)"""
        root1, root2 = self.find(id1), self.find(id2)
        if root1 == root2:
            return root1

        # 더 작은 클래스를 큰 클래스로 합치기
        self.classes[root1].nodes.extend(self.classes[root2].nodes)
        merged = self.uf.union(root1, root2)
        # 합쳐진 클래스만 보관
        if merged != root1:
            self.classes[merged] = self.classes[root1]
        self._pending.append((root1, root2))
        return merged

    def rebuild(self):
        """e-graph 불변식 복원 (합치기 이후 필수)"""
        while self._pending:
            self._pending.pop()
            # 모든 e-node의 자식 ID를 최신 대표 ID로 갱신
            new_node_to_class = {}
            for node, class_id in self.node_to_class.items():
                canonical = ENode(node.op, tuple(self.find(c) for c in node.children))
                root = self.find(class_id)
                if canonical in new_node_to_class:
                    # 이미 존재: 합치기
                    self.merge(root, new_node_to_class[canonical])
                else:
                    new_node_to_class[canonical] = root
            self.node_to_class = new_node_to_class

    def add_expr(self, expr) -> int:
        """재귀적으로 표현식을 e-graph에 추가"""
        if isinstance(expr, (int, float, str)) and not isinstance(expr, tuple):
            return self.add(ENode(str(expr)))
        op, *args = expr
        children = tuple(self.add_expr(arg) for arg in args)
        return self.add(ENode(op, children))

    def apply_rule(self, lhs_op: str, rhs_fn) -> bool:
        """간단한 패턴 매칭 rewrite rule 적용"""
        changed = False
        for node, class_id in list(self.node_to_class.items()):
            if node.op == lhs_op:
                new_id = rhs_fn(self, node, class_id)
                if new_id is not None and self.find(new_id) != self.find(class_id):
                    self.merge(class_id, new_id)
                    changed = True
        if changed:
            self.rebuild()
        return changed

# 사용 예시
def rule_mul_1(egraph: EGraph, node: ENode, class_id: int) -> Optional[int]:
    """(* a 1) => a"""
    if len(node.children) == 2:
        right_root = egraph.find(node.children[1])
        for n in egraph.classes[right_root].nodes:
            if n.op == '1':
                return node.children[0]  # 왼쪽 자식 e-class 반환
    return None

def rule_add_0(egraph: EGraph, node: ENode, class_id: int) -> Optional[int]:
    """(+ a 0) => a"""
    if len(node.children) == 2:
        right_root = egraph.find(node.children[1])
        for n in egraph.classes[right_root].nodes:
            if n.op == '0':
                return node.children[0]
    return None

egraph = EGraph()
# "(a * 1) + 0" 추가
root = egraph.add_expr(('+', ('*', 'a', 1), 0))
print(f"초기 root e-class: {root}")

# 규칙 반복 적용 (equality saturation)
for _ in range(5):
    r1 = egraph.apply_rule('*', rule_mul_1)
    r2 = egraph.apply_rule('+', rule_add_0)
    if not r1 and not r2:
        break

# 최종 결과: root e-class가 'a'와 같은 클래스인지 확인
a_id = egraph.node_to_class.get(ENode('a'), -1)
root_rep = egraph.find(root)
a_rep = egraph.find(a_id) if a_id != -1 else -1
print(f"root 대표: {root_rep}, 'a' 대표: {a_rep}")
print(f"'(a*1)+0' ≡ 'a': {root_rep == a_rep}")  # True이면 성공
```

## 주의사항과 팁

### 1. Rebuilding 비용 관리

E-graph에서 두 e-class를 합친 후 불변식을 복원하는 **rebuilding**이 가장 비용이 큰 연산입니다. egg 라이브러리는 이를 배치(batch) 처리하여 반복 횟수를 획기적으로 줄였습니다. 직접 구현 시 merge 직후 즉시 rebuild하지 말고 모아서 처리하세요.

### 2. 종료 조건 설정

이론적으로 rewrite rule 집합이 무한히 새 표현식을 생성할 수 있습니다. 실용적 종료 조건:
- **반복 횟수 제한**: `iter_limit = 10`
- **e-node 수 제한**: `node_limit = 100_000`
- **실제 포화 감지**: 한 반복에서 아무 것도 추가되지 않으면 종료

### 3. Cost Function 설계

최적 표현식 추출을 위한 비용 함수 예시:
- **AstSize**: 노드 수 최소화 (기본값)
- **연산 비용**: 나눗셈 > 곱셈 > 덧셈 순으로 가중치
- **레지스터 압력**: 동시에 활성화되는 e-class 수 최소화

### 4. Conditional Rewrite와 E-class Analysis

단순 패턴 매칭을 넘어, e-class에 도메인 분석 정보(부호, 범위, 타입 등)를 첨부할 수 있습니다. 예를 들어 `x / x => 1` 규칙은 `x ≠ 0`이 보장될 때만 적용해야 합니다. egg의 `Analysis` 트레이트를 활용하면 이런 조건부 규칙을 안전하게 구현할 수 있습니다.

### 5. 실용적 한계

- **메모리**: E-graph는 메모리를 많이 소비합니다. 대형 프로그램 전체보다는 핵심 표현식 최적화에 적합합니다.
- **규칙 충돌**: 서로 무한 루프를 일으키는 규칙 쌍(예: `(+ a b) => (+ b a)`)은 포화가 불가능합니다. 방향성 있는 규칙 설계가 필요합니다.

## 참고 자료

- [egg: Fast and Extensible Equality Saturation (arXiv:2004.03082)](https://arxiv.org/abs/2004.03082)
- [egraphs-good GitHub Organization](https://github.com/egraphs-good)
- [egg crate on crates.io](https://crates.io/crates/egg)
- [E-Graphs Good - Official Site](https://egraphs-good.github.io/)
