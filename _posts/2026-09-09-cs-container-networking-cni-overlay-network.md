---
layout: post
title: "컨테이너 네트워킹 완전 정복: CNI 플러그인·veth pair·VXLAN 오버레이 네트워크의 내부 동작 원리"
date: 2026-09-09
categories: [cs, computer-science]
tags: [container-networking, cni, vxlan, overlay-network, kubernetes, linux-networking, veth, flannel, calico]
---

## 개념 설명

컨테이너 기술의 핵심 인프라 중 하나인 **컨테이너 네트워킹**은, 컨테이너들이 서로 그리고 외부 세계와 통신하는 방법을 정의한다. Docker, Kubernetes 등이 리눅스 커널의 네트워크 네임스페이스, 가상 이더넷 페어(veth pair), 브릿지, 오버레이 네트워크를 조합해 격리되면서도 연결된 네트워크 환경을 제공한다.

### 리눅스 네트워크 네임스페이스

컨테이너 격리의 출발점은 **네트워크 네임스페이스(Network Namespace)**다. 각 네임스페이스는 독립적인 네트워크 스택(인터페이스, 라우팅 테이블, iptables 규칙, 소켓)을 가진다. 컨테이너 하나 = 네트워크 네임스페이스 하나가 대응한다.

```
호스트 네임스페이스
  ├── eth0 (물리 NIC)
  ├── docker0 (브릿지)
  └── veth0 ←──────────── veth1 (컨테이너 namespace의 eth0)
```

### veth pair (Virtual Ethernet Pair)

**veth pair**는 항상 쌍으로 생성되는 가상 네트워크 인터페이스다. 한쪽 끝에 데이터를 쓰면 반대쪽 끝에서 즉시 읽을 수 있다. 리눅스 파이프의 네트워크 버전이라고 이해하면 된다. 하나는 호스트 네임스페이스에, 다른 하나는 컨테이너 네임스페이스에 배치하면 컨테이너와 호스트가 통신할 수 있다.

### CNI (Container Network Interface)

**CNI(Container Network Interface)**는 CNCF가 관리하는 표준 인터페이스로, 컨테이너 런타임이 네트워킹 플러그인을 호출하는 방식을 정의한다. Kubernetes는 CNI 플러그인(Flannel, Calico, Cilium, Weave Net 등)을 통해 Pod 네트워킹을 구성한다. CNI 플러그인은 간단한 실행 파일 형태이며, JSON 설정과 환경 변수를 통해 런타임과 통신한다.

---

## 왜 컨테이너 네트워킹을 이해해야 하는가

### 운영 관점

Kubernetes 클러스터의 네트워크 문제를 디버깅하려면 iptables 규칙, 라우팅 테이블, CNI 설정을 직접 들여다봐야 한다. "Pod가 외부와 통신이 안 된다"는 증상은 수십 가지 원인이 있을 수 있으며, 내부 동작 원리를 알아야 빠르게 root cause를 찾을 수 있다.

### 성능 관점

오버레이 네트워크(VXLAN)는 캡슐화/역캡슐화 오버헤드가 있다. 고성능이 필요한 워크로드에서는 BGP 기반의 언더레이 라우팅(Calico eBGP)이나 eBPF 기반(Cilium)을 선택해야 한다. 이 차이를 이해하지 못하면 불필요한 성능 저하를 겪는다.

### 보안 관점

컨테이너 간 트래픽 격리, NetworkPolicy 적용, 이그레스 제어 등은 CNI 플러그인이 담당한다. 올바른 보안 경계를 설정하려면 트래픽 흐름을 정확히 이해해야 한다.

---

## 실제 구현 예제

### 예제 1: 리눅스에서 직접 컨테이너 네트워킹 구현하기

다음 셸 스크립트는 Docker 없이 순수 리눅스 커맨드만으로 컨테이너 네트워킹을 구성한다. Kubernetes CNI 플러그인이 내부적으로 하는 일을 직접 수행한다.

```bash
#!/bin/bash
# 컨테이너 네트워킹 수동 구성 스크립트
# root 권한 필요

set -euo pipefail

CONTAINER_NS="container-ns-1"
VETH_HOST="veth-host-1"
VETH_CONTAINER="veth-ctr-1"
BRIDGE_NAME="br-demo"
HOST_IP="10.100.0.1/24"
CONTAINER_IP="10.100.0.2/24"

echo "=== 1. 네트워크 네임스페이스 생성 ==="
ip netns add "$CONTAINER_NS"
ip netns list

echo "=== 2. 브릿지 생성 및 활성화 ==="
ip link add name "$BRIDGE_NAME" type bridge
ip addr add "$HOST_IP" dev "$BRIDGE_NAME"
ip link set "$BRIDGE_NAME" up

echo "=== 3. veth pair 생성 ==="
ip link add "$VETH_HOST" type veth peer name "$VETH_CONTAINER"

echo "=== 4. veth 한쪽을 컨테이너 네임스페이스로 이동 ==="
ip link set "$VETH_CONTAINER" netns "$CONTAINER_NS"

echo "=== 5. 호스트 쪽 veth를 브릿지에 연결 ==="
ip link set "$VETH_HOST" master "$BRIDGE_NAME"
ip link set "$VETH_HOST" up

echo "=== 6. 컨테이너 네임스페이스 내부 설정 ==="
# 컨테이너 네임스페이스에서 실행: IP 할당 + 루프백 + 기본 게이트웨이
ip netns exec "$CONTAINER_NS" ip link set lo up
ip netns exec "$CONTAINER_NS" ip link set "$VETH_CONTAINER" up
ip netns exec "$CONTAINER_NS" ip addr add "$CONTAINER_IP" dev "$VETH_CONTAINER"
ip netns exec "$CONTAINER_NS" ip route add default via 10.100.0.1

echo "=== 7. 호스트에서 컨테이너로 ping 테스트 ==="
ping -c 3 10.100.0.2

echo "=== 8. 컨테이너에서 외부로 통신 설정 (NAT) ==="
# IP 포워딩 활성화
echo 1 > /proc/sys/net/ipv4/ip_forward

# MASQUERADE: 컨테이너 패킷의 소스 IP를 호스트 IP로 변환
iptables -t nat -A POSTROUTING -s 10.100.0.0/24 -j MASQUERADE

echo "=== 9. 컨테이너에서 외부 ping 테스트 ==="
ip netns exec "$CONTAINER_NS" ping -c 3 8.8.8.8

echo "=== 정리 ==="
# ip netns del "$CONTAINER_NS"
# ip link del "$BRIDGE_NAME"
# iptables -t nat -D POSTROUTING -s 10.100.0.0/24 -j MASQUERADE

echo "네트워크 구성 완료!"
```

### 예제 2: 간단한 CNI 플러그인 구현 (Go)

CNI 플러그인은 단순한 실행 파일이다. `ADD` 명령을 받으면 네트워크를 설정하고, `DEL`을 받으면 정리한다.

```go
package main

import (
    "encoding/json"
    "fmt"
    "net"
    "os"
    "os/exec"

    "github.com/containernetworking/cni/pkg/skel"
    "github.com/containernetworking/cni/pkg/types"
    current "github.com/containernetworking/cni/pkg/types/100"
    "github.com/containernetworking/cni/pkg/version"
    "github.com/containernetworking/plugins/pkg/ip"
    "github.com/containernetworking/plugins/pkg/ns"
    "github.com/vishvananda/netlink"
)

// CNI 설정 구조체
type NetConf struct {
    types.NetConf
    Subnet  string `json:"subnet"`
    Gateway string `json:"gateway"`
}

func cmdAdd(args *skel.CmdArgs) error {
    conf := &NetConf{}
    if err := json.Unmarshal(args.StdinData, conf); err != nil {
        return fmt.Errorf("failed to parse config: %w", err)
    }

    // 컨테이너 네임스페이스 가져오기
    netns, err := ns.GetNS(args.Netns)
    if err != nil {
        return fmt.Errorf("failed to open netns %q: %w", args.Netns, err)
    }
    defer netns.Close()

    // veth pair 생성
    hostVethName, containerVeth, err := ip.SetupVeth(args.IfName, 1500, "", netns)
    if err != nil {
        return err
    }

    // 호스트 쪽 veth를 브릿지에 연결
    bridge, err := netlink.LinkByName("cni0")
    if err != nil {
        return fmt.Errorf("failed to find bridge cni0: %w", err)
    }

    hostVeth, err := netlink.LinkByName(hostVethName)
    if err != nil {
        return err
    }
    if err := netlink.LinkSetMaster(hostVeth, bridge); err != nil {
        return fmt.Errorf("failed to set master: %w", err)
    }

    // 컨테이너 네임스페이스 내부에서 IP 설정
    var result *current.Result
    err = netns.Do(func(_ ns.NetNS) error {
        // IP 주소 할당 (실제로는 IPAM 플러그인에서 가져옴)
        containerAddr, subnet, _ := net.ParseCIDR(conf.Subnet)
        addr := &netlink.Addr{
            IPNet: &net.IPNet{
                IP:   containerAddr,
                Mask: subnet.Mask,
            },
        }

        link, err := netlink.LinkByName(args.IfName)
        if err != nil {
            return err
        }
        if err := netlink.AddrAdd(link, addr); err != nil {
            return err
        }

        // 기본 게이트웨이 설정
        gw := net.ParseIP(conf.Gateway)
        route := &netlink.Route{
            LinkIndex: link.Attrs().Index,
            Gw:        gw,
        }
        if err := netlink.RouteAdd(route); err != nil {
            return err
        }

        result = &current.Result{
            CNIVersion: conf.CNIVersion,
            Interfaces: []*current.Interface{
                {Name: hostVethName, Sandbox: ""},
                {Name: containerVeth.Name, Sandbox: args.Netns},
            },
            IPs: []*current.IPConfig{
                {
                    Interface: current.Int(1),
                    Address:   *addr.IPNet,
                    Gateway:   gw,
                },
            },
        }
        return nil
    })
    if err != nil {
        return err
    }

    return types.PrintResult(result, conf.CNIVersion)
}

func cmdDel(args *skel.CmdArgs) error {
    // 컨테이너 네임스페이스의 veth 인터페이스 삭제 (veth pair 자동 삭제)
    netns, err := ns.GetNS(args.Netns)
    if err != nil {
        return nil // 네임스페이스가 이미 없으면 무시
    }
    defer netns.Close()

    return netns.Do(func(_ ns.NetNS) error {
        iface, err := netlink.LinkByName(args.IfName)
        if err != nil {
            return nil // 인터페이스가 없으면 이미 정리된 것
        }
        return netlink.LinkDel(iface)
    })
}

func main() {
    skel.PluginMain(cmdAdd, cmdCheck, cmdDel,
        version.All,
        "simple-cni-plugin v0.1.0")
}

func cmdCheck(args *skel.CmdArgs) error {
    return nil
}
```

### VXLAN 오버레이 네트워크 동작 원리

단일 호스트 내 통신은 브릿지로 해결되지만, **다른 호스트에 있는 컨테이너 간 통신**은 오버레이 네트워크가 필요하다. Flannel의 VXLAN 모드가 대표적이다.

```
호스트 A (192.168.1.10)              호스트 B (192.168.1.20)
┌─────────────────────────┐          ┌─────────────────────────┐
│ Pod A (10.244.0.2)      │          │ Pod B (10.244.1.3)      │
│   └── veth ── cni0      │          │   └── veth ── cni0      │
│         (10.244.0.1)    │          │         (10.244.1.1)    │
│              │          │          │              │          │
│         flannel.1       │          │         flannel.1       │
│    (VTEP: VXLAN 터널)   │          │    (VTEP: VXLAN 터널)   │
│              │          │          │              │          │
│           eth0          │──UDP────▶│           eth0          │
│      (192.168.1.10)     │ port 8472│      (192.168.1.20)     │
└─────────────────────────┘          └─────────────────────────┘

패킷 흐름:
원본: src=10.244.0.2, dst=10.244.1.3
  ↓ VXLAN 캡슐화 (flannel.1)
외부: src=192.168.1.10, dst=192.168.1.20, UDP 8472
  ↓ 호스트 B 수신, VXLAN 역캡슐화
원본: src=10.244.0.2, dst=10.244.1.3 → Pod B 전달
```

```bash
# VXLAN 인터페이스 수동 생성 및 확인
ip link add vxlan0 type vxlan id 42 \
    dstport 4789 \
    remote 192.168.1.20 \
    local 192.168.1.10 \
    dev eth0

ip link set vxlan0 up
ip addr add 10.244.0.1/24 dev vxlan0

# FDB (Forwarding Database) 확인: VTEP MAC → 원격 IP 매핑
bridge fdb show dev vxlan0

# 라우팅 테이블에서 오버레이 네트워크 확인
ip route show | grep 10.244

# ARP 캐시 확인: 오버레이 IP → MAC 매핑
ip neigh show dev vxlan0

# tcpdump로 VXLAN 패킷 캡처
tcpdump -i eth0 -n udp port 4789 -w /tmp/vxlan.pcap
```

---

## CNI 플러그인 비교

| 플러그인 | 데이터 플레인 | NetworkPolicy | 성능 | 사용 사례 |
|---------|------------|--------------|------|----------|
| Flannel | VXLAN / host-gw | 미지원 | 중간 | 단순 클러스터 |
| Calico | eBGP / VXLAN | 지원 (Calico NP) | 높음 | 엔터프라이즈 |
| Cilium | eBPF | 지원 (L7까지) | 매우 높음 | 고성능/보안 |
| Weave Net | VXLAN | 지원 | 낮음 | 간단한 멀티호스트 |

---

## 주의사항과 팁

### 1. MTU 설정에 주의하라

VXLAN은 각 패킷에 50바이트 오버헤드를 추가한다(VXLAN 헤더 8B + UDP 8B + IP 20B + 이더넷 14B). 호스트 MTU가 1500이라면 컨테이너 인터페이스의 MTU는 1450으로 설정해야 한다. MTU 불일치는 대용량 패킷이 자동으로 분할되어 성능 저하 및 간헐적 연결 끊김을 유발한다.

```bash
# CNI 설정에서 MTU 명시
{
  "type": "flannel",
  "delegate": {
    "mtu": 1450,
    "isDefaultGateway": true
  }
}
```

### 2. iptables vs eBPF 데이터 플레인

대규모 클러스터(수천 개 Pod)에서 iptables 기반 kube-proxy는 규칙 수가 O(n²)으로 증가해 성능 병목이 된다. Cilium의 eBPF 기반 데이터 플레인은 O(1) 룩업으로 이 문제를 해결한다.

### 3. Pod IP 재사용과 ARP 캐시

Pod가 삭제되고 새 Pod가 같은 IP를 받으면 ARP 캐시에 남은 이전 MAC 주소로 인해 통신 실패가 발생할 수 있다. CNI 플러그인은 `arping`이나 Gratuitous ARP로 이를 처리한다. 문제가 발생하면 `ip neigh flush all`로 ARP 캐시를 강제 정리할 수 있다.

### 4. NetworkPolicy는 CNI 플러그인이 구현한다

Kubernetes의 NetworkPolicy 리소스 자체는 아무것도 하지 않는다. CNI 플러그인이 이를 감지하고 iptables/eBPF 규칙으로 변환해야 Flannel은 NetworkPolicy를 지원하지 않으므로, 보안 격리가 필요하다면 Calico나 Cilium을 선택하라.

### 5. 디버깅 체크리스트

```bash
# Pod 네트워크 디버깅 순서
# 1. Pod IP 확인
kubectl get pod -o wide

# 2. 노드에서 Pod veth 인터페이스 확인
ip link show | grep veth

# 3. 브릿지에 연결된 인터페이스 확인
bridge link show

# 4. 라우팅 테이블
ip route show table all

# 5. iptables 규칙 (kube-proxy)
iptables -t nat -L KUBE-SERVICES -n --line-numbers

# 6. 컨테이너 네임스페이스에서 직접 확인
PID=$(docker inspect --format '{{.State.Pid}}' <container_id>)
nsenter -t $PID -n ip addr
nsenter -t $PID -n ip route

# 7. 연결 테스트
kubectl exec -it <pod> -- curl -v <target-ip>:<port>
```

---

## 참고 자료
- [CNI Specification — GitHub](https://github.com/containernetworking/cni/blob/main/SPEC.md)
- [Flannel 공식 문서](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md)
- [Calico 네트워킹 아키텍처 공식 문서](https://docs.tigera.io/calico/latest/reference/architecture/overview)
- [Cilium eBPF 데이터 플레인 공식 문서](https://docs.cilium.io/en/stable/network/ebpf/)
