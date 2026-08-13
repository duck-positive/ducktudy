---
layout: post
title: "양자 내성 암호학: CRYSTALS-Kyber와 Dilithium 심층 분석"
date: 2026-08-13
categories: [cs, computer-science]
tags: [cryptography, post-quantum, kyber, dilithium, lattice, nist, security]
---

양자 컴퓨터의 발전은 현재 인터넷 보안의 근간인 RSA, 타원곡선 암호(ECC), Diffie-Hellman 키 교환을 무력화할 위협을 가지고 있다. NIST(미국 국립표준기술연구소)는 2024년 8월, 양자 내성 암호(Post-Quantum Cryptography, PQC) 표준 3개를 최종 확정했다. 이 글에서는 그 핵심인 CRYSTALS-Kyber(ML-KEM)와 CRYSTALS-Dilithium(ML-DSA)의 수학적 기반과 실제 동작 원리를 심층적으로 분석한다.

## 왜 양자 내성 암호가 필요한가

### Shor 알고리즘의 위협

1994년 Peter Shor가 발표한 양자 알고리즘은 충분한 큐비트를 가진 양자 컴퓨터에서 RSA와 ECC를 다항 시간 내에 깰 수 있음을 증명했다. 현재 RSA-2048을 고전 컴퓨터로 인수분해하려면 우주 나이보다 긴 시간이 필요하지만, 이론적으로 약 4,000개의 논리 큐비트를 가진 양자 컴퓨터는 몇 시간 내에 이를 해결할 수 있다.

```
RSA 안전성 근거: 큰 수의 인수분해 어려움
  N = p × q (p, q는 큰 소수)
  고전 컴퓨터: O(exp(n^(1/3))) — 지수 시간
  양자 컴퓨터 (Shor): O(n^3)       — 다항 시간 ← 파괴적

ECC 안전성 근거: 이산 로그 문제
  Q = k × G (k를 G와 Q로부터 찾기 어려움)
  고전 컴퓨터: O(√n)
  양자 컴퓨터 (Shor): O(n^3)       — 다항 시간 ← 파괴적
```

"지금 수집해서 나중에 해독(Harvest Now, Decrypt Later)" 공격도 현실적 위협이다. 적대적 행위자가 현재 암호화된 트래픽을 대량 수집하고, 충분한 양자 컴퓨터가 등장했을 때 과거 데이터를 복호화하는 전략이다. 기밀 데이터의 유효 기간이 10~20년인 정부나 금융 기관에는 이미 위협이 현실화된 셈이다.

### Grover 알고리즘과 대칭키

대칭키 암호(AES)에 대해서는 Grover 알고리즘이 제곱근 속도 향상을 제공한다. AES-128은 양자 환경에서 64비트 수준으로 약화되므로, AES-256 사용이 양자 내성을 위한 권장 사항이 된다.

## 격자 기반 암호학의 수학적 기반

CRYSTALS 계열 알고리즘은 **격자(Lattice)** 위의 어려운 문제에 안전성을 의존한다.

### Learning With Errors (LWE) 문제

LWE는 격자 기반 암호학의 핵심 어려운 문제다.

```
[LWE 문제 정의]
- n차원 비밀 벡터 s ∈ Z_q^n
- 랜덤 행렬 A ∈ Z_q^(m×n)
- 작은 오류 벡터 e ∈ Z_q^m (각 원소가 작음)

공개: (A, b = A·s + e mod q)
비밀: s

문제: b가 균등 분포인지 A·s + e 형태인지 구별하라
     → 이 구별이 어려움 = LWE의 어려움
```

LWE는 최악의 경우(worst-case) 격자 문제인 SVP(Shortest Vector Problem)로 환원 가능하고, 현재 알려진 최선의 고전 및 양자 알고리즘 모두 지수 시간이 필요하다.

### Module-LWE (MLWE)

Kyber는 Module-LWE를 사용한다. 일반 LWE에서 정수 계수 대신 다항식 링 $R_q = \mathbb{Z}_q[X]/(X^n + 1)$의 원소를 사용하여 효율성을 높인다.

```
n = 256 (다항식 차수), q = 3329
k = 보안 레벨에 따라 2(512), 3(768), 4(1024)
```

## CRYSTALS-Kyber (ML-KEM, FIPS 203)

Kyber는 **키 캡슐화 메커니즘(Key Encapsulation Mechanism, KEM)**으로, TLS 키 교환에서 Diffie-Hellman을 대체한다.

### 키 생성 (KeyGen)

```python
import secrets
import hashlib

# 실제 Kyber 구현은 라이브러리를 사용하지만,
# 개념 이해를 위한 단순화된 예제

def kyber_keygen_concept(k=3, n=256, q=3329):
    """
    Kyber-768 (k=3) 키 생성 개념 설명
    """
    # 1. 시드에서 행렬 A 생성 (XOF = Extendable Output Function)
    seed_A = secrets.token_bytes(32)
    # A는 k×k 행렬, 각 원소는 R_q의 다항식
    print(f"행렬 A 시드: {seed_A.hex()[:16]}...")
    
    # 2. 비밀 벡터 s와 오류 벡터 e 샘플링 (작은 값)
    # 실제로는 centered binomial distribution 사용
    # s, e ∈ R_q^k, 계수는 작은 값 (-η ~ η)
    eta = 2  # Kyber-768의 경우
    
    # 3. 공개키 계산: pk = A·s + e
    # t = A·s + e (모든 연산은 R_q에서)
    
    # 4. 비밀키 = s, 공개키 = (seed_A, t)
    print(f"  비밀키 크기: {32 * k * n // 8 // k}바이트 (대략)")
    print(f"  공개키 크기: 1184바이트 (Kyber-768)")
    
    return {
        'pk_size': 1184,  # bytes for Kyber-768
        'sk_size': 2400,  # bytes for Kyber-768
        'ciphertext_size': 1088  # bytes for Kyber-768
    }

# 실제 사용: pqcrypto 또는 liboqs 라이브러리
def demo_kyber_with_liboqs():
    """
    liboqs-python 라이브러리를 사용한 실제 Kyber 예제
    pip install liboqs
    """
    try:
        import oqs
        
        # Kyber-768 KEM
        kem = oqs.KeyEncapsulation("Kyber768")
        
        # Alice: 키 생성
        public_key = kem.generate_keypair()
        print(f"공개키 크기: {len(public_key)} 바이트")
        
        # Bob: 공개키로 공유 비밀 캡슐화
        ciphertext, shared_secret_bob = kem.encap_secret(public_key)
        print(f"암호문 크기: {len(ciphertext)} 바이트")
        print(f"공유 비밀 크기: {len(shared_secret_bob)} 바이트")
        
        # Alice: 자신의 비밀키로 공유 비밀 복원
        shared_secret_alice = kem.decap_secret(ciphertext)
        
        assert shared_secret_alice == shared_secret_bob
        print("키 합의 성공! 동일한 공유 비밀 생성됨")
        print(f"공유 비밀 (첫 16바이트): {shared_secret_alice[:16].hex()}")
        
    except ImportError:
        print("liboqs가 설치되지 않은 환경. 개념 설명만 제공합니다.")
        sizes = kyber_keygen_concept()
        print(f"\nKyber-768 파라미터:")
        for k, v in sizes.items():
            print(f"  {k}: {v} bytes")

demo_kyber_with_liboqs()
```

### Kyber 캡슐화/복호화 원리

```
[캡슐화 — 송신자 Bob]
1. 랜덤 메시지 m ∈ {0,1}^256 생성
2. (K, r) = H(m || H(pk))  ← 결정적 랜덤성 생성
3. u = A^T · r + e1        ← 암호문 상단
4. v = t^T · r + e2 + ⌈q/2⌉·m  ← 암호문 하단
5. 공유 비밀 K = KDF(K)
6. 전송: 암호문 (u, v)

[복호화 — 수신자 Alice]
1. m' = 압축(v - s^T·u)
   = 압축(v - s^T·(A^T·r + e1))
   = 압축(t^T·r + e2 + ⌈q/2⌉·m - s^T·A^T·r - s^T·e1)
   ≈ 압축(⌈q/2⌉·m)  ← 오류항 e1, e2가 작으므로 소거됨
   = m
2. K = KDF(K), 검증 후 반환
```

## CRYSTALS-Dilithium (ML-DSA, FIPS 204)

Dilithium은 **디지털 서명** 알고리즘으로, RSA-PSS와 ECDSA를 대체한다.

### Fiat-Shamir with Aborts 기법

```python
def dilithium_sign_concept(message: bytes, secret_key: bytes) -> dict:
    """
    Dilithium 서명 개념 설명 (실제 구현은 liboqs 사용)
    
    Fiat-Shamir with Aborts 방식:
    1. 랜덤 마스킹 벡터 y 선택
    2. w = A·y (커밋)
    3. c = H(μ || w)  (도전)
    4. z = y + c·s1   (응답)
    5. 응답이 특정 범위를 벗어나면 재시도 (Abort)
    """
    
    # Dilithium3 파라미터
    params = {
        'n': 256,           # 다항식 차수
        'q': 8380417,       # 소수 (2^23 - 2^13 + 1)
        'k': 6,             # 공개키 행렬 행
        'l': 5,             # 공개키 행렬 열
        'gamma1': 2**19,    # 마스킹 범위
        'gamma2': (8380417-1)//32,  # 반올림 범위
        'tau': 49,          # 도전 다항식의 ±1 개수
        'beta': 196,        # 중간값 거부 임계
    }
    
    print("Dilithium3 서명 파라미터:")
    print(f"  공개키 크기: 1952 바이트")
    print(f"  비밀키 크기: 4000 바이트")
    print(f"  서명 크기: 3293 바이트")
    print(f"  비교 - RSA-3072 서명: 384 바이트 (더 작지만 양자 취약)")
    print(f"  비교 - Ed25519 서명: 64 바이트 (더 작지만 양자 취약)")
    
    return {
        'algorithm': 'Dilithium3 (ML-DSA-65)',
        'security_level': '128-bit quantum security',
        'assumption': 'Module-LWE + Module-SIS hardness'
    }

# OpenSSL 3.x 이상에서 PQC 지원 예제
def openssl_pqc_example():
    """
    OpenSSL 3.x + OQS Provider를 사용한 PQC TLS 예제
    """
    import subprocess
    
    print("OpenSSL + OQS Provider로 ML-DSA 키 생성:")
    print("$ openssl genpkey -algorithm mldsa65 -out private.pem")
    print("$ openssl pkey -in private.pem -pubout -out public.pem")
    print()
    print("ML-KEM을 사용한 TLS 1.3 핸드셰이크:")
    print("$ openssl s_server -groups mlkem768 ...")
    print("$ openssl s_client -groups mlkem768 ...")
    
    print("\n하이브리드 모드 (전통 + PQC, 권장):")
    print("$ openssl s_client -groups X25519MLKEM768 ...")
    # X25519MLKEM768: Diffie-Hellman + Kyber 동시 사용
    # 전통 알고리즘도 안전하다면 보안 유지
    # Kyber도 안전하다면 추가 보안 제공

openssl_pqc_example()
```

## 성능 비교 및 현황

| 알고리즘 | 공개키 | 비밀키/서명 | 보안 기반 | 양자 안전 |
|---------|--------|------------|---------|---------|
| RSA-2048 | 256B | 256B | 인수분해 | ❌ |
| Ed25519 | 32B | 64B | ECDLP | ❌ |
| ML-KEM-768 | 1184B | 2400B | MLWE | ✅ |
| ML-DSA-65 | 1952B | 3293B | MLWE+MSIS | ✅ |
| SLH-DSA-128s | 32B | 7856B | 해시함수 | ✅ |

NIST PQC 표준화 현황 (2024년 8월 최종 확정):
- **FIPS 203**: ML-KEM (CRYSTALS-Kyber 기반) — 키 교환
- **FIPS 204**: ML-DSA (CRYSTALS-Dilithium 기반) — 디지털 서명  
- **FIPS 205**: SLH-DSA (SPHINCS+ 기반) — 해시 기반 서명

## 주의사항 및 실전 팁

### 1. 하이브리드 배포 전략

순수 PQC로의 즉각 전환보다 **하이브리드 접근법**이 권장된다:

```
X25519MLKEM768 = X25519 (ECDH) + Kyber-768
→ 두 키 교환 결과를 KDF로 결합
→ 어느 하나라도 안전하면 전체 보안 유지
→ 과도기 구현에서 사이드 채널 공격 방어
```

### 2. 크기 고려사항

PQC 알고리즘은 키와 서명 크기가 훨씬 크다. DNS, IoT 디바이스, 인증서 체인에서 이 오버헤드를 고려해야 한다. TLS 핸드셰이크에서 Kyber 암호문(1088B)은 초기 데이터에 포함되어 1-RTT 지연 없이 전송 가능하다.

### 3. 구현 보안

격자 기반 암호는 수학적으로는 안전해도 구현 실수에 취약할 수 있다:
- **타이밍 공격**: 다항식 연산의 상수 시간 구현 필수
- **사이드 채널**: 전력 분석, EM 방사 주의 (임베디드 환경)
- **오류 주입**: 오류 파라미터의 올바른 분포 샘플링 검증

### 4. 마이그레이션 우선순위

```
즉시 필요: 장기 비밀 데이터, 인증서 인프라
곧 필요:   VPN, TLS, SSH 키 교환
나중:      서명 (양자 컴퓨터로 과거 서명 위조 불가)
```

양자 내성 암호로의 전환은 단순한 알고리즘 교체가 아니라 전체 PKI 인프라, 프로토콜, 하드웨어 가속기까지 영향을 미치는 거대한 마이그레이션이다. NIST 표준이 확정된 지금, 조직의 암호화 인벤토리를 파악하고 전환 계획을 수립하는 것이 시급한 과제다.

## 참고 자료
- [NIST Post-Quantum Cryptography Standardization](https://csrc.nist.gov/projects/post-quantum-cryptography/post-quantum-cryptography-standardization)
- [NIST Releases First 3 Finalized Post-Quantum Encryption Standards](https://www.nist.gov/news-events/news/2024/08/nist-releases-first-3-finalized-post-quantum-encryption-standards)
- [The Mathematical Foundation of Post-Quantum Cryptography](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12380341/)
- [NIST IR 8547: Transition to Post-Quantum Cryptography Standards](https://nvlpubs.nist.gov/nistpubs/ir/2024/NIST.IR.8547.ipd.pdf)
