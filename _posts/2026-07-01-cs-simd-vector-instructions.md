---
layout: post
title: "SIMD 벡터 연산 완전 정복: CPU가 데이터를 한 번에 처리하는 방법"
date: 2026-07-01
categories: [cs, computer-science]
tags: [SIMD, AVX, SSE, vectorization, performance, CPU, parallel, intrinsics]
---

현대 CPU는 단순히 "빠른" 것을 넘어 **동시에 여러 데이터를 처리**할 수 있도록 설계되어 있습니다. 1초에 수십억 번의 연산을 수행하는 CPU가 하나의 덧셈을 처리하는 동안, 스마트하게 설계된 코드는 같은 시간에 8개, 16개, 심지어 32개의 덧셈을 동시에 처리합니다. 이것이 **SIMD(Single Instruction, Multiple Data)**의 세계입니다.

## SIMD란 무엇인가?

플린(Flynn)의 분류에 따르면 컴퓨터 아키텍처는 명령어 흐름과 데이터 흐름의 수에 따라 4가지로 구분됩니다:

| 분류 | 설명 | 예시 |
|------|------|------|
| SISD | 단일 명령, 단일 데이터 | 전통적 스칼라 CPU |
| **SIMD** | **단일 명령, 다중 데이터** | **AVX, SSE, NEON** |
| MISD | 다중 명령, 단일 데이터 | 파이프라인 (이론적) |
| MIMD | 다중 명령, 다중 데이터 | 멀티코어, GPU |

SIMD의 핵심은 **벡터 레지스터**입니다. 일반 CPU 레지스터가 64비트 하나를 담는다면, AVX2의 YMM 레지스터는 256비트를 담습니다. 이 256비트 공간에 32비트 정수 8개, 또는 32비트 부동소수점 8개, 또는 8비트 정수 32개를 동시에 저장하고 연산합니다.

```
일반 덧셈 (스칼라):
  a + b = c  (단 하나의 값)

AVX2 덧셈 (벡터):
  [a0, a1, a2, a3, a4, a5, a6, a7]   ← 256비트 YMM 레지스터
+ [b0, b1, b2, b3, b4, b5, b6, b7]
= [c0, c1, c2, c3, c4, c5, c6, c7]   (8개를 1개 명령으로!)
```

## SIMD 명령어 집합의 역사

```
SSE   (1999) — 128비트 XMM 레지스터, 4× float32 또는 2× float64
SSE2  (2001) — 정수 연산 추가 (16× int8, 8× int16, 4× int32, 2× int64)
SSE4  (2007) — 블렌딩, 문자열 처리, dot product
AVX   (2011) — 256비트 YMM 레지스터 (FP 연산만)
AVX2  (2013) — 256비트 정수 연산 추가, FMA, gather/scatter
AVX-512 (2017) — 512비트 ZMM 레지스터, 64× int8

ARM:
NEON  (ARMv7) — 128비트 (Apple Silicon, Android)
SVE   (ARMv8.2) — 가변 길이 벡터 (128~2048비트)
```

2024년 기준 일반적인 x86_64 데스크탑/서버 CPU는 모두 AVX2를 지원합니다.

## 왜 SIMD가 중요한가?

현대 소프트웨어에서 SIMD는 다음 영역에서 결정적인 역할을 합니다:

- **이미지/영상 처리**: 픽셀 단위 연산 (밝기 조정, 필터, 코덱)
- **머신러닝**: 행렬 곱셈, 벡터 내적 (BLAS, PyTorch의 핵심)
- **데이터베이스**: 컬럼 스캔, 비트맵 필터링
- **암호화**: AES-NI, SHA-NI 하드웨어 가속
- **문자열 처리**: 검색, 파싱 (simd-json, hyperscan)
- **과학 계산**: FFT, 선형대수

## 실제 구현 예제

### 예제 1: C 언어로 AVX2 벡터 덧셈 구현

```c
#include <immintrin.h>  // AVX/AVX2 헤더
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <string.h>

#define N (1 << 24)  // 16M elements

// 스칼라 버전
void add_scalar(const float* a, const float* b, float* c, int n) {
    for (int i = 0; i < n; i++) {
        c[i] = a[i] + b[i];
    }
}

// AVX2 벡터 버전 (8× float32 동시 처리)
void add_avx2(const float* a, const float* b, float* c, int n) {
    int i = 0;
    // 8개씩 처리 (AVX2: 256비트 = 8× float32)
    for (; i <= n - 8; i += 8) {
        __m256 va = _mm256_loadu_ps(a + i);  // 8개 float 로드 (unaligned)
        __m256 vb = _mm256_loadu_ps(b + i);
        __m256 vc = _mm256_add_ps(va, vb);  // 8개 동시 덧셈
        _mm256_storeu_ps(c + i, vc);         // 결과 저장
    }
    // 나머지 처리 (8로 나누어 떨어지지 않는 경우)
    for (; i < n; i++) {
        c[i] = a[i] + b[i];
    }
}

// 정렬된 메모리를 사용한 최적화 버전
void add_avx2_aligned(const float* __restrict__ a,
                       const float* __restrict__ b,
                       float* __restrict__ c, int n) {
    int i = 0;
    for (; i <= n - 8; i += 8) {
        __m256 va = _mm256_load_ps(a + i);   // aligned load (더 빠름)
        __m256 vb = _mm256_load_ps(b + i);
        __m256 vc = _mm256_add_ps(va, vb);
        _mm256_store_ps(c + i, vc);
    }
    for (; i < n; i++) c[i] = a[i] + b[i];
}

// 수평 합산 (벡터의 모든 요소를 더하기)
float horizontal_sum_avx(const float* arr, int n) {
    __m256 acc = _mm256_setzero_ps();
    int i = 0;
    for (; i <= n - 8; i += 8) {
        acc = _mm256_add_ps(acc, _mm256_loadu_ps(arr + i));
    }
    // 256비트 → 128비트로 접기
    __m128 low  = _mm256_castps256_ps128(acc);
    __m128 high = _mm256_extractf128_ps(acc, 1);
    __m128 sum4 = _mm_add_ps(low, high);
    // 4개 → 2개 → 1개로 접기
    sum4 = _mm_hadd_ps(sum4, sum4);
    sum4 = _mm_hadd_ps(sum4, sum4);
    float result = _mm_cvtss_f32(sum4);
    // 나머지
    for (; i < n; i++) result += arr[i];
    return result;
}

int main() {
    // 32바이트 정렬 메모리 할당 (AVX2 aligned load 요구사항)
    float* a = (float*)aligned_alloc(32, N * sizeof(float));
    float* b = (float*)aligned_alloc(32, N * sizeof(float));
    float* c_scalar = (float*)aligned_alloc(32, N * sizeof(float));
    float* c_avx    = (float*)aligned_alloc(32, N * sizeof(float));
    
    for (int i = 0; i < N; i++) {
        a[i] = (float)i * 0.5f;
        b[i] = (float)i * 0.3f;
    }
    
    struct timespec t0, t1;
    
    // 스칼라 버전 벤치마크
    clock_gettime(CLOCK_MONOTONIC, &t0);
    add_scalar(a, b, c_scalar, N);
    clock_gettime(CLOCK_MONOTONIC, &t1);
    double scalar_ms = (t1.tv_sec - t0.tv_sec) * 1000.0 +
                       (t1.tv_nsec - t0.tv_nsec) / 1e6;
    
    // AVX2 버전 벤치마크
    clock_gettime(CLOCK_MONOTONIC, &t0);
    add_avx2_aligned(a, b, c_avx, N);
    clock_gettime(CLOCK_MONOTONIC, &t1);
    double avx_ms = (t1.tv_sec - t0.tv_sec) * 1000.0 +
                    (t1.tv_nsec - t0.tv_nsec) / 1e6;
    
    printf("Scalar: %.2f ms\n", scalar_ms);
    printf("AVX2:   %.2f ms\n", avx_ms);
    printf("Speedup: %.1fx\n", scalar_ms / avx_ms);
    
    // 수평 합산 테스트
    float sum = horizontal_sum_avx(a, N);
    printf("Sum of a: %.0f\n", sum);
    
    free(a); free(b); free(c_scalar); free(c_avx);
    return 0;
}

/*
 * 컴파일: gcc -O2 -march=native -mavx2 -o simd_demo simd_demo.c
 * 일반적인 결과:
 *   Scalar: 32.5 ms
 *   AVX2:    5.1 ms
 *   Speedup: 6.4x
 */
```

### 예제 2: Python NumPy로 SIMD 자동 벡터화 활용

Python에서 NumPy는 내부적으로 OpenBLAS/MKL을 통해 SIMD를 자동으로 활용합니다. 루프 대신 벡터 연산을 사용하는 것만으로 SIMD의 혜택을 누릴 수 있습니다.

```python
import numpy as np
import time

N = 10_000_000

# NumPy 배열 생성 (C-contiguous, 메모리 정렬됨)
a = np.random.rand(N).astype(np.float32)
b = np.random.rand(N).astype(np.float32)

# ===========================
# 방법 1: Python 순수 루프 (최악)
# ===========================
def pure_python_sum(a, b):
    result = []
    for x, y in zip(a, b):
        result.append(x + y)
    return result

# ===========================
# 방법 2: NumPy 벡터 연산 (SIMD 자동 활용)
# ===========================
def numpy_vectorized(a, b):
    return a + b  # 내부적으로 AVX2 SIMD 사용

# ===========================
# 방법 3: 캐시 친화적 접근 패턴
# ===========================
def cache_friendly_dot(A: np.ndarray, B: np.ndarray) -> np.ndarray:
    """행렬 곱: 전치를 사용해 열 접근을 행 접근으로 변환"""
    # B를 전치하면 내적 계산 시 연속 메모리 접근 가능
    return A @ B.T

# 벤치마크
print("=== 벡터화 성능 비교 ===")

# Python 루프 (처음 10만개만)
small_a = a[:100_000].tolist()
small_b = b[:100_000].tolist()
start = time.perf_counter()
_ = pure_python_sum(small_a, small_b)
loop_time = (time.perf_counter() - start) * 100  # 100만 기준으로 외삽
print(f"Python loop (extrapolated 1M): {loop_time:.0f}ms")

# NumPy 벡터 연산
for _ in range(3):  # 워밍업
    np.add(a, b)

start = time.perf_counter()
for _ in range(10):
    c = a + b
elapsed = (time.perf_counter() - start) / 10 * 1000
print(f"NumPy vectorized ({N//1_000_000}M): {elapsed:.1f}ms")
print(f"Throughput: {N * 4 / elapsed / 1e6:.0f} MB/ms")

# ===========================
# SIMD 친화적 코드 패턴
# ===========================
print("\n=== SIMD 친화적 패턴 비교 ===")

# 비효율: 조건부 루프 (벡터화 어려움)
def conditional_loop(arr):
    result = np.zeros_like(arr)
    for i, x in enumerate(arr):
        if x > 0.5:
            result[i] = x * 2
        else:
            result[i] = x * 0.5
    return result

# 효율: NumPy 마스크 연산 (벡터화 용이)
def vectorized_mask(arr):
    mask = arr > 0.5
    return np.where(mask, arr * 2, arr * 0.5)

data = np.random.rand(1_000_000).astype(np.float32)

start = time.perf_counter()
r1 = conditional_loop(data)
loop_time = (time.perf_counter() - start) * 1000
print(f"Conditional loop: {loop_time:.0f}ms")

start = time.perf_counter()
for _ in range(100):
    r2 = vectorized_mask(data)
elapsed = (time.perf_counter() - start) / 100 * 1000
print(f"Vectorized mask:  {elapsed:.1f}ms")
print(f"Speedup: {loop_time/elapsed:.0f}x")

# ===========================
# 내적(dot product) — SIMD의 꽃
# ===========================
print("\n=== Dot Product 비교 ===")
v1 = np.random.rand(1000).astype(np.float32)
v2 = np.random.rand(1000).astype(np.float32)

# 방법 1: Python 루프
start = time.perf_counter()
result_loop = sum(x * y for x, y in zip(v1, v2))
loop_t = time.perf_counter() - start

# 방법 2: NumPy dot (내부적으로 FMA + SIMD)
start = time.perf_counter()
for _ in range(10000):
    result_np = np.dot(v1, v2)
np_t = (time.perf_counter() - start) / 10000

print(f"Loop: {loop_t*1e6:.0f}μs, NumPy dot: {np_t*1e6:.2f}μs")
print(f"Speedup: {loop_t/np_t:.0f}x")
print(f"Results match: {abs(result_loop - result_np) < 0.01}")
```

## 컴파일러 자동 벡터화

직접 intrinsic을 쓰지 않아도 컴파일러가 자동으로 SIMD 코드를 생성할 수 있습니다:

```c
// auto_vectorization.c
// gcc -O2 -march=native -fopt-info-vec-optimized -o auto_vec auto_vectorization.c

// 컴파일러가 자동 벡터화하기 좋은 패턴
void add_arrays(float* __restrict__ a, const float* __restrict__ b,
                const float* __restrict__ c, int n) {
    // __restrict__: 포인터 앨리어싱 없음을 컴파일러에 알림
    for (int i = 0; i < n; i++) {
        a[i] = b[i] + c[i];  // → vaddps 명령어로 자동 변환
    }
}

// 자동 벡터화를 방해하는 패턴
void bad_vectorization(float* a, const float* b, int n) {
    // a와 b가 같은 배열을 가리킬 수 있으므로 (aliasing)
    // 컴파일러가 벡터화를 포기할 수 있음
    for (int i = 0; i < n; i++) {
        a[i] = b[i] + b[i-1];  // 이전 요소 의존성 → 벡터화 불가
    }
}

// -fopt-info-vec-optimized 출력 예시:
// add_arrays.c:10:5: optimized: loop vectorized using 32-byte vectors
// add_arrays.c:10:5: optimized: loop versioned for vectorization because of possible aliasing
```

## SIMD 인트린식 명명 규칙

```
_mm<width>_<operation>_<type>

width:
  (없음) → 64비트 MMX
  128     → SSE (XMM 레지스터)
  256     → AVX/AVX2 (YMM 레지스터)
  512     → AVX-512 (ZMM 레지스터)

operation:
  load/store, add/sub/mul/div, and/or/xor
  cmp, shuffle, blend, hadd, ...

type:
  ps = packed single (float32)
  pd = packed double (float64)
  epi8/16/32/64 = packed integer (8/16/32/64비트)
  si256 = 256비트 정수 (타입 무관)

예시:
_mm256_add_ps     → 256비트, 8× float32 덧셈
_mm256_mullo_epi32 → 256비트, 8× int32 곱셈 (하위 32비트)
_mm_cmpeq_epi8    → 128비트, 16× int8 비교
```

## 주의사항 및 팁

**1. 메모리 정렬 (Memory Alignment)**  
`_mm256_load_ps`는 32바이트 정렬을 요구합니다. 정렬되지 않은 메모리에는 `_mm256_loadu_ps`를 사용하세요. 단, 최신 CPU(Haswell 이후)에서는 두 명령어의 성능 차이가 거의 없습니다. `posix_memalign` 또는 `aligned_alloc`으로 정렬된 메모리를 할당하세요.

**2. 포인터 앨리어싱 주의**  
C에서 두 포인터가 같은 메모리를 가리킬 가능성이 있으면 컴파일러는 SIMD 변환을 포기합니다. `__restrict__` 키워드로 앨리어싱이 없음을 명시하거나, `#pragma GCC ivdep`으로 명시적으로 허용하세요.

**3. 분기(Branch) 최소화**  
SIMD 레지스터 내의 각 레인에 대해 다른 분기를 실행할 수 없습니다. 조건부 연산은 마스크 기반으로 처리하세요: `_mm256_blendv_ps`로 조건에 따라 두 벡터에서 선택합니다.

**4. 수평 연산의 비용**  
`_mm256_hadd_ps`처럼 같은 레지스터 내 요소끼리 연산하는 수평 연산은 수직 연산보다 느립니다. 알고리즘을 설계할 때 수직 연산(레인 간 독립)을 최대화하세요.

**5. CPU 기능 감지**  
배포 환경의 CPU가 AVX2를 지원하지 않을 수 있습니다. 런타임에 `__builtin_cpu_supports("avx2")`로 확인하고 폴백 경로를 준비하세요.

**6. 프로파일링 먼저**  
SIMD 최적화는 병목 지점에만 적용하세요. 코드 복잡도가 크게 증가하므로, 먼저 `perf stat`이나 VTune으로 실제 병목이 어디인지 확인하는 것이 우선입니다.

## 참고 자료

- [Intel Intrinsics Guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html)
- [Algorithmica HPC: SIMD Intrinsics](https://en.algorithmica.org/hpc/simd/intrinsics/)
- [Stack Overflow Blog: Improving performance with SIMD intrinsics](https://stackoverflow.blog/2020/07/08/improving-performance-with-simd-intrinsics-in-three-use-cases/)
- [Agner Fog: Optimizing software in C++](https://www.agner.org/optimize/optimizing_cpp.pdf)
