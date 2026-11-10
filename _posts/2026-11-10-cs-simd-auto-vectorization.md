---
layout: post
title: "SIMD와 자동 벡터화 완전 정복: CPU가 데이터를 한 번에 처리하는 병렬 연산의 비밀"
date: 2026-11-10
categories: [cs, computer-science]
tags: [simd, avx, sse, vectorization, cpu, performance, intrinsics, compiler-optimization]
---

## SIMD란 무엇인가?

SIMD(Single Instruction, Multiple Data)는 Flynn의 분류법에서 등장한 병렬 처리 아키텍처 범주입니다. 하나의 명령어가 여러 데이터 요소에 동시에 적용됩니다. 현대 범용 CPU(x86, ARM)는 모두 SIMD 명령어 세트를 내장하고 있으며, 이를 올바르게 활용하면 동일 클럭에서 2배~16배 이상의 처리량을 달성할 수 있습니다.

SIMD 대 스칼라 처리의 직관적 비교:

```
스칼라: a[0]+b[0], a[1]+b[1], a[2]+b[2], a[3]+b[3]  → 4번의 덧셈 명령
SIMD:   [a0,a1,a2,a3] + [b0,b1,b2,b3]               → 1번의 벡터 덧셈 명령
```

## 왜 SIMD가 중요한가?

현대 애플리케이션에서 성능 병목의 상당 부분은 대규모 배열 연산에서 발생합니다.

- **멀티미디어 처리**: 이미지 필터링, 비디오 인코딩, 오디오 DSP
- **머신러닝**: 행렬 곱, 합성곱, 활성화 함수 — PyTorch/TensorFlow 내부 커널
- **데이터베이스**: 컬럼형 스캔, 집계, 필터링 — DuckDB, ClickHouse의 핵심
- **암호화**: AES-NI, SHA 하드웨어 가속
- **과학 계산**: FFT, 선형 대수 (OpenBLAS, FFTW)

SIMD 없이는 현대 딥러닝 추론이나 4K 비디오 실시간 인코딩은 불가능에 가깝습니다.

## SIMD 명령어 세트 계보

### x86 계열

| 세대 | 도입 연도 | 레지스터 너비 | 대표 명령 |
|------|----------|-------------|----------|
| MMX | 1996 | 64비트 | 정수 연산 |
| SSE | 1999 | 128비트 (4×f32) | `_mm_add_ps` |
| SSE2 | 2001 | 128비트 (2×f64) | `_mm_add_pd` |
| AVX | 2011 | 256비트 (8×f32) | `_mm256_add_ps` |
| AVX2 | 2013 | 256비트 (정수) | `_mm256_add_epi32` |
| AVX-512 | 2017 | 512비트 (16×f32) | `_mm512_add_ps` |

### ARM 계열

ARM NEON(2004): 128비트 벡터 레지스터, 모바일/서버 모두에서 사용.  
ARM SVE/SVE2(2019): 가변 너비(128~2048비트), HPC 서버용.

## 실제 구현 예제

### 예제 1: C++ Intrinsics를 사용한 벡터 내적(Dot Product)

```cpp
#include <immintrin.h>  // AVX/AVX2 Intrinsics
#include <cstddef>
#include <numeric>

// 스칼라 버전: 기준선
float dot_product_scalar(const float* a, const float* b, size_t n) {
    float sum = 0.0f;
    for (size_t i = 0; i < n; ++i) {
        sum += a[i] * b[i];
    }
    return sum;
}

// AVX 버전: 8개 float를 한 번에 처리
float dot_product_avx(const float* a, const float* b, size_t n) {
    __m256 acc = _mm256_setzero_ps();  // 256비트 누산기 초기화 (8×0.0f)

    size_t i = 0;
    // 메인 루프: 8개씩 처리
    for (; i + 8 <= n; i += 8) {
        __m256 va = _mm256_loadu_ps(a + i);  // 비정렬 로드: a[i..i+7]
        __m256 vb = _mm256_loadu_ps(b + i);  // 비정렬 로드: b[i..i+7]
        // FMA(Fused Multiply-Add): acc += va * vb (반올림 오차 감소)
        acc = _mm256_fmadd_ps(va, vb, acc);
    }

    // 수평 합산: 8개 float를 1개로 축약
    __m128 lo = _mm256_castps256_ps128(acc);
    __m128 hi = _mm256_extractf128_ps(acc, 1);
    __m128 sum128 = _mm_add_ps(lo, hi);
    sum128 = _mm_hadd_ps(sum128, sum128);
    sum128 = _mm_hadd_ps(sum128, sum128);
    float result = _mm_cvtss_f32(sum128);

    // 나머지 요소 스칼라 처리
    for (; i < n; ++i) {
        result += a[i] * b[i];
    }
    return result;
}

// 사용 예
// 1M 요소 벡터: 스칼라 ~2ms vs AVX ~0.3ms (약 6.7배 향상)
```

컴파일:
```bash
g++ -O3 -mavx2 -mfma -o dot_product dot_product.cpp
# 또는 최신 아키텍처 전체 허용
g++ -O3 -march=native -o dot_product dot_product.cpp
```

### 예제 2: 자동 벡터화(Auto-Vectorization)와 컴파일러 힌트

Intrinsics를 직접 작성하지 않아도, 컴파일러가 패턴을 인식하면 자동으로 SIMD 코드를 생성합니다.

```c
#include <stddef.h>
#include <stdint.h>

// 이 함수는 GCC/Clang -O2 이상에서 SSE2/AVX로 자동 벡터화됨
void array_add(float* restrict dst,
               const float* restrict src1,
               const float* restrict src2,
               size_t n) {
    // restrict 키워드: 포인터가 겹치지 않음을 컴파일러에 보장
    for (size_t i = 0; i < n; ++i) {
        dst[i] = src1[i] + src2[i];
    }
}

// 벡터화를 방해하는 패턴: 의존성 있는 루프
void bad_for_vectorization(float* a, size_t n) {
    for (size_t i = 1; i < n; ++i) {
        // a[i]가 a[i-1]에 의존 → 벡터화 불가
        a[i] = a[i] + a[i-1];
    }
}

// 컴파일러 벡터화 보고서 확인:
// GCC:   -fopt-info-vec-missed
// Clang: -Rpass=loop-vectorize -Rpass-missed=loop-vectorize
```

Rust의 경우 `std::simd`(Portable SIMD)로 플랫폼 독립적 SIMD 코드 작성이 가능합니다:

```rust
#![feature(portable_simd)]
use std::simd::{f32x8, SimdFloat};

pub fn array_sum_simd(data: &[f32]) -> f32 {
    let mut acc = f32x8::splat(0.0);
    let chunks = data.chunks_exact(8);
    let remainder = chunks.remainder();

    for chunk in chunks {
        let v = f32x8::from_slice(chunk);
        acc += v;
    }

    let simd_sum: f32 = acc.reduce_sum();
    let tail_sum: f32 = remainder.iter().sum();
    simd_sum + tail_sum
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_simd_sum() {
        let data: Vec<f32> = (1..=100).map(|x| x as f32).collect();
        let expected: f32 = (1..=100).map(|x| x as f32).sum();
        let result = array_sum_simd(&data);
        assert!((result - expected).abs() < 1e-3);
    }
}
```

## 메모리 정렬과 성능

SIMD 연산에서 메모리 정렬은 성능에 결정적 영향을 미칩니다.

```cpp
#include <stdlib.h>
#include <immintrin.h>

// 정렬된 메모리 할당 (AVX는 32바이트 정렬 필요)
float* alloc_aligned(size_t n) {
    return (float*)aligned_alloc(32, n * sizeof(float));
}

void aligned_load_example(const float* aligned_ptr) {
    __m256 v1 = _mm256_loadu_ps(aligned_ptr);  // 비정렬 로드
    __m256 v2 = _mm256_load_ps(aligned_ptr);   // 정렬 로드 (32바이트 경계 필수)
    (void)v1; (void)v2;
}
```

## 주의사항과 팁

### 1. 플랫폼 가용성 확인

AVX-512는 모든 x86 CPU에서 지원되지 않습니다. 런타임에 CPUID로 지원 여부를 확인하거나, 컴파일 타임에 조건부 분기를 사용하세요.

```c
#include <cpuid.h>

int has_avx2(void) {
    unsigned int eax, ebx, ecx, edx;
    if (!__get_cpuid_count(7, 0, &eax, &ebx, &ecx, &edx))
        return 0;
    return (ebx & (1u << 5)) != 0;  // EBX bit 5 = AVX2
}
```

### 2. 분기를 피하라 (Branch-Free SIMD)

SIMD 루프 안에 조건 분기가 있으면 벡터화가 어렵습니다. 대신 블렌딩(blending)이나 마스킹을 활용합니다:

```cpp
// 조건 분기 대신 비교+블렌딩
__m256 abs_val(__m256 v) {
    __m256 mask = _mm256_set1_ps(-0.0f);  // 부호 비트만 1
    return _mm256_andnot_ps(mask, v);     // 부호 비트 제거 = 절댓값
}
```

### 3. 벡터화 보고서 활용

```bash
# GCC
gcc -O2 -fopt-info-vec-missed my_code.c 2>&1 | grep "not vectorized"

# Clang
clang -O2 -Rpass-missed=loop-vectorize my_code.c 2>&1
```

가장 흔한 실패 원인: 포인터 앨리어싱, 루프 종속성, 함수 호출, 비정형 메모리 접근 패턴.

### 4. 열 지하 현상(Frequency Throttling)

AVX-512 명령어는 CPU 주파수를 낮출 수 있습니다. Intel의 AVX-512 turbo frequency는 AVX2보다 낮으므로, 짧은 버스트 연산에서는 AVX2가 더 빠를 수도 있습니다. 항상 프로파일링으로 검증하세요.

## 참고 자료

- [Intel Intrinsics Guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html)
- [Agner Fog의 최적화 매뉴얼](https://www.agner.org/optimize/)
- [Rust Portable SIMD RFC](https://github.com/rust-lang/rfcs/blob/master/text/2948-portable-simd.md)
- [GCC Auto-Vectorization 문서](https://gcc.gnu.org/projects/tree-ssa/vectorization.html)
