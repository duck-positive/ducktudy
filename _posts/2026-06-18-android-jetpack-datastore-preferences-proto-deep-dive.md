---
layout: post
title: "Android Jetpack DataStore 심화: SharedPreferences를 완전히 대체하는 비동기 데이터 저장소"
date: 2026-06-18
categories: [android, kotlin]
tags: [android, jetpack, datastore, sharedpreferences, kotlin, coroutines, flow, proto]
---

Android 앱 개발에서 사용자 설정이나 간단한 데이터를 로컬에 저장할 때 오랫동안 `SharedPreferences`가 사실상의 표준이었습니다. 하지만 SharedPreferences는 동기적 I/O, 타입 안전성 부재, 예외 처리 불편함 등 여러 구조적 문제를 안고 있습니다. Google은 이를 해결하기 위해 Jetpack DataStore를 발표했고, 현재는 안정적인 프로덕션 수준에 올라와 있습니다. 이 글에서는 DataStore의 내부 동작 원리부터 두 가지 구현 방식, 실전 패턴, 그리고 마이그레이션 전략까지 깊이 있게 살펴봅니다.

## SharedPreferences의 무엇이 문제인가

SharedPreferences는 처음 Android에 등장한 이래 거의 변하지 않았습니다. 이 API의 핵심적인 문제는 다음과 같습니다.

**1. 동기적(Blocking) I/O**: `commit()`은 메인 스레드를 블로킹하며 I/O 작업을 수행합니다. `apply()`는 비동기처럼 보이지만 내부적으로 비동기 큐를 사용하고, 액티비티 생명주기(`onStop`)에서 완료를 기다리며 ANR(Application Not Responding)을 유발할 수 있습니다.

**2. 타입 안전성 없음**: 키를 문자열로 관리하기 때문에 오타가 발생해도 컴파일 타임에 잡히지 않으며, 잘못된 타입을 꺼내면 런타임 크래시가 발생합니다.

**3. 에러 처리의 어려움**: `SharedPreferences`는 실패 시 예외를 던지지 않고 기본값을 반환하거나 조용히 실패하므로 데이터 손실이 발생해도 알아채기 어렵습니다.

**4. 멀티스레드 일관성 미보장**: 여러 스레드에서 동시에 쓰기 작업 시 데이터 일관성을 직접 제어해야 합니다.

## DataStore란 무엇인가

Jetpack DataStore는 Kotlin Coroutines와 Flow를 기반으로 구축된 데이터 저장 솔루션입니다. 핵심 특징은 다음과 같습니다.

- **비동기 API**: 모든 읽기/쓰기가 suspend 함수 또는 Flow로 제공되어 백그라운드에서 안전하게 실행됩니다.
- **트랜잭셔널 업데이트**: 원자적(atomic) 업데이트를 보장하여 중간 상태의 데이터가 노출되지 않습니다.
- **에러 처리**: I/O 오류 시 예외를 발생시켜 명확하게 처리할 수 있습니다.
- **두 가지 구현 방식**: 간단한 키-값 쌍을 위한 **Preferences DataStore**와 타입 안전성이 보장되는 **Proto DataStore**를 제공합니다.

## Preferences DataStore: 키-값 저장의 현대적 방식

### 의존성 추가

```kotlin
// build.gradle.kts (Module: app)
dependencies {
    implementation("androidx.datastore:datastore-preferences:1.1.1")
}
```

### DataStore 인스턴스 생성과 데이터 읽기/쓰기

DataStore는 싱글톤으로 관리하는 것이 중요합니다. 동일한 파일을 대상으로 여러 인스턴스를 생성하면 데이터 손상이 발생합니다. Kotlin의 `by dataStore` 위임(delegate)을 사용하면 파일 수준의 싱글톤을 간단하게 보장할 수 있습니다.

```kotlin
import android.content.Context
import androidx.datastore.core.DataStore
import androidx.datastore.preferences.core.*
import androidx.datastore.preferences.preferencesDataStore
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.catch
import kotlinx.coroutines.flow.map
import java.io.IOException

// 파일 최상단(확장 프로퍼티)에 선언 — 앱 전체에서 하나의 인스턴스만 보장
private val Context.settingsDataStore: DataStore<Preferences>
        by preferencesDataStore(name = "settings")

class SettingsRepository(private val context: Context) {

    // 타입별 키 정의
    private object Keys {
        val DARK_MODE   = booleanPreferencesKey("dark_mode")
        val USER_NAME   = stringPreferencesKey("user_name")
        val FONT_SIZE   = intPreferencesKey("font_size")
        val LAST_SYNC   = longPreferencesKey("last_sync_timestamp")
    }

    // ────── 읽기: Flow로 반응형 스트림 노출 ──────
    val darkModeFlow: Flow<Boolean> = context.settingsDataStore.data
        .catch { cause ->
            // IOException은 복구 가능: 빈 Preferences 반환
            // 다른 예외는 상위로 전파
            if (cause is IOException) emit(emptyPreferences())
            else throw cause
        }
        .map { prefs -> prefs[Keys.DARK_MODE] ?: false }

    val userNameFlow: Flow<String> = context.settingsDataStore.data
        .catch { if (it is IOException) emit(emptyPreferences()) else throw it }
        .map { prefs -> prefs[Keys.USER_NAME] ?: "Guest" }

    // ────── 쓰기: updateData()로 원자적 업데이트 ──────
    suspend fun setDarkMode(enabled: Boolean) {
        context.settingsDataStore.edit { prefs ->
            prefs[Keys.DARK_MODE] = enabled
        }
    }

    suspend fun setUserName(name: String) {
        context.settingsDataStore.edit { prefs ->
            prefs[Keys.USER_NAME] = name
        }
    }

    // ────── 원자적 Read-Modify-Write ──────
    suspend fun incrementFontSize() {
        context.settingsDataStore.edit { prefs ->
            val current = prefs[Keys.FONT_SIZE] ?: 14
            // edit 블록 내에서 읽고 쓰면 원자성 보장
            prefs[Keys.FONT_SIZE] = (current + 2).coerceAtMost(32)
        }
    }

    // ────── 특정 키만 삭제 ──────
    suspend fun clearUserName() {
        context.settingsDataStore.edit { prefs ->
            prefs.remove(Keys.USER_NAME)
        }
    }
}
```

### ViewModel과 Compose에서 소비하기

```kotlin
@HiltViewModel
class SettingsViewModel @Inject constructor(
    private val repository: SettingsRepository
) : ViewModel() {

    // UI 레이어에서는 StateFlow로 변환하여 사용
    val darkMode: StateFlow<Boolean> = repository.darkModeFlow
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = false
        )

    val userName: StateFlow<String> = repository.userNameFlow
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = "Guest"
        )

    fun onDarkModeToggled(enabled: Boolean) {
        viewModelScope.launch {
            repository.setDarkMode(enabled)
        }
    }
}

// Composable에서 사용
@Composable
fun SettingsScreen(viewModel: SettingsViewModel = hiltViewModel()) {
    val darkMode by viewModel.darkMode.collectAsStateWithLifecycle()
    val userName by viewModel.userName.collectAsStateWithLifecycle()

    Column(modifier = Modifier.padding(16.dp)) {
        Text("안녕하세요, $userName 님")
        Switch(
            checked = darkMode,
            onCheckedChange = viewModel::onDarkModeToggled
        )
    }
}
```

## Proto DataStore: 타입 안전한 객체 저장

Preferences DataStore가 키-값 쌍이라면, Proto DataStore는 Protocol Buffers(protobuf) 스키마를 통해 타입이 완전히 정의된 객체를 저장합니다. 컴파일 타임에 모든 타입이 검증되므로 런타임 오류 가능성이 크게 줄어듭니다.

### 의존성 및 플러그인 설정

```kotlin
// build.gradle.kts (Project)
plugins {
    id("com.google.protobuf") version "0.9.4" apply false
}

// build.gradle.kts (Module: app)
plugins {
    id("com.google.protobuf") version "0.9.4"
}

dependencies {
    implementation("androidx.datastore:datastore:1.1.1")
    implementation("com.google.protobuf:protobuf-javalite:3.25.1")
}

protobuf {
    protoc {
        artifact = "com.google.protobuf:protoc:3.25.1"
    }
    generateProtoTasks {
        all().forEach { task ->
            task.builtins {
                create("java") { option("lite") }
            }
        }
    }
}
```

### Proto 스키마 정의

```protobuf
// src/main/proto/user_settings.proto
syntax = "proto3";

option java_package = "com.example.app.data";
option java_multiple_files = true;

enum Theme {
    THEME_UNSPECIFIED = 0;
    THEME_LIGHT       = 1;
    THEME_DARK        = 2;
    THEME_SYSTEM      = 3;
}

message UserSettings {
    string  user_name      = 1;
    Theme   theme          = 2;
    int32   font_size      = 3;
    bool    notifications  = 4;
    repeated string recent_searches = 5;
}
```

### Serializer 구현과 Repository

```kotlin
import androidx.datastore.core.CorruptionException
import androidx.datastore.core.DataStore
import androidx.datastore.core.Serializer
import androidx.datastore.dataStore
import com.google.protobuf.InvalidProtocolBufferException
import java.io.InputStream
import java.io.OutputStream

// ────── Serializer: 직렬화/역직렬화 규칙 정의 ──────
object UserSettingsSerializer : Serializer<UserSettings> {

    override val defaultValue: UserSettings = UserSettings.getDefaultInstance()

    override suspend fun readFrom(input: InputStream): UserSettings {
        return try {
            UserSettings.parseFrom(input)
        } catch (e: InvalidProtocolBufferException) {
            // 파일이 손상된 경우 CorruptionException으로 감싸 DataStore에 알림
            throw CorruptionException("Cannot read proto.", e)
        }
    }

    override suspend fun writeTo(t: UserSettings, output: OutputStream) {
        t.writeTo(output)
    }
}

// ────── 싱글톤 DataStore 인스턴스 ──────
private val Context.userSettingsStore: DataStore<UserSettings>
        by dataStore(
            fileName = "user_settings.pb",
            serializer = UserSettingsSerializer
        )

// ────── Repository ──────
class UserSettingsRepository(private val context: Context) {

    val settingsFlow: Flow<UserSettings> = context.userSettingsStore.data
        .catch { cause ->
            if (cause is IOException) emit(UserSettings.getDefaultInstance())
            else throw cause
        }

    // 스키마가 복잡해도 toBuilder()로 부분 업데이트 가능
    suspend fun updateTheme(theme: Theme) {
        context.userSettingsStore.updateData { current ->
            current.toBuilder()
                .setTheme(theme)
                .build()
        }
    }

    suspend fun addRecentSearch(query: String) {
        context.userSettingsStore.updateData { current ->
            val updatedSearches = current.recentSearchesList
                .toMutableList()
                .apply {
                    remove(query)          // 중복 제거
                    add(0, query)          // 맨 앞에 추가
                    if (size > 10) removeLast()  // 최대 10개 유지
                }
            current.toBuilder()
                .clearRecentSearches()
                .addAllRecentSearches(updatedSearches)
                .build()
        }
    }

    // ────── 두 필드를 하나의 트랜잭션으로 업데이트 ──────
    suspend fun updateProfile(userName: String, fontSize: Int) {
        context.userSettingsStore.updateData { current ->
            current.toBuilder()
                .setUserName(userName)
                .setFontSize(fontSize.coerceIn(10, 32))
                .build()
        }
    }
}
```

## SharedPreferences에서 DataStore로 마이그레이션

기존 앱에 이미 SharedPreferences 데이터가 있다면 `SharedPreferencesMigration`을 사용해 무손실 마이그레이션이 가능합니다. DataStore는 마이그레이션을 단 한 번만 실행하며, 완료 후에는 기존 SharedPreferences 파일을 삭제합니다.

```kotlin
private val Context.migratedDataStore: DataStore<Preferences>
        by preferencesDataStore(
            name = "settings",
            produceMigrations = { context ->
                listOf(
                    SharedPreferencesMigration(
                        context = context,
                        sharedPreferencesName = "my_prefs"  // 기존 파일명
                        // keysToMigrate를 지정하면 특정 키만 이전 가능
                    )
                )
            }
        )
```

## 멀티프로세스 환경: MultiProcessDataStore

Android 1.1.0부터 멀티프로세스 지원이 추가되었습니다. 위젯, WorkManager Job, 별도 프로세스 서비스처럼 여러 프로세스가 공유 데이터에 접근해야 할 때 사용합니다.

```kotlin
// 멀티프로세스 DataStore — 의존성: datastore:1.1.0+
private val Context.multiProcStore: DataStore<Preferences>
        by multiProcessPreferencesDataStore(name = "shared_settings")
```

일반 DataStore와 API는 동일하지만 내부적으로 파일 잠금(file lock)을 사용하여 프로세스 간 쓰기 충돌을 방지합니다. 단, 성능 오버헤드가 있으므로 단일 프로세스 앱이라면 일반 DataStore를 사용하세요.

## 주의사항 및 실전 팁

**1. 인스턴스는 반드시 싱글톤으로**: `by preferencesDataStore` 또는 `by dataStore` 위임을 파일 레벨 확장 프로퍼티로 선언하면 JVM이 보장하는 초기화 단 한 번을 활용할 수 있습니다. DI(의존성 주입)를 쓴다면 Hilt의 `@Singleton`으로 스코핑하세요.

**2. 대용량 데이터는 DataStore에 두지 마세요**: DataStore는 설정, 사용자 선호, 세션 토큰처럼 작고 자주 바뀌지 않는 데이터에 적합합니다. 수백 KB 이상의 데이터라면 Room 데이터베이스를 사용하세요.

**3. `edit { }` 블록 내에서 긴 연산 금지**: `edit` 람다는 코루틴의 Mutex로 보호됩니다. 이 안에서 네트워크 요청 등 오래 걸리는 작업을 하면 모든 쓰기 작업이 직렬화되어 병목이 생깁니다. 계산은 밖에서 끝내고 결과만 넣으세요.

**4. `first()` 대신 `Flow`를 구독하세요**: 일회성으로 값을 읽을 때 `.first()`를 쓰고 싶은 충동이 있지만, Flow를 구독하고 `stateIn`으로 변환하면 데이터 변경을 자동으로 반영받을 수 있습니다. 정말 일회성이 필요한 경우에만 `.first()`를 사용하세요.

**5. 테스트에서는 `TestScope` 활용**: 테스트 코드에서 DataStore를 다룰 때는 `preferencesTestDataStore` 또는 인메모리 구현을 사용하거나, 실제 파일을 임시 디렉터리에 생성하고 `cleanUpAfterEachTest`로 정리하는 패턴을 사용하세요.

```kotlin
// 단위 테스트 예시
class SettingsRepositoryTest {

    private val testScope = TestScope(UnconfinedTestDispatcher())
    private lateinit var dataStore: DataStore<Preferences>
    private lateinit var repository: SettingsRepository

    @Before
    fun setUp() {
        dataStore = PreferenceDataStoreFactory.create(
            scope = testScope,
            produceFile = { File(testScope.testDirectory, "test_settings.preferences_pb") }
        )
        repository = SettingsRepository(dataStore)
    }

    @Test
    fun `다크 모드 설정이 올바르게 저장되고 읽혀야 한다`() = testScope.runTest {
        repository.setDarkMode(true)
        val result = repository.darkModeFlow.first()
        assertThat(result).isTrue()
    }
}
```

## 마무리

Jetpack DataStore는 SharedPreferences의 오랜 문제들을 코루틴과 Flow 생태계와 자연스럽게 결합해 해결한 솔루션입니다. Preferences DataStore는 기존 SharedPreferences 사용자에게 낮은 진입 장벽을 제공하고, Proto DataStore는 복잡한 설정 구조를 타입 안전하게 관리할 수 있게 해줍니다. 새 프로젝트를 시작한다면 처음부터 DataStore를 채택하고, 기존 프로젝트라면 `SharedPreferencesMigration`을 활용해 점진적으로 전환하는 것을 권장합니다.

## 참고 자료
- [DataStore 공식 문서 — Android Developers](https://developer.android.com/topic/libraries/architecture/datastore)
- [Working with Preferences DataStore Codelab](https://developer.android.com/codelabs/android-preferences-datastore)
- [Working with Proto DataStore Codelab](https://developer.android.com/codelabs/android-proto-datastore)
