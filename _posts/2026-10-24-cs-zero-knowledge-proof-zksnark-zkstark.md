---
layout: post
title: "영지식 증명(ZKP): zk-SNARK와 zk-STARK의 원리와 구현"
date: 2026-10-24
categories: [cs, computer-science]
tags: [zero-knowledge-proof, zkp, zk-snark, zk-stark, cryptography, privacy, blockchain]
---

## 개요

영지식 증명(Zero-Knowledge Proof, ZKP)은 **한 쪽(증명자, Prover)이 어떤 사실을 상대방(검증자, Verifier)에게, 그 사실 외에 아무런 추가 정보도 노출하지 않으면서 증명**할 수 있는 암호학적 프로토콜입니다.

1989년 Shafi Goldwasser, Silvio Micali, Charles Rackoff가 논문 "The Knowledge Complexity of Interactive Proof Systems"에서 처음 형식화했으며, 현대 프라이버시 보호 기술, 블록체인 확장성, 신원 인증 시스템의 핵심 기반이 되었습니다.

---

## 왜 영지식 증명이 필요한가?

### 현실 속 문제들

1. **비밀번호 인증**: 서버에 비밀번호를 전송하면 서버가 유출될 경우 비밀번호도 노출됩니다.
2. **금융 프라이버시**: "잔액이 1,000만 원 이상"임을 증명하면서 정확한 잔액은 감추고 싶습니다.
3. **신원 인증**: 나이가 20세 이상임을 증명하면서 정확한 생년월일이나 이름은 공개하고 싶지 않습니다.
4. **블록체인 확장성**: 수천 건의 트랜잭션이 유효함을 하나의 증명으로 압축해 검증 비용을 줄이고 싶습니다.

ZKP는 이런 모든 상황에서 **검증 가능한 프라이버시**를 제공합니다.

---

## 영지식 증명의 세 가지 성질

유효한 ZKP는 반드시 다음 세 조건을 만족해야 합니다:

### 1. 완전성(Completeness)
증명자가 진실을 알고 있다면, 정직한 프로토콜 실행 후 검증자는 항상 설득됩니다.
> "올바른 증명자는 항상 검증을 통과한다."

### 2. 건전성(Soundness)
증명자가 거짓말을 하고 있다면, 정직한 검증자를 속일 확률이 극히 낮습니다.
> "거짓 증명자는 검증을 통과할 수 없다."

### 3. 영지식성(Zero-Knowledge)
검증자는 "증명이 유효하다"는 사실 외에 아무것도 배우지 못합니다.
> "검증자는 비밀을 알 수 없다."

---

## 직관적 예시: 색맹과 공

증명자는 두 공(빨강, 파랑)이 서로 다른 색임을 아는데, 검증자는 색맹입니다. 검증자는 두 공을 등 뒤에서 손에 하나씩 들고, 임의로 바꾸거나 바꾸지 않습니다. 그 후 "바꿨나?"를 묻습니다. 증명자는 색을 볼 수 있으므로 항상 정확히 답할 수 있습니다.

- 공이 실제로 같은 색이라면, 증명자는 1/2 확률로만 맞출 수 있습니다.
- k번 반복하면 속일 확률이 (1/2)^k로 감소합니다.
- 이 과정에서 검증자는 색에 대한 어떤 정보도 얻지 못합니다.

이것이 **인터랙티브 ZKP**의 본질입니다.

---

## zk-SNARK

### 개념

**zk-SNARK** = Zero-Knowledge Succinct Non-Interactive Argument of Knowledge

- **Succinct**: 증명 크기가 작고(수백 바이트) 검증이 빠릅니다.
- **Non-Interactive**: 증명자와 검증자 사이에 왕복 통신 없이 증명을 생성합니다.
- **Argument of Knowledge**: 증명자가 실제로 증인(witness)을 알고 있음을 보장합니다.

### 신뢰할 수 있는 셋업(Trusted Setup)

zk-SNARK는 초기 설정 시 **공개 파라미터(Structured Reference String, SRS)**를 생성합니다. 이 과정에서 생성된 비밀 값("독성 쓰레기")이 유출되면 가짜 증명 생성이 가능합니다. Zcash의 "Powers of Tau" 세레모니처럼 여러 참가자가 함께 설정합니다.

### R1CS와 QAP

zk-SNARK에서 계산은 다음과 같이 변환됩니다:

```
계산 문제 → 회로(Circuit) → R1CS → QAP → 타원 곡선 위 다항식 증명
```

**R1CS(Rank-1 Constraint System)**: 계산을 `A·w * B·w = C·w` 형태의 제약식으로 표현합니다.

---

## 구현 예제 1: Python으로 간단한 ZKP 시뮬레이션

아래는 "나는 비밀 x를 알고 있으며, x² + x = 6"임을 x를 노출하지 않고 증명하는 인터랙티브 프로토콜 시뮬레이션입니다.

```python
import random
import hashlib

# 공개 파라미터: 소수 p, 생성원 g
p = 2357  # 실제로는 훨씬 큰 소수 사용
g = 2

# 비밀: x = 2 (x^2 + x = 6을 만족)
secret_x = 2

# y = g^x mod p (공개 키)
y = pow(g, secret_x, p)
print(f"공개 키 y = g^{secret_x} mod {p} = {y}")

def prove_knowledge(secret: int, public_key: int, rounds: int = 10) -> bool:
    """
    Schnorr 프로토콜: 로그 이산 문제의 영지식 증명
    증명: 'g^secret = public_key'를 secret 없이 증명
    """
    for i in range(rounds):
        # 1. Commitment: 증명자가 임의 r 선택, R = g^r 공개
        r = random.randint(1, p - 2)
        R = pow(g, r, p)

        # 2. Challenge: 검증자가 도전값 c 생성
        c = int(hashlib.sha256(f"{R}{public_key}".encode()).hexdigest(), 16) % (p - 1)

        # 3. Response: 증명자가 s = r - c*secret (mod p-1) 계산
        s = (r - c * secret) % (p - 1)

        # 4. Verification: g^s * y^c == R?
        lhs = (pow(g, s, p) * pow(public_key, c, p)) % p
        if lhs != R:
            print(f"  Round {i+1}: FAIL")
            return False
        print(f"  Round {i+1}: PASS (R={R}, c={c}, s={s})")

    return True

print("\n[정직한 증명자]")
result = prove_knowledge(secret_x, y)
print(f"검증 결과: {'성공' if result else '실패'}")

print("\n[악의적 증명자 (wrong secret)]")
result = prove_knowledge(secret_x + 1, y, rounds=3)
print(f"검증 결과: {'성공' if result else '실패'}")
```

---

## zk-STARK

### 개념

**zk-STARK** = Zero-Knowledge Scalable Transparent Argument of Knowledge

zk-SNARK의 핵심 한계인 "신뢰할 수 있는 셋업"을 제거한 버전입니다. 2018년 Eli Ben-Sasson 등이 제안했습니다.

| 특성 | zk-SNARK | zk-STARK |
|------|----------|----------|
| 증명 크기 | ~200 바이트 (매우 작음) | 수십 KB (큰 편) |
| 검증 시간 | 매우 빠름 | 빠름 |
| 신뢰 설정 | 필요 | 불필요 (투명) |
| 양자 안전 | 아님 (ECC 기반) | 안전 (해시 기반) |
| 확장성 | O(n log n) | O(n polylog n) |

### FRI (Fast Reed-Solomon IOP of Proximity)

zk-STARK의 핵심 빌딩 블록입니다. 다항식 f가 특정 차수 이하임을 효율적으로 증명합니다. Reed-Solomon 에러 정정 코드와 Merkle 트리를 결합해 투명한(transparent) 증명을 만듭니다.

---

## 구현 예제 2: 해시 기반 커밋-공개 ZKP (STARK 개념)

```python
import hashlib
import secrets
import json

def hash_value(data: bytes) -> str:
    return hashlib.sha256(data).hexdigest()

def merkle_root(leaves: list) -> tuple:
    """Merkle 트리 루트 계산"""
    if len(leaves) == 1:
        return leaves[0], []
    
    layer = leaves
    proof_layers = []
    while len(layer) > 1:
        if len(layer) % 2 == 1:
            layer.append(layer[-1])
        new_layer = []
        for i in range(0, len(layer), 2):
            combined = hash_value((layer[i] + layer[i+1]).encode())
            new_layer.append(combined)
        proof_layers.append(layer)
        layer = new_layer
    return layer[0], proof_layers

class TransparentZKP:
    """
    STARK 개념 시뮬레이션:
    비밀 배열에 대한 통계적 성질 (예: 합이 특정 값)을 증명
    """
    
    def __init__(self, secret_values: list):
        self.secret = secret_values
        self.nonce = secrets.token_hex(16)
        
        # 각 값을 nonce와 함께 해시 → 커밋
        self.commitments = [
            hash_value(f"{self.nonce}:{v}".encode())
            for v in secret_values
        ]
        self.root, _ = merkle_root(self.commitments)
    
    def get_public_info(self) -> dict:
        return {
            "merkle_root": self.root,
            "claimed_sum": sum(self.secret),
            "count": len(self.secret)
        }
    
    def prove_sum(self) -> dict:
        """모든 값의 합이 claimed_sum임을 증명"""
        return {
            "revealed_values": self.secret,
            "nonce": self.nonce,
            "commitments": self.commitments
        }
    
    @staticmethod
    def verify_proof(public_info: dict, proof: dict) -> bool:
        values = proof["revealed_values"]
        nonce = proof["nonce"]
        commitments = proof["commitments"]
        
        # 1. 개수 확인
        if len(values) != public_info["count"]:
            return False
        
        # 2. 합 확인
        if sum(values) != public_info["claimed_sum"]:
            print("  합이 일치하지 않습니다.")
            return False
        
        # 3. 커밋 재계산 및 검증
        for i, (v, c) in enumerate(zip(values, commitments)):
            expected = hash_value(f"{nonce}:{v}".encode())
            if expected != c:
                print(f"  커밋 {i} 불일치")
                return False
        
        # 4. Merkle 루트 검증
        computed_root, _ = merkle_root(commitments)
        if computed_root != public_info["merkle_root"]:
            print("  Merkle 루트 불일치")
            return False
        
        print("  모든 검증 통과")
        return True


# 테스트
secret_data = [10, 25, 15, 30, 20]
prover = TransparentZKP(secret_data)

public = prover.get_public_info()
print(f"공개 정보: Merkle 루트 = {public['merkle_root'][:16]}..., 합 = {public['claimed_sum']}")

proof = prover.prove_sum()
print("\n검증 중...")
valid = TransparentZKP.verify_proof(public, proof)
print(f"검증 결과: {'유효' if valid else '무효'}")

# 조작 시도
print("\n[조작된 증명 시도]")
tampered_proof = dict(proof)
tampered_proof["revealed_values"] = [10, 25, 15, 30, 99]
valid = TransparentZKP.verify_proof(public, tampered_proof)
print(f"검증 결과: {'유효' if valid else '무효'}")
```

---

## 실제 응용 사례

### 1. 블록체인 Layer 2 (ZK-Rollup)
Ethereum의 zkSync, StarkNet은 수천 건의 트랜잭션을 하나의 ZK 증명으로 압축합니다. L1에서는 증명만 검증하므로 가스 비용이 크게 줄어듭니다.

### 2. 프라이버시 코인
Zcash는 zk-SNARK(Groth16)를 사용해 트랜잭션 금액과 송수신자를 감추면서 이중 지불 없음을 증명합니다.

### 3. 신원 인증
EU의 eIDAS 2.0에서는 "18세 이상"만 증명하고 이름/생년월일은 공개하지 않는 선택적 공개 기술에 ZKP를 도입합니다.

---

## 주의사항 및 팁

1. **신뢰 설정 vs. 투명 설정**: 프로덕션에서는 셋업의 신뢰 가정을 명확히 이해하고 선택하세요.
2. **증명 생성 비용**: 프루버(Prover) 측 계산이 무거울 수 있습니다. 하드웨어 가속(GPU, ASIC)이 필요한 경우가 있습니다.
3. **보안 파라미터**: 소수, 해시 함수 선택이 보안 레벨을 결정합니다. 표준 라이브러리(libsnark, gnark, arkworks)를 사용하세요.
4. **회로 최적화**: R1CS/STARK 회로 설계가 성능의 병목입니다. 게이트 수를 줄이는 최적화가 중요합니다.

---

## 정리

영지식 증명은 "비밀을 노출하지 않고 사실을 증명한다"는 역설적인 요구를 수학적으로 해결합니다. zk-SNARK는 작은 증명 크기와 빠른 검증이 장점이며, zk-STARK는 신뢰 설정 없이 투명하고 양자 안전합니다. 두 기술 모두 블록체인, 프라이버시, 규제 준수 분야에서 빠르게 채택되고 있습니다.

## 참고 자료
- [Wikipedia - Zero-Knowledge Proof](https://en.wikipedia.org/wiki/Zero-knowledge_proof)
- [Wikipedia - Non-interactive Zero-Knowledge Proof](https://en.wikipedia.org/wiki/Non-interactive_zero-knowledge_proof)
- [Wikipedia - zk-SNARK](https://en.wikipedia.org/wiki/Non-interactive_zero-knowledge_proof#zk-SNARK)
- [StarkWare - STARK 공식 문서](https://starkware.co/stark/)
