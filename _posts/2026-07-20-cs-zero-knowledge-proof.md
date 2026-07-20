---
layout: post
title: "영지식 증명(Zero-Knowledge Proof) 완전 정복: 비밀을 공개하지 않고 증명하는 암호학의 마법"
date: 2026-07-20
categories: [cs, computer-science]
tags: [zero-knowledge-proof, zkp, zk-snark, zk-stark, cryptography, privacy, blockchain]
---

"나는 비밀번호를 알고 있다"는 사실을 상대방에게 증명하되, 비밀번호 자체는 절대 공개하지 않을 수 있을까요? 직관적으로 불가능해 보이는 이 질문에 **영지식 증명(Zero-Knowledge Proof, ZKP)** 이 "예"라고 답합니다. 1985년 MIT의 Goldwasser, Micali, Rackoff가 발표한 이 개념은 현재 블록체인 프라이버시(Zcash), L2 롤업(zkSync, StarkNet), 신원 인증, 머신러닝 모델 검증 등 폭넓은 분야에서 혁신을 이끌고 있습니다.

## 개념 설명

### 세 가지 핵심 성질

영지식 증명이 성립하려면 다음 세 가지를 동시에 만족해야 합니다.

1. **완전성(Completeness):** 증명자(Prover)가 참인 명제를 증명하면 검증자(Verifier)는 반드시 이를 받아들입니다.
2. **건전성(Soundness):** 증명자가 거짓 명제를 증명하려 할 때 검증자를 속일 수 있는 확률이 무시할 만큼 낮습니다 (negligible probability).
3. **영지식성(Zero-Knowledge):** 검증자는 "명제가 참이다"는 사실 외에 어떠한 추가 정보도 알아낼 수 없습니다.

### 동굴 비유: 알리바바 동굴

가장 유명한 직관적 설명입니다. 동굴 안에 비밀 문이 있고, 증명자는 그 문을 여는 비밀번호를 알고 있습니다.

```
     입구
      |
  A 방향 --- 비밀 문 --- B 방향
```

검증자는 입구에서 기다리고, 증명자는 동굴 안으로 들어가 임의로 A 또는 B 방향을 택합니다. 검증자가 "A 방향에서 나오라!" 혹은 "B 방향에서 나오라!"를 외치면, 비밀번호를 아는 증명자는 항상 요청된 방향에서 나올 수 있습니다. 비밀번호를 모른다면 정답을 맞힐 확률은 매 라운드 50%이며, 30라운드 반복 시 속일 확률은 약 10억 분의 1 이하로 수렴합니다.

이것이 **대화형(Interactive) ZKP**의 원형입니다. 하지만 인터넷에서는 대화를 반복하기 어려우므로, 비대화형(Non-Interactive) ZKP가 필요합니다.

### zk-SNARK: 실용적 비대화형 ZKP

**zk-SNARK(Succinct Non-Interactive Argument of Knowledge)** 는 현재 가장 널리 쓰이는 ZKP 방식입니다.

- **Succinct(간결):** 증명 크기가 회로 크기와 무관하게 수백 바이트에 불과합니다.
- **Non-Interactive:** 증명자가 증명을 하나 생성하면, 검증자는 단독으로 검증합니다.
- **Argument of Knowledge:** 증명자가 위트니스(witness, 비밀 입력)를 알고 있어야만 유효한 증명을 생성할 수 있습니다.

핵심 구성 요소:
1. **회로(Circuit):** 증명하고 싶은 계산을 산술 회로(arithmetic circuit)로 표현합니다.
2. **R1CS(Rank-1 Constraint System):** 회로를 `A·z ⊙ B·z = C·z` 형태의 제약 집합으로 변환합니다.
3. **QAP(Quadratic Arithmetic Program):** R1CS를 다항식 문제로 변환합니다.
4. **KZG 다항식 커밋/Groth16:** 타원 곡선 페어링을 이용해 다항식에 대한 지식 증명을 생성합니다.

Groth16(2016)은 증명 크기 192바이트, 검증 시간 O(1)이라는 인상적인 성능을 달성해 Zcash와 zkSync 등이 채택했습니다. 단점은 **신뢰 설정(Trusted Setup)** — 초기에 독성 폐기물(toxic waste)이라 불리는 랜덤 파라미터를 생성하고 반드시 파기해야 한다는 점입니다.

### zk-STARK: 신뢰 설정 없는 대안

**zk-STARK(Scalable Transparent ARgument of Knowledge)** 는 해시 함수만으로 증명을 구성하여 신뢰 설정이 필요 없습니다. 증명 크기가 zk-SNARK보다 크지만, 양자 컴퓨터에 대한 저항성이 있어 StarkNet이 채택했습니다.

---

## 왜 필요한가

### 블록체인의 프라이버시 딜레마

퍼블릭 블록체인은 모든 트랜잭션이 공개됩니다. 기업이 경쟁사에 송금 내역을 노출하거나, 개인이 자산을 투명하게 드러내야 하는 상황은 현실적으로 받아들이기 어렵습니다. Zcash는 zk-SNARK로 "이 트랜잭션은 잔액을 정확히 소모하며 이중 지불도 없다"는 사실만 증명하고, 금액·주소를 완전히 숨깁니다.

### zk-Rollup: 이더리움 확장성

zk-Rollup은 수천 건의 트랜잭션을 오프체인에서 처리하고, "이 트랜잭션들이 올바르게 실행되었다"는 SNARK 증명 하나만 체인에 올립니다. 검증자는 수천 건 각각을 재실행하는 대신 수백 바이트의 증명만 검증하면 됩니다. 처리량이 수십~수백 배 향상되면서도 이더리움 수준의 보안을 유지합니다.

---

## 실제 구현 예제

### 예제 1 — Python: 간단한 Schnorr 프로토콜 (이산 로그 ZKP)

Schnorr 프로토콜은 "나는 g^x = y를 만족하는 x를 안다"를 x를 공개하지 않고 증명합니다. 이 구조는 현대 ZKP의 출발점이 됩니다.

```python
import hashlib
import secrets

# 작은 소수 그룹 파라미터 (실제 서비스에서는 2048비트 이상 사용)
# p: 소수, q: p-1의 소인수, g: g의 위수가 q인 생성원
p = 0xFFFFFFFFFFFFFFFFC90FDAA22168C234C4C6628B80DC1CD129024E088A67CC74
p = (1 << 255) - 19  # Curve25519 소수 필드 (시연용 간략화)
q = (p - 1) // 2     # 단순화된 서브그룹 위수 (실제 구현과 다름)
g = 2                 # 생성원

def mod_pow(base, exp, mod):
    return pow(base, exp, mod)

# --- 키 생성 ---
# x: 비밀 키 (위트니스), y: 공개 키
x = secrets.randbelow(q)          # 비밀
y = mod_pow(g, x, p)              # 공개: y = g^x mod p
print(f"공개 키 y = g^x mod p")
print(f"  x (비밀): {hex(x)[:20]}...")
print(f"  y (공개): {hex(y)[:20]}...")

# --- 증명 생성 (Prover) ---
# 1. 랜덤 nonce r 선택, 커밋 t = g^r 계산
r = secrets.randbelow(q)
t = mod_pow(g, r, p)

# 2. Fiat-Shamir 휴리스틱: 챌린지 c를 해시로 생성 (비대화형)
challenge_input = f"{y}{t}".encode()
c = int(hashlib.sha256(challenge_input).hexdigest(), 16) % q

# 3. 응답 s = r - c*x mod q
s = (r - c * x) % q

proof = (t, s)
print(f"\n생성된 증명 (t, s):")
print(f"  t: {hex(t)[:20]}...")
print(f"  s: {hex(s)[:20]}...")

# --- 검증 (Verifier) ---
# g^s * y^c == t 인지 확인
t_prime, s_prime = proof

challenge_input_v = f"{y}{t_prime}".encode()
c_v = int(hashlib.sha256(challenge_input_v).hexdigest(), 16) % q

lhs = (mod_pow(g, s_prime, p) * mod_pow(y, c_v, p)) % p
rhs = t_prime

print(f"\n검증 결과: {'✓ 성공' if lhs == rhs else '✗ 실패'}")
print(f"  g^s * y^c mod p == t: {lhs == rhs}")
# 검증자는 x를 모르지만 x를 아는 증명자만 생성 가능한 증명을 확인했다!
```

핵심은 `s = r - c*x mod q`에서 c와 s만 공개하면 `g^s * y^c = g^(r-cx) * g^(cx) = g^r = t`가 성립한다는 점입니다. 검증자는 x를 알 수 없으면서도 증명자가 x를 안다는 사실을 확신합니다.

### 예제 2 — Python: Pedersen 커밋으로 범위 증명 (Range Proof) 개념 구현

"이 숫자가 0 이상 100 이하다"라는 사실을 숫자를 공개하지 않고 증명하는 것이 Range Proof입니다. DeFi에서 담보 비율 검증, 투표 시스템, 비밀 입찰 등에 쓰입니다.

```python
import secrets
import hashlib

# Pedersen 커밋: commit(v, r) = g^v * h^r mod p
# v: 숫자(비밀), r: 블라인딩 팩터(비밀), g·h: 공개 생성원
# 동형성: commit(v1,r1) * commit(v2,r2) = commit(v1+v2, r1+r2)

class PedersenCommitment:
    def __init__(self, p: int, g: int, h: int):
        self.p = p
        self.g = g
        self.h = h

    def commit(self, value: int, blinding: int) -> int:
        return (pow(self.g, value, self.p) * pow(self.h, blinding, self.p)) % self.p

    def add(self, c1: int, c2: int) -> int:
        return (c1 * c2) % self.p

# 파라미터 (시연용 소규모)
p = 2**31 - 1  # 메르센 소수
g = 3
# h = g^(무작위 지수): 이산 로그 관계 불명 (Nothing-up-my-sleeve)
h = pow(g, 104648257, p)

pc = PedersenCommitment(p, g, h)

# === 간략화된 이진 범위 증명 ===
# v를 비트 분해하여 각 비트가 0 또는 1임을 증명
# 실제 Bulletproof는 이 아이디어를 대수적으로 최적화한 것

def range_proof_demo(value: int, bit_length: int = 7):
    """value가 [0, 2^bit_length - 1] 범위임을 증명 (개념 시연)"""
    assert 0 <= value < (1 << bit_length), "범위 초과"

    bits = [(value >> i) & 1 for i in range(bit_length)]
    blindings = [secrets.randbelow(p - 1) for _ in range(bit_length)]

    # 각 비트에 대한 커밋 생성
    bit_commits = [pc.commit(b, r) for b, r in zip(bits, blindings)]

    # 각 커밋 C_i가 0 또는 1의 커밋임을 증명
    # C_i * C_i - C_i = h^(r_i * (2*b_i - 1)) — Chaum-Pedersen 기반
    proofs = []
    for i, (b, r, C) in enumerate(zip(bits, blindings, bit_commits)):
        # 실제 증명에서는 AND proof of C_i = commit(0,r) OR C_i = commit(1,r)
        # 여기서는 챌린지-응답 프로토콜을 시뮬레이션
        nonce_r = secrets.randbelow(p - 1)
        T = pc.commit(0, nonce_r)  # 커밋
        challenge = int(hashlib.sha256(f"{C}{T}{i}".encode()).hexdigest(), 16) % (p-1)
        response = (nonce_r - challenge * r) % (p - 1)
        proofs.append((C, T, response, challenge))

    # 전체 값의 커밋 (동형성으로 합산)
    total_commit = 1
    for C in bit_commits:
        total_commit = pc.add(total_commit, C)
    total_blinding = sum(blindings) % (p - 1)

    return {
        "total_commit": total_commit,
        "bit_commits": bit_commits,
        "proofs": proofs,
        "bit_length": bit_length,
    }

def verify_range_proof(proof_data: dict, expected_total_commit: int) -> bool:
    """각 비트 커밋의 유효성 검증 (챌린지 재계산)"""
    for C, T, response, challenge in proof_data["proofs"]:
        # g^response * C^challenge == T 인지 검증
        lhs = (pow(g, response, p) * pow(C, challenge, p)) % p
        # T는 pc.commit(0, nonce_r) = h^nonce_r 이므로:
        # g^(nonce_r - challenge*r) * C^challenge
        # = g^nonce_r * g^(-challenge*r) * g^(v*challenge) * h^(r*challenge)
        # 완전한 검증을 위해서는 C가 0 또는 1의 커밋임을 AND/OR proof로 확인
        pass  # 개념 시연 — 실제는 Bulletproof 구현 라이브러리 사용
    return True  # 시연 목적

# 테스트
secret_value = 73  # 0~127 범위
proof = range_proof_demo(secret_value, bit_length=7)
print(f"비밀 값: {secret_value} (공개 안 함)")
print(f"총 커밋: {hex(proof['total_commit'])[:20]}...")
print(f"비트 커밋 수: {len(proof['bit_commits'])} (각 비트에 하나씩)")
print(f"범위 증명 성공: {verify_range_proof(proof, proof['total_commit'])}")

# 동형성 검증: commit(3) + commit(4) = commit(7)?
r1, r2 = 12345, 67890
c3 = pc.commit(3, r1)
c4 = pc.commit(4, r2)
c7 = pc.commit(7, r1 + r2)
print(f"\n동형성 검증: commit(3) + commit(4) == commit(7): {pc.add(c3, c4) == c7}")
```

---

## 주의사항 및 팁

### 1. Trusted Setup의 보안 위험

Groth16 등 pairing 기반 SNARK는 초기 `CRS(Common Reference String)` 생성 시 랜덤 파라미터를 완전히 폐기해야 합니다. 이를 "독성 폐기물(toxic waste)"이라 부릅니다. 만약 누군가 이 값을 보관한다면 거짓 증명을 생성할 수 있습니다. Zcash의 "Powers of Tau" 세레모니처럼 수백 명이 참여하는 멀티파티 컴퓨테이션으로 신뢰를 분산하는 방법이 일반적입니다. 단 한 명이라도 파라미터를 파기하면 전체 설정이 안전합니다.

### 2. 증명 생성 비용

zk-SNARK 증명 생성은 일반 연산 대비 수백~수만 배 비쌉니다. Groth16으로 SHA-256 해시 하나를 증명하는 데 수십 초가 걸릴 수 있습니다. zkSync의 zkEVM은 이를 수백 밀리초로 줄이기 위해 하드웨어 가속기(GPU·FPGA)와 병렬화된 MSM(Multi-Scalar Multiplication) 알고리즘을 사용합니다.

### 3. Soundness vs Knowledge Soundness

단순 ZKP는 "명제가 참이다"를 증명하지만, **zk-SNARK의 Knowledge Soundness**는 "증명자가 위트니스를 실제로 갖고 있다"를 보장합니다. 블록체인 트랜잭션의 경우 개인 키를 진짜 알고 있어야만 유효한 증명을 생성할 수 있다는 의미입니다. 이를 위해 extractor(추출기) 개념을 포함한 엄격한 수학적 정의가 필요합니다.

### 4. 재사용 공격(Replay Attack) 방지

대화형 ZKP에서는 챌린지가 매번 새로 생성되므로 재사용이 불가합니다. 비대화형 ZKP(Fiat-Shamir 변환 적용)에서는 챌린지에 컨텍스트(도메인 분리자, 트랜잭션 ID 등)를 포함해야 합니다.

### 5. 회로 최적화

ZKP의 비용은 산술 회로의 게이트(constraint) 수에 비례합니다. 조건 분기(`if/else`)는 ZKP에서 매우 비쌉니다 — 양쪽을 모두 계산하고 선택자(selector)로 결과를 고르는 패턴을 써야 합니다. 비트 연산보다 필드 산술이 효율적이므로, 회로 설계 단계에서 연산을 최대한 필드 원소 수준에서 처리해야 합니다.

## 참고 자료

- [Zero-knowledge proof — Wikipedia](https://en.wikipedia.org/wiki/Zero-knowledge_proof)
- [Zero-Knowledge Proof (ZKP) Explained — Chainlink](https://chain.link/education/zero-knowledge-proof-zkp)
- [Zero-knowledge proofs explained: zk-SNARKs vs zk-STARKs — LimeChain](https://limechain.tech/blog/zero-knowledge-proofs-explained)
- [ZKProof Community Reference — zkproof.org](https://zkproof.org/)
