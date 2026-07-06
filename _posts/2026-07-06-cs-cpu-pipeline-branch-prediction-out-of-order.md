---
layout: post
title: "CPU 파이프라인 심화: 분기 예측, 슈퍼스칼라, Out-of-Order 실행"
date: 2026-07-06
categories: [cs, computer-science]
tags: [cpu, pipeline, branch-prediction, superscalar, out-of-order-execution, computer-architecture, spectre]
---

현대 CPU가 초당 수십억 개의 명령을 처리할 수 있는 이유는 단순히 클럭 속도가 빠르기 때문만이 아닙니다. **파이프라인(Pipeline)**, **슈퍼스칼라(Superscalar)**, **비순서 실행(Out-of-Order Execution)**, 그리고 **분기 예측(Branch Prediction)** 같은 마이크로아키텍처 기법들이 명령어 수준 병렬성(ILP)을 최대한 끌어올리기 때문입니다. 이 아티클에서는 이 네 가지 기법의 원리를 깊이 살펴보고, 이들이 어떻게 맞물려 동작하는지, 그리고 Spectre 같은 보안 취약점이 왜 등장하는지까지 설명합니다.

---

## 1. CPU 파이프라인: 명령어를 단계별로 분해한다

### 1.1 파이프라인이 없던 시절

파이프라인이 없는 순차 실행 CPU는 하나의 명령어를 **완전히 완료한 뒤** 다음 명령어를 시작합니다. 예를 들어 `LOAD`, `ADD`, `STORE` 명령어가 각각 5 사이클이 걸린다면, 3개 명령어를 처리하는 데 총 15 사이클이 필요합니다.

### 1.2 파이프라인의 원리

파이프라인은 명령어 처리를 여러 **단계(Stage)**로 나누고, 각 단계가 서로 다른 명령어를 동시에 처리합니다. 전형적인 5단계 RISC 파이프라인은 다음과 같습니다:

```
IF  → ID  → EX  → MEM → WB
(명령어 인출) (해독) (실행) (메모리 접근) (결과 기록)
```

```
사이클:  1    2    3    4    5    6    7
명령어1: IF   ID   EX   MEM  WB
명령어2:      IF   ID   EX   MEM  WB
명령어3:           IF   ID   EX   MEM  WB
```

5단계 파이프라인에서 정상 상태에는 매 사이클마다 한 명령어가 완료됩니다. 이론적으로 CPI(Cycles Per Instruction)가 1이 됩니다.

### 1.3 파이프라인 해저드(Hazard)

파이프라인이 항상 완벽하게 흐르지는 않습니다. 세 가지 해저드가 파이프라인을 멈추게(Stall) 만듭니다.

**데이터 해저드(Data Hazard):** 이전 명령어의 결과를 다음 명령어가 필요로 할 때 발생합니다.

```c
// RAW(Read After Write) 해저드 예시
int a = b + c;   // 명령어 1: b+c를 계산하여 a에 저장
int d = a * 2;   // 명령어 2: a를 읽어야 하는데, 명령어 1이 WB 단계에 도달하기 전에 EX 단계 시작
```

**전방 전달(Forwarding / Bypassing):** 하드웨어가 WB까지 기다리지 않고 EX 단계 결과를 바로 다음 명령어의 EX 단계 입력으로 전달하여 해결합니다.

**제어 해저드(Control Hazard):** 분기 명령어(`if`, `for`)에서 다음에 실행할 명령어가 불확실할 때 발생합니다. 분기 방향이 결정될 때까지 파이프라인을 비워야(Flush) 합니다.

---

## 2. 슈퍼스칼라: 파이프라인을 여러 개 동시에

CPI 1을 달성했다면 더 이상 개선할 수 없을까요? **슈퍼스칼라 아키텍처**는 한 사이클에 여러 명령어를 동시에 발행(Issue)함으로써 CPI를 1 미만으로 끌어내립니다.

예를 들어 4-wide 슈퍼스칼라 CPU는 하나의 사이클에 4개의 명령어를 서로 다른 실행 유닛(ALU, FPU, 로드/스토어 유닛, 분기 유닛 등)에 동시에 보냅니다.

```
사이클 1: [ADD R1, R2, R3]  [MUL R4, R5, R6]  [LOAD R7, [addr]]  [CMP R8, R9]
                  ↓                ↓                  ↓                 ↓
              정수 ALU           FPU           로드/스토어 유닛       분기 유닛
```

이를 **명령어 수준 병렬성(ILP, Instruction-Level Parallelism)**이라 합니다. 단, 명령어들 사이에 데이터 의존성이 없어야 병렬 실행이 가능합니다.

---

## 3. Out-of-Order 실행: 순서를 지키되 결과만 맞추면 된다

### 3.1 순서대로 실행의 비효율

파이프라인과 슈퍼스칼라만으로는 여전히 한계가 있습니다. 아래 코드를 봅시다:

```asm
LOAD  R1, [slow_memory]   ; 캐시 미스! 200+ 사이클 대기
ADD   R2, R1, 10          ; R1 의존, 대기 필요
MUL   R3, R4, R5          ; R1과 무관! 기다릴 이유 없음
```

인-오더(In-Order) CPU는 LOAD가 완료될 때까지 그 뒤 명령어를 모두 멈춥니다. 하지만 `MUL R3, R4, R5`는 R1과 전혀 무관합니다. Out-of-Order(OoO) 실행은 이런 독립적인 명령어를 먼저 처리합니다.

### 3.2 OoO 실행의 핵심 구성요소

```
명령어 스트림
    ↓
[프론트엔드: Fetch → Decode → Rename]
    ↓
[ROB(ReOrder Buffer) & Issue Queue]
    ↓           ↓           ↓
[ALU 유닛]  [로드/스토어]  [FPU 유닛]   ← 준비된 명령어만 실행
    ↓
[커밋(Retire): ROB에서 프로그램 순서대로 결과 반영]
```

**레지스터 리네이밍(Register Renaming):** WAR(Write After Read), WAW(Write After Write) 의존성을 제거합니다. 아키텍처 레지스터(예: x86의 EAX)를 물리 레지스터(예: 192개) 중 하나에 동적 매핑합니다.

**ROB(ReOrder Buffer):** 명령어가 **비순서로** 실행되더라도 **순서대로 커밋**을 보장합니다. 인터럽트와 예외 처리의 정확성을 위해 필수입니다.

```python
# OoO 실행 시뮬레이션 (개념 설명용 의사코드)
class ReorderBuffer:
    def __init__(self, size=192):
        self.buffer = []
        self.size = size

    def issue(self, instruction):
        """명령어를 ROB에 등록, 실행은 비순서로 가능"""
        entry = {
            'instr': instruction,
            'result': None,
            'ready': False,
            'program_order': len(self.buffer)
        }
        self.buffer.append(entry)

    def execute_ready(self):
        """의존성 없는 명령어를 즉시 실행"""
        for entry in self.buffer:
            if not entry['ready'] and self.operands_available(entry['instr']):
                entry['result'] = self.execute(entry['instr'])
                entry['ready'] = True

    def commit(self):
        """프로그램 순서대로 커밋 (ROB 헤드부터)"""
        while self.buffer and self.buffer[0]['ready']:
            committed = self.buffer.pop(0)
            self.write_back(committed)

    def operands_available(self, instr):
        # 소스 레지스터가 이미 커밋됐거나 forwarding 가능한지 확인
        return True  # 단순화

    def execute(self, instr):
        return instr.compute()

    def write_back(self, entry):
        print(f"Committing: {entry['instr']} = {entry['result']}")
```

---

## 4. 분기 예측: 미래를 맞춰야 파이프라인이 멈추지 않는다

### 4.1 분기 예측이 왜 필요한가

현대 CPU의 파이프라인 깊이는 14~20단계에 달합니다. 분기 명령어(`if`, `for`, `while`)에서 어느 경로를 택할지 모른다면, 분기 결과가 확정될 때까지 파이프라인을 비워야 합니다. 이는 **14~20 사이클의 낭비**를 의미합니다. 일반적인 프로그램에서 분기는 5~10명령어마다 한 번씩 등장하므로, 예측 없이는 성능이 1/5~1/10로 떨어집니다.

### 4.2 분기 예측기의 발전 과정

**1단계: 정적 예측(Static Prediction)**  
컴파일러가 코드를 분석해 "이 분기는 항상 taken" 같은 힌트를 생성합니다. 루프 분기는 대개 `taken`으로 예측합니다.

**2단계: 1비트 예측기**  
마지막 분기 결과를 저장합니다. 루프 마지막 반복에서 오예측이 발생합니다.

**3단계: 2비트 포화 카운터(Saturating Counter)**  
`Strongly Not Taken → Weakly Not Taken → Weakly Taken → Strongly Taken` 상태로 한 번의 오예측으로는 예측이 바뀌지 않습니다.

**4단계: 상관 예측기(Correlated Predictor) / gshare**  
전역 분기 히스토리(최근 N개 분기 결과)와 현재 분기의 PC를 XOR하여 더 정확한 예측 테이블을 구성합니다.

**5단계: TAGE(Tagged Geometric Predictor)**  
Intel의 최신 CPU와 AMD의 Zen 아키텍처에서 사용하는 예측기입니다. 여러 길이의 히스토리를 사용하는 예측 테이블들을 계층적으로 구성합니다.

```python
# 2비트 포화 카운터 기반 분기 예측기 구현
class TwoBitBranchPredictor:
    NOT_TAKEN_STRONG = 0
    NOT_TAKEN_WEAK   = 1
    TAKEN_WEAK       = 2
    TAKEN_STRONG     = 3

    def __init__(self, table_size=4096):
        # PC 하위 비트로 인덱싱하는 예측 테이블
        self.table = [self.WEAKLY_TAKEN() for _ in range(table_size)]
        self.table_size = table_size
        self.correct = 0
        self.total = 0

    def WEAKLY_TAKEN(self):
        return self.TAKEN_WEAK

    def predict(self, pc):
        """분기가 taken될지 예측"""
        index = pc % self.table_size
        state = self.table[index]
        return state >= self.TAKEN_WEAK  # True: taken, False: not taken

    def update(self, pc, actually_taken):
        """실제 결과로 예측기 업데이트"""
        index = pc % self.table_size
        predicted = self.predict(pc)
        state = self.table[index]

        if actually_taken:
            self.table[index] = min(state + 1, self.TAKEN_STRONG)
        else:
            self.table[index] = max(state - 1, self.NOT_TAKEN_STRONG)

        self.total += 1
        if predicted == actually_taken:
            self.correct += 1

    def accuracy(self):
        return self.correct / self.total if self.total > 0 else 0


# 루프 패턴 시뮬레이션
predictor = TwoBitBranchPredictor()
pc = 0x1000

# 루프 10회 반복: 9번 taken, 1번 not taken
for iteration in range(5):  # 5번의 루프 실행
    for i in range(9):
        actually_taken = True  # 루프 내부
        predictor.update(pc, actually_taken)
    predictor.update(pc, False)  # 루프 탈출

print(f"예측 정확도: {predictor.accuracy():.1%}")
# 출력: 예측 정확도: 약 91%~95% (처음 몇 번의 오예측 후 안정)
```

### 4.3 추측 실행(Speculative Execution)과 Spectre 취약점

분기 예측의 강력함은 **추측 실행**에서 나옵니다. CPU는 예측 결과를 믿고 분기 이후 명령어를 미리 실행합니다. 예측이 맞으면 그 결과를 그대로 사용하고, 틀리면 파이프라인을 비우고(flush) 재실행합니다.

문제는 **추측 실행 중 캐시 상태가 바뀐다**는 점입니다. **Spectre(2018)** 취약점은 이를 악용합니다:

```c
// Spectre v1 공격 패턴 (취약한 코드)
// 공격자는 index를 out-of-bounds로 조작
if (index < array_size) {          // 분기 예측으로 통과될 것을 기대
    uint8_t val = secret_array[index];  // 비밀 데이터 접근 (추측 실행)
    // val에 따라 다른 캐시 라인을 접근 → 캐시 타이밍으로 val 추론 가능
    uint8_t probe = probe_array[val * 512];
}
// 분기 결과가 틀렸어도 캐시 상태는 이미 변경됨
```

추측 실행 중 접근된 데이터는 캐시에 남아 타이밍 채널(Timing Side Channel)로 노출됩니다. 완전한 수정은 하드웨어 마이크로코드 업데이트와 OS 패치를 통해 이루어졌습니다(`retpoline`, `IBRS`).

---

## 5. 성능에 미치는 실제 영향과 최적화 팁

### 5.1 분기 예측을 돕는 코드 작성

```c
#include <stdlib.h>
#include <time.h>
#include <stdio.h>

// 예측하기 어려운 코드: 무작위 분기
void bad_branch_example(int* arr, int size) {
    int sum = 0;
    for (int i = 0; i < size; i++) {
        if (arr[i] > 128) {  // 랜덤 데이터 → 50% 예측 실패
            sum += arr[i];
        }
    }
}

// 개선: 데이터를 정렬하면 분기 패턴이 규칙적으로 변함
// [0, 0, 0, ..., 1, 1, 1] 패턴 → 예측기가 쉽게 학습
void good_branch_example(int* sorted_arr, int size) {
    int sum = 0;
    for (int i = 0; i < size; i++) {
        if (sorted_arr[i] > 128) {  // 정렬 후: 예측 정확도 90%+
            sum += sorted_arr[i];
        }
    }
}

// 더 나은 방법: 분기 자체를 제거 (Branchless 코드)
void branchless_example(int* arr, int size) {
    int sum = 0;
    for (int i = 0; i < size; i++) {
        // 분기 없이 조건부 덧셈: cmov 명령어로 컴파일
        sum += arr[i] * (arr[i] > 128);
    }
}
```

실제 벤치마크에서 랜덤 데이터에 대해 정렬 후 처리는 약 **3~5배** 빠를 수 있습니다.

### 5.2 컴파일러 힌트: `__builtin_expect`

```c
// GCC/Clang에서 분기 예측 힌트 제공
#define likely(x)   __builtin_expect(!!(x), 1)
#define unlikely(x) __builtin_expect(!!(x), 0)

void process_packet(int* data, int size) {
    if (unlikely(data == NULL)) {  // 에러는 드물게 발생
        handle_error();
        return;
    }
    // 정상 경로: 예측기가 이 방향을 우선 처리
    process(data, size);
}
```

Linux 커널 코드 전반에서 이 패턴이 사용됩니다.

---

## 6. 주의사항과 정리

| 기법 | 이점 | 한계/비용 |
|------|------|----------|
| 파이프라인 | CPI ≈ 1 달성 | 해저드 시 스톨 발생 |
| 슈퍼스칼라 | CPI < 1 가능 | 의존성 분석 하드웨어 복잡 |
| OoO 실행 | 메모리 지연 숨김 | ROB/RS 크기 = 다이 면적 증가 |
| 분기 예측 | 제어 해저드 최소화 | 오예측 시 15~20사이클 패널티 |
| 추측 실행 | 예측과 실행 중첩 | Spectre 등 보안 취약점 |

현대 CPU 성능의 대부분은 이 메커니즘들의 조합에서 나옵니다. 성능 최적화를 할 때 프로파일러로 **branch misprediction** 지표를 확인하고, 핫 루프에서 무작위 분기를 줄이거나 branchless 코드로 교체하는 것이 효과적입니다.

---

## 참고 자료

- [Wikipedia — Branch predictor](https://en.wikipedia.org/wiki/Branch_predictor)
- [Wikipedia — Out-of-order execution](https://en.wikipedia.org/wiki/Out-of-order_execution)
- [Agner Fog's CPU Optimization Manuals](https://www.agner.org/optimize/)
- [Intel® 64 and IA-32 Architectures Optimization Reference Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
