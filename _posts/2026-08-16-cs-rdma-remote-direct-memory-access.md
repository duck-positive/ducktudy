---
layout: post
title: "RDMA(Remote Direct Memory Access) 완전 정복: 커널 바이패스로 마이크로초 레이턴시를 달성하는 네트워크 기술"
date: 2026-08-16
categories: [cs, computer-science]
tags: [rdma, infiniband, roce, networking, distributed-systems, high-performance-computing, verbs-api, zero-copy]
---

## RDMA란 무엇인가

RDMA(Remote Direct Memory Access)는 네트워크로 연결된 두 컴퓨터 사이에서, **운영체제·CPU·캐시의 개입 없이** 한쪽 컴퓨터의 메모리에서 다른 쪽 컴퓨터의 메모리로 데이터를 직접 전송하는 기술이다.

전통적인 TCP/IP 소켓 통신에서는 데이터가 다음 경로를 따라 이동한다:

```
송신측:  애플리케이션 버퍼 → 커널 소켓 버퍼 → NIC 드라이버 → NIC
수신측:  NIC → 커널 소켓 버퍼 → 애플리케이션 버퍼
```

이 과정에서 최소 2~4회의 메모리 복사(memcpy)가 발생하고, 각 단계마다 CPU 인터럽트와 컨텍스트 스위치가 필요하다. 1GbE 환경에서는 이 오버헤드가 수십 마이크로초에 불과하지만, 100GbE/200GbE 환경에서는 소프트웨어 처리 속도가 물리 링크 속도를 따라가지 못하는 CPU 병목이 발생한다.

RDMA는 이 문제를 다음 세 가지 핵심 속성으로 해결한다:

- **Zero-Copy**: 애플리케이션 메모리가 곧 전송 버퍼가 된다. 중간 복사가 없다.
- **Kernel Bypass**: 송수신 경로에서 커널 소켓 스택을 완전히 생략한다. NIC 하드웨어가 직접 DMA로 데이터를 옮긴다.
- **CPU Offload**: 데이터 전송이 진행되는 동안 CPU는 다른 작업을 수행할 수 있다. 완료 시 Completion Queue에 이벤트가 기록된다.

결과적으로 RDMA는 **수 마이크로초(μs) 이하의 레이턴시**와 **수백 Gbps의 처리량**을 달성할 수 있다.

---

## 왜 필요한가 — 데이터센터의 속도 제한

### 고성능 컴퓨팅(HPC)

기상 예측, 분자 동역학 시뮬레이션, 딥러닝 학습 등 HPC 워크로드는 수천 개의 노드가 밀접하게 협력한다. MPI(Message Passing Interface) 집합 연산(AllReduce, AllGather 등)은 노드 간 지연시간에 매우 민감하다. TCP/IP의 수십~수백 μs 레이턴시는 수천 노드 환경에서 동기화 병목이 된다. RDMA는 이를 1~3 μs 수준으로 낮춘다.

### 분산 인메모리 데이터베이스

Redis, Memcached, Apache Spark 같은 시스템에서 노드 간 데이터 이동은 핵심 성능 지표다. RDMA를 활용한 시스템(FaRM, DrTM, HERD 등)은 일반 소켓 기반 시스템 대비 5~10배 더 높은 처리량을 보인다.

### 스토리지 네트워크

NVMe-oF(NVMe over Fabrics)는 RDMA를 사용해 원격 NVMe SSD를 로컬 디바이스처럼 접근한다. 수십 μs 수준의 NVMe 레이턴시를 네트워크 계층에서도 보존하려면 RDMA가 필수다.

### 프로토콜 종류

RDMA를 지원하는 물리 계층은 세 가지다:

| 프로토콜 | 물리 계층 | 특징 |
|---------|----------|------|
| InfiniBand | IB 전용 패브릭 | 가장 낮은 레이턴시, 가장 높은 비용 |
| RoCE v2 | Ethernet (UDP/IP) | DCB/PFC 필요, 데이터센터 이더넷 |
| iWARP | TCP/IP over Ethernet | 손실 있는 네트워크 지원, 레이턴시 다소 높음 |

---

## 핵심 개념: Verbs API

RDMA 프로그래밍의 표준 인터페이스는 **InfiniBand Verbs API**이며, `libibverbs` 라이브러리로 제공된다.

### 주요 개념

- **Protection Domain (PD)**: 메모리 영역과 큐 쌍을 묶는 보안 컨텍스트
- **Memory Region (MR)**: RDMA 전송을 위해 NIC에 등록된 메모리 영역. 가상→물리 주소 매핑이 NIC에 고정된다(pin).
- **Queue Pair (QP)**: 각 연결을 위한 Send Queue(SQ)와 Receive Queue(RQ)의 쌍
- **Completion Queue (CQ)**: 전송 완료 이벤트가 쌓이는 큐
- **Work Request (WR)**: 전송 요청 디스크립터. 메모리 주소, 키, 길이를 포함한다.

### RDMA 연산 종류

| 연산 | 설명 |
|-----|------|
| Send/Recv | 양쪽 모두 개입하는 양방향 메시지 |
| RDMA Write | 원격 메모리에 데이터를 씀. 수신측 CPU 개입 없음 |
| RDMA Read | 원격 메모리에서 데이터를 읽음. 수신측 CPU 개입 없음 |
| Atomic | 원격 메모리의 원자적 CAS/Fetch-Add |

One-sided 연산(Write/Read)이 RDMA의 핵심이다. 수신측 CPU가 깨어나지 않아도 된다.

---

## 실제 구현 예제 1: RDMA 연결 수립 (서버 측)

다음은 `libibverbs`와 `librdmacm`을 이용해 RDMA 서버를 초기화하고 연결을 받는 코드다.

```c
#include <infiniband/verbs.h>
#include <rdma/rdma_cma.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define BUFFER_SIZE 4096
#define PORT        "7471"

struct rdma_context {
    struct rdma_cm_id     *cm_id;
    struct ibv_pd         *pd;
    struct ibv_mr         *mr;
    struct ibv_cq         *cq;
    struct ibv_qp         *qp;
    char                  *buf;
};

/* Memory Region 등록: NIC에 가상-물리 주소 매핑을 고정 */
static int setup_mr(struct rdma_context *ctx) {
    ctx->buf = aligned_alloc(4096, BUFFER_SIZE);
    if (!ctx->buf) return -1;

    /* IBV_ACCESS_LOCAL_WRITE | IBV_ACCESS_REMOTE_WRITE:
       로컬 쓰기 + 원격 쓰기 모두 허용 */
    ctx->mr = ibv_reg_mr(ctx->pd, ctx->buf, BUFFER_SIZE,
                         IBV_ACCESS_LOCAL_WRITE |
                         IBV_ACCESS_REMOTE_WRITE |
                         IBV_ACCESS_REMOTE_READ);
    if (!ctx->mr) {
        free(ctx->buf);
        return -1;
    }
    printf("[Server] MR registered: addr=%p, rkey=0x%x\n",
           ctx->buf, ctx->mr->rkey);
    return 0;
}

/* Queue Pair 생성 */
static int create_qp(struct rdma_context *ctx) {
    struct ibv_qp_init_attr qp_attr = {
        .send_cq       = ctx->cq,
        .recv_cq       = ctx->cq,
        .cap = {
            .max_send_wr  = 16,
            .max_recv_wr  = 16,
            .max_send_sge = 1,
            .max_recv_sge = 1,
        },
        .qp_type = IBV_QPT_RC,  /* Reliable Connected */
    };
    return rdma_create_qp(ctx->cm_id, ctx->pd, &qp_attr);
}

int main(void) {
    struct rdma_event_channel *ec = rdma_create_event_channel();
    struct rdma_cm_id *listener = NULL;
    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port   = htons(7471),
        .sin_addr.s_addr = INADDR_ANY,
    };

    rdma_create_id(ec, &listener, NULL, RDMA_PS_TCP);
    rdma_bind_addr(listener, (struct sockaddr *)&addr);
    rdma_listen(listener, 1);

    printf("[Server] Listening on port %s ...\n", PORT);

    struct rdma_cm_event *event;
    rdma_get_cm_event(ec, &event);  /* RDMA_CM_EVENT_CONNECT_REQUEST */

    struct rdma_context ctx = { .cm_id = event->id };
    rdma_ack_cm_event(event);

    /* Protection Domain, CQ, MR, QP 초기화 */
    ctx.pd  = ibv_alloc_pd(ctx.cm_id->verbs);
    ctx.cq  = ibv_create_cq(ctx.cm_id->verbs, 16, NULL, NULL, 0);
    setup_mr(&ctx);
    create_qp(&ctx);

    /* 연결 수락: rkey와 주소를 클라이언트에 전달 (private_data 활용) */
    struct rdma_conn_param conn_param = { .responder_resources = 1 };
    rdma_accept(ctx.cm_id, &conn_param);

    printf("[Server] Connection established. Waiting for RDMA Write...\n");

    /* Completion Queue에서 이벤트 폴링 */
    struct ibv_wc wc;
    while (ibv_poll_cq(ctx.cq, 1, &wc) == 0) { /* spin */ }
    if (wc.status == IBV_WC_SUCCESS)
        printf("[Server] Received data: %s\n", ctx.buf);

    /* 정리 */
    ibv_dereg_mr(ctx.mr);
    ibv_destroy_qp(ctx.qp);
    ibv_destroy_cq(ctx.cq);
    ibv_dealloc_pd(ctx.pd);
    free(ctx.buf);
    return 0;
}
```

---

## 실제 구현 예제 2: RDMA Write 연산 (클라이언트 측)

클라이언트가 서버의 메모리에 직접 데이터를 쓰는 One-sided 연산이다. 서버의 CPU는 이 과정에 개입하지 않는다.

```c
#include <infiniband/verbs.h>
#include <rdma/rdma_cma.h>
#include <stdio.h>
#include <string.h>

/* 서버로부터 받은 원격 메모리 정보 */
struct remote_mem_info {
    uint64_t addr;   /* 원격 버퍼의 가상 주소 */
    uint32_t rkey;   /* Remote Key: 원격 쓰기 권한 검증용 */
};

int rdma_write_to_remote(struct rdma_cm_id *cm_id,
                         struct ibv_mr     *local_mr,
                         const char        *data,
                         size_t             len,
                         struct remote_mem_info *remote) {
    /* Scatter-Gather Element: 로컬 버퍼 기술자 */
    struct ibv_sge sge = {
        .addr   = (uint64_t)local_mr->addr,
        .length = (uint32_t)len,
        .lkey   = local_mr->lkey,  /* Local Key */
    };

    /* Send Work Request: RDMA_WRITE */
    struct ibv_send_wr wr = {
        .wr_id      = 1,
        .sg_list    = &sge,
        .num_sge    = 1,
        .opcode     = IBV_WR_RDMA_WRITE,
        /* SIGNALED: 완료 시 CQ에 이벤트 생성 */
        .send_flags = IBV_SEND_SIGNALED,
        .wr.rdma = {
            .remote_addr = remote->addr,  /* 원격 주소 */
            .rkey        = remote->rkey,  /* 원격 권한 키 */
        },
    };

    /* 로컬 버퍼에 데이터 복사 */
    memcpy(local_mr->addr, data, len);

    struct ibv_send_wr *bad_wr;
    int ret = ibv_post_send(cm_id->qp, &wr, &bad_wr);
    if (ret) {
        perror("ibv_post_send");
        return ret;
    }

    /* Completion Queue 폴링으로 전송 완료 확인 */
    struct ibv_wc wc;
    struct ibv_cq *cq = cm_id->qp->send_cq;

    /* 프로덕션에서는 ibv_req_notify_cq + event channel로 교체 */
    while (ibv_poll_cq(cq, 1, &wc) == 0) { /* busy-wait */ }

    if (wc.status != IBV_WC_SUCCESS) {
        fprintf(stderr, "RDMA Write failed: %s\n",
                ibv_wc_status_str(wc.status));
        return -1;
    }

    printf("[Client] RDMA Write completed: %zu bytes → remote 0x%lx\n",
           len, remote->addr);
    return 0;
}
```

위 코드에서 핵심은 `ibv_post_send`를 호출하는 순간 NIC 하드웨어가 DMA를 수행하고, CPU는 `ibv_poll_cq`를 호출하기 전까지 자유롭게 다른 작업을 할 수 있다는 점이다.

---

## 성능 특성과 주의사항

### Memory Pinning 비용

`ibv_reg_mr`은 메모리 페이지를 물리 메모리에 고정(pin)한다. 이는 OS가 해당 페이지를 스왑 아웃하거나 재배치할 수 없음을 의미한다. 대규모 버퍼를 자주 등록/해제하면 TLB Shootdown 오버헤드가 발생한다. 일반적으로 시작 시 메모리 풀을 미리 등록해두고 재사용하는 방식이 권장된다.

### Busy Polling vs. Event-Driven

`ibv_poll_cq`를 반복 호출하는 Busy Polling은 지연시간을 최소화하지만 CPU를 100% 점유한다. 레이턴시가 핵심인 금융 거래 시스템에서는 Busy Polling이 일반적이다. 처리량이 목표인 스토리지 서비스에서는 `ibv_req_notify_cq`로 완료 이벤트를 등록하고 인터럽트를 기다리는 방식이 CPU 효율적이다.

### 흐름 제어: RoCE의 PFC 의존성

RoCE v2는 손실 없는 이더넷(Lossless Ethernet)을 전제한다. 패킷이 드롭되면 QP 상태가 ERROR로 전환되어 연결이 끊긴다. 이를 방지하려면 스위치에 PFC(Priority Flow Control)와 ECN(Explicit Congestion Notification)을 활성화해야 한다. 클라우드 환경처럼 PFC를 보장할 수 없는 네트워크에서는 DCQCN 알고리즘을 사용하거나 iWARP로 전환을 고려해야 한다.

### 보안 고려사항

RDMA Write/Read는 원격 메모리에 직접 접근한다. `rkey`가 탈취되면 공격자가 임의 데이터를 쓸 수 있다. 따라서 RDMA는 신뢰할 수 있는 내부 네트워크(데이터센터 패브릭)에서만 사용하는 것이 원칙이다. 최신 RoCE 구현은 `rkey` 교환 시 TLS 암호화를 추가로 적용한다.

### 일반적인 아키텍처 팁

1. **메시지 크기별 전략**: 64B 미만의 소형 메시지는 Send/Recv + Inline Data가 유리하고, 4KB 이상의 대형 데이터는 RDMA Write/Read가 압도적이다.
2. **Doorbell Batching**: 여러 WR를 쌓아두고 `ibv_post_send`를 한 번에 호출하면 PCIe 대역폭을 효율적으로 사용할 수 있다.
3. **메모리 레이아웃**: 로컬 버퍼를 NUMA 노드와 NIC가 같은 쪽에 배치하면 DMA 레이턴시가 줄어든다(NUMA-aware allocation).

---

## 참고 자료

- [animeshtrivedi/rdma-example — 주석이 풍부한 RDMA Write/Read 구현 예제](https://github.com/animeshtrivedi/rdma-example)
- [jcxue/RDMA-Tutorial — 4단계 예제 기반 RDMA 프로그래밍 튜토리얼](https://github.com/jcxue/RDMA-Tutorial)
