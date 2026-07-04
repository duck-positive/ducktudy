---
layout: post
title: "Android Jetpack Glance 심화: Compose 기반 홈 화면 위젯 완전 정복"
date: 2026-07-04
categories: [android, flutter]
tags: [android, jetpack, glance, app-widget, compose, kotlin, remoteviews]
---

앱 아이콘과 같은 홈 화면 위젯은 Android 1.5 시절부터 존재했지만, 그 구현 방식은 오랫동안 개발자에게 큰 고통이었습니다. `RemoteViews`라는 제한된 뷰 계층, XML 레이아웃 강제, 복잡한 `AppWidgetProvider` 생명주기, 불친절한 업데이트 API까지—Android의 다른 영역이 Jetpack Compose로 현대화되는 동안 홈 화면 위젯만은 낙오되어 있었습니다. **Jetpack Glance**는 바로 이 고통을 해결하기 위해 탄생한 라이브러리입니다.

## 1. Jetpack Glance란 무엇인가?

Jetpack Glance는 Jetpack Compose 런타임 위에 구축된 선언형 UI 프레임워크로, **홈 화면 앱 위젯**과 **Wear OS 타일**을 Compose 스타일로 작성할 수 있게 해줍니다. 핵심은 Glance가 Compose처럼 보이지만 내부적으로는 여전히 `RemoteViews`로 변환된다는 점입니다. 즉 개발자는 선언형 코드를 쓰고, Glance가 그것을 시스템이 요구하는 `RemoteViews`로 자동 변환합니다.

```
개발자 코드 (Glance Composable)
        ↓  Glance 변환 레이어
RemoteViews (시스템에 전달)
        ↓
홈 화면 렌더링
```

2024년 10월 기준 안정 버전은 `1.1.1`이며, 2026년 7월에 `1.3.0-alpha02`가 출시되었습니다.

### Glance vs 기존 AppWidget 비교

| 항목 | 기존 AppWidget | Jetpack Glance |
|---|---|---|
| UI 작성 | XML 레이아웃 | Kotlin Composable |
| 상태 관리 | 직접 구현 | GlanceStateDefinition |
| 업데이트 | sendBroadcast + update() | GlanceAppWidgetManager |
| 클릭 핸들링 | PendingIntent 직접 구성 | actionRunCallback, actionStartActivity |
| 미리보기 | 별도 XML | @Preview + 생성 도구 |

---

## 2. 왜 Glance가 필요한가?

### 문제 1: RemoteViews의 레이아웃 제약

기존 위젯은 `LinearLayout`, `FrameLayout`, `TextView`, `ImageView` 등 극히 제한된 뷰만 사용할 수 있었습니다. `ConstraintLayout`이나 `RecyclerView`는 물론, 커스텀 뷰 자체가 불가능했습니다. Glance는 `Column`, `Row`, `Box`, `LazyColumn` 같은 Compose 스타일 레이아웃을 제공하고 내부에서 RemoteViews로 변환합니다.

### 문제 2: 복잡한 상태 업데이트

기존에는 위젯 상태를 `SharedPreferences`에 직접 저장하고, 변경 시마다 `AppWidgetManager.updateAppWidget()`을 명시적으로 호출해야 했습니다. Glance는 `GlanceStateDefinition`과 `DataStore`를 통해 상태 변경 시 자동 재구성(recomposition)을 지원합니다.

### 문제 3: 생명주기 관리

`AppWidgetProvider`는 `BroadcastReceiver`를 상속하므로 비동기 작업 처리가 매우 까다로웠습니다. Glance의 `GlanceAppWidget`은 코루틴 기반으로 동작해 `suspend` 함수와 Flow를 자연스럽게 활용할 수 있습니다.

---

## 3. 프로젝트 설정

`build.gradle.kts`에 의존성을 추가합니다:

```kotlin
dependencies {
    implementation("androidx.glance:glance-appwidget:1.1.1")
    // Material 3 테마를 위젯에 적용하려면
    implementation("androidx.glance:glance-material3:1.1.1")
}
```

AndroidManifest.xml에 receiver를 등록합니다:

```xml
<receiver
    android:name=".widget.TodoGlanceWidgetReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
    </intent-filter>
    <meta-data
        android:name="android.appwidget.provider"
        android:resource="@xml/todo_widget_info" />
</receiver>
```

`res/xml/todo_widget_info.xml`:

```xml
<appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
    android:minWidth="250dp"
    android:minHeight="110dp"
    android:targetCellWidth="4"
    android:targetCellHeight="2"
    android:updatePeriodMillis="3600000"
    android:description="@string/widget_description"
    android:widgetCategory="home_screen" />
```

---

## 4. 실제 구현 예제 1: 할 일 목록 위젯

가장 실용적인 예제로, 앱의 DataStore에서 할 일 목록을 읽어 위젯에 표시하고, 완료 버튼을 클릭하면 상태를 업데이트하는 전체 흐름을 구현합니다.

```kotlin
// TodoGlanceWidget.kt

import androidx.compose.runtime.Composable
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.datastore.preferences.core.Preferences
import androidx.datastore.preferences.core.stringPreferencesKey
import androidx.glance.*
import androidx.glance.action.ActionParameters
import androidx.glance.action.clickable
import androidx.glance.appwidget.*
import androidx.glance.appwidget.action.ActionCallback
import androidx.glance.appwidget.action.actionRunCallback
import androidx.glance.appwidget.state.updateAppWidgetState
import androidx.glance.layout.*
import androidx.glance.state.GlanceStateDefinition
import androidx.glance.state.PreferencesGlanceStateDefinition
import androidx.glance.text.FontWeight
import androidx.glance.text.Text
import androidx.glance.text.TextStyle
import kotlinx.serialization.decodeFromString
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json

// 상태 정의: DataStore Preferences 사용
class TodoGlanceWidget : GlanceAppWidget() {

    // Glance가 사용할 StateDefinition — DataStore 기반
    override val stateDefinition: GlanceStateDefinition<Preferences> =
        PreferencesGlanceStateDefinition

    companion object {
        val TODO_LIST_KEY = stringPreferencesKey("todo_list")
    }

    @Composable
    override fun Content() {
        // currentState로 DataStore Preferences에 접근
        val prefs = currentState<Preferences>()
        val todoJson = prefs[TODO_LIST_KEY] ?: "[]"
        val todos: List<TodoItem> = runCatching {
            Json.decodeFromString(todoJson)
        }.getOrDefault(emptyList())

        GlanceTheme {
            Column(
                modifier = GlanceModifier
                    .fillMaxSize()
                    .background(GlanceTheme.colors.widgetBackground)
                    .padding(12.dp)
            ) {
                // 헤더
                Row(
                    modifier = GlanceModifier.fillMaxWidth(),
                    verticalAlignment = Alignment.CenterVertically
                ) {
                    Text(
                        text = "오늘의 할 일",
                        style = TextStyle(
                            fontWeight = FontWeight.Bold,
                            fontSize = 16.sp,
                            color = GlanceTheme.colors.onSurface
                        ),
                        modifier = GlanceModifier.defaultWeight()
                    )
                    // 새로고침 버튼
                    Image(
                        provider = ImageProvider(R.drawable.ic_refresh),
                        contentDescription = "새로고침",
                        modifier = GlanceModifier
                            .size(24.dp)
                            .clickable(actionRunCallback<RefreshTodoAction>())
                    )
                }

                Spacer(modifier = GlanceModifier.height(8.dp))

                if (todos.isEmpty()) {
                    Text(
                        text = "할 일이 없습니다 🎉",
                        style = TextStyle(
                            color = GlanceTheme.colors.secondaryText,
                            fontSize = 13.sp
                        )
                    )
                } else {
                    // LazyColumn으로 목록 표시
                    LazyColumn {
                        items(todos.take(5)) { todo ->
                            TodoRow(todo = todo)
                        }
                    }
                }
            }
        }
    }
}

@Composable
private fun TodoRow(todo: TodoItem) {
    val checkIcon = if (todo.done) R.drawable.ic_check_circle else R.drawable.ic_circle

    Row(
        modifier = GlanceModifier
            .fillMaxWidth()
            .padding(vertical = 4.dp)
            .clickable(
                actionRunCallback<ToggleTodoAction>(
                    parameters = actionParametersOf(
                        ToggleTodoAction.TODO_ID_KEY to todo.id
                    )
                )
            ),
        verticalAlignment = Alignment.CenterVertically
    ) {
        Image(
            provider = ImageProvider(checkIcon),
            contentDescription = if (todo.done) "완료" else "미완료",
            modifier = GlanceModifier.size(20.dp)
        )
        Spacer(modifier = GlanceModifier.width(8.dp))
        Text(
            text = todo.title,
            style = TextStyle(
                fontSize = 13.sp,
                color = if (todo.done)
                    GlanceTheme.colors.secondaryText
                else
                    GlanceTheme.colors.onSurface
            ),
            maxLines = 1
        )
    }
}

// ---- ActionCallback 구현 ----

// 할 일 완료 토글 액션
class ToggleTodoAction : ActionCallback {
    companion object {
        val TODO_ID_KEY = ActionParameters.Key<String>("todo_id")
    }

    override suspend fun onAction(
        context: Context,
        glanceId: GlanceId,
        parameters: ActionParameters
    ) {
        val todoId = parameters[TODO_ID_KEY] ?: return

        // DataStore에서 현재 상태를 읽고, 해당 항목의 done 상태를 토글
        updateAppWidgetState(context, PreferencesGlanceStateDefinition, glanceId) { prefs ->
            val current = prefs[TodoGlanceWidget.TODO_LIST_KEY] ?: "[]"
            val todos: List<TodoItem> = Json.decodeFromString(current)
            val updated = todos.map { if (it.id == todoId) it.copy(done = !it.done) else it }
            prefs.toMutablePreferences().apply {
                this[TodoGlanceWidget.TODO_LIST_KEY] = Json.encodeToString(updated)
            }
        }

        // 상태가 변경되면 위젯을 다시 그림
        TodoGlanceWidget().update(context, glanceId)
    }
}

// 새로고침 액션 — 앱 Repository에서 최신 데이터를 가져와 위젯 상태를 갱신
class RefreshTodoAction : ActionCallback {
    override suspend fun onAction(
        context: Context,
        glanceId: GlanceId,
        parameters: ActionParameters
    ) {
        val repository = TodoRepository(context)
        val freshTodos = repository.getLatestTodos()

        updateAppWidgetState(context, PreferencesGlanceStateDefinition, glanceId) { prefs ->
            prefs.toMutablePreferences().apply {
                this[TodoGlanceWidget.TODO_LIST_KEY] = Json.encodeToString(freshTodos)
            }
        }
        TodoGlanceWidget().update(context, glanceId)
    }
}

// GlanceAppWidgetReceiver: AppWidgetProvider 역할
class TodoGlanceWidgetReceiver : GlanceAppWidgetReceiver() {
    override val glanceAppWidget: GlanceAppWidget = TodoGlanceWidget()
}
```

---

## 5. 실제 구현 예제 2: 모든 위젯 인스턴스 일괄 업데이트 (WorkManager 연동)

위젯은 홈 화면에 여러 개가 동시에 올라가 있을 수 있습니다. `GlanceAppWidgetManager`를 사용하면 특정 위젯 클래스의 모든 인스턴스를 한 번에 업데이트할 수 있습니다. 이를 WorkManager와 결합해 주기적 데이터 동기화를 구현합니다.

```kotlin
// TodoWidgetSyncWorker.kt

import android.content.Context
import androidx.glance.appwidget.GlanceAppWidgetManager
import androidx.glance.appwidget.state.updateAppWidgetState
import androidx.glance.state.PreferencesGlanceStateDefinition
import androidx.work.*
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json
import java.util.concurrent.TimeUnit

class TodoWidgetSyncWorker(
    appContext: Context,
    params: WorkerParameters
) : CoroutineWorker(appContext, params) {

    override suspend fun doWork(): Result {
        return try {
            // 원격 서버 또는 로컬 DB에서 최신 할 일 목록 조회
            val repository = TodoRepository(applicationContext)
            val latestTodos = repository.getLatestTodos()
            val encodedTodos = Json.encodeToString(latestTodos)

            // 홈 화면에 배치된 TodoGlanceWidget의 모든 인스턴스 ID 조회
            val manager = GlanceAppWidgetManager(applicationContext)
            val glanceIds = manager.getGlanceIds(TodoGlanceWidget::class.java)

            // 각 인스턴스의 DataStore 상태를 업데이트
            glanceIds.forEach { glanceId ->
                updateAppWidgetState(
                    context = applicationContext,
                    definition = PreferencesGlanceStateDefinition,
                    id = glanceId
                ) { prefs ->
                    prefs.toMutablePreferences().apply {
                        this[TodoGlanceWidget.TODO_LIST_KEY] = encodedTodos
                    }
                }
            }

            // 모든 인스턴스를 재구성(recompose)하여 UI 반영
            val widget = TodoGlanceWidget()
            glanceIds.forEach { glanceId ->
                widget.update(applicationContext, glanceId)
            }

            Result.success()
        } catch (e: Exception) {
            // 재시도 가능한 오류라면 Result.retry()
            Result.retry()
        }
    }

    companion object {
        const val WORK_NAME = "todo_widget_sync"

        fun schedule(context: Context) {
            val request = PeriodicWorkRequestBuilder<TodoWidgetSyncWorker>(
                repeatInterval = 15,
                repeatIntervalTimeUnit = TimeUnit.MINUTES
            )
                .setConstraints(
                    Constraints.Builder()
                        .setRequiredNetworkType(NetworkType.CONNECTED)
                        .build()
                )
                .setBackoffCriteria(
                    BackoffPolicy.EXPONENTIAL,
                    WorkRequest.MIN_BACKOFF_MILLIS,
                    TimeUnit.MILLISECONDS
                )
                .build()

            WorkManager.getInstance(context).enqueueUniquePeriodicWork(
                WORK_NAME,
                ExistingPeriodicWorkPolicy.UPDATE,
                request
            )
        }

        fun scheduleImmediate(context: Context) {
            val request = OneTimeWorkRequestBuilder<TodoWidgetSyncWorker>()
                .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
                .build()
            WorkManager.getInstance(context).enqueue(request)
        }
    }
}
```

Application 클래스 또는 앱 시작 지점에서 스케줄을 등록합니다:

```kotlin
// MyApplication.kt
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        TodoWidgetSyncWorker.schedule(this)
    }
}
```

---

## 6. 주의사항 및 실전 팁

### 팁 1: Glance Composable은 일반 Compose와 다름

Glance는 `androidx.compose.ui` 위에 구축되지 않고 **별도의 Glance Composable 계층**을 사용합니다. 일반 Compose의 `Text`, `Row`, `Column`이 아닌 `androidx.glance.text.Text`, `androidx.glance.layout.Row`를 import해야 합니다. 혼용하면 컴파일 오류가 발생합니다.

```kotlin
// ❌ 잘못된 import
import androidx.compose.foundation.layout.Row
import androidx.compose.material3.Text

// ✅ 올바른 import
import androidx.glance.layout.Row
import androidx.glance.text.Text
```

### 팁 2: RemoteViews 제약을 항상 염두에 두기

Glance가 Compose 스타일을 제공하더라도 최종 출력은 RemoteViews입니다. 따라서 다음은 불가능합니다:
- **커스텀 드로잉(Canvas)**
- **애니메이션** (Android 12 미만에서는 특히 제한적)
- **무한 스크롤** (LazyColumn은 5~10개 항목 정도가 적절)
- **복잡한 그라디언트 배경** (API 수준에 따라 다름)

### 팁 3: `updateAppWidgetState` 후 반드시 `update()` 호출

`updateAppWidgetState`는 DataStore 상태만 업데이트합니다. 실제로 위젯 UI를 다시 그리려면 그 뒤에 반드시 `GlanceAppWidget().update(context, glanceId)`를 호출해야 합니다. 이 두 단계를 묶어 헬퍼 함수로 만들어 두면 실수를 줄일 수 있습니다.

### 팁 4: Android 12(API 31) 대응

Android 12부터 위젯은 `rounded corners`와 `widget padding`이 시스템에 의해 자동 적용됩니다. Glance 1.1.x에서는 이를 GlanceTheme으로 대응합니다. `GlanceTheme { ... }` 블록 안에 UI를 작성하면 시스템 테마(라이트/다크 모드)와 Material You 색상을 자동으로 따릅니다.

### 팁 5: 위젯 미리보기 (Generated Previews)

Android 14(API 34)부터 `@GlancePreview` 어노테이션으로 위젯 선택 화면에 실제 렌더링 미리보기를 제공할 수 있습니다. 이는 위젯 채택률을 크게 높이는 UX 개선입니다. 빌드 시 자동으로 미리보기 이미지가 생성되므로 별도 XML `previewImage` 리소스가 필요 없습니다.

### 팁 6: `onUpdate`에서 코루틴 사용

`GlanceAppWidgetReceiver`의 `onUpdate`는 브로드캐스트 리시버에서 호출되므로 기본적으로 메인 스레드에서 실행됩니다. 무거운 작업은 `WorkManager`에 위임하고, `onUpdate`에서는 WorkManager 잡을 트리거만 하는 패턴을 추천합니다.

```kotlin
class TodoGlanceWidgetReceiver : GlanceAppWidgetReceiver() {
    override val glanceAppWidget = TodoGlanceWidget()

    override fun onUpdate(
        context: Context,
        appWidgetManager: AppWidgetManager,
        appWidgetIds: IntArray
    ) {
        super.onUpdate(context, appWidgetManager, appWidgetIds)
        // 무거운 작업은 WorkManager에 위임
        TodoWidgetSyncWorker.scheduleImmediate(context)
    }
}
```

---

## 7. 정리

Jetpack Glance는 오랫동안 Android 개발의 고통지점이었던 홈 화면 위젯 개발을 현대화합니다. RemoteViews의 제약은 여전히 존재하지만, 선언형 Kotlin API, DataStore 기반 상태 관리, 코루틴 지원, GlanceTheme의 동적 색상 적용 덕분에 기존 대비 훨씬 생산적인 위젯 개발이 가능해졌습니다.

핵심 요점 정리:
- `GlanceAppWidget.Content()`에서 선언형으로 UI를 작성
- `GlanceStateDefinition`과 `updateAppWidgetState`로 상태 관리
- `actionRunCallback<T>()`로 버튼 클릭 이벤트 처리
- `GlanceAppWidgetManager.getGlanceIds()`로 모든 인스턴스 일괄 업데이트
- WorkManager와 결합해 주기적 데이터 동기화
- Glance Composable import는 `androidx.glance.*` 패키지를 사용

## 참고 자료

- [Create an app widget with Glance — Android Developers](https://developer.android.com/develop/ui/compose/glance/create-app-widget)
- [Manage and update GlanceAppWidget — Android Developers](https://developer.android.com/develop/ui/compose/glance/glance-app-widget)
- [Handle user interaction in Glance — Android Developers](https://developer.android.com/develop/ui/compose/glance/user-interaction)
- [Glance 릴리즈 노트 — Jetpack](https://developer.android.com/jetpack/androidx/releases/glance)
