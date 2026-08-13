---
layout: post
title: "보안 다자간 계산(MPC): Shamir의 비밀 공유와 가블드 서킷 심층 분석"
date: 2026-08-13
categories: [cs, computer-science]
tags: [cryptography, mpc, secret-sharing, garbled-circuit, privacy-preserving, distributed-computing, shamir]
---

"내 데이터를 공개하지 않고도 우리 모두의 데이터로 계산을 수행할 수 있을까?" 이 질문에 답하는 것이 **보안 다자간 계산(Secure Multi-Party Computation, MPC)**이다. 금융 기관들이 고객 정보를 공유하지 않고도 공통 사기꾼 목록을 만들거나, 의료 기관들이 환자 정보를 노출하지 않고도 통계를 집계하는 것이 MPC로 가능하다. 이 글에서는 MPC의 수학적 기반인 Shamir의 비밀 공유, 가블드 서킷, 그리고 실제 적용 사례를 심층 분석한다.

## MPC란 무엇인가

### 핵심 목표

n명의 참여자 P₁, P₂, ..., Pₙ이 각자 비밀 입력 x₁, x₂, ..., xₙ을 가지고 있을 때, 어떤 함수 f의 결과 f(x₁, ..., xₙ)을 계산하되 각자의 비밀 입력이 노출되지 않도록 한다:

```
목표: 모든 참여자가 f(x₁, x₂, ..., xₙ)을 알 수 있되
     각 xᵢ는 오직 Pᵢ만 알아야 함

예시 응용:
  - 경쟁사들이 가격 담합 없이 시장 평균가 계산
  - 병원들이 환자 정보 공유 없이 임상 통계 집계
  - 선거 시스템에서 투표 집계 (투표 비밀 보장)
  - 머신러닝 연합학습의 그래디언트 집계
  - 금융 리스크 계산 (각 기관의 포트폴리오 비공개)
```

### 보안 모델

```
정직하지만 호기심 있는 적(Semi-Honest / Honest-but-Curious):
  → 프로토콜은 따르지만 다른 참여자 정보를 엿보려 함
  → 실용적 MPC에서 주로 가정하는 위협 모델
  
악의적 적(Malicious / Byzantine):
  → 프로토콜을 임의로 위반하고 거짓 정보 제공 가능
  → 더 강한 보안, 더 높은 비용

부정 비율(Corruption Threshold):
  → t-out-of-n: t명 이하의 악의적 참여자 허용
  → BGW 프로토콜: n > 3t (비적응적 악의 모델)
  → Shamir SS (반정직): t < n/2
```

## Shamir의 비밀 공유 (Shamir's Secret Sharing)

### 수학적 원리: 다항식 보간

1979년 Adi Shamir가 제안한 비밀 공유는 **(t, n)-임계값 방식**이다. t개 이상의 쉐어가 있으면 비밀을 복원하고, t-1개 이하로는 아무 정보도 알 수 없다.

```python
import random
from functools import reduce
from typing import List, Tuple

# 소수 필드에서 연산 (실제로는 더 큰 소수 사용)
PRIME = 2**127 - 1  # Mersenne 소수

def mod_inverse(a: int, p: int) -> int:
    """확장 유클리드 알고리즘으로 모듈러 역원 계산"""
    def extended_gcd(a, b):
        if a == 0:
            return b, 0, 1
        gcd, x1, y1 = extended_gcd(b % a, a)
        return gcd, y1 - (b // a) * x1, x1
    
    _, x, _ = extended_gcd(a % p, p)
    return (x % p + p) % p


def shamir_split(secret: int, n: int, t: int, p: int = PRIME) -> List[Tuple[int, int]]:
    """
    비밀을 n개 쉐어로 분할, t개 이상이면 복원 가능
    
    원리:
    1. 랜덤 다항식 f(x) = secret + a₁x + a₂x² + ... + aₜ₋₁x^(t-1) (mod p)
    2. n개의 점 (i, f(i)) 를 각 참여자에게 배포
    3. t개의 점이면 다항식을 유일하게 결정 → secret = f(0)
    """
    assert t <= n, "임계값은 참여자 수 이하여야 합니다"
    assert 0 < secret < p
    
    # 랜덤 계수 생성 (t-1개)
    coefficients = [secret] + [random.randrange(1, p) for _ in range(t - 1)]
    
    def evaluate(x: int) -> int:
        """다항식 f(x) 계산 (Horner 방법)"""
        result = 0
        for coeff in reversed(coefficients):
            result = (result * x + coeff) % p
        return result
    
    # 각 참여자의 쉐어: (x좌표, y좌표)
    shares = [(i, evaluate(i)) for i in range(1, n + 1)]
    return shares


def shamir_reconstruct(shares: List[Tuple[int, int]], p: int = PRIME) -> int:
    """
    라그랑주 보간으로 비밀 복원
    
    f(0) = Σ yᵢ × Πⱼ≠ᵢ (0 - xⱼ)/(xᵢ - xⱼ)  (mod p)
    """
    def lagrange_basis(x_i: int, x_j_list: List[int]) -> int:
        """라그랑주 기저 다항식 L_i(0) 계산"""
        numerator = 1
        denominator = 1
        for x_j in x_j_list:
            if x_j != x_i:
                numerator = (numerator * (-x_j)) % p
                denominator = (denominator * (x_i - x_j)) % p
        return (numerator * mod_inverse(denominator, p)) % p
    
    x_coords = [x for x, _ in shares]
    secret = 0
    for x_i, y_i in shares:
        basis = lagrange_basis(x_i, x_coords)
        secret = (secret + y_i * basis) % p
    
    return secret


# 비밀 연산 (덧셈은 쉐어끼리 직접 가능!)
def mpc_addition_example():
    """
    MPC로 두 비밀값 합산 — 통신 없이 가능!
    
    핵심 특성: Shamir SS는 선형 (가법적)이다.
    if shares_A = [(i, f_A(i))]  와  shares_B = [(i, f_B(i))]
    then shares_A+B = [(i, (f_A(i) + f_B(i)) mod p)]  → f_A+B(0) = A + B
    """
    p = 2**13 - 1  # 데모용 작은 소수

    alice_secret = 150  # Alice의 비밀 (예: 연봉 150만)
    bob_secret = 200    # Bob의 비밀 (예: 연봉 200만)
    
    n, t = 3, 2  # 3명 중 2명으로 복원
    
    # 비밀 분할
    alice_shares = shamir_split(alice_secret, n, t, p)
    bob_shares = shamir_split(bob_secret, n, t, p)
    
    print(f"Alice 비밀: {alice_secret}")
    print(f"Bob 비밀:   {bob_secret}")
    print(f"실제 합: {alice_secret + bob_secret}\n")
    
    # 각 참여자가 자신의 쉐어를 더함 — 통신 불필요
    combined_shares = [
        (alice_shares[i][0], (alice_shares[i][1] + bob_shares[i][1]) % p)
        for i in range(n)
    ]
    
    # 2개 쉐어로 복원
    result = shamir_reconstruct(combined_shares[:2], p)
    print(f"MPC 결과 (쉐어 2개로 복원): {result}")
    print(f"검증: {result == (alice_secret + bob_secret) % p}")
    print(f"\n→ Alice와 Bob 모두 서로의 연봉을 모른 채로 합산 완료!")

mpc_addition_example()
```

## 가블드 서킷 (Garbled Circuit)

Shamir SS가 덧셈에 자연스럽지만 곱셈에 복잡한 반면, 가블드 서킷은 **임의의 불리언 회로**를 안전하게 계산한다. 1986년 Yao가 제안한 방식이다.

```python
import os
import hmac
import hashlib
from itertools import product

def garbled_circuit_and_gate_example():
    """
    AND 게이트 가블링 개념 예제
    
    2자간 계산 (2PC): Alice가 Garbler, Bob이 Evaluator
    Alice의 입력 a, Bob의 입력 b로 f(a, b) = a AND b 계산
    """
    
    # 각 와이어(입력/출력 선)에 두 개의 128비트 랜덤 레이블 배정
    # W0 = 0에 해당하는 레이블, W1 = 1에 해당하는 레이블
    def gen_wire_labels():
        return (os.urandom(16), os.urandom(16))
    
    # 와이어 레이블 생성
    wA0, wA1 = gen_wire_labels()  # Alice 입력 와이어
    wB0, wB1 = gen_wire_labels()  # Bob 입력 와이어
    wC0, wC1 = gen_wire_labels()  # 출력 와이어
    
    print("가블드 서킷 AND 게이트:")
    print(f"  입력 와이어 A: 레이블0={wA0.hex()[:8]}..., 레이블1={wA1.hex()[:8]}...")
    print(f"  입력 와이어 B: 레이블0={wB0.hex()[:8]}..., 레이블1={wB1.hex()[:8]}...")
    
    # AND 진리표를 암호화하여 가블드 테이블 생성
    # Enc(key_a, key_b, value) = AES(key_a||key_b, value)
    def encrypt(key_a: bytes, key_b: bytes, plaintext: bytes) -> bytes:
        key = hmac.new(key_a + key_b, digestmod=hashlib.sha256).digest()
        # 실제로는 AES-128 사용; 여기서는 XOR로 단순화
        return bytes(a ^ b for a, b in zip(key[:16], plaintext))
    
    # AND 게이트 진리표 암호화
    # a=0, b=0 → 0 (wC0)
    # a=0, b=1 → 0 (wC0)
    # a=1, b=0 → 0 (wC0)
    # a=1, b=1 → 1 (wC1)
    truth_table = [
        (wA0, wB0, wC0),  # 0 AND 0 = 0
        (wA0, wB1, wC0),  # 0 AND 1 = 0
        (wA1, wB0, wC0),  # 1 AND 0 = 0
        (wA1, wB1, wC1),  # 1 AND 1 = 1
    ]
    
    # 가블드 테이블: 암호화된 진리표 (순서 섞기)
    garbled_table = [encrypt(wA, wB, wC) for wA, wB, wC in truth_table]
    random.shuffle(garbled_table)  # 순서로 값 추측 방지
    
    print(f"\n  가블드 테이블 (암호화된 진리표, {len(garbled_table)}개 행):")
    for i, row in enumerate(garbled_table):
        print(f"    행 {i}: {row.hex()[:16]}...")
    
    # Oblivious Transfer로 Bob에게 입력 레이블 전달
    # (Bob의 입력값 노출 없이 해당 레이블 수신)
    print(f"\n  OT로 Bob의 입력 {1} → 레이블 전달 (Alice는 Bob 입력 모름)")
    
    # 평가: Bob이 자신의 레이블로 테이블 복호화 시도
    alice_input, bob_input = 1, 1
    alice_label = wA1 if alice_input else wA0
    bob_label = wB1 if bob_input else wB0
    
    print(f"\n  Alice 입력: {alice_input}, Bob 입력: {bob_input}")
    print(f"  f(A, B) = {alice_input} AND {bob_input} = {alice_input & bob_input}")
    print(f"\n  점 함수 보안:")
    print(f"    Bob은 자신의 입력(1)에 해당하는 레이블만 알고,")
    print(f"    Alice의 입력값은 모르지만 계산 결과는 알 수 있음")

garbled_circuit_and_gate_example()
```

### 망각 전송 (Oblivious Transfer, OT)

OT는 가블드 서킷에서 핵심 역할을 한다. 송신자가 여러 메시지 중 하나를 수신자의 선택에 따라 전송하되, 송신자는 어떤 것이 선택됐는지 모르고, 수신자는 선택하지 않은 메시지를 모른다.

```python
def simplified_1_of_2_ot_concept():
    """
    1-out-of-2 Oblivious Transfer 개념 설명
    
    Alice가 m0, m1을 가지고 있고
    Bob이 b ∈ {0,1}을 선택해 m_b를 받는다
    → Alice는 b를 모름, Bob은 m_{1-b}를 모름
    
    RSA 기반 구현 개념:
    """
    print("1-out-of-2 OT 프로토콜 (단순화):")
    print()
    print("1. Alice → Bob: 두 공개키 (k0, k1)")
    print("   (Alice가 k0 또는 k1 중 하나의 비밀키를 앎)")
    print()
    print("2. Bob → Alice: 자신의 선택 b로 암호화된 랜덤값 r")
    print("   Bob은 k_b로 r을 암호화 (Alice는 어느 키인지 모름)")
    print()
    print("3. Alice: 두 응답 생성")
    print("   c0 = m0 XOR H(decrypt(k0_sk, r) || 0)")
    print("   c1 = m1 XOR H(decrypt(k1_sk, r) || 1)")
    print("   → Alice는 k0, k1 중 하나의 비밀키만 알므로")
    print("     자신이 암호화한 것이 m0인지 m1인지 모름")
    print()
    print("4. Bob: c_b XOR H(r || b) = m_b 복원")
    print("   c_{1-b}는 r을 모르므로 복원 불가")
    print()
    print("결과:")
    print("  ✓ Alice: Bob의 선택 b를 모름")
    print("  ✓ Bob:   선택하지 않은 m_{1-b}를 모름")
    print("  ✓ Bob:   m_b를 올바르게 수신")

simplified_1_of_2_ot_concept()
```

## 실용적 MPC 프레임워크

### 현대 MPC 스택

```
┌──────────────────────────────────────────────┐
│           응용 계층                          │
│   연합학습, 프라이버시 분석, 금융 집계       │
├──────────────────────────────────────────────┤
│           프레임워크 계층                    │
│  MP-SPDZ, MOTION, ABY, Sharemind, SCALE-MAMBA│
├──────────────────────────────────────────────┤
│           프로토콜 계층                      │
│  SPDZ, GMW, BGW, ABY2.0, Overdrive          │
├──────────────────────────────────────────────┤
│           기본 기술                          │
│  Secret Sharing, Garbled Circuits, OT, ZK   │
└──────────────────────────────────────────────┘
```

### 실전 성능과 비용 분석

```python
def mpc_performance_analysis():
    """
    MPC 프로토콜 성능 비교 (개념적)
    """
    protocols = {
        'GMW (가블드 서킷)': {
            'rounds': 'O(depth)',      # 회로 깊이에 비례
            'comm': 'O(gates)',        # 게이트 수에 비례
            'model': '반정직',
            'strengths': '비교 연산 효율적',
            'weaknesses': '높은 통신 비용'
        },
        'BGW (비밀 공유)': {
            'rounds': 'O(depth)',
            'comm': 'O(n²·gates)',     # n² 참여자 통신
            'model': '악의적 (n > 3t)',
            'strengths': '정보이론적 보안, 덧셈 무료',
            'weaknesses': '곱셈에 상호작용 필요'
        },
        'SPDZ': {
            'rounds': 'O(1) 온라인',   # 오프라인 전처리 후
            'comm': 'O(gates) 온라인',
            'model': '악의적',
            'strengths': '실용적 성능, 악의 모델',
            'weaknesses': '오프라인 단계 비용'
        },
    }
    
    print("MPC 프로토콜 비교:\n")
    for name, props in protocols.items():
        print(f"[{name}]")
        for key, val in props.items():
            print(f"  {key}: {val}")
        print()
    
    print("연산별 비용 기준 (2자간, LAN 환경):")
    print("  덧셈:    ~0 ms  (Shamir SS에서 지역 연산)")
    print("  비교:    ~1 ms  (가블드 서킷)")
    print("  곱셈:    ~0.1 ms (SPDZ 온라인 단계)")
    print("  AES-128: ~100 ms (비교 게이트 6,800개)")
    print()
    print("vs. 동형 암호화 (FHE):")
    print("  덧셈:    ~0.01 ms")
    print("  곱셈:    ~1~10 ms")
    print("  AES-128: ~수십 초 (평문 대비 1,000,000x 느림)")
    print()
    print("→ MPC: 다자간 실시간 계산에 유리")
    print("→ FHE: 1자 연산, 서버가 연산하는 경우에 유리")

mpc_performance_analysis()
```

## 실제 응용 사례

### 프라이버시 보존 머신러닝

```python
def federated_learning_with_mpc():
    """
    연합학습 + MPC: 그래디언트 집계
    
    시나리오: n개 병원이 의료 모델 훈련
    → 각 병원의 환자 데이터는 로컬 유지
    → 그래디언트를 MPC로 안전하게 합산
    """
    print("MPC 기반 연합학습 그래디언트 집계:")
    print()
    print("라운드 t 진행:")
    print("  1. 서버 → 각 병원: 현재 모델 가중치 w_t")
    print("  2. 각 병원 i: 로컬 데이터로 그래디언트 g_i 계산")
    print("  3. MPC 집계 (Shamir SS 사용):")
    print("     - 병원 i: g_i를 n개 쉐어로 분할")
    print("     - 각 집계 서버 j: 자신에게 온 쉐어들을 더함")
    print("       (선형성에 의해 합의 쉐어 = 쉐어들의 합)")
    print("     - 서버들이 합 복원: G = Σ g_i")
    print("  4. 서버: w_{t+1} = w_t - η·G/n 업데이트")
    print()
    print("보안 보장:")
    print("  ✓ 개별 그래디언트 g_i가 노출되지 않음")
    print("  ✓ 멤버십 추론 공격 저항성 향상")
    print("  ✓ GDPR 준수 가능한 협력 학습")

federated_learning_with_mpc()
```

## 주의사항 및 실전 팁

### 1. 보안 가정 확인

MPC는 "완벽한 보안"이 아니다. 실제 시스템 설계 시 반드시 확인할 사항:

- **부정 임계값**: t명 이상 공모 시 보안 파괴. 현실적인 공모 가능성 평가 필수
- **입력 독립성**: 참여자들이 입력 조율 가능하면 MPC 자체가 의미 없음
- **함수 출력 프라이버시**: 출력값 자체가 입력을 추론하게 할 수 있음 (예: 평균 연봉 공개 → 이상치 추론)

### 2. 통신 비용이 병목

MPC의 주 병목은 네트워크 통신이다. WAN(광역망) 환경에서는 LAN 대비 수십~수백 배 느릴 수 있다. **오프라인-온라인 분리 전략**으로 데이터 독립적인 전처리(비버 트리플 생성 등)를 미리 수행하면 온라인 단계를 경량화할 수 있다.

### 3. 프레임워크 선택

| 사용 사례 | 추천 프레임워크 |
|----------|--------------|
| 연구/프로토타이핑 | MP-SPDZ |
| 2자간 고성능 | ABY / ABY2.0 |
| n자간 + 악의 모델 | SPDZ-2k, SCALE-MAMBA |
| 연합학습 | PySyft, TF Encrypted |
| 블록체인 통합 | Sharemind MPC |

### 4. 동형암호(FHE)와의 선택 기준

MPC와 FHE는 상호 보완적이다. **여러 파티가 실시간으로 상호작용**하는 경우 MPC가 유리하고, **한 클라우드가 암호화된 데이터로 연산**하는 경우 FHE가 유리하다. 최근에는 둘을 결합한 FHE-MPC 하이브리드 방식도 연구되고 있다.

MPC는 "데이터 없이 데이터로부터 배우는" 미래 프라이버시 보존 컴퓨팅의 핵심 기술이다. 양자 내성 암호화가 데이터를 저장할 때 지키는 것이라면, MPC는 데이터를 **사용할 때** 지키는 기술이다. 점점 중요해지는 개인정보 보호 규제와 AI 확산 시대에 그 중요성은 더욱 커질 것이다.

## 참고 자료
- [An Introduction to Secret-Sharing-Based Secure Multiparty Computation (IACR)](https://eprint.iacr.org/2022/062.pdf)
- [Concretely Efficient Secure Multi-Party Computation Protocols](https://sands.edpsciences.org/articles/sands/pdf/2022/01/sands20210001.pdf)
- [MP-SPDZ: A Versatile Framework for Multi-Party Computation](https://github.com/data61/MP-SPDZ)
- [Secure Computation — Yao's Garbled Circuits (Wikipedia)](https://en.wikipedia.org/wiki/Garbled_circuit)
