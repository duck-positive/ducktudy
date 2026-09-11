---
layout: post
title: "ZAB 프로토콜 완전 정복: ZooKeeper가 분산 합의를 구현하는 원자적 브로드캐스트"
date: 2026-09-11
categories: [cs, computer-science]
tags: [zookeeper, zab, distributed-systems, consensus, atomic-broadcast, leader-election]
---

ZooKeeper Atomic Broadcast(ZAB)는 Apache ZooKeeper의 핵심 합의 알고리즘입니다. Raft나 Paxos와 달리, ZAB는 단순한 합의를 넘어 **완전 순서가 보장된 원자적 브로드캐스트**를 목표로 설계되었습니다. Kafka의 컨트롤러 선출, Hadoop의 NameNode 고가용성, HBase의 마스터 선출 등 수많은 대형 분산 시스템의 뒤에 ZAB가 자리하고 있습니다.

## ZAB란 무엇인가: 개념과 배경

ZAB(ZooKeeper Atomic Broadcast)는 2010년 Yahoo! Research에서 발표한 **Primary-Backup 시스템을 위한 원자적 브로드캐스트 프로토콜**입니다. Paxos에서 영감을 받았지만 핵심적인 차이가 있습니다. Paxos는 단일 값에 대한 합의를 반복하는 반면, ZAB는 **전체 트랜잭션 히스토리의 순서를 보존**하며 이를 효율적으로 복제하는 데 특화되어 있습니다.

### ZAB의 핵심 보장

ZAB는 두 가지 핵심 속성을 보장합니다.

**원자성(Atomicity)**: 브로드캐스트된 메시지는 모든 서버에서 전달되거나 어느 서버에서도 전달되지 않습니다. 부분적으로만 커밋되는 상황은 발생하지 않습니다.

**순서 보존(Ordering)**: 메시지 A가 B보다 먼저 브로드캐스트되었다면, A는 반드시 B보다 먼저 전달됩니다. 더 나아가 인과적 순서(causal ordering)도 보장합니다. 즉, 메시지 A를 전달받은 후 생성된 메시지 B는 반드시 A 이후에 전달됩니다.

### ZooKeeper 앙상블과 쿼럼

ZooKeeper는 홀수 개의 서버로 구성된 **앙상블(Ensemble)**에서 실행됩니다. 보통 3개 또는 5개 서버를 사용하며, 과반수가 살아 있으면 서비스를 지속할 수 있습니다. 이를 **쿼럼(Quorum)** 이라 합니다. 3대 구성에서는 2대, 5대 구성에서는 3대가 살아 있으면 동작합니다.

```
ZooKeeper 앙상블 (5 노드, quorum = 3)

   Leader
   [서버 1]  ──────────── 상태 브로드캐스트 ──────────────┐
       │                                                    │
       ├──── ACK ────── [서버 2] Follower               [서버 3] Follower
       │                                                    │
       └──── ACK ────── [서버 4] Follower               [서버 5] Follower

Quorum(3)이 ACK를 보내면 COMMIT 전송 → 모든 Follower에 커밋 적용
```

## 왜 ZAB가 필요한가: 분산 환경의 과제

분산 시스템에서는 네트워크 파티션, 서버 크래시, 메시지 유실 등 다양한 장애가 발생합니다. 이런 환경에서도 데이터 일관성을 유지하려면 정교한 프로토콜이 필요합니다.

### Paxos의 한계

Paxos는 범용 합의 알고리즘이지만, ZooKeeper처럼 **연속된 상태 변경을 순서대로 복제**하는 시나리오에서는 비효율적입니다. 각 상태 변경마다 독립적으로 Paxos 라운드를 실행하면 인과 관계가 깨질 수 있고, 오버헤드도 큽니다.

### ZAB의 설계 목표

ZAB는 다음 시나리오에서 Paxos보다 효율적으로 동작하도록 설계되었습니다.

- **연속 쓰기 최적화**: 리더가 파이프라이닝 방식으로 여러 트랜잭션을 동시에 제안할 수 있습니다.
- **빠른 복구**: 새 리더가 선출되면 이전 리더의 미완료 트랜잭션을 안전하게 복구합니다.
- **단조 증가 zxid**: 각 트랜잭션은 `(epoch, counter)` 형식의 고유 식별자를 가지며, 엄격하게 단조 증가합니다.

## ZAB의 세 가지 단계

ZAB 프로토콜은 세 단계로 구성됩니다.

### 단계 1: Fast Leader Election (리더 선출)

서버가 시작하거나 현재 리더와의 연결이 끊어지면 선출 단계가 시작됩니다.

```
선출 투표 과정:
┌─────────────────────────────────────────────┐
│ 각 서버는 투표를 브로드캐스트               │
│ 투표 내용: (myId, myZxid)                   │
│                                             │
│ 규칙: 더 큰 zxid를 가진 서버 선호          │
│       zxid가 같으면 더 큰 myId 선호         │
│                                             │
│ 과반수(quorum) 투표를 받은 서버 → 리더 당선  │
└─────────────────────────────────────────────┘
```

핵심은 **가장 최신 데이터(가장 큰 zxid)를 가진 서버가 리더가 된다**는 점입니다. 이를 통해 리더는 항상 가장 완전한 트랜잭션 히스토리를 보유함을 보장합니다.

### 단계 2: Recovery (복구 및 동기화)

새 리더가 선출되면 팔로워들을 자신의 상태와 동기화합니다.

```python
# ZAB Recovery 단계 의사 코드

class Leader:
    def recovery_phase(self, epoch):
        # 새 에포크 시작
        self.epoch = epoch
        
        # 팔로워들로부터 히스토리 정보 수집
        follower_histories = {}
        for follower in self.followers:
            info = follower.get_history_info()
            follower_histories[follower] = info
        
        # 가장 앞선 팔로워의 히스토리를 기준으로 동기화
        for follower in self.followers:
            their_last_zxid = follower_histories[follower]['last_zxid']
            my_last_zxid = self.last_zxid
            
            if their_last_zxid < my_last_zxid:
                # 팔로워가 뒤처진 경우 누락된 트랜잭션 전송
                missing_txns = self.get_txns_since(their_last_zxid)
                follower.send_diff(missing_txns)
            elif their_last_zxid > my_last_zxid:
                # 팔로워가 더 앞선 경우 (비정상 - 트런케이트)
                follower.truncate(my_last_zxid)
        
        # 쿼럼이 동기화되면 브로드캐스트 단계 시작
        if self.synchronized_count >= self.quorum_size:
            self.start_broadcast_phase()

class Follower:
    def sync_with_leader(self, leader):
        # SNAP, DIFF, TRUNC 중 하나의 방식으로 동기화
        sync_type = leader.determine_sync_type(self.last_zxid)
        
        if sync_type == 'DIFF':
            self.apply_diff(leader.get_diff(self.last_zxid))
        elif sync_type == 'SNAP':
            # 전체 스냅샷 전송 (차이가 너무 클 때)
            self.restore_from_snapshot(leader.get_snapshot())
        elif sync_type == 'TRUNC':
            # 팔로워가 커밋되지 않은 데이터를 가진 경우 롤백
            self.truncate_to(leader.last_zxid)
```

### 단계 3: Broadcast (브로드캐스트)

정상 동작 단계입니다. 리더는 클라이언트의 쓰기 요청을 받아 트랜잭션으로 변환하고 팔로워에 복제합니다.

```java
// ZAB Broadcast 단계 - Java 의사 코드

public class ZabBroadcast {
    private final long epoch;
    private final AtomicLong counter = new AtomicLong(0);
    private final List<Follower> followers;
    private final int quorumSize;
    
    // 새 zxid 생성: 상위 32비트 = epoch, 하위 32비트 = counter
    private long nextZxid() {
        long cnt = counter.incrementAndGet();
        return (epoch << 32) | (cnt & 0xFFFFFFFFL);
    }
    
    // 브로드캐스트: 2단계 커밋 (propose → commit)
    public boolean broadcast(byte[] data) throws Exception {
        long zxid = nextZxid();
        
        // 1단계: PROPOSAL 전송 (2PC의 prepare에 해당)
        CountDownLatch ackLatch = new CountDownLatch(quorumSize);
        for (Follower f : followers) {
            f.sendProposal(zxid, data, ackLatch);
        }
        
        // 쿼럼 ACK를 기다림 (타임아웃 포함)
        boolean quorumAcked = ackLatch.await(5, TimeUnit.SECONDS);
        if (!quorumAcked) {
            throw new QuorumTimeoutException("Broadcast failed");
        }
        
        // 2단계: COMMIT 전송 (쿼럼이 ACK하면 즉시 커밋)
        for (Follower f : followers) {
            f.sendCommit(zxid);
        }
        
        // 리더도 로컬에 커밋
        applyToStateMachine(data);
        return true;
    }
    
    // 팔로워로부터 ACK 수신 처리
    public void onFollowerAck(long zxid, Follower follower) {
        AckTracker tracker = ackTrackers.get(zxid);
        if (tracker != null) {
            tracker.recordAck(follower);
            // 쿼럼 달성 여부는 CountDownLatch가 관리
        }
    }
}
```

**zxid 구조**: ZAB의 트랜잭션 ID인 `zxid`는 64비트로 구성됩니다. 상위 32비트는 **에포크(epoch)**로 리더가 바뀔 때마다 증가하고, 하위 32비트는 **카운터**로 동일 에포크 내 순서를 나타냅니다. 이 구조 덕분에 새 리더가 선출될 때 이전 에포크의 미완료 트랜잭션을 명확히 식별하고 처리할 수 있습니다.

## ZAB vs Raft: 무엇이 다른가

| 특성 | ZAB | Raft |
|------|-----|------|
| 설계 목적 | Primary-Backup 복제에 특화 | 범용 합의 |
| 파이프라이닝 | 지원 (여러 proposal 동시 진행) | 지원 |
| 스냅샷 복구 | SNAP/DIFF/TRUNC 세 가지 방식 | Snapshot 설치 |
| zxid | (epoch, counter) 64비트 | (term, index) 별도 관리 |
| 읽기 일관성 | 기본적으로 리더에서만 강한 일관성 | 구현에 따라 다름 |
| 주요 사용처 | ZooKeeper | etcd, TiKV, CockroachDB |

## 실전 구현: ZooKeeper 클라이언트로 분산 락 구현

ZAB 프로토콜이 제공하는 강한 일관성을 활용해 분산 락을 구현해 보겠습니다.

```java
import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.CountDownLatch;

public class DistributedLock implements Watcher {
    private final ZooKeeper zk;
    private final String lockPath;
    private String currentNode;
    private final CountDownLatch connectedLatch = new CountDownLatch(1);

    public DistributedLock(String connectString, String lockPath) throws Exception {
        this.lockPath = lockPath;
        // ZAB 기반 ZooKeeper 클러스터에 연결
        this.zk = new ZooKeeper(connectString, 3000, this);
        connectedLatch.await(); // 연결 완료 대기
    }

    @Override
    public void process(WatchedEvent event) {
        if (event.getState() == Event.KeeperState.SyncConnected) {
            connectedLatch.countDown();
        }
    }

    public void lock() throws Exception {
        // /locks 경로가 없으면 생성
        ensurePath(lockPath);
        
        // EPHEMERAL_SEQUENTIAL 노드 생성: 순서가 보장된 임시 노드
        // ZAB의 원자적 순서 보장 덕분에 zxid 기반으로 노드 번호가 결정됨
        currentNode = zk.create(
            lockPath + "/lock-",
            new byte[0],
            ZooDefs.Ids.OPEN_ACL_UNSAFE,
            CreateMode.EPHEMERAL_SEQUENTIAL
        );
        
        while (true) {
            // 현재 존재하는 모든 락 노드 조회
            List<String> children = zk.getChildren(lockPath, false);
            Collections.sort(children);
            
            String myNode = currentNode.substring(lockPath.length() + 1);
            int myIndex = children.indexOf(myNode);
            
            if (myIndex == 0) {
                // 가장 작은 번호 → 락 획득!
                System.out.println("Lock acquired: " + currentNode);
                return;
            }
            
            // 바로 앞 노드를 감시 (Watch)
            String prevNode = lockPath + "/" + children.get(myIndex - 1);
            CountDownLatch waitLatch = new CountDownLatch(1);
            
            Stat stat = zk.exists(prevNode, event -> {
                if (event.getType() == Event.EventType.NodeDeleted) {
                    waitLatch.countDown();
                }
            });
            
            if (stat != null) {
                // 앞 노드가 존재하면 삭제될 때까지 대기
                waitLatch.await();
            }
            // 앞 노드가 없어지면 다시 시도
        }
    }

    public void unlock() throws Exception {
        if (currentNode != null) {
            zk.delete(currentNode, -1);
            currentNode = null;
            System.out.println("Lock released");
        }
    }

    private void ensurePath(String path) throws Exception {
        if (zk.exists(path, false) == null) {
            try {
                zk.create(path, new byte[0],
                    ZooDefs.Ids.OPEN_ACL_UNSAFE,
                    CreateMode.PERSISTENT);
            } catch (KeeperException.NodeExistsException e) {
                // 다른 클라이언트가 먼저 생성 - 무시
            }
        }
    }
}
```

이 구현은 ZAB의 원자적 순서 보장 덕분에 안전합니다. `EPHEMERAL_SEQUENTIAL` 노드의 번호는 ZAB가 처리한 zxid 순서로 결정되므로, 어떤 서버에서 보더라도 동일한 순서가 관찰됩니다.

## 주의사항 및 팁

### 1. Watch 이벤트의 일회성

ZooKeeper의 Watch는 **한 번만** 발동됩니다. Watch를 설정한 후 이벤트가 발생하면, 다시 Watch를 설정해야 합니다. 이를 놓치면 변경사항을 놓칠 수 있습니다.

```java
// 잘못된 패턴: Watch가 한 번만 발동
zk.getChildren(path, true); // Watch 설정

// 올바른 패턴: 매번 재등록
void watchChildren(String path) throws Exception {
    zk.getChildren(path, event -> {
        handleChildrenChange(event);
        // 이벤트 처리 후 즉시 재등록
        try { watchChildren(path); } catch (Exception e) { /* handle */ }
    });
}
```

### 2. 세션 만료(Session Expiration) 처리

ZooKeeper 세션이 만료되면 `EPHEMERAL` 노드가 자동으로 삭제됩니다. 분산 락, 리더 선출 등에서 세션 만료를 반드시 처리해야 합니다.

```java
@Override
public void process(WatchedEvent event) {
    if (event.getState() == Event.KeeperState.Expired) {
        // 세션 만료 시 재연결 및 상태 복구 필수
        reconnect();
    }
}
```

### 3. ZAB의 쓰기 병목

모든 쓰기 요청은 리더를 거쳐야 합니다. 쓰기가 많은 워크로드에서는 리더가 병목이 될 수 있습니다. ZooKeeper는 읽기 중심 워크로드(설정 관리, 서비스 디스커버리 등)에 최적화되어 있습니다. 초당 수만 건 이상의 쓰기가 필요하다면 다른 솔루션을 고려하세요.

### 4. 홀수 개 서버 구성

ZooKeeper는 반드시 홀수 개 서버로 구성해야 합니다. 4대보다 3대, 6대보다 5대가 장애 허용 측면에서 동일한 성능을 제공하면서 비용을 줄입니다. 4대 구성은 3대와 마찬가지로 단 1대 장애만 허용하므로 서버만 낭비됩니다.

### 5. 데이터 크기 제한

ZooKeeper znode의 기본 최대 데이터 크기는 **1MB**입니다. ZooKeeper는 소규모 메타데이터 저장에 최적화되어 있으며, 대용량 데이터는 별도 저장소(HDFS, S3 등)에 저장하고 ZooKeeper에는 경로나 포인터만 저장하는 패턴을 사용하세요.

## 참고 자료
- [Apache ZooKeeper GitHub Repository](https://github.com/apache/zookeeper)
- [LMAX Disruptor - High Performance Inter-Thread Messaging](https://github.com/LMAX-Exchange/disruptor)
