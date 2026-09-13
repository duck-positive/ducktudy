---
layout: post
title: "Android SensorManager 심화: 가속도계·자이로스코프·만보기를 Kotlin Flow로 완전 정복"
date: 2026-09-13
categories: [android, kotlin]
tags: [android, sensormanager, accelerometer, gyroscope, stepcounter, kotlinflow, coroutines, sensor]
---

Android 기기에는 눈에 보이지 않는 수십 개의 센서가 탑재되어 있습니다. 가속도계(Accelerometer), 자이로스코프(Gyroscope), 만보기(Step Counter)는 그 중에서도 가장 널리 쓰이는 센서입니다. 그런데 전통적인 `SensorEventListener` 콜백 방식은 액티비티 생명주기와 얽히면서 등록/해제 관리가 복잡해지고, 메모리 누수까지 유발하기 쉽습니다.

이 글에서는 `SensorManager`의 내부 동작 원리부터 시작해, `callbackFlow`를 활용한 Kotlin Flow 래퍼 구현, 실전 흔들기 감지(Shake Detection)와 스텝 카운터 예제, 그리고 배터리·성능 최적화 팁까지 단계별로 살펴봅니다.

---

## 1. Android 센서 프레임워크 개요

Android 센서는 크게 **하드웨어 센서**와 **소프트웨어 센서(가상 센서)**로 나뉩니다.

| 범주 | 대표 센서 | 구현 방식 |
|---|---|---|
| 모션(Motion) | Accelerometer, Gyroscope, Rotation Vector | 하드웨어/소프트웨어 혼합 |
| 위치(Position) | Magnetic Field, Proximity | 하드웨어 |
| 환경(Environment) | Barometer, Light, Temperature | 하드웨어 |
| 활동(Activity) | Step Counter, Step Detector | 소프트웨어(Pedometer 칩) |

**핵심 클래스 4가지:**

- `SensorManager` – 시스템 서비스. 센서 목록 조회 및 리스너 등록/해제 담당.
- `Sensor` – 특정 센서의 메타데이터(타입, 이름, 최대 범위, 전력 소모량 등).
- `SensorEvent` – 센서 데이터 이벤트. `values[]` 배열에 측정값이 담김.
- `SensorEventListener` – `onSensorChanged()`와 `onAccuracyChanged()` 두 콜백을 제공하는 인터페이스.

### 1-1. 좌표계

Android 센서는 기기를 세로로 들었을 때 기준으로 오른손 좌표계를 사용합니다.

- **X 축**: 화면의 오른쪽 방향 (가로)
- **Y 축**: 화면의 위쪽 방향 (세로)
- **Z 축**: 화면에서 사용자 쪽으로 나오는 방향 (깊이)

### 1-2. Android 12+ 센서 속도 제한

Android 12(API 31)부터 일반 앱이 `SensorEventListener`를 등록할 때 사용할 수 있는 최대 갱신 주기가 **200Hz(5ms)**로 제한되었습니다. 더 빠른 속도(최대 400~800Hz)가 필요하다면 `HIGH_SAMPLING_RATE_SENSORS` 권한을 `AndroidManifest.xml`에 선언해야 합니다.

```xml
<uses-permission android:name="android.permission.HIGH_SAMPLING_RATE_SENSORS" />
```

---

## 2. 왜 Kotlin Flow로 래핑해야 하는가?

전통적인 `SensorEventListener` 패턴의 문제점을 살펴보겠습니다.

```kotlin
// 전통적인 방식 — 생명주기 관리가 분산되어 있고 실수하기 쉽다
class SensorActivity : AppCompatActivity(), SensorEventListener {
    private lateinit var sensorManager: SensorManager
    private var accelerometer: Sensor? = null

    override fun onResume() {
        super.onResume()
        sensorManager = getSystemService(SENSOR_SERVICE) as SensorManager
        accelerometer = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
        sensorManager.registerListener(this, accelerometer, SensorManager.SENSOR_DELAY_NORMAL)
    }

    override fun onPause() {
        super.onPause()
        sensorManager.unregisterListener(this) // 잊어버리면 메모리 누수!
    }

    override fun onSensorChanged(event: SensorEvent) { /* ... */ }
    override fun onAccuracyChanged(sensor: Sensor, accuracy: Int) { /* ... */ }
}
```

이 방식의 단점:
1. **생명주기 관리 분산** – 등록은 `onResume`, 해제는 `onPause`에 있어서 코드를 따라가기 어렵습니다.
2. **테스트 불가** – 콜백 기반이라 단위 테스트 작성이 번거롭습니다.
3. **변환·필터링 불편** – 이동 평균, 스로틀링 등을 직접 구현해야 합니다.
4. **결합도 높음** – ViewModel에서 직접 `SensorEventListener`를 구현하면 Context 의존이 생깁니다.

`callbackFlow`를 사용하면 이 모든 문제를 해결할 수 있습니다. Flow는 구독자가 없으면 자동으로 리스너를 해제하고, 다양한 연산자로 데이터를 손쉽게 변환할 수 있습니다.

---

## 3. 실제 구현 예제

### 예제 1: Kotlin Flow 기반 범용 센서 래퍼

아래는 어떤 센서든 Flow로 래핑할 수 있는 확장 함수입니다. `ViewModel`에서 `Repository` 역할로 활용하기에 적합합니다.

```kotlin
import android.content.Context
import android.hardware.Sensor
import android.hardware.SensorEvent
import android.hardware.SensorEventListener
import android.hardware.SensorManager
import kotlinx.coroutines.channels.awaitClose
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.callbackFlow
import kotlinx.coroutines.flow.conflate

data class SensorData(
    val x: Float,
    val y: Float,
    val z: Float,
    val timestamp: Long
)

fun Context.sensorFlow(
    sensorType: Int,
    samplingPeriodUs: Int = SensorManager.SENSOR_DELAY_NORMAL
): Flow<SensorData> = callbackFlow {
    val sensorManager = getSystemService(Context.SENSOR_SERVICE) as SensorManager
    val sensor = sensorManager.getDefaultSensor(sensorType)
        ?: run {
            // 센서가 없는 기기에서는 Flow를 즉시 종료
            close()
            return@callbackFlow
        }

    val listener = object : SensorEventListener {
        override fun onSensorChanged(event: SensorEvent) {
            // values 배열 크기가 3 미만인 경우 방어 처리
            if (event.values.size < 3) return
            val result = trySend(
                SensorData(
                    x = event.values[0],
                    y = event.values[1],
                    z = event.values[2],
                    timestamp = event.timestamp
                )
            )
            // 채널이 가득 차면 result.isFailure — conflate()로 최신값만 유지
            _ = result
        }

        override fun onAccuracyChanged(sensor: Sensor, accuracy: Int) = Unit
    }

    sensorManager.registerListener(listener, sensor, samplingPeriodUs)

    // 구독 취소 시(= Flow 컬렉션 중단 시) 자동으로 리스너 해제
    awaitClose {
        sensorManager.unregisterListener(listener)
    }
}.conflate() // 처리가 느릴 때 중간 이벤트를 건너뜀으로써 최신 데이터를 우선시

// --- ViewModel에서 사용 예시 ---
class SensorViewModel(application: Application) : AndroidViewModel(application) {

    // 가속도계 Flow — UI가 사라지면 viewModelScope 취소와 함께 자동 해제
    val accelerometerFlow: Flow<SensorData> = application
        .sensorFlow(Sensor.TYPE_ACCELEROMETER, SensorManager.SENSOR_DELAY_UI)

    // 자이로스코프 Flow
    val gyroscopeFlow: Flow<SensorData> = application
        .sensorFlow(Sensor.TYPE_GYROSCOPE, SensorManager.SENSOR_DELAY_GAME)
}
```

**UI에서 수집하는 방법 (Jetpack Compose)**

```kotlin
@Composable
fun SensorScreen(viewModel: SensorViewModel = viewModel()) {
    val sensorData by viewModel.accelerometerFlow
        .collectAsStateWithLifecycle(initialValue = null)

    Column(modifier = Modifier.padding(16.dp)) {
        Text("가속도계", style = MaterialTheme.typography.titleMedium)
        sensorData?.let { data ->
            Text("X: ${"%.3f".format(data.x)} m/s²")
            Text("Y: ${"%.3f".format(data.y)} m/s²")
            Text("Z: ${"%.3f".format(data.z)} m/s²")
        } ?: Text("센서 데이터 수신 중...")
    }
}
```

---

### 예제 2: 흔들기 감지(Shake Detection)와 만보기(Step Counter)

실전에서 자주 쓰이는 두 가지 기능을 구현합니다.

```kotlin
import android.content.Context
import android.hardware.Sensor
import android.hardware.SensorManager
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.flow.distinctUntilChanged
import kotlinx.coroutines.flow.map
import kotlin.math.sqrt

// ───────────────────────────────────────────────
// (1) 흔들기 감지: 중력을 제거한 선형 가속도의 크기가
//     임계값을 초과하면 Shake 이벤트를 방출합니다.
// ───────────────────────────────────────────────
class ShakeDetector(
    private val context: Context,
    private val shakeThresholdG: Float = 2.5f,   // 기본 임계값: 중력의 2.5배
    private val shakeSlopTimeMs: Long = 500L      // 반복 감지 억제 시간
) {
    private val _shakeEvents = MutableSharedFlow<Unit>(extraBufferCapacity = 1)
    val shakeEvents: Flow<Unit> = _shakeEvents

    private var lastShakeTime: Long = 0L

    // Linear Acceleration 센서: 중력이 이미 제거된 값
    val flow: Flow<Unit> = context
        .sensorFlow(Sensor.TYPE_LINEAR_ACCELERATION, SensorManager.SENSOR_DELAY_GAME)
        .map { data ->
            // 3축 벡터 크기 계산 (단위: m/s²)
            val magnitude = sqrt(data.x * data.x + data.y * data.y + data.z * data.z)
            // 중력(9.81 m/s²)으로 정규화 → G-force
            magnitude / SensorManager.GRAVITY_EARTH
        }
        .map { gForce ->
            val now = System.currentTimeMillis()
            if (gForce > shakeThresholdG && (now - lastShakeTime) > shakeSlopTimeMs) {
                lastShakeTime = now
                true // 흔들기 감지
            } else {
                false
            }
        }
        .map { isShake ->
            if (isShake) _shakeEvents.tryEmit(Unit)
        }

    // ViewModel에서 간단히 사용:
    // shakeDetector.shakeEvents.collect { showToast("흔들렸어요!") }
}

// ───────────────────────────────────────────────
// (2) 만보기: TYPE_STEP_COUNTER는 부팅 이후 누적 걸음 수를 반환합니다.
//     앱 실행 시 기준값(baseline)을 저장해 실제 걸음 수를 계산합니다.
// ───────────────────────────────────────────────
class StepCounterRepository(private val context: Context) {

    // 부팅 후 첫 수신 시 초기 값으로 사용
    private var baseline: Long? = null

    /**
     * 앱 실행 후 걸음 수 증분(delta)을 방출하는 Flow.
     * 앱을 재시작하면 baseline이 리셋됩니다.
     */
    val stepDeltaFlow: Flow<Long> = context
        .sensorFlow(Sensor.TYPE_STEP_COUNTER, SensorManager.SENSOR_DELAY_NORMAL)
        .map { data ->
            val totalSteps = data.x.toLong() // TYPE_STEP_COUNTER는 values[0]만 사용
            val base = baseline ?: totalSteps.also { baseline = it }
            (totalSteps - base).coerceAtLeast(0L)
        }
        .distinctUntilChanged() // 값이 변할 때만 방출

    /**
     * 만보기 앱 UI용 ViewModel 예시
     */
}

class StepCounterViewModel(application: Application) : AndroidViewModel(application) {
    private val repo = StepCounterRepository(application)

    val steps: Flow<Long> = repo.stepDeltaFlow

    // Compose에서 collectAsStateWithLifecycle()로 수집하면
    // 화면이 Background로 이동할 때 자동으로 수집 중단
}

// --- 만보기 Compose UI ---
@Composable
fun StepCounterScreen(viewModel: StepCounterViewModel = viewModel()) {
    val steps by viewModel.steps.collectAsStateWithLifecycle(initialValue = 0L)

    Box(
        modifier = Modifier.fillMaxSize(),
        contentAlignment = Alignment.Center
    ) {
        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            Text(
                text = "$steps",
                style = MaterialTheme.typography.displayLarge
            )
            Text(
                text = "걸음",
                style = MaterialTheme.typography.titleMedium,
                color = MaterialTheme.colorScheme.primary
            )
        }
    }
}
```

**AndroidManifest.xml 설정 (만보기 권한)**

```xml
<!-- Android 10+ 신체 활동 인식 권한 -->
<uses-permission android:name="android.permission.ACTIVITY_RECOGNITION" />

<!-- 특정 기기에서만 실행 가능하도록 하고 싶다면 -->
<uses-feature
    android:name="android.hardware.sensor.stepcounter"
    android:required="true" />
```

> Android 10(API 29) 이상에서는 `ACTIVITY_RECOGNITION` 런타임 권한을 사용자에게 요청해야 만보기 데이터를 읽을 수 있습니다. `ActivityResultLauncher`로 권한을 먼저 획득한 뒤 Flow를 구독하세요.

---

## 4. 센서 융합: 회전 벡터(Rotation Vector)로 방향 계산

자이로스코프와 가속도계, 지자기 센서를 소프트웨어적으로 결합한 `TYPE_ROTATION_VECTOR` 센서를 사용하면 기기의 절대적인 방향(방위각, 피치, 롤)을 안정적으로 얻을 수 있습니다.

```kotlin
data class DeviceOrientation(
    val azimuth: Float,  // Z축 회전: 0=북쪽, 90=동쪽 (도 단위)
    val pitch: Float,    // X축 회전: 앞뒤로 기울기
    val roll: Float      // Y축 회전: 좌우 기울기
)

fun Context.orientationFlow(): Flow<DeviceOrientation> =
    sensorFlow(Sensor.TYPE_ROTATION_VECTOR, SensorManager.SENSOR_DELAY_UI)
        .map { data ->
            val rotationMatrix = FloatArray(9)
            val orientationValues = FloatArray(3)

            // Rotation Vector → Rotation Matrix 변환
            SensorManager.getRotationMatrixFromVector(rotationMatrix, floatArrayOf(data.x, data.y, data.z))
            // Rotation Matrix → 오일러 각도(라디안) 변환
            SensorManager.getOrientation(rotationMatrix, orientationValues)

            DeviceOrientation(
                azimuth = Math.toDegrees(orientationValues[0].toDouble()).toFloat(),
                pitch   = Math.toDegrees(orientationValues[1].toDouble()).toFloat(),
                roll    = Math.toDegrees(orientationValues[2].toDouble()).toFloat()
            )
        }
```

이 Flow를 나침반 앱이나 AR 오버레이, 기울기 기반 게임 컨트롤러에 바로 연결할 수 있습니다.

---

## 5. 이동 평균 필터: 노이즈 제거

센서 데이터는 하드웨어 노이즈로 인해 순간적으로 튀는 값이 발생합니다. 이동 평균(Moving Average) 필터를 Flow 연산자로 적용하면 부드러운 데이터를 얻을 수 있습니다.

```kotlin
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.scan

/**
 * windowSize개의 최근 SensorData를 평균 내어 반환하는 확장 함수.
 * scan() 연산자를 사용해 누적 상태를 유지합니다.
 */
fun Flow<SensorData>.movingAverage(windowSize: Int = 5): Flow<SensorData> =
    scan(ArrayDeque<SensorData>()) { window, newData ->
        if (window.size >= windowSize) window.removeFirst()
        window.addLast(newData)
        window
    }
    .filter { it.isNotEmpty() }
    .map { window ->
        SensorData(
            x = window.map { it.x }.average().toFloat(),
            y = window.map { it.y }.average().toFloat(),
            z = window.map { it.z }.average().toFloat(),
            timestamp = window.last().timestamp
        )
    }

// 사용 예시
val smoothedAccelerometer: Flow<SensorData> = application
    .sensorFlow(Sensor.TYPE_ACCELEROMETER, SensorManager.SENSOR_DELAY_GAME)
    .movingAverage(windowSize = 10) // 10개 샘플 이동 평균
```

---

## 6. 주의사항 및 실전 팁

### 6-1. 배터리 절약이 최우선

| 상황 | 권장 샘플링 속도 |
|---|---|
| 만보기, 단순 기울기 감지 | `SENSOR_DELAY_NORMAL` (200ms) |
| UI 애니메이션 연동 | `SENSOR_DELAY_UI` (60ms) |
| 게임 입력, 제스처 인식 | `SENSOR_DELAY_GAME` (20ms) |
| 과학 측정, AR | `SENSOR_DELAY_FASTEST` (제한 없음) |

빠른 샘플링 속도는 배터리 소모와 CPU 부하를 급격히 증가시킵니다. 꼭 필요한 경우가 아니면 `SENSOR_DELAY_NORMAL`을 사용하세요.

### 6-2. 백그라운드에서의 센서 사용

`collectAsStateWithLifecycle()`는 앱이 백그라운드로 전환되면 Flow 수집을 자동으로 일시 중단합니다. 백그라운드에서도 센서 데이터가 필요한 경우(예: 피트니스 트래킹)에는 **Foreground Service**와 함께 사용해야 합니다.

```kotlin
// Foreground Service 내에서 사용할 때
class SensorForegroundService : LifecycleService() {
    private val viewModel: SensorViewModel by lazy { ... }

    override fun onCreate() {
        super.onCreate()
        startForeground(NOTIFICATION_ID, buildNotification())
        // lifecycleScope는 서비스가 살아있는 동안 유지됩니다
        lifecycleScope.launch {
            viewModel.accelerometerFlow.collect { data ->
                // 백그라운드 처리
            }
        }
    }
}
```

### 6-3. 센서 가용성 확인

모든 기기에 모든 센서가 탑재된 것은 아닙니다. `getDefaultSensor()`가 `null`을 반환하는 경우를 반드시 처리하세요. 위 `sensorFlow` 확장 함수에서는 `null` 반환 시 Flow를 즉시 `close()`하도록 구현했습니다.

```kotlin
// 센서 목록 전체 조회
val sensorManager = getSystemService(Context.SENSOR_SERVICE) as SensorManager
val availableSensors = sensorManager.getSensorList(Sensor.TYPE_ALL)
availableSensors.forEach { sensor ->
    Log.d("Sensor", "${sensor.name} | Power: ${sensor.power}mA")
}
```

### 6-4. `TYPE_STEP_COUNTER` vs `TYPE_STEP_DETECTOR`

| 속성 | TYPE_STEP_COUNTER | TYPE_STEP_DETECTOR |
|---|---|---|
| 반환값 | 부팅 이후 누적 총 걸음 수 | 매 걸음마다 `values[0] = 1.0` |
| 용도 | 누적 카운팅 | 즉각적인 걸음 감지 이벤트 |
| 배터리 | 낮음 (최적화된 칩) | 낮음 |
| 정확도 | 더 높음 | 낮음 (오감지 가능) |

만보기 앱에는 `TYPE_STEP_COUNTER`를, 실시간 걸음 피드백(진동, 소리)에는 `TYPE_STEP_DETECTOR`를 사용하세요.

### 6-5. 화면 방향 변환 시 좌표 재매핑

기기 방향이 세로/가로로 전환될 때 좌표계가 달라질 수 있습니다. 이때 `SensorManager.remapCoordinateSystem()`을 사용해 좌표를 화면 방향에 맞게 변환하세요.

```kotlin
val remappedMatrix = FloatArray(9)
SensorManager.remapCoordinateSystem(
    originalMatrix,
    SensorManager.AXIS_X,   // 새로운 X축 방향
    SensorManager.AXIS_Z,   // 새로운 Y축 방향
    remappedMatrix
)
```

---

## 7. 정리

Android `SensorManager`는 강력하지만 전통적인 콜백 방식은 생명주기 관리와 테스트에서 취약점이 많습니다. `callbackFlow`로 래핑하면:

- **생명주기 안전**: `awaitClose`가 구독 취소 시 리스너를 자동으로 해제합니다.
- **연산자 활용**: `map`, `filter`, `conflate`, `scan` 등으로 데이터 파이프라인을 선언적으로 구성합니다.
- **ViewModel 통합**: `viewModelScope`와 자연스럽게 결합해 UI 레이어와 명확하게 분리됩니다.
- **테스트 용이**: Turbine 라이브러리로 Flow를 단위 테스트할 수 있습니다.

흔들기 감지, 만보기, 나침반, AR 방향 계산 등 다양한 센서 활용 시나리오에 위 패턴을 바로 적용해 보세요. 센서를 올바르게 다루는 것만으로도 앱의 배터리 효율과 코드 품질을 동시에 높일 수 있습니다.

---

## 참고 자료

- [Android Sensors Overview — Android Developers](https://developer.android.com/develop/sensors-and-location/sensors/sensors_overview)
- [Motion Sensors — Android Developers](https://developer.android.com/develop/sensors-and-location/sensors/sensors_motion)
- [SensorManager API Reference — Android Developers](https://developer.android.com/reference/android/hardware/SensorManager)
