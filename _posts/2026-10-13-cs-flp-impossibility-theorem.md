---
layout: post
title: "FLP 불가능성 정리 — 비동기 분산 시스템에서 합의는 왜 불가능한가"
date: 2026-10-13
categories: [cs, computer-science]
tags: [분산시스템, 합의알고리즘, FLP, Paxos, Raft, 이론컴퓨터과학]
---

## 개요

1985년 Fischer, Lynch, Paterson 세 연구자는 컴퓨터 과학 역사상 가장 중요한 불가능성 결과 중 하나를 발표했습니다. 바로 **FLP 불가능성 정리(FLP Impossibility Theorem)**입니다. 이 정리는 "완전 비동기(fully asynchronous) 분산 시스템에서 단 하나의 프로세스가 크래시될 수 있다면, 어떤 결정론적 알고리즘도 합의(consensus)를 보장할 수 없다"는 것을 수학적으로 증명합니다.

Raft, Paxos 같은 실용적인 합의 알고리즘이 존재하는데 왜 이런 불가능성 결과가 중요할까요? 그 이유와 실제 시스템이 이 한계를 어떻게 우회하는지 깊이 살펴봅니다.

---

## 합의(Consensus)란 무엇인가

합의 문제는 분산 시스템의 여러 프로세스가 단일 값에 동의하는 것입니다. 올바른 합의 알고리즘은 다음 세 가지 속성을 동시에 만족해야 합니다:

- **Agreement(합의)**: 모든 결정을 내린 프로세스는 같은 값에 동의한다.
- **Validity(유효성)**: 결정된 값은 반드시 어떤 프로세스가 제안한 값이어야 한다.
- **Termination(종료성)**: 모든 올바른 프로세스는 결국 값을 결정한다.

분산 데이터베이스의 트랜잭션 커밋, 리더 선출, 상태 복제 등 대부분의 중요한 분산 문제가 합의 문제로 환원됩니다.

---

## 비동기 시스템 모델

FLP 정리에서 가정하는 **완전 비동기 모델**의 특징:

- 프로세스는 임의의 속도로 실행된다 (전역 클럭 없음).
- 메시지 전달 지연에 상한이 없다 (메시지는 결국 도착하지만 언제인지 모른다).
- 크래시 결함(crash fault)만 고려한다 (프로세스는 멈추거나 정상적으로 동작한다).

이 모델에서 핵심 문제는 **느린 프로세스와 죽은 프로세스를 구별할 수 없다**는 점입니다. 메시지 응답이 없을 때, 그 프로세스가 크래시된 것인지 아니면 단순히 느린 것인지 판단할 방법이 없습니다.

---

## FLP 정리 증명의 핵심 아이디어

증명은 **이진 가중치 상태(bivalent configuration)**라는 개념을 중심으로 전개됩니다.

**초기 상태 분석**: 어떤 합의 알고리즘이든, 프로세스들이 입력값을 어떻게 구성하느냐에 따라 0을 결정하는 실행이나 1을 결정하는 실행이 모두 존재합니다. 이 사이에는 "아직 결정이 열려 있는" 이진 상태(bivalent state)가 반드시 존재합니다.

**핵심 보조정리**: 시스템이 이진 상태에 있을 때, 메시지를 적절히 지연시키면 항상 또 다른 이진 상태로 이동시킬 수 있습니다.

이 두 사실을 결합하면 알고리즘이 이진 상태에서 영원히 탈출하지 못하도록 만드는 실행이 항상 존재함을 보여줄 수 있습니다. 즉, Termination(종료성)을 보장할 수 없습니다.

---

## 코드로 보는 FLP 딜레마

비동기 환경에서 합의를 시도하는 단순한 예시를 통해 문제를 직관적으로 이해해 봅시다.

```python
import asyncio
import random
from typing import Optional

class Process:
    def __init__(self, pid: int, initial_value: int):
        self.pid = pid
        self.value = initial_value
        self.decided: Optional[int] = None
        self.is_crashed = False

    async def propose(self, peers: list['Process']) -> Optional[int]:
        """단순 다수결 합의 시도 - 비동기 환경에서 취약점 노출"""
        if self.is_crashed:
            return None

        votes = {self.value: 1}

        for peer in peers:
            if peer.pid == self.pid:
                continue
            # 비동기 환경: 응답 지연이 얼마나 걸릴지 모름
            delay = random.uniform(0, 2.0)  # 예측 불가한 지연
            await asyncio.sleep(delay)

            if not peer.is_crashed:
                peer_value = peer.value
                votes[peer_value] = votes.get(peer_value, 0) + 1

        # 과반수 확보 여부 판단
        total = len(peers)
        for val, count in votes.items():
            if count > total // 2:
                self.decided = val
                return val

        # 과반수를 확보 못한 경우 → 결정 불가
        return None


async def simulate_flp_problem():
    """FLP 딜레마: 느린 프로세스 vs 죽은 프로세스 구분 불가"""
    processes = [Process(i, i % 2) for i in range(5)]

    # 프로세스 2가 크래시 (또는 단순히 아주 느린 것일 수도 있음)
    processes[2].is_crashed = True

    print("합의 시도 중...")
    tasks = [
        asyncio.create_task(p.propose(processes))
        for p in processes if not p.is_crashed
    ]

    # 타임아웃 설정 없이 기다리면 영원히 기다릴 수 있음
    try:
        results = await asyncio.wait_for(
            asyncio.gather(*tasks),
            timeout=3.0
        )
        decided = [r for r in results if r is not None]
        print(f"결정된 값들: {decided}")
        if len(set(decided)) > 1:
            print("경고: 합의 위반! 서로 다른 값이 결정됨")
        elif not decided:
            print("합의 실패: 아무도 결정을 내리지 못함")
        else:
            print(f"합의 성공: {decided[0]}")
    except asyncio.TimeoutError:
        print("합의 실패: 타임아웃 - 비동기 환경에서 종료 보장 불가")


asyncio.run(simulate_flp_problem())
```

위 코드에서 `processes[2].is_crashed = True`를 `await asyncio.sleep(10)`으로 바꿔도 외부에서는 구분할 수 없습니다. 이것이 FLP 불가능성의 핵심입니다.

---

## 실제 시스템은 어떻게 우회하는가

FLP는 **불가능성**을 말하지, **실용적 시스템이 만들어질 수 없다**고 말하지 않습니다. 실제 시스템들은 다음 전략으로 우회합니다.

### 전략 1: 부분 동기화(Partial Synchrony) 가정

Raft와 Paxos는 "대부분의 경우 메시지 지연에 상한이 있다"고 가정합니다. 완전한 비동기가 아닌 **부분 동기화 모델**에서 동작합니다.

```python
class RaftNode:
    ELECTION_TIMEOUT_MIN = 150  # ms
    ELECTION_TIMEOUT_MAX = 300  # ms

    def __init__(self, node_id: int):
        self.node_id = node_id
        self.current_term = 0
        self.voted_for = None
        self.role = "follower"  # follower | candidate | leader
        self.votes_received = set()
        # 핵심: 타임아웃을 통한 부분 동기화
        self.election_timeout = random.randint(
            self.ELECTION_TIMEOUT_MIN,
            self.ELECTION_TIMEOUT_MAX
        )

    def start_election(self):
        """리더가 응답 없을 때 선거 시작 — 타임아웃 기반 부분 동기화"""
        self.current_term += 1
        self.role = "candidate"
        self.voted_for = self.node_id
        self.votes_received = {self.node_id}
        print(f"Node {self.node_id}: term {self.current_term}에서 선거 시작")
        # RequestVote RPC 전송 (실제 구현에서는 네트워크 통신)
        return {
            "term": self.current_term,
            "candidate_id": self.node_id,
            "request_type": "RequestVote"
        }

    def receive_vote(self, voter_id: int, granted: bool, total_nodes: int):
        if granted:
            self.votes_received.add(voter_id)
        if len(self.votes_received) > total_nodes // 2:
            self.role = "leader"
            print(f"Node {self.node_id}: 리더로 선출됨 (term {self.current_term})")

    def handle_heartbeat(self, leader_term: int, leader_id: int):
        """리더의 하트비트 수신 → 선거 타임아웃 리셋"""
        if leader_term >= self.current_term:
            self.current_term = leader_term
            self.role = "follower"
            # 타임아웃 리셋 — 부분 동기화의 핵심
            self.election_timeout = random.randint(
                self.ELECTION_TIMEOUT_MIN,
                self.ELECTION_TIMEOUT_MAX
            )


# Raft는 FLP를 위반하지 않음:
# - 네트워크 파티션 중에는 새 리더를 선출하지 못할 수 있음 (Termination 포기)
# - 대신 Safety(Agreement + Validity)는 항상 보장
nodes = [RaftNode(i) for i in range(5)]
# 리더 0이 죽었다고 가정
election_req = nodes[1].start_election()
print(f"선거 요청: {election_req}")
```

### 전략 2: 확률론적 합의 (Randomized Consensus)

Ben-Or 알고리즘처럼 랜덤성을 도입하면 어떤 실행도 영원히 블록되지 않도록 확률적으로 보장할 수 있습니다. 결정론적 알고리즘이 아니기 때문에 FLP 정리의 적용 대상이 아닙니다.

---

## CAP 정리와의 관계

FLP 불가능성은 CAP 정리와 깊이 연관됩니다:

| 속성 | FLP 관점 | CAP 관점 |
|------|----------|----------|
| Safety | Agreement + Validity | Consistency |
| Liveness | Termination | Availability |
| 네트워크 분단 | 메시지 지연으로 표현 | Partition Tolerance |

Raft와 Paxos는 Safety를 보장하고 Liveness를 조건부로 보장합니다 (과반수 노드가 정상일 때). 이는 CAP에서 CP 시스템에 해당합니다.

---

## 주의사항 및 실전 팁

**1. "합의가 필요한가?" 먼저 물어보라**: 많은 시스템이 합의 없이도 단조 쓰기(monotonic writes), 읽기 자신의 쓰기(read-your-writes) 등의 약한 일관성으로 충분합니다.

**2. 타임아웃 값 튜닝**: Raft에서 선거 타임아웃(150~300ms)과 하트비트 간격(50ms)의 비율이 시스템 안정성에 결정적입니다. 타임아웃이 너무 짧으면 불필요한 선거가 자주 발생합니다.

**3. 네트워크 파티션 테스트**: Jepsen 같은 도구로 네트워크 분단 시나리오를 반드시 테스트하세요. 이론적으로 안전한 알고리즘도 구현 버그로 인해 Safety를 위반할 수 있습니다.

**4. FLP는 하한(lower bound)**: FLP는 최악의 경우에 대한 이야기입니다. 정상 운영 환경에서 Raft는 매우 빠르게 합의에 도달합니다.

---

## 정리

FLP 불가능성 정리는 분산 컴퓨팅의 근본적인 한계를 정의합니다. 비동기 시스템에서 Safety와 Liveness를 동시에 완벽히 보장하는 합의 알고리즘은 존재하지 않습니다. 실제 시스템(Raft, Paxos, ZAB 등)은 부분 동기화 가정을 추가하거나 Liveness를 조건부로만 보장함으로써 이 한계를 우회합니다. FLP를 이해하면 분산 시스템 설계 시 어떤 트레이드오프가 필연적인지를 명확히 파악할 수 있습니다.

## 참고 자료
- [Fischer, Lynch, Paterson (1985) - Impossibility of Distributed Consensus with One Faulty Process](https://dl.acm.org/doi/10.1145/3149.214121)
- [A Brief Tour of FLP Impossibility — The Paper Trail](https://www.the-paper-trail.org/post/2008-08-13-a-brief-tour-of-flp-impossibility/)
- [MIT 6.824 Distributed Systems Lectures](https://pdos.csail.mit.edu/6.824/)
- [Wikipedia: Fischer–Lynch–Paterson impossibility](https://en.wikipedia.org/wiki/Fischer%E2%80%93Lynch%E2%80%93Paterson_impossibility)
