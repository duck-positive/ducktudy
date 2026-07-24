---
layout: post
title: "Chandy-Lamport 분산 스냅샷 알고리즘 완전 정복: 실행 중인 분산 시스템의 전역 상태를 멈추지 않고 캡처하는 법"
date: 2026-07-24
categories: [cs, computer-science]
tags: [distributed-systems, snapshot, chandy-lamport, global-state, fault-tolerance, checkpointing]
---

## 개념 설명

분산 시스템에서 가장 다루기 어려운 문제 중 하나는 "지금 이 순간, 시스템 전체의 상태가 어떠한가?"라는 질문에 답하는 것이다. 단일 서버라면 프로세스를 잠시 멈추고 메모리를 덤프하면 그만이다. 하지만 수십, 수백 개의 노드가 메시지를 주고받으며 동시에 실행되는 분산 시스템에서는 이 질문이 훨씬 복잡해진다.

Chandy-Lamport 알고리즘은 1985년 K. Mani Chandy와 Leslie Lamport가 발표한 논문 "Distributed Snapshots: Determining Global States of Distributed Systems"에서 소개된 알고리즘이다. 이 알고리즘의 핵심 통찰은 단순하면서도 우아하다: **시스템을 멈추지 않고도 일관된 전역 상태(global consistent state)를 기록할 수 있다**는 것이다.

### 전역 상태란 무엇인가

분산 시스템의 전역 상태는 두 가지 요소로 구성된다:

- **로컬 상태(local state)**: 각 프로세스의 내부 상태 (변수, 메모리 등)
- **채널 상태(channel state)**: 네트워크 채널에 "전송 중"인 메시지들

문제는 이 두 상태를 동시에 캡처해야 "의미 있는" 스냅샷이 된다는 것이다. 프로세스 A가 메시지를 보내고, 그 메시지가 아직 채널에 있는데 프로세스 B의 상태만 기록하면, 메시지가 사라진 것처럼 보이는 불일치가 발생한다.

### 모델 가정

Chandy-Lamport 알고리즘이 동작하려면 다음 가정이 필요하다:

1. **FIFO 채널**: 동일 채널에서 메시지는 보낸 순서대로 도착한다
2. **신뢰할 수 있는 채널**: 메시지가 유실되지 않는다
3. **연결성**: 모든 프로세스는 임의의 다른 프로세스와 직간접적으로 통신할 수 있다
4. **프로세스 정상 동작**: 스냅샷 도중 프로세스가 죽지 않는다

---

## 왜 필요한가

분산 스냅샷은 실제 시스템에서 다양한 목적으로 활용된다.

**체크포인팅(Checkpointing)과 장애 복구**: 장시간 실행되는 분산 컴퓨팅 작업(예: 분산 머신러닝 훈련, 하둡 맵리듀스 잡)에서 중간 상태를 저장해 두면, 노드 장애 시 처음부터 재시작하지 않고 마지막 체크포인트에서 재개할 수 있다.

**분산 데드락 감지**: 프로세스 A가 B의 자원을 기다리고, B는 A의 자원을 기다리는 상황을 탐지하려면 시스템의 "대기 그래프(wait-for graph)"를 분석해야 한다. 이 그래프를 올바르게 캡처하려면 일관된 전역 스냅샷이 필요하다.

**분산 가비지 컬렉션**: 어떤 객체가 더 이상 어느 프로세스에서도 참조되지 않는지 판단하려면 시스템 전체의 참조 관계를 파악해야 한다.

**안정적 서술(stable property) 확인**: "시스템이 종료되었는가?" 또는 "교착 상태가 발생했는가?" 같은 질문은 한 번 참이 되면 영원히 참인 안정적 서술에 해당하며, 스냅샷으로 이를 검증할 수 있다.

---

## 실제 구현 예제

### 알고리즘 핵심 로직

알고리즘에서 핵심 역할을 하는 것은 **마커(Marker) 메시지**다. 마커는 일반 메시지와 구별되는 특별한 토큰으로, "지금부터 채널 상태 기록을 시작하라"는 신호다.

```python
import threading
from collections import defaultdict
from enum import Enum

class MarkerMessage:
    """스냅샷 시작을 알리는 마커 메시지"""
    pass

class SnapshotState(Enum):
    IDLE = "idle"
    RECORDING = "recording"
    DONE = "done"

class Process:
    def __init__(self, pid, channels_in, channels_out):
        self.pid = pid
        self.state = None          # 로컬 상태
        self.snapshot_state = SnapshotState.IDLE
        self.local_snapshot = None
        # 채널 상태: 채널별로 기록할 메시지 목록
        self.channel_snapshots = defaultdict(list)
        # 마커를 아직 받지 못한 입력 채널 집합
        self.waiting_for_marker_from = set()
        self.channels_in = channels_in    # {sender_pid: Queue}
        self.channels_out = channels_out  # {receiver_pid: Queue}
        self.lock = threading.Lock()

    def initiate_snapshot(self):
        """이 프로세스가 스냅샷을 시작한다."""
        with self.lock:
            if self.snapshot_state != SnapshotState.IDLE:
                return
            # 1. 자신의 로컬 상태를 기록
            self.local_snapshot = self.state
            self.snapshot_state = SnapshotState.RECORDING
            # 2. 모든 입력 채널에서 마커를 기다린다
            self.waiting_for_marker_from = set(self.channels_in.keys())
            print(f"[{self.pid}] 스냅샷 시작: 로컬 상태 = {self.local_snapshot}")

        # 3. 모든 출력 채널로 마커를 전송
        self._send_marker_to_all()

    def _send_marker_to_all(self):
        for receiver_pid, channel in self.channels_out.items():
            channel.put(MarkerMessage())
            print(f"[{self.pid}] → [{receiver_pid}] 마커 전송")

    def receive_message(self, sender_pid, msg):
        """메시지 수신 처리"""
        with self.lock:
            if isinstance(msg, MarkerMessage):
                self._handle_marker(sender_pid)
            else:
                self._handle_data_message(sender_pid, msg)

    def _handle_data_message(self, sender_pid, msg):
        if self.snapshot_state == SnapshotState.RECORDING:
            # 마커가 도착하기 전에 온 메시지는 채널 상태의 일부
            if sender_pid in self.waiting_for_marker_from:
                self.channel_snapshots[sender_pid].append(msg)
                print(f"[{self.pid}] 채널 상태 기록: {sender_pid} → 메시지 {msg}")
        # 항상 일반 처리도 수행
        self._process(msg)

    def _handle_marker(self, sender_pid):
        if self.snapshot_state == SnapshotState.IDLE:
            # 처음 마커를 받으면 스냅샷을 시작
            self.local_snapshot = self.state
            self.snapshot_state = SnapshotState.RECORDING
            self.waiting_for_marker_from = set(self.channels_in.keys())
            self.waiting_for_marker_from.discard(sender_pid)
            print(f"[{self.pid}] 첫 마커 수신 ({sender_pid}로부터): 로컬 상태 = {self.local_snapshot}")
            # 마커를 전달
            self._send_marker_to_all()
        elif self.snapshot_state == SnapshotState.RECORDING:
            # 이미 기록 중이면 해당 채널 기록 완료
            self.waiting_for_marker_from.discard(sender_pid)
            print(f"[{self.pid}] 마커 수신 완료 ({sender_pid}), 남은 채널: {self.waiting_for_marker_from}")

        # 모든 채널에서 마커를 받으면 스냅샷 완료
        if not self.waiting_for_marker_from and self.snapshot_state == SnapshotState.RECORDING:
            self.snapshot_state = SnapshotState.DONE
            print(f"[{self.pid}] 스냅샷 완료! 채널 상태: {dict(self.channel_snapshots)}")

    def _process(self, msg):
        """실제 비즈니스 로직 처리 (예시)"""
        self.state = msg
```

### 시뮬레이션 실행

```python
import queue
import time

def run_simulation():
    """3개 프로세스 간 Chandy-Lamport 스냅샷 시뮬레이션"""
    # 채널 생성: 단방향 FIFO 큐
    # P0 → P1, P1 → P2, P2 → P0 (링 토폴로지)
    ch_0_to_1 = queue.Queue()
    ch_1_to_2 = queue.Queue()
    ch_2_to_0 = queue.Queue()

    p0 = Process("P0",
                 channels_in={"P2": ch_2_to_0},
                 channels_out={"P1": ch_0_to_1})
    p1 = Process("P1",
                 channels_in={"P0": ch_0_to_1},
                 channels_out={"P2": ch_1_to_2})
    p2 = Process("P2",
                 channels_in={"P1": ch_1_to_2},
                 channels_out={"P0": ch_2_to_0})

    p0.state = 100
    p1.state = 200
    p2.state = 300

    # P0이 P1으로 메시지 전송 (스냅샷 시작 전)
    ch_0_to_1.put("msg_A")

    # P0이 스냅샷 시작
    p0.initiate_snapshot()

    # 스냅샷 시작 후 P0이 추가 메시지 전송 (채널 상태에 포함될 수 있음)
    ch_0_to_1.put("msg_B")

    # 각 프로세스가 메시지를 처리 (실제 환경에서는 별도 스레드)
    # P1이 메시지 처리
    while not ch_0_to_1.empty():
        msg = ch_0_to_1.get()
        p1.receive_message("P0", msg)

    # P1의 마커가 P2로 전달되었다고 가정
    while not ch_1_to_2.empty():
        msg = ch_1_to_2.get()
        p2.receive_message("P1", msg)

    # P2의 마커가 P0로 전달
    while not ch_2_to_0.empty():
        msg = ch_2_to_0.get()
        p0.receive_message("P2", msg)

    print("\n=== 스냅샷 결과 ===")
    for p in [p0, p1, p2]:
        print(f"[{p.pid}] 로컬 상태: {p.local_snapshot}, 채널 상태: {dict(p.channel_snapshots)}")

run_simulation()
```

이 시뮬레이션의 핵심 포인트는 `msg_A`는 P1의 로컬 상태 기록 후에 처리되더라도 "채널 상태"로 기록되고, `msg_B`는 마커 이후에 보내졌으므로 채널 상태에 포함되지 않는다는 점이다.

---

## 알고리즘 정확성 증명 직관

Chandy-Lamport 스냅샷의 일관성은 "마커가 전달되는 순서"에 의해 보장된다.

- 프로세스 Q가 채널 C를 통해 마커를 받기 전에 채널 C로 들어온 모든 메시지는 Q의 채널 상태에 기록된다.
- 마커가 도착한 후 채널 C로 들어오는 메시지는 Q의 채널 상태에 포함되지 않는다.

이 규칙 덕분에 기록된 전역 상태 (모든 로컬 상태 + 모든 채널 상태)는 **실제로 발생 가능한 실행의 한 시점**에 해당하는 일관된 컷(consistent cut)을 형성한다. 기록된 상태가 실제 실행 중 동시에 존재했던 상태일 필요는 없다. 하지만 이 상태에서 시스템을 재시작하면 올바른 실행이 이어진다는 것이 핵심이다.

---

## 주의사항과 실전 팁

**FIFO 채널 가정 위반 시 대처**: TCP 연결은 FIFO를 보장하지만, UDP나 여러 경로를 가진 네트워크에서는 그렇지 않다. 이 경우 각 메시지에 시퀀스 번호를 붙여 순서를 강제하거나, 마커와 함께 "몇 번째 메시지 이전은 채널 상태에 포함"이라는 정보를 함께 전달해야 한다.

**스냅샷 개시자(initiator) 선정**: 여러 프로세스가 동시에 스냅샷을 시작할 수 있다. 이 경우 스냅샷 ID를 마커에 포함시켜 서로 다른 스냅샷을 구분한다. 각 프로세스는 가장 높은 ID의 스냅샷만 유지하거나, 모든 스냅샷을 독립적으로 처리할 수 있다.

**실제 분산 시스템에서의 응용**: Apache Flink는 Chandy-Lamport 알고리즘을 변형한 ABS(Asynchronous Barrier Snapshotting)를 사용해 스트림 처리 중 정확히 한 번(exactly-once) 처리를 보장한다. Flink는 소스에서 배리어(barrier)를 주기적으로 주입하고, 각 연산자가 배리어를 받으면 자신의 상태를 체크포인트로 저장한다.

**채널 상태 오버헤드**: 실제로 채널에 메시지가 많다면 채널 상태를 저장하는 비용이 크다. 이를 줄이기 위해 "로컬 체크포인트"와 "메시지 로그"를 조합하는 방식도 사용된다. 메시지를 직접 저장하는 대신 "스냅샷 이후 수신한 메시지 로그"만 유지하면 재생(replay)으로 채널 상태를 복원할 수 있다.

**Flink와의 차이**: Flink의 ABS는 백프레셔(backpressure) 없이 동작하도록 설계되었고, 배리어 정렬(barrier alignment)과 비정렬(unaligned) 두 가지 모드를 제공한다. 비정렬 모드에서는 배리어보다 먼저 도착한 레코드를 버퍼에 저장해 처리량을 유지한다.

## 참고 자료

- [Chandy–Lamport algorithm - Wikipedia](https://en.wikipedia.org/wiki/Chandy%E2%80%93Lamport_algorithm)
- [COS418 Assignment 2: Chandy-Lamport Distributed Snapshots - Princeton](https://www.cs.princeton.edu/courses/archive/fall17/cos418/a2.html)
- [Distributed Snapshots Lecture - Princeton COS418](https://www.cs.princeton.edu/courses/archive/spring21/cos418/docs/L7-snapshots.pdf)
- [Apache Flink: Lightweight Asynchronous Snapshots for Distributed Dataflows](https://arxiv.org/abs/1506.08603)
