---
layout: post
title: "함수형 리액티브 프로그래밍(FRP) 완전 정복: 시간의 흐름을 함수로 다루는 선언적 패러다임"
date: 2026-09-03
categories: [cs, computer-science]
tags: [frp, functional-programming, reactive, rxjs, elm, behavior, event, signal, declarative]
---

**함수형 리액티브 프로그래밍(Functional Reactive Programming, FRP)**은 시간에 따라 변하는 값과 이벤트를 함수형 방식으로 다루는 프로그래밍 패러다임이다. 1997년 Conal Elliott과 Paul Hudak이 논문 "Functional Reactive Animation"에서 처음 제안했으며, 오늘날 Elm, RxJS, Haskell의 reflex와 reactive-banana, PureScript의 Halogen, Swift의 Combine 등 다양한 시스템의 이론적 토대를 이룬다.

FRP는 GUI 이벤트 처리, 게임 로직, 실시간 데이터 시각화 등 **시간과 상태가 복잡하게 얽힌 문제**를 선언적이고 합성 가능한 방식으로 해결한다. "무엇을 어떻게 처리하라"가 아니라 "값들 사이의 관계가 무엇인가"를 코드로 표현하는 것이 FRP의 핵심이다.

## 왜 FRP가 필요한가?

전통적인 명령형 이벤트 처리 방식은 다음 문제를 낳는다:

### 콜백 지옥과 분산된 상태

```javascript
// 전통적 콜백 방식: 검색창 구현
let debounceTimer;
let lastQuery = '';
let isLoading = false;  // 상태 1
let results = [];        // 상태 2
let currentPage = 1;    // 상태 3

document.getElementById('search').addEventListener('input', function(e) {
    clearTimeout(debounceTimer);
    debounceTimer = setTimeout(function() {
        const query = e.target.value;
        if (query === lastQuery) return;
        
        lastQuery = query;
        isLoading = true;
        updateLoadingUI();
        
        fetchResults(query, function(data) {
            if (query !== lastQuery) return;  // 경쟁 조건 방지 처리
            results = data;
            isLoading = false;
            currentPage = 1;
            renderResults(results, function() {
                updatePagination(data.total, function() {
                    updateURL(query, function() {
                        // 점점 깊어지는 콜백...
                    });
                });
            });
        }, function(err) {
            isLoading = false;
            showError(err);  // 오류 처리도 분산됨
        });
    }, 300);
});
```

**문제점:**
- 상태가 여러 변수에 분산되어 추적이 어렵다.
- 경쟁 조건(race condition) 방지 코드가 비즈니스 로직과 뒤섞인다.
- 이벤트 간 의존관계가 암묵적이다.
- 테스트가 어렵다 (사이드 이펙트와 타이밍 의존성).

FRP는 이 문제를 **"값 간의 관계를 선언"**하는 방식으로 해결한다.

## FRP의 두 핵심 추상화

### 1. Behavior — 연속적 시간 함수

**Behavior\<A\>**는 **모든 시점 t에서 값 A를 가지는 연속 함수**다.

```
Behavior<A> = Time → A

예시:
- 마우스 X 좌표: Behavior<Int>     -- 항상 현재 X 값이 존재
- 현재 시각: Behavior<DateTime>    -- 1초도 빠짐없이 현재 시간
- 볼륨 슬라이더 값: Behavior<Float> -- 항상 0.0~1.0 사이 값
- 사용자 이름: Behavior<String>     -- 로그인 상태의 이름

수학적 표현:
mouseX :: Behavior Int
mouseX t = getCurrentMouseX()  -- 시점 t에서 마우스 X 좌표
```

### 2. Event — 이산적 발생 스트림

**Event\<A\>**는 **특정 시점에 발생하는 값 A의 이산 스트림**이다.

```
Event<A> = [(Time, A)]

예시:
- 마우스 클릭: Event<(Int, Int)>  -- 클릭할 때마다 좌표 발생
- 키보드 입력: Event<Key>          -- 키 누를 때마다 발생
- HTTP 응답: Event<Response>       -- 요청 완료 시 발생
- 1초 타이머: Event<()>           -- 매 초 발생

시각적:
time:  ──────────────────────────────────►
click: ──●─────────●───────────────●──── 
         (x1,y1)   (x2,y2)         (x3,y3)
```

### 핵심 조합자 (Combinators)

FRP의 강력함은 Behavior와 Event를 변환·조합하는 순수 함수들에서 나온다:

```haskell
-- Haskell 타입 시그니처로 보는 핵심 조합자

-- 1. map: 값 변환
fmap  :: (a → b) → Event a   → Event b
fmap  :: (a → b) → Behavior a → Behavior b

-- 2. filter: 조건 필터
filter :: (a → Bool) → Event a → Event a

-- 3. hold: 마지막 이벤트 값을 Behavior로
--    "Event를 Behavior로 승격"
hold :: a → Event a → Behavior a

-- 4. sample: Event 발생 시점에 Behavior 값을 캡처
--    "두 스트림의 교차점"
sample :: Behavior a → Event b → Event a
(<@>) :: Behavior (a → b) → Event a → Event b

-- 5. merge: 두 Event 스트림 병합
mergeWith :: (a → a → a) → Event a → Event a → Event a

-- 6. switch: 이벤트로 Behavior를 동적 전환
switch :: Behavior (Behavior a) → Behavior a

-- 7. stepper / accumulate: 이벤트로 누적 상태 만들기
accumE :: a → Event (a → a) → Event a
```

## 코드 예제 1: Haskell reactive-banana로 구현하는 진짜 FRP

```haskell
{-# LANGUAGE RecursiveDo #-}

import Reactive.Banana
import Reactive.Banana.Frameworks

-- ── 예제: 온도 단위 변환기 ──
-- Celsius ↔ Fahrenheit 양방향 동기화

temperatureConverter :: MomentIO ()
temperatureConverter = do
    -- 이벤트 소스 (UI 입력 → Haskell Event)
    (celsiusInputE, fireCelsius)    <- newEvent
    (fahrenheitInputE, fireFahr)    <- newEvent
    (convertButtonE, fireConvert)   <- newEvent
    
    -- ── Behavior 구성 ──
    -- 섭씨 입력값 (초기값 0.0)
    celsiusB :: Behavior Double
            <- stepper 0.0 celsiusInputE
    
    -- 화씨 입력값
    fahrenheitB :: Behavior Double
               <- stepper 32.0 fahrenheitInputE
    
    -- 변환 버튼 클릭 시 현재 섭씨 값을 캡처
    let convertedFahrE :: Event Double
        convertedFahrE = (*9/5) . (+32) <$> (celsiusB <@ convertButtonE)
        --                               ^^^^^^^^^^^^^^^^^^^
        -- <@ : Event 발생 시점에 Behavior 샘플링
        -- 버튼 클릭이라는 Event가 섭씨 Behavior를 트리거
    
    -- 역방향: 화씨 → 섭씨
    let convertedCelE :: Event Double
        convertedCelE = (\f → (f - 32) * 5/9) <$> 
                        (fahrenheitB <@ convertButtonE)
    
    -- 결과 출력 (사이드 이펙트는 여기에만!)
    reactimate $ fmap (\f → putStrLn $ "화씨: " ++ show f) convertedFahrE
    reactimate $ fmap (\c → putStrLn $ "섭씨: " ++ show c) convertedCelE

-- ── 예제: 카운터 (순수 상태 누적) ──
counterApp :: MomentIO ()
counterApp = do
    (incE, fireInc) <- newEvent    -- 증가 버튼 이벤트
    (decE, fireDec) <- newEvent    -- 감소 버튼 이벤트
    (resetE, fireReset) <- newEvent -- 리셋 버튼 이벤트
    
    -- 모든 이벤트를 상태 변환 함수(Int → Int)로 통합
    let changes :: Event (Int → Int)
        changes = mergeWith (.)  -- 동시 이벤트 합성
                    (const (+1) <$> incE)      -- 증가: +1
                    (mergeWith (.)
                        (const (subtract 1) <$> decE)  -- 감소: -1
                        (const (const 0) <$> resetE))  -- 리셋: → 0
    
    -- accumB: 이벤트 스트림으로 누적 Behavior 생성
    -- 글리치 없음: 단일 원자적 업데이트 보장
    counterB :: Behavior Int <- accumB 0 changes
    
    -- Behavior 변화를 화면에 반영
    changes `observe` counterB $ \count →
        putStrLn $ "카운터: " ++ show count
```

reactive-banana에서 모든 상태 변화는 순수 함수(`Int → Int`)로 표현되며, 사이드 이펙트는 `reactimate`에만 격리된다. 이것이 FRP의 테스트 용이성의 핵심이다.

## 코드 예제 2: RxJS로 구현하는 실용적 FRP

RxJS는 FRP의 핵심 아이디어를 JavaScript에 가져온 라이브러리다. `Observable`이 Event와 Behavior를 통합한 개념으로 동작한다.

```typescript
import { fromEvent, interval, combineLatest, merge, NEVER, of } from 'rxjs';
import {
    map, filter, debounceTime, switchMap, distinctUntilChanged,
    startWith, scan, shareReplay, catchError, retry, takeUntil,
    withLatestFrom, bufferTime, throttleTime, auditTime
} from 'rxjs/operators';

// ── 예제 1: 실시간 검색 — FRP 방식 ──
function setupSearch(searchEl: HTMLInputElement, resultsEl: HTMLElement) {
    const searchQuery$ = fromEvent(searchEl, 'input').pipe(
        map((e: InputEvent) => (e.target as HTMLInputElement).value),
        debounceTime(300),           // 300ms 입력 중단 후 방출
        distinctUntilChanged(),      // 같은 값 중복 방출 제거
        filter(q => q.length > 1),   // 2자 이상만
        shareReplay(1)               // 멀티캐스트: 여러 구독자 공유
    );
    
    // 검색 결과 스트림 (이전 요청 자동 취소)
    const results$ = searchQuery$.pipe(
        switchMap(query =>           // 새 쿼리 오면 이전 요청 취소!
            fetch(`/api/search?q=${encodeURIComponent(query)}`)
                .then(r => r.json())
                .then(data => ({ status: 'success' as const, data, query }))
                .catch(err => ({ status: 'error' as const, err, query }))
        ),
        startWith({ status: 'idle' as const })
    );
    
    // UI 업데이트 (구독 = 사이드 이펙트)
    results$.subscribe(result => {
        if (result.status === 'idle') {
            resultsEl.innerHTML = '<p>검색어를 입력하세요</p>';
        } else if (result.status === 'success') {
            resultsEl.innerHTML = result.data
                .map((item: any) => `<li>${item.title}</li>`)
                .join('');
        } else {
            resultsEl.innerHTML = `<p>오류: ${result.err.message}</p>`;
        }
    });
}

// ── 예제 2: 게임 루프 — Behavior + Event 결합 ──
interface GameState {
    x: number; y: number; score: number; lives: number;
}

function createGameLoop(): void {
    const TICK_MS = 16;  // ~60fps
    
    // 키 입력 상태 (Behavior처럼 동작)
    const keysDown = new Set<string>();
    fromEvent<KeyboardEvent>(document, 'keydown')
        .subscribe(e => keysDown.add(e.key));
    fromEvent<KeyboardEvent>(document, 'keyup')
        .subscribe(e => keysDown.delete(e.key));
    
    // 게임 틱 (Event)
    const tick$ = interval(TICK_MS);
    
    // 상태 누적 (scan = FP의 fold/accumulate)
    const gameState$ = tick$.pipe(
        scan((state: GameState, _tick: number) => {
            // 순수 함수: 이전 상태 + 현재 입력 → 새 상태
            const speed = 3;
            let { x, y, score, lives } = state;
            
            if (keysDown.has('ArrowLeft'))  x = Math.max(0, x - speed);
            if (keysDown.has('ArrowRight')) x = Math.min(800, x + speed);
            if (keysDown.has('ArrowUp'))    y = Math.max(0, y - speed);
            if (keysDown.has('ArrowDown'))  y = Math.min(600, y + speed);
            
            return { x, y, score, lives };
        }, { x: 400, y: 300, score: 0, lives: 3 }),
        shareReplay(1)
    );
    
    // 렌더링 (사이드 이펙트)
    gameState$.subscribe(state => render(state));
}

// ── 예제 3: 연결 상태에 따른 데이터 소스 동적 전환 ──
// switchMap을 활용한 Behavior-like 전환

const isOnline$ = merge(
    fromEvent(window, 'online').pipe(map(() => true)),
    fromEvent(window, 'offline').pipe(map(() => false))
).pipe(startWith(navigator.onLine), distinctUntilChanged());

// 온라인: WebSocket 실시간, 오프라인: 폴링
const data$ = isOnline$.pipe(
    switchMap(online =>
        online
            ? createWebSocketObservable('wss://api.example.com/live')
            : interval(5000).pipe(
                switchMap(() => fetch('/api/data').then(r => r.json()))
              )
    )
);

data$.subscribe(data => updateDashboard(data));

// ── 마블 다이어그램으로 이해하는 핵심 연산자 ──
/*
debounceTime(300ms):
input:  --a-b--c---------d--e-f----|
output: --------c---------------f--|
                ↑ 300ms 후         ↑

switchMap (이전 구독 취소):
input:  ---a---------b------|
          \           \
           A--A--A--✗  B--B--B--|
output: ---A--A--A---B--B--B--|
                   ↑ a의 구독 취소

scan (누적):
events: --1--2--3--4--|
func:   (+)
acc:    0
output: --1--3--6--10--|

combineLatest (항상 최신 조합):
A:      1---------2------3--|
B:      ---10------20-------|
output: ---[1,10]-[2,10]-[2,20]-[3,20]--|
*/
```

## Elm 아키텍처: FRP를 실용화한 결정판

Elm은 FRP에서 영감을 받아 만들어진 언어로, 웹 프론트엔드에 특화된 아키텍처를 제공한다:

```elm
-- Elm 아키텍처: Model-Update-View
-- Model = 상태 (Behavior에 해당)
-- Msg = 이벤트 타입 (Event에 해당)
-- update = 순수 상태 전이 함수

module Search exposing (..)

import Html exposing (..)
import Html.Events exposing (..)
import Http

-- 상태 (현재 Behavior 값)
type alias Model =
    { query   : String
    , results : List String
    , status  : Status
    }

type Status = Idle | Loading | Loaded | Failed String

-- 이벤트 (발생 가능한 Event 집합)
type Msg
    = QueryChanged String    -- 입력 변경
    | SearchClicked          -- 버튼 클릭
    | GotResults (Result Http.Error (List String))  -- 응답 수신

-- 상태 전이 함수 (순수 함수! 사이드 이펙트 없음)
update : Msg → Model → ( Model, Cmd Msg )
update msg model =
    case msg of
        QueryChanged q →
            ( { model | query = q, status = Idle }
            , Cmd.none
            )
        
        SearchClicked →
            ( { model | status = Loading }
            , searchApi model.query  -- Cmd: 사이드 이펙트를 런타임에 위임
            )
        
        GotResults (Ok results) →
            ( { model | results = results, status = Loaded }
            , Cmd.none
            )
        
        GotResults (Err err) →
            ( { model | status = Failed (errorToString err) }
            , Cmd.none
            )

-- 뷰 (Model → Html Msg: 순수 렌더링 함수)
view : Model → Html Msg
view model =
    div []
        [ input
            [ onInput QueryChanged
            , value model.query
            , placeholder "검색어 입력..."
            ]
            []
        , button
            [ onClick SearchClicked
            , disabled (model.status == Loading)
            ]
            [ text (if model.status == Loading then "검색 중..." else "검색") ]
        , case model.status of
            Failed err → p [ class "error" ] [ text err ]
            Loaded     → ul [] (List.map (\r → li [] [ text r ]) model.results)
            _          → text ""
        ]

-- API 호출 (Cmd = 선언적 사이드 이펙트)
searchApi : String → Cmd Msg
searchApi query =
    Http.get
        { url = "/api/search?q=" ++ query
        , expect = Http.expectJson GotResults (Json.Decode.list Json.Decode.string)
        }
```

Elm 아키텍처의 핵심은 사이드 이펙트의 **완전한 분리**다. `update` 함수는 `(Model, Cmd Msg)`를 반환하는데, `Cmd`는 "이런 사이드 이펙트를 해주세요"라는 선언이지 실제 실행이 아니다. 실행은 Elm 런타임이 담당한다. 이로 인해 Elm 코드에서 런타임 예외(null pointer, undefined)는 구조적으로 불가능하다.

## 전통 FRP vs 실용적 FRP: 개념 차이

| 특성 | 전통 FRP (Conal Elliott) | 실용적 FRP (RxJS/Elm) |
|------|------------------------|---------------------|
| 시간 모델 | 연속 실수 시간 (real-valued time) | 이산 이벤트 순서 |
| Behavior 지원 | 진정한 연속 Behavior | 이산 이벤트로 근사 (BehaviorSubject 등) |
| 글리치 | 수학적으로 없음 | combineLatest에서 발생 가능 |
| 메모리 관리 | 자동 (FRP 런타임) | 수동 구독 해제 필요 |
| 학습 곡선 | 매우 높음 (Haskell 필요) | 중간 |
| 실용성 | Haskell/PureScript 생태계 | JS, Swift, Kotlin 광범위 지원 |

## 주의사항과 팁

### 1. 구독 해제 — 메모리 누수 방지

```typescript
// Angular/TypeScript 패턴
class SearchComponent implements OnInit, OnDestroy {
    private destroy$ = new Subject<void>();
    
    ngOnInit() {
        // takeUntil로 컴포넌트 소멸 시 자동 구독 해제
        searchResults$.pipe(
            takeUntil(this.destroy$)
        ).subscribe(results => this.displayResults(results));
    }
    
    ngOnDestroy() {
        this.destroy$.next();    // 모든 takeUntil 트리거
        this.destroy$.complete();
    }
}

// React hooks 패턴
function useObservable<T>(obs$: Observable<T>) {
    const [value, setValue] = useState<T>();
    useEffect(() => {
        const subscription = obs$.subscribe(setValue);
        return () => subscription.unsubscribe();  // cleanup
    }, [obs$]);
    return value;
}
```

### 2. 글리치(Glitch) 주의

```typescript
const a$ = new BehaviorSubject(1);
const b$ = a$.pipe(map(a => a * 2));    // 항상 a의 2배
const c$ = combineLatest([a$, b$]).pipe( // 항상 a + b = 3a
    map(([a, b]) => a + b)
);

// a가 1→2로 변할 때:
// 이상적: c = 2 + 4 = 6 (한 번 업데이트)
// 실제 (글리치 가능): c = 2 + 2 = 4 (b 아직 업데이트 전), 이후 c = 2 + 4 = 6
// → 중간에 잘못된 값 4가 방출될 수 있음

// 해결: zip (두 스트림의 같은 인덱스 값을 대응)
const c$ = zip(a$, b$).pipe(map(([a, b]) => a + b));
// 또는 파생 스트림을 직접 계산
const c$ = a$.pipe(map(a => a + a * 2));  // 3a
```

### 3. 빠른 생산자 처리 전략 (Back-pressure)

```typescript
const fastProducer$ = interval(1);  // 1ms마다 이벤트

// 전략 1: 버퍼링 후 배치 처리
fastProducer$.pipe(bufferTime(100)).subscribe(batch => processBatch(batch));

// 전략 2: 최신 값만 처리 (throttle)
fastProducer$.pipe(throttleTime(100)).subscribe(processOne);

// 전략 3: 주기적 샘플링 (sample)
fastProducer$.pipe(auditTime(100)).subscribe(processOne);

// 전략 4: 처리가 완료될 때까지 대기 (concatMap)
fastProducer$.pipe(
    concatMap(value => from(asyncProcess(value)))
).subscribe();
```

### 4. Hot vs Cold Observable

```typescript
// Cold Observable: 구독할 때마다 새 실행 (각 구독자가 독립적 스트림)
const cold$ = new Observable(subscriber => {
    subscriber.next(Math.random());  // 구독마다 다른 값!
});

// Hot Observable: 이미 실행 중, 구독자가 현재부터 수신
const hot$ = fromEvent(document, 'click');  // 클릭은 구독과 관계없이 발생

// shareReplay: Cold를 Hot으로 + 최신 N개 값 캐시
const shared$ = cold$.pipe(shareReplay(1));
// 첫 번째 구독이 실행을 시작, 이후 구독자는 캐시된 값 즉시 수신
```

## 결론

FRP는 시간과 상태를 다루는 복잡성을 **"값들 사이의 관계 선언"**으로 압축한다. 콜백 지옥, 분산된 뮤터블 상태, 경쟁 조건과 같은 문제들이 Behavior와 Event라는 두 추상화와 순수 함수 조합자들로 자연스럽게 해결된다. 전통적 FRP(reflex, reactive-banana)는 이론적 순수성을, RxJS와 Elm은 실용성을 각각 선택했지만 모두 같은 철학 위에 선다: **프로그램이란 시간의 함수다.** UI 이벤트 처리에서 벗어나 실시간 데이터 처리, 게임 로직, IoT 센서 스트림까지 시간적 데이터를 다루는 모든 도메인에서 FRP는 코드의 표현력과 정확성을 동시에 높여준다.

## 참고 자료
- [Conal Elliott & Paul Hudak — Functional Reactive Animation (1997)](http://conal.net/papers/icfp97/)
- [Elm 공식 사이트 — 웹을 위한 FRP 언어](https://elm-lang.org/)
- [RxJS 공식 문서 — Reactive Extensions for JavaScript](https://rxjs.dev/guide/overview)
- [reactive-banana Haskell 라이브러리](https://hackage.haskell.org/package/reactive-banana)
