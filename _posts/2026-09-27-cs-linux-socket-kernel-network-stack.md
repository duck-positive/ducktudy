---
layout: post
title: "Linux 소켓과 커널 네트워크 스택: sk_buff부터 TCP 송수신까지"
date: 2026-09-27
categories: [cs, computer-science]
tags: [linux, kernel, networking, socket, sk_buff, tcp, netfilter, protocol-stack]
---

## Linux 네트워크 스택 개요

패킷이 NIC(네트워크 인터페이스 카드)에 도착한 순간부터 애플리케이션의 `recv()` 호출이 데이터를 반환하기까지, 수십 개의 커널 함수를 거칩니다. Linux의 네트워킹은 계층적으로 설계되어 있으며, 각 계층은 명확한 역할을 갖습니다.

```
애플리케이션 (user space)
    │  recv() / send()
    ▼
소켓 레이어 (struct socket / struct sock)
    │
    ▼
전송 레이어 (TCP / UDP)
    │
    ▼
네트워크 레이어 (IPv4 / IPv6)
    │
    ▼
링크 레이어 (Ethernet, netfilter)
    │
    ▼
드라이버 레이어 (NIC 드라이버)
    │
    ▼
하드웨어 (NIC / DMA)
```

이 전체 흐름에서 패킷을 담는 핵심 자료구조가 바로 **`struct sk_buff`**입니다.

---

## struct sk_buff: 커널 패킷의 컨테이너

`sk_buff`(socket buffer, 줄여서 `skb`)는 Linux 커널에서 네트워크 패킷을 표현하는 핵심 구조체입니다. `include/linux/skbuff.h`에 정의되어 있으며, 수백 개의 필드를 가집니다. 핵심 개념만 추려보면 다음과 같습니다.

### 메모리 레이아웃

```
                head
                 │
                 ▼
┌────────────────────────────────────────────┐
│   헤드룸  │     실제 데이터     │  테일룸  │
└────────────────────────────────────────────┘
             ▲                   ▲           ▲
           data                tail         end
```

- **`head`**: 할당된 버퍼의 시작 주소
- **`data`**: 현재 유효한 데이터의 시작 (헤더 추가 시 감소)
- **`tail`**: 유효한 데이터의 끝
- **`end`**: 할당된 버퍼의 끝
- **헤드룸**: `head`와 `data` 사이의 공간. 상위 레이어가 하위 레이어 헤더를 **앞에** 추가하는 데 사용
- **테일룸**: `tail`과 `end` 사이의 공간. 데이터를 **뒤에** 추가하는 데 사용

각 레이어가 패킷을 처리하면서 헤더를 추가하거나 제거합니다:
- 송신 시: TCP 헤더 → IP 헤더 → Ethernet 헤더 순으로 **헤드룸에 push**
- 수신 시: Ethernet 헤더 → IP 헤더 → TCP 헤더 순으로 **헤드에서 pull**

### sk_buff 조작 함수

| 함수 | 동작 | 설명 |
|------|------|------|
| `skb_reserve(skb, len)` | `data += len, tail += len` | 헤드룸 확보 |
| `skb_put(skb, len)` | `tail += len` → 반환: 이전 `tail` | 테일에 데이터 공간 추가 |
| `skb_push(skb, len)` | `data -= len` → 반환: 새 `data` | 헤드에 헤더 추가 |
| `skb_pull(skb, len)` | `data += len` → 반환: 새 `data` | 헤드에서 헤더 제거 |

```c
// 송신 경로 예시 (개념적)
struct sk_buff *skb = alloc_skb(MAX_HEADER + payload_len, GFP_KERNEL);

// 헤드룸 확보 (TCP + IP + Ethernet 헤더 공간)
skb_reserve(skb, MAX_HEADER);

// 페이로드 복사
unsigned char *payload = skb_put(skb, payload_len);
memcpy(payload, user_data, payload_len);

// TCP 헤더 추가
struct tcphdr *th = (struct tcphdr *)skb_push(skb, sizeof(struct tcphdr));
// th 필드 채우기...

// IP 헤더 추가
struct iphdr *iph = (struct iphdr *)skb_push(skb, sizeof(struct iphdr));
// iph 필드 채우기...

// Ethernet 헤더 추가
struct ethhdr *eth = (struct ethhdr *)skb_push(skb, ETH_HLEN);
// eth 필드 채우기...

// 드라이버로 전송
dev_queue_xmit(skb);
```

---

## 코드 예제 1: 최소한의 netfilter 훅으로 패킷 관찰

Linux는 **netfilter** 프레임워크를 통해 네트워크 스택의 여러 지점에 훅(hook)을 등록할 수 있습니다. iptables, nftables, conntrack 모두 이 위에 구축됩니다.

```c
// packet_observer.c - 수신 패킷을 로깅하는 최소 커널 모듈
// 빌드: make -C /lib/modules/$(uname -r)/build M=$(pwd) modules
// 주의: 실제 환경에서 테스트 시 반드시 VM을 사용하세요

#include <linux/module.h>
#include <linux/netfilter.h>
#include <linux/netfilter_ipv4.h>
#include <linux/ip.h>
#include <linux/tcp.h>
#include <linux/skbuff.h>

static struct nf_hook_ops netfilter_ops;

// netfilter 훅 콜백: NF_INET_PRE_ROUTING 지점에서 호출됨
static unsigned int observe_packet(void *priv,
                                   struct sk_buff *skb,
                                   const struct nf_hook_state *state)
{
    struct iphdr *iph;
    struct tcphdr *tcph;

    if (!skb) return NF_ACCEPT;

    iph = ip_hdr(skb);           // sk_buff에서 IP 헤더 포인터 획득
    if (!iph) return NF_ACCEPT;

    // TCP 패킷만 관찰
    if (iph->protocol == IPPROTO_TCP) {
        tcph = tcp_hdr(skb);     // IP 헤더 다음의 TCP 헤더
        if (tcph) {
            printk(KERN_INFO "[observer] TCP: %pI4:%u → %pI4:%u, len=%u\n",
                   &iph->saddr, ntohs(tcph->source),
                   &iph->daddr, ntohs(tcph->dest),
                   ntohs(iph->tot_len));
        }
    }

    return NF_ACCEPT;  // 패킷을 계속 처리 (DROP이면 폐기)
}

static int __init observer_init(void)
{
    netfilter_ops.hook     = observe_packet;
    netfilter_ops.pf       = PF_INET;         // IPv4
    netfilter_ops.hooknum  = NF_INET_PRE_ROUTING;  // 라우팅 결정 전
    netfilter_ops.priority = NF_IP_PRI_FIRST;

    nf_register_net_hook(&init_net, &netfilter_ops);
    printk(KERN_INFO "[observer] 패킷 관찰 모듈 로드됨\n");
    return 0;
}

static void __exit observer_exit(void)
{
    nf_unregister_net_hook(&init_net, &netfilter_ops);
    printk(KERN_INFO "[observer] 모듈 언로드됨\n");
}

module_init(observer_init);
module_exit(observer_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("TCP 패킷 관찰 예제");
```

`ip_hdr(skb)`는 `skb->network_header`를 이용하여 sk_buff 데이터 영역 내의 IP 헤더 포인터를 반환합니다. 데이터를 복사하는 것이 아니라 **같은 메모리를 다른 타입으로 해석**합니다. 이것이 sk_buff 설계의 핵심—헤더 포인터는 동일한 버퍼를 오프셋으로 접근합니다.

---

## 코드 예제 2: TCP 에코 서버로 소켓 API 흐름 이해

유저 공간에서 소켓 API를 사용할 때도 내부적으로는 위에서 설명한 커널 경로를 거칩니다.

```c
// tcp_echo_server.c - TCP 에코 서버
// 컴파일: gcc -O2 -o echo_server tcp_echo_server.c

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/tcp.h>  // TCP_NODELAY
#include <arpa/inet.h>
#include <errno.h>

#define PORT 8080
#define BUF_SIZE 4096

int main(void) {
    // 1. 소켓 생성: 커널에서 struct socket + struct sock 생성
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd < 0) { perror("socket"); exit(1); }

    // SO_REUSEADDR: TIME_WAIT 상태의 포트 재사용 허용
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    // TCP_NODELAY: Nagle 알고리즘 비활성화 (소량 데이터 즉시 전송)
    setsockopt(server_fd, IPPROTO_TCP, TCP_NODELAY, &opt, sizeof(opt));

    // 2. bind: 소켓에 로컬 주소 바인딩
    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_addr.s_addr = INADDR_ANY,  // 모든 인터페이스
        .sin_port = htons(PORT),
    };
    if (bind(server_fd, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
        perror("bind"); exit(1);
    }

    // 3. listen: 연결 요청 대기 큐 설정 (backlog=128)
    //    커널이 SYN 큐 + accept 큐를 준비
    listen(server_fd, 128);
    printf("[server] %d 포트에서 대기 중...\n", PORT);

    while (1) {
        struct sockaddr_in client_addr;
        socklen_t client_len = sizeof(client_addr);

        // 4. accept: 완료된 3-way handshake 연결을 큐에서 꺼냄
        //    블로킹: 새 연결이 없으면 대기
        int client_fd = accept(server_fd, (struct sockaddr*)&client_addr, &client_len);
        if (client_fd < 0) { perror("accept"); continue; }

        char client_ip[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &client_addr.sin_addr, client_ip, sizeof(client_ip));
        printf("[server] 연결: %s:%d\n", client_ip, ntohs(client_addr.sin_port));

        // 단순화를 위해 단일 스레드로 처리 (실제 서버는 fork/thread/epoll 사용)
        char buf[BUF_SIZE];
        ssize_t n;

        // 5. recv: 커널의 TCP 수신 버퍼(sk_rcvbuf)에서 데이터 복사
        while ((n = recv(client_fd, buf, sizeof(buf), 0)) > 0) {
            // 6. send: 데이터를 커널의 TCP 송신 버퍼(sk_sndbuf)에 복사
            //    커널이 sk_buff를 생성하고 TCP 세그먼트로 분할하여 전송
            ssize_t sent = 0;
            while (sent < n) {
                ssize_t s = send(client_fd, buf + sent, n - sent, MSG_NOSIGNAL);
                if (s <= 0) goto close_conn;
                sent += s;
            }
        }
        close_conn:
        printf("[server] 연결 종료: %s:%d\n", client_ip, ntohs(client_addr.sin_port));
        close(client_fd);  // FIN 송신 → 4-way handshake 시작
    }

    close(server_fd);
    return 0;
}
```

`recv()`는 유저 공간으로의 진입점이지만, 내부적으로는 `sys_recvfrom → sock_recvmsg → tcp_recvmsg → skb_copy_datagram_iter` 경로를 거쳐 sk_buff의 데이터를 유저 버퍼로 복사합니다. 반대로 `send()`는 `sys_sendto → sock_sendmsg → tcp_sendmsg → tcp_push_one → tcp_transmit_skb` 경로로 sk_buff를 생성하고 하위 레이어로 전달합니다.

---

## 소켓 버퍼와 흐름 제어

TCP의 흐름 제어(flow control)는 소켓 버퍼 크기와 직접 연결됩니다.

- **`sk_rcvbuf`**: 수신 소켓 버퍼 크기. 수신 윈도우 크기를 결정합니다.
- **`sk_sndbuf`**: 송신 소켓 버퍼 크기. `send()`가 블로킹되는 조건을 결정합니다.

```bash
# 시스템 전역 TCP 버퍼 크기 확인
sysctl net.ipv4.tcp_rmem   # 최소/기본/최대 수신 버퍼
sysctl net.ipv4.tcp_wmem   # 최소/기본/최대 송신 버퍼

# 소켓별 버퍼 크기 조정 (코드에서)
int rcvbuf = 256 * 1024;  // 256KB
setsockopt(fd, SOL_SOCKET, SO_RCVBUF, &rcvbuf, sizeof(rcvbuf));
```

Linux의 자동 튜닝(`net.ipv4.tcp_moderate_rcvbuf=1`)이 활성화되면 커널이 연결 상황에 따라 동적으로 버퍼 크기를 조정합니다.

---

## 제로 카피와 sendfile

애플리케이션이 파일을 소켓으로 전송할 때의 전통적인 방식:

```
파일 → 커널 페이지 캐시 → 유저 버퍼(read) → 커널 소켓 버퍼(write) → NIC
```

데이터가 커널-유저 공간을 두 번 왕복합니다. `sendfile(2)` 시스템 콜은 이를 다음으로 단축합니다:

```
파일 → 커널 페이지 캐시 → NIC (DMA 직접)
```

sk_buff의 `skb_shinfo(skb)->frags[]` 필드를 통해 파일의 페이지를 sk_buff에 **참조(reference)**로 첨부하고, DMA가 직접 페이지에서 NIC 버퍼로 전송합니다. 유저 공간으로의 복사가 완전히 사라집니다.

---

## netfilter 훅 포인트

패킷은 네트워크 스택을 이동하면서 총 5개의 netfilter 훅 지점을 통과합니다:

| 훅 | 위치 | 주요 용도 |
|----|------|-----------|
| `NF_INET_PRE_ROUTING` | 라우팅 결정 전 | DNAT (포트 포워딩) |
| `NF_INET_LOCAL_IN` | 로컬 소켓으로 전달 전 | 인바운드 방화벽 |
| `NF_INET_FORWARD` | 포워딩 패킷 | 라우터/브리지 필터링 |
| `NF_INET_LOCAL_OUT` | 로컬 소켓에서 나감 | 아웃바운드 방화벽 |
| `NF_INET_POST_ROUTING` | 전송 직전 | SNAT (IP 마스커레이딩) |

각 훅의 콜백은 `NF_ACCEPT`, `NF_DROP`, `NF_STOLEN`, `NF_QUEUE` 중 하나를 반환하여 패킷 운명을 결정합니다.

---

## 주의사항과 팁

**sk_buff 참조 계수**: `skb_get(skb)`로 참조 계수를 증가시키고, `kfree_skb(skb)`로 감소시킵니다. 참조 계수가 0이 되면 메모리가 해제됩니다. 훅 콜백에서 `NF_STOLEN`을 반환했다면, 해당 skb의 메모리 관리를 직접 담당해야 합니다.

**GFP 플래그**: 인터럽트 컨텍스트(하드웨어 인터럽트, softirq)에서 sk_buff를 할당할 때는 반드시 `GFP_ATOMIC`을 사용해야 합니다. `GFP_KERNEL`은 슬립을 허용하는 컨텍스트에서만 사용 가능합니다.

**소프트 IRQ와 NAPI**: 고성능 NIC는 NAPI(New API)를 사용하여 인터럽트와 폴링을 혼합합니다. 패킷이 대량으로 도착할 때 매번 인터럽트를 발생시키지 않고, `ksoftirqd` 스레드가 배치로 패킷을 처리하여 인터럽트 오버헤드를 줄입니다.

**TCP 버퍼 튜닝**: 대역폭-지연 곱(BDP = bandwidth × RTT)이 큰 고속 네트워크에서는 기본 TCP 버퍼가 부족할 수 있습니다. 예를 들어 10Gbps, 10ms RTT 환경에서 최적 버퍼는 `10Gbps × 10ms / 8 = 12.5MB`입니다.

## 참고 자료
- [Linux kernel skbuff.h (torvalds/linux)](https://github.com/torvalds/linux/blob/master/include/linux/skbuff.h)
- [Linux kernel skbuff.c 구현 (torvalds/linux)](https://github.com/torvalds/linux/blob/master/net/core/skbuff.c)
- [Linux Networking Internals 노트 (jemmy512/book-notes)](https://github.com/jemmy512/book-notes/blob/master/linux/understanding-linux-network-internals.md)
