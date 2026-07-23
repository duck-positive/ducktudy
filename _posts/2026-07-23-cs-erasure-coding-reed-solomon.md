---
layout: post
title: "이레이저 코딩(Erasure Coding) 완전 정복: Reed-Solomon이 분산 스토리지 내결함성을 실현하는 방법"
date: 2026-07-23
categories: [cs, computer-science]
tags: [erasure-coding, reed-solomon, distributed-storage, fault-tolerance, data-recovery, galois-field, HDFS, Ceph]
---

분산 스토리지 시스템에서 데이터 내결함성을 구현하는 방법은 크게 두 가지입니다. 첫 번째는 **단순 복제(Replication)** — 데이터를 여러 노드에 동일하게 복사하는 방식이고, 두 번째는 **이레이저 코딩(Erasure Coding)** — 수학적 인코딩을 통해 최소한의 스토리지 오버헤드로 동일한 수준의 내결함성을 달성하는 방식입니다.

Amazon S3, Google Colossus, Ceph, HDFS 등 현대 분산 스토리지 시스템 대부분이 이레이저 코딩을 핵심 기술로 채택한 이유를 깊이 파헤쳐 보겠습니다.

## 이레이저 코딩이란 무엇인가?

이레이저 코딩(Erasure Coding)은 원본 데이터를 **k개의 데이터 블록**과 **m개의 패리티 블록**으로 분할·인코딩하여, 전체 k+m개의 블록 중 임의의 m개까지 손실(erasure)되어도 나머지 k개의 블록으로 원본을 완전히 복원할 수 있는 기법입니다.

이를 **(k, m) 이레이저 코드** 또는 **RS(k, m)** 코드라고 표기합니다.

### 단순 복제 vs 이레이저 코딩 비교

| 방식 | 원본 100GB, 내결함성 2개 노드 기준 | 스토리지 비용 |
|---|---|---|
| 3중 복제 | 300GB 저장 | 3× 오버헤드 |
| RS(4,2) | 150GB 저장 | 1.5× 오버헤드 |
| RS(6,3) | 150GB 저장 | 1.5× 오버헤드 |
| RS(10,4) | 140GB 저장 | 1.4× 오버헤드 |

Facebook은 2010년부터 Hadoop 클러스터에 RS(10,4) 코드를 적용하여 스토리지 비용을 3× 복제 대비 약 50% 절감했습니다.

## 왜 이레이저 코딩이 필요한가?

### 1. 스토리지 효율 vs 내결함성 딜레마

디스크 장애는 예외가 아니라 **정상**입니다. 수천 개의 디스크로 구성된 대규모 클러스터에서 매일 수 개의 디스크가 고장납니다. 이를 단순 3중 복제로 대응하면 스토리지 비용이 3배가 됩니다.

### 2. 실제 시스템에서의 채택

- **HDFS 3.x**: RS(6,3), RS(3,2) 지원
- **Ceph**: RS(k,m) 자유롭게 설정 가능
- **Azure Storage**: LRC(Local Reconstruction Codes) — RS의 변형
- **Google Colossus**: 전 세계 데이터센터에 RS 적용
- **Backblaze B2**: RS(17,3) 사용

## Reed-Solomon 코드의 수학적 원리

### 갈루아 필드(Galois Field, GF)

이레이저 코딩의 핵심은 **유한 체(Finite Field)** 또는 **갈루아 필드**에서의 연산입니다. 일반적으로 GF(2^8)을 사용하며, 이는 8비트 바이트(0~255)를 원소로 가집니다.

GF(2^8)에서의 덧셈은 **XOR 연산**이며, 곱셈은 다항식 곱셈을 원시 다항식으로 나눈 나머지입니다.

```python
# GF(2^8) 구현 - 원시 다항식: x^8 + x^4 + x^3 + x^2 + 1 (0x11d)
class GF256:
    PRIMITIVE_POLY = 0x11d  # x^8 + x^4 + x^3 + x^2 + 1
    
    def __init__(self):
        # 지수/로그 테이블 미리 계산 (빠른 곱셈용)
        self.exp_table = [0] * 512
        self.log_table = [0] * 256
        x = 1
        for i in range(255):
            self.exp_table[i] = x
            self.log_table[x] = i
            x = self._raw_mult(x, 2)
        for i in range(255, 512):
            self.exp_table[i] = self.exp_table[i - 255]
    
    def _raw_mult(self, a, b):
        result = 0
        while b:
            if b & 1:
                result ^= a
            a <<= 1
            if a & 0x100:
                a ^= self.PRIMITIVE_POLY
            b >>= 1
        return result & 0xFF
    
    def add(self, a, b):
        return a ^ b  # GF(2^8)에서 덧셈 = XOR
    
    def multiply(self, a, b):
        if a == 0 or b == 0:
            return 0
        return self.exp_table[self.log_table[a] + self.log_table[b]]
    
    def divide(self, a, b):
        if b == 0:
            raise ZeroDivisionError
        if a == 0:
            return 0
        return self.exp_table[self.log_table[a] - self.log_table[b] + 255]
    
    def inverse(self, a):
        return self.exp_table[255 - self.log_table[a]]

gf = GF256()
print(f"2 * 3 in GF(256) = {gf.multiply(2, 3)}")  # 6
print(f"5 + 5 in GF(256) = {gf.add(5, 5)}")         # 0 (자기 자신과의 XOR)
```

### 인코딩 행렬 (Encoding Matrix)

Reed-Solomon 인코딩은 **코시 행렬(Cauchy Matrix)** 또는 **반데르몬드 행렬(Vandermonde Matrix)** 기반의 인코딩 행렬을 사용합니다.

데이터 벡터 **d** = [d₀, d₁, ..., d_{k-1}]를 인코딩 행렬 **E** (k+m × k)와 곱하면 코드워드 **c** = [d₀, ..., d_{k-1}, p₀, ..., p_{m-1}]를 얻습니다.

```python
import numpy as np

class ReedSolomon:
    def __init__(self, k, m):
        """
        k: 데이터 블록 수
        m: 패리티 블록 수 (최대 m개 블록 손실까지 복구 가능)
        """
        self.k = k
        self.m = m
        self.gf = GF256()
        self.encode_matrix = self._build_encode_matrix()
    
    def _build_encode_matrix(self):
        """코시 행렬 기반 인코딩 행렬 구성"""
        # 상단 k×k는 단위 행렬 (체계적 코드)
        matrix = []
        # 단위 행렬 부분
        for i in range(self.k):
            row = [1 if i == j else 0 for j in range(self.k)]
            matrix.append(row)
        # 코시 행렬 패리티 부분
        # c[i][j] = 1 / (x_i XOR y_j), where x_i = i, y_j = k + j
        for i in range(self.m):
            row = []
            for j in range(self.k):
                xi = i
                yj = self.k + j
                row.append(self.gf.inverse(xi ^ yj))
            matrix.append(row)
        return matrix
    
    def encode(self, data_blocks):
        """
        data_blocks: k개의 데이터 블록 (각각 같은 크기의 바이트 배열)
        반환: k+m개의 블록 (원본 k개 + 패리티 m개)
        """
        block_size = len(data_blocks[0])
        result = [bytearray(block_size) for _ in range(self.k + self.m)]
        
        # 데이터 블록은 그대로 복사
        for i in range(self.k):
            result[i] = bytearray(data_blocks[i])
        
        # 패리티 블록 계산
        for i in range(self.k, self.k + self.m):
            for byte_idx in range(block_size):
                val = 0
                for j in range(self.k):
                    val = self.gf.add(
                        val,
                        self.gf.multiply(
                            self.encode_matrix[i][j],
                            data_blocks[j][byte_idx]
                        )
                    )
                result[i][byte_idx] = val
        
        return result
    
    def decode(self, received_blocks, available_indices):
        """
        received_blocks: 사용 가능한 블록들
        available_indices: 각 블록의 원래 인덱스
        최소 k개의 블록이 있어야 복구 가능
        """
        if len(available_indices) < self.k:
            raise ValueError(f"최소 {self.k}개의 블록이 필요합니다")
        
        # k개만 사용 (처음 k개 선택)
        indices = available_indices[:self.k]
        blocks = received_blocks[:self.k]
        
        # 선택된 인덱스에 해당하는 인코딩 행렬의 부분 행렬 추출
        sub_matrix = [self.encode_matrix[i] for i in indices]
        
        # 가우스 소거법으로 역행렬 계산 (GF(2^8) 위에서)
        inv_matrix = self._gf_matrix_inverse(sub_matrix)
        
        # 데이터 복구
        block_size = len(blocks[0])
        recovered = [bytearray(block_size) for _ in range(self.k)]
        
        for i in range(self.k):
            for byte_idx in range(block_size):
                val = 0
                for j in range(self.k):
                    val = self.gf.add(
                        val,
                        self.gf.multiply(inv_matrix[i][j], blocks[j][byte_idx])
                    )
                recovered[i][byte_idx] = val
        
        return recovered
    
    def _gf_matrix_inverse(self, matrix):
        """GF(2^8) 위에서 행렬 역원 계산 (가우스-조르단 소거법)"""
        n = len(matrix)
        # 확장 행렬 [matrix | I] 구성
        augmented = [row[:] + [1 if i == j else 0 for j in range(n)]
                     for i, row in enumerate(matrix)]
        
        for col in range(n):
            # 피벗 찾기
            pivot = -1
            for row in range(col, n):
                if augmented[row][col] != 0:
                    pivot = row
                    break
            if pivot == -1:
                raise ValueError("역행렬 없음: 행렬이 특이 행렬입니다")
            augmented[col], augmented[pivot] = augmented[pivot], augmented[col]
            
            # 피벗 정규화
            inv_pivot = self.gf.inverse(augmented[col][col])
            augmented[col] = [self.gf.multiply(x, inv_pivot) for x in augmented[col]]
            
            # 다른 행 소거
            for row in range(n):
                if row != col and augmented[row][col] != 0:
                    factor = augmented[row][col]
                    augmented[row] = [
                        self.gf.add(augmented[row][c],
                                    self.gf.multiply(factor, augmented[col][c]))
                        for c in range(2 * n)
                    ]
        
        return [row[n:] for row in augmented]


# 사용 예시: RS(4, 2) — 4개 데이터, 2개 패리티
rs = ReedSolomon(k=4, m=2)

# 원본 데이터 4블록 (각 4바이트)
original = [
    bytearray(b'Hell'),
    bytearray(b'o, W'),
    bytearray(b'orld'),
    bytearray(b'!   '),
]

# 인코딩
encoded = rs.encode(original)
print("인코딩된 블록:")
for i, block in enumerate(encoded):
    print(f"  블록 {i}: {bytes(block)}")

# 블록 0, 3 손실 시뮬레이션 (인덱스 1, 2, 4, 5 사용)
available = [encoded[i] for i in [1, 2, 4, 5]]
indices = [1, 2, 4, 5]

# 복구
recovered = rs.decode(available, indices)
print("\n복구된 데이터:")
for block in recovered:
    print(f"  {bytes(block)}")
```

## 실제 시스템에서의 최적화 기법

### 1. Intel ISA-L (Intelligent Storage Acceleration Library)

실제 프로덕션 시스템에서는 순수 Python이 아닌 **SIMD 명령어**를 활용한 최적화 라이브러리를 사용합니다. Intel ISA-L은 AVX2/AVX-512를 활용하여 GF(2^8) 연산을 CPU 레지스터 단위로 병렬 처리합니다.

```c
// Intel ISA-L을 사용한 Reed-Solomon 인코딩 (C)
#include <isa-l.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define DATA_BLOCKS   4
#define PARITY_BLOCKS 2
#define TOTAL_BLOCKS  (DATA_BLOCKS + PARITY_BLOCKS)
#define BLOCK_SIZE    (64 * 1024)  // 64KB per block

int main() {
    // 갈루아 필드 테이블 초기화
    unsigned char g_tbls[DATA_BLOCKS * PARITY_BLOCKS * 32];
    unsigned char encode_matrix[TOTAL_BLOCKS * DATA_BLOCKS];
    
    // 코시 행렬 기반 인코딩 행렬 생성
    gf_gen_cauchy1_matrix(encode_matrix, TOTAL_BLOCKS, DATA_BLOCKS);
    
    // SIMD 최적화용 갈루아 필드 테이블 사전 계산
    // encode_matrix의 패리티 부분(하단 PARITY_BLOCKS 행)만 필요
    ec_init_tables(DATA_BLOCKS, PARITY_BLOCKS,
                   &encode_matrix[DATA_BLOCKS * DATA_BLOCKS],
                   g_tbls);
    
    // 데이터/패리티 블록 버퍼 할당
    unsigned char *data[TOTAL_BLOCKS];
    for (int i = 0; i < TOTAL_BLOCKS; i++) {
        data[i] = (unsigned char *)aligned_alloc(64, BLOCK_SIZE);
    }
    
    // 데이터 블록 초기화 (예시)
    for (int i = 0; i < DATA_BLOCKS; i++) {
        memset(data[i], (char)(i + 1), BLOCK_SIZE);
    }
    
    // AVX2/AVX-512 활용 인코딩 — 수GB/s 처리량
    ec_encode_data(BLOCK_SIZE, DATA_BLOCKS, PARITY_BLOCKS,
                   g_tbls, data, &data[DATA_BLOCKS]);
    
    printf("RS(%d,%d) 인코딩 완료: %d블록 → %d블록\n",
           DATA_BLOCKS, PARITY_BLOCKS, DATA_BLOCKS, TOTAL_BLOCKS);
    
    // 블록 0 손실 시뮬레이션
    unsigned char *recovered[DATA_BLOCKS];
    recovered[0] = (unsigned char *)aligned_alloc(64, BLOCK_SIZE);
    
    // 손실 인덱스와 복구 행렬 계산
    unsigned char recovery_matrix[DATA_BLOCKS * DATA_BLOCKS];
    unsigned char recovery_tbls[DATA_BLOCKS * 1 * 32];
    
    // 패리티 블록 활용하여 데이터 블록 0 복구
    // (실제로는 gf_invert_matrix로 역행렬 계산 필요)
    
    // 정리
    for (int i = 0; i < TOTAL_BLOCKS; i++) free(data[i]);
    free(recovered[0]);
    return 0;
}
```

### 2. LRC(Local Reconstruction Codes) — Microsoft Azure의 선택

순수 RS 코드의 단점은 **단일 블록 복구 시 모든 k개의 블록을 읽어야 한다**는 점입니다. Azure Storage는 **지역 복구 코드(LRC)**를 도입하여 이 문제를 해결했습니다.

LRC(12, 2, 2)는 12개의 데이터 블록을 6개씩 두 그룹으로 나누고, 각 그룹에 로컬 패리티 2개씩, 전체 패리티 2개를 추가합니다. 단일 블록 손실 시 같은 그룹의 7개 블록(6 데이터 + 1 패리티)만으로 복구 가능합니다.

## 주의사항과 실전 팁

### 1. 인코딩/디코딩 CPU 오버헤드

이레이저 코딩은 복제 대비 CPU를 더 많이 사용합니다. 특히 소규모 랜덤 쓰기가 많은 워크로드에서는 복제가 더 유리할 수 있습니다. HDFS가 핫 데이터에는 복제, 콜드 데이터에는 이레이저 코딩을 사용하는 이유입니다.

### 2. 블록 크기 선택

블록 크기가 너무 작으면 메타데이터 오버헤드가 커지고, 너무 크면 단일 블록 복구 시 네트워크 트래픽이 증가합니다. 일반적으로 64KB~4MB 범위를 권장합니다.

### 3. 네트워크 지역성 고려

패리티 블록은 데이터 블록과 다른 렉(rack)이나 데이터센터에 배치하여 상관 장애(correlated failure)를 방지해야 합니다.

### 4. 스트라이프 폭 (k값) 선택

k값이 클수록 스토리지 효율은 높아지지만, 복구 시 읽어야 할 블록 수가 많아져 I/O 비용이 증가합니다. 일반적인 권장값: 소규모 클러스터는 RS(3,2), 대규모는 RS(6,3)~RS(10,4).

### 5. Silent Corruption 탐지

이레이저 코딩은 손실(erasure)에는 강하지만, **데이터 묵음 손상(silent corruption)**을 스스로 탐지하지 못합니다. 반드시 CRC-32C 또는 SHA-256 같은 체크섬과 함께 사용해야 합니다.

이레이저 코딩은 현대 분산 스토리지의 필수 기술입니다. GF(2^8) 위의 선형 대수학이라는 수학적 우아함과, SIMD를 통한 실용적 고성능 구현이 결합되어 수십 페타바이트 데이터를 비용 효율적으로 보호합니다.

## 참고 자료
- [Wikipedia: Erasure code](https://en.wikipedia.org/wiki/Erasure_code)
- [Wikipedia: Reed–Solomon error correction](https://en.wikipedia.org/wiki/Reed%E2%80%93Solomon_error_correction)
- [Intel ISA-L: Intelligent Storage Acceleration Library](https://github.com/intel/isa-l)
- [HDFS Erasure Coding — Apache Documentation](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HDFSErasureCoding.html)
