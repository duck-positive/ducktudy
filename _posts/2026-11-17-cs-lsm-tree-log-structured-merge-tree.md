---
layout: post
title: "LSM-Tree(Log-Structured Merge-Tree) 완전 정복: LevelDB·RocksDB·Cassandra가 선택한 쓰기 최적화 저장 엔진의 비밀"
date: 2026-11-17
categories: [cs, computer-science]
tags: [lsm-tree, rocksdb, leveldb, database, storage-engine, compaction, sstable, memtable]
---

현대 데이터베이스 엔진의 심장부에는 B-트리 대신 LSM-Tree를 선택한 시스템들이 있다. Apache Cassandra, RocksDB, LevelDB, InfluxDB, Apache HBase 모두 LSM-Tree 기반이다. 이 글에서는 LSM-Tree의 수학적 토대부터 실제 구현, 그리고 B-트리와의 근본적인 트레이드오프까지 깊이 파헤친다.

## 왜 LSM-Tree가 필요한가?

전통적인 B-트리 기반 저장소는 **랜덤 쓰기(random write)**에 취약하다. HDD에서 랜덤 I/O와 순차 I/O의 성능 차이는 100배 이상이고, NVMe SSD에서도 10배 이상 차이가 난다. B-트리는 삽입·업데이트 시마다 디스크의 임의 위치로 헤드를 이동시켜 쓰기 연산을 수행한다. 쓰기 증폭(Write Amplification)도 심각해서, 작은 데이터 하나를 쓰기 위해 전체 페이지(보통 4KB 또는 16KB)를 읽고 수정하여 다시 써야 한다.

LSM-Tree는 이 문제를 근본부터 다르게 접근한다. **모든 쓰기를 순차 I/O로 변환**하는 것이 핵심이다. 1996년 Patrick O'Neil 외 연구자들이 발표한 논문 "The Log-Structured Merge-Tree"에서 처음 제안된 이 구조는 다음 아이디어를 기반으로 한다:

> "쓰기를 먼저 메모리에 쌓고, 나중에 순차적으로 디스크에 병합한다."

이 단순한 아이디어가 현대 고성능 저장 엔진의 기반이 된다.

## LSM-Tree의 핵심 구성 요소

### 1. WAL(Write-Ahead Log)

내구성(Durability) 보장을 위해 모든 쓰기 연산을 먼저 WAL에 순차 기록한다. 디스크의 맨 끝에 append하는 방식이므로 매우 빠르다. 시스템 충돌 시 WAL로 MemTable을 복구할 수 있다.

### 2. MemTable

인메모리 정렬 자료구조(보통 Red-Black Tree 또는 Skip List)이다. WAL 기록 후 MemTable에 삽입하면 쓰기 완료다. 조회 시 MemTable부터 먼저 확인한다. MemTable이 임계값(보통 64MB)을 초과하면 Immutable MemTable로 전환하고, 새 MemTable을 생성한다.

### 3. SSTable(Sorted String Table)

Immutable MemTable을 디스크에 순차 기록한 불변(immutable) 파일이다. 키 기준으로 정렬된 키-값 쌍의 집합이다. 각 SSTable 끝에 블룸 필터(Bloom Filter)와 인덱스 블록을 포함한다.

### 4. 레벨과 컴팩션(Compaction)

SSTable은 여러 레벨(L0, L1, L2, ...)로 계층화된다. 레벨이 높을수록 용량이 크고(보통 10배씩), SSTable 간 키 범위가 겹치지 않는다. 컴팩션은 작은 레벨의 SSTable을 큰 레벨과 병합·정렬하는 백그라운드 작업이다.

## 실제 구현 예제 1: Java로 간단한 LSM-Tree 구현

```java
import java.util.*;
import java.util.concurrent.*;

public class SimpleLSMTree {

    // MemTable: 정렬된 인메모리 저장소
    private final TreeMap<String, String> memTable = new TreeMap<>();
    private final int memTableThreshold = 4; // 예시용 소값

    // SSTable 목록: 각 SSTable은 정렬된 Map
    private final List<TreeMap<String, String>> ssTables = new ArrayList<>();

    // WAL 역할 (간단화)
    private final List<String> wal = new ArrayList<>();

    public void put(String key, String value) {
        // 1. WAL에 먼저 기록
        wal.add("PUT:" + key + "=" + value);

        // 2. MemTable에 삽입
        memTable.put(key, value);

        // 3. MemTable이 가득 차면 플러시
        if (memTable.size() >= memTableThreshold) {
            flushMemTable();
        }
    }

    public void delete(String key) {
        // 삭제는 tombstone 마커로 표현
        put(key, "__DELETED__");
    }

    private void flushMemTable() {
        System.out.println("Flushing MemTable to SSTable (size=" + memTable.size() + ")");
        // MemTable을 복사하여 새 SSTable 생성
        ssTables.add(new TreeMap<>(memTable));
        memTable.clear();
        wal.clear();

        // 레벨 0에 SSTable이 너무 많으면 컴팩션 트리거
        if (ssTables.size() >= 3) {
            compact();
        }
    }

    private void compact() {
        System.out.println("Running compaction on " + ssTables.size() + " SSTables...");
        // 모든 SSTable을 하나로 병합 (실제로는 레벨별 처리)
        TreeMap<String, String> merged = new TreeMap<>();
        // 오래된 순서부터 적용하여 최신 값이 덮어쓰게 함
        for (TreeMap<String, String> ssTable : ssTables) {
            merged.putAll(ssTable);
        }
        // tombstone 제거
        merged.entrySet().removeIf(e -> "__DELETED__".equals(e.getValue()));

        ssTables.clear();
        if (!merged.isEmpty()) {
            ssTables.add(merged);
        }
        System.out.println("Compaction done. SSTables: " + ssTables.size());
    }

    public Optional<String> get(String key) {
        // 1. MemTable에서 먼저 조회
        if (memTable.containsKey(key)) {
            String val = memTable.get(key);
            return "__DELETED__".equals(val) ? Optional.empty() : Optional.of(val);
        }

        // 2. SSTable을 최신 순으로 조회 (최신 L0 SSTable 우선)
        for (int i = ssTables.size() - 1; i >= 0; i--) {
            TreeMap<String, String> ssTable = ssTables.get(i);
            if (ssTable.containsKey(key)) {
                String val = ssTable.get(key);
                return "__DELETED__".equals(val) ? Optional.empty() : Optional.of(val);
            }
        }

        return Optional.empty();
    }

    public static void main(String[] args) {
        SimpleLSMTree lsm = new SimpleLSMTree();

        // 데이터 삽입
        lsm.put("apple", "1");
        lsm.put("banana", "2");
        lsm.put("cherry", "3");
        lsm.put("date", "4");   // 임계값 도달 → MemTable 플러시
        lsm.put("elderberry", "5");
        lsm.put("fig", "6");
        lsm.put("grape", "7");
        lsm.put("honeydew", "8"); // 2번째 플러시

        // 조회
        System.out.println("apple: " + lsm.get("apple"));     // Optional[1]
        System.out.println("banana: " + lsm.get("banana"));   // Optional[2]

        // 삭제 후 조회
        lsm.delete("apple");
        System.out.println("apple after delete: " + lsm.get("apple")); // Optional.empty
    }
}
```

## 읽기 경로의 최적화: 블룸 필터

LSM-Tree의 단점은 읽기 증폭(Read Amplification)이다. 최악의 경우 모든 레벨의 모든 SSTable을 조회해야 한다. 이를 해결하기 위해 각 SSTable에 **블룸 필터**를 붙인다.

```java
import java.util.BitSet;

public class BloomFilter {
    private final BitSet bitSet;
    private final int size;
    private final int hashCount;

    public BloomFilter(int size, int hashCount) {
        this.size = size;
        this.hashCount = hashCount;
        this.bitSet = new BitSet(size);
    }

    private int[] getHashPositions(String key) {
        int[] positions = new int[hashCount];
        int hash1 = key.hashCode();
        int hash2 = key.hashCode() * 31;
        for (int i = 0; i < hashCount; i++) {
            positions[i] = Math.abs((hash1 + i * hash2) % size);
        }
        return positions;
    }

    public void add(String key) {
        for (int pos : getHashPositions(key)) {
            bitSet.set(pos);
        }
    }

    // false positive 가능하지만 false negative는 없음
    public boolean mightContain(String key) {
        for (int pos : getHashPositions(key)) {
            if (!bitSet.get(pos)) return false;
        }
        return true;
    }

    public static void main(String[] args) {
        BloomFilter bf = new BloomFilter(1000, 3);

        // SSTable에 있는 키 등록
        String[] keys = {"apple", "banana", "cherry", "date"};
        for (String k : keys) bf.add(k);

        // 조회 전 블룸 필터로 존재 여부 먼저 확인
        System.out.println(bf.mightContain("apple"));   // true (실제 존재)
        System.out.println(bf.mightContain("elderberry")); // false (존재 안 함 → SSTable 조회 스킵!)
        System.out.println(bf.mightContain("fig"));     // false → 불필요한 디스크 I/O 제거
    }
}
```

## 컴팩션 전략 비교

| 전략 | 특징 | 사용 사례 |
|------|------|----------|
| **Size-Tiered** | 비슷한 크기의 SSTable을 묶어서 병합 | 쓰기 집중형 (Cassandra 기본) |
| **Leveled** | 레벨별 용량 제한, 키 범위 비중복 보장 | 읽기 집중형 (RocksDB, LevelDB 기본) |
| **FIFO** | 오래된 SSTable 순서대로 삭제 | 시계열 데이터 (만료 정책) |

## RocksDB의 주요 최적화 기법

1. **컬럼 패밀리(Column Families)**: 독립적인 MemTable·SSTable을 그룹별로 관리
2. **압축(Compression)**: 각 레벨별로 Snappy, zstd, LZ4 등 적용
3. **직접 I/O(Direct I/O)**: OS 페이지 캐시를 우회하여 쓰기 증폭 감소
4. **Pipelined Writes**: WAL 쓰기와 MemTable 삽입을 파이프라인화

## 주의사항과 운영 팁

**공간 증폭(Space Amplification)**: 삭제된 데이터가 컴팩션 전까지 디스크에 남아있다. 최악의 경우 실제 데이터의 2~10배 공간을 차지할 수 있다.

**쓰기 중단(Write Stall)**: L0 SSTable이 너무 빠르게 쌓이면 쓰기를 의도적으로 늦춘다. RocksDB의 `level0_slowdown_writes_trigger`와 `level0_stop_writes_trigger` 설정으로 제어한다.

**컴팩션 튜닝**: I/O 대역폭의 80%를 컴팩션이 차지하면 실제 쓰기에 영향을 준다. `max_background_compactions`와 `compaction_readahead_size`를 적절히 설정해야 한다.

**포인트 조회 성능**: 많은 레벨을 거쳐야 할 때 블룸 필터가 없으면 성능이 급격히 저하된다. 블룸 필터의 오탐률(false positive rate)은 `bits_per_key` 설정으로 조절하며, 일반적으로 1% 미만으로 유지한다.

## 참고 자료

- [RocksDB 공식 GitHub](https://github.com/facebook/rocksdb)
- [LevelDB 공식 GitHub](https://github.com/google/leveldb)
- [원본 LSM-Tree 논문 (O'Neil et al., 1996)](https://www.cs.umb.edu/~poneil/lsmtree.pdf)
- [Wikipedia: Log-Structured Merge-Tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree)
