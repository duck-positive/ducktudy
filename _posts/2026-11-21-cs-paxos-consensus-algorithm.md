---
layout: post
title: "Paxos 합의 알고리즘 완전 정복: 분산 시스템의 아버지가 설계한 합의의 본질"
date: 2026-11-21
categories: [cs, computer-science]
tags: [paxos, distributed-systems, consensus, fault-tolerance, 분산시스템, 합의알고리즘]
---

## Paxos란 무엇인가

Paxos는 Leslie Lamport가 1989년 논문 "The Part-Time Parliament"에서 처음 발표하고, 2001년 "Paxos Made Simple"에서 재정리한 **분산 합의(Distributed Consensus) 알고리즘**입니다. 분산 환경에서 여러 노드가 하나의 값에 동의(합의)하는 문제를 해결하며, 일부 노드가 장애를 일으키거나 메시지가 지연·유실되는 환경에서도 안전하게 작동하도록 설계되었습니다.

분산 합의 문제는 얼핏 쉬워 보이지만, 1985년 발표된 **FLP 불가능성 정리(FLP Impossibility)**가 "비동기 분산 시스템에서 하나의 노드 장애만 있어도 합의를 완전히 보장하는 결정론적 알고리즘은 존재하지 않는다"는 것을 증명했습니다. Paxos는 이 한계를 인정하면서, **Safety(잘못된 값에 합의하지 않음)**를 항상 보장하고 충분한 노드가 살아있을 때 **Liveness(결국 합의에 도달함)**를 보장하는 실용적인 해법을 제시했습니다.

Raft, ZooKeeper의 ZAB, 구글의 Chubby 등 현대 분산 시스템에서 사용되는 거의 모든 합의 프로토콜은 Paxos에서 파생되었거나 Paxos의 한계를 보완하는 방향으로 설계되었습니다.

---

## 왜 분산 합의가 어려운가

분산 시스템을 어렵게 만드는 세 가지 핵심 문제가 있습니다:

1. **네트워크 분할(Network Partition)**: 노드들이 서로 통신하지 못하는 상황
2. **부분 실패(Partial Failure)**: 일부 노드만 죽고 나머지는 살아있는 상황
3. **메시지 순서 불일치**: 메시지가 보낸 순서와 다르게 도착하거나 아예 도착하지 않는 상황

예를 들어, 세 대의 서버가 "다음 트랜잭션을 실행할 것인가"에 대해 합의해야 한다고 가정해 봅시다. 서버 A가 "YES"를 보냈지만 메시지가 유실되면, 서버 B와 C는 A가 죽었는지 아직 응답 중인지 알 수 없습니다. 무한정 기다리면 Liveness를 잃고, 너무 빨리 판단하면 Safety를 잃을 수 있습니다.

---

## Paxos의 핵심 구조: 역할과 단계

### 세 가지 역할

Paxos에서 모든 노드는 다음 역할을 하나 이상 담당합니다:

| 역할 | 역할 |
|---|---|
| **Proposer(제안자)** | 합의될 값을 제안하는 노드 |
| **Acceptor(수락자)** | 제안을 수락하거나 거부하는 노드 (다수결 집단) |
| **Learner(학습자)** | 최종 합의된 값을 전달받는 노드 |

실제로는 한 노드가 세 역할을 모두 수행하는 경우가 많습니다.

### 2단계 합의 과정

**Phase 1: Prepare/Promise**

Proposer는 모든 Acceptor에게 `Prepare(n)` 메시지를 보냅니다. 여기서 `n`은 단조 증가하는 **제안 번호(proposal number)**입니다.

Acceptor는 `n`이 자신이 이전에 약속한 번호보다 크면 `Promise(n, v)` 응답을 보냅니다. 이때 `v`는 이전에 수락한 값(없으면 null)입니다. 약속 이후 Acceptor는 번호 `n` 미만의 Prepare 요청을 더 이상 수락하지 않습니다.

**Phase 2: Accept/Accepted**

과반수 이상의 Promise를 받은 Proposer는 `Accept(n, v)` 메시지를 보냅니다. `v`의 값은:
- Promise 응답에 이전 수락 값이 있다면, 그 중 가장 높은 번호의 값
- 없다면, Proposer 자신이 원하는 값

Acceptor가 `Accept(n, v)`를 받았을 때, 아직 `n`보다 큰 번호에 약속하지 않았다면 `Accepted(n, v)`를 Proposer와 Learner 모두에게 보냅니다.

과반수 이상의 `Accepted` 응답이 오면, 값 `v`가 선택(Chosen)됩니다.

---

## Python으로 구현하는 Basic Paxos

다음은 단일 프로세스 내에서 Paxos의 2단계를 시뮬레이션하는 코드입니다:

```python
import random
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class Acceptor:
    id: int
    promised_n: int = -1          # 약속한 최고 제안 번호
    accepted_n: int = -1          # 수락한 제안 번호
    accepted_v: Optional[str] = None  # 수락한 값

    def prepare(self, n: int) -> Optional[tuple]:
        """Phase 1: Prepare 메시지 처리"""
        if n > self.promised_n:
            self.promised_n = n
            # (약속 번호, 이전 수락 번호, 이전 수락 값) 반환
            return (self.promised_n, self.accepted_n, self.accepted_v)
        return None  # 거부

    def accept(self, n: int, v: str) -> bool:
        """Phase 2: Accept 메시지 처리"""
        if n >= self.promised_n:
            self.promised_n = n
            self.accepted_n = n
            self.accepted_v = v
            return True
        return False  # 거부


class Proposer:
    def __init__(self, proposal_id: int, acceptors: list[Acceptor]):
        self.id = proposal_id
        self.acceptors = acceptors
        self.quorum = len(acceptors) // 2 + 1  # 과반수

    def run(self, value: str) -> Optional[str]:
        n = self.id * 100 + random.randint(1, 99)  # 고유 번호 생성

        # ----- Phase 1: Prepare -----
        promises = []
        for acc in self.acceptors:
            resp = acc.prepare(n)
            if resp is not None:
                promises.append(resp)

        if len(promises) < self.quorum:
            print(f"[Proposer {self.id}] Phase 1 실패: 충분한 Promise 없음")
            return None

        # 이전 수락 값 중 가장 높은 번호의 값 선택
        best = max(promises, key=lambda p: p[1])
        chosen_value = best[2] if best[2] is not None else value

        print(f"[Proposer {self.id}] Phase 1 성공, 제안 값: '{chosen_value}'")

        # ----- Phase 2: Accept -----
        accepted_count = 0
        for acc in self.acceptors:
            if acc.accept(n, chosen_value):
                accepted_count += 1

        if accepted_count >= self.quorum:
            print(f"[Proposer {self.id}] 합의 성공! 선택된 값: '{chosen_value}'")
            return chosen_value
        else:
            print(f"[Proposer {self.id}] Phase 2 실패: 충분한 Accept 없음")
            return None


# 시뮬레이션: Acceptor 5개, Proposer 2개가 경쟁
acceptors = [Acceptor(i) for i in range(5)]

p1 = Proposer(1, acceptors)
p2 = Proposer(2, acceptors)

print("=== Proposer 1 시도 ===")
result1 = p1.run("value-from-p1")

print("\n=== Proposer 2 시도 ===")
result2 = p2.run("value-from-p2")
```

실행 결과:
```
=== Proposer 1 시도 ===
[Proposer 1] Phase 1 성공, 제안 값: 'value-from-p1'
[Proposer 1] 합의 성공! 선택된 값: 'value-from-p1'

=== Proposer 2 시도 ===
[Proposer 2] Phase 1 성공, 제안 값: 'value-from-p1'
[Proposer 2] 합의 성공! 선택된 값: 'value-from-p1'
```

Proposer 2가 "value-from-p2"를 제안했지만, Acceptor들이 이미 p1의 값을 수락한 상태이므로, Phase 1에서 그 값을 돌려받아 동일한 "value-from-p1"으로 합의하게 됩니다. 이것이 Paxos의 **Safety** 보장입니다.

---

## Multi-Paxos와 Liveness 문제

Basic Paxos는 단 하나의 값에 대한 합의만 처리합니다. 실제 시스템에서는 연속적인 명령(로그 항목)에 합의해야 하므로 **Multi-Paxos**를 사용합니다.

Multi-Paxos의 핵심 최적화는 **Leader 선출**입니다. 하나의 Proposer를 Leader로 지정하면, Phase 1을 한 번만 실행하고 이후 모든 합의에서 Phase 2만 반복 사용할 수 있어 오버헤드를 절반으로 줄입니다.

```python
class MultiPaxosLeader:
    """
    Leader로 선출된 Proposer. 한 번 준비 후 여러 값을 순서대로 제안.
    """
    def __init__(self, leader_n: int, acceptors: list[Acceptor]):
        self.n = leader_n
        self.acceptors = acceptors
        self.quorum = len(acceptors) // 2 + 1
        self.log: list[str] = []
        self._prepared = False

    def _do_prepare(self) -> bool:
        """Leader 임기 시작 시 한 번만 호출"""
        promises = [acc.prepare(self.n) for acc in self.acceptors]
        valid = [p for p in promises if p is not None]
        self._prepared = len(valid) >= self.quorum
        return self._prepared

    def propose(self, value: str) -> bool:
        """Phase 2만 반복 실행"""
        if not self._prepared and not self._do_prepare():
            return False

        count = sum(1 for acc in self.acceptors if acc.accept(self.n, value))
        if count >= self.quorum:
            self.log.append(value)
            return True
        return False


# 사용 예시
acceptors2 = [Acceptor(i) for i in range(5)]
leader = MultiPaxosLeader(leader_n=1000, acceptors=acceptors2)

commands = ["SET x=1", "SET y=2", "DEL z"]
for cmd in commands:
    success = leader.propose(cmd)
    print(f"{'✓' if success else '✗'} {cmd}")

print(f"\n커밋된 로그: {leader.log}")
```

---

## Paxos vs Raft: 무엇이 다른가

| 특성 | Paxos (Basic) | Raft |
|---|---|---|
| 리더 필요성 | 선택적 (Multi-Paxos는 필요) | 항상 필요 |
| 로그 복제 | 슬롯 독립적 | 순차 엄격 |
| 이해 난이도 | 높음 | 상대적으로 낮음 |
| 멤버십 변경 | 복잡 | Joint Consensus로 체계화 |
| 실사용 사례 | Chubby, Spanner, Zookeeper(ZAB) | etcd, CockroachDB, TiKV |

Raft는 "이해하기 쉬운 Paxos"를 목표로 설계되었으며, 로그가 반드시 순서대로 커밋되어야 한다는 제약을 추가해서 멤버십 변경과 스냅샷 처리를 단순화했습니다.

---

## 주의사항과 실전 팁

**1. Livelock 가능성**: 두 Proposer가 번갈아 Phase 1을 실행하면 서로의 제안을 무효화하는 무한 루프에 빠질 수 있습니다. 무작위 백오프(randomized backoff)로 해결합니다.

**2. 스토리지 필요**: Acceptor는 `promised_n`, `accepted_n`, `accepted_v`를 디스크에 영속화해야 재시작 후에도 약속을 지킬 수 있습니다. 이를 생략하면 Safety가 깨집니다.

**3. f+1 과반수 오해 주의**: `2f+1`개 노드에서 `f`개의 장애를 허용하려면 최소 `f+1`개의 Accept 응답이 필요한 것이 아니라 `f+1`개의 과반수(`f+1 > 2f+1 / 2`)가 필요합니다. 5개 노드에서 2개 장애 허용이면 과반수는 3입니다.

**4. Byzantine 장애 불가**: 기본 Paxos는 노드가 거짓말하지 않는 Crash-Stop 모델을 가정합니다. Byzantine 장애(악의적 노드)는 PBFT(Practical Byzantine Fault Tolerance) 같은 별도 프로토콜이 필요합니다.

**5. 네트워크 분할 시 진행 불가**: 과반수를 이루지 못하면 합의는 진행되지 않습니다(Safety 우선). 이것은 버그가 아니라 의도된 설계입니다. CAP 정리에서 Paxos 계열은 CP 선택입니다.

---

## 참고 자료

- [Paxos Made Simple — Leslie Lamport (2001)](https://www.cs.columbia.edu/~du/ds/assets/papers/paxos-simple.pdf)
- [The Part-Time Parliament — Leslie Lamport (1998)](https://www.cs.princeton.edu/courses/archive/fall07/cos518/papers/paxos-simple.pdf)
- [Paxos lecture — Harvard CS2620 (2026)](https://read.seas.harvard.edu/cs2620/2026/lectures/paxos)
- [Raft paper: In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf)
