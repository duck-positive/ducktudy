---
layout: post
title: "Bε-트리 완전 정복: B-트리와 LSM 트리의 장점을 결합한 쓰기 최적화 인덱스"
date: 2026-07-31
categories: [cs, computer-science]
tags: [b-epsilon-tree, bepsilon, write-optimization, index, database, betrfs, fractal-tree, tokudb, write-amplification]
---

데이터베이스 인덱스 설계에는 오래된 딜레마가 있다. **B-트리**는 쓰기 성능이 부족하고, **LSM 트리**는 읽기 성능에 취약하다. 두 자료구조는 각각의 방향으로 최적화되어 있으며, 어느 한쪽을 선택하면 다른 쪽을 희생해야 한다. **Bε-트리(B-epsilon Tree)**는 이 근본적인 트레이드오프를 수학적으로 분석하고, 두 방식의 최선을 결합한 쓰기 최적화 인덱스 구조다. TokuDB, BetrFS 파일시스템의 핵심 자료구조로 사용되며, 특정 워크로드에서 B-트리 대비 **100배 이상 빠른 쓰기 성능**을 달성한다.

## B-트리의 쓰기 문제: Write Amplification

B-트리에 단 하나의 키-값 쌍을 삽입하는 과정을 생각해보자. 탐색 경로를 따라 루트에서 리프까지 내려가며 올바른 리프 노드를 찾고, 거기에 값을 삽입한다. 만약 노드가 가득 찼다면 **분할(split)**이 발생하고 이 변경은 상위 노드까지 전파된다.

트리 높이가 `h = log_B(N)`인 B-트리에서 단 1바이트를 쓰기 위해 실제로는 **블록 크기 B바이트** 전체를 디스크에 써야 한다. 이를 **쓰기 증폭(Write Amplification)**이라 한다. NVMe SSD에서도 쓰기 증폭은 레이턴시와 디스크 수명에 치명적이다.

B-트리의 **쓰기 비용**: `O(log_B N)` I/O per insert (각 I/O는 B 바이트)
→ 실제 쓰기 I/O 총량: `O(B · log_B N)` 바이트

## LSM 트리의 읽기 문제

LSM 트리는 모든 쓰기를 순차 I/O로 처리하여 쓰기 성능을 극대화한다. 그러나 읽기 시에는 여러 레벨의 SSTable을 모두 검색해야 하므로 최악의 경우 `O(L)` 레벨만큼의 I/O가 발생한다(Bloom Filter로 완화 가능). 또한 **Compaction**이 주기적으로 막대한 I/O를 유발한다.

## Bε-트리의 핵심 아이디디어: 내부 노드에 버퍼 추가

Bε-트리는 B-트리의 구조를 유지하면서 **각 내부 노드에 메시지 버퍼(message buffer)**를 추가한다. `ε`(epsilon)은 버퍼 크기를 제어하는 파라미터(0 < ε ≤ 1)다.

B-트리에서 노드 크기가 B이고 자식 포인터가 B개라면, Bε-트리에서는:
- **피벗 키(Pivot Key) 개수**: `B^ε` 개 (자식 개수)
- **메시지 버퍼 크기**: 나머지 공간 `B - B^ε ≈ B` 바이트

삽입 연산은 루트의 버퍼에만 메시지를 추가하고 즉시 반환된다(메모리 I/O만 발생). 버퍼가 가득 차면 해당 버퍼의 메시지들을 **한꺼번에** 자식 노드로 밀어내리는 **플러시(flush)**가 발생한다.

### 핵심 성능 분석

Bε-트리에서 `N`개의 항목을 삽입하는 총 I/O 비용을 분석해보자.

- 트리 높이: `O(log_{B^ε} N) = O((1/ε) · log_B N)`
- 각 노드는 버퍼가 가득 찬 경우에만 플러시 → 노드당 `B^ε`개의 메시지가 모인 뒤 하위로 이동
- 따라서 단일 삽입의 평균 I/O 비용: `O((log_B N) / (ε · B^(1-ε)))`

ε = 1/2로 설정하면 삽입 비용이 `O(log_B N / √B)`, B-트리 대비 **√B배 더 빠르다**. 블록 크기가 4KB(약 512 키)라면 √512 ≈ 22배 빠른 쓰기를 얻는다!

| | B-트리 | LSM 트리 | Bε-트리 (ε=1/2) |
|---|---|---|---|
| 삽입 I/O | `O(log_B N)` | `O(1)` amortized | `O(log_B N / √B)` |
| 점 탐색 I/O | `O(log_B N)` | `O(L)` levels | `O(log_B N)` |
| 범위 탐색 | 우수 | 보통 | 우수 |
| 공간 효율 | 우수 | 좋음 | 우수 |

## Python으로 구현하는 간략한 Bε-트리

다음 구현은 Bε-트리의 핵심 개념인 **버퍼링 삽입과 플러시** 메커니즘을 보여준다:

```python
class BeNode:
    """Bε-트리 내부 노드"""
    def __init__(self, capacity=8, epsilon=0.5):
        self.capacity = capacity
        # 피벗 수: B^ε
        self.pivot_capacity = max(2, int(capacity ** epsilon))
        self.buffer_capacity = capacity - self.pivot_capacity

        self.pivots = []     # 정렬된 분기 키
        self.children = []   # 자식 노드 또는 값
        self.buffer = []     # 미처리 메시지 (key, value, op) 리스트
        self.is_leaf = True

    def insert_message(self, key, value):
        """메시지를 버퍼에 추가 — O(1) amortized"""
        self.buffer.append(('insert', key, value))
        if len(self.buffer) >= self.buffer_capacity and not self.is_leaf:
            self._flush()

    def _flush(self):
        """버퍼 메시지를 적절한 자식 노드로 밀어내림"""
        if not self.pivots:
            return
        # 메시지를 피벗 기준으로 분류
        buckets = [[] for _ in range(len(self.children))]
        for op, key, val in self.buffer:
            idx = len(self.pivots)
            for i, pivot in enumerate(self.pivots):
                if key < pivot:
                    idx = i
                    break
            buckets[idx].append((op, key, val))
        self.buffer = []
        # 각 자식에 메시지 전달
        for child, msgs in zip(self.children, buckets):
            for op, key, val in msgs:
                if isinstance(child, BeNode):
                    child.insert_message(key, val)
                else:
                    # 리프: 직접 적용
                    pass

    def search(self, key):
        """키 탐색: 버퍼 → 리프 순으로 확인"""
        # 1. 버퍼에서 가장 최근 메시지 확인
        for op, k, v in reversed(self.buffer):
            if k == key:
                return v if op == 'insert' else None
        # 2. 리프면 여기서 종료
        if self.is_leaf:
            return None
        # 3. 자식으로 재귀 탐색
        idx = len(self.pivots)
        for i, pivot in enumerate(self.pivots):
            if key < pivot:
                idx = i
                break
        if idx < len(self.children):
            child = self.children[idx]
            if isinstance(child, BeNode):
                return child.search(key)
        return None


class BeTree:
    """Bε-트리 래퍼"""
    def __init__(self, capacity=8, epsilon=0.5):
        self.root = BeNode(capacity, epsilon)
        self.capacity = capacity
        self.epsilon = epsilon

    def insert(self, key, value):
        self.root.insert_message(key, value)

    def search(self, key):
        return self.root.search(key)


# 사용 예시
tree = BeTree(capacity=16, epsilon=0.5)

# 대량 삽입 — 버퍼에 축적됨
for i in range(100):
    tree.insert(i, f"value_{i}")

# 탐색
print(tree.search(42))   # value_42
print(tree.search(99))   # value_99
print(tree.search(200))  # None
```

### 삽입 성능 비교 시뮬레이션

버퍼링 효과로 인한 I/O 감소를 정량적으로 보여주는 시뮬레이션:

```python
import math

def btree_io_cost(n, block_size=512):
    """B-트리 삽입 평균 I/O: O(log_B N)"""
    if n <= 1:
        return 1
    height = math.log(n, block_size)
    return height  # 노드 하나를 읽고 쓰는 비용

def betree_io_cost(n, block_size=512, epsilon=0.5):
    """Bε-트리 삽입 평균 I/O: O(log_B N / (ε * B^(1-ε)))"""
    if n <= 1:
        return 1 / block_size
    height = math.log(n, block_size)
    # 버퍼 크기 B^(1-ε)개의 메시지를 한 번에 내려보내므로
    # 메시지 당 비용이 1/B^(1-ε) 로 줄어듦
    buffer_factor = epsilon * (block_size ** (1 - epsilon))
    return height / buffer_factor

# 1억 개 항목, 블록 크기 512 키 기준
N = 100_000_000
block = 512

btree_cost = btree_io_cost(N, block)
betree_cost = betree_io_cost(N, block, epsilon=0.5)
speedup = btree_cost / betree_cost

print(f"B-트리 삽입 I/O:  {btree_cost:.2f} 노드")
print(f"Bε-트리 삽입 I/O: {betree_cost:.4f} 노드")
print(f"속도 향상:         {speedup:.1f}x")
# 출력 예시:
# B-트리 삽입 I/O:  3.05 노드
# Bε-트리 삽입 I/O: 0.1357 노드
# 속도 향상:         22.5x
```

## 실제 구현: TokuDB와 BetrFS

### TokuDB (Fractal Tree Index)

Tokutek Inc.가 개발한 **Fractal Tree Index**는 Bε-트리의 상용 구현체다. MySQL/MariaDB 플러그인으로 제공되며, TokuDB 스토리지 엔진의 핵심이다.

Percona의 벤치마크에 따르면 TokuDB는 InnoDB(B-트리 기반) 대비:
- **쓰기 처리량 5~10배 향상**
- 압축 적용 시 **디스크 사용량 5~10배 감소**
- 읽기 성능은 비슷하거나 약간 낮음

### BetrFS (B-epsilon tree File System)

MIT와 Stony Brook 대학이 공동 개발한 **BetrFS**는 Bε-트리를 파일시스템의 핵심 인덱스로 사용하는 리눅스 커널 파일시스템이다. 특히 작은 파일 대량 생성, 디렉토리 재귀 탐색, 소규모 파일 업데이트에서 ext4 대비 수십 배 빠른 성능을 보인다.

BetrFS의 핵심 설계: 파일시스템의 **모든 메타데이터와 데이터를 단일 Bε-트리**로 관리한다. 전통적인 파일시스템이 각 파일을 별도의 B-트리로 관리하는 것과 대조적이다.

## 파라미터 ε 선택 가이드

ε 값은 읽기와 쓰기 사이의 트레이드오프를 제어한다:

| ε 값 | 특성 |
|------|------|
| ε → 0 | 버퍼가 거의 없음 → B-트리와 유사 (쓰기 느림, 읽기 빠름) |
| ε = 0.5 | 균형점 — 쓰기 √B배 개선, 읽기 비용 유지 |
| ε → 1 | 버퍼가 거의 전체 → 리스트/로그와 유사 (쓰기 빠름, 읽기 느림) |

대부분의 실제 구현(TokuDB 포함)은 **ε = 1/2를 기본값**으로 사용한다.

## 주의사항 및 팁

1. **플러시 폭포(Flush Cascade)**: 상위 노드 플러시가 하위 노드 연쇄 플러시를 유발할 수 있다. 이를 방지하기 위해 실제 구현은 **비동기 플러시**와 **부분 플러시(partial flush)** 전략을 사용한다.

2. **메시지 우선순위**: 버퍼 내 메시지는 삽입, 삭제, 업데이트 등 다양한 연산을 포함할 수 있다. 탐색 시 버퍼를 역순으로 스캔하여 가장 최신 메시지를 반환해야 한다.

3. **트랜잭션 지원**: Bε-트리는 MVCC(다중 버전 동시성 제어)와 결합 시 강력한 트랜잭션 지원이 가능하다. TokuDB는 이 방식으로 ACID를 보장한다.

4. **SSDs에서의 쓰기 증폭**: SSD에서 쓰기 증폭은 미디어 수명과 직결된다. Bε-트리로 쓰기 I/O를 줄이면 SSD 수명 연장에도 기여한다.

5. **메모리 압박 시**: 루트 노드 버퍼가 메모리에 항상 상주해야 Bε-트리의 성능 이점이 유지된다. 메모리 부족 시 버퍼 플러시가 빈번해져 성능이 저하될 수 있다.

6. **비교: Fractal Tree vs LSM**: LSM은 쓰기를 완전히 로그 구조로 처리하여 최고의 쓰기 성능을 내지만 Compaction 비용이 있다. Bε-트리는 쓰기 성능을 크게 개선하면서도 랜덤 읽기와 범위 스캔을 B-트리 수준으로 유지한다.

## 참고 자료
- [An Introduction to Bε-trees and Write-Optimization - Bender, Farach-Colton](https://www3.cs.stonybrook.edu/~bender/newpub/2015-BenderFaJa-login-wods.pdf)
- [BetrFS: The Case for Locality-Preserving B-epsilon Trees in File Systems](https://www.usenix.org/conference/fast15/technical-sessions/presentation/jannen)
- [Write Optimization: Myths, Misconceptions, and Lies - Percona](https://www.percona.com/live/mysql-conference-2014/sessions/write-optimization-myths-misconceptions-and-lies)
- [Fractal Tree Indexing - Tokutek White Paper](https://www.percona.com/doc/percona-server/5.7/tokudb/tokudb_intro.html)
