---
layout: post
title: "OpenMP 완전 정복: Fork-Join 모델과 공유 메모리 병렬 프로그래밍의 모든 것"
date: 2026-10-04
categories: [cs, computer-science]
tags: [parallel-computing, OpenMP, multithreading, performance, HPC, shared-memory, C, C++]
---

## 개요

현대 CPU는 하나의 칩에 수십 개의 코어를 탑재하지만, 대부분의 프로그램은 이 중 하나의 코어만 사용한다. **OpenMP(Open Multi-Processing)**는 공유 메모리 병렬 프로그래밍을 위한 산업 표준 API로, C, C++, Fortran에서 `#pragma omp` 지시자(directive)를 추가하는 것만으로 순차 코드를 병렬 코드로 변환할 수 있다.

Intel, AMD, ARM, IBM 등 주요 하드웨어 벤더와 GCC, Clang, MSVC 등 주요 컴파일러가 모두 지원하며, 과학 계산(HPC), 이미지 처리, 데이터 분석, 시뮬레이션 분야에서 광범위하게 활용된다.

---

## 왜 OpenMP인가

### 스레드 API와의 비교

`pthreads`나 `std::thread`를 사용하면 스레드 생성, 동기화, 작업 분배를 모두 수동으로 구현해야 한다. 1000개 원소의 배열 합산을 4 스레드로 병렬화하려면 수십 줄의 boilerplate 코드가 필요하다.

OpenMP로는 단 두 줄이다:

```c
#pragma omp parallel for reduction(+:sum)
for (int i = 0; i < N; i++) sum += array[i];
```

### 점진적 병렬화

순차 코드에 `#pragma omp` 지시자를 추가하는 방식이기 때문에, 전체 코드를 한 번에 재작성하지 않고 **병목 구간만 선택적으로 병렬화**할 수 있다. 이는 유지보수성과 안정성 면에서 큰 장점이다.

---

## Fork-Join 실행 모델

OpenMP의 실행 모델은 **Fork-Join 패러다임**을 따른다.

```
마스터 스레드 (Thread 0)
      │
      │ #pragma omp parallel {
      │
  Fork ├──────────────────────────────┐
      │                               │
  Thread 0    Thread 1    Thread 2   Thread 3
      │           │           │          │
      │    <병렬 구간 실행>    │          │
      │           │           │          │
  Join └──────────────────────────────┘
      │
      │ } // 암묵적 배리어
      │
  계속 실행
```

`parallel` 블록에 진입하면 마스터 스레드가 지정된 수의 워커 스레드를 생성(Fork)하고, 블록이 끝나면 모든 스레드가 동기화된 뒤 마스터만 계속 실행된다(Join).

---

## 코드 예제 1: 기본 구조와 데이터 공유

```c
#include <stdio.h>
#include <omp.h>

int main() {
    int n = 8;
    int shared_var = 0;    // 공유 변수 (모든 스레드가 동일한 주소 참조)

    // 스레드 수 지정 (환경변수 OMP_NUM_THREADS로도 설정 가능)
    omp_set_num_threads(4);

    #pragma omp parallel
    {
        int tid = omp_get_thread_num();      // 스레드 고유 ID (0 ~ N-1)
        int nthreads = omp_get_num_threads(); // 총 스레드 수
        int private_var = tid * 10;           // 각 스레드의 스택에 할당 (자동 private)

        // 단일 스레드만 실행 (마스터 스레드)
        #pragma omp master
        {
            printf("[Master] 총 스레드 수: %d\n", nthreads);
        }

        // 암묵적 배리어 없음 (nowait 아니어도 master 블록 후 배리어 없음)
        // 모든 스레드가 출력
        printf("[Thread %d] private_var = %d\n", tid, private_var);

        // 임계 구간: 한 번에 하나의 스레드만 진입
        #pragma omp critical
        {
            shared_var += tid;  // Race condition 방지
        }
    } // ← 암묵적 배리어: 모든 스레드가 여기서 동기화

    printf("shared_var (합산 결과) = %d\n", shared_var);
    // 0+1+2+3 = 6
    return 0;
}
```

컴파일 및 실행:
```bash
gcc -fopenmp -O2 -o basic_omp basic_omp.c
./basic_omp

# 출력 (순서는 비결정적):
# [Master] 총 스레드 수: 4
# [Thread 0] private_var = 0
# [Thread 2] private_var = 20
# [Thread 1] private_var = 10
# [Thread 3] private_var = 30
# shared_var (합산 결과) = 6
```

### 데이터 공유 속성

OpenMP에서 변수는 **공유(shared)** 또는 **전용(private)**으로 분류된다.

| 절 | 의미 | 사용 시기 |
|---|---|---|
| `shared(x)` | 모든 스레드가 동일한 x를 참조 | 읽기 전용 배열, 결과 저장 |
| `private(x)` | 각 스레드가 독립적인 x를 보유 (초기화 안 됨) | 루프 인덱스, 임시 변수 |
| `firstprivate(x)` | private이지만 마스터 값으로 초기화 | 초기값이 필요한 누산기 |
| `lastprivate(x)` | private이지만 마지막 반복값을 마스터로 복사 | 루프 후 마지막 값 사용 |
| `reduction(op:x)` | private 복사본 생성 후 연산으로 합산 | 합계, 최댓값 등 집계 |

---

## 코드 예제 2: 행렬 곱셈 병렬화와 성능 측정

행렬 곱셈은 O(N³)의 계산 복잡도를 가지며 병렬화 효과가 뚜렷한 대표적인 벤치마크다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <omp.h>

#define N 1024

void matmul_sequential(double A[N][N], double B[N][N], double C[N][N]) {
    for (int i = 0; i < N; i++)
        for (int j = 0; j < N; j++) {
            double sum = 0.0;
            for (int k = 0; k < N; k++)
                sum += A[i][k] * B[k][j];
            C[i][j] = sum;
        }
}

void matmul_parallel(double A[N][N], double B[N][N], double C[N][N]) {
    // schedule(static): 각 스레드에 연속된 청크를 균등 분배
    // collapse(2): 중첩 루프 2개를 하나의 병렬 루프로 합침
    #pragma omp parallel for schedule(static) collapse(2)
    for (int i = 0; i < N; i++) {
        for (int j = 0; j < N; j++) {
            double sum = 0.0;
            for (int k = 0; k < N; k++)
                sum += A[i][k] * B[k][j];
            C[i][j] = sum;
        }
    }
}

// 캐시 친화적 버전: B 전치 후 행렬 곱
// B[k][j] 접근은 캐시 미스 유발 → B를 전치하면 연속 메모리 접근
void matmul_parallel_transposed(double A[N][N], double B[N][N], 
                                  double BT[N][N], double C[N][N]) {
    // B 전치 (병렬화)
    #pragma omp parallel for collapse(2)
    for (int i = 0; i < N; i++)
        for (int j = 0; j < N; j++)
            BT[j][i] = B[i][j];

    // 전치된 B를 사용한 행렬 곱 (행×행 접근으로 캐시 효율 향상)
    #pragma omp parallel for schedule(dynamic, 16)
    for (int i = 0; i < N; i++) {
        for (int j = 0; j < N; j++) {
            double sum = 0.0;
            for (int k = 0; k < N; k++)
                sum += A[i][k] * BT[j][k];  // BT[j][k]는 연속 접근
            C[i][j] = sum;
        }
    }
}

int main() {
    static double A[N][N], B[N][N], BT[N][N], C[N][N];
    
    // 행렬 초기화
    srand(42);
    for (int i = 0; i < N; i++)
        for (int j = 0; j < N; j++) {
            A[i][j] = (double)rand() / RAND_MAX;
            B[i][j] = (double)rand() / RAND_MAX;
        }

    double t_start, t_end;

    // 순차 실행
    t_start = omp_get_wtime();
    matmul_sequential(A, B, C);
    t_end = omp_get_wtime();
    printf("순차 실행: %.3f초\n", t_end - t_start);

    // 병렬 실행 (4 스레드)
    omp_set_num_threads(4);
    t_start = omp_get_wtime();
    matmul_parallel(A, B, C);
    t_end = omp_get_wtime();
    printf("병렬 실행 (4코어): %.3f초\n", t_end - t_start);

    // 캐시 최적화 병렬 실행
    t_start = omp_get_wtime();
    matmul_parallel_transposed(A, B, BT, C);
    t_end = omp_get_wtime();
    printf("캐시 최적화 병렬 (4코어): %.3f초\n", t_end - t_start);

    return 0;
}
```

전형적인 성능 결과 (1024×1024, Intel Core i7-12700 기준):
```
순차 실행: 8.4초
병렬 실행 (4코어): 2.3초  (속도 향상: 3.65배)
캐시 최적화 병렬 (4코어): 0.8초  (속도 향상: 10.5배!)
```

캐시 최적화가 추가 병렬화보다 더 큰 효과를 낸다는 점에 주목하자. 병렬 프로그래밍에서 **메모리 접근 패턴 최적화가 스레드 수 증가보다 더 중요한 경우**가 많다.

---

## 핵심 동기화 구조

### reduction 절

```c
// 위험한 코드: 여러 스레드가 sum에 동시 쓰기 → Race Condition
double sum = 0.0;
#pragma omp parallel for
for (int i = 0; i < N; i++) sum += arr[i];  // ❌

// 올바른 코드: reduction이 내부적으로 private 복사본 생성 후 합산
double sum = 0.0;
#pragma omp parallel for reduction(+:sum)
for (int i = 0; i < N; i++) sum += arr[i];  // ✓

// 지원 연산: +, *, -, &, |, ^, &&, ||, max, min
double max_val = -INFINITY;
#pragma omp parallel for reduction(max:max_val)
for (int i = 0; i < N; i++) max_val = fmax(max_val, arr[i]);
```

### schedule 절 — 부하 분산 전략

```c
int arr[1000];

// static: 컴파일 타임에 균등 분배. 각 반복의 작업량이 동일할 때
#pragma omp parallel for schedule(static)
for (int i = 0; i < 1000; i++) arr[i] = i * 2;

// static, chunk: 청크 단위로 라운드로빈 분배
#pragma omp parallel for schedule(static, 10)
for (int i = 0; i < 1000; i++) arr[i] = i * 2;

// dynamic: 스레드가 끝날 때마다 새 작업 할당. 작업 시간이 불균일할 때
#pragma omp parallel for schedule(dynamic, 1)
for (int i = 0; i < 1000; i++) {
    // arr[i]에 비례한 무거운 작업
    for (int j = 0; j < arr[i]; j++) {}
}

// guided: 청크 크기를 동적으로 감소시키며 분배 (마지막에 작은 청크)
#pragma omp parallel for schedule(guided)
for (int i = 0; i < 1000; i++) {}
```

---

## 태스크 기반 병렬화 (OpenMP Tasks)

루프 기반 병렬화는 **규칙적인 데이터 병렬성**에 적합하지만, 재귀 알고리즘이나 불규칙한 작업 그래프에는 **태스크(Task)** 모델이 더 적합하다.

```c
// 병렬 퀵정렬: 태스크 기반 병렬화
void quicksort_parallel(int* arr, int lo, int hi) {
    if (lo >= hi) return;
    
    int pivot = arr[hi];
    int i = lo - 1;
    for (int j = lo; j < hi; j++) {
        if (arr[j] <= pivot) {
            i++;
            int tmp = arr[i]; arr[i] = arr[j]; arr[j] = tmp;
        }
    }
    int tmp = arr[i+1]; arr[i+1] = arr[hi]; arr[hi] = tmp;
    int p = i + 1;
    
    // 서브문제를 태스크로 생성 — 스레드 풀에서 처리
    #pragma omp task shared(arr) if(hi - lo > 1000)
    quicksort_parallel(arr, lo, p - 1);
    
    #pragma omp task shared(arr) if(hi - lo > 1000)
    quicksort_parallel(arr, p + 1, hi);
    
    #pragma omp taskwait  // 자식 태스크 완료 대기
}

int main() {
    int arr[] = {9, 3, 7, 1, 5, 8, 2, 6, 4};
    int n = 9;
    
    #pragma omp parallel
    {
        #pragma omp single  // 태스크 생성은 하나의 스레드만
        quicksort_parallel(arr, 0, n - 1);
    }
    return 0;
}
```

---

## 주의사항과 성능 팁

### 암달의 법칙 (Amdahl's Law)

병렬화의 이론적 최대 속도 향상은 순차 부분의 비율에 의해 제한된다.

```
속도 향상 = 1 / (S + (1-S)/N)
```

여기서 S는 순차 부분의 비율, N은 스레드 수. 순차 부분이 10%만 되어도 아무리 많은 코어를 사용해도 10배 이상 빠르게 할 수 없다. **먼저 프로파일링으로 실제 병목을 찾고 병렬화**하라.

### Race Condition 방지

공유 변수에 대한 동시 쓰기는 반드시 `critical`, `atomic`, `reduction`, 또는 뮤텍스로 보호해야 한다. `#pragma omp atomic`이 `critical`보다 가볍지만 단순 연산(+=, -=, *=)에만 사용 가능하다.

```c
// atomic은 하드웨어 원자 명령 사용 — critical보다 빠름
#pragma omp atomic
counter++;

// critical은 범용 임계 구간 — 복잡한 연산에 사용
#pragma omp critical(my_section)
{
    complex_update(&shared_data);
}
```

### False Sharing 방지

서로 다른 스레드가 같은 캐시 라인의 다른 변수를 수정하면 불필요한 캐시 무효화가 발생한다. 배열을 각 스레드에 분배할 때 캐시 라인 크기(64바이트)를 고려하여 패딩을 추가해야 한다.

```c
// 위험: 스레드 0은 partial[0], 스레드 1은 partial[1] 수정
// → 같은 캐시 라인 → False Sharing
double partial[4];

// 안전: 캐시 라인(64바이트=8개 double) 단위로 분리
typedef struct { double val; char pad[56]; } CacheAligned;
CacheAligned partial_aligned[4];  // 각 원소가 별도 캐시 라인에 위치
```

### 스레드 오버헤드 주의

스레드 생성/소멸과 동기화에는 비용이 따른다. `parallel for`의 반복 횟수가 수십 건에 불과하거나, 루프 본문이 매우 단순하면 순차 실행보다 느려질 수 있다. **`if` 절**로 조건부 병렬화를 적용하라.

```c
// 데이터 크기가 충분히 클 때만 병렬화
#pragma omp parallel for if(n > 10000)
for (int i = 0; i < n; i++) arr[i] *= 2;
```

---

## 참고 자료
- [OpenMP Official Specification — openmp.org](https://www.openmp.org/specifications/)
- [Introduction to OpenMP — Lawrence Livermore National Lab](https://hpc-tutorials.llnl.gov/openmp/)
- [OpenMP Tutorial — NERSC](https://www.nersc.gov/assets/Uploads/Session-1-Intro-OpenMP-v3.pdf)
- [Tim Mattson's OpenMP Video Lectures — openmp.org](https://www.openmp.org/resources/tutorials-articles/)
