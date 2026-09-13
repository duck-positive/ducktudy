---
layout: post
title: "LMDB 완전 정복: 메모리 맵 B+트리 데이터베이스가 성능과 안전성을 동시에 잡는 법"
date: 2026-09-13
categories: [cs, computer-science]
tags: [lmdb, database, mmap, b-plus-tree, mvcc, copy-on-write, embedded-database]
---

Lightning Memory-Mapped Database(LMDB)는 2011년 Howard Chu가 OpenLDAP 프로젝트를 위해 개발한 임베디드 키-값 저장소다. SQLite와 함께 임베디드 데이터베이스의 양대 산맥을 이루지만, 설계 철학은 완전히 다르다. LMDB는 단 **32,000줄의 C 코드**로 구현됐음에도 메모리 맵, B+트리, MVCC, Copy-on-Write를 결합해 놀라운 읽기 성능과 ACID 트랜잭션 안전성을 동시에 달성한다. 이 글에서는 LMDB의 내부 원리를 구조부터 구현까지 완전히 해부한다.

## LMDB의 핵심 설계 원칙

LMDB를 이해하는 핵심은 세 가지 설계 결정이다:

**1. mmap 기반 아키텍처**  
전통적인 데이터베이스(SQLite, MySQL InnoDB)는 파일을 읽어 자체 버퍼 풀(buffer pool)을 관리한다. 커널이 이미 페이지 캐시를 운영하고 있음에도 불구하고 애플리케이션 레벨에서 또 다른 캐시를 두는 것이다. LMDB는 이 비효율을 거부한다. 데이터베이스 파일 전체를 `mmap()`으로 가상 주소 공간에 매핑하고, 커널의 페이지 캐시를 그대로 자신의 캐시로 사용한다.

결과적으로 LMDB는 별도의 캐시 관리 코드가 전혀 없다. 읽기는 메모리 접근이고, 쓰기 후 `msync()`를 호출하면 커널이 더티 페이지를 디스크에 플러시한다.

**2. Copy-on-Write B+트리**  
B+트리의 노드(페이지)를 수정할 때 LMDB는 **해당 노드와 루트까지의 모든 조상을 새 페이지에 복사**한다. 기존 페이지는 현재 진행 중인 읽기 트랜잭션이 마칠 때까지 보존된다. 이 방식이 WAL(Write-Ahead Log) 없이 ACID를 보장하는 비밀이다.

**3. 단일 쓰기자 MVCC**  
LMDB는 동시에 최대 한 개의 쓰기 트랜잭션만 허용한다(직렬화). 대신 읽기 트랜잭션은 무제한으로 동시 실행되며, 쓰기 잠금을 전혀 필요로 하지 않는다. 각 읽기 트랜잭션은 시작 시점의 루트 페이지 번호를 기록하고, 그 스냅샷을 통해 일관된 뷰를 얻는다.

## 왜 LMDB인가: 전통적 설계와의 비교

| 특성 | LMDB | SQLite WAL | RocksDB |
|------|------|-----------|-------|
| 읽기 잠금 | 없음 | 공유 잠금 | 없음 |
| 쓰기 잠금 | 전역 뮤텍스 | DB 레벨 | 없음 |
| 캐시 관리 | 커널 mmap | 자체 페이지 캐시 | Block cache |
| 쓰기 증폭 | 낮음(CoW) | 낮음(WAL) | 높음(compaction) |
| 읽기 증폭 | 매우 낮음 | 낮음 | 높음(level 탐색) |
| 스페이스 증폭 | 낮음 | 중간 | 높음 |

LMDB가 빛나는 환경은 **읽기 집중 워크로드**다. 쓰기 동시성이 낮은 대신, 읽기는 완전히 무잠금이고 커널 페이지 캐시를 직접 활용하므로 메모리에 데이터가 있을 때 대단히 빠르다.

## B+트리 페이지 구조

LMDB의 B+트리 페이지(노드)는 4KB(기본) 고정 크기다. 페이지 타입은 세 가지다:

- **Branch Page**: 내부 노드. 키와 자식 페이지 번호의 쌍으로 구성. 리프가 아니므로 값을 저장하지 않음.
- **Leaf Page**: 단말 노드. 실제 키-값 쌍 저장. B+트리이므로 모든 데이터는 리프에 있음.
- **Overflow Page**: 단일 값이 페이지 크기를 초과할 때 사용. 리프 페이지가 overflow page 번호를 참조.

페이지 헤더에는 페이지 번호, 타입, 사용 중인 바이트 수, 엔트리 수가 기록된다. 페이지 내 엔트리는 오프셋 배열로 참조되어 가변 길이 키-값 쌍을 효율적으로 저장한다.

## Copy-on-Write의 작동 원리

CoW 업데이트가 어떻게 이루어지는지 구체적으로 살펴보자:

1. 쓰기 트랜잭션이 시작되면 현재 루트 페이지를 기록한다.
2. 리프 페이지를 수정해야 할 때, **새 페이지 번호를 할당**하고 기존 페이지 내용을 복사한 후 수정한다.
3. 수정된 리프를 가리키는 부모(Branch Page)도 새 페이지로 복사·수정한다.
4. 이 과정이 루트까지 반복된다.
5. 트랜잭션 커밋 시 새 루트 페이지 번호를 **Meta Page**에 원자적으로 기록한다.

커밋 전까지 기존 루트는 변경되지 않으므로, 진행 중인 읽기 트랜잭션은 기존 B+트리를 아무런 잠금 없이 계속 읽을 수 있다. 커밋 후에는 기존 페이지들이 "해제 예정" 목록에 추가되며, 그것을 참조하는 마지막 읽기 트랜잭션이 종료될 때 해제된다.

```c
/* C API를 통한 LMDB 기본 사용법 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "lmdb.h"

/* 에러 체크 매크로 */
#define CHECK(rc, msg) \
    if ((rc) != MDB_SUCCESS) { \
        fprintf(stderr, "%s: %s\n", msg, mdb_strerror(rc)); \
        exit(1); \
    }

int main(void) {
    MDB_env  *env;
    MDB_dbi   dbi;
    MDB_txn  *txn;
    MDB_val   key, data;
    MDB_cursor *cursor;
    int rc;

    /* 1. 환경 생성 및 구성 */
    CHECK(mdb_env_create(&env), "env_create");
    CHECK(mdb_env_set_mapsize(env, 100UL * 1024 * 1024), "set_mapsize"); /* 100MB */
    CHECK(mdb_env_set_maxdbs(env, 4), "set_maxdbs");
    CHECK(mdb_env_open(env, "/tmp/lmdb-demo", MDB_NOSUBDIR, 0664), "env_open");
    /* MDB_NOSUBDIR: 환경 파일을 단일 파일로 생성 */

    /* 2. Named Database 열기 — 쓰기 트랜잭션에서 열어야 함 */
    CHECK(mdb_txn_begin(env, NULL, 0, &txn), "txn_begin");
    CHECK(mdb_dbi_open(txn, "users", MDB_CREATE, &dbi), "dbi_open");

    /* 3. 여러 레코드 삽입 — 하나의 트랜잭션으로 배치 */
    const char *keys[] = {"alice", "bob", "carol", "dave", "eve"};
    const char *vals[] = {"admin", "developer", "designer", "developer", "tester"};

    for (int i = 0; i < 5; i++) {
        key.mv_size = strlen(keys[i]);
        key.mv_data = (void *)keys[i];
        data.mv_size = strlen(vals[i]);
        data.mv_data = (void *)vals[i];
        CHECK(mdb_put(txn, dbi, &key, &data, 0), "put");
    }
    CHECK(mdb_txn_commit(txn), "commit"); /* msync: 디스크 플러시 */

    /* 4. 읽기 전용 트랜잭션 — 잠금 없음, 완전한 스냅샷 격리 */
    CHECK(mdb_txn_begin(env, NULL, MDB_RDONLY, &txn), "rdonly_txn");

    /* 단일 키 조회 */
    const char *lookup = "alice";
    key.mv_size = strlen(lookup);
    key.mv_data = (void *)lookup;
    rc = mdb_get(txn, dbi, &key, &data);
    if (rc == MDB_SUCCESS)
        printf("alice: %.*s\n", (int)data.mv_size, (char *)data.mv_data);

    /* 커서로 범위 스캔 — B+트리 리프 연결 리스트 순회 */
    CHECK(mdb_cursor_open(txn, dbi, &cursor), "cursor");
    printf("\n전체 레코드 (정렬된 순서):\n");
    while ((rc = mdb_cursor_get(cursor, &key, &data, MDB_NEXT)) == MDB_SUCCESS) {
        printf("  %.*s -> %.*s\n",
               (int)key.mv_size,  (char *)key.mv_data,
               (int)data.mv_size, (char *)data.mv_data);
    }
    mdb_cursor_close(cursor);
    mdb_txn_abort(txn); /* 읽기 트랜잭션은 abort로 릴리즈 */

    /* 5. 통계 확인 */
    CHECK(mdb_txn_begin(env, NULL, MDB_RDONLY, &txn), "stat_txn");
    MDB_stat stat;
    mdb_stat(txn, dbi, &stat);
    printf("\nDB 통계:\n");
    printf("  페이지 크기:   %u bytes\n", stat.ms_psize);
    printf("  B+트리 깊이:   %u\n",       stat.ms_depth);
    printf("  Branch 페이지: %zu\n",       stat.ms_branch_pages);
    printf("  Leaf 페이지:   %zu\n",       stat.ms_leaf_pages);
    printf("  레코드 수:     %zu\n",       stat.ms_entries);
    mdb_txn_abort(txn);

    mdb_dbi_close(env, dbi);
    mdb_env_close(env);
    return 0;
}
```

## Python 바인딩으로 성능 벤치마크

```python
import lmdb
import struct
import time
import random

def benchmark_lmdb(num_records: int = 100_000):
    """LMDB 읽기·쓰기 성능 측정"""
    env = lmdb.open(
        '/tmp/bench-lmdb',
        map_size=500 * 1024 * 1024,  # 500MB
        max_dbs=2
    )
    
    # === 쓰기 성능 ===
    start = time.perf_counter()
    with env.begin(write=True) as txn:
        for i in range(num_records):
            # Big-endian 정수 키: B+트리 정렬 순서 = 수치 순서
            key   = struct.pack('>I', i)
            value = f'user:{i:010d}|email:user{i}@example.com'.encode()
            txn.put(key, value)
    write_elapsed = time.perf_counter() - start
    print(f'쓰기 {num_records:,}건: {write_elapsed:.3f}초 '
          f'({num_records / write_elapsed:,.0f} ops/s)')
    
    # === 포인트 읽기 성능 ===
    random_keys = [random.randrange(num_records) for _ in range(10_000)]
    start = time.perf_counter()
    with env.begin() as txn:
        for k in random_keys:
            v = txn.get(struct.pack('>I', k))
            assert v is not None
    read_elapsed = time.perf_counter() - start
    print(f'랜덤 읽기 10,000건: {read_elapsed:.3f}초 '
          f'({10_000 / read_elapsed:,.0f} ops/s)')
    
    # === 범위 스캔 성능 (B+트리 리프 순차 탐색) ===
    start_key = struct.pack('>I', 50_000)
    end_key   = struct.pack('>I', 60_000)
    start = time.perf_counter()
    with env.begin() as txn:
        with txn.cursor() as cur:
            count = 0
            if cur.set_range(start_key):
                for k, v in cur.iternext():
                    if k >= end_key:
                        break
                    count += 1
    range_elapsed = time.perf_counter() - start
    print(f'범위 스캔 {count:,}건: {range_elapsed:.4f}초 '
          f'({count / range_elapsed:,.0f} ops/s)')
    
    # === 동시 읽기 트랜잭션 시뮬레이션 ===
    import threading
    
    results = []
    barrier = threading.Barrier(8)
    
    def reader(thread_id):
        barrier.wait()  # 모든 스레드 동시 시작
        start = time.perf_counter()
        with env.begin() as txn:  # 읽기 트랜잭션: 무잠금
            for i in range(1_000):
                k = struct.pack('>I', random.randrange(num_records))
                txn.get(k)
        results.append(time.perf_counter() - start)
    
    threads = [threading.Thread(target=reader, args=(i,)) for i in range(8)]
    for t in threads: t.start()
    for t in threads: t.join()
    
    avg_ms = sum(results) / len(results) * 1000
    print(f'\n8개 스레드 동시 읽기 (각 1,000건): 평균 {avg_ms:.1f}ms')
    
    # === LMDB 환경 통계 ===
    stat = env.stat()
    info = env.info()
    print(f'\nDB 통계:')
    print(f'  페이지 크기: {stat["psize"]:,} bytes')
    print(f'  B+트리 깊이: {stat["depth"]}')
    print(f'  총 레코드:  {stat["entries"]:,}')
    print(f'  현재 맵 크기: {info["map_size"] / 1024 / 1024:.1f} MB')
    print(f'  사용 중 페이지: {info["last_pgno"]:,}')
    
    env.close()


if __name__ == '__main__':
    benchmark_lmdb()
```

## Freelist와 페이지 재사용

레코드 삭제 시 LMDB는 해제된 페이지를 **Freelist**에 기록한다. Freelist 자체도 LMDB 내부의 B+트리로 관리된다. 새 페이지가 필요할 때 Freelist에서 재사용하거나, 없으면 파일 끝에 새 페이지를 추가한다.

Freelist가 단편화되면 데이터 크기보다 파일이 커질 수 있다. 이 경우 `mdb_copy()`로 새 환경에 복사하면 파일을 컴팩션할 수 있다.

## LMDB를 선택할 때와 피할 때

**선택해야 하는 경우:**
- 읽기가 쓰기보다 훨씬 많은 워크로드 (10:1 이상)
- 복잡한 쓰기 동시성이 불필요한 임베디드 환경
- 메모리에 데이터가 충분히 캐싱되는 환경 (mmap 효과 극대화)
- OpenLDAP, Cyrus IMAP, 여러 blockchain 구현체의 실제 사용 사례

**피해야 하는 경우:**
- 쓰기 처리량이 최우선인 워크로드 (RocksDB, 카산드라가 적합)
- 여러 프로세스가 동시에 쓰기를 해야 하는 경우 (단일 쓰기자 제한)
- 32비트 주소 공간 환경 (mmap 크기 제한)
- 맵 크기를 초과하는 데이터셋 (미리 충분히 설정 필요)

## 주의사항과 실전 팁

**1. 맵 크기는 넉넉하게 설정하라**  
`map_size`는 실제 데이터 크기가 아니라 가상 주소 공간 예약이다. 물리 메모리를 소비하지 않으므로 충분히 크게 잡아도 된다 (64비트 환경에서 TB 단위도 가능). 런타임에 크기를 줄이는 것은 위험하다.

**2. 읽기 트랜잭션 빨리 닫기**  
오래된 읽기 트랜잭션이 존재하면, 그것이 참조하는 모든 스냅샷 페이지가 해제되지 못한다. 이는 파일 크기가 계속 증가하는 "빅 리더 문제"를 일으킨다. 읽기 트랜잭션은 최대한 짧게 유지한다.

**3. `MDB_NOSYNC` 플래그 이해**  
`MDB_NOSYNC` 플래그는 커밋 시 `fsync()`를 생략해 쓰기 속도를 크게 높이지만, 시스템 충돌 시 마지막 커밋이 소실될 수 있다. 내구성보다 성능이 중요한 캐시 용도에만 사용한다.

**4. 환경 복사로 컴팩션**  
```c
/* 온라인 컴팩션: 파편화된 공간 회수 */
mdb_env_copy2(env, "/tmp/lmdb-compacted", MDB_CP_COMPACT);
```

**5. 다중 프로세스 접근**  
LMDB는 파일 잠금으로 다중 프로세스 접근을 지원하지만, 쓰기는 여전히 직렬화된다. 같은 환경에 접근하는 모든 프로세스는 동일한 `map_size`를 설정해야 한다.

## 참고 자료
- [LMDB 공식 문서 (Symas)](https://www.symas.com/lmdb)
- [LMDB GitHub 저장소](https://github.com/LMDB/lmdb)
- [LMDB Design — Howard Chu의 원 설계 문서](https://www.openldap.org/pub/hyc/mdm-paper.pdf)
- [Python lmdb 바인딩 문서](https://lmdb.readthedocs.io/en/release/)
