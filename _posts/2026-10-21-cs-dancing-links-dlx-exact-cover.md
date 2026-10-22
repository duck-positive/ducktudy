---
layout: post
title: "Dancing Links(DLX)와 정확 피복 문제: Knuth의 Algorithm X 완전 정복"
date: 2026-10-21
categories: [cs, computer-science]
tags: [algorithm, backtracking, dancing-links, dlx, exact-cover, sudoku, knuth]
---

# Dancing Links(DLX)와 정확 피복 문제: Knuth의 Algorithm X 완전 정복

정확 피복(Exact Cover) 문제는 수많은 조합 최적화 문제의 핵심 틀을 이룬다. 스도쿠, 수학 퍼즐, 타일링 문제, N-Queens까지 이들은 모두 정확 피복 문제로 변환될 수 있다. Donald Knuth는 2000년 논문 "Dancing Links"에서 이 문제를 우아하게 해결하는 **Algorithm X**와 이를 구현한 **DLX(Dancing Links)**를 발표하며 조합 탐색 알고리즘의 새 장을 열었다.

## 개념 설명

### 정확 피복 문제란?

0과 1로 이루어진 행렬 $M$이 주어질 때, 각 열에 정확히 하나의 1이 있도록 행을 선택하는 문제를 **정확 피복(Exact Cover)** 문제라 한다.

예를 들어 다음 행렬을 보자:

```
      A B C D E F G
행 1: 1 0 0 1 0 0 1
행 2: 1 0 0 1 0 0 0
행 3: 0 0 0 1 1 0 1
행 4: 0 0 1 0 1 1 0
행 5: 0 1 1 0 0 1 1
행 6: 0 1 0 0 0 0 1
```

여기서 {행 2, 행 4, 행 6}을 선택하면 A, B, C, D, E, F, G 모두 정확히 한 번씩 커버된다. 이것이 정확 피복의 해다.

이 문제는 **NP-완전**이지만, Knuth의 알고리즘은 실제 문제에서 경이로운 속도를 발휘한다.

### Algorithm X: 백트래킹의 정수

Algorithm X는 재귀적 백트래킹 알고리즘이다. 핵심 절차는 다음과 같다:

1. 행렬이 비어 있으면 성공 — 해를 반환.
2. 열을 하나 선택한다(일반적으로 1의 개수가 가장 적은 열, 즉 MRV 휴리스틱).
3. 선택한 열에 1이 있는 각 행 $r$에 대해:
   - $r$을 해 집합에 추가.
   - $r$에 1이 있는 모든 열 $c$, 그리고 $c$에 1이 있는 모든 행을 행렬에서 제거(cover).
   - 재귀적으로 Algorithm X 실행.
   - 재귀가 실패하면 제거를 취소(uncover)하고 다음 행 시도.
4. 3번 과정의 모든 행에서 실패하면 백트래킹.

MRV(Minimum Remaining Values) 휴리스틱: 선택 가능한 1의 수가 가장 적은 열을 먼저 선택하면 탐색 공간을 극적으로 줄인다. 만약 1이 하나도 없는 열이 존재하면 즉시 백트래킹한다.

### Dancing Links: 연결 리스트로 구현하는 Cover/Uncover

Algorithm X에서 가장 비용이 큰 연산은 커버와 언커버, 즉 행/열 삽입·삭제다. Knuth는 이를 **원형 이중 연결 리스트(circular doubly-linked list)**로 해결했다.

각 1에 해당하는 노드는 네 방향 포인터를 가진다:
- **Left / Right**: 같은 행의 이웃 노드
- **Up / Down**: 같은 열의 이웃 노드
- **Column Header**: 해당 열의 헤더 노드 (1의 개수 관리)

**Cover(열 c를 제거)**:
```
c를 헤더 리스트에서 제거(Left-Right 링크 갱신)
c 열에 있는 각 행 r의 모든 노드를 Up-Down 리스트에서 제거
```

**Uncover(복원)**:
```
c 열의 각 행을 Up-Down 리스트에 복원
c를 헤더 리스트에 복원
```

이 때 "춤추는 링크"의 핵심 마법은 다음이다:
- 노드 x를 리스트에서 제거할 때: `x.Right.Left = x.Left; x.Left.Right = x.Right`
- 복원할 때: `x.Right.Left = x; x.Left.Right = x`

x의 포인터는 변경하지 않으므로, 포인터를 원래대로 유지한 채 복원이 O(1)에 이루어진다. 이것이 "Dancing Links"라는 이름의 유래다.

## 왜 필요한가?

### 나이브한 구현의 문제

배열 기반으로 행/열을 삽입·삭제하면 O(N) 비용이 든다. 백트래킹 특성상 같은 연산을 수백만 번 반복하는 경우, 이 차이가 수십 배의 성능 격차를 만든다.

DLX에서 커버/언커버는 O(커버된 노드 수)이고, 각 노드를 제거·복원하는 개별 연산이 O(1)이므로 전체 오버헤드가 최소화된다.

### 실제 응용

- **스도쿠 풀기**: 스도쿠를 정확 피복 행렬로 변환하면 DLX가 최고 난이도 퍼즐도 밀리초 내에 해결한다.
- **N-Queens 문제**: N개의 퀸을 체스판에 배치하는 문제를 정확 피복으로 변환.
- **펜토미노 타일링**: 다양한 폴리오미노 조각으로 직사각형을 채우는 퍼즐.
- **스케줄링**: 자원 할당 문제의 특수 케이스.

## 실제 구현 예제

### 예제 1: Python으로 구현하는 DLX 핵심 구조

```python
class Node:
    def __init__(self, col=None):
        self.left = self.right = self.up = self.down = self
        self.col = col  # ColumnHeader 참조
        self.row_id = -1

class ColumnHeader(Node):
    def __init__(self, name):
        super().__init__(col=self)
        self.name = name
        self.size = 0  # 이 열의 1 개수

class DLX:
    def __init__(self, num_cols):
        self.root = ColumnHeader("root")
        self.solution = []
        self.cols = []
        
        # 헤더 리스트 구성
        prev = self.root
        for i in range(num_cols):
            h = ColumnHeader(str(i))
            h.left = prev
            prev.right = h
            prev = h
            self.cols.append(h)
        prev.right = self.root
        self.root.left = prev

    def add_row(self, row_id, col_indices):
        first = None
        for ci in col_indices:
            col = self.cols[ci]
            node = Node(col)
            node.row_id = row_id
            # 위아래 연결 (열의 마지막에 추가)
            node.up = col.up
            node.down = col
            col.up.down = node
            col.up = node
            col.size += 1
            # 좌우 연결 (행 내부)
            if first is None:
                first = node
            else:
                node.left = first.left
                node.right = first
                first.left.right = node
                first.left = node

    def cover(self, col):
        col.right.left = col.left
        col.left.right = col.right
        row = col.down
        while row != col:
            node = row.right
            while node != row:
                node.down.up = node.up
                node.up.down = node.down
                node.col.size -= 1
                node = node.right
            row = row.down

    def uncover(self, col):
        row = col.up
        while row != col:
            node = row.left
            while node != row:
                node.col.size += 1
                node.down.up = node
                node.up.down = node
                node = node.left
            row = row.up
        col.right.left = col
        col.left.right = col

    def search(self, depth=0):
        if self.root.right == self.root:
            return True  # 해 발견
        
        # MRV: 가장 작은 열 선택
        col = self.root.right
        best = col
        c = col.right
        while c != self.root:
            if c.size < best.size:
                best = c
            c = c.right
        col = best
        
        if col.size == 0:
            return False  # 불가능
        
        self.cover(col)
        row = col.down
        while row != col:
            self.solution.append(row.row_id)
            node = row.right
            while node != row:
                self.cover(node.col)
                node = node.right
            
            if self.search(depth + 1):
                return True
            
            self.solution.pop()
            node = row.left
            while node != row:
                self.uncover(node.col)
                node = node.left
            row = row.down
        
        self.uncover(col)
        return False

# 사용 예: 위의 6x7 행렬 풀기
solver = DLX(7)  # A B C D E F G = 0 1 2 3 4 5 6
solver.add_row(0, [0, 3, 6])  # 행 1
solver.add_row(1, [0, 3])     # 행 2
solver.add_row(2, [3, 4, 6])  # 행 3
solver.add_row(3, [2, 4, 5])  # 행 4
solver.add_row(4, [1, 2, 5, 6])  # 행 5
solver.add_row(5, [1, 6])     # 행 6

if solver.search():
    print("해:", [s + 1 for s in solver.solution])  # 행 2, 4, 6
```

### 예제 2: 스도쿠를 정확 피복으로 변환

스도쿠를 DLX로 풀려면 다음 4가지 제약 조건을 열로 표현한다:

1. **셀 제약**: 각 셀에 정확히 하나의 숫자 (81열)
2. **행 제약**: 각 행에 1~9가 각 한 번씩 (81열)
3. **열 제약**: 각 열에 1~9가 각 한 번씩 (81열)
4. **박스 제약**: 각 3×3 박스에 1~9가 각 한 번씩 (81열)

총 324열, 최대 729행(9×9×9)의 행렬이 생성된다.

```python
def encode_sudoku(grid):
    """9x9 스도쿠 그리드를 정확 피복 행렬로 변환"""
    # 324열: 셀(81) + 행(81) + 열(81) + 박스(81)
    solver = DLX(324)
    row_id = 0
    
    for r in range(9):
        for c in range(9):
            for d in range(1, 10):  # 숫자 1~9
                if grid[r][c] not in (0, d):
                    row_id += 1
                    continue
                box = (r // 3) * 3 + (c // 3)
                cols = [
                    r * 9 + c,               # 셀 제약 (0~80)
                    81 + r * 9 + (d - 1),    # 행 제약 (81~161)
                    162 + c * 9 + (d - 1),   # 열 제약 (162~242)
                    243 + box * 9 + (d - 1)  # 박스 제약 (243~323)
                ]
                solver.add_row(row_id, cols)
                row_id += 1
    return solver

# 빈 스도쿠 풀기 (0은 빈 칸)
grid = [
    [5, 3, 0, 0, 7, 0, 0, 0, 0],
    [6, 0, 0, 1, 9, 5, 0, 0, 0],
    [0, 9, 8, 0, 0, 0, 0, 6, 0],
    [8, 0, 0, 0, 6, 0, 0, 0, 3],
    [4, 0, 0, 8, 0, 3, 0, 0, 1],
    [7, 0, 0, 0, 2, 0, 0, 0, 6],
    [0, 6, 0, 0, 0, 0, 2, 8, 0],
    [0, 0, 0, 4, 1, 9, 0, 0, 5],
    [0, 0, 0, 0, 8, 0, 0, 7, 9]
]
solver = encode_sudoku(grid)
if solver.search():
    print("스도쿠 풀림! 선택된 행 IDs:", solver.solution)
```

## 알고리즘 복잡도 분석

- **최악의 경우**: 지수 시간 O(k^n) — NP-완전 문제이므로 불가피
- **실제 성능**: MRV 휴리스틱과 조기 가지치기로 일반 스도쿠는 수백 μs 내 해결
- **커버/언커버**: O(커버된 노드 수), 각 포인터 연산 O(1)
- **메모리**: O(총 노드 수) — 행렬의 1의 개수에 비례

## 주의사항과 팁

### 열 선택 전략

MRV(Minimum Remaining Values) 전략은 "실패 우선" 원칙을 구현한다. 가장 제약이 많은 변수(열)를 먼저 선택하면:
- 불가능한 분기를 빠르게 감지
- 성공적인 해로 조기 수렴

만약 어떤 열의 size가 0이면 즉시 실패 — O(1) 조기 종료.

### 중복 해 제거

스도쿠처럼 해가 유일한 문제가 아니라면, `search()` 함수가 첫 번째 해를 찾은 뒤에도 계속 실행하도록 수정하여 모든 해를 열거할 수 있다.

### 초기 제약 조건 활용

스도쿠처럼 일부 셀이 미리 채워진 경우, 해당 행을 먼저 강제 선택하고 관련 열을 커버한 뒤 탐색을 시작하면 초기 탐색 공간을 크게 줄일 수 있다.

### Knuth의 철학

Algorithm X와 DLX는 단순하고 우아한 아이디어의 승리다. "원형 이중 연결 리스트의 삭제 후 포인터는 살아있다"는 단 하나의 관찰이 효율적인 백트래킹을 가능하게 만든다. 복잡한 데이터 구조 없이도, 포인터가 스스로 길을 기억한다.

## 참고 자료
- [Donald Knuth, "Dancing Links" (arXiv:cs/0011047)](https://arxiv.org/abs/cs/0011047)
- [Wikipedia: Knuth's Algorithm X](https://en.wikipedia.org/wiki/Knuth%27s_Algorithm_X)
- [Wikipedia: Dancing Links](https://en.wikipedia.org/wiki/Dancing_Links)
- [Wikipedia: Exact Cover](https://en.wikipedia.org/wiki/Exact_cover)
