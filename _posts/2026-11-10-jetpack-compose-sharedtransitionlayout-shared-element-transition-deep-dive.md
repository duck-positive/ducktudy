---
layout: post
title: "Jetpack Compose SharedTransitionLayout 심화: 공유 요소 전환 애니메이션 완벽 가이드"
date: 2026-11-10
categories: [android, flutter]
tags: [android, jetpack-compose, animation, shared-element-transition, sharedtransitionlayout, navigation-compose]
---

## 공유 요소 전환(Shared Element Transition)이란?

앱을 사용할 때 목록 화면에서 상세 화면으로 넘어갈 때 이미지나 제목이 부드럽게 확장되며 이동하는 애니메이션을 본 적이 있을 것이다. 이것이 바로 **공유 요소 전환(Shared Element Transition)**이다. 두 화면 사이에서 동일한 콘텐츠를 시각적으로 연결해주는 이 기법은 사용자에게 맥락적 연속성을 제공하고, 앱의 완성도를 크게 높여준다.

기존 View 시스템에서는 `ActivityOptions.makeSceneTransitionAnimation()`이나 Fragment Transition API를 사용해야 했는데, 설정이 복잡하고 View와 Fragment 간 조율이 까다로웠다. Jetpack Compose 1.7에서 드디어 **`SharedTransitionLayout`** API가 안정화되면서, Compose만의 선언적이고 간결한 방식으로 공유 요소 전환을 구현할 수 있게 됐다.

## 왜 SharedTransitionLayout이 필요한가?

### 기존 방식의 한계

View 기반의 공유 요소 전환은 다음 문제가 있었다.

- `transitionName`을 XML과 코드 양쪽에서 관리해야 함
- 화면 간 전환 시 타이밍 맞추기가 어려움
- 중첩된 Fragment에서 정상 작동하지 않는 경우가 빈번
- Compose와 혼용 시 호환 문제 발생

### Compose 방식의 장점

`SharedTransitionLayout`은 다음을 제공한다.

- **단일 선언**: 같은 `key`를 가진 두 Composable이 자동으로 매칭됨
- **AnimatedContent/AnimatedVisibility 통합**: 화면 전환 타이밍을 프레임워크가 자동 조율
- **Navigation Compose 지원**: NavHost를 감싸기만 하면 모든 목적지에서 공유 요소 사용 가능
- **커스터마이징**: `BoundsTransform`으로 이동 경로(아크, 스프링 등)도 자유롭게 제어

## 핵심 개념 이해

### 세 가지 핵심 API

```
SharedTransitionLayout         ← 최상위 레이아웃, SharedTransitionScope 제공
    └── AnimatedContent        ← AnimatedVisibilityScope 제공
         ├── Modifier.sharedElement()    ← 동일한 콘텐츠 공유
         └── Modifier.sharedBounds()    ← 컨테이너 영역 공유
```

**`Modifier.sharedElement()`**: 두 화면에서 시각적으로 동일한 콘텐츠(이미지, 아이콘 등)를 공유할 때 사용. 전환 중에는 대상(target) 콘텐츠만 렌더링된다.

**`Modifier.sharedBounds()`**: 시각적으로 다르지만 같은 영역을 공유하는 컨테이너(카드 → 상세 화면 배경 등)에 사용. Enter/Exit 애니메이션이 함께 동작한다. Text 컴포저블에는 `sharedBounds()`를 권장한다.

## 기본 구현 예제: AnimatedContent와 함께 사용

가장 간단한 형태로 목록 아이템 클릭 시 상세 화면으로 전환하는 예제다.

```kotlin
@OptIn(ExperimentalSharedTransitionApi::class)
@Composable
fun ProductListDetailScreen() {
    var selectedProduct by remember { mutableStateOf<Product?>(null) }

    SharedTransitionLayout {
        AnimatedContent(
            targetState = selectedProduct,
            label = "product_transition"
        ) { product ->
            if (product == null) {
                ProductList(
                    onProductClick = { selectedProduct = it },
                    sharedTransitionScope = this@SharedTransitionLayout,
                    animatedVisibilityScope = this@AnimatedContent
                )
            } else {
                ProductDetail(
                    product = product,
                    onBack = { selectedProduct = null },
                    sharedTransitionScope = this@SharedTransitionLayout,
                    animatedVisibilityScope = this@AnimatedContent
                )
            }
        }
    }
}

@OptIn(ExperimentalSharedTransitionApi::class)
@Composable
fun ProductList(
    onProductClick: (Product) -> Unit,
    sharedTransitionScope: SharedTransitionScope,
    animatedVisibilityScope: AnimatedVisibilityScope
) {
    LazyColumn(modifier = Modifier.fillMaxSize()) {
        items(products) { product ->
            Row(
                modifier = Modifier
                    .fillMaxWidth()
                    .clickable { onProductClick(product) }
                    .padding(16.dp)
            ) {
                with(sharedTransitionScope) {
                    // 이미지: sharedElement로 동일한 이미지 공유
                    AsyncImage(
                        model = product.imageUrl,
                        contentDescription = product.name,
                        modifier = Modifier
                            .sharedElement(
                                rememberSharedContentState(key = "image-${product.id}"),
                                animatedVisibilityScope = animatedVisibilityScope
                            )
                            .size(80.dp)
                            .clip(RoundedCornerShape(8.dp)),
                        contentScale = ContentScale.Crop
                    )
                    Spacer(Modifier.width(12.dp))
                    // 제목: Text는 sharedBounds 권장
                    Text(
                        text = product.name,
                        style = MaterialTheme.typography.titleMedium,
                        modifier = Modifier
                            .sharedBounds(
                                rememberSharedContentState(key = "title-${product.id}"),
                                animatedVisibilityScope = animatedVisibilityScope,
                                enter = fadeIn(),
                                exit = fadeOut()
                            )
                            .align(Alignment.CenterVertically)
                    )
                }
            }
        }
    }
}

@OptIn(ExperimentalSharedTransitionApi::class)
@Composable
fun ProductDetail(
    product: Product,
    onBack: () -> Unit,
    sharedTransitionScope: SharedTransitionScope,
    animatedVisibilityScope: AnimatedVisibilityScope
) {
    with(sharedTransitionScope) {
        Column(
            modifier = Modifier
                .fillMaxSize()
                .clickable { onBack() }
        ) {
            AsyncImage(
                model = product.imageUrl,
                contentDescription = product.name,
                modifier = Modifier
                    .sharedElement(
                        rememberSharedContentState(key = "image-${product.id}"),
                        animatedVisibilityScope = animatedVisibilityScope
                    )
                    .fillMaxWidth()
                    .height(300.dp),
                contentScale = ContentScale.Crop
            )
            Text(
                text = product.name,
                style = MaterialTheme.typography.headlineMedium,
                modifier = Modifier
                    .sharedBounds(
                        rememberSharedContentState(key = "title-${product.id}"),
                        animatedVisibilityScope = animatedVisibilityScope,
                        enter = fadeIn(),
                        exit = fadeOut(),
                        resizeMode = SharedTransitionScope.ResizeMode.scaleToBounds()
                    )
                    .padding(16.dp)
            )
        }
    }
}
```

여기서 핵심은 `key`값이 목록과 상세 화면에서 동일해야 한다는 것이다. `"image-${product.id}"`처럼 고유 ID를 포함시켜야 목록에 여러 아이템이 있어도 올바른 쌍이 매칭된다.

## Navigation Compose와 통합

실제 프로젝트에서는 Navigation Compose와 함께 사용하는 경우가 대부분이다. `NavHost`를 `SharedTransitionLayout`으로 감싸고, `composable` 블록에서 제공되는 `AnimatedContentScope`를 활용한다.

```kotlin
@OptIn(ExperimentalSharedTransitionApi::class)
@Composable
fun AppNavGraph() {
    SharedTransitionLayout {
        val navController = rememberNavController()

        NavHost(
            navController = navController,
            startDestination = "home",
            modifier = Modifier.fillMaxSize()
        ) {
            composable("home") {
                // this@composable은 AnimatedContentScope이므로 animatedVisibilityScope로 사용 가능
                HomeScreen(
                    sharedTransitionScope = this@SharedTransitionLayout,
                    animatedVisibilityScope = this@composable,
                    onItemClick = { itemId ->
                        navController.navigate("detail/$itemId")
                    }
                )
            }
            composable(
                route = "detail/{itemId}",
                arguments = listOf(navArgument("itemId") { type = NavType.IntType })
            ) { backStackEntry ->
                val itemId = backStackEntry.arguments?.getInt("itemId") ?: return@composable
                DetailScreen(
                    itemId = itemId,
                    sharedTransitionScope = this@SharedTransitionLayout,
                    animatedVisibilityScope = this@composable,
                    onBack = { navController.popBackStack() }
                )
            }
        }
    }
}

@OptIn(ExperimentalSharedTransitionApi::class)
@Composable
fun HomeScreen(
    sharedTransitionScope: SharedTransitionScope,
    animatedVisibilityScope: AnimatedVisibilityScope,
    onItemClick: (Int) -> Unit
) {
    LazyVerticalGrid(
        columns = GridCells.Fixed(2),
        contentPadding = PaddingValues(8.dp),
        modifier = Modifier.fillMaxSize()
    ) {
        itemsIndexed(sampleItems) { index, item ->
            with(sharedTransitionScope) {
                Card(
                    modifier = Modifier
                        .padding(4.dp)
                        .sharedBounds(
                            rememberSharedContentState(key = "card-${item.id}"),
                            animatedVisibilityScope = animatedVisibilityScope,
                            enter = fadeIn(),
                            exit = fadeOut()
                        )
                        .clickable { onItemClick(index) }
                ) {
                    Column {
                        Image(
                            painter = painterResource(item.imageRes),
                            contentDescription = item.title,
                            modifier = Modifier
                                .sharedElement(
                                    rememberSharedContentState(key = "thumbnail-${item.id}"),
                                    animatedVisibilityScope = animatedVisibilityScope
                                )
                                .fillMaxWidth()
                                .aspectRatio(1f),
                            contentScale = ContentScale.Crop
                        )
                        Text(
                            text = item.title,
                            modifier = Modifier
                                .sharedBounds(
                                    rememberSharedContentState(key = "itemTitle-${item.id}"),
                                    animatedVisibilityScope = animatedVisibilityScope,
                                    boundsTransform = BoundsTransform { _, _ ->
                                        // 아크 모션으로 텍스트 이동
                                        keyframes {
                                            durationMillis = 400
                                        }
                                    }
                                )
                                .padding(8.dp),
                            style = MaterialTheme.typography.bodyMedium
                        )
                    }
                }
            }
        }
    }
}
```

`composable` 블록 자체가 `AnimatedContentScope`의 receiver이므로 `this@composable`로 `animatedVisibilityScope`를 얻을 수 있다. 이것이 Navigation Compose와의 통합에서 핵심 포인트다.

## CompositionLocal로 스코프 전달하기

화면이 복잡해져 Composable 계층이 깊어지면, 스코프를 매 함수 파라미터로 전달하는 것이 번거로워진다. 이럴 때는 `CompositionLocal`을 활용한다.

```kotlin
val LocalSharedTransitionScope = compositionLocalOf<SharedTransitionScope?> { null }
val LocalAnimatedVisibilityScope = compositionLocalOf<AnimatedVisibilityScope?> { null }

@OptIn(ExperimentalSharedTransitionApi::class)
@Composable
fun AppRoot() {
    SharedTransitionLayout {
        CompositionLocalProvider(LocalSharedTransitionScope provides this) {
            val navController = rememberNavController()
            NavHost(navController = navController, startDestination = "home") {
                composable("home") {
                    CompositionLocalProvider(LocalAnimatedVisibilityScope provides this) {
                        HomeScreen(onNavigate = { navController.navigate("detail") })
                    }
                }
                composable("detail") {
                    CompositionLocalProvider(LocalAnimatedVisibilityScope provides this) {
                        DetailScreen(onBack = { navController.popBackStack() })
                    }
                }
            }
        }
    }
}

// 깊은 계층의 하위 컴포저블에서 스코프를 꺼내 사용
@OptIn(ExperimentalSharedTransitionApi::class)
@Composable
fun ProductThumbnail(imageUrl: String, productId: String) {
    val sharedTransitionScope = LocalSharedTransitionScope.current
        ?: error("SharedTransitionScope not found")
    val animatedVisibilityScope = LocalAnimatedVisibilityScope.current
        ?: error("AnimatedVisibilityScope not found")

    with(sharedTransitionScope) {
        AsyncImage(
            model = imageUrl,
            contentDescription = null,
            modifier = Modifier
                .sharedElement(
                    rememberSharedContentState(key = "image-$productId"),
                    animatedVisibilityScope = animatedVisibilityScope
                )
                .size(120.dp)
                .clip(RoundedCornerShape(12.dp))
        )
    }
}
```

## 주의사항과 실전 팁

### 1. 모디파이어 순서가 결과를 결정한다

`sharedElement()`와 `sharedBounds()`의 위치는 매우 중요하다.

```kotlin
// ❌ 잘못된 예: padding이 sharedElement 앞에 있으면 padding 포함 크기로 애니메이션
Modifier
    .padding(16.dp)
    .sharedElement(rememberSharedContentState("image"), animatedVisibilityScope)
    .size(100.dp)

// ✅ 올바른 예: sharedElement를 먼저 적용하고 그 이후에 클리핑/시각 효과 적용
Modifier
    .sharedElement(rememberSharedContentState("image"), animatedVisibilityScope)
    .size(100.dp)
    .clip(RoundedCornerShape(8.dp))
```

sharedElement 이전 모디파이어는 **시작/끝 경계 계산에 반영**되고, 이후 모디파이어는 **그 경계 내에서 적용**된다. 두 화면에서 모디파이어 순서가 불일치하면 애니메이션이 뚝뚝 끊기는 현상이 발생한다.

### 2. 고유 키는 데이터 클래스로 관리하라

문자열 키는 오타 위험이 높다. 복잡한 목록에서는 데이터 클래스로 키를 타입 안전하게 관리하자.

```kotlin
enum class SharedElementType { Image, Title, Card, Background }

data class ProductSharedKey(
    val productId: String,
    val type: SharedElementType
)

// 사용 예
Modifier.sharedElement(
    rememberSharedContentState(
        key = ProductSharedKey(product.id, SharedElementType.Image)
    ),
    animatedVisibilityScope = animatedVisibilityScope
)
```

데이터 클래스는 `hashCode()`와 `equals()`를 자동 구현하므로 키 매칭이 정확하게 동작한다.

### 3. ResizeMode를 콘텐츠 특성에 맞게 선택하라

| ResizeMode | 적합한 경우 | 특징 |
|---|---|---|
| `scaleToBounds()` | Text, 아이콘 | 대상 크기로 먼저 레이아웃하고 스케일로 애니메이션. 텍스트 리플로우 없음 |
| `RemeasureToBounds` | 비율이 다른 컨테이너 | 매 프레임 재측정. 유동적 변화에 적합 |

### 4. BottomBar, FAB는 오버레이 위에 유지하라

공유 요소는 전환 중 오버레이 레이어로 올라간다. BottomBar처럼 항상 최상단에 있어야 하는 UI는 `renderInSharedTransitionScopeOverlay()`를 사용해야 가려지지 않는다.

```kotlin
BottomNavigationBar(
    modifier = Modifier
        .renderInSharedTransitionScopeOverlay(zIndexInOverlay = 1f)
        .animateEnterExit(
            enter = fadeIn() + slideInVertically { it },
            exit = fadeOut() + slideOutVertically { it }
        )
)
```

### 5. 현재 알려진 제한사항

- **Views와 호환 불가**: `AndroidView`, `Dialog`, `ModalBottomSheet` 내부의 View와 공유 요소 연결 불가
- **ContentScale 애니메이션 없음**: `Image`의 ContentScale은 전환 중 스냅됨 (sharedBounds + animateEnterExit으로 우회)
- **Shape 클리핑 자동 애니메이션 없음**: 원 → 사각형 등 형태 변화는 별도 처리 필요
- **다이얼로그 간 전환 미지원**

### 6. isMatchFound로 매칭 여부 확인

전환이 의도대로 동작하는지 디버깅하려면 `isMatchFound` 속성을 확인한다.

```kotlin
val sharedContentState = rememberSharedContentState(key = "hero-image")
val isMatched = sharedContentState.isMatchFound  // true면 반대쪽 화면에서 같은 키를 찾음

Image(
    ...,
    modifier = Modifier.sharedElement(sharedContentState, animatedVisibilityScope)
)
```

## 마치며

`SharedTransitionLayout`은 Compose 1.7에서 처음 안정화된 이후 빠르게 성숙해가고 있다. `AnimatedContent`, `AnimatedVisibility`, `NavHost` 모두와 긴밀하게 통합되어 있어 기존 Compose 코드에 점진적으로 도입하기 쉽다.

핵심 원칙 세 가지를 기억하자: **같은 key, 올바른 순서의 모디파이어, 적절한 sharedElement vs sharedBounds 선택**. 이 세 가지만 지켜도 대부분의 공유 요소 전환 시나리오를 깔끔하게 구현할 수 있다.

## 참고 자료
- [Shared element transitions in Compose - Android Developers](https://developer.android.com/develop/ui/compose/animation/shared-elements)
- [Shared element transitions with Navigation - Android Developers](https://developer.android.com/develop/ui/compose/animation/shared-elements/navigation)
- [Customize shared element transitions - Android Developers](https://developer.android.com/develop/ui/compose/animation/shared-elements/customize)
