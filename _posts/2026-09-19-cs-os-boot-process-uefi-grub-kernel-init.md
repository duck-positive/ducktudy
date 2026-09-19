---
layout: post
title: "운영체제 부트 프로세스 완전 정복: UEFI, GRUB, 커널 초기화의 모든 것"
date: 2026-09-19
categories: [cs, computer-science]
tags: [os, boot, uefi, grub, kernel, linux, systemd, initramfs]
---

## 개요

컴퓨터의 전원 버튼을 누른 순간부터 로그인 프롬프트가 뜨기까지, 수십 개의 복잡한 단계가 순서대로 실행됩니다. 이 과정을 **부트 프로세스(Boot Process)**라고 하며, 하드웨어 초기화 → 펌웨어 실행 → 부트로더 → 커널 적재 → 사용자 공간 초기화의 흐름을 따릅니다.

운영체제 개발자, 임베디드 엔지니어, 보안 연구자 모두에게 부트 프로세스는 핵심 지식입니다. Secure Boot 우회 공격, 커스텀 커널 빌드, 부트 성능 최적화 모두 이 흐름의 정밀한 이해를 요구합니다.

---

## 1단계: 전원 인가와 POST

CPU에 전원이 들어오면 리셋 벡터(Reset Vector)의 주소로 점프합니다. x86 아키텍처에서는 `0xFFFFFFF0` 주소에서 실행을 시작하며, 이 주소는 플래시 메모리에 저장된 **펌웨어 코드**를 가리킵니다.

펌웨어는 POST(Power-On Self Test)를 수행합니다.

1. CPU 레지스터, 캐시, 버스 초기화
2. RAM 검사 및 메모리 맵 구성
3. PCI/PCIe 장치 열거 및 초기화
4. USB, SATA 컨트롤러 초기화
5. 표준 I/O 장치(키보드, 화면) 설정

POST 결과는 ACPI(Advanced Configuration and Power Interface) 테이블에 기록되어 이후 OS가 참조합니다.

---

## 2단계: BIOS vs UEFI

### 레거시 BIOS

전통적인 BIOS는 MBR(Master Boot Record) 방식을 사용합니다. 부팅 디스크의 첫 512바이트에서 부트 코드를 읽어 실행합니다.

```
디스크 첫 512바이트 (MBR):
┌─────────────────────┬──────────────────┬──────────┐
│ 부트 코드 (446B)    │ 파티션 테이블(64B) │ 서명(2B) │
└─────────────────────┴──────────────────┴──────────┘
```

MBR 방식의 한계:
- 최대 디스크 크기 2TB (32비트 LBA)
- 최대 4개의 기본 파티션
- 보안 기능 없음

### UEFI (Unified Extensible Firmware Interface)

UEFI는 2006년 이후 BIOS를 대체하기 시작한 현대적 펌웨어 표준입니다.

```
UEFI 부트 순서:
UEFI 펌웨어 → ESP 탐색 → EFI 실행 파일 로드 → 부트로더 실행
             (EFI System Partition, FAT32 포맷)
```

UEFI의 핵심 기능:

1. **GPT(GUID Partition Table)**: 최대 9.4ZB 디스크, 128개 파티션 지원
2. **EFI 실행 환경**: 64비트 모드에서 직접 PE 형식 실행 파일을 실행
3. **Boot Manager**: NVRAM에 부팅 항목 저장, 여러 OS 지원
4. **Secure Boot**: 디지털 서명 검증으로 악성 부트로더 차단

```bash
# ESP 파티션 구조 확인
ls /boot/efi/EFI/
# ubuntu/  grub/  Boot/

# UEFI 부팅 항목 확인 (Linux)
efibootmgr -v
# BootCurrent: 0001
# Boot0001* ubuntu        HD(1,GPT,uuid,...)/File(\EFI\ubuntu\shimx64.efi)

# NVRAM에서 직접 EFI 실행 파일 경로 지정
efibootmgr --create --disk /dev/sda --part 1 \
  --label "Custom Linux" --loader '\EFI\custom\bootx64.efi'
```

---

## 3단계: 부트로더 (GRUB2)

GRUB(Grand Unified Bootloader)은 Linux에서 가장 널리 쓰이는 부트로더입니다. 2단계로 작동합니다.

### Stage 1: MBR 또는 EFI 실행

UEFI 환경에서는 `grubx64.efi`가 ESP에서 직접 실행됩니다. BIOS 환경에서는 MBR의 446바이트 코드가 `core.img`를 로드합니다.

### Stage 2: 커널과 initramfs 로드

GRUB2 설정 파일(`/boot/grub/grub.cfg`)을 읽어 메뉴를 표시하고, 선택된 항목의 커널과 initramfs를 메모리에 로드합니다.

```bash
# /boot/grub/grub.cfg의 핵심 구조
menuentry "Ubuntu 24.04 LTS" {
    # 루트 파티션 설정
    set root='hd0,gpt2'
    
    # 커널 이미지 로드 및 파라미터 전달
    linux   /boot/vmlinuz-6.8.0-generic \
            root=/dev/sda2 \
            ro quiet splash \
            mitigations=off     # Spectre/Meltdown 완화 비활성화(성능 우선)
    
    # initramfs 로드 (메모리 기반 초기 루트 파일시스템)
    initrd  /boot/initrd.img-6.8.0-generic
}
```

GRUB2가 로드하는 커널 이미지는 압축된 bzImage 형식입니다.

```
bzImage 구조:
┌───────────────┬────────────────────────────────────────┐
│ Setup code    │ 압축된 커널 (gzip/LZ4/zstd)           │
│ (실제 모드)   │ (보호 모드 진입 후 압축 해제)          │
└───────────────┴────────────────────────────────────────┘
```

---

## 4단계: Linux 커널 초기화

GRUB로부터 제어권을 넘겨받은 커널은 아키텍처 의존적 초기화를 먼저 수행합니다.

### 초기 설정 (arch/x86/boot/)

```c
// arch/x86/boot/main.c (실제 모드에서 실행)
void main(void) {
    /* 메모리 레이아웃 탐지 */
    detect_memory();
    
    /* 보호 모드로 전환 */
    go_to_protected_mode();
    // 이 시점에서 32비트 모드로 전환
    // 이후 64비트 롱 모드로 전환
}
```

### start_kernel() — 커널 핵심 초기화

커널 압축 해제 후 `start_kernel()` 함수가 실행됩니다. 이 함수는 수백 개의 서브시스템을 순서대로 초기화합니다.

```c
// init/main.c (커널 초기화의 핵심)
asmlinkage __visible void __init __no_sanitize_address start_kernel(void)
{
    char *command_line;
    
    /* CPU, 메모리, 인터럽트 핵심 초기화 */
    setup_arch(&command_line);      // CPU 아키텍처 설정
    setup_per_cpu_areas();          // CPU별 데이터 영역 설정
    trap_init();                    // 예외/인터럽트 핸들러 등록
    mm_init();                      // 메모리 관리자 초기화
    sched_init();                   // 스케줄러 초기화
    
    /* 버스, 드라이버 초기화 */
    early_initcall_calls();         // 초기 initcall 실행
    
    /* init 프로세스 생성 */
    rest_init();                    // PID 1 (init) 커널 스레드 생성
}

static noinline void __ref rest_init(void)
{
    /* kernel_init 스레드를 PID 1로 생성 */
    kernel_thread(kernel_init, NULL, CLONE_FS);
    
    /* 스케줄러 시작: 이제 멀티태스킹 가능 */
    schedule_preempt_disabled();
}
```

### initramfs: 임시 루트 파일시스템

커널은 최종 루트 파일시스템을 마운트하기 전에 **initramfs**를 임시 루트(`/`)로 마운트합니다. initramfs는 cpio + gzip 아카이브로, 실제 루트 파일시스템을 마운트하는 데 필요한 모든 것을 포함합니다.

```bash
# initramfs 내용 확인
mkdir /tmp/initramfs && cd /tmp/initramfs
unmkinitramfs /boot/initrd.img-6.8.0-generic .
ls
# bin/  dev/  etc/  lib/  lib64/  scripts/  ...

# initramfs의 init 스크립트 확인
cat init
# #!/bin/sh
# [ -d /dev ] || mkdir -m 0755 /dev
# mount -t devtmpfs devtmpfs /dev
# ...
# exec switch_root /root /sbin/init  ← 실제 루트로 전환!
```

initramfs에서 LUKS 암호화 파티션 해독, LVM 볼륨 활성화, NFS 루트 마운트 등 복잡한 초기 작업이 수행됩니다.

---

## 5단계: systemd — PID 1의 사용자 공간 초기화

`switch_root`로 실제 루트 파일시스템으로 전환한 뒤, `/sbin/init` (→ systemd)이 PID 1로 실행됩니다.

```
systemd 부트 순서:
sysinit.target → basic.target → multi-user.target → graphical.target
     ↑                ↑                 ↑                  ↑
  장치/FS       네트워크/소켓     시스템 서비스           GUI
```

```bash
# systemd 부팅 성능 분석
systemd-analyze
# Startup finished in 1.234s (firmware) + 3.456s (loader) + 0.789s (kernel) + 5.678s (userspace) = 11.157s

# 병목 서비스 분석
systemd-analyze blame | head -10
# 3.456s NetworkManager.service
# 2.345s plymouth-start.service
# 1.234s dev-sda2.device

# 의존성 그래프 시각화
systemd-analyze dot multi-user.target | dot -Tsvg > boot.svg
```

---

## Secure Boot: 부트 체인 신뢰 모델

UEFI Secure Boot는 부트 체인의 각 단계에서 디지털 서명을 검증합니다.

```
UEFI 펌웨어 (신뢰 앵커)
    ↓ 서명 검증
shimx64.efi (Microsoft 서명)
    ↓ 서명 검증
grubx64.efi (배포판 서명)
    ↓ 서명 검증
vmlinuz (배포판 서명)
```

커스텀 커널을 Secure Boot 환경에서 부팅하려면 자체 서명키를 MOK(Machine Owner Key)에 등록해야 합니다.

```bash
# 자체 서명키 생성
openssl req -new -x509 -newkey rsa:2048 \
  -keyout MOK.priv -out MOK.der \
  -days 36500 -subj "/CN=My Custom Kernel/"

# MOK 등록 (재부팅 시 UEFI에서 확인 필요)
mokutil --import MOK.der

# 커널 모듈 서명
/usr/src/linux-headers-$(uname -r)/scripts/sign-file \
  sha256 MOK.priv MOK.der my_module.ko
```

---

## 주의사항 및 트러블슈팅 팁

1. **부트 파라미터 디버깅**: `nomodeset`, `debug`, `systemd.log_level=debug` 파라미터로 부팅 문제 진단
2. **grub 복구**: LiveUSB에서 `chroot` 후 `grub-install`, `update-grub` 실행
3. **initramfs 재생성**: 드라이버 변경 후 `update-initramfs -u -k all` 실행
4. **부팅 시간 최적화**: `systemd-analyze critical-chain` 으로 임계 경로 분석
5. **Secure Boot 문제**: `mokutil --list-enrolled`로 등록된 키 확인

---

## 참고 자료

- [Linux 부팅 프로세스 — Wikipedia](https://en.wikipedia.org/wiki/Booting_process_of_Linux)
- [Arch Linux 부트 프로세스 — ArchWiki](https://wiki.archlinux.org/title/Arch_boot_process)
- [UEFI Specification 2.10 — UEFI Forum](https://uefi.org/specifications)
- [Linux 커널 문서 — kernel.org](https://www.kernel.org/doc/html/latest/)
