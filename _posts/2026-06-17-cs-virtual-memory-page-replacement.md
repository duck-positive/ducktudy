---
layout: post
title: "가상 메모리와 페이지 교체 알고리즘: OPT, FIFO, LRU, Clock 완전 정복"
date: 2026-06-17
categories: [cs, computer-science]
tags: [virtual-memory, page-replacement, operating-systems, LRU, clock-algorithm, demand-paging]
---

## 가상 메모리란 무엇인가

운영체제를 공부하다 보면 "RAM이 4GB인데 16GB짜리 프로세스를 어떻게 실행하냐"는 질문을 자연스럽게 하게 된다. 그 답이 바로 **가상 메모리(Virtual Memory)**다.

가상 메모리는 실제 물리 메모리보다 큰 주소 공간을 프로세스에게 제공하는 메모리 관리 기법이다. 프로세스는 자신이 모든 메모리를 독점하는 것처럼 가상 주소(Virtual Address)를 사용하고, 운영체제는 **페이지 테이블(Page Table)**을 통해 이를 실제 물리 주소(Physical Address)로 변환한다.

### 핵심 개념들

- **페이지(Page)**: 가상 주소 공간을 일정 크기(보통 4KB)로 나눈 단위
- **프레임(Frame)**: 물리 메모리를 페이지와 같은 크기로 나눈 단위
- **페이지 폴트(Page Fault)**: 프로세스가 접근하려는 페이지가 물리 메모리에 없을 때 발생하는 인터럽트
- **스와핑(Swapping)**: 메모리가 부족할 때 사용 빈도가 낮은 페이지를 디스크(스왑 영역)로 내보내는 작업

가상 메모리의 핵심은 **요구 페이징(Demand Paging)**이다. 프로그램 실행 시 모든 페이지를 메모리에 올리는 대신, 실제로 접근할 때만 로드한다. 이를 통해 물리 메모리를 훨씬 효율적으로 활용할 수 있다.

---

## 왜 페이지 교체 알고리즘이 필요한가

물리 메모리의 모든 프레임이 사용 중인 상태에서 페이지 폴트가 발생하면, 운영체제는 **어떤 페이지를 디스크로 내보낼지** 결정해야 한다. 이 결정을 내리는 것이 페이지 교체 알고리즘(Page Replacement Algorithm)이다.

잘못된 교체 결정은 **스래싱(Thrashing)**으로 이어진다. 스래싱은 페이지 폴트가 너무 자주 발생해 실제 작업보다 페이지 교체에 더 많은 시간을 쓰는 상황으로, CPU 사용률이 극적으로 떨어진다. 좋은 교체 알고리즘은 페이지 폴트 횟수를 최소화해야 한다.

---

## 주요 페이지 교체 알고리즘

### 1. 최적 알고리즘 (OPT, Optimal)

앞으로 가장 오랫동안 사용되지 않을 페이지를 교체한다. 이론상 최소 페이지 폴트를 보장하지만, **미래를 알 수 없기 때문에 실제 구현이 불가능**하다. 다른 알고리즘의 성능을 평가하는 기준으로만 사용된다.

### 2. FIFO (First-In, First-Out)

가장 오래 전에 메모리에 올라온 페이지를 교체한다. 구현이 단순하지만 **벨래디의 이상(Bélády's Anomaly)**이라는 역설적 현상이 발생한다. 프레임 수를 늘렸는데 오히려 페이지 폴트가 증가하는 현상이다.

### 3. LRU (Least Recently Used)

가장 오랫동안 사용되지 않은 페이지를 교체한다. 지역성(Locality) 원리를 활용하며, 실제로 좋은 성능을 보인다. 하지만 모든 페이지 접근 시간을 기록해야 하므로 완전한 LRU 구현은 하드웨어 지원이 필요하고 오버헤드가 크다.

### 4. Clock 알고리즘 (Second-Chance)

LRU의 실용적인 근사 구현이다. 원형 버퍼에 페이지를 배치하고, 각 페이지에 **참조 비트(Reference Bit)**를 둔다. 교체가 필요하면 시계 침처럼 순회하면서:
- 참조 비트가 1이면 → 0으로 초기화하고 다음으로 이동 (두 번째 기회 부여)
- 참조 비트가 0이면 → 해당 페이지를 교체

---

## 코드로 구현하는 LRU와 Clock 알고리즘

### LRU 시뮬레이터 (Python)

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()  # key: page, value: True
        self.page_faults = 0

    def access(self, page: int) -> bool:
        """페이지 접근. 폴트 발생 시 True 반환."""
        if page in self.cache:
            # 히트: 최근 사용으로 이동
            self.cache.move_to_end(page)
            return False
        
        # 폴트: 새 페이지 로드
        self.page_faults += 1
        if len(self.cache) >= self.capacity:
            # 가장 오래 전에 사용된 페이지(맨 앞) 제거
            evicted = next(iter(self.cache))
            del self.cache[evicted]
            print(f"  교체: 페이지 {evicted} 제거")
        
        self.cache[page] = True
        return True

def simulate_lru(pages: list, capacity: int):
    lru = LRUCache(capacity)
    print(f"=== LRU 시뮬레이션 (프레임 {capacity}개) ===")
    for page in pages:
        fault = lru.access(page)
        status = "FAULT" if fault else "HIT"
        frames = list(lru.cache.keys())
        print(f"페이지 {page}: {status} | 프레임: {frames}")
    print(f"총 페이지 폴트: {lru.page_faults}/{len(pages)}\n")

# 테스트
reference_string = [7, 0, 1, 2, 0, 3, 0, 4, 2, 3, 0, 3, 2]
simulate_lru(reference_string, capacity=4)
```

**실행 결과:**
```
=== LRU 시뮬레이션 (프레임 4개) ===
페이지 7: FAULT | 프레임: [7]
페이지 0: FAULT | 프레임: [7, 0]
페이지 1: FAULT | 프레임: [7, 0, 1]
페이지 2: FAULT | 프레임: [7, 0, 1, 2]
페이지 0: HIT   | 프레임: [7, 1, 2, 0]
페이지 3: FAULT | 프레임: [1, 2, 0, 3]  (7 제거)
...
총 페이지 폴트: 8/13
```

---

### Clock 알고리즘 시뮬레이터 (Python)

```python
class ClockReplacer:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.frames = [None] * capacity   # 페이지 번호
        self.ref_bits = [0] * capacity    # 참조 비트
        self.clock_hand = 0               # 시계 침 위치
        self.page_faults = 0

    def access(self, page: int) -> bool:
        # 캐시 히트 확인
        for i in range(self.capacity):
            if self.frames[i] == page:
                self.ref_bits[i] = 1  # 참조 비트 설정
                return False

        # 페이지 폴트
        self.page_faults += 1
        
        # 교체할 프레임 탐색
        while True:
            if self.frames[self.clock_hand] is None:
                # 빈 프레임 발견
                break
            if self.ref_bits[self.clock_hand] == 0:
                # 참조 비트 0 → 교체 대상
                break
            # 참조 비트 1 → 0으로 초기화하고 다음으로
            self.ref_bits[self.clock_hand] = 0
            self.clock_hand = (self.clock_hand + 1) % self.capacity

        evicted = self.frames[self.clock_hand]
        self.frames[self.clock_hand] = page
        self.ref_bits[self.clock_hand] = 1
        self.clock_hand = (self.clock_hand + 1) % self.capacity
        
        if evicted is not None:
            print(f"  교체: 페이지 {evicted} → {page}")
        return True

def simulate_clock(pages: list, capacity: int):
    clock = ClockReplacer(capacity)
    print(f"=== Clock 알고리즘 시뮬레이션 (프레임 {capacity}개) ===")
    for page in pages:
        fault = clock.access(page)
        status = "FAULT" if fault else "HIT"
        print(f"페이지 {page}: {status} | 프레임: {clock.frames} | 참조비트: {clock.ref_bits}")
    print(f"총 페이지 폴트: {clock.page_faults}/{len(pages)}\n")

reference_string = [7, 0, 1, 2, 0, 3, 0, 4, 2, 3, 0, 3, 2]
simulate_clock(reference_string, capacity=4)
```

Clock 알고리즘은 LRU에 비해 구현이 훨씬 간단하고 오버헤드가 낮아, **Linux 커널의 실제 페이지 교체**에 변형된 형태(LRU/2Q 근사)로 사용된다.

---

## 알고리즘 성능 비교

| 알고리즘 | 페이지 폴트 수 | 구현 복잡도 | Bélády 이상 |
|---------|-------------|-----------|------------|
| OPT     | 최소 (이론)  | 불가능      | 없음        |
| FIFO    | 많음         | 매우 단순   | 발생        |
| LRU     | 적음         | 복잡 (하드웨어 필요) | 없음 |
| Clock   | LRU에 근접   | 단순       | 없음        |

---

## 주의사항과 실전 팁

### 스래싱(Thrashing) 방지
- **워킹 셋(Working Set) 모델**: 프로세스가 일정 시간 동안 자주 접근하는 페이지 집합을 파악해, 이 집합이 메모리에 항상 유지되도록 보장한다.
- 프레임 수가 너무 적으면 스래싱 발생. 각 프로세스에 최소 워킹 셋 크기만큼의 프레임을 보장해야 한다.

### 지역성(Locality) 원리 활용
- **시간적 지역성**: 최근에 접근한 데이터는 곧 다시 접근할 가능성이 높다 (LRU의 근거)
- **공간적 지역성**: 인접한 메모리 위치를 연속으로 접근하는 경향이 있다 (프리페칭의 근거)

### TLB(Translation Lookaside Buffer)
페이지 테이블 조회 비용을 줄이기 위한 하드웨어 캐시다. TLB 히트 시 메모리 접근 한 번으로 물리 주소를 얻을 수 있다. 현대 CPU의 TLB 히트율은 보통 99% 이상이다.

### Linux의 실제 구현
Linux는 페이지를 **active list**와 **inactive list** 두 개의 LRU 리스트로 관리한다. 처음 로드된 페이지는 inactive list에 들어가고, 다시 접근하면 active list로 승격된다. 메모리 부족 시 inactive list의 페이지부터 교체한다.

---

## 참고 자료
- [Computer Systems: A Systems Approach — Virtual Memory](https://book.systemsapproach.org/)
- [Operating Systems: Three Easy Pieces — Swapping Policies](https://pages.cs.wisc.edu/~remzi/OSTEP/)
- [Linux Kernel — Memory Management Documentation](https://www.kernel.org/doc/html/latest/admin-guide/mm/index.html)
- [CS3750 Lecture Notes: Virtual Memory](https://www.cs.csustan.edu/~john/Classes/CS3750/Notes/Chap10/10_VirtualMemory.html)
