---
layout: post
title: "Android Paging3 & RemoteMediator 완전 정복: 네트워크·로컬DB 오프라인 아키텍처 심화"
date: 2026-11-18
categories: [android, flutter]
tags: [android, paging3, remotemediator, room, jetpack, kotlin, coroutines, offline]
---

앱을 개발하다 보면 서버에서 수백~수천 개의 아이템을 가져와 목록에 표시해야 하는 상황이 자주 생깁니다. 한 번에 모든 데이터를 불러오는 방식은 메모리를 과도하게 사용하고 초기 로딩 시간을 늘립니다. Android Jetpack **Paging 3** 라이브러리는 이 문제를 해결해 주는 공식 솔루션이며, 그 중에서도 **RemoteMediator**는 네트워크 소스와 로컬 데이터베이스를 결합해 오프라인 지원까지 가능하게 하는 핵심 컴포넌트입니다. 이 글에서는 Paging3의 전체 아키텍처를 이해하고, RemoteMediator를 활용해 "네트워크 → 로컬DB 캐시 → UI" 흐름을 완성하는 방법을 단계별로 살펴봅니다.

---

## Paging3 아키텍처 개요

Paging3은 세 계층으로 구성됩니다.

| 계층 | 컴포넌트 | 역할 |
|---|---|---|
| **Repository** | `PagingSource`, `RemoteMediator` | 데이터 로드 로직 |
| **ViewModel** | `Pager`, `PagingData` | 스트림 생성·캐싱 |
| **UI** | `PagingDataAdapter` / `LazyPagingItems` | 목록 표시·로딩 상태 처리 |

### PagingSource vs RemoteMediator

- **PagingSource**: 단일 소스(네트워크 or DB)에서 페이지 단위로 데이터를 불러옵니다.
- **RemoteMediator**: DB가 소진되면 호출되어 네트워크에서 추가 데이터를 가져와 DB에 저장합니다. UI는 항상 DB만 바라보며, DB가 단일 진실 공급원(source of truth)이 됩니다.

이 구조를 사용하면 오프라인 상태에서도 캐시된 데이터를 보여줄 수 있고, 스크롤 중 네트워크가 끊겨도 앱이 정상 작동합니다.

---

## 왜 RemoteMediator가 필요한가

순수 네트워크 기반 `PagingSource`는 간단하지만 치명적인 단점이 있습니다.

1. **오프라인 불가**: 네트워크가 없으면 목록 자체를 보여줄 수 없습니다.
2. **중복 요청**: 화면 회전이나 프로세스 재생성 시 처음부터 다시 불러와야 합니다.
3. **스크롤 위치 복원 어려움**: 메모리에만 존재하므로 프로세스가 종료되면 위치를 잃습니다.

RemoteMediator + Room 조합은 이 세 가지를 모두 해결합니다. 데이터가 DB에 영구 저장되므로 프로세스 재시작 후에도 마지막 스크롤 위치를 복원할 수 있고, 오프라인 상태에서도 캐시된 데이터를 즉시 표시합니다.

---

## 실제 구현 예제

GitHub 사용자 목록을 불러오는 예제를 통해 전체 구현을 살펴봅니다.

### 1단계: 엔티티 및 DAO 정의

```kotlin
// Room 엔티티
@Entity(tableName = "github_users")
data class GithubUser(
    @PrimaryKey val id: Long,
    val login: String,
    val avatarUrl: String,
    val htmlUrl: String
)

// 페이지 키를 별도 테이블로 관리
@Entity(tableName = "remote_keys")
data class RemoteKey(
    @PrimaryKey val userId: Long,
    val prevKey: Int?,
    val nextKey: Int?
)

@Dao
interface GithubUserDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(users: List<GithubUser>)

    // Room이 자동으로 PagingSource를 생성해 줌
    @Query("SELECT * FROM github_users ORDER BY id ASC")
    fun pagingSource(): PagingSource<Int, GithubUser>

    @Query("DELETE FROM github_users")
    suspend fun clearAll()
}

@Dao
interface RemoteKeyDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(remoteKeys: List<RemoteKey>)

    @Query("SELECT * FROM remote_keys WHERE userId = :userId")
    suspend fun remoteKeyByUser(userId: Long): RemoteKey?

    @Query("DELETE FROM remote_keys")
    suspend fun clearAll()
}
```

### 2단계: RemoteMediator 구현

RemoteMediator의 `load()` 메서드는 `LoadType`에 따라 세 가지 경우를 처리해야 합니다.

- **REFRESH**: 목록을 처음부터 새로 불러옵니다. DB를 비우고 1페이지부터 시작합니다.
- **PREPEND**: 목록 앞쪽에 추가 데이터가 필요할 때 호출됩니다. 대부분 REFRESH로 최신 데이터를 가져오므로 `endOfPaginationReached = true`를 반환합니다.
- **APPEND**: 스크롤이 끝에 도달해 다음 페이지가 필요할 때 호출됩니다.

```kotlin
@OptIn(ExperimentalPagingApi::class)
class GithubRemoteMediator(
    private val query: String,
    private val db: AppDatabase,
    private val apiService: GithubApiService
) : RemoteMediator<Int, GithubUser>() {

    private val userDao = db.githubUserDao()
    private val remoteKeyDao = db.remoteKeyDao()

    override suspend fun initialize(): InitializeAction {
        // 1시간 이내 캐시는 그대로 사용, 초과 시 전체 새로고침
        val lastUpdate = db.getLastUpdateTime()
        val cacheTimeoutMs = TimeUnit.HOURS.toMillis(1)
        return if (System.currentTimeMillis() - lastUpdate < cacheTimeoutMs) {
            InitializeAction.SKIP_INITIAL_REFRESH
        } else {
            InitializeAction.LAUNCH_INITIAL_REFRESH
        }
    }

    override suspend fun load(
        loadType: LoadType,
        state: PagingState<Int, GithubUser>
    ): MediatorResult {
        return try {
            // 로드할 페이지 번호 결정
            val page: Int = when (loadType) {
                LoadType.REFRESH -> {
                    // 현재 보이는 아이템 기준으로 가장 가까운 원격 키를 찾음
                    val remoteKey = getRemoteKeyClosestToCurrentPosition(state)
                    remoteKey?.nextKey?.minus(1) ?: STARTING_PAGE_INDEX
                }
                LoadType.PREPEND -> {
                    val remoteKey = getRemoteKeyForFirstItem(state)
                    // 첫 페이지 이전에는 데이터가 없으므로 페이지네이션 종료
                    val prevKey = remoteKey?.prevKey
                        ?: return MediatorResult.Success(endOfPaginationReached = remoteKey != null)
                    prevKey
                }
                LoadType.APPEND -> {
                    val remoteKey = getRemoteKeyForLastItem(state)
                    // 다음 페이지 키가 없으면 페이지네이션 종료
                    val nextKey = remoteKey?.nextKey
                        ?: return MediatorResult.Success(endOfPaginationReached = remoteKey != null)
                    nextKey
                }
            }

            // 네트워크 요청
            val response = apiService.searchUsers(
                query = query,
                page = page,
                perPage = state.config.pageSize
            )
            val users = response.items
            val endOfPagination = users.isEmpty()

            // DB에 트랜잭션으로 저장
            db.withTransaction {
                if (loadType == LoadType.REFRESH) {
                    remoteKeyDao.clearAll()
                    userDao.clearAll()
                }

                val prevKey = if (page == STARTING_PAGE_INDEX) null else page - 1
                val nextKey = if (endOfPagination) null else page + 1

                val remoteKeys = users.map { user ->
                    RemoteKey(userId = user.id, prevKey = prevKey, nextKey = nextKey)
                }
                remoteKeyDao.insertAll(remoteKeys)
                userDao.insertAll(users)
            }

            MediatorResult.Success(endOfPaginationReached = endOfPagination)

        } catch (e: IOException) {
            MediatorResult.Error(e)
        } catch (e: HttpException) {
            MediatorResult.Error(e)
        }
    }

    private suspend fun getRemoteKeyForLastItem(state: PagingState<Int, GithubUser>): RemoteKey? {
        return state.pages.lastOrNull { it.data.isNotEmpty() }
            ?.data?.lastOrNull()
            ?.let { user -> remoteKeyDao.remoteKeyByUser(user.id) }
    }

    private suspend fun getRemoteKeyForFirstItem(state: PagingState<Int, GithubUser>): RemoteKey? {
        return state.pages.firstOrNull { it.data.isNotEmpty() }
            ?.data?.firstOrNull()
            ?.let { user -> remoteKeyDao.remoteKeyByUser(user.id) }
    }

    private suspend fun getRemoteKeyClosestToCurrentPosition(state: PagingState<Int, GithubUser>): RemoteKey? {
        return state.anchorPosition?.let { anchorPosition ->
            state.closestItemToPosition(anchorPosition)?.id?.let { userId ->
                remoteKeyDao.remoteKeyByUser(userId)
            }
        }
    }

    companion object {
        const val STARTING_PAGE_INDEX = 1
    }
}
```

### 3단계: Pager 생성 및 ViewModel

```kotlin
class GithubViewModel(
    private val repository: GithubRepository
) : ViewModel() {

    private var currentQuery: String = ""

    val pagingDataFlow: Flow<PagingData<GithubUser>> = repository
        .getSearchResultStream("android")
        .cachedIn(viewModelScope)  // 화면 회전 시 재요청 방지
}

// Repository
class GithubRepository(
    private val db: AppDatabase,
    private val apiService: GithubApiService
) {
    @OptIn(ExperimentalPagingApi::class)
    fun getSearchResultStream(query: String): Flow<PagingData<GithubUser>> {
        return Pager(
            config = PagingConfig(
                pageSize = 30,
                enablePlaceholders = false,
                prefetchDistance = 5  // 끝에서 5개 전에 미리 로드
            ),
            remoteMediator = GithubRemoteMediator(
                query = query,
                db = db,
                apiService = apiService
            ),
            pagingSourceFactory = { db.githubUserDao().pagingSource() }
        ).flow
    }
}
```

### 4단계: Jetpack Compose UI에서 LoadState 처리

Compose에서는 `collectAsLazyPagingItems()`로 `PagingData`를 소비하고, `loadState`를 통해 로딩·에러 상태를 처리합니다.

```kotlin
@Composable
fun GithubUserListScreen(viewModel: GithubViewModel = hiltViewModel()) {
    val pagingItems = viewModel.pagingDataFlow.collectAsLazyPagingItems()

    // source(DB)와 mediator(네트워크)를 분리해 정교한 상태 처리
    val isRefreshing by remember(pagingItems) {
        derivedStateOf {
            // DB가 비어 있고 네트워크에서 가져오는 중일 때만 전체 화면 스피너 표시
            pagingItems.loadState.mediator?.refresh is LoadState.Loading &&
                    pagingItems.itemCount == 0
        }
    }

    Box(modifier = Modifier.fillMaxSize()) {
        when {
            isRefreshing -> {
                CircularProgressIndicator(modifier = Modifier.align(Alignment.Center))
            }
            pagingItems.loadState.refresh is LoadState.Error -> {
                val error = (pagingItems.loadState.refresh as LoadState.Error).error
                Column(
                    modifier = Modifier.align(Alignment.Center),
                    horizontalAlignment = Alignment.CenterHorizontally
                ) {
                    Text(text = "오류: ${error.localizedMessage}")
                    Spacer(modifier = Modifier.height(8.dp))
                    Button(onClick = { pagingItems.retry() }) {
                        Text("다시 시도")
                    }
                }
            }
            else -> {
                LazyColumn {
                    items(
                        count = pagingItems.itemCount,
                        key = pagingItems.itemKey { it.id }
                    ) { index ->
                        val user = pagingItems[index]
                        if (user != null) {
                            GithubUserItem(user = user)
                        }
                    }

                    // 목록 하단 append 로딩 표시
                    item {
                        when (val appendState = pagingItems.loadState.append) {
                            is LoadState.Loading -> {
                                Box(
                                    modifier = Modifier
                                        .fillMaxWidth()
                                        .padding(16.dp),
                                    contentAlignment = Alignment.Center
                                ) {
                                    CircularProgressIndicator(strokeWidth = 2.dp)
                                }
                            }
                            is LoadState.Error -> {
                                Row(
                                    modifier = Modifier
                                        .fillMaxWidth()
                                        .padding(16.dp),
                                    horizontalArrangement = Arrangement.Center
                                ) {
                                    Text("로드 실패")
                                    Spacer(modifier = Modifier.width(8.dp))
                                    TextButton(onClick = { pagingItems.retry() }) {
                                        Text("재시도")
                                    }
                                }
                            }
                            else -> Unit
                        }
                    }
                }
            }
        }
    }
}
```

---

## 주의사항 및 실전 팁

### 1. `cachedIn(viewModelScope)` 필수

ViewModel에서 `cachedIn`을 반드시 사용하세요. 없으면 화면 회전 시마다 새 `PagingData`가 생성되어 불필요한 네트워크 요청이 발생합니다.

```kotlin
// 올바른 방법
val flow = repository.getStream().cachedIn(viewModelScope)
```

### 2. `@ExperimentalPagingApi` 어노테이션

RemoteMediator는 아직 실험적 API입니다. 사용 시 `@OptIn(ExperimentalPagingApi::class)` 어노테이션이 필요합니다. 모듈 수준 build.gradle에 아래를 추가하면 프로젝트 전체에 적용할 수 있습니다.

```kotlin
// build.gradle.kts
android {
    kotlinOptions {
        freeCompilerArgs += listOf("-opt-in=androidx.paging.ExperimentalPagingApi")
    }
}
```

### 3. `enablePlaceholders` 주의

`PagingConfig(enablePlaceholders = true)`일 때 목록 크기를 미리 알아야 합니다. 대부분의 REST API는 총 개수를 명확히 제공하지 않으므로 `false`로 설정하는 것이 안전합니다. `true`일 경우 로딩 중인 아이템 위치에 `null`이 반환되므로 UI에서 null 처리가 필요합니다.

### 4. `initialize()` 메서드로 캐시 유효성 제어

`initialize()`에서 `SKIP_INITIAL_REFRESH`를 반환하면 앱 시작 시 DB에 데이터가 있으면 즉시 보여주고, 백그라운드에서 최신 데이터를 받아 조용히 업데이트합니다. 사용자 입장에서는 빠른 초기 로딩 경험을 얻을 수 있습니다.

### 5. 네트워크 에러와 DB 에러를 분리해서 처리

`CombinedLoadStates`의 `source`는 DB, `mediator`는 네트워크 상태를 나타냅니다.

```kotlin
// DB 소스 에러 → 치명적 오류 (캐시 자체가 망가진 경우)
val sourceError = pagingItems.loadState.source.refresh as? LoadState.Error

// 네트워크 에러 → 비연결 상태, 재시도 가능
val mediatorError = pagingItems.loadState.mediator?.refresh as? LoadState.Error
```

이 두 에러를 구별해 처리하면 사용자에게 더 적절한 메시지를 보여줄 수 있습니다. DB 에러라면 "데이터 로드 실패", 네트워크 에러라면 "오프라인 상태 - 캐시된 데이터를 보여드립니다" 등으로 분기할 수 있습니다.

### 6. PagingSource 무효화 타이밍

Room을 사용하면 `userDao.insertAll()` 호출 시 Room이 자동으로 `PagingSource`를 무효화합니다. Room을 사용하지 않는 경우 `PagingSource.invalidate()`를 직접 호출해야 합니다.

### 7. 테스트 전략

Paging3 테스트는 `androidx.paging:paging-testing` 아티팩트를 사용합니다.

```kotlin
// build.gradle.kts
testImplementation("androidx.paging:paging-testing:3.3.6")
```

```kotlin
// RemoteMediator 단위 테스트 예시
@Test
fun mediatorLoad_refreshSuccess_dbPopulated() = runTest {
    val mockDb = mockAppDatabase()
    val mockApi = mockApiService(users = fakeUsers(30))
    val mediator = GithubRemoteMediator("android", mockDb, mockApi)

    val pagingState = PagingState<Int, GithubUser>(
        pages = listOf(),
        anchorPosition = null,
        config = PagingConfig(pageSize = 30),
        leadingPlaceholderCount = 0
    )

    val result = mediator.load(LoadType.REFRESH, pagingState)
    assertTrue(result is MediatorResult.Success)
    assertFalse((result as MediatorResult.Success).endOfPaginationReached)
    assertEquals(30, mockDb.githubUserDao().count())
}
```

---

## 정리

Paging3 + RemoteMediator 패턴은 초기 구현 비용이 다소 높지만, 오프라인 지원·성능 최적화·스크롤 위치 복원이라는 세 가지 혜택을 한 번에 얻을 수 있습니다. 핵심 원칙은 단순합니다: **UI는 오직 DB만 본다, 네트워크는 DB를 채우는 역할만 한다.** 이 단일 진실 공급원 패턴을 지키면 복잡한 동기화 문제도 명확하게 해결됩니다.

## 참고 자료
- [Page from network and database — Android Developers](https://developer.android.com/topic/libraries/architecture/paging/v3-network-db)
- [Manage and present loading states — Android Developers](https://developer.android.com/topic/libraries/architecture/paging/load-state)
