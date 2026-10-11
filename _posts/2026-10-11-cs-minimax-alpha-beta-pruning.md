---
layout: post
title: "Minimax 알고리즘과 알파-베타 가지치기 완전 정복: 게임 AI가 최선의 수를 찾는 방법"
date: 2026-10-11
categories: [cs, computer-science]
tags: [minimax, alpha-beta-pruning, game-theory, ai, game-tree-search, iterative-deepening, mcts]
---

체스 프로그램은 어떻게 수백만 가지의 수 중에서 최선을 선택할까요? 바둑 AI는 어떤 원리로 다음 착점을 결정할까요? 이 모든 것의 근간에는 **Minimax 알고리즘**과 이를 극적으로 최적화하는 **알파-베타 가지치기(Alpha-Beta Pruning)**가 있습니다. 1950년대에 탄생한 이 알고리즘은 오늘날에도 체스 엔진 Stockfish를 비롯한 수많은 게임 AI의 핵심으로 사용됩니다.

## 게임 트리와 Minimax의 기본 원리

### 제로섬 게임과 게임 트리

체스, 바둑, 오목, 틱택토처럼 두 플레이어가 번갈아 두고, 한쪽의 이득이 다른 쪽의 손실이 되는 게임을 **제로섬 게임(Zero-sum game)**이라 합니다.

이런 게임의 모든 가능한 상태를 나타낸 것이 **게임 트리(Game Tree)**입니다:
- 루트: 현재 게임 상태
- 내부 노드: 중간 게임 상태
- 엣지: 가능한 수(move)
- 리프: 게임 종료 상태 (승/패/무승부) 또는 탐색 중단 지점

### Minimax의 전제

- **Maximizer**: 내 점수를 최대화하려는 플레이어 (나)
- **Minimizer**: 상대의 점수를 최소화하려는 플레이어 (상대)
- 두 플레이어 모두 **완벽하게 합리적으로 플레이**한다고 가정

이 가정 하에, Minimax는 게임 트리를 끝까지 탐색해 **내가 최선을 다했을 때 보장되는 최고의 결과**를 구합니다.

## Minimax 알고리즘 구현

```python
import math
from typing import List, Optional, Tuple

class TicTacToe:
    """틱택토 게임 상태"""
    
    def __init__(self):
        self.board = [' '] * 9  # 3x3 보드를 1차원으로
        self.current_player = 'X'
    
    def get_moves(self) -> List[int]:
        return [i for i, cell in enumerate(self.board) if cell == ' ']
    
    def apply(self, pos: int):
        self.board[pos] = self.current_player
        self.current_player = 'O' if self.current_player == 'X' else 'X'
    
    def undo(self, pos: int):
        self.board[pos] = ' '
        self.current_player = 'O' if self.current_player == 'X' else 'X'
    
    def is_terminal(self) -> bool:
        return self.get_winner() is not None or not self.get_moves()
    
    def get_winner(self) -> Optional[str]:
        lines = [
            (0,1,2), (3,4,5), (6,7,8),  # 행
            (0,3,6), (1,4,7), (2,5,8),  # 열
            (0,4,8), (2,4,6)             # 대각선
        ]
        for a, b, c in lines:
            if self.board[a] == self.board[b] == self.board[c] != ' ':
                return self.board[a]
        return None

def evaluate(game: TicTacToe) -> int:
    """게임 상태 평가 함수 (+10: X 승, -10: O 승, 0: 무승부)"""
    winner = game.get_winner()
    if winner == 'X': return 10
    if winner == 'O': return -10
    return 0

def minimax(game: TicTacToe, depth: int, is_maximizing: bool) -> int:
    """기본 Minimax 알고리즘"""
    if depth == 0 or game.is_terminal():
        score = evaluate(game)
        # 더 빨리 끝날수록 좋으므로 깊이로 점수 조정
        return score - depth if score > 0 else score + depth if score < 0 else 0
    
    if is_maximizing:
        best = -math.inf
        for move in game.get_moves():
            game.apply(move)
            best = max(best, minimax(game, depth - 1, False))
            game.undo(move)
        return best
    else:
        best = math.inf
        for move in game.get_moves():
            game.apply(move)
            best = min(best, minimax(game, depth - 1, True))
            game.undo(move)
        return best

def best_move_minimax(game: TicTacToe) -> int:
    """Minimax로 최선의 수 찾기"""
    best_val = -math.inf
    best_pos = -1
    for move in game.get_moves():
        game.apply(move)
        val = minimax(game, depth=9, is_maximizing=False)
        game.undo(move)
        if val > best_val:
            best_val = val
            best_pos = move
    return best_pos
```

## 알파-베타 가지치기: Minimax의 극적인 최적화

### 왜 가지치기가 필요한가

순수 Minimax는 탐색 공간이 기하급수적으로 커집니다. 체스의 경우:
- 평균 분기 계수(branching factor) ≈ 35
- 게임 평균 수(game length) ≈ 80수
- 전체 게임 트리: 35^80 ≈ 10^123 노드 (우주의 원자 수보다 많음)

10수만 내다봐도 35^10 ≈ 2.8조 노드를 탐색해야 합니다. 이를 해결하는 것이 알파-베타 가지치기입니다.

### 알파-베타의 핵심 아이디어

두 개의 경계값을 유지합니다:
- **α(알파)**: Maximizer가 현재까지 보장받은 최솟값 (최소 이만큼은 얻을 수 있음)
- **β(베타)**: Minimizer가 현재까지 보장받은 최댓값 (최대 이만큼밖에 줄 수 없음)

`β ≤ α`가 되는 순간, 현재 브랜치는 더 탐색할 필요가 없습니다. 상대방이 이 경로를 선택하지 않을 것이기 때문입니다.

```python
def alpha_beta(
    game: TicTacToe,
    depth: int,
    alpha: float,
    beta: float,
    is_maximizing: bool
) -> int:
    """알파-베타 가지치기"""
    if depth == 0 or game.is_terminal():
        score = evaluate(game)
        return score - depth if score > 0 else score + depth if score < 0 else 0
    
    if is_maximizing:
        best = -math.inf
        for move in game.get_moves():
            game.apply(move)
            value = alpha_beta(game, depth - 1, alpha, beta, False)
            game.undo(move)
            best = max(best, value)
            alpha = max(alpha, best)
            if beta <= alpha:
                break  # Beta cutoff: Minimizer가 이 브랜치를 선택하지 않음
        return best
    else:
        best = math.inf
        for move in game.get_moves():
            game.apply(move)
            value = alpha_beta(game, depth - 1, alpha, beta, True)
            game.undo(move)
            best = min(best, value)
            beta = min(beta, best)
            if beta <= alpha:
                break  # Alpha cutoff: Maximizer가 이 브랜치를 선택하지 않음
        return best

def best_move_ab(game: TicTacToe) -> Tuple[int, int]:
    """알파-베타로 최선의 수와 평가값 반환"""
    best_val = -math.inf
    best_pos = -1
    for move in game.get_moves():
        game.apply(move)
        val = alpha_beta(
            game, depth=9,
            alpha=-math.inf, beta=math.inf,
            is_maximizing=False
        )
        game.undo(move)
        if val > best_val:
            best_val = val
            best_pos = move
    return best_pos, best_val

# 성능 비교 테스트
import time

game = TicTacToe()
# 몇 수 두어서 중반 상태로
for pos in [4, 0, 2]:
    game.apply(pos)

t1 = time.perf_counter()
move1 = best_move_minimax(game)
t2 = time.perf_counter()
move2, _ = best_move_ab(game)
t3 = time.perf_counter()

print(f"Minimax:    {t2-t1:.4f}초, 선택 수={move1}")
print(f"Alpha-Beta: {t3-t2:.4f}초, 선택 수={move2}")
```

## 고급 최적화 기법

### 반복 심화 (Iterative Deepening, ID)

실제 게임 AI는 시간 제한이 있습니다(체스: 수당 수초). 고정된 깊이가 아닌, 시간 안에 완료 가능한 최대 깊이까지 탐색하는 방법이 **반복 심화**입니다.

```python
import time

def iterative_deepening_search(game: TicTacToe, time_limit: float) -> int:
    """
    시간 제한 내에 가능한 최대 깊이로 탐색
    얕은 탐색에서 얻은 수를 깊은 탐색에서 먼저 시도하여 가지치기 효율 극대화
    """
    start_time = time.perf_counter()
    best_move = game.get_moves()[0]  # 폴백: 첫 번째 수
    best_move_by_depth = {}
    
    for depth in range(1, 20):
        if time.perf_counter() - start_time > time_limit:
            break
        
        best_val = -math.inf
        # 이전 깊이에서 좋았던 수를 먼저 시도 (이동 순서 최적화)
        moves = game.get_moves()
        if depth > 1 and best_move in moves:
            moves.remove(best_move)
            moves.insert(0, best_move)
        
        for move in moves:
            if time.perf_counter() - start_time > time_limit:
                break
            game.apply(move)
            val = alpha_beta(game, depth - 1, -math.inf, math.inf, False)
            game.undo(move)
            if val > best_val:
                best_val = val
                best_move = move
        
        best_move_by_depth[depth] = best_move
        if abs(best_val) >= 10:  # 확실한 승리/패배 발견
            break
    
    return best_move
```

### 이동 순서 최적화 (Move Ordering)

알파-베타의 효율은 이동 순서에 크게 좌우됩니다. 좋은 수를 먼저 탐색하면 더 빨리 컷오프가 발생합니다:

- **최선의 경우**: 각 레벨에서 최선의 수를 먼저 탐색하면 O(b^(d/2))로 탐색 공간 절반으로 감소
- **실전 기법**: 킬러 무브(Killer Move), 히스토리 휴리스틱, MVV-LVA(Most Valuable Victim - Least Valuable Attacker)

### 트랜스포지션 테이블 (Transposition Table)

체스에서는 서로 다른 이동 순서로 같은 국면에 도달할 수 있습니다. 이를 **트랜스포지션(Transposition)**이라 하며, 이전 탐색 결과를 해시 테이블에 저장해 재계산을 방지합니다.

```python
from functools import lru_cache

# 단순화된 트랜스포지션 테이블 예제
transposition_table = {}

def alpha_beta_with_tt(game, depth, alpha, beta, is_maximizing):
    # 게임 상태를 해시키로 변환
    state_key = (tuple(game.board), is_maximizing, depth)
    
    # 캐시 조회
    if state_key in transposition_table:
        tt_entry = transposition_table[state_key]
        if tt_entry['flag'] == 'EXACT':
            return tt_entry['value']
        elif tt_entry['flag'] == 'LOWER':
            alpha = max(alpha, tt_entry['value'])
        elif tt_entry['flag'] == 'UPPER':
            beta = min(beta, tt_entry['value'])
        if alpha >= beta:
            return tt_entry['value']
    
    if depth == 0 or game.is_terminal():
        return evaluate(game)
    
    original_alpha = alpha
    best = -math.inf if is_maximizing else math.inf
    
    for move in game.get_moves():
        game.apply(move)
        value = alpha_beta_with_tt(game, depth-1, alpha, beta, not is_maximizing)
        game.undo(move)
        
        if is_maximizing:
            best = max(best, value)
            alpha = max(alpha, best)
        else:
            best = min(best, value)
            beta = min(beta, best)
        
        if beta <= alpha:
            break
    
    # 결과 저장
    flag = 'EXACT' if original_alpha < best < beta else \
           'LOWER' if best >= beta else 'UPPER'
    transposition_table[state_key] = {'value': best, 'flag': flag}
    return best
```

## Minimax를 넘어서: 현대의 게임 AI

### 몬테카를로 트리 탐색 (MCTS)

바둑처럼 분기 계수가 너무 크고(평균 250) 좋은 평가 함수를 만들기 어려운 게임에서는 **MCTS(Monte Carlo Tree Search)**가 사용됩니다. MCTS는 무작위 시뮬레이션을 통해 유망한 브랜치를 탐색하며, UCB1(Upper Confidence Bound) 공식으로 탐색과 활용의 균형을 맞춥니다.

AlphaGo는 MCTS와 딥러닝(정책 네트워크, 가치 네트워크)을 결합하여 인류 최강 바둑 기사를 꺾었습니다.

### 현대 체스 엔진의 구조

오늘날 최강 체스 엔진 Stockfish는 다음을 조합합니다:
- **알파-베타 + 반복 심화 + 트랜스포지션 테이블**
- **Late Move Reduction (LMR)**: 늦게 탐색하는 수는 더 얕게 탐색
- **Null Move Pruning**: 상대가 아무 수도 두지 않는 상황 시뮬레이션
- **NNUE (Efficiently Updatable Neural Network)**: 신경망 기반 평가 함수

## 참고 자료

- [Wikipedia: Minimax](https://en.wikipedia.org/wiki/Minimax)
- [Wikipedia: Alpha-beta pruning](https://en.wikipedia.org/wiki/Alpha%E2%80%93beta_pruning)
- [Wikipedia: Monte Carlo tree search](https://en.wikipedia.org/wiki/Monte_Carlo_tree_search)
- [Chessprogramming Wiki: Alpha-Beta](https://www.chessprogramming.org/Alpha-Beta)
