---
layout: post
title: "Huffman 코딩 완전 정복: 엔트로피를 활용한 최적 가변 길이 인코딩과 압축 원리"
date: 2026-06-30
categories: [cs, computer-science]
tags: [algorithms, compression, huffman, entropy, prefix-code, data-structure, priority-queue]
---

## 개념 설명

**Huffman 코딩**은 1952년 MIT 박사과정생 David A. Huffman이 발표한 최적의 무손실 데이터 압축 알고리즘이다. 핵심 아이디어는 단순하다: **자주 등장하는 기호에는 짧은 비트 코드를, 드물게 등장하는 기호에는 긴 비트 코드를 할당**하여 전체 비트 수를 최소화한다.

Huffman 코딩은 PNG, JPEG, MP3, ZIP, gzip, HTTP/2의 헤더 압축(HPACK) 등 수많은 현대 포맷의 핵심 구성 요소다.

### 핵심 개념

| 개념 | 설명 |
|------|------|
| **엔트로피 (Entropy)** | 정보 이론적 최소 인코딩 비트 수의 하한. H = -Σ p(x) log₂ p(x) |
| **프리픽스 코드 (Prefix Code)** | 어떤 코드도 다른 코드의 접두사가 되지 않는 가변 길이 코드. 모호함 없이 디코딩 가능 |
| **Huffman 트리** | 빈도가 낮은 기호일수록 잎이 깊게 위치하는 이진 트리 |
| **코드북 (Codebook)** | 각 기호와 그에 대응하는 비트 문자열의 테이블 |

---

## 왜 필요한가

### 고정 길이 인코딩의 낭비

ASCII는 모든 문자를 7비트(또는 8비트)로 인코딩한다. 'e'가 'q'보다 100배 자주 등장해도 두 문자는 같은 8비트를 사용한다.

예를 들어 "ABRACADABRA" (11글자, 5가지 기호)를 고정 길이로 표현하면 3비트/기호 × 11 = **33비트**가 필요하다.

Huffman 코딩으로 빈도 분석을 하면:

| 기호 | 빈도 | 확률 | Huffman 코드 | 비트 수 |
|------|------|------|--------------|---------|
| A    | 5    | 5/11 | `0`          | 1       |
| B    | 2    | 2/11 | `10`         | 2       |
| R    | 2    | 2/11 | `110`        | 3       |
| C    | 1    | 1/11 | `1110`       | 4       |
| D    | 1    | 1/11 | `1111`       | 4       |

평균 비트 수 = (5×1 + 2×2 + 2×3 + 1×4 + 1×4) / 11 = **23비트** → **30% 압축**

### Shannon 엔트로피와의 관계

Shannon 엔트로피는 기호당 최소 필요 비트 수의 이론적 하한이다:

H = -(5/11)log₂(5/11) - (2/11)log₂(2/11) - ... ≈ **2.04 비트/기호**

Huffman의 결과 23/11 ≈ **2.09 비트/기호**로 이론적 최적에 거의 근접한다. Huffman 코딩은 기호 확률이 2의 거듭제곱인 경우에만 엔트로피를 정확히 달성한다.

---

## 실제 구현 예제

### 예제 1: Huffman 코딩 인코더/디코더 완전 구현 (Python)

```python
import heapq
from collections import Counter
from dataclasses import dataclass, field
from typing import Optional, Dict


@dataclass(order=True)
class HuffmanNode:
    freq: int
    symbol: Optional[str] = field(default=None, compare=False)
    left: Optional['HuffmanNode'] = field(default=None, compare=False)
    right: Optional['HuffmanNode'] = field(default=None, compare=False)

    def is_leaf(self) -> bool:
        return self.left is None and self.right is None


def build_huffman_tree(text: str) -> HuffmanNode:
    """빈도 테이블 → 우선순위 큐 → Huffman 트리 생성"""
    freq = Counter(text)

    # 최소 힙에 잎 노드 삽입 (freq 기준 오름차순)
    heap = [HuffmanNode(f, sym) for sym, f in freq.items()]
    heapq.heapify(heap)

    # 단일 기호인 경우 예외 처리
    if len(heap) == 1:
        only = heapq.heappop(heap)
        return HuffmanNode(only.freq, left=only, right=HuffmanNode(0))

    while len(heap) > 1:
        # 빈도가 가장 낮은 두 노드를 꺼내 내부 노드로 합침
        left = heapq.heappop(heap)
        right = heapq.heappop(heap)
        merged = HuffmanNode(
            freq=left.freq + right.freq,
            left=left,
            right=right
        )
        heapq.heappush(heap, merged)

    return heap[0]


def build_codebook(root: HuffmanNode) -> Dict[str, str]:
    """트리를 DFS로 순회하여 코드북 생성"""
    codebook = {}

    def dfs(node: HuffmanNode, code: str) -> None:
        if node.is_leaf():
            codebook[node.symbol] = code or '0'  # 단일 기호 예외
            return
        if node.left:
            dfs(node.left, code + '0')
        if node.right:
            dfs(node.right, code + '1')

    dfs(root, '')
    return codebook


def huffman_encode(text: str) -> tuple[str, Dict[str, str]]:
    """텍스트를 Huffman 인코딩하여 (비트열, 코드북) 반환"""
    root = build_huffman_tree(text)
    codebook = build_codebook(root)
    encoded = ''.join(codebook[ch] for ch in text)
    return encoded, codebook


def huffman_decode(bits: str, codebook: Dict[str, str]) -> str:
    """비트열과 코드북으로 원문 복원 (역방향 코드북 사용)"""
    reverse_codebook = {v: k for k, v in codebook.items()}
    result = []
    buffer = ''

    for bit in bits:
        buffer += bit
        if buffer in reverse_codebook:
            result.append(reverse_codebook[buffer])
            buffer = ''

    if buffer:
        raise ValueError(f"디코딩 실패: 남은 비트 '{buffer}'")

    return ''.join(result)


# --- 실행 및 검증 ---
text = "ABRACADABRA"
encoded, codebook = huffman_encode(text)
decoded = huffman_decode(encoded, codebook)

print(f"원문: {text}")
print(f"코드북: {codebook}")
print(f"인코딩 결과: {encoded}")
print(f"원문 비트 수 (고정 3비트): {len(text) * 3}비트")
print(f"Huffman 비트 수: {len(encoded)}비트")
print(f"압축률: {(1 - len(encoded)/(len(text)*3))*100:.1f}%")
print(f"복원: {decoded}")
print(f"원문과 일치: {text == decoded}")

# Shannon 엔트로피 계산
import math
freq = Counter(text)
n = len(text)
entropy = -sum((c/n) * math.log2(c/n) for c in freq.values())
avg_bits = len(encoded) / len(text)
print(f"\nShannon 엔트로피: {entropy:.4f} 비트/기호")
print(f"Huffman 평균 비트: {avg_bits:.4f} 비트/기호")
print(f"엔트로피 대비 오버헤드: {(avg_bits - entropy):.4f} 비트/기호")
```

### 예제 2: Canonical Huffman Coding — 실용적인 포맷 저장

실제 파일 압축 포맷에서는 **Canonical Huffman** 코드를 사용한다. 이는 코드 길이만 저장하면 수신자가 표준 규칙으로 동일한 코드북을 재구성할 수 있어 헤더 크기를 크게 줄인다. gzip, zlib, PNG가 이 방식을 사용한다.

```python
from collections import defaultdict
from typing import Dict, List, Tuple


def build_canonical_codebook(
    text: str
) -> Tuple[Dict[str, str], Dict[str, int]]:
    """
    Canonical Huffman 코드북 생성.
    코드 길이만 저장해도 수신자가 재구성 가능한 표준 형식.
    """
    # 1. 일반 Huffman으로 코드 길이 계산
    _, codebook = huffman_encode(text)
    code_lengths: Dict[str, int] = {sym: len(code) for sym, code in codebook.items()}

    # 2. 코드 길이 기준으로 정렬 (같은 길이는 알파벳 순)
    sorted_symbols = sorted(code_lengths.items(), key=lambda x: (x[1], x[0]))

    # 3. Canonical 코드 할당 규칙:
    #    - 같은 길이 내에서 사전순으로 오름차순
    #    - 길이가 늘어날 때는 이전 코드에 1을 더하고 왼쪽 시프트
    canonical = {}
    code = 0
    prev_length = 0

    for symbol, length in sorted_symbols:
        if prev_length > 0:
            code = (code + 1) << (length - prev_length)
        canonical[symbol] = format(code, f'0{length}b')
        prev_length = length
        code = int(canonical[symbol], 2)

    return canonical, code_lengths


def reconstruct_canonical_codebook(
    code_lengths: Dict[str, int]
) -> Dict[str, str]:
    """코드 길이 정보만으로 Canonical 코드북 재구성 (디코더 측)"""
    sorted_symbols = sorted(code_lengths.items(), key=lambda x: (x[1], x[0]))
    canonical = {}
    code = 0
    prev_length = 0

    for symbol, length in sorted_symbols:
        if prev_length > 0:
            code = (code + 1) << (length - prev_length)
        canonical[symbol] = format(code, f'0{length}b')
        prev_length = length
        code = int(canonical[symbol], 2)

    return canonical


# --- 검증 ---
text = "this is an example of a huffman tree"
canonical, lengths = build_canonical_codebook(text)
reconstructed = reconstruct_canonical_codebook(lengths)

print("Canonical Huffman 코드북:")
for sym, code in sorted(canonical.items(), key=lambda x: (len(x[1]), x[0])):
    print(f"  '{sym}': {code} (길이 {len(code)})")

# 저장할 메타데이터 크기 비교
print(f"\n전체 코드북 저장 시: {sum(len(c) for c in canonical.values())} 비트")
print(f"길이만 저장 시: {len(lengths)} 개의 (기호, 길이) 쌍")

# 원본 코드북과 재구성된 코드북이 동일한지 확인
assert canonical == reconstructed, "Canonical 코드북 재구성 실패!"
print("\nCanonical 코드북 재구성 성공 (길이 정보만으로 완전 복원)")
```

---

## 주의사항 및 팁

### 1. 적응형 Huffman vs 정적 Huffman

정적 Huffman은 전체 데이터를 먼저 훑어 빈도를 계산해야 한다(Two-pass). 스트리밍 데이터에서는 **적응형 Huffman(Adaptive Huffman)** 알고리즘을 사용하며, 기호를 읽으면서 트리를 실시간으로 업데이트한다.

### 2. 기호 확률이 2의 거듭제곱이 아닐 때의 비효율

Huffman의 유일한 이론적 단점은 기호 확률이 2의 거듭제곱이 아닐 때 비트를 낭비한다는 점이다. 예를 들어 p(A) = 0.9이면 Huffman은 1비트를 할당하지만 이론적으로는 -log₂(0.9) ≈ 0.15비트면 충분하다. **산술 코딩(Arithmetic Coding)**은 이 비효율을 제거하여 엔트로피에 더 근접하지만 구현이 복잡하다.

### 3. 코드북을 함께 전송해야 한다

Huffman 압축 파일에는 반드시 코드북(또는 코드 길이 테이블)이 포함되어야 한다. 짧은 파일을 압축하면 코드북 오버헤드 때문에 오히려 파일이 커질 수 있다.

### 4. 같은 데이터에 대해 여러 최적 Huffman 트리가 존재한다

빈도가 같은 기호들의 처리 순서에 따라 트리 구조가 달라지므로, 항상 동일한 최적 코드북이 존재하는 것은 아니다. 이를 표준화한 것이 Canonical Huffman이다.

### 5. gzip과 DEFLATE에서의 실용

실제 gzip은 Huffman + LZ77(LZ77은 반복 문자열 참조로 사전 압축)을 결합한 DEFLATE 알고리즘을 사용한다. LZ77이 반복 패턴을 제거한 후, Huffman이 남은 기호를 비트 최적으로 인코딩한다.

```python
import zlib

# Python의 zlib은 DEFLATE(LZ77 + Huffman)를 사용
original = b"ABRACADABRA" * 1000
compressed = zlib.compress(original, level=9)
ratio = len(compressed) / len(original)
print(f"원본: {len(original)} bytes")
print(f"압축: {len(compressed)} bytes")
print(f"압축률: {ratio:.3f} ({(1-ratio)*100:.1f}% 감소)")
assert zlib.decompress(compressed) == original
```

---

## 요약

Huffman 코딩은 Shannon 엔트로피라는 이론적 한계에 거의 도달하는 최적 프리픽스 코드를 O(n log n) 시간에 구성한다. 70년이 지난 지금도 PNG, gzip, HPACK 등 수십 개의 현대 포맷에서 핵심 구성 요소로 쓰이는 이유는, 구현의 단순함과 최적성의 증명이 완벽하게 맞아 떨어지기 때문이다. 압축 알고리즘을 공부한다면 Huffman을 먼저 이해하는 것이 필수다.

---

## 참고 자료

- [Huffman Coding — Wikipedia](https://en.wikipedia.org/wiki/Huffman_coding)
- [A Method for the Construction of Minimum-Redundancy Codes (Huffman 1952 원본 논문)](https://www.semanticscholar.org/paper/A-method-for-the-construction-of-minimum-redundancy-Huffman/e9c0ea21e2a91deefb4a7c30a8e37ff7a3d3e8f0)
- [Huffman Coding Algorithm — GeeksforGeeks](https://www.geeksforgeeks.org/huffman-coding-greedy-algo-3/)
- [DEFLATE Compressed Data Format Specification (RFC 1951)](https://datatracker.ietf.org/doc/html/rfc1951)
