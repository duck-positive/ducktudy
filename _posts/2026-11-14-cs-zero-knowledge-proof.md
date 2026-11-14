---
layout: post
title: "제로 지식 증명(Zero-Knowledge Proof) 심화: 증명하되 드러내지 않는 암호학의 마법"
date: 2026-11-14
categories: [cs, computer-science]
tags: [zero-knowledge-proof, zkp, cryptography, interactive-proof, zk-snark, privacy]
---

## 개념 설명

**제로 지식 증명(Zero-Knowledge Proof, ZKP)**은 증명자(Prover)가 어떤 명제가 참임을 검증자(Verifier)에게 증명하면서, **그 명제가 참이라는 사실 외에는 어떠한 정보도 유출하지 않는** 암호학적 프로토콜입니다.

1985년 Shafi Goldwasser, Silvio Micali, Charles Rackoff가 논문 "The Knowledge Complexity of Interactive Proof Systems"에서 처음 제안했으며, 2012년 Goldwasser와 Micali는 이 연구로 튜링상을 수상했습니다.

### 직관적 이해 — 동굴 비유

가장 유명한 비유는 **알리바바 동굴(Ali Baba Cave)** 예시입니다.

```
           입구
            │
       ┌────┴────┐
       │         │
      A경로      B경로
       │         │
       └────┬────┘
      비밀 문 (주문 필요)
```

Peggy는 비밀 주문을 알고 있고, Victor는 Peggy가 알고 있음을 확인하고 싶습니다. 하지만 Peggy는 주문을 공개하고 싶지 않습니다.

프로토콜:
1. Peggy가 먼저 동굴에 들어가 A 또는 B 경로를 선택 (Victor는 모름)
2. Victor가 동굴 입구에서 "A경로로 나와!" 또는 "B경로로 나와!"를 외침
3. Peggy가 비밀 문을 통해 Victor가 원하는 경로로 나옴

Peggy가 주문을 모른다면 각 라운드에서 성공 확률이 50%입니다. n번 반복 시 속일 확률은 (1/2)^n으로 감소합니다. 20번 반복하면 약 100만 분의 1의 확률로만 속일 수 있습니다.

---

## 왜 필요한가?

ZKP는 프라이버시와 보안이 충돌하는 현실 문제를 해결합니다.

**전통적 인증의 문제**: 비밀번호를 증명하려면 비밀번호 자체를 전송해야 합니다. 서버가 해킹당하면 비밀번호가 유출됩니다.

**ZKP를 활용하면**:
- **나이 증명**: "나는 18세 이상이다"를 주민번호 없이 증명
- **잔액 증명**: "내 계좌에 100만원 이상 있다"를 계좌 내역 공개 없이 증명
- **블록체인 프라이버시**: Zcash는 ZK-SNARK를 사용해 거래를 완전히 비공개로 처리
- **롤업(ZK-Rollup)**: Ethereum L2에서 수천 개의 트랜잭션을 하나의 ZK 증명으로 압축

---

## ZKP의 세 가지 성질

ZKP 프로토콜은 반드시 다음 세 가지 성질을 만족해야 합니다.

1. **완전성(Completeness)**: 명제가 참이고 증명자가 비밀을 알고 있다면, 정직한 검증자를 항상 납득시킬 수 있다.
2. **건전성(Soundness)**: 명제가 거짓이면, 부정직한 증명자가 검증자를 납득시킬 수 있는 확률이 무시할 수 있을 만큼 작다.
3. **영지식성(Zero-Knowledge)**: 검증자는 명제가 참임 외에는 아무것도 배우지 못한다. 시뮬레이터가 비밀 없이도 프로토콜 대화를 완벽하게 재현할 수 있어야 한다.

---

## 실제 구현 예제 1 — 그래프 3-색칠 ZKP

**그래프 3-색칠 문제**: 그래프의 모든 간선에서 양 끝 정점의 색이 다르도록 3가지 색으로 색칠하는 것이 NP-완전 문제입니다. 이 문제의 해법을 아는 것을 해법 공개 없이 증명할 수 있습니다.

```python
import random
import hashlib
import secrets

def commit(value: str, secret: str) -> str:
    """암호학적 커밋 — SHA-256 해시로 값을 숨김"""
    return hashlib.sha256(f"{secret}:{value}".encode()).hexdigest()

def verify_commit(commitment: str, value: str, secret: str) -> bool:
    """커밋 검증"""
    return commitment == hashlib.sha256(f"{secret}:{value}".encode()).hexdigest()


class GraphColoringZKP:
    """
    그래프 3-색칠 제로 지식 증명 프로토콜
    증명자는 색칠 방법을 알고 있지만 공개하지 않음
    """

    # 테스트 그래프 (삼각형 — 정점 0,1,2는 모두 연결됨)
    GRAPH_EDGES = [(0, 1), (1, 2), (0, 2)]
    COLORING = {0: 'R', 1: 'G', 2: 'B'}  # 증명자만 아는 비밀 색칠

    def prover_commit(self) -> dict:
        """
        [증명자 단계 1] 색칠을 무작위로 순열 치환하고 각 정점의 색을 커밋
        커밋은 색을 숨기면서도 나중에 공개를 증명할 수 있음
        """
        colors = ['R', 'G', 'B']
        random.shuffle(colors)
        perm = dict(zip(['R', 'G', 'B'], colors))

        self.current_perm_coloring = {v: perm[c] for v, c in self.COLORING.items()}
        self.current_secrets = {v: secrets.token_hex(16) for v in self.COLORING}

        commitments = {
            v: commit(self.current_perm_coloring[v], self.current_secrets[v])
            for v in self.COLORING
        }
        return commitments

    def verifier_challenge(self) -> tuple[int, int]:
        """[검증자 단계 2] 무작위로 간선 하나를 선택해 공개 요청"""
        return random.choice(self.GRAPH_EDGES)

    def prover_reveal(self, edge: tuple[int, int]) -> dict:
        """[증명자 단계 3] 선택된 간선의 두 정점 색만 공개"""
        u, v = edge
        return {
            u: (self.current_perm_coloring[u], self.current_secrets[u]),
            v: (self.current_perm_coloring[v], self.current_secrets[v]),
        }

    def verifier_verify(self, commitments: dict, edge: tuple[int, int],
                        revealed: dict) -> bool:
        """[검증자 단계 4] 공개된 색이 커밋과 일치하고 두 색이 다른지 확인"""
        u, v = edge
        color_u, secret_u = revealed[u]
        color_v, secret_v = revealed[v]

        # 커밋 일치 확인
        if not verify_commit(commitments[u], color_u, secret_u):
            return False
        if not verify_commit(commitments[v], color_v, secret_v):
            return False

        # 두 정점의 색이 다른지 확인
        return color_u != color_v


def run_zkp_protocol(rounds: int = 20):
    """n 라운드 ZKP 실행 — 각 라운드에서 성공하면 신뢰도 증가"""
    prover = GraphColoringZKP()

    print(f"그래프 3-색칠 ZKP 프로토콜 ({rounds} 라운드)")
    print(f"증명자의 비밀 색칠: {GraphColoringZKP.COLORING}")
    print(f"그래프 간선: {GraphColoringZKP.GRAPH_EDGES}\n")

    success_count = 0

    for round_num in range(1, rounds + 1):
        # 1. 증명자가 커밋
        commitments = prover.prover_commit()

        # 2. 검증자가 도전
        challenged_edge = prover.verifier_challenge()

        # 3. 증명자가 공개
        revealed = prover.prover_reveal(challenged_edge)

        # 4. 검증자가 검증
        is_valid = prover.verifier_verify(commitments, challenged_edge, revealed)

        if is_valid:
            success_count += 1

        if round_num <= 5 or round_num == rounds:
            print(f"라운드 {round_num:2d}: 간선 {challenged_edge} 검증 → {'통과' if is_valid else '실패'}")

    cheat_prob = (1 / len(GraphColoringZKP.GRAPH_EDGES)) ** rounds
    print(f"\n결과: {success_count}/{rounds} 라운드 통과")
    print(f"부정직한 증명자가 모든 라운드를 통과할 확률: {cheat_prob:.2e}")
    print("→ 색칠 방법을 단 한 번도 공개하지 않고 증명 완료!")


run_zkp_protocol()
```

---

## 실제 구현 예제 2 — Schnorr 프로토콜 (이산 로그 ZKP)

**Schnorr 프로토콜**은 "나는 g^x ≡ y (mod p)를 만족하는 x를 알고 있다"를 x를 공개하지 않고 증명합니다. ECDSA 서명, 많은 ZK 시스템의 기반입니다.

```python
import secrets as sec

class SchnorrZKP:
    """
    Schnorr 프로토콜 — 이산 로그 제로 지식 증명
    공개 정보: (p, g, y)  비밀: x (y = g^x mod p)
    """

    def __init__(self):
        # 교육용 소수 (실제로는 2048비트 이상 사용)
        self.p = 2357  # 소수
        self.g = 2     # 생성자
        self.q = (self.p - 1)  # 군의 위수

    def setup(self):
        """비밀 x와 공개키 y = g^x mod p 생성"""
        self.x = sec.randbelow(self.q - 2) + 2  # 비밀 (2 이상)
        self.y = pow(self.g, self.x, self.p)    # 공개키
        print(f"공개 정보: p={self.p}, g={self.g}, y={self.y}")
        print(f"비밀 (x): {self.x}  ← 증명자만 알고 있음\n")

    # === 증명자 단계 ===

    def prover_commit(self) -> int:
        """[증명자 1] 무작위 r 선택 후 R = g^r mod p 커밋"""
        self.r = sec.randbelow(self.q - 2) + 2
        self.R = pow(self.g, self.r, self.p)
        return self.R  # 커밋 R만 공개

    def prover_respond(self, challenge: int) -> int:
        """[증명자 3] 도전 c에 대한 응답 s = r + c*x mod q"""
        s = (self.r + challenge * self.x) % self.q
        return s

    # === 검증자 단계 ===

    def verifier_challenge(self) -> int:
        """[검증자 2] 무작위 도전값 c 생성"""
        return sec.randbelow(1000) + 1

    def verifier_verify(self, R: int, c: int, s: int) -> bool:
        """
        [검증자 4] g^s ≡ R * y^c (mod p) 검증
        비밀 x를 모르는 채로 g^(r + cx) = g^r * (g^x)^c = R * y^c 임을 확인
        """
        lhs = pow(self.g, s, self.p)         # g^s mod p
        rhs = (R * pow(self.y, c, self.p)) % self.p  # R * y^c mod p
        return lhs == rhs


def run_schnorr():
    protocol = SchnorrZKP()
    protocol.setup()

    total_rounds = 10
    print(f"Schnorr 프로토콜 ({total_rounds} 라운드)")

    for i in range(1, total_rounds + 1):
        R = protocol.prover_commit()       # 1. 증명자 커밋
        c = protocol.verifier_challenge()  # 2. 검증자 도전
        s = protocol.prover_respond(c)     # 3. 증명자 응답
        ok = protocol.verifier_verify(R, c, s)  # 4. 검증
        print(f"라운드 {i:2d}: R={R:4d}, c={c:4d}, s={s:4d} → {'통과' if ok else '실패'}")

    print(f"\n비밀 x={protocol.x}를 한 번도 공개하지 않고 이산 로그 지식 증명 완료!")
    print("검증자가 얻은 정보: (p, g, y, R, c, s) — x로 역산 불가")


run_schnorr()
```

---

## ZK-SNARK와 ZK-STARK

현대 ZKP는 **비대화형(Non-Interactive)** 방식으로 발전했습니다.

### ZK-SNARK (Succinct Non-interactive ARgument of Knowledge)

- **간결성(Succinct)**: 증명 크기가 O(1) 또는 O(log n)으로 매우 작음
- **비대화형**: 한 번의 메시지로 증명
- **신뢰 셋업 필요**: 초기 파라미터 생성 단계에서 "독성 폐기물(toxic waste)" 문제 존재
- 활용: Zcash 프라이버시 코인, Ethereum ZK-Rollup (zkSync, StarkNet)

### ZK-STARK (Scalable Transparent ARgument of Knowledge)

- **투명성(Transparent)**: 신뢰 셋업 불필요 (랜덤 오라클만 사용)
- **양자 내성**: 해시 함수 기반 — 양자 컴퓨터에 안전
- **증명 크기**: SNARK보다 크지만 검증 속도는 빠름
- 활용: StarkWare, Cairo 언어

---

## 주의사항과 팁

### 현실적 고려사항

**1. 교육용 구현 ≠ 프로덕션 사용**

위 예제는 교육 목적으로 작은 소수를 사용합니다. 실제로는 2048비트 이상의 소수 또는 타원 곡선(secp256k1, BLS12-381)을 사용해야 합니다. 직접 구현하지 말고 검증된 라이브러리를 사용하세요.

**2. 비대화형 변환 — Fiat-Shamir 휴리스틱**

Schnorr 같은 대화형 ZKP를 비대화형으로 변환하려면 **Fiat-Shamir 변환**을 사용합니다. 검증자의 무작위 도전값을 `c = H(g, y, R, context)` 해시로 대체합니다. 이때 맥락 정보(context)를 충분히 포함하지 않으면 재플레이 공격에 취약해집니다.

**3. ZK-Rollup의 현실**

Ethereum의 ZK-Rollup은 수천 개의 트랜잭션을 하나의 ZK 증명으로 압축합니다. 증명 생성이 GPU 집약적(수십 초~수 분)이지만, 검증은 O(1)로 매우 빠릅니다. 이 비대칭성이 Rollup의 핵심입니다.

**4. 양자 컴퓨터와 ZKP**

ZK-SNARK는 타원 곡선 암호에 의존하므로 Shor 알고리즘에 취약합니다. ZK-STARK는 해시 함수 기반이므로 양자 내성이 있습니다. 장기적 보안이 필요하다면 STARK 기반 시스템을 고려하세요.

**5. 프라이버시의 한계**

ZKP는 계산의 정확성을 증명하지만, **입력 데이터의 출처**는 보장하지 않습니다. 예를 들어 "나이가 18세 이상"임을 증명해도, 그 나이 데이터가 신뢰할 수 있는 출처에서 왔는지는 별도의 메커니즘이 필요합니다.

---

## 참고 자료

- [Wikipedia: Zero-knowledge proof](https://en.wikipedia.org/wiki/Zero-knowledge_proof)
- [Wikipedia: Schnorr signature](https://en.wikipedia.org/wiki/Schnorr_signature)
- [Stanford Code the Change: Zero Knowledge Guide](https://codethechange.stanford.edu/guides/guide_zk.html)
- [MIT CSAIL: Zero-knowledge basics](https://people.csail.mit.edu/matth/blog/2019/05/zero-knowledge-basics)
