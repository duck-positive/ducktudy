---
layout: post
title: "DPDK 완전 정복: 커널 바이패스로 달성하는 초고속 패킷 처리"
date: 2026-08-18
categories: [cs, computer-science]
tags: [dpdk, kernel-bypass, networking, pmd, hugepage, zero-copy, user-space, high-performance]
---

초당 수천만 패킷을 처리해야 하는 고성능 네트워크 애플리케이션에서 Linux 커널의 네트워크 스택은 걸림돌이 된다. 인터럽트 처리, 컨텍스트 스위칭, 시스템 콜 오버헤드, 그리고 커널↔유저스페이스 간의 메모리 복사가 누적되면 패킷 하나당 수십 마이크로초의 지연이 발생한다. **DPDK(Data Plane Development Kit)** 는 이 모든 병목을 제거하고 유저 스페이스에서 직접 NIC를 제어함으로써 10GbE, 100GbE 와이어 속도에 근접한 처리량을 달성한다. 본 글에서는 DPDK의 핵심 아키텍처와 최적화 원리를 코드와 함께 깊이 있게 살펴본다.

## 개념 설명: 커널 스택의 병목과 DPDK의 해법

### 기존 Linux 네트워크 스택의 문제

일반 소켓 기반 패킷 처리 경로는 다음과 같다:

```
NIC 하드웨어
  → DMA로 커널 메모리에 패킷 저장
  → 하드웨어 인터럽트 발생
  → 커널 인터럽트 핸들러 실행
  → 소프트 IRQ(NAPI) 처리
  → 프로토콜 스택(IP, TCP/UDP) 처리
  → 소켓 버퍼 복사
  → 유저스페이스 read() 시스템 콜
  → 메모리 복사 (커널 → 유저 버퍼)
```

패킷 하나당 발생하는 비용:
- **인터럽트**: CPU 컨텍스트를 저장하고 인터럽트 핸들러를 실행
- **시스템 콜**: 커널 모드 전환 비용 (약 1,000+ CPU 사이클)
- **메모리 복사**: 커널 버퍼 → 유저 버퍼 (캐시 무효화 포함)
- **Lock 경합**: 소켓 버퍼 접근 시 스핀락

10Gbps 링크에서 최소 패킷(64바이트)은 초당 약 1,488만 개다. 위 비용이 패킷당 수백 나노초라도 누적되면 처리 불가능해진다.

### DPDK의 핵심 원리

DPDK는 네 가지 근본적인 변화를 통해 이 병목을 해결한다.

**1. 폴링 모드 드라이버(PMD, Poll Mode Driver)**
인터럽트 대신 전용 코어가 NIC RX 큐를 바쁜 대기(busy polling)로 지속 감시한다. 인터럽트 오버헤드와 컨텍스트 스위칭이 완전히 사라진다.

**2. 유저스페이스 NIC 드라이버**
VFIO(Virtual Function I/O) 또는 UIO(Userspace I/O)를 통해 NIC를 유저스페이스에서 직접 제어한다. 커널 모드 전환과 소켓 API를 우회한다.

**3. 거대 페이지(Hugepage) 메모리**
2MB 또는 1GB 거대 페이지를 사용하면 TLB 미스가 크게 줄어 메모리 접근 레이턴시가 감소한다.

**4. 제로 카피(Zero-Copy)**
패킷 버퍼를 NIC DMA 영역과 애플리케이션이 직접 공유한다. 커널↔유저스페이스 간의 복사가 없다.

## 왜 필요한가: DPDK의 적용 사례

DPDK는 레이턴시와 처리량이 극단적으로 요구되는 환경에서 사용된다.

- **통신 장비**: 5G 기지국, 코어 라우터, 방화벽
- **클라우드 가상화**: OVS-DPDK (Open vSwitch with DPDK)로 VM 간 고속 네트워킹
- **트레이딩 시스템**: 초저지연 시장 데이터 처리 및 주문 전송
- **NFV(Network Function Virtualization)**: 소프트웨어 기반 로드 밸런서, DPI(Deep Packet Inspection)
- **보안 장비**: 고속 IDS/IPS, DDoS 방어 시스템

## 실제 구현 예제 1: DPDK 기본 패킷 수신 루프

아래는 DPDK C API를 사용한 최소 패킷 처리 루프다. 실제로 컴파일하려면 DPDK 라이브러리가 설치되어 있어야 하지만, 구조를 이해하는 데 집중하자.

```c
#include <rte_eal.h>
#include <rte_ethdev.h>
#include <rte_mbuf.h>

#define RX_RING_SIZE 1024
#define TX_RING_SIZE 1024
#define NUM_MBUFS    8191       /* 메모리 풀 크기 (2^n - 1 권장) */
#define MBUF_CACHE_SIZE 250
#define BURST_SIZE   32         /* 한 번에 처리할 최대 패킷 수 */

static struct rte_mempool *mbuf_pool;
static uint16_t port_id = 0;

/* NIC 포트 초기화 */
static int port_init(uint16_t port) {
    struct rte_eth_conf port_conf = {
        .rxmode = { .max_lro_pkt_size = RTE_ETHER_MAX_LEN },
    };
    int ret;
    uint16_t nb_rxd = RX_RING_SIZE;
    uint16_t nb_txd = TX_RING_SIZE;

    /* NIC 설정 */
    ret = rte_eth_dev_configure(port, 1, 1, &port_conf);
    if (ret < 0) return ret;

    /* RX/TX 큐 조정 */
    rte_eth_dev_adjust_nb_rx_tx_desc(port, &nb_rxd, &nb_txd);

    /* RX 큐 설정: mbuf_pool에서 버퍼를 직접 사용 (제로 카피 기반) */
    ret = rte_eth_rx_queue_setup(port, 0, nb_rxd,
        rte_eth_dev_socket_id(port), NULL, mbuf_pool);
    if (ret < 0) return ret;

    /* TX 큐 설정 */
    ret = rte_eth_tx_queue_setup(port, 0, nb_txd,
        rte_eth_dev_socket_id(port), NULL);
    if (ret < 0) return ret;

    /* 포트 시작 */
    return rte_eth_dev_start(port);
}

/* 메인 처리 루프 (전용 코어에서 실행) */
static int lcore_main(__attribute__((unused)) void *arg) {
    struct rte_mbuf *bufs[BURST_SIZE];
    uint16_t nb_rx;

    printf("처리 시작: 코어 %u, 포트 %u\n",
           rte_lcore_id(), port_id);

    /* 무한 폴링 루프 - 인터럽트 없음! */
    for (;;) {
        /* NIC RX 큐에서 최대 BURST_SIZE개의 패킷을 한 번에 수신 */
        nb_rx = rte_eth_rx_burst(port_id, 0, bufs, BURST_SIZE);

        if (unlikely(nb_rx == 0))
            continue;  /* 패킷 없음 → 즉시 재폴링 */

        /* 수신한 패킷 처리 */
        for (uint16_t i = 0; i < nb_rx; i++) {
            struct rte_mbuf *m = bufs[i];
            struct rte_ether_hdr *eth;
            eth = rte_pktmbuf_mtod(m, struct rte_ether_hdr *);

            /* 이더타입 확인 예시 */
            if (rte_be_to_cpu_16(eth->ether_type) == RTE_ETHER_TYPE_IPV4) {
                /* IPv4 패킷 처리 로직 */
                /* ... */
            }

            /* mbuf 해제 (풀에 반환) */
            rte_pktmbuf_free(m);
        }
    }
    return 0;
}

int main(int argc, char *argv[]) {
    int ret;
    unsigned lcore_id;

    /* EAL(Environment Abstraction Layer) 초기화:
       거대 페이지 매핑, CPU 친화성 설정, NIC 바인딩 등 */
    ret = rte_eal_init(argc, argv);
    if (ret < 0) rte_exit(EXIT_FAILURE, "EAL 초기화 실패\n");

    /* mbuf 메모리 풀 생성 (거대 페이지에 할당) */
    mbuf_pool = rte_pktmbuf_pool_create(
        "MBUF_POOL",          /* 풀 이름 */
        NUM_MBUFS,            /* 총 mbuf 수 */
        MBUF_CACHE_SIZE,      /* 코어당 캐시 크기 */
        0,                    /* private data 크기 */
        RTE_MBUF_DEFAULT_BUF_SIZE,  /* 버퍼 크기 */
        rte_socket_id()       /* NUMA 노드 */
    );

    /* 포트 초기화 */
    port_init(port_id);

    /* 워커 코어에서 처리 루프 실행 */
    RTE_LCORE_FOREACH_WORKER(lcore_id) {
        rte_eal_remote_launch(lcore_main, NULL, lcore_id);
    }

    rte_eal_mp_wait_lcore();
    rte_eal_cleanup();
    return 0;
}
```

핵심 포인트:
- `rte_eth_rx_burst()`는 커널 시스템 콜이 아닌 **직접 NIC 레지스터 폴링**이다.
- `rte_pktmbuf_mtod()`는 mbuf 내 데이터 포인터를 직접 반환한다. 복사 없음.
- mbuf 메모리 풀은 **거대 페이지**에 미리 할당되어 TLB 미스가 거의 없다.

## 실제 구현 예제 2: RSS와 멀티코어 스케일 아웃

단일 코어만으로는 100GbE를 처리하기 어렵다. DPDK는 RSS(Receive Side Scaling)를 활용해 NIC가 여러 RX 큐로 패킷을 자동 분산하고, 각 큐를 전용 코어가 담당하는 구조를 지원한다.

```c
/* 멀티 큐 설정 예시 */
static int setup_multi_queue(uint16_t port, uint16_t nb_queues) {
    struct rte_eth_conf port_conf = {
        .rxmode = {
            .mq_mode = RTE_ETH_MQ_RX_RSS,  /* RSS 활성화 */
        },
        .rx_adv_conf = {
            .rss_conf = {
                /* 5-튜플(src IP, dst IP, src port, dst port, proto) 기반 해싱 */
                .rss_hf = RTE_ETH_RSS_TCP | RTE_ETH_RSS_UDP | RTE_ETH_RSS_IP,
            },
        },
    };

    /* nb_queues개의 RX/TX 큐 설정 */
    rte_eth_dev_configure(port, nb_queues, nb_queues, &port_conf);

    for (uint16_t q = 0; q < nb_queues; q++) {
        rte_eth_rx_queue_setup(port, q, RX_RING_SIZE,
            rte_eth_dev_socket_id(port), NULL, mbuf_pool);
        rte_eth_tx_queue_setup(port, q, TX_RING_SIZE,
            rte_eth_dev_socket_id(port), NULL);
    }

    return rte_eth_dev_start(port);
}

/* 각 워커 코어는 자신의 큐만 폴링 */
static int worker_loop(void *arg) {
    uint16_t queue_id = (uint16_t)(uintptr_t)arg;
    struct rte_mbuf *bufs[BURST_SIZE];

    for (;;) {
        uint16_t nb_rx = rte_eth_rx_burst(port_id, queue_id, bufs, BURST_SIZE);
        for (uint16_t i = 0; i < nb_rx; i++) {
            process_packet(bufs[i]);
            rte_pktmbuf_free(bufs[i]);
        }
    }
    return 0;
}
```

RSS는 동일 TCP 연결(5-튜플이 같은)의 패킷이 항상 같은 큐로 들어오도록 보장한다. 덕분에 연결별 상태(connection state)를 코어 간 공유 없이 로컬하게 유지할 수 있어 **락(lock) 없이 수평 확장**이 가능하다.

### NUMA 친화적 메모리 배치

멀티소켓 서버에서는 NIC가 연결된 NUMA 노드와 같은 노드의 메모리를 사용해야 한다. 다른 노드의 메모리에 접근하면 QPI/UPI 인터커넥트를 통해 지연이 2~3배 증가한다.

```c
/* NIC 소켓과 같은 NUMA 노드에 메모리 풀 생성 */
uint8_t socket_id = rte_eth_dev_socket_id(port_id);
mbuf_pool = rte_pktmbuf_pool_create("POOL", NUM_MBUFS,
    MBUF_CACHE_SIZE, 0, RTE_MBUF_DEFAULT_BUF_SIZE, socket_id);
```

## DPDK의 핵심 컴포넌트

### EAL (Environment Abstraction Layer)

DPDK의 기반 계층으로 플랫폼 독립적인 추상화를 제공한다.

- **거대 페이지 매핑**: `/dev/hugepages` 에서 2MB/1GB 페이지 할당
- **CPU 친화성**: `rte_eal_init()`에서 `--lcores` 옵션으로 코어-스레드 매핑
- **메모리 채널 정보**: DRAM 채널 수에 맞게 메모리 배치 최적화
- **PCI 장치 탐색**: DPDK가 관리할 NIC를 커널 드라이버에서 언바인딩

### mempool

`rte_mempool`은 고정 크기 객체의 락-프리 풀이다. 멀티코어 환경에서 각 코어가 자신만의 캐시를 가져 중앙 풀 접근을 최소화한다.

```
전역 링 (Global Ring)
    ↑ 비어있을 때 보충     ↓ 넘칠 때 반환
코어 0 로컬 캐시    코어 1 로컬 캐시    ...
```

로컬 캐시 크기가 충분하면 코어 간 경합 없이 O(1) 할당/해제가 가능하다.

### rte_ring

DPDK의 락-프리 단일 생산자-단일 소비자(SPSC) 또는 다중(MPMC) 링 큐다. CAS(Compare-And-Swap) 연산 기반으로 구현되어 있어 `lock_free-cas-atomic-operations` 포스트에서 다룬 원리와 동일하다.

## 주의사항 및 팁

### 1. 코어를 OS 스케줄러에서 격리하라

DPDK PMD 코어는 `isolcpus` 커널 파라미터로 Linux 스케줄러에서 격리해야 한다. 다른 프로세스나 커널 스레드가 해당 코어에서 실행되면 폴링 레이턴시가 크게 올라간다.

```bash
# GRUB 설정에 추가 (CPU 2, 3 격리)
GRUB_CMDLINE_LINUX="isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3"
```

### 2. 거대 페이지를 부팅 시 예약하라

런타임에 거대 페이지를 할당하면 메모리 단편화로 실패할 수 있다.

```bash
# 2MB 거대 페이지 1024개 예약
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```

### 3. NIC를 VFIO 드라이버에 바인딩하라

```bash
# 커널 드라이버 언바인딩 후 VFIO 바인딩
dpdk-devbind.py --bind=vfio-pci 0000:01:00.0
```

VFIO는 IOMMU를 통해 DMA 보안을 유지하면서도 유저스페이스 접근을 허용한다. UIO보다 안전하다.

### 4. 배치 크기(Burst Size)를 적절히 설정하라

`rte_eth_rx_burst()`의 배치 크기는 성능에 큰 영향을 미친다. 너무 작으면 함수 호출 오버헤드가 크고, 너무 크면 레이턴시가 증가한다. 32~64가 일반적인 최적 값이다.

### 5. prefetch로 캐시 미스를 숨겨라

DPDK의 많은 샘플에서 다음 패킷을 미리 prefetch하여 캐시 미스 레이턴시를 처리 시간 뒤로 숨기는 패턴을 볼 수 있다.

```c
for (uint16_t i = 0; i < nb_rx; i++) {
    /* 두 번째 다음 패킷을 미리 L1/L2 캐시에 올림 */
    rte_prefetch0(rte_pktmbuf_mtod(bufs[i + 2], void *));
    process_packet(bufs[i]);
}
```

DPDK는 단순한 라이브러리가 아니라 **하드웨어와 OS의 경계를 재정의하는 아키텍처적 선택**이다. 커널 스택이 주는 이식성과 안전성을 포기하는 대가로, 와이어 속도에 근접하는 처리량과 수십 나노초의 레이턴시를 달성할 수 있다. 5G, 클라우드 데이터센터, 금융 인프라에서 DPDK가 사실상의 표준으로 자리 잡은 이유가 여기에 있다.

## 참고 자료
- [DPDK/dpdk - Official Repository](https://github.com/DPDK/dpdk)
- [harshraj1695/DPDK_For_Beginners - EAL 초기화부터 패킷 처리까지](https://github.com/harshraj1695/DPDK_For_Beginners)
