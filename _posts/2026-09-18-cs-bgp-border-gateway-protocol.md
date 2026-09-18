---
layout: post
title: "BGP 라우팅 프로토콜 완전 정복: 인터넷을 연결하는 경로 벡터 프로토콜의 내부 동작"
date: 2026-09-18
categories: [cs, computer-science]
tags: [BGP, routing, networking, autonomous-system, internet, path-vector, eBGP, iBGP, RPKI, network-security]
---

인터넷은 하나의 단일 네트워크가 아닙니다. 전 세계에 흩어진 수만 개의 독립적인 네트워크 조각들이 서로 연결된 **네트워크의 네트워크**입니다. 이 거대한 퍼즐 조각들이 서로 데이터를 주고받을 수 있도록 하는 핵심 프로토콜이 바로 **BGP(Border Gateway Protocol)**입니다. BGP는 인터넷에서 데이터 패킷의 경로를 결정하는 **유일한 EGP(Exterior Gateway Protocol)**로, 현재 인터넷의 라우팅 인프라 전체를 사실상 혼자 떠받치고 있습니다.

## 자율 시스템 (Autonomous System, AS)이란?

인터넷을 이해하려면 먼저 **자율 시스템(AS)**을 알아야 합니다. AS는 단일 관리 도메인 하에 있는 IP 네트워크의 집합으로, 고유한 **AS 번호(ASN)**를 부여받습니다.

- **KT**: AS4766
- **SK브로드밴드**: AS9318
- **Google**: AS15169
- **Cloudflare**: AS13335
- **Amazon AWS**: AS16509

AS 내부의 라우팅은 **IGP(Interior Gateway Protocol)** — OSPF, IS-IS, RIP 등 — 이 담당합니다. AS 간의 라우팅은 **EGP**인 BGP가 담당합니다.

인터넷은 크게 세 종류의 AS로 구성됩니다:

- **Tier 1**: 전 세계 어디에도 비용을 내지 않고 연결되는 대형 통신사 (AT&T, Lumen, NTT 등)
- **Tier 2**: Tier 1에게 비용을 내고, 소규모 AS에게 트랜짓을 제공 (KT, SK, LG U+ 등)
- **Tier 3**: 인터넷 접속만 구매하는 최종 소비자 AS (일반 기업, 호스팅 업체 등)

---

## BGP의 기본 동작 원리

BGP는 **경로 벡터 프로토콜(Path Vector Protocol)**입니다. 각 경로 광고에는 목적지 IP 대역(Prefix)과 그 경로를 거쳐온 AS 번호들의 목록(**AS_PATH**)이 포함됩니다.

### BGP 세션 수립 과정

BGP는 **TCP 포트 179** 위에서 동작합니다. 두 BGP 라우터가 연결되면 다음 상태를 거칩니다:

```
Idle → Connect → Active → OpenSent → OpenConfirm → Established
```

1. **Idle**: 초기 상태, 연결 대기
2. **Connect**: TCP 연결 시도
3. **Active**: TCP 연결 실패, 재시도
4. **OpenSent**: OPEN 메시지 전송 완료
5. **OpenConfirm**: OPEN 메시지 수신, KEEPALIVE 대기
6. **Established**: BGP 세션 확립, 라우팅 정보 교환 시작

BGP 메시지 타입:

| 메시지 | 용도 |
|--------|------|
| **OPEN** | 세션 수립 시 AS 번호, 버전, Hold Time 등 협상 |
| **UPDATE** | 새 경로 광고 또는 기존 경로 철회 |
| **KEEPALIVE** | 세션 유지 (기본 60초마다) |
| **NOTIFICATION** | 오류 보고 및 세션 종료 |

---

## eBGP vs iBGP

### eBGP (External BGP)

서로 다른 AS 간의 BGP 세션입니다. 일반적으로 직접 연결된(인접한) 라우터 간에 설정합니다. eBGP로 수신한 경로는 다른 eBGP 피어에게 전달됩니다.

### iBGP (Internal BGP)

같은 AS 내부의 BGP 세션입니다. iBGP의 핵심 규칙: **iBGP로 수신한 경로는 다른 iBGP 피어에게 전달하지 않습니다** (루프 방지). 이 때문에 AS 내 모든 BGP 라우터가 서로 연결된 **풀 메시(Full Mesh)** 토폴로지가 필요하거나, **Route Reflector** 또는 **BGP Confederation**을 사용해야 합니다.

```
eBGP: AS간 연결, 직접 연결 라우터
iBGP: AS내 연결, 풀 메시 또는 Route Reflector
```

---

## BGP 경로 속성 (Path Attributes)

BGP는 단순히 최단 경로를 선택하지 않습니다. 여러 경로 속성을 순서대로 평가하여 최선의 경로를 선택합니다.

### 주요 Path Attributes

| 속성 | 타입 | 설명 |
|------|------|------|
| **AS_PATH** | 필수 | 경로를 거쳐온 AS 번호 목록 (루프 방지 및 경로 선택) |
| **NEXT_HOP** | 필수 | 다음 홉 IP 주소 |
| **LOCAL_PREF** | 선택적 | AS 내부 선호도 (높을수록 선호) |
| **MED (MULTI_EXIT_DISC)** | 선택적 | AS 간 진입점 선호도 (낮을수록 선호) |
| **ORIGIN** | 필수 | 경로 유래 (IGP < EGP < INCOMPLETE) |
| **COMMUNITY** | 선택적 | 경로 태깅 (정책 적용에 활용) |

### BGP 경로 선택 알고리즘 (Best Path Selection)

BGP는 동일 목적지에 여러 경로가 있을 때 아래 순서로 평가합니다:

```
1. Weight (Cisco 전용, 높을수록 선호)
2. LOCAL_PREF (높을수록 선호)
3. 로컬 AS에서 생성된 경로 선호
4. AS_PATH 길이 (짧을수록 선호)
5. ORIGIN 타입 (IGP < EGP < INCOMPLETE)
6. MED (낮을수록 선호)
7. eBGP > iBGP
8. IGP 메트릭이 낮은 NEXT_HOP 선호
9. 오래된 eBGP 경로 선호 (안정성)
10. BGP Router ID가 낮은 피어 선호
11. 클러스터 목록이 짧은 경로 선호
12. 피어의 IP 주소가 낮은 경로 선호
```

---

## 코드 예제 1: FRRouting(FRR)을 사용한 BGP 설정

FRRouting은 오픈소스 IP 라우팅 소프트웨어입니다. 실제 서버에서 BGP를 구성하는 방법입니다.

```bash
# FRRouting BGP 설정 (vtysh 인터페이스)

# 라우터 A (AS 65001)
router bgp 65001
  bgp router-id 10.0.0.1
  
  # eBGP 피어 설정 (AS 65002와 연결)
  neighbor 192.168.1.2 remote-as 65002
  neighbor 192.168.1.2 description "Peer with AS65002"
  
  # iBGP 피어 설정 (같은 AS 내 라우터 B)
  neighbor 10.0.0.2 remote-as 65001
  neighbor 10.0.0.2 update-source lo  # 루프백 인터페이스 사용
  
  # IPv4 유니캐스트 설정
  address-family ipv4 unicast
    # 자신의 네트워크 광고
    network 203.0.113.0/24
    network 203.0.114.0/24
    
    # iBGP 피어에게 Next-Hop 변경
    neighbor 10.0.0.2 next-hop-self
    
    # eBGP 피어에게 경로 정책 적용
    neighbor 192.168.1.2 route-map IMPORT_POLICY in
    neighbor 192.168.1.2 route-map EXPORT_POLICY out
  exit-address-family
!
# Route-Map으로 경로 정책 설정
route-map IMPORT_POLICY permit 10
  set local-preference 200    # 이 피어 경로를 선호
!
route-map EXPORT_POLICY permit 10
  match ip address prefix-list MY_PREFIXES
!
ip prefix-list MY_PREFIXES seq 5 permit 203.0.113.0/24
ip prefix-list MY_PREFIXES seq 10 permit 203.0.114.0/24
```

---

## 코드 예제 2: Python으로 BGP 경로 선택 시뮬레이션

```python
from dataclasses import dataclass, field
from typing import List, Optional
import ipaddress


@dataclass
class BGPRoute:
    prefix: str
    as_path: List[int]          # AS_PATH 목록
    next_hop: str
    local_pref: int = 100       # 기본값 100
    med: int = 0
    origin: str = 'IGP'         # IGP, EGP, INCOMPLETE
    weight: int = 0             # Cisco 전용 (시뮬레이션용)
    is_ebgp: bool = True

    @property
    def as_path_length(self) -> int:
        return len(self.as_path)

    @property
    def origin_value(self) -> int:
        return {'IGP': 0, 'EGP': 1, 'INCOMPLETE': 2}.get(self.origin, 2)


def select_best_path(routes: List[BGPRoute]) -> Optional[BGPRoute]:
    """BGP 최선 경로 선택 알고리즘 시뮬레이션"""
    if not routes:
        return None

    def sort_key(r: BGPRoute):
        return (
            -r.weight,           # 1. Weight 높을수록 선호 (역순)
            -r.local_pref,       # 2. LOCAL_PREF 높을수록 선호 (역순)
            r.as_path_length,    # 4. AS_PATH 짧을수록 선호
            r.origin_value,      # 5. Origin: IGP < EGP < INCOMPLETE
            r.med,               # 6. MED 낮을수록 선호
            0 if r.is_ebgp else 1,  # 7. eBGP 선호
        )

    sorted_routes = sorted(routes, key=sort_key)
    return sorted_routes[0]


def simulate_as_path_loop_detection(as_path: List[int], local_asn: int) -> bool:
    """AS_PATH에 자신의 ASN이 포함되어 있으면 루프 → 거절"""
    return local_asn in as_path


# 동일 목적지(192.168.10.0/24)에 대한 여러 경로 시뮬레이션
routes = [
    BGPRoute(
        prefix="192.168.10.0/24",
        as_path=[65002, 65003],
        next_hop="10.0.0.1",
        local_pref=150,   # 높은 LOCAL_PREF
        med=100,
        origin='IGP',
        is_ebgp=True,
    ),
    BGPRoute(
        prefix="192.168.10.0/24",
        as_path=[65004, 65005, 65003],
        next_hop="10.0.0.2",
        local_pref=100,
        med=50,
        origin='IGP',
        is_ebgp=True,
    ),
    BGPRoute(
        prefix="192.168.10.0/24",
        as_path=[65006],
        next_hop="10.0.0.3",
        local_pref=100,
        med=200,
        origin='INCOMPLETE',
        is_ebgp=False,    # iBGP 경로
    ),
]

print("=== BGP 경로 선택 시뮬레이션 ===")
for i, route in enumerate(routes, 1):
    print(f"경로 {i}: AS_PATH={route.as_path}, LOCAL_PREF={route.local_pref}, "
          f"MED={route.med}, Origin={route.origin}, eBGP={route.is_ebgp}")

best = select_best_path(routes)
print(f"\n최선 경로 선택: NEXT_HOP={best.next_hop}, AS_PATH={best.as_path}")
print(f"선택 이유: LOCAL_PREF={best.local_pref} (가장 높음)")

# 루프 감지
local_asn = 65001
test_path = [65002, 65001, 65003]  # 자신의 ASN 포함
print(f"\nAS_PATH {test_path}에서 루프 감지: "
      f"{'거절' if simulate_as_path_loop_detection(test_path, local_asn) else '허용'}")
```

---

## BGP 보안: BGP Hijacking과 RPKI

### BGP Hijacking

BGP는 기본적으로 **신뢰 기반 프로토콜**입니다. 어떤 AS든 임의의 IP 대역을 광고할 수 있습니다. 이를 악용한 공격이 **BGP Hijacking**입니다.

**2008년 Pakistan Telecom 사건**: AS17557(Pakistan Telecom)이 YouTube(AS36561)의 IP 대역 `208.65.153.0/24`를 더 구체적인 `/25`로 광고하면서 전 세계 YouTube 트래픽이 약 2시간 동안 파키스탄으로 빨려 들어갔습니다.

**2010년 중국 China Telecom 사건**: 약 15분간 미국 정부 기관을 포함한 수만 개의 IP 대역이 중국으로 우회됐습니다.

### RPKI (Resource Public Key Infrastructure)

RPKI는 BGP Hijacking을 방어하는 현대적 솔루션입니다. IP 대역의 합법적인 소유자가 어느 AS에서 광고해야 하는지를 **ROA(Route Origin Authorization)** 인증서로 서명합니다.

```
ROA: 203.0.113.0/24 → AS65001로만 광고 가능
BGP 라우터: 다른 AS에서 이 대역 광고 시 INVALID로 필터링
```

ROA 상태:
- **Valid**: ROA와 일치하는 광고 → 허용
- **Invalid**: ROA와 불일치 (다른 AS 또는 더 구체적인 prefix) → 거절
- **Not Found**: ROA 없음 → 정책에 따라 처리

---

## BGP 실전 운영 팁

### 최대 prefix 제한 (Max-Prefix)
피어로부터 수신할 수 있는 최대 라우팅 엔트리 수를 제한합니다. 실수로 전체 라우팅 테이블을 재광고하는 사고를 방지합니다.

### Communities를 활용한 정책 관리
BGP Community는 `AS:값` 형태의 32비트 태그로, 경로 정책을 유연하게 제어합니다:

- `65001:100` = "이 경로를 국내 피어에게만 광고"
- `65001:200` = "이 경로를 업스트림에게도 광고"
- `65535:666` = "블랙홀 커뮤니티 — DDoS 대응 시 트래픽 차단"

### BFD (Bidirectional Forwarding Detection)
BGP의 기본 Hold Time은 90초입니다. 링크 장애 감지가 최대 90초 걸릴 수 있다는 뜻입니다. **BFD**는 서브초 수준으로 링크 장애를 감지하여 BGP 수렴 시간을 크게 단축합니다.

### 인터넷 라우팅 테이블 현황 (2026년 기준)
- IPv4 BGP 테이블 크기: 약 96만 개 이상의 prefix
- IPv6 BGP 테이블 크기: 약 18만 개 이상의 prefix
- 전 세계 AS 수: 약 7만 5천 개 이상

---

## 마무리

BGP는 인터넷을 하나로 묶는 접착제입니다. 단순한 라우팅 프로토콜이 아니라, AS 간의 비즈니스 정책과 기술적 경로 선택이 맞물리는 복잡한 시스템입니다. BGP의 동작 원리를 이해하면 인터넷 장애 분석, 클라우드 서비스의 Anycast 구성, 멀티홈(Multi-homed) 네트워크 설계를 훨씬 깊이 있게 이해할 수 있습니다. 특히 RPKI와 BGPsec의 도입으로 BGP 보안이 강화되고 있는 현재, 인터넷의 신뢰성은 점점 더 향상되고 있습니다.

## 참고 자료
- [What Is BGP? Border Gateway Protocol Explained - Cloudflare](https://www.cloudflare.com/learning/security/glossary/what-is-bgp/)
- [Border Gateway Protocol (BGP) - GeeksforGeeks](https://www.geeksforgeeks.org/computer-networks/border-gateway-protocol-bgp/)
- [RFC 4271: A Border Gateway Protocol 4 (BGP-4)](https://www.rfc-editor.org/rfc/rfc4271)
- [What is BGP? - AWS Networking](https://aws.amazon.com/what-is/border-gateway-protocol/)
