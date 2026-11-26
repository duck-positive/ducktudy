---
layout: post
title: "B-Tree vs LSM-Tree 완전 정복: 데이터베이스 스토리지 엔진의 핵심 인덱스 구조 선택 가이드"
date: 2026-11-26
categories: [cs, computer-science]
tags: [B-Tree, LSM-Tree, 데이터베이스, 인덱스, RocksDB, PostgreSQL, 스토리지엔진]
---

데이터베이스의 성능을 결정하는 가장 중요한 요소 중 하나는 스토리지 엔진이 어떤 인덱스 자료구조를 사용하는가입니다. PostgreSQL, MySQL InnoDB, SQLite는 B-Tree를, Cassandra, RocksDB, LevelDB는 LSM-Tree를 채택합니다. 각각의 선택에는 명확한 이유가 있습니다.

## B-Tree란 무엇인가?

B-Tree(Balanced Tree)는 1970년 Rudolf Bayer와 Edward McCreight가 고안한 자가 균형 트리로, 디스크 I/O를 최소화하도록 설계되었습니다. 각 노드가 여러 자식을 가질 수 있으며, 모든 리프 노드가 같은 깊이를 유지합니다.

### B-Tree의 핵심 특징

**노드(페이지) 구조:** B-Tree의 기본 단위는 고정 크기 페이지(보통 4KB~16KB)입니다. 각 페이지는 정렬된 키-값 쌍을 저장하며, 삽입/삭제 시 페이지 분할이나 병합이 발생합니다.

**읽기 성능:** 루트에서 리프까지 O(log N) 단계만에 임의 접근이 가능하고, 범위 쿼리(range query)에 최적화되어 있습니다.

**쓰기 방식:** 업데이트는 해당 페이지를 직접 수정(in-place update)합니다. WAL(Write-Ahead Log)로 내구성을 보장합니다.

## LSM-Tree란 무엇인가?

LSM-Tree(Log-Structured Merge Tree)는 1996년 Patrick O'Neil 등이 쓰기 성능을 극대화하기 위해 고안했습니다. 쓰기를 순차 I/O로 처리해 랜덤 디스크 접근을 최소화합니다.

### LSM-Tree의 핵심 특징

**쓰기 경로:** 모든 쓰기는 먼저 메모리의 MemTable(보통 레드-블랙 트리)에 저장됩니다. MemTable이 가득 차면 불변(immutable) SSTable(Sorted String Table)로 디스크에 플러시됩니다.

**컴팩션(Compaction):** 디스크에 쌓인 여러 SSTable 레이어를 주기적으로 병합해 중복과 삭제 마커(tombstone)를 제거합니다. 쓰기 증폭(Write Amplification)이 발생하는 지점입니다.

**읽기 경로:** MemTable → 최신 L0 SSTable → L1 → Ln 순서로 탐색합니다. 최악의 경우 여러 파일을 열어봐야 합니다. Bloom Filter로 불필요한 파일 I/O를 필터링합니다.

## 직접 구현으로 이해하는 차이점

### B-Tree 간단 구현 (Python)

```python
class BTreeNode:
    def __init__(self, t: int, leaf: bool = False):
        self.t = t            # 최소 차수 (minimum degree)
        self.keys = []        # 정렬된 키 목록
        self.values = []      # 키에 대응하는 값
        self.children = []    # 자식 노드들
        self.leaf = leaf

class BTree:
    def __init__(self, t: int = 3):
        self.t = t
        self.root = BTreeNode(t, leaf=True)

    def search(self, node: BTreeNode, key: int):
        # 현재 노드에서 key 위치 찾기
        i = 0
        while i < len(node.keys) and key > node.keys[i]:
            i += 1

        if i < len(node.keys) and node.keys[i] == key:
            return node.values[i]  # 찾음

        if node.leaf:
            return None  # 없음

        return self.search(node.children[i], key)

    def insert(self, key: int, value):
        root = self.root
        # 루트가 꽉 찼으면 분할 (높이 증가)
        if len(root.keys) == 2 * self.t - 1:
            new_root = BTreeNode(self.t, leaf=False)
            new_root.children.append(self.root)
            self._split_child(new_root, 0)
            self.root = new_root

        self._insert_non_full(self.root, key, value)

    def _insert_non_full(self, node: BTreeNode, key: int, value):
        i = len(node.keys) - 1
        if node.leaf:
            # 올바른 위치에 삽입 (in-place 수정 — B-Tree의 핵심)
            node.keys.append(None)
            node.values.append(None)
            while i >= 0 and node.keys[i] > key:
                node.keys[i + 1] = node.keys[i]
                node.values[i + 1] = node.values[i]
                i -= 1
            node.keys[i + 1] = key
            node.values[i + 1] = value
        else:
            while i >= 0 and node.keys[i] > key:
                i -= 1
            i += 1
            if len(node.children[i].keys) == 2 * self.t - 1:
                self._split_child(node, i)
                if key > node.keys[i]:
                    i += 1
            self._insert_non_full(node.children[i], key, value)

    def _split_child(self, parent: BTreeNode, i: int):
        t = self.t
        full_child = parent.children[i]
        new_child = BTreeNode(t, leaf=full_child.leaf)

        # 중간 키를 부모로 올림
        mid = t - 1
        parent.keys.insert(i, full_child.keys[mid])
        parent.values.insert(i, full_child.values[mid])
        parent.children.insert(i + 1, new_child)

        # 오른쪽 절반을 새 자식으로
        new_child.keys = full_child.keys[mid + 1:]
        new_child.values = full_child.values[mid + 1:]
        if not full_child.leaf:
            new_child.children = full_child.children[mid + 1:]
            full_child.children = full_child.children[:mid + 1]

        full_child.keys = full_child.keys[:mid]
        full_child.values = full_child.values[:mid]


# 사용 예
tree = BTree(t=3)
for i in [10, 20, 5, 6, 12, 30, 7, 17]:
    tree.insert(i, f"value_{i}")

print(tree.search(tree.root, 6))   # value_6
print(tree.search(tree.root, 99))  # None
```

### LSM-Tree 핵심 구조 구현 (Python)

```python
import time
from sortedcontainers import SortedDict

class SSTable:
    """불변(Immutable) 정렬 파일 — 한번 쓰면 수정 불가"""
    def __init__(self, data: dict, timestamp: float):
        self.data = SortedDict(data)  # 정렬된 키-값 쌍
        self.timestamp = timestamp
        # 실제 구현에서는 Bloom Filter로 불필요한 탐색을 줄임
        self.bloom_filter = set(data.keys())

    def get(self, key: str):
        if key not in self.bloom_filter:
            return None  # Bloom Filter: 없으면 확실히 없음
        return self.data.get(key)

    def __len__(self):
        return len(self.data)


class LSMTree:
    def __init__(self, memtable_limit: int = 4):
        self.memtable = SortedDict()     # 인메모리 버퍼 (쓰기 먼저 여기로)
        self.memtable_limit = memtable_limit
        self.sstables: list[SSTable] = []  # 디스크 레이어 (최신 → 오래된 순)

    def put(self, key: str, value: str):
        """모든 쓰기는 MemTable로 → 순차 I/O만 발생 (랜덤 없음!)"""
        self.memtable[key] = value
        if len(self.memtable) >= self.memtable_limit:
            self._flush()

    def delete(self, key: str):
        """삭제는 Tombstone 마커 삽입 (실제 삭제는 컴팩션 때)"""
        self.memtable[key] = "__TOMBSTONE__"
        if len(self.memtable) >= self.memtable_limit:
            self._flush()

    def get(self, key: str):
        """읽기: MemTable → 최신 SSTable → 오래된 SSTable 순 탐색"""
        # 1. MemTable 먼저 확인 (가장 최신)
        if key in self.memtable:
            val = self.memtable[key]
            return None if val == "__TOMBSTONE__" else val

        # 2. SSTable을 최신 순으로 탐색
        for sst in reversed(self.sstables):
            val = sst.get(key)
            if val is not None:
                return None if val == "__TOMBSTONE__" else val

        return None

    def _flush(self):
        """MemTable을 디스크 SSTable로 플러시 (순차 쓰기)"""
        sst = SSTable(dict(self.memtable), time.time())
        self.sstables.append(sst)
        self.memtable.clear()
        print(f"  [Flush] MemTable → SSTable (size={len(sst)})")

        # 간단한 컴팩션: SSTable이 많아지면 병합
        if len(self.sstables) >= 3:
            self._compact()

    def _compact(self):
        """컴팩션: 여러 SSTable을 하나로 병합, tombstone 제거"""
        merged = {}
        for sst in self.sstables:  # 오래된 것부터 (나중에 덮어씀)
            merged.update(sst.data)

        # Tombstone 제거
        clean = {k: v for k, v in merged.items() if v != "__TOMBSTONE__"}
        self.sstables = [SSTable(clean, time.time())]
        print(f"  [Compact] {len(merged)} → {len(clean)} entries")


# 시뮬레이션
lsm = LSMTree(memtable_limit=3)

lsm.put("user:1", "Alice")
lsm.put("user:2", "Bob")
lsm.put("user:3", "Charlie")  # → Flush 발생
lsm.put("user:1", "Alice Updated")  # 덮어쓰기
lsm.put("user:4", "Dave")
lsm.put("user:2", "__delete__")
lsm.delete("user:2")          # Tombstone 삽입
lsm.put("user:5", "Eve")      # → Flush, 그 후 Compact 가능

print(lsm.get("user:1"))  # Alice Updated
print(lsm.get("user:2"))  # None (삭제됨)
print(lsm.get("user:3"))  # Charlie
```

## B-Tree vs LSM-Tree: 핵심 비교

| 항목 | B-Tree | LSM-Tree |
|---|---|---|
| **쓰기 방식** | In-place (랜덤 I/O) | Append-only (순차 I/O) |
| **읽기 성능** | 우수 (O log N) | 보통 (여러 레이어 탐색) |
| **쓰기 성능** | 보통 | 우수 |
| **쓰기 증폭** | 낮음 | 높음 (컴팩션) |
| **읽기 증폭** | 낮음 | 높음 (여러 SSTable) |
| **공간 효율** | 높음 | 보통 (tombstone, 중복) |
| **대표 DB** | PostgreSQL, MySQL, SQLite | Cassandra, RocksDB, LevelDB, ScyllaDB |

### 세 가지 증폭(Amplification) 비교

- **쓰기 증폭(Write Amplification)**: 실제 사용자 데이터 1바이트를 쓰기 위해 디스크에 쓰는 바이트 수 비율. LSM이 컴팩션으로 높음.
- **읽기 증폭(Read Amplification)**: 1번 읽기 요청에 필요한 I/O 횟수. LSM은 여러 레이어를 뒤져야 함.
- **공간 증폭(Space Amplification)**: 실제 데이터 대비 사용 공간. LSM은 컴팩션 전까지 중복 데이터 존재.

## 주의사항과 선택 가이드

**읽기 중심 워크로드 → B-Tree** (예: 보고서 쿼리, OLAP, 범위 스캔이 많은 경우)

**쓰기 중심 워크로드 → LSM-Tree** (예: 로그 수집, IoT 센서 데이터, 타임시리즈 DB)

**혼합 워크로드:** 최신 Titan(TiKV의 B-Tree 위에 LSM 혼용), WiredTiger(MongoDB)처럼 하이브리드 접근도 존재합니다.

**Bloom Filter는 필수:** LSM-Tree에서 존재하지 않는 키 조회 비용을 줄이는 Bloom Filter 없이는 읽기 성능이 급격히 하락합니다. FPR(False Positive Rate)을 1% 이하로 유지하세요.

**컴팩션 전략 선택:** Level Compaction(크기 증폭 최소화), Size-Tiered Compaction(쓰기 증폭 최소화), FIFO Compaction(타임시리즈)을 워크로드에 맞게 선택하세요.

## 참고 자료
- [ByteByteGo: B-Tree vs LSM-Tree](https://bytebytego.com/guides/b-tree-vs/)
- [The Great Storage Engine Debate: B-Trees vs LSM-Trees (Medium)](https://medium.com/@sylvain.tiset/the-great-storage-engine-debate-b-trees-vs-lsm-trees-35d6975f2ede)
- [FOSDEM 2026: LSM vs B-Tree - RocksDB vs WiredTiger](https://fosdem.org/2026/events/attachments/TCWURN-lsm-vs-btree-rocksdb-wiredtiger/slides/267165/20260131_wmq1gjp.pdf)
