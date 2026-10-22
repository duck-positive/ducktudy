---
layout: post
title: "XDP(eXpress Data Path) 완전 정복: eBPF로 구현하는 커널 우회 초고속 패킷 처리"
date: 2026-10-22
categories: [cs, computer-science]
tags: [XDP, eBPF, Linux, 네트워크, 패킷처리, AF_XDP, 고성능네트워킹, DDoS방어]
---

현대 네트워크 인프라는 초당 수백만 개의 패킷을 처리해야 합니다. 전통적인 Linux 네트워킹 스택은 유연하고 기능이 풍부하지만, 그 유연성 때문에 레이턴시와 처리량에서 한계를 보입니다. XDP(eXpress Data Path)는 이 문제를 eBPF 프로그램을 네트워크 드라이버 수준에서 실행하여 해결합니다. 이 글에서는 XDP의 동작 원리, 구현 방법, 그리고 실제 활용 사례를 깊이 있게 살펴봅니다.

## XDP란 무엇인가: 개념과 배경

XDP는 Linux 커널 4.8(2016년)에 처음 도입된 고성능 네트워크 처리 프레임워크입니다. 핵심 아이디어는 단순합니다: **패킷이 커널 네트워킹 스택에 진입하기 전에**, 즉 네트워크 드라이버(NIC 드라이버)가 패킷을 수신한 직후에 eBPF 프로그램을 실행하는 것입니다.

기존 Linux 네트워킹 스택에서 패킷은 다음 경로를 따릅니다:

```
NIC 하드웨어 → 드라이버 → sk_buff 할당 → netif_rx() → 네트워킹 스택
→ 소켓 버퍼 → 사용자 공간 애플리케이션
```

XDP는 드라이버 바로 다음에 끼어듭니다:

```
NIC 하드웨어 → 드라이버 → [XDP 훅] → (필요시) sk_buff 할당 → 네트워킹 스택
```

이 위치가 왜 중요할까요? `sk_buff`(소켓 버퍼) 할당은 패킷당 약 200~300 CPU 사이클을 소비합니다. XDP는 이 단계 전에 실행되므로 불필요한 패킷에 대해 이 비용을 완전히 생략할 수 있습니다.

### XDP가 해결하는 문제

기존의 고성능 네트워킹 솔루션으로는 DPDK(Data Plane Development Kit)가 있었습니다. DPDK는 커널을 완전히 우회하여 사용자 공간에서 직접 NIC를 제어합니다. 성능은 뛰어나지만:

- **커널 네트워킹 스택과 통합 불가**: 같은 NIC로 일반 TCP/IP와 DPDK를 동시에 사용하기 어렵습니다.
- **특수 드라이버 필요**: 대부분의 NIC는 별도의 DPDK 드라이버가 필요합니다.
- **복잡한 운영**: 전용 CPU 코어가 필요하고 운영 복잡도가 높습니다.
- **보안 문제**: 사용자 공간이 하드웨어를 직접 제어하므로 커널 격리가 무의미해집니다.

XDP는 이러한 문제 없이 DPDK에 준하는 성능을 제공합니다. eBPF 검증기(verifier)가 프로그램의 안전성을 보장하므로, XDP 프로그램이 커널을 크래시시킬 수 없습니다.

## XDP의 동작 원리

### XDP 훅 위치

XDP 훅은 세 가지 모드로 동작합니다:

**1. Native XDP (드라이버 모드)**
NIC 드라이버가 XDP를 직접 지원하는 경우입니다. 패킷이 DMA 버퍼에 수신된 직후, `sk_buff` 할당 전에 XDP 프로그램이 실행됩니다. Intel ixgbe, Mellanox mlx5, Amazon ENA 등 주요 드라이버가 지원합니다.

**2. Generic XDP (SKB 모드)**
드라이버가 XDP를 지원하지 않을 때 커널이 제공하는 폴백 모드입니다. `sk_buff` 할당 후에 실행되므로 성능 이점이 줄어들지만, 모든 NIC에서 동작합니다. 개발 및 테스트에 유용합니다.

**3. Offloaded XDP**
일부 스마트 NIC(Netronome Agilio 등)는 XDP 프로그램을 NIC 하드웨어 자체에서 실행할 수 있습니다. CPU를 전혀 사용하지 않아 가장 높은 성능을 제공합니다.

### XDP 프로그램의 반환 코드

XDP 프로그램은 각 패킷을 처리한 후 다음 5가지 중 하나를 반환합니다:

| 반환 코드 | 의미 |
|-----------|------|
| `XDP_PASS` | 패킷을 정상적으로 커널 스택으로 전달 |
| `XDP_DROP` | 패킷을 즉시 폐기 (메모리 해제 포함) |
| `XDP_ABORTED` | 에러로 인한 폐기 (추적 가능) |
| `XDP_TX` | 수신한 NIC로 패킷을 다시 전송 (반사) |
| `XDP_REDIRECT` | 다른 NIC나 CPU, AF_XDP 소켓으로 전달 |

`XDP_DROP`은 DDoS 방어에서 핵심 도구입니다. 악의적인 패킷을 커널 스택 진입 전에 폐기하므로, 패킷당 처리 비용이 수십 나노초에 불과합니다.

## 실제 구현: XDP로 IP 블랙리스트 구현하기

### 환경 준비

```bash
# 필요한 도구 설치 (Ubuntu/Debian)
sudo apt-get install -y clang llvm libelf-dev libbpf-dev linux-headers-$(uname -r)

# bpftool 설치
sudo apt-get install -y linux-tools-$(uname -r)
```

### XDP 프로그램 작성 (C 코드)

XDP 프로그램은 C로 작성하고 eBPF 바이트코드로 컴파일됩니다:

```c
// xdp_drop_ip.c - 특정 IP를 블랙리스트로 차단하는 XDP 프로그램
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

// 블랙리스트 IP를 저장하는 BPF 해시맵
// key: IPv4 주소 (big-endian __u32), value: 폐기 카운터 (__u64)
struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __uint(max_entries, 65536);
    __type(key, __u32);
    __type(value, __u64);
} blocklist SEC(".maps");

SEC("xdp")
int xdp_drop_blocked_ips(struct xdp_md *ctx) {
    // ctx->data: 패킷 시작 포인터 (DMA 버퍼)
    // ctx->data_end: 패킷 끝 포인터
    void *data_end = (void *)(long)ctx->data_end;
    void *data     = (void *)(long)ctx->data;

    // 이더넷 헤더 파싱
    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return XDP_PASS; // 패킷이 너무 짧음 → 스택에 넘김

    // IPv4만 처리
    if (eth->h_proto != bpf_htons(ETH_P_IP))
        return XDP_PASS;

    // IP 헤더 파싱
    struct iphdr *iph = (void *)(eth + 1);
    if ((void *)(iph + 1) > data_end)
        return XDP_PASS;

    // 소스 IP로 블랙리스트 조회
    __u32 src_ip = iph->saddr;
    __u64 *counter = bpf_map_lookup_elem(&blocklist, &src_ip);

    if (counter) {
        // 블랙리스트에 있음: 카운터 증가 후 폐기
        __sync_fetch_and_add(counter, 1);
        return XDP_DROP;
    }

    return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
```

```bash
# eBPF 바이트코드로 컴파일
clang -O2 -target bpf -c xdp_drop_ip.c -o xdp_drop_ip.o

# XDP 프로그램을 인터페이스에 로드 (eth0에 Native 모드로)
sudo ip link set dev eth0 xdp obj xdp_drop_ip.o sec xdp

# 로드 확인
sudo ip link show eth0 | grep xdp

# 블랙리스트에 IP 추가 (Python bpftool 또는 bpfctl 사용)
sudo bpftool map show  # map id 확인
sudo bpftool map update id <MAP_ID> key hex c0 a8 01 64 value hex 00 00 00 00 00 00 00 00
# 위는 192.168.1.100을 블랙리스트에 추가 (big-endian)

# XDP 프로그램 제거
sudo ip link set dev eth0 xdp off
```

### libbpf를 이용한 사용자 공간 제어 프로그램

```c
// xdp_controller.c - libbpf를 사용하는 사용자 공간 컨트롤러
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <arpa/inet.h>
#include <bpf/libbpf.h>
#include <bpf/bpf.h>
#include <net/if.h>
#include <linux/if_link.h>

int main(int argc, char **argv) {
    if (argc < 3) {
        fprintf(stderr, "Usage: %s <ifname> <block_ip>\n", argv[0]);
        return 1;
    }

    const char *ifname = argv[1];
    const char *block_ip_str = argv[2];

    // BPF 오브젝트 열기 및 로드
    struct bpf_object *obj = bpf_object__open("xdp_drop_ip.o");
    if (!obj) {
        fprintf(stderr, "Failed to open BPF object\n");
        return 1;
    }

    if (bpf_object__load(obj)) {
        fprintf(stderr, "Failed to load BPF object: %s\n", strerror(errno));
        return 1;
    }

    // XDP 프로그램 찾기
    struct bpf_program *prog = bpf_object__find_program_by_name(obj, "xdp_drop_blocked_ips");
    if (!prog) {
        fprintf(stderr, "Program not found\n");
        return 1;
    }

    int prog_fd = bpf_program__fd(prog);
    int ifindex = if_nametoindex(ifname);

    // 인터페이스에 XDP 프로그램 연결 (Native 모드 시도, 실패시 Generic 모드)
    int flags = XDP_FLAGS_UPDATE_IF_NOEXIST;
    if (bpf_xdp_attach(ifindex, prog_fd, flags, NULL) < 0) {
        fprintf(stderr, "Failed to attach XDP: %s\n", strerror(errno));
        // Generic 모드로 폴백
        flags |= XDP_FLAGS_SKB_MODE;
        if (bpf_xdp_attach(ifindex, prog_fd, flags, NULL) < 0) {
            fprintf(stderr, "Failed to attach XDP (generic): %s\n", strerror(errno));
            return 1;
        }
        printf("Attached in generic (SKB) mode\n");
    } else {
        printf("Attached in native mode\n");
    }

    // 블랙리스트 맵에 IP 추가
    struct bpf_map *map = bpf_object__find_map_by_name(obj, "blocklist");
    int map_fd = bpf_map__fd(map);

    struct in_addr addr;
    inet_aton(block_ip_str, &addr);
    __u32 key = addr.s_addr; // 이미 network byte order
    __u64 value = 0;

    if (bpf_map_update_elem(map_fd, &key, &value, BPF_ANY) < 0) {
        fprintf(stderr, "Failed to update map: %s\n", strerror(errno));
        return 1;
    }

    printf("Blocking IP: %s on interface %s\n", block_ip_str, ifname);
    printf("Press Ctrl+C to remove and exit\n");

    // 주기적으로 차단 통계 출력
    while (1) {
        sleep(5);
        __u64 counter;
        if (bpf_map_lookup_elem(map_fd, &key, &counter) == 0) {
            printf("Dropped packets from %s: %llu\n", block_ip_str, counter);
        }
    }

    return 0;
}
```

## AF_XDP: 사용자 공간 제로 카피 소켓

Linux 4.18에서 도입된 `AF_XDP`는 XDP와 짝을 이루는 새로운 소켓 패밀리입니다. XDP 프로그램에서 `XDP_REDIRECT`로 패킷을 AF_XDP 소켓으로 전달하면, 사용자 공간 애플리케이션이 **커널 복사 없이** 패킷 데이터에 직접 접근할 수 있습니다.

AF_XDP의 핵심 구성 요소:
- **UMEM(User Memory)**: 사용자 공간이 할당하고 커널과 공유하는 메모리 영역
- **RX/TX Ring**: SPSC(Single Producer Single Consumer) 락 프리 링 버퍼
- **Fill Ring**: 드라이버에게 빈 버퍼를 제공하는 큐
- **Completion Ring**: 전송 완료된 버퍼를 반환받는 큐

```
사용자 공간                     커널
┌──────────────────┐           ┌──────────────────┐
│  UMEM 할당       │◄──공유────►│  NIC 드라이버     │
│  Fill Ring →     │           │  (DMA 직접 쓰기)  │
│  RX Ring ←       │           │                  │
│  TX Ring →       │           │  XDP 프로그램     │
│  Completion ←    │           │  (XDP_REDIRECT)  │
└──────────────────┘           └──────────────────┘
```

이 방식으로 패킷당 복사 비용이 0이 되어, 100Gbps NIC에서도 단일 코어로 라인 레이트 처리가 가능합니다.

## 성능 벤치마크와 실제 활용 사례

### 처리량 비교

커널 버전과 하드웨어에 따라 다르지만, 대표적인 벤치마크 결과입니다:

| 방법 | 패킷 처리량 (pps, 단일 코어) |
|------|-------------------------------|
| 기존 iptables DROP | ~1.5M pps |
| Generic XDP DROP | ~6M pps |
| Native XDP DROP | ~20-24M pps |
| AF_XDP 사용자 공간 처리 | ~12-15M pps |
| Offloaded XDP (스마트 NIC) | ~100M+ pps |

### 실제 사용 사례

**1. DDoS 방어 (Cloudflare, Facebook)**
Cloudflare는 XDP를 사용하여 Memcached UDP amplification 공격(2018년 GitHub 공격에 사용된 방식)을 방어합니다. 악의적인 UDP 패킷을 XDP 레벨에서 차단하여 서버 CPU 부하를 최소화합니다.

**2. 로드 밸런서 (Facebook Katran)**
Facebook의 오픈소스 L4 로드 밸런서 Katran은 XDP로 구현되었습니다. `XDP_TX`와 `XDP_REDIRECT`를 사용하여 패킷을 백엔드로 전달하고, 초당 수십 기가비트의 트래픽을 처리합니다.

**3. 네트워크 모니터링 (Cilium)**
Kubernetes 네트워킹 플러그인 Cilium은 XDP와 eBPF를 기반으로 합니다. 파드 간 트래픽을 XDP 레벨에서 처리하여 iptables 기반 솔루션보다 훨씬 빠른 성능을 제공합니다.

**4. 고성능 패킷 캡처**
tcpdump/libpcap의 대안으로, XDP와 AF_XDP를 사용한 패킷 캡처는 패킷 손실 없이 10Gbps+ 트래픽을 캡처할 수 있습니다.

## 주의사항과 개발 팁

### 검증기(Verifier) 제약 사항

eBPF 검증기는 XDP 프로그램의 안전성을 보장하지만, 개발자에게 몇 가지 제약을 부과합니다:

1. **루프 제한**: 무한 루프 불가. BPF_MAP_TYPE_ARRAY를 순회할 때 최대 반복 횟수를 명시해야 합니다. Linux 5.3+ 에서 bounded loops 지원.

2. **스택 크기 제한**: 512 바이트. 대용량 데이터는 BPF 맵을 사용해야 합니다.

3. **함수 호출 제한**: `bpf_*` 헬퍼 함수만 호출 가능. `memcpy`는 `__builtin_memcpy`로 대체.

4. **포인터 검증 필수**: 모든 포인터 접근 전에 `data_end` 경계 체크가 필요합니다. 이를 놓치면 컴파일은 되지만 로드 시 검증기가 거부합니다.

```c
// 잘못된 예 - 경계 체크 없음 (검증기 거부)
struct ethhdr *eth = data;
__u16 proto = eth->h_proto; // 위험!

// 올바른 예 - 경계 체크 포함
struct ethhdr *eth = data;
if ((void *)(eth + 1) > data_end)
    return XDP_PASS;
__u16 proto = eth->h_proto; // 안전
```

### 개발 환경 팁

- **bpftrace**: 빠른 프로토타이핑에 유용. C 코드 없이 eBPF 프로그램 작성 가능.
- **BCC (BPF Compiler Collection)**: Python 인터페이스로 eBPF 개발. 프로덕션보다 개발/디버깅에 적합.
- **libbpf + CO-RE**: 현대적인 프로덕션 권장 방식. BTF(BPF Type Format)를 사용하여 커널 버전 호환성 문제를 해결합니다.
- **XDP 튜토리얼**: `xdp-tutorial` GitHub 저장소(xdp-project/xdp-tutorial)는 단계별 예제를 제공합니다.
- **veth 디바이스 테스트**: 물리 NIC 없이도 `veth` 가상 이더넷으로 XDP를 테스트할 수 있습니다.

XDP는 Linux 네트워킹의 게임 체인저입니다. 커널의 안전성과 기존 스택과의 호환성을 유지하면서도 DPDK에 준하는 성능을 달성할 수 있습니다. DDoS 방어, 고성능 로드 밸런싱, 네트워크 모니터링 등 다양한 분야에서 XDP는 이미 프로덕션 환경에서 검증된 기술입니다.

## 참고 자료
- [XDP 공식 커널 문서 (prototype-kernel.readthedocs.io)](https://prototype-kernel.readthedocs.io/en/latest/networking/XDP/)
- [LWN.net: A thorough introduction to eBPF](https://lwn.net/Articles/740157/)
- [xdp-project/xdp-tutorial (GitHub)](https://github.com/xdp-project/xdp-tutorial)
- [Cilium BPF and XDP Reference Guide](https://docs.cilium.io/en/stable/bpf/)
