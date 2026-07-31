---
layout: post
title: "스택 기반 VM vs 레지스터 기반 VM: JVM과 Dalvik이 선택한 서로 다른 길"
date: 2026-07-31
categories: [cs, computer-science]
tags: [vm, jvm, dalvik, bytecode, virtual-machine, stack-machine, register-machine, cpython]
---

프로그래밍 언어를 실행하는 가상 머신(VM)을 설계할 때 가장 근본적인 아키텍처 결정 중 하나는 **스택 기반**으로 만들 것인가, **레지스터 기반**으로 만들 것인가이다. Java의 JVM은 스택 기반을, 안드로이드의 Dalvik은 레지스터 기반을 선택했다. CPython 역시 스택 기반 VM이다. 이 두 방식은 동일한 문제를 해결하는 서로 다른 철학이며, 각각 명확한 장단점이 존재한다. 이 글에서는 두 아키텍처의 내부 동작 원리를 깊이 있게 살펴본다.

## 가상 머신이란 무엇인가

가상 머신(VM)은 실제 하드웨어 CPU와 메모리를 소프트웨어로 모사하는 추상 계층이다. VM 기반 언어는 소스 코드를 직접 네이티브 코드로 컴파일하지 않고, 중간 표현인 **바이트코드(bytecode)**로 컴파일한 뒤 VM이 이를 해석·실행한다. 이 접근법은 플랫폼 독립성("Write Once, Run Anywhere")과 런타임 최적화(JIT)를 가능하게 한다.

VM 내부에서 피연산자(operand)와 중간 계산 결과를 **어디에 저장하는가**에 따라 두 가지 아키텍처로 나뉜다.

- **스택 기반 VM**: 피연산자를 LIFO 스택에 push/pop하며 처리
- **레지스터 기반 VM**: 피연산자를 유한 개수의 가상 레지스터에 저장하여 처리

## 왜 이 선택이 중요한가

언어 VM의 아키텍처는 다음 세 가지에 직접적인 영향을 미친다:

1. **바이트코드 크기**: 코드가 얼마나 compact한가
2. **명령어 실행 횟수**: 동일 연산에 필요한 명령어 수
3. **인터프리터 구현 복잡도**: VM 구현이 얼마나 단순한가

## 스택 기반 VM — JVM의 방식

JVM은 클래스 파일(.class)에 저장된 바이트코드를 스택 기반으로 실행한다. 각 스레드는 독립적인 **JVM 스택**을 갖고, 메서드 호출마다 **스택 프레임(Stack Frame)**이 생성된다. 각 프레임에는 지역 변수 배열(Local Variable Array), 피연산자 스택(Operand Stack), 상수 풀 참조가 포함된다.

`a + b`를 계산하는 JVM 바이트코드를 살펴보자:

```
iload_0    ; 지역 변수 0번(a)을 스택에 push
iload_1    ; 지역 변수 1번(b)을 스택에 push
iadd       ; 스택에서 두 값을 pop하여 더한 뒤 결과를 push
istore_2   ; 스택 top을 지역 변수 2번(c)에 store
```

4개의 명령어가 필요하다. 각 명령어는 1~3 바이트이며, `iload_0`처럼 암묵적으로 피연산자 위치를 스택으로 가정하기 때문에 **별도의 피연산자 인코딩이 필요 없다**. 결과적으로 바이트코드가 매우 compact하다.

### 스택 기반 VM 시뮬레이터 구현 (Python)

```python
class StackVM:
    def __init__(self):
        self.stack = []
        self.locals = {}

    def execute(self, bytecode, args):
        for i, arg in enumerate(args):
            self.locals[i] = arg

        pc = 0
        while pc < len(bytecode):
            op = bytecode[pc]
            pc += 1

            if op == 'LOAD_LOCAL':
                idx = bytecode[pc]; pc += 1
                self.stack.append(self.locals[idx])

            elif op == 'LOAD_CONST':
                val = bytecode[pc]; pc += 1
                self.stack.append(val)

            elif op == 'ADD':
                b = self.stack.pop()
                a = self.stack.pop()
                self.stack.append(a + b)

            elif op == 'MUL':
                b = self.stack.pop()
                a = self.stack.pop()
                self.stack.append(a * b)

            elif op == 'STORE_LOCAL':
                idx = bytecode[pc]; pc += 1
                self.locals[idx] = self.stack.pop()

            elif op == 'RETURN':
                return self.stack.pop()

        return None

# (a + b) * c 계산
vm = StackVM()
program = [
    'LOAD_LOCAL', 0,   # a
    'LOAD_LOCAL', 1,   # b
    'ADD',             # a + b
    'LOAD_LOCAL', 2,   # c
    'MUL',             # (a + b) * c
    'RETURN',
]
result = vm.execute(program, args=[3, 4, 5])
print(result)  # 35
```

스택 기반 VM의 핵심은 명령어가 **암묵적으로 스택 top을 피연산자로 사용**한다는 점이다. 명령어 인코딩이 단순하고 인터프리터 구현이 쉽지만, 각 연산마다 push/pop이 반복된다.

## 레지스터 기반 VM — Dalvik의 방식

안드로이드의 Dalvik VM(ART의 전신)은 레지스터 기반 VM이다. Dalvik 바이트코드는 `.dex` 파일 포맷을 사용하며, 각 명령어는 연산에 사용할 레지스터 번호를 명시적으로 인코딩한다.

동일한 `a + b` 연산의 Dalvik 바이트코드:

```
add-int v2, v0, v1  ; v2 = v0 + v1 (단 1개의 명령어!)
```

단 **1개의 명령어**로 같은 연산을 처리한다. 하지만 레지스터 번호를 명시해야 하므로 명령어 자체가 더 넓다(16비트 명령어 셋). Dalvik에서 각 메서드의 레지스터 수는 컴파일 시 정적으로 결정되며, 일반적으로 `v0`~`v15`를 사용한다.

Dalvik에서 메서드 인수는 메서드의 **마지막 N개 레지스터**에 배치된다. 예를 들어 4개의 레지스터를 사용하는 메서드에 인수 2개(`a`, `b`)를 전달하면:
- `v0`, `v1`: 로컬 변수용
- `v2`: 인수 a
- `v3`: 인수 b

### 레지스터 기반 VM 시뮬레이터 구현 (Python)

```python
class RegisterVM:
    def __init__(self, num_registers=16):
        self.regs = [0] * num_registers

    def execute(self, bytecode, args):
        # 인수를 마지막 N개 레지스터에 배치
        n = len(args)
        for i, arg in enumerate(args):
            self.regs[len(self.regs) - n + i] = arg

        pc = 0
        while pc < len(bytecode):
            instr = bytecode[pc]
            pc += 1
            op = instr[0]

            if op == 'MOVE':
                dst, src = instr[1], instr[2]
                self.regs[dst] = self.regs[src]

            elif op == 'CONST':
                dst, val = instr[1], instr[2]
                self.regs[dst] = val

            elif op == 'ADD':
                dst, src1, src2 = instr[1], instr[2], instr[3]
                self.regs[dst] = self.regs[src1] + self.regs[src2]

            elif op == 'MUL':
                dst, src1, src2 = instr[1], instr[2], instr[3]
                self.regs[dst] = self.regs[src1] * self.regs[src2]

            elif op == 'RETURN':
                src = instr[1]
                return self.regs[src]

        return None

# (a + b) * c 계산 (레지스터 0~5, 인수 a=v3, b=v4, c=v5)
vm = RegisterVM(num_registers=6)
program = [
    ('ADD',    0, 3, 4),  # v0 = v3(a) + v4(b)
    ('MUL',    1, 0, 5),  # v1 = v0 * v5(c)
    ('RETURN', 1),         # return v1
]
result = vm.execute(program, args=[3, 4, 5])
print(result)  # 35
```

레지스터 기반 VM은 명령어 수가 적어 **디스패치 오버헤드**가 줄어든다. 반면 명령어에 레지스터 번호가 포함되어 바이트코드 크기가 커지고, 인터프리터 구현이 다소 복잡해진다.

## 정량적 비교

| 특성 | 스택 기반 VM (JVM) | 레지스터 기반 VM (Dalvik) |
|------|-------------------|------------------------|
| 코드 크기 | 더 작음 (묵시적 피연산자) | 더 큼 (명시적 레지스터) |
| 명령어 수 | 더 많음 | 더 적음 |
| 인터프리터 복잡도 | 단순 | 복잡 |
| 명령어 디스패치 | 더 빈번 | 덜 빈번 |
| JIT 친화성 | 보통 | 우수 |
| 플랫폼 독립성 | 매우 높음 | 높음 |

Google의 Dalvik 설계 문서에 따르면, 동일한 Java 프로그램에서 Dalvik 바이트코드는 JVM 바이트코드 대비 **명령어 수가 약 30% 적다**. 대신 코드 크기는 약 35% 더 크다.

## CPython의 스택 기반 VM

Python의 기본 구현인 CPython 역시 스택 기반 VM이다. CPython 3.11부터는 **특화 적응형(Specializing Adaptive) 인터프리터**를 도입하여 자주 실행되는 명령어를 런타임에 특화 버전으로 교체하는 최적화를 수행한다.

Python 코드를 디스어셈블하면 JVM과 유사한 스택 기반 구조를 확인할 수 있다:

```python
import dis

def add(a, b):
    return a + b

dis.dis(add)
# LOAD_FAST 'a'   → 스택에 a push
# LOAD_FAST 'b'   → 스택에 b push
# BINARY_OP +     → 두 값 pop, 더해서 push
# RETURN_VALUE    → 스택 top 반환
```

## 현대의 선택: WebAssembly와 V8

WebAssembly(Wasm) 역시 **스택 기반 VM**을 채택했다. 스택 기반은 바이너리 크기를 최소화하고 형식 검증(validation)이 단순하다는 장점이 있기 때문이다. 반면 V8(JavaScript 엔진)의 Ignition 인터프리터는 **레지스터 기반**이며, 이후 Turbofan JIT 컴파일러가 네이티브 코드로 최적화한다.

## 주의사항 및 팁

1. **VM 아키텍처가 성능을 결정하지 않는다**: 최종 성능은 JIT 컴파일러 품질에 더 크게 좌우된다. JVM의 HotSpot JIT는 스택 기반임에도 네이티브 코드 수준 성능을 낸다.

2. **바이트코드 분석 도구 활용**: JVM은 `javap -c`, CPython은 `dis` 모듈, Dalvik은 `baksmali`로 바이트코드를 분석할 수 있다.

3. **스택 기반은 검증이 쉽다**: WebAssembly가 스택 기반을 선택한 이유 중 하나는 타입 검사와 안전성 검증이 단순하기 때문이다.

4. **레지스터 수 제한**: 레지스터 기반 VM에서 레지스터 수는 컴파일 시 고정된다. 레지스터가 부족하면 스택으로 스필(spill)해야 하는 문제가 발생한다.

5. **인터프리터 루프 최적화**: 두 방식 모두 `switch-dispatch`보다 **computed goto**(C의 `goto *table[op]`)를 사용하면 명령어 디스패치 비용을 크게 줄일 수 있다.

## 참고 자료
- [Java Virtual Machine Specification - SE 21](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-2.html)
- [Dalvik Bytecode Reference - Android Open Source Project](https://source.android.com/docs/core/runtime/dalvik-bytecode)
- [CPython Internals - dis module documentation](https://docs.python.org/3/library/dis.html)
- [WebAssembly Core Specification](https://webassembly.github.io/spec/core/)
