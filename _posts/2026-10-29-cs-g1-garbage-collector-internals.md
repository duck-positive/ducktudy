---
layout: post
title: "G1 가비지 컬렉터 완전 정복: 리전 기반 힙과 예측 가능한 GC 일시 정지의 비밀"
date: 2026-10-29
categories: [cs, computer-science]
tags: [jvm, garbage-collection, g1gc, java, memory-management, hotspot]
---

Java 9부터 기본 가비지 컬렉터로 채택된 G1(Garbage First) GC는 기존 CMS, Parallel GC의 한계를 극복하기 위해 설계된 지역 기반(region-based) 컬렉터다. "예측 가능한 일시 정지 시간(Predictable Pause Time)"을 목표로, 수 GB~수십 GB의 대형 힙에서도 밀리초 단위의 GC 일시 정지를 실현한다. 이 글에서는 G1 GC의 내부 구조, 동작 원리, 튜닝 방법을 깊이 분석한다.

## 1. G1 GC가 등장한 이유

### 기존 GC의 한계

**Parallel GC (Throughput Collector):**
- 전체 힙을 Eden/Survivor/Old로 고정 구분
- Old Generation 수집 시 Stop-The-World(STW) 전체 힙 수집 → 힙이 클수록 긴 일시 정지
- 수십 초의 Full GC가 발생 가능

**CMS (Concurrent Mark Sweep):**
- Old GC를 대부분 동시(Concurrent)에 수행 → 짧은 일시 정지
- 단점: **메모리 단편화(Fragmentation)**. CMS는 압축(Compaction)을 하지 않아 장기 운영 시 단편화 심화
- **Concurrent Mode Failure**: 동시 수집 중 Young GC가 Old Gen 공간을 초과하면 Full GC 발생

G1은 이 두 가지 문제를 동시에 해결하고자 했다:
1. 예측 가능한 짧은 일시 정지 (`-XX:MaxGCPauseMillis`)
2. 점진적 압축으로 단편화 방지

## 2. G1 GC의 핵심 개념: 리전(Region)

### 힙 분할 방식

G1의 가장 큰 혁신은 힙을 **동일 크기의 리전(Region)**으로 분할하는 것이다:

```
전통적 GC 힙:
┌──────────────┬──────────┬────────────────────────┐
│  Eden Space  │ Survivor │    Old Generation      │
└──────────────┴──────────┴────────────────────────┘

G1 GC 힙:
┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
│E │E │O │O │E │S │O │H │F │O │E │E │S │O │F │O │
└──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
E=Eden, S=Survivor, O=Old, H=Humongous, F=Free
```

- **리전 크기**: 힙 크기에 따라 1MB ~ 32MB (2의 거듭제곱), JVM이 자동 계산
- **리전 역할**: 각 리전은 Eden/Survivor/Old/Humongous/Free 중 하나로 동적 지정
- **리전 수**: 힙을 약 2048개 이하의 리전으로 분할하도록 크기 결정

### Humongous Region

객체 크기가 리전 크기의 절반을 초과하면 **Humongous Object**로 분류:
- 연속된 Humongous Region에 직접 할당
- Old Generation처럼 취급 (Young GC에서 수집 안됨)
- Java 8u60부터 Reclaimable Humongous Object를 Young GC 중에 수집 가능

## 3. G1 GC 동작 사이클

G1의 수집 과정은 여러 단계로 구성된다:

### Phase 1: Young GC (Evacuation Pause)

가장 빈번하게 발생하는 단계. STW 상태에서 병렬로 수행:

```
1. Eden 리전의 살아있는 객체 식별
2. 살아있는 객체를 Survivor 또는 Old 리전으로 복사(Evacuation)
3. 기존 Eden 리전을 Free로 반환
4. Survivor 리전 중 일부를 Old로 승격(Tenuring)
```

특징:
- **복사 방식 수집**: 객체를 이동시키므로 단편화 없음
- **일시 정지 예측**: G1은 과거 데이터를 기반으로 충분한 리전을 수집해 `MaxGCPauseMillis` 목표 달성

### Phase 2: Concurrent Marking

힙이 일정 수준 차오르면(`InitiatingHeapOccupancyPercent`, 기본 45%) 동시 마킹 시작:

```
Step 1: Initial Mark (STW, Young GC에 편승)
  - 루트(Root)에서 직접 참조되는 객체 마킹
  - 짧은 STW (Young GC와 함께 수행)

Step 2: Root Region Scanning (Concurrent)
  - Survivor 리전에서 Old Gen 참조를 스캔 (Young GC 이전 완료 필요)

Step 3: Concurrent Marking (Concurrent)
  - 전체 힙을 순회하며 살아있는 객체 마킹
  - SATB(Snapshot-At-The-Beginning) 알고리즘으로 변경사항 추적

Step 4: Remark (STW)
  - SATB 큐를 처리하여 동시 마킹 중 변경된 참조 처리
  - Reference Processing (WeakReference, SoftReference 등)

Step 5: Cleanup (STW + Concurrent)
  - 완전히 비어있는 리전 즉시 회수 (STW, 매우 빠름)
  - 각 리전의 live data 비율 계산
  - 다음 Mixed GC를 위한 수집 후보 리전 선정
```

### Phase 3: Mixed GC (Mixed Evacuation Pause)

동시 마킹 완료 후 여러 번의 Mixed GC 수행:
- Young 리전 + **선택된 Old 리전**을 함께 수집
- G1은 가장 "수익성 높은(Garbage First)" Old 리전부터 선택 → 이름의 유래
- `G1MixedGCCountTarget` (기본 8)번의 Mixed GC 후 일반 Young GC로 복귀

### Phase 4: Full GC (최후 수단)

G1이 할당 속도를 따라잡지 못하면 Serial Full GC 발생 (Java 10+에서 Parallel Full GC):
- 전체 힙 압축
- 매우 긴 STW
- 이 상태는 비정상 → 튜닝 필요

## 4. 핵심 내부 구조

### Remembered Set (RSet)

리전 간 참조를 추적하는 데이터 구조:

```
┌──────────────────────────────────────────────────────┐
│ Region A (Old)                                        │
│  Object X ──────────────────────────────────────┐    │
│  RSet: {(Region B, card 5), (Region C, card 2)}  │    │
└──────────────────────────────────────────────────│────┘
                                                   │
┌──────────────────────┐    ┌──────────────────────▼──┐
│ Region B (Eden)       │    │ Region C (Old)          │
│  ...card 5: obj ref──┘    │  ...card 2: obj ref     │
└───────────────────────┘    └─────────────────────────┘
```

각 리전은 **자신을 가리키는 다른 리전의 참조 정보**를 RSet에 저장한다. Young GC 시 전체 힙을 스캔하지 않고 RSet만 확인해도 루트를 찾을 수 있다.

**Card Table**: 힙을 512바이트 단위(Card)로 분할. 객체 참조가 변경되면 해당 Card를 "dirty"로 마킹. Dirty Card를 스캔해 RSet 업데이트.

### SATB (Snapshot-At-The-Beginning)

동시 마킹 중 변경되는 참조를 처리하는 쓰기 배리어(Write Barrier):

```java
// 의사 코드: SATB 쓰기 배리어
void write_barrier(Object* field, Object* new_value) {
    Object* old_value = *field;
    if (old_value != null && is_marking()) {
        // 이전 참조를 SATB 큐에 기록 (마킹 시작 시점의 스냅샷 보존)
        satb_queue.enqueue(old_value);
    }
    *field = new_value;
}
```

"마킹 시작 시점의 스냅샷(Snapshot)"을 기준으로 살아있는 객체를 결정한다. 마킹 중 참조가 삭제되어도 SATB 큐를 통해 원래 참조를 추적한다.

### Pause Prediction Model

G1은 각 리전의 수집 시간과 확보 공간을 예측하는 **적응형 모델**을 사용한다:

```
예측 데이터 (Decaying Average):
- 각 리전의 live data 크기
- 과거 GC 단계별 소요 시간 (Young 수집, RSet 업데이트 등)
- 리전당 객체 복사 속도

수집 계획:
MaxGCPauseMillis 목표 내에서 최대한 많은 리전 수집
→ 일시 정지 예측 = Σ(선택 리전별 예측 시간)
→ 합계가 목표 초과 직전까지 리전 선택
```

## 5. G1 GC 모니터링과 로그 분석

### GC 로그 활성화

```bash
# Java 11+
java -Xms4g -Xmx4g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -Xlog:gc*:file=gc.log:time,uptime,level,tags \
     -jar app.jar
```

### G1 GC 로그 해석

```
[0.532s][info][gc] GC(0) Pause Young (Normal) (G1 Evacuation Pause) 512M->128M(4096M) 15.234ms
[0.532s][info][gc,phases] GC(0)   Pre Evacuate Collection Set: 0.1ms
[0.532s][info][gc,phases] GC(0)   Merge Heap Roots: 0.8ms
[0.532s][info][gc,phases] GC(0)   Evacuate Collection Set: 12.3ms
[0.532s][info][gc,phases] GC(0)   Post Evacuate Collection Set: 1.8ms
[0.532s][info][gc,phases] GC(0)   Other: 0.3ms

해석:
- Pause Young: Young GC (Evacuation)
- 512M→128M(4096M): GC 전→후 사용 메모리 (최대 힙)
- 15.234ms: 총 일시 정지 시간
- Evacuate: 객체 복사 단계 (가장 많은 시간)

[3.421s][info][gc] GC(12) Pause Young (Concurrent Start) 2048M->1024M(4096M) 18.5ms
→ 동시 마킹 시작 (힙 점유율이 IHOP 임계값 도달)

[3.440s][info][gc] GC(12) Concurrent Cycle
[3.440s][info][gc,marking] GC(12) Concurrent Clear Claimed Marks 0.1ms
[3.440s][info][gc,marking] GC(12) Concurrent Scan Root Regions 2.3ms
[3.443s][info][gc,marking] GC(12) Concurrent Mark 45.2ms
[3.488s][info][gc,marking] GC(12) Concurrent Mark Abort       ← 중단 (Young GC 필요)
```

## 6. JVM 코드로 이해하는 G1: Java API 활용

```java
import java.lang.management.*;
import java.util.List;

public class G1GCMonitor {

    public static void analyzeGCActivity() {
        List<GarbageCollectorMXBean> gcBeans =
            ManagementFactory.getGarbageCollectorMXBeans();

        for (GarbageCollectorMXBean gc : gcBeans) {
            System.out.printf("GC Name: %s%n", gc.getName());
            System.out.printf("  Collection Count: %d%n", gc.getCollectionCount());
            System.out.printf("  Collection Time:  %d ms%n", gc.getCollectionTime());
            System.out.printf("  Memory Pools: %s%n",
                String.join(", ", gc.getMemoryPoolNames()));
            System.out.println();
        }
    }

    // G1 GC 로그에서 GC 알림 수신
    public static void registerGCNotification() {
        List<GarbageCollectorMXBean> gcBeans =
            ManagementFactory.getGarbageCollectorMXBeans();

        for (GarbageCollectorMXBean gcBean :
                gcBeans.stream()
                       .filter(b -> b instanceof NotificationEmitter)
                       .toList()) {
            NotificationEmitter emitter = (NotificationEmitter) gcBean;
            emitter.addNotificationListener((notification, handback) -> {
                if (notification.getType()
                        .equals(GarbageCollectionNotificationInfo.GARBAGE_COLLECTION_NOTIFICATION)) {
                    GarbageCollectionNotificationInfo info =
                        GarbageCollectionNotificationInfo.from(
                            (CompositeData) notification.getUserData());

                    GcInfo gcInfo = info.getGcInfo();
                    long pauseMs = gcInfo.getDuration();
                    String gcAction = info.getGcAction();
                    String gcCause = info.getGcCause();

                    System.out.printf("[GC Event] action=%s, cause=%s, pause=%dms%n",
                        gcAction, gcCause, pauseMs);

                    // 힙 사용량 출력
                    gcInfo.getMemoryUsageAfterGc().forEach((pool, usage) -> {
                        if (usage.getMax() > 0) {
                            double pct = 100.0 * usage.getUsed() / usage.getMax();
                            System.out.printf("  %s: %.1f%% used (%dMB / %dMB)%n",
                                pool, pct,
                                usage.getUsed() >> 20,
                                usage.getMax() >> 20);
                        }
                    });
                }
            }, null, null);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        analyzeGCActivity();
        registerGCNotification();

        // GC 유발 테스트
        System.out.println("Creating garbage to trigger GC...");
        for (int i = 0; i < 1000; i++) {
            byte[] arr = new byte[1024 * 1024]; // 1MB 할당 후 즉시 해제
            Thread.sleep(5);
        }
    }
}
```

### G1 힙 구조 직접 관찰

```java
import java.lang.management.*;

public class HeapInspector {

    public static void printG1HeapDetails() {
        MemoryMXBean memBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heapUsage = memBean.getHeapMemoryUsage();

        System.out.printf("Heap: init=%dMB, used=%dMB, committed=%dMB, max=%dMB%n",
            heapUsage.getInit() >> 20,
            heapUsage.getUsed() >> 20,
            heapUsage.getCommitted() >> 20,
            heapUsage.getMax() >> 20);

        System.out.println("\nMemory Pool Details:");
        for (MemoryPoolMXBean pool : ManagementFactory.getMemoryPoolMXBeans()) {
            MemoryUsage usage = pool.getUsage();
            if (usage == null) continue;

            System.out.printf("  [%s] %s: used=%dMB / max=%s%n",
                pool.getType(),
                pool.getName(),
                usage.getUsed() >> 20,
                usage.getMax() < 0 ? "N/A" : (usage.getMax() >> 20) + "MB");
        }
    }

    // G1 리전 크기 계산 (JVM 내부 로직 시뮬레이션)
    public static int estimateG1RegionSize(long heapMaxBytes) {
        // G1은 약 2048개 리전이 되도록 크기 결정
        long targetRegions = 2048;
        long regionSize = heapMaxBytes / targetRegions;

        // 2의 거듭제곱으로 올림, 1MB ~ 32MB 범위 내
        int sizeMB = 1;
        while (sizeMB * 1024L * 1024L < regionSize && sizeMB < 32) {
            sizeMB <<= 1;
        }
        return sizeMB;
    }

    public static void main(String[] args) {
        printG1HeapDetails();

        long maxHeap = Runtime.getRuntime().maxMemory();
        System.out.printf("%nEstimated G1 Region Size: %dMB (for %dGB heap)%n",
            estimateG1RegionSize(maxHeap), maxHeap >> 30);
    }
}
```

## 7. G1 GC 핵심 JVM 옵션과 튜닝 가이드

### 필수 옵션

```bash
# 기본 설정 (Java 9+ 기본값이 이미 G1)
-XX:+UseG1GC

# 일시 정지 목표 (기본 200ms, 낮출수록 처리량 감소)
-XX:MaxGCPauseMillis=100

# 힙 크기 (G1은 Xms=Xmx 권장)
-Xms8g -Xmx8g

# 동시 마킹 시작 힙 점유율 임계값 (기본 45%)
-XX:InitiatingHeapOccupancyPercent=35

# 병렬 GC 스레드 수 (기본 CPU 수에서 자동 설정)
-XX:ParallelGCThreads=8
-XX:ConcGCThreads=4
```

### 문제별 튜닝 방법

**문제 1: Young GC 일시 정지가 너무 길다**
```bash
# 원인: Eden 리전이 너무 많음
# 해결: Young 리전 수 제한
-XX:G1NewSizePercent=5      # Young Gen 최소 비율 (기본 5%)
-XX:G1MaxNewSizePercent=20  # Young Gen 최대 비율 (기본 60%)
```

**문제 2: Full GC 발생**
```bash
# 원인 1: 할당 속도가 수집 속도 초과
# → 힙 증가 또는 -XX:InitiatingHeapOccupancyPercent 감소

# 원인 2: Humongous Object 과다
# → 객체 크기 줄이거나 리전 크기 증가
-XX:G1HeapRegionSize=16m

# 원인 3: Concurrent Mode Failure
# → 동시 GC 스레드 증가
-XX:ConcGCThreads=8
```

**문제 3: To-space Exhausted (Evacuation Failure)**
```bash
# 원인: Survivor/Old Gen 공간 부족으로 객체 복사 실패 → 즉시 Full GC
# 해결: 힙 여유 공간 증가
-XX:G1ReservePercent=20  # 예비 공간 비율 (기본 10%)
```

### G1 vs ZGC vs Shenandoah 선택 기준

| 요소 | G1 GC | ZGC | Shenandoah |
|------|-------|-----|------------|
| 최소 Java 버전 | Java 7 | Java 11 | Java 12 |
| 최대 힙 | ~수십 GB | 수 TB | 수십 GB |
| GC 일시 정지 | 수십~수백ms | < 1ms | < 10ms |
| 처리량 | 높음 | 중간 | 중간 |
| 권장 환경 | 범용, 레거시 호환 | 초저지연, 대용량 힙 | 저지연, 중간 힙 |

## 8. 주의사항과 실전 팁

### 메모리 설정 주의
- G1은 Xms=Xmx로 설정해야 OS에 메모리를 반납/재할당하는 오버헤드를 피할 수 있다
- 힙 크기가 너무 작으면 (4GB 미만) ZGC 대비 이점이 적을 수 있음

### RSet 업데이트 오버헤드
- Old Gen 리전이 매우 많아지면 RSet 업데이트 비용 증가
- `G1SummarizeRSetStats` 플래그로 RSet 통계 출력 가능

### Large Page 사용 (성능 최적화)
```bash
-XX:+UseLargePages  # TLB 미스 감소, 성능 5~15% 향상
```

### GC 로그 필수 수집
프로덕션 환경에서 GC 로그는 후처리 비용이 매우 낮으므로 항상 활성화:
```bash
-Xlog:gc*:file=/var/log/gc-%t.log:time,uptime,level,tags:filecount=10,filesize=20m
```

GCViewer, GCEasy 등의 도구로 GC 로그를 시각화해 병목 분석이 가능하다.

## 참고 자료
- [Oracle Java SE 21 G1 GC Tuning Guide](https://docs.oracle.com/en/java/javase/21/gctuning/garbage-first-g1-garbage-collector1.html)
- [JEP 248: Make G1 the Default Garbage Collector](https://openjdk.org/jeps/248)
- [Getting Started with the G1 Garbage Collector (Oracle)](https://www.oracle.com/technetwork/tutorials/tutorials-1876574.html)
- [HotSpot Virtual Machine Garbage Collection Tuning Guide](https://docs.oracle.com/en/java/javase/21/gctuning/introduction-garbage-collection-tuning.html)
