---
layout: post
title: "Android 메모리 누수 탐지 및 최적화: LeakCanary부터 실전 패턴까지"
date: 2026-06-14
categories: [android, flutter]
tags: [android, memory-leak, leakcanary, kotlin, viewmodel, fragment, performance]
---

## 메모리 누수란 무엇인가?

메모리 누수(Memory Leak)란 프로그램이 더 이상 사용하지 않는 객체에 대한 참조를 여전히 유지함으로써 가비지 컬렉터(GC)가 해당 메모리를 회수하지 못하는 현상입니다. Android에서는 특히 Activity, Fragment, View와 같은 수명 주기(Lifecycle)를 가진 컴포넌트들이 누수의 주요 원인이 됩니다.

Java와 Kotlin은 JVM의 가비지 컬렉터 덕분에 메모리를 자동으로 관리하지만, 가비지 컬렉터는 **GC 루트(GC Root)에서 도달 가능한 객체는 절대 수거하지 않습니다**. GC 루트에는 전역 변수(static), 실행 중인 스레드, JNI 참조 등이 포함됩니다. 따라서 static 변수나 오래 살아있는 백그라운드 스레드가 Activity 인스턴스에 대한 참조를 유지하고 있다면, 해당 Activity는 `finish()`가 호출된 뒤에도 메모리에서 해제되지 않습니다.

---

## 왜 중요한가?

Android는 메모리 제약이 심한 환경입니다. 기기마다 다르지만 일반적으로 앱 프로세스 하나에 허용되는 힙(Heap) 크기는 256MB~512MB 정도입니다. 메모리 누수가 쌓이면 다음과 같은 문제가 발생합니다.

- **OutOfMemoryError(OOM)**: 힙이 꽉 차면 새 객체를 할당할 수 없어 앱이 크래시됩니다.
- **GC 압박(GC Pressure)**: 메모리 부족 시 GC가 더 자주 실행되어 UI 스레드가 잠시 멈추는 **jank** 현상이 발생합니다.
- **ANR(App Not Responding)**: GC로 인한 정지가 길어지면 ANR로 이어질 수 있습니다.
- **배터리 소모 증가**: 불필요한 GC 실행이 CPU 사용량을 높여 배터리를 소모시킵니다.

Square의 엔지니어들이 LeakCanary를 자사 POS 앱에 최초 도입했을 때, 여러 누수를 수정한 결과 OOM 크래시율이 **94%** 감소했습니다.

---

## 주요 누수 패턴

### 1. static 참조에 Context 저장

가장 흔하면서도 치명적인 패턴입니다. `Activity`나 `Fragment`의 Context를 전역 싱글톤에 저장하면 앱이 종료될 때까지 해당 인스턴스가 메모리에서 해제되지 않습니다.

```kotlin
// 잘못된 예시: static 변수에 Activity Context 저장
object SomeManager {
    var context: Context? = null  // Activity를 저장하면 누수 발생!
}

// 올바른 예시: ApplicationContext 사용
object SomeManager {
    lateinit var appContext: Context

    fun init(context: Context) {
        appContext = context.applicationContext  // Application 범위 Context
    }
}
```

`applicationContext`는 앱이 살아있는 동안 유효하며 Activity와 독립적입니다.

### 2. Inner Class와 Anonymous Class

Kotlin/Java의 내부 클래스(non-static inner class)는 암묵적으로 외부 클래스의 참조를 유지합니다. `Runnable`, `Callable`, `Thread` 등의 익명 구현이 Activity 내부에 정의될 경우 동일한 문제가 발생합니다.

### 3. Fragment의 View Binding 누수

Fragment는 View보다 오래 살아남을 수 있습니다(Back Stack에 있을 때). `onDestroyView()`에서 binding을 null로 처리하지 않으면 Fragment가 View 트리 전체를 붙잡아 둡니다.

### 4. Listener/Callback 미해제

`BroadcastReceiver`, `SensorManager`, `LocationManager` 등 시스템 서비스에 등록한 리스너를 적절한 생명주기 시점에 해제하지 않으면 Activity/Fragment가 GC 대상이 되지 않습니다.

---

## LeakCanary 설정

LeakCanary는 Android 개발에서 가장 널리 쓰이는 메모리 누수 탐지 라이브러리입니다. Debug 빌드에서만 동작하고, 별도 초기화 코드 없이 의존성 추가만으로 자동 설정됩니다.

```kotlin
// build.gradle.kts (app 모듈)
dependencies {
    debugImplementation("com.squareup.leakcanary:leakcanary-android:2.14")

    // CI 환경의 Instrumentation 테스트와 연동 시 추가
    androidTestImplementation("com.squareup.leakcanary:leakcanary-android-instrumentation:2.14")
}
```

앱을 Debug 빌드로 실행하면 LeakCanary가 자동으로 다음 작업을 수행합니다.

1. Activity, Fragment, ViewModel, RootView, Service의 생명주기를 감시합니다.
2. 해당 객체가 destroy된 후 5초가 지나도 GC에 의해 수거되지 않으면 힙 덤프를 캡처합니다.
3. 백그라운드 스레드에서 누수 경로(Leak Trace)를 분석하여 알림으로 보여줍니다.

---

## 실전 구현 예제

### 예제 1 — Fragment View Binding 누수 방지

```kotlin
class HomeFragment : Fragment(R.layout.fragment_home) {

    // nullable로 선언하고 onDestroyView에서 반드시 null 처리
    private var _binding: FragmentHomeBinding? = null
    private val binding get() = _binding!!

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        _binding = FragmentHomeBinding.bind(view)

        binding.button.setOnClickListener {
            viewModel.doSomething()
        }

        // Flow 수집 시 viewLifecycleOwner 사용 (Fragment 자체가 아님)
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    binding.textView.text = state.message
                }
            }
        }
    }

    override fun onDestroyView() {
        super.onDestroyView()
        // View가 파괴될 때 binding 참조 해제 — 핵심!
        _binding = null
    }
}
```

`viewLifecycleOwner.lifecycleScope`를 사용하면 View가 파괴될 때 코루틴도 자동으로 취소됩니다. Fragment의 `lifecycleScope`가 아닌 `viewLifecycleOwner`의 것을 써야 한다는 점에 주의하세요. Fragment는 Back Stack에서 Pop되기 전까지 살아있기 때문에 `lifecycleScope`는 View가 없는 상태에서도 계속 실행됩니다.

### 예제 2 — Handler/Runnable 누수 방지와 Coroutine 대체

```kotlin
// 방법 1: Handler 사용 시 콜백 명시적 제거
class MainActivity : AppCompatActivity() {

    private val handler = Handler(Looper.getMainLooper())

    private val updateRunnable = Runnable {
        binding.statusText.text = "업데이트 완료"
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        handler.postDelayed(updateRunnable, 5_000L)
    }

    override fun onDestroy() {
        super.onDestroy()
        // Activity 파괴 시 예약된 콜백을 반드시 제거
        handler.removeCallbacks(updateRunnable)
    }
}

// 방법 2 (권장): lifecycleScope + delay 로 대체
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Activity가 DESTROYED 상태가 되면 자동 취소 — removeCallbacks 불필요
        lifecycleScope.launch {
            delay(5_000L)
            binding.statusText.text = "업데이트 완료"
        }
    }
}
```

`lifecycleScope`는 `Lifecycle.State.DESTROYED` 상태가 되면 자동으로 코루틴을 취소합니다. 별도의 해제 코드가 필요 없어 버그 발생 가능성이 낮습니다.

### 예제 3 — ViewModel에서의 안전한 Context 사용

ViewModel은 Activity/Fragment보다 오래 살아남기 때문에 UI Context를 직접 참조하면 누수가 발생합니다.

```kotlin
// 잘못된 예시: Activity Context를 ViewModel에 직접 주입
class BadViewModel(
    private val context: Context  // Activity라면 누수!
) : ViewModel()

// 올바른 예시 1: AndroidViewModel로 Application Context 사용
class GoodViewModel(application: Application) : AndroidViewModel(application) {

    fun getLabel(): String {
        // getApplication()은 Application 스코프이므로 안전
        return getApplication<Application>().getString(R.string.app_name)
    }
}

// 올바른 예시 2: Context 의존성을 Repository 계층으로 분리
class ResourceRepository(private val appContext: Context) {
    fun getString(@StringRes resId: Int): String = appContext.getString(resId)
}

class BetterViewModel(
    private val resourceRepo: ResourceRepository
) : ViewModel() {
    val label = resourceRepo.getString(R.string.app_name)
}
```

Hilt를 사용한다면 `@ApplicationContext`를 주입받아 `Context` 의존성을 명확하게 관리할 수 있습니다.

```kotlin
@HiltViewModel
class HiltViewModel @Inject constructor(
    @ApplicationContext private val context: Context,
    private val repository: MyRepository
) : ViewModel() {
    // context는 ApplicationContext이므로 안전
}
```

### 예제 4 — StrictMode로 런타임 감지

```kotlin
class MyApplication : Application() {

    override fun onCreate() {
        super.onCreate()

        if (BuildConfig.DEBUG) {
            StrictMode.setVmPolicy(
                StrictMode.VmPolicy.Builder()
                    .detectLeakedSqlLiteObjects()
                    .detectLeakedClosableObjects()
                    .detectActivityLeaks()
                    .detectLeakedRegistrationObjects()  // 미해제 리스너 감지
                    .penaltyLog()  // Logcat에 스택 트레이스 출력
                    // .penaltyDeath()  // 누수 발생 시 강제 크래시 (CI에서 유용)
                    .build()
            )
        }
    }
}
```

---

## Android Studio Memory Profiler 활용

LeakCanary 외에도 Android Studio의 **Memory Profiler**를 활용하면 런타임 메모리 상태를 시각적으로 분석할 수 있습니다.

1. **Android Studio → View → Tool Windows → Profiler** 실행
2. 앱을 실행하고 프로파일러 연결
3. **Memory 트랙** 선택 → **Capture heap dump** 클릭
4. 힙 덤프에서 `Activity`, `Fragment` 인스턴스 수 확인 (정상이라면 현재 표시 중인 것만 1개여야 함)
5. **Allocation tracking**으로 메모리가 어느 코드에서 할당되는지 추적

특히 화면 회전(Configuration Change) 후 힙 덤프를 찍어 `Activity` 인스턴스가 2개 이상이라면 누수가 확실합니다.

---

## 주의사항 및 핵심 팁

**1. LeakCanary는 반드시 `debugImplementation`으로**  
`implementation`으로 잘못 추가하면 Release 빌드에도 포함되어 앱 크기가 증가하고 힙 덤프가 외부에 노출될 수 있습니다.

**2. `viewLifecycleOwner` vs `lifecycleOwner` 혼동 주의**  
Fragment 안에서 Flow를 수집하거나 LiveData를 관찰할 때는 반드시 `viewLifecycleOwner`를 사용하세요. Fragment 자체의 수명 주기는 View보다 길어 Back Stack에 있을 때도 살아있습니다.

**3. Bitmap 관리**  
대용량 비트맵은 `BitmapFactory.Options.inSampleSize`로 다운샘플링하고, Glide나 Coil 같은 이미지 로더를 사용하면 자동 캐싱 및 메모리 관리를 제공합니다.

**4. `WeakReference` 남용 금지**  
`WeakReference`는 해결책이 아니라 임시방편입니다. GC가 언제든 참조를 끊을 수 있어 null 체크 없이 사용하면 NPE가 발생합니다. 수명 주기(Lifecycle)로 근본 원인을 해결하는 것이 우선입니다.

**5. CI 파이프라인에 통합**  
LeakCanary의 `leakcanary-android-instrumentation`을 Espresso/UI Automator 테스트에 추가하면 PR 머지 전에 누수를 자동 감지할 수 있습니다.

**6. 코루틴 스코프 선택 기준**

| 스코프 | 수명 | 사용 위치 |
|--------|------|-------|
| `lifecycleScope` | Activity/Fragment 생명주기 | Activity, Fragment |
| `viewLifecycleOwner.lifecycleScope` | View 생명주기 | Fragment의 View 관련 작업 |
| `viewModelScope` | ViewModel 생명주기 | ViewModel |
| `GlobalScope` | 앱 프로세스 | 사용 지양 |

---

## 참고 자료
- [Manage your app's memory - Android Developers](https://developer.android.com/topic/performance/memory)
- [Capture a heap dump - Android Studio](https://developer.android.com/studio/profile/memory-profiler)
- [ViewModel overview - Android Developers](https://developer.android.com/topic/libraries/architecture/viewmodel)
- [LeakCanary - GitHub (square/leakcanary)](https://github.com/square/leakcanary)
