---
layout: post
title: "WebAssembly 런타임 내부 구조 완전 정복: 바이너리 포맷·스택 머신·WASI의 작동 원리"
date: 2026-10-19
categories: [cs, computer-science]
tags: [webassembly, wasm, wasi, runtime, stack-machine, binary-format, jit, compiler]
---

WebAssembly(Wasm)는 2019년 W3C 공식 표준이 된 이후 브라우저를 넘어 서버, 엣지 컴퓨팅, 플러그인 시스템 등 다양한 영역으로 확산되고 있습니다. "한 번 컴파일하면 어디서든 실행된다"는 약속은 매력적이지만, 그 내부에서 어떤 일이 벌어지는지 이해하는 개발자는 많지 않습니다. 이 글에서는 Wasm 바이너리 포맷, 스택 머신 실행 모델, WASI 인터페이스, 그리고 JIT 컴파일 최적화까지 런타임 전체를 해부합니다.

## 왜 WebAssembly인가

기존 JavaScript는 동적 타이핑, GC 오버헤드, 최적화 어려움 등으로 연산 집약적 작업에서 한계를 보였습니다. Wasm은 이를 해결하기 위해 설계된 저수준 바이트코드 포맷으로, 다음 특성을 갖습니다.

- **컴팩트한 바이너리 포맷**: JavaScript 소스보다 평균 60~70% 작음
- **빠른 디코딩**: 병렬 디코딩 가능한 구조로 파싱 시간 최소화
- **타입 안전성**: 정적 타입 검증으로 JIT 최적화 수월
- **샌드박스 격리**: 선형 메모리 모델로 호스트 메모리 보호
- **이식성**: CPU 아키텍처 독립적 IR로 크로스 플랫폼 실행

## Wasm 바이너리 포맷 구조

Wasm 모듈은 섹션(section)들의 연속으로 구성됩니다. 각 섹션은 1바이트 ID, LEB128 인코딩된 길이, 그리고 본문으로 이루어집니다.

```
Magic Number: 0x00 0x61 0x73 0x6D  (\0asm)
Version:      0x01 0x00 0x00 0x00  (1)

섹션 목록:
 1  Type Section      — 함수 시그니처 목록 (파라미터·반환 타입)
 2  Import Section    — 외부에서 가져올 함수·메모리·테이블·전역
 3  Function Section  — 각 함수가 Type Section의 몇 번 시그니처를 쓰는지
 4  Table Section     — 참조 타입(funcref, externref) 테이블
 5  Memory Section    — 선형 메모리 크기 선언 (64 KB 페이지 단위)
 6  Global Section    — 전역 변수 선언
 7  Export Section    — 호스트에 노출할 이름 맵
 8  Start Section     — 모듈 인스턴스화 시 자동 실행할 함수
 9  Element Section   — 테이블 초기화 데이터
10  Code Section      — 함수 본문 (지역 변수 + 명령어 시퀀스)
11  Data Section      — 메모리 초기화 데이터
12  DataCount Section — Data Section 개수 (스트리밍 검증용)
```

### LEB128 인코딩

정수는 LEB128(Little Endian Base 128)로 인코딩되어 가변 길이를 가집니다. 작은 수는 1바이트, 큰 수는 여러 바이트를 사용해 바이너리를 압축합니다.

```python
def encode_uleb128(value: int) -> bytes:
    """부호 없는 LEB128 인코딩"""
    result = []
    while True:
        byte = value & 0x7F   # 하위 7비트 추출
        value >>= 7
        if value != 0:
            byte |= 0x80      # 다음 바이트 있음 플래그
        result.append(byte)
        if value == 0:
            break
    return bytes(result)

def decode_uleb128(data: bytes, offset: int) -> tuple[int, int]:
    """LEB128 디코딩, (값, 읽은 바이트 수) 반환"""
    result = 0
    shift = 0
    for i, byte in enumerate(data[offset:]):
        result |= (byte & 0x7F) << shift
        shift += 7
        if not (byte & 0x80):   # MSB가 0이면 종료
            return result, i + 1
    raise ValueError("LEB128 종료 바이트 없음")

# 예: 300 → [0xAC, 0x02]
print(encode_uleb128(300).hex())  # ac02
val, n = decode_uleb128(bytes([0xAC, 0x02]), 0)
print(val, n)  # 300, 2
```

## 스택 머신 실행 모델

Wasm은 레지스터 머신이 아닌 **스택 머신**입니다. 모든 연산은 피연산자를 스택에서 꺼내고 결과를 스택에 넣는 방식으로 동작합니다.

### 명령어 실행 흐름

```
(module
  (func $add (param i32 i32) (result i32)
    local.get 0   ;; 스택: [a]
    local.get 1   ;; 스택: [a, b]
    i32.add       ;; 스택: [a+b]  (두 값 소비 → 합 생성)
  )
)
```

Wasm 명령어는 약 200여 가지이며 카테고리별로 분류됩니다:

| 카테고리 | 예시 명령어 |
|---|---|
| 숫자 연산 | `i32.add`, `f64.mul`, `i64.div_u` |
| 비교 | `i32.lt_s`, `f32.eq`, `i32.eqz` |
| 변환 | `i32.wrap_i64`, `f64.convert_i32_s` |
| 메모리 | `i32.load`, `i64.store`, `memory.grow` |
| 제어 흐름 | `block`, `loop`, `if`, `br`, `call` |
| 테이블 | `table.get`, `call_indirect` |
| 참조 | `ref.null`, `ref.func` |

### 구조화된 제어 흐름

Wasm은 임의 점프(goto)를 허용하지 않습니다. 모든 분기는 `block`, `loop`, `if` 구조 안에서 레이블 기반으로 동작합니다. 이는 타입 검증과 JIT 최적화를 단순화합니다.

```wasm
;; C: for(int i=0; i<10; i++) sum += i;
(func $sum_1_to_10 (result i32)
  (local i32 i32)        ;; local 0 = sum, local 1 = i
  (loop $loop
    local.get 1          ;; i
    i32.const 10
    i32.lt_s             ;; i < 10?
    (if
      (then
        local.get 0
        local.get 1
        i32.add
        local.set 0      ;; sum += i
        local.get 1
        i32.const 1
        i32.add
        local.set 1      ;; i++
        br $loop         ;; 루프 계속
      )
    )
  )
  local.get 0            ;; return sum
)
```

## 검증(Validation) 단계

런타임은 코드를 실행하기 전에 반드시 검증 단계를 거칩니다. 검증은 타입 스택을 추적하며 단 **한 번의 선형 패스**로 완료됩니다.

```python
class WasmValidator:
    """단순화된 Wasm 타입 스택 검증기"""
    
    def __init__(self):
        self.type_stack: list[str] = []
        self.errors: list[str] = []
    
    def push(self, type_: str):
        self.type_stack.append(type_)
    
    def pop(self, expected: str) -> bool:
        if not self.type_stack:
            self.errors.append(f"스택 언더플로: {expected} 기대")
            return False
        actual = self.type_stack.pop()
        if actual != expected:
            self.errors.append(f"타입 불일치: {expected} 기대, {actual} 실제")
            return False
        return True
    
    def validate_i32_add(self):
        """i32.add: [i32, i32] → [i32]"""
        self.pop("i32")
        self.pop("i32")
        self.push("i32")
    
    def validate_f64_mul(self):
        """f64.mul: [f64, f64] → [f64]"""
        self.pop("f64")
        self.pop("f64")
        self.push("f64")
    
    def validate_i32_load(self, align: int, offset: int):
        """i32.load: [i32 주소] → [i32 값]"""
        if align > 2:  # i32는 최대 4바이트 정렬
            self.errors.append(f"잘못된 정렬: {align}")
            return
        self.pop("i32")  # 주소
        self.push("i32")  # 로드된 값
    
    def is_valid(self) -> bool:
        return len(self.errors) == 0

validator = WasmValidator()
validator.push("i32")
validator.push("i32")
validator.validate_i32_add()
print(validator.type_stack)  # ['i32']
print(validator.is_valid())  # True
```

## 선형 메모리 모델

Wasm의 메모리는 64KB 페이지 단위로 관리되는 **선형(flat) 바이트 배열**입니다. 인덱스는 32비트 정수(Memory64 확장에서는 64비트)로, 범위를 벗어난 접근은 즉시 트랩(trap)을 발생시킵니다.

```
Wasm 선형 메모리 레이아웃 (일반적 패턴):
┌─────────────────────────────────────────┐
│  0x0000 ~ 0x00FF  시스템 예약/null 감지  │
│  0x0100 ~         정적 데이터 (Data Seg) │
│         ~ stack_top  스택 (하향 성장)    │
│  heap_base ~      힙 (상향 성장)         │
│         ~ memory_end  미할당            │
└─────────────────────────────────────────┘
```

```rust
// Rust WebAssembly 코드: 선형 메모리를 직접 조작
use std::alloc::{alloc, dealloc, Layout};

#[no_mangle]
pub extern "C" fn wasm_alloc(size: usize) -> *mut u8 {
    let layout = Layout::from_size_align(size, 8).unwrap();
    unsafe { alloc(layout) }
}

#[no_mangle]
pub extern "C" fn wasm_dealloc(ptr: *mut u8, size: usize) {
    let layout = Layout::from_size_align(size, 8).unwrap();
    unsafe { dealloc(ptr, layout) }
}

// 메모리 내 문자열 처리
#[no_mangle]
pub extern "C" fn string_length(ptr: *const u8) -> usize {
    let mut len = 0;
    unsafe {
        while *ptr.add(len) != 0 {
            len += 1;
        }
    }
    len
}
```

## WASI: WebAssembly System Interface

WASI는 Wasm 모듈이 파일 시스템, 네트워크, 시계 등 OS 기능에 접근할 수 있도록 표준화된 인터페이스입니다. 핵심 설계 원칙은 **capability-based security**로, 모듈은 명시적으로 전달받은 파일 디스크립터 외에는 아무것도 접근할 수 없습니다.

```
WASI Preview 1 주요 syscall:
  fd_read(fd, iovs, nread)           — 파일 디스크립터에서 읽기
  fd_write(fd, iovs, nwritten)       — 파일 디스크립터에 쓰기
  fd_seek(fd, offset, whence, pos)   — 파일 위치 변경
  fd_close(fd)                       — 파일 디스크립터 닫기
  path_open(dirfd, path, flags, fd)  — 파일 열기
  clock_time_get(id, precision, t)   — 현재 시각 조회
  proc_exit(code)                    — 프로세스 종료
  environ_get(environ, buf)          — 환경 변수 조회
  args_get(argv, buf)                — 커맨드라인 인수 조회
```

```python
# Python으로 구현한 최소 WASI 런타임 (wasmtime 라이브러리 사용)
from wasmtime import Store, Module, Linker, WasiConfig

def run_wasm_with_wasi(wasm_path: str, args: list[str]) -> int:
    """WASI 활성화된 Wasm 모듈 실행"""
    store = Store()
    
    # WASI 설정: stdin/stdout/stderr와 환경 변수
    wasi_config = WasiConfig()
    wasi_config.inherit_stdin()
    wasi_config.inherit_stdout()
    wasi_config.inherit_stderr()
    wasi_config.argv = args
    wasi_config.env = [("HOME", "/home/wasm")]
    # capability: 특정 디렉토리만 접근 허용
    wasi_config.preopen_dir("./data", "/data")
    store.set_wasi(wasi_config)
    
    # 모듈 컴파일 및 링킹
    module = Module.from_file(store.engine, wasm_path)
    linker = Linker(store.engine)
    linker.define_wasi()
    
    instance = linker.instantiate(store, module)
    
    # _start 함수 호출 (main 진입점)
    start = instance.exports(store)["_start"]
    try:
        start(store)
        return 0
    except SystemExit as e:
        return e.code

exit_code = run_wasm_with_wasi("./hello.wasm", ["hello.wasm", "world"])
print(f"종료 코드: {exit_code}")
```

## JIT 컴파일 최적화

인터프리터 방식은 간단하지만 성능이 부족합니다. V8(TurboFan), SpiderMonkey(Warp), Wasmtime(Cranelift)은 Wasm 바이트코드를 네이티브 기계어로 JIT 컴파일합니다.

### 컴파일 파이프라인

```
Wasm 바이트코드
      ↓
  [디코딩 & 검증]  — O(n) 선형 패스
      ↓
  [IR 변환]        — SSA(Static Single Assignment) 형식
      ↓
  [최적화 패스]    — 인라이닝, 상수 폴딩, 죽은 코드 제거, 루프 최적화
      ↓
  [레지스터 할당]  — 그래프 채색 또는 선형 스캔
      ↓
  [코드 생성]      — x86-64 / ARM64 / RISC-V 네이티브 코드
      ↓
  [가드 페이지]    — 메모리 접근 범위 검사 (하드웨어 지원)
```

### 메모리 접근 최적화

Wasm의 모든 메모리 접근은 범위 검사가 필요하지만, 가상 메모리 기법으로 오버헤드를 0에 가깝게 줄일 수 있습니다.

```
전통적 범위 검사:
  if (addr + offset >= memory_size) trap();
  native_ptr = linear_memory_base + addr + offset;

가상 메모리 최적화 (4GB 가드 페이지):
  native_ptr = linear_memory_base + addr + offset;
  // 범위 초과 시 SIGSEGV → 자동으로 Wasm 트랩으로 변환
  // 조건 분기 명령어 제거 → 분기 예측 미스 없음
```

## 주의사항과 실전 팁

### 1. 메모리 관리는 개발자 몫
Wasm에는 GC가 없습니다(GC 제안은 별도). Rust는 소유권 시스템으로 자동 관리되지만, C/C++는 직접 관리해야 합니다. JS 측에서 Wasm 메모리를 해제하지 않으면 누수가 발생합니다.

### 2. JS ↔ Wasm 경계 비용
함수 호출 경계를 가로지를 때마다 타입 변환 비용이 발생합니다. 작은 함수를 자주 호출하기보다 배치 처리 방식으로 경계 통과 횟수를 최소화하세요.

### 3. SIMD 지원 확인
Wasm SIMD(128-bit)는 대부분의 모던 브라우저에서 지원되지만, 레거시 환경에서는 폴백 경로가 필요합니다. `wasm-feature-detect` 라이브러리로 런타임 감지를 권장합니다.

### 4. 디버깅: Source Map
Wasm 바이너리는 난독화된 것처럼 보이지만, DWARF 디버그 정보와 Source Map을 함께 배포하면 브라우저 DevTools에서 원본 C/C++/Rust 소스로 디버깅할 수 있습니다.

### 5. 스레딩: SharedArrayBuffer
Wasm 스레드는 SharedArrayBuffer 위에서 동작합니다. 이를 위해서는 응답 헤더에 `Cross-Origin-Opener-Policy: same-origin`과 `Cross-Origin-Embedder-Policy: require-corp`가 필요합니다.

## 정리

WebAssembly는 단순한 "느린 JS 대안"이 아니라, 언어·플랫폼 독립적인 안전한 범용 실행 환경입니다. 바이너리 포맷의 컴팩트함, 스택 머신의 단순성, WASI의 capability 보안 모델이 결합되어 브라우저, 서버, 엣지, IoT 어디서든 동일한 코드를 안전하게 실행할 수 있게 합니다. Wasm 런타임 내부를 이해하면 성능 병목을 진단하고, 메모리를 올바르게 관리하며, 최적의 컴파일 전략을 선택할 수 있습니다.

## 참고 자료
- [Research on WebAssembly Runtimes: A Survey (arXiv 2024)](https://arxiv.org/html/2404.12621v1)
- [WebAssembly Specification — W3C](https://webassembly.github.io/spec/core/)
- [Bytecode Alliance — WASI 공식 문서](https://wasi.dev/)
- [Wasmtime 런타임 문서](https://docs.wasmtime.dev/)
