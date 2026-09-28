---
layout: post
title: "브라우저 렌더링 파이프라인 완전 정복: DOM·CSSOM·Layout·Paint·Compositing의 내부 동작"
date: 2026-09-28
categories: [cs, computer-science]
tags: [browser, rendering, dom, cssom, layout, reflow, repaint, compositing, performance, web]
---

웹 페이지를 열었을 때 브라우저는 HTML 텍스트를 픽셀로 변환하기 위해 복잡한 과정을 거친다. 이 과정을 **렌더링 파이프라인(Rendering Pipeline)** 또는 **Critical Rendering Path(CRP)**라고 부른다. 최신 브라우저(Chromium/Blink, Firefox/Gecko, Safari/WebKit)는 렌더링 속도를 극대화하기 위해 GPU 합성(compositing), 레이어 트리, 오프스레드 파이프라인 등 정교한 구조를 채택하고 있다. 이 글에서는 픽셀이 만들어지기까지의 전체 파이프라인을 단계별로 분해하고, 각 단계의 성능 비용과 최적화 전략을 다룬다.

---

## 왜 렌더링 파이프라인을 이해해야 하는가

60fps의 부드러운 애니메이션을 구현하려면 각 프레임을 **16.67ms(= 1000ms ÷ 60)** 안에 완성해야 한다. 하지만 JavaScript 실행, 스타일 계산, 레이아웃, 페인트, 합성을 모두 이 시간 안에 처리해야 한다. 어느 한 단계라도 병목이 생기면 프레임이 드롭(jank)된다. 파이프라인의 각 단계가 어떤 비용을 갖는지 이해하면 "어떤 CSS 속성이 비싼지", "왜 `transform`이 `left`보다 빠른지"를 논리적으로 설명할 수 있다.

---

## 전체 렌더링 파이프라인 개요

```
HTML Bytes → 토큰화 → DOM 트리
CSS Bytes  → 토큰화 → CSSOM 트리
                   ↓
             Render Tree (렌더 트리)
                   ↓
             Layout (Reflow)   ← 요소의 크기·위치 계산
                   ↓
             Paint             ← 레이어별 그리기 명령(Display List) 생성
                   ↓
             Compositing       ← GPU로 레이어 합성 → 화면 출력
```

이 파이프라인은 선형적이지 않다. 일부 단계는 건너뛸 수 있으며, 이를 잘 활용하는 것이 성능 최적화의 핵심이다.

---

## 1단계: HTML 파싱과 DOM 트리

브라우저는 서버로부터 HTML 바이트 스트림을 받아 토큰(token)으로 분리하고, 각 토큰을 노드(Node)로 변환하여 **DOM(Document Object Model) 트리**를 구성한다. 이 과정은 스트리밍 방식으로 동작하므로, HTML 전체가 도착하기 전에도 파싱이 시작된다.

### 파서 블로킹 리소스

`<script>` 태그를 만나면 HTML 파싱이 중단(parser blocking)된다. JavaScript가 DOM을 동적으로 수정할 수 있기 때문이다. `async`와 `defer` 속성은 이 블로킹을 제거한다:

```html
<!-- 파서 블로킹 (나쁜 패턴 — 크리티컬 경로 아닌 스크립트에) -->
<script src="analytics.js"></script>

<!-- async: 다운로드는 병렬, 실행 시 파싱 중단 (순서 보장 없음) -->
<script async src="analytics.js"></script>

<!-- defer: 다운로드는 병렬, DOM 완성 후 순서대로 실행 (권장) -->
<script defer src="analytics.js"></script>
```

### CSS는 렌더 블로킹

CSS는 CSSOM이 완성되기 전에 렌더 트리를 구성할 수 없으므로, `<link rel="stylesheet">`는 **렌더 블로킹** 리소스다. 반면 스크린 미디어와 무관한 CSS는 블로킹에서 제외할 수 있다:

```html
<!-- 항상 렌더 블로킹 -->
<link rel="stylesheet" href="styles.css">

<!-- 미디어 쿼리로 비블로킹화 — 프린트용 CSS는 렌더를 막지 않음 -->
<link rel="stylesheet" href="print.css" media="print">
<link rel="stylesheet" href="landscape.css" media="(orientation:landscape)">
```

---

## 2단계: CSSOM 트리

CSS 바이트는 HTML과 유사하게 토큰화되어 **CSSOM(CSS Object Model) 트리**로 변환된다. CSSOM의 중요한 특성은 **폭포수(cascade)**와 **상속(inheritance)**이다. 자식 요소의 스타일을 계산하려면 반드시 부모의 스타일이 먼저 결정되어야 하므로, CSSOM 구성은 HTML 파싱과 달리 부분적 처리가 어렵다.

```css
/* 선택자 특이성(Specificity) 계산 예 */
/* 브라우저는 모든 CSS 규칙을 수집한 뒤 각 요소에 적용될 최종 스타일을 계산 */

div { color: black; }              /* 특이성: 0,0,1 */
.container { color: blue; }        /* 특이성: 0,1,0 */
#header .title { color: red; }     /* 특이성: 1,1,0 → 최종 승자 */
div { color: green !important; }   /* !important는 별도 계층 */
```

---

## 3단계: Render Tree(렌더 트리) 구성

DOM 트리와 CSSOM 트리가 합쳐져 **렌더 트리(Render Tree)**가 만들어진다. 렌더 트리는 화면에 실제로 표시되는 노드만 포함한다.

- `display: none` 요소 → 렌더 트리에서 제외
- `<head>`, `<script>`, `<meta>` → 제외
- `visibility: hidden` 요소 → 포함(공간 차지, 보이지 않음)

---

## 4단계: Layout (Reflow)

렌더 트리가 완성되면 **레이아웃(Layout)** 또는 **리플로우(Reflow)** 단계에서 각 요소의 **정확한 위치와 크기**를 픽셀 단위로 계산한다. 레이아웃은 뷰포트 크기를 기준으로 박스 모델(Box Model)을 적용하며, 요소의 기하(geometry)를 결정한다.

### 리플로우 트리거

리플로우는 DOM의 기하적 속성이 변경될 때마다 발생하며, 이는 매우 비싼 작업이다. 자식, 형제, 부모 요소까지 연쇄적으로 재계산이 필요할 수 있기 때문이다.

```javascript
// 예제 1: 강제 동기 레이아웃(Forced Synchronous Layout) 피하기

// 나쁜 패턴: 레이아웃 무효화 후 즉시 쿼리 → 매 반복마다 리플로우 발생
const boxes = document.querySelectorAll('.box');
for (let i = 0; i < boxes.length; i++) {
  boxes[i].style.width = boxes[i].offsetWidth + 10 + 'px';
  // offsetWidth 읽기가 layout을 강제로 flush시킴 → 성능 재앙
}

// 좋은 패턴: 읽기와 쓰기를 분리 (배치 쓰기)
const widths = Array.from(boxes).map(b => b.offsetWidth);  // 1) 모두 읽기
boxes.forEach((b, i) => {
  b.style.width = widths[i] + 10 + 'px';                  // 2) 모두 쓰기
});
// 리플로우를 한 번으로 줄임

// requestAnimationFrame으로 배치 최적화
function update() {
  requestAnimationFrame(() => {
    const widths = Array.from(boxes).map(b => b.offsetWidth);
    boxes.forEach((b, i) => { b.style.width = widths[i] + 10 + 'px'; });
  });
}
```

리플로우를 유발하는 주요 속성: `width`, `height`, `top`, `left`, `margin`, `padding`, `border`, `font-size`, `position` 등 기하에 영향을 주는 모든 CSS 속성.

---

## 5단계: Paint

레이아웃이 끝나면 브라우저는 각 레이어에 대해 **그리기 명령(Drawing Commands)**을 생성한다. 이를 **페인트(Paint)** 또는 **래스터화(Rasterization)** 단계라 한다. 페인트 단계에서는 텍스트, 색상, 이미지, 그림자 같은 **시각적** 속성을 처리한다.

리플로우보다는 덜 비싸지만, 페인트도 넓은 영역에서 발생하면 병목이 된다. 페인트를 유발하는 속성: `color`, `background`, `border-radius`, `box-shadow`, `outline` 등.

```javascript
// 예제 2: Paint 비용을 줄이는 will-change 활용

// 브라우저에게 "이 요소는 곧 변할 것"을 힌트 — 별도 레이어로 격리
.animated-box {
  will-change: transform, opacity;
  /* 브라우저는 이 요소를 별도의 합성 레이어(compositing layer)로 분리
     → 이 요소의 변화가 다른 요소의 리페인트를 유발하지 않음 */
}

/* JavaScript에서 동적으로 적용 */
function promoteToLayer(el) {
  el.style.willChange = 'transform';
}

function demoteLayer(el) {
  el.style.willChange = 'auto';  // 애니메이션 끝나면 반드시 해제!
}
// 주의: 모든 요소에 will-change를 남발하면 GPU 메모리 낭비
```

---

## 6단계: Compositing (합성)

현대 브라우저의 렌더링 파이프라인에서 가장 혁신적인 부분이 **합성(Compositing)** 단계다. 브라우저는 페이지를 여러 **레이어(Layer)**로 분리하고, 각 레이어를 GPU에서 개별적으로 처리한 뒤 최종적으로 합성한다.

합성 레이어로 승격되는 조건:
- `will-change: transform` 또는 `will-change: opacity`
- `<video>`, `<canvas>`, `<iframe>` 요소
- CSS 3D 변환(`transform: translateZ(0)` 트릭)
- `position: fixed` 요소 (브라우저에 따라 다름)

### transform과 opacity가 빠른 이유

`transform`과 `opacity`는 레이아웃이나 페인트를 건너뛰고 **합성 단계만**으로 처리된다:

```css
/* 느린 방법: 리플로우 → 리페인트 → 합성 세 단계 모두 실행 */
.bad-animation {
  animation: moveLeft 1s linear;
}
@keyframes moveLeft {
  from { left: 0; }
  to { left: 200px; }
}

/* 빠른 방법: 합성 단계만 실행 — GPU에서 처리 */
.good-animation {
  animation: moveLeft 1s linear;
}
@keyframes moveLeft {
  from { transform: translateX(0); }
  to { transform: translateX(200px); }
}
/* transform, opacity: 합성 레이어 내에서 GPU가 처리 → 메인 스레드 무관 */
```

---

## Chromium의 스레드 구조

현대 Chromium은 렌더링을 여러 스레드로 분산 처리한다:

| 스레드 | 역할 |
|---|---|
| 메인 스레드 | HTML 파싱, JavaScript 실행, 스타일 계산, 레이아웃 |
| 컴포지터 스레드 | 레이어 합성, 스크롤 처리 (메인 스레드 독립) |
| 래스터 스레드(들) | 레이어 타일 래스터화 |
| GPU 프로세스 | 실제 GPU 명령 제출 |

컴포지터 스레드가 별도로 동작하기 때문에, JavaScript가 메인 스레드를 오래 점유해도 `transform`/`opacity` 애니메이션은 부드럽게 유지된다. 이것이 CSS 애니메이션을 JavaScript `setInterval`보다 선호하는 핵심 이유다.

---

## 성능 측정과 디버깅

```javascript
// 예제 3: Performance API로 렌더링 타이밍 측정
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.entryType === 'layout-shift') {
      console.log('CLS 발생:', entry.value, entry.sources);
    }
    if (entry.entryType === 'largest-contentful-paint') {
      console.log('LCP:', entry.startTime.toFixed(2), 'ms');
    }
  }
});

observer.observe({ entryTypes: ['layout-shift', 'largest-contentful-paint'] });

// Long Task 감지 (50ms 초과 작업 → 리플로우 병목 탐지)
const longTaskObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.warn('Long Task:', entry.duration.toFixed(2), 'ms', entry.attribution);
  }
});
longTaskObserver.observe({ entryTypes: ['longtask'] });
```

Chrome DevTools의 **Performance 탭**에서 각 프레임을 기록하면, 리플로우(보라색), 페인트(초록색), 합성(회색) 단계의 소요 시간을 직접 확인할 수 있다.

---

## 최적화 체크리스트

1. **레이아웃 읽기와 쓰기를 배치 처리** — `offsetWidth`, `getBoundingClientRect()` 같은 레이아웃 쿼리 후 즉시 쓰기 금지
2. **애니메이션은 `transform`과 `opacity`만 사용** — 리플로우·리페인트 없이 합성만으로 처리
3. **`will-change` 신중하게 사용** — 실제 애니메이션 직전에 추가하고 이후 제거
4. **Critical CSS 인라인화** — 첫 화면에 필요한 CSS를 `<head>`에 인라인으로 포함
5. **이미지 크기 명시** — `width`·`height` 속성으로 CLS(Cumulative Layout Shift) 방지
6. **`requestAnimationFrame` 활용** — DOM 변경을 다음 프레임으로 배치

---

## 정리

브라우저 렌더링 파이프라인은 DOM + CSSOM → 렌더 트리 → 레이아웃 → 페인트 → 합성 순으로 진행된다. 각 단계마다 트리거 조건과 비용이 다르며, CSS `transform`·`opacity` 애니메이션은 합성 단계만 거쳐 GPU에서 처리되므로 가장 효율적이다. 리플로우를 최소화하고, 합성 레이어를 올바르게 활용하면 60fps의 부드러운 UX를 달성할 수 있다.

## 참고 자료

- [Rendering Performance - web.dev](https://web.dev/articles/rendering-performance)
- [How Browsers Work - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/Performance/How_browsers_work)
- [Inside look at modern web browser - Google Chrome Developers](https://developer.chrome.com/blog/inside-browser-part3)
- [브라우저 렌더링 파이프라인 — The Browser Render Pipeline Every Frontend Engineer Should Know](https://israynotarray.com/en/misc/2025/04/24/browser-render-pipeline-for-frontend-engineers/)
