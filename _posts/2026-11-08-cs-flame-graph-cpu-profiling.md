---
layout: post
title: "플레임 그래프(Flame Graph) 완전 정복: CPU 프로파일링으로 성능 병목을 한눈에 찾는 법"
date: 2026-11-08
categories: [cs, computer-science]
tags: [flame-graph, profiling, perf, async-profiler, performance, linux, jvm, cpu]
---

## 1. 플레임 그래프란 무엇인가?

플레임 그래프(Flame Graph)는 소프트웨어의 CPU 사용 패턴을 시각화하는 기법으로, Brendan Gregg(Netflix, 현 Oracle)가 2011년에 고안했다. 기존의 텍스트 기반 프로파일 출력이나 콜 트리는 수천 줄의 출력 중에서 병목을 찾기 어려웠는데, 플레임 그래프는 이를 직관적인 SVG 이미지 하나로 압축한다.

**핵심 개념 세 가지:**

1. **X축(너비)**: 샘플링된 스택 프레임의 빈도. 넓을수록 CPU 시간을 많이 차지함을 의미한다. 단, 시간 순서가 아닌 알파벳 정렬이므로 "왼쪽이 먼저 실행된 것"이 아님을 유의해야 한다.
2. **Y축(높이)**: 콜 스택의 깊이. 아래가 루트 프레임(main, 스레드 진입점), 위가 최상위 호출이다.
3. **색상**: 기본 플레임 그래프에서 색상은 임의이며 의미를 갖지 않는다. 단, Differential Flame Graph에서는 빨간색/파란색으로 증가/감소를 표현한다.

플레임 그래프를 읽는 핵심 질문은 하나다: **"스택 최상단(top of stack)에서 가장 넓은 프레임이 어디인가?"** 그것이 CPU를 가장 많이 소비하는 함수이다.

## 2. 왜 플레임 그래프가 필요한가?

기존 프로파일링 도구들이 가진 한계를 이해해야 플레임 그래프의 필요성을 체감할 수 있다.

### 텍스트 프로파일의 한계

Java의 `jstack`이나 `gprof` 출력물은 각 함수가 소비한 시간을 나열하지만, 호출 맥락을 잃어버린다. `hashCode()`가 CPU의 10%를 차지한다는 사실은 알아도, **어디서** 호출되어 비용이 발생하는지 알 수 없다.

### 프로파일러의 Safepoint 편향 문제

JVM 기반 프로파일러 중 JVMTI(Java Virtual Machine Tool Interface)를 사용하는 것들은 safepoint에서만 스택을 수집한다. Safepoint란 JVM이 모든 스레드를 일시 정지시킬 수 있는 지점인데, 문제는 긴 루프 내부나 JNI 코드, 특정 최적화된 코드 구간은 safepoint가 드물다는 것이다. 결과적으로 CPU를 많이 쓰는 구간이 샘플에서 누락될 수 있다.

### 분산 시스템에서의 필요성

마이크로서비스 환경에서 특정 서비스의 레이턴시가 높을 때, 로그만으로는 "어느 함수에서 CPU를 많이 쓰는가"를 알기 어렵다. 플레임 그래프 하나로 1분 내에 병목 함수를 특정할 수 있다.

## 3. 플레임 그래프 생성 방법

### 방법 1: Linux `perf` + FlameGraph 스크립트 (C/C++/Go/Rust)

Linux 시스템 전체 또는 특정 PID의 CPU 사용을 샘플링한다.

```bash
# 1단계: perf로 스택 샘플 수집 (99Hz, 30초, 모든 콜 스택 포함)
sudo perf record -F 99 -a -g -- sleep 30
# 특정 PID만 프로파일하려면:
sudo perf record -F 99 -p <PID> -g -- sleep 30

# 2단계: 텍스트 형태의 스택으로 변환
sudo perf script > out.perf

# 3단계: FlameGraph 스크립트로 SVG 생성
# (https://github.com/brendangregg/FlameGraph 에서 스크립트 클론)
./stackcollapse-perf.pl out.perf > out.folded
./flamegraph.pl out.folded > flamegraph.svg

# 브라우저로 열기
open flamegraph.svg
```

**주의사항:**
- `/proc/sys/kernel/perf_event_paranoid`를 1 이하로 설정해야 한다: `sudo sysctl -w kernel.perf_event_paranoid=1`
- 디버그 심볼이 없으면 함수명 대신 주소가 표시된다. 패키지 디버그 심볼을 설치하거나 `-fno-omit-frame-pointer`로 컴파일해야 한다.
- Go 1.12+는 기본적으로 frame pointer를 포함해 perf와 잘 연동된다.

### 방법 2: async-profiler (JVM — Java/Kotlin/Scala)

async-profiler는 `AsyncGetCallTrace` API와 `perf_events`를 조합하여 safepoint 편향 없이 정확한 스택을 수집한다.

```bash
# 설치 (Linux x64 기준)
wget https://github.com/async-profiler/async-profiler/releases/download/v3.0/async-profiler-3.0-linux-x64.tar.gz
tar xzf async-profiler-3.0-linux-x64.tar.gz
cd async-profiler-3.0-linux-x64

# 실행 중인 JVM 프로세스에 붙이기 (30초 CPU 프로파일, HTML 형식)
./bin/asprof -d 30 -e cpu -f profile.html <PID>

# 메모리 할당 프로파일 (힙 압박 원인 찾기)
./bin/asprof -d 30 -e alloc -f alloc.html <PID>

# 락 경합 프로파일
./bin/asprof -d 30 -e lock -f lock.html <PID>

# Wall-clock 프로파일 (블로킹 스레드 포함 — I/O 병목 찾기에 유용)
./bin/asprof -d 30 -e wall -t -f wall.html <PID>
```

HTML 형식으로 출력하면 인터랙티브 플레임 그래프가 생성되어 함수 클릭으로 확대/축소, 텍스트 검색이 가능하다.

## 4. 플레임 그래프 읽기: 실전 패턴

### 패턴 1: 넓은 단일 함수 (CPU Hog)

```
|          processRequest()           |   <- 넓고 단독으로 솟아있음 → CPU 독점
|       _encrypt()   |   parseJson()  |
```

최상단에 단일 함수가 넓게 자리 잡고 있으면 그 함수 자체가 CPU를 소비하는 것이다. 직접 최적화 대상이다.

### 패턴 2: 깊은 스택 + 좁은 최상단 (추적 필요)

```
|A|B|C|D|E|F|G|H|       <- 모두 좁음 → 어디가 문제?
```

이 경우 `A`에서 여러 경로로 분기되어 각각 조금씩 소비하는 것이다. 넓은 `A`에서 하위 경로들을 분석해야 한다.

### 패턴 3: 평탄한 그래프 (I/O 대기)

CPU 프로파일에서 거의 아무것도 없다면 프로그램이 I/O를 기다리는 것이다. 이때는 `-e wall` (Wall-clock 프로파일)로 블로킹 스레드까지 포함해서 분석해야 한다.

## 5. Differential Flame Graph: 배포 전후 비교

새 버전 배포 후 성능이 나빠졌는지 비교할 때 Differential Flame Graph를 사용한다.

```bash
# 버전 A 프로파일
./bin/asprof -d 30 -e cpu -f v1.collapsed <PID_V1> --format collapsed

# 버전 B 프로파일
./bin/asprof -d 30 -e cpu -f v2.collapsed <PID_V2> --format collapsed

# Differential SVG 생성 (빨간색: B에서 증가, 파란색: B에서 감소)
./difffolded.pl v1.collapsed v2.collapsed | ./flamegraph.pl > diff.svg
```

## 6. 프로그래밍 언어별 플레임 그래프 도구 정리

| 언어 | 도구 | 비고 |
|------|------|------|
| C/C++ | perf + FlameGraph | `-fno-omit-frame-pointer` 필수 |
| Go | perf 또는 `go tool pprof` | 1.12+부터 frame pointer 기본 포함 |
| Java/Kotlin | async-profiler | Safepoint 편향 없음 |
| Python | `py-spy` | sudo 권한으로 attach 가능 |
| Node.js | `0x` 또는 `--prof` | V8 내장 샘플러 활용 |
| Rust | perf + addr2line | DWARF 심볼 필요 |

## 7. 주의사항 및 고급 팁

**인라인 함수 문제:** 컴파일러 최적화로 함수가 인라인되면 플레임 그래프에 나타나지 않는다. Release 빌드 분석 시 `-fno-inline`을 추가해 디버그 빌드에서 확인하거나, DWARF unwinding을 사용하면 일부 인라인 정보를 복원할 수 있다.

**샘플링 주파수 선택:** 너무 높은 주파수(예: 999Hz)는 오버헤드를 증가시키고, 너무 낮으면(예: 10Hz) 짧게 실행되는 함수가 누락된다. 99Hz는 오버헤드와 정확성의 균형점으로, perf의 기본값으로 널리 사용된다.

**컨테이너 환경:** Docker/Kubernetes 내부에서 perf를 실행하려면 `--privileged` 모드가 필요하거나, `SYS_PTRACE` 및 `SYS_PERFMON` capability가 필요하다. async-profiler는 attach 방식이라 컨테이너 내부에서도 sudo 없이 동작 가능하다.

**지속적 프로파일링(Continuous Profiling):** 운영 환경에서 항상 낮은 오버헤드(< 1%)로 샘플을 수집하고 저장하는 Continuous Profiling 솔루션(Pyroscope, Parca, Grafana Pyroscope)을 사용하면 사후에 특정 시점의 플레임 그래프를 조회할 수 있어 인시던트 대응 시간을 크게 단축할 수 있다.

## 참고 자료
- [Brendan Gregg's Flame Graphs (공식 사이트)](https://www.brendangregg.com/flamegraphs.html)
- [FlameGraph GitHub Repository](https://github.com/brendangregg/FlameGraph)
- [async-profiler GitHub Repository](https://github.com/async-profiler/async-profiler)
- [Brendan Gregg - USENIX ATC 2017 Flame Graphs 슬라이드](https://www.brendangregg.com/Slides/USENIX_ATC2017_flamegraphs.pdf)
