---
layout: post
title: "ARM AArch64 아키텍처 심화: 64비트 명령어 집합과 레지스터 모델 완전 정복"
date: 2026-10-29
categories: [cs, computer-science]
tags: [arm, aarch64, isa, architecture, assembly, risc, processor]
---

스마트폰부터 서버까지, 현대 컴퓨팅의 중심에 ARM 아키텍처가 있다. Apple Silicon(M1/M2/M3), AWS Graviton, Qualcomm Snapdragon 등 고성능 ARM 프로세서가 x86-64의 아성을 무너뜨리고 있다. AArch64(ARM 64비트 실행 상태)는 ARMv8-A 아키텍처에서 도입된 64비트 명령어 집합으로, RISC 철학을 계승하면서도 현대 고성능 컴퓨팅의 요구를 충족하도록 설계되었다.

## 1. ARM 아키텍처의 역사와 AArch64의 등장

ARM(Advanced RISC Machine)은 1983년 Acorn Computers에서 시작된 RISC 기반 프로세서 아키텍처다. 초저전력 설계와 고성능의 조화가 핵심이다.

**주요 버전 변천:**
- **ARMv7 (32비트)**: Cortex-A9, A15 등. 모바일 황금기의 주역
- **ARMv8-A (2011)**: 64비트 AArch64와 하위호환 AArch32 동시 지원
- **ARMv9-A (2021)**: SVE2, Realm Management Extension, AI/ML 가속 추가

AArch64가 ARMv7 대비 갖는 핵심 차별점:
- **레지스터**: 16개 → 31개 범용 레지스터
- **주소 공간**: 32비트 → 최대 52비트 물리/가상 주소
- **부동소수점**: FP/SIMD 강화
- **보안**: TrustZone, Pointer Authentication(PAC), Branch Target Identification(BTI)
- **단순화**: 복잡한 32비트 조건 실행 명령어 제거

## 2. 레지스터 아키텍처

### 범용 레지스터 (General-Purpose Registers)

AArch64는 31개의 64비트 범용 레지스터를 제공한다:

| 표기 | 비트 | 설명 |
|------|------|------|
| X0 ~ X30 | 64비트 | 64비트 전체 사용 |
| W0 ~ W30 | 32비트 | 하위 32비트만 사용 (X 레지스터의 별칭) |
| XZR / WZR | - | 하드웨어 제로 레지스터 (읽기 시 0, 쓰기 시 무시) |
| SP | 64비트 | 스택 포인터 |
| PC | 64비트 | 프로그램 카운터 (직접 접근 불가) |

### 절차 호출 표준 (AAPCS64)

AArch64 함수 호출 규약(ABI):

```
X0-X7:   함수 인자 & 반환값 (caller-saved)
X8:      간접 결과 레지스터 (대형 구조체 반환 주소)
X9-X15:  임시 레지스터 (caller-saved)
X16-X17: 인트라 프로시저 호출 레지스터 (링커 벤더 사용)
X18:     플랫폼 레지스터 (OS별 예약)
X19-X28: Callee-saved 레지스터
X29:     프레임 포인터 (FP)
X30:     링크 레지스터 (LR) - 반환 주소 저장
```

### 시스템 레지스터와 PSTATE

**PSTATE**: 프로세서 현재 상태를 나타내는 플래그 집합
- **N (Negative)**: 결과가 음수
- **Z (Zero)**: 결과가 0
- **C (Carry)**: 올림(Carry) 발생
- **V (Overflow)**: 부호 있는 오버플로우

**EL (Exception Level)**: 실행 권한 레벨
- EL0: 사용자 프로그램
- EL1: OS 커널
- EL2: 하이퍼바이저
- EL3: Secure Monitor (TrustZone)

### SIMD/FP 레지스터

128비트 SIMD 레지스터 32개 (V0~V31), 다양한 벡터 관점으로 접근 가능:

```
Q0 : 128비트 전체
D0 : 하위 64비트
S0 : 하위 32비트
H0 : 하위 16비트
B0 : 하위 8비트

V0.16B  : 8비트 정수 16개 (SIMD 벡터)
V0.8H   : 16비트 정수 8개
V0.4S   : 32비트 정수 4개
V0.2D   : 64비트 정수 2개
V0.4F   : 32비트 부동소수점 4개
```

## 3. 명령어 집합 구조

### 고정 길이 명령어

AArch64의 모든 명령어는 **32비트 고정 길이**다. 이는 x86의 가변 길이(1~15바이트)와 대비되며:
- 명령어 페치/디코드 단순화
- 파이프라인 효율성 향상
- 코드 밀도는 다소 낮을 수 있음 (Thumb-2가 없으므로)

### 주요 명령어 카테고리

**데이터 처리:**
```asm
ADD  X0, X1, X2          ; X0 = X1 + X2
SUB  X0, X1, #10         ; X0 = X1 - 10  (즉시값)
MUL  X0, X1, X2          ; X0 = X1 * X2 (하위 64비트)
UMULH X0, X1, X2         ; X0 = (X1 * X2) >> 64 (상위 64비트)
SDIV X0, X1, X2          ; X0 = X1 / X2 (부호 있는 나눗셈)
AND  X0, X1, X2          ; 비트 AND
ORR  X0, X1, X2          ; 비트 OR
EOR  X0, X1, X2          ; 비트 XOR (EXclusive OR)
LSL  X0, X1, #3          ; 논리 좌측 시프트 3비트 (* 8)
LSR  X0, X1, #3          ; 논리 우측 시프트 3비트
ASR  X0, X1, #3          ; 산술 우측 시프트 (부호 확장)
```

**메모리 접근:**
```asm
LDR  X0, [X1]            ; X0 = Mem[X1] (64비트 로드)
LDR  W0, [X1]            ; W0 = Mem[X1] (32비트 로드, 상위 32비트 0)
LDRB W0, [X1]            ; W0 = Mem[X1] (8비트 로드)
STR  X0, [X1]            ; Mem[X1] = X0 (64비트 저장)
STR  W0, [X1, #8]        ; Mem[X1+8] = W0 (오프셋 주소 지정)
LDP  X0, X1, [X2]        ; X0=Mem[X2], X1=Mem[X2+8] (페어 로드)
STP  X0, X1, [SP, #-16]! ; 스택에 2개 저장 (사전 감소 인덱싱)
LDP  X29, X30, [SP], #16 ; 스택에서 2개 복원 (사후 증가 인덱싱)
```

**분기:**
```asm
B    label               ; 무조건 분기
BL   function            ; 함수 호출 (X30에 반환주소 저장)
RET                      ; 반환 (X30으로 분기)
BR   X1                  ; 레지스터 간접 분기
BLR  X1                  ; 레지스터 간접 함수 호출
CBZ  X0, label           ; X0==0이면 분기
CBNZ X0, label           ; X0!=0이면 분기
TBZ  X0, #5, label       ; X0의 5번 비트==0이면 분기
B.EQ label               ; 조건 분기 (Z==1)
B.NE label               ; 조건 분기 (Z==0)
B.LT label               ; 조건 분기 (N!=V, 부호있는 <)
B.GT label               ; 조건 분기 (Z==0 && N==V, 부호있는 >)
```

**이동과 즉시값:**
```asm
MOV  X0, X1              ; X0 = X1
MOV  X0, #42             ; X0 = 42 (16비트 즉시값)
MOVZ X0, #0x1234         ; X0 = 0x1234 (나머지 0)
MOVK X0, #0x5678, LSL #16; X0의 [31:16]에 0x5678, 나머지 유지
; 64비트 상수 로드: MOVZ + MOVK 조합
MOVZ X0, #0xEFCD         ; X0 = 0x000000000000EFCD
MOVK X0, #0x89AB, LSL 16 ; X0 = 0x0000000089ABEFCD
MOVK X0, #0x4567, LSL 32 ; X0 = 0x0000456789ABEFCD
MOVK X0, #0x0123, LSL 48 ; X0 = 0x0123456789ABEFCD
```

## 4. AArch64 어셈블리 실습: C 함수 직접 구현

```c
// C 함수 원형 (컴파일러가 생성하는 AArch64 어셈블리 이해)
long factorial(long n);
long sum_array(const long* arr, int len);
```

```asm
// factorial.S - 재귀 팩토리얼 구현
.section .text
.global factorial

// long factorial(long n)
// 입력:  X0 = n
// 반환:  X0 = n!
factorial:
    // 프롤로그: 스택 프레임 설정
    STP  X29, X30, [SP, #-16]!  // FP와 LR 스택에 저장
    MOV  X29, SP                 // 프레임 포인터 갱신

    // 베이스 케이스: n <= 1이면 1 반환
    CMP  X0, #1
    B.LE .base_case

    // 재귀 케이스: n * factorial(n-1)
    STP  X19, XZR, [SP, #-16]!  // X19(n 보존용) 스택에 저장
    MOV  X19, X0                 // n을 X19에 보존
    SUB  X0, X0, #1              // 인자: n-1
    BL   factorial               // 재귀 호출, 결과 X0에
    MUL  X0, X19, X0             // n * factorial(n-1)
    LDP  X19, XZR, [SP], #16     // X19 복원

    // 에필로그
    LDP  X29, X30, [SP], #16    // FP, LR 복원
    RET

.base_case:
    MOV  X0, #1
    LDP  X29, X30, [SP], #16
    RET


// sum_array.S - 배열 합산 (루프와 벡터화)
.section .text
.global sum_array

// long sum_array(const long* arr, int len)
// 입력: X0 = arr 포인터, W1 = len
// 반환: X0 = 합계
sum_array:
    MOV  X2, XZR           // sum = 0
    CBZ  W1, .done         // len == 0이면 종료
    MOV  W3, W1            // 카운터

.loop:
    LDR  X4, [X0], #8      // X4 = *arr, arr++ (사후 인덱싱으로 8바이트 증가)
    ADD  X2, X2, X4        // sum += *arr
    SUBS W3, W3, #1        // --count, 플래그 갱신
    B.NE .loop             // count != 0이면 반복

.done:
    MOV  X0, X2            // 반환값 설정
    RET
```

### 컴파일 및 실행

```bash
# AArch64 어셈블러로 빌드
as -o factorial.o factorial.S
as -o sum_array.o sum_array.S

# 또는 GCC로 C와 함께 링크
gcc -o test main.c factorial.S sum_array.S

# QEMU로 x86 호스트에서 실행 (크로스 컴파일)
aarch64-linux-gnu-gcc -o test main.c factorial.S -static
qemu-aarch64 ./test
```

## 5. NEON SIMD: 벡터 연산 가속

AArch64의 NEON(Advanced SIMD)은 데이터 병렬 처리의 핵심이다.

```asm
// 4개의 float 값 동시 덧셈 (NEON SIMD)
// void vec_add(float* a, float* b, float* c, int n)
// c[i] = a[i] + b[i]
.global vec_add
vec_add:
    // X0=a, X1=b, X2=c, W3=n
    LSR  W4, W3, #2         // W4 = n / 4 (SIMD 청크 수)
    CBZ  W4, .scalar        // 4개 미만이면 스칼라 루프

.simd_loop:
    LD1  {V0.4S}, [X0], #16 // V0 = a[0..3], X0 += 16
    LD1  {V1.4S}, [X1], #16 // V1 = b[0..3], X1 += 16
    FADD V2.4S, V0.4S, V1.4S // V2[i] = V0[i] + V1[i] (4개 동시)
    ST1  {V2.4S}, [X2], #16 // c[0..3] = V2, X2 += 16
    SUBS W4, W4, #1
    B.NE .simd_loop

.scalar:
    // 나머지 원소 처리 (n % 4)
    AND  W4, W3, #3
    CBZ  W4, .end

.scalar_loop:
    LDR  S0, [X0], #4
    LDR  S1, [X1], #4
    FADD S2, S0, S1
    STR  S2, [X2], #4
    SUBS W4, W4, #1
    B.NE .scalar_loop

.end:
    RET
```

## 6. Pointer Authentication Code (PAC): 보안 기능

ARMv8.3-A에서 도입된 PAC는 리턴 주소 변조를 암호학적으로 감지한다.

```asm
// PAC를 사용하는 함수 (컴파일러가 자동 생성)
function_with_pac:
    PACIASP          // SP와 X30에 PAC 삽입 (X30 = HMAC(X30, SP, APIAKey))
    STP  X29, X30, [SP, #-16]!
    MOV  X29, SP

    // ... 함수 본문 ...

    LDP  X29, X30, [SP], #16
    AUTIASP          // PAC 검증 (변조 시 PAC 비트 오류로 세그폴트)
    RET
```

PAC는 스택 카나리와 달리 암호학적 MAC를 사용하므로 위조가 극도로 어렵다. Apple M1부터 기본 활성화되어 있다.

## 7. 메모리 모델과 배리어

AArch64의 메모리 모델은 **약한 순서(Weakly Ordered)**다. 컴파일러와 하드웨어 모두 성능을 위해 메모리 접근을 재순서화할 수 있다.

```asm
// 메모리 배리어 명령어
DMB ISH          // Data Memory Barrier (Inner Shareable Domain)
                 // 이전 메모리 접근이 완료된 후 이후 접근 시작
DSB ISH          // Data Synchronization Barrier (더 강함)
                 // 이전 명령어 모두 완료 후 다음 진행
ISB              // Instruction Synchronization Barrier
                 // 파이프라인 완전 플러시, 명령어 재패치
```

**LDXR/STXR: 독점 접근 (Load-link/Store-conditional)**

```asm
// 원자적 Compare-And-Swap 구현
// int cas(int* ptr, int expected, int desired)
cas:
.retry:
    LDXR  W3, [X0]          // 독점 로드
    CMP   W3, W1            // expected와 비교
    B.NE  .fail             // 다르면 실패
    STXR  W4, W2, [X0]      // 독점 저장, W4=0(성공), 1(실패)
    CBNZ  W4, .retry        // 다른 코어가 선점했으면 재시도
    MOV   W0, #1            // 성공 반환
    RET
.fail:
    CLREX                   // 독점 모니터 초기화
    MOV   W0, #0
    RET
```

ARMv8.1-A부터는 더 효율적인 `LDADD`, `SWPAL` 같은 원자 명령어가 추가되었다.

## 8. 성능 특성과 마이크로아키텍처

### 파이프라인 특성 (Apple M1 Firestorm 코어 기준)
- **명령어 폭 (Fetch Width)**: 8 명령어/사이클
- **OOO 윈도우**: ~630 ROB 엔트리 (역대 최대 수준)
- **분기 예측**: 매우 정교한 하이브리드 예측기
- **캐시**: L1I 192KB, L1D 128KB, L2 12MB, L3 (공유)

### AArch64 vs x86-64 코드 비교

동일 기능의 코드 크기:
```
x86-64: 가변 길이 명령어로 코드 밀도 높음
AArch64: 모든 명령어 4바이트 고정 → 코드 크기 다소 큼
         단, 디코딩 복잡도 낮아 고클럭/저전력 설계에 유리
```

## 9. 주의사항과 실전 팁

### 정렬(Alignment) 요구사항
AArch64는 원칙적으로 자연 정렬을 요구한다:
- `LDR X0, [X1]`: X1이 8바이트 경계 정렬 필요
- `LDP X0, X1, [X2]`: X2가 16바이트 경계 정렬 필요
- SCTLR_EL1의 A 비트로 미정렬 접근 허용/금지 제어 가능

### SP 정렬
AArch64 ABI에서 SP는 **항상 16바이트 정렬** 유지 필수. SP를 홀수 8바이트로 조정하면 메모리 접근 시 정렬 예외 발생.

### 링크 레지스터 보존
BL로 함수를 호출하면 X30(LR)이 덮어써진다. 중첩 호출 시 반드시 스택에 저장:
```asm
STP X29, X30, [SP, #-16]!  // 항상 쌍으로 저장 (16바이트 정렬)
// ... 내부 호출 ...
LDP X29, X30, [SP], #16    // 복원 후 RET
RET
```

## 참고 자료
- [ARM Architecture Reference Manual for A-profile architecture](https://developer.arm.com/documentation/ddi0487/latest)
- [Learn the Architecture: A64 ISA Guide](https://developer.arm.com/documentation/102374/latest)
- [AArch64 Exception Model Overview](https://developer.arm.com/documentation/102412/latest)
- [AAPCS64: Procedure Call Standard](https://github.com/ARM-software/abi-aa/blob/main/aapcs64/aapcs64.rst)
