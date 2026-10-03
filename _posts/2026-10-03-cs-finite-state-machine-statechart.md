---
layout: post
title: "유한 상태 기계(FSM)와 상태 차트: 복잡한 상태 로직을 우아하게 설계하는 법"
date: 2026-10-03
categories: [cs, computer-science]
tags: [finite-state-machine, statechart, xstate, concurrency, design-pattern, typescript]
---

"이 버튼이 로딩 중일 때 클릭되면 어떻게 되죠?", "에러 상태에서 네트워크가 복구되면요?", "로그아웃 중에 세션이 만료되면요?" — 이런 엣지 케이스들을 boolean 플래그 몇 개로 처리하다 보면 어느 순간 불가능한 상태 조합(`isLoading: true, hasError: true, isSuccess: true`)이 코드베이스에 존재하게 된다. 유한 상태 기계(FSM)와 상태 차트는 이런 문제를 수학적으로 제거한다.

## 개념 설명

유한 상태 기계(Finite State Machine, FSM)는 컴퓨터 과학의 가장 오래되고 실용적인 수학 모델 중 하나다. 1950년대 오토마타 이론에서 탄생했지만, 오늘날 UI 컴포넌트부터 네트워크 프로토콜, 게임 AI까지 광범위하게 활용된다.

### FSM의 수학적 정의

FSM은 5가지 요소로 정의된다:

- **Q**: 유한한 상태(state)들의 집합 → `{idle, loading, success, failure}`
- **Σ**: 입력 알파벳(이벤트)의 집합 → `{FETCH, SUCCESS, ERROR, RESET}`
- **δ**: 전이 함수 (Q × Σ → Q) → `(loading, SUCCESS) → success`
- **q₀**: 초기 상태 → `idle`
- **F**: 수락 상태(accepting states)의 집합 (유한 오토마톤에서 사용)

핵심 불변 조건: **시스템은 항상 정확히 하나의 상태**에 있다. `loading`이면서 동시에 `success`인 상태는 존재하지 않는다.

### 결정론적 vs 비결정론적 FSM

- **DFA(Deterministic FA)**: 각 상태-이벤트 쌍에 대해 전이가 최대 하나. 구현이 단순하고 실행이 예측 가능.
- **NFA(Non-deterministic FA)**: 하나의 이벤트로 여러 상태로 동시에 전이 가능. 정규 표현식 엔진 내부에서 사용됨.

소프트웨어 설계에서는 대부분 DFA를 사용한다.

### 상태 차트 (Statechart)

David Harel이 1987년 발표한 논문 "Statecharts: A Visual Formalism for Complex Systems"에서 제안한 확장이다. 세 가지 핵심 문제를 해결한다:

**문제 1: 상태 폭발(State Explosion)**
n개의 독립적 특성이 있으면 2ⁿ개의 상태가 필요하다. 전등 켜짐/꺼짐, 에어컨 켜짐/꺼짐, 방범 모드 켜짐/꺼짐만 해도 8개 상태가 필요하고, 이 중 무효 조합을 모두 열거해야 한다.  
→ **해결**: 병렬 상태(Parallel States)로 독립적 차원을 분리

**문제 2: 상태 계층 없음**
부모 상태의 공통 전이를 모든 자식 상태에 반복해야 한다. 예를 들어 "로딩 중/성공/실패 어느 상태에서도 ESC를 누르면 취소 상태로"를 3번씩 써야 한다.  
→ **해결**: 계층적 상태(Hierarchical States, 중첩 상태)로 상속 구현

**문제 3: 수치 정보 없음**
재시도 횟수, 사용자 ID, 에러 메시지 등 수치 정보를 FSM 자체로 표현할 수 없다.  
→ **해결**: 확장 상태(Extended State, Context)로 수치 정보 저장

### 핵심 구성 요소

| 개념 | 설명 | 예시 |
|------|------|------|
| **State** | 시스템의 현재 위치 | `idle`, `loading`, `success` |
| **Event** | 상태 전이를 유발하는 입력 | `FETCH`, `SUCCESS`, `CANCEL` |
| **Transition** | 이벤트에 의한 상태 간 이동 | `loading + SUCCESS → success` |
| **Guard** | 전이 발생 조건 | `context.retries < 3` |
| **Action** | 전이 시 실행되는 부수 효과 | `assign({retries: n+1})` |
| **Context** | 확장 상태 (수치 데이터) | `{userId, retries, errorMsg}` |
| **Entry/Exit Action** | 상태 진입/이탈 시 실행 | `onEntry: startTimer()` |

## 왜 필요한가

### boolean 플래그의 함정

다음 코드는 흔한 패턴이다:

```typescript
// 안티패턴: boolean 플래그 조합
interface ComponentState {
    isLoading: boolean;
    isError: boolean;
    isSuccess: boolean;
    data: unknown;
    error: unknown;
}
```

이 코드의 이론적 상태 공간은 2³ × (data 타입 수) × (error 타입 수)다. 실제로 `isLoading: true, isSuccess: true`는 불가능한 조합이지만, 코드 어딘가에서 발생할 수 있다. FSM을 사용하면:

```typescript
// FSM 접근: 불가능한 상태 조합 원천 제거
type State = 'idle' | 'loading' | 'success' | 'failure';
// 이 중 동시에 두 개가 될 수 없음 → 컴파일 타임에 보장
```

### 실제 활용 사례

1. **TCP 프로토콜**: RFC 793에 FSM으로 공식 정의됨. CLOSED → SYN_SENT → ESTABLISHED → FIN_WAIT_1 → ...
2. **신호등 제어**: 도로 교통 안전에서 상태 정의가 생명과 직결
3. **결제 시스템**: 결제 중 → 인증 대기 → 승인/거절 → 환불 가능 등
4. **게임 AI**: 적 캐릭터의 순찰 → 경계 → 공격 → 도주 상태 전환
5. **CI/CD 파이프라인**: pending → running → success/failure → cancelled

## 실제 구현 예제

### 예제 1: 순수 TypeScript로 타입-안전 FSM 구현

```typescript
/**
 * 범용 FSM 구현 (의존성 없음)
 * 신호등 예제로 기본 개념 설명
 */

// 상태와 이벤트를 유니온 타입으로 정의
type TrafficLightState = 'red' | 'yellow' | 'green';
type TrafficLightEvent = 'TIMER' | 'EMERGENCY' | 'POWER_FAILURE';

interface TransitionConfig<S, E, C> {
    target: S;
    guard?: (context: C) => boolean;
    action?: (context: C) => Partial<C>;
}

class FSM<
    S extends string,
    E extends string,
    C extends Record<string, unknown> = Record<string, unknown>
> {
    private state: S;
    private context: C;
    private transitions = new Map<string, TransitionConfig<S, E, C>>();
    private onEnterActions = new Map<S, (ctx: C) => void>();
    private onExitActions  = new Map<S, (ctx: C) => void>();
    private listeners: Array<(state: S, context: C) => void> = [];

    constructor(initialState: S, initialContext: C) {
        this.state  = initialState;
        this.context = initialContext;
    }

    addTransition(from: S, event: E, config: TransitionConfig<S, E, C>): this {
        this.transitions.set(`${from}:${event}`, config);
        return this;
    }

    onEnter(state: S, action: (ctx: C) => void): this {
        this.onEnterActions.set(state, action);
        return this;
    }

    onExit(state: S, action: (ctx: C) => void): this {
        this.onExitActions.set(state, action);
        return this;
    }

    subscribe(listener: (state: S, context: C) => void): () => void {
        this.listeners.push(listener);
        return () => { this.listeners = this.listeners.filter(l => l !== listener); };
    }

    send(event: E): boolean {
        const key = `${this.state}:${event}`;
        const transition = this.transitions.get(key);

        if (!transition) return false;  // 정의되지 않은 전이 → 무시

        // Guard 확인
        if (transition.guard && !transition.guard(this.context)) {
            console.log(`[FSM] Guard 차단: ${this.state} + ${event}`);
            return false;
        }

        const prevState = this.state;

        // Exit action 실행
        this.onExitActions.get(prevState)?.(this.context);

        // 상태 전이
        this.state = transition.target;

        // Context 업데이트 (불변성 유지)
        if (transition.action) {
            this.context = { ...this.context, ...transition.action(this.context) };
        }

        // Entry action 실행
        this.onEnterActions.get(this.state)?.(this.context);

        console.log(`[FSM] ${prevState} --[${event}]--> ${this.state}`);
        this.listeners.forEach(l => l(this.state, this.context));
        return true;
    }

    getState(): S       { return this.state; }
    getContext(): C     { return { ...this.context }; }
}

// 신호등 FSM 구성
interface TrafficContext {
    cycleCount: number;
    emergencyCount: number;
}

const light = new FSM<TrafficLightState, TrafficLightEvent, TrafficContext>(
    'red',
    { cycleCount: 0, emergencyCount: 0 }
);

light
    // 일반 순환
    .addTransition('red',    'TIMER', {
        target: 'green',
        action: ctx => ({ cycleCount: ctx.cycleCount + 1 })
    })
    .addTransition('green',  'TIMER', { target: 'yellow' })
    .addTransition('yellow', 'TIMER', { target: 'red' })
    // 긴급 상황 처리 (green/yellow에서만 → red)
    .addTransition('green',  'EMERGENCY', {
        target: 'red',
        action: ctx => ({ emergencyCount: ctx.emergencyCount + 1 })
    })
    .addTransition('yellow', 'EMERGENCY', {
        target: 'red',
        action: ctx => ({ emergencyCount: ctx.emergencyCount + 1 })
    })
    // Entry/Exit 액션
    .onEnter('green',  () => console.log('  🟢 녹색: 통행 허가'))
    .onEnter('yellow', () => console.log('  🟡 황색: 정지 준비'))
    .onEnter('red',    () => console.log('  🔴 적색: 정지'))
    .onExit ('green',  () => console.log('  [green 종료]'));

// 이벤트 구독 (리액티브 업데이트)
const unsubscribe = light.subscribe((state, ctx) => {
    console.log(`  Context: 싸이클=${ctx.cycleCount}, 긴급=${ctx.emergencyCount}`);
});

// 시뮬레이션
console.log('=== 신호등 FSM 시뮬레이션 ===\n');
light.send('TIMER');      // red → green
light.send('TIMER');      // green → yellow
light.send('EMERGENCY');  // yellow → red (긴급)
light.send('TIMER');      // red → green
light.send('TIMER');      // green → yellow
light.send('TIMER');      // yellow → red

console.log('\n최종 상태:', light.getState());
console.log('최종 컨텍스트:', light.getContext());
unsubscribe();
```

### 예제 2: XState로 계층적 상태 기계 구현 (비동기 데이터 페칭)

```typescript
/**
 * XState v5를 사용한 데이터 페칭 Statechart
 * 계층적 상태 + 재시도 로직 + 타임아웃
 *
 * npm install xstate
 */
import { createMachine, assign, fromPromise, interpret } from 'xstate';

interface FetchContext {
    url: string;
    data: unknown;
    error: string | null;
    retries: number;
    maxRetries: number;
}

type FetchEvent =
    | { type: 'FETCH'; url: string }
    | { type: 'RETRY' }
    | { type: 'CANCEL' }
    | { type: 'RESET' };

const fetchMachine = createMachine({
    id: 'dataFetch',
    types: {} as {
        context: FetchContext;
        events: FetchEvent;
    },
    initial: 'idle',
    context: {
        url: '',
        data: null,
        error: null,
        retries: 0,
        maxRetries: 3,
    },

    states: {
        idle: {
            on: {
                FETCH: {
                    target: 'active',
                    actions: assign({
                        url:     ({ event }) => event.url,
                        retries: 0,
                        error:   null,
                        data:    null,
                    }),
                },
            },
        },

        // 계층적 상태: active는 loading/success/failure를 포함
        active: {
            initial: 'loading',
            // active 수준의 공통 전이 (모든 자식 상태에서 처리)
            on: {
                CANCEL: 'idle',
                RESET: {
                    target: 'idle',
                    actions: assign({ data: null, error: null, retries: 0 }),
                },
            },
            states: {
                loading: {
                    // 비동기 서비스 호출
                    invoke: {
                        src: fromPromise(async ({ input }: { input: { url: string } }) => {
                            const controller = new AbortController();
                            const timeoutId = setTimeout(() => controller.abort(), 5000); // 5초 타임아웃
                            try {
                                const res = await fetch(input.url, { signal: controller.signal });
                                if (!res.ok) throw new Error(`HTTP ${res.status}: ${res.statusText}`);
                                return await res.json();
                            } finally {
                                clearTimeout(timeoutId);
                            }
                        }),
                        input: ({ context }) => ({ url: context.url }),
                        onDone: {
                            target: 'success',
                            actions: assign({ data: ({ event }) => event.output }),
                        },
                        onError: {
                            target: 'failure',
                            actions: assign({
                                error:   ({ event }) => (event.error as Error).message,
                                retries: ({ context }) => context.retries + 1,
                            }),
                        },
                    },
                },

                success: {
                    type: 'final' as const,
                    entry: ({ context }) => {
                        console.log('✅ 데이터 로드 성공:', JSON.stringify(context.data).substring(0, 80));
                    },
                    on: {
                        FETCH: {
                            target: 'loading',
                            actions: assign({ data: null, error: null }),
                        },
                    },
                },

                failure: {
                    entry: ({ context }) => {
                        console.log(`❌ 실패 (${context.retries}/${context.maxRetries}): ${context.error}`);
                    },
                    on: {
                        RETRY: [
                            {
                                // Guard: 재시도 횟수 제한
                                guard: ({ context }) => context.retries < context.maxRetries,
                                target: 'loading',
                                actions: assign({ error: null }),
                            },
                            {
                                // maxRetries 초과 → 최종 실패
                                target: '#dataFetch.idle',
                                actions: assign({
                                    error: 'Maximum retries exceeded. Please try again later.',
                                }),
                            },
                        ],
                    },
                },
            },
        },
    },
});

// 인터프리터로 상태 기계 실행
const actor = interpret(fetchMachine);

actor.subscribe(snapshot => {
    const stateValue = JSON.stringify(snapshot.value);
    const ctx = snapshot.context;
    console.log(`[상태] ${stateValue} | retries=${ctx.retries} | error=${ctx.error || '-'}`);
});

actor.start();

// 시뮬레이션 (실제 앱에서는 UI 이벤트로 대체)
async function simulate() {
    console.log('\n=== 데이터 페칭 Statechart 시뮬레이션 ===\n');

    // 1. 성공 케이스
    actor.send({ type: 'FETCH', url: 'https://jsonplaceholder.typicode.com/todos/1' });
    await new Promise(r => setTimeout(r, 3000));

    // 2. 실패 후 재시도 케이스
    actor.send({ type: 'FETCH', url: 'https://api.invalid-url-for-demo.com/data' });
    await new Promise(r => setTimeout(r, 6000));

    actor.send({ type: 'RETRY' });
    await new Promise(r => setTimeout(r, 6000));

    // 3. 취소 케이스
    actor.send({ type: 'FETCH', url: 'https://httpbin.org/delay/10' });
    await new Promise(r => setTimeout(r, 500));
    actor.send({ type: 'CANCEL' });
    console.log('\n취소 후 상태:', JSON.stringify(actor.getSnapshot().value));

    actor.stop();
}

simulate().catch(console.error);
```

## 주의사항 및 팁

**1. 상태 폭발의 징조를 파악하라**  
단순 FSM의 상태가 급격히 늘어난다면, 두 가지 독립적인 차원이 하나의 FSM으로 합쳐진 신호다. 예를 들어 "로그인됨/로그아웃됨" × "온라인/오프라인"을 하나의 FSM으로 표현하면 4개 상태가 필요하고, 각 조합이 모두 의미를 가질 수도 있다. 이럴 때 Statechart의 병렬 상태(Parallel States)로 분리하면 N×M → N+M으로 줄어든다.

**2. 이벤트 이름은 명령형/현재형으로**  
`CLICKED`, `LOADED`, `ERRORED`처럼 과거형을 쓰면 "내부에서 일어난 일"처럼 느껴진다. FSM 이벤트는 외부에서 시스템으로 들어오는 것이므로 `CLICK`, `LOAD`, `FETCH`, `SUBMIT` 같은 명사 또는 동사 원형을 사용하라. XState 문서도 이 컨벤션을 권장한다.

**3. Context(확장 상태) 최소화**  
Context에는 상태 전이 로직에 영향을 주는 데이터만 저장하라. 예를 들어 `retryCount`는 Guard에서 사용하므로 Context에 적합하다. 하지만 순수하게 UI 표시용인 데이터(예: 상품 목록의 정렬 기준)는 컴포넌트 로컬 상태로 관리하는 것이 더 적절하다.

**4. 처리되지 않은 이벤트를 명시적으로 정의하라**  
XState는 정의되지 않은 상태-이벤트 조합을 자동으로 무시한다. 이는 편리하지만 실수를 숨길 수 있다. 명시적으로 `{ type: 'SOME_EVENT' }: undefined`로 "의도적으로 무시"를 표현하거나, TypeScript strict 모드와 함께 모든 가능한 이벤트를 exhaustive하게 처리하라.

**5. 모델 기반 테스팅 활용**  
XState의 `@xstate/graph`로 상태 기계의 모든 경로를 자동 생성하고, `@xstate/test`로 각 경로를 테스트 케이스로 만들 수 있다. 이를 통해 수동으로 엣지 케이스를 생각하지 않아도 가능한 모든 경로에 대한 테스트를 얻을 수 있다.

**6. 직렬화와 영속성**  
XState의 상태 스냅샷은 JSON으로 직렬화 가능하다. 이를 활용하면 장기 실행 워크플로우(주문 처리, 결제 흐름)를 데이터베이스에 저장하고 서버 재시작 후에도 재개할 수 있다. 다만, `fromPromise`로 생성된 액터나 스폰된 자식 액터는 직렬화되지 않으므로 별도 처리가 필요하다.

## 참고 자료

- [XState 공식 GitHub 저장소](https://github.com/statelyai/xstate)
- [ethereumjs/ethereumjs-monorepo](https://github.com/ethereumjs/ethereumjs-monorepo)
- [wiredtiger/wiredtiger](https://github.com/wiredtiger/wiredtiger)
