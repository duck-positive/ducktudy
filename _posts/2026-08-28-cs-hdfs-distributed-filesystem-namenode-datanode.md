---
layout: post
title: "HDFS 분산 파일 시스템 완전 정복: NameNode·DataNode·블록 복제와 Rack Awareness"
date: 2026-08-28
categories: [cs, computer-science]
tags: [hdfs, hadoop, distributed-filesystem, namenode, datanode, block-replication, rack-awareness, fault-tolerance, big-data]
---

2003년 Google이 발표한 GFS(Google File System) 논문은 분산 파일 시스템 설계의 패러다임을 바꿨다. Hadoop의 **HDFS(Hadoop Distributed File System)**는 GFS의 아이디어를 오픈소스로 구현한 것으로, 수천 대의 범용 서버에서 페타바이트 규모의 데이터를 저장하고 처리하는 기반이 됐다. 스트리밍 읽기와 일괄 처리에 최적화된 HDFS의 설계 결정을 이해하면, 왜 특정 워크로드에는 완벽하지만 다른 워크로드에는 맞지 않는지 명확하게 알 수 있다.

## 개념 설명: HDFS의 설계 철학

### 핵심 설계 가정

HDFS는 다음과 같은 특수한 환경을 가정하고 설계됐다.

1. **하드웨어 장애는 예외가 아닌 정상이다**: 수천 대의 범용 서버로 구성된 클러스터에서는 매일 일부 디스크나 서버가 고장난다. HDFS는 장애를 감지하고 자동으로 복구한다.
2. **파일은 크고 한 번 쓰고 여러 번 읽는다**: MapReduce 잡이나 로그 집계처럼 대용량 파일(GB~TB)을 순차적으로 읽는 워크로드에 최적화됐다.
3. **랜덤 쓰기보다 순차 스트리밍이 우선이다**: HDFS는 낮은 지연(latency)보다 높은 처리량(throughput)을 선택한다.
4. **Write-Once, Read-Many**: 파일을 한 번 쓰면 수정하지 않는 불변(immutable) 패턴을 따른다.

### HDFS의 주요 한계

이런 설계 가정 때문에 HDFS는 다음 용도에는 적합하지 않다.
- 수백만 개의 소규모 파일(NameNode 메모리 부족)
- 낮은 지연이 필요한 랜덤 액세스
- 다중 쓰기(POSIX 표준의 `O_RDWR` 패턴)

## HDFS의 구성 요소

### NameNode: 메타데이터의 중앙 관리자

**NameNode**는 HDFS 클러스터의 마스터 노드다. 파일 시스템의 네임스페이스(디렉터리 트리, 파일 권한, 파일→블록 매핑)를 전적으로 관리한다.

NameNode가 보관하는 정보:
- **FsImage**: 파일 시스템 메타데이터의 전체 스냅샷 (디스크에 영구 저장)
- **EditLog**: FsImage 생성 이후의 모든 변경사항 (트랜잭션 로그)
- **블록→DataNode 위치 매핑**: 재시작 시 DataNode의 Block Report로 재구성 (메모리에만 유지)

NameNode는 실제 파일 데이터를 저장하지 않는다. 단지 "어떤 파일의 어떤 블록이 어느 DataNode에 있는가"만 안다.

**SPOF(Single Point of Failure) 문제**: 초기 Hadoop은 NameNode가 하나뿐이어서, NameNode가 죽으면 전체 클러스터가 마비됐다. Hadoop 2.x부터 **HA(High Availability) NameNode**가 도입되어 Active/Standby 쌍으로 운영한다. 두 NameNode는 JournalNode 클러스터(ZooKeeper 기반)를 통해 EditLog를 공유한다.

### DataNode: 실제 데이터 저장소

**DataNode**는 슬레이브 노드로, 파일 블록을 로컬 디스크에 저장한다. DataNode는 주기적으로 NameNode에게 두 가지 메시지를 보낸다.

- **Heartbeat** (기본 3초 간격): "나는 살아 있다" 신호. 10분 이상 Heartbeat가 없으면 NameNode는 해당 DataNode를 죽은 것으로 간주하고 블록을 재복제한다.
- **Block Report** (기본 6시간 간격): 해당 DataNode가 보유한 모든 블록 목록. NameNode가 재시작될 때 블록 위치 정보를 재구성하는 데 사용된다.

### Secondary NameNode: 이름의 함정

**Secondary NameNode**는 백업 NameNode가 아니다. 이름이 오해를 일으키지만, 실제 역할은 **CheckPointing**: FsImage와 EditLog를 주기적으로 병합하여 EditLog가 무한정 커지는 것을 방지한다. Primary NameNode 장애 시 즉시 failover 할 수 없다.

## 블록 복제 메커니즘과 Rack Awareness

### 블록 구조

HDFS는 파일을 고정 크기의 **블록(Block)**으로 나눠 저장한다. 기본 블록 크기는 128MB(Hadoop 2.x 이전은 64MB)이다. 100MB 파일은 1개의 블록을 사용하고, 300MB 파일은 3개의 블록(128MB + 128MB + 44MB)을 사용한다.

큰 블록 크기의 이점:
- 클라이언트-NameNode 통신 횟수 감소
- 스트리밍 읽기 시 디스크 탐색 시간 비율 감소

### 복제 정책 (기본 복제 계수 3)

HDFS는 각 블록을 기본 3개의 DataNode에 복제한다. 단순히 3곳에 무작위로 복제하면 같은 랙(Rack)의 서버가 전원 공급 장치나 스위치 고장으로 동시에 사라질 때 데이터 손실이 발생할 수 있다.

**Rack Awareness** 복제 배치 전략 (기본 3-복제):
1. **첫 번째 복제본**: 클라이언트와 같은 랙의 DataNode (단, 쓰기 클라이언트가 DataNode라면 로컬)
2. **두 번째 복제본**: 첫 번째와 **다른 랙**의 DataNode
3. **세 번째 복제본**: 두 번째와 **같은 랙**의 다른 DataNode

이 전략은 다음 균형을 달성한다.
- **신뢰성**: 단일 랙 전체 장애에도 두 번째 복제본이 다른 랙에 존재
- **쓰기 성능**: 두 번째·세 번째 복제본이 같은 랙이므로 크로스-랙 트래픽을 최소화
- **읽기 성능**: 클라이언트에서 가장 가까운 복제본을 선택

## 실제 구현 예제

### 예제 1: HDFS 파일 읽기/쓰기 흐름 시뮬레이터 (Python)

```python
import random
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class Block:
    block_id: int
    size: int
    replicas: list[str] = field(default_factory=list)  # DataNode 목록

@dataclass
class DataNode:
    name: str
    rack: str
    blocks: set[int] = field(default_factory=set)
    alive: bool = True

class HDFSSimulator:
    def __init__(self, replication_factor: int = 3):
        self.replication_factor = replication_factor
        self.file_to_blocks: dict[str, list[Block]] = {}
        self.block_counter = 0
        self.block_size = 128  # MB
        self.datanodes: dict[str, DataNode] = {}
        self.racks: dict[str, list[str]] = {}  # rack → datanode names

    def add_datanode(self, name: str, rack: str):
        dn = DataNode(name=name, rack=rack)
        self.datanodes[name] = dn
        if rack not in self.racks:
            self.racks[rack] = []
        self.racks[rack].append(name)
        print(f"  DataNode {name} registered on rack {rack}")

    def _select_replicas_rack_aware(self) -> list[str]:
        """Rack Awareness 복제 배치 선택"""
        alive_racks = {
            rack: [dn for dn in nodes
                   if self.datanodes[dn].alive]
            for rack, nodes in self.racks.items()
        }
        alive_racks = {r: dns for r, dns in alive_racks.items() if dns}
        rack_list = list(alive_racks.keys())

        selected = []
        # 첫 번째: 랜덤 DataNode
        first_rack = random.choice(rack_list)
        first_dn = random.choice(alive_racks[first_rack])
        selected.append(first_dn)

        # 두 번째: 다른 랙
        other_racks = [r for r in rack_list if r != first_rack]
        if other_racks:
            second_rack = random.choice(other_racks)
            second_dn = random.choice(alive_racks[second_rack])
            selected.append(second_dn)
        else:
            # 랙이 하나뿐이면 같은 랙의 다른 노드
            candidates = [dn for dn in alive_racks[first_rack]
                         if dn != first_dn]
            if candidates:
                selected.append(random.choice(candidates))

        # 세 번째: 두 번째와 같은 랙의 다른 노드
        if len(selected) >= 2 and len(selected) < self.replication_factor:
            second_dn = selected[1]
            second_rack = self.datanodes[second_dn].rack
            candidates = [dn for dn in alive_racks.get(second_rack, [])
                         if dn != second_dn]
            if candidates:
                selected.append(random.choice(candidates))
            else:
                # 같은 랙에 다른 노드 없으면 아무 노드
                remaining = [dn for dn in
                             [n for nodes in alive_racks.values()
                              for n in nodes]
                             if dn not in selected]
                if remaining:
                    selected.append(random.choice(remaining))

        return selected[:self.replication_factor]

    def write_file(self, path: str, size_mb: int):
        """파일 쓰기: 블록 분할 → 복제 배치"""
        if path in self.file_to_blocks:
            print(f"  File {path} already exists!")
            return

        num_blocks = max(1, (size_mb + self.block_size - 1) // self.block_size)
        blocks = []
        print(f"\n  Writing {path} ({size_mb}MB) → {num_blocks} block(s)")

        for i in range(num_blocks):
            self.block_counter += 1
            block_sz = min(self.block_size, size_mb - i * self.block_size)
            replicas = self._select_replicas_rack_aware()
            blk = Block(block_id=self.block_counter,
                       size=block_sz, replicas=replicas)
            blocks.append(blk)
            for dn_name in replicas:
                self.datanodes[dn_name].blocks.add(self.block_counter)
            racks_used = [self.datanodes[dn].rack for dn in replicas]
            print(f"    Block {blk.block_id} ({block_sz}MB): "
                  f"{replicas} (racks: {racks_used})")

        self.file_to_blocks[path] = blocks
        print(f"  File {path} written successfully.")

    def read_file(self, path: str, client_rack: Optional[str] = None):
        """파일 읽기: 가장 가까운 복제본 선택"""
        if path not in self.file_to_blocks:
            print(f"  File {path} not found!")
            return

        blocks = self.file_to_blocks[path]
        print(f"\n  Reading {path} ({len(blocks)} block(s))")
        for blk in blocks:
            alive_replicas = [dn for dn in blk.replicas
                             if self.datanodes[dn].alive]
            if not alive_replicas:
                print(f"    Block {blk.block_id}: NO AVAILABLE REPLICA!")
                continue
            # 클라이언트와 같은 랙의 DataNode 우선
            if client_rack:
                same_rack = [dn for dn in alive_replicas
                            if self.datanodes[dn].rack == client_rack]
                chosen = same_rack[0] if same_rack else alive_replicas[0]
            else:
                chosen = alive_replicas[0]
            print(f"    Block {blk.block_id} → read from {chosen} "
                  f"(rack: {self.datanodes[chosen].rack})")

    def simulate_datanode_failure(self, dn_name: str):
        """DataNode 장애 시뮬레이션 및 자동 재복제"""
        if dn_name not in self.datanodes:
            return
        print(f"\n  [FAILURE] DataNode {dn_name} is DOWN!")
        self.datanodes[dn_name].alive = False

        # 영향 받은 블록 탐색
        affected_blocks = self.datanodes[dn_name].blocks.copy()
        for path, blocks in self.file_to_blocks.items():
            for blk in blocks:
                if blk.block_id in affected_blocks:
                    blk.replicas = [r for r in blk.replicas if r != dn_name]
                    if len(blk.replicas) < self.replication_factor:
                        # 재복제 시도
                        new_replicas = self._select_replicas_rack_aware()
                        needed = self.replication_factor - len(blk.replicas)
                        added = []
                        for nr in new_replicas:
                            if nr not in blk.replicas and len(added) < needed:
                                added.append(nr)
                                blk.replicas.append(nr)
                                self.datanodes[nr].blocks.add(blk.block_id)
                        if added:
                            print(f"  [RE-REPLICATE] Block {blk.block_id} "
                                  f"copied to {added}")


# 시뮬레이션
hdfs = HDFSSimulator(replication_factor=3)
hdfs.add_datanode("dn1", rack="rack1")
hdfs.add_datanode("dn2", rack="rack1")
hdfs.add_datanode("dn3", rack="rack2")
hdfs.add_datanode("dn4", rack="rack2")
hdfs.add_datanode("dn5", rack="rack3")

hdfs.write_file("/user/logs/access.log", size_mb=300)
hdfs.read_file("/user/logs/access.log", client_rack="rack1")
hdfs.simulate_datanode_failure("dn3")
```

### 예제 2: NameNode 메타데이터 구조와 CheckPoint 시뮬레이션 (Python)

```python
import json
import time
from copy import deepcopy

class NameNodeSimulator:
    """NameNode FsImage + EditLog + CheckPointing 시뮬레이션"""

    def __init__(self):
        # FsImage: 파일 시스템 스냅샷
        self.fs_image: dict = {
            "root": {"type": "dir", "children": {}, "permissions": "755"}
        }
        # EditLog: FsImage 이후 변경사항
        self.edit_log: list[dict] = []
        self.txn_id = 0

    def _log(self, op: str, **kwargs):
        self.txn_id += 1
        entry = {"txn_id": self.txn_id, "op": op,
                 "timestamp": time.time(), **kwargs}
        self.edit_log.append(entry)
        return entry

    def mkdir(self, path: str):
        parts = path.strip("/").split("/")
        node = self.fs_image["root"]
        for part in parts:
            if part not in node["children"]:
                node["children"][part] = {
                    "type": "dir", "children": {}, "permissions": "755"
                }
                self._log("MKDIR", path=path)
            node = node["children"][part]
        print(f"  mkdir {path} → txn_id={self.txn_id}")

    def create_file(self, path: str, blocks: list[dict]):
        parts = path.strip("/").split("/")
        parent = self.fs_image["root"]
        for part in parts[:-1]:
            parent = parent["children"][part]
        filename = parts[-1]
        parent["children"][filename] = {
            "type": "file", "blocks": blocks, "permissions": "644"
        }
        self._log("CREATE", path=path, num_blocks=len(blocks))
        print(f"  create {path} ({len(blocks)} blocks) → txn_id={self.txn_id}")

    def checkpoint(self) -> dict:
        """
        Secondary NameNode의 CheckPointing:
        FsImage + EditLog를 병합하여 새 FsImage 생성
        """
        print(f"\n  [CHECKPOINT] Merging {len(self.edit_log)} edit log entries "
              f"into FsImage...")
        # 실제로는 EditLog를 재적용하지만, 여기서는 현재 상태가 이미 최신
        new_fs_image = deepcopy(self.fs_image)
        applied_txns = self.txn_id
        self.edit_log.clear()
        print(f"  [CHECKPOINT] Done. Applied txn 1~{applied_txns}. "
              f"EditLog cleared.")
        return new_fs_image

    def show_namespace(self, node=None, prefix=""):
        if node is None:
            node = self.fs_image["root"]
            print("\n  --- HDFS Namespace ---")
            print("  /")
        for name, child in node.get("children", {}).items():
            icon = "📁" if child["type"] == "dir" else "📄"
            block_info = ""
            if child["type"] == "file":
                block_info = f" [{len(child['blocks'])} blocks]"
            print(f"  {prefix}├── {icon} {name}{block_info}")
            if child["type"] == "dir":
                self.show_namespace(child, prefix + "│   ")

    def edit_log_summary(self):
        print(f"\n  --- EditLog ({len(self.edit_log)} entries) ---")
        for entry in self.edit_log[-5:]:  # 최근 5개만 표시
            print(f"  txn={entry['txn_id']} op={entry['op']} "
                  f"path={entry.get('path', 'N/A')}")
        if len(self.edit_log) > 5:
            print(f"  ... and {len(self.edit_log) - 5} more entries")


nn = NameNodeSimulator()
nn.mkdir("/user")
nn.mkdir("/user/hadoop")
nn.mkdir("/user/hadoop/logs")
nn.create_file("/user/hadoop/logs/app.log",
               blocks=[{"id": 1, "size": 128}, {"id": 2, "size": 72}])
nn.create_file("/user/hadoop/data.csv",
               blocks=[{"id": 3, "size": 128}])
nn.show_namespace()
nn.edit_log_summary()
nn.checkpoint()
nn.edit_log_summary()
```

## 주의사항과 팁

### NameNode 메모리 = 클러스터 파일 수 상한선

NameNode는 모든 메타데이터를 메모리에 유지한다. 파일 하나당 약 150바이트의 메모리를 소비한다. 32GB 메모리의 NameNode라면 약 2억 개의 블록을 관리할 수 있다. 수백만 개의 소파일을 저장하는 워크로드는 NameNode 메모리를 금방 소진한다. 소파일 문제 해결법으로는 HAR(Hadoop Archive), SequenceFile, 또는 병합 후 저장이 있다.

### HDFS Federation: NameNode 수평 확장

Hadoop 2.x에서 도입된 **HDFS Federation**은 여러 NameNode가 독립적인 네임스페이스를 관리하도록 한다. 각 NameNode는 서로 다른 디렉터리 트리를 담당하고, DataNode는 모든 NameNode와 통신한다. NameNode의 메모리 확장 문제를 해결하지만 운영 복잡성이 증가한다.

### HDFS vs. 오브젝트 스토리지 (S3)

클라우드 환경에서는 HDFS 대신 S3, GCS 같은 오브젝트 스토리지를 사용하는 추세다. Spark, Hive 등은 `s3a://` 프로토콜로 오브젝트 스토리지를 직접 접근한다. HDFS는 컴퓨팅과 스토리지가 함께(co-located) 있어 데이터 지역성(data locality)이 높지만, 클라우드에서는 컴퓨팅과 스토리지를 분리하는 방향으로 진화했다.

## 참고 자료
- [Apache Hadoop 3.3.2 – HDFS Architecture (Official Docs)](https://hadoop.apache.org/docs/r3.3.2/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html)
- [HDFS Architecture Guide (Hadoop 1.2.1)](https://hadoop.apache.org/docs/r1.2.1/hdfs_design.html)
- [Evaluating Fault Tolerance and Scalability in Distributed File Systems: GFS, HDFS, and MinIO (arXiv)](https://arxiv.org/pdf/2502.01981)
- [Understanding HDFS Architecture – Medium](https://medium.com/@mukulranjan/understanding-hdfs-architecture-a58ecb1bf808)
