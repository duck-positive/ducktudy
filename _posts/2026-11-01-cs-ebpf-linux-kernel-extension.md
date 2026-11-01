---
layout: post
title: "eBPF 완전 정복: 리눅스 커널을 안전하게 확장하는 혁명적 기술"
date: 2026-11-01
categories: [cs, computer-science]
tags: [ebpf, linux, kernel, networking, observability, security, bpf]
---

## eBPF란 무엇인가?

eBPF(Extended Berkeley Packet Filter)는 커널 소스 코드를 수정하거나 커널 모듈을 로드하지 않고도 리눅스 커널 내부에서 임의의 프로그램을 안전하게 실행할 수 있게 해주는 혁명적인 기술입니다. 원래 1992년에 네트워크 패킷 필터링을 위해 설계된 BPF(Berkeley Packet Filter)가 그 뿌리이지만, 2014년 리눅스 3.18에서 대폭 확장된 eBPF로 진화하면서 커널 관찰성(observability), 네트워킹, 보안, 성능 분석 등 광범위한 영역에서 쓰이고 있습니다.

eBPF 프로그램은 커널 이벤트(시스템 콜, 네트워크 패킷 수신, 함수 진입/반환, 하드웨어 이벤트 등)에 훅(hook)되어 실행됩니다. 중요한 점은 eBPF 프로그램이 커널에 로드되기 전에 **검증기(Verifier)**가 안전성을 철저히 검사한다는 것입니다. 무한 루프, 메모리 경계 초과, 검증되지 않은 포인터 역참조 등을 컴파일 타임에 거부하므로, 커널 패닉 없이 사용자 정의 로직을 커널 공간에서 실행할 수 있습니다.

## 왜 eBPF가 필요한가?

전통적으로 커널을 확장하는 방법은 두 가지였습니다.

1. **커널 패치**: 리눅스 커널 소스를 수정하고 재컴파일하는 방법. 프로덕션 배포가 극히 어렵고 수개월의 검토 사이클이 필요합니다.
2. **커널 모듈(LKM)**: 동적으로 로드 가능하지만, 버그 하나가 전체 시스템을 크래시시킬 수 있어 매우 위험합니다.

eBPF는 이 두 방법의 단점을 모두 극복합니다. 커널 공간의 성능을 유지하면서도 **사용자 공간에서 프로그램을 개발·배포**할 수 있고, 검증기가 안전성을 보장합니다. Cloudflare, Meta, Netflix, Google 등 대형 기술 기업들이 DDoS 방어, 로드 밸런싱, 관찰성 플랫폼에 eBPF를 적극 활용하는 것이 이를 증명합니다.

### eBPF의 핵심 구성 요소

- **eBPF 프로그램**: C로 작성 후 LLVM/Clang으로 eBPF 바이트코드로 컴파일
- **검증기(Verifier)**: 프로그램의 안전성을 정적으로 분석
- **JIT 컴파일러**: 바이트코드를 네이티브 머신코드로 변환하여 성능 최적화
- **Maps**: eBPF 프로그램과 사용자 공간 간 데이터 공유를 위한 자료구조 (Hash, Array, Ring Buffer 등)
- **Helper Functions**: 커널이 제공하는 안전한 API (패킷 수정, 맵 조작, 시스템 정보 조회 등)

## 실제 구현 예제

### 예제 1: XDP를 이용한 패킷 카운터

XDP(eXpress Data Path)는 네트워크 드라이버 레벨에서 eBPF 프로그램을 실행하는 훅으로, 커널 네트워크 스택을 거치기 전에 패킷을 처리합니다.

```c
// packet_counter.bpf.c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>
#include <linux/if_ether.h>
#include <linux/ip.h>

// BPF Map: 프로토콜별 패킷 카운터
struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __uint(max_entries, 256);
    __type(key, __u32);
    __type(value, __u64);
} proto_count SEC(".maps");

SEC("xdp")
int xdp_packet_counter(struct xdp_md *ctx) {
    void *data_end = (void *)(long)ctx->data_end;
    void *data     = (void *)(long)ctx->data;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return XDP_PASS;

    if (eth->h_proto != __constant_htons(ETH_P_IP))
        return XDP_PASS;

    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end)
        return XDP_PASS;

    __u32 proto = ip->protocol;
    __u64 *count = bpf_map_lookup_elem(&proto_count, &proto);
    if (count)
        __sync_fetch_and_add(count, 1);  // 원자적 증가

    return XDP_PASS;  // 패킷을 계속 처리
}

char LICENSE[] SEC("license") = "GPL";
```

```python
# loader.py — 사용자 공간에서 BPF 프로그램을 로드하고 Map 읽기 (bcc 라이브러리 사용)
from bcc import BPF
import socket, time

prog = open("packet_counter.bpf.c").read()
b = BPF(text=prog)
fn = b.load_func("xdp_packet_counter", BPF.XDP)
b.attach_xdp("eth0", fn, 0)

proto_map = b["proto_count"]
proto_names = {6: "TCP", 17: "UDP", 1: "ICMP"}

print("프로토콜별 패킷 수 (10초 동안 측정):")
time.sleep(10)

for proto_num, name in proto_names.items():
    key = proto_map.Key(proto_num)
    try:
        val = proto_map[key].value
        print(f"  {name}: {val} 패킷")
    except KeyError:
        print(f"  {name}: 0 패킷")

b.remove_xdp("eth0", 0)
```

이 예제에서 XDP 프로그램은 네트워크 드라이버 레벨에서 실행되어, 패킷이 커널 TCP/IP 스택으로 올라가기 전에 프로토콜별 카운터를 증가시킵니다. `XDP_PASS`를 반환하면 패킷이 정상 처리되고, `XDP_DROP`을 반환하면 즉시 폐기됩니다.

---

### 예제 2: kprobe를 이용한 시스템 콜 추적

kprobe는 커널 함수의 진입/반환 지점에 eBPF 프로그램을 부착하는 메커니즘입니다.

```c
// syscall_trace.bpf.c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

struct event {
    __u32 pid;
    __u32 uid;
    char  comm[16];
    char  filename[64];
};

// Ring Buffer Map — 이벤트를 사용자 공간으로 효율적으로 전달
struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 256 * 1024);  // 256KB
} events SEC(".maps");

// open 시스템 콜 진입점 후킹
SEC("kprobe/do_sys_openat2")
int trace_openat(struct pt_regs *ctx) {
    struct event *e;
    e = bpf_ringbuf_reserve(&events, sizeof(*e), 0);
    if (!e)
        return 0;

    e->pid = bpf_get_current_pid_tgid() >> 32;
    e->uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;
    bpf_get_current_comm(e->comm, sizeof(e->comm));

    // 두 번째 인자(파일명 포인터)를 안전하게 읽기
    const char __user *fname = (const char *)(PT_REGS_PARM2(ctx));
    bpf_probe_read_user_str(e->filename, sizeof(e->filename), fname);

    bpf_ringbuf_submit(e, 0);
    return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

```bash
# bpftool로 프로그램 로드 및 상태 확인
$ bpftool prog load syscall_trace.bpf.o /sys/fs/bpf/trace_open
$ bpftool prog list
123: kprobe  name trace_openat  tag a1b2c3d4e5f6  gpl
        loaded_at 2026-11-01T10:00:00  uid 0
        xlated 312B  jited 248B  memlock 4096B  map_ids 5

# Ring Buffer에서 이벤트 폴링 (go-ebpf 또는 libbpf 사용 시 실제로는 라이브러리로 처리)
$ bpftool map dump id 5
```

Ring Buffer Map은 기존의 perf event array보다 CPU 캐시 효율이 높고, 이벤트 손실이 적어 높은 이벤트 빈도에서도 신뢰할 수 있습니다.

## eBPF의 주요 사용 사례

| 분야 | 도구/프로젝트 | 기능 |
|------|-------------|------|
| 네트워킹 | Cilium, Katran | L4 로드 밸런싱, 서비스 메시 |
| 보안 | Falco, Tetragon | 런타임 위협 탐지 |
| 관찰성 | BCC, bpftrace | CPU 프로파일링, syscall 추적 |
| 성능 | libbpf-bootstrap | 레이턴시 측정, 병목 분석 |

## 주의사항 및 팁

### 1. 검증기의 복잡성 이해
eBPF 검증기는 **정적 분석**을 수행하므로, 모든 코드 경로에서 메모리 접근이 유효해야 합니다. 포인터를 역참조하기 전에 반드시 범위 검사(`data + offset <= data_end`)를 해야 합니다. 이 검사를 누락하면 검증기가 프로그램 로드를 거부합니다.

### 2. 루프 제한
커널 5.3 이전에는 루프가 완전히 금지되었습니다. 5.3부터 bounded loop가 허용되지만, 검증기가 반복 횟수가 유한함을 증명할 수 있어야 합니다. 알 수 없는 횟수의 반복은 `bpf_loop()` 헬퍼를 사용해야 합니다.

### 3. 스택 크기 제한
eBPF 프로그램의 스택은 **512바이트**로 제한됩니다. 큰 구조체는 BPF Map에 저장하거나, `bpf_ringbuf_reserve()`로 동적 할당을 활용하세요.

### 4. CO-RE (Compile Once, Run Everywhere)
커널 버전마다 내부 구조체 레이아웃이 다릅니다. BTF(BPF Type Format)와 CO-RE를 활용하면 하나의 바이너리가 여러 커널 버전에서 동작합니다. libbpf 라이브러리가 런타임에 구조체 오프셋을 재배치합니다.

### 5. 성능 함정: Map 접근 비용
Map 조회(`bpf_map_lookup_elem`)는 해시 맵의 경우 ~100ns의 비용이 발생합니다. 고빈도 경로에서는 배열 맵이나 percpu 맵을 우선 고려하세요. percpu 맵은 CPU 로컬 데이터를 유지하므로 락이 필요 없어 훨씬 빠릅니다.

## 참고 자료
- [Linux Kernel BPF Documentation](https://www.kernel.org/doc/html/v5.10/bpf/index.html)
- [BPF Documentation (kernel 6.3)](https://docs.kernel.org/6.3/bpf/index.html)
- [eBPF Foundation](https://ebpf.foundation/?p=86)
- [BPF and XDP Reference Guide (Cilium)](https://docs.cilium.io/en/stable/bpf/)
