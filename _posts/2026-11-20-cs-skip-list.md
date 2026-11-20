---
layout: post
title: "Skip List 완전 정복: 확률적 균형으로 O(log N) 검색을 달성하는 천재적 자료구조"
date: 2026-11-20
categories: [cs, computer-science]
tags: [skip-list, data-structure, probabilistic, sorted-set, concurrent, redis]
---

균형 이진 탐색 트리(AVL, Red-Black Tree)는 O(log N) 검색을 보장하지만, 삽입과 삭제 시 복잡한 회전(rotation) 연산이 필요하다. 특히 동시성 환경에서의 잠금(locking) 처리는 구현을 극도로 복잡하게 만든다. 1989년 William Pugh는 간결하면서도 동일한 성능을 내는 대안을 제시했다: 무작위성을 이용해 균형을 "확률적으로" 달성하는 **Skip List**.

Redis의 Sorted Set, LevelDB의 메모리 테이블, Java의 `ConcurrentSkipListMap` — 현대 소프트웨어의 핵심 구성 요소들이 Skip List를 선택한 이유를 깊이 파헤쳐 보자.

## 개념: 여러 층의 고속도로

Skip List를 직관적으로 이해하려면 고속도로 시스템을 떠올려보자.

- **1층 (일반도로)**: 모든 요소가 연결된 완전한 연결 리스트
- **2층 (국도)**: 약 1/2 확률로 선택된 요소들의 연결 리스트
- **3층 (고속도로)**: 약 1/4 확률로 선택된 요소들의 연결 리스트
- **k층**: 약 1/2^k 확률로 선택된 요소들의 연결 리스트

검색할 때는 가장 높은 층(가장 적은 요소)에서 시작해 목표보다 커지면 한 층 아래로 내려가며 탐색한다. 이렇게 하면 평균 O(log N) 단계만에 목표를 찾을 수 있다.

```
레벨 3: 1 ─────────────────────────── 50 ──────────────────── NIL
레벨 2: 1 ──────────── 20 ─────────── 50 ──────── 80 ──────── NIL
레벨 1: 1 ──── 10 ──── 20 ──── 30 ─── 50 ──────── 80 ─── 90 ─ NIL
레벨 0: 1 ─ 5 ─ 10 ─── 20 ─ 25 ─ 30 ─ 50 ─ 60 ─── 80 ─── 90 ─ NIL
```

30을 찾는다면: 레벨3에서 1→50 (너무 큼, 아래로) → 레벨2에서 1→20→50 (너무 큼, 아래로) → 레벨1에서 20→30 (발견!). 단 5번의 비교로 완료.

## Skip List 구현 (Java)

```java
import java.util.Random;

public class SkipList<K extends Comparable<K>, V> {
    private static final int MAX_LEVEL = 16;
    private static final double P = 0.5;

    private static class Node<K, V> {
        K key;
        V value;
        Node<K, V>[] next;

        @SuppressWarnings("unchecked")
        Node(K key, V value, int level) {
            this.key = key;
            this.value = value;
            this.next = new Node[level + 1];
        }
    }

    private final Node<K, V> head;
    private int level;
    private int size;
    private final Random random = new Random();

    @SuppressWarnings("unchecked")
    public SkipList() {
        head = new Node<>(null, null, MAX_LEVEL);
        level = 0;
        size = 0;
    }

    private int randomLevel() {
        int lvl = 0;
        while (lvl < MAX_LEVEL && random.nextDouble() < P) {
            lvl++;
        }
        return lvl;
    }

    public void put(K key, V value) {
        @SuppressWarnings("unchecked")
        Node<K, V>[] update = new Node[MAX_LEVEL + 1];
        Node<K, V> curr = head;

        // 각 레벨에서 삽입 위치 직전 노드를 update[]에 기록
        for (int i = level; i >= 0; i--) {
            while (curr.next[i] != null &&
                   curr.next[i].key.compareTo(key) < 0) {
                curr = curr.next[i];
            }
            update[i] = curr;
        }

        // 이미 존재하면 값 업데이트
        if (curr.next[0] != null && curr.next[0].key.compareTo(key) == 0) {
            curr.next[0].value = value;
            return;
        }

        int newLevel = randomLevel();
        if (newLevel > level) {
            for (int i = level + 1; i <= newLevel; i++) {
                update[i] = head;
            }
            level = newLevel;
        }

        Node<K, V> newNode = new Node<>(key, value, newLevel);
        for (int i = 0; i <= newLevel; i++) {
            newNode.next[i] = update[i].next[i];
            update[i].next[i] = newNode;
        }
        size++;
    }

    public V get(K key) {
        Node<K, V> curr = head;
        for (int i = level; i >= 0; i--) {
            while (curr.next[i] != null &&
                   curr.next[i].key.compareTo(key) < 0) {
                curr = curr.next[i];
            }
        }
        curr = curr.next[0];
        if (curr != null && curr.key.compareTo(key) == 0) {
            return curr.value;
        }
        return null;
    }

    public boolean remove(K key) {
        @SuppressWarnings("unchecked")
        Node<K, V>[] update = new Node[MAX_LEVEL + 1];
        Node<K, V> curr = head;

        for (int i = level; i >= 0; i--) {
            while (curr.next[i] != null &&
                   curr.next[i].key.compareTo(key) < 0) {
                curr = curr.next[i];
            }
            update[i] = curr;
        }

        curr = curr.next[0];
        if (curr == null || curr.key.compareTo(key) != 0) {
            return false;
        }

        for (int i = 0; i <= level; i++) {
            if (update[i].next[i] != curr) break;
            update[i].next[i] = curr.next[i];
        }

        while (level > 0 && head.next[level] == null) {
            level--;
        }
        size--;
        return true;
    }

    public int size() { return size; }
}
```

### 사용 예시

```java
public class SkipListDemo {
    public static void main(String[] args) {
        SkipList<Integer, String> sl = new SkipList<>();

        // 삽입
        sl.put(5, "five");
        sl.put(3, "three");
        sl.put(7, "seven");
        sl.put(1, "one");
        sl.put(9, "nine");

        // 검색
        System.out.println(sl.get(3));  // three
        System.out.println(sl.get(6));  // null

        // 삭제
        sl.remove(5);
        System.out.println(sl.get(5));  // null
        System.out.println("Size: " + sl.size());  // 4
    }
}
```

## 성능 분석

Skip List의 시간 복잡도는 확률적으로 분석된다.

| 연산 | 평균 | 최악 (매우 드묾) |
|------|------|----------------|
| 검색 | O(log N) | O(N) |
| 삽입 | O(log N) | O(N) |
| 삭제 | O(log N) | O(N) |
| 범위 조회 | O(log N + K) | O(N) |

**공간 복잡도**: 각 노드가 평균 `1 / (1-P)` 개의 포인터를 가지므로, P=0.5일 때 평균 2개의 포인터. 전체 공간은 O(N).

이론적 최악의 경우는 모든 노드가 레벨 0에만 존재하는 경우이지만, `MAX_LEVEL = 16`, P = 0.5로 설정하면 N = 65,536개 요소에서 이 확률은 `2^-64` 보다 작다. 실용적으로는 O(log N)이 보장된다고 볼 수 있다.

## Lock-Free Skip List: 동시성의 강점

Skip List가 균형 BST보다 선호되는 가장 큰 이유는 **Lock-Free 구현의 용이성**이다. Java의 `ConcurrentSkipListMap`은 CAS(Compare-And-Swap) 연산만으로 완전한 스레드 안전성을 달성한다.

```python
# Python으로 보는 삽입 시 CAS 개념 (의사코드)
def concurrent_insert(skip_list, key, value):
    # 1. 선행 노드 탐색 (읽기 전용, 잠금 없음)
    update = find_predecessors(skip_list, key)
    
    # 2. 새 노드 생성
    new_node = Node(key, value, random_level())
    
    # 3. 최하위 레벨부터 CAS로 연결
    for level in range(new_node.height):
        while True:
            succ = update[level].next[level]
            new_node.next[level] = succ
            # CAS: update[level].next[level]가 여전히 succ이면 new_node로 교체
            if compare_and_swap(update[level].next[level], succ, new_node):
                break  # 성공
            # 실패 시 재시도 (다른 스레드가 먼저 수정함)
            update = find_predecessors(skip_list, key)  # 재탐색
```

Red-Black Tree를 Lock-Free로 구현하면 회전 연산 중 여러 포인터를 원자적으로 수정해야 해서 극도로 복잡해진다. Skip List에서는 각 레벨을 독립적으로 처리할 수 있어 CAS 연산이 자연스럽게 적용된다.

## Redis Sorted Set: 실전 적용

Redis는 Sorted Set을 내부적으로 Skip List + Hash Map 조합으로 구현한다:
- **Hash Map**: `member → score` 매핑 (O(1) 점수 조회)
- **Skip List**: score 기반 정렬 순서 유지 (O(log N) 범위 조회)

```redis
# Redis Sorted Set 명령어들이 Skip List의 특성을 활용
ZADD leaderboard 1500 "Alice"
ZADD leaderboard 2300 "Bob"
ZADD leaderboard 1800 "Charlie"

# 범위 조회 - Skip List의 O(log N + K) 강점
ZRANGEBYSCORE leaderboard 1000 2000 WITHSCORES
# → "Alice" 1500, "Charlie" 1800

# 순위 조회
ZRANK leaderboard "Bob"
# → 2 (0-indexed)
```

LevelDB와 RocksDB도 MemTable(쓰기 버퍼)을 Skip List로 구현한다. 동시 쓰기 처리량이 중요하고, 메모리 내 정렬 순서 유지가 필요한 SSTable 플러시 전 버퍼로 Skip List가 최적이다.

## Skip List vs 균형 BST 선택 기준

**Skip List를 선택해야 할 때:**
- 동시성이 중요하고 Lock-Free 구현이 필요할 때
- 범위 조회(range scan)가 빈번할 때 (연결 리스트이므로 순차 접근이 캐시 친화적)
- 구현 복잡도를 낮추고 싶을 때
- 레벨/크기를 동적으로 조정해야 할 때

**균형 BST를 선택해야 할 때:**
- 최악의 경우 O(log N)을 반드시 보장해야 할 때
- 메모리 사용량을 최소화해야 할 때
- 단일 스레드 환경에서 최대 성능이 필요할 때

## 주의사항과 팁

1. **P 값 선택**: P=0.5가 이론적으로 최적이지만, Redis는 P=0.25를 사용해 메모리와 성능의 균형을 맞춘다. P가 낮을수록 평균 레벨이 낮아 메모리는 절약되지만 검색 속도가 약간 저하된다.

2. **MAX_LEVEL 설정**: `MAX_LEVEL ≥ log_{1/P}(N)`으로 설정해야 최적 성능이 보장된다. N=10^6, P=0.5이면 MAX_LEVEL=20이 적절하다.

3. **캐시 미스 주의**: Skip List의 노드들은 메모리 상 연속적이지 않아 캐시 미스가 잦을 수 있다. 고성능이 필요하면 커스텀 메모리 풀(memory pool)을 사용하여 노드를 연속된 메모리 영역에 할당하면 성능을 크게 개선할 수 있다.

4. **랜덤성의 중요성**: Skip List의 성능은 무작위 레벨 생성기의 품질에 의존한다. 암호학적 보안이 필요한 환경에서는 `java.security.SecureRandom` 등 고품질 PRNG를 사용해야 한다.

## 참고 자료
- [Skip List - Wikipedia](https://en.wikipedia.org/wiki/Skip_list)
- [Skip Lists: A Probabilistic Alternative to Balanced Trees - William Pugh (UC Davis mirror)](https://www.cs.ucdavis.edu/~amenta/w04/skiplists.pdf)
- [Skip Lists: Done Right - CppCon 2016 (Brilliant slides on modern implementation)](https://en.wikipedia.org/wiki/Skip_list#References)
- [Redis Sorted Sets Internals](https://redis.io/docs/latest/develop/data-types/sorted-sets/)
