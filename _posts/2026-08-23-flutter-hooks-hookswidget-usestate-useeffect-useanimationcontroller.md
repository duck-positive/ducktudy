---
layout: post
title: "Flutter Hooks 심화: HookWidget·useState·useEffect·useAnimationController로 위젯을 함수형으로 혁신하기"
date: 2026-08-23
categories: [android, flutter]
tags: [flutter, hooks, flutter_hooks, useState, useEffect, useAnimationController, HookWidget, 상태관리, 함수형프로그래밍]
---

## 1. Flutter Hooks란 무엇인가

Flutter Hooks는 React의 Hooks 패러다임을 Flutter에 이식한 라이브러리(`flutter_hooks`)입니다. `StatefulWidget`이 독점하던 생명주기 로직(`initState`, `dispose`, `didUpdateWidget`)을 **훅(hook)**이라는 재사용 가능한 함수 단위로 분리하여, 더 간결하고 조합 가능한 위젯 코드를 작성할 수 있게 해줍니다.

핵심 컴포넌트는 세 가지입니다.

- **HookWidget**: `StatelessWidget`처럼 `build` 메서드만 구현하지만, 내부에서 훅을 자유롭게 호출할 수 있습니다.
- **Hook\<T\>**: 상태와 생명주기를 캡슐화하는 기본 단위입니다. `StatelessWidget`과 유사하지만 Element와 연결되지 않습니다.
- **HookState\<R, H\>**: 훅의 실제 상태와 생명주기 메서드(`initHook`, `dispose`, `build`)를 구현합니다.

현재(2026년 기준) flutter_hooks 최신 안정 버전은 **0.21.3+1**이며, Android·iOS·Web·Desktop 모두 지원합니다.

---

## 2. 왜 Flutter Hooks가 필요한가

### StatefulWidget의 구조적 한계

`AnimationController`를 사용하는 전형적인 StatefulWidget을 살펴봅시다.

```dart
class TraditionalAnimatedWidget extends StatefulWidget {
  const TraditionalAnimatedWidget({super.key});

  @override
  State<TraditionalAnimatedWidget> createState() =>
      _TraditionalAnimatedWidgetState();
}

class _TraditionalAnimatedWidgetState
    extends State<TraditionalAnimatedWidget>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;
  late final Animation<double> _fadeAnimation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 800),
    )..repeat(reverse: true);
    _fadeAnimation = Tween<double>(begin: 0.3, end: 1.0).animate(
      CurvedAnimation(parent: _controller, curve: Curves.easeInOut),
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return FadeTransition(
      opacity: _fadeAnimation,
      child: const FlutterLogo(size: 100),
    );
  }
}
```

이 코드에는 세 가지 문제가 있습니다.

1. **보일러플레이트 과다**: `initState`와 `dispose`에 동일한 리소스의 생성·해제 로직이 흩어져 있습니다.
2. **재사용 불가**: AnimationController 패턴을 다른 위젯에 쓰려면 코드를 통째로 복사해야 합니다.
3. **믹스인 제약**: `SingleTickerProviderStateMixin`은 클래스당 한 번만 사용 가능합니다. 컨트롤러가 두 개 필요하면 `TickerProviderStateMixin`으로 교체해야 하는 등 내부 구현 변경이 발생합니다.

### Hooks로 해결하기

Flutter Hooks는 `useAnimationController()` 하나로 위의 모든 문제를 해결합니다.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';

class HookAnimatedWidget extends HookWidget {
  const HookAnimatedWidget({super.key});

  @override
  Widget build(BuildContext context) {
    final controller = useAnimationController(
      duration: const Duration(milliseconds: 800),
    )..repeat(reverse: true);

    final fadeAnimation = Tween<double>(begin: 0.3, end: 1.0).animate(
      CurvedAnimation(parent: controller, curve: Curves.easeInOut),
    );

    return FadeTransition(
      opacity: fadeAnimation,
      child: const FlutterLogo(size: 100),
    );
  }
}
```

코드가 절반으로 줄었습니다. `SingleTickerProviderStateMixin`도, `initState`도, `dispose`도 없습니다. `useAnimationController`가 내부적으로 티커 프로바이더를 생성하고, 위젯이 트리에서 제거될 때 자동으로 `dispose`를 호출합니다.

---

## 3. 핵심 훅 상세 분석

### 3.1 useState

`useState<T>(T initialData)`는 반응형 변수를 생성합니다. 반환값은 `ValueNotifier<T>` 객체이며, `.value`를 변경하는 것만으로 재빌드가 트리거됩니다. `setState(() {...})` 호출이 불필요합니다.

```dart
class CounterWithHooks extends HookWidget {
  const CounterWithHooks({super.key});

  @override
  Widget build(BuildContext context) {
    final count = useState(0);
    final isLoading = useState(false);

    Future<void> handleIncrement() async {
      isLoading.value = true;
      await Future<void>.delayed(const Duration(milliseconds: 300));
      count.value++;
      isLoading.value = false;
    }

    return Scaffold(
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            AnimatedSwitcher(
              duration: const Duration(milliseconds: 200),
              child: isLoading.value
                  ? const SizedBox(
                      key: ValueKey('loading'),
                      width: 24,
                      height: 24,
                      child: CircularProgressIndicator(strokeWidth: 2),
                    )
                  : Text(
                      '카운트: ${count.value}',
                      key: ValueKey(count.value),
                      style: Theme.of(context).textTheme.headlineMedium,
                    ),
            ),
            const SizedBox(height: 16),
            FilledButton(
              onPressed: isLoading.value ? null : handleIncrement,
              child: const Text('증가'),
            ),
          ],
        ),
      ),
    );
  }
}
```

위처럼 여러 `useState`를 같은 `build` 메서드 안에서 독립적으로 선언할 수 있고, 각각은 서로의 리빌드에 영향을 주지 않습니다.

### 3.2 useEffect

`useEffect(VoidCallback effect, [List<Object?>? keys])`는 사이드 이펙트를 관리합니다. `initState`와 `dispose` 로직을 하나의 클로저에 응집시킨다고 이해하면 됩니다.

| keys 값 | 실행 시점 |
|---------|---------|
| null | 매 빌드마다 실행 |
| `[]` (빈 리스트) | 최초 한 번만 실행 |
| `[dep1, dep2]` | dep이 변경될 때마다 재실행 |

`effect`가 반환하는 클로저는 **정리 함수(cleanup)**로, 다음 effect 실행 직전 또는 위젯 dispose 시 호출됩니다.

```dart
class StreamSubscriptionExample extends HookWidget {
  final Stream<String> messageStream;
  const StreamSubscriptionExample({super.key, required this.messageStream});

  @override
  Widget build(BuildContext context) {
    final latestMessage = useState<String>('메시지 없음');
    final messageCount = useState(0);

    useEffect(
      () {
        final subscription = messageStream.listen((message) {
          latestMessage.value = message;
          messageCount.value++;
        });
        // 정리 함수: 위젯 dispose 또는 messageStream 변경 시 구독 해제
        return subscription.cancel;
      },
      [messageStream], // messageStream 참조가 바뀔 때만 재구독
    );

    return Padding(
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text('최신 메시지: ${latestMessage.value}'),
          const SizedBox(height: 8),
          Text('총 수신 횟수: ${messageCount.value}회'),
        ],
      ),
    );
  }
}
```

### 3.3 커스텀 훅: useDebouncedValue

훅의 진정한 가치는 **재사용 가능한 커스텀 훅**을 만들 수 있다는 점입니다. 아래는 검색창의 디바운싱 로직을 커스텀 훅으로 추출한 예시입니다.

```dart
// lib/hooks/use_debounced_value.dart
import 'dart:async';
import 'package:flutter_hooks/flutter_hooks.dart';

T useDebouncedValue<T>(T value, Duration delay) {
  final debouncedValue = useState(value);

  useEffect(
    () {
      final timer = Timer(delay, () {
        debouncedValue.value = value;
      });
      return timer.cancel;
    },
    [value, delay],
  );

  return debouncedValue.value;
}

// 사용 예시
class SearchPage extends HookWidget {
  const SearchPage({super.key});

  @override
  Widget build(BuildContext context) {
    final query = useState('');
    final debouncedQuery = useDebouncedValue(
      query.value,
      const Duration(milliseconds: 500),
    );
    final results = useState<List<String>>([]);

    useEffect(
      () {
        if (debouncedQuery.isEmpty) {
          results.value = [];
          return null;
        }
        // 실제 앱에서는 API 호출로 대체
        results.value = ['결과: $debouncedQuery', '관련: ${debouncedQuery}s'];
        return null;
      },
      [debouncedQuery],
    );

    return Column(
      children: [
        Padding(
          padding: const EdgeInsets.all(16),
          child: TextField(
            onChanged: (v) => query.value = v,
            decoration: const InputDecoration(
              hintText: '검색어 (500ms 디바운스 적용)',
              prefixIcon: Icon(Icons.search),
            ),
          ),
        ),
        Expanded(
          child: ListView.builder(
            itemCount: results.value.length,
            itemBuilder: (_, i) => ListTile(title: Text(results.value[i])),
          ),
        ),
      ],
    );
  }
}
```

`useDebouncedValue`를 한 번 정의하면 프로젝트 어디서든 한 줄로 디바운싱을 적용할 수 있습니다. StatefulWidget 기반이었다면 믹스인이나 중복 코드가 불가피했던 로직입니다.

---

## 4. 고급 패턴: useReducer로 복잡한 상태 관리

상태 전환이 복잡해지면 `useReducer`를 사용하여 명시적인 이벤트 기반 업데이트를 구현할 수 있습니다. BLoC의 간소화 버전처럼 동작합니다.

```dart
import 'dart:async';
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';

// 상태 정의
class TimerState {
  final int seconds;
  final bool isRunning;
  const TimerState({required this.seconds, required this.isRunning});
}

// 액션 정의 (sealed class, Dart 3+)
sealed class TimerAction {}
class StartTimer extends TimerAction {}
class StopTimer extends TimerAction {}
class TickTimer extends TimerAction {}
class ResetTimer extends TimerAction {}

// 순수 리듀서 함수
TimerState _timerReducer(TimerState state, TimerAction action) {
  return switch (action) {
    StartTimer() => TimerState(seconds: state.seconds, isRunning: true),
    StopTimer() => TimerState(seconds: state.seconds, isRunning: false),
    TickTimer() => TimerState(
        seconds: state.seconds + 1,
        isRunning: state.isRunning,
      ),
    ResetTimer() => const TimerState(seconds: 0, isRunning: false),
  };
}

class TimerWidget extends HookWidget {
  const TimerWidget({super.key});

  @override
  Widget build(BuildContext context) {
    final store = useReducer<TimerState, TimerAction>(
      _timerReducer,
      initialState: const TimerState(seconds: 0, isRunning: false),
    );

    useEffect(
      () {
        if (!store.state.isRunning) return null;
        final timer = Timer.periodic(
          const Duration(seconds: 1),
          (_) => store.dispatch(TickTimer()),
        );
        return timer.cancel;
      },
      [store.state.isRunning],
    );

    final minutes = store.state.seconds ~/ 60;
    final secs = store.state.seconds % 60;

    return Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        Text(
          '${minutes.toString().padLeft(2, '0')}:${secs.toString().padLeft(2, '0')}',
          style: const TextStyle(fontSize: 64, fontWeight: FontWeight.w200),
        ),
        const SizedBox(height: 24),
        Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            FilledButton.icon(
              icon: Icon(
                store.state.isRunning ? Icons.pause : Icons.play_arrow,
              ),
              label: Text(store.state.isRunning ? '일시정지' : '시작'),
              onPressed: () => store.dispatch(
                store.state.isRunning ? StopTimer() : StartTimer(),
              ),
            ),
            const SizedBox(width: 12),
            OutlinedButton.icon(
              icon: const Icon(Icons.refresh),
              label: const Text('초기화'),
              onPressed: () => store.dispatch(ResetTimer()),
            ),
          ],
        ),
      ],
    );
  }
}
```

`useReducer`를 사용하면 상태 전환 로직이 순수 함수(`_timerReducer`)로 분리되어 단위 테스트가 매우 쉬워집니다.

---

## 5. hooks_riverpod과의 시너지

`hooks_riverpod` 패키지를 사용하면 `HookConsumerWidget`에서 훅과 Riverpod 프로바이더를 동시에 활용할 수 있습니다. 로컬 UI 상태는 훅으로, 전역 비즈니스 로직은 Riverpod으로 관리하는 이상적인 구조가 완성됩니다.

```dart
// pubspec.yaml
// dependencies:
//   flutter_riverpod: ^2.x
//   hooks_riverpod: ^2.x
//   flutter_hooks: ^0.21.0

import 'package:flutter_hooks/flutter_hooks.dart';
import 'package:hooks_riverpod/hooks_riverpod.dart';

class ProductListPage extends HookConsumerWidget {
  const ProductListPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // 로컬 UI 상태 — 훅
    final searchQuery = useState('');
    final debouncedQuery = useDebouncedValue(
      searchQuery.value,
      const Duration(milliseconds: 400),
    );

    // 전역 비즈니스 로직 — Riverpod
    final productsAsync = ref.watch(
      productSearchProvider(debouncedQuery),
    );

    return Scaffold(
      appBar: AppBar(
        title: TextField(
          onChanged: (v) => searchQuery.value = v,
          decoration: const InputDecoration(
            hintText: '상품 검색...',
            border: InputBorder.none,
          ),
        ),
      ),
      body: productsAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(child: Text('오류가 발생했습니다: $e')),
        data: (products) => ListView.builder(
          itemCount: products.length,
          itemBuilder: (_, i) => ListTile(
            title: Text(products[i].name),
            trailing: Text('${products[i].price}원'),
          ),
        ),
      ),
    );
  }
}
```

---

## 6. 주의사항과 실전 팁

### 6.1 훅 호출 규칙 (Rules of Hooks)

React와 동일한 규칙이 Flutter Hooks에도 적용됩니다.

**규칙 1**: 항상 `build` 메서드 최상위 레벨에서 호출합니다. 조건문, 반복문, 콜백 내부에서 호출하면 훅 순서가 바뀌어 런타임 오류가 발생합니다.

```dart
// ❌ 잘못된 사용 — 조건부 훅 호출
Widget build(BuildContext context) {
  if (widget.showCounter) {
    final count = useState(0); // 빌드마다 순서가 달라져 오류 발생
  }
  ...
}

// ✅ 올바른 사용 — 항상 최상위에서 선언
Widget build(BuildContext context) {
  final count = useState(0);
  if (widget.showCounter) {
    return Text('${count.value}');
  }
  return const SizedBox.shrink();
}
```

**규칙 2**: `HookWidget`, `HookConsumerWidget`, `HookBuilder` 내부에서만 훅을 호출합니다.

### 6.2 useEffect keys의 함정

`keys`에 전달되는 객체는 `==` 연산자로 비교됩니다. 매 빌드마다 새로 생성되는 컬렉션 객체를 키로 전달하면 의도치 않게 매 빌드마다 이펙트가 재실행됩니다.

```dart
// ❌ 매 빌드마다 새 리스트가 생성됨 → 매번 재구독
useEffect(() { ... }, [widget.ids.toList()]);

// ✅ 안정적인 프리미티브 값이나 참조 사용
useEffect(() { ... }, [widget.userId]);

// ✅ 컬렉션이 필요하면 useMemoized로 캐싱
final stableIds = useMemoized(() => widget.ids.toSet(), [widget.ids]);
useEffect(() { ... }, [stableIds]);
```

### 6.3 StatefulHookWidget을 써야 하는 경우

대부분의 상황에서 `HookWidget`으로 충분하지만, 다음 상황에서는 `StatefulHookWidget`을 고려합니다.

- `didUpdateWidget`: 부모에서 전달된 위젯 속성이 변경될 때 특정 로직을 실행해야 할 때
- `didChangeDependencies`: `InheritedWidget` 변경에 반응하는 로직을 명시적으로 제어할 때

```dart
class StatefulHookExample extends StatefulHookWidget {
  final String userId;
  const StatefulHookExample({super.key, required this.userId});

  @override
  State<StatefulHookExample> createState() => _StatefulHookExampleState();
}

class _StatefulHookExampleState extends State<StatefulHookExample> {
  @override
  void didUpdateWidget(StatefulHookExample oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (oldWidget.userId != widget.userId) {
      // 사용자 ID 변경 시 특정 처리
    }
  }

  @override
  Widget build(BuildContext context) {
    // 여기서도 훅 사용 가능
    final userData = useState<Map<String, dynamic>>({});
    ...
    return Container();
  }
}
```

### 6.4 성능 최적화: useMemoized와 useCallback

불필요한 리빌드를 방지하려면 비용이 큰 연산과 콜백 함수를 캐싱합니다.

```dart
Widget build(BuildContext context) {
  final items = useState<List<int>>(List.generate(1000, (i) => i));

  // 정렬된 리스트를 매 빌드마다 재계산하지 않음
  final sortedItems = useMemoized(
    () => [...items.value]..sort((a, b) => b.compareTo(a)),
    [items.value],
  );

  // 자식 위젯에 전달되는 콜백 참조를 안정적으로 유지
  final handleTap = useCallback(
    (int item) => debugPrint('탭: $item'),
    const [],
  );

  return ListView.builder(
    itemCount: sortedItems.length,
    itemBuilder: (_, i) => GestureDetector(
      onTap: () => handleTap(sortedItems[i]),
      child: Text('${sortedItems[i]}'),
    ),
  );
}
```

---

## 7. 설치 및 마이그레이션 가이드

```yaml
# pubspec.yaml
dependencies:
  flutter_hooks: ^0.21.0
  # Riverpod과 함께 사용할 경우
  hooks_riverpod: ^2.0.0
```

기존 `StatefulWidget`을 `HookWidget`으로 마이그레이션하는 기준은 다음과 같습니다.

| 상황 | 권장 |
|------|------|
| `initState` + `dispose`에 동일 리소스 관리 | HookWidget으로 전환 |
| 동일 생명주기 로직이 여러 위젯에 반복 | 커스텀 훅으로 추출 |
| `didUpdateWidget`, `didChangeDependencies` 필요 | StatefulHookWidget |
| 단순한 `bool` 토글 하나 | `StatefulWidget` 유지도 무방 |

Flutter Hooks는 코드의 **응집도**를 높이고 **재사용성**을 극대화하는 강력한 도구입니다. 프로젝트 규모가 커질수록 중복된 생명주기 코드가 누적되는 StatefulWidget의 한계를 훅 기반 설계로 효과적으로 극복할 수 있습니다.

## 참고 자료
- [flutter_hooks 공식 패키지 (pub.dev)](https://pub.dev/packages/flutter_hooks)
- [flutter_hooks API 문서](https://pub.dev/documentation/flutter_hooks/latest/flutter_hooks/)
- [flutter_hooks GitHub 저장소](https://github.com/rrousselGit/flutter_hooks)
