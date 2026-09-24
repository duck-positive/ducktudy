---
layout: post
title: "Android AppSearch 심화: @Document 스키마·인덱싱·쿼리로 온디바이스 전문 검색 완전 정복"
date: 2026-09-24
categories: [android, kotlin]
tags: [android, appsearch, jetpack, full-text-search, kotlin, on-device-search, indexing]
---

Android 앱에 검색 기능을 추가할 때 가장 먼저 떠오르는 방법은 Room의 FTS(Full-Text Search)나 단순한 `LIKE` 쿼리입니다. 하지만 접두사 검색, 결과 스니펫 강조, 관련도 기반 랭킹처럼 사용자가 기대하는 고급 검색 기능이 필요한 순간, Room FTS는 한계를 드러냅니다. **Android AppSearch**는 바로 이 간극을 채우기 위해 Jetpack 팀이 설계한 온디바이스 전문 검색 라이브러리입니다.

---

## AppSearch란?

AppSearch는 로컬에 저장된 구조화 데이터를 인덱싱하고, 전문 검색(Full-Text Search)으로 빠르게 조회할 수 있도록 설계된 Jetpack 라이브러리입니다. Android 12부터는 시스템 서비스에 통합되어 여러 앱이 서로의 검색 인덱스를 공유하거나, 기기 어시스턴트에 콘텐츠를 노출하는 것도 가능해졌습니다.

AppSearch가 지원하는 스토리지 구현은 세 가지입니다.

| 스토리지 | 최소 API | 특징 |
|----------|----------|------|
| **LocalStorage** | 제한 없음 | 앱 프로세스 내에서 동작, APK 크기 약간 증가 |
| **PlatformStorage** | API 31 (Android 12) | 시스템 서비스 활용, 앱 간 데이터 공유 가능 |
| **PlayServicesStorage** | 제한 없음 (GMS 필요) | 구형 기기에서 PlatformStorage 수준의 기능 제공 |

### AppSearch의 핵심 개념

- **Document**: AppSearch에 저장·인덱싱되는 데이터 단위. `@Document` 애노테이션으로 Kotlin 클래스를 매핑합니다.
- **Schema**: Document 클래스 구조를 AppSearch에 등록한 타입 정보입니다.
- **Namespace**: 같은 데이터베이스 안에서 Document를 논리적으로 분리하는 그룹입니다.
- **AppSearchSession**: 데이터베이스 연결을 나타내며, 모든 읽기·쓰기 작업의 진입점입니다.
- **SearchSpec**: 검색 방식, 필터, 랭킹 전략 등을 담은 설정 객체입니다.

---

## 왜 AppSearch를 선택해야 하는가?

### Room FTS와의 비교

Room의 `@Fts4` / `@Fts5`도 전문 검색을 제공하지만, 아래 표와 같이 AppSearch가 더 풍부한 기능을 제공합니다.

| 기능 | Room FTS | AppSearch |
|------|----------|-----------|
| 접두사 검색 | 제한적 | ✅ (INDEXING_TYPE_PREFIXES) |
| 스니펫 강조 | ❌ | ✅ (MatchInfo API) |
| 관련도 랭킹 | 기본 RANK() | ✅ (다양한 전략) |
| 속성별 가중치 | ❌ | ✅ (PropertyWeight) |
| 시스템 검색 통합 | ❌ | ✅ (PlatformStorage) |
| 앱 간 데이터 공유 | ❌ | ✅ |
| 임베딩 벡터 검색 | ❌ | ✅ (1.1.0+, 실험적) |

### AppSearch를 선택해야 하는 상황

- **노트 앱, 이메일 클라이언트, 할 일 관리 앱**처럼 사용자가 직접 입력한 콘텐츠를 검색하는 경우
- 검색 결과에서 매칭된 부분을 **하이라이트**로 표시해야 하는 경우
- Android 12 이상 기기에서 **기기 어시스턴트**에 앱 콘텐츠를 노출하고 싶은 경우
- Room FTS 쿼리의 복잡성이나 성능 한계를 느끼는 경우

---

## 실제 구현 예제

### 의존성 추가

```kotlin
// build.gradle.kts (app)
dependencies {
    val appsearchVersion = "1.1.0"

    implementation("androidx.appsearch:appsearch:$appsearchVersion")
    // 애노테이션 프로세서 (Kotlin KSP 사용 시 ksp로 변경)
    kapt("androidx.appsearch:appsearch-compiler:$appsearchVersion")
    // 로컬 스토리지: API 레벨 제한 없음
    implementation("androidx.appsearch:appsearch-local-storage:$appsearchVersion")
    // 코루틴에서 ListenableFuture를 await()로 사용하기 위해 필수
    implementation("androidx.concurrent:concurrent-futures-ktx:1.2.0")
}
```

---

### 예제 1: @Document 스키마 정의와 AppSearchSession 초기화

AppSearch를 사용하는 첫 번째 단계는 인덱싱할 데이터 구조를 `@Document` 애노테이션으로 정의하고, 세션을 열어 스키마를 설정하는 것입니다.

```kotlin
import androidx.appsearch.annotation.Document
import androidx.appsearch.app.*
import androidx.appsearch.localstorage.LocalStorage
import kotlinx.coroutines.guava.await

// ① @Document 클래스 정의: 노트 앱의 메모 데이터
@Document
data class Note(
    @Document.Namespace
    val namespace: String,               // 논리적 파티션 (예: "personal", "work")

    @Document.Id
    val id: String,                      // 문서 고유 식별자

    @Document.Score
    val score: Int = 0,                  // 수동 점수 (부스팅에 활용)

    @Document.CreationTimestampMillis
    val createdAt: Long = System.currentTimeMillis(),

    // 제목: 접두사 검색 활성화 ("kott" → "kotlin" 매칭)
    @Document.StringProperty(
        indexingType = AppSearchSchema.StringPropertyConfig.INDEXING_TYPE_PREFIXES
    )
    val title: String,

    // 본문: 정확한 토큰 매칭
    @Document.StringProperty(
        indexingType = AppSearchSchema.StringPropertyConfig.INDEXING_TYPE_EXACT_TERMS
    )
    val content: String,

    // 태그: 접두사 검색 활성화
    @Document.StringProperty(
        indexingType = AppSearchSchema.StringPropertyConfig.INDEXING_TYPE_PREFIXES
    )
    val tag: String = ""
)

// ② AppSearchSession 열기 및 스키마 등록
class NoteRepository(private val context: Context) {

    private var session: AppSearchSession? = null

    suspend fun openSession() {
        // LocalStorage 사용: API 레벨 제한 없음
        session = LocalStorage.createSearchSession(
            LocalStorage.SearchContext.Builder(context, "notes_db").build()
        ).await()

        // Note 타입을 스키마로 등록
        val setSchemaRequest = SetSchemaRequest.Builder()
            .addDocumentClasses(Note::class.java)
            .build()

        session!!.setSchema(setSchemaRequest).await()
    }

    fun close() {
        session?.close()
        session = null
    }
}
```

**핵심 포인트**:

- `@Document.Namespace`는 같은 DB 안에서 데이터를 논리적으로 격리합니다. 검색 시 특정 namespace로 범위를 좁히면 성능이 향상됩니다.
- `INDEXING_TYPE_PREFIXES`는 타이핑 중 실시간 검색 경험("kott" 입력 시 "Kotlin" 노출)에 적합합니다.
- `INDEXING_TYPE_EXACT_TERMS`는 정확한 단어 단위 일치만 허용하므로 오탐(false positive)을 줄입니다.
- `LocalStorage.createSearchSession()`은 `ListenableFuture`를 반환하므로, `concurrent-futures-ktx`의 `await()`로 코루틴에서 비동기 처리합니다.

---

### 예제 2: 문서 인덱싱·검색·삭제 완전 구현

두 번째 예제에서는 `Note` 문서를 인덱싱하고, `SearchSpec`으로 검색 조건을 설정해 결과를 가져오는 전체 플로우와 ViewModel 연동까지 보여줍니다.

```kotlin
class NoteRepository(private val context: Context) {

    private var session: AppSearchSession? = null

    suspend fun openSession() { /* 예제 1과 동일 */ }

    // ③ 문서 인덱싱 (배치 처리 지원)
    suspend fun indexNotes(vararg notes: Note) {
        val request = PutDocumentsRequest.Builder()
            .addDocuments(*notes)
            .build()

        val result = session!!.put(request).await()

        // 실패한 항목만 로깅 (성공 항목은 result.successes에서 확인)
        result.failures.forEach { (id, failure) ->
            Log.e("AppSearch", "인덱싱 실패 [$id]: ${failure.errorMessage}")
        }
    }

    // ④ 전문 검색: 쿼리 + 필터 + 스니펫 강조
    suspend fun searchNotes(
        query: String,
        namespace: String = "personal",
        maxResults: Int = 20
    ): List<NoteSearchResult> {
        val searchSpec = SearchSpec.Builder()
            .addFilterNamespaces(namespace)              // namespace 범위 제한
            .setSnippetCount(maxResults)                 // 스니펫을 제공할 최대 결과 수
            .setSnippetCountPerProperty(2)               // 속성당 최대 스니펫 2개
            .setMaxSnippetSize(150)                      // 스니펫 최대 150자
            .setResultCountPerPage(maxResults)
            .setRankingStrategy(SearchSpec.RANKING_STRATEGY_RELEVANCE_SCORE)
            .build()

        val searchResults = session!!.search(query, searchSpec)
        val page = searchResults.nextPage.await()

        return page.mapNotNull { searchResult ->
            val note = searchResult.getDocument(Note::class.java)

            // 제목 필드에서 매칭된 위치 정보 추출 (UI 하이라이트용)
            val titleMatches = searchResult.getMatchInfos(Note::title.name)
            val titleSnippet = titleMatches
                .firstOrNull()
                ?.let { it.fullText.substring(it.exactMatchRange.start, it.exactMatchRange.endInclusive + 1) }

            NoteSearchResult(note = note, titleHighlight = titleSnippet)
        }
    }

    // ⑤ 특정 문서 삭제
    suspend fun deleteNote(namespace: String, id: String) {
        val request = RemoveByDocumentIdRequest.Builder(namespace)
            .addIds(id)
            .build()
        session!!.remove(request).await()
    }

    // ⑥ namespace 내 모든 문서 삭제
    suspend fun clearNamespace(namespace: String) {
        session!!.remove(
            "",
            SearchSpec.Builder().addFilterNamespaces(namespace).build()
        ).await()
    }

    fun close() {
        session?.close()
        session = null
    }
}

// 검색 결과 래퍼
data class NoteSearchResult(
    val note: Note,
    val titleHighlight: String?  // 매칭된 부분 텍스트 (UI 강조 표시용)
)

// ⑦ ViewModel에서 사용 예시
class NoteViewModel(private val repo: NoteRepository) : ViewModel() {

    val searchResults = MutableStateFlow<List<NoteSearchResult>>(emptyList())

    init {
        viewModelScope.launch {
            repo.openSession()

            // 샘플 노트 인덱싱
            repo.indexNotes(
                Note(
                    namespace = "personal", id = "note-001",
                    title = "Kotlin Coroutines 핵심 패턴",
                    content = "SupervisorScope와 CoroutineExceptionHandler를 결합해 계층적 에러를 처리한다.",
                    tag = "kotlin"
                ),
                Note(
                    namespace = "personal", id = "note-002",
                    title = "Android AppSearch 사용법",
                    content = "Document 클래스를 정의하고 LocalStorage로 세션을 열어 인덱싱한다.",
                    tag = "android"
                ),
                Note(
                    namespace = "personal", id = "note-003",
                    title = "Jetpack Compose 성능 최적화",
                    content = "derivedStateOf와 key()를 활용해 불필요한 리컴포지션을 줄인다.",
                    tag = "compose"
                )
            )
        }
    }

    fun search(query: String) {
        viewModelScope.launch {
            searchResults.value = repo.searchNotes(query)
        }
    }

    override fun onCleared() {
        repo.close()
        super.onCleared()
    }
}
```

**핵심 포인트**:

- `session!!.search()` 는 `SearchResults` 객체를 즉시 반환하며, 실제 데이터는 `nextPage.await()`를 통해 페이지 단위로 가져옵니다. 이어서 `nextPage`를 반복 호출하면 무한 스크롤 구현이 가능합니다.
- `getMatchInfos(propertyName)` 으로 어떤 필드의 어느 위치에서 검색어가 매칭됐는지 정확한 오프셋을 얻을 수 있습니다. `SpannableString`이나 `AnnotatedString`(Compose)으로 하이라이트를 적용하기에 이상적입니다.
- `PutDocumentsRequest`는 배치 인덱싱을 지원하므로, 여러 문서를 한 번에 등록하면 개별 호출보다 훨씬 효율적입니다.
- `ViewModel.onCleared()`에서 반드시 `repo.close()`를 호출해 세션을 닫아야 합니다.

---

## 주의사항 및 팁

### 1. 스키마 변경 시 `setForceOverride` 위험성

기존 스키마에 새 필드를 추가하거나 타입을 변경할 때, 하위 호환되지 않는 변경사항이 있으면 AppSearch가 예외를 던집니다. `setForceOverride(true)`를 설정하면 충돌 없이 변경할 수 있지만, **기존에 인덱싱된 모든 데이터가 삭제**됩니다.

```kotlin
// ⚠️ 프로덕션에서는 데이터 손실 주의
val request = SetSchemaRequest.Builder()
    .addDocumentClasses(Note::class.java)
    .setForceOverride(true)
    .build()
```

마이그레이션 전략: `setForceOverride` 사용 전 데이터를 백업하거나, 스키마 버전 관리를 통해 증분 업데이트 경로를 설계하세요.

### 2. 검색 쿼리 문법

AppSearch는 간단하지만 강력한 쿼리 언어를 지원합니다.

| 쿼리 예시 | 의미 |
|-----------|------|
| `kotlin flow` | 두 단어 모두 포함 (AND) |
| `kotlin OR compose` | 둘 중 하나 이상 포함 |
| `-deprecated` | 'deprecated'를 포함하지 않음 |
| `title:kotlin` | `title` 속성에서만 검색 |
| `kott*` | 'kott'로 시작하는 모든 단어 (와일드카드) |

### 3. 스토리지 선택 전략

```kotlin
// 런타임에 기기 조건에 따라 스토리지를 선택하는 패턴
suspend fun createSession(context: Context): AppSearchSession {
    return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        // Android 12+: 시스템 서비스 활용, 앱 간 공유 가능
        PlatformStorage.createSearchSession(
            PlatformStorage.SearchContext.Builder(context, "notes_db").build()
        ).await()
    } else {
        // 구형 기기: 앱 내 독립 인덱스
        LocalStorage.createSearchSession(
            LocalStorage.SearchContext.Builder(context, "notes_db").build()
        ).await()
    }
}
```

### 4. 임베딩 벡터 검색 (AppSearch 1.1.0+)

AppSearch 1.1.0부터 `@Document.EmbeddingProperty`를 통해 벡터 임베딩을 저장하고, 코사인 유사도·내적 기반의 **의미론적 검색(Semantic Search)**을 실험적으로 사용할 수 있습니다. 로컬 LLM과 결합하면 "이번 달 회의록 요약" 같은 자연어 질의도 처리할 수 있어, 온디바이스 AI 앱의 검색 레이어로 강력한 도구가 됩니다.

### 5. TakenAction API로 검색 품질 개선

사용자가 검색 결과 중 어떤 항목을 클릭했는지(`ClickAction`)를 AppSearch에 피드백하면, 이후 검색에서 클릭된 문서의 랭킹이 자동으로 향상됩니다.

```kotlin
// 사용자가 결과 클릭 시 피드백
val clickAction = TakenAction.ClickAction.Builder()
    .setQuery(query)
    .setResultRankGlobal(clickedPosition)
    .setDocumentNamespace(note.namespace)
    .setDocumentId(note.id)
    .build()

session.reportSystemUsage(
    ReportSystemUsageRequest.Builder(note.namespace, note.id)
        .build()
).await()
```

---

## 마무리

Android AppSearch는 노트 앱, 이메일 클라이언트, 이커머스 검색 등 사용자가 콘텐츠를 탐색해야 하는 모든 앱에서 Room FTS보다 훨씬 풍부한 검색 경험을 제공합니다. `@Document` 애노테이션 기반의 타입-세이프 스키마, 접두사 검색, 스니펫 하이라이팅, 관련도 랭킹까지 — 기존에 직접 구현해야 했던 복잡한 검색 로직을 Jetpack이 대신 처리해 줍니다.

앱의 검색 기능을 한 단계 끌어올리고 싶다면, AppSearch를 바로 적용해 보세요.

## 참고 자료
- [AppSearch 공식 가이드 — Android Developers](https://developer.android.com/develop/ui/views/search/appsearch)
- [AppSearch Jetpack 릴리스 노트](https://developer.android.com/jetpack/androidx/releases/appsearch)
- [AppSearch GitHub 공식 샘플](https://github.com/android/search-samples/tree/main/AppSearchSample)
