---
layout: post
title: "Merkle 트리 완전 정복: Git·Bitcoin·IPFS가 선택한 무결성 검증 자료구조"
date: 2026-06-25
categories: [cs, computer-science]
tags: [merkle-tree, hash-tree, blockchain, git, distributed-systems, cryptography, 자료구조]
---

`git fetch`는 수십만 개의 파일 중 변경된 것만 정확히 골라 전송합니다. Bitcoin 지갑 앱은 전체 블록체인(수백 GB)을 다운로드하지 않고도 특정 거래가 블록에 포함되었는지 검증합니다. 이 모든 마법의 핵심에 **Merkle 트리(Merkle Tree)**가 있습니다. 1979년 Ralph Merkle이 고안한 이 자료구조가 40년이 지난 지금도 분산 시스템의 근간을 이루는 이유를 깊이 파헤쳐 봅니다.

## Merkle 트리란 무엇인가

Merkle 트리(또는 해시 트리)는 **모든 리프 노드가 데이터의 암호화 해시**이고, **모든 내부 노드가 자식 노드들의 해시**인 이진 트리입니다. 트리의 루트를 **Merkle Root**라고 부르며, 이 단 하나의 해시값이 트리 아래에 있는 모든 데이터를 대표합니다.

```
                 [Root Hash]
                /            \
        [Hash AB]            [Hash CD]
        /      \             /      \
   [Hash A]  [Hash B]  [Hash C]  [Hash D]
      |          |         |         |
   Data A     Data B    Data C    Data D
```

데이터 A의 한 비트라도 바뀌면 Hash A → Hash AB → Root Hash 전체가 연쇄적으로 바뀝니다. 루트 해시만 비교해도 전체 데이터셋의 무결성을 즉시 확인할 수 있습니다.

## 왜 필요한가: 세 가지 핵심 문제를 해결한다

### 문제 1: 대규모 데이터셋에서 변경점 찾기 (O(n) → O(log n))

1TB짜리 데이터셋을 두 노드가 동기화하려고 할 때, 변경된 1KB를 찾기 위해 전체를 비교하면 1TB를 전송해야 합니다. Merkle 트리는 루트 해시만 비교하고, 다르면 자식 노드로 내려가는 이진 탐색으로 변경점을 O(log n)에 찾습니다.

### 문제 2: 신뢰 없이 데이터 존재 증명 (Merkle Proof)

분산 환경에서 "이 데이터가 실제로 해당 집합에 포함되어 있다"를 증명하려면 모든 데이터를 공유해야 할까요? Merkle 트리는 O(log n) 크기의 증명 경로(Merkle Proof)만으로 이를 증명합니다. Bitcoin의 SPV(Simplified Payment Verification)가 바로 이 원리입니다.

### 문제 3: 분산 복제본 검증

Cassandra, DynamoDB 같은 분산 데이터베이스는 Merkle 트리를 사용해 복제본 간 불일치를 빠르게 감지하고 수리합니다(Anti-Entropy Repair).

## 구현: Python으로 Merkle 트리 만들기

```python
import hashlib
from typing import Optional

def sha256(data: bytes) -> bytes:
    return hashlib.sha256(data).digest()

def sha256_hex(data: bytes) -> str:
    return hashlib.sha256(data).hexdigest()

class MerkleNode:
    def __init__(self, left: Optional['MerkleNode'], 
                 right: Optional['MerkleNode'], 
                 data: Optional[bytes] = None):
        self.left = left
        self.right = right
        if data:
            # 리프 노드: 실제 데이터를 해싱
            self.hash = sha256(data)
        else:
            # 내부 노드: 두 자식 해시를 합쳐 다시 해싱
            combined = left.hash + (right.hash if right else left.hash)
            self.hash = sha256(combined)

class MerkleTree:
    def __init__(self, data_blocks: list[bytes]):
        if not data_blocks:
            raise ValueError("데이터가 비어 있습니다")
        
        # 1. 리프 노드 생성
        leaves = [MerkleNode(None, None, block) for block in data_blocks]
        
        # 홀수 개라면 마지막 리프를 복제 (Bitcoin 방식)
        if len(leaves) % 2 == 1:
            leaves.append(leaves[-1])
        
        self.root = self._build(leaves)

    def _build(self, nodes: list[MerkleNode]) -> MerkleNode:
        if len(nodes) == 1:
            return nodes[0]
        
        next_level = []
        for i in range(0, len(nodes), 2):
            left = nodes[i]
            right = nodes[i + 1] if i + 1 < len(nodes) else nodes[i]
            next_level.append(MerkleNode(left, right))
        
        return self._build(next_level)

    @property
    def root_hash(self) -> str:
        return self.root.hash.hex()


# 실제 사용 예시
transactions = [
    b"Alice sends 1 BTC to Bob",
    b"Bob sends 0.5 BTC to Charlie",
    b"Charlie sends 0.3 BTC to Dave",
    b"Eve sends 2 BTC to Frank",
]

tree = MerkleTree(transactions)
print(f"Merkle Root: {tree.root_hash}")

# 데이터 하나가 바뀌면 루트도 바뀜
tampered = transactions.copy()
tampered[2] = b"Charlie sends 0.3 BTC to HACKER"
tampered_tree = MerkleTree(tampered)
print(f"Tampered Root: {tampered_tree.root_hash}")
print(f"루트 해시 일치: {tree.root_hash == tampered_tree.root_hash}")
# 출력: False — 변조 즉시 감지!
```

## Merkle Proof 구현

특정 트랜잭션이 블록에 포함되어 있음을 O(log n) 크기의 증거로 증명합니다.

```python
def get_merkle_proof(data_blocks: list[bytes], target_index: int) -> list[dict]:
    """
    target_index번째 데이터에 대한 Merkle Proof 생성.
    반환값: [{'hash': ..., 'position': 'left'|'right'}, ...] 형태의 증명 경로
    """
    leaves = [sha256(b) for b in data_blocks]
    if len(leaves) % 2 == 1:
        leaves.append(leaves[-1])
    
    proof = []
    current_index = target_index
    current_level = leaves
    
    while len(current_level) > 1:
        # 형제 노드 찾기
        if current_index % 2 == 0:
            sibling_index = current_index + 1
            sibling_position = 'right'
        else:
            sibling_index = current_index - 1
            sibling_position = 'left'
        
        if sibling_index < len(current_level):
            proof.append({
                'hash': current_level[sibling_index].hex(),
                'position': sibling_position
            })
        
        # 다음 레벨로 이동
        next_level = []
        for i in range(0, len(current_level), 2):
            left = current_level[i]
            right = current_level[i + 1] if i + 1 < len(current_level) else left
            next_level.append(sha256(left + right))
        
        current_level = next_level
        current_index //= 2
    
    return proof

def verify_merkle_proof(data: bytes, proof: list[dict], root_hash: str) -> bool:
    """Merkle Proof 검증: 루트 해시만 알면 특정 데이터의 포함 여부를 확인"""
    current_hash = sha256(data)
    
    for step in proof:
        sibling_hash = bytes.fromhex(step['hash'])
        if step['position'] == 'right':
            current_hash = sha256(current_hash + sibling_hash)
        else:
            current_hash = sha256(sibling_hash + current_hash)
    
    return current_hash.hex() == root_hash

# SPV 검증 시뮬레이션
transactions = [
    b"TX: Alice→Bob 1BTC",
    b"TX: Bob→Charlie 0.5BTC",
    b"TX: Charlie→Dave 0.3BTC",
    b"TX: Eve→Frank 2BTC",
]
tree = MerkleTree(transactions)
proof = get_merkle_proof(transactions, index=1)  # 두 번째 트랜잭션 증명

is_valid = verify_merkle_proof(transactions[1], proof, tree.root_hash)
print(f"포함 여부 검증: {'유효 ✓' if is_valid else '무효 ✗'}")
# 전체 블록체인 없이도 검증 가능!
```

## 실제 시스템에서의 Merkle 트리

### Git의 Merkle DAG

Git은 순수한 이진 트리가 아닌 **Merkle DAG(Directed Acyclic Graph)**를 사용합니다. 각 커밋은 트리 오브젝트의 해시를 포함하고, 트리 오브젝트는 블롭(파일) 오브젝트들의 해시를 포함합니다.

```
Commit A (hash: 3a4f...)
  └── Tree (hash: 8b2c...)
        ├── blob: README.md   (hash: 1a2b...)
        ├── blob: src/main.py (hash: 9d3e...)
        └── tree: tests/      (hash: 4f5a...)
              └── blob: test_main.py (hash: 7c8b...)
```

`git fetch`는 원격 저장소의 커밋 해시와 로컬 커밋 해시를 비교하고, 달라진 서브트리만 재귀적으로 탐색해 필요한 오브젝트만 전송합니다.

### Bitcoin의 블록 구조

Bitcoin 블록 헤더는 80바이트이며, 해당 블록의 모든 트랜잭션에 대한 Merkle Root를 포함합니다.

```
Block Header (80 bytes)
├── Version (4 bytes)
├── Previous Block Hash (32 bytes)
├── Merkle Root (32 bytes)  ← 모든 트랜잭션의 요약
├── Timestamp (4 bytes)
├── Difficulty Target (4 bytes)
└── Nonce (4 bytes)
```

SPV 지갑은 80바이트 블록 헤더만 다운로드하고(전체 블록 크기의 0.02% 이하), Merkle Proof를 통해 특정 트랜잭션의 존재를 검증합니다.

### IPFS와 분산 데이터베이스

IPFS(InterPlanetary File System)는 파일의 내용 해시를 주소로 사용하는 CID(Content Identifier) 체계에 Merkle DAG를 활용합니다. Cassandra의 Anti-Entropy 프로세스는 Merkle 트리를 사용해 복제본 간 불일치를 O(log n)으로 탐지합니다.

## 주의사항 및 실전 팁

**1. 해시 함수 선택**: SHA-256이 표준이지만, 성능이 중요한 내부 시스템에서는 SHA-3, BLAKE3, xxHash를 검토하세요.

**2. 홀수 리프 처리**: 리프 수가 홀수이면 마지막 리프를 복제하거나, 빈 해시로 패딩합니다. Bitcoin은 복제 방식을 사용합니다.

**3. 이중 해싱 (Bitcoin)**: Bitcoin은 SHA256(SHA256(data))를 사용합니다. Length-extension 공격을 방어하기 위한 설계입니다.

**4. 업데이트 비용**: 리프 데이터 하나를 바꾸면 루트까지의 경로(O(log n) 노드)를 재계산해야 합니다. 빈번한 업데이트가 필요한 경우 성능 영향을 고려하세요.

**5. Sparse Merkle Tree**: 인덱스 공간이 매우 크지만 실제 데이터는 드문 경우(예: 이더리움 상태 트리), 존재하지 않는 리프에 기본 해시값을 사용하는 Sparse Merkle Tree를 활용합니다.

Merkle 트리는 "신뢰 없이 무결성을 증명"한다는 철학을 구현한 자료구조입니다. 중앙화된 권위자 없이도 데이터의 진실성을 수학적으로 보장할 수 있다는 점에서, 분산 시스템 시대의 핵심 기반 기술로 자리잡고 있습니다.

## 참고 자료
- [Merkle Tree in System Design: A Complete Guide - Dev Cookies](https://devcookies.medium.com/merkle-tree-in-system-design-a-complete-guide-46b8fab599c4)
- [Merkle Trees: The Data Structure for Verifiable State - Pratik Pandey](https://pratikpandey.substack.com/p/merkle-trees-the-data-structure-for)
- [Merkle Trees in Git and Bitcoin - Initial Commit](https://initialcommit.com/blog/git-bitcoin-merkle-tree)
- [Merkle Trees in System Design - AlgoMaster](https://algomaster.io/learn/system-design/merkle-trees)
