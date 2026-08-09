---
layout: post
title: "Flutter Drift 심화: 타입 안전한 SQLite 로컬 데이터베이스 완전 정복"
date: 2026-08-09
categories: [android, flutter]
tags: [flutter, drift, sqlite, local-database, orm, code-generation, riverpod, dao]
---

Flutter 앱에서 로컬 데이터를 영속적으로 저장해야 할 때, 가장 강력한 선택지 중 하나가 **Drift**입니다. Drift(구 Moor)는 SQLite 위에 올라간 반응형(Reactive) ORM 라이브러리로, 타입 안전한 코드 생성, SQL과 Dart 쿼리 혼용, 스트림 기반 실시간 업데이트, 안전한 마이그레이션을 한 번에 제공합니다. 이 아티클에서는 Drift의 핵심 개념부터 실전 패턴까지, 깊이 있게 파고듭니다.

---

## 왜 Drift인가?

Flutter 생태계에는 로컬 데이터베이스 선택지가 여럿 있습니다.

| 라이브러리 | 특징 | 한계 |
|---|---|---|
| `sqflite` | 낮은 수준의 SQLite 래퍼 | 타입 안전성 없음, 수동 마이그레이션 |
| `hive` | Key-Value NoSQL | 관계형 쿼리 불가 |
| `isar` | 빠른 NoSQL ORM | 복잡한 관계형 쿼리 한계 |
| `drift` | 타입 안전 관계형 ORM | 코드 생성 필요 |

Drift는 **컴파일 타임 타입 안전성**을 보장합니다. 컬럼 이름을 문자열로 쓰다 오타를 내던 시대는 끝납니다. 쿼리 작성 시점에 IDE가 자동완성과 오류를 잡아줍니다. 또한 쿼리 결과가 `Stream<List<T>>`로 반환되어, 데이터가 변경될 때마다 UI가 자동으로 갱신됩니다.

---

## 개념: Drift 아키텍처 이해하기

Drift는 네 가지 핵심 요소로 구성됩니다.

### 1. Table (테이블 정의)
Dart 클래스로 SQL 테이블 스키마를 선언합니다. `@DataClassName` 어노테이션으로 생성될 데이터 클래스 이름을 지정할 수 있습니다.

### 2. Database (데이터베이스 클래스)
`@DriftDatabase` 어노테이션으로 데이터베이스를 선언하고, 어떤 테이블과 DAO를 포함하는지 명시합니다.

### 3. DAO (Data Access Object)
데이터베이스 쿼리를 논리적으로 그룹화하는 클래스입니다. `@DriftAccessor`로 선언하며, 각 기능별(예: `TodoDao`, `UserDao`)로 분리해 유지보수성을 높입니다.

### 4. 코드 생성 (build_runner)
`build_runner`가 `.g.dart` 파일을 생성하고, 선언한 테이블과 쿼리를 기반으로 타입 안전한 Dart 코드를 만들어냅니다.

---

## 의존성 설정

`pubspec.yaml`에 다음 패키지를 추가합니다:

```yaml
dependencies:
  drift: ^2.34.3
  drift_flutter: ^0.3.1
  path_provider: ^2.1.0

dev_dependencies:
  drift_dev: ^2.34.5
  build_runner: ^2.4.0
```

`drift_flutter` 패키지는 플랫폼별 SQLite 연결을 자동으로 처리해줍니다. Android, iOS, macOS, Windows, Linux, Web 모두 단 하나의 API(`driftDatabase`)로 사용할 수 있습니다.

---

## 실전 구현 예제 1: Todo 앱 — 기본 CRUD와 스트림

### 테이블 및 데이터베이스 선언

```dart
// database/app_database.dart
import 'package:drift/drift.dart';
import 'package:drift_flutter/drift_flutter.dart';

part 'app_database.g.dart';

// 1. 테이블 정의
class Todos extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get title => text().withLength(min: 1, max: 200)();
  TextColumn get body => text().nullable()();
  BoolColumn get completed => boolean().withDefault(const Constant(false))();
  DateTimeColumn get createdAt => dateTime().withDefault(currentDateAndTime)();

  // 복합 인덱스 정의 (선택적)
  @override
  List<Set<Column>> get uniqueKeys => [
        {title, createdAt},
      ];
}

class Categories extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get name => text().unique()();
  IntColumn get color => integer()(); // ARGB 색상값
}

class TodoEntries extends Table {
  IntColumn get id => integer().autoIncrement()();
  IntColumn get todoId => integer().references(Todos, #id)();
  IntColumn get categoryId => integer().nullable().references(Categories, #id)();
  TextColumn get note => text().nullable()();
}

// 2. 조인 결과를 담을 데이터 클래스
class TodoWithCategory {
  final Todo todo;
  final Category? category;

  TodoWithCategory({required this.todo, this.category});
}

// 3. 데이터베이스 선언
@DriftDatabase(tables: [Todos, Categories, TodoEntries])
class AppDatabase extends _$AppDatabase {
  AppDatabase() : super(_openConnection());

  static QueryExecutor _openConnection() {
    return driftDatabase(name: 'app_database');
  }

  @override
  int get schemaVersion => 1;

  // 조인 쿼리: Todo와 Category를 함께 조회
  Future<List<TodoWithCategory>> getTodosWithCategories() async {
    final query = select(todos).join([
      leftOuterJoin(
        todoEntries,
        todoEntries.todoId.equalsExp(todos.id),
      ),
      leftOuterJoin(
        categories,
        categories.id.equalsExp(todoEntries.categoryId),
      ),
    ]);

    final rows = await query.get();
    return rows.map((row) {
      return TodoWithCategory(
        todo: row.readTable(todos),
        category: row.readTableOrNull(categories),
      );
    }).toList();
  }

  // 스트림: 완료되지 않은 Todo를 실시간으로 관찰
  Stream<List<Todo>> watchIncompleteTodos() {
    return (select(todos)
          ..where((t) => t.completed.not())
          ..orderBy([(t) => OrderingTerm.desc(t.createdAt)]))
        .watch();
  }

  // 트랜잭션: Todo와 엔트리를 원자적으로 삽입
  Future<void> addTodoWithCategory({
    required String title,
    String? body,
    int? categoryId,
    String? note,
  }) async {
    await transaction(() async {
      final todoId = await into(todos).insert(
        TodosCompanion.insert(
          title: title,
          body: Value(body),
        ),
      );

      if (categoryId != null) {
        await into(todoEntries).insert(
          TodoEntriesCompanion.insert(
            todoId: todoId,
            categoryId: Value(categoryId),
            note: Value(note),
          ),
        );
      }
    });
  }
}
```

### UI에서 스트림 소비 (Riverpod 연동)

```dart
// providers/database_providers.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../database/app_database.dart';

final databaseProvider = Provider<AppDatabase>((ref) {
  final db = AppDatabase();
  ref.onDispose(() => db.close());
  return db;
});

final incompleteTodosProvider = StreamProvider<List<Todo>>((ref) {
  return ref.watch(databaseProvider).watchIncompleteTodos();
});

// todo_list_screen.dart
class TodoListScreen extends ConsumerWidget {
  const TodoListScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final todosAsync = ref.watch(incompleteTodosProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('할 일 목록')),
      body: todosAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, st) => Center(child: Text('오류: $e')),
        data: (todos) {
          if (todos.isEmpty) {
            return const Center(child: Text('할 일이 없습니다'));
          }
          return ListView.builder(
            itemCount: todos.length,
            itemBuilder: (context, index) {
              final todo = todos[index];
              return CheckboxListTile(
                title: Text(todo.title),
                subtitle: todo.body != null ? Text(todo.body!) : null,
                value: todo.completed,
                onChanged: (completed) async {
                  final db = ref.read(databaseProvider);
                  await (db.update(db.todos)
                        ..where((t) => t.id.equals(todo.id)))
                      .write(TodosCompanion(
                    completed: Value(completed ?? false),
                  ));
                },
              );
            },
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () async {
          final db = ref.read(databaseProvider);
          await db.addTodoWithCategory(title: '새 할 일 ${DateTime.now()}');
        },
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

---

## 실전 구현 예제 2: DAO 패턴과 커스텀 SQL 쿼리

DAO를 사용하면 데이터베이스 로직을 기능 단위로 분리해 테스트하기 쉬워집니다.

```dart
// database/daos/todo_dao.dart
import 'package:drift/drift.dart';
import '../app_database.dart';

part 'todo_dao.g.dart';

@DriftAccessor(tables: [Todos, TodoEntries, Categories])
class TodoDao extends DatabaseAccessor<AppDatabase> with _$TodoDaoMixin {
  TodoDao(super.db);

  // Dart API를 사용한 쿼리
  Future<List<Todo>> getAllTodos() => select(todos).get();

  Stream<List<Todo>> watchAllTodos() => select(todos).watch();

  Future<Todo?> getTodoById(int id) =>
      (select(todos)..where((t) => t.id.equals(id))).getSingleOrNull();

  Future<int> insertTodo(TodosCompanion todo) =>
      into(todos).insert(todo);

  Future<bool> updateTodo(TodosCompanion todo) =>
      update(todos).replace(todo);

  Future<int> deleteTodo(int id) =>
      (delete(todos)..where((t) => t.id.equals(id))).go();

  // 커스텀 SQL 쿼리 (복잡한 집계에 유용)
  Future<int> countByStatus({required bool completed}) async {
    final countExpr = todos.id.count();
    final query = selectOnly(todos)
      ..addColumns([countExpr])
      ..where(todos.completed.equals(completed));
    final result = await query.getSingle();
    return result.read(countExpr) ?? 0;
  }

  // 날짜 범위 검색
  Stream<List<Todo>> watchTodosBetween(DateTime from, DateTime to) {
    return (select(todos)
          ..where((t) =>
              t.createdAt.isBetweenValues(from, to))
          ..orderBy([(t) => OrderingTerm.asc(t.createdAt)]))
        .watch();
  }

  // 전문(Full-Text) 검색
  Future<List<Todo>> searchTodos(String keyword) {
    final term = '%$keyword%';
    return (select(todos)
          ..where((t) =>
              t.title.like(term) | t.body.like(term)))
        .get();
  }

  // 배치(Batch) 삽입: 대량 데이터를 빠르게 삽입
  Future<void> insertAllTodos(List<TodosCompanion> newTodos) async {
    await batch((b) {
      b.insertAll(todos, newTodos);
    });
  }

  // upsert (삽입 또는 업데이트)
  Future<void> upsertTodo(TodosCompanion todo) async {
    await into(todos).insertOnConflictUpdate(todo);
  }
}
```

### DAO를 데이터베이스에 등록하고 Riverpod과 연동

```dart
// database/app_database.dart (DAO 추가)
@DriftDatabase(tables: [Todos, Categories, TodoEntries], daos: [TodoDao])
class AppDatabase extends _$AppDatabase {
  // ... (이전과 동일)

  // DAO 인스턴스
  late final TodoDao todoDao = TodoDao(this);
}

// providers/database_providers.dart (DAO Provider 추가)
final todoDaoProvider = Provider<TodoDao>((ref) {
  return ref.watch(databaseProvider).todoDao;
});

final todoSearchProvider =
    FutureProvider.family<List<Todo>, String>((ref, keyword) async {
  if (keyword.isEmpty) return [];
  return ref.watch(todoDaoProvider).searchTodos(keyword);
});
```

---

## 스키마 마이그레이션 전략

앱이 배포된 이후 스키마를 변경할 때는 마이그레이션이 필수입니다. Drift는 `MigrationStrategy`를 통해 안전하게 버전을 올릴 수 있습니다.

```dart
@DriftDatabase(tables: [Todos, Categories, TodoEntries])
class AppDatabase extends _$AppDatabase {
  // ...

  @override
  int get schemaVersion => 3; // 버전 업

  @override
  MigrationStrategy get migration {
    return MigrationStrategy(
      onCreate: (Migrator m) async {
        await m.createAll();
      },
      onUpgrade: (Migrator m, int from, int to) async {
        // v1 → v2: Todos 테이블에 priority 컬럼 추가
        if (from < 2) {
          await m.addColumn(todos, todos.createdAt);
          // 기존 행에 기본값 설정이 필요할 경우
          await customStatement(
            'UPDATE todos SET created_at = ? WHERE created_at IS NULL',
            [DateTime.now().millisecondsSinceEpoch],
          );
        }

        // v2 → v3: 새 테이블 추가
        if (from < 3) {
          await m.createTable(categories);
        }
      },
      // 앱 시작 시마다 integrity check 실행 (선택)
      beforeOpen: (details) async {
        if (details.wasCreated) {
          // 처음 생성 시 시드 데이터 삽입
          await into(categories).insert(
            CategoriesCompanion.insert(name: '업무', color: 0xFF2196F3),
          );
          await into(categories).insert(
            CategoriesCompanion.insert(name: '개인', color: 0xFF4CAF50),
          );
        }
        // WAL 모드 활성화 (읽기 성능 향상)
        await customStatement('PRAGMA journal_mode=WAL');
        await customStatement('PRAGMA foreign_keys=ON');
      },
    );
  }
}
```

---

## Isolate를 활용한 백그라운드 처리

Drift는 별도의 설정 없이 Isolate에서 데이터베이스 접근을 지원합니다. 대용량 데이터 처리 시 메인 스레드를 블록하지 않을 수 있습니다.

```dart
// database/app_database.dart
static QueryExecutor _openConnection() {
  return driftDatabase(
    name: 'app_database',
    // 별도 Isolate에서 데이터베이스 열기 (Flutter 앱에서 권장)
    native: const DriftNativeOptions(
      shareAcrossIsolates: true,
    ),
  );
}
```

백그라운드 Isolate에서 대량 데이터를 처리하는 예:

```dart
// 백그라운드 작업: 10만 건 데이터 일괄 삽입
Future<void> importLargeDataset(List<Map<String, dynamic>> rawData) async {
  final db = AppDatabase();
  try {
    await db.transaction(() async {
      // 배치 삽입으로 1건씩 삽입할 때보다 ~100배 빠름
      await db.batch((batch) {
        batch.insertAll(
          db.todos,
          rawData.map((row) => TodosCompanion.insert(
                title: row['title'] as String,
                body: Value(row['body'] as String?),
              )),
        );
      });
    });
  } finally {
    await db.close();
  }
}
```

---

## 주의사항과 팁

### 1. `part` 지시문을 반드시 포함하라

```dart
// ❌ 잊으면 코드 생성이 연결되지 않음
class AppDatabase extends _$AppDatabase { ... }

// ✅ 파일 상단에 반드시 선언
part 'app_database.g.dart';
```

### 2. build_runner는 watch 모드를 활용하라

개발 중에는 `dart run build_runner watch --delete-conflicting-outputs`를 사용하면 코드 변경 시 자동으로 재생성합니다. CI 환경에서는 `build`를 사용합니다.

### 3. `Companion` 클래스의 `Value`와 `absent` 구분

```dart
// Value(null): 명시적으로 NULL을 쓰겠다
// absent(): 해당 필드를 UPDATE에서 제외하겠다 (기존 값 유지)
await (update(todos)..where((t) => t.id.equals(id))).write(
  TodosCompanion(
    completed: const Value(true),  // completed를 true로 갱신
    // title은 absent()이므로 변경하지 않음
  ),
);
```

### 4. 스트림 구독 해제

Riverpod의 `StreamProvider`나 `ref.onDispose`를 활용하면 자동으로 구독이 해제됩니다. 수동으로 `StreamSubscription`을 관리하는 경우 반드시 `cancel()`을 호출하세요.

### 5. PRAGMA 설정으로 성능 튜닝

```dart
// beforeOpen 콜백에서 설정
await customStatement('PRAGMA cache_size=-64000'); // 64MB 캐시
await customStatement('PRAGMA synchronous=NORMAL'); // fsync 빈도 감소
await customStatement('PRAGMA temp_store=MEMORY');  // 임시 테이블을 메모리에
```

### 6. 테스트는 인메모리 데이터베이스를 사용하라

```dart
// test/todo_dao_test.dart
import 'package:drift/native.dart';
import 'package:flutter_test/flutter_test.dart';

AppDatabase createTestDatabase() {
  return AppDatabase.forTesting(NativeDatabase.memory());
}

void main() {
  late AppDatabase db;

  setUp(() => db = createTestDatabase());
  tearDown(() => db.close());

  test('insertTodo inserts a row', () async {
    await db.into(db.todos).insert(
      TodosCompanion.insert(title: '테스트 할 일'),
    );
    final all = await db.select(db.todos).get();
    expect(all, hasLength(1));
    expect(all.first.title, '테스트 할 일');
  });
}
```

### 7. 데이터베이스 파일 경로 디버깅

```dart
import 'package:path_provider/path_provider.dart';
import 'package:path/path.dart' as p;

Future<void> printDbPath() async {
  final dir = await getApplicationDocumentsDirectory();
  final path = p.join(dir.path, 'app_database.db');
  debugPrint('DB 경로: $path');
}
```

---

## 정리

Drift는 SQLite 위에서 타입 안전성, 반응형 스트림, 안전한 마이그레이션을 동시에 제공하는 Flutter 최고의 관계형 로컬 데이터베이스 솔루션입니다.

| 기능 | Drift 제공 방식 |
|---|---|
| 타입 안전 쿼리 | 코드 생성 (`build_runner`) |
| 실시간 UI 갱신 | `Stream<List<T>>` 자동 방출 |
| 관계형 쿼리 | `join`, `leftOuterJoin` |
| 안전한 마이그레이션 | `MigrationStrategy.onUpgrade` |
| 대량 삽입 | `batch()` API |
| 멀티스레드 | `shareAcrossIsolates` |
| 테스트 | `NativeDatabase.memory()` |

Drift를 Riverpod과 결합하면, 데이터 변경 → 스트림 방출 → UI 자동 갱신이라는 완전한 반응형 아키텍처를 간결하게 구현할 수 있습니다.

## 참고 자료
- [drift | pub.dev](https://pub.dev/packages/drift)
- [drift_flutter | pub.dev](https://pub.dev/packages/drift_flutter)
- [drift_dev | pub.dev](https://pub.dev/packages/drift_dev)
