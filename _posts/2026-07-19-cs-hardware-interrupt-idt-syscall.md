---
layout: post
title: "하드웨어 인터럽트와 시스템 콜 완전 정복: IDT, APIC, syscall 내부 동작 원리"
date: 2026-07-19
categories: [cs, computer-science]
tags: [interrupt, hardware-interrupt, IDT, APIC, syscall, kernel, operating-system, x86]
---

## 개념 설명

CPU는 명령어를 순차적으로 실행하지만, 외부 세계(키보드, 네트워크 카드, 타이머)는 예측 불가능한 시점에 CPU의 주의를 요구합니다. **인터럽트(Interrupt)** 는 이런 비동기적 사건을 처리하는 핵심 메커니즘으로, 현대 운영체제가 다중 프로세스, I/O 처리, 실시간 응답성을 구현할 수 있는 토대입니다.

### 인터럽트의 종류

| 종류 | 발생 원인 | 예시 |
|------|----------|------|
| **하드웨어 인터럽트** | 외부 장치의 신호 | 키보드 입력, 네트워크 패킷 수신, 타이머 틱 |
| **소프트웨어 인터럽트** | 프로그램 실행 중 의도적 트리거 | `INT 0x80` (시스템 콜) |
| **예외(Exception)** | CPU가 명령 처리 중 오류 감지 | Division by zero, Page Fault, Invalid Opcode |

### 인터럽트 벡터 테이블: IDT (Interrupt Descriptor Table)

x86-64 아키텍처에서 인터럽트 처리 루틴의 주소를 저장하는 자료구조가 **IDT(Interrupt Descriptor Table)** 입니다.

- 256개의 엔트리(0번 ~ 255번)
- 각 엔트리는 16바이트의 **게이트 디스크립터(Gate Descriptor)**
- IDTR 레지스터가 IDT의 베이스 주소와 크기를 가리킴

```
IDT 레이아웃:
[0]   Divide Error (DE)
[1]   Debug Exception (DB)
[2]   NMI Interrupt
[3]   Breakpoint (BP) - INT 3
[4]   Overflow (OF)
[5]   BOUND Range Exceeded (BR)
[6]   Invalid Opcode (UD)
[7]   Device Not Available (NM)
[8]   Double Fault (DF)
...
[14]  Page Fault (PF)
...
[32~255] Hardware Interrupts & Software Interrupts
[128]   System Call Interface (Linux: INT 0x80 레거시)
```

### APIC: 인터럽트 컨트롤러

멀티코어 시스템에서 인터럽트 라우팅을 담당하는 것이 **APIC(Advanced Programmable Interrupt Controller)** 입니다.

- **Local APIC (LAPIC)**: 각 CPU 코어에 내장. 타이머 인터럽트, IPI(Inter-Processor Interrupt) 처리
- **I/O APIC**: 외부 장치 인터럽트를 받아 적절한 CPU 코어의 LAPIC에 전달

```
[키보드] → [I/O APIC] → [CPU 0의 LAPIC] → CPU 0이 인터럽트 처리
                      ↘ [CPU 1의 LAPIC] → (라우팅에 따라 다른 코어가 처리)
```

---

## 왜 필요한가?

### 폴링(Polling) vs 인터럽트

인터럽트 없이 I/O를 처리하는 가장 단순한 방법은 **폴링**입니다.

```c
// 폴링 방식: CPU가 계속 장치 상태를 확인
while (!(keyboard_status & KEY_READY)) {
    // CPU 사이클 낭비
}
char key = keyboard_data;
```

폴링은 구현이 단순하지만 CPU가 계속 루프를 돌아야 해서 낭비가 심합니다. 인터럽트 방식에서는 CPU가 다른 작업을 수행하다가 장치가 준비됐을 때만 알림을 받습니다.

### 인터럽트가 운영체제에서 하는 역할

1. **타이머 인터럽트 → 선점 스케줄링**: 주기적 타이머 인터럽트(일반적으로 1ms~10ms마다)가 발생하면 OS 스케줄러가 실행되어 CPU를 다른 프로세스에 넘깁니다.
2. **페이지 폴트 핸들러**: 가상 메모리가 물리 메모리에 없을 때 #PF 예외가 발생하고 OS가 디스크에서 페이지를 로드합니다.
3. **시스템 콜 게이트웨이**: 유저 프로세스가 커널 기능을 안전하게 사용하는 유일한 경로입니다.

---

## 실제 구현 예제

### 예제 1: x86-64 인터럽트 처리 흐름 (어셈블리 + C)

Linux 커널의 인터럽트 핸들러 등록 과정을 단순화한 예제입니다.

```c
/* 인터럽트 게이트 디스크립터 구조체 (x86-64) */
struct idt_entry {
    uint16_t offset_low;    // ISR 주소 [15:0]
    uint16_t selector;      // 코드 세그먼트 셀렉터
    uint8_t  ist;           // 인터럽트 스택 테이블 인덱스
    uint8_t  type_attr;     // 타입 + 권한 레벨
    uint16_t offset_mid;    // ISR 주소 [31:16]
    uint32_t offset_high;   // ISR 주소 [63:32]
    uint32_t reserved;
} __attribute__((packed));

struct idt_entry idt[256];
struct { uint16_t limit; uint64_t base; } __attribute__((packed)) idtr;

/* 인터럽트 게이트 설정 */
void set_idt_gate(int n, uint64_t handler_addr) {
    idt[n].offset_low  = handler_addr & 0xFFFF;
    idt[n].selector    = 0x08;          // 커널 코드 세그먼트
    idt[n].ist         = 0;
    idt[n].type_attr   = 0x8E;          // Present | DPL=0 | Interrupt Gate
    idt[n].offset_mid  = (handler_addr >> 16) & 0xFFFF;
    idt[n].offset_high = (handler_addr >> 32) & 0xFFFFFFFF;
    idt[n].reserved    = 0;
}

/* 키보드 ISR (Interrupt Service Routine) */
void keyboard_handler(void) {
    uint8_t scancode = inb(0x60);  // 키보드 데이터 포트에서 스캔코드 읽기
    handle_key(scancode);

    // EOI(End of Interrupt): 인터럽트 처리 완료를 APIC에 알림
    outb(0x20, 0x20);  // 마스터 PIC에 EOI 전송
}

void init_idt(void) {
    /* 키보드 인터럽트(IRQ 1) = 벡터 0x21 */
    set_idt_gate(0x21, (uint64_t)keyboard_handler);

    idtr.limit = sizeof(idt) - 1;
    idtr.base  = (uint64_t)idt;

    /* IDTR 레지스터에 IDT 주소 로드 */
    __asm__ volatile ("lidt %0" : : "m"(idtr));
}
```

인터럽트 발생 시 CPU가 자동으로 수행하는 동작:
```
1. 현재 실행 중인 명령 완료 후 인터럽트 체크
2. 권한 레벨 전환: Ring 3 (User) → Ring 0 (Kernel)
3. 스택 전환: TSS(Task State Segment)에서 커널 스택 주소 로드
4. SS, RSP, RFLAGS, CS, RIP 레지스터를 커널 스택에 push
5. IDT에서 벡터 번호에 해당하는 핸들러 주소 조회
6. 핸들러 실행
7. IRET 명령으로 복귀 (스택에서 레지스터 복원, 권한 레벨 전환)
```

### 예제 2: 시스템 콜 메커니즘 — 현대 방식 (`syscall` 명령어)

현대 x86-64 Linux는 `INT 0x80` 대신 훨씬 빠른 `SYSCALL/SYSRET` 명령어 쌍을 사용합니다.

```c
/*
 * C 언어 라이브러리(glibc)가 없는 환경에서
 * 직접 syscall을 호출하는 예시 (x86-64 Linux)
 */

/* 시스템 콜 번호 (arch/x86/entry/syscalls/syscall_64.tbl) */
#define SYS_write  1
#define SYS_exit   60

/* 인라인 어셈블리로 syscall 호출 */
static inline long syscall3(long number, long a1, long a2, long a3) {
    long ret;
    __asm__ volatile (
        "syscall"
        : "=a"(ret)                          // 출력: RAX에 반환값
        : "0"(number),                        // RAX: 시스템 콜 번호
          "D"(a1), "S"(a2), "d"(a3)          // RDI, RSI, RDX: 인자
        : "rcx", "r11", "memory"             // syscall이 RCX, R11을 변경함
    );
    return ret;
}

void _start(void) {
    const char *msg = "Hello from raw syscall!\n";
    long len = 24;

    // sys_write(fd=1, buf=msg, count=len)
    syscall3(SYS_write, 1, (long)msg, len);

    // sys_exit(status=0)
    syscall3(SYS_exit, 0, 0, 0);
}
```

**SYSCALL 명령어 처리 흐름:**

```
유저 공간:
  RAX = 시스템 콜 번호
  RDI, RSI, RDX, R10, R8, R9 = 인자 (최대 6개)
  SYSCALL 명령어 실행
    → CPU: RCX ← RIP, R11 ← RFLAGS
    → CPU: CS, SS를 커널 세그먼트로 변경
    → CPU: RIP ← IA32_LSTAR MSR 값 (커널의 syscall 진입점)

커널 공간 (entry_SYSCALL_64):
  swapgs                    ; GS 레지스터를 커널 percpu로 교환
  mov %rsp, %gs:... ;       ; 유저 스택 포인터 저장
  mov %gs:..., %rsp         ; 커널 스택으로 전환
  push registers            ; 레지스터 저장 (pt_regs)
  
  call do_syscall_64:
    syscall_table[RAX](...)  ; sys_call_table에서 함수 포인터 호출
  
  pop registers             ; 레지스터 복원
  swapgs
  SYSRET                    ; RIP ← RCX, RFLAGS ← R11, 유저 공간 복귀
```

### 예제 3: Linux 커널 모듈에서 인터럽트 핸들러 등록

```c
#include <linux/kernel.h>
#include <linux/interrupt.h>
#include <linux/module.h>

#define IRQ_NUMBER  1    /* 키보드 IRQ */
#define DEVICE_NAME "my_kbd"

static irqreturn_t my_keyboard_handler(int irq, void *dev_id) {
    uint8_t status = inb(0x64);
    if (status & 0x01) {
        uint8_t scancode = inb(0x60);
        pr_info("Scancode: 0x%02x\n", scancode);
        return IRQ_HANDLED;
    }
    return IRQ_NONE;
}

static int __init my_module_init(void) {
    int ret;
    /*
     * request_irq: IRQ 라인을 요청하고 핸들러 등록
     * IRQF_SHARED: 같은 IRQ를 다른 드라이버와 공유 허용
     */
    ret = request_irq(IRQ_NUMBER, my_keyboard_handler,
                      IRQF_SHARED, DEVICE_NAME, (void *)my_keyboard_handler);
    if (ret) {
        pr_err("Failed to request IRQ %d: %d\n", IRQ_NUMBER, ret);
        return ret;
    }
    pr_info("Keyboard IRQ handler registered\n");
    return 0;
}

static void __exit my_module_exit(void) {
    free_irq(IRQ_NUMBER, (void *)my_keyboard_handler);
    pr_info("Keyboard IRQ handler unregistered\n");
}

module_init(my_module_init);
module_exit(my_module_exit);
MODULE_LICENSE("GPL");
```

### 하드 인터럽트 vs 소프트 인터럽트 (softirq / tasklet)

인터럽트 핸들러는 실행 시간이 매우 짧아야 합니다. 처리할 작업이 많다면 **두 단계로 분리**합니다.

```
┌─────────────────────────────────────┐
│  하드 인터럽트 (인터럽트 비활성화) │  ← 최소한의 작업만: 하드웨어 상태 읽기, ACK
│  예: NIC가 패킷 수신 완료 알림     │
└───────────────┬─────────────────────┘
                │ softirq/tasklet 스케줄링
┌───────────────▼─────────────────────┐
│  소프트 인터럽트 / tasklet          │  ← 지연 처리: TCP/IP 스택 처리, 버퍼 복사
│  (인터럽트 활성화 상태에서 실행)    │
└─────────────────────────────────────┘
```

---

## 주의사항 및 팁

### 1. 인터럽트 핸들러 내에서의 제약

- **sleep 금지**: 인터럽트 핸들러는 원자적 컨텍스트에서 실행되므로 blocking 연산 불가
- **동기화**: `spinlock` 사용 (mutex는 sleep 가능성이 있으므로 금지)
- **재진입 불가**: 같은 IRQ가 핸들러 실행 중에 재발생하면 대부분 마스킹됨

### 2. 인터럽트 레이턴시와 실시간 시스템

일반 Linux 커널은 인터럽트 레이턴시가 수백 마이크로초에서 밀리초까지 달할 수 있습니다. 로봇 제어, 산업 자동화처럼 마이크로초 단위 응답이 필요한 시스템에서는 **PREEMPT_RT 패치** 또는 **Xenomai** 같은 실시간 확장을 사용하세요.

### 3. MSI(Message Signaled Interrupts)

최신 PCIe 장치는 물리 IRQ 라인 대신 **MSI(Message Signaled Interrupts)** 를 사용합니다. 장치가 메모리 주소에 직접 메시지를 쓰는 방식으로 인터럽트를 신호하며, 공유 IRQ 라인 충돌이 없고 멀티코어 분산이 쉽습니다.

### 4. vsyscall과 vDSO: syscall 오버헤드 줄이기

`gettimeofday()`, `clock_gettime()` 같이 자주 호출되는 시스템 콜은 **vDSO(virtual Dynamic Shared Object)** 를 통해 실제 커널 모드 전환 없이 처리됩니다. 커널이 읽기 전용 메모리 영역에 최신 시간 데이터를 업데이트하고 유저 프로세스가 직접 읽는 방식입니다.

```bash
# vDSO 확인
cat /proc/self/maps | grep vdso
# 출력 예: 7ffe3a9d8000-7ffe3a9da000 r-xp 00000000 00:00 0 [vdso]
```

### 5. 인터럽트 어피니티(Interrupt Affinity)

다중 코어 시스템에서 네트워크 집약적 워크로드는 특정 IRQ를 특정 CPU 코어에 고정하면 성능이 향상됩니다.

```bash
# IRQ 43번을 CPU 2에 고정
echo 4 > /proc/irq/43/smp_affinity  # 비트마스크: CPU 2 = 1 << 2 = 4

# 현재 IRQ 분배 확인
cat /proc/interrupts
```

---

## 참고 자료

- [Interrupts — The Linux Kernel Documentation](https://linux-kernel-labs.github.io/refs/heads/master/lectures/interrupts.html)
- [Linux Insides: Interrupts and Interrupt Handling — 0xAX](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-1.html)
- [SO2 Lecture 04: Interrupts — Linux Kernel Labs](https://linux-kernel-labs.github.io/refs/heads/master/so2/lec4-interrupts.html)
- [Interrupt Handling in Linux — OReilly Understanding the Linux Kernel](https://www.oreilly.com/library/view/understanding-the-linux/0596005652/ch04s06.html)
