---
layout: post
title: "소프트웨어 정의 네트워킹(SDN) 심화: 컨트롤 플레인 분리와 OpenFlow 프로토콜의 동작 원리"
date: 2026-10-30
categories: [cs, computer-science]
tags: [sdn, software-defined-networking, openflow, network-architecture, control-plane, data-plane, ryu, onos]
---

## 개념 설명

소프트웨어 정의 네트워킹(Software-Defined Networking, SDN)은 네트워크의 제어 평면(Control Plane)과 데이터 평면(Data Plane)을 물리적으로 분리하는 네트워크 아키텍처 패러다임이다. 전통적인 네트워크 장비(라우터, 스위치)는 패킷 포워딩 로직(데이터 평면)과 라우팅 결정 로직(제어 평면)이 하나의 장비 내에 통합되어 있다. SDN은 이 구조를 해체하여 제어 평면을 중앙화된 소프트웨어 컨트롤러로 분리한다.

### SDN 3계층 아키텍처

```
┌─────────────────────────────────────────────────────┐
│              애플리케이션 계층 (Application Layer)        │
│     부하 분산 | 방화벽 | QoS | 트래픽 엔지니어링          │
└────────────────────┬────────────────────────────────┘
                     │ Northbound API (REST, gRPC)
┌────────────────────▼────────────────────────────────┐
│              제어 계층 (Control Layer)                  │
│    SDN 컨트롤러 (ONOS, OpenDaylight, Ryu)              │
│    글로벌 네트워크 뷰 | 경로 계산 | 플로우 정책            │
└────────────────────┬────────────────────────────────┘
                     │ Southbound API (OpenFlow, NETCONF)
┌────────────────────▼────────────────────────────────┐
│         인프라스트럭처 계층 (Infrastructure Layer)        │
│    OpenFlow 스위치 | 라우터 | 가상 스위치 (OVS)           │
│    플로우 테이블 기반 패킷 포워딩                          │
└─────────────────────────────────────────────────────┘
```

이 세 계층 간의 인터페이스는:
- **사우스바운드 API(Southbound API)**: 컨트롤러와 네트워크 장비 간 인터페이스. OpenFlow가 가장 대표적인 표준이다 (TCP 포트 6653 또는 TLS).
- **노스바운드 API(Northbound API)**: 컨트롤러와 애플리케이션 간 인터페이스. REST API 형태로 제공된다.

### OpenFlow 플로우 테이블 구조

OpenFlow 스위치는 하나 이상의 **플로우 테이블(Flow Table)**을 유지한다. 각 플로우 엔트리는 다음으로 구성된다:

| 필드 | 설명 |
|------|------|
| 매칭 필드 (Match Fields) | 소스/목적지 IP·MAC, 포트, VLAN, 프로토콜 등 |
| 우선순위 (Priority) | 여러 규칙 매칭 시 적용 순서 |
| 카운터 (Counters) | 패킷 수, 바이트 수, 지속 시간 |
| 명령 (Instructions) | Apply-Actions, Write-Actions, Goto-Table 등 |
| 액션 (Actions) | Output, Drop, Modify-Field, Push/Pop VLAN 등 |
| 만료 시간 (Timeouts) | idle_timeout, hard_timeout |

## 왜 필요한가

전통적인 네트워크 관리의 문제점:

1. **벤더 종속(Vendor Lock-in)**: 각 장비 벤더(Cisco, Juniper 등)는 자체 CLI와 독점 프로토콜을 사용하여 관리 복잡도가 높다.
2. **느린 혁신 속도**: 새로운 기능 추가를 위해 장비 펌웨어 업데이트가 필요하고 표준화 과정이 수년씩 걸린다.
3. **분산된 제어 복잡성**: 각 라우터가 개별적으로 라우팅 테이블을 계산하므로, 전역 최적화나 정책 일관성 확보가 어렵다.
4. **클라우드 데이터센터 요구**: 수만 대의 서버를 연결하는 데이터센터에서 네트워크 토폴로지를 동적으로 변경해야 하는 요구가 증가했다.

SDN은 제어 평면 중앙화로 이 문제를 해결한다. 컨트롤러가 전체 토폴로지를 파악하여 최적 경로를 계산하고, 각 스위치에 플로우 규칙을 동적으로 배포할 수 있다.

## 실제 구현 예제

### 예제 1: Ryu 컨트롤러로 구현하는 L2 학습 스위치

OpenFlow 1.3 기반 L2 스위치를 Ryu(파이썬 SDN 컨트롤러)로 구현한다. 실제 OpenFlow 메시지 교환 흐름을 보여준다.

```python
# simple_switch_13.py - Ryu SDN 컨트롤러 기반 L2 스위치
from ryu.base import app_manager
from ryu.controller import ofp_event
from ryu.controller.handler import CONFIG_DISPATCHER, MAIN_DISPATCHER
from ryu.controller.handler import set_ev_cls
from ryu.ofproto import ofproto_v1_3
from ryu.lib.packet import packet, ethernet, ether_types
import logging

class SimpleSwitch13(app_manager.RyuApp):
    OFP_VERSIONS = [ofproto_v1_3.OFP_VERSION]

    def __init__(self, *args, **kwargs):
        super(SimpleSwitch13, self).__init__(*args, **kwargs)
        # dpid → {mac → port} 매핑 (MAC 주소 학습 테이블)
        self.mac_to_port = {}
        self.logger.setLevel(logging.INFO)

    @set_ev_cls(ofp_event.EventOFPSwitchFeatures, CONFIG_DISPATCHER)
    def switch_features_handler(self, ev):
        """
        스위치 연결 시 호출.
        테이블-미스(Table-Miss) 엔트리를 설치한다:
        매칭되지 않는 모든 패킷을 컨트롤러로 전송.
        """
        datapath = ev.msg.datapath
        ofproto = datapath.ofproto
        parser = datapath.ofproto_parser

        match = parser.OFPMatch()
        actions = [parser.OFPActionOutput(
            ofproto.OFPP_CONTROLLER,
            ofproto.OFPCML_NO_BUFFER
        )]
        self.add_flow(datapath, priority=0, match=match, actions=actions)
        self.logger.info(f"Switch {datapath.id} connected. Table-miss flow installed.")

    def add_flow(self, datapath, priority, match, actions,
                 idle_timeout=0, hard_timeout=0, buffer_id=None):
        """OpenFlow FlowMod 메시지로 플로우 테이블 엔트리 추가"""
        ofproto = datapath.ofproto
        parser = datapath.ofproto_parser

        inst = [parser.OFPInstructionActions(
            ofproto.OFPIT_APPLY_ACTIONS, actions
        )]

        kwargs = dict(
            datapath=datapath,
            priority=priority,
            match=match,
            instructions=inst,
            idle_timeout=idle_timeout,
            hard_timeout=hard_timeout,
        )
        if buffer_id is not None:
            kwargs['buffer_id'] = buffer_id

        mod = parser.OFPFlowMod(**kwargs)
        datapath.send_msg(mod)

    @set_ev_cls(ofp_event.EventOFPPacketIn, MAIN_DISPATCHER)
    def packet_in_handler(self, ev):
        """
        Packet-In 이벤트: 스위치가 플로우 테이블에 매칭 규칙이 없어
        패킷을 컨트롤러로 전달한 경우 처리.
        """
        msg = ev.msg
        datapath = msg.datapath
        ofproto = datapath.ofproto
        parser = datapath.ofproto_parser
        in_port = msg.match['in_port']

        pkt = packet.Packet(msg.data)
        eth = pkt.get_protocols(ethernet.ethernet)[0]

        if eth.ethertype == ether_types.ETH_TYPE_LLDP:
            return

        dst_mac = eth.dst
        src_mac = eth.src
        dpid = datapath.id

        self.mac_to_port.setdefault(dpid, {})
        self.mac_to_port[dpid][src_mac] = in_port

        if dst_mac in self.mac_to_port[dpid]:
            out_port = self.mac_to_port[dpid][dst_mac]
        else:
            out_port = ofproto.OFPP_FLOOD

        actions = [parser.OFPActionOutput(out_port)]

        if out_port != ofproto.OFPP_FLOOD:
            match = parser.OFPMatch(
                in_port=in_port,
                eth_dst=dst_mac,
                eth_src=src_mac
            )
            if msg.buffer_id != ofproto.OFP_NO_BUFFER:
                self.add_flow(datapath, priority=1, match=match,
                             actions=actions, idle_timeout=10,
                             buffer_id=msg.buffer_id)
                return
            else:
                self.add_flow(datapath, priority=1, match=match,
                             actions=actions, idle_timeout=10)

        data = None if msg.buffer_id != ofproto.OFP_NO_BUFFER else msg.data
        out = parser.OFPPacketOut(
            datapath=datapath,
            buffer_id=msg.buffer_id,
            in_port=in_port,
            actions=actions,
            data=data
        )
        datapath.send_msg(out)
        self.logger.info(f"SW{dpid}: {src_mac} → {dst_mac} via port {out_port}")

# 실행: ryu-manager simple_switch_13.py
```

### 예제 2: SDN 기반 트래픽 엔지니어링 (경로 최적화)

SDN의 강력한 사용 사례는 트래픽 엔지니어링(TE)이다. 컨트롤러가 전체 토폴로지를 알고 있어 최적 경로를 계산하고 동적으로 재경로화할 수 있다.

```python
import heapq
from collections import defaultdict
from dataclasses import dataclass
from typing import Dict, List, Tuple

@dataclass
class Link:
    src: str
    dst: str
    bandwidth: float   # Gbps
    latency: float     # ms
    utilization: float = 0.0

class SDNController:
    def __init__(self):
        self.links: Dict[Tuple[str, str], Link] = {}
        self.switches = set()
        self.flow_tables: Dict[str, List[dict]] = defaultdict(list)

    def add_link(self, src: str, dst: str, bandwidth: float, latency: float):
        self.links[(src, dst)] = Link(src, dst, bandwidth, latency)
        self.links[(dst, src)] = Link(dst, src, bandwidth, latency)
        self.switches.update([src, dst])

    def compute_path_latency(self, src: str, dst: str) -> List[str]:
        """지연 최소화 경로 (Dijkstra)"""
        dist = {sw: float('inf') for sw in self.switches}
        dist[src] = 0
        prev = {sw: None for sw in self.switches}
        heap = [(0.0, src)]

        while heap:
            d, u = heapq.heappop(heap)
            if d > dist[u]:
                continue
            for (a, b), link in self.links.items():
                if a != u:
                    continue
                cost = dist[u] + link.latency
                if cost < dist[b]:
                    dist[b] = cost
                    prev[b] = u
                    heapq.heappush(heap, (cost, b))

        return self._reconstruct_path(prev, src, dst)

    def compute_path_bandwidth(self, src: str, dst: str) -> List[str]:
        """가용 대역폭 최대화 경로 (역수를 비용으로)"""
        dist = {sw: float('inf') for sw in self.switches}
        dist[src] = 0
        prev = {sw: None for sw in self.switches}
        heap = [(0.0, src)]

        while heap:
            d, u = heapq.heappop(heap)
            if d > dist[u]:
                continue
            for (a, b), link in self.links.items():
                if a != u:
                    continue
                avail_bw = link.bandwidth * (1.0 - link.utilization)
                if avail_bw < 0.01:
                    continue
                cost = dist[u] + (1.0 / avail_bw)
                if cost < dist[b]:
                    dist[b] = cost
                    prev[b] = u
                    heapq.heappush(heap, (cost, b))

        return self._reconstruct_path(prev, src, dst)

    def _reconstruct_path(self, prev, src, dst) -> List[str]:
        path = []
        cur = dst
        while cur is not None:
            path.append(cur)
            cur = prev[cur]
        return path[::-1] if path and path[-1] == src else []

    def install_path_flows(self, path: List[str], dst_ip: str):
        """경로의 각 스위치에 플로우 규칙 설치"""
        for i, sw in enumerate(path):
            if i < len(path) - 1:
                action = f"forward_to_{path[i+1]}"
            else:
                action = "deliver_to_host"
            entry = {"match": f"ip_dst={dst_ip}", "action": action, "priority": 10}
            self.flow_tables[sw].append(entry)
            print(f"  [FlowMod] {sw}: {entry['match']} → {action}")

    def reroute_on_congestion(self, congested_link: Tuple[str, str], dst_ip: str, src: str, dst: str):
        """링크 혼잡 시 대체 경로로 재경로화"""
        a, b = congested_link
        if (a, b) in self.links:
            self.links[(a, b)].utilization = 1.0
            self.links[(b, a)].utilization = 1.0
        new_path = self.compute_path_bandwidth(src, dst)
        print(f"\n[Reroute] 혼잡 감지 {a}→{b}, 새 경로: {' → '.join(new_path)}")
        self.flow_tables = defaultdict(list)
        self.install_path_flows(new_path, dst_ip)

if __name__ == "__main__":
    ctrl = SDNController()
    ctrl.add_link("SW1", "SW2", bandwidth=10, latency=1)
    ctrl.add_link("SW2", "SW4", bandwidth=10, latency=1)
    ctrl.add_link("SW1", "SW3", bandwidth=10, latency=2)
    ctrl.add_link("SW3", "SW4", bandwidth=10, latency=2)
    ctrl.add_link("SW1", "SW4", bandwidth=1,  latency=10)

    path = ctrl.compute_path_latency("SW1", "SW4")
    print(f"최저 지연 경로: {' → '.join(path)}")
    ctrl.install_path_flows(path, dst_ip="192.168.1.100")
    ctrl.reroute_on_congestion(("SW1", "SW2"), "192.168.1.100", "SW1", "SW4")
```

## 주의사항 및 팁

### 1. 컨트롤러 고가용성(HA)

SDN에서 컨트롤러는 단일 장애 지점(Single Point of Failure)이 될 수 있다. 대규모 배포에서는 컨트롤러 클러스터링이 필수적이다.

- **ONOS**: AT&T·Google 등이 사용하는 분산 SDN 컨트롤러. Apache Karaf 기반, Raft 합의로 일관성 보장.
- **OpenDaylight**: 리눅스 재단의 오픈소스 SDN 플랫폼. 마이크로서비스 아키텍처.
- **Ryu**: Python 기반 경량 컨트롤러. 연구·실습에 적합.

### 2. Packet-In 스톰(Storm) 방지

처음 연결된 호스트의 패킷은 모두 컨트롤러로 전달된다. 수천 개의 플로우가 동시에 시작되면 Packet-In 이벤트가 컨트롤러를 과부하에 빠뜨린다. 완화 방법:

- **프로액티브 플로우 설치**: 컨트롤러가 미리 플로우 규칙을 배포.
- **리액티브 + 프로액티브 혼합**: 기본 트래픽은 프로액티브, 예외는 리액티브.
- **Rate Limiting**: Packet-In 이벤트 속도 제한.

### 3. P4 언어의 등장

OpenFlow의 한계(고정된 헤더 파싱 로직)를 극복하기 위해 **P4(Programming Protocol-independent Packet Processors)**가 등장했다. P4는 데이터 평면 프로그래밍 언어로, 패킷 파싱·처리 로직을 유연하게 정의하여 커스텀 프로토콜도 처리 가능하다.

### 4. SDN의 실제 적용 사례

- **Google B4**: 구글 내부 WAN을 SDN으로 운영. OpenFlow 기반 트래픽 엔지니어링으로 링크 활용률을 거의 100%까지 향상.
- **Facebook Fabric**: 데이터센터 네트워크를 SDN으로 통합 관리.
- **AT&T ECOMP**: 통신 사업자 네트워크 SDN/NFV 플랫폼.

### 5. SDN과 NFV의 관계

SDN(소프트웨어 정의 네트워크)은 제어 평면 소프트웨어화를 다루고, NFV(Network Function Virtualization)는 방화벽·로드밸런서 등 네트워크 기능의 가상화를 다룬다. 두 기술은 상호 보완적이며 클라우드 네이티브 네트워킹의 핵심이다.

## 참고 자료
- [Software Defined Networking - Architecture and role of OpenFlow](https://www.howtoforge.com/software-defined-networking-sdn-architecture-and-role-of-openflow)
- [SDN: State of the Art and Research Challenges (arXiv)](https://arxiv.org/pdf/1406.0124)
- [IEEE TechNav: Software Defined Networks](https://technav.ieee.org/area/software-defined-networks/)
- [Open Networking Foundation SDN Definition](https://opennetworking.org/sdn-definition/)
