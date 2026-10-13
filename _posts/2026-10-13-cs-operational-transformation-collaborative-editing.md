---
layout: post
title: "Operational Transformation(OT) — 구글 독스 협업 편집의 핵심 알고리즘"
date: 2026-10-13
categories: [cs, computer-science]
tags: [OT, OperationalTransformation, 협업편집, CRDT, 분산시스템, 실시간동기화]
---

## 개요

구글 독스에서 여러 사람이 동시에 같은 문서를 편집할 때 서로의 작업이 충돌하지 않고 자연스럽게 합쳐지는 것을 경험해보셨나요? 이 마법 같은 경험의 핵심에는 **Operational Transformation(OT, 연산 변환)** 알고리즘이 있습니다.

OT는 1989년 Ellis와 Gibbs가 처음 제안했으며, Google Wave, Google Docs, Apache Wave 등 다양한 실시간 협업 도구의 근간이 됩니다. 현재는 CRDT(Conflict-free Replicated Data Types)와 함께 협업 편집의 양대 산맥을 이루고 있습니다.

---

## 문제 정의: 왜 OT가 필요한가

두 사용자 Alice와 Bob이 동시에 같은 문서를 편집하는 시나리오를 생각해 봅시다.

**초기 문서**: `"Hello"`

- **Alice**: 인덱스 5에 `" World"` 삽입 → `"Hello World"`
- **Bob**: 인덱스 0에 `"Hi! "` 삽입 → `"Hi! Hello"`

두 연산이 동시에 발생하면, 각 클라이언트는 자신의 연산을 먼저 적용한 뒤 상대방의 연산을 받습니다. 만약 단순하게 수신된 연산을 그대로 적용하면:

- Alice의 클라이언트: `"Hello World"` → Bob의 `insert(0, "Hi! ")` 적용 → `"Hi! Hello World"` ✓
- Bob의 클라이언트: `"Hi! Hello"` → Alice의 `insert(5, " World")` 적용 → `"Hi! Hello World"` ... 아니라 `"Hi!  World Hello"` ✗

Bob의 클라이언트에서 Alice의 연산이 잘못된 인덱스에 적용되어 문서가 달라집니다. OT는 이 문제를 **연산 변환(transform)**을 통해 해결합니다.

---

## OT의 핵심 개념

### 연산(Operation) 표현

문서에 대한 기본 연산을 다음과 같이 정의합니다:

```python
from dataclasses import dataclass
from typing import Literal, Union

@dataclass
class Insert:
    """문자열을 특정 위치에 삽입"""
    position: int
    text: str
    op_type: Literal["insert"] = "insert"

@dataclass
class Delete:
    """특정 위치에서 n개 문자를 삭제"""
    position: int
    length: int
    op_type: Literal["delete"] = "delete"

Operation = Union[Insert, Delete]


def apply_operation(doc: str, op: Operation) -> str:
    """문서에 연산 적용"""
    if isinstance(op, Insert):
        return doc[:op.position] + op.text + doc[op.position:]
    elif isinstance(op, Delete):
        return doc[:op.position] + doc[op.position + op.length:]
    return doc
```

### 변환 함수(Transform Function)

OT의 핵심은 두 동시 연산 `op_a`와 `op_b`가 있을 때, `op_a`를 `op_b`의 효과를 고려하여 변환하는 함수 `transform(op_a, op_b) → op_a'`입니다.

변환된 `op_a'`은 `op_b`가 이미 적용된 문서 상태에서 올바르게 적용될 수 있습니다.

```python
def transform(op_a: Operation, op_b: Operation) -> Operation:
    """
    op_b가 먼저 적용된 상황에서 op_a를 변환.
    두 연산이 동시에 발생했을 때 올바른 위치 조정을 담당.
    """
    if isinstance(op_a, Insert) and isinstance(op_b, Insert):
        return transform_insert_insert(op_a, op_b)
    elif isinstance(op_a, Insert) and isinstance(op_b, Delete):
        return transform_insert_delete(op_a, op_b)
    elif isinstance(op_a, Delete) and isinstance(op_b, Insert):
        return transform_delete_insert(op_a, op_b)
    elif isinstance(op_a, Delete) and isinstance(op_b, Delete):
        return transform_delete_delete(op_a, op_b)
    return op_a


def transform_insert_insert(op_a: Insert, op_b: Insert) -> Insert:
    """두 삽입 연산 변환"""
    if op_b.position < op_a.position:
        # op_b가 op_a 앞에 삽입됐으므로 op_a의 위치를 op_b 길이만큼 뒤로 이동
        return Insert(
            position=op_a.position + len(op_b.text),
            text=op_a.text
        )
    elif op_b.position == op_a.position:
        # 같은 위치: 타이브레이킹 규칙 (예: 클라이언트 ID 기준)
        # 여기서는 op_b를 우선시 (op_a를 뒤로 밀기)
        return Insert(
            position=op_a.position + len(op_b.text),
            text=op_a.text
        )
    else:
        # op_b가 op_a 뒤에 삽입됐으므로 op_a 위치 변경 없음
        return op_a


def transform_insert_delete(op_a: Insert, op_b: Delete) -> Insert:
    """삽입 연산을 삭제 연산에 대해 변환"""
    if op_b.position + op_b.length <= op_a.position:
        # 삭제가 삽입 위치 앞에서 발생: 삽입 위치 앞당김
        return Insert(
            position=op_a.position - op_b.length,
            text=op_a.text
        )
    elif op_b.position >= op_a.position:
        # 삭제가 삽입 위치 뒤에서 발생: 삽입 위치 변경 없음
        return op_a
    else:
        # 삭제가 삽입 위치를 포함: 삭제된 시작 위치로 이동
        return Insert(position=op_b.position, text=op_a.text)


def transform_delete_insert(op_a: Delete, op_b: Insert) -> Delete:
    """삭제 연산을 삽입 연산에 대해 변환"""
    if op_b.position <= op_a.position:
        return Delete(
            position=op_a.position + len(op_b.text),
            length=op_a.length
        )
    elif op_b.position >= op_a.position + op_a.length:
        return op_a
    else:
        # 삽입이 삭제 범위 중간에 발생: 삭제 범위 확장
        return Delete(
            position=op_a.position,
            length=op_a.length + len(op_b.text)
        )


def transform_delete_delete(op_a: Delete, op_b: Delete) -> Delete:
    """두 삭제 연산 변환"""
    op_b_end = op_b.position + op_b.length
    op_a_end = op_a.position + op_a.length

    if op_b_end <= op_a.position:
        # op_b가 op_a 완전히 앞: op_a 위치 앞당김
        return Delete(position=op_a.position - op_b.length, length=op_a.length)
    elif op_b.position >= op_a_end:
        # op_b가 op_a 완전히 뒤: 변경 없음
        return op_a
    else:
        # 겹치는 영역 처리: 이미 삭제된 부분 제외
        new_position = min(op_a.position, op_b.position)
        # 실제 삭제해야 할 길이 재계산
        overlap_start = max(op_a.position, op_b.position)
        overlap_end = min(op_a_end, op_b_end)
        overlap = max(0, overlap_end - overlap_start)
        new_length = op_a.length - overlap
        return Delete(position=new_position, length=max(0, new_length))
```

---

## OT 서버 아키텍처

```python
from collections import defaultdict
from typing import List, Tuple

class OTServer:
    """중앙 서버: 연산 순서를 결정하고 변환을 중재"""

    def __init__(self):
        self.document = ""
        self.history: List[Tuple[str, Operation]] = []  # (client_id, op)

    def apply(self, client_id: str, op: Operation, client_revision: int) -> Operation:
        """
        클라이언트 연산을 서버 기준으로 변환 후 적용.
        client_revision: 클라이언트가 마지막으로 본 서버 리비전
        """
        # 클라이언트가 모르는 서버의 연산들에 대해 변환
        concurrent_ops = self.history[client_revision:]
        transformed_op = op

        for _, server_op in concurrent_ops:
            transformed_op = transform(transformed_op, server_op)

        # 변환된 연산 적용
        self.document = apply_operation(self.document, transformed_op)
        self.history.append((client_id, transformed_op))

        print(f"  서버 문서 상태: '{self.document}'")
        return transformed_op  # 다른 클라이언트에게 브로드캐스트


# 시나리오: Alice와 Bob이 동시에 편집
server = OTServer()
server.document = "Hello"
server.history = []

print("초기 문서:", repr(server.document))
print()

# Alice: 인덱스 5에 " World" 삽입 (서버 rev=0 기준)
alice_op = Insert(position=5, text=" World")
print("Alice의 연산:", alice_op)
server_alice_op = server.apply("alice", alice_op, client_revision=0)

# Bob: 인덱스 0에 "Hi! " 삽입 (서버 rev=0 기준 — 동시 발생)
bob_op = Insert(position=0, text="Hi! ")
print("Bob의 연산:", bob_op)
server_bob_op = server.apply("bob", bob_op, client_revision=0)

print()
print("최종 서버 문서:", repr(server.document))
# 출력: "Hi! Hello World" (두 클라이언트 모두 이 상태에 수렴)
```

---

## TP1, TP2 정확성 조건

OT 알고리즘의 정확성을 위해 변환 함수는 두 가지 속성을 만족해야 합니다:

**TP1 (Transformation Property 1)**: 두 동시 연산 `a`, `b`에 대해
```
apply(apply(S, a), transform(b, a)) == apply(apply(S, b), transform(a, b))
```
즉, 어떤 순서로 연산을 적용해도 최종 상태가 같아야 합니다.

**TP2 (Transformation Property 2)**: 세 동시 연산 `a`, `b`, `c`에 대해
```
transform(a, compose(b, transform(c, b))) == transform(a, compose(transform(b, c), b))
```
연산 순서가 달라도 최종 변환 결과가 같아야 합니다. TP2는 서버 없이 P2P 방식으로 OT를 구현할 때 필요합니다.

---

## OT vs CRDT

| 비교 항목 | OT | CRDT |
|-----------|-----|------|
| 중앙 서버 | 필요 (연산 순서 결정) | 불필요 |
| 구현 복잡도 | 높음 (변환 함수 복잡) | 중간 |
| 메모리 사용 | 효율적 | 높음 (삭제된 요소 유지) |
| 오프라인 지원 | 제한적 | 완전 지원 |
| 적용 사례 | Google Docs, Quill | Automerge, Yjs, Figma |

현재 추세는 구현이 더 명확한 CRDT로 이동하고 있으나, Google Docs처럼 중앙 서버가 있는 환경에서는 OT가 여전히 효율적입니다.

---

## 주의사항 및 팁

**1. TP2 구현의 어려움**: 많은 OT 구현이 TP2를 제대로 구현하지 못해 P2P 환경에서 문서가 달라지는 버그가 발생합니다. 중앙 서버 아키텍처를 사용하면 TP1만으로 충분합니다.

**2. Undo/Redo 처리**: 실행 취소 연산을 OT 환경에서 처리하는 것은 추가적인 복잡성을 수반합니다. Exclusion Transformation(ET)이 필요합니다.

**3. 리비전 동기화**: 클라이언트가 오프라인 상태에서 많은 수의 연산을 누적하면, 서버에 다시 연결할 때 대량의 변환이 필요할 수 있습니다.

**4. 보안**: 악의적인 클라이언트가 의도적으로 잘못된 client_revision을 전송하면 문서가 손상될 수 있습니다. 서버에서 반드시 검증해야 합니다.

---

## 정리

Operational Transformation은 분산 환경에서 동시 편집의 일관성을 보장하는 강력한 알고리즘입니다. 변환 함수를 통해 동시 발생하는 연산들의 위치를 조정함으로써 모든 클라이언트가 동일한 최종 상태에 수렴하게 만듭니다. 구현이 복잡하지만, 실시간 협업 도구의 핵심 기술로서 그 중요성은 여전히 높습니다.

## 참고 자료
- [Ellis & Gibbs (1989) - Concurrency Control in Groupware Systems](https://dl.acm.org/doi/10.1145/67544.66963)
- [Wikipedia: Operational transformation](https://en.wikipedia.org/wiki/Operational_transformation)
- [Operational Transformation FAQ](https://operational-transformation.github.io/)
- [ShareDB - OT 기반 협업 편집 라이브러리](https://github.com/share/sharedb)
