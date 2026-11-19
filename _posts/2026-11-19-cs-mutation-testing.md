---
layout: post
title: "돌연변이 테스팅(Mutation Testing) 완전 정복: 테스트 스위트의 품질을 측정하고 테스트가 없는 버그를 찾아내는 소프트웨어 테스트 심화 기법"
date: 2026-11-19
categories: [cs, computer-science]
tags: [mutation-testing, software-testing, pitest, stryker, test-quality, 돌연변이테스팅, 소프트웨어테스트, 테스트품질]
---

## 개요

코드 커버리지 100%가 버그 없는 소프트웨어를 보장하지 않는다는 사실은 경험 있는 개발자라면 모두 안다. 모든 라인을 실행해도 실제로 로직이 맞는지 검증하지 않은 테스트가 수두룩하다. **돌연변이 테스팅(Mutation Testing)**은 이 문제를 직접 공략하는 기법이다. 의도적으로 소스 코드에 작은 버그(돌연변이)를 심고, 테스트 스위트가 그 버그를 잡아내는지 검사함으로써 테스트의 실질적 품질을 측정한다.

## 돌연변이 테스팅이란 무엇인가

### 핵심 아이디어

돌연변이 테스팅의 아이디어는 간단하다. 프로그램 소스 코드에 작은 변경을 가해 **변이체(Mutant)**를 만든다. 예를 들어 `>` 연산자를 `>=`로 바꾸거나, `+`를 `-`로 바꾼다. 이 변이체에 대해 기존 테스트 스위트를 실행한다.

- **제거된(Killed) 변이체**: 기존 테스트가 변이체를 감지하여 테스트가 실패하면, 그 변이체는 "제거되었다"고 한다.
- **생존한(Survived) 변이체**: 모든 테스트가 통과하면, 변이체는 "생존했다"고 한다. 이는 테스트가 그 버그를 잡지 못했다는 의미다.

**돌연변이 점수(Mutation Score)**:

```
Mutation Score = (제거된 변이체 수) / (전체 변이체 수) × 100%
```

높은 돌연변이 점수는 테스트가 실질적으로 코드를 검증하고 있다는 강한 증거다.

### 왜 돌연변이 테스팅이 필요한가

**코드 커버리지의 함정**: 다음 코드를 생각해 보자.

```java
public int max(int a, int b) {
    if (a > b) return a;
    return b;
}

@Test
public void testMax() {
    assertEquals(5, max(5, 3)); // 커버리지 100%
}
```

이 테스트는 `a > b` 분기를 실행하므로 커버리지 100%다. 하지만 `a > b`를 `a >= b`로 바꿔도 테스트는 통과한다. 즉, `max(3, 3)` 케이스의 버그를 잡지 못한다.

돌연변이 테스팅은 이런 허점을 자동으로 찾아준다.

## 주요 돌연변이 연산자(Mutation Operators)

| 연산자 종류 | 약어 | 예시 |
|-----------|-----|-----|
| 산술 연산자 교체 | AOR | `a + b` → `a - b` |
| 관계 연산자 교체 | ROR | `a > b` → `a >= b`, `a < b` |
| 논리 연산자 교체 | LCR | `a && b` → `a \|\| b` |
| 단항 연산자 삽입 | UOI | `a` → `-a`, `a++` |
| 조건 경계 변환 | CBS | `for (i < n)` → `for (i <= n)` |
| 반환값 변조 | RVR | `return x` → `return 0`, `return -x` |
| 메서드 호출 제거 | MCD | `list.add(item)` 줄 전체 삭제 |
| 상수 치환 | CSR | `1` → `0`, `-1`, `Integer.MAX_VALUE` |

## 코드 예제

### 예제 1: Java + PITest로 돌연변이 테스팅 실행

**Maven 프로젝트 pom.xml 설정**:

```xml
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.15.3</version>
    <configuration>
        <!-- 돌연변이 연산자 선택 -->
        <mutators>
            <mutator>STRONGER</mutator>
        </mutators>
        <!-- 돌연변이 점수 최소 임계값: 80% 미달 시 빌드 실패 -->
        <mutationThreshold>80</mutationThreshold>
        <!-- 커버리지 임계값: 90% 미달 시 빌드 실패 -->
        <coverageThreshold>90</coverageThreshold>
        <!-- 병렬 실행으로 속도 향상 -->
        <threads>4</threads>
        <!-- 타임아웃: 원본의 1.5배 이상 걸리면 생존 처리 -->
        <timeoutFactor>1.5</timeoutFactor>
        <timeoutConstant>3000</timeoutConstant>
    </configuration>
    <dependencies>
        <!-- JUnit 5 지원 플러그인 -->
        <dependency>
            <groupId>org.pitest</groupId>
            <artifactId>pitest-junit5-plugin</artifactId>
            <version>1.2.1</version>
        </dependency>
    </dependencies>
</plugin>
```

**검사할 대상 코드**: `Calculator.java`

```java
public class Calculator {

    public int add(int a, int b) {
        return a + b;
    }

    public double divide(double a, double b) {
        if (b == 0) {
            throw new ArithmeticException("Division by zero");
        }
        return a / b;
    }

    public boolean isPositive(int n) {
        return n > 0;
    }

    public int clamp(int value, int min, int max) {
        if (value < min) return min;
        if (value > max) return max;
        return value;
    }
}
```

**테스트 코드**: `CalculatorTest.java`

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {

    private final Calculator calc = new Calculator();

    @Test
    void testAdd() {
        assertEquals(5, calc.add(2, 3));
    }

    @Test
    void testDivideByZero() {
        assertThrows(ArithmeticException.class, () -> calc.divide(1, 0));
    }

    @Test
    void testIsPositive() {
        assertTrue(calc.isPositive(1));
        assertFalse(calc.isPositive(-1));
        // 주의: isPositive(0) 테스트 없음!
    }

    @Test
    void testClamp() {
        assertEquals(10, calc.clamp(5, 10, 20));
        assertEquals(20, calc.clamp(25, 10, 20));
        assertEquals(15, calc.clamp(15, 10, 20));
    }
}
```

**실행 명령**:

```bash
mvn test-compile org.pitest:pitest-maven:mutationCoverage
```

**PITest 보고서 출력 (일부)**:

```
================================================================================
- Statistics
================================================================================
>> Generated 23 mutations Killed 17 (74%)
>> Ran 48 tests (2.09 tests per mutation)

SURVIVED:
  Calculator.java:12 → isPositive → ROR: n > 0 → n >= 0
  Calculator.java:12 → isPositive → ROR: n > 0 → n != 0
  Calculator.java:17 → clamp → ROR: value < min → value <= min
```

`isPositive(0)` 케이스와 `clamp(10, 10, 20)` 경계 케이스 테스트가 없어 돌연변이가 생존했다.

**테스트 보강 후**:

```java
@Test
void testIsPositiveZero() {
    assertFalse(calc.isPositive(0)); // 경계값 추가
}

@Test
void testClampBoundary() {
    assertEquals(10, calc.clamp(10, 10, 20)); // 최소 경계
    assertEquals(20, calc.clamp(20, 10, 20)); // 최대 경계
}
```

이제 돌연변이 점수는 약 90% 이상으로 올라간다.

### 예제 2: JavaScript/TypeScript + Stryker로 돌연변이 테스팅

**설치**:

```bash
npm install --save-dev @stryker-mutator/core @stryker-mutator/jest-runner
```

**stryker.config.mjs**:

```javascript
// @ts-check
/** @type {import('@stryker-mutator/api/core').PartialStrykerOptions} */
const config = {
    testRunner: 'jest',
    reporters: ['html', 'clear-text', 'progress', 'dashboard'],
    coverageAnalysis: 'perTest', // 테스트별 커버리지로 속도 향상
    mutate: ['src/**/*.ts', '!src/**/*.spec.ts'],
    thresholds: {
        high: 80,     // 80% 이상: 녹색
        low: 60,      // 60% 이상: 노란색
        break: 50,    // 50% 미만: 빌드 실패
    },
    // 증분 실행: 변경된 파일만 재실행
    incremental: true,
    // 병렬 실행
    concurrency: 4,
};

export default config;
```

**검사할 코드**: `src/shopping-cart.ts`

```typescript
export interface CartItem {
    name: string;
    price: number;
    quantity: number;
}

export class ShoppingCart {
    private items: CartItem[] = [];

    addItem(item: CartItem): void {
        const existing = this.items.find(i => i.name === item.name);
        if (existing) {
            existing.quantity += item.quantity;
        } else {
            this.items.push({ ...item });
        }
    }

    removeItem(name: string): boolean {
        const idx = this.items.findIndex(i => i.name === name);
        if (idx >= 0) {
            this.items.splice(idx, 1);
            return true;
        }
        return false;
    }

    getTotal(): number {
        return this.items.reduce(
            (sum, item) => sum + item.price * item.quantity,
            0
        );
    }

    applyDiscount(percent: number): number {
        if (percent < 0 || percent > 100) {
            throw new RangeError('Discount must be between 0 and 100');
        }
        return this.getTotal() * (1 - percent / 100);
    }
}
```

**테스트 코드**: `src/shopping-cart.spec.ts`

```typescript
import { ShoppingCart } from './shopping-cart';

describe('ShoppingCart', () => {
    let cart: ShoppingCart;

    beforeEach(() => {
        cart = new ShoppingCart();
    });

    it('addItem: 새 상품 추가', () => {
        cart.addItem({ name: 'Apple', price: 1000, quantity: 2 });
        expect(cart.getTotal()).toBe(2000);
    });

    it('addItem: 이미 있는 상품 수량 증가', () => {
        cart.addItem({ name: 'Apple', price: 1000, quantity: 2 });
        cart.addItem({ name: 'Apple', price: 1000, quantity: 3 });
        expect(cart.getTotal()).toBe(5000);
    });

    it('removeItem: 존재하는 상품 제거', () => {
        cart.addItem({ name: 'Apple', price: 1000, quantity: 1 });
        expect(cart.removeItem('Apple')).toBe(true);
        expect(cart.getTotal()).toBe(0);
    });

    it('removeItem: 없는 상품 제거 시 false 반환', () => {
        expect(cart.removeItem('Banana')).toBe(false);
    });

    it('applyDiscount: 정상 할인', () => {
        cart.addItem({ name: 'Apple', price: 10000, quantity: 1 });
        expect(cart.applyDiscount(20)).toBe(8000);
    });

    it('applyDiscount: 범위 오류', () => {
        expect(() => cart.applyDiscount(-1)).toThrow(RangeError);
        expect(() => cart.applyDiscount(101)).toThrow(RangeError);
    });

    // 경계값 테스트 추가
    it('applyDiscount: 0% 할인', () => {
        cart.addItem({ name: 'Apple', price: 1000, quantity: 1 });
        expect(cart.applyDiscount(0)).toBe(1000);
    });

    it('applyDiscount: 100% 할인', () => {
        cart.addItem({ name: 'Apple', price: 1000, quantity: 1 });
        expect(cart.applyDiscount(100)).toBe(0);
    });
});
```

**실행**:

```bash
npx stryker run
```

**Stryker 출력 예시**:

```
Mutation testing is done. Check log for more information.

All files  │ 85.71 │  7/8 killed │ 1 survived │ 0 timeout │ 0 no coverage

Survived mutant:
  shopping-cart.ts:line 20 (removeItem)
  Replaced >= with > on condition: idx >= 0
  → This mutant survived. Add a test for removeItem when idx === 0.
```

첫 번째 아이템을 제거하는 테스트가 없어 `idx >= 0` → `idx > 0` 변이체가 생존했다. 이를 추가하면 85.71% → 100%가 된다.

## 동등 변이체(Equivalent Mutants) 문제

돌연변이 테스팅의 가장 큰 한계는 **동등 변이체**다. 동등 변이체는 원래 프로그램과 모든 입력에 대해 동일한 행동을 하는 변이체다. 예를 들어:

```java
// 원본
for (int i = 0; i < n; i++) { ... }

// 변이체 (동등 변이체)
for (int i = 0; i != n; i++) { ... }
```

`i`가 항상 1씩 증가하고 `n >= 0`이면 두 코드는 동일하게 동작한다. 이런 변이체는 아무리 좋은 테스트를 써도 제거할 수 없다. 동등 변이체의 비율은 보통 전체 변이체의 10~30%다.

**대처법**:
- PITest의 `NON_VOID_METHOD_CALLS` 등 무의미한 연산자를 제외
- 선택적 변이(Selective Mutation): 모든 연산자 대신 고품질 연산자만 선택

## 주의사항과 실전 팁

### 1. 대규모 코드베이스에서의 성능

돌연변이 테스팅은 느리다. 1000개의 변이체 × 10ms 테스트 = 10초지만, 100개 테스트 파일을 각각 실행하면 수 분이 걸린다.

**속도 향상 전략**:
- **증분 실행**: 변경된 파일만 재실행 (Stryker의 `incremental` 옵션)
- **커버리지 분석**: 각 테스트가 커버하는 코드만 해당 테스트로 실행 (PITest의 `--coverageThreshold`)
- **선택적 변이**: 핵심 비즈니스 로직만 대상
- **높은 위험 연산자 우선**: AOR, ROR, LCR만 적용

### 2. CI/CD 파이프라인 통합

```yaml
# GitHub Actions 예시
- name: Mutation Testing
  run: mvn test-compile org.pitest:pitest-maven:mutationCoverage
  # 돌연변이 점수 70% 미만이면 빌드 실패
  
- name: Upload PITest Report
  uses: actions/upload-artifact@v3
  with:
    name: pitest-report
    path: target/pit-reports/
```

풀 리퀘스트 단계에서 돌연변이 점수가 임계값 이하로 떨어지면 머지를 차단하는 것이 효과적이다.

### 3. 테스트 커버리지와 함께 사용

돌연변이 점수는 커버리지를 보완하는 지표다. 목표는 "커버리지 90% + 돌연변이 점수 80%" 같은 조합 임계값을 설정하는 것이다.

| 커버리지 | 돌연변이 점수 | 품질 평가 |
|---------|-------------|---------|
| 100% | 40% | 불량 (테스트가 형식적) |
| 80% | 85% | 우수 (실질적 검증) |
| 70% | 90% | 매우 우수 (높은 신뢰도) |

### 4. 어떤 코드에 적용할까

모든 코드에 돌연변이 테스팅을 적용하는 것은 비효율적이다. 우선순위를 두어야 할 곳:

- **핵심 비즈니스 로직**: 결제, 인증, 계산 로직
- **자주 변경되는 코드**: 회귀 버그 위험이 높음
- **복잡한 조건문**: `if-else` 중첩, 다중 비교
- **가장 중요한 모듈**: 장애 시 영향이 큰 컴포넌트

## 참고 자료

- [Mutation Testing — Wikipedia](https://en.wikipedia.org/wiki/Mutation_testing)
- [PITest — Java 돌연변이 테스팅 도구 공식 문서](https://pitest.org)
- [Stryker Mutator — JavaScript/TypeScript 돌연변이 테스팅 공식 문서](https://stryker-mutator.io)
- [Mutation Testing Tools: Stryker, mutmut, and PITest — kindatechnical.com](https://kindatechnical.com/test-driven-development/mutation-testing-tools-stryker-mutmut-pitest.html)
