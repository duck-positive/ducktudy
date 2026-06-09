---
layout: post
title: "Flutter Riverpod 고급 패턴: AsyncNotifier·family·autoDispose 완전 정복"
date: 2026-06-09
categories: [android, flutter]
tags: [flutter, riverpod, asyncnotifier, state-management, dart]
---

## 들어가며

Flutter 생태계에서 상태 관리(State Management)는 언제나 뜨거운 주제입니다. Provider, BLoC, GetX, MobX 등 수많은 선택지가 있지만, **Riverpod**은 그 중에서도 "Provider의 진화형"으로서 꾸준히 주목받아 왔습니다. Riverpod 2.0을 거쳐 최근 3.x에 이르러 `AsyncNotifier`, `Notifier`, `family`, `autoDispose` 등의 개념이 정교하게 다듬어졌습니다.

이번 아티클에서는 단순한 `StateProvider` 수준을 벗어나, **실무에서 자주 마주치는 복잡한 시나리오**를 Riverpod 고급 패턴으로 해결하는 방법을 깊이 있게 살펴봅니다.

---

## 1. 개념 설명: Riverpod 핵심 클래스 해부

### 1-1. `Notifier` vs `AsyncNotifier`

Riverpod 2.0 이후, 상태를 변경하는 주체는 `Notifier` 계열 클래스가 담당합니다.

| 클래스 | `build()` 반환 타입 | 사용 시점 |
|---|---|---|
| `Notifier<T>` | `T` (동기) | 즉시 계산 가능한 상태 |
| `AsyncNotifier<T>` | `Future<T>` | API 호출, DB 읽기 등 비동기 초기화 |
| `StreamNotifier<T>` | `Stream<T>` | WebSocket, Firestore 실시간 스트림 |

`AsyncNotifier`는 내부적으로 `AsyncValue<T>`를 state로 관리합니다. `AsyncValue`는 세 가지 상태를 하나의 sealed union으로 표현합니다.

```
AsyncValue<T>
  ├── AsyncLoading<T>   // 로딩 중
  ├── AsyncData<T>      // 데이터 보유
  └── AsyncError<T>     // 에러 발생
```

이 덕분에 `when()` 메서드 하나로 UI 분기 처리를 깔끔하게 해결할 수 있습니다.

### 1-2. `family` 수정자

`family`는 프로바이더에 **외부 파라미터**를 주입하는 수정자입니다. 예를 들어 게시글 상세 화면에서 `postId`에 따라 서로 다른 프로바이더 인스턴스를 생성해야 할 때 사용합니다.

```dart
final postProvider = AsyncNotifierProvider.family<PostNotifier, Post, int>(
  PostNotifier.new,
);
```

`family`로 생성된 프로바이더는 `ref.watch(postProvider(42))`처럼 파라미터를 넘겨서 읽습니다.

### 1-3. `autoDispose` 수정자

기본적으로 Riverpod 프로바이더는 **전역 캐시**처럼 동작합니다. 즉, 한 번 초기화되면 앱이 종료될 때까지 메모리에 남아 있습니다. `autoDispose`를 붙이면 해당 프로바이더를 구독하는 위젯이 모두 사라졌을 때 자동으로 dispose됩니다.

```dart
final tempSearchProvider = AsyncNotifierProvider.autoDispose<
    SearchNotifier, List<SearchResult>>(
  SearchNotifier.new,
);
```

`family`와 `autoDispose`는 함께 사용할 수 있습니다. `AsyncNotifierProvider.autoDispose.family`가 그 형태입니다.

---

## 2. 왜 필요한가: 실무에서 만나는 문제들

### 문제 1 — 낙관적 업데이트(Optimistic Update)

사용자가 "좋아요" 버튼을 누르면, 서버 응답을 기다리지 않고 **즉시 UI를 갱신**하고 싶습니다. 서버가 실패하면 원래 상태로 롤백해야 합니다. 기존 `StateNotifier`로는 임시 상태 저장과 롤백 로직을 일일이 구현해야 했습니다.

### 문제 2 — 동일 화면의 중복 API 호출 방지

`family`를 쓰지 않으면, 여러 위젯이 동일한 상세 데이터를 각자 가져오게 됩니다. 결과적으로 같은 `postId`에 대해 API가 여러 번 호출됩니다.

### 문제 3 — 검색/필터 화면의 메모리 누수

검색 화면을 열 때마다 결과 목록이 메모리에 쌓이면 앱이 무거워집니다. `autoDispose`가 없으면 화면을 닫아도 이전 검색 결과가 남아있게 됩니다.

---

## 3. 실제 구현 예제

### 예제 1 — AsyncNotifier를 이용한 CRUD + 낙관적 업데이트

아래는 할 일(Todo) 목록을 관리하는 `TodoListNotifier` 전체 구현입니다. 핵심은 `update()` 메서드를 통한 **낙관적 업데이트**와, 실패 시 자동 롤백입니다.

```dart
// todo_model.dart
class Todo {
  const Todo({
    required this.id,
    required this.title,
    this.completed = false,
  });

  final String id;
  final String title;
  final bool completed;

  Todo copyWith({String? title, bool? completed}) {
    return Todo(
      id: id,
      title: title ?? this.title,
      completed: completed ?? this.completed,
    );
  }
}

// todo_repository.dart
abstract class TodoRepository {
  Future<List<Todo>> fetchAll();
  Future<void> toggle(String id);
  Future<void> delete(String id);
}

// todo_notifier.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'todo_notifier.g.dart';

@riverpod
class TodoListNotifier extends _$TodoListNotifier {
  @override
  Future<List<Todo>> build() async {
    // 앱 시작 시 전체 목록 로드
    return ref.watch(todoRepositoryProvider).fetchAll();
  }

  /// 낙관적 업데이트: 완료 상태 토글
  Future<void> toggle(String id) async {
    // 1) 현재 상태 스냅샷 저장 (롤백용)
    final previous = state;

    // 2) 즉시 UI 갱신 — AsyncData 안의 리스트를 직접 교체
    state = AsyncData(
      await future.then(
        (todos) => todos.map((t) {
          return t.id == id ? t.copyWith(completed: !t.completed) : t;
        }).toList(),
      ),
    );

    // 3) 실제 서버 요청
    try {
      await ref.read(todoRepositoryProvider).toggle(id);
    } catch (e, st) {
      // 4) 실패 시 롤백
      state = previous;
      // 에러를 상위로 전파해 SnackBar 등에서 처리 가능
      Error.throwWithStackTrace(e, st);
    }
  }

  /// 낙관적 삭제
  Future<void> delete(String id) async {
    final previous = state;

    state = AsyncData(
      await future.then(
        (todos) => todos.where((t) => t.id != id).toList(),
      ),
    );

    try {
      await ref.read(todoRepositoryProvider).delete(id);
    } catch (e, st) {
      state = previous;
      Error.throwWithStackTrace(e, st);
    }
  }
}

// todo_screen.dart (UI)
class TodoScreen extends ConsumerWidget {
  const TodoScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final todosAsync = ref.watch(todoListNotifierProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Todo')),
      body: todosAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(child: Text('오류: $e')),
        data: (todos) => ListView.builder(
          itemCount: todos.length,
          itemBuilder: (_, i) {
            final todo = todos[i];
            return ListTile(
              title: Text(todo.title),
              leading: Checkbox(
                value: todo.completed,
                onChanged: (_) async {
                  try {
                    await ref
                        .read(todoListNotifierProvider.notifier)
                        .toggle(todo.id);
                  } catch (_) {
                    ScaffoldMessenger.of(context).showSnackBar(
                      const SnackBar(content: Text('업데이트 실패. 되돌렸습니다.')),
                    );
                  }
                },
              ),
              trailing: IconButton(
                icon: const Icon(Icons.delete_outline),
                onPressed: () =>
                    ref.read(todoListNotifierProvider.notifier).delete(todo.id),
              ),
            );
          },
        ),
      ),
    );
  }
}
```

**핵심 포인트:**
- `build()` 메서드 하나가 초기 로딩을 담당합니다. 별도의 `init()` 호출이 불필요합니다.
- `state = AsyncData(...)` 구문으로 동기적으로 즉시 UI를 바꿀 수 있습니다.
- `try-catch`에서 `state = previous`로 원자적 롤백이 가능합니다.

---

### 예제 2 — `family` + `autoDispose`로 파라미터화된 상세 화면

게시글 상세 화면처럼 `id`에 따라 다른 데이터를 로드해야 하는 시나리오입니다. `autoDispose`를 함께 사용하여 화면을 나가면 메모리가 자동 해제됩니다.

```dart
// post_detail_notifier.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'post_detail_notifier.g.dart';

@riverpod
class PostDetailNotifier extends _$PostDetailNotifier {
  // family 파라미터는 build() 의 인자로 선언
  @override
  Future<Post> build(int postId) async {
    // keepAlive 를 원한다면 ref.keepAlive() 호출
    // 여기서는 autoDispose 기본 동작 사용
    final repo = ref.watch(postRepositoryProvider);
    return repo.fetchById(postId);
  }

  Future<void> likePost() async {
    // state를 copyWith 로 업데이트하는 패턴
    final current = await future;
    state = AsyncData(current.copyWith(likes: current.likes + 1));
    try {
      await ref.read(postRepositoryProvider).like(current.id);
    } catch (_) {
      // 롤백
      state = AsyncData(current);
    }
  }
}

// post_detail_screen.dart
class PostDetailScreen extends ConsumerWidget {
  const PostDetailScreen({super.key, required this.postId});

  final int postId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // family 파라미터 전달 — 캐싱도 postId 별로 분리됨
    final postAsync = ref.watch(postDetailNotifierProvider(postId));

    return Scaffold(
      body: postAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, st) => ErrorView(error: e, stackTrace: st),
        data: (post) => CustomScrollView(
          slivers: [
            SliverAppBar(
              title: Text(post.title),
              actions: [
                IconButton(
                  icon: const Icon(Icons.favorite_outline),
                  onPressed: () => ref
                      .read(postDetailNotifierProvider(postId).notifier)
                      .likePost(),
                ),
              ],
            ),
            SliverToBoxAdapter(
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: Text(post.body),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

// 라우팅 — GoRouter와 함께 사용하는 경우
GoRoute(
  path: '/posts/:id',
  builder: (context, state) {
    final id = int.parse(state.pathParameters['id']!);
    return PostDetailScreen(postId: id);
  },
),
```

**핵심 포인트:**
- `@riverpod` 어노테이션과 `build(int postId)`처럼 파라미터를 선언하면, 코드 생성기가 `family`를 자동으로 붙여줍니다.
- 동일한 `postId`를 가진 화면이 2개 열려도 네트워크 요청은 1번만 발생합니다 (캐싱).
- 모든 화면이 닫히면 해당 `postId`에 대한 상태가 자동 폐기됩니다.

---

## 4. 주의사항 및 실무 팁

### 팁 1 — `ref.invalidate()` vs `ref.refresh()`

| 메서드 | 동작 |
|---|---|
| `ref.invalidate(provider)` | 프로바이더를 무효화. 다음 watch 시 재실행 |
| `ref.refresh(provider)` | 즉시 재실행 후 새 값 반환 |

강제 새로고침(Pull-to-Refresh)에는 `ref.invalidate()`를 권장합니다. `ref.refresh()`는 반환값을 직접 사용해야 할 때 씁니다.

```dart
// Pull-to-Refresh 구현
RefreshIndicator(
  onRefresh: () async {
    ref.invalidate(todoListNotifierProvider);
    // 새 데이터가 로드될 때까지 기다림
    await ref.read(todoListNotifierProvider.future);
  },
  child: listView,
)
```

### 팁 2 — `ref.listenManual()` 로 1회성 사이드 이펙트 처리

화면 이동, 다이얼로그 표시처럼 상태 변화에 반응하는 사이드 이펙트는 `ref.listen()`을 사용하지만, **`ConsumerStatefulWidget` 없이 일회성으로** 처리하려면 `ref.listenManual()`이 유용합니다.

```dart
class AuthScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    ref.listen<AsyncValue<User?>>(
      authNotifierProvider,
      (previous, next) {
        // 로그인 성공 시 홈 화면으로 이동
        if (next is AsyncData && next.value != null) {
          context.go('/home');
        }
        // 에러 시 스낵바
        if (next is AsyncError) {
          ScaffoldMessenger.of(context).showSnackBar(
            SnackBar(content: Text(next.error.toString())),
          );
        }
      },
    );
    // ... UI
  }
}
```

### 팁 3 — `keepAlive`로 선택적 캐싱

`autoDispose` 프로바이더라도 특정 조건에서는 캐시를 유지하고 싶을 수 있습니다. `ref.keepAlive()`를 호출하면 dispose를 지연할 수 있습니다.

```dart
@override
Future<Post> build(int postId) async {
  // 즐겨찾기 등록된 포스트는 캐시 유지
  final isFav = ref.watch(favoriteIdsProvider).contains(postId);
  if (isFav) {
    ref.keepAlive(); // dispose 하지 않음
  }
  return ref.watch(postRepositoryProvider).fetchById(postId);
}
```

### 팁 4 — `AsyncValue.guard()`로 try-catch 단순화

```dart
// before
try {
  state = AsyncData(await api.fetchData());
} catch (e, st) {
  state = AsyncError(e, st);
}

// after — 훨씬 간결
state = await AsyncValue.guard(() => api.fetchData());
```

### 팁 5 — 테스트에서 `ProviderContainer` 활용

Riverpod의 가장 큰 장점 중 하나는 **테스트 친화성**입니다. `ProviderContainer`를 사용하면 위젯 없이도 비즈니스 로직을 단독으로 테스트할 수 있습니다.

```dart
test('toggle updates completed state', () async {
  final container = ProviderContainer(
    overrides: [
      todoRepositoryProvider.overrideWithValue(FakeTodoRepository()),
    ],
  );
  addTearDown(container.dispose);

  // 초기 로딩 완료 대기
  await container.read(todoListNotifierProvider.future);

  // toggle 실행
  await container.read(todoListNotifierProvider.notifier).toggle('1');

  // 검증
  final todos = await container.read(todoListNotifierProvider.future);
  expect(todos.first.completed, isTrue);
});
```

---

## 마치며

Riverpod의 `AsyncNotifier`, `family`, `autoDispose`는 각각 독립적인 기능이지만, **함께 사용할 때 진가를 발휘합니다**. 낙관적 업데이트로 사용자 경험을 높이고, `family`로 불필요한 중복 요청을 제거하며, `autoDispose`로 메모리를 효율적으로 관리하는 이 세 가지 패턴을 숙달하면, 실무 수준의 Flutter 앱을 훨씬 더 자신감 있게 구축할 수 있습니다.

다음 포스트에서는 Riverpod과 `go_router`를 결합하여 인증(Auth) 상태에 따른 라우팅 가드를 구현하는 방법을 다룰 예정입니다.

---

## 참고 자료
- [flutter_riverpod | pub.dev](https://pub.dev/packages/flutter_riverpod)
- [riverpod | pub.dev](https://pub.dev/packages/riverpod)
- [AsyncNotifier class - Dart API docs](https://pub.dev/documentation/flutter_riverpod/latest/flutter_riverpod/AsyncNotifier-class.html)
