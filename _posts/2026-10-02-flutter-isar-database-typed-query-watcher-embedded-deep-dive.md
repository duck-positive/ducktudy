---
layout: post
title: "Flutter Isar 3 심화: 타입 안전 쿼리·임베디드 객체·Watcher로 온디바이스 NoSQL 완전 정복"
date: 2026-10-02
categories: [android, flutter]
tags: [flutter, dart, isar, nosql, local-database, watcher, riverpod, offline-first]
---

모바일 앱의 오프라인 지원과 빠른 로컬 검색은 사용자 경험을 결정짓는 핵심 요소입니다. Flutter 생태계에서 가장 빠른 온디바이스 NoSQL 데이터베이스인 **Isar**는 `SharedPreferences`의 단순함과 `Drift`(SQLite)의 타입 안전성을 넘어, 스키마 기반 CRUD·복합 인덱스·실시간 Watcher를 하나의 패키지로 제공합니다. 이 글에서는 Isar 3.x(프로덕션 안정 버전)를 기준으로 실제 앱에 적용할 수 있는 심화 패턴을 다룹니다.

---

## Isar란 무엇인가

Isar는 Dart/Flutter를 위해 처음부터 설계된 NoSQL 임베디드 데이터베이스입니다. 내부적으로 Rust로 작성된 경량 스토리지 엔진(libisar)을 FFI로 호출하여, 네이티브에 필적하는 읽기/쓰기 속도를 달성합니다.

**주요 특징:**

| 특성 | 설명 |
|------|------|
| 비동기 ACID | 모든 쓰기 연산은 트랜잭션 보장 |
| 타입 안전 쿼리 | 코드 생성으로 컴파일 타임 쿼리 검증 |
| 복합 인덱스 | 다중 필드 인덱스, 전문 검색(Full-Text) 지원 |
| Watcher | 컬렉션/문서 단위 실시간 변경 스트림 |
| 멀티 아이솔레이트 | 백그라운드 아이솔레이트에서 동시 접근 가능 |
| 플랫폼 | iOS·Android·macOS·Linux·Windows·웹(제한) |

---

## 왜 Isar인가

로컬 데이터 저장 옵션을 선택할 때 흔히 고민하는 세 가지 도구와 비교합니다.

- **SharedPreferences / flutter_secure_storage**: 단순 키-값 저장에 최적화. 구조적 데이터, 관계형 쿼리, 대용량 데이터에 부적합.
- **Hive**: NoSQL, 빠름. 하지만 타입 어댑터를 수동으로 관리해야 하고, 복잡한 쿼리 DSL이 없어 필터링을 Dart 코드로 직접 구현해야 합니다.
- **Drift (SQLite)**: 완전한 SQL 표현력과 마이그레이션 지원. 대신 테이블 정의·쿼리 모두 SQL 패러다임에 익숙해야 하며, 빌드 오버헤드가 큽니다.
- **Isar**: 스키마를 Dart 클래스로 정의하면 코드 생성기가 타입 안전 쿼리 빌더를 만들어 줍니다. 인덱스·Watcher·임베디드 객체를 선언만으로 사용할 수 있어, 복잡한 오프라인-퍼스트 앱에 이상적입니다.

---

## 설치 및 초기화

```yaml
# pubspec.yaml
dependencies:
  isar: ^3.1.0+1
  isar_flutter_libs: ^3.1.0+1  # 네이티브 바이너리 번들

dev_dependencies:
  isar_generator: ^3.1.0+1
  build_runner: ^2.4.0
```

스키마 파일을 변경할 때마다 코드 생성을 실행합니다:

```bash
flutter pub run build_runner build --delete-conflicting-outputs
```

---

## 스키마 설계 — @Collection, @Embedded, @Index

Isar의 핵심은 Dart 클래스 어노테이션입니다.

- `@Collection()`: 독립 컬렉션(테이블에 해당). `id` 필드는 자동 증가 `int`.
- `@Embedded()`: 다른 컬렉션 안에 중첩 저장되는 객체. 별도 컬렉션 ID 없음.
- `@Index()`: 단일/복합 인덱스, 전문 검색 인덱스 설정.

```dart
import 'package:isar/isar.dart';

part 'todo.g.dart';  // build_runner가 생성

@Collection()
class Todo {
  Id id = Isar.autoIncrement;

  @Index(type: IndexType.value)
  late String title;

  @Index(type: IndexType.hash)
  late String category;

  bool isDone = false;

  DateTime createdAt = DateTime.now();

  // 임베디드 객체: 별도 컬렉션 없이 Todo 안에 JSON처럼 저장
  TodoMeta? meta;
}

@Embedded()
class TodoMeta {
  String? priority;       // 'high' | 'medium' | 'low'
  List<String> tags = [];
  int estimatedMinutes = 0;
}
```

---

## 실제 구현 예제 1: Repository 패턴으로 CRUD + 복합 쿼리 구현

아래는 실제 앱에서 사용 가능한 `TodoRepository`입니다. Isar 인스턴스를 생성하고, 카테고리·완료 여부·우선순위로 필터링하는 복합 쿼리를 보여 줍니다.

```dart
import 'package:isar/isar.dart';
import 'package:path_provider/path_provider.dart';
import 'todo.dart';

class TodoRepository {
  late final Isar _isar;

  Future<void> init() async {
    final dir = await getApplicationDocumentsDirectory();
    _isar = await Isar.open(
      [TodoSchema],
      directory: dir.path,
      name: 'todos_db',
    );
  }

  // ─────────────── CREATE / UPDATE ───────────────
  Future<Id> upsert(Todo todo) async {
    return _isar.writeTxn(() => _isar.todos.put(todo));
  }

  Future<void> upsertAll(List<Todo> todos) async {
    await _isar.writeTxn(() => _isar.todos.putAll(todos));
  }

  // ─────────────── READ ───────────────
  Future<Todo?> findById(Id id) => _isar.todos.get(id);

  /// 카테고리별 + 미완료 항목만 + 생성 시간 내림차순
  Future<List<Todo>> fetchByCategory(String category) {
    return _isar.todos
        .where()
        .categoryEqualTo(category)   // 인덱스 활용
        .filter()
        .isDoneFalse()
        .sortByCreatedAtDesc()
        .findAll();
  }

  /// 제목 전문 검색 (IndexType.value 필요)
  Future<List<Todo>> search(String keyword) {
    return _isar.todos
        .where()
        .titleStartsWith(keyword, caseSensitive: false)
        .findAll();
  }

  /// 복합 조건: 우선순위가 high이고 태그에 'work'가 포함된 항목
  Future<List<Todo>> fetchHighPriorityWork() {
    return _isar.todos
        .filter()
        .meta((meta) => meta.priorityEqualTo('high')
            .and()
            .tagsElementContains('work'))
        .findAll();
  }

  // ─────────────── DELETE ───────────────
  Future<bool> delete(Id id) async {
    return _isar.writeTxn(() => _isar.todos.delete(id));
  }

  Future<int> deleteAllDone() async {
    return _isar.writeTxn(() {
      return _isar.todos.filter().isDoneTrue().deleteAll();
    });
  }

  // ─────────────── 집계 ───────────────
  Future<int> countByCategory(String category) {
    return _isar.todos.where().categoryEqualTo(category).count();
  }
}
```

**코드 포인트:**
- `.where()` 절은 인덱스를 사용하여 O(log n) 조회를 보장합니다. 반면 `.filter()` 절은 풀 스캔이므로, 자주 조회하는 필드에는 반드시 `@Index`를 선언하세요.
- `writeTxn`은 자동으로 트랜잭션을 열고 커밋합니다. 예외 발생 시 롤백됩니다.
- 임베디드 객체(`.meta(...)`) 안의 필드도 `.filter()` DSL로 접근할 수 있습니다.

---

## 실제 구현 예제 2: Watcher + Riverpod으로 실시간 UI 반응

Isar의 `watchLazy()`/`watch()` API는 컬렉션 변경 시 이벤트를 방출하는 `Stream`을 반환합니다. Riverpod `StreamProvider`와 결합하면 DB 변경이 UI에 자동으로 반영됩니다.

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:isar/isar.dart';
import 'todo.dart';
import 'todo_repository.dart';

// ── 1. 레포지토리 Provider ──────────────────────────────
final todoRepoProvider = FutureProvider<TodoRepository>((ref) async {
  final repo = TodoRepository();
  await repo.init();
  return repo;
});

// ── 2. Watcher: 카테고리별 Todo 실시간 스트림 ─────────────
final categoryTodosProvider =
    StreamProvider.family<List<Todo>, String>((ref, category) async* {
  final repo = await ref.watch(todoRepoProvider.future);

  // 초기 데이터 방출
  yield await repo.fetchByCategory(category);

  // 컬렉션 변경 감지 → 재조회 후 방출
  await for (final _ in repo.watchCollection()) {
    yield await repo.fetchByCategory(category);
  }
});

// ── 3. Repository에 Watcher 메서드 추가 ────────────────────
// (TodoRepository 내부)
Stream<void> watchCollection() {
  return _isar.todos.watchLazy(fireImmediately: false);
}

// ── 4. 화면 UI ───────────────────────────────────────────
class TodoListScreen extends ConsumerWidget {
  final String category;
  const TodoListScreen({required this.category, super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final asyncTodos = ref.watch(categoryTodosProvider(category));

    return asyncTodos.when(
      loading: () => const CircularProgressIndicator(),
      error: (e, st) => Text('오류: $e'),
      data: (todos) => ListView.builder(
        itemCount: todos.length,
        itemBuilder: (context, i) {
          final todo = todos[i];
          return CheckboxListTile(
            title: Text(todo.title),
            subtitle: Text(todo.meta?.priority ?? '우선순위 없음'),
            value: todo.isDone,
            onChanged: (done) async {
              final repo = await ref.read(todoRepoProvider.future);
              todo.isDone = done ?? false;
              await repo.upsert(todo);
              // Watcher가 감지하여 UI 자동 갱신 — 수동 setState 불필요
            },
          );
        },
      ),
    );
  }
}
```

**동작 원리:**
1. `watchLazy()`는 해당 컬렉션에 쓰기 트랜잭션이 커밋될 때마다 빈 이벤트를 방출합니다.
2. `StreamProvider.family`가 이를 수신하면 최신 데이터를 재조회하여 위젯 트리에 전달합니다.
3. 체크박스를 토글하면 → `upsert()` → 트랜잭션 커밋 → Watcher 이벤트 → UI 갱신의 단방향 흐름이 완성됩니다.

---

## 주의사항 및 팁

### 1. 마이그레이션 전략

Isar는 스키마 변경 시 자동 마이그레이션을 **지원하지 않습니다**. 필드를 추가·삭제·타입 변경 할 때는 `Isar.open()` 호출 시 `inspector`와 `schemaVersion`을 함께 관리하세요.

```dart
_isar = await Isar.open(
  [TodoSchema],
  directory: dir.path,
  // 스키마 변경 시 버전을 올리고 migration 콜백에서 데이터 변환
  // Isar 3.x에서는 스키마 불일치 시 컬렉션을 clear하거나 재설계 필요
);
```

실제 프로덕션에서는 마이그레이션이 필요한 경우 **버전 컬럼을 별도 키-값 저장소(SharedPreferences)에 기록**하고, 앱 시작 시 구버전 DB를 삭제 후 재생성하는 전략이 가장 안전합니다.

### 2. 인덱스 과다 사용 금지

인덱스는 읽기 속도를 높이지만 쓰기 성능과 저장 공간을 희생합니다. 잦은 삽입·수정이 있는 컬렉션에서 모든 필드에 인덱스를 붙이면 오히려 성능이 저하됩니다. **쿼리에 실제로 사용하는 필드**에만 선별적으로 인덱스를 적용하세요.

### 3. 대용량 데이터 일괄 처리

수천 건 이상을 한 번에 저장할 때는 `putAll()`을 단일 트랜잭션으로 묶습니다. 개별 `put()`을 반복 호출하면 트랜잭션 오버헤드가 선형으로 증가합니다.

```dart
// 느림: 각 put마다 트랜잭션 발생
for (final todo in todos) {
  await _isar.writeTxn(() => _isar.todos.put(todo));
}

// 빠름: 단일 트랜잭션
await _isar.writeTxn(() => _isar.todos.putAll(todos));
```

### 4. 멀티 아이솔레이트 주의

Isar는 멀티 아이솔레이트를 지원하지만, **동일한 `name`으로 `Isar.open()`을 호출하면 이미 열린 인스턴스를 재사용**합니다. 아이솔레이트 간 공유 시에는 `Isar.getInstance()`로 기존 인스턴스를 참조하고, 아이솔레이트 종료 시 `close()`를 호출하지 않도록 주의하세요.

### 5. 웹 플랫폼 제한

`isar_flutter_libs`의 웹 지원은 IndexedDB 기반으로 네이티브 대비 쿼리 기능이 제한됩니다. 웹을 주요 타겟으로 삼는 경우 Drift + Drift Web(WebAssembly) 조합이 더 안정적입니다.

### 6. Inspector 활용

개발 중 Isar Inspector 웹 UI(`http://localhost:8080`)를 통해 컬렉션 데이터와 쿼리 실행 계획을 실시간으로 확인할 수 있습니다. `Isar.open()` 후 앱을 실행하면 자동으로 연결됩니다.

---

## 정리

Isar는 "설정 없이 시작해서 복잡한 쿼리까지 커버하는" 실용적인 로컬 데이터베이스입니다. 핵심 패턴을 요약하면:

1. `@Collection` + `@Embedded` + `@Index`로 스키마를 선언한다.
2. `where()`는 인덱스 경로, `filter()`는 추가 조건으로 분리한다.
3. `watchLazy()`를 Riverpod `StreamProvider`와 연결해 단방향 반응형 흐름을 만든다.
4. 일괄 쓰기는 반드시 단일 `writeTxn` 안에서 `putAll()`로 처리한다.
5. 마이그레이션은 버전 관리 + 재생성 전략으로 안전하게 처리한다.

오프라인-퍼스트 앱, 복잡한 로컬 필터링이 필요한 앱, 또는 실시간 데이터 반응이 중요한 앱이라면 Isar가 탁월한 선택입니다.

---

## 참고 자료

- [Isar — pub.dev 패키지 페이지](https://pub.dev/packages/isar)
- [Isar API 공식 문서 (Dart)](https://pub.dev/documentation/isar/latest/)
- [isar_flutter_libs — pub.dev](https://pub.dev/packages/isar_flutter_libs)
- [Isar GitHub 저장소](https://github.com/isar/isar)
