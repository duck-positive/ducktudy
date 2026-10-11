---
layout: post
title: "동형 암호화(Homomorphic Encryption) 완전 정복: 암호화된 데이터 위에서 연산하는 마법"
date: 2026-10-11
categories: [cs, computer-science]
tags: [homomorphic-encryption, fhe, ckks, bfv, lattice-cryptography, privacy, cryptography]
---

"클라우드 서버에 내 데이터를 암호화한 채로 보내고, 서버는 데이터를 복호화하지 않고도 필요한 연산을 수행한 뒤 결과만 돌려준다." 이것이 **동형 암호화(Homomorphic Encryption, HE)**의 꿈입니다. 의료 데이터 분석, 금융 서비스, 프라이버시 보존 머신러닝 등 데이터 보호가 필수인 시대에 동형 암호화는 점점 현실로 다가오고 있습니다.

## 동형 암호화란 무엇인가

수학적으로, 암호화 함수 `Enc`와 연산 `⊕`(덧셈), `⊗`(곱셈)에 대해 다음이 성립하는 암호 체계를 **동형(Homomorphic)** 암호라 합니다:

```
Enc(a) ⊕ Enc(b) = Enc(a + b)
Enc(a) ⊗ Enc(b) = Enc(a × b)
```

즉, 암호화된 데이터를 복호화하지 않고도 연산을 수행할 수 있으며, 결과를 복호화하면 평문에서 같은 연산을 수행한 것과 동일한 값이 나옵니다.

### 동형 암호화의 종류

| 유형 | 지원 연산 | 대표 스킴 |
|------|----------|----------|
| 부분 동형(PHE) | 덧셈 또는 곱셈 중 하나 | Paillier(덧셈), RSA(곱셈) |
| 준동형(SHE/LHE) | 제한된 횟수의 덧셈+곱셈 | 초기 BFV, BGV |
| 완전 동형(FHE) | 임의 회로(무한 연산) | TFHE, CKKS, BFV+부트스트래핑 |

2009년 Craig Gentry가 이론적으로 FHE의 가능성을 증명한 이후, 실용적인 구현이 꾸준히 등장하고 있습니다.

## 왜 필요한가: 실제 활용 사례

### 의료 데이터 분석

병원이 환자 데이터를 클라우드 AI 서비스로 분석을 의뢰할 때, 환자 프라이버시를 완전히 보호하면서도 AI 추론이 가능합니다. 암호화된 MRI 데이터를 복호화 없이 질병 분류 모델에 통과시킬 수 있습니다.

### 금융 서비스

신용 점수 계산, 부정 거래 탐지를 서드파티에 아웃소싱할 때, 원본 금융 데이터를 노출하지 않아도 됩니다.

### 프라이버시 보존 머신러닝 (PPML)

여러 기관이 각자의 데이터를 공유하지 않고도 공동으로 모델을 학습할 수 있습니다(연합 학습과 조합 가능).

## 격자 암호학 기초: 동형 암호의 수학적 토대

현대 FHE 스킴은 **격자(Lattice)** 위의 어려운 문제에 보안을 의존합니다.

### LWE (Learning With Errors) 문제

비밀 벡터 **s** ∈ Z_q^n과 오류 e ∈ Z_q를 숨기도록 설계된 암호화:

```
암호문: (a, b) = (a, a·s + e) mod q
복호화: b - a·s = e ≈ 0 (올바른 s로 복호화하면 오류 e만 남음)
```

오류 e가 작다면 메시지를 복구할 수 있지만, s를 모르는 공격자가 e를 추정하는 것은 계산적으로 불가능합니다(LWE 문제의 어려움).

### RLWE (Ring Learning With Errors)

LWE를 다항식 환(ring) 위에서 정의하여 효율성을 높인 버전:

```
R_q = Z_q[x] / (x^n + 1)  (n은 2의 거듭제곱)
암호문: (a, b) = (a, a·s + e) ∈ R_q × R_q
```

## 부분 동형 암호화: Paillier 구현

Paillier 암호화는 덧셈에 대한 동형성을 제공하는 대표적인 PHE입니다. 두 암호문의 **곱셈**이 평문의 **덧셈**에 해당합니다.

```python
import random
import math
from sympy import isprime, mod_inverse

def generate_prime(bits: int) -> int:
    """안전한 소수 생성"""
    while True:
        p = random.getrandbits(bits)
        p |= (1 << bits - 1) | 1  # 최상위 비트와 최하위 비트 설정
        if isprime(p):
            return p

def paillier_keygen(key_bits: int = 512):
    """Paillier 키 쌍 생성"""
    while True:
        p = generate_prime(key_bits // 2)
        q = generate_prime(key_bits // 2)
        if p == q:
            continue
        
        n = p * q
        n2 = n * n
        
        # λ = lcm(p-1, q-1)
        lam = (p - 1) * (q - 1) // math.gcd(p - 1, q - 1)
        
        # g = n + 1 선택 (최적화된 버전)
        g = n + 1
        
        # μ = (L(g^λ mod n²))^-1 mod n
        # L(x) = (x - 1) / n
        L = lambda x: (x - 1) // n
        mu = mod_inverse(L(pow(g, lam, n2)), n)
        
        if mu is None:
            continue
        
        public_key = (n, g)
        private_key = (lam, mu)
        return public_key, private_key

def paillier_encrypt(plaintext: int, pub_key: tuple) -> int:
    """평문을 암호화 (0 ≤ plaintext < n)"""
    n, g = pub_key
    n2 = n * n
    
    # 무작위 r 선택 (gcd(r, n) = 1 보장)
    while True:
        r = random.randint(2, n - 1)
        if math.gcd(r, n) == 1:
            break
    
    # c = g^m * r^n mod n²
    c = (pow(g, plaintext, n2) * pow(r, n, n2)) % n2
    return c

def paillier_decrypt(ciphertext: int, pub_key: tuple, priv_key: tuple) -> int:
    """복호화"""
    n, g = pub_key
    lam, mu = priv_key
    n2 = n * n
    L = lambda x: (x - 1) // n
    
    # m = L(c^λ mod n²) * μ mod n
    m = (L(pow(ciphertext, lam, n2)) * mu) % n
    return m

def paillier_add_encrypted(c1: int, c2: int, pub_key: tuple) -> int:
    """암호문끼리 곱 = 평문 덧셈"""
    n, _ = pub_key
    return (c1 * c2) % (n * n)

def paillier_add_plaintext(c: int, plaintext: int, pub_key: tuple) -> int:
    """암호문 + 평문 = 암호화된 합"""
    n, g = pub_key
    n2 = n * n
    return (c * pow(g, plaintext, n2)) % n2

def paillier_scalar_mul(c: int, scalar: int, pub_key: tuple) -> int:
    """암호문의 스칼라 배 = 평문 scalar배"""
    n, _ = pub_key
    return pow(c, scalar, n * n)

# 사용 예제: 프라이버시 보존 투표 집계
def privacy_preserving_vote(votes: list, pub_key: tuple, priv_key: tuple):
    """각 투표자는 자신의 표만 암호화하여 제출, 서버는 합산만 수행"""
    print("=== 프라이버시 보존 투표 집계 ===")
    encrypted_votes = [paillier_encrypt(v, pub_key) for v in votes]
    
    # 서버: 복호화 없이 암호화된 표 합산
    encrypted_total = encrypted_votes[0]
    for enc_vote in encrypted_votes[1:]:
        encrypted_total = paillier_add_encrypted(encrypted_total, enc_vote, pub_key)
    
    # 결과 복호화 (투표 관리자만 가능)
    total = paillier_decrypt(encrypted_total, pub_key, priv_key)
    print(f"실제 투표: {votes}")
    print(f"집계 결과: {total}표 (찬성)")
    print(f"검증:     {sum(votes) == total}")
    return total

pub_key, priv_key = paillier_keygen(key_bits=512)
votes = [1, 0, 1, 1, 0, 1, 1, 0, 1, 1]  # 1=찬성, 0=반대
privacy_preserving_vote(votes, pub_key, priv_key)
```

## CKKS: 실수 연산을 위한 근사 동형 암호화

CKKS(Cheon-Kim-Kim-Song, 2016)는 **실수와 복소수**의 근사 연산을 지원하는 FHE 스킴으로, 머신러닝 추론에 적합합니다.

```python
# Microsoft SEAL 또는 TenSEAL 라이브러리를 활용한 CKKS 예제
# pip install tenseal

import tenseal as ts
import numpy as np

def ckks_demo():
    """CKKS로 암호화된 벡터에서 내적(dot product) 계산"""
    
    # 1. CKKS 컨텍스트 생성
    context = ts.context(
        ts.SCHEME_TYPE.CKKS,
        poly_modulus_degree=8192,          # 보안 파라미터 (클수록 안전하지만 느림)
        coeff_mod_bit_sizes=[60, 40, 40, 60]  # 계수 모듈러스 체인
    )
    context.generate_galois_keys()  # 로테이션 연산용 갈루아 키
    context.global_scale = 2**40    # 스케일 (정밀도 결정)
    
    # 2. 데이터 암호화
    user_features = [0.5, 0.3, 0.8, 0.1, 0.9]   # 사용자 특성 벡터
    model_weights = [0.2, 0.7, 0.4, 0.6, 0.3]    # 서버의 모델 가중치
    
    enc_features = ts.ckks_vector(context, user_features)
    
    # 3. 서버 측: 암호화된 데이터로 내적 계산 (복호화 없이!)
    # 실제 환경에서는 서버가 평문 가중치와 암호화된 특성을 곱셈
    enc_dot_product = enc_features.dot(model_weights)
    
    # 4. 결과 복호화 (클라이언트 측)
    result = enc_dot_product.decrypt()
    expected = np.dot(user_features, model_weights)
    
    print(f"=== CKKS 동형 내적 연산 ===")
    print(f"암호화된 계산 결과: {result[0]:.6f}")
    print(f"평문 계산 결과:     {expected:.6f}")
    print(f"오차:               {abs(result[0] - expected):.2e}")
    
    return result[0]

def ckks_neural_network_inference():
    """암호화된 상태에서 단층 신경망 추론"""
    context = ts.context(
        ts.SCHEME_TYPE.CKKS,
        poly_modulus_degree=16384,
        coeff_mod_bit_sizes=[60, 40, 40, 40, 60]
    )
    context.generate_galois_keys()
    context.global_scale = 2**40
    
    # 단층 신경망 파라미터 (서버 보유)
    weights = np.random.randn(4, 4).tolist()
    bias = np.random.randn(4).tolist()
    
    # 클라이언트의 입력 데이터 (암호화하여 전송)
    x = [0.5, -0.3, 0.8, 0.1]
    enc_x = ts.ckks_vector(context, x)
    
    # 서버: 행렬 곱셈 + 편향 추가 (복호화 없이)
    enc_output = enc_x.matmul(weights) + bias
    
    # 근사 활성화 함수: ReLU 대신 제곱 함수 사용 (미분 가능한 다항식 근사)
    # 실제로는 다항식으로 근사된 sigmoid/tanh 사용
    enc_activated = enc_output.square()
    
    result = enc_activated.decrypt()
    print(f"\n=== 암호화 신경망 추론 결과 ===")
    print(f"암호화된 추론 결과: {[f'{r:.4f}' for r in result[:4]]}")
    return result

# ckks_demo()  # tenseal 설치 환경에서 실행
```

## 노이즈 관리와 부트스트래핑

FHE의 핵심 도전 과제는 **노이즈(Noise)**입니다.

- 각 연산(특히 곱셈)마다 암호문의 노이즈가 증가
- 노이즈가 임계값을 초과하면 복호화 실패
- **부트스트래핑(Bootstrapping)**: 노이즈가 큰 암호문을 동형 복호화하여 노이즈를 "새로고침"

```python
# 노이즈 레벨 시뮬레이션 (개념 설명용)
class SimplifiedFHEContext:
    def __init__(self, q: int, noise_bound: int):
        self.q = q               # 모듈러스
        self.noise_bound = noise_bound  # 허용 노이즈 상한
    
    def add(self, ct1, ct2):
        """덧셈: 노이즈 선형 증가"""
        a1, b1, noise1 = ct1
        a2, b2, noise2 = ct2
        result_noise = noise1 + noise2
        if result_noise > self.noise_bound:
            raise OverflowError(f"노이즈 초과! {result_noise} > {self.noise_bound}")
        return ((a1 + a2) % self.q, (b1 + b2) % self.q, result_noise)
    
    def multiply(self, ct1, ct2):
        """곱셈: 노이즈 제곱 증가 (매우 빠른 증가)"""
        a1, b1, noise1 = ct1
        a2, b2, noise2 = ct2
        result_noise = noise1 * noise2 + noise1 + noise2  # 단순화된 모델
        if result_noise > self.noise_bound:
            raise OverflowError(f"노이즈 초과! 부트스트래핑 필요")
        # 재선형화(Relinearization) 후 차수 축소
        return (a1 * b1 % self.q, b1 * b2 % self.q, result_noise)
    
    def bootstrap(self, ct):
        """부트스트래핑: 노이즈 초기화 (비용이 매우 큼)"""
        a, b, noise = ct
        print(f"부트스트래핑 수행: 노이즈 {noise} → 1 (재설정)")
        return (a, b, 1)  # 노이즈 초기화

ctx = SimplifiedFHEContext(q=2**64, noise_bound=2**32)
```

## 주요 FHE 라이브러리 생태계

| 라이브러리 | 지원 스킴 | 언어 | 특징 |
|-----------|---------|------|------|
| **Microsoft SEAL** | BFV, CKKS | C++/Python | 가장 널리 사용됨 |
| **OpenFHE** | BFV, BGV, CKKS, FHEW, TFHE | C++ | 여러 스킴 통합 지원 |
| **TenSEAL** | CKKS, BFV | Python | PyTorch 연동 특화 |
| **Concrete** | TFHE | Python/Rust | Zama, ML에 최적화 |
| **HEAAN** | CKKS | C++ | CKKS 원조 구현 |

## 성능 현실과 미래

### 현재 성능 한계

2026년 기준, FHE는 평문 연산 대비:
- **단순 덧셈**: ~100배 느림
- **곱셈 (한 레벨)**: ~1,000배 느림
- **부트스트래핑**: ~수만 배 느림 (수 초~수십 초)

### 하드웨어 가속의 부상

- **GPU 가속**: CUDA 기반 NTT(Number Theoretic Transform) 최적화
- **ASIC**: Intel의 HE Accelerator, DARPA의 DPRIVE 프로그램
- **FPGA**: 유연한 하드웨어 프로토타이핑

### 표준화 진행

ISO/IEC JTC1/SC27에서 동형 암호화 표준화가 진행 중이며, HomomorphicEncryption.org에서 보안 파라미터 표준을 제공합니다.

## 주의사항과 실전 고려사항

### CKKS는 근사 암호화

CKKS는 정확한 값이 아닌 **근사값**을 반환합니다. 금융 결제처럼 정확한 정수 연산이 필요한 경우 BFV/BGV를 사용해야 합니다.

### 파라미터 선택의 중요성

- `poly_modulus_degree` (n): 클수록 보안 강화, 성능 저하
- `coeff_mod_bit_sizes`: 허용 가능한 연산 레벨(깊이) 결정
- `scale`: CKKS 정밀도 조절

잘못된 파라미터 선택은 보안 취약점이나 계산 실패로 이어질 수 있습니다. 반드시 HomomorphicEncryption.org의 보안 표준을 참조하세요.

### 키 관리

공개 키, 비밀 키, 평가 키(Galois 키, 재선형화 키)를 적절히 관리해야 합니다. 비밀 키가 유출되면 모든 암호문이 복호화됩니다.

## 참고 자료

- [Wikipedia: Homomorphic encryption](https://en.wikipedia.org/wiki/Homomorphic_encryption)
- [Microsoft SEAL 공식 문서](https://github.com/microsoft/SEAL)
- [HomomorphicEncryption.org — 보안 파라미터 표준](https://homomorphicencryption.org)
- [OpenFHE 라이브러리](https://openfhe-development.readthedocs.io)
