---
layout: post
title: "L-시스템(Lindenmayer System) 완전 정복: 단순한 규칙에서 무한한 자연을 생성하는 형식 문법"
date: 2026-11-05
categories: [cs, computer-science]
tags: [l-system, lindenmayer, formal-grammar, procedural-generation, fractal, turtle-graphics, string-rewriting]
---

나뭇잎의 잎맥 패턴, 고사리의 프랙탈 구조, 눈송이의 반복적인 대칭성. 이 복잡한 자연의 형태들은 놀랍도록 단순한 규칙에서 출발한다. **L-시스템(Lindenmayer System)**은 1968년 헝가리 생물학자 Aristid Lindenmayer가 식물 세포의 성장 과정을 수학적으로 모델링하기 위해 고안한 병렬 문자열 재작성 형식 문법이다. CS의 관점에서 L-시스템은 형식 언어 이론, 절차적 생성(Procedural Generation), 컴퓨터 그래픽스가 만나는 아름다운 교차점이다.

## L-시스템이란 무엇인가

L-시스템은 세 가지 구성 요소로 정의된다.

```
G = (V, ω, P)
```

- **V (알파벳)**: 기호의 유한 집합. 변수(재작성 가능)와 상수(재작성 불가)로 나뉜다.
- **ω (공리, Axiom)**: 초기 문자열
- **P (생성 규칙, Production Rules)**: V → V*의 매핑

매 세대(세대 = 한 번의 적용)마다 현재 문자열의 **모든 기호에 동시에** 규칙을 적용한다. 이것이 한 번에 하나의 규칙만 적용하는 일반 형식 문법과의 근본적인 차이다.

### Lindenmayer의 원래 조류 모델 (Algae)

Lindenmayer가 처음 제시한 단순한 예시:

```
알파벳: {A, B}
공리: A
규칙: A → AB, B → A
```

| 세대 | 문자열 | 길이 |
|------|--------|------|
| 0 | A | 1 |
| 1 | AB | 2 |
| 2 | ABA | 3 |
| 3 | ABAAB | 5 |
| 4 | ABAABABA | 8 |
| 5 | ABAABABAABAAB | 13 |

각 세대의 문자열 길이가 피보나치 수열(1, 1, 2, 3, 5, 8, 13, ...)을 따른다는 점이 흥미롭다. 이는 A와 B의 개수가 각각 피보나치 수열을 이루기 때문이다.

### 거북이 그래픽스(Turtle Graphics)로 시각화

L-시스템의 문자열을 거북이 명령으로 해석하면 복잡한 도형을 그릴 수 있다.

| 기호 | 의미 |
|------|------|
| F | 현재 방향으로 step만큼 전진하며 선 그리기 |
| f | 선 없이 전진 |
| + | 왼쪽으로 angle만큼 회전 |
| - | 오른쪽으로 angle만큼 회전 |
| [ | 현재 위치와 방향을 스택에 저장 |
| ] | 스택에서 위치와 방향 복원 (가지 분기의 핵심) |

## 왜 L-시스템을 알아야 하는가

### 절차적 생성의 핵심 도구

게임 개발에서 "무한히 다양한" 지형, 식물, 던전을 생성하려면 L-시스템이 강력한 도구다. No Man's Sky, Minecraft의 식물 생성 등에 유사한 방법이 사용된다. 단 몇 줄의 규칙으로 수천 가지 고유한 나무 모양을 생성할 수 있다.

### 형식 언어 이론과의 연결

L-시스템은 Chomsky 위계(Chomsky Hierarchy)에서 흥미로운 위치를 차지한다. 문맥 자유 L-시스템(D0L-System)은 문맥 자유 문법과 유사하지만, 병렬 재작성 덕분에 Chomsky의 일반 문법으로 표현하기 어려운 패턴을 생성한다. 충분히 복잡한 L-시스템은 튜링 완전성도 달성할 수 있다.

### 생물정보학(Bioinformatics) 응용

단백질 구조, DNA 서열의 반복 패턴 분석에도 L-시스템 아이디어가 활용된다. Lindenmayer 자신도 원래 세포 성장과 분열을 모델링하려 했다.

## 실제 구현 예제

### 예제 1: Python L-시스템 엔진과 거북이 렌더러

```python
import math
from dataclasses import dataclass, field
from typing import Dict, Tuple, List

@dataclass
class TurtleState:
    x: float
    y: float
    angle: float  # 도(degree) 단위

class LSystem:
    """결정론적 L-시스템 엔진"""

    def __init__(self, axiom: str, rules: Dict[str, str]):
        self.axiom = axiom
        self.rules = rules

    def generate(self, iterations: int) -> str:
        current = self.axiom
        for _ in range(iterations):
            # 모든 기호에 동시 규칙 적용 (병렬 재작성)
            current = ''.join(self.rules.get(c, c) for c in current)
        return current

    def to_lines(
        self,
        sequence: str,
        angle_deg: float,
        step: float = 5.0,
        start: Tuple[float, float] = (0.0, 0.0),
        start_angle: float = 90.0,
    ) -> List[Tuple[Tuple[float, float], Tuple[float, float]]]:
        """
        L-시스템 문자열을 2D 선분 목록으로 변환한다.
        각 선분은 ((x1, y1), (x2, y2)) 튜플이다.
        """
        state = TurtleState(start[0], start[1], start_angle)
        stack: List[TurtleState] = []
        lines: List[Tuple[Tuple[float, float], Tuple[float, float]]] = []

        for cmd in sequence:
            if cmd in ('F', 'G'):
                # 전진하며 선 그리기
                rad = math.radians(state.angle)
                nx = state.x + step * math.cos(rad)
                ny = state.y + step * math.sin(rad)
                lines.append(((state.x, state.y), (nx, ny)))
                state.x, state.y = nx, ny
            elif cmd == 'f':
                # 선 없이 전진
                rad = math.radians(state.angle)
                state.x += step * math.cos(rad)
                state.y += step * math.sin(rad)
            elif cmd == '+':
                state.angle += angle_deg
            elif cmd == '-':
                state.angle -= angle_deg
            elif cmd == '[':
                # 가지 분기 시작: 현재 상태 저장
                stack.append(TurtleState(state.x, state.y, state.angle))
            elif cmd == ']':
                # 가지 분기 종료: 저장된 상태 복원
                state = stack.pop()

        return lines


# ───────────────────────────────────────────────
#  유명한 L-시스템 예제들
# ───────────────────────────────────────────────

# 1. 코흐 눈송이 (Koch Snowflake)
#    각도: 60°
#    F → F+F--F+F
koch = LSystem(
    axiom='F--F--F',
    rules={'F': 'F+F--F+F'}
)
seq_3 = koch.generate(3)
lines = koch.to_lines(seq_3, angle_deg=60, step=10)
print(f"코흐 눈송이 (3회): {len(lines)}개 선분")

# 2. 드래곤 커브 (Dragon Curve)
#    각도: 90°
#    X → X+YF+, Y → -FX-Y
dragon = LSystem(
    axiom='FX',
    rules={
        'X': 'X+YF+',
        'Y': '-FX-Y'
    }
)
seq_12 = dragon.generate(12)
lines = dragon.to_lines(seq_12, angle_deg=90, step=5)
print(f"드래곤 커브 (12회): {len(lines)}개 선분")

# 3. 시에르핀스키 삼각형 (Sierpinski Triangle)
#    각도: 60°
#    F → F-G+F+G-F, G → GG
sierpinski = LSystem(
    axiom='F-G-G',
    rules={'F': 'F-G+F+G-F', 'G': 'GG'}
)
seq_6 = sierpinski.generate(6)
lines = sierpinski.to_lines(seq_6, angle_deg=120, step=5)
print(f"시에르핀스키 삼각형 (6회): {len(lines)}개 선분")

# 4. 프랙탈 식물 (Fractal Plant)
#    각도: 25°
#    X → F+[[X]-X]-F[-FX]+X, F → FF
plant = LSystem(
    axiom='X',
    rules={
        'X': 'F+[[X]-X]-F[-FX]+X',
        'F': 'FF'
    }
)
for i in range(1, 7):
    seq = plant.generate(i)
    branch_count = seq.count('[')
    print(f"프랙탈 식물 세대 {i}: 길이={len(seq):>8}, 가지={branch_count:>5}개")
```

### 예제 2: 확률적 L-시스템(Stochastic L-System)

자연의 불규칙성을 모델링하려면 같은 기호에 여러 규칙을 각기 다른 확률로 적용하는 확률적 L-시스템을 사용한다.

```python
import random
from typing import List, Tuple

class StochasticLSystem:
    """
    확률적 L-시스템.
    rules: {symbol: [(probability, replacement), ...]}
    확률의 합은 1.0이어야 한다.
    """

    def __init__(self, axiom: str, rules: Dict[str, List[Tuple[float, str]]], seed: int = 42):
        self.axiom = axiom
        self.rules = rules
        random.seed(seed)
        # 각 심볼의 확률 합이 1인지 검증
        for sym, candidates in rules.items():
            total = sum(p for p, _ in candidates)
            assert abs(total - 1.0) < 1e-9, f"기호 '{sym}'의 확률 합이 {total}로 1이 아닙니다."

    def _apply(self, symbol: str) -> str:
        if symbol not in self.rules:
            return symbol
        r = random.random()
        cumulative = 0.0
        for prob, replacement in self.rules[symbol]:
            cumulative += prob
            if r <= cumulative:
                return replacement
        return self.rules[symbol][-1][1]  # 부동소수점 오차 대비

    def generate(self, iterations: int) -> str:
        current = self.axiom
        for _ in range(iterations):
            current = ''.join(self._apply(c) for c in current)
        return current


# 세 가지 다른 가지 패턴을 무작위로 적용하는 나무
stochastic_tree = StochasticLSystem(
    axiom='F',
    rules={
        'F': [
            (0.33, 'F[+F]F[-F]F'),    # 양쪽 대칭 가지
            (0.33, 'F[+F]F'),          # 오른쪽 가지만
            (0.34, 'F[-F[+F]]F'),      # 중첩 가지
        ]
    }
)

print("=== 확률적 L-시스템: 같은 규칙으로 다른 나무 생성 ===")
for seed in range(3):
    stochastic_tree = StochasticLSystem(
        axiom='F',
        rules={
            'F': [
                (0.33, 'F[+F]F[-F]F'),
                (0.33, 'F[+F]F'),
                (0.34, 'F[-F[+F]]F'),
            ]
        },
        seed=seed
    )
    result = stochastic_tree.generate(5)
    branches = result.count('[')
    print(f"  seed={seed}: 길이={len(result):>6}, 가지={branches:>4}개 (형태가 매번 달라짐)")


# ─────────────────────────────────────────────────
#  매개변수 L-시스템 (Parametric L-System) 간단 예시
# ─────────────────────────────────────────────────
# 실제 가지 길이와 각도를 수치 파라미터로 제어한다.
# 기호: F(l): 길이 l로 전진, +(a): a도 회전

@dataclass
class ParametricSymbol:
    symbol: str
    params: Tuple[float, ...] = ()

def parametric_step(symbols: List[ParametricSymbol]) -> List[ParametricSymbol]:
    """간단한 매개변수 L-시스템 1 세대 적용 예시"""
    result = []
    for sym in symbols:
        if sym.symbol == 'F':
            # F(l) → F(l*0.7)[+(30)F(l*0.6)]F(l*0.7)[-(30)F(l*0.6)]F(l*0.7)
            l = sym.params[0] if sym.params else 10.0
            result.extend([
                ParametricSymbol('F', (l * 0.7,)),
                ParametricSymbol('['),
                ParametricSymbol('+', (30.0,)),
                ParametricSymbol('F', (l * 0.6,)),
                ParametricSymbol(']'),
                ParametricSymbol('F', (l * 0.7,)),
                ParametricSymbol('['),
                ParametricSymbol('-', (30.0,)),
                ParametricSymbol('F', (l * 0.6,)),
                ParametricSymbol(']'),
                ParametricSymbol('F', (l * 0.7,)),
            ])
        elif sym.symbol in ('[', ']'):
            result.append(sym)
        else:
            result.append(sym)
    return result

axiom = [ParametricSymbol('F', (100.0,))]
state = axiom
for gen in range(4):
    state = parametric_step(state)
    total_len = sum(s.params[0] for s in state if s.symbol == 'F')
    segments = sum(1 for s in state if s.symbol == 'F')
    print(f"매개변수 L-시스템 세대 {gen+1}: "
          f"{segments}개 선분, 총 길이={total_len:.1f}")
```

## 주의사항과 실전 팁

### 문자열 폭발(String Explosion) 주의

L-시스템은 각 세대마다 문자열이 지수적으로 늘어난다. `F → F[+F]F[-F]F` 규칙의 경우 5세대에서 약 3^5 = 243배, 7세대에서 2187배가 된다. 메모리 한계와 렌더링 시간을 고려해 세대 수를 제한하거나, 문자열 대신 직접 그래픽스 명령을 스트리밍 방식으로 처리해야 한다.

### 맥락 감응 L-시스템(Context-Sensitive)

이웃 기호를 고려하는 규칙으로 더 사실적인 성장 모델링이 가능하다. `<A>B → AB` 형태로 A 왼쪽에 특정 기호가 있을 때만 규칙이 적용된다. 광합성 신호 전달이나 영양소 흐름 모델링에 활용된다.

### 각도 선택의 중요성

황금 각도(137.5°)는 자연에서 잎차례(phyllotaxis)를 모델링할 때 사용된다. 60°는 눈송이와 삼각형 프랙탈에, 90°는 직각 패턴에, 25°는 나뭇가지에 자주 쓰인다.

### 3D L-시스템

6개의 방향 명령(앞/뒤/위/아래/좌/우 회전)을 추가하면 3D 식물 모델링이 가능하다. `^`, `&`, `\`, `/` 등의 기호로 3D 회전을 표현하며, 실제 식물학 소프트웨어(Vlab, L-studio)가 이를 사용한다.

## 참고 자료
- [L-system - Wikipedia](https://en.wikipedia.org/wiki/L-system)
- [The Algorithmic Beauty of Plants (Prusinkiewicz & Lindenmayer, 1990) - Chapter 1](https://algorithmicbotany.org/papers/abop/abop-ch1.pdf)
- [Lindenmayer Systems, Fractals, and Plants (PDF)](https://algorithmicbotany.org/papers/lsfp.pdf)
- [UMD CMSC425: L-Systems Lecture Notes](https://www.cs.umd.edu/class/fall2018/cmsc425/Lects/lect13-L-systems.pdf)
