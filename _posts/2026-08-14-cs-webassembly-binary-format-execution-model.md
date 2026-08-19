---
layout: post
title: "WebAssembly 바이너리 포맷과 실행 모델 완전 정복: 브라우저에서 네이티브 속도를 달성하는 기술"
date: 2026-08-14
categories: [cs, computer-science]
tags: [webassembly, wasm, binary-format, execution-model, browser, virtual-machine, compilation, rust]
---

## WebAssembly란 무엇인가

WebAssembly(Wasm)는 W3C가 표준화한 이진 명령어 형식(binary instruction format)으로, 안전하고 이식성 높은 저수준 가상 머신 위에서 동작합니다. 2017년 모든 주요 브라우저에서 기본 지원되기 시작했으며, 현재는 브라우저뿐만 아니라 서버, 엣지 컴퓨팅, 임베디드 시스템에서도 광범위하게 활용됩니다.

Wasm의 핵심 목표는 네 가지입니다:

- **빠른 실행 속도**: 네이티브 코드에 근접한 성능
- **안전성**: 메모리 안전성을 보장하는 샌드박스 실행 환경
- **이식성**: 다양한 CPU 아키텍처에서 동일하게 동작
- **개방성**: C, C++, Rust, Go, AssemblyScript 등 다양한 언어에서 컴파일 가능

## 왜 WebAssembly가 필요한가

JavaScript는 인터프리터 방식으로 시작해 JIT(Just-In-Time) 컴파일러를 거쳐 상당히 빨라졌지만, 본질적으로 동적 타입 언어라는 한계가 있습니다. 타입을 런타임에 추론하고, 가비지 컬렉션이 비결정적으로 실행되며, 최악의 경우 JIT의 최적화 가정이 어긋나 디옵티마이제이션(deoptimization)이 발생합니다.

이에 비해 WebAssembly는 다음과 같은 특성을 가집니다:

- **정적 타입 시스템**: `i32`, `i64`, `f32`, `f64`, `v128` 등의 기본 타입
- **선형 메모리(linear memory)**: 가비지 컬렉션 없이 직접 메모리 관리
- **검증 가능한 구조적 제어 흐름**: `block`, `loop`, `if` 등의 명시적 구조
- **사전 타입 검사 가능한 명령어 집합**: 로드 즉시 검증 후 컴파일

2023년 기준 벤치마크에서 Wasm은 네이티브 코드 대비 평균 약 1.45배의 실행 시간 오버헤드를 가지며, 이는 순수 JavaScript 대비 수 배에서 수십 배의 성능 차이입니다. 게임 엔진(Unity, Unreal), 이미지/영상 처리, 암호화 연산, 과학 시뮬레이션 등 CPU 집약적인 작업에서 Wasm이 특히 빛을 발합니다.

## WebAssembly 바이너리 포맷 구조

### 매직 넘버와 버전 헤더

Wasm 파일은 항상 8바이트 헤더로 시작합니다. 처음 4바이트는 매직 넘버이고 다음 4바이트는 버전 번호입니다:

```
00 61 73 6D  ; magic: "\0asm" (ASCII)
01 00 00 00  ; version: 1 (리틀 엔디안)
```

이 헤더는 파일이 유효한 Wasm 모듈임을 나타냅니다.

### 섹션 구조

헤더 이후에는 여러 섹션(Section)이 순서대로 나열됩니다. 각 섹션은 `[섹션 ID 1바이트][바이트 길이 LEB128][내용]` 구조를 따릅니다:

| Section ID | 이름 | 역할 |
|---|---|---|
| 0 | Custom | 디버그 정보, 이름, DWARF 심볼 등 |
| 1 | Type | 함수 타입 시그니처 정의 |
| 2 | Import | 외부에서 임포트된 함수/메모리/테이블/전역변수 |
| 3 | Function | 각 함수가 사용하는 타입 섹션 인덱스 참조 |
| 4 | Table | 함수 포인터 테이블 (간접 호출용) |
| 5 | Memory | 선형 메모리 최솟값/최댓값 |
| 6 | Global | 전역 변수 (타입, 가변성, 초기값) |
| 7 | Export | 외부(호스트)에 노출할 이름과 항목 |
| 8 | Start | 인스턴스화 시 자동 실행 함수 인덱스 |
| 9 | Element | 테이블 초기화 데이터 |
| 10 | Code | 함수 본문 (로컬 변수 + 표현식) |
| 11 | Data | 메모리 초기화 데이터 |

### LEB128 인코딩

Wasm은 정수 인코딩에 LEB128(Little Endian Base 128)을 사용합니다. 가변 길이 인코딩으로 작은 수는 1바이트, 큰 수는 여러 바이트로 표현합니다. 각 바이트의 최상위 비트(MSB)가 1이면 다음 바이트가 더 있다는 의미입니다:

```python
def encode_uleb128(value: int) -> bytes:
    """부호 없는 정수를 LEB128으로 인코딩"""
    result = []
    while True:
        byte = value & 0x7F       # 하위 7비트 추출
        value >>= 7
        if value != 0:
            byte |= 0x80          # 더 많은 바이트가 남아있음을 표시
        result.append(byte)
        if value == 0:
            break
    return bytes(result)

def decode_uleb128(data: bytes, offset: int = 0):
    """LEB128으로 인코딩된 부호 없는 정수 디코딩"""
    result = 0
    shift = 0
    while True:
        byte = data[offset]
        offset += 1
        result |= (byte & 0x7F) << shift
        if (byte & 0x80) == 0:   # MSB가 0이면 마지막 바이트
            break
        shift += 7
    return result, offset

# 예시: 300 = 0b100101100
print(encode_uleb128(300).hex())       # ac02
print(decode_uleb128(bytes([0xAC, 0x02])))  # (300, 2)
print(encode_uleb128(127).hex())       # 7f
print(encode_uleb128(128).hex())       # 8001
```

## WebAssembly 실행 모델

### 스택 기반 가상 머신

Wasm은 스택 기반(stack-based) 가상 머신입니다. 모든 명령어는 값 스택(value stack)에서 피연산자를 꺼내 연산 후 결과를 다시 스택에 넣습니다. WAT(WebAssembly Text Format)으로 표현한 예시입니다:

```wat
;; WAT으로 작성한 피보나치 함수
(module
  (func $fib (param $n i32) (result i32)
    ;; n < 2이면 n 반환 (기저 조건)
    (if (i32.lt_s (local.get $n) (i32.const 2))
      (then (return (local.get $n)))
    )
    ;; fib(n-1) + fib(n-2)
    (i32.add
      (call $fib
        (i32.sub (local.get $n) (i32.const 1))
      )
      (call $fib
        (i32.sub (local.get $n) (i32.const 2))
      )
    )
  )
  ;; JavaScript에서 호출할 수 있도록 내보내기
  (export "fib" (func $fib))
)
```

`i32.add`는 스택에서 두 i32 값을 꺼내 더한 결과를 스택에 넣습니다. 타입 검증은 로드 시 정적으로 완료되므로 런타임 타입 체크가 불필요합니다.

### 선형 메모리 모델

Wasm 모듈은 하나 이상의 선형 메모리(Linear Memory)를 가질 수 있습니다. 이 메모리는 연속된 바이트 배열이며, 64KB 단위(페이지)로 확장 가능합니다. 모든 메모리 접근에는 인덱스 경계 검사가 수행됩니다:

```javascript
// JavaScript에서 WebAssembly 메모리 접근 예시
const memory = new WebAssembly.Memory({
  initial: 1,    // 초기 크기: 1페이지 = 65,536바이트
  maximum: 100   // 최대 크기: 100페이지 = 6,553,600바이트
});

const importObject = { env: { memory } };

WebAssembly.instantiateStreaming(fetch('module.wasm'), importObject)
  .then(({ instance }) => {
    const result = instance.exports.compute(42);
    console.log('결과:', result);

    const i32View = new Int32Array(memory.buffer);
    i32View[0] = 12345;              // JavaScript → Wasm
    instance.exports.process();      // Wasm이 i32View[0] 처리
    console.log(i32View[1]);         // Wasm → JavaScript 결과

    const prevPages = memory.grow(10);
    console.log('이전 크기:', prevPages, '페이지');
  });
```

### 검증(Validation) 단계

Wasm 모듈을 실행하기 전에 런타임은 단일 패스(single-pass)로 검증을 수행합니다:

1. **타입 스택 검사**: 각 명령어 실행 전후의 스택 타입이 예상과 일치하는지 확인
2. **구조적 제어 흐름 검사**: `block`/`loop`/`if`가 올바르게 중첩되고 `end`로 닫히는지 확인
3. **함수 시그니처 검사**: `call` 명령어가 올바른 인자 타입과 개수로 함수를 호출하는지 확인
4. **메모리/테이블 인덱스 검사**: 존재하는 메모리/테이블 인덱스만 참조하는지 확인

이 검증은 O(n) 시간에 완료되며, 여기서 n은 바이트코드의 크기입니다.

### 전체 실행 파이프라인

```
소스 코드 (C/C++/Rust/Go/AssemblyScript)
        ↓  컴파일 (LLVM, wasm-pack, TinyGo 등)
WebAssembly 바이너리 (.wasm)
        ↓  파싱: 섹션별 추상 구문 생성
추상 구문 표현 (Abstract Syntax)
        ↓  검증: 타입 체크, 제어 흐름 검사
검증된 모듈 (Validated Module)
        ↓  컴파일 (JIT 또는 AOT)
네이티브 기계어 코드
        ↓  인스턴스화: 메모리/테이블 초기화, import 연결
실행 가능한 인스턴스 (Module Instance)
        ↓  실행 (Execution)
결과값
```

## 코드 예제: Rust에서 Wasm 모듈 작성

```rust
// src/lib.rs
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[wasm_bindgen]
pub fn matrix_multiply(
    a: &[f64], b: &[f64], n: usize
) -> Vec<f64> {
    let mut result = vec![0.0f64; n * n];
    for i in 0..n {
        for k in 0..n {
            let a_ik = a[i * n + k];
            for j in 0..n {
                result[i * n + j] += a_ik * b[k * n + j];
            }
        }
    }
    result
}

#[wasm_bindgen]
pub fn alloc(size: usize) -> *mut u8 {
    let mut buf = Vec::with_capacity(size);
    let ptr = buf.as_mut_ptr();
    std::mem::forget(buf);
    ptr
}

#[wasm_bindgen]
pub fn dealloc(ptr: *mut u8, size: usize) {
    unsafe { Vec::from_raw_parts(ptr, 0, size) };
}
```

```bash
# wasm-pack으로 빌드
wasm-pack build --target web --release

# wasm 바이너리 최적화
wasm-opt -O3 -o optimized.wasm pkg/my_module_bg.wasm
wasm-objdump -h optimized.wasm
```

```html
<!DOCTYPE html>
<html>
<head><title>Wasm Demo</title></head>
<body>
<script type="module">
  import init, { add, matrix_multiply } from './pkg/my_module.js';

  async function run() {
    await init();
    console.log('1 + 2 =', add(1, 2));

    const n = 512;
    const a = new Float64Array(n * n).fill(1.0);
    const b = new Float64Array(n * n).fill(2.0);
    const start = performance.now();
    const result = matrix_multiply(a, b, n);
    console.log(`${n}x${n} 행렬 곱셈: ${(performance.now()-start).toFixed(1)}ms`);
  }

  run().catch(console.error);
</script>
</body>
</html>
```

## WASI: 브라우저 밖에서 Wasm 실행

WebAssembly System Interface(WASI)는 Wasm이 파일 시스템, 네트워크, 시스템 콜 등에 접근할 수 있는 표준 인터페이스입니다. Wasmtime, Wasmer 같은 런타임으로 서버에서 Wasm을 실행할 수 있으며, 도커 컨테이너와 유사한 격리 수준을 훨씬 가벼운 비용으로 제공합니다:

```bash
# Rust로 WASI 바이너리 컴파일
rustup target add wasm32-wasip1
cargo build --target wasm32-wasip1 --release

# Wasmtime으로 실행
wasmtime target/wasm32-wasip1/release/hello.wasm

# 파일시스템 접근 권한 명시적으로 부여
wasmtime --dir=/tmp target/wasm32-wasip1/release/hello.wasm
```

## 주의사항과 팁

### 1. JS ↔ Wasm 경계 비용

Wasm과 JavaScript 사이의 함수 호출(FFI call)에는 일정한 오버헤드가 있습니다. 해결책:
- **메모리 공유**: 공유 `ArrayBuffer`를 통해 복사 없이 데이터 교환
- **배치 처리**: 작은 연산 여러 번보다 큰 배치 한 번을 Wasm에 넘기기
- **인터페이스 최소화**: Wasm 내부에서 루프를 처리하고 JS에는 결과만 반환

### 2. SIMD 명령어 활용

Wasm은 `v128` 타입을 통한 128비트 SIMD 연산을 지원합니다. Rust에서는 `std::arch::wasm32` 또는 `packed_simd` 크레이트로 접근 가능하며, ML 추론이나 이미지 처리에서 성능이 크게 향상됩니다.

### 3. 보안 고려사항

Wasm은 선형 메모리 내에서만 동작하며 호스트 시스템에 직접 접근할 수 없습니다. 그러나:
- **Spectre/Meltdown**: 타이밍 공격에 취약할 수 있어 브라우저는 `performance.now()` 정밀도를 낮추는 등의 완화책을 적용합니다.
- **메모리 안전성 경계**: C/C++에서 컴파일된 코드는 선형 메모리 내에서 버퍼 오버플로 등의 취약점을 가질 수 있습니다.

### 4. Component Model (Wasm 2.0의 핵심)

Wasm의 Component Model은 여러 Wasm 모듈이 인터페이스 타입(WIT: Wasm Interface Type)을 통해 서로 통신할 수 있게 합니다. 언어에 독립적인 플러그인 시스템 구축이 가능하며, Wasmtime 1.0+에서 안정화되었습니다.

## 참고 자료
- [WebAssembly Specification Overview](https://webassembly.github.io/spec/core/intro/overview.html)
- [WebAssembly Binary Format Reference](https://webassembly.github.io/spec/core/binary/index.html)
- [MDN WebAssembly](https://developer.mozilla.org/en-US/docs/WebAssembly)
- [wasm-bindgen 공식 문서](https://rustwasm.github.io/docs/wasm-bindgen/)
