---
layout: post
title: "CPU 파이프라인과 분기 예측: 현대 프로세서가 성능을 극대화하는 방법"
date: 2026-11-01
categories: [cs, computer-science]
tags: [cpu, pipeline, branch-prediction, computer-architecture, performance, spectre, out-of-order-execution]
---

## CPU 파이프라인이란 무엇인가?

현대 CPU는 단순히 명령어를 하나씩 실행하지 않습니다. 명령어를 **여러 단계(스테이지)**로 나누어 동시에 처리함으로써 처리량(throughput)을 극적으로 높입니다. 이 메커니즘을 **파이프라인(Pipeline)**이라 합니다.

공장 조립 라인을 떠올리면 이해하기 쉽습니다. 자동차 조립에서 섀시 조립, 엔진 설치, 도장, 품질 검사를 각각 다른 팀이 동시에 수행하듯, CPU도 각 명령어의 단계를 병렬로 처리합니다.

### 클래식 5단계 파이프라인(RISC 기반)

```
사이클:  1    2    3    4    5    6    7
Inst1:  [IF][ID][EX][MA][WB]
Inst2:       [IF][ID][EX][MA][WB]
Inst3:            [IF][ID][EX][MA][WB]
Inst4:                 [IF][ID][EX][MA][WB]
Inst5:                      [IF][ID][EX][MA][WB]

IF = Instruction Fetch (명령어 인출)
ID = Instruction Decode (명령어 해독 + 레지스터 읽기)
EX = Execute (ALU 연산)
MA = Memory Access (메모리 읽기/쓰기)
WB = Write Back (레지스터 결과 저장)
```

이상적으로는 5개 명령어가 5사이클에 처리됩니다. 파이프라인이 없다면 같은 작업에 25사이클(5명령 × 5단계)이 걸립니다. **5배의 처리량 향상**입니다.

현대 Intel/AMD CPU는 10~20단계의 슈퍼스칼라 파이프라인을 사용하며, 사이클당 여러 명령을 동시 발행(Multiple Issue)합니다.

## 파이프라인 해저드(Hazard): 성능의 적

파이프라인의 매끄러운 흐름을 방해하는 세 가지 해저드가 있습니다.

### 1. 구조적 해저드(Structural Hazard)
두 명령어가 같은 하드웨어 자원을 동시에 필요로 할 때 발생합니다. 예: 단일 메모리 포트에서 명령어 인출과 데이터 읽기가 충돌. 현대 CPU는 L1 명령어 캐시와 데이터 캐시를 분리하여 해결합니다.

### 2. 데이터 해저드(Data Hazard)
명령어 간 데이터 의존성으로 발생합니다.

```
ADD R1, R2, R3   # R1 = R2 + R3 (WB는 5사이클에 완료)
SUB R4, R1, R5   # R4 = R1 - R5 (EX는 3사이클에 시작 → R1이 아직 없음!)
```

해결책은 **포워딩(Forwarding/Bypassing)**: EX 단계의 결과를 WB를 기다리지 않고 다음 명령어의 EX 입력으로 직접 전달합니다. 이 하드웨어 경로가 없으면 **버블(stall, NOP 삽입)**이 발생하여 파이프라인이 지연됩니다.

### 3. 제어 해저드(Control Hazard) — 분기 문제
조건 분기(`if`, `for`, `while`)는 파이프라인이 분기 결과를 모르는 상태에서 다음 명령어를 인출해야 할 때 해저드를 일으킵니다. 분기 목적지가 EX 단계에서야 확정되면, 그 사이 파이프라인에 들어온 2~3개 명령어를 **버려야(flush)** 합니다. 이것이 **분기 패널티(Branch Penalty)**이며, 현대 CPU에서는 10~20사이클에 달합니다.

## 분기 예측(Branch Prediction): 해결책

분기 패널티를 줄이기 위해 CPU는 **분기가 어느 방향으로 갈지 미리 예측**하고, 예측된 경로의 명령어를 투기적으로 실행(Speculative Execution)합니다. 예측이 맞으면 패널티가 없고, 틀리면 투기적으로 실행한 결과를 폐기하고 올바른 경로로 재시작합니다.

### 정적 분기 예측

컴파일러 힌트나 단순 규칙에 의존합니다.

```c
#include <stdbool.h>

// GCC/Clang의 likely/unlikely 힌트
#define likely(x)   __builtin_expect(!!(x), 1)
#define unlikely(x) __builtin_expect(!!(x), 0)

int process_packet(struct packet *pkt) {
    // 95% 확률로 유효한 패킷이 들어온다고 힌트
    if (likely(pkt->is_valid)) {
        return fast_path(pkt);
    }
    // 오류 처리는 드문 경우 — 분기 예측기가 이쪽을 "불가능"으로 간주
    if (unlikely(pkt->is_corrupted)) {
        log_error(pkt);
        return -1;
    }
    return slow_path(pkt);
}
```

`__builtin_expect`는 분기 방향에 대한 힌트를 컴파일러에 제공하여, 자주 실행되는 경로가 점프 없이 순차적으로 배치되도록 코드를 구성합니다. 이는 명령어 캐시 지역성도 개선합니다.

### 동적 분기 예측: 2비트 포화 카운터

하드웨어 분기 예측기는 분기의 **실행 이력**을 기반으로 예측합니다. 가장 기본적인 구현은 2비트 포화 카운터(Saturating Counter)입니다.

```
상태: 00(강하게 Not Taken) → 01(약하게 NT) → 10(약하게 Taken) → 11(강하게 Taken)
       ←──────────── Not Taken ────────────          ──────── Taken ───────→

예시: for 루프 10회 반복
  초기: 00(NT)
  1회:  분기 Taken → 01
  2회:  분기 Taken → 10
  3회:  분기 Taken → 11
  4~9회: 분기 Taken → 11 유지
  10회: 분기 Not Taken (루프 종료) → 10
  → 10회 중 1회만 오예측: 90% 정확도
```

현대 CPU(Intel Raptor Lake, AMD Zen 4 등)는 훨씬 정교한 **TAGE(TAgged GEometric history length predictor)** 알고리즘을 사용합니다. 서로 다른 이력 길이를 가진 여러 예측기의 앙상블로, 99% 이상의 정확도를 달성합니다.

## 실제 구현 예제

### 예제 1: 분기 예측 영향을 확인하는 벤치마크 (C)

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

#define N 100000000  // 1억

// 테스트 1: 정렬된 배열 — 분기 패턴이 규칙적 → 예측 쉬움
long sum_sorted(int *arr, int n) {
    long sum = 0;
    for (int i = 0; i < n; i++) {
        if (arr[i] >= 128)   // 정렬됨: 128보다 작은 값들 → 128 이상 값들 순서
            sum += arr[i];
    }
    return sum;
}

// 테스트 2: 무작위 배열 — 분기 패턴이 불규칙 → 예측 어려움
long sum_random(int *arr, int n) {
    long sum = 0;
    for (int i = 0; i < n; i++) {
        if (arr[i] >= 128)   // 동일한 조건, 무작위 순서
            sum += arr[i];
    }
    return sum;
}

// 분기 없는 버전 (조건 없는 비트 마스크)
long sum_branchless(int *arr, int n) {
    long sum = 0;
    for (int i = 0; i < n; i++) {
        int mask = -(arr[i] >= 128);  // 조건 참이면 0xFFFFFFFF, 거짓이면 0
        sum += arr[i] & mask;
    }
    return sum;
}

int compare_int(const void *a, const void *b) {
    return *(int*)a - *(int*)b;
}

int main() {
    int *arr = malloc(N * sizeof(int));
    srand(42);
    for (int i = 0; i < N; i++) arr[i] = rand() % 256;

    // 무작위 배열 테스트
    struct timespec start, end;
    clock_gettime(CLOCK_MONOTONIC, &start);
    long r1 = sum_random(arr, N);
    clock_gettime(CLOCK_MONOTONIC, &end);
    printf("무작위: %ldms (sum=%ld)\n",
        (end.tv_sec - start.tv_sec) * 1000 +
        (end.tv_nsec - start.tv_nsec) / 1000000, r1);

    // 정렬 후 테스트
    qsort(arr, N, sizeof(int), compare_int);
    clock_gettime(CLOCK_MONOTONIC, &start);
    long r2 = sum_sorted(arr, N);
    clock_gettime(CLOCK_MONOTONIC, &end);
    printf("정렬됨: %ldms (sum=%ld)\n",
        (end.tv_sec - start.tv_sec) * 1000 +
        (end.tv_nsec - start.tv_nsec) / 1000000, r2);

    // 분기 없는 버전 (순서 무관)
    srand(42);
    for (int i = 0; i < N; i++) arr[i] = rand() % 256;  // 다시 무작위
    clock_gettime(CLOCK_MONOTONIC, &start);
    long r3 = sum_branchless(arr, N);
    clock_gettime(CLOCK_MONOTONIC, &end);
    printf("분기없음: %ldms (sum=%ld)\n",
        (end.tv_sec - start.tv_sec) * 1000 +
        (end.tv_nsec - start.tv_nsec) / 1000000, r3);

    free(arr);
    return 0;
}
```

```bash
$ gcc -O2 -o branch_test branch_test.c && ./branch_test
무작위:  312ms (sum=9563184448)
정렬됨:   94ms (sum=9563184448)
분기없음:  78ms (sum=9563184448)
```

정렬된 배열은 분기 패턴이 단조롭기 때문에 예측기가 99% 이상 정확합니다. 무작위 배열은 예측이 50%에 불과해 대량의 파이프라인 플러시가 발생합니다. 분기 없는(branchless) 버전은 예측이 필요 없어 가장 빠릅니다.

---

### 예제 2: Spectre 취약점과 분기 예측의 보안 함의 (Python 설명 코드)

```python
"""
Spectre 변형 1(CVE-2017-5753) 원리 설명
투기적 실행 중 캐시 사이드 채널로 비밀 데이터를 유출하는 원리

경고: 이 코드는 개념 설명용이며, 실제 공격 코드가 아닙니다.
"""

# 피해자 코드 (커널 or 다른 프로세스)
array1_size = 16
array1 = list(range(array1_size))   # 접근 허용 범위
array2 = [0] * 256 * 512            # 캐시 프로빙용 배열 (각 요소 512바이트 간격)
secret = ord('S')                   # 비밀 데이터 (예: 메모리 어딘가의 비밀)

def victim_function(x: int) -> None:
    """
    분기 예측기 훈련 후, 악의적 x로 호출 시:
    1. CPU는 array1_size 검사가 참일 것으로 예측 (훈련됨)
    2. 투기적으로 array1[x] 실행 — 범위 밖 접근!
    3. 그 결과로 array2[array1[x] * 512] 를 캐시에 로드
    4. 분기가 '거짓'임이 밝혀지면 레지스터는 복구되지만
       캐시 상태는 복구되지 않는다 ← 사이드 채널 유출 지점
    """
    if x < array1_size:  # 경계 검사 — 하지만 투기적 실행이 이를 우회
        _ = array2[array1[x] * 512]  # 캐시에 로드

def attacker_probe_cache(b: int) -> float:
    """array2[b*512]의 캐시 히트 여부로 접근 여부 추론"""
    import time
    t = time.perf_counter_ns()
    _ = array2[b * 512]
    elapsed = time.perf_counter_ns() - t
    # 캐시 히트: ~4ns, 캐시 미스: ~100ns (DRAM 접근)
    return elapsed

# 공격 흐름 설명 (실제 타이밍 공격은 하드웨어 수준 정밀도 필요)
print("Spectre 원리:")
print("1. 분기 예측기를 합법적 x로 반복 훈련")
print("2. 악의적 x (범위 밖, 비밀 데이터 가리킴)로 victim_function 호출")
print("3. CPU가 투기적으로 비밀 데이터를 읽어 array2를 캐시에 로드")
print("4. 256개 후보를 타이밍 측정 → 가장 빠른 것이 비밀 값")
print()
print("완화 방법:")
print("  - Retpoline: 간접 분기를 ret-loop으로 대체, 투기적 실행 방지")
print("  - IBRS/IBPB: 분기 예측 버퍼 격리 (성능 5~30% 저하)")
print("  - KPTI: 커널 페이지 테이블 격리 (커널 주소 숨김)")
print("  - lfence 명령: 투기적 실행 명시적 차단 (고비용)")
```

## 비순서 실행(Out-of-Order Execution)과의 관계

현대 CPU는 분기 예측 위에 **비순서 실행(OoO, Out-of-Order Execution)**을 추가로 구현합니다. 데이터 의존성이 없는 명령어들을 실행 준비가 된 순서대로 실행하고, 결과는 원래 프로그램 순서대로 **재순서화 버퍼(ROB, Re-Order Buffer)**에서 커밋합니다.

```
프로그램 순서:
  1: LOAD R1, [addr]  ← 캐시 미스, 100 사이클 대기
  2: ADD R2, R1, R3   ← R1에 의존, 대기
  3: MUL R4, R5, R6   ← 독립적!
  4: ADD R7, R4, R8   ← R4에 의존, 3번 완료 후 실행 가능

OoO 실행 순서:
  사이클 1: 1번 발행 (LOAD 시작)
  사이클 2: 3번 먼저 실행 (독립적)
  사이클 3: 4번 실행 (3번 완료)
  사이클 101: 1번 완료, 2번 실행
  사이클 102: 1,2,3,4 순서대로 커밋
```

분기 예측이 없다면 OoO는 분기 이후 명령어를 투기적으로 실행할 수 없어 효과가 반감됩니다. 두 기술은 현대 CPU 성능의 핵심 쌍입니다.

## 주의사항 및 팁

**1. 조건부 이동(cmov) 활용**  
컴파일러의 `-O2` 이상 최적화는 단순한 `if-else`를 `cmov`(conditional move) 명령어로 변환하여 분기 자체를 없앨 수 있습니다. GCC/Clang이 이 최적화를 적용하는지 어셈블리를 확인하세요:
```bash
gcc -O2 -S -o out.s source.c && grep cmov out.s
```

**2. 루프 내 분기 최소화**  
루프에서의 분기는 반복적으로 예측되어야 합니다. 루프 내 조건부 로직을 가능하면 루프 전/후로 이동하거나, 마스크를 사용한 무분기 코드로 변환하세요.

**3. Spectre 완화의 성능 비용**  
KPTI, Retpoline, IBRS 등의 Spectre 완화책은 syscall 비용을 2~5배까지 늘릴 수 있습니다. I/O 집약적 서버에서는 성능 프로파일링 후 필요에 따라 완화책 수준을 조정하세요.

**4. 성능 카운터로 예측 정확도 측정**  
Linux `perf` 도구로 분기 예측 실패율을 직접 측정할 수 있습니다:
```bash
perf stat -e branches,branch-misses ./your_program
# branch-misses / branches × 100 = 오예측률(%)
# 1% 이상이면 최적화 여지가 있음
```

**5. 파이프라인 스테이지 수와 클럭 속도의 트레이드오프**  
스테이지가 많을수록 클럭 주파수를 높일 수 있지만(각 스테이지 작업이 줄어듦), 분기 패널티도 커집니다. Intel Prescott(2004)의 31단계 파이프라인은 고클럭을 달성했지만 분기 패널티 증가로 IPC가 하락했습니다. 이후 Intel은 Core 아키텍처에서 단계 수를 줄이고 IPC를 높이는 방향으로 전환했습니다.

## 참고 자료
- [Branch Prediction and its Impact on Performance — Packt](https://www.packtpub.com/en-bg/product/c-high-performance-for-financial-systems-6-9781805124528/chapter/chapter-6-low-latency-programming-strategies-and-techniques/branch-prediction-and-its-impact-on-performance)
- [Out-of-Order Execution, Speculative Execution, and Branch Prediction — CS Wiki FAQ](https://faq.computersciencewiki.org/index.php/home/article/out-of-order-execution-speculative-execution-and-branch-prediction)
- [Branch Prediction — Startup House Glossary](https://startup-house.com/glossary/branch-prediction)
- [Branch Prediction: All Processors — DEV Community](https://dev.to/dima853/branch-prediction-all-processors-2bk6)
