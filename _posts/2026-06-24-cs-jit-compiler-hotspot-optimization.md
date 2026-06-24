---
layout: post
title: "JIT 컴파일러와 HotSpot 최적화: 바이트코드가 네이티브 코드가 되는 과정"
date: 2026-06-24
categories: [cs, computer-science]
tags: [jit, jvm, hotspot, compiler, optimization, inline-caching, deoptimization, java]
---

## JIT 컴파일러란 무엇인가

JIT(Just-In-Time) 컴파일러는 프로그램이 **실행되는 도중에** 바이트코드(또는 중간 표현)를 머신 코드로 변환하는 컴파일러다. 전통적인 AOT(Ahead-Of-Time) 컴파일러가 실행 전에 모든 코드를 컴파일하는 것과 달리, JIT는 실제로 실행되는 코드 경로만 선별적으로 컴파일한다.

JVM의 경우를 예로 들면, Java 소스 코드는 먼저 `javac`에 의해 플랫폼 독립적인 바이트코드(`.class` 파일)로 변환된다. 이 바이트코드는 처음에는 **인터프리터(Interpreter)** 가 한 줄씩 해석해 실행한다. 그러다 특정 메서드가 충분히 많이 호출되어 "뜨거워지면(hot)" JIT 컴파일러가 해당 메서드를 네이티브 코드로 컴파일해 이후 호출부터는 빠르게 실행하도록 만든다.

이 전략이 순수 인터프리터보다 훨씬 빠른 이유는 두 가지다:
1. **반복 실행 코드의 가속**: 루프 본체나 자주 호출되는 메서드는 컴파일된 머신 코드로 수십~수백 배 빠르게 실행된다.
2. **런타임 프로파일링 기반 최적화**: 실제 실행 데이터를 보고 최적화하므로 AOT보다 더 공격적인 최적화가 가능하다.

## 왜 JIT가 필요한가

### 정적 컴파일의 한계

C/C++ 컴파일러는 소스 코드만 보고 최적화를 결정해야 한다. 그래서 가상 함수 호출(virtual dispatch)을 인라인하기 어렵다 — 어떤 서브클래스의 메서드가 실제로 불릴지 컴파일 시점에는 알 수 없기 때문이다.

반면 JIT는 런타임에 "이 호출 사이트에서 지금까지 `ArrayList`만 왔고 `LinkedList`는 한 번도 안 왔다"는 사실을 안다. 이 정보를 바탕으로 **다형성 호출을 단형(monomorphic) 직접 호출로 변환**하고, 나아가 메서드 본체를 통째로 인라인할 수 있다.

### 적응형 최적화(Adaptive Optimization)

HotSpot JVM은 두 가지 JIT 컴파일러를 탑재한다:
- **C1 (Client Compiler)**: 빠르게 컴파일, 제한적 최적화 — 시작 지연을 줄이기 위해 사용.
- **C2 (Server Compiler)**: 컴파일에 시간이 걸리지만 극도로 공격적인 최적화 — 장기 실행 서버 애플리케이션에 적합.

실제 HotSpot은 **Tiered Compilation**으로 두 컴파일러를 단계적으로 사용한다:

| 계층 | 실행 방식 | 목적 |
|------|----------|------|
| Tier 0 | 인터프리터 | 첫 실행, 프로파일링 시작 |
| Tier 1–3 | C1 컴파일 (프로파일링 포함) | 빠른 초기 컴파일 |
| Tier 4 | C2 완전 최적화 | 핫 코드의 극한 최적화 |

## JIT 핵심 최적화 기법

### 1. 인라인 캐싱(Inline Caching)

메서드를 호출할 때마다 vtable을 통한 간접 조회가 발생한다. 인라인 캐싱은 이 비용을 줄이기 위해 **직전에 사용된 수신자 타입과 그 타입에 대응하는 메서드 주소를 콜 사이트 바로 옆에 기억**해두는 기법이다.

```
// 메서드 호출 전: vtable 간접 참조
call [receiver.klass.vtable[index]]

// 인라인 캐시 적용 후: 타입 체크 + 직접 점프
cmp [receiver.klass], LastSeenKlass
jne slow_path
call DirectMethodAddress   ; 캐시 히트 시 직접 점프
```

타입이 하나만 등장하는 **단형(Monomorphic)** 호출 사이트에서 최대 효과를 발휘한다. 두 가지 타입이 오가면 **이형(Bimorphic)** 으로 전환되고, 그 이상은 **다형(Megamorphic)** 이 되어 인라인 캐시의 이점이 사라지고 vtable lookup으로 폴백된다.

### 2. 탈출 분석(Escape Analysis)과 스택 할당

객체가 현재 메서드 또는 스레드 바깥으로 "탈출"하지 않는다면 힙 대신 **스택에 할당**할 수 있다. 스택 할당은 GC 부담을 없애고 캐시 지역성을 높인다.

```java
// 탈출 분석 예: Point 객체가 메서드 밖으로 나가지 않음
double distance(double x, double y) {
    Point p = new Point(x, y); // 힙 대신 스택에 할당 가능
    return Math.sqrt(p.x * p.x + p.y * p.y);
}
```

C2 컴파일러는 이 경우 `Point` 객체 생성 자체를 제거하고 `x`, `y`를 레지스터에 직접 저장하는 **스칼라 치환(Scalar Replacement)** 까지 수행한다.

### 3. 루프 언롤링(Loop Unrolling)과 벡터화(Vectorization)

```java
// 원본 루프
for (int i = 0; i < n; i++) {
    sum += arr[i];
}

// C2가 생성하는 언롤 + SIMD 벡터화 후 (개념적 표현)
// SSE/AVX 레지스터로 4~8개 원소를 한 번에 더함
for (int i = 0; i < n - 7; i += 8) {
    ymm0 = VADDPS(ymm0, load256(&arr[i]));
}
// 나머지 처리
for (int i = ...; i < n; i++) sum += arr[i];
```

HotSpot C2는 **Auto-Vectorization**을 통해 적절한 루프를 자동으로 SIMD 명령어로 변환한다.

## 코드 예제: JIT 최적화 확인

### 예제 1: JVM 플래그로 컴파일 로그 출력

```bash
# JIT가 어떤 메서드를 컴파일하는지 확인
java -XX:+PrintCompilation -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintInlining MyApp 2>&1 | head -50

# 출력 예시:
#    73    1     3     java.lang.String::hashCode (67 bytes)
#    74    2     4     java.util.HashMap::get (23 bytes)
#    75    3     4     com.example.MyApp::hotMethod (45 bytes)
#                       @ 12   com.example.Helper::compute (22 bytes)   inline (hot)
```

각 컬럼의 의미:
- **컴파일 번호**: JVM 시작 후 누적 컴파일 횟수
- **컴파일 계층(Tier)**: 1(C1) ~ 4(C2)
- **메서드 서명과 바이트코드 크기**
- `inline (hot)`: 인라인된 메서드

### 예제 2: 인라인 캐싱 효과를 체감하는 벤치마크 (Java)

```java
import org.openjdk.jmh.annotations.*;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)
public class InlineCacheBenchmark {

    interface Processor {
        int process(int x);
    }

    static class FastProcessor implements Processor {
        public int process(int x) { return x * 2; }
    }

    static class SlowProcessor implements Processor {
        public int process(int x) { return x + 1; }
    }

    private Processor mono = new FastProcessor();
    // 단형 호출: JIT가 직접 호출로 최적화
    @Benchmark
    public int monomorphic() {
        int sum = 0;
        for (int i = 0; i < 1000; i++) sum += mono.process(i);
        return sum;
    }

    private Processor[] mixed = {new FastProcessor(), new SlowProcessor()};
    // 다형 호출: 인라인 캐시 미스 발생
    @Benchmark
    public int megamorphic() {
        int sum = 0;
        for (int i = 0; i < 1000; i++) sum += mixed[i % 2].process(i);
        return sum;
    }
}
// 결과: monomorphic이 megamorphic보다 보통 2~5배 빠름
```

## 역최적화(Deoptimization)

JIT의 낙관적 가정이 깨질 때 발생하는 과정이 **역최적화**다. 예를 들어, 단형 인라인 캐시가 걸린 상태에서 처음 보는 서브클래스 인스턴스가 등장하면:

1. JIT가 컴파일한 네이티브 코드에 **트랩(trap)** 이 걸린다.
2. 현재 스택 프레임을 인터프리터가 이해할 수 있는 상태로 **역조립(decompile)** 한다.
3. 인터프리터로 전환해 실행을 계속한다.
4. 새 프로파일 데이터를 바탕으로 새 가정 하에 **재컴파일**한다.

역최적화는 비싼 연산이지만 드물게 발생하고, 이후 더 나은 코드로 재컴파일되므로 전체 성능에 미치는 영향은 작다.

```
// JVM 로그에서 역최적화 확인
# -XX:+TraceDeoptimization 옵션 사용
Uncommon trap in method: com/example/MyApp.processItem(Ljava/lang/Object;)V
  at bci 23
  reason: class_check  (새로운 타입 등장)
  action: reinterpret
```

## 주의사항과 실전 팁

### 워밍업(Warm-up) 필수

마이크로벤치마크를 작성할 때 JIT 워밍업 없이 측정하면 인터프리터 성능만 측정된다. JMH(Java Microbenchmark Harness)가 표준적으로 사용되는 이유다.

```java
// 잘못된 벤치마크 — JIT가 아직 충분히 최적화하지 못한 상태
long start = System.nanoTime();
result = hotMethod();          // 첫 호출, 아직 인터프리터 수준
long elapsed = System.nanoTime() - start;
```

### 메서드 크기 제어 (-XX:MaxInlineSize)

너무 큰 메서드는 C2가 인라인하지 않는다. 기본 임계값은 35 바이트코드다. 성능이 중요한 핫 경로의 메서드는 작게 유지하는 것이 유리하다.

### GraalVM Native Image와의 차이

GraalVM의 Native Image는 AOT 방식으로 모든 것을 미리 컴파일한다. 시작 시간이 극히 짧지만 런타임 프로파일링 기반 최적화가 없어 장시간 실행되는 서버 워크로드에서는 HotSpot JIT에 뒤질 수 있다.

| 항목 | HotSpot JIT | GraalVM Native Image |
|------|------------|---------------------|
| 시작 시간 | 느림 (워밍업 필요) | 매우 빠름 |
| 최고 처리량 | 높음 (런타임 최적화) | 낮을 수 있음 |
| 메모리 사용 | 더 큼 | 더 작음 |
| 대상 | 장기 실행 서버 | 서버리스, CLI |

## 마치며

JIT 컴파일러는 "실행 중에 배우는" 컴파일러다. 정적 컴파일러가 결코 알 수 없는 런타임 정보를 활용해 인라인 캐싱, 탈출 분석, SIMD 벡터화 같은 강력한 최적화를 수행한다. JVM 기반 언어(Java, Kotlin, Scala)를 사용하는 개발자라면 JIT의 동작 원리를 이해함으로써 — 메서드를 작게 유지하고, 불필요한 다형성을 피하고, 벤치마크 시 워밍업을 포함하는 것만으로도 — 상당한 성능 이득을 얻을 수 있다.

## 참고 자료
- [OpenJDK HotSpot Wiki — Compiler](https://wiki.openjdk.org/display/HotSpot/Compiler)
- [JEP 295: Ahead-of-Time Compilation (OpenJDK)](https://openjdk.org/jeps/295)
- [Wikipedia: Just-in-time compilation](https://en.wikipedia.org/wiki/Just-in-time_compilation)
- [Wikipedia: Inline caching](https://en.wikipedia.org/wiki/Inline_caching)
