---
layout: post
title: "폴리헤드럴 컴파일러 모델 완전 정복: 루프 타일링·퓨전·교환으로 캐시 효율을 극대화하는 최적화 기법"
date: 2026-11-16
categories: [cs, computer-science]
tags: [compiler, loop-optimization, polyhedral-model, loop-tiling, llvm, polly, cache-optimization, hpc]
---

## 개념 설명

현대 고성능 컴퓨팅(HPC) 애플리케이션의 병목은 대부분 **루프(loop)**에 있습니다. 행렬 곱셈, 과학 계산, 딥러닝 연산 등에서 중첩 루프(nested loop)의 효율은 전체 프로그램 성능을 좌우합니다. 이를 자동으로 최적화하기 위해 1980년대부터 연구된 강력한 수학적 프레임워크가 바로 **폴리헤드럴 모델(Polyhedral Model)**입니다.

폴리헤드럴 모델은 루프 반복(iteration)을 **정수 선형 대수학(Integer Linear Algebra)**의 점(point)으로 표현합니다. n중 루프의 각 반복은 n차원 정수 격자 공간의 한 점에 해당하고, 루프 전체는 그 공간의 **볼록 다면체(convex polytope)**가 됩니다. 의존성은 점들 사이의 관계(화살표)로, 변환은 선형 맵(linear map)으로 표현됩니다.

이 수학적 표현이 가능해지면, 컴파일러는 다음과 같은 질문에 자동으로 답할 수 있습니다.

- **"이 루프를 타일링(tiling)하면 캐시 미스가 줄어드는가?"**
- **"이 두 루프를 하나로 합쳐도(fusion) 결과가 달라지지 않는가?"**
- **"루프의 순서를 바꿔도(interchange) 올바른 결과가 나오는가?"**
- **"이 루프를 병렬로 실행할 수 있는가?"**

---

## 왜 필요한가

### 하드웨어와 소프트웨어의 간극

현대 CPU의 메모리 계층 구조에서 L1 캐시(~4ns)와 메인 메모리(~100ns) 사이의 속도 차이는 25배 이상입니다. 중첩 루프가 메모리를 비효율적으로 접근하면, 계산 자체보다 메모리 대기 시간이 훨씬 길어지는 **메모리 바운드(Memory-Bound)** 상황이 됩니다.

### 순진한(Naive) 행렬 곱셈의 문제

다음은 가장 단순한 행렬 곱셈 코드입니다.

```c
// N x N 행렬 곱셈 (순진한 구현)
for (int i = 0; i < N; i++)
    for (int j = 0; j < N; j++)
        for (int k = 0; k < N; k++)
            C[i][j] += A[i][k] * B[k][j];
```

이 코드에서 `B[k][j]`는 j가 가장 안쪽 루프에서 느리게 변하므로(C 언어 행 우선 배치), `B`의 접근 패턴이 열 단위(column-major)가 되어 캐시 미스가 폭발적으로 증가합니다. N=1024일 때, 순진한 구현과 최적화된 구현의 성능 차이는 10배 이상 발생합니다.

---

## 핵심 개념: 반복 도메인과 의존성

### 반복 도메인(Iteration Domain)

n중 루프의 각 반복은 루프 카운터의 벡터 **i = (i₁, i₂, ..., iₙ)**로 표현됩니다. 반복 도메인 D는 이 벡터들이 만족하는 선형 부등식들의 집합입니다.

```
for (int i = 0; i < N; i++)
    for (int j = 0; j < N; j++)
        → 반복 도메인: {(i, j) | 0 ≤ i < N, 0 ≤ j < N}
```

이는 2차원 격자에서 N×N 크기의 정사각형 다면체입니다.

### 의존성 분석(Dependence Analysis)

의존성은 한 반복이 다른 반복의 결과에 의존하는 관계입니다. 의존성이 있으면 두 반복의 순서를 바꿀 수 없습니다.

```
의존성 유형:
- RAW (Read After Write): 쓰기 후 읽기 — 진짜 의존성
- WAR (Write After Read): 읽기 후 쓰기 — 반의존성
- WAW (Write After Write): 쓰기 후 쓰기 — 출력 의존성
```

의존 벡터(dependence vector) **d = i' - i**가 양수 방향이면 변환이 의존성을 위반하지 않습니다.

---

## 주요 루프 변환

### 1. 루프 타일링(Loop Tiling / Loop Blocking)

루프 반복 공간을 블록(타일)으로 나누어, 블록 단위로 모든 계산을 수행합니다. 블록 크기를 캐시 크기에 맞추면 캐시 재사용성이 극대화됩니다.

```c
// 타일 크기 T
#define T 32

// 타일링된 행렬 곱셈
for (int ii = 0; ii < N; ii += T)
    for (int jj = 0; jj < N; jj += T)
        for (int kk = 0; kk < N; kk += T)
            // 타일 내 계산
            for (int i = ii; i < min(ii + T, N); i++)
                for (int j = jj; j < min(jj + T, N); j++)
                    for (int k = kk; k < min(kk + T, N); k++)
                        C[i][j] += A[i][k] * B[k][j];
```

타일링의 효과: T=32, N=1024일 때, 타일 내에서 `A`의 T×T 블록과 `B`의 T×T 블록이 각각 L1 캐시에 적합하게 되어 캐시 미스가 O(N³/T)로 감소합니다.

### 2. 루프 교환(Loop Interchange)

루프의 중첩 순서를 변경합니다. 가장 안쪽 루프가 연속된 메모리를 접근하도록 순서를 조정합니다.

```c
// 원래: i-j-k 순서 (B에 열 접근 → 캐시 비효율)
for i, for j, for k: C[i][j] += A[i][k] * B[k][j]

// 교환 후: i-k-j 순서 (B에 행 접근 → 캐시 친화적)
for (int i = 0; i < N; i++)
    for (int k = 0; k < N; k++)
        for (int j = 0; j < N; j++)
            C[i][j] += A[i][k] * B[k][j];
// B[k][j]가 j에 대해 연속 접근 → 완벽한 스트라이드-1
```

의존성 검사: `C[i][j]`에 대한 의존성이 i-k-j 순서에서도 위반되지 않는지 폴리헤드럴 모델로 자동 검증합니다.

### 3. 루프 퓨전(Loop Fusion)과 피션(Loop Fission)

**루프 퓨전**: 두 개의 루프를 하나로 합칩니다. 임시 배열의 생성을 줄이고 캐시 재사용성을 높입니다.

```c
// 퓨전 전
for (int i = 0; i < N; i++) B[i] = A[i] * 2;
for (int i = 0; i < N; i++) C[i] = B[i] + 1;

// 퓨전 후 (B에 대한 중간 배열 접근 제거)
for (int i = 0; i < N; i++) {
    B[i] = A[i] * 2;
    C[i] = B[i] + 1;
}
```

**루프 피션**: 하나의 루프를 여러 개로 분리합니다. 각 루프가 더 작은 작업 집합을 가지도록 하여 캐시 적합성을 개선합니다.

---

## 실제 구현: LLVM Polly와 수동 분석

### 예제 1: 의존성 분석 도구 구현 (Python)

```python
from itertools import product

def analyze_dependencies(reads, writes, loop_bounds):
    """
    단순화된 의존성 분석 도구
    reads: [(배열명, 인덱스 함수), ...]
    writes: [(배열명, 인덱스 함수), ...]
    loop_bounds: [(시작, 끝), ...]
    """
    iterations = list(product(*[range(s, e) for s, e in loop_bounds]))
    
    # 각 반복에서 접근하는 메모리 위치를 기록
    write_locations = {}  # (배열, 위치) → 반복 인덱스
    dependencies = []
    
    for idx, it in enumerate(iterations):
        # 쓰기 의존성 기록
        for arr, f in writes:
            loc = (arr, f(*it))
            if loc in write_locations:
                prev_idx = write_locations[loc]
                if prev_idx != idx:
                    dependencies.append({
                        'type': 'WAW',
                        'from': iterations[prev_idx],
                        'to': it,
                        'location': loc
                    })
            write_locations[loc] = idx
        
        # 읽기 의존성 기록 (RAW 탐지)
        for arr, f in reads:
            loc = (arr, f(*it))
            if loc in write_locations:
                prev_idx = write_locations[loc]
                if prev_idx != idx and iterations[prev_idx] != it:
                    dep_vec = tuple(a - b for a, b in zip(it, iterations[prev_idx]))
                    dependencies.append({
                        'type': 'RAW',
                        'from': iterations[prev_idx],
                        'to': it,
                        'dep_vector': dep_vec,
                        'location': loc
                    })
    
    return dependencies

# 예시: 2중 루프 의존성 분석
# for i in range(3):
#     for j in range(3):
#         C[i][j] = A[i][j] + B[i-1][j]  (i ≥ 1)

reads  = [('A', lambda i, j: (i, j)),
           ('B', lambda i, j: (i-1, j) if i > 0 else None)]
writes = [('C', lambda i, j: (i, j))]

# 실제 의존성 분석 (단순화)
print("의존성 분석 예시:")
print("C[i][j] = A[i][j] + B[i-1][j]")
print("→ C[i][j]는 B[i-1][j]를 읽으므로, 이전 행의 B 값에 의존")
print("→ 의존 벡터: (1, 0) — i 방향으로 1 차이")
print("→ j 방향으로의 루프 교환은 안전. i 방향으로는 불가.")

# 타일링 가능 여부 체크 (단순화)
def can_tile(dep_vectors, tile_sizes):
    """타일링이 의존성을 위반하지 않는지 확인"""
    for dep in dep_vectors:
        for dim, (d, t) in enumerate(zip(dep, tile_sizes)):
            if abs(d) >= t:
                print(f"  차원 {dim}: 의존 거리 {d} ≥ 타일 크기 {t} → 타일링 불가능")
                return False
    return True

# 의존 거리 (1,0)으로 타일 크기 (32,32) 체크
dep_vecs = [(1, 0)]
tile = (32, 32)
result = can_tile(dep_vecs, tile)
print(f"\n타일 크기 {tile}로 타일링 가능: {result}")
```

### 예제 2: 행렬 곱셈 성능 비교 (Python + NumPy)

```python
import time
import numpy as np

def naive_matmul(A, B, N):
    """순진한 행렬 곱셈 (캐시 비친화적)"""
    C = np.zeros((N, N))
    for i in range(N):
        for j in range(N):
            for k in range(N):
                C[i][j] += A[i][k] * B[k][j]
    return C

def interchange_matmul(A, B, N):
    """루프 교환 후 행렬 곱셈 (i-k-j 순서, 캐시 친화적)"""
    C = np.zeros((N, N))
    for i in range(N):
        for k in range(N):
            # A[i][k]는 단일 값이므로 레지스터에 캐싱
            a_ik = A[i][k]
            for j in range(N):
                # B[k][j]는 j 증가에 따라 연속 메모리 접근
                C[i][j] += a_ik * B[k][j]
    return C

def tiled_matmul(A, B, N, T=32):
    """타일링된 행렬 곱셈 (캐시 최적화)"""
    C = np.zeros((N, N))
    for ii in range(0, N, T):
        for jj in range(0, N, T):
            for kk in range(0, N, T):
                i_end = min(ii + T, N)
                j_end = min(jj + T, N)
                k_end = min(kk + T, N)
                # 타일 내 계산 — NumPy 슬라이스로 최적화
                C[ii:i_end, jj:j_end] += (
                    A[ii:i_end, kk:k_end] @ B[kk:k_end, jj:j_end]
                )
    return C

# 성능 측정
N = 128
A = np.random.rand(N, N)
B = np.random.rand(N, N)
expected = A @ B

# 타일링 버전 테스트
t0 = time.time()
C_tiled = tiled_matmul(A, B, N, T=16)
t1 = time.time()
print(f"타일링 행렬 곱셈 (N={N}, T=16): {t1-t0:.4f}초")
print(f"결과 정확도 오차: {np.max(np.abs(C_tiled - expected)):.2e}")

# LLVM Polly 활성화 방법 (C 컴파일 시)
print("""
LLVM Polly 활성화 방법:
  clang -O3 -mllvm -polly -mllvm -polly-vectorizer=stripmine matmul.c

주요 Polly 옵션:
  -polly-parallel        자동 병렬화 (OpenMP 생성)
  -polly-vectorizer      SIMD 벡터화
  -polly-opt-fusion      루프 퓨전 최적화
  -polly-report          최적화 레포트 출력
""")
```

---

## LLVM Polly: 실전 폴리헤드럴 최적화

LLVM의 **Polly** 서브프로젝트는 폴리헤드럴 모델을 실제 컴파일러에 구현한 것입니다. Polly는 LLVM IR을 입력받아 다음을 자동 수행합니다.

**1단계 — 스코프 감지(SCoP Detection)**: 폴리헤드럴 모델로 표현 가능한 루프 영역(Static Control Part, SCoP)을 식별합니다.

**2단계 — 의존성 분석**: isl(Integer Set Library)을 사용하여 정확한 의존성을 계산합니다.

**3단계 — 스케줄 최적화**: Pluto 알고리즘을 사용하여 지역성(locality)과 병렬성을 최대화하는 새 스케줄을 계산합니다.

**4단계 — 코드 생성**: 최적화된 스케줄에 따라 새 루프 코드를 생성합니다. 타일링, 벡터화, 병렬화가 자동으로 적용됩니다.

```
폴리헤드럴 최적화 파이프라인:

원본 C 코드
    ↓ Clang
LLVM IR (-O1 이상)
    ↓ SCoP 감지
정수 다면체 표현 (isl 형식)
    ↓ 스케줄 최적화 (Pluto/ISS)
최적화된 스케줄
    ↓ CLooG 코드 생성
최적화된 LLVM IR (타일링/벡터화 적용)
    ↓ 백엔드
최적화된 기계어
```

---

## 주의사항과 팁

**1. SCoP 조건의 제약**: 폴리헤드럴 모델은 **정적 제어(Static Control)** 구조에서만 작동합니다. 루프 경계나 배열 인덱스가 반복 변수의 아핀(affine) 함수여야 하고, 루프 내에서 함수 호출이나 포인터 별칭(aliasing)이 없어야 합니다.

**2. 타일 크기 선택**: 타일 크기는 L1 캐시 크기에 따라 결정합니다. 타일 하나에 필요한 데이터가 L1 캐시에 들어올 때 효과가 최대입니다. 일반적으로 32~64가 많이 사용됩니다.

**3. 의존성 방향 보존**: 모든 루프 변환은 의존성 벡터의 어휘적 양수성(lexicographically positive)을 보존해야 합니다. Polly의 isl 라이브러리가 이를 자동으로 검증합니다.

**4. 병렬화와 타일링의 조합**: 외부 타일 루프(outer tile loops)를 OpenMP로 병렬화하고, 내부 루프(inner loops)를 SIMD로 벡터화하는 2단계 최적화가 최대 성능을 냅니다.

**5. 프로파일링 필수**: 폴리헤드럴 변환이 항상 성능을 개선하지는 않습니다. 반드시 `perf`, VTune, Instruments 등으로 캐시 미스, 사이클 수를 프로파일링하여 효과를 확인하세요.

---

## 참고 자료

- [LLVM/Polly GitHub — 폴리헤드럴 최적화 프레임워크](https://github.com/llvm/llvm-project/tree/main/polly)
- [LLVM Project — 컴파일러 최적화 패스 모음](https://github.com/llvm/llvm-project)
