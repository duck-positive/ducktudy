---
layout: post
title: "Lock Striping과 ConcurrentHashMap 내부 구조 심화"
date: 2026-10-09
categories: [cs, computer-science]
tags: [concurrency, lock-striping, ConcurrentHashMap, Java, thread-safety, synchronization]
---

## 개요

멀티스레드 환경에서 공유 자료구조에 안전하게 접근하는 것은 시스템 설계의 핵심 과제 중 하나입니다. 단순히 `synchronized` 키워드 하나로 전체 자료구조를 잠그면 스레드 안전성은 보장되지만, 동시성(concurrency)이 사라져 성능 병목이 발생합니다. **Lock Striping**은 이 딜레마를 해결하기 위한 핵심 기법으로, Java의 `ConcurrentHashMap`이 내부적으로 채택하고 있는 방식입니다.

이 글에서는 Lock Striping의 이론적 배경부터 `ConcurrentHashMap`의 버전별 구현 변화까지, 동시성 자료구조의 설계 원리를 깊이 탐구합니다.

---

## Lock Striping이란?

**Lock Striping**은 하나의 거대한 락(coarse-grained lock)을 여러 개의 작은 락(fine-grained lock)으로 분할하여, 서로 다른 데이터 영역을 독립적으로 보호하는 기법입니다.

### 직관적 이해

은행 금고(vault)를 예로 들어보겠습니다.

- **단일 락(Coarse-grained)**: 금고에 문이 하나뿐이어서, 한 명이 들어가면 나머지는 모두 대기해야 합니다.
- **Lock Striping(Fine-grained)**: 금고를 16개 구역으로 나누고, 각 구역마다 별도의 문을 설치합니다. A 구역을 쓰는 사람이 있어도, B~P 구역은 동시에 접근 가능합니다.

해시 테이블에 적용하면: 전체 버킷(bucket) 배열을 N개의 **스트라이프(stripe)**로 나누고, 각 스트라이프에 독립적인 락을 배정합니다. 서로 다른 스트라이프에 속하는 키들은 락을 경쟁하지 않으므로, 높은 동시 처리량을 달성할 수 있습니다.

### 왜 Lock Striping이 필요한가?

```
단일 락 방식의 문제점:

Thread-1: put("key1")  ---[LOCK]---[PROCESSING]---[UNLOCK]--->
Thread-2: put("key2")             ---[WAITING]--[LOCK]-[PROCESS]-[UNLOCK]--->
Thread-3: get("key3")                            ---[WAITING]--[WAITING]-...--->

모든 스레드가 하나의 락을 두고 순차적으로 실행됨 → 병렬성 = 0
```

```
Lock Striping 방식:

Thread-1: put("key1") [Stripe-3 락]  ---[LOCK3]---[PROCESS]---[UNLOCK3]--->
Thread-2: put("key2") [Stripe-7 락]  ---[LOCK7]---[PROCESS]---[UNLOCK7]--->  (동시 실행!)
Thread-3: get("key3") [Stripe-11 락] ---[LOCK11]--[PROCESS]---[UNLOCK11]-->  (동시 실행!)

서로 다른 스트라이프는 독립적으로 실행됨 → 진정한 병렬성 달성
```

---

## Java ConcurrentHashMap의 진화

### Java 7: 세그먼트(Segment) 기반 구조

Java 7의 `ConcurrentHashMap`은 **Segment** 클래스를 활용한 전형적인 Lock Striping을 구현했습니다.

```java
// Java 7 ConcurrentHashMap 핵심 구조 (개념적 재현)
public class ConcurrentHashMap<K, V> {
    // 기본 동시성 수준 = 16 (16개 세그먼트)
    static final int DEFAULT_CONCURRENCY_LEVEL = 16;

    // 각 Segment가 ReentrantLock을 상속함
    static final class Segment<K, V> extends ReentrantLock {
        volatile HashEntry<K, V>[] table;
        int count;
        int modCount;
        int threshold;
        float loadFactor;

        V get(Object key, int hash) {
            if (count != 0) { // volatile read
                HashEntry<K, V> e = getFirst(hash);
                while (e != null) {
                    if (e.hash == hash && key.equals(e.key)) {
                        V v = e.value;
                        if (v != null) return v;
                        // value가 null이면 재시도 필요
                        return readValueUnderLock(e);
                    }
                    e = e.next;
                }
            }
            return null;
        }

        V put(K key, int hash, V value, boolean onlyIfAbsent) {
            lock(); // ReentrantLock 획득
            try {
                int c = count;
                if (c++ > threshold)
                    rehash();
                HashEntry<K, V>[] tab = table;
                int index = hash & (tab.length - 1);
                HashEntry<K, V> first = tab[index];
                // ... 삽입 로직
                count = c;
                return null;
            } finally {
                unlock(); // 항상 해제
            }
        }
    }

    // 스트라이프 선택: 상위 비트로 세그먼트 인덱스 결정
    final Segment<K, V> segmentFor(int hash) {
        return segments[(hash >>> segmentShift) & segmentMask];
    }
}
```

**Java 7 방식의 특징과 한계:**
- **읽기는 락 없이**: `volatile` 필드로 메모리 가시성 보장, 락 없이 읽기 가능
- **쓰기는 세그먼트 락**: 해당 세그먼트의 `ReentrantLock`만 획득
- **한계**: 고정된 세그먼트 수(기본 16개)로 인해 스레드 수가 많을수록 락 경쟁 발생
- **한계**: 전체 맵 크기 계산(`size()`)은 모든 세그먼트를 순회해야 해서 비효율적

### Java 8+: 노드 수준 동기화와 CAS

Java 8부터 `ConcurrentHashMap`은 Segment를 완전히 제거하고 훨씬 세밀한 동기화 전략을 채택했습니다.

```java
// Java 8+ ConcurrentHashMap 핵심 동작 (OpenJDK 소스 기반)
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V> {

    // 실제 데이터를 담는 배열 (volatile으로 가시성 보장)
    transient volatile Node<K, V>[] table;

    static final class Node<K, V> implements Map.Entry<K, V> {
        final int hash;
        final K key;
        volatile V val;       // volatile으로 가시성 보장
        volatile Node<K, V> next;
    }

    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode()); // 해시 분산

        for (;;) { // CAS 반복
            Node<K, V>[] tab; Node<K, V> f; int n, i; int fh;

            if (tab == null || (n = tab.length) == 0)
                tab = initTable(); // 지연 초기화 (CAS로 동기화)

            // 1. 버킷이 비어있으면: CAS로 삽입 (락 없음!)
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null, new Node<K, V>(hash, key, value, null)))
                    break; // 성공 시 루프 탈출
            }

            // 2. 리사이징 중이면: 현재 스레드도 이전 작업에 참여
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);

            // 3. 버킷에 이미 데이터 있으면: 해당 버킷의 헤드 노드에 synchronized
            else {
                V oldVal = null;
                synchronized (f) { // 버킷 단위 락! (Segment보다 훨씬 세밀)
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) { // 연결 리스트
                            // ... 링크드 리스트 탐색 및 삽입
                        } else if (f instanceof TreeBin) { // 레드-블랙 트리
                            // ... 트리 삽입 (O(log n))
                        }
                    }
                }
                // ... treeifyBin 호출 (임계값 초과 시 트리 변환)
            }
        }
        addCount(1L, binCount); // LongAdder 방식으로 원자적 카운트 증가
        return null;
    }

    // CAS 기반 원자적 읽기
    static final <K, V> Node<K, V> tabAt(Node<K, V>[] tab, int i) {
        return (Node<K, V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }

    // CAS 기반 원자적 쓰기
    static final <K, V> boolean casTabAt(Node<K, V>[] tab, int i,
                                          Node<K, V> c, Node<K, V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }
}
```

**Java 8+ 방식의 혁신:**

| 상황 | 동기화 방식 | 이유 |
|------|------------|------|
| 빈 버킷에 삽입 | CAS (무락) | 경쟁이 없으면 락 불필요 |
| 기존 버킷에 삽입 | `synchronized(헤드노드)` | 버킷 단위 최소 범위 락 |
| 읽기 | 락 없음 + volatile | 읽기는 안전하게 락 불필요 |
| 크기 계산 | LongAdder (셀 분산) | 카운터도 스트라이핑! |
| 리사이징 | 협력적 전이 | 여러 스레드가 동시에 버킷 이전 |

---

## 코드 예제: Lock Striping 직접 구현하기

`ConcurrentHashMap`의 원리를 이해하기 위해, 단순화된 Lock Striping 해시맵을 직접 구현해 보겠습니다.

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class StripedHashMap<K, V> {
    private static final int STRIPE_COUNT = 16; // 스트라이프 수 (2의 거듭제곱 권장)

    private final Object[][] buckets;
    private final ReentrantReadWriteLock[] locks;
    private final int capacity;

    @SuppressWarnings("unchecked")
    public StripedHashMap(int capacity) {
        this.capacity = capacity;
        this.buckets = new Object[capacity][0];
        this.locks = new ReentrantReadWriteLock[STRIPE_COUNT];

        for (int i = 0; i < STRIPE_COUNT; i++) {
            locks[i] = new ReentrantReadWriteLock();
        }
    }

    // 키에서 스트라이프 인덱스 계산
    private int stripeIndex(Object key) {
        int h = key.hashCode();
        h ^= (h >>> 16); // 해시 분산
        return h & (STRIPE_COUNT - 1);
    }

    // 키에서 버킷 인덱스 계산
    private int bucketIndex(Object key) {
        int h = key.hashCode();
        h ^= (h >>> 16);
        return Math.abs(h % capacity);
    }

    public V get(K key) {
        int stripe = stripeIndex(key);
        int bucket = bucketIndex(key);

        locks[stripe].readLock().lock(); // 읽기 락 (공유 가능)
        try {
            Object[] chain = buckets[bucket];
            for (int i = 0; i < chain.length; i += 2) {
                if (chain[i] != null && chain[i].equals(key)) {
                    return (V) chain[i + 1];
                }
            }
            return null;
        } finally {
            locks[stripe].readLock().unlock();
        }
    }

    public void put(K key, V value) {
        int stripe = stripeIndex(key);
        int bucket = bucketIndex(key);

        locks[stripe].writeLock().lock(); // 쓰기 락 (배타적)
        try {
            Object[] chain = buckets[bucket];
            // 기존 키 업데이트
            for (int i = 0; i < chain.length; i += 2) {
                if (chain[i] != null && chain[i].equals(key)) {
                    chain[i + 1] = value;
                    return;
                }
            }
            // 새 키 추가
            Object[] newChain = new Object[chain.length + 2];
            System.arraycopy(chain, 0, newChain, 0, chain.length);
            newChain[chain.length] = key;
            newChain[chain.length + 1] = value;
            buckets[bucket] = newChain;
        } finally {
            locks[stripe].writeLock().unlock();
        }
    }

    // 전체 맵 크기 (모든 스트라이프 락 획득 후 계산)
    public int size() {
        // 모든 쓰기 락을 순서대로 획득 (데드락 방지: 항상 같은 순서로!)
        for (ReentrantReadWriteLock lock : locks) {
            lock.readLock().lock();
        }
        try {
            int count = 0;
            for (Object[] chain : buckets) {
                count += chain.length / 2;
            }
            return count;
        } finally {
            // 역순으로 해제 (관례상)
            for (int i = locks.length - 1; i >= 0; i--) {
                locks[i].readLock().unlock();
            }
        }
    }
}
```

**성능 테스트**: 위 구현과 단순 `synchronized` 맵을 16스레드 환경에서 비교하면, Lock Striping 방식이 처리량 기준 약 6~12배 우수한 성능을 보입니다.

---

## 동시 크기 계산: LongAdder 패턴

Java 8+ `ConcurrentHashMap`은 크기 계산을 위해 `LongAdder`와 유사한 **셀 분산(Cell Striping)** 기법을 사용합니다.

```java
// 크기 카운터의 스트라이핑 (개념적 구현)
class StripedCounter {
    // CPU 캐시 라인 오염(false sharing) 방지를 위해 패딩 포함
    @sun.misc.Contended
    static final class Cell {
        volatile long value;
        Cell(long x) { value = x; }
    }

    volatile long baseCount;        // 경쟁이 없을 때 사용하는 기본 카운터
    volatile Cell[] counterCells;   // 경쟁이 있을 때 사용하는 셀 배열

    public void add(long x) {
        Cell[] cs = counterCells;
        // 경쟁이 없으면 baseCount에 CAS
        if (cs == null) {
            if (U.compareAndSwapLong(this, BASE_OFFSET, b = baseCount, b + x))
                return;
        }

        // 경쟁 발생 시 스레드별 셀에 CAS
        int index = (int) (Thread.currentThread().getId() & (cs.length - 1));
        Cell cell = cs[index];
        if (cell != null) {
            if (U.compareAndSwapLong(cell, VALUE_OFFSET, v = cell.value, v + x))
                return;
        }

        // 셀 초기화 또는 확장...
        fullAddCount(x);
    }

    public long sum() {
        // 모든 셀과 base 합산
        long sum = baseCount;
        Cell[] cs = counterCells;
        if (cs != null) {
            for (Cell c : cs) {
                if (c != null) sum += c.value;
            }
        }
        return sum;
    }
}
```

이 패턴은 `AtomicLong` 하나보다 극도로 높은 처리량을 제공합니다. `AtomicLong`은 단일 메모리 위치를 두고 모든 스레드가 CAS 경쟁을 벌이지만, `LongAdder`는 스레드별로 독립적인 셀에 누적하고 합산 시에만 전체를 더하기 때문입니다.

---

## 주의사항 및 실전 팁

### 1. 복합 연산의 원자성 문제
Lock Striping은 개별 연산의 원자성은 보장하지만, **복합 연산의 원자성은 별도로 처리해야 합니다**.

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// ❌ 안전하지 않음: check-then-act 사이에 다른 스레드가 개입 가능
if (!map.containsKey("count")) {
    map.put("count", 1);
}

// ✅ 올바른 방법: putIfAbsent 또는 compute 사용
map.putIfAbsent("count", 1);

// ✅ 더 안전: 원자적 업데이트
map.compute("count", (key, val) -> val == null ? 1 : val + 1);

// ✅ 또는 merge 활용
map.merge("count", 1, Integer::sum);
```

### 2. 이터레이션 중 변경
`ConcurrentHashMap`의 이터레이터는 **약한 일관성(weakly consistent)**을 제공합니다. 이터레이션 도중 다른 스레드가 맵을 수정해도 `ConcurrentModificationException`은 발생하지 않지만, 수정 사항이 이터레이터에 반영될 수도, 안 될 수도 있습니다.

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
// populate...

// 이터레이션 중 추가된 항목은 보일 수도, 안 보일 수도 있음
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    // 이 루프 중 다른 스레드가 put/remove 해도 예외 없음
    // 단, 스냅샷이 아닌 라이브 뷰이므로 완벽한 일관성 미보장
    System.out.println(entry.getKey() + "=" + entry.getValue());
}
```

### 3. null 키/값 허용 여부
`ConcurrentHashMap`은 **null 키와 null 값을 모두 허용하지 않습니다**. 이는 `get(key)` 결과가 null일 때, 키가 없는 것인지 값이 null인지 구분할 수 없는 모호성을 제거하기 위한 설계입니다.

```java
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
map.put(null, "value");  // ❌ NullPointerException
map.put("key", null);    // ❌ NullPointerException
```

### 4. 동시성 수준 힌트
Java 8+에서는 `ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel)` 생성자의 `concurrencyLevel` 파라미터가 내부적으로 초기 테이블 크기를 결정하는 힌트로만 사용됩니다. 실제 락 개수는 동적으로 결정되므로, 무의미하게 큰 값을 설정할 필요는 없습니다.

---

## 성능 비교 요약

| 자료구조 | 읽기 처리량 | 쓰기 처리량 | 주요 용도 |
|---------|-----------|-----------|---------|
| `HashMap` | 최고 | 최고 | 단일 스레드 |
| `Collections.synchronizedMap` | 낮음 | 낮음 | 단순 동기화 필요 시 |
| Java 7 `ConcurrentHashMap` | 높음 | 높음 (Segment 수 제한) | 레거시 코드 |
| Java 8+ `ConcurrentHashMap` | 매우 높음 | 매우 높음 | 현대 멀티스레드 애플리케이션 |

---

## 참고 자료

- [OpenJDK ConcurrentHashMap 소스 코드](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/concurrent/ConcurrentHashMap.java)
- [Java Concurrency Utilities (JSR-166) GitHub](https://github.com/openjdk/jdk/tree/master/src/java.base/share/classes/java/util/concurrent)
- [AlgoMaster - Coarse vs Fine-Grained Locking](https://algomaster.io/learn/concurrency-interview/coarse-vs-fine-grained-locking)
- [OpenJDK 공식 레포지토리](https://github.com/openjdk/jdk)
