---
layout: post
title: "커버리지 기반 퍼징(Coverage-Guided Fuzzing) 완전 정복: AFL++와 libFuzzer로 버그를 자동 발굴하는 기술"
date: 2026-08-16
categories: [cs, computer-science]
tags: [fuzzing, afl, libfuzzer, security, testing, coverage, sanitizers, program-analysis]
---

## 퍼징이란 무엇인가

퍼징(Fuzzing)은 프로그램에 무작위 또는 변형된 입력을 대량으로 제공해 크래시, 메모리 오염, 정의되지 않은 동작(UB) 등을 자동으로 찾아내는 소프트웨어 테스팅 기법이다.

1988년 위스콘신 대학의 Barton Miller 교수가 Unix 유틸리티들이 랜덤 입력에 얼마나 취약한지 실험한 것이 퍼징 연구의 시작이다. 당시 테스트한 Unix 툴의 약 25~33%가 크래시를 일으켰다. 이후 자동화된 퍼저들이 등장했고, 2013년 Google이 공개한 AFL(American Fuzzy Lop)이 코드 커버리지를 피드백으로 활용하는 **커버리지 기반 퍼징(Coverage-Guided Fuzzing, CGF)**을 대중화시켰다.

커버리지 기반 퍼징이 기존 무작위 퍼징보다 압도적으로 효과적인 이유는 **탐색 방향성**에 있다. 단순 무작위 입력은 복잡한 조건문(`if magic == 0xDEADBEEF`)을 통과할 확률이 거의 없지만, CGF는 "이 입력이 새로운 코드 경로를 발견했는가"를 기준으로 입력을 진화시킨다.

---

## 왜 필요한가 — 자동화된 취약점 발굴

### 기존 테스팅의 한계

단위 테스트(Unit Test)는 개발자가 예상한 경계 조건만 검증한다. 개발자가 상상하지 못한 입력에서 발생하는 취약점은 테스트 코드가 존재하지 않는다. 코드 리뷰도 마찬가지다. 10만 줄짜리 파서 라이브러리에서 특정 바이트 조합이 힙 오버플로우를 일으키는 패턴을 사람이 찾아내기는 극히 어렵다.

### CGF의 성과

커버리지 기반 퍼징은 다음과 같은 실세계 취약점을 발굴했다:

- **Heartbleed (OpenSSL CVE-2014-0160)**: libFuzzer 튜토리얼에서 재현 가능한 고전 취약점
- **zlib, libpng, libjpeg**에서 수백 개의 메모리 오염 버그
- **Google OSS-Fuzz**: 700개 이상의 오픈소스 프로젝트를 상시 퍼징하여 2024년 기준 10,000개 이상의 취약점 발굴
- **Chrome, Firefox, SQLite, curl** 등 주요 소프트웨어의 보안 버그 수천 개

---

## 커버리지 기반 퍼징의 동작 원리

### 1. 계측(Instrumentation)

퍼저는 소스 코드 컴파일 시 또는 바이너리 변환을 통해 기본 블록(Basic Block)이나 엣지(Edge, 분기 전이) 경계에 **커버리지 추적 코드**를 삽입한다.

AFL과 AFL++는 컴파일 시 `afl-clang-fast` 또는 `afl-gcc`를 사용한다. 내부적으로 LLVM Pass를 통해 각 기본 블록에 고유한 ID를 부여하고, 공유 메모리 비트맵에 `bitmap[cur_id ^ prev_id]++` 형태로 엣지 전이를 기록한다. 이 비트맵이 새로운 패턴을 보이면 해당 입력이 "흥미롭다"고 판단해 코퍼스(corpus)에 추가한다.

### 2. 코퍼스 진화(Corpus Evolution)

```
[초기 코퍼스] → 변이(mutation) → [후보 입력] → 실행 → 
커버리지 증가? → Yes → 코퍼스에 추가
                  No  → 버림
```

### 3. 변이 전략

AFL++가 사용하는 주요 변이 전략:

- **Bit/Byte Flips**: 비트 또는 바이트를 무작위로 뒤집음
- **Arithmetic Mutations**: 정수 필드에 ±1, ±35 등 산술 연산
- **Known Interesting Values**: `0`, `0xFF`, `INT_MAX`, `INT_MIN` 등 경계값 삽입
- **Splicing**: 두 코퍼스 입력을 조합(crossover)
- **Custom Mutators**: Python/C API로 포맷 인식 변이(JSON, PNG 구조 이해)

---

## 실제 구현 예제 1: libFuzzer 퍼즈 타겟 작성

libFuzzer는 LLVM에 내장된 커버리지 기반 퍼징 엔진이다. 테스트 대상 라이브러리와 함께 컴파일되어 in-process 방식으로 동작한다.

```c
// fuzz_json_parser.c
// 컴파일: clang -g -fsanitize=fuzzer,address fuzz_json_parser.c json_parser.c -o fuzzer

#include <stdint.h>
#include <stddef.h>
#include <string.h>
#include "json_parser.h"  /* 테스트할 파서 헤더 */

/* libFuzzer가 반복 호출하는 진입점.
   Data: 퍼저가 생성한 가변 길이 바이트 배열
   Size: 현재 입력의 바이트 수 */
int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    /* 최소 길이 필터: 너무 짧은 입력은 흥미롭지 않음 */
    if (size < 2) return 0;

    /* null-terminated 문자열로 변환 */
    char *buf = (char *)malloc(size + 1);
    if (!buf) return 0;
    memcpy(buf, data, size);
    buf[size] = '\0';

    /* 파서 호출 — 크래시·ASan/UBSan 경고가 발생하면 libFuzzer가 기록 */
    json_value_t *result = json_parse(buf);
    if (result) {
        json_free(result);
    }

    free(buf);
    return 0;  /* 0: 정상 처리됨 (입력을 거부해도 크래시가 아님) */
}
```

**실행 방법**:
```bash
# 빌드 (AddressSanitizer + libFuzzer 동시 활성화)
clang -g -fsanitize=fuzzer,address fuzz_json_parser.c json_parser.c -o fuzzer

# 초기 코퍼스 디렉토리 생성
mkdir corpus
echo '{}' > corpus/seed1.json
echo '{"key":"value"}' > corpus/seed2.json

# 퍼징 실행
./fuzzer corpus/ -max_total_time=3600 -jobs=4

# 크래시 재현
./fuzzer crash-deadbeef1234
```

libFuzzer는 새로운 커버리지를 발견하면 `corpus/` 디렉토리에 해당 입력을 자동으로 저장한다. 크래시가 발생하면 `crash-<hash>` 파일로 저장된다.

---

## 실제 구현 예제 2: AFL++ 퍼징 설정

AFL++는 별도 프로세스로 퍼징하는 방식이라 소스 코드 없는 바이너리도 퍼징할 수 있다(QEMU 모드). 소스가 있는 경우 계측 컴파일이 훨씬 효율적이다.

```bash
#!/usr/bin/env bash
# afl_fuzz_setup.sh

# 1. AFL++ 계측 컴파일
export CC=afl-clang-fast
export CXX=afl-clang-fast++
# AddressSanitizer 통합: 메모리 오염 감지
export AFL_USE_ASAN=1

cd target_project
./configure --disable-shared
make clean && make -j$(nproc)

# 2. 코퍼스 준비
mkdir -p corpus_in corpus_out
# 유효한 소형 샘플 파일 수집 (다양한 구조를 커버할수록 좋음)
cp test/samples/*.input corpus_in/

# 3. AFL++ 실행 (단일 코어)
afl-fuzz -i corpus_in/ -o corpus_out/ -- ./target_binary @@
# @@는 AFL이 생성한 테스트 파일 경로로 치환됨

# 4. 병렬 퍼징 (다중 코어 활용)
# 메인 인스턴스 (Deterministic 변이 + 랜덤 변이)
afl-fuzz -i corpus_in/ -o corpus_out/ -M main_fuzzer -- ./target_binary @@

# 보조 인스턴스 (랜덤 변이만 - 별도 터미널에서 실행)
afl-fuzz -i corpus_in/ -o corpus_out/ -S slave1 -- ./target_binary @@
afl-fuzz -i corpus_in/ -o corpus_out/ -S slave2 -- ./target_binary @@

# 5. 크래시 트리아지 (충돌 재현 및 중복 제거)
afl-triage corpus_out/main_fuzzer/crashes/ ./target_binary @@
```

AFL++가 실행되면 터미널에 통계 대시보드가 표시된다:

```
       american fuzzy lop ++4.21c {main_fuzzer} ...
┌─ process timing ────────────────┬─ overall results ─────┐
│        run time : 0 days, 2 hrs │  cycles done : 47      │
│   last new path : 0 days, 0 min │  total paths : 3,847   │
│ last uniq crash : 0 days, 0 min │ uniq crashes : 12      │
│  last uniq hang : none seen yet │   uniq hangs : 0       │
├─ stage progress ────────────────┼─ map coverage ─────────┤
│  now processing : 234 (6.08%)   │    map density : 8.54% │
│ execs done this : 4,891         │ count coverage : 3.71  │
│  exec speed     : 9,841/sec     └────────────────────────┘
```

`total paths` 수가 계속 증가하면 새로운 코드 경로가 발견되고 있다는 뜻이다. `uniq crashes`는 발견된 고유 크래시 수다.

---

## Sanitizer와의 시너지

퍼저 단독으로는 명시적 크래시(세그멘테이션 폴트 등)만 감지한다. Sanitizer를 함께 사용하면 훨씬 많은 버그 유형을 잡을 수 있다:

| Sanitizer | 감지 대상 | 컴파일 플래그 |
|-----------|----------|-------------|
| ASan (AddressSanitizer) | 힙/스택/전역 버퍼 오버플로우, Use-After-Free | `-fsanitize=address` |
| UBSan (UndefinedBehaviorSanitizer) | 정수 오버플로우, null 역참조, misaligned 접근 | `-fsanitize=undefined` |
| MSan (MemorySanitizer) | 초기화되지 않은 메모리 사용 | `-fsanitize=memory` |
| TSan (ThreadSanitizer) | 데이터 레이스 | `-fsanitize=thread` |

단, ASan과 TSan은 동시에 사용할 수 없다.

---

## 주의사항과 실전 팁

### 코퍼스 품질이 성공의 열쇠

빈 파일이나 완전히 무작위 바이트로 시작하면 퍼저가 유효한 입력 구조를 발견하는 데 오랜 시간이 걸린다. PNG 퍼징이라면 실제 PNG 파일들을, HTTP 파서 퍼징이라면 다양한 HTTP 요청 샘플을 초기 코퍼스로 제공해야 한다. 코퍼스 미니멀화(`afl-cmin`, `llvm-profdata`)로 중복 커버리지를 가진 파일을 제거하면 효율이 올라간다.

### 실행 속도와 코드 격리

퍼저는 초당 수천~수만 번 실행을 반복한다. 실행 경로에 네트워크 I/O, 파일 시스템 접근, 랜덤 시드가 있으면 재현성이 떨어지고 속도가 저하된다. 퍼징 대상 함수는 최대한 순수(pure)하게 만들어야 한다. 상태를 가진 라이브러리는 `LLVMFuzzerInitialize`로 전역 초기화를 한 번만 수행하고, `LLVMFuzzerTestOneInput`에서는 상태를 리셋하는 방식이 권장된다.

### 지속적 통합(CI)과 퍼징 통합

Google의 OSS-Fuzz 프로젝트는 GitHub Actions나 별도 클러스터에서 오픈소스 프로젝트를 상시 퍼징한다. 비공개 프로젝트에서도 CI 파이프라인에 짧은 퍼징 세션(10~30분)을 포함시키는 것이 권장된다. 크래시가 발생하면 최소 재현 케이스를 자동으로 이슈로 등록하는 워크플로우를 구성할 수 있다.

### 구조적 퍼징 (Grammar-Based Fuzzing)

JSON, XML, 프로그래밍 언어 파서처럼 복잡한 문법 구조를 가진 대상은 바이트 수준 변이만으로는 유효한 입력을 생성하기 어렵다. 이 경우 문법 기반 퍼저(Grammar-Based Fuzzer)나 `libprotobuf-mutator` 같은 구조 인식 뮤테이터를 활용하면 훨씬 깊은 코드 경로에 도달할 수 있다.

---

## 참고 자료

- [Google libFuzzer 공식 튜토리얼 (Heartbleed 재현 포함)](https://github.com/google/fuzzing/blob/master/tutorial/libFuzzerTutorial.md)
- [AFL++ 공식 저장소 — 커뮤니티 패치 적용 AFL 발전형](https://github.com/AFLplusplus/AFLplusplus)
