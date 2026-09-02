---
layout: post
title: "Jetpack Compose와 기존 Android View 상호운용 완전 정복: AndroidView·ComposeView·마이그레이션 전략"
date: 2026-09-02
categories: [android, flutter]
tags: [android, jetpack-compose, androidview, composeview, migration, interoperability, kotlin]
---

## 개요

수백만 줄의 XML 레이아웃과 커스텀 View로 이루어진 기존 Android 앱을 하루아침에 Jetpack Compose로 전환하는 것은 현실적으로 불가능합니다. Google이 공식적으로 권장하는 전략은 **점진적 마이그레이션(Incremental Migration)**입니다. 즉, Compose와 기존 View 시스템을 공존시키면서 화면 단위, 컴포넌트 단위로 Compose를 도입해 나가는 방식입니다.

이를 가능하게 해주는 두 가지 핵심 API가 바로 **AndroidView**와 **ComposeView**입니다.

- **AndroidView**: Compose 화면 안에서 기존 View를 렌더링
- **ComposeView**: 기존 XML 레이아웃(Fragment, Activity) 안에서 Compose UI를 렌더링

이 두 API를 깊이 이해하면 마이그레이션 위험을 최소화하면서도 신규 기능은 Compose로 빠르게 개발할 수 있는 이상적인 구조를 갖출 수 있습니다.

---

## 왜 상호운용이 필요한가?

### 현실적인 제약

대형 프로젝트일수록 마이그레이션에는 다음과 같은 현실적 제약이 존재합니다.

1. **테스트 비용**: 수천 개의 테스트가 View 기반으로 작성되어 있는 경우, 전면 재작성은 QA 부담을 폭발적으로 증가시킵니다.
2. **팀 역량**: 팀원 전원이 Compose에 숙련되기까지 시간이 필요합니다.
3. **미지원 SDK 컴포넌트**: `MapView`, `AdView`, `SurfaceView`, `VideoView` 같이 Compose 버전이 없거나 불완전한 컴포넌트들이 존재합니다.
4. **레거시 코드와의 통합**: Navigation, BottomSheetDialogFragment 등 View 기반 라이브러리와의 통합이 필요합니다.

상호운용 API는 이 모든 제약을 수용하면서 Compose를 점진적으로 확대할 수 있는 다리 역할을 합니다.

---

## 핵심 개념 이해

### AndroidView: Compose 안에서 View 사용하기

`AndroidView`는 Compose의 `@Composable` 함수 안에서 전통적인 Android View 계층을 포함시킬 수 있게 해주는 Composable입니다.

```kotlin
AndroidView(
    factory = { context -> /* View 생성 */ },
    update = { view -> /* 상태 변경 시 View 갱신 */ },
    modifier = Modifier
)
```

- **factory**: View 인스턴스를 처음 생성할 때 단 한 번 호출됩니다. `Context`를 파라미터로 받습니다.
- **update**: Compose의 상태(State)가 변경될 때마다 재호출되어 View를 최신 상태로 동기화합니다.
- **onRelease** (LazyList용): `LazyColumn`·`LazyRow` 등에서 View 재사용 풀에서 릴리즈될 때 호출됩니다.

### ComposeView: 기존 View 안에서 Compose 사용하기

`ComposeView`는 XML 레이아웃에 선언하거나 코드로 생성할 수 있는 특수한 `ViewGroup`입니다. 내부에서 `setContent { }` 블록으로 Compose UI를 설정합니다.

#### ViewCompositionStrategy의 중요성

`ComposeView`에서 Composition이 언제 dispose될지를 결정하는 전략입니다. 잘못 설정하면 메모리 누수나 예기치 않은 UI 초기화가 발생합니다.

| 전략 | 사용 상황 |
|------|----------|
| `DisposeOnDetachedFromWindowOrReleasedFromPool` | 기본값. RecyclerView ViewHolder에 적합 |
| `DisposeOnLifecycleDestroyed` | Fragment에서 특정 Lifecycle을 알고 있을 때 |
| `DisposeOnViewTreeLifecycleDestroyed` | Fragment에서 Lifecycle을 직접 지정하기 어려울 때 (권장) |

---

## 실제 구현 예제

### 예제 1: AndroidView로 WebView 래핑하기

Compose에는 공식 WebView Composable이 없습니다. 아래는 기존 `WebView`를 `AndroidView`로 감싸 Compose UI 안에서 완전히 동작하는 웹 브라우저 컴포넌트를 만드는 예제입니다.

```kotlin
import android.webkit.WebView
import android.webkit.WebViewClient
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.viewinterop.AndroidView

@Composable
fun EmbeddedWebView(
    url: String,
    modifier: Modifier = Modifier,
    onPageFinished: (String) -> Unit = {}
) {
    // Compose State와 동기화를 위해 최신 url을 remember로 추적
    val currentUrl by rememberUpdatedState(url)

    AndroidView(
        modifier = modifier,
        factory = { context ->
            WebView(context).apply {
                settings.javaScriptEnabled = true
                settings.domStorageEnabled = true
                webViewClient = object : WebViewClient() {
                    override fun onPageFinished(view: WebView?, loadedUrl: String?) {
                        loadedUrl?.let { onPageFinished(it) }
                    }
                }
                loadUrl(currentUrl)
            }
        },
        update = { webView ->
            // 외부에서 url 상태가 바뀔 때마다 WebView도 새 URL 로드
            if (webView.url != currentUrl) {
                webView.loadUrl(currentUrl)
            }
        },
        onRelease = { webView ->
            // LazyList에서 사용 시, 풀로 반환될 때 리소스 정리
            webView.stopLoading()
        }
    )
}

// 사용 예시
@Composable
fun BrowserScreen() {
    var targetUrl by remember { mutableStateOf("https://developer.android.com") }

    Column {
        Button(onClick = { targetUrl = "https://android-developers.googleblog.com" }) {
            Text("블로그로 이동")
        }
        EmbeddedWebView(
            url = targetUrl,
            modifier = Modifier.fillMaxSize(),
            onPageFinished = { loadedUrl ->
                println("페이지 로드 완료: $loadedUrl")
            }
        )
    }
}
```

**핵심 포인트**: `rememberUpdatedState`를 사용해 `factory` 람다가 캡처한 클로저가 항상 최신 `url` 값을 참조하게 합니다. `update` 콜백에서 현재 URL과 다를 때만 `loadUrl`을 호출해 불필요한 재로딩을 방지합니다.

---

### 예제 2: RecyclerView ViewHolder 안에 ComposeView 삽입하기

기존 `RecyclerView`를 유지하면서 각 아이템의 UI만 Compose로 전환하는 하이브리드 패턴입니다. 이 방법은 Compose로 전환할 여력이 없는 복잡한 리스트 화면에서 유용합니다.

```kotlin
import android.view.ViewGroup
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.ComposeView
import androidx.compose.ui.platform.ViewCompositionStrategy
import androidx.compose.ui.unit.dp
import androidx.recyclerview.widget.RecyclerView

data class Article(val title: String, val summary: String, val isRead: Boolean)

class ArticleViewHolder(
    private val composeView: ComposeView
) : RecyclerView.ViewHolder(composeView) {

    init {
        // RecyclerView에서 View 재사용 시 Composition이 올바르게 dispose/재생성되도록 설정
        composeView.setViewCompositionStrategy(
            ViewCompositionStrategy.DisposeOnRecycled
        )
    }

    fun bind(article: Article, onItemClick: (Article) -> Unit) {
        composeView.setContent {
            // RecyclerView의 MaterialTheme을 상속받아 일관된 스타일 적용
            MaterialTheme {
                ArticleItem(
                    article = article,
                    onClick = { onItemClick(article) }
                )
            }
        }
    }
}

@Composable
private fun ArticleItem(
    article: Article,
    onClick: () -> Unit
) {
    val textColor = if (article.isRead) MaterialTheme.colorScheme.onSurface.copy(alpha = 0.5f)
                    else MaterialTheme.colorScheme.onSurface

    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 16.dp, vertical = 8.dp)
            .clickable(onClick = onClick),
        elevation = CardDefaults.cardElevation(defaultElevation = 2.dp)
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(
                text = article.title,
                style = MaterialTheme.typography.titleMedium,
                color = textColor
            )
            Spacer(modifier = Modifier.height(4.dp))
            Text(
                text = article.summary,
                style = MaterialTheme.typography.bodySmall,
                color = textColor,
                maxLines = 2
            )
        }
    }
}

class ArticleAdapter(
    private val onItemClick: (Article) -> Unit
) : RecyclerView.Adapter<ArticleViewHolder>() {

    private val items = mutableListOf<Article>()

    fun submitList(newItems: List<Article>) {
        items.clear()
        items.addAll(newItems)
        notifyDataSetChanged()
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ArticleViewHolder {
        return ArticleViewHolder(
            ComposeView(parent.context).apply {
                layoutParams = ViewGroup.LayoutParams(
                    ViewGroup.LayoutParams.MATCH_PARENT,
                    ViewGroup.LayoutParams.WRAP_CONTENT
                )
            }
        )
    }

    override fun onBindViewHolder(holder: ArticleViewHolder, position: Int) {
        holder.bind(items[position], onItemClick)
    }

    override fun getItemCount(): Int = items.size
}
```

**핵심 포인트**: `ViewCompositionStrategy.DisposeOnRecycled`를 설정해 RecyclerView의 View 재사용 풀에서 ViewHolder가 릴리즈될 때 Composition을 안전하게 dispose합니다. 이를 생략하면 불필요한 Composition이 메모리에 계속 남아 누수가 발생합니다.

---

## 점진적 마이그레이션 전략

Google이 권장하는 Bottom-Up 마이그레이션 전략은 다음 단계로 이루어집니다.

### 1단계: 리프 컴포넌트부터 시작
화면의 최하단 구성 요소(버튼, 카드, 텍스트 등)를 Composable로 전환합니다. 이들은 의존성이 가장 적어 위험도가 낮습니다.

### 2단계: 화면 일부를 ComposeView로 대체
Fragment 내에서 특정 섹션을 `ComposeView`로 전환합니다. 기존 Fragment 구조는 유지한 채로 내부 View 계층을 Compose로 교체합니다.

### 3단계: 전체 화면을 Compose로 전환
Fragment 자체를 제거하고 Navigation Compose로 이동합니다. Activity에서 `setContent { }` 방식으로 전환합니다.

### 4단계: 단일 Activity 구조로 통합
여러 Activity를 하나의 Activity + Navigation Compose 구조로 통합합니다.

---

## 주의사항과 실전 팁

### 1. AndroidView에서 리소스 누수 방지
`factory` 람다에서 생성한 View가 `BroadcastReceiver`, `Sensor`, `Camera` 등의 리소스를 점유하는 경우, `DisposableEffect`를 함께 사용해 Composition이 종료될 때 리소스를 해제해야 합니다.

```kotlin
@Composable
fun SensorView(modifier: Modifier = Modifier) {
    val context = LocalContext.current
    DisposableEffect(Unit) {
        val sensorManager = context.getSystemService(SensorManager::class.java)
        // ... 센서 등록
        onDispose {
            // Composition 종료 시 센서 해제
            sensorManager.unregisterListener(/* listener */)
        }
    }
    AndroidView(factory = { /* View 생성 */ }, modifier = modifier)
}
```

### 2. 테마 일관성 유지
`ComposeView` 안에서 기존 앱의 Material You 테마를 사용하려면 `MaterialTheme`을 반드시 감싸야 합니다. Compose는 자체적인 테마 시스템을 가지므로, XML Theme 속성이 자동으로 전파되지 않습니다. `AppCompatTheme` (Accompanist 라이브러리) 등을 활용하면 XML Theme을 Compose Theme으로 자동 변환할 수 있습니다.

### 3. ComposeView의 savedInstanceState
하나의 레이아웃에 여러 `ComposeView`가 있다면, 각각에 고유한 `id`를 XML에서 지정해야 합니다. 그래야 `savedInstanceState`가 올바르게 복원됩니다.

```xml
<androidx.compose.ui.platform.ComposeView
    android:id="@+id/compose_header"
    android:layout_width="match_parent"
    android:layout_height="wrap_content" />

<androidx.compose.ui.platform.ComposeView
    android:id="@+id/compose_body"
    android:layout_width="match_parent"
    android:layout_height="0dp"
    android:layout_weight="1" />
```

### 4. AndroidViewBinding으로 XML 레이아웃 전체 포함
개별 View 하나가 아니라 기존 XML 레이아웃 전체를 Compose 안에 삽입하고 싶다면 `AndroidViewBinding`을 사용합니다.

```kotlin
@Composable
fun LegacyFormScreen() {
    AndroidViewBinding(LegacyFormBinding::inflate) {
        submitButton.setOnClickListener {
            // 폼 제출 처리
        }
        titleEditText.hint = "제목을 입력하세요"
    }
}
```

### 5. 성능 고려사항
- `AndroidView`의 `update` 람다는 Recomposition마다 호출됩니다. View에 변경이 없어도 update가 실행되므로, 내부에서 변경 전후 값을 비교해 불필요한 작업을 피하세요.
- `LazyColumn` 안에서 `AndroidView`를 사용할 때는 반드시 `onReset` 콜백을 제공해 View 재사용 풀에서 올바르게 초기화되도록 하세요.

---

## 마무리

Jetpack Compose로의 마이그레이션은 단거리 경주가 아닌 마라톤입니다. `AndroidView`와 `ComposeView`는 기존 코드를 버리지 않고도 Compose의 장점(선언형 UI, 상태 기반 렌더링, 강력한 테스트 지원)을 점진적으로 도입할 수 있는 가장 현실적인 도구입니다.

**핵심 원칙**을 기억하세요:
- Compose에 동등한 API가 있다면 View를 Compose로 전환하세요.
- SDK 컴포넌트나 서드파티 View는 `AndroidView`로 래핑하세요.
- Fragment 전환은 마지막 단계로 남겨두고, `ComposeView`로 내부만 먼저 교체하세요.
- `ViewCompositionStrategy`는 반드시 사용 맥락에 맞게 설정하세요.

---

## 참고 자료
- [Using Views in Compose (AndroidView)](https://developer.android.com/develop/ui/compose/migrate/interoperability-apis/views-in-compose)
- [Using Compose in Views (ComposeView)](https://developer.android.com/develop/ui/compose/migrate/interoperability-apis/compose-in-views)
- [Migrate XML Views to Jetpack Compose](https://developer.android.com/develop/ui/compose/migrate)
