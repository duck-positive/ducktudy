---
layout: post
title: "VXLAN 오버레이 네트워크: 클라우드 네이티브 가상 네트워킹의 내부 구조"
date: 2026-10-28
categories: [cs, computer-science]
tags: [VXLAN, overlay-network, networking, cloud, kubernetes, SDN, encapsulation, VTEP, VNI]
---

쿠버네티스 클러스터가 수백 대의 노드에 걸쳐 있을 때, 각 파드(Pod)는 마치 같은 L2 네트워크에 있는 것처럼 서로 통신한다. 물리적으로는 전혀 다른 랙, 다른 스위치, 심지어 다른 데이터센터에 있어도 말이다. 이것을 가능하게 하는 핵심 기술이 **VXLAN(Virtual eXtensible LAN)**이다. 2014년 RFC 7348로 표준화된 VXLAN은 오늘날 AWS VPC, GCP VPC, Azure VNet, Kubernetes CNI 플러그인(Flannel, Cilium, Calico 등)의 근간이 된다.

## 왜 오버레이 네트워크가 필요한가

전통적인 데이터센터 네트워크에서 VM이나 컨테이너는 VLAN(802.1Q)으로 격리되었다. 그러나 VLAN에는 근본적인 한계가 있다:

- **4096개 제한:** VLAN ID는 12비트(0~4095). 수천 개의 테넌트를 지원하는 퍼블릭 클라우드에는 턱없이 부족하다.
- **L2 확장 불가:** VLAN은 L2 도메인이므로, 물리적으로 분리된 네트워크 사이에 VLAN을 연장하려면 복잡한 DCI(Data Center Interconnect)가 필요하다.
- **스패닝 트리 병목:** 대규모 L2 네트워크는 STP(Spanning Tree Protocol)로 인한 링크 낭비와 수렴 지연이 발생한다.

VXLAN은 이 문제를 우아하게 해결한다: **L2 프레임을 L3 UDP 패킷 안에 터널링한다.** L3 언더레이는 ECMP(Equal-Cost Multi-Path)로 모든 경로를 활용하고, L2 오버레이는 가상 네트워크 세그먼트를 수천만 개까지 지원한다.

## VXLAN의 핵심 개념

### VNI (VXLAN Network Identifier)

VNI는 24비트 식별자로, **약 1600만 개**의 가상 네트워크 세그먼트를 표현한다. VLAN의 4096개에 비해 4000배 이상 많다. 각 VNI는 독립된 L2 도메인이다. VNI 100의 VM과 VNI 200의 VM은 같은 IP 대역을 사용해도 서로 통신할 수 없다.

### VTEP (VXLAN Tunnel Endpoint)

VTEP는 VXLAN 터널의 시작점이자 끝점이다. VM이나 컨테이너 쪽을 바라보는 내부 인터페이스(L2)와, 물리 네트워크를 바라보는 외부 인터페이스(L3 UDP)를 모두 갖는다. 리눅스 커널은 소프트웨어 VTEP를 기본으로 지원하며(`ip link add vxlan0 type vxlan`), 고성능 환경에서는 스마트NIC나 ASIC이 하드웨어 VTEP 역할을 한다.

### VXLAN 패킷 구조

VXLAN 캡슐화는 다음 계층으로 구성된다:

```
[ 외부 이더넷 헤더 ]  ← 물리 네트워크 MAC (VTEP 간)
[ 외부 IP 헤더    ]  ← 물리 네트워크 IP (VTEP 간)
[ 외부 UDP 헤더   ]  ← 목적지 포트 4789 (IANA 표준)
[ VXLAN 헤더 8B  ]  ← Flags(8b) + 예약(24b) + VNI(24b) + 예약(8b)
[ 내부 이더넷 프레임 ]  ← 원본 VM/컨테이너 트래픽
```

VXLAN 헤더의 핵심은 24비트 VNI 필드다. 수신 VTEP는 VNI를 보고 이 패킷을 어떤 가상 네트워크로 전달할지 결정한다.

오버헤드는 **50바이트**(외부 이더넷 14B + IP 20B + UDP 8B + VXLAN 8B)다. 1500바이트 MTU 환경에서 실제 페이로드는 1450바이트로 줄어든다. 따라서 VXLAN을 사용하는 네트워크에서는 점보 프레임(9000 MTU)을 권장한다.

## 리눅스에서 VXLAN 직접 구성하기

두 서버(192.168.1.10, 192.168.1.20) 사이에 VNI 100번 가상 네트워크를 구성하는 예시:

```bash
# ─── 서버 A (192.168.1.10) ───

# VXLAN 인터페이스 생성
# dstport 4789: IANA 표준 포트
# nolearning: MAC 학습 비활성화 (정적 FDB 또는 컨트롤 플레인 사용)
ip link add vxlan100 type vxlan \
    id 100 \
    dstport 4789 \
    local 192.168.1.10 \
    nolearning

# IP 주소 할당 (오버레이 주소)
ip addr add 10.0.100.1/24 dev vxlan100
ip link set vxlan100 up

# 서버 B의 VTEP를 FDB(Forwarding DataBase)에 추가
# ff:ff:ff:ff:ff:ff = 브로드캐스트 → 서버 B VTEP로 플러딩
bridge fdb append ff:ff:ff:ff:ff:ff dev vxlan100 dst 192.168.1.20 via eth0

# 서버 B의 오버레이 MAC 주소를 알고 있으면 유니캐스트로 추가
# (실제 환경에서는 EVPN 등 컨트롤 플레인이 이를 자동화)
# bridge fdb add <서버B_오버레이_MAC> dev vxlan100 dst 192.168.1.20

# ─── 서버 B (192.168.1.20) ───
ip link add vxlan100 type vxlan \
    id 100 \
    dstport 4789 \
    local 192.168.1.20 \
    nolearning

ip addr add 10.0.100.2/24 dev vxlan100
ip link set vxlan100 up

bridge fdb append ff:ff:ff:ff:ff:ff dev vxlan100 dst 192.168.1.10 via eth0

# 검증: 서버 A에서 서버 B 오버레이 IP로 핑
# ping 10.0.100.2
# tcpdump -i eth0 'udp port 4789' -nn   ← 물리 인터페이스에서 캡슐화 확인

# VXLAN 패킷 디코딩 확인 (Wireshark 또는 tcpdump)
# tcpdump -i eth0 -v 'udp port 4789' 출력:
# 192.168.1.10.12345 > 192.168.1.20.4789: VXLAN, flags [I] (0x08), vni 100
#   IP 10.0.100.1 > 10.0.100.2: ICMP echo request
```

## Kubernetes에서 VXLAN: Flannel의 동작 원리

Kubernetes의 대표적인 CNI 플러그인인 Flannel은 VXLAN 백엔드를 기본으로 사용한다. 내부 동작을 Python으로 시뮬레이션하면:

```python
import socket
import struct
import threading
from dataclasses import dataclass
from typing import Dict, Optional

@dataclass
class VTEPEntry:
    """VTEP 테이블 엔트리: 오버레이 서브넷 → 물리 IP 매핑"""
    overlay_subnet: str  # e.g., "10.244.1.0/24"
    vtep_ip: str         # e.g., "192.168.1.20"
    vtep_mac: str        # e.g., "52:54:00:ab:cd:ef"


class VXLANController:
    """
    간단한 VXLAN 컨트롤 플레인 시뮬레이터
    Flannel의 etcd 기반 라우팅 테이블과 유사한 구조
    """

    VXLAN_PORT = 4789
    VXLAN_VNI = 1  # Flannel 기본 VNI

    def __init__(self, local_ip: str, overlay_subnet: str):
        self.local_ip = local_ip
        self.overlay_subnet = overlay_subnet
        self.vtep_table: Dict[str, VTEPEntry] = {}
        self._lock = threading.Lock()

    def register_node(self, entry: VTEPEntry):
        """새 노드의 VTEP 정보를 테이블에 등록 (etcd watch 시뮬레이션)"""
        with self._lock:
            self.vtep_table[entry.overlay_subnet] = entry
            print(f"[컨트롤러] 노드 등록: {entry.overlay_subnet} → {entry.vtep_ip}")

    def find_vtep(self, dest_ip: str) -> Optional[VTEPEntry]:
        """목적지 IP가 어느 VTEP에 속하는지 LPM(Longest Prefix Match)으로 검색"""
        import ipaddress
        dest = ipaddress.ip_address(dest_ip)
        best_match = None
        best_prefix_len = -1

        with self._lock:
            for subnet, entry in self.vtep_table.items():
                network = ipaddress.ip_network(subnet)
                if dest in network:
                    if network.prefixlen > best_prefix_len:
                        best_prefix_len = network.prefixlen
                        best_match = entry

        return best_match

    def encapsulate(self, inner_payload: bytes, src_ip: str, dst_ip: str) -> bytes:
        """
        VXLAN 캡슐화 (실제 패킷 구성 시뮬레이션)
        
        반환: VXLAN 헤더 + 내부 페이로드 (UDP/IP 헤더는 OS가 처리)
        """
        # VXLAN 헤더 (8바이트): I 플래그 + 예약 + VNI + 예약
        # I 플래그(0x08): VNI 유효
        vxlan_flags = 0x08000000  # Flags[7:4] = 0b1000, 나머지 예약
        vni_field = (self.VXLAN_VNI << 8)  # VNI는 상위 24비트
        vxlan_header = struct.pack('>I', vxlan_flags) + struct.pack('>I', vni_field)

        encapsulated = vxlan_header + inner_payload

        vtep = self.find_vtep(dst_ip)
        if vtep:
            print(f"[데이터플레인] 캡슐화: {src_ip} → {dst_ip}")
            print(f"  VTEP 터널: {self.local_ip} → {vtep.vtep_ip}:4789")
            print(f"  VNI: {self.VXLAN_VNI}, 페이로드: {len(inner_payload)}B")
            print(f"  총 오버헤드: {len(vxlan_header) + 8 + 20 + 14}B "
                  f"(VXLAN 8B + UDP 8B + IP 20B + Ethernet 14B)")
        else:
            print(f"[데이터플레인] VTEP 없음 → 패킷 드롭: {dst_ip}")

        return encapsulated

    def decapsulate(self, packet: bytes) -> Optional[bytes]:
        """VXLAN 패킷 역캡슐화"""
        if len(packet) < 8:
            return None

        flags_reserved = struct.unpack('>I', packet[:4])[0]
        i_flag = (flags_reserved >> 24) & 0x08

        if not i_flag:
            print("[데이터플레인] VNI 유효 플래그 없음 — 무시")
            return None

        vni_field = struct.unpack('>I', packet[4:8])[0]
        vni = vni_field >> 8

        if vni != self.VXLAN_VNI:
            print(f"[데이터플레인] VNI 불일치: {vni} ≠ {self.VXLAN_VNI} — 무시")
            return None

        inner_payload = packet[8:]
        print(f"[데이터플레인] 역캡슐화: VNI={vni}, 내부 페이로드 {len(inner_payload)}B")
        return inner_payload


# 시뮬레이션
controller_a = VXLANController("192.168.1.10", "10.244.0.0/24")
controller_b = VXLANController("192.168.1.20", "10.244.1.0/24")

# 노드 A가 노드 B의 VTEP 정보를 학습
controller_a.register_node(VTEPEntry(
    overlay_subnet="10.244.1.0/24",
    vtep_ip="192.168.1.20",
    vtep_mac="52:54:00:ab:cd:ef"
))

# 파드 A(10.244.0.5)에서 파드 B(10.244.1.7)로 패킷 전송
fake_inner_frame = b'\x00' * 64  # 내부 이더넷 프레임 (더미)
packet = controller_a.encapsulate(fake_inner_frame, "10.244.0.5", "10.244.1.7")
inner = controller_b.decapsulate(packet)
```

## EVPN: VXLAN의 컨트롤 플레인

VXLAN은 데이터 플레인 프로토콜이다. MAC 주소 학습을 어떻게 할 것인지(컨트롤 플레인)는 별도다. 초기에는 멀티캐스트나 Flood-and-Learn을 사용했지만, 대규모 환경에서는 **EVPN(Ethernet VPN, RFC 7432)**을 BGP 위에서 컨트롤 플레인으로 사용한다.

EVPN은 각 VTEP가 로컬에서 학습한 MAC/IP 정보를 BGP로 전파한다. 덕분에:
- 브로드캐스트 플러딩 최소화
- ARP 프록시로 L2 트래픽 감소
- 서브넷 간 라우팅(IRB, Integrated Routing and Bridging) 지원

Kubernetes 환경에서는 Cilium이 VXLAN + eBPF, Calico가 BGP + IPIP/VXLAN, Flannel이 VXLAN/WireGuard를 선택적으로 사용한다.

## 주의사항과 실무 팁

**1. MTU를 반드시 확인하라.** VXLAN 오버헤드 50바이트 때문에 물리 MTU 1500 환경에서는 VM/컨테이너 MTU를 1450으로 낮춰야 한다. MTU 불일치는 대용량 패킷 유실이라는 간헐적이고 진단하기 어려운 문제를 일으킨다. 가능하면 점보 프레임(9000 MTU) 환경을 구성하라.

**2. UDP 흐름 기반 해싱.** VXLAN은 UDP 위에서 동작하므로, ECMP 로드 밸런싱이 UDP 5-tuple(src IP, dst IP, proto, src port, dst port) 기반으로 된다. 발신 UDP 소스 포트를 내부 패킷의 5-tuple로 해싱하면 ECMP 분산 효율이 높아진다. 리눅스 커널은 이를 기본으로 지원한다.

**3. 암호화가 필요하면 WireGuard.** VXLAN은 기본적으로 암호화를 제공하지 않는다. Kubernetes Flannel의 WireGuard 백엔드나 Cilium의 WireGuard 모드처럼, VXLAN 터널에 암호화가 필요한 경우 WireGuard를 오버레이 위에 올리는 것이 일반적이다.

**4. conntrack 오버헤드.** 리눅스의 conntrack이 VXLAN 캡슐화된 UDP 패킷을 추적하면 CPU 부하가 증가할 수 있다. Cilium 같은 eBPF 기반 CNI는 conntrack을 우회해 이 오버헤드를 줄인다.

VXLAN은 현대 클라우드 인프라의 보이지 않는 기반이다. 쿠버네티스 파드끼리, 서로 다른 AZ의 VM끼리 "같은 L2에 있는 것처럼" 통신하는 마법 뒤에는 이 단순하지만 강력한 캡슐화 기술이 있다.

## 참고 자료
- [RFC 7348: VXLAN - A Framework for Overlaying Virtualized Layer 2 Networks over Layer 3 Networks](https://datatracker.ietf.org/doc/rfc7348)
- [IETF NVO3 Working Group: RFC 7348bis Draft](https://datatracker.ietf.org/doc/draft-ietf-nvo3-rfc7348bis/)
- [Linux Kernel VXLAN Documentation](https://www.kernel.org/doc/html/latest/networking/vxlan.html)
- [Flannel CNI: VXLAN Backend](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md)
