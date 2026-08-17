---
layout: post
title: "GPU 아키텍처와 CUDA 프로그래밍 모델 완전 정복: SM·Warp·메모리 계층 구조부터 커널 최적화까지"
date: 2026-08-17
categories: [cs, computer-science]
tags: [gpu, cuda, nvidia, sm, warp, parallel-computing, hpc, performance]
---

CPU는 수십 개의 복잡한 코어로 단일 스레드 성능을 극대화한다. GPU는 반대다. 수천 개의 단순한 코어가 동시에 같은 명령을 수행한다. 이 차이는 설계 철학의 차이다. CPU는 **지연 시간(latency)**을 줄이도록 설계됐고, GPU는 **처리량(throughput)**을 극대화하도록 설계됐다.

딥러닝이 폭발적으로 성장한 이유도 여기 있다. 행렬 곱셈(matrix multiplication)은 수백만 개의 독립적인 곱셈-덧셈 연산으로 분해된다. GPU는 이를 병렬로 처리해 CPU 대비 수십 배 빠른 속도를 낸다. 비디오 렌더링, 물리 시뮬레이션, 암호화폐 채굴도 같은 이유로 GPU를 쓴다.

NVIDIA의 CUDA(Compute Unified Device Architecture)는 GPU를 범용 병렬 컴퓨팅에 사용하기 위한 플랫폼이다. 이 글에서는 GPU의 하드웨어 구조부터 CUDA 프로그래밍 모델, 성능 최적화 원리까지 단계적으로 풀어낸다.

---

## GPU 하드웨어 구조: SM과 CUDA 코어

### Streaming Multiprocessor (SM)

SM은 GPU의 핵심 연산 단위다. 하나의 GPU는 수십~수백 개의 SM을 포함한다. 각 SM은 다음을 포함한다:

- **CUDA 코어**: 정수/부동소수점 ALU. NVIDIA Ampere(A100) 기준 SM당 128개.
- **Tensor 코어**: 행렬 곱셈 전용 하드웨어. FP16/BF16/INT8 연산을 대규모로 가속.
- **레지스터 파일**: SM 내 모든 스레드가 공유하는 레지스터 풀. 보통 64K × 32bit.
- **공유 메모리(Shared Memory)**: 같은 SM의 스레드 블록 내 스레드들이 공유. L1 캐시와 통합 구성.
- **Warp 스케줄러**: 매 클록 사이클에 실행할 warp를 선택.

A100 기준 SM이 108개이고 SM당 CUDA 코어 128개면, 총 13,824개 코어다. H100은 더 많다.

### Warp: GPU의 기본 실행 단위

GPU는 **warp** 단위로 스레드를 실행한다. warp는 **32개 스레드**로 구성되며, 이 32개는 항상 동일한 명령을 실행한다. 이 실행 모델을 **SIMT(Single Instruction, Multiple Threads)**라 부른다.

SIMD와의 차이: SIMD는 프로그래머가 명시적으로 벡터 타입을 사용해야 한다. SIMT는 일반 스칼라 코드로 작성해도 하드웨어가 자동으로 병렬 실행한다. 단, 분기(branch)가 발생하면 문제가 생긴다.

#### Warp 다이버전스 (Warp Divergence)

```cuda
__global__ void divergent_kernel(int *data, int n) {
    int tid = blockIdx.x * blockDim.x + threadIdx.x;
    if (tid < n) {
        if (data[tid] % 2 == 0) {
            data[tid] *= 2;  /* 짝수 스레드 경로 */
        } else {
            data[tid] += 1;  /* 홀수 스레드 경로 */
        }
    }
}
```

warp 내 32개 스레드가 다른 분기를 탄다면, GPU는 두 경로를 **순차적으로** 실행하고 각자 다른 쪽 결과를 버린다. 이 경우 병렬성이 절반으로 줄어든다. warp 내 모든 스레드가 같은 경로를 타도록 데이터를 구성하면 성능이 크게 향상된다.

---

## 메모리 계층 구조

GPU 성능 최적화의 핵심은 **메모리 접근 패턴**이다.

| 메모리 종류 | 위치 | 크기 | 지연시간 | 대역폭 |
|-----------|------|------|---------|--------|
| 레지스터 | SM 내부 | 스레드당 ~255개 | 1 사이클 | 최고 |
| 공유 메모리 | SM 내부 | 48~164 KB/SM | ~5 사이클 | ~10 TB/s |
| L1 캐시 | SM 내부 | 공유 메모리와 통합 | ~30 사이클 | - |
| L2 캐시 | GPU 칩 | 40~60 MB | ~150 사이클 | ~5 TB/s |
| 글로벌 메모리 | HBM/GDDR | 16~80 GB | ~400 사이클 | 2~3.35 TB/s |

**접근 원칙**: 레지스터 → 공유 메모리 → L2 → 글로벌 메모리 순으로 빠르다. 알고리즘을 설계할 때 핫 데이터를 공유 메모리에 올려두는 것이 가장 중요한 최적화다.

---

## CUDA 스레드 계층 구조

CUDA는 3단계 계층으로 스레드를 조직한다:

```
Grid
└── Block (= Thread Block)
    └── Warp (32 threads, implicit)
        └── Thread
```

- **Thread**: 커널 함수를 실행하는 최소 단위. `threadIdx.x/y/z`로 식별.
- **Block**: 동일한 SM에서 실행되는 스레드 묶음. `blockIdx.x/y/z`로 식별. 블록 내 스레드는 공유 메모리와 `__syncthreads()`를 공유한다.
- **Grid**: 커널 실행 전체. 블록들의 집합. 블록끼리는 독립적으로 실행된다.

---

## CUDA 코드 예제 1: 벡터 덧셈

```cuda
// vector_add.cu
#include <stdio.h>
#include <cuda_runtime.h>

/* GPU에서 실행되는 커널 함수 */
__global__ void vector_add(const float *a, const float *b, float *c, int n) {
    /* 전역 스레드 인덱스 계산 */
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx < n) {
        c[idx] = a[idx] + b[idx];
    }
}

int main(void) {
    const int N = 1 << 20;  /* 1M 원소 */
    size_t size = N * sizeof(float);

    /* 호스트(CPU) 메모리 할당 */
    float *h_a = (float *)malloc(size);
    float *h_b = (float *)malloc(size);
    float *h_c = (float *)malloc(size);

    for (int i = 0; i < N; i++) {
        h_a[i] = (float)i;
        h_b[i] = (float)(N - i);
    }

    /* 디바이스(GPU) 메모리 할당 */
    float *d_a, *d_b, *d_c;
    cudaMalloc(&d_a, size);
    cudaMalloc(&d_b, size);
    cudaMalloc(&d_c, size);

    /* 호스트 → 디바이스 전송 (H2D) */
    cudaMemcpy(d_a, h_a, size, cudaMemcpyHostToDevice);
    cudaMemcpy(d_b, h_b, size, cudaMemcpyHostToDevice);

    /* 커널 실행: 블록당 256 스레드, 필요한 블록 수 계산 */
    int blockSize = 256;
    int gridSize  = (N + blockSize - 1) / blockSize;
    vector_add<<<gridSize, blockSize>>>(d_a, d_b, d_c, N);

    /* 동기화 및 결과 전송 (D2H) */
    cudaDeviceSynchronize();
    cudaMemcpy(h_c, d_c, size, cudaMemcpyDeviceToHost);

    printf("c[0]=%f, c[N-1]=%f\n", h_c[0], h_c[N-1]);  /* 1048576, 1048576 */

    /* 메모리 해제 */
    cudaFree(d_a); cudaFree(d_b); cudaFree(d_c);
    free(h_a); free(h_b); free(h_c);
    return 0;
}
```

컴파일: `nvcc vector_add.cu -o vector_add`

`<<<gridSize, blockSize>>>` 표기가 CUDA 문법의 핵심이다. 첫 번째는 그리드 내 블록 수, 두 번째는 블록 내 스레드 수다.

---

## CUDA 코드 예제 2: 공유 메모리를 활용한 행렬 곱셈

Naive 구현은 글로벌 메모리를 반복 접근해 성능이 낮다. **타일링(tiling)**으로 공유 메모리를 활용하면 드라마틱하게 빨라진다.

```cuda
// matmul_tiled.cu
#define TILE_SIZE 32

__global__ void matmul_tiled(const float *A, const float *B, float *C,
                              int M, int N, int K) {
    /* 공유 메모리 타일 선언 */
    __shared__ float tileA[TILE_SIZE][TILE_SIZE];
    __shared__ float tileB[TILE_SIZE][TILE_SIZE];

    int row = blockIdx.y * TILE_SIZE + threadIdx.y;
    int col = blockIdx.x * TILE_SIZE + threadIdx.x;
    float sum = 0.0f;

    /* K 방향으로 타일 슬라이딩 */
    for (int t = 0; t < (K + TILE_SIZE - 1) / TILE_SIZE; t++) {
        /* 타일을 공유 메모리로 로드 */
        if (row < M && t * TILE_SIZE + threadIdx.x < K)
            tileA[threadIdx.y][threadIdx.x] = A[row * K + t * TILE_SIZE + threadIdx.x];
        else
            tileA[threadIdx.y][threadIdx.x] = 0.0f;

        if (col < N && t * TILE_SIZE + threadIdx.y < K)
            tileB[threadIdx.y][threadIdx.x] = B[(t * TILE_SIZE + threadIdx.y) * N + col];
        else
            tileB[threadIdx.y][threadIdx.x] = 0.0f;

        /* 모든 스레드가 로드 완료할 때까지 대기 */
        __syncthreads();

        /* 타일 내에서 내적 계산 — 공유 메모리 접근 */
        for (int k = 0; k < TILE_SIZE; k++)
            sum += tileA[threadIdx.y][k] * tileB[k][threadIdx.x];

        /* 다음 타일 로드 전 동기화 */
        __syncthreads();
    }

    if (row < M && col < N)
        C[row * N + col] = sum;
}
```

Naive 버전과 비교:
- Naive: 행렬 원소 하나를 계산하기 위해 글로벌 메모리를 K번 접근.
- Tiled: TILE_SIZE × TILE_SIZE 원소를 공유 메모리에 올리고, 각 원소가 TILE_SIZE번 재사용. 글로벌 메모리 접근 횟수가 TILE_SIZE배 줄어든다.

---

## 성능 최적화 원칙

### 코얼레스드 메모리 접근 (Coalesced Memory Access)

warp 내 32개 스레드가 연속된 메모리 주소를 접근하면 하드웨어가 이를 하나의 메모리 트랜잭션으로 합쳐 처리한다. 이를 **메모리 코얼레싱**이라 한다.

```cuda
/* 좋은 패턴: 스레드 0→addr0, 스레드 1→addr1, ... (연속) */
data[threadIdx.x] = input[blockIdx.x * blockDim.x + threadIdx.x];

/* 나쁜 패턴: 스레드 0→addr0, 스레드 1→addr32, ... (분산) */
data[threadIdx.x] = input[threadIdx.x * 32];
```

분산된 접근은 32개의 별도 트랜잭션을 유발해 메모리 대역폭 효율이 1/32로 떨어진다.

### 뱅크 충돌 (Bank Conflict)

공유 메모리는 32개의 뱅크로 나뉜다. 같은 warp의 여러 스레드가 같은 뱅크에 접근하면 직렬화된다. 32×32 행렬을 공유 메모리에 올릴 때 `[32][33]`으로 패딩 한 열을 추가하면 충돌을 피할 수 있다.

### 점유율 (Occupancy)

SM에 동시에 상주할 수 있는 warp 수를 **occupancy**라 한다. 레지스터와 공유 메모리 사용량이 클수록 SM이 수용할 수 있는 블록이 줄어 점유율이 낮아진다. `nvcc --ptxas-options=-v`로 레지스터 사용량을 확인하고, `cudaOccupancyMaxPotentialBlockSize()`로 최적 블록 크기를 구하라.

---

## 주의사항 및 팁

**비동기 실행 모델 이해**: `cudaMemcpy`는 기본적으로 동기이지만, `cudaMemcpyAsync`는 비동기다. 커널 실행은 항상 비동기다. CUDA 스트림(`cudaStream_t`)으로 메모리 전송과 커널 실행을 겹치면 호스트-디바이스 대역폭을 효과적으로 숨길 수 있다.

**Unified Memory 함정**: `cudaMallocManaged()`는 편리하지만, 페이지 폴트 오버헤드가 있다. 실제 운영 환경에서는 명시적 메모리 관리가 더 빠른 경우가 많다.

**Tensor 코어 활용**: cuBLAS, cuDNN, CUTLASS 같은 라이브러리는 내부적으로 Tensor 코어를 활용한다. FP32 행렬 곱셈을 FP16으로 바꾸면 Tensor 코어가 동작해 속도가 크게 향상될 수 있다.

**프로파일링 없이 최적화하지 말 것**: Nsight Systems, Nsight Compute, `nvprof`로 먼저 병목을 찾아라. 메모리 대역폭 제한인지, 연산 제한인지 확인 후 최적화 방향을 결정해야 한다.

## 참고 자료
- [NVIDIA GPU Architecture Explained — Thunder Compute](https://www.thundercompute.com/blog/nvidia-gpu-architecture-explained)
- [GPU Architecture Fundamentals — BentoML LLM Inference Handbook](https://bentoml.com/llm/kernel-optimization/gpu-architecture-fundamentals)
- [CUDA C Programming Guide — NVIDIA Documentation](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)
- [CUDA Best Practices Guide — NVIDIA](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/)
