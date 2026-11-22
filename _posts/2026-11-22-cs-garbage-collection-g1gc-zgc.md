---
layout: post
title: "가비지 컬렉션 알고리즘 완전 정복: Mark-Sweep부터 G1GC·ZGC까지 JVM GC의 진화"
date: 2026-11-22
categories: [cs, computer-science]
tags: [garbage-collection, jvm, g1gc, zgc, mark-sweep, java, memory-management]
---

## 개요

현대 프로그래밍 언어는 대부분 자동 메모리 관리(Automatic Memory Management)를 제공합니다. Java, Go, Python, C# 등의 런타임은 **가비지 컬렉터(Garbage Collector, GC)** 를 통해 더 이상 사용되지 않는 객체를 자동으로 회수합니다. 하지만 GC는 "무료 점심(free lunch)"이 아닙니다. 잘못 이해하고 사용하면 예측 불가능한 **Stop-The-World(STW)** 멈춤이 발생하고, 대용량 힙에서 수 초의 지연이 생길 수 있습니다.

이 글에서는 가비지 컬렉션의 기본 알고리즘부터, JVM의 최신 GC인 **G1GC**와 **ZGC**까지 내부 동작을 깊이 살펴봅니다.

---

## 가비지란 무엇인가?

가비지(garbage)는 더 이상 도달할 수 없는(unreachable) 객체입니다. **GC 루트(GC Root)** 에서 시작하여 객체 참조 그래프를 따라가, 도달할 수 없는 객체가 가비지입니다.

GC 루트의 예:
- 현재 실행 중인 모든 스레드의 스택 변수
- 정적(static) 변수
- JNI 참조
- 클래스 로더

GC의 두 가지 기본 접근법:
- **참조 계산(Reference Counting)**: 각 객체의 참조 수를 추적. 순환 참조를 처리하지 못함 (Python의 기본 방식, 보조적으로 Cycle Detector 사용)
- **추적(Tracing) GC**: GC 루트에서 그래프를 순회하여 도달 가능한 객체를 표시. JVM, Go, .NET이 사용하는 방식

---

## 고전적 GC 알고리즘

### Mark-Sweep

가장 단순한 추적 GC입니다.

1. **Mark 단계**: GC 루트에서 출발하여 모든 도달 가능한 객체에 비트를 설정합니다.
2. **Sweep 단계**: 힙 전체를 순회하여 표시되지 않은 객체를 해제합니다.

**단점**:
- Mark와 Sweep 동안 애플리케이션이 완전히 멈춥니다 (STW).
- **단편화(fragmentation)**: 메모리가 듬성듬성 비어 큰 객체 할당에 실패할 수 있습니다.

### Mark-Compact

Mark 이후 살아있는 객체를 힙의 한쪽으로 압축합니다. 단편화가 없지만 압축 비용이 큽니다.

### Copying GC (반 공간 복사)

힙을 두 반쪽(From-space, To-space)으로 나눕니다. Mark 후 살아있는 객체를 To-space로 복사하고 From-space를 통째로 비웁니다. 단편화 없고 빠르지만 메모리의 절반만 사용합니다.

### Generational GC (세대별 GC)

경험적 관찰 — **대부분의 객체는 금방 죽는다(Weak Generational Hypothesis)** — 에 기반합니다. 힙을 세대(Generation)로 나누어, 대부분 죽는 Young 영역을 자주, 오래 사는 Old 영역을 가끔 수집합니다.

JVM의 경우:
- **Young 영역 (Eden + Survivor S0, S1)**: Minor GC로 자주 수집, Copying 알고리즘 사용
- **Old 영역 (Tenured)**: Major/Full GC, 더 큰 비용
- **Metaspace**: 클래스 메타데이터 저장 (Java 8+ — PermGen 대체)

---

## 구현 예제 1: Python으로 Mark-Sweep 시뮬레이션

```python
from typing import Optional

class GCObject:
    _id_counter = 0

    def __init__(self, name: str):
        GCObject._id_counter += 1
        self.id = GCObject._id_counter
        self.name = name
        self.references: list['GCObject'] = []
        self.marked = False

    def add_ref(self, obj: 'GCObject'):
        self.references.append(obj)

    def __repr__(self):
        return f"Obj({self.id}:{self.name})"


class SimpleMarkSweepGC:
    def __init__(self):
        self.heap: list[GCObject] = []
        self.roots: list[GCObject] = []

    def allocate(self, name: str) -> GCObject:
        obj = GCObject(name)
        self.heap.append(obj)
        return obj

    def add_root(self, obj: GCObject):
        self.roots.append(obj)

    def remove_root(self, obj: GCObject):
        self.roots.remove(obj)

    def _mark(self, obj: GCObject):
        if obj.marked:
            return
        obj.marked = True
        for ref in obj.references:
            self._mark(ref)

    def collect(self) -> int:
        # Mark 단계
        for obj in self.heap:
            obj.marked = False
        for root in self.roots:
            self._mark(root)

        # Sweep 단계
        before = len(self.heap)
        dead = [obj for obj in self.heap if not obj.marked]
        self.heap = [obj for obj in self.heap if obj.marked]
        collected = before - len(self.heap)

        print(f"GC: {before} → {len(self.heap)} objects, collected {collected}")
        for obj in dead:
            print(f"  Collected: {obj}")
        return collected


# 시뮬레이션
gc = SimpleMarkSweepGC()

# 객체 생성
root_obj = gc.allocate("root")
child1 = gc.allocate("child1")
child2 = gc.allocate("child2")
orphan1 = gc.allocate("orphan1")  # 도달 불가 — 가비지
orphan2 = gc.allocate("orphan2")  # 도달 불가 — 가비지

# 순환 참조 (가비지)
cycle_a = gc.allocate("cycle_a")
cycle_b = gc.allocate("cycle_b")
cycle_a.add_ref(cycle_b)
cycle_b.add_ref(cycle_a)

# 루트에서 도달 가능한 참조
root_obj.add_ref(child1)
root_obj.add_ref(child2)

gc.add_root(root_obj)

print(f"힙 크기 (GC 전): {len(gc.heap)}")
gc.collect()
print(f"힙 크기 (GC 후): {len(gc.heap)}")
```

**출력:**
```
힙 크기 (GC 전): 7
GC: 7 → 3 objects, collected 4
  Collected: Obj(4:orphan1)
  Collected: Obj(5:orphan2)
  Collected: Obj(6:cycle_a)
  Collected: Obj(7:cycle_b)
힙 크기 (GC 후): 3
```

순환 참조(cycle_a ↔ cycle_b)도 추적 GC는 완전히 수집함을 확인할 수 있습니다.

---

## G1GC (Garbage First Garbage Collector)

### 왜 G1GC인가?

Java 9 이후 기본 GC로 채택된 G1GC는 고처리량(throughput)과 예측 가능한 짧은 pause를 동시에 목표로 합니다. 대용량 멀티프로세서 머신(힙 6GB 이상)을 위해 설계되었습니다.

### 핵심 아이디어: 영역(Region)

G1은 힙을 고정된 Young/Old 세대 대신 **동일 크기의 영역(Region)** (기본 1~32MB)으로 나눕니다. 각 영역은 동적으로 Eden, Survivor, Old, Humongous(대형 객체) 역할을 맡습니다.

**Garbage First의 의미**: 가장 많은 가비지를 포함한 영역부터 수집하여 같은 시간에 최대한 많은 공간을 회수합니다.

### G1GC의 수집 단계

1. **Young GC (Minor GC)**: Eden 영역을 완전히 수집, STW 발생. Survivor와 Old로 객체 이동.
2. **Concurrent Marking Cycle**: 애플리케이션과 동시에 Old 영역의 살아있는 객체를 표시.
   - Initial Mark (STW): GC 루트 직접 도달 Old 객체 표시
   - Concurrent Mark: 그래프 전체 순회 (애플리케이션 동시 실행)
   - Remark (STW): SATB(Snapshot-At-The-Beginning) 버퍼 처리
   - Cleanup (STW+Concurrent): 완전히 비어 있는 영역 회수
3. **Mixed GC**: Young GC + 선택된 Old 영역 함께 수집
4. **Full GC**: 힙이 가득 찰 때 단일 스레드 Mark-Compact (최후의 수단)

### G1GC 튜닝 예제

```bash
# G1GC 사용 (Java 9+에서는 기본값)
java -XX:+UseG1GC \
     -Xms4g -Xmx8g \
     -XX:MaxGCPauseMillis=200 \       # 목표 pause 시간 (ms) — G1이 영역 수 조절
     -XX:G1HeapRegionSize=16m \       # 영역 크기 (1~32MB, 2의 배수)
     -XX:G1NewSizePercent=20 \        # Young 영역 최소 비율
     -XX:G1MaxNewSizePercent=40 \     # Young 영역 최대 비율
     -XX:G1MixedGCLiveThresholdPercent=85 \  # Mixed GC 대상 영역의 live 비율 임계값
     -XX:G1HeapWastePercent=5 \       # Mixed GC 중단 임계 (힙의 5% 이하 가비지 시 중단)
     -XX:InitiatingHeapOccupancyPercent=45 \  # Concurrent Marking 시작 힙 점유율
     -XX:+PrintGCDetails -XX:+PrintGCDateStamps \
     -Xloggc:/var/log/app/gc.log \
     -jar myapp.jar
```

**G1GC 로그 분석 (JDK 11+ 통합 로깅):**
```bash
java -Xlog:gc*:file=gc.log:time,uptime,level,tags \
     -XX:+UseG1GC MyApp
```

로그에서 주의할 항목:
- `[GC pause (G1 Evacuation Pause) (young)]` — Young GC pause 시간
- `[GC pause (G1 Mixed)...]` — Mixed GC pause 시간
- `[Full GC (Allocation Failure)]` — Full GC 발생 시 조사 필요

---

## ZGC (Z Garbage Collector)

### ZGC의 목표

JEP 333(Java 11 실험적)에서 시작, Java 15에서 프로덕션 준비 완료. Java 21에서 Generational ZGC로 발전.

**목표**: 힙 크기와 무관하게 **1ms 미만의 STW pause** (최대 1ms는 ZGC의 설계 목표이며 JDK 버전에 따라 개선됨).

### 색상 포인터 (Colored Pointers)

ZGC의 가장 혁신적인 기법은 64비트 포인터의 상위 비트를 GC 메타데이터로 사용하는 것입니다:

```
63     47    43    41    40    0
+-------+-----+-----+-----+---+
| 미사용 | Fin | Rem | Mrk | 주소 |
+-------+-----+-----+-----+---+

Fin(Finalizable): 종료자가 있는 객체
Rem(Remapped): 재배치 완료
Mrk(Marked 0/1): 표시 플래그 (0과 1을 번갈아 사용)
```

이를 통해 **로드 배리어(Load Barrier)** 를 삽입하여 애플리케이션 스레드가 객체를 읽을 때 실시간으로 포인터를 치유(heal)합니다. STW 없이 대부분의 GC 작업이 동시에 진행됩니다.

### ZGC 수집 단계

| 단계 | STW 여부 | 설명 |
|---|---|---|
| Pause Mark Start | STW (< 1ms) | GC 루트에서 Mark 시작 |
| Concurrent Mark | 동시 | 전체 그래프 순회 |
| Pause Mark End | STW (< 1ms) | Mark 완료 |
| Concurrent Relocate Prep | 동시 | 재배치 집합 선택 |
| Pause Relocate Start | STW (< 1ms) | GC 루트 재배치 시작 |
| Concurrent Relocate | 동시 | 객체 이동 (로드 배리어가 포인터 치유) |

**STW가 항상 1ms 미만**입니다. 힙이 1TB여도 STW는 1ms 이내입니다.

### ZGC 튜닝 예제

```bash
# Java 21+: Generational ZGC 권장
java -XX:+UseZGC \
     -XX:+ZGenerational \            # Generational ZGC 활성화 (Java 21+)
     -Xms2g -Xmx16g \               # 초기/최대 힙 크기 (ZGC는 Xmx 설정이 가장 중요)
     -XX:SoftMaxHeapSize=12g \       # 소프트 최대 힙 (가능하면 이 이하 유지)
     -XX:ZUncommitDelay=300 \        # 미사용 메모리 반환 지연 (초)
     -XX:ConcGCThreads=4 \           # 동시 GC 스레드 수
     -XX:ZCollectionInterval=0 \     # 주기적 GC 간격 (0=비활성)
     -Xlog:gc*:file=zgc.log:time \
     -jar myapp.jar

# ZGC 주요 메트릭 모니터링
# jstat -gcutil <pid> 1000
# jcmd <pid> VM.native_memory
```

**ZGC vs G1GC 비교 측정 (힙 16GB, throughput 테스트):**

```java
// ZGC pause 측정 예시 (실제 프로덕션 로그 분석)
// ZGC 로그에서 pause 시간 추출
import java.io.*;
import java.util.regex.*;

public class GCLogAnalyzer {
    public static void main(String[] args) throws Exception {
        // ZGC pause 패턴: "GC(N) Pause Mark Start N.NNms"
        Pattern pausePattern = Pattern.compile(
            "GC\\(\\d+\\) Pause \\w+ \\w+ (\\d+\\.\\d+)ms"
        );

        double totalPause = 0;
        int count = 0;
        double maxPause = 0;

        try (BufferedReader br = new BufferedReader(new FileReader("zgc.log"))) {
            String line;
            while ((line = br.readLine()) != null) {
                Matcher m = pausePattern.matcher(line);
                if (m.find()) {
                    double pause = Double.parseDouble(m.group(1));
                    totalPause += pause;
                    count++;
                    maxPause = Math.max(maxPause, pause);
                }
            }
        }

        System.out.printf("GC Pause 분석 결과:%n");
        System.out.printf("  총 STW 횟수: %d%n", count);
        System.out.printf("  평균 pause: %.3f ms%n", count > 0 ? totalPause / count : 0);
        System.out.printf("  최대 pause: %.3f ms%n", maxPause);
        System.out.printf("  총 pause 시간: %.3f ms%n", totalPause);
    }
}
```

---

## GC 선택 가이드

```
힙 크기
  │
  ├─ < 4GB
  │    └─ Serial GC (-XX:+UseSerialGC) : 단일 스레드, 최소 오버헤드
  │
  ├─ 4GB ~ 32GB
  │    ├─ G1GC (기본값)             : 균형 잡힌 처리량+지연
  │    └─ Parallel GC (-XX:+UseParallelGC) : 최대 처리량, STW 무관
  │
  └─ > 32GB 또는 지연 < 10ms 목표
       ├─ ZGC (-XX:+UseZGC)         : 1ms 미만 STW, 대용량 힙
       └─ Shenandoah GC             : ZGC 유사 (Red Hat 주도)
```

---

## 주의사항과 튜닝 팁

1. **Full GC를 피하라**: Full GC는 단일 스레드 Mark-Compact로 수 초가 걸릴 수 있습니다. `Allocation Failure`, `Ergonomics`, `Heap Dump Initiated`가 원인인 경우 힙 크기나 GC 파라미터를 조정하세요.

2. **큰 객체(Humongous Objects) 주의**: G1GC에서 Region 크기의 50% 이상인 객체는 Humongous 영역에 직접 할당되어 Young GC에 포함됩니다. 대형 배열 생성을 최소화하거나 Region 크기를 키우세요.

3. **ZGC는 Xmx가 가장 중요**: ZGC는 동시 재배치를 위해 여분의 힙 공간이 필요합니다. 실제 사용량의 1.5~2배 Xmx를 설정하세요.

4. **GC 로그는 필수**: 프로덕션에서 GC 로그를 반드시 활성화하세요. JDK 11+에서는 `-Xlog:gc*` 옵션으로 통합 로깅을 사용합니다.

5. **GC를 코드로 직접 호출하지 말 것**: `System.gc()`는 힌트일 뿐이며, 실제 GC 발생을 보장하지 않습니다. 오히려 불필요한 Full GC를 유발할 수 있습니다.

6. **약한 참조 활용**: 메모리 민감한 캐시는 `WeakHashMap`이나 `SoftReference`를 활용하면 GC가 필요 시 자동으로 회수합니다.

---

## 참고 자료

- [Oracle JDK 21 GC Tuning Guide - Available Collectors](https://docs.oracle.com/en/java/javase/21/gctuning/available-collectors.html)
- [Oracle JDK 21 - The Z Garbage Collector](https://docs.oracle.com/en/java/javase/21/gctuning/z-garbage-collector.html)
- [OpenJDK JEP 333: ZGC 제안서](https://openjdk.org/jeps/333)
- [Baeldung: JVM Garbage Collectors 가이드](https://www.baeldung.com/jvm-garbage-collectors)
