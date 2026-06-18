---
layout: post
title: "가비지 컬렉션 심화: Mark-Sweep부터 JVM G1GC까지 메모리 관리 완전 정복"
date: 2026-06-18
categories: [cs, computer-science]
tags: [garbage-collection, jvm, g1gc, mark-sweep, generational-gc, memory-management, java, python]
---

## 개념 설명

**가비지 컬렉션(GC)**은 프로그램이 더 이상 사용하지 않는 메모리를 자동으로 회수하는 메커니즘입니다. C/C++처럼 수동 메모리 관리를 요구하지 않아 개발 생산성을 크게 높이지만, GC의 동작 방식을 이해하지 못하면 "Stop-The-World" 지연, 메모리 누수, OOM(Out of Memory)과 같은 예기치 못한 문제로 이어집니다.

### GC의 핵심 알고리즘

**1. Reference Counting (참조 카운팅)**  
각 객체에 자신을 가리키는 참조 수를 저장합니다. 카운트가 0이 되면 즉시 해제합니다. Python, Swift(ARC)가 이 방식을 기본으로 사용합니다. 장점은 즉각적인 회수이지만, **순환 참조(Circular Reference)**를 처리하지 못한다는 치명적 단점이 있습니다.

```
A → B → C → A  (순환 참조: 누구도 카운트가 0이 되지 않음)
```

Python은 이를 보완하기 위해 별도의 Cycle Detector를 실행합니다.

**2. Mark-Sweep**  
GC Roots(스택, 전역 변수, JNI 등)에서 출발해 도달 가능한 모든 객체를 **마킹**하고, 마킹되지 않은 객체를 **스위프(해제)**합니다. 순환 참조 문제를 해결하지만, **단편화(Fragmentation)**가 발생하고 전체 힙을 순회하는 동안 Stop-The-World가 발생합니다.

**3. Mark-Compact (Mark-Sweep-Compact)**  
Mark-Sweep 이후 살아있는 객체를 힙의 한쪽으로 밀어 단편화를 제거합니다. 메모리 할당이 `free pointer`를 한 칸 앞으로 당기는 간단한 포인터 범프(Bump Pointer Allocation)로 가능해집니다.

**4. Copying GC**  
힙을 동일한 크기의 두 영역(From-space, To-space)으로 나누고, 살아있는 객체만 To-space로 복사합니다. 단편화 없고 할당이 빠르지만 힙의 절반만 실제로 사용할 수 있습니다.

### 세대별 GC (Generational GC)

실제 프로그램을 프로파일링하면 **"대부분의 객체는 젊어서 죽는다"**는 패턴이 관찰됩니다 (Weak Generational Hypothesis). 이를 활용해 힙을 **Young Generation**과 **Old Generation**으로 분리합니다:

- **Young Gen (Eden + Survivor 0/1)**: 새 객체가 할당되는 곳. Minor GC가 자주(수십ms 간격) 실행되며, 대부분 객체가 여기서 수거됨
- **Old Gen (Tenured)**: Minor GC를 여러 번 살아남은 객체가 승격(Promote)됨. Major GC(Full GC)는 드물지만 더 오래 걸림

---

## JVM G1GC (Garbage First GC)

G1GC는 Java 9부터 기본 GC로 채택된 현대적 수집기입니다. 기존 Parallel GC나 CMS(Concurrent Mark-Sweep)와 달리, 힙을 **동등한 크기의 Region(1~32MB)** 수백 개로 나눕니다.

### Region 기반 접근의 핵심 아이디어

각 Region은 동적으로 Eden, Survivor, Old, Humongous(큰 객체) 역할을 맡습니다. GC 시, G1은 **가장 가비지가 많은 Region부터 우선 수거**합니다("Garbage First"의 어원). 이를 통해 **예측 가능한 STW 목표 시간(-XX:MaxGCPauseMillis)**을 제어할 수 있습니다.

### G1GC 단계별 흐름

```
Young GC (Minor)
  → Eden Regions 소진 시 실행
  → Eden/Survivor → Survivor/Old로 복사
  → 빠름 (보통 수ms~수십ms)

Concurrent Marking (병렬)
  → Old Gen 사용률이 임계값(-XX:InitiatingHeapOccupancyPercent, 기본 45%) 초과 시
  → 1. Initial Mark (STW, 짧음)
  → 2. Root Region Scan (Concurrent)
  → 3. Concurrent Mark (Concurrent, 앱과 동시 실행)
  → 4. Remark (STW, SATB 알고리즘으로 변경 처리)
  → 5. Cleanup (STW + Concurrent)

Mixed GC
  → Young Region + 가비지 많은 Old Region을 함께 수거
  → 점진적으로 Old Gen 정리
```

---

## 실제 구현 예제

### 예제 1: Python에서 순환 참조와 GC 동작 확인

```python
import gc
import weakref

class Node:
    def __init__(self, name: str):
        self.name = name
        self.ref = None
    def __del__(self):
        print(f"  [GC] {self.name} 소멸")

# 순환 참조 생성
print("=== 순환 참조 생성 ===")
a = Node("A")
b = Node("B")
a.ref = b   # A → B
b.ref = a   # B → A (순환)

print("del 호출 — 참조 카운트 0이 되지 않으므로 즉시 소멸 안 됨")
del a
del b
print("del 후: 아직 소멸되지 않음 (순환 참조로 refcount > 0)")

print("\n=== gc.collect() 호출 ===")
collected = gc.collect()   # Cycle Detector 실행
print(f"  수거된 객체 수: {collected}")

# weakref로 순환 참조 방지
print("\n=== weakref로 해결 ===")
class SafeNode:
    def __init__(self, name: str):
        self.name = name
        self.ref = None
    def __del__(self):
        print(f"  [GC] {self.name} 소멸")

x = SafeNode("X")
y = SafeNode("Y")
x.ref = weakref.ref(y)   # 약한 참조 — 참조 카운트 증가 안 함
y.ref = weakref.ref(x)
del x
del y
print("weakref 사용 시 즉시 소멸됨")
```

### 예제 2: Java G1GC 튜닝 — JVM 플래그와 GC 로그 분석

```java
// G1GC 권장 JVM 플래그 (예시 주석 포함)
// java -Xms4g -Xmx4g \
//      -XX:+UseG1GC \
//      -XX:MaxGCPauseMillis=200 \         // 목표 STW 시간 (ms)
//      -XX:InitiatingHeapOccupancyPercent=45 \ // Concurrent Marking 시작 임계값
//      -XX:G1HeapRegionSize=16m \          // Region 크기 (1~32MB, 2의 거듭제곱)
//      -XX:G1ReservePercent=10 \           // 예약 힙 비율 (Promotion 실패 방지)
//      -Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=20m \
//      -jar myapp.jar

import java.util.ArrayList;
import java.util.List;

public class GCDemo {

    // Humongous 객체 (Region 크기의 50% 초과) — G1이 별도로 처리
    static void humongousDemo(int regionSizeMB) {
        int threshold = regionSizeMB * 1024 * 1024 / 2;
        // 이 크기를 넘으면 Humongous Region에 직접 할당됨
        byte[] bigObj = new byte[threshold + 1];
        System.out.println("Humongous 객체 할당: " + bigObj.length + " bytes");
        // bigObj를 null로 만들어도 다음 GC 때까지 Humongous Region 점유
    }

    // Young Gen 압박 시뮬레이션
    static void youngGenPressure() {
        List<byte[]> survivor = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            byte[] shortLived = new byte[1024 * 10]; // 10KB, 금방 사망
            if (i % 100 == 0) {
                survivor.add(new byte[1024 * 50]); // 50KB, Old로 승격될 후보
            }
        }
        // Minor GC가 여러 번 발생하며 survivor의 객체들은 Old Gen으로 이동
        System.out.println("Young Gen 압박 완료. Survivor 크기: " + survivor.size());
    }

    // GC 로그 파싱 예시 (G1GC 로그 포맷)
    static void parseGCLogLine(String line) {
        // 예: [2026-06-18T12:00:00.100+0900][0.100s][GC][info] GC(1) Pause Young
        //         (Normal) (G1 Evacuation Pause) 50M->20M(256M) 12.345ms
        if (line.contains("Pause Young")) {
            System.out.println("Minor GC 감지: " + line);
        } else if (line.contains("Pause Mixed")) {
            System.out.println("Mixed GC 감지 (Old Gen 청소 중): " + line);
        } else if (line.contains("Pause Full")) {
            System.err.println("⚠️  Full GC 발생! 원인 조사 필요: " + line);
        }
    }

    public static void main(String[] args) {
        youngGenPressure();
        humongousDemo(16); // -XX:G1HeapRegionSize=16m 가정
    }
}
```

---

## 주의사항 및 실무 팁

**Stop-The-World 최소화**: G1GC의 `-XX:MaxGCPauseMillis`는 목표이지, 보장이 아닙니다. Old Gen이 꽉 차면 Full GC(Serial Mark-Compact)가 발생해 수 초의 STW가 생길 수 있습니다. Old Gen 사용률을 70% 이하로 유지하는 것이 중요합니다.

**ZGC / Shenandoah**: Java 15+에서 프로덕션 준비가 된 ZGC는 대부분의 GC 작업을 Concurrent하게 처리해 **STW < 1ms**를 목표로 합니다. 단, 처리량(Throughput)은 G1보다 약간 낮을 수 있습니다. 수십 GB~TB 힙 환경에서 특히 유리합니다.

**메모리 누수는 GC가 막지 못함**: GC는 "도달 불가능한" 객체만 수거합니다. 컬렉션(List, Map)에 계속 추가되지만 제거되지 않는 객체, 리스너 해제 누락, ThreadLocal 남용은 GC 환경에서도 메모리 누수를 일으킵니다. **Heap Dump 분석**(Eclipse MAT, VisualVM)이 필수입니다.

**GC 로그 필수 설정**: 운영 환경에서 반드시 GC 로그를 활성화하세요. 문제가 터진 후 로그 없이 분석하는 것은 불가능에 가깝습니다.

```bash
# Java 11+ 통합 로깅 플래그
-Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level:filecount=10,filesize=50m
```

**Python GC 튜닝**: Python의 Cycle Detector는 3세대(generation 0, 1, 2)로 나뉘어 작동합니다. `gc.set_threshold(700, 10, 10)`으로 각 세대의 수집 임계값을 조정할 수 있습니다. 성능 민감 코드에서 `gc.disable()`로 Cycle Detector를 끄고, 명시적으로 `gc.collect()`를 호출하는 전략도 사용됩니다(CPython Discord 봇 등에서 사례 있음).

## 참고 자료
- [JVM Garbage Collectors - Baeldung](https://www.baeldung.com/jvm-garbage-collectors)
- [9 Garbage-First Garbage Collector - Oracle Docs](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc.html)
- [How to Choose the Best Java Garbage Collector - Red Hat Developer](https://developers.redhat.com/articles/2021/11/02/how-choose-best-java-garbage-collector)
- [Understanding the G1 Garbage Collector – Dynatrace Blog](https://www.dynatrace.com/news/blog/understanding-g1-garbage-collector-java-9/)
