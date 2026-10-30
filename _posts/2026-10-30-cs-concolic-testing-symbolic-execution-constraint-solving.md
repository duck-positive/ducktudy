---
layout: post
title: "콘콜릭 테스팅(Concolic Testing) 심화: 심볼릭 실행과 제약 해결을 통한 자동화 버그 탐지"
date: 2026-10-30
categories: [cs, computer-science]
tags: [concolic-testing, symbolic-execution, smt-solver, fuzzing, klee, dart, sage, z3, software-testing]
---

## 개념 설명

콘콜릭 테스팅(Concolic Testing)은 "CONCrete + symbOLIC"의 합성어로, 구체적 실행(Concrete Execution)과 심볼릭 실행(Symbolic Execution)을 동시에 수행하는 소프트웨어 테스팅 기법이다. 2005년 DART(Directed Automated Random Testing) 논문(Patrice Godefroid, Nils Klarlund, Koushik Sen)에서 처음 제안되었으며, 코드 커버리지를 체계적으로 극대화하고 경계 조건(edge case) 버그를 자동으로 발견하는 데 탁월한 성능을 발휘한다.

### 기존 테스팅 기법의 한계

- **랜덤 퍼징(Random Fuzzing)**: 임의의 입력을 생성해 프로그램을 실행한다. 단순하지만 복잡한 조건(`if (x == 0xDEADBEEF)`)을 우연히 통과할 확률이 매우 낮다.
- **순수 심볼릭 실행(Pure Symbolic Execution)**: 입력 값을 심볼(symbol)로 추상화하여 모든 실행 경로를 탐색하지만, 경로 폭발(Path Explosion) 문제로 확장성이 제한된다.

### 콘콜릭 테스팅의 동작 원리

콘콜릭 테스팅은 두 방식의 장점을 결합한다:

```
1. 초기 구체적 입력(seed)으로 프로그램 실행
   ↓
2. 실행 중 각 조건 분기에서 심볼릭 경로 조건(Path Condition) 누적
   ↓
3. 경로 조건 중 하나를 부정(negate)하여 새로운 분기로 진입하는 입력 생성
   ↓ (SMT 솔버 호출: Z3, STP, CVC5 등)
4. 새 입력으로 1번부터 반복 → 체계적으로 모든 경로 탐색
```

예를 들어, `if (x > 0) { if (y == 42) { bug(); } }` 코드를 탐색하는 과정:

| 라운드 | 구체적 입력 | 실행 경로 | 경로 조건 | 다음 목표 |
|--------|------------|-----------|-----------|----------|
| 1 | x=1, y=0 | x>0 ✓, y==42 ✗ | [x>0, y≠42] | y==42 부정 |
| 2 | x=1, y=42 | x>0 ✓, y==42 ✓ | [x>0, y==42] | **bug() 도달!** |

## 왜 필요한가

실제 소프트웨어에서 버그는 특정 입력 조건이 충족될 때만 드러나는 경우가 많다:

- **버퍼 오버플로우**: 특정 크기의 입력에서만 발생하는 경계 조건
- **정수 오버플로우**: 특정 큰 숫자 조합 (예: `INT_MAX + 1`)
- **NULL 포인터 역참조**: 특정 API 호출 순서
- **암호화 라이브러리 버그**: 특정 키·메시지 패턴

단위 테스트와 수동 퍼징은 이런 버그를 모두 잡기 어렵다. 콘콜릭 테스팅은 SMT 솔버를 활용하여 수학적으로 새로운 버그 유발 입력을 생성하므로, 코드 커버리지를 체계적으로 향상시킨다.

**실제 활용 사례**:
- Microsoft SAGE: 내부 바이너리에서 수백 개의 취약점 발견. 현재도 일상적으로 사용.
- NASA: 우주선 소프트웨어 검증에 KLEE 활용.
- OpenSSL·SQLite: KLEE로 수십 개의 버그 발견.

## 실제 구현 예제

### 예제 1: Z3 SMT 솔버를 이용한 콘콜릭 테스팅 시뮬레이션

파이썬과 Z3 솔버를 사용하여 콘콜릭 테스팅의 동작 원리를 직접 구현한다.

```python
"""
콘콜릭 테스팅 시뮬레이터 (z3-solver 필요: pip install z3-solver)

대상 함수의 모든 실행 경로를 체계적으로 탐색하여
버그를 유발하는 입력을 자동으로 발견한다.
"""
from z3 import Int, Solver, sat, And, Not
from typing import List, Tuple

def vulnerable_function(x: int, y: int) -> str:
    """
    특정 조건에서만 버그가 발생하는 함수.
    랜덤 테스트로는 발견하기 매우 어렵다.
    """
    if x > 0:               # 조건 A
        if y > 100:         # 조건 B
            if x + y == 200:   # 조건 C (매우 구체적!)
                result = 1000 // (x - 1)  # x == 1이면 크래시!
                return f"Result: {result}"
    return "Safe"

class ConcolicEngine:
    def __init__(self):
        self.bugs_found = []

    def run(self, max_iterations: int = 20):
        worklist = [(0, 0, [])]
        visited_conditions = set()

        print("=" * 60)
        print("콘콜릭 테스팅 시작")
        print("=" * 60)

        iteration = 0
        while worklist and iteration < max_iterations:
            x_val, y_val, _ = worklist.pop(0)
            iteration += 1
            print(f"\n[Round {iteration}] 입력: x={x_val}, y={y_val}")

            x_sym, y_sym = Int('x'), Int('y')
            new_conditions = self._execute_symbolically(x_val, y_val, x_sym, y_sym)

            try:
                result = vulnerable_function(x_val, y_val)
                print(f"  실행 결과: {result}")
            except ZeroDivisionError as e:
                self.bugs_found.append((x_val, y_val, str(e)))
                print(f"  [!!! BUG FOUND !!!] x={x_val}, y={y_val}: {e}")

            for i, condition in enumerate(new_conditions):
                negated = new_conditions[:i] + [Not(condition)]
                key = str(negated)
                if key in visited_conditions:
                    continue
                visited_conditions.add(key)

                new_input = self._solve_constraints(x_sym, y_sym, negated)
                if new_input is not None:
                    nx, ny = new_input
                    print(f"  → 새 경로 탐색: x={nx}, y={ny} (조건 {i+1} 부정)")
                    worklist.append((nx, ny, negated))

        print("\n" + "=" * 60)
        print(f"탐색 완료: {iteration}개 경로 탐색")
        print(f"발견된 버그: {len(self.bugs_found)}개")
        for bug in self.bugs_found:
            print(f"  - 입력 x={bug[0]}, y={bug[1]}: {bug[2]}")

    def _execute_symbolically(self, x_val, y_val, x_sym, y_sym):
        conditions = []
        if x_val > 0:
            conditions.append(x_sym > 0)
            if y_val > 100:
                conditions.append(y_sym > 100)
                if x_val + y_val == 200:
                    conditions.append(x_sym + y_sym == 200)
        else:
            conditions.append(x_sym <= 0)
        return conditions

    def _solve_constraints(self, x_sym, y_sym, constraints):
        solver = Solver()
        solver.add(x_sym >= -1000, x_sym <= 1000)
        solver.add(y_sym >= -1000, y_sym <= 1000)
        for c in constraints:
            solver.add(c)
        if solver.check() == sat:
            model = solver.model()
            x_v = model[x_sym].as_long() if model[x_sym] is not None else 0
            y_v = model[y_sym].as_long() if model[y_sym] is not None else 0
            return x_v, y_v
        return None

if __name__ == "__main__":
    engine = ConcolicEngine()
    engine.run(max_iterations=15)
```

### 예제 2: KLEE로 C 언어 프로그램의 취약점 탐지

실제 콘콜릭 테스팅 도구 KLEE는 LLVM IR 수준에서 심볼릭 실행을 수행한다. 아래는 네트워크 패킷 파서의 취약점을 탐지하는 예제다.

```c
/**
 * packet_parser.c - 취약한 패킷 파서 (KLEE로 버그 탐지)
 *
 * 컴파일 및 실행:
 *   clang -I/path/to/klee/include -emit-llvm -c -g -O0 packet_parser.c -o packet_parser.bc
 *   klee --max-time=60 --optimize packet_parser.bc
 *   klee-stats klee-last/
 */
#include <stdio.h>
#include <string.h>
#include <klee/klee.h>

#define MAX_PAYLOAD 256

typedef struct {
    unsigned char  type;
    unsigned short length;
    unsigned char  flags;
    unsigned char  payload[MAX_PAYLOAD];
} Packet;

// 버그 1: 버퍼 길이 검증 누락
int parse_data_packet(Packet *pkt, int buf_len) {
    if (pkt->type != 0x01) return -1;

    unsigned short payload_len = (pkt->payload[0] << 8) | pkt->payload[1];
    // 올바른 코드: if (payload_len > buf_len - 4) return -1;
    
    char output[64];
    memcpy(output, pkt->payload + 2, payload_len);  // 버퍼 오버플로우 취약점!
    output[payload_len] = '\0';
    printf("Data: %s\n", output);
    return 0;
}

// 버그 2: 정수 오버플로우로 인한 경계 검사 우회
int parse_control_packet(Packet *pkt) {
    if (pkt->type != 0xFF) return -1;

    unsigned int count = pkt->flags;
    unsigned int total_size = count * sizeof(int);  // 정수 오버플로우 가능!
    
    if (total_size > MAX_PAYLOAD) {
        return -1;  // count가 크면 오버플로우로 우회됨!
    }
    
    int buffer[MAX_PAYLOAD / sizeof(int)];
    for (unsigned int i = 0; i < count; i++) {
        buffer[i] = pkt->payload[i];  // 경계 초과 가능!
    }
    return 0;
}

int main() {
    Packet pkt;
    int buf_len;

    klee_make_symbolic(&pkt, sizeof(pkt), "packet");
    klee_make_symbolic(&buf_len, sizeof(buf_len), "buf_len");

    klee_assume(buf_len >= 0 && buf_len <= MAX_PAYLOAD + 4);
    klee_assume(pkt.length <= MAX_PAYLOAD);

    if (pkt.type == 0x01) {
        parse_data_packet(&pkt, buf_len);
    } else if (pkt.type == 0xFF) {
        parse_control_packet(&pkt);
    }

    return 0;
}

/*
 * KLEE 실행 결과 예시:
 *   KLEE: ERROR: memory error: out of bound pointer (버그 1)
 *   KLEE: ERROR: memory error: out of bound pointer (버그 2)
 *
 * 생성된 테스트 케이스:
 *   klee-last/test000001.ktest → 버그 1 유발 입력
 *   klee-last/test000002.ktest → 버그 2 유발 입력
 */
```

## 주요 콘콜릭 테스팅 도구

| 도구 | 대상 | 언어/레벨 | 특징 |
|------|------|-----------|------|
| **DART** | 소스 코드 | C | 최초 콘콜릭 테스팅 도구 (2005) |
| **SAGE** | 바이너리 | x86 | Microsoft 내부 사용, 대규모 취약점 탐지 |
| **KLEE** | LLVM IR | C/C++ | 오픈소스, POSIX 환경 시뮬레이션 |
| **Angr** | 바이너리 | Python API | CTF·역공학·취약점 연구에 널리 사용 |
| **Triton** | 바이너리 | C++/Python | 동적 이진 분석, DBA 프레임워크 |
| **JBSE** | JVM 바이트코드 | Java | 자바 힙 형태 심볼릭 분석 |
| **Jalangi2** | 소스 코드 | JavaScript | 동적 분석·테인트 추적 |

## 주의사항 및 팁

### 1. 경로 폭발(Path Explosion) 문제와 완화 전략

루프·재귀를 포함하는 코드는 탐색 경로 수가 지수적으로 증가한다.

```python
# 완화 전략 1: 루프 언바운딩(Loop Unwinding) 제한
# KLEE: --max-loops=10 플래그로 루프 반복 상한 설정

# 완화 전략 2: 경로 병합(Path Merging)
# 여러 경로의 심볼릭 상태를 하나로 합쳐 탐색 공간 축소
# Veritesting (Avgerinos et al., 2014) 기법

# 완화 전략 3: 검색 휴리스틱
# - 커버리지 기반 우선 탐색 (Coverage-Guided Search)
# - 가장 적게 방문된 명령 근처 경로 우선
# - DFS보다 BFS가 짧은 경로를 먼저 탐색하여 빠른 버그 발견
```

### 2. 환경 의존성 처리

파일 시스템, 네트워크 소켓, 난수 생성기 등 외부 환경과 상호작용하는 코드는 심볼릭으로 모델링하기 어렵다.

- **KLEE POSIX**: 파일 I/O, 환경 변수, 명령줄 인수를 심볼릭으로 처리하는 POSIX 환경 모델 제공.
- **UC-KLEE**: 외부 함수를 심볼릭 요약(Summary)으로 대체.
- **S2E**: 전체 시스템(OS + 하이퍼바이저)을 심볼릭으로 실행. 드라이버·커널 코드도 분석 가능.

### 3. SMT 솔버 선택 가이드

| 솔버 | 강점 | 주요 사용처 |
|------|------|-------------|
| **Z3** | 다양한 이론 지원, 광범위한 채택 | KLEE, Angr, 대부분의 도구 |
| **STP** | 비트벡터 특화, 빠름 | KLEE 기본, 바이너리 분석 |
| **CVC5** | 일부 이론에서 Z3보다 빠름 | 형식 검증, 프로그램 합성 |
| **Boolector** | 비트벡터·배열 특화 | 임베디드 시스템 분석 |

### 4. 커버리지 기반 퍼징과의 결합 (Hybrid Fuzzing)

현대 퍼징 기법은 콘콜릭 테스팅과 커버리지 기반 퍼징(AFL, libFuzzer)을 결합한 **하이브리드 퍼징**을 사용한다.

```
AFL/libFuzzer (빠른 무작위 변형)
    ↓ 막힌 경계 조건 발견
Concolic Engine (SMT 솔버로 정밀 입력 생성)
    ↓ 새 경로 통과 입력
AFL/libFuzzer로 피드백 → 더 많은 커버리지
```

대표 도구: **Driller** (Shellphish 팀), **QSYM** (삼성리서치), **SymCC** (컴파일러 통합).

### 5. 실전 활용 패턴

```bash
# KLEE 실행 예시
clang -I$(llvm-config --includedir) \
      -emit-llvm -c -g -O0 target.c -o target.bc

klee \
  --max-time=300 \
  --max-memory=4096 \
  --optimize \
  --write-kqueries \
  target.bc

# 발견된 버그 케이스 재현
ktest-tool klee-last/test000001.ktest
klee-replay target klee-last/test000001.ktest
```

콘콜릭 테스팅은 전통적인 테스팅과 공식 검증의 중간 지점에 위치한다. 완전한 자동화로 코드 커버리지를 극대화하면서도, 현실적인 시간 안에 결과를 제공한다. 특히 보안 취약점 탐지, 임베디드 시스템 검증, 프로토콜 파서 분석 등에서 강력한 효과를 발휘한다.

## 참고 자료
- [Concolic Testing - Wikipedia](https://en.wikipedia.org/wiki/Concolic_testing)
- [DART: Directed Automated Random Testing (PLDI 2005 PDF)](https://www.cse.iitd.ac.in/~siy117527/sil765/readings/fuzzing%20testing.pdf)
- [KLEE Symbolic Execution Engine](https://klee.github.io/)
- [Angr Binary Analysis Framework](https://angr.io/)
