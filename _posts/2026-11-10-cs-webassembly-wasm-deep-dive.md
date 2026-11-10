---
layout: post
title: "WebAssembly(WASM) 완전 정복: 바이너리 포맷·스택 머신·브라우저 외 실행 환경까지"
date: 2026-11-10
categories: [cs, computer-science]
tags: [webassembly, wasm, wasi, compiler, virtual-machine, stack-machine, binary-format]
---

## WebAssembly란 무엇인가?

WebAssembly(이하 WASM)는 2019년 W3C 공식 권고안으로 채택된 이진 명령어 형식(binary instruction format)이자 가상 머신 사양(virtual machine specification)입니다. 처음에는 "웹 브라우저에서 C/C++를 실행한다"는 목적으로 설계되었지만, 오늘날에는 서버사이드, 엣지 컴퓨팅, 플러그인 샌드박스 등 훨씬 광범위한 맥락에서 활용되고 있습니다.

WASM의 핵심 설계 목표는 세 가지입니다.

- **이식성(Portability)**: x86, ARM, RISC-V 등 아키텍처에 무관하게 동일한 바이너리가 실행됩니다.
- **안전성(Safety)**: 선형 메모리 모델과 타입 검사를 통해 메모리 안전성을 보장합니다.
- **성능(Performance)**: 네이티브 코드에 근접한 실행 속도를 추구합니다. 대부분의 벤치마크에서 네이티브 대비 1.0~1.5배 오버헤드 수준에 머뭅니다.

WASM은 소스 언어가 아닙니다. C/C++, Rust, Go, Python, Java 등 다양한 언어를 컴파일 타겟으로 사용하며, 개발자는 선호하는 언어로 코드를 작성하고 WASM 바이너리로 컴파일합니다.

## 왜 WebAssembly가 필요한가?

JavaScript는 브라우저의 유일한 언어였지만, 여러 한계를 갖고 있습니다.

**JavaScript의 한계:**
- 동적 타입: 타입 추론 실패 시 JIT 최적화가 무너집니다.
- 가비지 컬렉션 일시 정지: 실시간 그래픽·오디오 처리에서 GC 중단이 문제가 됩니다.
- 대형 코드베이스 이식의 어려움: OpenCV, FFmpeg 같은 거대한 C 라이브러리를 JS로 포팅하는 것은 비현실적입니다.

asm.js는 이 문제를 완화하려 했지만, 텍스트 형식의 한계와 파싱 오버헤드가 있었습니다. WASM은 이를 근본적으로 해결합니다.

**WASM의 장점:**
- 사전 타입 검사로 JIT의 불필요한 추측 제거
- 스트리밍 컴파일: 다운로드와 컴파일을 병렬 수행
- 선형 메모리: 경계 검사는 하되, 직접 주소 지정 가능
- 샌드박스: 호스트 환경으로부터 격리된 실행

## 스택 머신과 바이너리 포맷 내부 구조

### 스택 기반 가상 머신

WASM은 레지스터 기반이 아닌 **스택 기반(stack-based)** 가상 머신입니다. 모든 연산은 묵시적인 평가 스택(evaluation stack)을 통해 수행됩니다.

```
;; WAT(WebAssembly Text Format) 예시: 두 수의 합
(module
  (func $add (param $a i32) (param $b i32) (result i32)
    local.get $a   ;; 스택: [a]
    local.get $b   ;; 스택: [a, b]
    i32.add        ;; 스택: [a+b]  — 두 값을 팝하고 합을 푸시
  )
  (export "add" (func $add))
)
```

이 명령어들은 바이너리로 인코딩됩니다. 각 opcode는 1바이트이며, 정수는 LEB128(Little Endian Base 128) 가변 길이 인코딩을 사용합니다.

### 모듈 바이너리 구조

WASM 모듈은 매직 넘버 `\0asm`(0x00 0x61 0x73 0x6D)으로 시작하고, 버전 번호(현재 0x01 0x00 0x00 0x00)가 뒤따릅니다. 이후 여러 섹션(section)이 연속됩니다.

| 섹션 ID | 이름 | 역할 |
|---------|------|------|
| 1 | Type | 함수 시그니처 목록 |
| 2 | Import | 외부 함수/메모리/테이블 임포트 |
| 3 | Function | 함수가 사용할 타입 인덱스 |
| 4 | Table | 함수 포인터 테이블 |
| 5 | Memory | 선형 메모리 크기(단위: 64KB 페이지) |
| 7 | Export | 외부로 노출할 이름-인덱스 쌍 |
| 10 | Code | 함수 바이트코드 본문 |

### 선형 메모리 모델

WASM 모듈은 단일 선형 메모리(linear memory) 공간을 갖습니다. 이 메모리는 64KB 단위 페이지로 관리되며, 초기 크기와 최대 크기를 선언할 수 있습니다. 호스트(브라우저 또는 런타임)는 이 메모리를 `ArrayBuffer`나 직접 메모리 영역으로 노출합니다.

```c
// C 코드 예시: WASM으로 컴파일되는 선형 메모리 활용
#include <string.h>
#include <stdlib.h>

// WASM 선형 메모리 위에서 동작하는 간단한 문자열 처리
__attribute__((export_name("process_string")))
int process_string(char* input, int len) {
    int count = 0;
    for (int i = 0; i < len; i++) {
        if (input[i] == 'a') count++;
    }
    return count;
}
```

이 코드를 Emscripten이나 wasm-pack으로 컴파일하면 `.wasm` 파일과 JS 글루 코드가 생성됩니다.

## 실제 구현 예제

### 예제 1: Rust로 작성한 WASM 모듈

Rust는 WASM의 1급 타겟 언어입니다. `wasm-bindgen`을 사용하면 JavaScript와의 경계에서 타입 변환을 자동화할 수 있습니다.

```rust
// src/lib.rs
use wasm_bindgen::prelude::*;

// 피보나치 수열 계산 (반복 방식)
#[wasm_bindgen]
pub fn fibonacci(n: u32) -> u64 {
    if n <= 1 {
        return n as u64;
    }
    let (mut a, mut b) = (0u64, 1u64);
    for _ in 2..=n {
        let next = a.checked_add(b).unwrap_or(u64::MAX);
        a = b;
        b = next;
    }
    b
}

// 소수 판별 (에라토스테네스의 체)
#[wasm_bindgen]
pub fn sieve_of_eratosthenes(limit: usize) -> Vec<u32> {
    let mut is_prime = vec![true; limit + 1];
    is_prime[0] = false;
    if limit > 0 { is_prime[1] = false; }

    let mut i = 2;
    while i * i <= limit {
        if is_prime[i] {
            let mut j = i * i;
            while j <= limit {
                is_prime[j] = false;
                j += i;
            }
        }
        i += 1;
    }

    is_prime.iter().enumerate()
        .filter(|(_, &p)| p)
        .map(|(i, _)| i as u32)
        .collect()
}
```

빌드 명령:
```bash
# Cargo.toml에 [lib] crate-type = ["cdylib"] 선언 필요
wasm-pack build --target web
```

JavaScript에서 사용:
```javascript
import init, { fibonacci, sieve_of_eratosthenes } from './pkg/my_wasm.js';

async function run() {
    await init(); // WASM 모듈 초기화

    console.log(fibonacci(50));           // 12586269025
    const primes = sieve_of_eratosthenes(100);
    console.log(primes);                  // [2, 3, 5, 7, 11, ...]
}

run();
```

### 예제 2: WASI — 브라우저 밖의 WebAssembly

WASI(WebAssembly System Interface)는 POSIX 유사 시스템 콜을 표준화한 인터페이스입니다. WASM 모듈이 파일 I/O, 네트워크, 환경 변수 등에 접근할 수 있게 해줍니다.

Wasmtime 런타임을 사용한 서버사이드 WASM 실행:

```python
# host.py — Wasmtime Python SDK로 WASM 실행
from wasmtime import Store, Module, Engine, WasiConfig, Linker

def run_wasm_module(wasm_path: str, input_data: str) -> str:
    engine = Engine()
    store = Store(engine)

    # WASI 설정: 표준 스트림 활성화, 특정 디렉터리만 접근 허용
    wasi_config = WasiConfig()
    wasi_config.inherit_stdout()
    wasi_config.inherit_stderr()
    # 현재 디렉터리의 /data 폴더만 게스트에게 노출
    wasi_config.preopen_dir("./data", "/data")
    store.set_wasi(wasi_config)

    linker = Linker(engine)
    linker.define_wasi()

    module = Module.from_file(engine, wasm_path)
    instance = linker.instantiate(store, module)

    # _start = main() 함수 실행
    start = instance.exports(store)["_start"]
    start(store)

    return "완료"

# 사용: Python에서 완전히 격리된 WASM 샌드박스 실행
result = run_wasm_module("./my_app.wasm", "hello")
```

이 패턴은 Cloudflare Workers, Fastly Compute@Edge, Shopify Functions 등 엣지 컴퓨팅 플랫폼에서 널리 사용됩니다. 각 요청을 독립된 WASM 샌드박스에서 실행하여 보안을 강화합니다.

## 주의사항과 현실적인 팁

### WASM이 항상 빠른 것은 아닙니다

WASM과 JavaScript 사이의 경계 호출(boundary crossing)은 비용이 있습니다. 특히 문자열이나 복잡한 객체를 주고받을 때는 메모리 복사가 발생합니다. 따라서:

- **좋은 사용 사례**: CPU 집약적 연산(이미지 처리, 암호화, 압축, 물리 시뮬레이션)
- **나쁜 사용 사례**: DOM 조작, 작은 함수의 잦은 호출

### 디버깅 어려움

WASM 바이너리는 사람이 읽기 어렵습니다. DWARF 디버그 정보를 포함시켜야 브라우저 개발자 도구에서 원본 소스 레벨 디버깅이 가능합니다:

```bash
# Rust의 경우 디버그 심볼 포함 빌드
wasm-pack build --dev
# Emscripten의 경우
emcc -g4 source.c -o output.html
```

### WASM GC 제안

현재 표준화 중인 WASM GC(Garbage Collection) 제안이 완성되면, Java/Kotlin/Python 같은 GC 기반 언어가 훨씬 효율적으로 WASM에서 실행될 수 있습니다. 현재는 각 언어의 GC 런타임 전체를 WASM 바이너리에 번들해야 하므로, 바이너리가 커지는 단점이 있습니다.

### 멀티스레딩과 SharedArrayBuffer

WASM Threads 제안(현재 대부분 브라우저 지원)을 통해 Web Workers와 SharedArrayBuffer를 사용한 병렬 실행이 가능합니다. 단, COOP(Cross-Origin-Opener-Policy)와 COEP(Cross-Origin-Embedder-Policy) 헤더가 반드시 설정되어야 합니다:

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

## 참고 자료

- [WebAssembly 공식 명세](https://webassembly.github.io/spec/core/)
- [MDN WebAssembly 가이드](https://developer.mozilla.org/en-US/docs/WebAssembly)
- [Wasmtime WASI 런타임](https://wasmtime.dev/)
- [wasm-bindgen 공식 가이드](https://rustwasm.github.io/docs/wasm-bindgen/)
