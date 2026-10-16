---
layout: post
title: "NAT 내부 구조 완전 정복: SNAT·DNAT·conntrack으로 이해하는 리눅스 패킷 변환"
date: 2026-10-16
categories: [cs, computer-science]
tags: [NAT, SNAT, DNAT, conntrack, netfilter, iptables, nftables, 네트워크, 리눅스]
---

## NAT(Network Address Translation)란 무엇인가

NAT는 네트워크 패킷의 IP 주소(와 포트)를 라우터 또는 방화벽이 변환하는 메커니즘입니다. 사무실 공유기가 내부 PC 수십 대에 하나의 공인 IP를 나눠 쓰게 하거나, Kubernetes가 클러스터 내부 Pod IP를 외부에 노출할 때 NAT가 사용됩니다. 현대 인터넷은 IPv4 고갈 문제를 NAT로 수십 년째 버텨왔으며, 컨테이너·클라우드 네트워킹의 기반 기술이기도 합니다.

NAT는 단순해 보이지만 상태(state)를 관리해야 합니다. 내부 호스트 192.168.1.10:5000이 공인 IP 1.2.3.4:45678로 변환됐다면, 서버의 응답 패킷이 1.2.3.4:45678로 돌아왔을 때 이를 다시 192.168.1.10:5000으로 되돌려야 합니다. 이 상태 관리를 리눅스에서는 **conntrack(connection tracking)** 서브시스템이 담당합니다.

## 왜 NAT 내부 구조를 알아야 하는가

- **장애 진단:** "연결이 갑자기 끊긴다", "특정 포트만 안 된다" 류의 문제는 conntrack 테이블 고갈이나 NAT 규칙 충돌이 원인인 경우가 많습니다.
- **클라우드 네이티브 아키텍처:** Docker 포트 매핑, Kubernetes kube-proxy의 iptables/IPVS 모드, AWS Security Group, GCP VPC 모두 NAT 위에 구축됩니다.
- **보안 설계:** NAT가 방화벽을 대체한다는 오해를 교정하고 정확한 보안 경계를 설계할 수 있습니다.
- **성능 튜닝:** 고트래픽 환경에서 conntrack 테이블 크기, NAT 해시 버킷 수를 올바르게 조정하려면 내부 동작을 알아야 합니다.

## 리눅스 netfilter 아키텍처와 NAT의 위치

리눅스 커널의 패킷 처리 경로에는 **netfilter 훅(hook)** 포인트가 다섯 군데 있습니다.

```
패킷 수신 ──→ PREROUTING ──→ (라우팅 결정) ──→ FORWARD ──→ POSTROUTING ──→ 송신
                                   │
                                   ↓
                               INPUT → 로컬 프로세스 → OUTPUT → POSTROUTING
```

NAT는 두 가지 주요 변환 방향이 있습니다.

| 유형 | 훅 포인트 | 설명 |
|------|-----------|------|
| **DNAT** (목적지 변환) | PREROUTING (또는 OUTPUT) | 들어오는 패킷의 목적지 IP/포트 변경. 포트 포워딩, 로드밸런서 VIP가 이에 해당 |
| **SNAT** (출발지 변환) | POSTROUTING | 나가는 패킷의 출발지 IP/포트 변경. 사설 IP → 공인 IP 변환(masquerade)이 대표적 |

## conntrack(연결 추적) 내부 구조

### 튜플(Tuple)과 연결 추적 항목

conntrack은 각 연결을 **5-tuple** (프로토콜, 출발지 IP, 출발지 포트, 목적지 IP, 목적지 포트)로 식별합니다. NAT가 적용된 연결은 **원본 튜플**과 **응답 튜플** 두 개를 저장합니다.

```bash
# conntrack 테이블 조회
$ sudo conntrack -L
tcp  6 86380 ESTABLISHED \
    src=192.168.1.10 dst=93.184.216.34 sport=42000 dport=80 \
    src=93.184.216.34 dst=1.2.3.4 sport=80 dport=42000 [ASSURED] \
    mark=0 use=1

# 위 항목 해석:
# 원본 방향: 192.168.1.10:42000 → 93.184.216.34:80
# 응답 방향: 93.184.216.34:80 → 1.2.3.4:42000  (1.2.3.4가 공인 IP)
# 즉, SNAT가 적용돼 내부 IP가 공인 IP로 변환됨

# DNAT된 항목 예시 (포트 포워딩 80→8080)
tcp  6 ESTABLISHED \
    src=203.0.113.1 dst=1.2.3.4 sport=55000 dport=80 \
    src=192.168.1.20 dst=203.0.113.1 sport=8080 dport=55000 [ASSURED]
# 목적지 1.2.3.4:80 → 내부 192.168.1.20:8080으로 DNAT됨
```

### conntrack 상태 머신

TCP 연결은 conntrack 내에서 상태 머신으로 추적됩니다.

```
NEW → ESTABLISHED → TIME_WAIT → CLOSE
```

UDP나 ICMP처럼 연결 개념이 없는 프로토콜도 conntrack이 타이머로 상태를 관리합니다. UDP 기본 타임아웃은 30초, TCP ESTABLISHED는 5일(432000초)이 기본값입니다.

## iptables로 NAT 구현하기

### SNAT (인터넷 공유, Masquerade)

```bash
# eth0이 공인 IP를 가진 인터페이스일 때
# 내부 192.168.1.0/24 → 인터넷 SNAT 설정

# 방법 1: 고정 공인 IP (--to-source)
sudo iptables -t nat -A POSTROUTING \
    -s 192.168.1.0/24 \
    -o eth0 \
    -j SNAT --to-source 1.2.3.4

# 방법 2: 동적 IP (DHCP 환경, MASQUERADE)
sudo iptables -t nat -A POSTROUTING \
    -s 192.168.1.0/24 \
    -o eth0 \
    -j MASQUERADE

# 규칙 확인
sudo iptables -t nat -L POSTROUTING -v -n

# IP 포워딩 활성화 (필수)
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
```

### DNAT (포트 포워딩)

```bash
# 외부 1.2.3.4:80 → 내부 192.168.1.20:8080 포워딩

sudo iptables -t nat -A PREROUTING \
    -d 1.2.3.4 \
    -p tcp --dport 80 \
    -j DNAT --to-destination 192.168.1.20:8080

# 로컬 호스트에서도 같은 포워딩 적용 (hairpin NAT)
sudo iptables -t nat -A OUTPUT \
    -d 1.2.3.4 \
    -p tcp --dport 80 \
    -j DNAT --to-destination 192.168.1.20:8080

# FORWARD 체인에서 허용 (기본 정책이 DROP인 경우)
sudo iptables -A FORWARD \
    -d 192.168.1.20 \
    -p tcp --dport 8080 \
    -j ACCEPT
sudo iptables -A FORWARD \
    -s 192.168.1.20 \
    -p tcp --sport 8080 \
    -j ACCEPT
```

### nftables로 동일 설정 (현대적 방법)

```bash
# /etc/nftables.conf
table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;
        # 포트 포워딩: 공인 IP:80 → 내부 서버:8080
        ip daddr 1.2.3.4 tcp dport 80 dnat to 192.168.1.20:8080
    }

    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        # 사설망 → 인터넷 MASQUERADE
        ip saddr 192.168.1.0/24 oif "eth0" masquerade
    }
}

# 적용
sudo nft -f /etc/nftables.conf
sudo nft list ruleset
```

## conntrack 테이블 용량 관리

높은 트래픽 환경에서 conntrack 테이블이 가득 차면 새 연결이 모두 거부됩니다. 이는 "nf_conntrack: table full, dropping packet" 커널 로그로 확인할 수 있습니다.

```bash
# 현재 conntrack 항목 수 확인
cat /proc/sys/net/netfilter/nf_conntrack_count

# 최대 항목 수 확인 및 조정
cat /proc/sys/net/netfilter/nf_conntrack_max
sudo sysctl -w net.netfilter.nf_conntrack_max=524288

# conntrack 해시 테이블 크기 (항목 수의 1/8 권장)
sudo sysctl -w net.netfilter.nf_conntrack_buckets=65536

# 각 프로토콜의 타임아웃 조정 (초)
sudo sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=300
sudo sysctl -w net.netfilter.nf_conntrack_udp_timeout=30
sudo sysctl -w net.netfilter.nf_conntrack_udp_timeout_stream=180

# /etc/sysctl.d/99-conntrack.conf에 영구 적용
cat << 'EOF' | sudo tee /etc/sysctl.d/99-conntrack.conf
net.netfilter.nf_conntrack_max = 524288
net.netfilter.nf_conntrack_buckets = 65536
net.netfilter.nf_conntrack_tcp_timeout_established = 300
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

## NAT와 애플리케이션 계층 프로토콜의 충돌

NAT는 IP/포트 레이어에서만 동작하기 때문에, 애플리케이션 페이로드에 IP 주소를 포함하는 프로토콜(FTP, SIP, H.323 등)은 NAT를 통과할 때 문제가 생깁니다.

**예: FTP Active 모드**
- 클라이언트가 `PORT 192,168,1,10,20,100` (IP:포트)를 서버에 전송
- 이 내부 IP가 NAT를 통과하면 외부에서 접근 불가능

이를 해결하기 위해 리눅스 커널은 **ALG(Application Layer Gateway)** 모듈을 제공합니다.

```bash
# FTP ALG 모듈 로드
sudo modprobe nf_nat_ftp
sudo modprobe nf_conntrack_ftp

# 로드된 모듈 확인
lsmod | grep -E "nf_nat|nf_conntrack"
```

## Kubernetes에서의 NAT: kube-proxy iptables 모드

Kubernetes의 ClusterIP 서비스는 DNAT로 구현됩니다. 클라이언트 Pod가 ClusterIP:Port로 접근하면 kube-proxy가 설치한 DNAT 규칙이 실제 Pod IP:Port로 변환합니다.

```bash
# kube-proxy가 생성한 NAT 규칙 확인
sudo iptables -t nat -L KUBE-SERVICES -n | head -20

# 예시 출력:
# KUBE-SVC-XXXX tcp -- 0.0.0.0/0 10.96.0.10 tcp dpt:53  # CoreDNS ClusterIP
# KUBE-SVC-YYYY tcp -- 0.0.0.0/0 10.100.0.1 tcp dpt:443 # API server

# KUBE-SVC 체인 내부 (로드밸런싱)
sudo iptables -t nat -L KUBE-SVC-XXXX -n
# -m statistic --mode random --probability 0.33333 -j KUBE-SEP-POD1
# -m statistic --mode random --probability 0.50000 -j KUBE-SEP-POD2
# -j KUBE-SEP-POD3
```

## 주의사항 및 팁

### 1. NAT는 방화벽이 아니다
NAT의 상태 기반 특성이 일부 보안을 제공하지만(내부에서 시작하지 않은 연결은 들어올 수 없음), NAT는 방화벽 설계 원칙이 아닌 IP 주소 변환 메커니즘입니다. 명시적인 iptables/nftables 필터 규칙을 함께 설정해야 합니다.

### 2. 포트 소모(Port Exhaustion)
SNAT는 공인 IP의 포트 번호(1~65535)를 소모합니다. 하나의 공인 IP로 65000개 이상의 동시 외부 TCP 연결을 유지하려면 공인 IP를 추가하거나 포트 범위를 확장해야 합니다.

```bash
# 로컬 포트 범위 확인 및 확장
cat /proc/sys/net/ipv4/ip_local_port_range
sudo sysctl -w net.ipv4.ip_local_port_range="1024 65535"
```

### 3. conntrack과 TIME_WAIT 소켓의 상호작용
TCP TIME_WAIT 상태는 conntrack에서도 지속됩니다. `tw_reuse` 옵션과 conntrack 타임아웃을 함께 조정해야 고부하 서버에서 성능을 최적화할 수 있습니다.

### 4. 비대칭 라우팅 금지
SNAT/DNAT가 설정된 장비에서는 나가는 패킷과 들어오는 패킷이 같은 장비를 통과해야 합니다. 비대칭 라우팅(들어올 때와 나갈 때 다른 경로 사용)이 발생하면 conntrack이 응답 패킷을 역변환하지 못해 연결이 끊깁니다.

### 5. 성능 최적화
SNAT 연산은 CPU 비용이 있습니다. 고트래픽 환경에서는 IPVS(IP Virtual Server) 모드(Kubernetes kube-proxy의 대안)나 eBPF 기반 Cilium을 검토하세요. 이들은 conntrack을 부분적으로 우회하여 수십만 개의 동시 연결에서도 낮은 레이턴시를 유지합니다.

## 참고 자료
- [ArthurChiao's Blog: conntrack Design and Implementation](https://arthurchiao.art/blog/conntrack-design-and-implementation/)
- [Linux man page: netstat-nat](https://man.archlinux.org/man/netstat-nat.1.en)
- [netfilter.org 공식 문서](https://www.netfilter.org/documentation/)
- [Kernel Source: nf_nat_core.c](https://github.com/torvalds/linux/blob/master/net/netfilter/nf_nat_core.c)
