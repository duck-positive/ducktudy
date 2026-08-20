---
layout: post
title: "동형 암호화(FHE) 완전 정복: 암호화된 채로 계산하는 마법 — BGV·CKKS·TFHE 심층 분석"
date: 2026-08-20
categories: [cs, computer-science]
tags: [homomorphic-encryption, FHE, CKKS, TFHE, BGV, cryptography, privacy, lattice-cryptography, LWE, RLWE]
---

현대 클라우드 컴퓨팅의 가장 근본적인 딜레마는 이것이다: 데이터를 서드파티 서버에 맡기면 그 서버는 데이터를 볼 수 있다. 암호화하면 서버가 계산을 할 수 없다. 이 두 요구 사이에서 동형 암호화(Homomorphic Encryption, HE)는 수십 년간 "꿈의 암호"로 불려 왔다. 암호문을 복호화하지 않고도 그 위에서 임의의 연산을 수행할 수 있고, 결과를 복호화하면 평문에서 연산한 것과 동일한 값이 나온다. 2009년 Craig Gentry가 최초의 완전 동형 암호화(Fully Homomorphic Encryption, FHE) 체계를 증명한 이후, 이 분야는 폭발적으로 발전해 실용 단계에 진입하고 있다.

## 동형 암호화란 무엇인가

동형 암호화는 암호화 함수 E와 연산 ○에 대해 다음 등식을 만족하는 시스템이다:

```
E(a) ○_enc E(b) = E(a ○_plain b)
```

즉, 암호문 E(a)와 E(b) 위에서의 암호문 연산 결과를 복호화하면, 평문 a와 b에 직접 연산한 결과 `a ○_plain b`와 동일해야 한다.

### 동형성의 종류

- **부분 동형 암호화(PHE, Partially HE)**: 덧셈 또는 곱셈 중 하나만 지원. Paillier는 덧셈 동형, 초기 RSA는 곱셈 동형.
- **다소 동형 암호화(Leveled HE, SWHE)**: 사전에 정해진 깊이의 회로까지만 지원. 곱셈 횟수에 제한이 있음.
- **완전 동형 암호화(FHE)**: Bootstrapping을 통해 임의 깊이의 회로를 평가 가능. 계산 횟수에 제한 없음.

## 왜 동형 암호화가 필요한가

**의료 데이터 분석**: 병원이 환자 데이터를 암호화한 채로 클라우드 AI 분석 서비스에 보내고, 서비스는 복호화 없이 진단 모델을 돌린다. 결과만 병원으로 돌아와 복호화된다. 환자 개인정보는 외부에 노출되지 않는다.

**프라이버시 보존 머신러닝(PPML)**: ML as a Service 업체가 모델 가중치를 알 필요 없이 사용자 데이터로 추론을 수행하거나, 사용자가 자신의 데이터를 공개하지 않고도 공유 모델로 학습을 진행한다.

**암호화된 데이터베이스 쿼리**: 클라우드 데이터베이스에 암호화된 채로 저장된 레코드에 대해, 서버가 복호화 없이 집계(SUM, AVG)나 필터링을 수행한다.

**금융 기밀 계산**: 여러 금융 기관이 개별 데이터를 공개하지 않고 공동으로 리스크 분석이나 사기 탐지 모델을 학습한다.

## 핵심 수학적 기반: LWE와 RLWE

현대 FHE의 보안은 **LWE(Learning With Errors)** 문제의 난해성에 의존한다.

### LWE 문제

비밀 벡터 **s** ∈ Zq^n 이 주어질 때, 다음을 구별하기 어렵다:
- 균등 랜덤 쌍 (A, b) ∈ Zq^(m×n) × Zq^m
- LWE 샘플 (A, b = As + e) — 여기서 e는 작은 에러 벡터

에러 e가 없으면 선형 연립방정식으로 s를 쉽게 구할 수 있다. 에러가 있으면 격자 문제로 환원되어 양자 컴퓨터로도 효율적인 알고리즘이 알려져 있지 않다.

### RLWE (Ring-LWE)

LWE는 키와 암호문 크기가 n²에 비례해 비실용적이다. RLWE는 다항식 환 R = Z[x]/(x^n + 1) 위에서 LWE를 정의해 키 크기를 O(n)으로 줄이고 NTT(Number Theoretic Transform)를 통해 곱셈을 O(n log n)에 계산한다.

## 주요 FHE 스킴

### BGV (Brakerski-Gentry-Vaikuntanathan, 2012)

정수 모듈러 산술(Zq)을 지원하며, 레벨마다 모듈러스 q를 줄여가는 모듈러스 체인 방식으로 노이즈를 관리한다. 배치(batching)를 통해 하나의 암호문에 n개의 정수 슬롯을 패킹해 SIMD 방식으로 병렬 계산한다. 현재 가장 널리 쓰이는 스킴 중 하나.

### CKKS (Cheon-Kim-Kim-Song, 2017)

부동소수점 근사 산술에 최적화되어 있다. 결과가 정확한 정수가 아니라 근사값이어도 되는 경우(통계, 머신러닝 추론)에 사용한다. 노이즈 자체를 "정밀도 손실"로 취급하여 BGV/BFV보다 훨씬 효율적이다.

특징:
- 실수(Real Number) 암호화 지원
- SIMD 배치: n/2개의 복소수 슬롯
- Rescaling: 곱셈 후 정밀도를 줄여 노이즈 관리

### TFHE (Torus FHE, 2016)

부울(Boolean) 게이트를 빠르게 평가하는데 특화되었다. 게이트 하나당 약 13ms (CPU), 최신 구현에서는 수 ms 이하까지 개선되었다. **Programmable Bootstrapping**을 지원해 임의의 단변수 함수를 bootstrapping과 동시에 평가할 수 있다.

특징:
- 게이트 단위 bootstrapping (연산마다 노이즈 초기화)
- 복잡한 제어 흐름(조건문, 루프)도 회로로 표현 가능
- 낮은 파라미터 수로 빠른 단일 게이트 실행

## 핵심 메커니즘: 노이즈와 Bootstrapping

FHE 암호문에는 항상 에러(노이즈)가 내포된다. 이 노이즈는 덧셈에서 선형 증가, 곱셈에서 제곱으로 증가한다. 노이즈가 복호화 임계값을 초과하면 복호화 실패가 발생한다.

**Bootstrapping**: Gentry의 핵심 아이디어. 복호화 회로 자체를 암호화된 채로 평가해 노이즈를 줄인다. 즉, 암호화된 비밀키를 가지고 암호화된 암호문을 복호화하는 회로를 FHE로 돌린다. 결과는 같은 평문을 가리키지만 "신선한" 낮은 노이즈의 암호문이 된다.

**Leveled HE vs FHE**: Bootstrapping 없이 일정 깊이의 회로만 평가 가능한 것을 Leveled HE, Bootstrapping으로 무한히 깊은 회로를 평가 가능한 것을 FHE라 한다. 실용적으로는 Leveled HE로도 많은 응용을 커버한다.

## 실제 구현 예제

### 예제 1: Python + TenSEAL 라이브러리로 CKKS 암호화 연산

```python
import tenseal as ts
import numpy as np

# CKKS 컨텍스트 생성
# poly_modulus_degree: 다항식 차수 (보안성과 성능에 영향)
# coeff_mod_bit_sizes: 계수 모듈러스 체인
context = ts.context(
    ts.SCHEME_TYPE.CKKS,
    poly_modulus_degree=8192,
    coeff_mod_bit_sizes=[60, 40, 40, 60]
)
context.generate_galois_keys()  # 회전(rotation) 연산에 필요
context.global_scale = 2**40   # 스케일 팩터 (정밀도 결정)

# 평문 벡터 암호화
plain_a = [1.5, 2.7, 3.1, 4.0]
plain_b = [0.5, 1.3, 2.9, 1.0]

enc_a = ts.ckks_vector(context, plain_a)
enc_b = ts.ckks_vector(context, plain_b)

# 암호문 위에서의 연산 (평문이 아니라 암호문끼리)
enc_sum = enc_a + enc_b       # 암호화된 덧셈
enc_product = enc_a * enc_b   # 암호화된 곱셈
enc_scaled = enc_a * 2.0      # 스칼라 곱

# 복호화 및 결과 확인
result_sum = enc_sum.decrypt()
result_product = enc_product.decrypt()
result_scaled = enc_scaled.decrypt()

print("평문 덧셈:", [a + b for a, b in zip(plain_a, plain_b)])
print("암호문 덧셈 복호화:", [round(x, 4) for x in result_sum])
# 출력: 평문 덧셈: [2.0, 4.0, 6.0, 5.0]
# 출력: 암호문 덧셈 복호화: [2.0, 4.0, 6.0, 5.0]  (근사값)

print("\n평문 곱셈:", [a * b for a, b in zip(plain_a, plain_b)])
print("암호문 곱셈 복호화:", [round(x, 4) for x in result_product])
# 출력: 평문 곱셈: [0.75, 3.51, 8.99, 4.0]
# 출력: 암호문 곱셈 복호화: [0.7500, 3.5100, 8.9900, 4.0000]  (근사값)

# CKKS 배치: n/2개 슬롯에 동시에 서로 다른 값을 병렬 처리
print("\n슬롯 수:", context.slot_count())  # 4096 (8192/2)
```

### 예제 2: 암호화된 선형 회귀 추론

```python
import tenseal as ts
import numpy as np

def create_fhe_context():
    """머신러닝 추론에 최적화된 CKKS 컨텍스트"""
    ctx = ts.context(
        ts.SCHEME_TYPE.CKKS,
        poly_modulus_degree=16384,
        coeff_mod_bit_sizes=[60, 40, 40, 40, 60]
    )
    ctx.generate_galois_keys()
    ctx.global_scale = 2**40
    return ctx

def fhe_linear_regression_inference(
    context,
    encrypted_features,  # 암호화된 입력 특성
    weights,             # 평문 모델 가중치 (서버가 보유)
    bias                 # 평문 편향값
):
    """
    복호화 없이 암호화된 데이터로 선형 회귀 추론.
    서버는 weights/bias를 알지만, encrypted_features의 실제 값은 모름.
    """
    result = encrypted_features.dot(weights)
    result = result + bias
    return result

# 클라이언트 측: 데이터 암호화 후 서버에 전송
context = create_fhe_context()

client_features = [0.5, 1.2, -0.3, 0.8]
enc_features = ts.ckks_vector(context, client_features)

# 서버 측: 평문 모델 가중치
server_weights = np.array([2.1, -0.5, 1.8, 0.9])
server_bias = 0.3

# 서버가 암호화된 데이터로 추론 수행 (복호화 없이!)
enc_prediction = fhe_linear_regression_inference(
    context, enc_features, server_weights, server_bias
)

# 클라이언트만 복호화 가능 (비밀키 보유)
prediction = enc_prediction.decrypt()[0]
expected = np.dot(client_features, server_weights) + server_bias

print(f"예측값 (FHE 복호화): {prediction:.6f}")
print(f"기댓값 (평문 계산):  {expected:.6f}")
print(f"오차:               {abs(prediction - expected):.2e}")
```

## 성능 최적화 기법

### 배치(Batching)와 SIMD

CKKS/BGV에서 하나의 암호문에 수천 개의 값을 패킹해 한 번의 연산으로 모두 처리한다. poly_modulus_degree = 8192이면 4096개 슬롯에 독립적인 값을 담아 4096개를 한 번에 처리한다.

### NTT (Number Theoretic Transform)

RLWE 기반 암호문의 곱셈은 다항식 곱셈인데, 이를 NTT를 사용해 O(n log n)에 계산한다. GPU 가속을 통해 NTT 속도를 수십 배 향상시킬 수 있다.

### 모듈러스 스위칭 (Modulus Switching)

BGV에서 곱셈 후 모듈러스 q를 q'(q' < q)로 줄여 노이즈 증가를 억제한다. 암호문의 계수가 작아지므로 저장 공간과 계산 비용도 줄어든다.

## 주의사항과 현실적 한계

**성능 오버헤드**: 현재 FHE는 평문 연산 대비 수천~수만 배 느리다. CKKS로 간단한 신경망 추론도 수초~수분이 걸릴 수 있다.

**메모리 사용**: 암호문 크기가 평문 대비 수백~수천 배 크다. 하나의 CKKS 암호문이 수십~수백 KB에 달한다.

**근사 오차**: CKKS는 근사 연산이므로 연산이 깊어질수록 오차가 누적된다. 금융 계산처럼 정확한 정수가 필요한 경우에는 BGV/BFV를 사용해야 한다.

**회로 깊이 제한**: Bootstrapping 없이는 곱셈 깊이에 제한이 있다. 깊은 신경망은 회로 깊이를 줄이는 재설계가 필요하다.

**비결정적 오류**: 파라미터를 잘못 선택하면 보안 수준이 낮아지거나 복호화 오류가 발생한다. homomorphicencryption.org의 보안 표준 파라미터 권고를 반드시 참고해야 한다.

## 참고 자료
- [OpenFHE 공식 문서 및 GitHub](https://github.com/openfheorg/openfhe-development)
- [IACR ePrint — CKKS 원논문 (Cheon et al., 2017)](https://eprint.iacr.org/2016/421)
- [Microsoft SEAL 라이브러리](https://github.com/microsoft/SEAL)
- [HElib (IBM) GitHub](https://github.com/homenc/HElib)
