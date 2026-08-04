---
layout: post
title: "JVM 클래스 로딩과 동적 바이트코드 조작 완전 정복: ClassLoader 계층 구조부터 Java Agent까지"
date: 2026-08-04
categories: [cs, computer-science]
tags: [jvm, java, classloader, bytecode, asm, java-agent, instrumentation, aop]
---

JVM(Java Virtual Machine)에서 Java 클래스는 컴파일 시점에 완전히 결정되는 것이 아니라, 런타임에 **필요할 때** 동적으로 로드된다. 이 메커니즘은 JVM의 가장 강력한 기능 중 하나다. 플러그인 시스템, AOP(Aspect-Oriented Programming), APM 모니터링 에이전트, 핫 리로딩, Hibernate의 프록시 생성, Spring의 의존성 주입 — 이 모든 것이 클래스 로딩 메커니즘과 바이트코드 조작 위에 구현된다.

## ClassLoader 계층 구조

JVM은 세 가지 기본 ClassLoader를 계층적으로 구성한다 (Java 8 기준).

### 1. Bootstrap ClassLoader (부트스트랩 클래스 로더)

JVM 자체에 내장된 C/C++ 코드로 구현되어 있다. `java.lang.Object`, `java.lang.String`, `java.util.ArrayList` 등 `java.base` 모듈의 핵심 클래스들을 로드한다. Java 코드에서 `String.class.getClassLoader()`를 호출하면 `null`이 반환되는데, 이것이 부트스트랩 로더를 나타낸다.

```java
System.out.println(String.class.getClassLoader());
// 출력: null (부트스트랩 로더는 Java로 표현 불가)

System.out.println(ArrayList.class.getClassLoader());
// 출력: null
```

### 2. Extension/Platform ClassLoader

Java 8에서는 `$JAVA_HOME/lib/ext` 디렉토리의 JAR를 로드하는 Extension ClassLoader가 있었다. **Java 9+ 모듈 시스템**에서는 `jdk.internal.loader.ClassLoaders$PlatformClassLoader`로 대체되어, 플랫폼 모듈(`java.se`, `java.xml`, `java.sql` 등)을 담당한다.

### 3. Application (System) ClassLoader

`CLASSPATH` 환경변수나 `-classpath` 옵션에 지정된 경로에서 클래스를 로드한다. 우리가 직접 작성한 코드와 서드파티 라이브러리가 여기서 로드된다.

```java
System.out.println(MyClass.class.getClassLoader());
// 출력: jdk.internal.loader.ClassLoaders$AppClassLoader@...
```

### Delegation Model (부모 위임 모델)

ClassLoader의 핵심 원리는 **부모 위임(Parent Delegation)**이다. 클래스 로드 요청이 들어오면 먼저 부모 ClassLoader에게 위임하고, 부모가 찾지 못한 경우에만 자신이 로드를 시도한다.

```
Application ClassLoader → Platform ClassLoader → Bootstrap ClassLoader
                                (위임 방향)
```

이 메커니즘이 중요한 이유: 애플리케이션 클래스패스에 악의적인 `java.lang.String` 클래스가 있어도, 부트스트랩이 먼저 진짜 String을 로드하므로 무력화된다. 또한 같은 클래스가 여러 곳에서 로드되는 중복을 방지한다.

## 클래스 로딩의 세 단계

### 1. Loading (로딩)

`.class` 파일(바이트 배열)을 찾아 읽고, JVM의 메서드 영역(Method Area)에 `Class` 객체를 생성한다. `ClassLoader.defineClass()` 메서드가 바이트 배열을 `Class` 객체로 변환한다.

### 2. Linking (링킹)

3단계로 나뉜다.

- **Verification(검증)**: 바이트코드가 JVM 명세를 올바르게 따르는지 검사한다. 스택 오버플로우/언더플로우 없음, 올바른 타입 사용, 초기화되지 않은 변수 참조 없음, 접근 제어자 준수 등을 확인한다. 이 검증 덕분에 JVM은 실행 중 추가적인 타입 체크 없이 최적화된 코드를 실행할 수 있다.
- **Preparation(준비)**: `static` 필드에 기본값을 할당한다 (`int`는 0, 참조 타입은 null). 실제 초기화는 Initialization 단계에서 이루어진다.
- **Resolution(결합)**: 심볼릭 참조(symbolic reference)를 실제 메모리 참조로 변환한다. 예를 들어 바이트코드의 `"java/lang/String"` 문자열이 실제 `Class` 객체 주소로 바뀐다.

### 3. Initialization (초기화)

`<clinit>` (클래스 이니셜라이저) 메서드를 실행한다. `static` 블록과 `static` 필드 초기화 코드가 여기서 실행된다. JVM은 이 과정의 스레드 안전성을 보장한다.

```java
class MyClass {
    static int VALUE;
    static {
        // <clinit> 메서드에서 실행
        VALUE = computeValue();
        System.out.println("클래스 초기화!");
    }
}
```

## 코드 예제 1: 커스텀 ClassLoader로 핫 리로딩 구현

```java
import java.io.*;
import java.lang.reflect.*;
import java.nio.file.*;

/**
 * 파일시스템에서 클래스를 직접 로드하는 커스텀 ClassLoader.
 * 매번 새 인스턴스를 생성해 클래스 파일 변경 시 재로드(핫 리로딩) 효과를 낸다.
 */
public class HotReloadClassLoader extends ClassLoader {
    private final Path classDir;

    public HotReloadClassLoader(Path classDir, ClassLoader parent) {
        super(parent);
        this.classDir = classDir;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        // 패키지 구분자(.)를 경로 구분자(/)로 변환
        String classFile = name.replace('.', File.separatorChar) + ".class";
        Path path = classDir.resolve(classFile);

        try {
            byte[] bytes = Files.readAllBytes(path);
            // defineClass: JVM이 바이트 배열을 검증 후 Class 객체 생성
            return defineClass(name, bytes, 0, bytes.length);
        } catch (IOException e) {
            // 파일을 못 찾으면 부모에게 위임
            return super.loadClass(name);
        }
    }
}

public class HotReloadDemo {
    interface Plugin {
        String execute();
    }

    public static void main(String[] args) throws Exception {
        Path classDir = Path.of("build/hot");
        String className = "com.example.PluginImpl";

        for (int i = 0; i < 5; i++) {
            // 매 반복마다 새 ClassLoader 생성 = 새 클래스 버전 로드 가능
            ClassLoader loader = new HotReloadClassLoader(
                classDir, Thread.currentThread().getContextClassLoader()
            );

            Class<?> clazz = loader.loadClass(className);
            Plugin plugin = (Plugin) clazz.getDeclaredConstructor().newInstance();

            System.out.println("버전 " + i + ": " + plugin.execute());

            // 실제 핫 리로딩에서는 여기서 파일 변경 감지 후 재로드
            Thread.sleep(1000);
        }

        // 주의: 서로 다른 ClassLoader에서 로드된 같은 클래스는 JVM이 다른 타입으로 취급
        ClassLoader loader1 = new HotReloadClassLoader(classDir, null);
        ClassLoader loader2 = new HotReloadClassLoader(classDir, null);
        Class<?> c1 = loader1.loadClass(className);
        Class<?> c2 = loader2.loadClass(className);
        System.out.println("동일 클래스?: " + (c1 == c2));  // false!
        // c1의 인스턴스를 c2 타입으로 캐스팅하면 ClassCastException 발생
    }
}
```

## 바이트코드 검증 심화

JVM 검증기는 4단계로 동작한다.

**1단계**: 클래스 파일 구조 검증 — 매직 넘버 `0xCAFEBABE`, 버전 번호, 상수 풀 구조의 올바름을 확인한다.

**2단계**: 클래스 메타데이터 검증 — 슈퍼클래스가 `final`이 아님, 인터페이스 구현의 완전성 등을 확인한다.

**3단계 (데이터 플로우 분석)**: 가장 복잡한 단계. 모든 가능한 실행 경로를 추적해 스택과 지역 변수의 타입 정확성을 검증한다. Java 7부터는 `StackMapTable` 속성을 사용해 컴파일러가 타입 정보를 제공하여 검증 속도를 높인다.

**4단계**: 심볼릭 참조 검증 — Resolution 단계에서 실제 클래스/메서드/필드 접근 가능성을 확인한다.

## Java Instrumentation API와 Java Agent

Java Agent는 JVM 시작 시 또는 실행 중에 `ClassFileTransformer`를 등록해 클래스 로딩 과정에 개입한다. New Relic, Datadog APM, JaCoCo 코드 커버리지, Spring AOP weaving이 이 방식을 활용한다.

Java Agent는 `MANIFEST.MF`에 `Premain-Class`를 선언하고 JAR로 패키징한 후, `-javaagent:agent.jar` JVM 옵션으로 로드한다.

```java
// TimingAgent.java
import java.lang.instrument.*;

public class TimingAgent {
    /**
     * JVM 기동 시 main() 이전에 호출됨
     * java -javaagent:timing-agent.jar com.example.Main
     */
    public static void premain(String agentArgs, Instrumentation inst) {
        System.out.println("[Agent] 메서드 타이밍 계측 에이전트 시작");

        // ClassFileTransformer 등록 — 이후 로드되는 모든 클래스에 적용
        inst.addTransformer(new TimingTransformer(), true /* canRetransform */);
    }

    /**
     * JVM 실행 중 동적으로 로드 가능 (Attach API 사용)
     * jcmd <pid> ManagementAgent.start
     */
    public static void agentmain(String agentArgs, Instrumentation inst) {
        System.out.println("[Agent] 실행 중 에이전트 동적 로드");
        inst.addTransformer(new TimingTransformer(), true);

        // 이미 로드된 클래스 재변환
        for (Class<?> clazz : inst.getAllLoadedClasses()) {
            if (inst.isModifiableClass(clazz)) {
                try {
                    inst.retransformClasses(clazz);
                } catch (Exception e) {
                    // 일부 클래스는 재변환 불가
                }
            }
        }
    }
}
```

## 코드 예제 2: ASM 라이브러리로 메서드 실행 시간 계측

ASM은 JVM 바이트코드를 직접 조작하는 저수준 라이브러리로, Spring, Hibernate, CGLib이 내부적으로 사용한다.

```java
// pom.xml 의존성:
// <dependency><groupId>org.ow2.asm</groupId>
//   <artifactId>asm</artifactId><version>9.6</version></dependency>
// <dependency><groupId>org.ow2.asm</groupId>
//   <artifactId>asm-commons</artifactId><version>9.6</version></dependency>

import org.objectweb.asm.*;
import org.objectweb.asm.commons.*;
import java.lang.instrument.*;
import java.security.ProtectionDomain;

class TimingTransformer implements ClassFileTransformer {
    @Override
    public byte[] transform(
            ClassLoader loader,
            String className,
            Class<?> classBeingRedefined,
            ProtectionDomain domain,
            byte[] classfileBuffer) {

        // JVM 내부 클래스와 에이전트 자신은 변환 제외
        if (className == null
                || className.startsWith("java/")
                || className.startsWith("javax/")
                || className.startsWith("jdk/")
                || className.startsWith("sun/")
                || className.startsWith("com/example/agent/")) {
            return null; // null 반환 = 원본 바이트코드 유지
        }

        try {
            ClassReader reader = new ClassReader(classfileBuffer);
            ClassWriter writer = new ClassWriter(
                reader, ClassWriter.COMPUTE_FRAMES | ClassWriter.COMPUTE_MAXS
            );

            // ClassVisitor로 클래스를 방문하면서 각 메서드를 변환
            ClassVisitor visitor = new ClassVisitor(Opcodes.ASM9, writer) {
                @Override
                public MethodVisitor visitMethod(
                        int access, String name, String descriptor,
                        String signature, String[] exceptions) {

                    MethodVisitor mv = super.visitMethod(
                        access, name, descriptor, signature, exceptions
                    );

                    // 생성자와 정적 이니셜라이저는 제외
                    if (name.equals("<init>") || name.equals("<clinit>")) {
                        return mv;
                    }

                    // TimingAdvice로 래핑: 메서드 진입/종료 시 코드 삽입
                    return new TimingAdvice(mv, access, name, descriptor, className);
                }
            };

            reader.accept(visitor, ClassReader.EXPAND_FRAMES);
            return writer.toByteArray();

        } catch (Exception e) {
            System.err.println("[Agent] 변환 실패: " + className + " — " + e.getMessage());
            return null; // 실패 시 원본 유지
        }
    }
}

/**
 * AdviceAdapter를 상속해 메서드 진입(onMethodEnter)과
 * 종료(onMethodExit) 시점에 바이트코드를 삽입한다.
 */
class TimingAdvice extends AdviceAdapter {
    private final String className;
    private final String methodName;
    private int startTimeVar; // 지역 변수 슬롯 인덱스

    public TimingAdvice(MethodVisitor mv, int access,
                        String name, String desc, String className) {
        super(Opcodes.ASM9, mv, access, name, desc);
        this.className = className;
        this.methodName = name;
    }

    @Override
    protected void onMethodEnter() {
        // 삽입: long startTime = System.nanoTime();
        mv.visitMethodInsn(
            INVOKESTATIC, "java/lang/System", "nanoTime", "()J", false
        );
        startTimeVar = newLocal(Type.LONG_TYPE);
        mv.visitVarInsn(LSTORE, startTimeVar);
    }

    @Override
    protected void onMethodExit(int opcode) {
        // 예외 경로는 별도 처리 (finally 블록처럼 동작하게 하려면 visitMaxs에서 처리)
        if (opcode == ATHROW) return;

        // 삽입: long elapsed = System.nanoTime() - startTime;
        mv.visitMethodInsn(
            INVOKESTATIC, "java/lang/System", "nanoTime", "()J", false
        );
        mv.visitVarInsn(LLOAD, startTimeVar);
        mv.visitInsn(LSUB); // nanoTime() - startTime

        // 삽입: TimingRecorder.record(className, methodName, elapsed);
        mv.visitLdcInsn(className.replace('/', '.'));
        mv.visitLdcInsn(methodName);
        // 스택: [elapsed(long), className(String), methodName(String)] 순서 맞추기
        // DUP2_X2로 long을 위로 올리거나 로컬 변수에 저장
        int elapsedVar = newLocal(Type.LONG_TYPE);
        mv.visitVarInsn(LSTORE, elapsedVar);
        mv.visitVarInsn(LLOAD, elapsedVar);
        mv.visitMethodInsn(
            INVOKESTATIC,
            "com/example/agent/TimingRecorder",
            "record",
            "(Ljava/lang/String;Ljava/lang/String;J)V",
            false
        );
    }
}

// 계측 데이터 수집기
class TimingRecorder {
    public static void record(String className, String methodName, long elapsedNs) {
        if (elapsedNs > 1_000_000) { // 1ms 이상만 로깅
            System.out.printf("[Timing] %s.%s: %.2f ms%n",
                className, methodName, elapsedNs / 1_000_000.0);
        }
    }
}
```

MANIFEST.MF 설정:
```
Manifest-Version: 1.0
Premain-Class: TimingAgent
Agent-Class: TimingAgent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
```

빌드 및 실행:
```bash
# 컴파일
javac -cp asm-9.6.jar:asm-commons-9.6.jar *.java

# JAR 생성
jar cfm timing-agent.jar MANIFEST.MF *.class

# 에이전트와 함께 실행
java -javaagent:timing-agent.jar -cp myapp.jar com.example.Main
```

## ClassLoader 격리와 OSGi

여러 플러그인이 동일 JVM에서 실행되면서 서로 다른 버전의 같은 라이브러리를 사용해야 하는 경우, **ClassLoader 격리**가 필요하다. OSGi(Open Services Gateway initiative) 프레임워크는 번들(bundle)마다 독립적인 ClassLoader를 사용해 버전 충돌을 방지한다. Eclipse, Jenkins 등이 OSGi를 활용한다.

```
OSGi ClassLoader 격리:
플러그인A (Guava 30) ──┐
                        ├── JVM
플러그인B (Guava 31) ──┘

각 플러그인의 ClassLoader가 자신의 Guava 버전만 로드
서로 다른 ClassLoader의 Class 객체는 타입 호환 불가
```

## PermGen/Metaspace와 ClassLoader 누수

클래스 메타데이터는 Java 7까지는 PermGen(Permanent Generation)에, Java 8+에서는 Metaspace(네이티브 메모리)에 저장된다. **클래스 언로딩은 해당 ClassLoader가 GC될 때만 발생한다.**

커스텀 ClassLoader를 매번 생성하면서 이전 로더에 대한 참조를 유지하면 클래스들이 언로드되지 않아 Metaspace를 채운다. Tomcat이 웹 애플리케이션 재배포 시 Metaspace 누수를 일으키는 주요 원인이다.

```java
// 누수 발생 패턴
Map<String, ClassLoader> loaderMap = new HashMap<>();
while (true) {
    HotReloadClassLoader loader = new HotReloadClassLoader(dir, parent);
    loaderMap.put("v" + version++, loader); // 참조 유지 → 누수!
    // loader가 GC되지 않으므로 클래스도 언로드 안 됨
}

// 올바른 패턴: 이전 로더 참조 해제
ClassLoader currentLoader = null;
while (true) {
    currentLoader = new HotReloadClassLoader(dir, parent);
    // 이전 currentLoader는 참조 해제 → GC 가능 → 클래스 언로드
}
```

## 주의사항 및 실전 팁

**서로 다른 ClassLoader의 클래스는 다른 타입이다.** 같은 클래스 이름이라도 다른 ClassLoader에서 로드된 인스턴스는 instanceof 검사나 캐스팅이 실패한다. `ClassCastException: X cannot be cast to X` 에러가 발생하면 ClassLoader 불일치를 의심하라.

**`-verbose:class` 옵션으로 클래스 로딩을 모니터링하라.** 어떤 클래스가 언제 어디서 로드되는지 추적할 수 있다.

**Java 9+ 모듈 시스템과 강한 캡슐화:** Java 16+부터 `--illegal-access` 옵션이 제거되고, 강한 캡슐화(strong encapsulation)가 기본으로 적용된다. 일부 바이트코드 조작이 제한될 수 있으며, 필요한 경우 `--add-opens java.base/java.lang=ALL-UNNAMED` 형태로 명시적으로 열어야 한다.

**ASM 버전 매칭:** ASM API 버전(`ASM9`, `ASM8` 등)은 타겟 JVM 버전과 맞춰야 한다. 최신 JVM의 새 바이트코드 명령어를 구식 ASM이 모르면 변환 중 예외가 발생한다.

**ClassWriter.COMPUTE_FRAMES 플래그:** 바이트코드 조작 시 스택 프레임을 직접 계산하는 것은 복잡하고 오류가 많다. `COMPUTE_FRAMES` 플래그를 사용하면 ASM이 자동으로 계산하지만, 성능 오버헤드가 있으므로 프로덕션 에이전트에서는 `COMPUTE_MAXS`만 사용하고 프레임은 직접 관리하는 것이 좋다.

## 참고 자료
- [JVM 명세 5장 - 클래스 로딩](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-5.html)
- [java.lang.instrument API 문서](https://docs.oracle.com/en/java/docs/java/lang/instrument/package-summary.html)
- [ASM 라이브러리 가이드](https://asm.ow2.io/asm4-guide.pdf)
- [Baeldung - Java ClassLoader 심층 분석](https://www.baeldung.com/java-classloaders)
