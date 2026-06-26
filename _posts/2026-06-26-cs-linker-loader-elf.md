---
layout: post
title: "링커와 로더 완전 정복: 실행 파일이 탄생하는 과정과 동적 링킹의 비밀"
date: 2026-06-26
categories: [cs, computer-science]
tags: [linker, loader, elf, dynamic-linking, shared-library, got, plt, symbol-resolution, relocation]
---

## 개요

`gcc hello.c -o hello`를 입력하면 실행 파일이 생긴다. 하지만 이 단순한 명령어 뒤에는 전처리기, 컴파일러, 어셈블러, **링커**라는 네 단계의 복잡한 파이프라인이 숨어 있다. 그리고 프로그램을 실행할 때는 운영체제의 **로더**와 **동적 링커**가 개입한다. 이 글에서는 소스 코드에서 실행 파일까지, 그리고 실행 파일에서 메모리 위의 프로세스까지 이어지는 전 과정을 C 코드와 실제 바이너리 분석을 통해 완전히 해부한다.

---

## 1. 컴파일 파이프라인 4단계

`hello.c` 하나를 컴파일하는 과정을 단계별로 살펴보자.

### 1.1 전처리 (Preprocessing)

```bash
gcc -E hello.c -o hello.i
```

`#include`, `#define`, 매크로를 처리한다. `stdio.h`의 내용이 인라인으로 삽입되어 수천 줄짜리 `.i` 파일이 만들어진다.

### 1.2 컴파일 (Compilation)

```bash
gcc -S hello.i -o hello.s
```

C 소스를 어셈블리 코드(`.s`)로 변환한다. 이 단계에서 최적화(`-O2`, `-O3`)가 적용된다.

### 1.3 어셈블리 (Assembly)

```bash
gcc -c hello.s -o hello.o
```

어셈블리 코드를 기계어로 변환하여 **오브젝트 파일(`.o`)**을 생성한다. 아직 완전한 실행 파일이 아니다. 외부 함수(`printf` 등)의 주소가 미결정 상태(unresolved)로 남아 있다.

### 1.4 링킹 (Linking)

```bash
gcc hello.o -o hello        # 동적 링킹 (기본)
gcc hello.o -static -o hello_static  # 정적 링킹
```

링커(`ld`)가 여러 오브젝트 파일과 라이브러리를 하나로 합쳐 실행 파일을 만든다. 미결정 심볼(symbol)의 주소를 해결하는 과정이 **심볼 해석(Symbol Resolution)** 과 **재배치(Relocation)** 다.

---

## 2. ELF 파일 형식 해부

리눅스 실행 파일은 **ELF(Executable and Linkable Format)** 형식을 사용한다. ELF는 오브젝트 파일, 실행 파일, 공유 라이브러리, 코어 덤프를 모두 같은 형식으로 표현한다.

```
ELF 파일 구조
┌─────────────────────────┐
│      ELF Header         │  ← 파일 타입, 아키텍처, 진입점(entry point) 주소
├─────────────────────────┤
│   Program Headers       │  ← 로더에게 세그먼트를 메모리에 어떻게 올릴지 지시
│  (Segment 정보, 실행 시) │
├─────────────────────────┤
│  .text  (코드 섹션)      │  ← 실행 가능한 기계어 코드
├─────────────────────────┤
│  .data  (초기화 데이터)  │  ← 초기값 있는 전역/정적 변수
├─────────────────────────┤
│  .bss   (미초기화 데이터)│  ← 초기값 없는 전역/정적 변수 (파일엔 크기만)
├─────────────────────────┤
│  .rodata (읽기전용 데이터)│  ← 문자열 리터럴 등
├─────────────────────────┤
│  .plt   (프로시저 링킹 테이블) │  ← 동적 링킹 stub 코드
├─────────────────────────┤
│  .got   (전역 오프셋 테이블)  │  ← 동적 심볼 주소 저장
├─────────────────────────┤
│  .dynsym / .dynstr      │  ← 동적 심볼 테이블 및 문자열
├─────────────────────────┤
│  Section Headers        │  ← 링커에게 섹션 위치/크기 정보 제공
└─────────────────────────┘
```

실제 파일을 분석해보자.

```bash
# ELF 헤더 확인
readelf -h hello

# 섹션 목록 확인
readelf -S hello

# 심볼 테이블 확인 (정적 심볼)
nm hello

# 동적 심볼 확인
nm -D hello

# 의존 공유 라이브러리 확인
ldd hello

# 어셈블리 역어셈블
objdump -d hello | head -50

# 동적 섹션 (.got, .plt) 확인
objdump -d -j .plt hello
readelf -r hello  # 재배치 테이블
```

---

## 3. 정적 링킹 vs 동적 링킹

### 3.1 정적 링킹

컴파일 시 모든 라이브러리 코드가 실행 파일에 포함된다.

```c
// math_static_demo.c — 정적 링킹으로 빌드하는 예시
#include <stdio.h>
#include <math.h>

int main(void) {
    double x = 2.0;
    double result = sqrt(x);
    printf("sqrt(%.1f) = %.6f\n", x, result);
    return 0;
}
```

```bash
# 정적 링킹: libm.a의 sqrt 코드가 실행 파일에 포함된다
gcc math_static_demo.c -o math_static -static -lm

# 동적 링킹: libm.so를 런타임에 로드
gcc math_static_demo.c -o math_dynamic -lm

ls -lh math_static math_dynamic
# math_static: ~900KB (libc, libm 포함)
# math_dynamic: ~16KB
```

| 비교 항목 | 정적 링킹 | 동적 링킹 |
|-----------|-----------|-----------|
| 파일 크기 | 크다 | 작다 |
| 실행 속도 | 약간 빠름 (로드 오버헤드 없음) | 첫 호출 시 약간 느림 |
| 메모리 공유 | 불가 (각 프로세스가 별도 복사본) | 가능 (공유 라이브러리를 여러 프로세스가 공유) |
| 배포 | 의존성 없음 | 공유 라이브러리 버전 관리 필요 |
| 보안 패치 | 재컴파일 필요 | so 교체만으로 즉시 적용 |

---

## 4. 동적 링킹의 비밀: GOT와 PLT

동적 링킹에서 가장 중요한 구조가 **GOT(Global Offset Table)** 과 **PLT(Procedure Linkage Table)** 다.

### 4.1 왜 GOT/PLT가 필요한가?

공유 라이브러리(`.so`)는 메모리의 **어떤 주소**에 로드될지 미리 알 수 없다(ASLR 때문). 따라서 `printf`의 실제 주소를 컴파일 타임에 실행 파일에 하드코딩할 수 없다.

해결책: 실행 파일은 `printf`의 주소를 담을 **빈 슬롯(GOT 엔트리)**을 만들어두고, 런타임에 동적 링커가 이 슬롯을 채운다.

### 4.2 Lazy Binding: 처음 호출될 때 주소를 결정

성능 최적화를 위해 리눅스의 동적 링커는 기본적으로 **Lazy Binding**을 사용한다. 함수를 처음 호출할 때만 심볼을 해석한다.

```
printf() 첫 번째 호출 흐름:

1. main() → call printf@plt
2. PLT stub 실행:
   - GOT[printf] 값 확인 → 아직 채워지지 않음 (PLT+6 주소 가리킴)
   - push <printf의 릴로케이션 인덱스>
   - jmp to _dl_runtime_resolve
3. 동적 링커(_dl_runtime_resolve):
   - libc.so 내에서 printf 심볼 탐색
   - 실제 printf 주소를 GOT[printf]에 저장
   - printf 실행

두 번째 호출:
1. main() → call printf@plt
2. PLT stub:
   - GOT[printf] → 이미 실제 주소가 채워져 있음
   - jmp to printf (직접 점프, _dl_runtime_resolve 불필요)
```

실제로 GOT/PLT를 확인해보자.

```bash
# PLT 섹션 역어셈블 — printf@plt stub을 확인
objdump -d -j .plt math_dynamic

# 예시 출력:
# 0000000000401020 <printf@plt>:
#   401020: ff 25 ea 2f 00 00    jmp *0x2fea(%rip)   # GOT[printf] 참조
#   401026: 68 00 00 00 00       push $0x0            # 릴로케이션 인덱스
#   40102b: e9 e0 ff ff ff       jmp 401010           # _dl_runtime_resolve로 점프
```

---

## 5. 공유 라이브러리 직접 만들기

```c
// mymath.c — 공유 라이브러리 소스
#include "mymath.h"
#include <math.h>

// 이 심볼은 외부에서 사용 가능
double circle_area(double radius) {
    return M_PI * radius * radius;
}

// 이 심볼은 라이브러리 내부에서만 사용 (숨김)
__attribute__((visibility("hidden")))
static double _internal_helper(double x) {
    return x * x;
}

double hypotenuse(double a, double b) {
    return sqrt(_internal_helper(a) + _internal_helper(b));
}
```

```c
// mymath.h
#ifndef MYMATH_H
#define MYMATH_H

double circle_area(double radius);
double hypotenuse(double a, double b);

#endif
```

```c
// main_dynamic.c — 공유 라이브러리 사용
#include <stdio.h>
#include "mymath.h"

int main(void) {
    printf("원 넓이(r=5): %.4f\n", circle_area(5.0));
    printf("빗변(3,4):    %.4f\n", hypotenuse(3.0, 4.0));
    return 0;
}
```

```bash
# 1. 위치 독립 코드(PIC)로 오브젝트 파일 생성
gcc -fPIC -c mymath.c -o mymath.o

# 2. 공유 라이브러리 생성
#    -shared: 공유 라이브러리
#    -Wl,-soname,libmymath.so.1: soname 설정 (버전 관리)
gcc -shared -fPIC mymath.o -o libmymath.so.1.0 -lm \
    -Wl,-soname,libmymath.so.1

# 3. soname 심볼릭 링크 생성 (관례)
ln -sf libmymath.so.1.0 libmymath.so.1
ln -sf libmymath.so.1   libmymath.so

# 4. 실행 파일 빌드 (현재 디렉토리에서 라이브러리 탐색)
gcc main_dynamic.c -L. -lmymath -o main_app

# 5. 현재 디렉토리를 라이브러리 탐색 경로에 추가해 실행
LD_LIBRARY_PATH=. ./main_app

# 6. 심볼 확인
nm -D libmymath.so | grep -E "circle|hypotenuse|internal"
# U sqrt                (undefined — libc에서 가져옴)
# T circle_area         (T = text section, 외부 공개)
# T hypotenuse          (T = text section, 외부 공개)
# (내부 심볼 _internal_helper는 visibility=hidden으로 보이지 않음)

# 7. 의존성 확인
ldd main_app
# libmymath.so.1 => ./libmymath.so.1
# libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
```

### PIC(Position Independent Code)가 필요한 이유

공유 라이브러리는 여러 프로세스의 가상 주소 공간에 각기 다른 주소로 매핑된다. PIC는 절대 주소 대신 **RIP 상대 주소** 또는 **GOT를 통한 간접 참조**를 사용하여 어떤 주소에 로드되더라도 올바르게 동작한다.

---

## 6. 동적 로더의 실행 과정

`./main_app`을 실행하면 커널과 동적 링커가 어떤 순서로 작동하는지 추적해보자.

```bash
# strace로 실행 파일 로딩 과정 추적
strace -e trace=openat,mmap,mprotect ./main_app 2>&1 | head -30

# 주요 이벤트:
# 1. execve("./main_app", ...) → 커널이 ELF 헤더 파싱
# 2. 인터프리터 섹션 확인: /lib64/ld-linux-x86-64.so.2 (동적 링커)
# 3. 동적 링커가 메모리에 로드됨
# 4. 동적 링커가 .dynamic 섹션 파싱 → 필요한 .so 목록 확인
# 5. openat: libmymath.so.1 열기
# 6. mmap: 공유 라이브러리를 메모리에 매핑
# 7. 재배치(Relocation) 처리: GOT 슬롯 채우기
# 8. main() 실행 시작

# LD_DEBUG로 심볼 해석 과정 상세 확인
LD_DEBUG=bindings LD_LIBRARY_PATH=. ./main_app 2>&1 | head -20
```

---

## 7. 주의사항 및 실전 팁

### 7.1 RPATH vs LD_LIBRARY_PATH

`LD_LIBRARY_PATH` 환경변수는 보안 위험이 있다(권한 상승 취약점). 배포 환경에서는 빌드 시 RPATH를 실행 파일에 내장하는 것이 더 안전하다.

```bash
# RPATH를 실행 파일에 내장 ($ORIGIN = 실행 파일이 있는 디렉토리)
gcc main_dynamic.c -L. -lmymath -o main_app \
    -Wl,-rpath,'$ORIGIN/lib'
```

### 7.2 심볼 충돌과 심볼 가시성

여러 공유 라이브러리에 같은 이름의 심볼이 있으면 먼저 로드된 라이브러리의 심볼이 선택된다. `__attribute__((visibility("hidden")))` 또는 `-fvisibility=hidden` 빌드 플래그로 라이브러리 내부 심볼을 숨기면 충돌을 방지하고 로딩 속도도 개선된다.

### 7.3 dlopen으로 런타임 플러그인 로딩

```c
#include <dlfcn.h>

// 런타임에 플러그인 로드 (플러그인 시스템 구현)
void* handle = dlopen("./plugin.so", RTLD_LAZY | RTLD_LOCAL);
if (!handle) {
    fprintf(stderr, "dlopen: %s\n", dlerror());
    return 1;
}

typedef int (*plugin_func_t)(const char*);
plugin_func_t func = (plugin_func_t)dlsym(handle, "plugin_run");

func("hello from plugin");
dlclose(handle);
```

---

## 8. 정리

링커는 컴파일 타임에 오브젝트 파일들을 합치고 심볼을 해석한다. 로더와 동적 링커는 런타임에 실행 파일을 메모리에 올리고 공유 라이브러리를 로드한다. GOT와 PLT는 위치 독립적인 코드와 Lazy Binding을 가능하게 하는 핵심 메커니즘이다.

`gcc hello.c -o hello`라는 한 줄 명령어가 전처리 → 컴파일 → 어셈블 → 링크라는 네 단계를 거치고, 실행 시에는 커널 → 동적 링커 → 공유 라이브러리 로드 → main()이라는 복잡한 과정을 통해 프로그램이 시작된다. 이 과정을 이해하면 공유 라이브러리 충돌, 느린 시작 시간, 보안 취약점 등 실무에서 마주치는 많은 문제를 근본적으로 해결할 수 있다.

## 참고 자료
- [ld.so(8) — Linux Dynamic Linker Manual](https://linux.die.net/man/8/ld-linux)
- [ld-linux(8) — Linux man page](https://linux.die.net/man/8/ld-linux)
- [A look at dynamic linking — LWN.net](https://lwn.net/Articles/961117/)
- [Dynamic Linker — OSDev Wiki](https://wiki.osdev.org/Dynamic_Linker)
