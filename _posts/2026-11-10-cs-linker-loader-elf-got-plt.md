---
layout: post
title: "링커와 로더 완전 정복: ELF 바이너리·GOT·PLT·동적 링킹의 내부 동작 원리"
date: 2026-11-10
categories: [cs, computer-science]
tags: [linker, loader, elf, got, plt, dynamic-linking, shared-library, relocation, symbol-resolution]
---

## 링커와 로더란 무엇인가?

소스 코드를 작성하고 컴파일하면 `.o`(오브젝트 파일)가 생성됩니다. 여러 오브젝트 파일과 라이브러리를 하나의 실행 파일로 묶는 것이 **링커(Linker)**의 역할이고, 그 실행 파일을 메모리에 올려 프로세스를 시작하는 것이 **로더(Loader)**의 역할입니다.

이 과정은 현대 개발자에게는 대부분 투명하게 처리되지만, 내부를 이해하면 다음과 같은 문제를 근본적으로 해결할 수 있습니다:

- `undefined reference to ...` 링크 에러의 원인 파악
- `symbol lookup error: undefined symbol` 런타임 에러 해결
- 라이브러리 충돌, 버전 불일치 디버깅
- 공유 라이브러리 제작 및 `LD_PRELOAD` 활용
- 성능 최적화: LTO(Link-Time Optimization), 프로파일 가이드 최적화

## 왜 링킹이 필요한가?

컴파일러는 하나의 번역 단위(translation unit, 즉 `.c` 또는 `.cpp` 파일)를 독립적으로 처리합니다. `printf`는 C 표준 라이브러리에 있고, 여러분의 `main.c`는 그 위치를 모릅니다. 링커가 이 심볼(symbol) 참조를 실제 주소로 연결합니다.

```
main.o          libc.a            실행파일
+----------+    +----------+      +-----------+
|  main()  | +->| printf() |  ->  | main()    |
| printf?? |    | malloc() |      | printf()  |
+----------+    +----------+      | malloc()  |
                                  +-----------+
```

## ELF 바이너리 형식

Linux의 실행 파일, 공유 라이브러리, 오브젝트 파일은 모두 **ELF(Executable and Linkable Format)**를 사용합니다.

### ELF 헤더 구조

```c
// /usr/include/elf.h (요약)
typedef struct {
    unsigned char e_ident[16];  // 매직 넘버 \x7fELF + 아키텍처 정보
    uint16_t      e_type;       // ET_EXEC(실행파일), ET_DYN(공유라이브러리), ET_REL(.o)
    uint16_t      e_machine;    // EM_X86_64, EM_AARCH64 등
    uint32_t      e_version;    // 항상 1
    uint64_t      e_entry;      // 진입점 가상 주소 (_start 함수)
    uint64_t      e_phoff;      // 프로그램 헤더 테이블 오프셋
    uint64_t      e_shoff;      // 섹션 헤더 테이블 오프셋
    // ...
} Elf64_Ehdr;
```

매직 넘버 확인:
```bash
$ xxd /bin/ls | head -2
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
# 7f 45 4c 46 = \x7f E L F
```

### 주요 ELF 섹션

| 섹션 | 역할 |
|------|------|
| `.text` | 실행 코드 (읽기 전용, 실행 가능) |
| `.rodata` | 문자열 리터럴 등 읽기 전용 데이터 |
| `.data` | 초기화된 전역/정적 변수 |
| `.bss` | 초기화되지 않은 전역/정적 변수 (파일에는 크기만 기록) |
| `.symtab` | 심볼 테이블 (디버깅용, strip 가능) |
| `.dynsym` | 동적 링킹용 심볼 테이블 |
| `.rel.text` / `.rela.text` | 재배치 엔트리 (.text 섹션의 수정 위치) |
| `.got` | Global Offset Table |
| `.plt` | Procedure Linkage Table |
| `.dynamic` | 동적 링커 정보 |

## 정적 링킹 vs 동적 링킹

### 정적 링킹

```bash
# 정적 링킹: libc.a를 바이너리에 포함
gcc -static -o hello_static hello.c
ls -lh hello_static  # 약 800KB ~ 2MB
```

**장점**: 의존성 없음, 배포 단순, 미세하게 더 빠른 함수 호출  
**단점**: 바이너리 크기 증가, 라이브러리 업데이트 시 재컴파일 필요

### 동적 링킹

```bash
# 동적 링킹 (기본값): libc.so를 런타임에 연결
gcc -o hello_dynamic hello.c
ldd hello_dynamic     # 의존 공유 라이브러리 목록
# linux-vdso.so.1 (공유 라이브러리 로딩 도구)
# libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
# /lib64/ld-linux-x86-64.so.2 (동적 링커 자체)
```

## 실제 구현 예제

### 예제 1: GOT와 PLT — 지연 바인딩의 작동 원리

동적 링킹의 핵심 메커니즘은 **GOT(Global Offset Table)**와 **PLT(Procedure Linkage Table)**입니다. `printf` 같은 외부 함수의 주소는 프로그램 시작 시점이 아닌, 첫 호출 시점에 해석됩니다(**지연 바인딩, lazy binding**).

```c
// main.c
#include <stdio.h>

int main(void) {
    printf("Hello, World!\n");   // 첫 번째 호출
    printf("Second call\n");     // 두 번째 호출
    return 0;
}
```

내부적으로 컴파일러는 `printf`를 `printf@plt`(PLT 엔트리)로 대체합니다:

```
[첫 번째 printf 호출]
1. call printf@plt          → PLT 엔트리로 점프
2. PLT: jmp [printf@got]    → GOT 엔트리 주소로 점프
3. GOT 초기값 = 다음 PLT 명령 주소 (아직 미해석)
4. PLT: push 함수 인덱스    → 동적 링커에 "printf 주소 알려줘"
5. jmp _dl_runtime_resolve  → ld.so가 printf 실제 주소 탐색
6. ld.so: GOT[printf] = 실제 printf 주소  → GOT 업데이트
7. printf 실행

[두 번째 printf 호출]
1. call printf@plt
2. PLT: jmp [printf@got]    → GOT에 이미 실제 주소가 있음
3. printf 직접 실행         (ld.so 개입 없음)
```

`objdump`로 직접 확인:

```bash
$ objdump -d hello_dynamic | grep -A 5 "plt"
Disassembly of section .plt:
0000000000401020 <printf@plt>:
  401020: ff 25 ea 2f 00 00  jmp    *0x2fea(%rip)  # 404010 <printf@GLIBC_2.2.5>
  401026: 68 00 00 00 00     push   $0x0
  40102b: e9 e0 ff ff ff     jmp    401010 <_init+0x10>

$ objdump -R hello_dynamic
OFFSET           TYPE              VALUE
0000000000404010 R_X86_64_JUMP_SLOT  printf@GLIBC_2.2.5
```

### 예제 2: 공유 라이브러리 제작과 `LD_PRELOAD` 함수 훅킹

`LD_PRELOAD`는 동적 링커에게 "이 라이브러리를 다른 모든 라이브러리보다 먼저 로드하라"고 지시합니다. 같은 이름의 심볼이 있으면 LD_PRELOAD의 것이 우선합니다.

```c
// malloc_hook.c: malloc 호출을 추적하는 훅
#define _GNU_SOURCE
#include <stdio.h>
#include <dlfcn.h>
#include <stdlib.h>

static void* (*real_malloc)(size_t) = NULL;

static void init_hook(void) __attribute__((constructor));
static void init_hook(void) {
    real_malloc = dlsym(RTLD_NEXT, "malloc");
    if (!real_malloc) {
        fprintf(stderr, "dlsym 실패: %s\n", dlerror());
        exit(1);
    }
}

void* malloc(size_t size) {
    void* ptr = real_malloc(size);
    fprintf(stderr, "[MALLOC] size=%zu -> ptr=%p\n", size, ptr);
    return ptr;
}
```

빌드 및 사용:
```bash
# 공유 라이브러리로 빌드 (-fPIC: Position Independent Code 필수)
gcc -shared -fPIC -o malloc_hook.so malloc_hook.c -ldl

# 대상 프로그램에 훅 적용 (소스 수정 없음)
LD_PRELOAD=./malloc_hook.so ls
# [MALLOC] size=552 -> ptr=0x55a8f1b2f260
# [MALLOC] size=120 -> ptr=0x55a8f1b2f4b0
```

이 기법은 Valgrind, tcmalloc, jemalloc 등이 메모리 추적을 구현하는 방식과 동일한 원리입니다.

## 재배치(Relocation)의 내부 동작

오브젝트 파일의 `.o`은 최종 주소를 모른 채 컴파일됩니다. 링커가 최종 주소를 결정한 뒤, 재배치 엔트리에 따라 바이너리를 수정합니다.

```bash
$ objdump -r main.o
RELOCATION RECORDS FOR [.text]:
OFFSET   TYPE              VALUE
000000d  R_X86_64_PLT32    printf-0x0000000000000004
0000019  R_X86_64_PLT32    my_function-0x0000000000000004

# 타입별 의미:
# R_X86_64_32:       절대 주소 (정적 링킹)
# R_X86_64_PC32:     PC 상대 주소 (정적 링킹)
# R_X86_64_PLT32:    PLT를 통한 호출 (동적 링킹)
# R_X86_64_GLOB_DAT: 전역 변수 (GOT를 통한 접근)
```

## 심볼 가시성과 최적화

공유 라이브러리를 제작할 때 불필요한 심볼을 외부에 노출하면 성능과 보안에 문제가 됩니다.

```c
__attribute__((visibility("default")))
void public_api(void) { /* 외부에서 접근 가능 */ }

__attribute__((visibility("hidden")))
void internal_helper(void) { /* 라이브러리 내부용 */ }
```

```bash
# 기본을 hidden으로 설정하고, default만 노출
gcc -shared -fPIC -fvisibility=hidden -o libmylib.so mylib.c
```

## 주의사항과 팁

### 1. RPATH vs LD_LIBRARY_PATH

공유 라이브러리 탐색 순서:
1. `DT_RPATH` (바이너리에 하드코딩, 현재는 거의 사용 안 함)
2. `LD_LIBRARY_PATH` 환경 변수
3. `DT_RUNPATH` (바이너리에 하드코딩, 현재 권장)
4. `/etc/ld.so.cache` (ldconfig가 생성)
5. `/lib`, `/usr/lib`

```bash
# RUNPATH를 바이너리에 포함 (배포 시 유용)
gcc -Wl,-rpath,'$ORIGIN/../lib' -o myapp main.c -L./lib -lmylib
# $ORIGIN: 실행 파일의 디렉터리를 의미하는 특수 변수
```

### 2. 링크 타임 최적화(LTO)

LTO는 링킹 단계에서 여러 번역 단위를 통합하여 인라이닝, 데드 코드 제거 등 추가 최적화를 적용합니다:

```bash
gcc -O2 -flto -c a.c -o a.o
gcc -O2 -flto -c b.c -o b.o
gcc -O2 -flto a.o b.o -o myapp  # 링커가 IR을 통합 최적화
```

대규모 프로젝트(Chrome, Firefox, Linux 커널)에서 LTO로 5~15% 성능 향상을 보고하고 있습니다.

### 3. `nm`, `readelf`, `objdump`로 바이너리 분석

```bash
nm -D libmylib.so    # 동적 심볼 (T=코드, U=미정의, D=데이터)
readelf -a myapp | less
objdump -d -M intel myapp | less
readelf -d myapp | grep NEEDED
```

## 참고 자료

- [System V ABI (ELF 명세)](https://refspecs.linuxbase.org/elf/elf.pdf)
- [ld.so(8) Linux 매뉴얼 페이지](https://man7.org/linux/man-pages/man8/ld.so.8.html)
- [Linkers and Loaders — John Levine](https://linker.iecc.com/)
- [How the ELF Ruined Christmas (PLT/GOT 시각화)](https://www.technovelty.org/linux/plt-and-got-the-key-to-code-sharing-and-dynamic-libraries.html)
