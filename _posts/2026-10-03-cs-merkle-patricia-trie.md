---
layout: post
title: "머클 패트리샤 트라이 (Merkle Patricia Trie): 이더리움 상태의 핵심 자료구조"
date: 2026-10-03
categories: [cs, computer-science]
tags: [merkle-patricia-trie, ethereum, blockchain, trie, data-structure, cryptography]
---

이더리움 블록체인은 수억 개의 계정 잔액과 스마트 컨트랙트 상태를 어떻게 단 하나의 해시값으로 표현하고 검증할까? 그 답이 바로 머클 패트리샤 트라이(Merkle Patricia Trie, MPT)다.

## 개념 설명

머클 패트리샤 트라이(MPT)는 이더리움이 전체 블록체인 상태를 저장하기 위해 사용하는 핵심 자료구조다. 이름에서 알 수 있듯 세 가지 고전적 개념을 결합했다.

- **Trie(트라이)**: 키를 문자 단위로 분해하여 계층적으로 저장하는 트리. 공통 접두사를 공유하는 키들이 같은 경로를 사용해 메모리를 절약한다.
- **Patricia Trie(PATRICIA: Practical Algorithm To Retrieve Information Coded in Alphanumeric)**: 단일 자식을 가진 내부 노드를 하나의 엣지로 압축한 Radix Trie. 경로 압축(path compression)으로 트라이의 비효율을 제거한다.
- **Merkle Tree**: 각 노드가 자식들의 해시를 포함하는 트리. 데이터 무결성 검증과 효율적인 증명(proof) 생성에 사용된다.

이더리움의 MPT는 이 세 가지를 결합하여 **암호학적으로 인증 가능한 키-값 저장소**를 만든다. 블록체인의 모든 계정 잔액, 스마트 컨트랙트 코드, 스토리지 상태가 MPT로 표현되며, 트라이의 루트 해시 하나만으로 전체 상태를 검증할 수 있다.

### MPT 노드 유형

MPT는 4가지 노드 유형으로 구성된다:

| 노드 유형 | 구조 | 설명 |
|-----------|------|------|
| **Null** | `""` | 빈 노드, 트라이가 비어있음 |
| **Branch** | `[v0, v1, ..., v15, value]` | 16개 자식 포인터 + 값 (17-요소 배열) |
| **Leaf** | `[encodedPath, value]` | 경로 끝의 실제 값 저장 |
| **Extension** | `[encodedPath, nextNode]` | 공통 prefix 압축하는 중간 노드 |

키는 nibble(4비트, 0~f) 단위로 분해되어 저장된다. 각 노드는 RLP(Recursive Length Prefix)로 직렬화되고, 32바이트 이상이면 keccak256 해시로 대체되어 디스크에 별도 저장된다.

### Compact Encoding (HP Encoding)

키의 경로를 저장할 때 HP(Hex Prefix) 인코딩을 사용하여 노드 타입과 nibble 수의 홀짝 여부를 표현한다:

```
nibble 수 짝수, Extension: prefix 00
nibble 수 홀수, Extension: prefix 1
nibble 수 짝수, Leaf:      prefix 20
nibble 수 홀수, Leaf:      prefix 3
```

예를 들어 `[1, 2, 3, 4, 5]` nibble 경로의 Leaf 노드는 `[0x35, 0x12, 0x34, 0x50]`으로 인코딩된다.

### 이더리움에서의 활용

이더리움 블록 헤더에는 세 가지 MPT 루트 해시가 포함된다:

- **stateRoot**: 전체 계정 상태 트라이의 루트 (주소 → 계정 정보)
- **transactionsRoot**: 블록의 트랜잭션 집합 트라이의 루트
- **receiptsRoot**: 트랜잭션 실행 영수증 트라이의 루트

## 왜 필요한가

이더리움처럼 탈중앙화된 블록체인에서는 수천 개의 노드가 동일한 상태를 독립적으로 검증해야 한다. 단순 해시맵이나 B-트리로는 다음 문제를 해결할 수 없다:

**1. 상태 증명(State Proof)**  
특정 계정의 잔액이 특정 블록 상태에 존재함을 O(log n) 크기의 증명으로 검증 가능해야 한다. MPT의 Merkle 속성이 이를 가능하게 한다. 증명 크기는 트라이 깊이에 비례하며, 수십억 개의 항목이 있어도 수십 개의 노드만으로 증명이 완성된다.

**2. 상태 일치 검증**  
두 노드가 동일한 상태를 가지는지 루트 해시 비교만으로 O(1)에 확인 가능하다. 단 하나의 계정이라도 다르면 루트 해시가 완전히 달라진다(암호학적 충돌 저항성).

**3. 경량 클라이언트(Light Client) 지원**  
전체 상태(수백 GB)를 저장하지 않는 모바일/브라우저 클라이언트도 Merkle proof만으로 특정 데이터를 검증 가능하다. EIP-1186은 `eth_getProof` RPC를 표준화하여 이를 지원한다.

**4. 효율적인 델타 업데이트**  
한 트랜잭션이 일부 상태를 변경하면, 변경된 경로의 노드들만 재계산하면 된다. 복잡도는 O(log n)이며, 나머지 노드들은 그대로 재사용된다.

## 실제 구현 예제

### 예제 1: Python으로 MPT 핵심 로직 구현

```python
import hashlib
from typing import Optional

def keccak256(data: bytes) -> bytes:
    """실제 구현에서는 pysha3 등 keccak256 라이브러리 사용"""
    return hashlib.sha256(data).digest()

def nibbles_to_bytes(nibbles: list) -> bytes:
    """nibble 리스트를 바이트로 변환"""
    result = []
    for i in range(0, len(nibbles), 2):
        if i + 1 < len(nibbles):
            result.append(nibbles[i] * 16 + nibbles[i + 1])
        else:
            result.append(nibbles[i] * 16)
    return bytes(result)

def encode_path(nibbles: list, is_leaf: bool) -> bytes:
    """HP(Hex Prefix) 인코딩 구현"""
    flag = 2 if is_leaf else 0
    if len(nibbles) % 2 == 1:
        # 홀수: prefix nibble에 flag+1을 포함
        prefix_nibble = flag + 1
        all_nibbles = [prefix_nibble] + nibbles
    else:
        # 짝수: 앞에 00(extension) 또는 20(leaf) 추가
        all_nibbles = [flag, 0] + nibbles
    return nibbles_to_bytes(all_nibbles)

def to_nibbles(key: bytes) -> list:
    """바이트 키를 nibble 리스트로 변환"""
    result = []
    for byte in key:
        result.append(byte >> 4)
        result.append(byte & 0x0F)
    return result

class MPTNode:
    """간단한 MPT 노드 표현"""
    pass

class LeafNode(MPTNode):
    def __init__(self, key_nibbles: list, value: bytes):
        self.key_nibbles = key_nibbles
        self.value = value
        self.encoded_path = encode_path(key_nibbles, is_leaf=True)

    def hash(self) -> bytes:
        # 실제로는 RLP 직렬화 후 keccak256
        data = self.encoded_path + self.value
        return keccak256(data)

    def __repr__(self):
        return f"Leaf(path={self.key_nibbles}, value={self.value!r})"

class ExtensionNode(MPTNode):
    def __init__(self, shared_nibbles: list, next_node: MPTNode):
        self.shared_nibbles = shared_nibbles
        self.next_node = next_node
        self.encoded_path = encode_path(shared_nibbles, is_leaf=False)

    def hash(self) -> bytes:
        data = self.encoded_path + self.next_node.hash()
        return keccak256(data)

    def __repr__(self):
        return f"Extension(prefix={self.shared_nibbles}, next={self.next_node})"

class BranchNode(MPTNode):
    def __init__(self):
        self.children = [None] * 16  # nibble 0~f 각 자식
        self.value: Optional[bytes] = None

    def hash(self) -> bytes:
        child_hashes = b''.join(
            child.hash() if child else b'' for child in self.children
        )
        data = child_hashes + (self.value or b'')
        return keccak256(data)

    def __repr__(self):
        active = [i for i, c in enumerate(self.children) if c is not None]
        return f"Branch(children={active}, value={self.value!r})"

# 간단한 MPT 조작 예시
def find_common_prefix(a: list, b: list) -> list:
    common = []
    for x, y in zip(a, b):
        if x != y:
            break
        common.append(x)
    return common

# MPT 구축 데모
key1 = to_nibbles(b'\x64\x6f')   # "do"
key2 = to_nibbles(b'\x64\x6f\x67')  # "dog"
key3 = to_nibbles(b'\x68\x6f\x72\x73\x65')  # "horse"

print(f"Key 'do'    nibbles: {key1}")
print(f"Key 'dog'   nibbles: {key2}")
print(f"Key 'horse' nibbles: {key3}")

# 'do'와 'dog'의 공통 prefix
common = find_common_prefix(key1, key2)
print(f"\nCommon prefix of 'do'/'dog': {common}")

leaf1 = LeafNode(key1[len(common):], b'verb')
leaf2 = LeafNode(key2[len(common):], b'puppy')

branch = BranchNode()
branch.children[key1[len(common)]] = leaf1  # 분기점에서 각 자식 배정
branch.children[key2[len(common)]] = leaf2

if common:
    root = ExtensionNode(common, branch)
else:
    root = branch

print(f"\nMPT Root: {root}")
print(f"Root hash: {root.hash().hex()[:16]}...")

# Leaf 노드 해시 검증
print(f"\nLeaf 'do' hash:  {leaf1.hash().hex()[:16]}...")
print(f"Leaf 'dog' hash: {leaf2.hash().hex()[:16]}...")
```

### 예제 2: JavaScript로 @ethereumjs/trie를 사용한 Merkle Proof 검증

```javascript
// npm install @ethereumjs/trie @ethereumjs/util
const { Trie } = require('@ethereumjs/trie');

async function demonstrateMPT() {
    // 해시 키 옵션 비활성화 (원본 키 경로 확인을 위해)
    const trie = new Trie({ useKeyHashing: false });

    // 이더리움 계정 상태 시뮬레이션
    const accounts = [
        { address: Buffer.from('1234567890abcdef1234567890abcdef12345678', 'hex'),
          balance: Buffer.from('0de0b6b3a7640000', 'hex') },  // 1 ETH
        { address: Buffer.from('abcdef1234567890abcdef1234567890abcdef12', 'hex'),
          balance: Buffer.from('1bc16d674ec80000', 'hex') },  // 2 ETH
        { address: Buffer.from('deadbeefdeadbeefdeadbeefdeadbeefdeadbeef', 'hex'),
          balance: Buffer.from('4563918244f40000', 'hex') },  // 5 ETH
    ];

    // 상태 트라이에 계정 삽입
    for (const account of accounts) {
        await trie.put(account.address, account.balance);
    }

    const rootHash = trie.root();
    console.log('State Root:', rootHash.toString('hex'));
    console.log('(이 해시 하나로 전체 상태를 인증할 수 있다)');

    // Merkle Proof 생성 및 검증
    const targetAddress = accounts[0].address;

    // 1. Full Node가 증명 생성
    const proof = await trie.createProof(targetAddress);
    console.log(`\nProof 노드 수: ${proof.length} (트라이 깊이에 비례)`);
    console.log(`Proof 총 크기: ${proof.reduce((s, n) => s + n.length, 0)} bytes`);

    // 2. Light Client가 루트 해시만으로 검증
    const verified = await Trie.verifyProof(rootHash, targetAddress, proof);
    console.log('\n[Light Client 검증]');
    console.log('검증된 잔액:', verified ? verified.toString('hex') : 'null');
    console.log('예상 잔액:  ', accounts[0].balance.toString('hex'));
    console.log('검증 성공:  ', verified?.equals(accounts[0].balance));

    // 3. 존재하지 않는 키의 부재 증명 (Non-existence Proof)
    const nonExistentAddr = Buffer.from(
        'ffffffffffffffffffffffffffffffffffffffff', 'hex'
    );
    const absenceProof = await trie.createProof(nonExistentAddr);
    const absentValue = await Trie.verifyProof(rootHash, nonExistentAddr, absenceProof);
    console.log('\n[존재하지 않는 계정 검증]');
    console.log('반환값:', absentValue);  // null → 부재 증명 성공

    // 4. 단일 항목 수정 후 루트 해시 변화 확인
    const newBalance = Buffer.from('8ac7230489e80000', 'hex');  // 10 ETH
    await trie.put(accounts[0].address, newBalance);
    const newRootHash = trie.root();
    console.log('\n[잔액 변경 후 루트 해시]');
    console.log('이전:', rootHash.toString('hex').substring(0, 16) + '...');
    console.log('이후:', newRootHash.toString('hex').substring(0, 16) + '...');
    console.log('동일 여부:', rootHash.equals(newRootHash));  // false: 해시 완전히 변경
}

demonstrateMPT().catch(console.error);
```

## 주의사항 및 팁

**1. 성능 트레이드오프를 이해하라**  
MPT는 암호학적 무결성을 위해 성능을 희생한다. 각 업데이트마다 경로 상의 모든 노드를 재해시해야 하며, 단순한 잔액 변경 하나가 수십 번의 해시 연산과 디스크 I/O를 유발할 수 있다. 이더리움이 Verkle Tree로 전환을 준비하는 주요 이유 중 하나다.

**2. 메모리 캐싱이 필수**  
프로덕션 이더리움 클라이언트(geth, besu, reth)는 모두 대용량 인메모리 트라이 캐시를 운영한다. 캐시 미스 시 디스크에서 노드를 읽어야 하므로, 적절한 캐시 크기 설정이 블록 처리 속도에 직결된다. geth의 `--cache` 플래그가 이를 제어한다.

**3. 트라이 프루닝에 주의**  
오래된 상태 노드는 안전하게 삭제할 수 있지만(state pruning), 체인 재조직(reorg) 가능성 때문에 최근 128개 블록의 상태는 반드시 보존해야 한다. 프루닝을 너무 공격적으로 하면 reorg 처리 시 상태를 재계산해야 하는 문제가 생긴다.

**4. 키 해싱의 의미**  
이더리움에서 계정 주소(20바이트)를 트라이 키로 사용할 때 keccak256으로 해시하여 사용한다. 이렇게 하면 키가 균일하게 분포되어 트라이의 균형이 자연스럽게 유지된다. 만약 주소를 직접 키로 쓰면 특정 접두사를 가진 주소들이 트라이의 특정 경로에 집중되어 불균형이 발생할 수 있다.

**5. EIP-1186 Merkle Proof 활용**  
`eth_getProof` JSON-RPC를 사용하면 특정 계정과 스토리지 슬롯에 대한 Merkle proof를 받을 수 있다. ZK-롤업의 검증자, 크로스체인 브리지, 옵티미스틱 롤업의 fraud proof 시스템이 모두 이 API를 핵심적으로 활용한다. 라이트 클라이언트 지갑(MetaMask의 Snaps 등)도 이를 통해 신뢰 최소화 검증을 구현한다.

**6. Verkle Tree와의 차이**  
이더리움은 MPT를 Verkle Tree(벡터 커밋먼트 기반)로 교체할 계획이다(EIP-6800). Verkle Tree는 증명 크기를 O(depth) → O(1)에 가깝게 줄여 스테이트리스 클라이언트를 가능하게 한다. MPT의 개념을 이해하면 Verkle Tree로의 전환 의의도 더 명확하게 파악할 수 있다.

## 참고 자료

- [ethereum/ethereumjs-monorepo — @ethereumjs/trie 구현](https://github.com/ethereumjs/ethereumjs-monorepo)
- [statelyai/xstate (상태 기계 참고)](https://github.com/statelyai/xstate)
- [wiredtiger/wiredtiger](https://github.com/wiredtiger/wiredtiger)
