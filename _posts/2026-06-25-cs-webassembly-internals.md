---
layout: post
title: "WebAssembly 내부 구조 완전 정복: 컴파일 파이프라인·샌드박스·메모리 모델"
date: 2026-06-25
categories: [cs, computer-science]
tags: [webassembly, wasm, wasi, compilation, sandbox, memory-model, performance, 컴파일러]
---

2015년 등장한 WebAssembly(Wasm)는 브라우저에서 C/C++/Rust로 작성된 코드를 네이티브에 가까운 속도로 실행하는 것을 목표로 설계되었습니다. 2026년 현재 WebAssembly 3.0이 W3C 표준이 되고, WASI 0.3.0이 서버사이드 런타임을 지원하면서 Wasm은 브라우저를 넘어 **범용 샌드박스 런타임**으로 진화했습니다. 이 글에서는 Wasm이 어떻게 동작하는지 내부 구조를 깊이 파헤칩니다.

## WebAssembly란 무엇인가

WebAssembly는 **스택 기반 가상 머신을 위한 이진 명령어 형식(Binary Instruction Format)**입니다. 특정 CPU 아키텍처에 종속되지 않으며, 모든 플랫폼에서 동일하게 실행됩니다. 핵심 설계 목표는 다음 네 가지입니다.

- **빠름(Fast)**: JIT/AOT 컴파일로 네이티브 속도에 근접
- **안전(Safe)**: 메모리 안전, 타입 안전, 샌드박스 격리
- **이식성(Portable)**: CPU, OS 무관하게 동일 바이너리 실행
- **컴팩트(Compact)**: 바이너리 형식으로 JSON/JavaScript 대비 작은 크기

JavaScript는 동적 타입 언어로 JIT 최적화에 한계가 있지만, Wasm은 정적 타입의 낮은 수준 IR(Intermediate Representation)이라 훨씬 예측 가능한 최적화가 가능합니다.

## 컴파일 파이프라인: 소스 코드가 실행되기까지

### 1단계: 소스 → LLVM IR → Wasm 바이너리

```
C/C++ 소스
    ↓  (clang -emit-llvm)
LLVM IR (.ll)
    ↓  (wasm-ld / lld)
Wasm 바이너리 (.wasm)
    ↓  (브라우저 / wasmtime)
네이티브 머신 코드
```

Rust를 예로 들어 실제 Wasm 바이너리를 만들어 봅니다.

```rust
// src/lib.rs
#[no_mangle]
pub extern "C" fn fibonacci(n: u32) -> u64 {
    if n <= 1 {
        return n as u64;
    }
    let mut a: u64 = 0;
    let mut b: u64 = 1;
    for _ in 2..=n {
        let temp = a + b;
        a = b;
        b = temp;
    }
    b
}
```

```bash
# wasm32-unknown-unknown 타겟으로 컴파일
rustup target add wasm32-unknown-unknown
cargo build --target wasm32-unknown-unknown --release

# 생성된 .wasm 파일을 WAT(WebAssembly Text Format)으로 변환
wasm2wat target/wasm32-unknown-unknown/release/fib.wasm
```

WAT(텍스트 형식)으로 변환하면 Wasm의 스택 머신 구조를 확인할 수 있습니다.

```wat
;; WAT 형식: 피보나치 함수 핵심 부분
(module
  (func $fibonacci (export "fibonacci") (param i32) (result i64)
    (local i64)  ;; temp 변수
    (local i64)  ;; a
    (local i64)  ;; b
    ;; ... 스택 기반 명령어들
    local.get 0  ;; n을 스택에 푸시
    i32.const 1
    i32.le_u      ;; n <= 1 비교
    if (result i64)
      local.get 0
      i64.extend_i32_u  ;; i32 → i64 확장
    else
      ;; 루프 본체...
    end
  )
)
```

### 2단계: 브라우저/런타임 내부의 컴파일

Wasm 바이너리를 받은 런타임(V8, SpiderMonkey, wasmtime)은 두 가지 방식으로 실행합니다.

**Baseline 컴파일 (빠른 시작)**: 바이트코드를 선형으로 스캔하며 빠르게 머신 코드를 생성합니다. V8의 Liftoff 컴파일러가 이에 해당합니다. 최적화는 적지만 즉시 실행 가능합니다.

**Optimizing 컴파일 (최고 성능)**: Baseline 실행 중에 백그라운드에서 더 정교한 최적화를 적용한 코드를 생성합니다. V8의 Turbofan, SpiderMonkey의 IonMonkey가 담당합니다. 완료되면 최적화 코드로 교체합니다(Tier-Up).

```
Wasm 바이너리 다운로드
      ↓
Baseline 컴파일 (Liftoff)  →  즉시 실행 시작
      +
Optimizing 컴파일 (Turbofan)  →  백그라운드 실행
      ↓
최적화 완료 → Tier-Up (hot 함수부터 교체)
```

## 선형 메모리 모델 (Linear Memory)

Wasm의 메모리 모델은 의도적으로 단순합니다. 각 모듈은 **단일 연속 바이트 배열(Linear Memory)**을 가집니다.

```
Wasm Linear Memory (최대 4GB, 64KB 페이지 단위)
┌─────────────────────────────────────────────────┐
│  Stack (함수 호출 스택, 낮은 주소에서 성장)         │
├─────────────────────────────────────────────────┤
│  Heap (malloc/new 할당 공간)                      │
├─────────────────────────────────────────────────┤
│  Global Data (전역 변수, 문자열 리터럴)             │
└─────────────────────────────────────────────────┘
    ↑ 주소 0                           ↑ memory.size
```

```javascript
// JavaScript에서 Wasm 메모리 접근
const memory = new WebAssembly.Memory({ 
    initial: 1,    // 1페이지 = 64KB
    maximum: 16,   // 최대 16페이지 = 1MB
    shared: true   // SharedArrayBuffer 기반 (멀티스레딩용)
});

// Wasm 모듈 로딩
const { instance } = await WebAssembly.instantiateStreaming(
    fetch('/fibonacci.wasm'),
    { env: { memory } }
);

// Wasm 함수 호출: JavaScript ↔ Wasm 경계를 넘나듦
const result = instance.exports.fibonacci(40);
console.log(`fib(40) = ${result}`);

// 직접 메모리 읽기/쓰기
const view = new Uint8Array(memory.buffer);
view[0] = 42;  // Wasm 모듈의 메모리 주소 0에 값 쓰기
```

**메모리 안전성의 핵심**: Wasm 모듈은 자신의 Linear Memory 범위 밖에 절대 접근할 수 없습니다. 범위를 벗어난 메모리 접근은 트랩(Trap)을 발생시키고 모듈이 종료됩니다. 호스트 메모리(JavaScript 힙, OS 메모리)는 완전히 분리되어 있습니다.

## 샌드박스 보안 모델

Wasm의 보안은 **능력 기반(Capability-Based) 보안** 모델을 따릅니다. 기본적으로 모듈은 아무것도 할 수 없습니다.

```
Wasm 모듈 (기본 상태)
├── ✗ 파일시스템 접근 불가
├── ✗ 네트워크 소켓 불가
├── ✗ 시스템 콜 불가
├── ✗ 다른 모듈 메모리 접근 불가
└── ✗ 환경 변수 접근 불가

명시적으로 허용한 기능만 사용 가능 (Import를 통해)
```

```javascript
// 허용된 기능만 Import로 전달
const importObject = {
    env: {
        // 허용: 콘솔 출력 함수
        print: (ptr, len) => {
            const bytes = new Uint8Array(memory.buffer, ptr, len);
            console.log(new TextDecoder().decode(bytes));
        },
        // 허용: 현재 시각 조회
        now: () => Date.now(),
        // 허용: 메모리 확장
        memory: memory
    }
    // 그 외 모든 기능은 전달하지 않음
};
```

### WASI: 서버사이드 표준 시스템 인터페이스

WASI(WebAssembly System Interface)는 브라우저 밖에서 Wasm을 실행할 때 파일, 네트워크, 환경 변수에 접근하는 표준 API를 정의합니다.

```rust
// WASI 환경에서 파일 읽기 (Rust)
use std::fs;
use std::io;

fn main() -> io::Result<()> {
    // WASI가 허용한 디렉토리만 접근 가능
    let content = fs::read_to_string("data.txt")?;
    println!("파일 내용: {}", content);
    Ok(())
}
```

```bash
# WASI 타겟으로 컴파일
rustup target add wasm32-wasip1
cargo build --target wasm32-wasip1 --release

# wasmtime으로 실행 (파일 접근 명시적 허용)
wasmtime run \
    --dir=/allowed/path \   # 이 경로만 접근 허용
    target/wasm32-wasip1/release/myapp.wasm
```

WASI 0.3.0(2026년 2월)에서는 비동기 I/O가 네이티브로 지원되어, 동시 연결을 처리하는 서버 애플리케이션 구현이 가능해졌습니다.

## 성능 특성과 최적화

Wasm의 실제 성능은 작업 유형에 따라 크게 다릅니다.

**CPU 집약적 연산**: 네이티브 바이너리 대비 약 1.1~1.3배 느린 수준으로, JavaScript 대비 압도적으로 빠릅니다.

**I/O 집약적 작업**: 현재 2~3배 느린 수준이나 WASI 0.3.0의 async I/O로 개선되고 있습니다.

**냉시작(Cold Start)**: AOT 컴파일 환경에서 5ms 미만으로, 컨테이너(~100ms)보다 훨씬 빠릅니다.

```c
// C에서 SIMD를 활용한 벡터 연산 최적화 예시
#include <wasm_simd128.h>

void vector_add(float* a, float* b, float* result, int n) {
    int i = 0;
    // SIMD: 한 번에 4개의 float 처리
    for (; i <= n - 4; i += 4) {
        v128_t va = wasm_v128_load(&a[i]);
        v128_t vb = wasm_v128_load(&b[i]);
        v128_t vc = wasm_f32x4_add(va, vb);
        wasm_v128_store(&result[i], vc);
    }
    // 나머지 처리
    for (; i < n; i++) {
        result[i] = a[i] + b[i];
    }
}
```

WebAssembly 3.0(2025년 9월 표준화)에서는 다음 기능이 추가되었습니다.

- **WasmGC**: 가비지 컬렉션 지원 → JVM 언어(Kotlin, Java), Dart 등이 이제 Wasm을 타겟으로 할 수 있음
- **128-bit SIMD**: 더 넓은 벡터 연산
- **64-bit 메모리**: 4GB 이상 메모리 지원
- **Exception Handling**: try/catch 구문 지원

## 주의사항 및 실전 팁

**1. DOM 접근 비용**: JavaScript와 Wasm 사이의 호출 경계(JS-Wasm boundary crossing)에는 비용이 있습니다. Wasm 내부에서 수백만 번 반복하는 계산은 빠르지만, 매 프레임 DOM을 조작하는 작업은 여전히 JavaScript가 적합합니다.

**2. 파일 크기 최적화**:
```bash
# wasm-opt으로 최적화 (Binaryen)
wasm-opt -O3 -o optimized.wasm input.wasm

# wasm-strip으로 디버그 심볼 제거
wasm-strip output.wasm
```

**3. 메모리 관리**: GC가 없는 언어(C, C++)로 작성된 Wasm 모듈은 메모리를 직접 관리해야 합니다. 메모리 누수가 Linear Memory를 채우면 `memory.grow` 요청이 실패하며 OOM이 발생합니다.

**4. 스트리밍 컴파일**: `WebAssembly.instantiateStreaming()`을 사용하면 다운로드와 컴파일을 동시에 진행해 초기화 시간을 단축합니다.

**5. 모듈 캐싱**: `WebAssembly.Module`은 직렬화 가능합니다. IndexedDB에 저장해두면 다음 방문 시 재컴파일 없이 즉시 인스턴스화할 수 있습니다.

WebAssembly는 "한 번 컴파일하면 어디서든 실행"이라는 Java의 꿈을 훨씬 강력한 형태로 실현했습니다. 브라우저의 컴퓨팅 엔진을 넘어, 이제 서버리스 함수, 플러그인 시스템, 엣지 컴퓨팅의 핵심 기반 기술로 자리잡고 있습니다.

## 참고 자료
- [WebAssembly Explained: Complete Wasm & WASI Guide for Developers 2026 - DevToolLab](https://www.devtoollab.com/blog/webassembly-guide)
- [WebAssembly (WASM) Security: The Sandbox Model, Linear Memory, and Real Risks - AquilaX](https://aquilax.ai/blog/webassembly-wasm-security-risks)
- [WebAssembly in 2026: A Practical Guide to Wasm and WASI for Modern Developers - WebNuz](https://www.webnuz.com/article/2026-06-14/WebAssembly%20in%202026:%20A%20Practical%20Guide%20to%20Wasm%20and%20WASI%20for%20Modern%20Developers)
- [WebAssembly in 2026: Where It Has Landed, What WASI 0.2 Changes - Java Code Geeks](https://www.javacodegeeks.com/2026/04/webassembly-in-2026-where-it-has-landed-what-wasi-0-2-changes-and-why-java-and-kotlin-developers-should-pay-attention-now.html)
