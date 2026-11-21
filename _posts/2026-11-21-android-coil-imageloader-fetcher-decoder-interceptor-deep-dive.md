---
layout: post
title: "Android Coil 3 심화: 커스텀 Interceptor·Fetcher·Decoder로 이미지 파이프라인 완전 정복"
date: 2026-11-21
categories: [android, flutter]
tags: [android, kotlin, coil, image-loading, coroutines, interceptor, fetcher, decoder]
---

## 개요

현대 Android 앱에서 이미지 로딩은 거의 모든 화면에서 발생하는 작업이다. 사용자는 빠른 로딩과 부드러운 스크롤을 기대하고, 개발자는 메모리 누수 없이 네트워크·디스크·메모리 캐시를 효율적으로 활용해야 한다. **Coil(Coroutine Image Loader)**은 Kotlin 코루틴을 기반으로 설계된 이미지 로딩 라이브러리로, 2026년 현재 버전 3.x에 이르러 Compose Multiplatform까지 지원하는 가장 현대적인 선택지가 되었다.

이 글에서는 `AsyncImage` 한 줄로 끝나는 기본 사용법을 넘어, Coil 3의 **이미지 파이프라인 내부 구조**를 완전히 이해하고 커스텀 `Interceptor`, `Fetcher`, `Decoder`를 작성하는 방법을 깊이 있게 다룬다.

---

## 왜 Coil인가?

### Glide·Picasso와의 비교

| 항목 | Coil 3 | Glide 4 | Picasso 2 |
|------|--------|---------|-----------|
| 언어 기반 | Kotlin-first | Java (Kotlin 지원) | Java |
| 비동기 처리 | 코루틴 네이티브 | Thread Pool + LiveData | AsyncTask 계열 |
| Compose 지원 | 공식 1급 지원 | 별도 확장 필요 | 미지원 |
| Kotlin Multiplatform | Coil 3부터 지원 | Android 전용 | Android 전용 |
| 의존성 크기 | ~125KB (코어) | ~440KB | ~120KB |
| R8 최적화 | 완전 지원 | 완전 지원 | 완전 지원 |

가장 중요한 차이는 **코루틴 네이티브**라는 점이다. Glide는 내부적으로 별도의 스레드 풀과 콜백 기반 아키텍처를 사용하는 반면, Coil은 처음부터 `suspend function`과 `CoroutineScope`로 설계되었다. 이는 취소(cancellation), 백프레셔(backpressure), 생명주기 연동이 훨씬 자연스럽다는 의미다.

### Coil 3의 핵심 설계 원칙

Coil 3는 Coil 2에서 큰 변화가 있었다. Android 전용이던 구조에서 **Kotlin Multiplatform(KMP)** 지원으로 확장되면서, `android.net.Uri` 대신 `coil3.Uri`를 사용하도록 타입이 변경되었다. OkHttp 의존성도 선택적이 되어, Ktor를 사용하거나 직접 HTTP 클라이언트를 연결할 수 있다.

```
의존성 (build.gradle.kts):

// 코어 + Compose 통합
implementation("io.coil-kt.coil3:coil-compose:3.6.3")

// 네트워크 (OkHttp 사용 시)
implementation("io.coil-kt.coil3:coil-network-okhttp:3.6.3")

// SVG 디코딩 (선택)
implementation("io.coil-kt.coil3:coil-svg:3.6.3")

// GIF 지원 (선택)
implementation("io.coil-kt.coil3:coil-gif:3.6.3")
```

---

## 이미지 파이프라인 구조

Coil의 요청 처리 흐름을 이해하는 것이 커스터마이징의 핵심이다.

```
ImageRequest
    │
    ▼
[Interceptors] ← 요청 가로채기/변환/캐시 단락
    │
    ▼
[Mapper] ← 입력 데이터 타입 변환 (예: MyModel → Uri)
    │
    ▼
[Keyer] ← 캐시 키 생성
    │
    ▼
[Fetcher] ← 데이터 소스에서 raw 데이터 가져오기
    │  (URL → 네트워크, File → 디스크, ResourceId → 리소스)
    ▼
[Decoder] ← raw 데이터 → Image 변환
    │  (JPEG/PNG → Bitmap, GIF → 애니메이션, SVG → Bitmap 렌더링)
    ▼
Image (최종 결과)
```

각 단계는 `ComponentRegistry`에 등록되며, Coil은 타입 매칭을 통해 적합한 컴포넌트를 선택한다. 커스텀 컴포넌트는 항상 **빌트인 컴포넌트보다 먼저** 등록해야 우선순위가 적용된다.

---

## 기본 설정: 싱글턴 ImageLoader

Coil에서 `ImageLoader`는 앱 전체에서 하나를 공유하는 것이 권장된다. `Application` 클래스에서 `ImageLoaderFactory`를 구현하면 Coil이 자동으로 사용한다.

```kotlin
// MyApplication.kt
class MyApplication : Application(), ImageLoaderFactory {

    override fun newImageLoader(): ImageLoader {
        return ImageLoader.Builder(this)
            .memoryCache {
                MemoryCache.Builder()
                    .maxSizePercent(this@MyApplication, percent = 0.25)
                    .build()
            }
            .diskCache {
                DiskCache.Builder()
                    .directory(cacheDir.resolve("image_cache"))
                    .maxSizeBytes(50L * 1024 * 1024) // 50MB
                    .build()
            }
            .components {
                // 커스텀 컴포넌트 등록 (뒤에서 다룬다)
                add(SvgDecoder.Factory())
            }
            .build()
    }
}
```

Compose에서는 단순히 `AsyncImage`를 사용하면 된다.

```kotlin
AsyncImage(
    model = "https://example.com/photo.jpg",
    contentDescription = "사진",
    modifier = Modifier.fillMaxWidth(),
    contentScale = ContentScale.Crop,
    placeholder = painterResource(R.drawable.placeholder),
    error = painterResource(R.drawable.error_image),
)
```

---

## 실전 예제 1: 인증 토큰이 필요한 이미지용 커스텀 Interceptor

API 서버가 `Authorization` 헤더를 요구하는 비공개 이미지를 로딩할 때, 모든 요청 호출부에 헤더를 명시하는 것은 번거롭고 오류가 발생하기 쉽다. `Interceptor`를 활용하면 **한 곳에서 일괄 처리**할 수 있다.

```kotlin
/**
 * 인증이 필요한 도메인에 대해 자동으로 Bearer 토큰을 주입하는 Interceptor.
 * tokenProvider는 항상 최신 토큰을 반환해야 한다 (토큰 갱신 포함).
 */
class AuthImageInterceptor(
    private val privateDomain: String,
    private val tokenProvider: suspend () -> String,
) : Interceptor {

    override suspend fun intercept(chain: Interceptor.Chain): ImageResult {
        val request = chain.request
        val url = request.data.toString()

        // 비공개 도메인 요청에만 헤더 추가
        val newRequest = if (url.contains(privateDomain)) {
            val token = tokenProvider()
            request.newBuilder()
                .addHeader("Authorization", "Bearer $token")
                .build()
        } else {
            request
        }

        return chain.proceed(newRequest)
    }
}
```

`ImageLoader` 빌더에 등록한다.

```kotlin
ImageLoader.Builder(context)
    .components {
        add(
            AuthImageInterceptor(
                privateDomain = "cdn.private.example.com",
                tokenProvider = { authRepository.getValidToken() },
            )
        )
        // 기타 컴포넌트
    }
    .build()
```

이제 `AsyncImage`는 아무 추가 코드 없이 인증된 이미지를 가져온다. 토큰 갱신 로직이 `tokenProvider` 람다에 캡슐화되어 있으므로, 토큰 만료 처리도 한 군데서 관리된다.

> **팁**: `Interceptor`는 OkHttp의 인터셉터와 동일한 체인 패턴을 사용한다. `chain.proceed(request)`를 호출하지 않고 직접 `SuccessResult`나 `ErrorResult`를 반환하면 파이프라인을 단락시킬 수 있다. 이를 활용해 커스텀 인메모리 캐시나 A/B 테스트용 이미지 교체가 가능하다.

---

## 실전 예제 2: SVG 지원을 위한 커스텀 Fetcher + Decoder

Coil 3는 `coil-svg` 아티팩트를 제공하지만, 직접 구현해 보면 Fetcher/Decoder 패턴을 깊이 이해할 수 있다. 여기서는 `File`로 저장된 SVG를 읽어 `Bitmap`으로 변환하는 커스텀 `Decoder`를 작성한다. 렌더링에는 [AndroidSVG](https://bigbadaboom.github.io/androidsvg/) 라이브러리를 사용한다.

```kotlin
// implementation("com.caverock:androidsvg-aar:1.4")

class LocalSvgDecoder(
    private val source: ImageSource,
    private val options: Options,
) : Decoder {

    override suspend fun decode(): DecodeResult {
        // ① SVG 파싱
        val svg = source.source().use { bufferedSource ->
            SVG.getFromInputStream(bufferedSource.inputStream())
        }

        // ② 렌더링 크기 결정 (요청 크기 or SVG 고유 크기)
        val svgWidth = svg.documentWidth.takeIf { it > 0 } ?: 512f
        val svgHeight = svg.documentHeight.takeIf { it > 0 } ?: 512f

        val (targetWidth, targetHeight) = when (val size = options.size) {
            is Size -> {
                val w = if (size.width is Dimension.Pixels) (size.width as Dimension.Pixels).px else svgWidth.toInt()
                val h = if (size.height is Dimension.Pixels) (size.height as Dimension.Pixels).px else svgHeight.toInt()
                w to h
            }
            else -> svgWidth.toInt() to svgHeight.toInt()
        }

        // ③ Bitmap에 SVG 렌더링
        val bitmap = Bitmap.createBitmap(targetWidth, targetHeight, Bitmap.Config.ARGB_8888)
        val canvas = Canvas(bitmap)
        svg.renderToCanvas(canvas, RectF(0f, 0f, targetWidth.toFloat(), targetHeight.toFloat()))

        return DecodeResult(
            image = bitmap.asImage(),
            isSampled = false,
        )
    }

    class Factory : Decoder.Factory {
        override fun create(
            result: SourceFetchResult,
            options: Options,
            imageLoader: ImageLoader,
        ): Decoder? {
            // MIME 타입 또는 파일 확장자로 SVG 판별
            if (!result.mimeType.equals("image/svg+xml", ignoreCase = true) &&
                result.source.file()?.name?.endsWith(".svg", ignoreCase = true) != true
            ) {
                return null
            }
            return LocalSvgDecoder(result.source, options)
        }
    }
}
```

`ImageLoader`에 등록하면 `.svg` 파일이나 `image/svg+xml` MIME 타입의 소스는 자동으로 이 Decoder를 거쳐 Bitmap으로 변환된다.

```kotlin
ImageLoader.Builder(context)
    .components {
        add(LocalSvgDecoder.Factory()) // 빌트인보다 먼저 등록
    }
    .build()
```

---

## 메모리 & 디스크 캐시 세밀 조정

### 메모리 캐시 키 커스터마이징

기본적으로 Coil은 URL 문자열을 메모리 캐시 키로 사용한다. 쿼리 파라미터가 동적으로 변하는 URL(서명 URL, 타임스탬프 포함 URL 등)에서는 캐시 적중률이 0%에 가까울 수 있다. 이때 `Keyer`를 커스터마이즈한다.

```kotlin
// 쿼리 파라미터를 제거하고 경로만으로 캐시 키를 생성
class StableUrlKeyer : Keyer<String> {
    override fun key(data: String, options: Options): String? {
        return try {
            val uri = data.toUri()
            "${uri.scheme}://${uri.host}${uri.path}"
        } catch (e: Exception) {
            null // null 반환 시 캐시하지 않음
        }
    }
}
```

### 개별 요청에 대한 캐시 정책 제어

```kotlin
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data("https://example.com/photo.jpg")
        .memoryCachePolicy(CachePolicy.ENABLED)
        .diskCachePolicy(CachePolicy.DISABLED) // 항상 최신 버전 필요
        .networkCacheControl(CacheControl.FORCE_NETWORK)
        .crossfade(true)
        .build(),
    contentDescription = "사진",
)
```

---

## 주의사항 & 실전 팁

### 1. Coil 2 → Coil 3 마이그레이션 시 타입 주의

Coil 3는 `android.net.Uri` 대신 `coil3.Uri`를 사용한다. 기존에 `Uri` 타입으로 직접 데이터를 전달하던 코드는 `toString()`으로 String 변환 후 전달하거나, Coil의 Uri로 변환해야 한다.

```kotlin
// Coil 2 (Android Uri 직접 사용)
imageView.load(androidUri)

// Coil 3 (String으로 변환하거나 coil3.Uri 사용)
AsyncImage(model = androidUri.toString(), contentDescription = null)
```

### 2. ImageLoader는 싱글턴으로 유지하라

`ImageLoader`는 내부적으로 OkHttpClient, 메모리 캐시, 디스크 캐시 등의 무거운 자원을 보유한다. 매 `Composable` 호출이나 화면 이동 시 새로 생성하면 심각한 메모리 낭비가 발생한다. `Application` 클래스에서 `ImageLoaderFactory`를 구현하거나 DI 컨테이너(Hilt)에 `@Singleton`으로 등록하라.

### 3. Placeholder와 Error 이미지에 `tint` 주의

벡터 드로어블을 Placeholder로 사용할 때 Coil은 동적 색상(Dynamic Color)을 자동으로 적용하지 않는다. `ColorFilter`나 `tint` 모디파이어를 명시적으로 설정해야 한다.

### 4. 큰 이미지의 메모리 OOM 방지

`Precision.INEXACT` 옵션은 Coil이 요청된 크기보다 작은 캐시된 이미지를 반환하도록 허용한다. 고해상도 이미지가 많은 갤러리 화면에서 메모리 사용량을 크게 줄일 수 있다.

```kotlin
ImageRequest.Builder(context)
    .data(url)
    .size(200, 200)
    .precision(Precision.INEXACT)
    .build()
```

### 5. 테스트에서 FakeImageLoader 사용

UI 테스트에서 네트워크 이미지 로딩은 불안정한 요소다. Coil은 `FakeImageLoader`를 제공해 테스트 환경에서 고정된 이미지나 에러를 반환하도록 주입할 수 있다.

```kotlin
// Compose 테스트에서
val fakeImageLoader = FakeImageLoader.Builder()
    .defaultResult(
        SuccessResult(
            image = BitmapImage(BitmapFactory.decodeResource(context.resources, R.drawable.test_image)),
            request = TODO(),
            dataSource = DataSource.MEMORY,
        )
    )
    .build()

composeTestRule.setContent {
    CompositionLocalProvider(LocalImageLoader provides fakeImageLoader) {
        MyScreen()
    }
}
```

---

## 마무리

Coil 3는 단순한 이미지 로딩 도구가 아니다. OkHttp의 인터셉터 패턴을 계승한 `Interceptor`, 데이터 소스 추상화를 위한 `Fetcher`, 포맷 디코딩을 위한 `Decoder`가 조합되어 **확장 가능한 이미지 파이프라인**을 구성한다. 커스텀 컴포넌트를 작성할 때의 핵심은 `ComponentRegistry`의 등록 순서와 Factory의 타입 판별 로직이다. 이 두 가지를 이해하면 인증, 커스텀 포맷, 비즈니스 로직 삽입 등 어떤 요구사항도 파이프라인으로 깔끔하게 처리할 수 있다.

## 참고 자료

- [Coil 공식 GitHub 저장소 (coil-kt/coil)](https://github.com/coil-kt/coil)
- [Android 개발자 가이드: Compose에서 이미지 로드하기](https://developer.android.com/develop/ui/compose/quick-guides/content/load-images)
- [Coil 이미지 파이프라인 문서 (image_pipeline.md)](https://raw.githubusercontent.com/coil-kt/coil/main/docs/image_pipeline.md)
