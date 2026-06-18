---
layout: post
title: "LSM 트리 완전 정복: RocksDB·LevelDB가 선택한 쓰기 최적화 자료구조"
date: 2026-06-18
categories: [cs, computer-science]
tags: [lsm-tree, rocksdb, leveldb, sstable, memtable, compaction, database, data-structure]
---

## 개념 설명

LSM 트리(Log-Structured Merge-Tree)는 1996년 Patrick O'Neil 등이 제안한 자료구조로, **쓰기 성능을 극대화**하기 위해 설계되었습니다. 오늘날 RocksDB, LevelDB, Apache Cassandra, HBase, ScyllaDB 등 수많은 현대 데이터베이스의 스토리지 엔진 핵심으로 사용됩니다.

전통적인 B-Tree 기반 스토리지는 삽입·수정 시 트리의 특정 노드를 찾아가 인플레이스(in-place) 업데이트를 수행합니다. 이 방식은 랜덤 I/O가 많이 발생해, SSD·HDD 모두에서 쓰기 처리량이 제한됩니다. LSM 트리는 반대 전략을 택합니다: **모든 쓰기를 순차적으로 처리**하고, 실제 정렬·병합은 나중에 백그라운드에서 수행합니다.

### 핵심 구성 요소

**1. WAL (Write-Ahead Log)**  
데이터가 메모리에 쓰이기 전, 먼저 디스크의 로그 파일에 순차적으로 기록됩니다. 프로세스 충돌 시 이 로그를 통해 복구합니다. 순차 쓰기이므로 매우 빠릅니다.

**2. MemTable**  
인메모리 정렬 자료구조(보통 레드-블랙 트리 또는 스킵 리스트)로, 새로운 키-값 쌍을 받아 정렬된 상태로 유지합니다. 읽기 요청이 오면 MemTable을 먼저 검색합니다.

**3. SSTable (Sorted String Table)**  
MemTable이 특정 크기에 도달하면 **불변(immutable) 파일**로 디스크에 플러시됩니다. 키가 정렬된 상태로 기록되어 이진 탐색이 가능합니다. 각 SSTable에는 내부 인덱스와 블룸 필터(Bloom Filter)가 함께 저장됩니다.

**4. 레벨 구조와 Compaction**  
SSTable은 레벨 0, 1, 2, ... 와 같이 계층적으로 관리됩니다. 레벨 0에 SSTable이 쌓이면 백그라운드에서 **Compaction** 작업이 수행됩니다: 여러 SSTable을 읽어 정렬·병합하고, 중복/삭제된 키를 제거한 뒤 상위 레벨에 새 SSTable로 기록합니다.

---

## 왜 필요한가

현대 애플리케이션은 초당 수십만 건의 쓰기 요청을 처리해야 하는 경우가 많습니다. 특히 다음과 같은 환경에서 LSM 트리가 빛을 발합니다:

- **이벤트 로깅 / 시계열 데이터**: 항상 새로운 타임스탬프로 삽입이 발생
- **소셜 그래프 / 메시지 시스템**: 대규모 팔로우·메시지 쓰기
- **리얼타임 애널리틱스**: 집계 전 원시 이벤트 고속 수집

B-Tree의 최악 쓰기 복잡도가 O(log n)인 반면, LSM 트리는 MemTable에 O(log n) 삽입 + 디스크 순차 쓰기이므로 실제 처리량이 수배~수십 배 높습니다. 단, 읽기는 여러 레벨의 SSTable을 조회해야 할 수 있어 B-Tree보다 느릴 수 있습니다.

---

## 실제 구현 예제

### 예제 1: Python으로 구현한 미니 LSM 트리

```python
import os
import json
from sortedcontainers import SortedDict

class LSMTree:
    MEMTABLE_SIZE_LIMIT = 4  # 데모용 소형 임계값

    def __init__(self, data_dir="./lsm_data"):
        self.data_dir = data_dir
        os.makedirs(data_dir, exist_ok=True)
        self.memtable = SortedDict()      # 인메모리 정렬 딕셔너리
        self.sstable_files = []           # 디스크 SSTable 경로 목록
        self.wal_path = os.path.join(data_dir, "wal.log")

    def _write_wal(self, key, value):
        with open(self.wal_path, "a") as f:
            f.write(json.dumps({"k": key, "v": value}) + "\n")

    def put(self, key, value):
        self._write_wal(key, value)       # WAL 먼저 기록
        self.memtable[key] = value        # MemTable에 삽입
        if len(self.memtable) >= self.MEMTABLE_SIZE_LIMIT:
            self._flush_memtable()

    def _flush_memtable(self):
        idx = len(self.sstable_files)
        path = os.path.join(self.data_dir, f"sst_{idx:04d}.sst")
        with open(path, "w") as f:
            for k, v in self.memtable.items():
                f.write(json.dumps({"k": k, "v": v}) + "\n")
        self.sstable_files.append(path)
        self.memtable.clear()
        print(f"  [flush] MemTable → {path}")

    def get(self, key):
        # 1) MemTable 우선 탐색
        if key in self.memtable:
            return self.memtable[key]
        # 2) SSTable 역순 탐색 (최신 → 구)
        for path in reversed(self.sstable_files):
            with open(path) as f:
                for line in f:
                    entry = json.loads(line)
                    if entry["k"] == key:
                        return entry["v"]
        return None


# 사용 예시
if __name__ == "__main__":
    tree = LSMTree()
    for i in range(6):
        tree.put(f"user:{i}", {"name": f"Alice{i}", "score": i * 10})

    print(tree.get("user:0"))   # {'name': 'Alice0', 'score': 0}
    print(tree.get("user:5"))   # {'name': 'Alice5', 'score': 50}
```

### 예제 2: Compaction 로직 — 두 SSTable 병합

```python
import json

def compact(sst_paths_old: list[str], sst_path_new: str):
    """
    여러 SSTable을 읽어 최신 버전만 남기는 단순 Compaction.
    실제 RocksDB는 레벨 단위로 멀티-way merge를 수행합니다.
    """
    merged: dict = {}
    for path in sst_paths_old:
        with open(path) as f:
            for line in f:
                entry = json.loads(line)
                # 나중에 읽힌 값이 더 최신 (파일 순서대로 처리)
                merged[entry["k"]] = entry["v"]

    # 키 정렬 후 새 SSTable로 기록
    with open(sst_path_new, "w") as f:
        for k in sorted(merged):
            f.write(json.dumps({"k": k, "v": merged[k]}) + "\n")

    # 구 파일 제거
    import os
    for path in sst_paths_old:
        os.remove(path)
    print(f"  [compact] {len(sst_paths_old)} SSTables → {sst_path_new}")


# 예시
compact(
    sst_paths_old=["./lsm_data/sst_0000.sst", "./lsm_data/sst_0001.sst"],
    sst_path_new="./lsm_data/sst_compacted_0000.sst"
)
```

---

## 주의사항 및 실무 팁

**Write Amplification**: Compaction은 동일 데이터를 여러 번 디스크에 쓰게 만듭니다. RocksDB의 Leveled Compaction에서는 최대 10~30배의 Write Amplification이 발생할 수 있습니다. Tiered Compaction(Cassandra 방식)은 Write Amplification은 낮지만 Space Amplification이 증가합니다.

**Read Amplification**: 최악의 경우 레벨 수만큼 SSTable을 순회해야 합니다. 이를 완화하기 위해 각 SSTable에 **블룸 필터(Bloom Filter)**를 붙여 키가 없는 SSTable은 디스크 접근 없이 건너뜁니다.

**삭제 처리 — Tombstone**: LSM 트리에서 삭제는 실제로 데이터를 지우지 않습니다. `DELETE` 마커(Tombstone)를 삽입하고, Compaction 단계에서 실제로 제거합니다. 대량 삭제 후 즉시 디스크 공간이 회수되지 않을 수 있습니다.

**RocksDB 실무 튜닝 포인트**:
- `write_buffer_size`: MemTable 크기 (기본 64MB)
- `max_write_buffer_number`: 동시 MemTable 수
- `level0_file_num_compaction_trigger`: L0→L1 Compaction 임계값
- `compression`: 레벨별 압축 알고리즘 (Snappy, Zstd 등)

## 참고 자료
- [What Is a Log-Structured Merge Tree? - Aerospike](https://aerospike.com/blog/log-structured-merge-tree-explained/)
- [Deep Dive LSM Tree (Cassandra, LevelDB, RocksDB) - Medium](https://medium.com/@aqilzeka99/deep-dive-lsm-tree-internals-of-cassandra-leveldb-rocksdb-scylladb-2c4149db8d92)
- [Building an LSM-Tree from Scratch - Medium](https://medium.com/@rahulhind/building-an-lsm-tree-from-scratch-implementing-memtable-sstable-and-wal-805e2660664b)
- [LSM-Tree: Core of NoSQL Storage Systems - dingyuqi.com](https://www.dingyuqi.com/en/article/lsm-tree/)
