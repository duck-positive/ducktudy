---
layout: post
title: "메모리 보안 메커니즘 심화: ASLR, Stack Canary, PIE, CFI"
date: 2026-10-09
categories: [cs, computer-science]
tags: [security, ASLR, stack-canary, PIE, CFI, memory-safety, exploit-mitigation, linux]
---

## 개요

버퍼 오버플로우(buffer overflow)는 1988년 Morris Worm 이후 수십 년이 지난 지금도 활발히 악용되는 취약점입니다. 운영체제와 컴파일러 생태계는 이를 방어하기 위해 **다층 방어(defense in depth)** 전략으로 여러 보안 메커니즘을 발전시켜 왔습니다.

이 글에서는 현대 시스템에서 사용되는 네 가지 핵심 메모리 보안 메커니즘인 **ASLR**, **Stack Canary**, **PIE**, **CFI**의 원리와 구현, 그리고 각각의 한계를 심층 분석합니다.

---

## 배경: 메모리 공격의 기초

메모리 보안 메커니즘을 이해하려면 먼저 공격 원리를 알아야 합니다.

### 고전적 스택 버퍼 오버플로우

```c
// 취약한 코드 예시
#include <stdio.h>
#include <string.h>

void greet(char *name) {
    char buffer[64];       // 스택에 64바이트 할당
    strcpy(buffer, name);  // 크기 검사 없는 복사 → 오버플로우 가능!
    printf("Hello, %s!\n", buffer);
}

int main(int argc, char *argv[]) {
    greet(argv[1]);
    return 0;
}
```

```
스택 메모리 레이아웃 (x86-64, 고주소 → 저주소):

  ┌─────────────────────────────┐ ← 높은 주소
  │   반환 주소 (return addr)    │  ← 여기를 덮어쓰면 제어 흐름 탈취!
  ├─────────────────────────────┤
  │   저장된 rbp (saved rbp)     │
  ├─────────────────────────────┤
  │   buffer[63]                │
  │   buffer[62]                │
  │   ...                       │
  │   buffer[0]                 │  ← strcpy가 여기부터 씀
  └─────────────────────────────┘ ← 낮은 주소

64바이트 초과 입력 시: buffer → saved rbp → return addr 순서로 덮어씌워짐
```

공격자가 반환 주소를 자신의 shellcode 주소나 system() 함수 주소로 교체하면, 프로그램은 의도치 않은 코드를 실행합니다.

---

## 1. Stack Canary (스택 카나리)

### 원리

Stack Canary는 **광부들이 탄광에서 독가스 탐지용으로 사용한 카나리아 새**에서 이름을 땄습니다. 버퍼와 반환 주소 사이에 검증값(canary)을 삽입하고, 함수 반환 직전에 이 값이 변조되었는지 확인합니다.

```c
// 컴파일러가 생성하는 canary 코드 (개념적 표현)
void greet_with_canary(char *name) {
    // 프롤로그: 카나리 값 저장
    unsigned long canary = __stack_chk_guard; // 프로세스 시작 시 랜덤 설정
    // 카나리를 스택의 적절한 위치에 배치

    char buffer[64];
    strcpy(buffer, name);
    printf("Hello, %s!\n", buffer);

    // 에필로그: 카나리 검증
    if (canary != __stack_chk_guard) {
        __stack_chk_fail(); // 스택 오염 감지! 프로세스 종료
    }
}
```

```
카나리가 포함된 스택 레이아웃:

  ┌─────────────────────────────┐
  │   반환 주소                  │
  ├─────────────────────────────┤
  │   저장된 rbp                 │
  ├─────────────────────────────┤
  │   [CANARY VALUE]            │  ← 0x00 포함 랜덤 8바이트
  ├─────────────────────────────┤
  │   buffer[63] ... buffer[0]  │
  └─────────────────────────────┘

오버플로우 발생 시: buffer → CANARY → rbp → return addr 순으로 덮어씌워짐
반환 전 카나리 검증에서 변조 감지 → abort()
```

### 실제 어셈블리 분석

```bash
# GCC로 컴파일 (-fstack-protector-strong 옵션)
gcc -fstack-protector-strong -o vulnerable vulnerable.c

# 생성된 어셈블리 확인
objdump -d vulnerable | grep -A 30 "<greet>"
```

```asm
; x86-64 어셈블리 (카나리 포함)
<greet>:
    push   rbp
    mov    rbp, rsp
    sub    rsp, 0x60

    ; === 카나리 설정 (프롤로그) ===
    mov    rax, QWORD PTR fs:0x28   ; TLS에서 카나리 값 로드
    mov    QWORD PTR [rbp-0x8], rax ; 스택에 카나리 저장
    xor    eax, eax

    ; ... 함수 본문 ...

    ; === 카나리 검증 (에필로그) ===
    mov    rax, QWORD PTR [rbp-0x8]  ; 스택에서 카나리 읽기
    xor    rax, QWORD PTR fs:0x28    ; TLS 값과 XOR 비교
    je     <greet+success>           ; 일치하면 정상 반환
    call   <__stack_chk_fail@plt>    ; 불일치 시 abort!

<greet+success>:
    leave
    ret
```

### 카나리 종류와 한계

| 종류 | 구성 | 우회 난이도 |
|------|------|------------|
| Terminator Canary | `\0`, `\n`, `-1` 포함 | 낮음 (NULL 바이트 복사 불가 이용) |
| Random Canary | 런타임 랜덤 값 | 중간 (정보 누출 취약점으로 우회) |
| Random XOR Canary | 랜덤값 XOR 반환 주소 | 높음 |

**한계점:**
- **정보 누출(info leak) 취약점**이 있으면 카나리 값 노출 가능
- `fork()` 서버에서는 카나리 값이 자식 프로세스에 복사됨
- **4바이트씩 덮어쓰는 비순차적 오버플로우**는 카나리를 건너뜀

---

## 2. ASLR (Address Space Layout Randomization)

### 원리

ASLR은 프로세스의 **메모리 레이아웃을 매 실행마다 랜덤화**합니다. 공격자가 shellcode의 주소나 libc 함수의 주소를 하드코딩할 수 없게 만드는 것이 목적입니다.

```bash
# ASLR 레벨 확인 (Linux)
cat /proc/sys/kernel/randomize_va_space
# 0: ASLR 비활성화
# 1: 스택, mmap, vdso 랜덤화
# 2: 스택, mmap, vdso, 힙 랜덤화 (권장)

# ASLR 효과 확인: 같은 프로그램 두 번 실행
./test &  cat /proc/$!/maps | grep "libc"
# 7f3b2a000000  ← 첫 실행 libc 주소
./test &  cat /proc/$!/maps | grep "libc"
# 7f9c1d000000  ← 두 번째 실행 libc 주소 (다름!)
```

```
ASLR 전/후 메모리 맵 비교:

ASLR 비활성화:              ASLR 활성화:
  0x400000: executable        0x400000: executable (PIE 없이 고정)
  0x7ffff7a00000: libc         0x7f3b2a000000: libc (랜덤)
  0x7ffff7bff000: ld-linux      0x7f3b2c100000: ld-linux (랜덤)
  0x7ffffffde000: stack        0x7ffc4a8de000: stack (랜덤)
```

### ASLR 엔트로피

엔트로피가 높을수록 공격자가 올바른 주소를 추측하기 어렵습니다.

```
x86-64 Linux ASLR 엔트로피:
- 스택: 20비트 (~1,048,576가지 가능한 위치)
- mmap: 28비트
- 힙: 13비트

32비트 시스템: 엔트로피가 낮아 브루트포스 가능
→ 500ms 안에 주소 추측 성공 가능
```

### ASLR 우회 기법

```c
// 정보 누출(info leak)을 이용한 ASLR 우회
// printf의 형식 문자열 취약점으로 스택 주소 노출

char buf[100];
snprintf(buf, sizeof(buf), user_input); // user_input = "%p %p %p %p"
printf(buf); // → 0x7ffc4a8de120 0x7f3b2a3f2440 ...
//               ↑ 스택 주소 노출! → ASLR 우회 가능
```

**한계점:**
- **정보 누출 취약점** 하나면 ASLR 무력화
- **32비트 환경**에서는 엔트로피 부족으로 브루트포스 공격 가능
- `fork()` 기반 서버는 자식 프로세스가 같은 메모리 레이아웃 공유

---

## 3. PIE (Position Independent Executable)

### ASLR만으로는 부족한 이유

ASLR은 **스택, 힙, 공유 라이브러리** 주소를 랜덤화하지만, **실행 파일(ELF) 자체**는 고정 주소(일반적으로 `0x400000`)에 로드됩니다. 공격자가 실행 파일 내부의 코드/데이터 주소를 알고 있다면 ROP(Return-Oriented Programming)이 가능합니다.

**PIE**는 실행 파일 자체도 위치 독립적 코드로 컴파일하여, ASLR이 실행 파일 기반 주소도 랜덤화할 수 있게 합니다.

```bash
# PIE 없이 컴파일
gcc -no-pie -o binary_nopie main.c
readelf -h binary_nopie | grep "Type\|Entry"
# Type: EXEC            ← 고정 주소 실행 파일
# Entry: 0x401060        ← 항상 같은 주소

# PIE 활성화
gcc -fPIE -pie -o binary_pie main.c
readelf -h binary_pie | grep "Type\|Entry"
# Type: DYN             ← 공유 오브젝트처럼 위치 독립적
# Entry: 0x1060         ← 상대 주소 (런타임에 base + 0x1060으로 결정)
```

### PIE의 작동 방식

```
PIE + ASLR 활성화 시:

실행 1:
  [ELF base]   = 0x555555554000  (랜덤)
  main()        = 0x555555554000 + 0x1060 = 0x555555555060

실행 2:
  [ELF base]   = 0x561a22000000  (다른 랜덤 값)
  main()        = 0x561a22000000 + 0x1060 = 0x561a22001060

공격자는 ELF base를 모르므로 실행 파일 내 가젯(gadget) 주소를 예측 불가!
```

### PIE의 성능 비용

```c
// PIE 활성화 시 전역 변수 접근 방식
// GOT(Global Offset Table)를 통한 간접 참조
// 추가 메모리 참조 → 약 1~5% 성능 오버헤드 (workload에 따라 상이)

// 비PIE: 직접 주소
mov    eax, DWORD PTR [0x601040]  // 1 메모리 접근

// PIE: RIP-relative 어드레싱
lea    rax, [rip + 0x2fe9]        // RIP 기반 상대 주소
mov    eax, DWORD PTR [rax]       // 2 메모리 접근 (GOT 경유)
```

---

## 4. CFI (Control Flow Integrity)

### ROP 공격과 CFI의 필요성

ASLR + PIE로 메모리 주소를 랜덤화해도, **정보 누출 취약점**이 있으면 여전히 공격이 가능합니다. 더 근본적인 문제는 **코드 재사용 공격(Code Reuse Attack)**입니다.

**ROP(Return-Oriented Programming)**: 이미 실행 파일/라이브러리에 존재하는 코드 조각("가젯")을 연결하여 공격자가 원하는 동작을 구성합니다.

```
ROP 공격 원리:

실제 libc 코드 조각들 (가젯):
  0x7f..1234: pop rdi; ret     ← 가젯 1
  0x7f..5678: pop rsi; ret     ← 가젯 2
  0x7f..abcd: syscall; ret     ← 가젯 3

오버플로우로 스택을 조작:
  [가젯1 주소] ["/bin/sh" 주소] [가젯2 주소] [0] [syscall 주소]
      ↓               ↓              ↓          ↓        ↓
  pop rdi     → rdi="/bin/sh"  pop rsi    rsi=0  syscall(execve)

결과: execve("/bin/sh", 0, 0) 실행 → 쉘 획득!
```

**CFI**는 **프로그램의 제어 흐름(control flow)이 미리 정의된 합법적 경로만 따르도록 강제**합니다.

### Forward-Edge CFI vs Backward-Edge CFI

**Forward-Edge CFI**: 간접 호출(indirect call/jump) 대상을 제한

```c
// 함수 포인터를 통한 간접 호출 (전통적 방식)
typedef void (*func_t)(int);
func_t fp = get_function(); // 공격자가 fp를 조작할 수 있음
fp(42);                     // 공격자가 임의 함수를 호출!

// LLVM CFI 적용 시: 타입 기반 검증
// fp가 가리키는 함수가 올바른 타입 서명을 가지는지 런타임 검증
// 잘못된 타입 → 즉시 abort()
```

**Backward-Edge CFI**: 함수 반환 주소를 제한 (Shadow Stack)

```
Shadow Stack 메커니즘 (Intel CET / ARM PAC):

일반 스택:           Shadow Stack (보호됨):
  [return addr]   ←──── [return addr 사본]
  [saved rbp]
  [locals...]

함수 반환 시:
  1. 일반 스택의 return addr 읽기
  2. Shadow Stack의 return addr 읽기
  3. 두 값이 다르면 → ROP 공격 감지! → 프로세스 종료

Shadow Stack은 쓰기 보호된 메모리 영역에 있어 오버플로우로 변조 불가
```

### LLVM CFI 실제 구현

```bash
# LLVM CFI 컴파일 옵션
clang -flto -fvisibility=hidden -fsanitize=cfi -o secure_binary main.c

# CFI 종류
-fsanitize=cfi-icall      # 간접 함수 호출 검증
-fsanitize=cfi-vcall      # 가상 함수 호출 검증 (C++)
-fsanitize=cfi-nvcall     # 비가상 멤버 함수 호출 검증
-fsanitize=cfi-unrelated-cast # 타입 캐스트 검증
```

```c
// CFI 동작 예시 (C++)
class Animal {
public:
    virtual void speak() = 0;
};

class Dog : public Animal {
public:
    void speak() override { printf("Woof!\n"); }
};

class Cat : public Animal {
public:
    void speak() override { printf("Meow!\n"); }
};

void test(Animal *a) {
    a->speak(); // 가상 함수 호출 (간접 호출)
}

// CFI 없이: 공격자가 vtable 포인터를 조작하여 임의 함수 호출 가능
// CFI 적용 시: a가 Animal 계층의 올바른 vtable을 가지는지 컴파일러가 검증 코드 삽입
// → vtable 오염 공격 (vtable hijacking) 방어!
```

### Intel CET (Control-flow Enforcement Technology)

하드웨어 수준의 CFI 지원으로, 소프트웨어 오버헤드가 거의 없습니다.

```
Intel CET 두 가지 기능:

1. IBT (Indirect Branch Tracking):
   - 모든 합법적 간접 점프 대상에 ENDBR64 명령 배치
   - ENDBR64 없는 주소로의 간접 점프 → #CP 예외 발생
   
   합법적 가젯:                불법 가젯:
   0x1234: ENDBR64            0x5678: pop rax
   0x1238: pop rdi              (ENDBR64 없음)
   0x123c: ret                0x5679: ret

   공격자가 0x5679로 점프 시도 → 하드웨어가 차단!

2. Shadow Stack (SHSTK):
   - RSP와 별도의 SSP(Shadow Stack Pointer) 유지
   - CALL 시 Shadow Stack에도 return address 복사
   - RET 시 두 주소 자동 비교 → 불일치 시 #CP 예외
```

---

## 종합: 보안 메커니즘 계층 분석

```bash
# checksec 도구로 바이너리 보안 속성 확인
checksec --file=/usr/bin/ls
# [*] '/usr/bin/ls'
#     Arch:     amd64-64-little
#     RELRO:    Full RELRO          ← GOT 쓰기 보호
#     Stack:    Canary found        ← 스택 카나리
#     NX:       NX enabled          ← 스택/힙 실행 불가 (DEP)
#     PIE:      PIE enabled         ← 위치 독립 실행 파일
#     RUNPATH:  No RUNPATH
#     Symbols:  No Symbols          ← 심볼 정보 제거 (추가 보호)
```

| 메커니즘 | 방어 대상 | 구현 위치 | 성능 오버헤드 | 우회 기법 |
|---------|---------|---------|------------|---------|
| Stack Canary | 스택 오버플로우 | 컴파일러 | <1% | 정보 누출, 비순차 오버플로우 |
| ASLR | 주소 예측 | OS 커널 | <1% | 정보 누출, 브루트포스(32비트) |
| PIE | ELF 고정 주소 | 컴파일러 + OS | 1~5% | 정보 누출로 base 주소 획득 |
| CFI (소프트웨어) | ROP/JOP 공격 | 컴파일러 | 1~10% | CFI 정책 우회 가젯 탐색 |
| CFI (하드웨어/CET) | ROP/JOP 공격 | CPU + OS | <1% | 미성숙 (현재 연구 중) |

---

## 실전: 보안 옵션으로 컴파일하기

```bash
# 권장 보안 컴파일 플래그 (GCC/Clang)
CFLAGS = \
    -fstack-protector-strong \  # 스택 카나리 (더 강력한 버전)
    -D_FORTIFY_SOURCE=2 \        # 표준 라이브러리 경계 검사
    -fPIE \                      # PIE 활성화
    -Wformat -Wformat-security \ # 형식 문자열 경고

LDFLAGS = \
    -pie \                       # PIE 링킹
    -Wl,-z,relro \               # RELRO (GOT 읽기 전용화)
    -Wl,-z,now \                 # Full RELRO (전체 GOT 즉시 바인딩)
    -Wl,-z,noexecstack           # 스택 실행 불가

# 예시
gcc $(CFLAGS) -o myapp main.c $(LDFLAGS)

# 커널 수준 ASLR 최대화
echo 2 > /proc/sys/kernel/randomize_va_space

# ARM PAC(Pointer Authentication Code) 활성화 (iOS/Android/Apple Silicon)
# Clang:
clang -arch arm64 -mbranch-protection=pac-ret+bti main.c
```

---

## 주의사항 및 실전 팁

1. **단일 메커니즘으로는 부족**: 각 메커니즘은 서로의 한계를 보완합니다. ASLR + PIE + Canary + CFI를 모두 적용하는 것이 원칙.

2. **정보 누출 취약점이 모든 것을 무너뜨림**: 형식 문자열 취약점, out-of-bounds read 등으로 단 하나의 주소라도 누출되면 ASLR/PIE가 약화됩니다.

3. **레거시 코드 주의**: `-no-pie`, `-fno-stack-protector` 옵션을 가진 서드파티 라이브러리 하나가 전체 보안 체인을 약화시킬 수 있습니다.

4. **CFI는 LTO(Link-Time Optimization) 필요**: LLVM CFI는 전체 프로그램의 타입 정보가 필요하므로 LTO와 함께 사용해야 합니다.

---

## 참고 자료

- [Linux 커널 ASLR 구현 코드](https://github.com/torvalds/linux/blob/master/arch/x86/mm/mmap.c)
- [GCC Stack Protection 구현](https://github.com/gcc-mirror/gcc/blob/master/gcc/stack-check.cc)
- [LLVM CFI 구현 소스](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/ControlFlowIntegrity.cpp)
- [Intel CET 명세 GitHub 저장소](https://github.com/intel/indirect-branch-tracking)
