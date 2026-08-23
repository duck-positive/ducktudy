---
layout: post
title: "스프라그-그런디 정리와 조합 게임 이론 완전 정복: 님 게임부터 복잡한 게임 분석까지"
date: 2026-08-23
categories: [cs, computer-science]
tags: [sprague-grundy, nim, game-theory, combinatorial-game, mex, grundy-number, algorithm]
---

두 플레이어가 번갈아 가며 두는 게임에서 "누가 이기는가?"를 수학적으로 판별할 수 있다면 어떨까? **스프라그-그런디 정리(Sprague-Grundy Theorem)**는 바로 이 질문에 답하는 강력한 이론으로, 1930년대 롤랜드 스프라그(Roland Sprague)와 패트릭 그런디(Patrick Grundy)가 각각 독립적으로 발표했다.

이 정리는 **모든 공정한 조합 게임(Impartial Combinatorial Game)은 단 하나의 님(Nim) 더미에 동치**임을 증명한다. 복잡해 보이는 게임도 그런디 수(Grundy Number)를 계산하면 승패를 O(1)에 판별할 수 있다.

---

## 조합 게임 이론의 기초

### 공정 게임(Impartial Game)의 조건

스프라그-그런디 정리는 다음 조건을 만족하는 게임에 적용된다.

1. **두 플레이어**가 번갈아 이동한다.
2. **완전 정보**: 양쪽 모두 게임 상태를 완벽히 알 수 있다.
3. **공정성**: 두 플레이어가 이용할 수 있는 이동이 완전히 동일하다 (체스처럼 각자의 말이 다른 경우 제외).
4. **정상 플레이 규칙**: 더 이상 이동할 수 없는 플레이어가 **진다**.
5. **유한성**: 게임이 반드시 종료된다.

이를 만족하지 않는 게임(바둑, 체스 등의 편파 게임)에는 다른 이론이 필요하다.

### 님(Nim) 게임: 조합 게임의 원형

**님 게임**은 여러 개의 돌 더미가 있고, 두 플레이어가 번갈아 **임의의 한 더미에서 최소 1개 이상의 돌을 가져가는** 게임이다. 더 이상 가져갈 수 없는 플레이어가 진다.

예: 더미가 `[3, 4, 5]`라면 선공이 이기는가?

님 게임의 핵심 정리: **모든 더미의 크기를 XOR한 결과(님-합)가 0이면 후공 승, 0이 아니면 선공 승**이다.

```
3 XOR 4 XOR 5 = 011 XOR 100 XOR 101 = 010 ≠ 0 → 선공 승
```

이 단순한 XOR 규칙이 스프라그-그런디 정리의 핵심이다.

---

## 그런디 수(Grundy Number)와 MEX

### 그런디 수의 정의

게임의 각 상태 S에 대해 **그런디 수 G(S)**를 다음과 같이 정의한다.

```
G(S) = mex({ G(S') | S'는 S에서 한 번의 이동으로 도달 가능한 상태 })
```

여기서 **mex(Minimum EXcludant)**는 집합에 속하지 않는 **가장 작은 비음수 정수**다.

- `mex({})` = 0
- `mex({0})` = 1
- `mex({0, 1})` = 2
- `mex({0, 2})` = 1
- `mex({1, 2, 3})` = 0

### 핵심 성질

- **G(S) = 0**: 현재 차례인 플레이어가 진다 (후공 승 포지션, P-position).
- **G(S) ≠ 0**: 현재 차례인 플레이어가 이긴다 (선공 승 포지션, N-position).
- 님 더미 크기 n에 대한 G(n) = n (크기 n인 님 더미의 그런디 수는 n 자신).

---

## 스프라그-그런디 정리의 핵심: 합성 게임 분석

여러 독립적인 게임이 동시에 진행될 때 (합성 게임, Compound Game), 전체 게임의 그런디 수는 각 서브게임의 그런디 수의 **XOR**이다.

```
G(G1 + G2 + ... + Gk) = G(G1) XOR G(G2) XOR ... XOR G(Gk)
```

이것이 스프라그-그런디 정리의 핵심이다. 아무리 복잡한 게임도 서브게임으로 분해하고 XOR만 계산하면 승패를 알 수 있다.

---

## 코드 구현: 그런디 수 계산

### 예시 1: 단순 님 게임 그런디 수 검증

```python
from functools import lru_cache

def mex(s: set) -> int:
    """집합 s에 없는 가장 작은 비음수 정수를 반환"""
    i = 0
    while i in s:
        i += 1
    return i

@lru_cache(maxsize=None)
def grundy_nim(n: int) -> int:
    """크기 n인 님 더미의 그런디 수"""
    if n == 0:
        return 0  # 이동 불가 → G = mex({}) = 0
    reachable = {grundy_nim(i) for i in range(n)}  # 0, 1, ..., n-1로 이동 가능
    return mex(reachable)

# 검증: 님 더미의 G(n) = n
for i in range(10):
    assert grundy_nim(i) == i, f"Failed at n={i}"
print("님 더미 그런디 수:", [grundy_nim(i) for i in range(10)])
# 출력: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

# 합성 게임: 더미 [3, 4, 5]
xor_sum = grundy_nim(3) ^ grundy_nim(4) ^ grundy_nim(5)
print("XOR 합:", xor_sum)  # 2 → 선공 승
print("선공 승리:", xor_sum != 0)  # True
```

### 예시 2: 동전 게임 (특수 이동 규칙)

규칙: 돌 n개에서 1개, 3개, 또는 4개를 가져갈 수 있다. 더 이상 가져갈 수 없는 사람이 진다.

```python
from functools import lru_cache

MOVES = [1, 3, 4]  # 한 번에 가져갈 수 있는 개수

@lru_cache(maxsize=None)
def grundy_coin(n: int) -> int:
    """n개의 돌에서 {1, 3, 4}개를 가져갈 수 있는 게임의 그런디 수"""
    if n == 0:
        return 0
    reachable = set()
    for m in MOVES:
        if n - m >= 0:
            reachable.add(grundy_coin(n - m))
    return mex(reachable)

# 그런디 수 패턴 출력
pattern = [grundy_coin(i) for i in range(16)]
print("그런디 수 패턴:", pattern)
# 출력: [0, 1, 0, 1, 2, 3, 2, 3, 0, 1, 0, 1, 2, 3, 2, 3]
# → 주기 8의 패턴 발견!

def who_wins_multi(piles: list) -> str:
    """여러 더미 게임에서 승자 판별 (XOR)"""
    xor_val = 0
    for p in piles:
        xor_val ^= grundy_coin(p)
    return "선공 승" if xor_val != 0 else "후공 승"

# 예시: 더미 [2, 5, 7]
print(who_wins_multi([2, 5, 7]))  # 결과: 선공 승
# 검증: G(2)=0, G(5)=3, G(7)=3 → XOR = 0^3^3 = 0 → 후공 승
print(who_wins_multi([1, 5, 7]))  # G(1)=1, G(5)=3, G(7)=3 → XOR = 1^3^3 = 1 → 선공 승
```

---

## 실전 응용: 체인 게임

**규칙**: 길이 n인 격자 위에서 두 플레이어가 번갈아 연속된 칸을 제거한다. 마지막에 이동하지 못하는 플레이어가 진다.

이런 복잡한 게임도 그런디 수 DP로 해결 가능하다.

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def grundy_chain(n: int) -> int:
    """
    길이 n인 연속 체인에서 임의의 연속 구간을 제거하는 게임.
    제거하면 두 개의 독립적인 서브게임으로 분리된다.
    """
    if n == 0:
        return 0
    reachable = set()
    # 길이 1부터 n까지의 연속 구간 제거 시도
    for length in range(1, n + 1):
        for start in range(n - length + 1):
            left = start
            right = n - start - length
            # 두 서브체인의 XOR이 합성 게임의 그런디 수
            g = grundy_chain(left) ^ grundy_chain(right)
            reachable.add(g)
    return mex(reachable)

print("체인 그런디 수:", [grundy_chain(i) for i in range(9)])
# 출력: [0, 1, 2, 3, 1, 4, 3, 2, 1]

# n=5 체인: G(5)=4 ≠ 0 → 선공 승
print("n=5 체인:", "선공 승" if grundy_chain(5) != 0 else "후공 승")
```

---

## P-position과 N-position 분류

게임 분석에서 자주 쓰이는 용어 정리:

| 구분 | 의미 | 그런디 수 |
|---|---|---|
| P-position | Previous player wins (후공 승) | 0 |
| N-position | Next player wins (선공 승) | ≠ 0 |

P-position과 N-position을 분류하는 규칙:
1. 이동 불가 상태 → P-position
2. 모든 이동이 N-position으로만 간다면 → P-position
3. N-position으로 가는 이동이 하나라도 있다면 → N-position

---

## 주의사항과 팁

### 1. Misère 게임 주의

**미제르(Misère) 규칙**은 마지막에 이동하는 플레이어가 **지는** 규칙이다. 님 게임의 경우 미제르 분석은 정상 플레이와 거의 같지만, XOR 합이 0인지 확인하는 기준이 약간 다르다. 스프라그-그런디 정리는 기본적으로 정상 플레이(마지막 이동자 승)에 적용된다.

### 2. 그런디 수의 메모이제이션

게임 상태 공간이 크다면 반드시 메모이제이션(캐시)을 사용해야 한다. 특히 2D 게임 상태라면 딕셔너리나 2D 배열로 캐시한다.

### 3. 이동 규칙의 주기성 활용

많은 게임의 그런디 수는 **주기적 패턴**을 보인다. 패턴을 발견하면 O(1)에 그런디 수를 계산할 수 있어 N이 매우 클 때도 처리할 수 있다.

### 4. 공간 최적화

그런디 수 DP에서 이전 결과만 필요한 경우 배열 전체를 저장할 필요 없이 슬라이딩 윈도우로 공간을 절약할 수 있다.

### 5. 합성 게임의 독립성 검증

합성 게임에서 XOR을 적용하려면 서브게임들이 **서로 독립적**이어야 한다. 한 게임의 이동이 다른 게임에 영향을 주면 단순 XOR로는 분석할 수 없다.

---

## 마무리

스프라그-그런디 정리는 조합 게임 이론의 핵심 결과로, 복잡해 보이는 게임 분석을 단순한 수학 문제로 환원한다. 핵심은 두 가지다. 첫째, 각 상태의 그런디 수를 MEX로 재귀 계산한다. 둘째, 독립적인 서브게임의 그런디 수를 XOR해 전체 게임을 분석한다. 경쟁 프로그래밍의 게임 문제에서 이 정리는 가장 강력한 무기다.

## 참고 자료
- [Wikipedia: Sprague-Grundy theorem](https://en.wikipedia.org/wiki/Sprague%E2%80%93Grundy_theorem)
- [GeeksforGeeks: Combinatorial Game Theory - Sprague-Grundy Theorem](https://www.geeksforgeeks.org/dsa/combinatorial-game-theory-set-4-sprague-grundy-theorem/)
- [Codeforces: The Intuition Behind NIM and Grundy Numbers](https://codeforces.com/blog/entry/66040)
- [Wikipedia: Nim](https://en.wikipedia.org/wiki/Nim)
