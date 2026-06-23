---
layout: post
title: "해시 테이블 충돌 해결 완전 정복: Robin Hood, Cuckoo Hashing과 현대적 설계"
date: 2026-06-23
categories: [cs, computer-science]
tags: [hash-table, robin-hood-hashing, cuckoo-hashing, data-structures, open-addressing, algorithms]
---

## 개념 설명: 해시 테이블 충돌이란

해시 테이블(Hash Table)은 O(1) 평균 시간복잡도로 삽입·검색·삭제를 제공하는 자료구조다. 하지만 해시 함수가 서로 다른 두 키를 같은 버킷 인덱스로 매핑하면 **충돌(Collision)**이 발생한다. 아무리 좋은 해시 함수를 사용해도 **Birthday Paradox** 에 의해 충돌은 피할 수 없다.

테이블 크기가 n이고 원소가 k개일 때, 충돌 확률은 다음과 같다:

```
P(충돌 없음) ≈ e^(-k²/2n)
```

k = √n 만 되어도 충돌 확률이 약 39%에 달한다. 충돌 해결 방법이 곧 해시 테이블 성능을 결정한다.

### 고전적 충돌 해결: 체이닝(Chaining)

가장 단순한 방법으로, 각 버킷에 연결 리스트를 두어 충돌 원소를 연결한다.

```
Bucket 0: [키A] → [키B] → null
Bucket 1: [키C]
Bucket 2: null
Bucket 3: [키D] → [키E] → [키F] → null
```

**장점**: 구현 단순, 적재율(load factor)이 높아져도 잘 동작  
**단점**: 포인터 추적으로 캐시 미스 빈발, 동적 메모리 할당 오버헤드, 연결 리스트 탐색 시 O(k)

### 오픈 어드레싱(Open Addressing)

체이닝과 달리 모든 원소를 테이블 배열 자체에 저장한다. 충돌 시 프로빙(probing) 전략으로 빈 슬롯을 찾는다.

**선형 프로빙(Linear Probing)**: `(h(k) + i) mod n`  
- 캐시 친화적이지만 **1차 군집(primary clustering)** 문제 발생  
- 한 곳에 원소가 몰리면 프로브 길이가 폭증

**이차 프로빙(Quadratic Probing)**: `(h(k) + i²) mod n`  
- 군집 완화하지만 **2차 군집(secondary clustering)** 과 테이블 크기에 의존

**이중 해시(Double Hashing)**: `(h₁(k) + i·h₂(k)) mod n`  
- 군집 최소화, 구현 복잡

---

## Robin Hood 해싱: 부익빈 부익부를 역전한다

### 핵심 아이디어

Robin Hood 해싱은 Pedro Celis가 1986년 박사 논문에서 제안한 오픈 어드레싱 변형이다. 핵심 불변 조건은 다음과 같다:

> **삽입 시, 현재 슬롯에 있는 원소보다 새 원소의 '프로브 거리(probe distance)'가 길면, 두 원소의 위치를 교환한다.**

**프로브 거리(DIB: Distance from Initial Bucket)**는 원소가 이상적인 위치에서 얼마나 멀리 떨어진 슬롯에 저장됐는지를 나타낸다.

```
이상적 위치 = h(key)
실제 위치   = 현재 인덱스
DIB = 실제 위치 - 이상적 위치
```

이름에서 알 수 있듯, "부유한"(프로브 거리가 짧은) 원소에게서 슬롯을 빼앗아 "가난한"(프로브 거리가 긴) 원소에게 준다. 이를 통해 **프로브 거리의 분산을 최소화**하여 평균 검색 시간이 극도로 낮아진다.

### 왜 Robin Hood 해싱이 우수한가

1. **조기 종료(Early Termination)**: 검색 시 현재 슬롯 원소의 DIB가 탐색 중인 키의 DIB보다 작으면, 이후 슬롯에 해당 키가 있을 수 없으므로 즉시 "없음"을 반환할 수 있다.
2. **분산 최소화**: 모든 원소의 DIB가 비슷하게 유지되어 최악 케이스 성능이 크게 개선된다.
3. **캐시 효율**: 오픈 어드레싱이므로 배열에 연속 저장되어 캐시 히트율이 높다.

---

## 왜 필요한가: 실제 사용 사례

Robin Hood 해싱은 다음 시스템에서 활용된다:
- **Rust의 `HashMap`**: Rust 표준 라이브러리의 `std::collections::HashMap`은 Robin Hood 해싱 기반의 `hashbrown` 크레이트를 사용한다.
- **Python 3.6+ dict**: Python 딕셔너리도 오픈 어드레싱 변형을 사용한다.
- **C++ absl::flat_hash_map**: Google Abseil 라이브러리.

---

## 실제 구현 예제

### 예제 1: Robin Hood 해시 테이블 구현 (Python)

```python
class RobinHoodHashTable:
    EMPTY = object()  # 빈 슬롯 센티넬

    def __init__(self, capacity=16):
        self.capacity = capacity
        self.size = 0
        self.keys   = [self.EMPTY] * capacity
        self.values = [None]       * capacity
        self.dibs   = [0]          * capacity  # Distance from Initial Bucket

    def _hash(self, key):
        return hash(key) % self.capacity

    def _probe(self, idx):
        return (idx + 1) % self.capacity

    def insert(self, key, value):
        if self.size / self.capacity >= 0.7:
            self._resize()

        idx  = self._hash(key)
        dib  = 0
        cur_key = key
        cur_val = value

        while True:
            if self.keys[idx] is self.EMPTY:
                # 빈 슬롯 발견 — 삽입
                self.keys[idx]   = cur_key
                self.values[idx] = cur_val
                self.dibs[idx]   = dib
                self.size += 1
                return

            if self.keys[idx] == cur_key:
                # 같은 키 업데이트
                self.values[idx] = cur_val
                return

            # Robin Hood: 내 DIB가 현재 슬롯 원소보다 크면 교환
            if dib > self.dibs[idx]:
                self.keys[idx],   cur_key = cur_key,   self.keys[idx]
                self.values[idx], cur_val = cur_val,   self.values[idx]
                self.dibs[idx],   dib     = dib,       self.dibs[idx]

            idx = self._probe(idx)
            dib += 1

    def search(self, key):
        idx = self._hash(key)
        dib = 0

        while True:
            if self.keys[idx] is self.EMPTY:
                return None  # 없음

            if self.keys[idx] == key:
                return self.values[idx]

            # 조기 종료: 내 DIB보다 현재 슬롯 원소 DIB가 작으면 없음
            if dib > self.dibs[idx]:
                return None

            idx = self._probe(idx)
            dib += 1

    def delete(self, key):
        idx = self._hash(key)
        dib = 0

        while True:
            if self.keys[idx] is self.EMPTY:
                return False
            if self.keys[idx] == key:
                break
            if dib > self.dibs[idx]:
                return False
            idx = self._probe(idx)
            dib += 1

        # 백워드 시프트 삭제 (빈 슬롯 없이 연속성 유지)
        self.keys[idx] = self.EMPTY
        self.size -= 1
        prev = idx
        idx  = self._probe(idx)
        while self.keys[idx] is not self.EMPTY and self.dibs[idx] > 0:
            self.keys[prev]   = self.keys[idx]
            self.values[prev] = self.values[idx]
            self.dibs[prev]   = self.dibs[idx] - 1
            self.keys[idx]    = self.EMPTY
            prev = idx
            idx  = self._probe(idx)
        return True

    def _resize(self):
        old_keys, old_vals = self.keys, self.values
        self.capacity *= 2
        self.size = 0
        self.keys   = [self.EMPTY] * self.capacity
        self.values = [None]       * self.capacity
        self.dibs   = [0]          * self.capacity
        for k, v in zip(old_keys, old_vals):
            if k is not self.EMPTY:
                self.insert(k, v)


# 테스트
ht = RobinHoodHashTable()
for i in range(10):
    ht.insert(f"key{i}", i * 100)

print(ht.search("key5"))   # 500
print(ht.search("key9"))   # 900
ht.delete("key5")
print(ht.search("key5"))   # None
print(ht.search("key6"))   # 600 (삭제 후에도 정상 동작)
```

---

## Cuckoo 해싱: O(1) 최악 케이스 검색 보장

### 핵심 아이디어

Cuckoo 해싱(Cuckoo Hashing)은 Rasmus Pagh와 Flemming Friche Rodler가 2001년 제안했다. 두 개의 해시 함수 h₁, h₂와 두 개의 테이블 T₁, T₂를 사용한다.

**불변 조건**: 키 k는 반드시 `T₁[h₁(k)]` 또는 `T₂[h₂(k)]` 중 한 곳에만 존재한다.

**검색**: 두 슬롯만 확인하면 되므로 **O(1) 최악 케이스** 보장.

**삽입 "뻐꾸기 알" 전략**:
1. T₁[h₁(k)]가 비어 있으면 삽입.
2. 비어 있지 않으면 그 원소를 퇴거(evict)시키고 그 자리에 k 삽입.
3. 퇴거된 원소를 반대 테이블의 해당 슬롯에 삽입 (재귀적으로 반복).
4. 사이클이 감지되면 rehash.

### 예제 2: Cuckoo 해시 테이블 구현 (Java)

```java
import java.util.Arrays;
import java.util.Random;

public class CuckooHashTable<K, V> {
    private static final int MAX_LOOP = 100; // 사이클 방지용 최대 이동 횟수
    private int capacity;
    private Object[] keys1, keys2;
    private Object[] vals1, vals2;
    private long seed1, seed2;
    private int size;

    public CuckooHashTable(int capacity) {
        this.capacity = capacity;
        this.keys1 = new Object[capacity];
        this.keys2 = new Object[capacity];
        this.vals1 = new Object[capacity];
        this.vals2 = new Object[capacity];
        Random rng = new Random();
        this.seed1 = rng.nextLong();
        this.seed2 = rng.nextLong();
    }

    private int h1(Object key) {
        return (int) ((key.hashCode() ^ seed1) & Integer.MAX_VALUE) % capacity;
    }

    private int h2(Object key) {
        return (int) ((key.hashCode() ^ seed2) & Integer.MAX_VALUE) % capacity;
    }

    @SuppressWarnings("unchecked")
    public V get(K key) {
        int i1 = h1(key), i2 = h2(key);
        if (key.equals(keys1[i1])) return (V) vals1[i1];
        if (key.equals(keys2[i2])) return (V) vals2[i2];
        return null;  // O(1) 최악 케이스!
    }

    @SuppressWarnings("unchecked")
    public boolean put(K key, V value) {
        // 업데이트 처리
        int i1 = h1(key), i2 = h2(key);
        if (key.equals(keys1[i1])) { vals1[i1] = value; return true; }
        if (key.equals(keys2[i2])) { vals2[i2] = value; return true; }

        Object curKey = key;
        Object curVal = value;

        for (int loop = 0; loop < MAX_LOOP; loop++) {
            // T1 시도
            i1 = h1(curKey);
            if (keys1[i1] == null) {
                keys1[i1] = curKey; vals1[i1] = curVal;
                size++;
                return true;
            }
            // T1에서 퇴거
            Object evictKey = keys1[i1];
            Object evictVal = vals1[i1];
            keys1[i1] = curKey; vals1[i1] = curVal;
            curKey = evictKey; curVal = evictVal;

            // T2 시도
            i2 = h2(curKey);
            if (keys2[i2] == null) {
                keys2[i2] = curKey; vals2[i2] = curVal;
                size++;
                return true;
            }
            // T2에서 퇴거
            evictKey = keys2[i2];
            evictVal = vals2[i2];
            keys2[i2] = curKey; vals2[i2] = curVal;
            curKey = evictKey; curVal = evictVal;
        }

        // MAX_LOOP 초과 → 사이클 감지, rehash 필요
        rehash();
        return put((K) curKey, (V) curVal);
    }

    public boolean remove(K key) {
        int i1 = h1(key), i2 = h2(key);
        if (key.equals(keys1[i1])) { keys1[i1] = null; vals1[i1] = null; size--; return true; }
        if (key.equals(keys2[i2])) { keys2[i2] = null; vals2[i2] = null; size--; return true; }
        return false;
    }

    private void rehash() {
        Object[] ok1 = keys1, ok2 = keys2;
        Object[] ov1 = vals1, ov2 = vals2;
        capacity *= 2;
        keys1 = new Object[capacity]; keys2 = new Object[capacity];
        vals1 = new Object[capacity]; vals2 = new Object[capacity];
        Random rng = new Random();
        seed1 = rng.nextLong(); seed2 = rng.nextLong();
        size = 0;
        for (int i = 0; i < ok1.length; i++) {
            if (ok1[i] != null) put((K)ok1[i], (V)ov1[i]);
            if (ok2[i] != null) put((K)ok2[i], (V)ov2[i]);
        }
    }

    public static void main(String[] args) {
        CuckooHashTable<String, Integer> table = new CuckooHashTable<>(16);
        for (int i = 0; i < 10; i++) {
            table.put("key" + i, i * 10);
        }
        System.out.println(table.get("key3"));  // 30
        System.out.println(table.get("key9"));  // 90
        table.remove("key3");
        System.out.println(table.get("key3"));  // null
    }
}
```

---

## Robin Hood vs Cuckoo vs Chaining 비교

| 특성 | Chaining | 선형 프로빙 | Robin Hood | Cuckoo |
|------|----------|------------|------------|--------|
| 검색 평균 | O(1) | O(1) | O(1) | O(1) |
| 검색 최악 | O(n) | O(n) | O(log n) | **O(1)** |
| 삽입 평균 | O(1) | O(1) | O(1) | O(1) amortized |
| 삽입 최악 | O(n) | O(n) | O(log n) | O(n) (rehash) |
| 캐시 효율 | 낮음 | 매우 높음 | 높음 | 중간 |
| 메모리 효율 | 낮음 (포인터) | 높음 | 높음 | 중간 |
| 구현 복잡도 | 낮음 | 낮음 | 중간 | 높음 |

---

## 주의사항 및 팁

### 1. 적재율(Load Factor) 임계값

- 체이닝: 0.75~1.0 허용
- Robin Hood / 선형 프로빙: **0.7 이하** 유지 필수 (그 이상이면 성능 급락)
- Cuckoo: **0.5 이하** 유지 (두 테이블 합산 기준)

### 2. 해시 함수 품질이 최우선

충돌 해결 전략보다 **해시 함수 품질**이 더 중요하다. 프로덕션에서는 MurmurHash3, xxHash, WyHash 같은 비암호학적 고속 해시 함수를 사용하자. Java `hashCode()`나 Python `hash()`는 충분하지 않을 수 있다.

### 3. Cuckoo 해싱의 rehash 비용

Cuckoo 삽입 실패율은 적재율 50%에서 약 0.1%다. 예기치 않은 rehash가 레이턴시 스파이크를 야기할 수 있으므로, 실시간 시스템에서는 백그라운드 rehash 또는 점진적 rehash를 고려해야 한다.

### 4. 언어별 기본 구현

- **Java**: `HashMap` — chaining (링크드 리스트 → 8개 초과 시 레드-블랙 트리로 전환)
- **Python**: `dict` — open addressing (compact design, 버전별 세부 구현 다름)
- **Rust**: `std::collections::HashMap` → `hashbrown` (Robin Hood 기반 SIMD 최적화 버전인 SwissTable 도입)
- **Go**: `map` — chaining 기반 버킷 구조

### 5. SwissTable: 현대적 Robin Hood의 진화

Google이 설계한 SwissTable은 Robin Hood 해싱에 SIMD(SSE/AVX) 명령어를 결합해 16개 슬롯을 한 번에 스캔한다. Rust의 `hashbrown`, C++의 `absl::flat_hash_map`이 이를 채택했다.

---

## 정리

해시 테이블 충돌 해결은 체이닝의 간단함에서 Robin Hood의 균등한 프로브 분포, Cuckoo의 O(1) 최악 보장으로 발전했다. 현대 고성능 시스템은 Robin Hood 계열(SwissTable)이 캐시 효율과 성능을 가장 잘 균형잡는다. 자신이 사용하는 언어의 해시 맵 구현을 이해하면 예측 가능한 성능 설계가 가능해진다.

## 참고 자료
- [Hash Table — Wikipedia](https://en.wikipedia.org/wiki/Hash_table)
- [Hash Table Internals & Collision Strategies — Medium](https://mjmichael.medium.com/hash-table-internals-collision-strategies-a-deep-dive-python-go-rust-2eb2bc414208)
- [Cuckoo Hashing — DEV Community](https://dev.to/vivekyadav200988/cuckoo-hashing-an-efficient-collision-resolution-technique-with-o1-lookup-time-2j9p)
- [How to Handle Hash Collisions: A Deep Dive — Deploy Mastery](https://www.deploymastery.com/2023/05/24/how-to-handle-hash-collisions-a-deep-dive/)
