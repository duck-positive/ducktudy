---
layout: post
title: "분할 상환 분석(Amortized Analysis) 완전 정복: 총계법·회계사법·물리학자법으로 자료구조 비용 분석하기"
date: 2026-07-21
categories: [cs, computer-science]
tags: [amortized-analysis, algorithms, data-structures, dynamic-array, union-find, complexity]
---

## 분할 상환 분석이란 무엇인가?

알고리즘의 시간 복잡도를 분석할 때 우리는 보통 최악의 경우(worst-case)를 기준으로 삼는다. 그런데 이 방식은 지나치게 비관적인 결론을 낼 때가 있다. 예를 들어, 동적 배열(dynamic array)에서 `append` 연산은 대부분 O(1)이지만, 배열이 꽉 찰 때 O(n)의 크기 조정(resize)이 발생한다. 그렇다고 `append`의 시간 복잡도를 O(n)이라 말하는 것은 현실과 동떨어진다.

**분할 상환 분석(Amortized Analysis)**은 이런 문제를 해결한다. 개별 연산의 비용이 아니라, **일련의 연산 전체에 대한 평균 비용**을 분석하는 방법이다. 핵심은 비싼 연산이 드물게 발생하고, 그 비용이 이전의 많은 저렴한 연산들로 "사전에 지불"되었다는 관점이다.

분할 상환 분석은 세 가지 방법으로 수행된다:
1. **총계법 (Aggregate Method)**: 전체 연산 비용의 합계를 구하고 연산 수로 나눈다
2. **회계사법 (Banker's Method, Accounting Method)**: 각 연산에 "가상 비용(amortized cost)"을 부여하고 크레딧을 관리한다
3. **물리학자법 (Physicist's Method, Potential Method)**: 자료구조의 "포텐셜 함수(potential function)"를 정의해 분석한다

---

## 왜 분할 상환 분석이 필요한가?

최악의 경우 분석만으로는 실제 성능을 과대평가하는 경우가 많다.

- **동적 배열 (Python list, Java ArrayList)**: `append`는 최악의 경우 O(n)이지만, n번 수행하면 총 비용이 O(n)이다. 즉 분할 상환 비용은 O(1)이다.
- **Union-Find with Path Compression**: 단일 `find` 연산은 O(log n)이지만, m번 연산의 분할 상환 비용은 사실상 O(α(n))으로 거의 상수다.
- **스플레이 트리 (Splay Tree)**: 단일 연산은 O(n)일 수 있지만, m번 연산의 분할 상환 비용은 O(m log n)이다.
- **피보나치 힙 (Fibonacci Heap)**: `decrease-key`와 `delete` 연산의 분할 상환 비용이 O(log n), O(1)이어서 Dijkstra 알고리즘의 총 복잡도를 O(E + V log V)로 낮춘다.

요약하면, 분할 상환 분석은 **실제 사용 시나리오에서의 효율성**을 더 정확하게 표현하는 도구다.

---

## 방법 1: 총계법 (Aggregate Method)

총계법은 가장 직관적인 방법이다. n번의 연산에 걸쳐 발생하는 **총 비용의 합 T(n)**을 계산한 뒤, 이를 n으로 나누어 연산당 분할 상환 비용을 구한다.

### 예제: 동적 배열 분석

```python
class DynamicArray:
    def __init__(self):
        self.data = [None]
        self.size = 0      # 현재 원소 수
        self.capacity = 1  # 내부 배열 크기
        self.total_ops = 0 # 총 비용 추적 (분석용)

    def append(self, item):
        if self.size == self.capacity:
            # 크기 조정: 현재 원소 전체를 새 배열에 복사
            self.total_ops += self.size  # 복사 비용
            new_data = [None] * (self.capacity * 2)
            for i in range(self.size):
                new_data[i] = self.data[i]
            self.data = new_data
            self.capacity *= 2

        self.data[self.size] = item
        self.size += 1
        self.total_ops += 1  # 삽입 비용

    def __len__(self):
        return self.size


# 총계법으로 분석
arr = DynamicArray()
for i in range(16):
    arr.append(i)

print(f"16번 append 후 총 비용: {arr.total_ops}")
# 크기 조정 발생: 1→2(1번 복사), 2→4(2번 복사), 4→8(4번 복사), 8→16(8번 복사)
# 복사 비용 합계: 1+2+4+8 = 15
# 삽입 비용: 16
# 총 비용: 31
# 분할 상환 비용 per operation: 31/16 ≈ 1.9 = O(1)

n = 1024
arr2 = DynamicArray()
for i in range(n):
    arr2.append(i)
print(f"{n}번 append 후 총 비용: {arr2.total_ops}")
print(f"분할 상환 비용 per op: {arr2.total_ops / n:.2f}")
# 출력: ~2047 / 1024 ≈ 2.0 → O(1) 확인
```

**총계법 증명**: n번 append에서 크기 조정은 크기가 1, 2, 4, 8, ..., 2^⌊log n⌋일 때 발생한다. 총 복사 비용은 1 + 2 + 4 + ... + n/2 = n - 1 < n이다. 삽입 비용 n을 더하면 총 비용은 3n 이하. 따라서 분할 상환 비용은 O(1)이다.

---

## 방법 2: 회계사법 (Banker's Method)

회계사법에서는 각 연산에 **실제 비용(actual cost)** 대신 **분할 상환 비용(amortized cost)**을 부여한다. 분할 상환 비용이 실제 비용보다 크면 차액을 **크레딧(credit)**으로 저장하고, 실제 비용이 더 크면 저장된 크레딧에서 인출한다. 언제나 크레딧 잔액이 0 이상임을 보이면 된다.

### 예제: 스택 연산 (MultiPop이 있는 스택)

```python
class CreditStack:
    def __init__(self):
        self.stack = []
        self.credits = []  # 각 원소의 크레딧

    def push(self, item):
        # 실제 비용: 1
        # 분할 상환 비용: 2 (1은 push 비용, 1은 나중의 pop을 위해 저장)
        self.stack.append(item)
        self.credits.append(1)  # 크레딧 1 저장

    def pop(self):
        if not self.stack:
            return None
        # 실제 비용: 1
        # 분할 상환 비용: 0 (크레딧에서 인출)
        self.credits.pop()
        return self.stack.pop()

    def multipop(self, k):
        # 실제 비용: min(k, len(stack))
        # 분할 상환 비용: 0 (각 원소의 크레딧으로 지불)
        count = 0
        while self.stack and k > 0:
            self.credits.pop()  # 크레딧 소비
            self.stack.pop()
            k -= 1
            count += 1
        return count

    @property
    def total_credits(self):
        return sum(self.credits)


# 시뮬레이션
s = CreditStack()
operations = [
    ('push', 1), ('push', 2), ('push', 3),
    ('push', 4), ('push', 5),
    ('multipop', 3),
    ('push', 6), ('push', 7),
    ('multipop', 10),
]

for op, val in operations:
    if op == 'push':
        s.push(val)
        print(f"push({val}): stack={s.stack}, credits={s.total_credits}")
    elif op == 'multipop':
        removed = s.multipop(val)
        print(f"multipop({val}): removed={removed}, credits={s.total_credits}")

# 핵심 관찰: total_credits는 항상 0 이상
# push는 분할 상환 비용 2, pop/multipop은 분할 상환 비용 0
# n번의 연산에 대해 총 분할 상환 비용 ≤ 2n → O(n)
```

---

## 방법 3: 물리학자법 (Physicist's Method, Potential Method)

물리학자법은 자료구조의 상태를 수치로 표현하는 **포텐셜 함수 Φ**를 정의한다.

```
분할 상환 비용 = 실제 비용 + ΔΦ = 실제 비용 + Φ(이후 상태) - Φ(이전 상태)
```

Φ(초기 상태) = 0, Φ(모든 상태) ≥ 0을 만족하면, 모든 연산의 분할 상환 비용 합이 실제 비용 합의 상한임을 보장한다.

### 예제: 동적 배열의 포텐셜 분석

```python
class PotentialDynamicArray:
    """
    포텐셜 Φ = 2 * size - capacity 로 정의
    (size가 capacity의 절반이면 Φ = 0, capacity가 꽉 차면 Φ = capacity)
    """
    def __init__(self):
        self.items = []
        self.size = 0
        self.capacity = 1

    def potential(self):
        return 2 * self.size - self.capacity

    def append(self, item):
        prev_potential = self.potential()

        if self.size == self.capacity:
            # resize 발생: 실제 비용 = size + 1 (복사 + 삽입)
            actual_cost = self.size + 1
            # 크기 2배로
            new_cap = self.capacity * 2
            self.capacity = new_cap
        else:
            actual_cost = 1

        self.size += 1
        if len(self.items) < self.size:
            self.items.append(item)
        else:
            self.items[self.size - 1] = item

        curr_potential = self.potential()
        amortized_cost = actual_cost + (curr_potential - prev_potential)
        return actual_cost, amortized_cost


# 분할 상환 비용이 상수인지 확인
arr = PotentialDynamicArray()
print(f"{'연산':>4} {'실제비용':>8} {'분할상환비용':>12} {'포텐셜':>8}")
for i in range(9):
    actual, amortized = arr.append(i)
    print(f"{i+1:>4} {actual:>8} {amortized:>12} {arr.potential():>8}")

# 출력 예시:
#    1        1            3        1  (capacity=1→2 시 resize)
#    2        1            3        2
#    3        3            3        3  (capacity=2→4 시 resize)
#  ...
# 분할 상환 비용이 항상 3 = O(1)임을 확인
```

포텐셜 함수 Φ = 2·size - capacity로 설정하면:
- 일반 삽입(크기 조정 없음): 실제 비용 1, ΔΦ = +2, 분할 상환 비용 = 3
- 크기 조정 삽입: 실제 비용 = size + 1, ΔΦ = 2·(size+1) - 2·capacity - (2·size - capacity) = 2 - capacity = 2 - size, 분할 상환 비용 = (size+1) + (2-size) = 3

어떤 경우든 분할 상환 비용이 3으로 상수임을 엄밀히 증명할 수 있다.

---

## 주의사항과 실전 팁

### 1. 포텐셜 함수 선택
물리학자법에서 적절한 포텐셜 함수를 찾는 것이 핵심이다. 일반적으로 "나중에 비쌀 연산을 준비하는 상태를 높은 포텐셜로"라는 직관을 따른다.

### 2. 크레딧 보존 불변식 확인
회계사법에서는 크레딧이 항상 0 이상임을 엄밀히 증명해야 한다. 이 불변식이 깨지면 분석이 무효가 된다.

### 3. 분할 상환 분석 ≠ 평균 분석
분할 상환 분석은 최악의 경우에도 성립하는 상한이다. 평균 분석(average-case analysis)은 입력 분포에 의존하지만, 분할 상환 분석은 임의의 연산 순서에 대해 항상 성립한다.

### 4. 적용 가능한 자료구조
- 동적 배열 (Python list): append O(1) 분할 상환
- 해시 테이블: insert/lookup O(1) 분할 상환 (재해싱 포함)
- Union-Find (경로 압축 + 크기 합치기): find O(α(n)) 분할 상환
- Splay Tree: 모든 기본 연산 O(log n) 분할 상환
- 피보나치 힙: decrease-key O(1), delete-min O(log n) 분할 상환

### 5. 실제 코드에서의 시사점
분할 상환 O(1)은 개별 연산이 가끔 O(n)임을 숨기지 않는다. 실시간 시스템(real-time system)이나 레이턴시에 민감한 시스템에서는 최악의 경우가 더 중요하다. 예를 들어 게임 엔진에서 파티클 풀(particle pool)을 동적 배열로 구현하면 특정 프레임에서 갑작스러운 긴 대기가 발생할 수 있다.

---

## 참고 자료

- [Amortized Analysis - Wikipedia](https://en.wikipedia.org/wiki/Amortized_analysis)
- [CMU CS 451 Amortized Analysis Lecture Notes](https://www.cs.cmu.edu/~15451-s15/LectureNotes/lecture06/victor-notes.pdf)
- [Cornell CS3110: Banker's and Physicist's Methods](https://courses.cs.cornell.edu/cs3110/2021sp/textbook/eff/amortized_methods.html)
- [Stanford CS166 Amortized Analysis Slides](https://web.stanford.edu/class/archive/cs/cs166/cs166.1166/lectures/07/Small07.pdf)
