---
layout: post
title: "Android Room + Paging3 심화: 대용량 데이터 효율적 처리 전략"
date: 2026-06-08
categories: [android, flutter]
tags: [android, room, paging3, kotlin, coroutines, remotemediator, pagingsource]
---

# Android Room + Paging3 심화: 대용량 데이터 효율적 처리 전략

앱에서 수천, 수만 건의 데이터를 다루다 보면 한 번에 모두 로딩하는 방식이 얼마나 위험한지 금방 체감하게 됩니다. 메모리는 한계에 달하고, 네트워크 응답은 느려지며, UI 프레임이 끊깁니다. Android Jetpack의 **Paging3 라이브러리**는 이 문제를 체계적으로 해결합니다. 이번 글에서는 Room과 Paging3를 함께 사용하는 심화 전략을 다루고, 실무에서 자주 마주치는 RemoteMediator 패턴까지 코드와 함께 살펴봅니다.

---

## 1. 개념 설명: Paging3의 3계층 구조

Paging3는 데이터 흐름을 세 계층으로 명확하게 분리합니다.

```
[Repository Layer]      [ViewModel Layer]    [UI Layer]
PagingSource       →       Pager           →   LazyPagingItems
RemoteMediator     →       PagingData      →   LazyColumn / RecyclerView
```

### PagingSource
데이터를 페이지 단위로 **로드하는 추상 클래스**입니다. Room은 DAO에서 `PagingSource<Int, Entity>`를 자동으로 반환할 수 있어 별도의 구현 없이도 로컬 캐시 기반 페이지네이션이 가능합니다.

### RemoteMediator
네트워크와 로컬 DB를 함께 사용하는 **오프라인 우선(Offline-First)** 패턴의 핵심입니다. 사용자가 스크롤을 끝까지 내렸을 때 추가 데이터를 서버에서 불러와 Room에 저장하고, UI는 Room에서만 데이터를 읽습니다. 즉, **Room이 단일 진실의 원천(Single Source of Truth)**이 됩니다.

### PagingData / Pager
`Pager`는 `PagingSource`와 `RemoteMediator`를 연결하고, `PagingConfig`로 페이지 크기 등 동작 방식을 설정합니다. 결과물인 `Flow<PagingData<T>>`를 ViewModel이 보유하고 UI가 구독합니다.

---

## 2. 왜 필요한가: 수동 페이지네이션의 문제점

과거에 많은 개발자들이 `RecyclerView.OnScrollListener`에 `lastVisibleItemPosition`을 비교하는 방식으로 페이지네이션을 구현했습니다. 이 접근 방식은 다음 문제를 내포합니다.

- **중복 요청**: 스크롤 이벤트가 여러 번 발생하면 같은 페이지를 여러 번 요청할 수 있습니다.
- **메모리 누수**: 로딩된 데이터를 리스트에 무한정 쌓다가 OOM(Out of Memory) 발생 가능.
- **로딩/에러 상태 관리 분산**: 로딩 인디케이터, 에러 뷰, 재시도 버튼 로직이 여러 곳에 흩어집니다.
- **구성 변경 대응 미흡**: 화면 회전 시 다시 처음부터 로딩하는 경우가 많습니다.

Paging3는 이 모든 문제를 해결합니다. 요청 중복 방지, 페이지별 메모리 관리, `LoadState`를 통한 단일 상태 관리, `cachedIn(viewModelScope)`으로 구성 변경 생존까지 모두 내장되어 있습니다.

---

## 3. 구현 예제 1: Room PagingSource 단독 사용

먼저 의존성을 추가합니다.

```kotlin
// build.gradle.kts
implementation("androidx.paging:paging-runtime:3.4.2")
implementation("androidx.paging:paging-compose:3.4.2")
implementation("androidx.room:room-paging:2.7.1")
```

### Room Entity & DAO

```kotlin
@Entity(tableName = "articles")
data class ArticleEntity(
    @PrimaryKey val id: Int,
    val title: String,
    val body: String,
    val publishedAt: Long
)

@Dao
interface ArticleDao {
    // Room이 PagingSource를 자동 생성
    @Query("SELECT * FROM articles ORDER BY publishedAt DESC")
    fun pagingSource(): PagingSource<Int, ArticleEntity>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(articles: List<ArticleEntity>)

    @Query("DELETE FROM articles")
    suspend fun clearAll()
}
```

### ViewModel

```kotlin
@HiltViewModel
class ArticleViewModel @Inject constructor(
    private val dao: ArticleDao
) : ViewModel() {

    val articles: Flow<PagingData<ArticleEntity>> = Pager(
        config = PagingConfig(
            pageSize = 20,           // 한 페이지에 로드할 항목 수
            prefetchDistance = 5,    // 끝에서 5개 남았을 때 미리 로드
            enablePlaceholders = false
        ),
        pagingSourceFactory = { dao.pagingSource() }
    ).flow
        .cachedIn(viewModelScope)  // 구성 변경 시 데이터 유지
}
```

### Compose UI

```kotlin
@Composable
fun ArticleListScreen(viewModel: ArticleViewModel = hiltViewModel()) {
    val articles = viewModel.articles.collectAsLazyPagingItems()

    LazyColumn {
        items(
            count = articles.itemCount,
            key = articles.itemKey { it.id }
        ) { index ->
            val article = articles[index]
            if (article != null) {
                ArticleItem(article)
            }
        }

        // 페이지 로딩 상태 처리
        when (val state = articles.loadState.append) {
            is LoadState.Loading -> item { LoadingIndicator() }
            is LoadState.Error -> item {
                ErrorItem(
                    message = state.error.localizedMessage ?: "오류 발생",
                    onRetry = { articles.retry() }
                )
            }
            else -> Unit
        }

        // 초기 로딩 상태 처리
        when (val state = articles.loadState.refresh) {
            is LoadState.Loading -> item { FullScreenLoading() }
            is LoadState.Error -> item {
                FullScreenError(
                    message = state.error.localizedMessage ?: "오류 발생",
                    onRetry = { articles.refresh() }
                )
            }
            else -> Unit
        }
    }
}
```

---

## 4. 구현 예제 2: RemoteMediator로 네트워크 + Room 연동

실무에서는 서버에서 데이터를 불러와 Room에 캐시하고, UI는 항상 Room만 바라보는 패턴이 더 일반적입니다. 이때 `RemoteMediator`가 핵심 역할을 합니다.

### RemoteKeys Entity (페이지 키 저장)

```kotlin
@Entity(tableName = "remote_keys")
data class RemoteKeyEntity(
    @PrimaryKey val articleId: Int,
    val prevKey: Int?,
    val nextKey: Int?
)

@Dao
interface RemoteKeyDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(keys: List<RemoteKeyEntity>)

    @Query("SELECT * FROM remote_keys WHERE articleId = :id")
    suspend fun keyByArticleId(id: Int): RemoteKeyEntity?

    @Query("DELETE FROM remote_keys")
    suspend fun clearAll()
}
```

### RemoteMediator 구현

```kotlin
@OptIn(ExperimentalPagingApi::class)
class ArticleRemoteMediator(
    private val api: ArticleApi,
    private val db: AppDatabase
) : RemoteMediator<Int, ArticleEntity>() {

    private val articleDao = db.articleDao()
    private val remoteKeyDao = db.remoteKeyDao()

    override suspend fun initialize(): InitializeAction {
        // 캐시가 충분히 신선하다면 SKIP_INITIAL_REFRESH
        val lastUpdate = articleDao.getLastUpdateTime() ?: return InitializeAction.LAUNCH_INITIAL_REFRESH
        val cacheTimeout = TimeUnit.HOURS.toMillis(1)
        return if (System.currentTimeMillis() - lastUpdate <= cacheTimeout) {
            InitializeAction.SKIP_INITIAL_REFRESH
        } else {
            InitializeAction.LAUNCH_INITIAL_REFRESH
        }
    }

    override suspend fun load(
        loadType: LoadType,
        state: PagingState<Int, ArticleEntity>
    ): MediatorResult {
        return try {
            val page = when (loadType) {
                LoadType.REFRESH -> {
                    // 당겨서 새로고침 또는 최초 로드
                    val remoteKey = state.anchorPosition?.let { anchor ->
                        state.closestItemToPosition(anchor)?.id?.let { id ->
                            remoteKeyDao.keyByArticleId(id)
                        }
                    }
                    remoteKey?.prevKey?.plus(1) ?: 1
                }
                LoadType.PREPEND -> {
                    val remoteKey = state.pages.firstOrNull { it.data.isNotEmpty() }
                        ?.data?.firstOrNull()
                        ?.let { remoteKeyDao.keyByArticleId(it.id) }
                    // PREPEND에서 prevKey가 null이면 첫 페이지에 도달한 것
                    remoteKey?.prevKey ?: return MediatorResult.Success(endOfPaginationReached = true)
                }
                LoadType.APPEND -> {
                    val remoteKey = state.pages.lastOrNull { it.data.isNotEmpty() }
                        ?.data?.lastOrNull()
                        ?.let { remoteKeyDao.keyByArticleId(it.id) }
                    // APPEND에서 nextKey가 null이면 마지막 페이지에 도달
                    remoteKey?.nextKey ?: return MediatorResult.Success(endOfPaginationReached = true)
                }
            }

            val response = api.getArticles(page = page, pageSize = state.config.pageSize)

            db.withTransaction {
                if (loadType == LoadType.REFRESH) {
                    // 새로고침 시 기존 캐시 전부 삭제
                    articleDao.clearAll()
                    remoteKeyDao.clearAll()
                }

                val prevKey = if (page == 1) null else page - 1
                val nextKey = if (response.isLastPage) null else page + 1

                val keys = response.articles.map { article ->
                    RemoteKeyEntity(
                        articleId = article.id,
                        prevKey = prevKey,
                        nextKey = nextKey
                    )
                }
                remoteKeyDao.insertAll(keys)
                articleDao.insertAll(response.articles.map { it.toEntity() })
            }

            MediatorResult.Success(endOfPaginationReached = response.isLastPage)
        } catch (e: IOException) {
            MediatorResult.Error(e)
        } catch (e: HttpException) {
            MediatorResult.Error(e)
        }
    }
}
```

### Repository에서 Pager 구성

```kotlin
@OptIn(ExperimentalPagingApi::class)
class ArticleRepository @Inject constructor(
    private val api: ArticleApi,
    private val db: AppDatabase
) {
    fun getArticleStream(): Flow<PagingData<ArticleEntity>> = Pager(
        config = PagingConfig(
            pageSize = 20,
            prefetchDistance = 5,
            enablePlaceholders = false
        ),
        remoteMediator = ArticleRemoteMediator(api, db),
        pagingSourceFactory = { db.articleDao().pagingSource() }
    ).flow
}
```

---

## 5. 주의사항 및 팁

### ① `@ExperimentalPagingApi` 어노테이션 필수
`RemoteMediator`는 아직 실험적 API입니다. 사용하는 모든 클래스와 함수에 `@OptIn(ExperimentalPagingApi::class)`를 붙이거나, 모듈 수준의 `build.gradle`에 아래 옵션을 추가해야 합니다.

```kotlin
// build.gradle.kts
android {
    kotlinOptions {
        freeCompilerArgs += listOf("-opt-in=androidx.paging.ExperimentalPagingApi")
    }
}
```

### ② `cachedIn(viewModelScope)` 반드시 사용
ViewModel에서 `cachedIn`을 빠뜨리면 화면 회전 시 데이터가 초기화됩니다. 이 연산자는 `PagingData`를 뷰모델 스코프에 캐시해 구성 변경에도 상태를 유지합니다.

### ③ `LoadState`를 적극 활용하세요

| 상태 | 의미 |
|------|------|
| `refresh` | 초기 로드 또는 `invalidate()` 후 전체 재로드 |
| `prepend` | 리스트 상단 이전 페이지 로드 |
| `append` | 리스트 하단 다음 페이지 로드 |

각 상태는 `Loading`, `NotLoading`, `Error` 세 가지 서브타입을 가집니다. 에러 처리는 `articles.retry()`로, 새로고침은 `articles.refresh()`로 처리합니다.

### ④ `withTransaction` 내에서 원자적 처리
RemoteMediator에서 데이터를 저장할 때는 반드시 `db.withTransaction { ... }`을 사용해 데이터 삭제와 삽입이 원자적으로 처리되도록 해야 합니다. 트랜잭션 없이 처리하면 삭제는 됐으나 삽입 전 앱이 종료되는 경우 빈 화면이 표시될 수 있습니다.

### ⑤ `PagingConfig.enablePlaceholders`
`true`로 설정하면 아직 로드되지 않은 항목이 `null`로 표시되어 스크롤바가 실제 데이터 크기를 반영합니다. 단, 총 데이터 개수를 알 수 있는 경우에만 의미 있습니다. 서버가 total count를 제공하지 않는다면 `false`로 설정하세요.

### ⑥ Compose에서 `key` 람다 활용
`LazyPagingItems`를 사용할 때 `items(count = ..., key = articles.itemKey { it.id }) { ... }` 형태로 키를 지정하면 리스트 항목이 추가/삭제될 때 불필요한 recomposition을 줄일 수 있습니다.

---

## 마무리

Room + Paging3 조합은 단순한 오프라인 캐시를 넘어 **오프라인 우선 아키텍처**의 핵심 기반이 됩니다. `PagingSource`만으로도 충분한 경우가 많지만, 서버 데이터를 로컬에 캐시하며 원활한 UX를 제공해야 한다면 `RemoteMediator`는 필수입니다. LoadState를 통한 상태 관리, 트랜잭션 기반의 원자적 저장, `cachedIn`을 통한 구성 변경 생존까지 — 이 세 가지를 제대로 이해하고 나면 대용량 리스트 처리에 자신감이 생길 것입니다.

---

## 참고 자료
- [Paging 3 라이브러리 개요 — Android Developers](https://developer.android.com/topic/libraries/architecture/paging/v3-overview)
- [네트워크와 데이터베이스에서 페이지 로드 — Android Developers](https://developer.android.com/topic/libraries/architecture/paging/v3-network-db)
- [Kotlin Coroutines 공식 문서 — Kotlin](https://kotlinlang.org/docs/coroutines-overview.html)
