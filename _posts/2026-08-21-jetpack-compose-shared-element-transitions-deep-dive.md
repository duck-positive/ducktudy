---
layout: post
title: "Jetpack Compose Shared Element Transitions 심화: SharedTransitionLayout으로 매끄러운 화면 전환 구현하기"
date: 2026-08-21
categories: [android, flutter]
tags: [android, jetpack-compose, animation, shared-element, navigation, material3, ui]
---

화면 전환은 사용자 경험에서 가장 눈에 띄는 순간 중 하나입니다. 목록에서 항목을 탭하면 선택한 이미지가 자연스럽게 상세 화면으로 "날아가는" 효과 — 바로 **Shared Element Transition(공유 요소 전환)**입니다. Jetpack Compose 1.7.0에서 안정 API로 승격된 `SharedTransitionLayout`과 `SharedTransitionScope`를 사용하면, 복잡한 전통적인 Transition API 없이도 선언적 방식으로 이 효과를 구현할 수 있습니다.

---

## 1. 개념: Shared Element Transition이란?

Shared Element Transition은 두 화면 사이에 "공통적으로 존재하는" 시각적 요소가 마치 하나의 객체가 이동하는 것처럼 보이게 하는 애니메이션 기법입니다. 실제로는 두 개의 독립적인 컴포저블이지만, 시스템이 시작 위치와 끝 위치를 계산해 부드럽게 보간합니다.

Compose에서는 세 가지 핵심 개념으로 구현됩니다:

- **`SharedTransitionLayout`**: 공유 전환의 최상위 컨테이너. `SharedTransitionScope`를 제공합니다.
- **`Modifier.sharedElement()`**: 두 화면 모두에 동일하게 존재하는 요소(이미지, 아이콘 등)에 적용합니다.
- **`Modifier.sharedBounds()`**: 두 화면에서 *다른* 컴포저블이지만 경계(bounds)를 공유할 때 사용합니다(텍스트, 카드 컨테이너 등).

---

## 2. 왜 필요한가?

Material Design 3의 핵심 모션 원칙 중 하나는 **연속성(Continuity)**입니다. 화면 전환 시 사용자가 "내가 어디 있었고 어디로 가는가"를 직관적으로 파악할 수 있어야 합니다.

공유 요소 전환이 제공하는 구체적인 가치는 다음과 같습니다:

- **컨텍스트 유지**: 목록 항목과 상세 화면이 같은 이미지를 공유함을 시각적으로 전달합니다.
- **인지적 부담 감소**: 갑작스러운 화면 교체 대신 물리적으로 자연스러운 모션을 제공합니다.
- **앱 품질 향상**: Google Play의 앱 품질 가이드라인에서 매끄러운 전환을 권장 기준으로 명시합니다.
- **선언적 구현**: 기존 View 시스템의 `ActivityOptions.makeSceneTransitionAnimation()`과 달리, 코드량이 훨씬 적고 Navigation Component와 자연스럽게 연동됩니다.

---

## 3. 의존성 추가

```kotlin
// build.gradle.kts (모듈 수준)
dependencies {
    implementation("androidx.compose.animation:animation:1.7.0")
    implementation("androidx.navigation:navigation-compose:2.8.0")
    implementation("io.coil-kt:coil-compose:2.7.0") // 이미지 로딩용
}
```

---

## 4. 실제 구현 예제 1 — 상품 목록 → 상세 화면 (AnimatedContent 방식)

`AnimatedContent`와 함께 사용하는 것이 가장 기본적인 패턴입니다.

```kotlin
// 데이터 모델
data class Product(
    val id: Int,
    val name: String,
    val imageUrl: String,
    val description: String,
    val price: String
)

// 루트 컴포저블: SharedTransitionLayout이 AnimatedContent를 감쌉니다
@Composable
fun ProductApp() {
    var selectedProduct by remember { mutableStateOf<Product?>(null) }

    SharedTransitionLayout {
        AnimatedContent(
            targetState = selectedProduct,
            label = "product_transition"
        ) { product ->
            if (product == null) {
                ProductListScreen(
                    sharedTransitionScope = this@SharedTransitionLayout,
                    animatedVisibilityScope = this@AnimatedContent,
                    onProductClick = { selectedProduct = it }
                )
            } else {
                ProductDetailScreen(
                    product = product,
                    sharedTransitionScope = this@SharedTransitionLayout,
                    animatedVisibilityScope = this@AnimatedContent,
                    onBack = { selectedProduct = null }
                )
            }
        }
    }
}

// 목록 화면 — sharedElement로 이미지에 마커를 붙입니다
@Composable
fun ProductListScreen(
    sharedTransitionScope: SharedTransitionScope,
    animatedVisibilityScope: AnimatedVisibilityScope,
    onProductClick: (Product) -> Unit
) {
    with(sharedTransitionScope) {
        LazyColumn {
            items(sampleProducts, key = { it.id }) { product ->
                Row(
                    modifier = Modifier
                        .clickable { onProductClick(product) }
                        .padding(16.dp)
                ) {
                    AsyncImage(
                        model = product.imageUrl,
                        contentDescription = product.name,
                        modifier = Modifier
                            .size(80.dp)
                            .clip(RoundedCornerShape(12.dp))
                            // sharedElement: 두 화면에서 동일한 이미지 요소를 연결
                            .sharedElement(
                                state = rememberSharedContentState(key = "image-${product.id}"),
                                animatedVisibilityScope = animatedVisibilityScope,
                                boundsTransform = { _, _ ->
                                    spring(
                                        dampingRatio = Spring.DampingRatioMediumBouncy,
                                        stiffness = Spring.StiffnessMedium
                                    )
                                }
                            )
                    )
                    Spacer(modifier = Modifier.width(16.dp))
                    Column(modifier = Modifier.weight(1f)) {
                        Text(
                            text = product.name,
                            style = MaterialTheme.typography.titleMedium,
                            // sharedBounds: 텍스트 컨테이너의 경계를 공유
                            modifier = Modifier.sharedBounds(
                                sharedContentState = rememberSharedContentState(key = "name-${product.id}"),
                                animatedVisibilityScope = animatedVisibilityScope
                            )
                        )
                        Text(
                            text = product.price,
                            style = MaterialTheme.typography.bodyMedium,
                            color = MaterialTheme.colorScheme.primary
                        )
                    }
                }
            }
        }
    }
}

// 상세 화면 — 동일한 key로 sharedElement를 연결합니다
@Composable
fun ProductDetailScreen(
    product: Product,
    sharedTransitionScope: SharedTransitionScope,
    animatedVisibilityScope: AnimatedVisibilityScope,
    onBack: () -> Unit
) {
    with(sharedTransitionScope) {
        Column(
            modifier = Modifier
                .fillMaxSize()
                .verticalScroll(rememberScrollState())
        ) {
            Box {
                AsyncImage(
                    model = product.imageUrl,
                    contentDescription = product.name,
                    contentScale = ContentScale.Crop,
                    modifier = Modifier
                        .fillMaxWidth()
                        .aspectRatio(4f / 3f)
                        // 목록과 동일한 key → 시스템이 두 요소를 연결해 전환 생성
                        .sharedElement(
                            state = rememberSharedContentState(key = "image-${product.id}"),
                            animatedVisibilityScope = animatedVisibilityScope
                        )
                )
                IconButton(
                    onClick = onBack,
                    modifier = Modifier
                        .align(Alignment.TopStart)
                        .padding(8.dp)
                        // 전환 중에는 오버레이 레이어에서 렌더링하여 이미지 위에 표시
                        .renderInSharedTransitionScopeOverlay(zIndexInOverlay = 1f)
                ) {
                    Icon(Icons.AutoMirrored.Filled.ArrowBack, "뒤로가기")
                }
            }
            Column(modifier = Modifier.padding(16.dp)) {
                Text(
                    text = product.name,
                    style = MaterialTheme.typography.headlineMedium,
                    modifier = Modifier.sharedBounds(
                        sharedContentState = rememberSharedContentState(key = "name-${product.id}"),
                        animatedVisibilityScope = animatedVisibilityScope
                    )
                )
                Spacer(modifier = Modifier.height(8.dp))
                Text(
                    text = product.price,
                    style = MaterialTheme.typography.titleLarge,
                    color = MaterialTheme.colorScheme.primary
                )
                Spacer(modifier = Modifier.height(16.dp))
                // 공유 전환과 무관한 콘텐츠는 animateEnterExit로 별도 처리
                Text(
                    text = product.description,
                    style = MaterialTheme.typography.bodyMedium,
                    modifier = Modifier.animateEnterExit(
                        enter = fadeIn(tween(300, delayMillis = 200)) + slideInVertically { it / 3 },
                        exit = fadeOut(tween(100))
                    )
                )
            }
        }
    }
}
```

---

## 5. 실제 구현 예제 2 — Navigation Component와 통합

실제 앱에서는 `NavHost`와 함께 사용하는 경우가 훨씬 많습니다. `SharedTransitionLayout`이 `NavHost`를 감싸야 하며, 각 화면은 `composable` 블록의 `AnimatedVisibilityScope`를 수신합니다.

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    // NavHost 전체를 SharedTransitionLayout으로 감쌉니다
    SharedTransitionLayout {
        NavHost(
            navController = navController,
            startDestination = "article_list"
        ) {
            composable("article_list") {
                // this@composable이 AnimatedVisibilityScope를 제공합니다
                ArticleListScreen(
                    sharedTransitionScope = this@SharedTransitionLayout,
                    animatedVisibilityScope = this@composable,
                    onArticleClick = { id -> navController.navigate("article_detail/$id") }
                )
            }
            composable(
                route = "article_detail/{articleId}",
                arguments = listOf(navArgument("articleId") { type = NavType.IntType }),
                // 입장 애니메이션: 공유 전환 외 화면 자체 슬라이드
                enterTransition = { fadeIn(tween(300)) },
                exitTransition = { fadeOut(tween(200)) },
                popEnterTransition = { fadeIn(tween(300)) },
                popExitTransition = { fadeOut(tween(200)) }
            ) { backStackEntry ->
                val articleId = backStackEntry.arguments?.getInt("articleId") ?: return@composable
                ArticleDetailScreen(
                    articleId = articleId,
                    sharedTransitionScope = this@SharedTransitionLayout,
                    animatedVisibilityScope = this@composable,
                    onBack = { navController.popBackStack() }
                )
            }
        }
    }
}

@Composable
fun ArticleListScreen(
    sharedTransitionScope: SharedTransitionScope,
    animatedVisibilityScope: AnimatedVisibilityScope,
    onArticleClick: (Int) -> Unit
) {
    with(sharedTransitionScope) {
        LazyVerticalGrid(
            columns = GridCells.Fixed(2),
            contentPadding = PaddingValues(8.dp)
        ) {
            items(articles, key = { it.id }) { article ->
                Card(
                    onClick = { onArticleClick(article.id) },
                    modifier = Modifier
                        .padding(8.dp)
                        // 카드 전체를 sharedBounds로 감싸 컨테이너 변환 효과
                        .sharedBounds(
                            sharedContentState = rememberSharedContentState(key = "card-${article.id}"),
                            animatedVisibilityScope = animatedVisibilityScope,
                            exit = fadeOut(),
                            enter = fadeIn()
                        )
                ) {
                    Column {
                        AsyncImage(
                            model = article.thumbnailUrl,
                            contentDescription = null,
                            contentScale = ContentScale.Crop,
                            modifier = Modifier
                                .fillMaxWidth()
                                .aspectRatio(16f / 9f)
                                .sharedElement(
                                    state = rememberSharedContentState(key = "thumbnail-${article.id}"),
                                    animatedVisibilityScope = animatedVisibilityScope
                                )
                        )
                        Text(
                            text = article.title,
                            style = MaterialTheme.typography.titleSmall,
                            maxLines = 2,
                            overflow = TextOverflow.Ellipsis,
                            modifier = Modifier
                                .padding(8.dp)
                                .sharedBounds(
                                    sharedContentState = rememberSharedContentState(key = "title-${article.id}"),
                                    animatedVisibilityScope = animatedVisibilityScope
                                )
                        )
                    }
                }
            }
        }
    }
}

@Composable
fun ArticleDetailScreen(
    articleId: Int,
    sharedTransitionScope: SharedTransitionScope,
    animatedVisibilityScope: AnimatedVisibilityScope,
    onBack: () -> Unit
) {
    val article = remember(articleId) { articles.first { it.id == articleId } }

    with(sharedTransitionScope) {
        Scaffold(
            topBar = {
                TopAppBar(
                    title = {},
                    navigationIcon = {
                        IconButton(onClick = onBack) {
                            Icon(Icons.AutoMirrored.Filled.ArrowBack, "뒤로")
                        }
                    }
                )
            }
        ) { padding ->
            Column(
                modifier = Modifier
                    .fillMaxSize()
                    .padding(padding)
                    .verticalScroll(rememberScrollState())
                    // 카드 컨테이너 전환과 동일한 key
                    .sharedBounds(
                        sharedContentState = rememberSharedContentState(key = "card-${article.id}"),
                        animatedVisibilityScope = animatedVisibilityScope,
                        enter = fadeIn(),
                        exit = fadeOut()
                    )
            ) {
                AsyncImage(
                    model = article.thumbnailUrl,
                    contentDescription = null,
                    contentScale = ContentScale.Crop,
                    modifier = Modifier
                        .fillMaxWidth()
                        .height(280.dp)
                        // 목록과 동일한 thumbnail key → 이미지 공유 전환
                        .sharedElement(
                            state = rememberSharedContentState(key = "thumbnail-${article.id}"),
                            animatedVisibilityScope = animatedVisibilityScope
                        )
                )
                Column(modifier = Modifier.padding(16.dp)) {
                    Text(
                        text = article.title,
                        style = MaterialTheme.typography.headlineMedium,
                        modifier = Modifier
                            .sharedBounds(
                                sharedContentState = rememberSharedContentState(key = "title-${article.id}"),
                                animatedVisibilityScope = animatedVisibilityScope
                            )
                            .skipToLookaheadSize() // 텍스트 깜빡임 방지
                    )
                    Spacer(modifier = Modifier.height(12.dp))
                    Text(
                        text = article.body,
                        style = MaterialTheme.typography.bodyMedium,
                        modifier = Modifier.animateEnterExit(
                            enter = fadeIn(tween(400, delayMillis = 250)),
                            exit = fadeOut(tween(100))
                        )
                    )
                }
            }
        }
    }
}
```

---

## 6. sharedElement vs sharedBounds — 언제 어떤 것을 사용하나?

두 수정자의 차이를 이해하는 것이 공유 전환 구현의 핵심입니다.

| 구분 | `sharedElement()` | `sharedBounds()` |
|------|------------------|-----------------|
| **대상** | 두 화면에서 **동일한** 컴포저블 | 두 화면에서 **다른** 컴포저블이지만 경계 공유 |
| **클리핑** | 요소 경계로 클리핑 | 클리핑 없음, 경계만 보간 |
| **주요 예시** | 이미지, 아이콘, 비디오 썸네일 | 텍스트, 카드, 컨테이너 |
| **크기 변환** | 콘텐츠를 함께 스케일 | 경계만 변환, 내부 레이아웃 재계산 |
| **`enter`/`exit`** | 내부 페이드 기본 적용 | 명시적 지정 필요 |

실무 규칙:
- **이미지, 아이콘** → `sharedElement()`
- **텍스트** → `sharedBounds()` + `skipToLookaheadSize()`
- **카드/컨테이너** → `sharedBounds()`로 래핑하고 내부 요소에 별도 `sharedElement()` 추가

---

## 7. boundsTransform으로 애니메이션 커스터마이징

`boundsTransform` 파라미터로 보간 곡선을 세밀하게 제어할 수 있습니다.

```kotlin
Modifier.sharedElement(
    state = rememberSharedContentState(key = "hero-image"),
    animatedVisibilityScope = animatedVisibilityScope,
    boundsTransform = { initialBounds, targetBounds ->
        // 위쪽으로 올라갈 때는 빠르게, 아래로 내려올 때는 천천히
        keyframes {
            durationMillis = 500
            using(FastOutSlowInEasing)
        }
    }
)
```

`spring` 스펙을 쓰면 물리 기반의 탄성감 있는 전환을 구현할 수 있습니다:

```kotlin
boundsTransform = { _, _ ->
    spring(
        dampingRatio = Spring.DampingRatioMediumBouncy,  // 탄성 정도
        stiffness = Spring.StiffnessMediumLow             // 강성(빠르기)
    )
}
```

---

## 8. 주의사항과 팁

### 팁 1: key는 전역적으로 유일해야 합니다
`rememberSharedContentState`의 `key`는 `SharedTransitionLayout` 범위 전체에서 고유해야 합니다. LazyList처럼 여러 항목이 있으면 반드시 ID를 포함하세요.

```kotlin
// 잘못된 예 — 모든 아이템이 같은 key를 공유
rememberSharedContentState(key = "thumbnail")

// 올바른 예 — 아이템 ID로 유일성 보장
rememberSharedContentState(key = "thumbnail-${article.id}")

// 더 안전한 예 — data class를 key로 사용
data class ArticleImageKey(val id: Int, val type: String = "thumbnail")
rememberSharedContentState(key = ArticleImageKey(id = article.id))
```

### 팁 2: skipToLookaheadSize로 텍스트 깜빡임 방지
`sharedBounds`로 감싼 텍스트는 전환 중 크기 재계산으로 인해 깜빡일 수 있습니다.

```kotlin
Text(
    text = article.title,
    modifier = Modifier
        .sharedBounds(
            sharedContentState = rememberSharedContentState(key = "title"),
            animatedVisibilityScope = animatedVisibilityScope
        )
        .skipToLookaheadSize() // 미리 측정된 최종 크기로 레이아웃 고정
)
```

### 팁 3: 이미지 캐시 키 통일로 깜빡임 방지
Coil 등 이미지 라이브러리에서 캐시 키가 다르면 두 화면에서 동일한 이미지를 다시 로드해 깜빡임이 발생합니다.

```kotlin
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data(article.thumbnailUrl)
        .memoryCacheKey("article-thumb-${article.id}") // 일관된 캐시 키
        .diskCacheKey("article-thumb-${article.id}")
        .crossfade(false) // 공유 전환과 crossfade 중복 방지
        .build(),
    contentDescription = null,
    modifier = Modifier.sharedElement(...)
)
```

### 팁 4: 전환 중 UI 요소 오버레이 처리
공유 전환이 진행되는 동안 FAB나 TopAppBar 같은 요소가 애니메이션 위에 겹치는 문제가 있습니다. `renderInSharedTransitionScopeOverlay`로 오버레이 레이어에서 렌더링하세요.

```kotlin
FloatingActionButton(
    onClick = { },
    modifier = Modifier
        .renderInSharedTransitionScopeOverlay(
            renderInOverlay = { isTransitionActive }, // 전환 중에만 오버레이
            zIndexInOverlay = 1f
        )
        .animateEnterExit(
            enter = scaleIn(),
            exit = scaleOut()
        )
) {
    Icon(Icons.Filled.Add, contentDescription = null)
}
```

### 팁 5: 깊은 컴포저블 계층에서 CompositionLocal 활용
여러 계층을 거쳐 `SharedTransitionScope`와 `AnimatedVisibilityScope`를 전달하면 파라미터가 폭발적으로 늘어납니다. `CompositionLocal`로 공유하세요.

```kotlin
val LocalSharedTransitionScope = compositionLocalOf<SharedTransitionScope> {
    error("SharedTransitionScope not provided")
}
val LocalAnimatedVisibilityScope = compositionLocalOf<AnimatedVisibilityScope> {
    error("AnimatedVisibilityScope not provided")
}

// 제공 위치
SharedTransitionLayout {
    CompositionLocalProvider(LocalSharedTransitionScope provides this) {
        AnimatedContent(...) {
            CompositionLocalProvider(LocalAnimatedVisibilityScope provides this) {
                // 하위 어디서든 LocalSharedTransitionScope.current로 접근 가능
                ChildScreen()
            }
        }
    }
}
```

### 팁 6: View-Compose 혼용 환경의 한계
현재 Compose Shared Element Transition은 **순수 Compose 계층**에서만 동작합니다. `AndroidView`로 래핑된 기존 View 요소는 공유 전환에 참여할 수 없습니다. View 기반 화면과의 전환이 필요하다면 기존 Activity Transition API를 병행해야 합니다.

---

## 9. 최소 요구사항 정리

| 항목 | 최솟값 |
|------|--------|
| Compose Animation | `1.7.0` 이상 |
| Kotlin | `1.9.0` 이상 |
| Android API Level | 제한 없음 (Compose 최소 API와 동일) |
| Navigation Compose | `2.8.0` 이상 (NavHost 통합 시) |

---

## 참고 자료

- [Shared Element Transitions in Compose — Android Developers](https://developer.android.com/develop/ui/compose/animation/shared-elements)
- [Compose 애니메이션 빠른 가이드 — Android Developers](https://developer.android.com/develop/ui/compose/animation/quick-guide)
