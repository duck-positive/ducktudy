---
layout: post
title: "Paillier 준동형 암호화 완전 정복: 개인정보를 노출하지 않고 연산하는 가산적 준동형 암호의 수학적 원리"
date: 2026-10-04
categories: [cs, computer-science]
tags: [cryptography, homomorphic-encryption, privacy, number-theory, paillier, public-key]
---

## 개요

전자 투표 시스템을 구현한다고 가정해보자. 각 유권자의 투표 내용을 집계 서버에 전송해야 하는데, 서버가 개별 투표 내용을 알 수 없으면서도 최종 합산 결과를 알 수 있어야 한다. 의료 데이터 분석에서는 환자 정보를 병원이 직접 보지 않고 통계 연산을 수행해야 한다. **Paillier 준동형 암호화**는 바로 이런 문제를 해결하기 위해 1999년 Pascal Paillier가 제안한 공개키 암호 시스템이다.

Paillier는 **가산적 준동형(Additively Homomorphic)** 암호화 시스템이다. 즉, 두 암호문을 곱하면 그에 대응하는 평문의 합을 복호화할 수 있다. 완전 준동형 암호(FHE)가 임의의 연산을 지원하는 것과 달리, Paillier는 덧셈 연산만 지원하지만 그 대신 월등히 빠르고 실용적이다.

---

## 왜 Paillier가 필요한가

### 완전 준동형 암호(FHE)의 한계

FHE(Fully Homomorphic Encryption)는 암호화된 데이터에 대해 임의의 연산(덧셈, 곱셈 모두)을 지원하지만, 계산 비용이 극도로 높다. 10만 건의 정수 덧셈에 수분~수십분이 소요될 수 있다. 반면 Paillier는 다음 상황에서 FHE보다 수백 배 빠르다.

- **전자 투표**: 각 후보에 대한 투표를 암호화 후 집계 (합산만 필요)
- **연합 학습(Federated Learning)**: 각 기기의 그레이디언트를 암호화하여 중앙 서버에 전송, 서버는 합산만
- **의료 통계**: 환자 수, 평균 혈압 등 단순 집계
- **프라이버시 보존 경매**: 입찰가를 노출하지 않고 낙찰자 결정

---

## 수학적 기반

Paillier 암호화는 **합성 잉여류의 계산적 어려움(Decisional Composite Residuosity Assumption, DCRA)** 위에 구축된다.

### 핵심 수학 개념

**카마이클 함수(Carmichael's Function)**: λ(n) = lcm(p-1, q-1), 여기서 p, q는 큰 소수, n = p×q

**n²에서의 구조**: Z*_{n²} (n² 이하의 n²과 서로소인 정수들의 곱셈군)에는 특별한 구조가 있다. 임의의 원소 w ∈ Z*_{n²}는 유일하게 다음과 같이 표현된다:

```
w = g^m · r^n mod n²
```

여기서 m ∈ Z_n, r ∈ Z*_n이다. 이 m을 평문으로 해석하고 r을 임의의 난수(노이즈)로 사용하는 것이 Paillier 암호화의 핵심이다.

---

## 알고리즘 구현

### 키 생성

1. 큰 소수 p, q 생성 (각 512비트 이상)
2. n = p × q
3. λ = lcm(p-1, q-1)
4. g = n + 1 (또는 Z*_{n²}의 임의 원소)
5. μ = λ⁻¹ mod n (λ의 n에 대한 모듈러 역원)
6. 공개키: (n, g), 비밀키: (λ, μ)

### 암호화

평문 m ∈ Z_n에 대해 임의의 r ∈ Z*_n을 선택하여:

```
c = g^m · r^n mod n²
```

### 복호화

암호문 c에 대해:

```
m = L(c^λ mod n²) · μ mod n
```

여기서 L(x) = (x-1)/n 함수이다.

### 준동형 특성

암호문 c₁ = Enc(m₁), c₂ = Enc(m₂)에 대해:

```
c₁ · c₂ mod n² = Enc(m₁ + m₂)
```

---

## 코드 예제 1: Python으로 Paillier 구현

```python
import math
import random

def extended_gcd(a, b):
    if a == 0:
        return b, 0, 1
    g, x, y = extended_gcd(b % a, a)
    return g, y - (b // a) * x, x

def modinv(a, m):
    g, x, _ = extended_gcd(a % m, m)
    if g != 1:
        raise ValueError("역원이 존재하지 않습니다")
    return x % m

def is_prime(n, k=10):
    """Miller-Rabin 소수 판별"""
    if n < 2:
        return False
    if n == 2 or n == 3:
        return True
    if n % 2 == 0:
        return False
    d, r = n - 1, 0
    while d % 2 == 0:
        d //= 2
        r += 1
    for _ in range(k):
        a = random.randrange(2, n - 1)
        x = pow(a, d, n)
        if x == 1 or x == n - 1:
            continue
        for _ in range(r - 1):
            x = pow(x, 2, n)
            if x == n - 1:
                break
        else:
            return False
    return True

def generate_prime(bits):
    while True:
        p = random.getrandbits(bits) | (1 << bits - 1) | 1
        if is_prime(p):
            return p

class PaillierKeyPair:
    def __init__(self, bits=512):
        # 키 생성
        p = generate_prime(bits)
        q = generate_prime(bits)
        while p == q:
            q = generate_prime(bits)
        
        self.n = p * q
        self.n_sq = self.n * self.n
        self.g = self.n + 1  # 간소화된 g 선택
        
        lam = (p - 1) * (q - 1) // math.gcd(p - 1, q - 1)
        self._lambda = lam
        self._mu = modinv(lam, self.n)
    
    def encrypt(self, m):
        """평문 m을 암호화"""
        assert 0 <= m < self.n
        r = random.randrange(1, self.n)
        while math.gcd(r, self.n) != 1:
            r = random.randrange(1, self.n)
        
        # c = g^m * r^n mod n²
        c = (pow(self.g, m, self.n_sq) * pow(r, self.n, self.n_sq)) % self.n_sq
        return c
    
    def decrypt(self, c):
        """암호문 c를 복호화"""
        # L(c^λ mod n²) * μ mod n
        lc = pow(c, self._lambda, self.n_sq)
        L_val = (lc - 1) // self.n
        m = (L_val * self._mu) % self.n
        return m
    
    def add_encrypted(self, c1, c2):
        """두 암호문의 준동형 덧셈: Dec(c1 * c2) = m1 + m2"""
        return (c1 * c2) % self.n_sq
    
    def multiply_plain(self, c, k):
        """암호문에 평문 상수 k 곱셈: Dec(c^k) = m * k"""
        return pow(c, k, self.n_sq)


# 사용 예시
paillier = PaillierKeyPair(bits=256)  # 데모용 작은 키

m1, m2, m3 = 42, 17, 100

c1 = paillier.encrypt(m1)
c2 = paillier.encrypt(m2)
c3 = paillier.encrypt(m3)

# 준동형 덧셈: c1 * c2 * c3 → m1 + m2 + m3
c_sum = paillier.add_encrypted(paillier.add_encrypted(c1, c2), c3)
result = paillier.decrypt(c_sum)

print(f"m1={m1}, m2={m2}, m3={m3}")
print(f"암호화된 상태에서 합산 후 복호화: {result}")
print(f"검증 (직접 합산): {m1 + m2 + m3}")
assert result == m1 + m2 + m3, "준동형 덧셈 실패!"

# 준동형 상수 곱셈
k = 3
c_mul = paillier.multiply_plain(c1, k)
result_mul = paillier.decrypt(c_mul)
print(f"\nm1 * k = {m1} * {k} = {result_mul} (기댓값: {m1 * k})")
```

---

## 코드 예제 2: 전자 투표 시스템 시뮬레이션

실제 활용 사례로, 준동형 암호화를 이용한 전자 투표 집계를 구현한다. 각 유권자의 투표는 암호화된 채로 집계 서버에 전송되고, 서버는 개별 투표를 알 수 없다.

```python
class ElectionSystem:
    """
    Paillier 준동형 암호를 이용한 전자 투표 시스템
    - 각 후보는 비트 벡터로 표현: 후보 0에 투표하면 [1, 0, 0]
    - 집계 서버는 각 비트 위치의 암호문을 곱산하여 합산
    """
    
    def __init__(self, num_candidates, key_bits=256):
        self.num_candidates = num_candidates
        self.paillier = PaillierKeyPair(bits=key_bits)
        self.tallier_public_key = (self.paillier.n, self.paillier.g)
        
        # 집계 서버가 보관하는 누적 암호문
        # 초기값: Enc(0) = 1^n * r^n ≈ r^n (실제로는 0의 암호문)
        self.encrypted_tallies = [
            self.paillier.encrypt(0)
            for _ in range(num_candidates)
        ]
    
    def cast_vote(self, candidate_idx):
        """
        유권자가 candidate_idx 후보에 투표.
        서버는 encrypted_ballot만 받으며 내용을 알 수 없음.
        """
        # 비트 벡터: 선택한 후보만 1, 나머지 0
        ballot = [1 if i == candidate_idx else 0 
                  for i in range(self.num_candidates)]
        encrypted_ballot = [self.paillier.encrypt(b) for b in ballot]
        
        # 집계: 각 위치의 암호문을 누적 (준동형 덧셈)
        for i, enc_vote in enumerate(encrypted_ballot):
            self.encrypted_tallies[i] = self.paillier.add_encrypted(
                self.encrypted_tallies[i], enc_vote
            )
        
        return encrypted_ballot  # 영수증용
    
    def tally(self):
        """선거 종료 후 집계 — 비밀키로 복호화"""
        results = [self.paillier.decrypt(ct) for ct in self.encrypted_tallies]
        return results


# 시뮬레이션: 후보 3명, 투표자 10명
election = ElectionSystem(num_candidates=3)

# 투표 (집계 서버는 개별 투표 내용을 알 수 없음)
votes = [0, 0, 1, 2, 0, 1, 0, 2, 0, 1]
for v in votes:
    election.cast_vote(v)

# 개표
results = election.tally()
candidates = ["김민준", "이서연", "박지호"]
print("\n=== 개표 결과 ===")
for i, (name, count) in enumerate(zip(candidates, results)):
    print(f"{name}: {count}표")

# 검증
expected = [votes.count(i) for i in range(3)]
assert results == expected, "집계 오류!"
print(f"\n총 투표수: {sum(results)}, 검증 완료 ✓")
```

출력 예시:
```
=== 개표 결과 ===
김민준: 5표
이서연: 3표
박지호: 2표

총 투표수: 10, 검증 완료 ✓
```

---

## 보안 분석

### DCRA 가정의 의미

**DCRA(Decisional Composite Residuosity Assumption)**: n = p×q일 때, Z*_{n²}에서 임의의 원소 w가 n-th residue인지 아닌지를 다항식 시간 안에 구분할 수 없다는 가정이다.

이 가정이 깨지지 않는 한 Paillier는 **CPA(Chosen Plaintext Attack) 안전**을 보장한다.

### 의미론적 안전성 (Semantic Security)

암호화가 확률적(probabilistic)이기 때문에 같은 평문을 두 번 암호화해도 다른 암호문이 생성된다. 이로 인해 공격자는 "이 암호문이 m=0인가 m=1인가"를 구분하지 못한다.

```python
# 같은 평문, 다른 암호문 — 의미론적 안전성 시연
paillier_demo = PaillierKeyPair(bits=256)
m = 42
c_a = paillier_demo.encrypt(m)
c_b = paillier_demo.encrypt(m)
print(f"암호문 A: {c_a}")
print(f"암호문 B: {c_b}")
print(f"같은 암호문? {c_a == c_b}")  # False
print(f"복호화 A: {paillier_demo.decrypt(c_a)}")  # 42
print(f"복호화 B: {paillier_demo.decrypt(c_b)}")  # 42
```

### 주의사항과 한계

**정수 오버플로우**: 평문 합계가 n을 초과하면 자동으로 mod n이 적용되어 결과가 잘못된다. 연산 전에 항상 최대 합산값 < n임을 보장해야 한다.

**암호문 크기**: 암호문 크기가 평문의 2배(2048비트 키라면 4096비트 암호문)이다. 많은 양의 데이터를 다룰 때는 네트워크 비용을 고려해야 한다.

**곱셈 불가**: 두 암호문의 곱셈(평문의 곱셈에 해당)은 지원되지 않는다. 이를 위해서는 FHE 또는 안전한 다자간 연산(SMPC)이 필요하다.

**부동소수점 처리**: Paillier는 정수만 다룬다. 부동소수점을 다루려면 정수로 스케일링해야 한다 (예: 3.14 → 314, 스케일 팩터 100).

---

## 실무 팁

**라이브러리 활용**: 프로덕션 환경에서는 직접 구현 대신 검증된 라이브러리를 사용하라.
- Python: `python-paillier` 라이브러리 (`pip install paillier`)
- Java: `javallier`
- C++: `libpaillier`

**연합 학습 통합**: 딥러닝 프레임워크와 통합 시 그레이디언트 값을 정수로 스케일링하여 암호화하고, 중앙 서버에서 준동형 평균을 계산한 후 복호화하여 모델 업데이트에 활용한다.

**성능 최적화**: 실제 성능 병목은 암호화가 아니라 n²에 대한 모듈러 지수 계산이다. 중국인의 나머지 정리(CRT)를 활용하면 복호화 속도를 2~4배 향상시킬 수 있다.

---

## 참고 자료
- [Paillier, P. (1999). Public-Key Cryptosystems Based on Composite Degree Residuosity Classes — Wikipedia](https://en.wikipedia.org/wiki/Paillier_cryptosystem)
- [python-paillier Documentation](https://python-paillier.readthedocs.io/en/stable/)
- [OpenMined: Privacy-Preserving Machine Learning](https://openmined.org/)
- [Homomorphic Encryption Standardization — homomorphicencryption.org](https://homomorphicencryption.org/)
