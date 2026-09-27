---
layout: post
title: "RISC-V와 ISA 설계 원리: CISC vs RISC 완전 정복"
date: 2026-09-27
categories: [cs, computer-science]
tags: [risc-v, isa, cisc, risc, cpu, instruction-set, assembly, computer-architecture]
---

## ISA란 무엇인가

**명령어 집합 구조(Instruction Set Architecture, ISA)**는 하드웨어와 소프트웨어 사이의 추상화 계층입니다. 어떤 연산을 지원하는지, 레지스터가 몇 개인지, 메모리를 어떻게 접근하는지, 명령어 인코딩 형식이 어떤지를 정의합니다. 컴파일러는 ISA 명세에 따라 기계어를 생성하고, CPU는 그 명세를 구현합니다. ISA는 두 진영 사이의 계약서입니다.

ISA 설계는 수십 년간 두 가지 철학으로 나뉘어 왔습니다. 하나는 **CISC(Complex Instruction Set Computer)**이고, 다른 하나는 **RISC(Reduced Instruction Set Computer)**입니다. 이 두 접근법의 장단점을 이해하고, 현재 가장 주목받는 오픈 ISA인 **RISC-V**의 설계 철학을 살펴봅니다.

---

## CISC: 복잡한 명령어의 세계

**CISC**를 대표하는 아키텍처는 **x86**입니다. 1978년 Intel 8086에서 시작된 x86은 하위 호환성을 유지하면서 반세기 가까이 확장되어, 현재 수천 개의 명령어를 보유하고 있습니다.

CISC의 핵심 특징:

- **가변 길이 명령어**: x86 명령어는 1바이트부터 15바이트까지 다양합니다.
- **마이크로코드**: 복잡한 명령어를 내부적으로 더 단순한 마이크로 연산으로 분해합니다.
- **메모리-레지스터 연산**: `ADD [mem], reg`처럼 메모리와 레지스터를 직접 조합하는 명령어가 있습니다.
- **고밀도 코드**: 복잡한 연산을 적은 명령어로 표현할 수 있어 코드 크기가 작아집니다.

CISC의 근본적인 설계 동기는 **시멘틱 갭(semantic gap)** 축소였습니다. 고급 언어의 복잡한 구문을 단일 명령어로 대응시켜 컴파일러 부담을 줄이고, 메모리 대역폭이 병목이던 시대에 코드 밀도를 높이는 것이었습니다.

---

## RISC: 단순함의 힘

1980년대 David Patterson(UC Berkeley)과 John Hennessy(Stanford)의 연구에서 출발한 RISC는 "간단하고 균일한 명령어를 빠르게 실행하는 것이 복잡한 명령어를 느리게 실행하는 것보다 낫다"는 철학을 담습니다.

RISC의 핵심 원칙:

- **고정 길이 명령어**: 모든 명령어가 동일한 크기(보통 32비트)입니다. 디코딩이 단순합니다.
- **로드-스토어 아키텍처**: 메모리 접근은 `LOAD`/`STORE` 전용 명령어만 담당합니다. 연산 명령어는 레지스터-레지스터만 허용합니다.
- **풍부한 레지스터**: 보통 32개의 범용 레지스터를 제공합니다.
- **파이프라인 친화성**: 단순하고 균일한 명령어 덕분에 깊은 파이프라인 설계가 쉽습니다.

대표적인 RISC 아키텍처로는 ARM, MIPS, SPARC, PowerPC, 그리고 RISC-V가 있습니다.

---

## RISC-V: 오픈 ISA의 등장

**RISC-V**는 2010년 UC Berkeley에서 개발된 오픈 소스 ISA입니다. 기존 ISA(x86, ARM)는 라이선스 비용과 독점 조항이 있었지만, RISC-V는 완전히 오픈되어 있습니다. 누구나 자유롭게 프로세서를 설계하고, 교육 목적으로 활용하고, 상용 제품에 탑재할 수 있습니다.

### 기본 ISA

RISC-V는 **모듈화된 설계**가 특징입니다. 기본 정수 명령어 집합만 필수이고, 나머지는 선택적 확장으로 추가합니다.

| 이름 | 설명 |
|------|------|
| `RV32I` | 32비트 정수 기본 명령어 집합 |
| `RV64I` | 64비트 정수 기본 명령어 집합 |
| `M` | 정수 곱셈/나눗셈 |
| `A` | 원자적 연산 (멀티코어 동기화) |
| `F` | 단정도 부동소수점 |
| `D` | 배정도 부동소수점 |
| `C` | 압축 명령어 (16비트 인코딩) |
| `V` | 벡터 연산 |

`RV64GC`처럼 여러 확장을 조합하여 사용합니다. `G = IMAFD`를 의미합니다.

### 레지스터 구성

RISC-V는 `x0`~`x31`의 32개 정수 레지스터를 가집니다. `x0`는 항상 0을 읽으며 쓰기를 무시하는 특별한 레지스터입니다.

| 레지스터 | ABI 이름 | 역할 |
|----------|----------|------|
| `x0` | `zero` | 하드와이어드 0 |
| `x1` | `ra` | 반환 주소 |
| `x2` | `sp` | 스택 포인터 |
| `x5`~`x7` | `t0`~`t2` | 임시 레지스터 |
| `x8`~`x9` | `s0`~`s1` | 저장 레지스터 |
| `x10`~`x11` | `a0`~`a1` | 함수 인자 / 반환값 |

---

## 코드 예제 1: RISC-V 어셈블리로 피보나치 계산

```asm
# RISC-V 어셈블리: 재귀 없는 피보나치 수열 (n번째 피보나치 수 계산)
# 입력: a0 = n (n >= 0)
# 출력: a0 = fib(n)
#
# 레지스터 사용:
#   t0 = 현재 값 (fib_i)
#   t1 = 이전 값 (fib_i-1)
#   t2 = 루프 카운터
#   t3 = 임시

.section .text
.global fib
fib:
    # n <= 1이면 n 반환
    li      t2, 1
    bge     t2, a0, .return_n   # if n <= 1, return n

    li      t1, 0               # prev = 0 (fib[0])
    li      t0, 1               # curr = 1 (fib[1])
    li      t2, 2               # i = 2

.loop:
    bgt     t2, a0, .done       # if i > n, done
    add     t3, t0, t1          # next = curr + prev
    mv      t1, t0              # prev = curr
    mv      t0, t3              # curr = next
    addi    t2, t2, 1           # i++
    j       .loop

.done:
    mv      a0, t0              # return curr
    ret

.return_n:
    ret                         # return a0 (= n, unchanged)
```

이 코드에서 RISC-V의 특징이 잘 드러납니다. 모든 명령어가 명확하게 레지스터 간 연산을 수행하며(`add t3, t0, t1`), 분기 명령어는 두 레지스터를 직접 비교하여 조건부 점프합니다(`bgt t2, a0, .done`). 메모리 접근 없이 레지스터만으로 연산하는 전형적인 RISC 패턴입니다.

---

## 코드 예제 2: C 코드와 RISC-V 어셈블리 대응 이해

다음 C 코드가 RISC-V에서 어떻게 변환되는지 살펴봅니다.

```c
// C 소스코드
long sum_array(long *arr, int n) {
    long sum = 0;
    for (int i = 0; i < n; i++) {
        sum += arr[i];
    }
    return sum;
}
```

```asm
# RISC-V 어셈블리 (RV64I, -O1 최적화 수준과 유사)
# a0 = long *arr, a1 = int n
# 반환: a0 = sum

sum_array:
    li      a5, 0           # sum = 0
    li      a4, 0           # i = 0
    bge     a4, a1, .done   # if n <= 0, skip loop

.loop:
    slli    a3, a4, 3       # offset = i * 8 (long = 8바이트)
    add     a3, a0, a3      # ptr = arr + offset
    ld      a3, 0(a3)       # *ptr: 메모리에서 8바이트 로드 (LOAD 명령어!)
    add     a5, a5, a3      # sum += *ptr
    addi    a4, a4, 1       # i++
    blt     a4, a1, .loop   # if i < n, continue

.done:
    mv      a0, a5          # return sum
    ret
```

**RISC 로드-스토어 원칙**이 명확합니다. `ld a3, 0(a3)` 명령어만이 메모리를 접근하고, 나머지 연산은 모두 레지스터 간에 이루어집니다. x86이었다면 `add rax, [rbx + rdi*8]` 한 줄로 표현했겠지만, RISC-V는 주소 계산, 로드, 덧셈을 각각의 명령어로 분리합니다.

---

## CISC vs RISC: 현대적 관점

현대 CPU에서는 CISC와 RISC의 경계가 흐려졌습니다. x86 프로세서는 내부적으로 복잡한 CISC 명령어를 **마이크로 연산(μops)**으로 분해하여 RISC처럼 파이프라인에서 처리합니다. 결국 실행 엔진은 RISC 스타일이고, 외부 인터페이스만 CISC인 셈입니다.

반대로 ARM은 RISC이지만 Thumb-2 확장에서 16비트 명령어를 지원하는 등 코드 밀도를 위한 CISC적 요소를 도입했습니다.

RISC-V가 주목받는 이유는 성능보다 **설계 자유도**에 있습니다. IoT 마이크로컨트롤러부터 데이터센터 서버까지, 필요한 확장만 조합하여 최적화된 프로세서를 설계할 수 있습니다. 중국의 T-Head(알리바바), SiFive, Western Digital 등 다양한 기업이 RISC-V 기반 프로세서를 출시하고 있습니다.

---

## 주의사항과 팁

**ISA ≠ 마이크로아키텍처**: ISA는 "무엇을 실행하는가"를 정의하고, 마이크로아키텍처는 "어떻게 실행하는가"를 정의합니다. 같은 RISC-V ISA를 구현하더라도 파이프라인 깊이, 캐시 구성, 슈퍼스칼라 여부에 따라 성능이 천차만별입니다.

**ABI 준수**: 어셈블리를 작성할 때는 ABI(Application Binary Interface) 규약을 반드시 지켜야 합니다. `s0`~`s11`은 호출 전후로 값을 보존해야 하는 callee-saved 레지스터이고, `t0`~`t6`은 caller-saved입니다.

**C 확장 활용**: 임베디드 환경에서 RISC-V를 사용한다면 `C` 확장(16비트 압축 명령어)을 활성화해 코드 크기를 약 25~30% 절감할 수 있습니다.

**RISC-V 시뮬레이터**: 실제 하드웨어 없이 RISC-V를 실험하고 싶다면 QEMU, Spike(공식 시뮬레이터), 또는 온라인 Godbolt 컴파일러 탐색기를 활용하세요.

## 참고 자료
- [RISC-V ISA Manual (riscv/riscv-isa-manual)](https://github.com/riscv/riscv-isa-manual)
- [sail-riscv: RISC-V 공식 형식 명세](https://github.com/riscv/sail-riscv)
- [RISC-V Instruction Set Reference Cheat Sheet](https://github.com/dvoytik/riscv-cheats)
