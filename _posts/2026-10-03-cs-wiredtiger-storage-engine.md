---
layout: post
title: "WiredTiger 스토리지 엔진 심화: Copy-on-Write B-트리와 MVCC의 구현 원리"
date: 2026-10-03
categories: [cs, computer-science]
tags: [wiredtiger, mongodb, storage-engine, mvcc, b-tree, database, concurrency]
---

MongoDB가 2014년 WiredTiger를 인수하고 3.2 버전부터 기본 스토리지 엔진으로 채택한 이유는 단순하다. 기존 MMAPv1 엔진의 고질적인 락 경합 문제를 근본적으로 해결했기 때문이다. WiredTiger는 어떻게 문서 수준의 동시성(document-level concurrency)을 달성하면서도 ACID 보장을 유지할까?

## 개념 설명

WiredTiger는 오픈소스 임베디드 데이터베이스 엔진으로, MongoDB의 기본 스토리지 백엔드 역할을 한다. 독립적으로도 C API로 직접 사용 가능하다. 핵심 설계 원칙은 **쓰기 시 복사(Copy-on-Write)**와 **낙관적 동시성 제어(Optimistic Concurrency Control)**다.

### 1. B-트리 기반 페이지 구조

WiredTiger는 전통적인 B+-트리 구조를 사용하지만, 페이지를 직접 수정하는 대신 새 버전을 생성하는 CoW 방식을 택한다.

**페이지 유형:**
- **Internal Page (내부 페이지)**: 자식 페이지를 가리키는 포인터만 포함. B-트리의 라우팅 노드 역할.
- **Leaf Page (리프 페이지)**: 실제 키-값 레코드를 저장. 실제 데이터가 있는 곳.
- **Overflow Page**: 매우 큰 값(기본 페이지보다 큰)을 별도로 저장.

**페이지 상태:**
- **Clean**: 디스크 내용과 동일한 상태
- **Dirty**: 메모리에서 수정되었으나 아직 디스크에 반영되지 않은 상태
- **Reconciling**: 체크포인트 또는 Eviction 중인 상태

메모리 내 페이지는 **Update Chain** 구조로 수정 사항을 관리한다. 단일 레코드에 여러 업데이트가 있을 때 링크드 리스트로 체이닝하여 각 트랜잭션이 자신의 스냅샷에 맞는 버전을 찾을 수 있게 한다.

### 2. MVCC (다중 버전 동시성 제어)

WiredTiger의 MVCC 구현은 다음 두 가지 핵심 개념에 기반한다:

**스냅샷 트랜잭션 ID (Transaction Snapshot):**
- 각 트랜잭션이 시작할 때 현재 활성 트랜잭션 ID의 스냅샷을 기록한다.
- 읽기 시 "내 스냅샷 이전에 커밋된 가장 최신 버전"을 선택한다.
- 이를 통해 읽기-쓰기 간 블로킹이 완전히 제거된다.

**Update Visibility Rules:**
```
어떤 업데이트가 현재 트랜잭션에게 보이는가?

1. 업데이트의 트랜잭션 ID < 내 스냅샷의 시작 ID
   → 내가 시작하기 전에 이미 커밋됨 → 보임
2. 업데이트의 트랜잭션 ID = 내 자신의 ID
   → 내가 만든 업데이트 → 보임
3. 업데이트가 내 스냅샷 목록에 있는 활성 트랜잭션의 것
   → 아직 커밋되지 않음 → 안 보임
4. 업데이트가 Rolled back됨
   → 안 보임
```

### 3. 체크포인트 (Checkpoint)

체크포인트는 WiredTiger의 Crash Recovery 메커니즘이다. 기본적으로 60초마다 또는 WAL(Write-Ahead Log, 저널) 파일이 2GB에 도달하면 발생한다.

**체크포인트 프로세스:**
1. 현재 Dirty 페이지 목록 스냅샷 생성
2. 스냅샷에 포함된 Dirty 페이지들을 디스크에 기록
3. 새 루트 페이지 주소를 WiredTiger 메타데이터 파일에 원자적으로 기록
4. 이전 체크포인트를 참조하는 저널 파일 정리

체크포인트 자체가 원자적이기 때문에, 중간에 크래시가 발생해도 이전 체크포인트 + 저널 리플레이로 완전한 복구가 가능하다.

### 4. 저널 (Journal / WAL)

WiredTiger의 저널은 모든 데이터 수정을 디스크에 순차적으로 기록하는 Write-Ahead Log다.

- 기본적으로 100ms마다 fsync하여 내구성 보장
- 트랜잭션 커밋 시 저널에 먼저 기록 후 애플리케이션에 성공 반환
- 체크포인트 후 이미 반영된 저널 파일은 삭제 가능
- 압축 가능: `log=(compressor=snappy)` 설정으로 저널 공간 절약

### 5. Eviction 시스템

캐시 메모리가 부족할 때 WiredTiger는 페이지를 디스크로 내보내는 Eviction을 수행한다.

**Eviction 트리거:**
- `cache_size`의 기본 80% 도달 → 백그라운드 Eviction 시작
- `cache_size`의 기본 95% 도달 → 애플리케이션 스레드도 Eviction에 참여 (성능 급락 징조)
- `cache_size`의 100% 도달 → 새로운 쓰기 차단

Eviction 대상 페이지 선택은 LRU와 Dirty 비율을 고려하며, Dirty 페이지는 Clean 페이지보다 Eviction 비용이 높다(디스크 기록이 필요하므로).

## 왜 필요한가

### MMAPv1 대비 혁신

MongoDB의 이전 엔진 MMAPv1의 주요 한계:
- **컬렉션 수준 락**: 동일 컬렉션에 대한 쓰기가 직렬화됨
- **페이지 단편화**: mmap 기반으로 삭제 후 공간 재사용 비효율
- **압축 미지원**: 모든 데이터를 원본 크기로 저장

WiredTiger가 제공하는 개선:
- **문서 수준 동시성**: 서로 다른 문서 수정이 전혀 간섭 없음
- **30~80% 압축**: Snappy 기준 일반적인 MongoDB 워크로드에서
- **SSD 친화적**: CoW로 인해 동일 위치 덮어쓰기 없음 → Write Amplification 감소

### 현대 하드웨어 최적화

WiredTiger는 멀티코어 CPU와 NVMe SSD를 적극 활용하도록 설계됐다:
- **스레드 기반 Eviction**: 전용 eviction 스레드가 I/O를 비동기로 처리
- **병렬 체크포인트**: 여러 파일의 Dirty 페이지를 병렬로 기록
- **블록 압축**: CPU 사이클을 사용해 I/O 대역폭 절약 (CPU는 남아돌고 I/O가 병목인 현대 서버에서 유리)

## 실제 구현 예제

### 예제 1: WiredTiger C API 직접 사용

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <wiredtiger.h>

/*
 * WiredTiger 직접 사용 예제
 * 컴파일: gcc -o wt_demo wt_demo.c -lwiredtiger
 */
int main(void) {
    WT_CONNECTION *conn;
    WT_SESSION   *session;
    WT_CURSOR    *cursor;
    int ret;

    /* 1. 데이터베이스 열기 (없으면 생성) */
    ret = wiredtiger_open(
        "/tmp/wt_database", NULL,
        "create,"
        "cache_size=512MB,"
        "eviction=(threads_min=2,threads_max=8),"
        "log=(enabled=true,compressor=snappy),"
        "checkpoint=(wait=60,log_size=2GB)",
        &conn
    );
    if (ret != 0) { fprintf(stderr, "Open error: %s\n", wiredtiger_strerror(ret)); return 1; }

    /* 2. 세션 생성 (스레드당 1개) */
    conn->open_session(conn, NULL, "isolation=snapshot", &session);

    /* 3. 테이블 생성 (Snappy 압축 + 컬럼 정의) */
    session->create(session,
        "table:orders",
        "key_format=Q,"               /* Q = uint64_t */
        "value_format=SSIQ,"          /* S=string, I=int32, Q=uint64 */
        "columns=(order_id,customer,status,qty,amount),"
        "block_compressor=snappy,"
        "internal_page_max=16KB,"
        "leaf_page_max=64KB"
    );

    /* 4. 보조 인덱스 생성 */
    session->create(session,
        "index:orders:by_customer",
        "columns=(customer)"
    );

    /* 5. 트랜잭션으로 데이터 삽입 */
    session->open_cursor(session, "table:orders", NULL, NULL, &cursor);
    session->begin_transaction(session, "isolation=snapshot");

    uint64_t order_ids[] = {1001, 1002, 1003};
    const char *customers[] = {"alice", "bob", "alice"};
    const char *statuses[] = {"pending", "shipped", "delivered"};
    int32_t quantities[] = {5, 3, 1};
    uint64_t amounts[] = {150000, 90000, 30000};  /* 원 단위 */

    for (int i = 0; i < 3; i++) {
        cursor->set_key(cursor, order_ids[i]);
        cursor->set_value(cursor, customers[i], statuses[i],
                          quantities[i], amounts[i]);
        if ((ret = cursor->insert(cursor)) != 0) {
            fprintf(stderr, "Insert error: %s\n", wiredtiger_strerror(ret));
            session->rollback_transaction(session, NULL);
            goto cleanup;
        }
    }

    ret = session->commit_transaction(session, NULL);
    printf("트랜잭션 커밋: %s\n", ret == 0 ? "성공" : wiredtiger_strerror(ret));

    /* 6. 스냅샷 읽기 (MVCC 격리) */
    session->begin_transaction(session, "isolation=snapshot");
    cursor->reset(cursor);

    printf("\n=== 주문 목록 (스냅샷 격리) ===\n");
    while ((ret = cursor->next(cursor)) == 0) {
        uint64_t oid, amt;
        const char *cust, *stat;
        int32_t qty;
        cursor->get_key(cursor, &oid);
        cursor->get_value(cursor, &cust, &stat, &qty, &amt);
        printf("주문#%llu | 고객: %-10s | 상태: %-10s | 수량: %d | 금액: %llu원\n",
               (unsigned long long)oid, cust, stat, qty, (unsigned long long)amt);
    }
    session->rollback_transaction(session, NULL);  /* 읽기 전용이므로 롤백으로 종료 */

    /* 7. 수동 체크포인트 */
    session->checkpoint(session, "force=true");
    printf("\n체크포인트 완료\n");

cleanup:
    cursor->close(cursor);
    session->close(session, NULL);
    conn->close(conn, NULL);
    return ret;
}
```

### 예제 2: Python + pymongo로 WiredTiger 특성 활용

```python
"""
WiredTiger(MongoDB) 심화 활용:
- 문서 수준 동시성을 활용한 고성능 동시 업데이트
- MVCC 스냅샷 격리를 활용한 일관된 읽기
- WiredTiger 통계 모니터링
"""
from pymongo import MongoClient, ASCENDING, DESCENDING
from pymongo.errors import OperationFailure
from datetime import datetime
import concurrent.futures
import time

client = MongoClient('mongodb://localhost:27017/',
                     # WiredTiger 캐시 크기는 mongod.conf에서 설정
                     maxPoolSize=50)
db = client['wiredtiger_demo']
orders = db['orders']
inventory = db['inventory']


def setup_collections():
    """컬렉션 인덱스 설정"""
    orders.create_index([('customerId', ASCENDING), ('createdAt', DESCENDING)])
    orders.create_index('status')
    inventory.create_index('sku', unique=True)

    # WiredTiger 압축 설정 (MongoDB 4.2+)
    try:
        db.command({
            'collMod': 'orders',
            'validator': {'$jsonSchema': {
                'bsonType': 'object',
                'required': ['customerId', 'items', 'status']
            }}
        })
    except OperationFailure:
        pass


def concurrent_document_updates():
    """
    WiredTiger 문서 수준 동시성 데모:
    서로 다른 문서를 동시에 업데이트해도 락 경합 없음
    """
    # 초기 재고 데이터 삽입
    skus = [f'SKU-{i:04d}' for i in range(100)]
    inventory.insert_many([
        {'sku': sku, 'stock': 1000, 'reserved': 0}
        for sku in skus
    ])

    def update_inventory(sku):
        """각 문서를 독립적으로 업데이트 (문서 수준 락)"""
        result = inventory.find_one_and_update(
            {'sku': sku, 'stock': {'$gte': 10}},
            {
                '$inc': {'stock': -10, 'reserved': 10},
                '$set': {'lastUpdated': datetime.utcnow()}
            },
            return_document=True
        )
        return result is not None

    # 100개 SKU를 병렬로 동시 업데이트
    start = time.time()
    with concurrent.futures.ThreadPoolExecutor(max_workers=20) as executor:
        futures = [executor.submit(update_inventory, sku) for sku in skus]
        results = [f.result() for f in concurrent.futures.as_completed(futures)]
    elapsed = time.time() - start

    success_count = sum(1 for r in results if r)
    print(f'병렬 업데이트: {success_count}/100 성공 ({elapsed:.3f}초)')
    print('(WiredTiger 문서 수준 동시성으로 서로 다른 SKU 업데이트가 직렬화 없이 처리됨)')


def snapshot_isolation_demo():
    """
    MVCC 스냅샷 격리 데모:
    트랜잭션 시작 시점의 일관된 뷰 제공
    """
    orders.drop()
    orders.insert_many([
        {'orderId': i, 'amount': 100 * i, 'status': 'pending'}
        for i in range(1, 6)
    ])

    with client.start_session() as session:
        # 트랜잭션 A 시작 (스냅샷 고정)
        session.start_transaction(
            read_concern={'level': 'snapshot'},
            write_concern={'w': 'majority'}
        )

        # 트랜잭션 A에서 읽기 (스냅샷 기준)
        count_before = orders.count_documents({'status': 'pending'}, session=session)
        print(f'\n스냅샷 격리 - 트랜잭션 A 시작 시 pending 주문: {count_before}개')

        # 외부에서 다른 연결이 새 주문 추가 (세션 없이)
        orders.insert_one({'orderId': 99, 'amount': 9900, 'status': 'pending'})

        # 트랜잭션 A에서 다시 읽기 → 스냅샷이라 새 주문이 안 보임
        count_after = orders.count_documents({'status': 'pending'}, session=session)
        print(f'외부 추가 후 트랜잭션 A의 pending 주문: {count_after}개 (스냅샷 보존!)')
        print(f'스냅샷 격리 동작: {count_before == count_after}')

        session.commit_transaction()

    # 세션 외부에서는 새 주문 보임
    total = orders.count_documents({'status': 'pending'})
    print(f'트랜잭션 외부 pending 주문: {total}개 (새 주문 포함)')


def monitor_wiredtiger_cache():
    """WiredTiger 캐시 및 Eviction 상태 모니터링"""
    stats = db.command('serverStatus')
    wt = stats.get('wiredTiger', {})

    cache = wt.get('cache', {})
    print('\n=== WiredTiger 캐시 상태 ===')
    print(f"현재 캐시 사용량: {cache.get('bytes currently in the cache', 0) / 1024**2:.1f} MB")
    print(f"Dirty 바이트:     {cache.get('tracked dirty bytes in the cache', 0) / 1024**2:.1f} MB")
    print(f"최대 캐시 크기:   {cache.get('maximum bytes configured', 0) / 1024**2:.1f} MB")
    print(f"캐시 사용률:      {cache.get('percent overhead in the cache', 0):.1f}%")

    eviction = wt.get('cache', {})
    app_eviction = eviction.get('pages evicted by application threads', 0)
    bg_eviction = eviction.get('pages evicted in background', 0)
    print(f"\n애플리케이션 스레드 Eviction: {app_eviction}")
    print(f"백그라운드 Eviction:          {bg_eviction}")

    if app_eviction > 0:
        print("⚠️  경고: 캐시 압박으로 애플리케이션 스레드가 Eviction 참여 중!")
        print("   → cache_size 증가 또는 쓰기 처리량 감소 필요")

    # 체크포인트 정보
    checkpoint = wt.get('transaction', {})
    cp_count = checkpoint.get('transaction checkpoint currently running', 0)
    cp_time = checkpoint.get('transaction checkpoint most recent time (msecs)', 0)
    print(f"\n마지막 체크포인트 소요 시간: {cp_time}ms")
    print(f"현재 체크포인트 진행 중: {'예' if cp_count else '아니오'}")


if __name__ == '__main__':
    setup_collections()
    concurrent_document_updates()
    snapshot_isolation_demo()
    monitor_wiredtiger_cache()
    client.close()
```

## 주의사항 및 팁

**1. 캐시 크기 설정이 가장 중요하다**  
WiredTiger의 기본 캐시는 시스템 RAM의 50% 또는 256MB 중 큰 값이다. MongoDB는 이외에도 인덱스, 연결 풀, 집계 파이프라인, OS 페이지 캐시에 메모리를 사용한다. 일반적으로 WiredTiger 캐시를 전체 RAM의 40~50%로 제한하고, 나머지를 OS 페이지 캐시와 MongoDB의 다른 용도에 남겨두는 것이 권장된다.

**2. Eviction 경고 신호를 놓치지 말라**  
`serverStatus`의 `wiredTiger.cache.pages evicted by application threads` 값이 0보다 크다면 이미 캐시 압박이 발생하고 있다는 신호다. 이 값이 증가하면 쓰기 레이턴시가 급등한다. 알람 임계값을 설정하고 모니터링하라.

**3. 압축 알고리즘 선택**  
- **snappy** (기본): CPU 부담이 적고 압축/해제가 빠름. 범용 워크로드에 적합.
- **zlib** / **zstd**: 더 높은 압축률. 쓰기 처리량보다 저장 공간이 중요한 경우.
- **none**: 압축 CPU 오버헤드를 피하고 싶을 때. 단, 저장 공간은 2~3배 증가.
MongoDB 4.2+에서는 zstd가 추가되어 snappy보다 30~40% 나은 압축률을 제공하면서 압축 속도는 유사하다.

**4. 동시 트랜잭션 수 제한**  
WiredTiger는 기본적으로 128개의 동시 트랜잭션 슬롯을 유지한다. 이보다 많은 동시 연결이 있으면 성능이 비선형적으로 저하된다. `wiredTigerConcurrentReadTransactions`와 `wiredTigerConcurrentWriteTransactions` 설정으로 조정 가능하지만, 근본적으로 연결 풀 크기를 조정하는 것이 더 바람직하다.

**5. 저널 sync 주기 조정**  
`storage.journal.commitIntervalMs` 기본값은 100ms다. 이를 줄이면 장애 시 데이터 손실 창이 작아지지만 쓰기 처리량이 감소한다. `w: "majority"` 쓰기 우려 설정과 함께 적절한 트레이드오프를 선택하라.

**6. WiredTiger 로그 분석**  
문제 진단 시 `db.adminCommand({setParameter: 1, wiredTigerEngineRuntimeConfig: "verbose=[evict:2,evictserver:1]"})`으로 상세 로그를 활성화할 수 있다. 단, 프로덕션에서는 성능 영향이 있으므로 디버깅 후 즉시 비활성화하라.

## 참고 자료

- [WiredTiger 공식 GitHub 저장소](https://github.com/wiredtiger/wiredtiger)
- [ethereumjs/ethereumjs-monorepo](https://github.com/ethereumjs/ethereumjs-monorepo)
- [XState — 상태 기계 라이브러리](https://github.com/statelyai/xstate)
