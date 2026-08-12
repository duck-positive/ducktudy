---
layout: post
title: "Profile-Guided Optimization(PGO) 완전 정복: 런타임 프로파일로 컴파일러를 훈련시켜 5~20% 성능을 끌어내는 법"
date: 2026-08-12
categories: [cs, computer-science]
tags: [pgo, profile-guided-optimization, compiler, gcc, clang, llvm, performance, optimization]
---

컴파일러는 코드를 최적화할 때 수많은 결정을 내린다. 이 함수를 인라인해야 할까? 이 루프를 언롤링해야 할까? 이 분기는 참일 가능성이 더 높을까, 거짓일 가능성이 더 높을까? 문제는 컴파일 시점에 이 질문들에 대한 답이 없다는 것이다. 컴파일러는 추측(heuristic)에 의존할 수밖에 없고, 그 추측이 틀리면 최적화가 오히려 역효과를 낸다. **Profile-Guided Optimization(PGO)**은 이 근본적인 한계를 "실제로 프로그램을 실행해 데이터를 수집한 뒤, 그 데이터를 컴파일에 피드백한다"는 방식으로 돌파한다.

## 개념 설명: PGO의 작동 원리

PGO는 세 단계로 이루어진다.

### Phase 1: 계측 빌드 (Instrumented Build)

컴파일러는 프로그램 곳곳에 **계측 코드(instrumentation)**를 삽입한 바이너리를 생성한다. 이 코드는 실행 중에 다음을 추적한다:

- **엣지 프로파일**: 각 제어 흐름 엣지(분기의 각 방향)가 몇 번 실행되었는가
- **함수 호출 횟수**: 어떤 함수가 얼마나 자주 호출되는가
- **값 프로파일(선택적)**: 특정 간접 호출(virtual call)의 대상이 되는 함수의 분포

계측 코드는 성능 오버헤드가 있다. 계측 빌드 바이너리는 보통 원본보다 10~30% 느리게 동작한다.

### Phase 2: 프로파일 수집 (Profile Gather)

계측된 바이너리를 **대표적인 워크로드**로 실행한다. "대표적"이 핵심이다. PGO의 효과는 훈련 데이터가 실제 프로덕션 워크로드를 얼마나 잘 반영하는지에 달려있다. 실행 후 프로파일 데이터 파일(`.gcda`, `.profraw`)이 생성된다.

### Phase 3: 최적화 빌드 (Optimized Build)

컴파일러가 프로파일 데이터를 읽어 다음 최적화에 활용한다:

| 최적화 기법 | PGO 활용 방식 |
|------------|--------------|
| **함수 인라이닝** | 호출 빈도가 높은 함수를 적극적으로 인라인, 드물게 호출되는 함수는 아웃라인 |
| **코드 레이아웃** | 핫 코드(hot code)를 메모리상 인접하게 배치해 I-Cache 효율 극대화 |
| **분기 예측 힌트** | 더 자주 실행되는 분기를 "expected" 방향으로 지정 |
| **루프 최적화** | 실제 반복 횟수 기반으로 언롤링 여부 결정 |
| **가상 함수 호출 최적화** | 특정 타입이 압도적으로 많으면 직접 호출로 변환(devirtualization) |
| **콜드/핫 분리** | 거의 실행되지 않는 코드를 별도 섹션으로 분리해 페이지 캐시 효율 향상 |

## 왜 PGO가 필요한가

### 정적 분석의 근본적 한계

컴파일러는 코드의 구조를 분석할 수 있지만, 실행 패턴은 알 수 없다. 예를 들어:

```cpp
void process(int type) {
    if (type == 1) {          // 99%의 경우 type이 1
        fast_path();
    } else {
        slow_path();
    }
}
```

PGO 없이는 컴파일러가 어느 분기가 더 자주 실행될지 모른다. 기껏해야 `__builtin_expect` 같은 힌트에 의존해야 한다. PGO를 적용하면 컴파일러는 `type == 1` 분기를 CPU 분기 예측기에 유리한 방향으로 배치하고, `slow_path()`를 콜드 섹션으로 내보낸다.

### 실제 성능 향상 사례

- **Chromium**: PGO 적용으로 V8 JavaScript 엔진이 5~8% 빨라졌다.
- **CPython**: 파이썬 3.12부터 PGO를 기본으로 활성화. 벤치마크에서 최대 15% 향상.
- **Clang 자체**: LLVM 공식 문서는 Clang을 PGO로 빌드하면 컴파일 속도가 15~20% 향상된다고 명시한다.
- **MySQL**: PGO 적용으로 SELECT-heavy 워크로드에서 10~15% 처리량 증가.

## 실제 구현 예제

### 예제 1: GCC/Clang PGO 워크플로우

```bash
#!/bin/bash
# PGO 3단계 빌드 스크립트 (GCC 예시)
set -e

SOURCE="myapp.cpp"
OPTIMIZE_FLAGS="-O2 -march=native"

echo "=== Phase 1: 계측 빌드 ==="
g++ ${OPTIMIZE_FLAGS} -fprofile-generate -o myapp_instrumented ${SOURCE}
echo "계측 바이너리 생성 완료"

echo "=== Phase 2: 프로파일 수집 ==="
# 대표적인 워크로드로 실행
./myapp_instrumented --benchmark --input=typical_workload.dat
# .gcda 파일이 현재 디렉토리에 생성됨
echo "프로파일 데이터 수집 완료 (*.gcda)"

echo "=== Phase 3: 최적화 빌드 ==="
g++ ${OPTIMIZE_FLAGS} -fprofile-use -fprofile-correction -o myapp_pgo ${SOURCE}
echo "PGO 최적화 빌드 완료"

echo "=== 성능 비교 ==="
echo "기본 최적화:"
g++ ${OPTIMIZE_FLAGS} -o myapp_baseline ${SOURCE}
time ./myapp_baseline --benchmark --input=typical_workload.dat

echo "PGO 최적화:"
time ./myapp_pgo --benchmark --input=typical_workload.dat
```

Clang/LLVM을 사용할 때는 프로파일 병합 단계가 추가된다:

```bash
# Clang PGO 워크플로우
# Phase 1
clang++ -O2 -fprofile-instr-generate -o myapp_instrumented myapp.cpp

# Phase 2
LLVM_PROFILE_FILE="myapp-%p.profraw" ./myapp_instrumented --benchmark

# 프로파일 병합 (여러 실행 결과를 하나로)
llvm-profdata merge -output=myapp.profdata myapp-*.profraw

# Phase 3
clang++ -O2 -fprofile-instr-use=myapp.profdata -o myapp_pgo myapp.cpp
```

### 예제 2: 실전 C++ 예제로 PGO 효과 직접 확인하기

```cpp
// hot_cold.cpp: 핫/콜드 경로가 명확한 예시
#include <iostream>
#include <vector>
#include <chrono>
#include <random>
using namespace std::chrono;

// 실제로는 99%의 경우 is_valid == true인 데이터를 처리
struct Record {
    int value;
    bool is_valid;
};

// 콜드 경로: 거의 실행되지 않음
void __attribute__((noinline)) handle_invalid(const Record& r) {
    std::cerr << "Invalid record: " << r.value << std::endl;
}

// 핫 경로: 거의 항상 실행됨
void __attribute__((noinline)) process_valid(const Record& r, long long& sum) {
    sum += r.value * r.value;   // 간단한 연산
}

long long process_records(const std::vector<Record>& records) {
    long long sum = 0;
    for (const auto& r : records) {
        if (r.is_valid) {
            process_valid(r, sum);    // 99% 실행
        } else {
            handle_invalid(r);         // 1% 실행
        }
    }
    return sum;
}

int main() {
    // 데이터 생성: 1%만 invalid
    std::mt19937 rng(42);
    std::uniform_int_distribution<int> val_dist(1, 1000);
    std::uniform_int_distribution<int> valid_dist(0, 99);

    std::vector<Record> records;
    records.reserve(10'000'000);
    for (int i = 0; i < 10'000'000; ++i) {
        records.push_back({val_dist(rng), valid_dist(rng) != 0});
    }

    // 워밍업
    volatile long long warmup = process_records(records);

    // 성능 측정
    auto start = high_resolution_clock::now();
    long long result = 0;
    for (int iter = 0; iter < 10; ++iter) {
        result += process_records(records);
    }
    auto end = high_resolution_clock::now();

    double ms = duration_cast<microseconds>(end - start).count() / 1000.0;
    std::cout << "결과: " << result << "\n";
    std::cout << "시간: " << ms << " ms (10회 평균)\n";
    std::cout << "처리량: " << (10 * 10'000'000) / (ms / 1000.0) / 1e6 << " M records/sec\n";

    return 0;
}
```

빌드 및 비교:
```bash
# 기본 최적화
g++ -O2 -o hot_cold_baseline hot_cold.cpp

# PGO Phase 1
g++ -O2 -fprofile-generate -o hot_cold_instr hot_cold.cpp
./hot_cold_instr   # 프로파일 수집

# PGO Phase 3
g++ -O2 -fprofile-use -fprofile-correction -o hot_cold_pgo hot_cold.cpp

echo "=== Baseline ==="
./hot_cold_baseline

echo "=== PGO ==="
./hot_cold_pgo

# 어셈블리 차이 확인
objdump -d hot_cold_baseline | grep -A5 "<_Z14process_recordsRKSt6vectorI6RecordSaIS0_EE>"
objdump -d hot_cold_pgo | grep -A5 "<_Z14process_recordsRKSt6vectorI6RecordSaIS0_EE>"
```

PGO 적용 전후의 어셈블리를 비교해보면, 적용 후에는:
- `is_valid == true` 분기가 폴스루(fall-through) 방향으로 배치되어 CPU 분기 예측기가 기본적으로 맞는 방향이 된다.
- `handle_invalid` 함수가 콜드 섹션(`.text.cold`)으로 분리되어 I-Cache 압박을 줄인다.

## 주의사항 및 팁

### 1. 훈련 워크로드의 대표성이 가장 중요하다

PGO는 훈련 데이터와 다른 패턴의 워크로드에서 오히려 성능을 저하시킬 수 있다. 프로덕션 트래픽의 다양한 케이스를 커버하는 대표 벤치마크를 만드는 것이 필수다. 단일 케이스만으로 훈련하면 해당 케이스에는 완벽하지만 나머지에는 역효과가 난다.

### 2. AutoFDO: 프로덕션에서 직접 수집하기

계측 빌드의 성능 오버헤드가 부담스럽다면 **AutoFDO(Auto Feedback-Directed Optimization)**를 사용한다. `perf` 같은 하드웨어 성능 카운터로 프로덕션에서 샘플링 프로파일을 수집한 뒤 이를 컴파일에 활용한다. 오버헤드는 1~5%에 불과하다.

```bash
# 프로덕션 서버에서 샘플링 (루트 권한 필요)
perf record -e br_inst_retired.near_taken:pp -b -o perf.data -- ./myapp --production

# AutoFDO 형식으로 변환 (google/autofdo 도구 사용)
create_gcov --binary=myapp --profile=perf.data --gcov=myapp.gcov

# 프로파일 사용
gcc -O2 -fauto-profile=myapp.gcov -o myapp_afdo myapp.cpp
```

### 3. BOLT: 링크 이후 최적화

PGO가 컴파일 시점의 최적화라면, **BOLT(Binary Optimization and Layout Tool)**는 이미 컴파일된 바이너리를 프로파일 기반으로 재배치하는 도구다. PGO와 BOLT를 함께 사용하면 추가 5~10%의 성능 향상을 얻을 수 있다. Meta 내부에서 Clang 컴파일러 자체에 BOLT를 적용해 20% 이상 빌드 속도를 향상시킨 사례가 있다.

### 4. 소스 변경 시 프로파일 무효화

소스 코드가 크게 변경되면 기존 프로파일 데이터가 새로운 코드 구조와 맞지 않는 **stale profile** 문제가 발생한다. GCC의 `-fprofile-correction` 플래그는 이를 어느 정도 완화하지만, 주기적으로 프로파일을 재수집하는 CI 파이프라인을 구축해야 한다. Clang은 소스 변경을 감지해 경고를 출력한다.

### 5. CI/CD 통합 전략

프로덕션 릴리즈 빌드에만 PGO를 적용하는 것이 일반적이다. 개발 중에는 빠른 피드백이 중요하므로 PGO 없이 빌드하고, 릴리즈 파이프라인에서만 3단계 빌드를 수행한다.

```yaml
# GitHub Actions 예시
jobs:
  release-build:
    steps:
      - name: Phase 1 - Instrumented Build
        run: cmake -DCMAKE_CXX_FLAGS="-fprofile-generate" -B build_instr
        
      - name: Phase 2 - Run Benchmark for Profile
        run: ./build_instr/myapp --benchmark
        
      - name: Phase 3 - PGO Build
        run: |
          cmake -DCMAKE_CXX_FLAGS="-fprofile-use -fprofile-correction" -B build_pgo
          cmake --build build_pgo --config Release
```

### 6. C 이외의 언어에서의 PGO

- **Rust**: `RUSTFLAGS="-Cprofile-generate=/tmp/pgo-data"` → 실행 → `llvm-profdata merge` → `RUSTFLAGS="-Cprofile-use=merged.profdata"`
- **Go**: `go build -pgo=auto` (Go 1.21+)는 `default.pgo` 파일을 자동으로 인식한다.
- **Java**: JVM의 JIT 컴파일러(HotSpot)는 런타임에 프로파일을 수집하고 적용하는 것을 내부적으로 수행한다. GraalVM AOT 모드에서는 명시적 PGO가 지원된다.

## 정리

PGO는 "컴파일러가 추측해야 했던 것들을 실측 데이터로 대체한다"는 단순하지만 강력한 아이디어다. 특히 CPU 바운드 워크로드에서 5~20%의 성능 향상은 하드웨어 업그레이드 없이 얻을 수 있는 가장 비용 효율적인 방법 중 하나다. 핵심은 훈련 데이터의 대표성이다. 프로파일이 실제 프로덕션 패턴을 반영하지 못하면 PGO는 오히려 독이 될 수 있다. AutoFDO로 프로덕션 트래픽을 직접 수집하고, CI/CD에 통합해 릴리즈마다 자동으로 프로파일을 갱신하는 파이프라인을 구축하는 것이 PGO의 올바른 활용법이다.

## 참고 자료
- [GitHub: lto-pgo — LTO와 PGO 효과 비교 예제](https://github.com/the-risk-taker/lto-pgo)
- [GitHub: facebook/zstd — PGO 적용 사례가 포함된 대규모 C 프로젝트](https://github.com/facebook/zstd)
