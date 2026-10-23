---
layout: post
title: "샤미르의 비밀 분산 완전 정복: 라그랑주 보간법으로 구현하는 임계값 암호학"
date: 2026-10-23
categories: [cs, computer-science]
tags: [cryptography, secret-sharing, shamir, threshold-cryptography, polynomial-interpolation, finite-field]
---

## 개요

키 관리의 가장 오래된 딜레마 중 하나는 "비밀 키를 어디에 저장할 것인가?"입니다. 한 곳에 저장하면 단일 실패 지점(Single Point of Failure)이 생기고, 여러 곳에 복사하면 노출 위험이 커집니다. 1979년 Adi Shamir가 발표한 **비밀 분산(Secret Sharing)** 방식은 이 딜레마를 우아하게 해결합니다. n명 중 k명 이상이 협력해야만 비밀을 복원할 수 있는 **(k, n) 임계값 방식**을 수학적으로 구현하는 방법을 이 글에서 자세히 살펴봅니다.

---

## 1. 문제 정의와 직관적 이해

### 단순 복사의 문제

암호화 키를 세 개의 금고에 나눠 보관한다고 상상해보세요. 각 금고가 독립적으로 안전하다면, 하나를 열어도 키를 알 수 없어야 합니다. 하지만 단순히 키를 세 군데 복사하면, 하나만 털려도 키 전체가 노출됩니다.

이상적인 방식은:
- **n개**의 조각으로 분산: 어떤 한 조각(또는 k-1개)만으로는 비밀에 관해 **아무런 정보도 얻을 수 없다**
- **k개** 이상이 모이면 비밀을 **완전히 복원**한다

이것이 **(k, n)-임계값 비밀 분산**의 핵심 요건입니다.

### 직관: 두 점은 직선을 결정한다

중학교 수학을 떠올려 보세요. **두 점은 하나의 직선을 유일하게 결정합니다.** 반대로, 점이 하나만 있으면 그 점을 지나는 직선이 무한히 많습니다.

Shamir의 아이디어는 이것을 일반화한 것입니다:
- **k개의 점은 차수 k-1인 다항식을 유일하게 결정합니다.**
- 비밀 `s`를 다항식의 상수항(y절편)에 숨기고, n개의 좌표 쌍 `(i, f(i))`를 조각으로 나눠줍니다.
- k개 이상의 조각이 모이면 라그랑주 보간으로 다항식을 복원하고 `f(0) = s`를 얻습니다.

---

## 2. 수학적 배경: 유한 체 위의 라그랑주 보간

### 왜 실수 대신 유한 체를 쓰는가?

실수 위에서 구현하면 **부동소수점 오차** 때문에 정확한 복원이 어렵고, 조각 값이 비밀 크기보다 커질 수 있어 정보 이론적 안전성이 깨집니다. 따라서 소수 `p`를 법으로 하는 **유한 체 GF(p)** (또는 GF(2^8))에서 연산합니다.

GF(p)에서의 나눗셈은 **모듈러 역원**으로 대체됩니다:
```
a / b ≡ a × b^(p-2) mod p   (페르마의 소정리, p는 소수)
```

### 라그랑주 보간 공식

k개의 점 `(x₁, y₁), ..., (xₖ, yₖ)`가 주어졌을 때, `f(0)`은 다음과 같이 계산합니다:

```
f(0) = Σᵢ [yᵢ × Lᵢ(0)]  (mod p)

Lᵢ(0) = Π_{j≠i} [ xⱼ / (xⱼ - xᵢ) ]  (mod p)
```

각 `Lᵢ(0)`는 **라그랑주 기저 다항식**의 0에서의 값으로, 나머지 조각들의 인덱스만 알면 미리 계산할 수 있습니다.

---

## 3. Python 구현

### 예제 1: 기본 (k, n) 비밀 분산 구현

```python
import secrets
from typing import List, Tuple

# 충분히 큰 소수 (256비트 이상 권장)
PRIME = (1 << 127) - 1  # 메르센 소수 2^127 - 1

def _mod_inverse(a: int, m: int) -> int:
    """페르마의 소정리로 모듈러 역원 계산 (m이 소수일 때)"""
    return pow(a, m - 2, m)

def make_shares(secret: int, k: int, n: int, prime: int = PRIME) -> List[Tuple[int, int]]:
    """비밀을 n개의 조각으로 분산, k개 이상 있어야 복원 가능"""
    assert 0 < k <= n
    assert 0 <= secret < prime

    # 차수 k-1인 다항식의 계수 생성: f(x) = secret + a1*x + a2*x^2 + ...
    coefficients = [secret] + [secrets.randbelow(prime) for _ in range(k - 1)]

    def evaluate_poly(x: int) -> int:
        result = 0
        for i, coef in enumerate(coefficients):
            result = (result + coef * pow(x, i, prime)) % prime
        return result

    # 1부터 n까지 점 평가
    return [(i, evaluate_poly(i)) for i in range(1, n + 1)]

def recover_secret(shares: List[Tuple[int, int]], prime: int = PRIME) -> int:
    """라그랑주 보간으로 f(0) 복원"""
    k = len(shares)
    secret = 0

    for i, (xi, yi) in enumerate(shares):
        # 라그랑주 기저 다항식 Li(0) 계산
        numerator = 1
        denominator = 1
        for j, (xj, _) in enumerate(shares):
            if i != j:
                numerator = (numerator * (-xj)) % prime
                denominator = (denominator * (xi - xj)) % prime

        lagrange_coef = (numerator * _mod_inverse(denominator, prime)) % prime
        secret = (secret + yi * lagrange_coef) % prime

    return secret

# 사용 예시: (3, 5) 분산 — 5개 조각 중 3개 이상이면 복원
secret = 0xDEADBEEFCAFEBABE  # 복원할 비밀값
shares = make_shares(secret, k=3, n=5)

print(f"원본 비밀: 0x{secret:X}")
print(f"생성된 조각 수: {len(shares)}")

# 조각 3개(인덱스 0, 2, 4)로 복원
subset = [shares[0], shares[2], shares[4]]
recovered = recover_secret(subset)
print(f"복원된 비밀: 0x{recovered:X}")
print(f"복원 성공: {recovered == secret}")

# 조각 2개(k-1)만으로 시도 — 잘못된 값이 나옴
bad_subset = [shares[0], shares[1]]
bad_recovered = recover_secret(bad_subset)
print(f"조각 2개만으로 복원: 0x{bad_recovered:X} (원본과 다름)")
```

### 예제 2: 문자열 비밀(마스터 키) 분산 실전 예제

```python
import hashlib
import base64

def string_to_int(s: str) -> int:
    """UTF-8 문자열을 정수로 변환"""
    return int.from_bytes(s.encode('utf-8'), 'big')

def int_to_string(n: int, byte_len: int) -> str:
    """정수를 UTF-8 문자열로 복원"""
    return n.to_bytes(byte_len, 'big').decode('utf-8')

def share_master_key(master_key: str, k: int, n: int) -> List[str]:
    """마스터 키를 n개 조각으로 분산 (Base64 인코딩)"""
    key_bytes = master_key.encode('utf-8')
    key_int = int.from_bytes(key_bytes, 'big')
    byte_len = len(key_bytes)

    raw_shares = make_shares(key_int, k, n)
    # (x, y, byte_len)을 인코딩하여 보관
    encoded = []
    for x, y in raw_shares:
        chunk = f"{x}:{y}:{byte_len}"
        encoded.append(base64.b64encode(chunk.encode()).decode())
    return encoded

def recover_master_key(encoded_shares: List[str]) -> str:
    """인코딩된 조각으로 마스터 키 복원"""
    decoded = []
    byte_len = None
    for enc in encoded_shares:
        chunk = base64.b64decode(enc).decode()
        x, y, bl = chunk.split(':')
        decoded.append((int(x), int(y)))
        byte_len = int(bl)

    key_int = recover_secret(decoded)
    return int_to_string(key_int, byte_len)

# 실전 시나리오: HSM 마스터 키를 5명의 임원에게 분산
master_key = "MASTER_KEY_v1_prod_2026"
n_shares = share_master_key(master_key, k=3, n=5)

print("=== 마스터 키 분산 ===")
for i, share in enumerate(n_shares):
    print(f"임원 {i+1}의 조각: {share[:30]}...")

print("\n=== 3명의 임원이 모여 복원 ===")
# 임원 1, 3, 5의 조각으로 복원
recovered_key = recover_master_key([n_shares[0], n_shares[2], n_shares[4]])
print(f"복원된 마스터 키: {recovered_key}")
print(f"복원 성공: {recovered_key == master_key}")

# 무결성 검증 (실제 시스템에서는 MAC/HMAC 사용 권장)
expected_hash = hashlib.sha256(master_key.encode()).hexdigest()[:16]
actual_hash   = hashlib.sha256(recovered_key.encode()).hexdigest()[:16]
print(f"해시 일치: {expected_hash == actual_hash}")
```

---

## 4. 실제 활용 사례

### 4.1 HSM(Hardware Security Module) 마스터 키 관리

금융기관·클라우드 사업자는 HSM 마스터 키를 Shamir 방식으로 분산 보관합니다. AWS CloudHSM, Vault(HashiCorp)의 Shamir unseal 기능이 대표적입니다.

### 4.2 블록체인 다중 서명(Multi-sig)

이더리움의 Gnosis Safe 등은 m-of-n 다중 서명을 Shamir 기반으로 구현하지 않고 온체인 컨트랙트로 처리하지만, 개인 키를 안전하게 나눠 보관하는 데 Shamir를 사용합니다.

### 4.3 Distributed Key Generation (DKG)

임계값 서명(Threshold Signature)을 구현할 때 Shamir를 확장하여 키 생성 자체를 분산합니다. 이 경우 어느 한 참여자도 전체 키를 알지 못한 채로 서명이 생성됩니다.

---

## 5. 주의사항과 한계

### 정보 이론적 안전성 vs. 무결성

Shamir의 비밀 분산은 **완벽한 비밀성(perfect secrecy)**을 제공합니다. k-1개 이하의 조각으로는 비밀에 관해 어떤 정보도 얻을 수 없습니다. 그러나 **무결성(integrity)**은 제공하지 않습니다. 조작된 조각을 가져와도 다른 값이 복원될 뿐이며, 조각이 유효한지 검증할 방법이 없습니다.

해결책: **Verifiable Secret Sharing(VSS)** — 각 조각과 함께 증명 정보를 배포하여 조각의 정당성을 검증합니다(Feldman VSS, Pedersen VSS).

### 딜러 신뢰 문제

표준 Shamir 방식에서는 초기 조각을 생성하는 **딜러가 전체 비밀을 알고 있습니다.** 딜러를 신뢰할 수 없는 경우 Distributed Key Generation(DKG)을 사용해 딜러 없이 조각을 생성해야 합니다.

### 조각 재사용 금지

동일한 비밀을 두 번 분산할 때 같은 다항식을 재사용하면 공격자가 조각 조합으로 다항식을 추론할 수 있습니다. 반드시 새로운 랜덤 계수를 사용해야 합니다.

### GF(256) 구현 권고

실용 시스템에서는 바이트 배열 형태의 비밀을 처리하기 위해 `GF(2^8)` 체 위에서 구현하는 것이 일반적입니다. `sss`(Daan Sprenkels), `tink`(Google) 같은 라이브러리가 이를 지원합니다.

---

## 참고 자료

- [Shamir, A. (1979). "How to Share a Secret" — 원본 논문 (MIT 코스 자료)](https://courses.csail.mit.edu/6.857/2015/files/L08-Shamir-HotToShareASecret.pdf)
- [Feldman, P. (1987). "A Practical Scheme for Non-interactive Verifiable Secret Sharing"](https://www.cs.umd.edu/~gasarch/TOPICS/secretsharing/feldmanVSS.pdf)
- [HashiCorp Vault — Shamir Seal Documentation](https://developer.hashicorp.com/vault/docs/concepts/seal)
- [NIST SP 800-57 — Recommendation for Key Management](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf)
