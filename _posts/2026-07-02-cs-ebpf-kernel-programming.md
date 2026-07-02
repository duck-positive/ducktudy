---
layout: post
title: "eBPF 완전 정복: 커널 재컴파일 없이 Linux 커널을 프로그래밍하는 혁명적 기술"
date: 2026-07-02
categories: [cs, computer-science]
tags: [ebpf, linux, kernel, networking, observability, xdp, bpf, tracing, security]
---

## eBPF란 무엇인가?

eBPF(extended Berkeley Packet Filter)는 **커널 소스 코드를 변경하거나 커널 모듈을 로드하지 않고도 Linux 커널 내에서 샌드박스 프로그램을 실행**할 수 있게 해주는 혁명적인 기술입니다. 원래 BPF는 1992년 McCanne과 Jacobson이 발표한 네트워크 패킷 필터링 논문에서 유래했습니다. Linux 3.18(2014)에서 확장된 eBPF가 등장하면서 단순한 패킷 필터를 넘어 **범용 커널 프로그래밍 플랫폼**으로 진화했습니다.

eBPF의 위치는 운영체제 관점에서 매우 독특합니다. 애플리케이션 코드는 유저스페이스에서 실행되고, 커널 코드는 커널 공간에서 실행됩니다. eBPF 프로그램은 **커널 공간에서 실행되지만**, 검증기(verifier)가 안전성을 보장하므로 커널 모듈처럼 시스템을 불안정하게 만들 위험이 없습니다.

Meta(Facebook), Google, Netflix, Cloudflare, Cilium 프로젝트 등이 eBPF를 프로덕션 환경에서 적극적으로 활용하고 있습니다.

## 왜 eBPF가 필요한가?

### 전통적 접근법의 한계

커널 내부 동작을 관찰하거나 수정하는 전통적인 방법은 두 가지였습니다:

**커널 모듈 (Kernel Module)**
- 장점: 커널 전체에 완전한 접근 권한
- 단점: 버그 시 커널 패닉, 커널 버전마다 재컴파일 필요, 보안 위험 큼

**ptrace / strace 계열 도구**
- 장점: 유저스페이스에서 안전하게 시스템 콜 추적
- 단점: 오버헤드가 크고(syscall 인터셉트마다 컨텍스트 전환 2회), 커널 내부 함수 추적 불가

eBPF는 두 방식의 장점만 취했습니다: **커널 내부 접근 + 안전한 샌드박스 실행**.

### eBPF의 주요 사용 사례

| 분야 | 도구/프레임워크 | 설명 |
|------|----------------|------|
| 관찰 가능성 | BCC, bpftrace, Pixie | CPU 프로파일링, 레이턴시 히스토그램, 함수 추적 |
| 네트워킹 | Cilium, Katran | 쿠버네티스 네트워킹, L4 로드밸런서 |
| 보안 | Falco, Tetragon | 런타임 위협 탐지, 시스템 콜 기반 정책 |
| 성능 | XDP, AF_XDP | 패킷을 NIC 드라이버 레벨에서 처리 |

## eBPF 프로그램의 생명 주기

eBPF 프로그램은 다음 단계를 거쳐 실행됩니다:

```
1. C 코드 작성 (restricted C subset)
        ↓
2. clang -target bpf 로 eBPF 바이트코드(.o) 컴파일
        ↓
3. sys_bpf() 시스템 콜로 커널에 로드
        ↓
4. 커널 Verifier가 안전성 검증
   - DAG(방향 비순환 그래프) 검사로 무한 루프 금지
   - 포인터 경계 검사 (out-of-bounds 메모리 접근 불허)
   - 미초기화 메모리 사용 금지
   - 스택 깊이 512바이트 제한
        ↓
5. JIT 컴파일로 x86_64 네이티브 코드로 변환
        ↓
6. Hook 포인트에 부착
   - kprobe/kretprobe: 커널 함수 진입/반환
   - tracepoint: 커널 정적 추적 지점
   - XDP: 네트워크 드라이버 레벨
   - TC (Traffic Control): 네트워크 스케줄러
   - uprobe: 유저스페이스 함수 추적
        ↓
7. BPF Maps로 유저스페이스와 데이터 교환
```

### BPF Maps: 커널 ↔ 유저스페이스 통신

BPF Maps는 eBPF 프로그램이 데이터를 저장하고 유저스페이스와 공유하는 **키-값 데이터 구조**입니다. 타입에 따라 해시맵, 배열, 링 버퍼, 페르캐퍼(per-CPU) 등 다양한 형태가 있습니다.

## 실제 구현 예제

### 예제 1: BCC Python으로 시스템 콜 레이턴시 추적

BCC(BPF Compiler Collection)는 Python/Lua 프론트엔드로 eBPF 프로그램을 작성하게 해주는 가장 쉬운 진입점입니다.

```python
#!/usr/bin/env python3
"""
openat() 시스템 콜의 레이턴시를 마이크로초 단위 히스토그램으로 출력.
실행: sudo python3 syscall_latency.py
필요 패키지: bcc (sudo apt install bpfcc-tools python3-bpfcc)
"""
from bcc import BPF
import time

# eBPF C 프로그램 (커널 공간에서 실행)
BPF_PROGRAM = r"""
#include <uapi/linux/ptrace.h>

// BPF Hash Map: pid -> 진입 시각 (nanoseconds)
BPF_HASH(start_time, u32, u64);

// Histogram: 레이턴시 분포 (2^n 버킷)
BPF_HISTOGRAM(latency_hist, u64);

// kprobe: openat 시스템 콜 진입 시 실행
int kprobe__sys_openat(struct pt_regs *ctx)
{
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    u64 ts  = bpf_ktime_get_ns();
    start_time.update(&pid, &ts);
    return 0;
}

// kretprobe: openat 시스템 콜 반환 시 실행
int kretprobe__sys_openat(struct pt_regs *ctx)
{
    u32 pid  = bpf_get_current_pid_tgid() >> 32;
    u64 *tsp = start_time.lookup(&pid);
    if (!tsp) return 0;  // 진입 기록 없으면 스킵

    u64 delta_us = (bpf_ktime_get_ns() - *tsp) / 1000;  // ns -> us
    start_time.delete(&pid);
    latency_hist.increment(bpf_log2l(delta_us));  // log2 버킷에 기록
    return 0;
}
"""

b = BPF(text=BPF_PROGRAM)
print("openat() 레이턴시 추적 중... Ctrl+C로 종료 후 히스토그램 출력")

try:
    time.sleep(10)
except KeyboardInterrupt:
    pass

print("\nopenat() 레이턴시 히스토그램 (마이크로초):")
b["latency_hist"].print_log2_hist("latency (us)")
# 출력 예시:
#      latency (us)  : count    distribution
#          0 -> 1    : 12432   |████████████████████|
#          2 -> 3    : 3801    |██████              |
#          4 -> 7    : 892     |█                   |
#          8 -> 15   : 234     |                    |
#         16 -> 31   : 45      |                    |
```

### 예제 2: libbpf + C로 XDP 패킷 드롭 방화벽 구현

XDP(eXpress Data Path)는 **NIC 드라이버 레벨에서 패킷을 처리**하는 가장 빠른 eBPF 훅입니다. 패킷이 커널 네트워크 스택에 진입하기 전에 처리하므로 iptables보다 10~100배 빠른 패킷 필터링이 가능합니다.

```c
/* xdp_drop_ip.bpf.c — 특정 소스 IP 블록 XDP 프로그램 */
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>
#include <arpa/inet.h>

/* BPF Hash Map: 차단할 IP 주소 (key=ip, value=1) */
struct {
    __uint(type,       BPF_MAP_TYPE_HASH);
    __uint(max_entries, 1024);
    __type(key,        __u32);  /* IPv4 주소 (network byte order) */
    __type(value,      __u8);
} blocked_ips SEC(".maps");

/* XDP 프로그램 진입점 */
SEC("xdp")
int xdp_firewall(struct xdp_md *ctx)
{
    void *data     = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    /* 이더넷 헤더 파싱 */
    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return XDP_PASS;  /* 너무 짧은 패킷은 통과 */

    if (bpf_ntohs(eth->h_proto) != ETH_P_IP)
        return XDP_PASS;  /* IPv4가 아니면 통과 */

    /* IP 헤더 파싱 */
    struct iphdr *iph = (void *)(eth + 1);
    if ((void *)(iph + 1) > data_end)
        return XDP_PASS;

    /* 차단 목록 조회 */
    __u32 src_ip = iph->saddr;
    __u8 *blocked = bpf_map_lookup_elem(&blocked_ips, &src_ip);
    if (blocked)
        return XDP_DROP;  /* 드롭: NIC에서 즉시 폐기 */

    return XDP_PASS;  /* 패스: 커널 네트워크 스택으로 전달 */
}

char LICENSE[] SEC("license") = "GPL";
```

유저스페이스 제어 프로그램 (Python + ctypes):

```python
#!/usr/bin/env python3
"""
xdp_controller.py — XDP 방화벽 로드 및 IP 차단 관리
"""
import subprocess
import socket
import struct
import ctypes

def ip_to_int(ip_str: str) -> int:
    """IP 문자열을 네트워크 바이트 오더 정수로 변환"""
    return struct.unpack("I", socket.inet_aton(ip_str))[0]

def load_xdp_program(interface: str, bpf_obj: str) -> None:
    """XDP 프로그램을 네트워크 인터페이스에 부착"""
    result = subprocess.run(
        ["ip", "link", "set", "dev", interface, "xdp", "obj", bpf_obj, "sec", "xdp"],
        capture_output=True, text=True
    )
    if result.returncode != 0:
        raise RuntimeError(f"XDP 로드 실패: {result.stderr}")
    print(f"XDP 프로그램이 {interface}에 부착되었습니다.")

def block_ip(map_path: str, ip: str) -> None:
    """BPF Map에 차단할 IP 추가"""
    ip_int = ip_to_int(ip)
    # bpftool map update 사용 (프로덕션에서는 libbpf API 직접 호출)
    result = subprocess.run(
        ["bpftool", "map", "update", "pinned", map_path,
         "key", "hex"] + [f"{(ip_int >> (i*8)) & 0xff:02x}" for i in range(4)] +
        ["value", "hex", "01"],
        capture_output=True, text=True
    )
    if result.returncode == 0:
        print(f"차단됨: {ip}")
    else:
        print(f"차단 실패: {result.stderr}")

def unload_xdp(interface: str) -> None:
    """XDP 프로그램 제거"""
    subprocess.run(["ip", "link", "set", "dev", interface, "xdp", "off"])
    print(f"{interface}에서 XDP 제거 완료")

# 사용 예시
if __name__ == "__main__":
    IFACE = "eth0"
    MAP_PATH = "/sys/fs/bpf/blocked_ips"

    # XDP 프로그램 로드 (root 권한 필요)
    # load_xdp_program(IFACE, "xdp_drop_ip.bpf.o")

    # 악성 IP 차단
    malicious_ips = ["192.168.1.100", "10.0.0.254", "172.16.5.33"]
    for ip in malicious_ips:
        print(f"IP 차단 추가: {ip} → 정수값 {ip_to_int(ip):#010x}")
        # block_ip(MAP_PATH, ip)  # 실제 환경에서 활성화

    print(f"\n총 {len(malicious_ips)}개 IP 차단 설정 완료")
    print("XDP는 NIC 드라이버 레벨에서 처리하므로 iptables 대비 CPU 오버헤드 최소화")
```

## 주의사항과 실전 팁

### 1. Verifier의 엄격한 제약

eBPF 코드는 C이지만 **제한된 서브셋**만 사용 가능합니다:

- **반복문**: Linux 5.3+에서 제한적 허용, 이전 버전은 루프 전개(loop unrolling) 필요
- **함수 호출**: 허용된 BPF 헬퍼 함수만 호출 가능 (`bpf_trace_printk`, `bpf_map_lookup_elem` 등)
- **포인터 산술**: verifier가 추적하는 포인터만 허용, 범위 검사 필수
- **스택 크기**: 512바이트 제한 → 대용량 데이터는 반드시 BPF Maps 사용

### 2. 커널 버전 호환성

eBPF 기능은 커널 버전마다 점진적으로 추가됩니다. CO-RE(Compile Once, Run Everywhere) 기술과 BTF(BPF Type Format)를 사용하면 **하나의 바이너리로 여러 커널 버전에서 실행**할 수 있습니다. libbpf 1.0+과 `vmlinux.h` 사용을 권장합니다.

### 3. 성능 특성

- XDP Native 모드: 패킷당 **수십 나노초** 처리 (커널 네트워크 스택 우회)
- kprobe: 함수 호출당 **100~300ns** 오버헤드 (인터럽트 기반)
- tracepoint: kprobe보다 안정적이고 오버헤드 낮음 (컴파일 시 삽입된 정적 지점)

### 4. 디버깅 도구

```bash
# 로드된 BPF 프로그램 목록
sudo bpftool prog list

# BPF Map 내용 조회
sudo bpftool map dump id <map_id>

# BPF 프로그램의 검증된 명령어 출력
sudo bpftool prog dump xlated id <prog_id>

# eBPF 프로그램 실시간 로그 (bpf_trace_printk 출력)
sudo cat /sys/kernel/debug/tracing/trace_pipe
```

### 5. 실무 권장 프레임워크

- **BCC**: 빠른 프로토타이핑, Python/Lua 프론트엔드
- **libbpf + CO-RE**: 프로덕션 배포, 커널 버전 독립적
- **Aya (Rust)**: 타입 안전성이 중요한 환경
- **bpftrace**: 원라이너 스크립트, `awk` 스타일의 간결한 문법

## 참고 자료

- [ebpf.io — 공식 eBPF 커뮤니티 포털](https://ebpf.io/)
- [Linux Kernel BPF 문서](https://www.kernel.org/doc/html/latest/bpf/index.html)
- [eBPF Docs — 심화 기술 레퍼런스](https://docs.ebpf.io/linux/)
- [Wikipedia: eBPF](https://en.wikipedia.org/wiki/EBPF)
