---
layout: post
title: "오토마타 이론과 촘스키 계층 구조 완전 정복: 컴파일러와 파서의 수학적 토대"
date: 2026-09-18
categories: [cs, computer-science]
tags: [automata, chomsky-hierarchy, formal-language, finite-automata, pushdown-automata, turing-machine, context-free-grammar, regular-expression, compiler-theory]
---

오토마타 이론(Automata Theory)은 컴퓨터 과학의 수학적 토대 중 하나로, **계산 가능성(Computability)**과 **형식 언어(Formal Language)**를 연구하는 분야입니다. 정규표현식 엔진부터 프로그래밍 언어 파서, 자연어 처리까지—이 모든 기술은 오토마타 이론이라는 하나의 수학적 체계 위에 서 있습니다. 1956년 노암 촘스키(Noam Chomsky)가 제시한 **촘스키 계층 구조(Chomsky Hierarchy)**는 언어의 표현력에 따라 언어를 4가지 계층으로 분류하는 체계입니다.

## 왜 오토마타 이론이 필요한가?

컴파일러가 소스 코드를 분석할 때, 가장 먼저 하는 작업은 어휘 분석(Lexical Analysis)입니다. `if`, `while`, `int`와 같은 토큰을 인식하는 것은 **정규 언어(Regular Language)**의 영역입니다. 그 다음 구문 분석(Parsing)에서 `if (cond) { ... } else { ... }`와 같은 중첩 구조를 파악하는 것은 **문맥 자유 언어(Context-Free Language)**의 영역입니다.

만약 이 수학적 기반이 없다면, 언어 처리 시스템은 임시방편적인 코드의 집합에 불과하게 됩니다. 오토마타 이론은 다음을 가능하게 합니다:

- **"이 언어는 정규표현식으로 표현할 수 있는가?"**를 수학적으로 증명
- **"이 문법은 파싱 가능한가?"**를 결정론적으로 판별
- **"이 알고리즘은 종료하는가?"**를 이론적으로 분석

## 촘스키 계층 구조 (Chomsky Hierarchy)

촘스키 계층은 Type 0 ~ Type 3으로 구성되며, 번호가 클수록 표현력이 낮고 처리가 단순해집니다.

| 타입 | 언어 유형 | 문법 | 인식 기계 |
|------|-----------|------|-----------|
| Type 0 | 재귀 열거 가능 언어 | 무제한 문법 | 튜링 기계 |
| Type 1 | 문맥 의존 언어 | 문맥 의존 문법 | 선형 한계 오토마타(LBA) |
| Type 2 | 문맥 자유 언어 | 문맥 자유 문법(CFG) | 푸시다운 오토마타(PDA) |
| Type 3 | 정규 언어 | 정규 문법 | 유한 오토마타(FA) |

이 계층은 포함 관계를 형성합니다: **정규 ⊂ 문맥자유 ⊂ 문맥의존 ⊂ 재귀열거가능**.

---

## Type 3: 정규 언어와 유한 오토마타 (Finite Automata)

정규 언어는 **결정론적 유한 오토마타(DFA)** 또는 **비결정론적 유한 오토마타(NFA)**로 인식됩니다. DFA는 `(Q, Σ, δ, q₀, F)` 5-튜플로 정의됩니다:

- `Q`: 상태의 유한 집합
- `Σ`: 입력 알파벳
- `δ: Q × Σ → Q`: 전이 함수
- `q₀ ∈ Q`: 초기 상태
- `F ⊆ Q`: 수락 상태 집합

### 코드 예제 1: Python으로 구현하는 DFA

아래는 이진 문자열에서 짝수 개의 0을 인식하는 DFA입니다.

```python
class DFA:
    def __init__(self, states, alphabet, transitions, start, accepting):
        self.states = states
        self.alphabet = alphabet
        self.transitions = transitions  # {(state, char): next_state}
        self.start = start
        self.accepting = accepting

    def accepts(self, string):
        current = self.start
        for char in string:
            if char not in self.alphabet:
                return False
            current = self.transitions.get((current, char))
            if current is None:
                return False
        return current in self.accepting


# 짝수 개의 '0'을 포함하는 이진 문자열을 인식하는 DFA
# 상태: q0 (0의 개수가 짝수), q1 (0의 개수가 홀수)
even_zeros_dfa = DFA(
    states={'q0', 'q1'},
    alphabet={'0', '1'},
    transitions={
        ('q0', '0'): 'q1',
        ('q0', '1'): 'q0',
        ('q1', '0'): 'q0',
        ('q1', '1'): 'q1',
    },
    start='q0',
    accepting={'q0'}  # q0이 수락 상태 (짝수 개의 0)
)

test_cases = [
    ("", True),        # 0개의 '0' → 짝수
    ("00", True),      # 2개의 '0' → 짝수
    ("0101", True),    # 2개의 '0' → 짝수
    ("0", False),      # 1개의 '0' → 홀수
    ("100", False),    # 1개의 '0' → 홀수
]

for string, expected in test_cases:
    result = even_zeros_dfa.accepts(string)
    status = "✓" if result == expected else "✗"
    print(f"{status} '{string}': accepts={result}")
```

**NFA와 DFA의 관계**: 어떤 NFA도 동등한 DFA로 변환 가능합니다(서브셋 구성법, Subset Construction). 이를 통해 NFA의 설계 편의성과 DFA의 효율적인 실행을 모두 얻을 수 있습니다.

**펌프 보조 정리(Pumping Lemma for Regular Languages)**: 어떤 언어가 정규 언어가 **아님**을 증명하는 데 사용됩니다. `{0ⁿ1ⁿ | n ≥ 0}`과 같이 쌍이 맞는 구조는 정규 언어로 표현 불가능합니다.

---

## Type 2: 문맥 자유 언어와 푸시다운 오토마타 (PDA)

**문맥 자유 문법(Context-Free Grammar, CFG)**은 `(V, Σ, R, S)` 4-튜플로 정의됩니다:

- `V`: 비단말 기호(변수)의 유한 집합
- `Σ`: 단말 기호(터미널)의 유한 집합
- `R`: 생성 규칙 집합 `(A → α, A ∈ V, α ∈ (V ∪ Σ)*)`
- `S ∈ V`: 시작 기호

예를 들어, 올바른 괄호 쌍을 생성하는 CFG:

```
S → ε | SS | (S)
```

이 문법에서 `(())`, `()()`, `((()))`는 모두 유효한 문자열입니다. 이는 정규 언어로 표현 불가능하지만 CFG로는 간단히 표현됩니다.

**푸시다운 오토마타(PDA)**는 스택을 추가한 유한 오토마타로, 모든 문맥 자유 언어를 인식합니다.

### 코드 예제 2: Python으로 구현하는 재귀 하강 파서 (CFG 기반)

아래는 간단한 산술 표현식을 파싱하는 재귀 하강 파서입니다. 이 파서는 아래 CFG를 구현합니다:

```
expr   → term (('+' | '-') term)*
term   → factor (('*' | '/') factor)*
factor → NUMBER | '(' expr ')'
```

```python
class Parser:
    """
    CFG 기반 재귀 하강 파서
    expr → term (('+' | '-') term)*
    term → factor (('*' | '/') factor)*
    factor → NUMBER | '(' expr ')'
    """

    def __init__(self, tokens):
        self.tokens = tokens
        self.pos = 0

    def peek(self):
        return self.tokens[self.pos] if self.pos < len(self.tokens) else None

    def consume(self, expected=None):
        token = self.tokens[self.pos]
        if expected and token != expected:
            raise SyntaxError(f"Expected '{expected}', got '{token}'")
        self.pos += 1
        return token

    def parse_expr(self):
        left = self.parse_term()
        while self.peek() in ('+', '-'):
            op = self.consume()
            right = self.parse_term()
            left = (op, left, right)
        return left

    def parse_term(self):
        left = self.parse_factor()
        while self.peek() in ('*', '/'):
            op = self.consume()
            right = self.parse_factor()
            left = (op, left, right)
        return left

    def parse_factor(self):
        token = self.peek()
        if token == '(':
            self.consume('(')
            node = self.parse_expr()
            self.consume(')')
            return node
        elif token and token.isdigit():
            return int(self.consume())
        else:
            raise SyntaxError(f"Unexpected token: {token}")


def evaluate(ast):
    if isinstance(ast, int):
        return ast
    op, left, right = ast
    l, r = evaluate(left), evaluate(right)
    return {'+': l + r, '-': l - r, '*': l * r, '/': l // r}[op]


# 토크나이저
def tokenize(expr):
    tokens = []
    i = 0
    while i < len(expr):
        if expr[i].isspace():
            i += 1
        elif expr[i].isdigit():
            j = i
            while j < len(expr) and expr[j].isdigit():
                j += 1
            tokens.append(expr[i:j])
            i = j
        else:
            tokens.append(expr[i])
            i += 1
    return tokens

expressions = ["3 + 4 * 2", "(3 + 4) * 2", "10 - 2 * 3 + 1"]
for expr in expressions:
    tokens = tokenize(expr)
    parser = Parser(tokens)
    ast = parser.parse_expr()
    result = evaluate(ast)
    print(f"'{expr}' = {result}, AST: {ast}")
```

---

## Type 1과 Type 0: 문맥 의존 언어와 튜링 기계

**문맥 의존 언어(Context-Sensitive Language)**는 `{aⁿbⁿcⁿ | n ≥ 1}`과 같이 CFG로 표현 불가능한 언어들을 포함합니다. 이를 인식하는 **선형 한계 오토마타(Linear Bounded Automata)**는 입력 크기에 비례하는 메모리만 사용하는 튜링 기계입니다.

**튜링 기계(Turing Machine)**는 Type 0 언어(재귀 열거 가능 언어)를 인식하며, 이론적으로 가장 강력한 계산 모델입니다. 처치-튜링 테제(Church-Turing Thesis)에 따르면, 알고리즘으로 계산 가능한 모든 함수는 튜링 기계로 계산 가능합니다.

하지만 **정지 문제(Halting Problem)** — "임의의 프로그램이 주어진 입력에 대해 종료하는가?" — 는 어떤 알고리즘으로도 결정 불가능합니다. 이는 컴퓨터 과학의 가장 근본적인 한계 중 하나입니다.

---

## 실용적 응용과 주의사항

### 정규 표현식 최적화
대부분의 정규 표현식 엔진은 NFAd에서 DFA로 변환하거나(Thompson Construction), 직접 NFA를 시뮬레이션합니다. 복잡한 정규표현식에서 역추적(Backtracking)이 발생하면 **ReDoS(Regular Expression Denial of Service)** 취약점이 생길 수 있습니다. 예를 들어 `(a+)+` 패턴은 입력 `aaaaab`에서 지수적 시간이 걸립니다.

### CYK 알고리즘 (Cocke-Younger-Kasami)
임의의 CFG에 대해 문자열이 그 문법에 속하는지 판별하는 O(n³) 알고리즘입니다. 문법을 먼저 **촘스키 정규형(Chomsky Normal Form, CNF)**으로 변환한 뒤 동적 프로그래밍으로 파싱합니다.

### 언어 계층의 경계
- **정규 언어** → 정규표현식, 어휘 분석기(Lex/Flex)
- **문맥 자유 언어** → 파서 생성기(Yacc/Bison, ANTLR), JSON/XML 파서
- **문맥 의존 언어** → 실제 프로그래밍 언어의 타입 검사 일부
- **재귀 열거 가능 언어** → 일반 프로그램의 계산 가능 문제

### 팁: 적절한 계층 선택하기
CSV 파싱에 CFG 파서를 쓰는 것은 과잉이고, JSON 파싱에 정규표현식을 쓰는 것은 잘못된 선택입니다. 문제의 특성에 맞는 올바른 계층의 도구를 선택하는 것이 핵심입니다.

---

## 마무리

오토마타 이론과 촘스키 계층 구조는 단순한 이론이 아닙니다. 매일 사용하는 컴파일러, 텍스트 에디터의 문법 강조, 데이터 검증 로직, 자연어 처리 시스템이 모두 이 체계 위에 구축됩니다. "이 언어는 파싱 가능한가?"라는 질문에 수학적으로 답할 수 있는 능력은 소프트웨어 엔지니어에게 강력한 무기가 됩니다.

## 참고 자료
- [Chomsky Hierarchy in Theory of Computation - GeeksforGeeks](https://www.geeksforgeeks.org/theory-of-computation/chomsky-hierarchy-in-theory-of-computation/)
- [Theory of Formal Languages, Automata, and Computation - Wikibooks](https://en.wikibooks.org/wiki/Theory_of_Formal_Languages,_Automata,_and_Computation/Automata_and_the_Chomsky_Hierarchy)
- [Automata Chomsky Hierarchy - Javatpoint](https://www.javatpoint.com/automata-chomsky-hierarchy)
- [Formal Languages and Automata Theory - GeeksforGeeks](https://www.geeksforgeeks.org/introduction-of-theory-of-computation/)
