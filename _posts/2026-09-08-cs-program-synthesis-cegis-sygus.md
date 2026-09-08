---
layout: post
title: "프로그램 합성(Program Synthesis) 완전 정복: CEGIS와 SyGuS로 명세에서 코드를 자동 생성하는 원리"
date: 2026-09-08
categories: [cs, computer-science]
tags: [program-synthesis, CEGIS, SyGuS, formal-methods, SMT-solver, code-generation, automated-reasoning]
---

## 개념 설명

**프로그램 합성(Program Synthesis)**은 프로그래머가 "이렇게 동작해야 한다"는 **명세(specification)**를 제시하면, 시스템이 그 명세를 만족하는 **구체적인 프로그램 코드**를 자동으로 생성하는 기술이다. 수십 년간 인공지능과 프로그래밍 언어 이론의 교차점에서 연구되어 왔으며, 최근 LLM의 등장으로 다시 주목받고 있다.

### 핵심 삼각형: 명세 · 탐색 공간 · 합성 엔진

프로그램 합성은 세 가지 요소로 정의된다.

1. **명세(Specification)**: 프로그램이 만족해야 할 조건. 형태는 다양하다.
   - **입출력 예제(I/O examples)**: `sort([3,1,2]) → [1,2,3]`
   - **논리 공식**: `∀x. f(x) ≥ x`
   - **자연어**: "리스트를 오름차순 정렬하라" (LLM 기반 합성에서 주로 사용)
   - **참조 구현(Reference implementation)**: 정확하지만 느린 구현을 빠른 버전으로 합성

2. **탐색 공간(Search Space)**: 합성 가능한 프로그램의 집합. 문법(grammar)이나 스케치(sketch)로 제한한다.

3. **합성 엔진**: 탐색 공간에서 명세를 만족하는 프로그램을 찾는 알고리즘.

### 프로그램 합성의 역사

- **1957**: 첫 자동 프로그래밍 연구 (Galanter)
- **1969**: Hoare 논리의 등장 — 명세 기반 검증의 토대
- **2006**: **SKETCH** 시스템 — 부분 프로그램(스케치)을 완성하는 합성
- **2010**: **SyGuS** 경쟁 문제 형식화 (Alur et al.)
- **2015**: **PROSE** (Microsoft) — Excel 플래시 필 기반, 수억 명이 사용
- **2021~**: Codex, Copilot, Claude 등 LLM 기반 합성의 폭발적 성장

---

## 왜 필요한가

### 프로그래밍의 근본적 어려움

프로그래머는 "무엇을 원하는가(WHAT)"를 알지만, 컴퓨터에게는 "어떻게 해야 하는가(HOW)"를 정확히 전달해야 한다. 이 간극(semantic gap)이 버그와 개발 비용의 근원이다.

프로그램 합성이 빛나는 영역:

- **반복적인 데이터 변환**: Excel 수식 자동 생성 (Flash Fill), SQL 쿼리 합성
- **API 사용법 탐색**: 방대한 라이브러리에서 올바른 API 호출 순서 합성
- **최적화 코드 생성**: 고성능 행렬 연산 커널 자동 합성 (Halide, TVM)
- **테스트 자동화**: 입출력 예제로 오라클 자동 생성
- **보안 패치**: 취약점 명세로 수정 코드 합성

---

## 실제 구현 예제

### 예제 1: CEGIS — 반례 기반 귀납적 합성

**CEGIS(Counterexample-Guided Inductive Synthesis)**는 현재 가장 널리 쓰이는 합성 알고리즘이다. 합성기(Synthesizer)와 검증기(Verifier)가 상호작용하며 정답을 좁혀간다.

```
CEGIS 루프:
1. Synthesizer: 현재까지 알려진 반례들을 모두 만족하는 후보 프로그램 생성
2. Verifier: 후보 프로그램이 모든 입력에 대해 명세를 만족하는지 검증
   - 만족 → 합성 성공! 프로그램 반환
   - 불만족 → 새로운 반례(counterexample) 추출 → 1번으로 돌아감
```

```python
"""
CEGIS로 f(x) = x * x (제곱 함수) 합성하기
명세: 입출력 예제 기반
탐색 공간: ax + b 또는 ax^2 + bx + c 형태의 다항식
"""
from itertools import product

def synthesize_polynomial(examples, max_degree=2):
    """
    주어진 입출력 예제를 만족하는 최소 다항식 합성.
    CEGIS의 단순화 버전: 계수 탐색 + 검증.
    """
    # 탐색 공간: 계수 후보 [-3, -2, -1, 0, 1, 2, 3]
    coeff_range = range(-3, 4)
    
    counterexamples = list(examples[:1])  # 초기 반례 = 첫 번째 예제

    while True:
        # [Synthesizer] 현재 반례들을 만족하는 후보 프로그램 탐색
        candidate = None
        for coeffs in product(coeff_range, repeat=max_degree + 1):
            # coeffs = (a2, a1, a0) → f(x) = a2*x^2 + a1*x + a0
            def poly(x, c=coeffs):
                return sum(c[i] * x ** (max_degree - i) for i in range(max_degree + 1))
            
            # 모든 현재 반례를 통과하는지 확인
            if all(poly(x) == y for x, y in counterexamples):
                candidate = poly
                candidate_coeffs = coeffs
                break
        
        if candidate is None:
            return None  # 탐색 공간에서 해 없음

        # [Verifier] 모든 예제에 대해 검증
        all_correct = True
        new_counterexample = None
        for x, y in examples:
            if candidate(x) != y:
                all_correct = False
                new_counterexample = (x, y)
                break
        
        if all_correct:
            print(f"합성 성공! 계수: a2={candidate_coeffs[0]}, "
                  f"a1={candidate_coeffs[1]}, a0={candidate_coeffs[2]}")
            print(f"f(x) = {candidate_coeffs[0]}x² + {candidate_coeffs[1]}x + {candidate_coeffs[2]}")
            return candidate
        
        # 새 반례 추가 후 재시도
        counterexamples.append(new_counterexample)
        print(f"반례 추가: f({new_counterexample[0]}) = {new_counterexample[1]}, "
              f"후보는 {candidate(new_counterexample[0])} 반환")


# 테스트: f(x) = x^2 합성
examples = [(0, 0), (1, 1), (2, 4), (3, 9), (-1, 1), (-2, 4)]
result = synthesize_polynomial(examples)

if result:
    print(f"\n검증: f(5) = {result(5)}")   # 25
    print(f"검증: f(-3) = {result(-3)}")  # 9
```

### 예제 2: SyGuS — 문법 유도 합성

**SyGuS(Syntax-Guided Synthesis)**는 탐색 공간을 **형식 문법(formal grammar)**으로 제한하여 합성 효율을 높인다. SMT 솔버(Z3 등)를 검증 엔진으로 사용하는 것이 일반적이다.

```python
"""
SyGuS 스타일의 비트 연산 합성기
명세: f(x) = x를 2의 다음 거듭제곱으로 올림
탐색 공간 문법:
  Expr := x | Const | (Expr | Expr) | (Expr & Expr) | (Expr >> Const) | (Expr - Const)
"""

def next_power_of_2_reference(n):
    """참조 구현 (명세 역할)"""
    if n <= 0:
        return 1
    n -= 1
    n |= n >> 1
    n |= n >> 2
    n |= n >> 4
    n |= n >> 8
    n |= n >> 16
    return n + 1

# SyGuS 문법으로 생성 가능한 비트 조작 패턴들
def generate_candidates():
    """문법에서 파생 가능한 후보 프로그램들 (비트 해킹 패턴)"""
    candidates = []
    
    # 패턴 1: x - 1 → OR 연쇄 → + 1 (표준 방법)
    def pattern1(n):
        if n <= 1: return 1
        n -= 1
        n |= n >> 1; n |= n >> 2; n |= n >> 4; n |= n >> 8; n |= n >> 16
        return n + 1
    candidates.append(("표준 비트OR 연쇄", pattern1))
    
    # 패턴 2: 클리닝 후 1 시프트
    def pattern2(n):
        if n <= 0: return 1
        p = 1
        while p < n:
            p <<= 1
        return p
    candidates.append(("시프트 루프", pattern2))
    
    # 패턴 3: 비트 길이 기반
    def pattern3(n):
        if n <= 0: return 1
        return 1 << (n - 1).bit_length()
    candidates.append(("bit_length 활용", pattern3))
    
    return candidates

def sygus_synthesize(test_inputs):
    """
    CEGIS + 문법 탐색으로 명세를 만족하는 프로그램 합성.
    명세: next_power_of_2_reference와 동일한 출력
    """
    candidates = generate_candidates()
    
    for name, candidate in candidates:
        # 검증: 모든 테스트 입력에 대해 명세와 동일한가?
        valid = True
        for x in test_inputs:
            expected = next_power_of_2_reference(x)
            actual = candidate(x)
            if expected != actual:
                print(f"  [{name}] 실패: f({x})={actual}, 기댓값={expected}")
                valid = False
                break
        
        if valid:
            print(f"합성 성공: [{name}]")
            print(f"  f(0)={candidate(0)}, f(1)={candidate(1)}, "
                  f"f(5)={candidate(5)}, f(16)={candidate(16)}, f(17)={candidate(17)}")
            return candidate
    
    return None

# 합성 실행
test_inputs = list(range(0, 20)) + [100, 255, 256, 1000]
print("SyGuS 합성 시작:")
result = sygus_synthesize(test_inputs)
```

---

## 주의사항 및 팁

### 1. 탐색 공간 폭발 문제

프로그램 합성의 근본 어려움은 탐색 공간이 **무한대**라는 것이다. 길이 n의 프로그램 수는 n에 대해 지수적으로 증가한다. 이를 제어하는 방법:

- **문법 제한(SyGuS)**: 가능한 프로그램을 문법으로 제한
- **타입 제약**: 타입 시스템으로 무의미한 후보 제거
- **확률적 탐색**: MCMC, 유전 알고리즘으로 가능성 높은 프로그램 우선 탐색
- **신경망 유도(neural-guided)**: LLM으로 유망한 후보를 먼저 생성

### 2. 명세의 불완전성

입출력 예제만으로는 의도를 완전히 명세화하기 어렵다. `sort([1,1,2])`를 예제로 줘도 합성기가 `return [1,1,2]`(하드코딩)를 반환할 수 있다 — **과적합(overfitting)**이다. 해결책:

- 랜덤 테스트 케이스 자동 생성으로 과적합 검출
- **최단 프로그램 원칙(Occam's Razor)**: 동일 출력 중 가장 단순한 프로그램 선택
- **모달 명세**: "항상", "가끔", "절대 안 됨" 등의 확률적 명세

### 3. 실제 활용 사례

**Microsoft Excel Flash Fill**: 2013년 도입. 사용자가 몇 가지 입출력 예제를 제공하면 셀 변환 공식을 자동 합성. 내부적으로 문자열 변환 도메인 특화 언어(DSL)와 PROSE 합성기를 사용한다.

**SQL 합성**: 자연어로 쿼리를 설명하면 SQL을 자동 생성. 현대 AI 코딩 어시스턴트의 핵심 기능이다.

**하드웨어 합성(Hardware Synthesis)**: Chisel, CIRCT 등의 툴이 고수준 하드웨어 기술 언어를 최적화된 Verilog/VHDL로 합성한다.

### 4. LLM 기반 합성의 장단점

LLM(GPT, Claude 등)은 방대한 코드 데이터로 학습되어 강력한 합성 능력을 보이지만:

- **장점**: 자연어 명세 이해, 다양한 언어·도메인 지원, 빠른 초안 생성
- **단점**: 합성 결과의 **정확성 보장이 없음** (hallucination), 명세를 엄밀히 만족한다는 증명 불가

따라서 안전-크리티컬 시스템에서는 LLM 생성 코드를 SMT 솔버나 정형 검증기로 반드시 검증해야 한다.

## 참고 자료
- [Program Synthesis: Challenges and Opportunities (PMC)](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC5597726/)
- [Syntax-Guided Synthesis - SyGuS (Alur et al., 2013)](https://www.researchgate.net/publication/261037468_Syntax-guided_synthesis)
- [Guiding Enumerative Program Synthesis with Large Language Models (arXiv)](https://arxiv.org/abs/2403.03997)
- [SyGuS Competition](https://sygus.org)
