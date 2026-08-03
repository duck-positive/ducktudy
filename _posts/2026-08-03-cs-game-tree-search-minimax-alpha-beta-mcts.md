---
layout: post
title: "게임 트리 탐색 완전 정복: 미니맥스, 알파-베타 가지치기, MCTS"
date: 2026-08-03
categories: [cs, computer-science]
tags: [game-tree, minimax, alpha-beta, mcts, ai, game-ai, search-algorithm]
---

## 개요

체스 AI가 수십 수 앞을 내다보거나, AlphaGo가 인간 챔피언을 꺾는 모습은 한때 공상과학 소설의 영역처럼 느껴졌다. 그 핵심에는 **게임 트리 탐색(Game Tree Search)** 알고리즘이 있다. 미니맥스(Minimax)에서 출발해 알파-베타 가지치기(Alpha-Beta Pruning)로 최적화하고, 몬테카를로 트리 탐색(MCTS)으로 확장하는 과정은 AI 역사에서 가장 아름다운 알고리즘 진화 중 하나다.

이 글에서는 세 알고리즘의 수학적 토대와 구현을 단계별로 설명하고, 실제 게임 AI에서 어떻게 활용되는지 살펴본다.

---

## 게임 트리란?

2인 제로섬 게임(두 플레이어의 이익 합이 0인 게임)에서 모든 가능한 상태(State)와 이동(Move)을 트리 구조로 표현한 것이 **게임 트리**다.

- **노드**: 게임 상태(보드 배치, 점수 등)
- **간선**: 한 플레이어의 이동
- **리프 노드**: 게임 종료 상태(승/패/무)
- **깊이(Depth)**: 현재까지의 이동 횟수

틱택토(Tic-Tac-Toe)의 경우 총 9! = 362,880개의 노드가 가능하지만, 체스는 평균 분기 인수(Branching Factor) ≈ 35로, 깊이 10만 탐색해도 35¹⁰ ≈ 2.8 × 10¹⁵개의 노드가 생성된다. 완전 탐색이 불가능하기 때문에 효율적인 탐색 전략이 필수다.

---

## 미니맥스(Minimax) 알고리즘

미니맥스는 두 플레이어를 **최대화 플레이어(Maximizer)**와 **최소화 플레이어(Minimizer)**로 구분한다. 최대화 플레이어는 자신에게 유리한 값을 최대화하고, 최소화 플레이어는 그 값을 최소화한다.

### 알고리즘 정의

```
minimax(node, depth, isMaximizingPlayer):
  if depth == 0 or node is terminal:
    return evaluate(node)
  
  if isMaximizingPlayer:
    value = -∞
    for child in children(node):
      value = max(value, minimax(child, depth-1, False))
    return value
  else:
    value = +∞
    for child in children(node):
      value = min(value, minimax(child, depth-1, True))
    return value
```

### Python 구현 — 틱택토 AI

```python
import math

def evaluate(board):
    """승/패/무를 평가하는 함수"""
    lines = [
        [0,1,2],[3,4,5],[6,7,8],  # 행
        [0,3,6],[1,4,7],[2,5,8],  # 열
        [0,4,8],[2,4,6]           # 대각선
    ]
    for line in lines:
        if board[line[0]] == board[line[1]] == board[line[2]]:
            if board[line[0]] == 'X':
                return 10
            elif board[line[0]] == 'O':
                return -10
    return 0

def is_terminal(board):
    """게임 종료 여부 확인"""
    return evaluate(board) != 0 or ' ' not in board

def minimax(board, depth, is_maximizing):
    score = evaluate(board)
    if score != 0:
        return score
    if is_terminal(board):
        return 0

    if is_maximizing:
        best = -math.inf
        for i in range(9):
            if board[i] == ' ':
                board[i] = 'X'
                best = max(best, minimax(board, depth + 1, False))
                board[i] = ' '
        return best
    else:
        best = math.inf
        for i in range(9):
            if board[i] == ' ':
                board[i] = 'O'
                best = min(best, minimax(board, depth + 1, True))
                board[i] = ' '
        return best

def best_move(board):
    """X 플레이어의 최적 이동 찾기"""
    best_val = -math.inf
    move = -1
    for i in range(9):
        if board[i] == ' ':
            board[i] = 'X'
            val = minimax(board, 0, False)
            board[i] = ' '
            if val > best_val:
                best_val = val
                move = i
    return move

# 사용 예시
board = ['X', 'O', 'X',
         'O', ' ', 'O',
         ' ', ' ', 'X']
print(f"최적 이동: {best_move(board)}")  # 인덱스 4 (가운데)
```

미니맥스는 정확하지만, 탐색 깊이가 깊어질수록 지수적으로 증가하는 노드 수가 치명적인 단점이다.

---

## 알파-베타 가지치기(Alpha-Beta Pruning)

알파-베타 가지치기는 미니맥스의 결과를 변경하지 않으면서 **불필요한 탐색을 건너뛰는** 최적화 기법이다. 최적 플레이를 가정할 때 절대 선택되지 않는 분기를 조기에 차단한다.

### 핵심 개념

- **α (Alpha)**: 최대화 플레이어가 현재까지 보장할 수 있는 최댓값 (하한선)
- **β (Beta)**: 최소화 플레이어가 현재까지 보장할 수 있는 최솟값 (상한선)
- **가지치기 조건**: `α ≥ β`이면 해당 분기를 탐색할 필요가 없다

### 가지치기가 발생하는 이유

최소화 플레이어가 이미 β = 3을 보장하는 상태에서, 최대화 플레이어가 α = 5인 분기를 탐색 중이라면, 최소화 플레이어는 절대 이 분기를 선택하지 않는다. 더 탐색해봤자 의미가 없으므로 즉시 건너뛴다.

### Python 구현 — 알파-베타 가지치기

```python
import math

def alpha_beta(board, depth, alpha, beta, is_maximizing):
    score = evaluate(board)
    if score != 0 or is_terminal(board):
        return score

    if is_maximizing:
        best = -math.inf
        for i in range(9):
            if board[i] == ' ':
                board[i] = 'X'
                val = alpha_beta(board, depth + 1, alpha, beta, False)
                board[i] = ' '
                best = max(best, val)
                alpha = max(alpha, best)
                if beta <= alpha:
                    break  # β 가지치기: 최소화 플레이어가 이 분기를 선택하지 않음
        return best
    else:
        best = math.inf
        for i in range(9):
            if board[i] == ' ':
                board[i] = 'O'
                val = alpha_beta(board, depth + 1, alpha, beta, True)
                board[i] = ' '
                best = min(best, val)
                beta = min(beta, best)
                if beta <= alpha:
                    break  # α 가지치기: 최대화 플레이어가 이 분기를 선택하지 않음
        return best

def best_move_ab(board):
    best_val = -math.inf
    move = -1
    for i in range(9):
        if board[i] == ' ':
            board[i] = 'X'
            val = alpha_beta(board, 0, -math.inf, math.inf, False)
            board[i] = ' '
            if val > best_val:
                best_val = val
                move = i
    return move
```

### 성능 비교

| 조건 | 미니맥스 노드 수 | 알파-베타 노드 수 |
|------|----------------|-----------------|
| 최악의 경우 | b^d | b^d |
| 평균 | b^d | b^(3d/4) |
| 최선의 경우 (최적 순서) | b^d | b^(d/2) |

분기 인수 b = 35, 깊이 d = 10이면, 최선의 경우 알파-베타는 35^5 ≈ 5천만 노드만 탐색하면 된다. 원본 대비 약 **56,000배** 감소!

---

## 몬테카를로 트리 탐색(MCTS)

체스나 바둑처럼 경우의 수가 폭발적으로 많거나, 정확한 평가 함수(Evaluation Function)를 만들기 어려운 게임에서는 알파-베타 가지치기도 한계에 부딪힌다. MCTS는 이 문제를 **랜덤 시뮬레이션**으로 해결한다.

### MCTS의 4단계

**1. 선택(Selection)**: UCB1(Upper Confidence Bound) 공식으로 탐색할 노드를 선택한다.

```
UCB1 = (w_i / n_i) + C × √(ln(N) / n_i)
```

- `w_i`: 노드 i의 승리 횟수
- `n_i`: 노드 i의 방문 횟수  
- `N`: 부모 노드의 방문 횟수
- `C`: 탐색-이용 균형 상수 (일반적으로 √2)

**2. 확장(Expansion)**: 선택된 리프 노드에서 미탐색 자식 노드를 추가한다.

**3. 시뮬레이션(Rollout/Playout)**: 랜덤하게 게임을 끝까지 진행해 결과를 얻는다.

**4. 역전파(Backpropagation)**: 시뮬레이션 결과를 선택 경로의 모든 노드에 반영한다.

### Python 구현 — MCTS 핵심 구조

```python
import math
import random
from collections import defaultdict

class MCTSNode:
    def __init__(self, state, parent=None, action=None):
        self.state = state
        self.parent = parent
        self.action = action        # 이 노드로 이어진 이동
        self.children = []
        self.wins = 0
        self.visits = 0
        self.untried_actions = state.get_legal_actions()

    def ucb1(self, c=math.sqrt(2)):
        if self.visits == 0:
            return float('inf')
        exploitation = self.wins / self.visits
        exploration = c * math.sqrt(math.log(self.parent.visits) / self.visits)
        return exploitation + exploration

    def is_fully_expanded(self):
        return len(self.untried_actions) == 0

    def best_child(self, c=math.sqrt(2)):
        return max(self.children, key=lambda node: node.ucb1(c))

    def expand(self):
        action = self.untried_actions.pop()
        next_state = self.state.apply_action(action)
        child = MCTSNode(next_state, parent=self, action=action)
        self.children.append(child)
        return child

def mcts(root_state, num_simulations=1000):
    root = MCTSNode(root_state)

    for _ in range(num_simulations):
        # 1. 선택
        node = root
        while not node.state.is_terminal() and node.is_fully_expanded():
            node = node.best_child()

        # 2. 확장
        if not node.state.is_terminal() and not node.is_fully_expanded():
            node = node.expand()

        # 3. 시뮬레이션 (랜덤 롤아웃)
        state = node.state
        while not state.is_terminal():
            action = random.choice(state.get_legal_actions())
            state = state.apply_action(action)
        result = state.get_result()

        # 4. 역전파
        while node is not None:
            node.visits += 1
            node.wins += result
            node = node.parent

    # 방문 횟수가 가장 많은 자식 선택 (탐색 없이 이용만)
    return max(root.children, key=lambda n: n.visits).action
```

---

## 실전 활용: 왜 AlphaGo가 혁명적이었나?

전통적인 게임 AI의 문제점:
- **바둑**: 분기 인수 ≈ 250, 평균 게임 길이 ≈ 150수 → 250^150 ≈ 10^360 (우주의 원자 수보다 많음)
- 평가 함수를 만들기도 극히 어려움

AlphaGo의 해법:
1. **MCTS** 기반 탐색
2. **Policy Network**: 어떤 수를 두면 좋을지 예측해 탐색 범위 축소
3. **Value Network**: 평가 함수 대신 딥러닝으로 승률 예측
4. **Self-Play**: 스스로 대전해 무한히 강화

이후 AlphaZero는 같은 구조로 체스, 장기, 바둑을 모두 정복했다.

---

## 실무 최적화 팁

### 1. 반복 심화(Iterative Deepening)
깊이 1, 2, 3, … 순서로 반복 탐색하며 시간 제한 내에 최선 이동을 찾는다. 각 반복에서 **킬러 무브(Killer Move)** 테이블을 재활용해 다음 반복의 정렬에 활용한다.

### 2. 전치표(Transposition Table)
동일한 게임 상태는 서로 다른 이동 순서로 도달할 수 있다. 해시맵으로 이미 계산한 상태의 값을 캐싱하면 중복 계산을 방지한다.

```python
transposition_table = {}

def alpha_beta_with_tt(board, depth, alpha, beta, is_max):
    key = (tuple(board), depth, is_max)
    if key in transposition_table:
        return transposition_table[key]
    # ... 계산 후
    transposition_table[key] = result
    return result
```

### 3. 이동 순서 최적화
알파-베타의 효율은 이동 순서에 크게 의존한다. 좋은 이동을 먼저 평가하면 가지치기가 더 많이 발생한다. MVV-LVA(Most Valuable Victim - Least Valuable Attacker) 같은 휴리스틱으로 이동 순서를 최적화한다.

### 4. MCTS 병렬화
루트 병렬화(Root Parallelization), 트리 병렬화(Tree Parallelization), 리프 병렬화(Leaf Parallelization) 세 가지 전략으로 MCTS를 다중 스레드로 병렬 실행할 수 있다.

---

## 주의사항

1. **Horizon Effect**: 미니맥스는 탐색 깊이 직후의 나쁜 상황을 놓칠 수 있다. Quiescence Search(전술적 교환이 완료될 때까지 추가 탐색)로 완화한다.

2. **MCTS 수렴 속도**: MCTS는 시뮬레이션 횟수가 적을 때 부정확하다. 복잡한 전술적 상황에서는 알파-베타보다 느리게 수렴할 수 있다.

3. **평가 함수 설계**: 미니맥스 기반 AI의 성능은 평가 함수 품질에 절대적으로 의존한다. 잘못된 평가 함수는 알고리즘이 아무리 정확해도 나쁜 AI를 만든다.

4. **시간 관리**: 실전 게임에서는 탐색 시간을 동적으로 관리해야 한다. 남은 시간, 게임 단계, 포지션 복잡도를 고려해 탐색 깊이를 조절한다.

---

## 참고 자료

- [Minimax Algorithm in Game Theory - GeeksforGeeks](https://www.geeksforgeeks.org/minimax-algorithm-in-game-theory-set-1-introduction/)
- [Minimax and MCTS - muens.io](https://muens.io/minimax-and-mcts/)
- [A Survey of Monte Carlo Tree Search Methods - Browne et al., IEEE TCIAIG 2012](https://ieeexplore.ieee.org/document/6145622)
- [Mastering the Game of Go with Deep Neural Networks and Tree Search - Silver et al., Nature 2016](https://www.nature.com/articles/nature16961)
