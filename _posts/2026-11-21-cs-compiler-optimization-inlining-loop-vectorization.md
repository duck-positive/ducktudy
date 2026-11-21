---
layout: post
title: "컴파일러 최적화 기법 심화: 인라이닝·루프 변환·자동 벡터화의 원리와 실전"
date: 2026-11-21
categories: [cs, computer-science]
tags: [compiler-optimization, inlining, loop-unrolling, vectorization, LLVM, GCC, 컴파일러최적화]
---

## 컴파일러 최적화란 무엇인가

현대 컴파일러는 단순히 소스 코드를 기계어로 번역하는 것을 넘어서, 개발자가 작성한 코드보다 **훨씬 빠르게 실행되는 기계어**를 생성합니다. LLVM/Clang, GCC, rustc 등의 최적화 컴파일러는 수백 가지의 최적화 패스를 통해 이를 달성합니다.

최적화는 크게 세 수준으로 나뉩니다:

1. **로컬 최적화(Local)**: 하나의 기본 블록(basic block) 안에서만 적용. 예: 상수 접기(constant folding)
2. **전역 최적화(Global)**: 함수 전체 범위. 예: 불필요한 코드 제거(dead code elimination)
3. **프로시저 간 최적화(Interprocedural)**: 함수 경계를 넘어서. 예: 인라이닝(inlining)

GCC에서는 `-O0`(최적화 없음)부터 `-O3`(공격적 최적화), `-Os`(크기 최적화)까지 레벨을 선택할 수 있고, Clang/LLVM도 동일한 플래그를 사용합니다. 이 글에서는 그 중 성능에 가장 큰 영향을 미치는 세 가지를 깊이 살펴봅니다: **함수 인라이닝**, **루프 변환(언롤링, 타일링)**, **자동 벡터화**.

---

## 1. 함수 인라이닝(Function Inlining)

### 개념

인라이닝은 함수 호출 지점에 함수 본문을 직접 복사해 넣는 최적화입니다. 함수 호출에는 비용이 있습니다:
- 콜 스택 프레임 생성/소멸 (스택 포인터 조정)
- 레지스터 저장/복원 (caller-saved/callee-saved 규약)
- 분기 명령어(CALL, RET)
- CPU 분기 예측 미스

인라이닝을 하면 이 오버헤드가 사라지고, 더 중요하게는 **호출자와 피호출자 컨텍스트를 함께 최적화**할 수 있습니다. 예를 들어, 인라이닝 후 상수 인수가 드러나면 추가로 상수 전파(constant propagation)와 불필요한 코드 제거가 연쇄적으로 일어납니다.

### 인라이닝 전/후 어셈블리 비교

다음 C 코드를 컴파일할 때 어떤 일이 일어나는지 살펴봅시다:

```c
// 인라이닝 대상 함수
static inline int square(int x) {
    return x * x;
}

int compute(int n) {
    int result = 0;
    for (int i = 0; i < n; i++) {
        result += square(i);  // 함수 호출 → 인라이닝
    }
    return result;
}
```

`-O0`에서는 `square`가 실제 함수 호출로 컴파일됩니다:
```asm
; -O0 (최적화 없음)
compute:
    ...
    call square    ; 함수 호출 (비용 발생)
    add eax, ...
    ...
```

`-O2` 이상에서는 인라이닝 후 추가 최적화까지 적용됩니다:
```asm
; -O2 이상 (인라이닝 적용)
compute:
    xor eax, eax     ; result = 0
    xor ecx, ecx     ; i = 0
.loop:
    mov edx, ecx
    imul edx, ecx    ; i * i (square 본문이 직접 삽입됨)
    add eax, edx     ; result += i*i
    inc ecx          ; i++
    cmp ecx, edi     ; i < n
    jl .loop
    ret
```

### 인라이닝 휴리스틱

컴파일러는 모든 함수를 무조건 인라이닝하지 않습니다. 함수가 너무 크면 코드 크기(code bloat)가 증가해 **명령어 캐시(I-cache) 미스**가 오히려 늘어납니다. LLVM의 인라이닝 결정 기준에는 다음이 포함됩니다:

- 함수 크기 (IR 명령어 수)
- 호출 빈도 (프로파일 가이드 최적화 시)
- 재귀 여부 (재귀 함수는 인라이닝 불가)
- `__attribute__((always_inline))` 또는 `__attribute__((noinline))` 힌트

---

## 2. 루프 변환: 언롤링과 타일링

루프는 프로그램 실행 시간의 대부분을 차지합니다. 컴파일러는 루프를 여러 방식으로 변환해 성능을 높입니다.

### 루프 언롤링(Loop Unrolling)

루프 언롤링은 한 번의 반복에서 여러 번의 작업을 수행하도록 루프 본문을 복제해, **루프 제어 오버헤드**(증분, 비교, 분기)를 줄이고 **명령어 수준 병렬성(ILP)**을 높입니다.

```c
// 원본 코드
void add_arrays(float *a, float *b, float *c, int n) {
    for (int i = 0; i < n; i++) {
        c[i] = a[i] + b[i];
    }
}

// 4배 언롤 (컴파일러가 자동으로 생성)
void add_arrays_unrolled(float *a, float *b, float *c, int n) {
    int i = 0;
    int limit = n - (n % 4);  // 4의 배수 경계
    for (; i < limit; i += 4) {
        c[i]     = a[i]     + b[i];
        c[i + 1] = a[i + 1] + b[i + 1];
        c[i + 2] = a[i + 2] + b[i + 2];
        c[i + 3] = a[i + 3] + b[i + 3];
    }
    // 나머지 처리
    for (; i < n; i++) {
        c[i] = a[i] + b[i];
    }
}
```

현대 OOO(Out-of-Order) CPU는 연속된 독립적인 명령어들을 병렬 실행할 수 있습니다. 언롤링으로 연속된 독립 연산을 나열하면 CPU가 더 많은 실행 유닛을 활용할 수 있습니다.

### 루프 타일링(Loop Tiling / Cache Blocking)

행렬 연산에서 가장 중요한 최적화 중 하나입니다. 타일링은 캐시 용량에 맞는 "타일" 단위로 데이터를 처리해 **캐시 미스**를 줄입니다.

```c
// 순진한 행렬 곱셈 (캐시 비친화적)
void naive_matmul(float A[][N], float B[][N], float C[][N]) {
    for (int i = 0; i < N; i++)
        for (int j = 0; j < N; j++)
            for (int k = 0; k < N; k++)
                C[i][j] += A[i][k] * B[k][j];  // B[k][j]: 열 방향 접근 → 캐시 미스
}

// 타일링 적용 행렬 곱셈 (캐시 친화적)
#define TILE 64  // L1 캐시에 맞는 타일 크기

void tiled_matmul(float A[][N], float B[][N], float C[][N]) {
    for (int i = 0; i < N; i += TILE)
        for (int j = 0; j < N; j += TILE)
            for (int k = 0; k < N; k += TILE)
                // 타일 내부 계산 — 타일 전체가 캐시 안에 존재
                for (int ii = i; ii < i + TILE && ii < N; ii++)
                    for (int jj = j; jj < j + TILE && jj < N; jj++)
                        for (int kk = k; kk < k + TILE && kk < N; kk++)
                            C[ii][jj] += A[ii][kk] * B[kk][jj];
}
```

타일 크기를 L1/L2 캐시에 맞게 조정하면 행렬 곱셈 성능이 10배 이상 향상될 수 있습니다. OpenBLAS, MKL 같은 최적화 수학 라이브러리는 모두 이 기법을 집중적으로 사용합니다.

---

## 3. 자동 벡터화(Auto-Vectorization)

### SIMD와 벡터 연산

현대 CPU는 **SIMD(Single Instruction Multiple Data)** 명령어를 통해 한 번의 명령으로 여러 데이터를 동시에 처리합니다:

- **SSE2**: 128비트 레지스터 → 4개의 float 동시 처리
- **AVX2**: 256비트 레지스터 → 8개의 float 동시 처리
- **AVX-512**: 512비트 레지스터 → 16개의 float 동시 처리

자동 벡터화는 컴파일러가 스칼라 루프를 SIMD 명령어를 사용하는 벡터 루프로 자동 변환하는 기능입니다.

### 자동 벡터화 조건

LLVM/GCC가 루프를 자동 벡터화하려면 다음 조건이 충족되어야 합니다:

1. **루프 경계가 컴파일 타임에 알려지거나 추론 가능해야 함**
2. **루프 내 데이터 의존성이 없어야 함** (루프 반복 간 의존성 = vectorization barrier)
3. **포인터 별칭(aliasing)이 없어야 함** (`__restrict__` 힌트로 알려줄 수 있음)

```c
// 자동 벡터화 가능한 루프
void scale(float *out, const float *in, float factor, int n) {
    for (int i = 0; i < n; i++) {
        out[i] = in[i] * factor;  // 반복 간 의존성 없음
    }
}

// 자동 벡터화 불가 (루프 간 의존성)
void prefix_sum(float *a, int n) {
    for (int i = 1; i < n; i++) {
        a[i] += a[i - 1];  // a[i]가 a[i-1]에 의존 → 의존성 체인
    }
}
```

### 벡터화 진단 및 힌트

```bash
# GCC로 벡터화 여부 확인
gcc -O2 -march=native -fopt-info-vec-optimized -fopt-info-vec-missed code.c

# Clang으로 최적화 보고서 출력
clang -O2 -Rpass=loop-vectorize \
      -Rpass-missed=loop-vectorize \
      -Rpass-analysis=loop-vectorize code.c
```

포인터 별칭 힌트로 벡터화를 강제로 활성화:

```c
// restrict 없음: 컴파일러가 out과 in이 겹칠 수 있다고 가정 → 벡터화 포기
void add(float *out, float *in1, float *in2, int n) {
    for (int i = 0; i < n; i++)
        out[i] = in1[i] + in2[i];
}

// __restrict__: 포인터가 겹치지 않음을 보증 → 벡터화 허용
void add_opt(float *__restrict__ out,
             const float *__restrict__ in1,
             const float *__restrict__ in2, int n) {
    for (int i = 0; i < n; i++)
        out[i] = in1[i] + in2[i];
}
```

### 벡터화 성능 측정 예시

```python
import numpy as np
import time

n = 100_000_000
a = np.random.rand(n).astype(np.float32)
b = np.random.rand(n).astype(np.float32)

# NumPy는 내부적으로 SIMD(AVX2) 사용
t0 = time.perf_counter()
c = a + b
t1 = time.perf_counter()
print(f"NumPy SIMD 덧셈: {(t1 - t0) * 1000:.1f}ms")

# Python 루프 (스칼라)
a_list = a.tolist()
b_list = b.tolist()
t2 = time.perf_counter()
c_list = [a_list[i] + b_list[i] for i in range(n)]
t3 = time.perf_counter()
print(f"Python 루프 스칼라 덧셈: {(t3 - t2) * 1000:.1f}ms")
print(f"속도 비율: {(t3 - t2) / (t1 - t0):.0f}x 차이")
```

일반적으로 SIMD NumPy 연산은 순수 Python 루프보다 100배 이상 빠릅니다.

---

## 최적화 플래그와 PGO(Profile-Guided Optimization)

```bash
# 단계 1: 프로파일 수집용 빌드
gcc -O2 -fprofile-generate program.c -o program_pgo

# 단계 2: 대표 입력으로 실행 → profiling data 수집
./program_pgo < representative_input.txt

# 단계 3: 프로파일 데이터를 활용한 최적화 빌드
gcc -O2 -fprofile-use -fprofile-correction program.c -o program_fast
```

PGO는 실제 실행 빈도를 기반으로 인라이닝 결정, 루프 언롤 횟수, 분기 예측 힌트를 최적화해 `O3` 대비 추가로 10~30% 성능 향상을 달성합니다. Chrome, Firefox, MySQL 모두 PGO를 프로덕션 빌드에 사용합니다.

---

## 주의사항과 실전 팁

**1. `-O3`이 항상 빠르지는 않다**: `-O3`의 공격적인 인라이닝과 언롤링은 코드 크기를 크게 늘려 명령어 캐시 미스를 유발할 수 있습니다. 작은 코어에서 실행되는 임베디드 프로그램이나 캐시 압박이 심한 서버 환경에서는 `-O2`가 더 빠를 수 있습니다.

**2. 부동소수점 최적화 주의**: `-ffast-math`는 IEEE 754 규정을 완화해 더 빠른 부동소수점 연산을 가능하게 하지만, 결합 법칙 가정으로 인해 결과값이 달라질 수 있습니다. 금융 계산에는 절대 사용하지 마세요.

**3. 디버깅 어려움**: 최적화된 코드는 소스 코드와 어셈블리 간 대응이 흐트러져 디버거에서 변수 값이 올바르게 표시되지 않거나 중단점이 정확히 작동하지 않을 수 있습니다. 디버깅 시에는 `-O0 -g`를 사용하세요.

**4. Compiler Explorer(Godbolt) 활용**: [godbolt.org](https://godbolt.org)에서 C/C++/Rust 코드를 입력하고 다양한 컴파일러와 최적화 레벨에서 생성된 어셈블리를 실시간으로 비교할 수 있습니다. 최적화를 이해하는 최고의 도구입니다.

**5. 벡터화 방해 패턴 피하기**: 함수 포인터 호출, 간접 메모리 접근(포인터의 포인터), `volatile` 변수, 루프 내 함수 호출(인라이닝 안 되는 경우)은 모두 자동 벡터화를 막습니다.

---

## 참고 자료

- [Auto-Vectorization in LLVM — 공식 문서](https://llvm.org/docs/Vectorizers.html)
- [GCC Optimize Options — 공식 문서](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html)
- [Compiler Explorer (Godbolt)](https://godbolt.org)
- [What Every Programmer Should Know About Memory — Ulrich Drepper](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)
