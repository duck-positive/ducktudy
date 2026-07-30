---
layout: post
title: "LLVM IR과 컴파일러 최적화 패스 완전 정복: 중간 표현이 성능을 만드는 방법"
date: 2026-07-30
categories: [cs, computer-science]
tags: [llvm, compiler, ir, optimization, clang, passes, ssa, inlining, loop-optimization]
---

컴파일러는 소스코드를 기계어로 변환하는 과정에서 **수십~수백 가지 최적화**를 적용한다. 이 최적화들이 실제로 어떻게 동작하는지, 그리고 어떤 자료구조 위에서 실행되는지 이해하면, 더 빠른 코드를 작성하는 직관을 얻을 수 있다. 그 중심에는 **LLVM IR(Intermediate Representation, 중간 표현)**이 있다.

## LLVM이란 무엇인가?

LLVM은 원래 "Low Level Virtual Machine"의 약자였지만, 지금은 범용 컴파일러 인프라스트럭처를 의미한다. Clang(C/C++/Objective-C 프론트엔드), Rust 컴파일러, Swift 컴파일러, Julia, Kotlin Native, WebAssembly 등 수많은 언어의 백엔드로 LLVM이 사용된다.

LLVM의 핵심 아이디어는 **세 단계 분리**다:

```
[소스코드]  →  [프론트엔드]  →  [LLVM IR]  →  [최적화 패스들]  →  [백엔드]  →  [기계어]
  .c/.cpp      Clang           .ll 파일       opt 툴체인          x86/ARM
  .rs          rustc
  .swift       swiftc
```

LLVM IR은 모든 언어가 공통으로 만들어내는 중간 언어다. 프론트엔드는 각 언어를 LLVM IR로 변환하는 데 집중하고, 최적화와 코드 생성은 LLVM이 담당한다. 덕분에 새로운 언어를 만들 때 프론트엔드만 구현하면 LLVM의 강력한 최적화를 바로 활용할 수 있다.

---

## LLVM IR의 구조

LLVM IR은 세 가지 형태로 표현된다:
- **텍스트 형식** (`.ll`): 사람이 읽을 수 있는 어셈블리처럼 생긴 표현
- **비트코드** (`.bc`): 이진 형식의 IR, 링크 타임 최적화(LTO)에 사용
- **메모리 내 표현**: 컴파일러가 실제로 조작하는 C++ 객체 그래프

### IR의 계층 구조

```
Module         (파일 단위)
 └─ Function   (함수)
     └─ BasicBlock  (기본 블록: 분기 없는 순차 명령 묶음)
         └─ Instruction  (개별 명령)
```

**기본 블록(BasicBlock)**은 핵심 개념이다. 기본 블록은 "진입점이 하나, 탈출점이 하나"인 명령 시퀀스다. 기본 블록 내부에는 절대 점프나 분기가 없다. 기본 블록들이 제어 흐름 그래프(Control Flow Graph, CFG)의 노드가 된다.

### 실제 LLVM IR 예시

C 코드:

```c
int add(int a, int b) {
    return a + b;
}

int factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}
```

`clang -S -emit-llvm -O0 example.c -o example.ll` 로 생성한 LLVM IR (간략화):

```llvm
; 모듈 선언
; ModuleID = 'example.c'
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-pc-linux-gnu"

; add 함수
define dso_local i32 @add(i32 noundef %a, i32 noundef %b) {
entry:
  %a.addr = alloca i32, align 4
  %b.addr = alloca i32, align 4
  store i32 %a, ptr %a.addr, align 4
  store i32 %b, ptr %b.addr, align 4
  %0 = load i32, ptr %a.addr, align 4
  %1 = load i32, ptr %b.addr, align 4
  %add = add nsw i32 %0, %1
  ret i32 %add
}

; factorial 함수
define dso_local i32 @factorial(i32 noundef %n) {
entry:
  %retval = alloca i32, align 4
  %n.addr = alloca i32, align 4
  store i32 %n, ptr %n.addr, align 4
  %0 = load i32, ptr %n.addr, align 4
  %cmp = icmp sle i32 %0, 1
  br i1 %cmp, label %if.then, label %if.else  ; 분기 명령

if.then:                                        ; 기본 블록 1
  store i32 1, ptr %retval, align 4
  br label %return                              ; 무조건 점프

if.else:                                        ; 기본 블록 2
  %1 = load i32, ptr %n.addr, align 4
  %2 = load i32, ptr %n.addr, align 4
  %sub = sub nsw i32 %2, 1
  %call = call i32 @factorial(i32 noundef %sub)
  %mul = mul nsw i32 %1, %call
  store i32 %mul, ptr %retval, align 4
  br label %return                              ; 무조건 점프

return:                                         ; 기본 블록 3 (수렴점)
  %3 = load i32, ptr %retval, align 4
  ret i32 %3
}
```

IR에서 눈에 띄는 특징:
- `%변수명` 형식의 가상 레지스터
- `i32`, `ptr` 같은 명시적 타입
- `alloca`로 스택 공간 예약, `load`/`store`로 메모리 접근
- `-O0`(최적화 없음)이라 alloca/load/store가 많다. `-O2`에서는 대부분 제거된다.

---

## SSA(Static Single Assignment) 형식

LLVM IR은 **SSA 형식**을 강제한다. SSA에서는 모든 변수는 **정확히 한 번만 대입**된다. 이 규칙이 수많은 최적화를 단순하게 만든다.

```llvm
; SSA: 각 변수는 단 한 번만 정의된다
%1 = add i32 %a, %b    ; %1에 한 번 대입
%2 = mul i32 %1, 2     ; %2에 한 번 대입
```

같은 변수를 여러 번 대입하려면 새 이름을 써야 한다. 하지만 제어 흐름이 합쳐지는 지점(join point)에서 문제가 생긴다:

```c
int x;
if (cond) x = 1; else x = 2;
use(x);  // x는 1인가? 2인가?
```

이를 해결하는 것이 **φ(Phi) 노드**다:

```llvm
if.then:
  br label %join
if.else:
  br label %join
join:
  %x = phi i32 [ 1, %if.then ], [ 2, %if.else ]
  ; x는 if.then에서 왔으면 1, if.else에서 왔으면 2
```

φ 노드는 SSA의 핵심으로, 어느 경로로 실행이 흘러왔는지에 따라 다른 값을 선택한다. 이 구조 덕분에 데이터 흐름 분석(Def-Use Chain)이 훨씬 단순해진다.

---

## LLVM 최적화 패스의 종류

LLVM의 최적화는 **패스(Pass)**로 구성된다. 각 패스는 IR을 읽어서 분석하거나 변환한다.

패스의 분류:
- **분석 패스(Analysis Pass)**: IR을 읽기만 하고 정보를 수집한다 (수정 없음)
- **변환 패스(Transform Pass)**: IR을 직접 수정하여 최적화한다
- **유틸리티 패스(Utility Pass)**: 출력, 검증 등 보조 작업

### 1. Mem2Reg (메모리를 레지스터로)

가장 먼저 실행되는 중요 패스다. `-O0`에서 생성된 `alloca`/`load`/`store` 패턴을 SSA 레지스터로 변환한다.

```llvm
; Before Mem2Reg (-O0):
entry:
  %x = alloca i32
  store i32 5, ptr %x
  %val = load i32, ptr %x
  %result = add i32 %val, 3
  ret i32 %result

; After Mem2Reg (-O1 이상):
entry:
  %result = add i32 5, 3   ; %x alloca/load/store 전부 제거됨
  ret i32 %result
```

### 2. Constant Folding (상수 접기)

컴파일 시점에 계산 가능한 식을 미리 계산한다.

```llvm
; Before:
%a = mul i32 6, 7
%b = add i32 %a, 2

; After:
%b = add i32 42, 2   ; 6*7 = 42로 교체
; 더 진행하면:
; (상수 propagation 후)
; 결국 44로 교체
```

### 3. Dead Code Elimination (DCE, 죽은 코드 제거)

실행 결과가 사용되지 않는 코드를 제거한다.

```llvm
; Before:
%unused = mul i32 %a, 100  ; 결과가 어디에도 쓰이지 않음
%result = add i32 %a, 1
ret i32 %result

; After:
%result = add i32 %a, 1    ; %unused 제거됨
ret i32 %result
```

### 4. Function Inlining (함수 인라이닝)

짧은 함수 호출을 호출 지점에 직접 삽입한다. 함수 호출 오버헤드(스택 프레임 설정, 레지스터 저장/복원 등)를 제거하고, 이후 다른 패스들이 더 많은 최적화 기회를 얻는다.

```c
// Before inlining
static inline int square(int x) { return x * x; }
int compute(int n) { return square(n) + 1; }

// After inlining
int compute(int n) { return n * n + 1; }  // square 호출 제거
```

LLVM은 함수의 크기(명령 수), 호출 빈도, `-O` 레벨을 고려해 인라이닝 여부를 결정한다.

### 5. Loop Vectorization (루프 벡터화)

루프를 SIMD 명령으로 변환하여 한 번에 여러 원소를 처리한다.

```c
// Before vectorization
void addArrays(float *a, float *b, float *c, int n) {
    for (int i = 0; i < n; i++)
        c[i] = a[i] + b[i];
}
```

LLVM이 `-O2` 이상에서 생성하는 코드는 실제로 `addps` (SSE) 또는 `vaddps` (AVX) 같은 SIMD 명령을 사용하여 4~8개의 float를 동시에 더한다.

### 6. Loop Unrolling (루프 언롤링)

루프 반복 횟수를 줄여 브랜치 오버헤드를 감소시킨다.

```c
// Before: 4번 반복
for (int i = 0; i < 4; i++)
    sum += arr[i];

// After unrolling (LLVM이 자동 변환):
sum += arr[0];
sum += arr[1];
sum += arr[2];
sum += arr[3];
```

---

## 최적화 레벨과 패스 파이프라인

`clang`의 `-O` 플래그는 최적화 패스의 집합을 선택한다:

| 레벨 | 패스 수 | 특징 |
|------|---------|------|
| `-O0` | 최소 | 디버깅 용이, 빠른 컴파일, 최적화 없음 |
| `-O1` | 기본 | Mem2Reg, DCE, 기본 인라이닝 |
| `-O2` | 적극적 | 루프 최적화, 벡터화, 고급 인라이닝 |
| `-O3` | 공격적 | `-O2` + 루프 언롤링, 자동 병렬화 |
| `-Os` | 크기 우선 | 코드 크기 최소화 |
| `-Oz` | 최소 크기 | 가장 작은 이진 파일 |

`opt` 도구로 특정 패스만 수동으로 실행할 수 있다:

```bash
# IR 생성
clang -S -emit-llvm -O0 example.c -o example.ll

# 특정 패스 실행
opt -S -passes="mem2reg,simplifycfg,dce" example.ll -o optimized.ll

# 전체 O2 파이프라인 실행
opt -S -O2 example.ll -o optimized.ll

# 최적화 과정 출력 (디버깅용)
opt -S -O2 --debug-pass=Structure example.ll 2>&1 | head -50
```

---

## 링크 타임 최적화(LTO)

일반 컴파일은 각 소스파일을 독립적으로 최적화한다. 하지만 서로 다른 파일 간 함수 호출은 최적화가 어렵다. LTO는 링크 단계에서 전체 프로그램의 IR을 합쳐서 전역 최적화를 수행한다.

```bash
# Thin LTO 활성화 (확장성이 좋은 LTO 모드)
clang -O2 -flto=thin -c file1.c -o file1.o
clang -O2 -flto=thin -c file2.c -o file2.o
clang -O2 -flto=thin file1.o file2.o -o program

# LTO는 서로 다른 파일의 함수들도 인라이닝 가능
# 전체 프로그램 분석으로 더 공격적인 DCE도 가능
```

---

## 직접 IR 최적화 체험하기

```bash
# 예제 C 코드 작성
cat > test.c << 'EOF'
#include <stdio.h>

static int square(int x) { return x * x; }

int main() {
    int result = 0;
    for (int i = 0; i < 10; i++) {
        result += square(i);  // square가 인라이닝될 예정
    }
    // result는 실제로 컴파일 타임에 285로 계산될 수 있다
    printf("%d\n", result);
    return 0;
}
EOF

# O0 IR 생성
clang -S -emit-llvm -O0 test.c -o test_O0.ll

# O2 IR 생성
clang -S -emit-llvm -O2 test.c -o test_O2.ll

# 차이 비교
diff test_O0.ll test_O2.ll | head -60
```

`-O2` 버전에서는 루프가 풀리고, `square` 함수가 인라이닝되고, 상수 전파에 의해 `result = 285`로 즉시 계산되어 `main`이 거의 빈 함수가 된다.

---

## 주의사항과 팁

1. **volatile을 남용하지 마라**: `volatile` 변수는 컴파일러가 최적화하지 못하도록 막는다. 임베디드/드라이버 코드 외에는 사용하지 않는 것이 좋다.
2. **미정의 동작(Undefined Behavior)**: LLVM은 UB가 없다고 가정하고 공격적으로 최적화한다. C/C++에서 정수 오버플로우, null 포인터 역참조, 범위 초과 접근은 UB이므로, 이를 악용한 코드는 최적화 후 예상과 전혀 다르게 동작할 수 있다.
3. **`__attribute__((noinline))`**: 함수 인라이닝을 명시적으로 막고 싶을 때 사용한다. 프로파일링 시 함수 경계를 유지해야 할 때 유용하다.
4. **PGO(Profile-Guided Optimization)**: 실행 프로파일 정보로 최적화 힌트를 주는 기법. `clang -fprofile-generate`로 실행 데이터를 수집하고, `-fprofile-use`로 이를 활용하면 `-O3`보다 더 좋은 성능을 낼 수 있다.

---

## 참고 자료
- [LLVM Language Reference Manual](https://llvm.org/docs/LangRef.html)
- [LLVM 20.1.0 Language Reference](https://releases.llvm.org/20.1.0/docs/LangRef.html)
- [LLVM Performance Tips for Frontend Authors](https://llvm.org/docs/Frontend/PerformanceTips.html)
- [LLVM - Wikipedia](https://en.wikipedia.org/wiki/LLVM)
