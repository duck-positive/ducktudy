---
layout: post
title: "KVM 하이퍼바이저 가상화 내부 구조 완전 분석"
date: 2026-08-26
categories: [cs, computer-science]
tags: [virtualization, kvm, hypervisor, linux-kernel, qemu, vcpu, intel-vtx, amd-v]
---

## 가상화와 하이퍼바이저란 무엇인가

가상화(Virtualization)는 단일 물리 하드웨어 위에 여러 개의 독립적인 운영체제 인스턴스(가상 머신, VM)를 동시에 실행하는 기술입니다. 각 VM은 자신이 전용 하드웨어를 사용하는 것처럼 동작하지만, 실제로는 하이퍼바이저(Hypervisor)라 불리는 소프트웨어 계층이 물리 자원을 분할·중개합니다.

### 하이퍼바이저 유형

**Type 1 (Bare-metal Hypervisor)**: 하드웨어 위에 직접 설치되어 동작. OS 없이 하이퍼바이저 자체가 자원을 관리. Xen, VMware ESXi, Hyper-V, KVM이 여기에 해당. 낮은 오버헤드와 높은 성능이 특징.

**Type 2 (Hosted Hypervisor)**: 호스트 운영체제 위에서 일반 프로세스처럼 동작. VirtualBox, VMware Workstation, Parallels Desktop이 여기에 해당. 설치와 사용이 간편하지만 호스트 OS를 거치는 추가 오버헤드 존재.

**KVM의 특수성**: KVM(Kernel-based Virtual Machine)은 독특하게 **Linux 커널 자체를 Type 1 하이퍼바이저로 전환**합니다. Linux 커널 모듈로 동작하면서, 호스트 커널의 스케줄러·메모리 관리·디바이스 드라이버를 그대로 활용합니다. 2006년 개발되어 2007년 2월 Linux 2.6.20에 공식 병합되었습니다.

---

## 하드웨어 가상화 지원: Intel VT-x / AMD-V

현대 x86 가상화의 핵심은 CPU의 하드웨어 가상화 확장입니다.

### Intel VT-x (Virtualization Technology for IA-32/64)

Intel VT-x는 CPU에 두 가지 실행 모드를 추가합니다.

- **VMX root mode**: 하이퍼바이저가 실행되는 모드. 전통적인 Ring 0–3 모두 접근 가능.
- **VMX non-root mode**: 게스트 VM이 실행되는 모드. Ring 0–3이 존재하지만 하이퍼바이저보다 낮은 권한.

**VMCS (Virtual Machine Control Structure)**: 각 vCPU마다 하나씩 존재하는 4KB 데이터 구조체. 게스트/호스트의 레지스터 상태, VM-entry/VM-exit 조건, I/O 비트맵 등을 저장합니다.

**주요 명령어**:
- `VMXON`: VMX 모드 진입
- `VMLAUNCH`: 새 VM 최초 실행
- `VMRESUME`: 중단된 VM 재시작
- `VMEXIT`: 게스트 → 호스트 전환 (예외, 인터럽트, I/O 등)
- `VMREAD/VMWRITE`: VMCS 필드 읽기/쓰기

### VM-exit 원인

게스트 VM이 특정 연산을 수행하면 VM-exit가 발생하여 하이퍼바이저로 제어가 넘어옵니다.

| 원인 | 설명 |
|------|------|
| `HLT` 명령 | 게스트 CPU 정지 시도 |
| I/O 포트 접근 | IN/OUT 명령으로 장치 접근 |
| EPT 위반 | 확장 페이지 테이블 오류 |
| CPUID | CPU 정보 쿼리 |
| MSR 접근 | Model-Specific Register 읽기/쓰기 |
| 인터럽트 | 외부 인터럽트 발생 |
| 예외 | 페이지 폴트 등 |

---

## 왜 KVM이 필요한가

### 성능과 격리의 균형

순수 소프트웨어 에뮬레이션(QEMU의 TCG 모드)은 모든 CPU 명령을 인터프리트하여 높은 오버헤드를 발생시킵니다. KVM은 하드웨어 가상화를 활용해 게스트의 대부분 명령이 네이티브 속도로 실행되게 하며, 권한 명령어나 장치 접근만 VM-exit를 통해 처리합니다. 실제로 KVM + QEMU 조합은 네이티브 대비 2~5% 수준의 오버헤드만 발생합니다.

### 클라우드 컴퓨팅의 기반

AWS EC2, Google Compute Engine, Oracle Cloud 등 주요 퍼블릭 클라우드의 가상화 레이어는 KVM 기반입니다. 단일 물리 서버에서 수십 개의 VM을 효율적으로 실행할 수 있기 때문에 클라우드의 경제성을 가능하게 합니다.

---

## 구현 예제

### 예제 1: C — KVM API로 간단한 VM 생성 및 실행

KVM은 `/dev/kvm` 캐릭터 디바이스를 통해 ioctl() 인터페이스를 제공합니다.

```c
#include <linux/kvm.h>
#include <fcntl.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <stdint.h>

/* 간단한 16비트 실제 모드 게스트 코드: HLT 실행 */
static const uint8_t guest_code[] = {
    0xf4,   /* HLT */
};

int main(void) {
    /* 1. KVM 서브시스템 파일 디스크립터 획득 */
    int kvm = open("/dev/kvm", O_RDWR | O_CLOEXEC);
    if (kvm < 0) { perror("open /dev/kvm"); return 1; }

    /* API 버전 확인 (반드시 12여야 함) */
    int ver = ioctl(kvm, KVM_GET_API_VERSION, NULL);
    if (ver != 12) { fprintf(stderr, "KVM API 버전 불일치: %d\n", ver); return 1; }

    /* 2. 가상 머신 생성 */
    int vmfd = ioctl(kvm, KVM_CREATE_VM, (unsigned long)0);
    if (vmfd < 0) { perror("KVM_CREATE_VM"); return 1; }

    /* 3. 게스트 물리 메모리 설정 (1MB, 0x0 ~ 0xFFFFF) */
    size_t mem_size = 1 << 20;   /* 1 MiB */
    void *mem = mmap(NULL, mem_size,
                     PROT_READ | PROT_WRITE,
                     MAP_SHARED | MAP_ANONYMOUS, -1, 0);
    if (mem == MAP_FAILED) { perror("mmap"); return 1; }

    /* 게스트 코드를 물리 주소 0x0에 배치 */
    memcpy(mem, guest_code, sizeof(guest_code));

    struct kvm_userspace_memory_region region = {
        .slot            = 0,
        .flags           = 0,
        .guest_phys_addr = 0x0000,       /* 게스트가 볼 물리 주소 시작 */
        .memory_size     = mem_size,
        .userspace_addr  = (uintptr_t)mem,
    };
    if (ioctl(vmfd, KVM_SET_USER_MEMORY_REGION, &region) < 0) {
        perror("KVM_SET_USER_MEMORY_REGION"); return 1;
    }

    /* 4. vCPU 생성 */
    int vcpufd = ioctl(vmfd, KVM_CREATE_VCPU, (unsigned long)0);
    if (vcpufd < 0) { perror("KVM_CREATE_VCPU"); return 1; }

    /* 5. kvm_run 구조체 매핑 (하이퍼바이저-유저스페이스 통신용) */
    size_t mmap_size = ioctl(kvm, KVM_GET_VCPU_MMAP_SIZE, NULL);
    struct kvm_run *run = mmap(NULL, mmap_size,
                               PROT_READ | PROT_WRITE,
                               MAP_SHARED, vcpufd, 0);

    /* 6. 초기 레지스터 설정 (16비트 실제 모드) */
    struct kvm_sregs sregs;
    ioctl(vcpufd, KVM_GET_SREGS, &sregs);
    sregs.cs.base  = 0;
    sregs.cs.selector = 0;
    ioctl(vcpufd, KVM_SET_SREGS, &sregs);

    struct kvm_regs regs = {
        .rip    = 0,       /* 실행 시작 주소 (물리 0x0) */
        .rflags = 0x2,     /* Reserved 비트 1 항상 세팅 */
    };
    ioctl(vcpufd, KVM_SET_REGS, &regs);

    /* 7. vCPU 실행 루프 */
    while (1) {
        int ret = ioctl(vcpufd, KVM_RUN, NULL);
        if (ret < 0) { perror("KVM_RUN"); break; }

        switch (run->exit_reason) {
            case KVM_EXIT_HLT:
                printf("VM-exit: HLT — 게스트가 정지했습니다\n");
                goto done;

            case KVM_EXIT_IO:
                printf("VM-exit: I/O port=0x%x, size=%d, direction=%s\n",
                       run->io.port, run->io.size,
                       run->io.direction == KVM_EXIT_IO_OUT ? "OUT" : "IN");
                break;

            case KVM_EXIT_MMIO:
                printf("VM-exit: MMIO addr=0x%llx, len=%d, write=%d\n",
                       run->mmio.phys_addr, run->mmio.len, run->mmio.is_write);
                break;

            case KVM_EXIT_FAIL_ENTRY:
                fprintf(stderr, "VM-entry 실패: hardware_entry_failure_reason=0x%llx\n",
                        run->fail_entry.hardware_entry_failure_reason);
                goto done;

            default:
                fprintf(stderr, "알 수 없는 VM-exit: %d\n", run->exit_reason);
                goto done;
        }
    }
done:
    munmap(run, mmap_size);
    munmap(mem, mem_size);
    close(vcpufd);
    close(vmfd);
    close(kvm);
    return 0;
}
```

### 예제 2: Python — libvirt API로 KVM VM 생성 및 모니터링

실제 운영 환경에서는 KVM을 직접 제어하기보다 libvirt 추상화 계층을 사용합니다.

```python
import libvirt
import sys
import time

DOMAIN_XML = """
<domain type='kvm'>
  <name>my-vm</name>
  <uuid>12345678-1234-1234-1234-123456789abc</uuid>
  <memory unit='MiB'>2048</memory>
  <currentMemory unit='MiB'>2048</currentMemory>
  <vcpu placement='static'>2</vcpu>
  <os>
    <type arch='x86_64' machine='pc-i440fx-6.2'>hvm</type>
    <boot dev='hd'/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <vmport state='off'/>
  </features>
  <cpu mode='host-passthrough' check='none' migratable='on'/>
  <clock offset='utc'>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
  </clock>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='none' io='native'/>
      <source file='/var/lib/libvirt/images/my-vm.qcow2'/>
      <target dev='vda' bus='virtio'/>
    </disk>
    <interface type='network'>
      <source network='default'/>
      <model type='virtio'/>
    </interface>
    <console type='pty'>
      <target type='serial' port='0'/>
    </console>
    <channel type='unix'>
      <target type='virtio' name='org.qemu.guest_agent.0'/>
    </channel>
    <memballoon model='virtio'>
      <stats period='5'/>
    </memballoon>
  </devices>
</domain>
"""

STATE_NAMES = {
    libvirt.VIR_DOMAIN_NOSTATE:  "nostate",
    libvirt.VIR_DOMAIN_RUNNING:  "running",
    libvirt.VIR_DOMAIN_BLOCKED:  "blocked",
    libvirt.VIR_DOMAIN_PAUSED:   "paused",
    libvirt.VIR_DOMAIN_SHUTDOWN: "shutdown",
    libvirt.VIR_DOMAIN_SHUTOFF:  "shutoff",
    libvirt.VIR_DOMAIN_CRASHED:  "crashed",
}

def create_and_monitor():
    # QEMU/KVM 연결 (로컬 시스템)
    conn = libvirt.open("qemu:///system")
    if conn is None:
        print("libvirt 연결 실패", file=sys.stderr)
        return

    # VM 정의 및 시작
    try:
        dom = conn.defineXML(DOMAIN_XML)
        dom.create()
        print(f"VM '{dom.name()}' 시작됨")
    except libvirt.libvirtError as e:
        print(f"VM 생성 오류: {e}", file=sys.stderr)
        conn.close()
        return

    # 5초 간격으로 상태 모니터링
    for _ in range(3):
        time.sleep(5)

        state, _ = dom.state()
        print(f"\n=== VM 상태: {STATE_NAMES.get(state, 'unknown')} ===")

        # CPU 통계
        try:
            cpu_stats = dom.getCPUStats(True)  # True = 전체 vCPU 합산
            cpu_time_ns = cpu_stats[0].get('cpu_time', 0)
            print(f"CPU 시간: {cpu_time_ns / 1e9:.3f}초")
        except libvirt.libvirtError:
            pass

        # 메모리 통계
        try:
            mem = dom.memoryStats()
            rss = mem.get('rss', 0)
            actual = mem.get('actual', 0)
            print(f"메모리: 실사용 {rss//1024}MiB / 할당 {actual//1024}MiB")
        except libvirt.libvirtError:
            pass

        # 블록 I/O 통계 (첫 번째 디스크)
        try:
            block = dom.blockStats('vda')
            rd_bytes, wr_bytes = block[1], block[3]
            print(f"디스크 I/O: 읽기 {rd_bytes//1024}KiB / 쓰기 {wr_bytes//1024}KiB")
        except libvirt.libvirtError:
            pass

    # VM 종료
    dom.destroy()
    dom.undefine()
    conn.close()
    print("\nVM 정리 완료")

if __name__ == "__main__":
    create_and_monitor()
```

---

## KVM 내부 아키텍처 심층 이해

### 메모리 가상화: EPT (Extended Page Tables)

전통적인 섀도 페이지 테이블(Shadow Page Table) 방식에서 하이퍼바이저는 게스트 가상 주소 → 호스트 물리 주소 변환을 직접 관리해야 했습니다. Intel EPT(AMD의 경우 NPT, Nested Page Tables)는 CPU가 두 단계의 페이지 테이블 순회를 하드웨어에서 처리합니다.

```
게스트 가상 주소 (GVA)
    → [게스트 페이지 테이블] → 게스트 물리 주소 (GPA)
    → [EPT 페이지 테이블]    → 호스트 물리 주소 (HPA)
```

EPT miss 시 EPT violation VM-exit가 발생하고 KVM이 새 EPT 엔트리를 채웁니다.

### vCPU 스케줄링

KVM에서 각 vCPU는 호스트 OS의 일반 스레드처럼 스케줄됩니다. `kvm_vcpu_run()` 함수가 `VMLAUNCH`/`VMRESUME`를 실행하고, VM-exit 후에는 다시 호스트 스케줄러로 돌아옵니다. 이 덕분에 Linux의 CFS(Completely Fair Scheduler)가 VM 간 공정한 CPU 배분을 담당합니다.

### I/O 가상화

KVM은 I/O 가상화를 주로 유저스페이스(QEMU)에 위임합니다.

- **전통적 에뮬레이션**: I/O 접근 시 VM-exit → QEMU에서 하드웨어 에뮬레이션 (e.g. virtio)
- **SR-IOV (Single Root I/O Virtualization)**: 물리 NIC·GPU를 여러 VF(Virtual Function)로 분할하여 게스트에 직접 할당. IOMMU(Intel VT-d / AMD-Vi) 필요.
- **vhost-net**: KVM 커널 모듈 내에서 네트워크 I/O를 직접 처리하여 유저스페이스 전환 비용 절감.

---

## 주의사항 및 운영 팁

**Nested Virtualization (중첩 가상화)**: KVM 위에 KVM을 실행하는 것은 가능하지만(Intel의 경우 `kvm-intel.ko`의 `nested=1` 파라미터), 성능 오버헤드가 상당합니다. 테스트 환경에만 권장.

**CPU Pinning**: 레이턴시에 민감한 워크로드는 vCPU를 특정 물리 코어에 고정(CPU Pinning)하고 NUMA 토폴로지에 맞게 구성해야 합니다. `virsh vcpupin` 명령으로 설정 가능.

**보안 격리**: Spectre/Meltdown 이후 KVM에는 다양한 완화책이 추가되었습니다. `retpoline`, IBRS/IBPB 등의 CPU 마이크로코드 패치와 KVM의 `spec_ctrl` 지원을 반드시 확인하세요. VM 간 사이드 채널 공격의 위험이 여전히 연구되고 있습니다.

**Live Migration**: VM을 다운타임 없이 다른 호스트로 이동하는 기술. KVM은 `KVM_CAP_MIGRATION_PD_NOTIFY`와 같은 기능으로 메모리 더티 트래킹을 지원합니다. 마이그레이션 중 발생하는 메모리 변경을 추적하여 최소한의 데이터만 전송합니다.

**Huge Pages**: 일반 4KB 페이지 대신 2MB Huge Page를 사용하면 TLB 미스가 크게 줄어 VM 성능이 향상됩니다. `/sys/kernel/mm/hugepages/` 설정 후 libvirt XML에서 `<memoryBacking><hugepages/></memoryBacking>` 추가.

---

## 참고 자료

- [KVM — The Linux Kernel documentation](https://docs.kernel.org/virt/kvm/index.html)
- [KVM API Documentation (kernel.org)](https://www.kernel.org/doc/html/v6.3/virt/kvm/api.html)
- [Linux KVM Official Site](https://linux-kvm.org/page/Main_Page)
- [Kernel-based Virtual Machine - Wikipedia](https://en.wikipedia.org/wiki/Kernel-based_Virtual_Machine)
