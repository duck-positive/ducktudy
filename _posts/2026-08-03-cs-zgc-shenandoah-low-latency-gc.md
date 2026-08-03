---
layout: post
title: "ZGC와 Shenandoah: 저지연 가비지 컬렉터의 내부 동작 원리"
date: 2026-08-03
categories: [cs, computer-science]
tags: [zgc, shenandoah, garbage-collection, jvm, low-latency, concurrent-gc, colored-pointer, brooks-pointer]
---

## 개요

"JVM은 GC 때문에 C++을 쓸 수 없다" — 한때 실시간 시스템이나 금융 거래 시스템 개발자들이 Java를 외면하던 이유였다. Stop-the-World(STW) 시간이 수백 밀리초에서 수 초에 달할 수 있었기 때문이다.

JDK 11에서 도입된 **ZGC(Z Garbage Collector)**와 JDK 12에 통합된 **Shenandoah GC**는 이 패러다임을 바꿨다. 수 테라바이트의 힙에서도 **1밀리초 미만의 STW 시간**을 달성하며, Java가 C++ 수준의 저지연 시스템에 도전할 수 있게 만들었다.

이 글에서는 두 컬렉터의 내부 동작 원리, 핵심 기술, 성능 특성을 상세히 분석한다.

---

## 전통적 GC의 한계: G1GC를 넘어서

G1GC(Garbage First)는 JDK 9부터 기본 컬렉터로 채택되었고, 수백 메가바이트에서 수십 기가바이트 힙을 위한 균형 잡힌 선택이다. 그러나 G1GC도 다음 상황에서 긴 STW를 피하기 어렵다:

- **Full GC**: 동시 마킹이 힙 성장을 따라잡지 못할 때 발생하는 단일 스레드 Full GC
- **Remark 단계**: 동시 마킹 완료 후 변경된 포인터를 처리하는 STW
- **Mixed GC 이전 초기 마크**: STW 마킹 단계

G1GC의 목표 STW 시간은 약 200ms이다. 주식 거래 시스템이나 게임 서버처럼 10ms 이하의 레이턴시가 요구되는 환경에서는 부족하다.

---

## ZGC 내부 구조

### 설계 철학

ZGC의 핵심 목표: **힙 크기(8MB ~ 16TB)에 관계없이 STW 시간 1ms 미만 유지**

이를 달성하기 위해 ZGC는 거의 모든 GC 작업을 애플리케이션 스레드와 **동시에(Concurrently)** 실행한다. STW가 발생하는 시점은 **루트 스캔(Root Scanning)**뿐이며, 루트 수는 힙 크기가 아니라 스레드 수에 비례한다.

### 컬러드 포인터(Colored Pointers)

ZGC의 가장 독창적인 기술이다. 일반적인 64비트 포인터에서 주소 지정에 실제로 필요한 비트는 42~48비트다. 나머지 비트를 **GC 메타데이터** 저장에 활용한다.

```
64비트 포인터 레이아웃:
Bit 63-46: 미사용 (0)
Bit 45: Finalizable (파이널라이저 도달 가능)
Bit 44: Remapped (재배치 완료)
Bit 43: Marked1 (마킹 라운드 1)
Bit 42: Marked0 (마킹 라운드 0)
Bit 41-0: 실제 객체 주소 (4TB 최대)
```

이 4개의 메타데이터 비트로 ZGC는 별도의 카드 테이블(Card Table)이나 기억 집합(Remembered Set) 없이 객체 상태를 추적한다.

### 로드 배리어(Load Barrier)

컬러드 포인터는 포인터를 읽을 때마다 검사가 필요하다. ZGC는 **로드 배리어(Load Barrier)**를 JIT 컴파일러에 삽입해 이를 처리한다.

```java
// 원본 Java 코드
Object ref = obj.field;

// JIT가 삽입하는 로드 배리어 (개념적 의사 코드)
Object ref = obj.field;
if (!is_good_colored(ref)) {
    ref = slow_path(ref);  // 재배치/마킹 처리
    obj.field = ref;       // self-healing: 포인터 업데이트
}
```

**Self-Healing**: 배리어가 포인터를 수정할 때 해당 필드를 즉시 업데이트한다. 이후 같은 필드에 접근할 때는 이미 올바른 포인터가 되어 있어 배리어 비용이 0이 된다.

### ZGC GC 사이클

```
1. STW: 초기 마크 (Initial Mark)
   - 루트 집합(스택, 전역 변수)을 마킹
   - STW 시간: O(스레드 수)

2. 동시 마크 (Concurrent Mark)
   - 전체 객체 그래프 순회
   - 애플리케이션 스레드와 동시 실행

3. STW: 마크 종료 (Mark End)
   - 동시 마킹 중 변경된 포인터 처리
   - STW 시간: O(SATB 버퍼 크기)

4. 동시 재배치 준비 (Concurrent Prepare for Relocate)
   - 재배치할 Region 집합 계산

5. STW: 재배치 루트 (Relocate Roots)
   - 루트 포인터를 새 주소로 업데이트

6. 동시 재배치 (Concurrent Relocate)
   - 살아있는 객체를 새 Region으로 복사
   - 이전 주소 → 새 주소 매핑을 포워딩 테이블에 기록

7. 동시 재매핑 (Concurrent Remap) - 다음 사이클과 겹침
   - 모든 포인터를 새 주소로 업데이트
```

STW 단계는 1, 3, 5뿐이며, 각각 매우 짧다.

### ZGC 설정 예시

```bash
# 기본 ZGC 활성화 (JDK 15+부터 production ready)
java -XX:+UseZGC \
     -Xms4g -Xmx4g \
     -XX:ZAllocationSpikeTolerance=2 \
     -XX:ZUncommitDelay=300 \
     -Xlog:gc*:gc.log:time,uptime:filecount=5,filesize=20m \
     MyApplication

# JDK 21+ Generational ZGC (더 낮은 CPU 오버헤드)
java -XX:+UseZGC \
     -XX:+ZGenerational \
     -Xms4g -Xmx4g \
     MyApplication
```

```java
// ZGC 진단 출력 예시 분석
// [0.100s][info][gc] GC(0) Garbage Collection (Warmup)
// [0.100s][info][gc,phases] GC(0) Pause Mark Start 0.459ms   ← STW
// [0.111s][info][gc,phases] GC(0) Concurrent Mark 10.612ms
// [0.111s][info][gc,phases] GC(0) Pause Mark End 0.209ms      ← STW
// [0.111s][info][gc,phases] GC(0) Concurrent Process Non-Strong References 0.810ms
// [0.112s][info][gc,phases] GC(0) Concurrent Reset Relocation Set 0.038ms
// [0.112s][info][gc,phases] GC(0) Concurrent Select Relocation Set 0.979ms
// [0.113s][info][gc,phases] GC(0) Pause Relocate Start 0.108ms ← STW
// [0.124s][info][gc,phases] GC(0) Concurrent Relocate 11.007ms

// 핵심: 3번의 STW가 모두 1ms 미만
```

---

## Shenandoah GC 내부 구조

### 개발 배경

Shenandoah는 Red Hat이 개발해 OpenJDK에 기여한 저지연 GC다. ZGC와 목표는 같지만 구현 방식이 근본적으로 다르다.

- **ZGC**: 64비트 포인터의 여유 비트를 활용 (컬러드 포인터)
- **Shenandoah**: 객체 헤더에 추가 포인터를 삽입 (Brooks 포인터)

### Brooks 포인터(Forwarding Pointer)

모든 객체는 **헤더에 자기 자신을 가리키는 포인터를 하나 더** 가진다.

```
일반 객체 레이아웃:
┌─────────────────────────────────┐
│ 마크 워드 (hashcode, 잠금 등)    │
│ 클래스 포인터                    │
│ 인스턴스 필드들...               │
└─────────────────────────────────┘

Shenandoah 객체 레이아웃:
┌─────────────────────────────────┐
│ Brooks 포인터 (자신 or 새 위치) │ ← Shenandoah가 추가
│ 마크 워드                        │
│ 클래스 포인터                    │
│ 인스턴스 필드들...               │
└─────────────────────────────────┘
```

객체를 이동할 때:
1. 새 위치에 객체 복사
2. 이전 위치의 Brooks 포인터를 새 위치로 업데이트

이후 이전 주소로의 모든 포인터는 Brooks 포인터를 통해 새 위치로 자동 리다이렉트된다.

**메모리 오버헤드**: 모든 객체에 64비트 포인터 하나 추가 → 일반적으로 객체당 8바이트 증가

### 읽기/쓰기 배리어

```java
// 원본 코드
Object x = obj.field;
obj.field = value;

// Shenandoah 읽기 배리어 (약한 배리어, JDK 13부터)
Object x = obj.field;
// 읽기 배리어: 간접 포인터 역참조 없이 직접 읽기

// Shenandoah 쓰기 배리어 (동시 업데이트 지원)
if (is_in_collection_set(obj)) {
    obj = obj.brooks_ptr;  // 새 위치로 이동
}
obj.field = value;
```

### Shenandoah GC 사이클

```
1. STW: 초기 마크 (Initial Mark)
   - 루트 스캔
   
2. 동시 마크 (Concurrent Mark)
   - 전체 힙 마킹
   
3. STW: 최종 마크 (Final Mark)
   - SATB 버퍼 플러시, 마킹 완료
   
4. 동시 청소 (Concurrent Cleanup)
   - 완전히 빈 Region 즉시 회수
   
5. 동시 이동 (Concurrent Evacuation)
   - 살아있는 객체를 새 Region으로 복사 ← ZGC와의 핵심 차이
   - 읽기/쓰기 배리어로 동시 접근 처리
   
6. STW: 초기 업데이트 참조 (Init Update References)
   - 포인터 업데이트 시작 마킹
   
7. 동시 업데이트 참조 (Concurrent Update References)
   - 모든 포인터를 새 주소로 업데이트
   
8. STW: 최종 업데이트 참조 (Final Update References)
   - 루트 포인터 업데이트 완료
```

Shenandoah의 특징: **동시 이동(Concurrent Evacuation)**. 애플리케이션이 실행 중에도 객체를 이동시킨다. ZGC는 이동 후 동시 재매핑을 수행하지만, Shenandoah는 이동 자체를 동시에 처리한다.

### Shenandoah 설정 예시

```bash
# Shenandoah 활성화
java -XX:+UseShenandoahGC \
     -Xms4g -Xmx4g \
     -XX:ShenandoahGCMode=iu \  # Incremental Update 모드 (기본)
     -XX:ShenandoahPacingMaxDelay=10 \
     -Xlog:gc*:gc.log:time,uptime:filecount=5,filesize=20m \
     MyApplication

# GC 동작 확인
# [0.530s] GC(0) Using 4 of 4 workers for init marking
# [0.530s] GC(0) Pause Init Mark (process weakrefs) 0.92ms ← STW
# [0.530s] GC(0) Concurrent marking 18.790ms
# [0.549s] GC(0) Pause Final Mark (process weakrefs) 1.51ms ← STW
# [0.549s] GC(0) Concurrent cleanup 0.293ms
# [0.550s] GC(0) Concurrent evacuation 7.314ms
# [0.557s] GC(0) Pause Init Update Refs 0.028ms ← STW
# [0.557s] GC(0) Concurrent update references 9.447ms
# [0.567s] GC(0) Pause Final Update Refs 0.334ms ← STW
```

---

## ZGC vs Shenandoah: 비교 분석

```java
/**
 * 간단한 벤치마크: 할당/해제 부하 하에서 지연 시간 측정
 */
public class GCLatencyBenchmark {
    private static final int ARRAY_SIZE = 1024;
    private static final int ITERATIONS = 1_000_000;

    public static void main(String[] args) throws InterruptedException {
        List<long[]> live = new ArrayList<>();
        long[] latencies = new long[ITERATIONS];

        for (int i = 0; i < ITERATIONS; i++) {
            long start = System.nanoTime();

            // 할당
            live.add(new long[ARRAY_SIZE]);

            // 오래된 객체 간헐적 해제
            if (live.size() > 10000) {
                live.remove(0);
            }

            latencies[i] = System.nanoTime() - start;
        }

        // 퍼센타일 분석
        Arrays.sort(latencies);
        System.out.printf("p50: %.2f us%n", latencies[(int)(ITERATIONS * 0.50)] / 1000.0);
        System.out.printf("p99: %.2f us%n", latencies[(int)(ITERATIONS * 0.99)] / 1000.0);
        System.out.printf("p99.9: %.2f us%n", latencies[(int)(ITERATIONS * 0.999)] / 1000.0);
        System.out.printf("max: %.2f us%n", latencies[ITERATIONS - 1] / 1000.0);
    }
}
```

### 성능 특성 비교표

| 특성 | G1GC | ZGC | Shenandoah |
|------|------|-----|------------|
| STW 목표 | ~200ms | < 1ms | < 10ms |
| 최대 힙 | ~수십 GB | 16TB | ~수 TB |
| CPU 오버헤드 | 낮음 | 5~10% | 5~15% |
| 메모리 오버헤드 | 낮음 | 낮음 (포인터 비트) | 중간 (Brooks 포인터) |
| 처리량(Throughput) | 높음 | 중간 | 중간 |
| JDK 지원 | 9+ | 11+ (15+ 권장) | 12+ |
| 세대별 GC | 있음 | JDK 21+ (Generational) | JDK 15+ (Generational) |

---

## 언제 어떤 컬렉터를 선택해야 하나?

### G1GC를 선택하는 경우
- 힙 크기 4GB ~ 32GB
- 전반적인 처리량이 레이턴시보다 중요
- 레이턴시 목표: ~200ms 허용
- 성숙한 기능이 필요한 프로덕션 환경

### ZGC를 선택하는 경우
- 힙 크기 32GB 이상의 대형 힙
- 레이턴시 목표: < 1ms
- JDK 21+ 환경에서 Generational ZGC 활용
- 최신 하드웨어 (충분한 CPU 코어)

### Shenandoah를 선택하는 경우
- 힙 크기 8GB ~ 수백 GB
- 레이턴시 목표: < 10ms
- Red Hat Enterprise Linux / OpenJDK 배포판 사용
- 더 작은 힙에서 ZGC보다 나은 처리량 필요

---

## 실전 튜닝 가이드

### ZGC 튜닝

```bash
# 힙 크기: min = max로 설정해 리사이징 오버헤드 제거
-Xms16g -Xmx16g

# GC 스레드 수 조절 (기본: CPU 코어의 1/4 ~ 1/8)
-XX:ConcGCThreads=4

# 힙 점유율 기반 GC 트리거 (기본 75%)
-XX:ZCollectionInterval=0   # 시간 기반 트리거 비활성화

# 미사용 메모리 반납 지연 (컨테이너 환경에서 유용)
-XX:ZUncommitDelay=300   # 5분

# Generational ZGC (JDK 21+, 처리량-레이턴시 균형 개선)
-XX:+ZGenerational
```

### Shenandoah 튜닝

```bash
# GC 모드 선택
-XX:ShenandoahGCMode=iu       # Incremental Update (기본, 좋은 처리량)
-XX:ShenandoahGCMode=passive  # GC를 최대한 지연 (테스트용)
-XX:ShenandoahGCMode=aggressive # 모든 할당 전 GC (테스트용)

# 휴리스틱 선택
-XX:ShenandoahGCHeuristics=adaptive    # 기본, 대부분 환경에 적합
-XX:ShenandoahGCHeuristics=static      # 고정 트리거
-XX:ShenandoahGCHeuristics=compact     # 힙 컴팩션 우선
-XX:ShenandoahGCHeuristics=throughput  # 처리량 우선

# 페이싱: GC가 느릴 때 할당 스레드를 늦춰 OOM 방지
-XX:ShenandoahPacingMaxDelay=10   # ms, 기본 10ms
```

---

## 주의사항

1. **CPU 오버헤드**: 동시 GC는 애플리케이션과 CPU 자원을 경쟁한다. CPU 코어가 부족한 환경(1~2 코어)에서는 STW GC보다 더 나쁜 처리량을 보일 수 있다.

2. **배리어 비용**: 로드/쓰기 배리어는 모든 포인터 접근에 오버헤드를 추가한다. 캐시 지역성이 좋은 코드에서 체감 성능 저하가 발생할 수 있다.

3. **Floating Garbage**: 동시 마킹 중 죽은 객체가 이미 라이브로 마킹된 경우, 다음 사이클까지 회수되지 않는다. 메모리 사용량이 일시적으로 높아질 수 있다.

4. **힙 크기 여유**: 동시 이동 중 이전/새 위치가 동시에 존재하므로, 힙 사용률이 높아지는 시점이 있다. ZGC/Shenandoah는 최대 힙의 75~80% 이상 사용하지 않도록 설계하는 것이 좋다.

5. **JVM 버전 선택**: ZGC와 Shenandoah 모두 JDK 버전마다 significant한 성능 개선이 이루어지고 있다. 가능하면 최신 LTS(JDK 21 이상)를 사용하라.

---

## 참고 자료

- [ZGC vs Shenandoah: Ultra-Low Latency GC for Java - Java Code Geeks](https://www.javacodegeeks.com/2025/04/zgc-vs-shenandoah-ultra-low-latency-gc-for-java.html)
- [Deep Dive into ZGC - ACM Transactions on Programming Languages and Systems](https://dl.acm.org/doi/full/10.1145/3538532)
- [OpenJDK ZGC Project](https://openjdk.org/projects/zgc/)
- [Shenandoah GC Wiki - OpenJDK](https://wiki.openjdk.org/display/shenandoah/Main)
