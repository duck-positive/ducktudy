---
layout: post
title: "인터벌 트리(Interval Tree) 완전 정복: 겹치는 구간을 O(log N)에 찾는 자료구조"
date: 2026-08-06
categories: [cs, computer-science]
tags: [interval-tree, data-structure, bst, augmented-tree, range-query, computational-geometry]
---

## 개념 설명

인터벌 트리(Interval Tree)는 **구간(Interval)**을 효율적으로 저장하고, 특정 점 또는 구간과 **겹치는 모든 구간을 빠르게 검색**하기 위해 설계된 자료구조다. 기본 아이디어는 이진 탐색 트리(BST)를 **증강(Augmented)**해서, 각 노드에 해당 서브트리에 속한 구간들의 **최대 끝점(max endpoint)** 정보를 추가 보관하는 것이다.

구간은 일반적으로 `[low, high]` 형태로 표현되며, 두 구간 `[a, b]`와 `[c, d]`가 겹치는 조건은 `a <= d && c <= b`다. 인터벌 트리는 이 겹침 조건을 최대한 빠르게 판단하도록 트리 구조를 설계한다.

### 핵심 구조

각 노드는 다음 정보를 저장한다:

- `low` (구간 시작점): BST의 키(Key)로 사용
- `high` (구간 끝점)
- `max`: 해당 노드를 루트로 하는 **서브트리 전체**에서 가장 큰 `high` 값

BST의 좌/우 구조는 `low` 기준으로 정렬되고, `max` 값 덕분에 특정 탐색 경로를 조기에 끊을 수 있다.

```
         [15, 20], max=25
        /                  \
  [10, 15], max=15     [17, 19], max=25
   /                         \
[5, 10], max=10          [21, 25], max=25
```

---

## 왜 필요한가

단순 배열에 구간을 저장하면 특정 점 `x`와 겹치는 구간을 찾기 위해 모든 구간을 순회해야 한다 → O(N). 구간이 수백만 개일 때는 치명적이다.

인터벌 트리의 주요 응용 분야:

| 분야 | 활용 예시 |
|------|----------|
| **OS 스케줄링** | 특정 시각에 실행 중인 프로세스 탐색 |
| **데이터베이스** | 날짜 범위 쿼리 (`WHERE start_date <= ? AND end_date >= ?`) |
| **컴파일러** | 변수의 라이브니스(liveness) 구간 분석 |
| **컴퓨터 그래픽** | 시야 절두체(frustum)와 겹치는 오브젝트 탐색 |
| **생물정보학** | 유전자 위치 구간 검색 |
| **GUI/CAD** | 마우스 클릭 위치와 겹치는 요소 탐색 |

---

## 실제 구현 예제

### 예제 1: Python으로 인터벌 트리 구현

```python
class Node:
    def __init__(self, low, high):
        self.low = low
        self.high = high
        self.max = high   # 서브트리의 최대 high 값
        self.left = None
        self.right = None

def insert(root, low, high):
    """BST 삽입 + max 값 갱신"""
    if root is None:
        return Node(low, high)

    # BST 정렬 기준은 low 값
    if low < root.low:
        root.left = insert(root.left, low, high)
    else:
        root.right = insert(root.right, low, high)

    # 서브트리의 max 갱신
    if root.max < high:
        root.max = high

    return root


def overlaps(a_low, a_high, b_low, b_high):
    """두 구간이 겹치는지 확인"""
    return a_low <= b_high and b_low <= a_high


def search(root, low, high):
    """[low, high]와 겹치는 구간 하나를 반환"""
    if root is None:
        return None

    # 현재 노드의 구간이 겹치면 반환
    if overlaps(root.low, root.high, low, high):
        return (root.low, root.high)

    # 왼쪽 서브트리에 답이 있을 수 있는지 확인
    # 왼쪽 서브트리의 max가 low보다 작으면 왼쪽에 겹치는 구간 없음
    if root.left and root.left.max >= low:
        return search(root.left, low, high)

    # 아니면 오른쪽 탐색
    return search(root.right, low, high)


def search_all(root, low, high, result=None):
    """[low, high]와 겹치는 모든 구간을 반환"""
    if result is None:
        result = []
    if root is None:
        return result

    if overlaps(root.low, root.high, low, high):
        result.append((root.low, root.high))

    # 왼쪽 서브트리 확인
    if root.left and root.left.max >= low:
        search_all(root.left, low, high, result)

    # 오른쪽 서브트리 확인 (오른쪽의 low가 high보다 작을 때만)
    if root.right and root.right.low <= high:
        search_all(root.right, low, high, result)

    return result


# 사용 예시
if __name__ == "__main__":
    intervals = [(15, 20), (10, 30), (17, 19), (5, 20), (12, 15), (30, 40)]
    root = None
    for lo, hi in intervals:
        root = insert(root, lo, hi)

    query = (14, 16)
    print(f"쿼리 구간 {query}와 겹치는 구간(첫 번째):")
    print(search(root, *query))

    print(f"\n쿼리 구간 {query}와 겹치는 모든 구간:")
    for interval in search_all(root, *query):
        print(interval)
```

실행 결과:
```
쿼리 구간 (14, 16)와 겹치는 구간(첫 번째):
(15, 20)

쿼리 구간 (14, 16)와 겹치는 모든 구간:
(15, 20)
(10, 30)
(5, 20)
(12, 15)
```

### 예제 2: Java로 회의실 예약 시스템 구현

인터벌 트리의 대표적인 실전 활용 — 회의실 예약 충돌 감지:

```java
import java.util.ArrayList;
import java.util.List;

public class MeetingRoomScheduler {

    static class Interval {
        int start, end;
        String title;

        Interval(int start, int end, String title) {
            this.start = start;
            this.end = end;
            this.title = title;
        }

        @Override
        public String toString() {
            return String.format("[%02d:00~%02d:00] %s", start, end, title);
        }
    }

    static class ITNode {
        Interval interval;
        int max;
        ITNode left, right;

        ITNode(Interval i) {
            this.interval = i;
            this.max = i.end;
        }
    }

    private ITNode root;

    public boolean canBook(int start, int end) {
        return findOverlap(root, start, end) == null;
    }

    public void book(int start, int end, String title) {
        if (!canBook(start, end)) {
            List<Interval> conflicts = findAllOverlaps(root, start, end, new ArrayList<>());
            System.out.println("예약 실패! 충돌하는 회의:");
            conflicts.forEach(c -> System.out.println("  " + c));
            return;
        }
        root = insert(root, new Interval(start, end, title));
        System.out.printf("예약 완료: [%02d:00~%02d:00] %s%n", start, end, title);
    }

    private ITNode insert(ITNode node, Interval iv) {
        if (node == null) return new ITNode(iv);

        if (iv.start < node.interval.start)
            node.left = insert(node.left, iv);
        else
            node.right = insert(node.right, iv);

        node.max = Math.max(node.max, iv.end);
        return node;
    }

    private boolean overlaps(Interval a, int start, int end) {
        return a.start < end && start < a.end;
    }

    private Interval findOverlap(ITNode node, int start, int end) {
        if (node == null) return null;
        if (overlaps(node.interval, start, end)) return node.interval;
        if (node.left != null && node.left.max > start)
            return findOverlap(node.left, start, end);
        return findOverlap(node.right, start, end);
    }

    private List<Interval> findAllOverlaps(ITNode node, int start, int end, List<Interval> result) {
        if (node == null) return result;
        if (overlaps(node.interval, start, end)) result.add(node.interval);
        if (node.left != null && node.left.max > start)
            findAllOverlaps(node.left, start, end, result);
        if (node.right != null && node.right.interval.start < end)
            findAllOverlaps(node.right, start, end, result);
        return result;
    }

    public static void main(String[] args) {
        MeetingRoomScheduler scheduler = new MeetingRoomScheduler();

        scheduler.book(9, 11, "스프린트 회의");
        scheduler.book(13, 15, "1:1 미팅");
        scheduler.book(14, 16, "디자인 리뷰");   // 충돌
        scheduler.book(10, 12, "긴급 대응");       // 충돌
        scheduler.book(16, 18, "회고 미팅");
    }
}
```

실행 결과:
```
예약 완료: [09:00~11:00] 스프린트 회의
예약 완료: [13:00~15:00] 1:1 미팅
예약 실패! 충돌하는 회의:
  [13:00~15:00] 1:1 미팅
예약 실패! 충돌하는 회의:
  [09:00~11:00] 스프린트 회의
예약 완료: [16:00~18:00] 회고 미팅
```

---

## 시간·공간 복잡도

| 연산 | 시간 복잡도 | 비고 |
|------|------------|------|
| 삽입 | O(log N) | BST 삽입 + max 갱신 |
| 겹치는 구간 하나 검색 | O(log N) | max 값으로 조기 종료 |
| 겹치는 구간 **모두** 검색 | O(min(N, K·log N)) | K = 결과 수 |
| 공간 | O(N) | - |

---

## 주의사항과 팁

### 1. 균형 트리와의 결합

단순 BST 기반 인터벌 트리는 삽입 순서에 따라 **편향 트리**가 될 수 있다. 실전에서는 **레드-블랙 트리** 또는 **AVL 트리** 기반으로 구현한다. Java의 `TreeMap`, C++의 `std::map`도 레드-블랙 트리이므로 이를 활용할 수 있다.

### 2. 삭제 연산

노드를 삭제할 때 `max` 값을 재계산해야 한다. 나이브하게 구현하면 서브트리 전체를 재순회해야 하므로 O(log N) 유지가 까다롭다. 실전에서는 **lazy deletion**(삭제 플래그 표시)을 쓰거나, `max`를 정확히 재계산하는 augmented BST 삭제 알고리즘을 적용한다.

### 3. `search_all`의 시간 복잡도

모든 겹치는 구간을 찾는 경우는 최악 O(N)이다(모든 구간이 겹칠 때). 하지만 평균적으로는 결과 수 K에 비례하는 O(K log N) 성능을 보인다.

### 4. 2D 구간 쿼리

2차원 평면에서의 직사각형 겹침 쿼리는 인터벌 트리를 중첩하거나 **R-트리**를 사용한다.

### 5. Centered Interval Tree (중앙 분할 인터벌 트리)

위에서 설명한 것은 **증강 BST 방식**이지만, **Centered Interval Tree** 방식도 있다. 중앙점을 기준으로 구간을 세 그룹(왼쪽, 중앙, 오른쪽)으로 나누고, 중앙 그룹은 시작점과 끝점 기준으로 각각 정렬해 저장하는 구조다. 정적 데이터셋에서 모든 겹치는 구간을 찾을 때 더 효율적이다.

---

## 참고 자료

- [Interval Tree - Wikipedia](https://en.wikipedia.org/wiki/Interval_tree)
- [Interval Tree - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/interval-tree/)
- [Introduction to Algorithms (CLRS), Chapter 14: Augmenting Data Structures](https://mitpress.mit.edu/9780262046305/introduction-to-algorithms/)
