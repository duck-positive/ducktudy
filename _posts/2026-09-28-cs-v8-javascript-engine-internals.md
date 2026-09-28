---
layout: post
title: "V8 JavaScript 엔진 내부 구조 완전 정복: Hidden Classes·Ignition·TurboFan으로 이해하는 자바스크립트 실행의 비밀"
date: 2026-09-28
categories: [cs, computer-science]
tags: [v8, javascript, jit, hidden-class, ignition, turbofan, optimization, compiler]
---

자바스크립트는 인터프리터 언어임에도 오늘날 네이티브 코드에 근접한 성능을 발휘한다. 그 중심에는 Google이 개발한 V8 엔진이 있다. V8은 Chrome, Node.js, Deno, Edge를 포함한 수십억 개의 런타임을 구동하며, 단순한 인터프리터를 훨씬 넘어선 정교한 다단계 JIT(Just-In-Time) 컴파일러 파이프라인을 갖추고 있다. 이 글에서는 V8의 내부 동작 원리를 Hidden Classes(숨겨진 클래스), Ignition 인터프리터, TurboFan JIT 컴파일러를 중심으로 깊이 있게 분석한다.

---

## 왜 V8의 내부를 알아야 하는가

자바스크립트 개발자가 V8 내부를 이해해야 하는 이유는 명확하다. V8이 최적화하는 코드 패턴을 따르면 같은 알고리즘이라도 10배 이상 빠른 실행 속도를 얻을 수 있다. 반대로 V8의 가정을 깨는 코드를 작성하면 최적화된 기계 코드가 폐기(deoptimization)되어 성능이 급락한다. 이른바 "V8-friendly 코드"를 작성하려면 엔진의 내부 구조를 이해하는 것이 선결 조건이다.

---

## V8의 전체 파이프라인

V8의 코드 실행 파이프라인은 다음과 같은 단계로 구성된다:

```
소스 코드
  ↓
[스캐너(Scanner)]  — 토큰 생성
  ↓
[파서(Parser)]     — AST(Abstract Syntax Tree) 생성
  ↓
[Ignition]        — 바이트코드 생성 및 인터프리팅 (타입 피드백 수집)
  ↓
[Sparkplug]       — 빠른 기계 코드 생성 (최소한의 최적화)
  ↓
[Maglev]          — 중간 단계 최적화 JIT
  ↓
[TurboFan]        — 최고 성능 최적화 JIT (핫 코드 대상)
```

각 단계는 "코드가 얼마나 자주 실행되는가"를 기준으로 적절한 컴파일 전략을 선택한다. 자주 실행되지 않는 코드는 인터프리팅으로, 핫 코드(hot code)는 TurboFan으로 최적화된다.

---

## Hidden Classes: V8의 객체 형태 추적

V8이 자바스크립트 객체를 빠르게 접근할 수 있는 핵심 비결은 **Hidden Class**(내부적으로 Map 또는 Shape라고도 불림)다. 자바스크립트는 동적 타입 언어이므로 객체의 프로퍼티는 언제든 추가·삭제될 수 있다. 만약 V8이 모든 프로퍼티를 해시 테이블로 관리한다면 접근 비용이 O(1)이지만 캐시 친화성이 낮다. 이를 해결하기 위해 V8은 Hidden Class를 도입했다.

### Hidden Class 전이(Transition)

```javascript
// 예제 1: Hidden Class 전이 시연
function Point(x, y) {
  this.x = x;  // Hidden Class C0 → C1 (x 추가)
  this.y = y;  // Hidden Class C1 → C2 (y 추가)
}

const p1 = new Point(1, 2);  // Hidden Class C2 할당
const p2 = new Point(3, 4);  // 동일한 Hidden Class C2 공유 → 최적화 가능

// 문제: 프로퍼티 추가 순서가 다르면 별도의 Hidden Class가 생성됨
function BadPoint(flag) {
  if (flag) {
    this.x = 0;
    this.y = 0;
  } else {
    this.y = 0;  // y를 먼저 추가!
    this.x = 0;
  }
}

const b1 = new BadPoint(true);   // Hidden Class BC1 (x→y 순서)
const b2 = new BadPoint(false);  // Hidden Class BC2 (y→x 순서) — 다른 Hidden Class!
// p1, p2는 같은 Hidden Class를 공유하므로 인라인 캐시(IC)가 단형(monomorphic)
// b1, b2는 다른 Hidden Class이므로 IC가 이형(polymorphic) → 성능 저하
```

`Point` 생성자가 항상 같은 순서로 프로퍼티를 초기화하면 모든 `Point` 인스턴스가 동일한 Hidden Class를 공유한다. V8은 해당 Hidden Class에 대해 각 프로퍼티의 메모리 오프셋을 고정하고, 프로퍼티 접근을 해시 탐색 없이 단순한 오프셋 연산으로 처리한다.

### 인라인 캐시(Inline Cache, IC)

Hidden Class와 짝을 이루는 개념이 인라인 캐시다. Ignition이 `obj.x`와 같은 프로퍼티 접근을 처음 실행할 때, 해당 객체의 Hidden Class와 프로퍼티 오프셋을 캐시해 둔다. 이후 동일한 Hidden Class의 객체가 오면 캐시된 오프셋을 직접 사용한다.

- **단형(monomorphic) IC**: 항상 같은 Hidden Class → 최고 성능
- **이형(polymorphic) IC**: 2~4가지 Hidden Class → 약간의 오버헤드
- **메가모픽(megamorphic) IC**: 5가지 이상 → 캐시 포기, 해시 탐색으로 폴백

---

## Ignition: 바이트코드 인터프리터

Ignition은 V8의 기저 레이어로, 파서가 생성한 AST를 **바이트코드(Bytecode)**로 변환하고 레지스터 기반 가상 머신 위에서 실행한다. Ignition의 역할은 단순히 코드를 실행하는 것 이상으로, 실행 중에 **타입 피드백(type feedback)**을 수집하는 것이다.

```javascript
// 예제 2: 타입 피드백이 TurboFan 최적화에 미치는 영향
function add(a, b) {
  return a + b;
}

// Ignition이 이 함수를 여러 번 실행하며 타입 피드백 수집
for (let i = 0; i < 10000; i++) {
  add(1, 2);  // 항상 number + number
}
// TurboFan은 "a, b는 항상 number"라는 가정 하에 정수 덧셈으로 최적화

// 하지만 다음을 실행하면 가정이 깨짐!
add("hello", " world");  // string + string → 역최적화(deoptimization) 발생
// TurboFan이 생성한 최적화 코드가 폐기되고 Ignition으로 복귀
```

Ignition이 수집하는 타입 피드백에는 다음이 포함된다:
- 함수 인수의 타입(number, string, object, undefined 등)
- 연산자의 피연산자 타입
- 프로퍼티 접근 시의 Hidden Class
- 분기 예측을 위한 조건문 결과 통계

---

## TurboFan: 최적화 JIT 컴파일러

TurboFan은 V8의 최상위 JIT 컴파일러로, Ignition이 수집한 타입 피드백을 바탕으로 핫 함수를 극도로 최적화된 기계 코드로 컴파일한다. TurboFan은 **Sea-of-Nodes**라는 독특한 중간 표현(IR)을 사용하며, 다음과 같은 최적화를 적용한다:

| 최적화 기법 | 설명 |
|---|---|
| 타입 특화(Type Specialization) | "이 변수는 항상 int32" 가정으로 타입 검사 제거 |
| 함수 인라이닝(Inlining) | 소규모 함수를 호출 지점에 직접 삽입 |
| 이스케이프 분석(Escape Analysis) | 스택에만 머무는 객체를 힙 할당 없이 처리 |
| 범위 체크 제거(Bounds Check Elim.) | 루프 내 배열 인덱스 유효성 검사 제거 |
| 데드 코드 제거(DCE) | 절대 실행되지 않는 코드 삭제 |

### 역최적화(Deoptimization)의 위험

TurboFan은 수집된 타입 피드백을 기반으로 "투기적(speculative)" 최적화를 수행한다. 만약 런타임에 가정이 깨지면 **역최적화**가 발생한다:

```javascript
// 예제 3: 역최적화를 피하는 코드 패턴
function sumArray(arr) {
  let sum = 0;
  for (let i = 0; i < arr.length; i++) {
    sum += arr[i];
  }
  return sum;
}

// 최적화 유도: 항상 같은 타입의 배열 전달
const nums = [1, 2, 3, 4, 5];
sumArray(nums);  // 워밍업 — TurboFan이 SMI(Small Integer) 배열로 최적화

// 나쁜 패턴: 배열 구멍(holes)이 있으면 최적화 저하
const holey = [1, , 3, , 5];  // 구멍(hole) 있는 배열 — HOLEY_SMI_ELEMENTS
sumArray(holey);  // 구멍 처리 로직이 추가되어 최적화 수준 저하

// 최적화를 방해하는 또 다른 패턴: 배열에 다른 타입 혼합
const mixed = [1, 2, "three", 4];  // PACKED_ELEMENTS로 폴백
sumArray(mixed);  // 제네릭 요소 접근으로 역최적화 → 느림
```

---

## 성능을 높이는 V8 친화적 코딩 패턴

### 1. 객체 프로퍼티 순서 일관성 유지

```javascript
// 나쁜 패턴: 동적 프로퍼티 추가
function createUser(name, age, role) {
  const user = {};
  if (name) user.name = name;
  if (age) user.age = age;       // 조건에 따라 Hidden Class가 달라짐
  if (role) user.role = role;
  return user;
}

// 좋은 패턴: 초기화 시 모든 프로퍼티를 선언
function createUserFast(name, age, role) {
  return {
    name: name || null,
    age: age || 0,
    role: role || 'guest'
  };
}
```

### 2. 배열 타입 안정성 유지

```javascript
// V8 배열 요소의 종류(ElementsKind):
// PACKED_SMI_ELEMENTS  → 정수만 있고 구멍 없음 (최고 성능)
// PACKED_DOUBLE_ELEMENTS → 부동소수점, 구멍 없음
// PACKED_ELEMENTS      → 혼합 타입, 구멍 없음
// HOLEY_SMI_ELEMENTS   → 정수 + 구멍
// HOLEY_ELEMENTS       → 혼합 + 구멍 (최저 성능)

const fast = [];
for (let i = 0; i < 100; i++) {
  fast.push(i);  // PACKED_SMI_ELEMENTS 유지
}

// 한 번 다운그레이드된 배열은 되돌릴 수 없음
fast.push(3.14);   // → PACKED_DOUBLE_ELEMENTS
fast.push("str");  // → PACKED_ELEMENTS
```

---

## 주의사항과 팁

### delete 연산자를 피하라

```javascript
const obj = { x: 1, y: 2 };
delete obj.x;  // Hidden Class가 새로운 클래스로 전이되어 최적화 방해
// 대신 null 할당 사용:
obj.x = null;  // Hidden Class 유지
```

### arguments 객체와 eval 사용 금지

`arguments` 객체 사용이나 `eval()` 호출은 V8이 해당 함수에 대해 전제할 수 있는 정적 구조를 파괴하여 TurboFan 최적화를 막는다. ES6의 나머지 매개변수(`...args`)를 대신 사용하라.

### 함수 크기를 적절히 유지

너무 큰 함수는 TurboFan이 인라이닝하지 않는다. 핫 경로의 함수는 작고 단일 책임을 갖도록 분리하면 인라이닝 최적화 혜택을 받을 수 있다.

---

## 정리

V8은 Hidden Classes를 통해 동적 프로퍼티 접근을 정적 오프셋 접근으로 변환하고, Ignition으로 타입 피드백을 수집하며, TurboFan으로 핫 코드를 네이티브에 가까운 속도로 컴파일한다. 개발자가 이 파이프라인을 이해하고 V8-친화적인 패턴(일관된 Hidden Class, 단일 타입 배열, delete 회피)을 따른다면 자바스크립트 코드의 성능을 몇 배씩 끌어올릴 수 있다.

## 참고 자료

- [V8 Blog - Maglev: V8's Fastest Optimizing JIT](https://v8.dev/blog/maglev)
- [V8 Blog - Ignition: A fast low-overhead register interpreter](https://v8.dev/blog/ignition-interpreter)
- [V8 JavaScript Engine in Node.js: Architecture, Tiers, Shapes, and Deoptimization](https://www.thenodebook.com/node-arch/v8-engine-intro)
- [V8 JavaScript Engine Optimization: TurboFan, Hidden Classes & Performance Tips](https://huntize.com/learn/understanding-v8-and-code-optimization/)
