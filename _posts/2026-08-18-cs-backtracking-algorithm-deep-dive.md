---
layout: post
title: "백트래킹(Backtracking) 완전 정복: N-Queens부터 스도쿠까지 제약 충족 문제 해결 전략"
date: 2026-08-18
categories: [cs, computer-science]
tags: [backtracking, algorithm, recursion, constraint-satisfaction, n-queens, sudoku, pruning]
---

백트래킹(Backtracking)은 수많은 경쟁 프로그래밍 문제와 실전 알고리즘 설계에서 핵심이 되는 기법이다. 단순한 완전 탐색을 넘어, **가지치기(pruning)** 를 통해 불가능한 경우를 조기에 차단함으로써 탐색 공간을 극적으로 줄인다. 본 글에서는 백트래킹의 동작 원리부터 핵심 최적화 기법, 그리고 N-Queens와 스도쿠 같은 대표 문제의 완전한 구현까지 깊이 있게 다룬다.

## 개념 설명: 백트래킹이란 무엇인가

백트래킹은 **해를 점진적으로 구성**하면서, 현재 부분 해가 최종 해가 될 수 없다고 판단되는 순간 즉시 탐색을 포기(prune)하고 이전 상태로 되돌아가는 알고리즘 패러다임이다.

```
해 후보를 하나씩 확장
    → 제약 조건 위반 시 즉시 되돌아가기 (prune)
    → 위반 없으면 다음 단계로 진행
    → 모든 단계가 채워지면 해 발견
```

개념적으로 **깊이 우선 탐색(DFS)** 과 동일한 구조를 가지지만, 핵심 차이는 **유효하지 않은 상태를 일찍 잘라낸다**는 점이다. 이를 통해 탐색 공간이 지수적으로 커지는 문제에서도 실용적인 실행 시간을 달성할 수 있다.

### 재귀 트리 모델

백트래킹 알고리즘은 재귀 트리(recursion tree)로 시각화된다. 트리의 각 노드는 현재까지의 부분 해를 나타내며, 리프 노드는 완전한 해 또는 실패한 후보다. 가지치기는 서브트리 전체를 탐색하지 않고 건너뛰는 것이다.

### 핵심 요소 3가지

1. **선택(Choice)**: 현재 단계에서 무엇을 선택할 수 있는가
2. **제약(Constraint)**: 어떤 조건이 위반되면 바로 포기할 수 있는가
3. **목표(Goal)**: 언제 해를 찾았다고 판단하는가

## 왜 필요한가: 완전 탐색과의 차이

N개의 원소에서 순열을 구하는 문제를 생각해 보자. 완전 탐색은 N! 개의 경우를 모두 나열한다. 하지만 백트래킹은 중간에 조건을 만족하지 않는 경우를 잘라내어 실제 탐색 노드 수를 대폭 줄인다.

예를 들어 8-Queens 문제에서 단순 완전 탐색은 8^8 = 16,777,216 가지의 배치를 확인해야 한다. 반면 백트래킹은 첫 번째 퀸 배치 후 두 번째 퀸의 유효 위치만 시도하는 방식으로 실제 탐색 노드 수를 약 15,720개 수준으로 줄일 수 있다.

## 실제 구현 예제 1: N-Queens 문제

N×N 체스판에 N개의 퀸을 서로 공격하지 않도록 배치하는 고전 문제다. 퀸은 같은 행, 열, 대각선 방향으로 공격한다.

```python
def solve_n_queens(n: int) -> list[list[str]]:
    results = []
    # 각 행에 퀸이 배치된 열 번호를 저장
    queens = [-1] * n
    # 열, 주대각선(row-col), 부대각선(row+col) 충돌 여부
    cols = set()
    diag1 = set()  # row - col
    diag2 = set()  # row + col

    def backtrack(row: int):
        if row == n:
            # 모든 행에 퀸 배치 완료 → 해 기록
            board = []
            for r in range(n):
                line = ['Q' if queens[r] == c else '.' for c in range(n)]
                board.append(''.join(line))
            results.append(board)
            return

        for col in range(n):
            if col in cols or (row - col) in diag1 or (row + col) in diag2:
                continue  # 충돌 발생 → 가지치기
            # 선택: 현재 행의 col 열에 퀸 배치
            queens[row] = col
            cols.add(col)
            diag1.add(row - col)
            diag2.add(row + col)

            backtrack(row + 1)  # 다음 행 탐색

            # 되돌리기(undo)
            queens[row] = -1
            cols.remove(col)
            diag1.remove(row - col)
            diag2.remove(row + col)

    backtrack(0)
    return results


# 사용 예시
solutions = solve_n_queens(8)
print(f"8-Queens 해의 수: {len(solutions)}")  # 92
for board in solutions[:1]:
    for row in board:
        print(row)
```

여기서 핵심은 `cols`, `diag1`, `diag2` 집합을 이용해 **O(1)** 시간에 충돌 여부를 판단하는 것이다. 충돌이 발생하면 해당 열(col)에 대한 탐색을 즉시 포기하고 다음 열로 넘어간다.

### 비트마스크를 활용한 최적화

메모리 접근 패턴을 개선하고 연산을 더 빠르게 하기 위해 비트마스크를 사용할 수 있다.

```python
def solve_n_queens_bitmask(n: int) -> int:
    full = (1 << n) - 1  # n개의 비트가 모두 1인 마스크

    def backtrack(cols: int, diag1: int, diag2: int) -> int:
        if cols == full:
            return 1  # 모든 열에 퀸 배치 완료
        
        # 가능한 위치: 세 마스크의 OR를 뒤집어서 아직 비어있는 열 추출
        available = full & ~(cols | diag1 | diag2)
        count = 0
        while available:
            # 가장 오른쪽 비트(가장 작은 유효 위치) 선택
            pos = available & (-available)
            available &= available - 1  # 선택한 비트 제거
            count += backtrack(
                cols | pos,
                (diag1 | pos) << 1,  # 주대각선: 다음 행에서 한 칸 오른쪽
                (diag2 | pos) >> 1,  # 부대각선: 다음 행에서 한 칸 왼쪽
            )
        return count

    return backtrack(0, 0, 0)


print(f"8-Queens 해의 수 (비트마스크): {solve_n_queens_bitmask(8)}")   # 92
print(f"12-Queens 해의 수 (비트마스크): {solve_n_queens_bitmask(12)}")  # 14200
```

비트마스크 버전은 집합 연산 대신 비트 연산을 사용해 캐시 효율이 훨씬 높다. `available & (-available)` 패턴은 가장 오른쪽 설정 비트를 추출하는 고전적인 비트 트릭이다.

## 실제 구현 예제 2: 스도쿠 풀이

9×9 스도쿠 퍼즐을 백트래킹으로 푸는 구현이다. 각 행, 열, 3×3 박스에 1~9가 정확히 한 번씩 나타나야 한다.

```python
def solve_sudoku(board: list[list[str]]) -> None:
    """board를 in-place로 수정하여 스도쿠를 해결한다."""
    rows = [set() for _ in range(9)]
    cols = [set() for _ in range(9)]
    boxes = [set() for _ in range(9)]
    empty_cells = []

    # 초기 상태 파싱
    for r in range(9):
        for c in range(9):
            val = board[r][c]
            if val == '.':
                empty_cells.append((r, c))
            else:
                num = int(val)
                rows[r].add(num)
                cols[c].add(num)
                boxes[(r // 3) * 3 + c // 3].add(num)

    def backtrack(idx: int) -> bool:
        if idx == len(empty_cells):
            return True  # 모든 빈 칸을 채움 → 성공

        r, c = empty_cells[idx]
        box_idx = (r // 3) * 3 + c // 3

        for num in range(1, 10):
            if num in rows[r] or num in cols[c] or num in boxes[box_idx]:
                continue  # 제약 위반 → 가지치기

            # 선택
            board[r][c] = str(num)
            rows[r].add(num)
            cols[c].add(num)
            boxes[box_idx].add(num)

            if backtrack(idx + 1):
                return True  # 성공이면 되돌리지 않음

            # 되돌리기
            board[r][c] = '.'
            rows[r].remove(num)
            cols[c].remove(num)
            boxes[box_idx].remove(num)

        return False  # 어떤 숫자도 유효하지 않음

    backtrack(0)


# 예시 스도쿠 (0은 빈 칸을 '.'로 표시)
board = [
    ['5','3','.','.','7','.','.','.','.'],
    ['6','.','.','1','9','5','.','.','.'],
    ['.','9','8','.','.','.','.','6','.'],
    ['8','.','.','.','6','.','.','.','3'],
    ['4','.','.','8','.','3','.','.','1'],
    ['7','.','.','.','2','.','.','.','6'],
    ['.','6','.','.','.','.','2','8','.'],
    ['.','.','.','4','1','9','.','.','5'],
    ['.','.','.','.','8','.','.','7','9'],
]
solve_sudoku(board)
for row in board:
    print(' '.join(row))
```

이 구현의 특징은 `rows`, `cols`, `boxes` 집합을 미리 구축해 각 숫자의 유효성을 O(1)에 확인한다는 점이다. 또한 빈 칸 리스트를 미리 만들어 순서대로 채우므로 탐색 순서가 일관적이다.

### 더 강력한 최적화: MRV(Minimum Remaining Values) 휴리스틱

CSP(Constraint Satisfaction Problem) 맥락에서 효과적인 최적화 전략은 **가장 선택지가 적은 변수부터 시도**하는 것이다. 스도쿠에서는 가능한 숫자가 가장 적은 빈 칸부터 채우면 불필요한 탐색을 더 일찍 차단할 수 있다.

## 백트래킹의 설계 패턴

대부분의 백트래킹 문제는 다음 틀로 표현된다:

```
def backtrack(상태):
    if 종료 조건:
        결과 기록/반환
        return

    for 선택지 in 가능한_선택지들:
        if 제약 조건 위반:
            continue  # 가지치기

        선택지 적용 (상태 변경)
        backtrack(다음_상태)
        선택지 취소 (상태 복원)  # undo
```

이 패턴의 핵심은 **상태 변경 후 반드시 되돌리는** 것이다. 이를 통해 각 재귀 호출이 독립적인 탐색 경로를 갖는다.

## 주의사항 및 팁

### 1. 가지치기 조건을 최대한 이른 시점에 적용하라

제약 위반 확인을 최대한 일찍 할수록 더 많은 서브트리를 건너뛸 수 있다. 확인 비용이 다르다면 저렴한 조건을 먼저 확인하라.

### 2. 상태 되돌리기(undo)를 빠트리지 마라

상태 복원을 잊으면 탐색 경로 간에 상태가 오염되어 잘못된 결과가 나온다. 집합 자료구조를 쓰면 `add`/`remove`로 깔끔하게 관리된다.

### 3. 해의 개수 vs 하나의 해

모든 해를 구해야 한다면 성공 후에도 계속 탐색한다. 하나의 해만 필요하다면 첫 번째 성공에서 즉시 `True`를 반환해 나머지 탐색을 건너뛴다.

### 4. 순열 vs 조합에서의 시작 인덱스

순열 문제에서는 매 재귀마다 처음부터 탐색하고 방문 여부를 관리한다. 조합 문제에서는 시작 인덱스를 증가시켜 중복을 피한다.

```python
# 순열: 방문 배열 사용
def perm(path, visited):
    if len(path) == n:
        print(path)
        return
    for i in range(n):
        if not visited[i]:
            visited[i] = True
            perm(path + [nums[i]], visited)
            visited[i] = False

# 조합: 시작 인덱스 증가
def comb(start, path):
    if len(path) == k:
        print(path)
        return
    for i in range(start, n):
        comb(i + 1, path + [nums[i]])
```

### 5. 비트마스크로 상태를 압축하라

N이 작을 때(≤20) 방문 여부나 선택 상태를 정수 비트마스크로 관리하면 메모리와 속도 면에서 유리하다.

백트래킹은 알고리즘의 기초이자 많은 고급 기법의 출발점이다. 제약 전파(Constraint Propagation), Forward Checking, Arc Consistency 같은 CSP 최적화 기법은 모두 백트래킹 위에서 동작하며, SAT 솔버의 CDCL 알고리즘 역시 백트래킹의 정교한 발전형이다.

## 참고 자료
- [TheAlgorithms/Python - backtracking/n_queens.py](https://github.com/TheAlgorithms/Python/blob/master/backtracking/n_queens.py)
- [labuladong/fucking-algorithm - Backtracking Algorithm](https://github.com/labuladong/fucking-algorithm)
