---
layout: post
title: "Android SurfaceView vs TextureView 심화: 고성능 커스텀 렌더링과 OpenGL ES 완전 정복"
date: 2026-11-01
categories: [android]
tags: [android, surfaceview, textureview, opengl-es, rendering, glsurfaceview, canvas, graphics]
---

Android에서 카메라 프리뷰, 게임 루프, 동영상 플레이백, 커스텀 그래픽 같은 **고성능 렌더링**이 필요할 때 `View.onDraw()`만으로는 한계가 있습니다. 메인 스레드(UI 스레드)에서만 그려야 하기 때문입니다. 이 문제를 해결하기 위해 Android는 `SurfaceView`, `TextureView`, `GLSurfaceView`라는 세 가지 특수 뷰를 제공합니다. 각각의 내부 동작 방식, 성능 특성, 올바른 사용 시나리오를 코드와 함께 완전히 정복해 보겠습니다.

---

## 1. 왜 일반 View로는 부족한가?

일반 Android `View`의 `onDraw(Canvas)`는 **UI 스레드**에서 실행됩니다. 복잡한 그래픽 연산을 메인 스레드에서 처리하면 다음 문제가 생깁니다.

- **ANR(Application Not Responding)**: 16ms(60fps) 안에 처리를 마치지 못하면 프레임 드랍 발생
- **입력 이벤트 블로킹**: 렌더링 중 터치·스크롤 이벤트를 처리하지 못함
- **UI 계층 동기화 비용**: `invalidate()` → `Choreographer` → `ViewRootImpl` 연쇄 호출

60fps를 유지하면서 복잡한 그래픽을 처리하려면 **별도 스레드에서 그릴 수 있는 Surface**가 필요합니다.

---

## 2. SurfaceView: 별도 Surface를 가진 뷰

### 2.1 동작 원리

`SurfaceView`는 **자체 독립 Surface**(Window Surface)를 갖습니다. 이 Surface는 일반 View 계층과 별도로 **SurfaceFlinger**에 의해 직접 합성됩니다.

```
앱 UI 계층 (ViewRootImpl)
  ┗━ SurfaceView (투명한 구멍 = "hole punch")
       ┗━ 독립 Surface → SurfaceFlinger → 디스플레이
```

- SurfaceView의 View 부분은 UI 계층에서 **투명한 구멍(hole-punch)**으로 표시됩니다.
- 실제 콘텐츠는 SurfaceFlinger가 구멍 아래에 합성합니다.
- **렌더링 스레드를 자유롭게 선택**할 수 있습니다.

### 2.2 커스텀 SurfaceView 구현 (Kotlin)

게임 루프 형태로 동작하는 커스텀 SurfaceView 예제입니다.

```kotlin
class GameSurfaceView(context: Context) : SurfaceView(context), SurfaceHolder.Callback {

    private var renderThread: Thread? = null
    private var isRunning = false

    // 움직이는 공의 위치
    private var ballX = 100f
    private var ballY = 100f
    private var velX = 8f
    private var velY = 6f
    private val radius = 50f

    private val paint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = Color.CYAN
        style = Paint.Style.FILL
    }
    private val bgPaint = Paint().apply {
        color = Color.parseColor("#1a1a2e")
    }

    init {
        holder.addCallback(this)
        // 투명 배경을 사용하려면: setZOrderOnTop(true) + PixelFormat.TRANSPARENT
    }

    override fun surfaceCreated(holder: SurfaceHolder) {
        isRunning = true
        renderThread = Thread {
            while (isRunning) {
                val canvas = holder.lockHardwareCanvas() ?: continue
                try {
                    update()
                    draw(canvas)
                } finally {
                    holder.unlockCanvasAndPost(canvas)
                }
                // 목표: ~60fps
                Thread.sleep(16)
            }
        }.also { it.start() }
    }

    override fun surfaceChanged(holder: SurfaceHolder, format: Int, width: Int, height: Int) {
        ballX = width / 2f
        ballY = height / 2f
    }

    override fun surfaceDestroyed(holder: SurfaceHolder) {
        isRunning = false
        renderThread?.join() // 스레드가 완전히 종료될 때까지 대기
        renderThread = null
    }

    private fun update() {
        val w = width.toFloat()
        val h = height.toFloat()
        ballX += velX
        ballY += velY
        if (ballX - radius < 0 || ballX + radius > w) velX = -velX
        if (ballY - radius < 0 || ballY + radius > h) velY = -velY
    }

    private fun draw(canvas: Canvas) {
        // 배경
        canvas.drawRect(0f, 0f, width.toFloat(), height.toFloat(), bgPaint)
        // 공
        canvas.drawCircle(ballX, ballY, radius, paint)
    }
}
```

> **핵심 포인트**
> - `lockHardwareCanvas()`는 GPU 가속 Canvas를 반환합니다. `lockCanvas()`보다 성능이 좋습니다.
> - `surfaceDestroyed()`에서 반드시 `renderThread?.join()`으로 스레드 종료를 기다려야 합니다. 스레드가 살아있는 채로 Surface가 파괴되면 크래시가 발생합니다.

---

## 3. TextureView: View 계층 안에서 렌더링

### 3.1 동작 원리

`TextureView`는 `SurfaceTexture`를 래핑하며, 콘텐츠를 View 계층 안의 OpenGL 텍스처로 처리합니다.

```
앱 View 계층
  ┗━ TextureView
       ┗━ SurfaceTexture (GL 텍스처로 캡처)
            ┗━ View 합성 시 GPU 텍스처로 복사 → 일반 View처럼 합성
```

- **장점**: 일반 View처럼 `alpha`, `rotation`, `scaleX/Y`, 애니메이션 모두 지원
- **단점**: GPU 텍스처 복사 단계(extra copy)가 추가되어 SurfaceView보다 소비 전력과 성능 측면에서 불리

### 3.2 언제 TextureView를 선택해야 하나?

| 시나리오 | 권장 뷰 |
|---|---|
| 전체 화면 동영상, 카메라, 게임 | `SurfaceView` |
| 카드 내 비디오 + 애니메이션 | `TextureView` |
| 스크롤 가능한 리스트 내 비디오 | `TextureView` |
| OpenGL ES 렌더링 | `GLSurfaceView` |
| 고성능 + 변환 필요 | `TextureView` (트레이드오프 감수) |

---

## 4. GLSurfaceView: OpenGL ES 렌더링의 표준 방법

### 4.1 GLSurfaceView 구조

`GLSurfaceView`는 `SurfaceView`의 하위 클래스로, EGL 컨텍스트 생성과 GL 렌더링 스레드 관리를 자동으로 처리합니다. 개발자는 `GLSurfaceView.Renderer` 인터페이스만 구현하면 됩니다.

```
GLSurfaceView
  ┣━ EGL 컨텍스트 자동 생성/소멸
  ┣━ GL 렌더링 스레드 자동 관리
  ┗━ Renderer.onSurfaceCreated()
       Renderer.onSurfaceChanged()
       Renderer.onDrawFrame()
```

### 4.2 OpenGL ES 2.0 삼각형 그리기 (Kotlin)

OpenGL ES 2.0으로 형광 삼각형을 그리는 완전한 예제입니다.

```kotlin
// 1. Renderer 구현
class TriangleRenderer : GLSurfaceView.Renderer {

    private val vertexShaderCode = """
        attribute vec4 vPosition;
        void main() {
            gl_Position = vPosition;
        }
    """.trimIndent()

    private val fragmentShaderCode = """
        precision mediump float;
        uniform vec4 vColor;
        void main() {
            gl_FragColor = vColor;
        }
    """.trimIndent()

    // 삼각형 좌표 (NDC: -1.0 ~ 1.0)
    private val triangleCoords = floatArrayOf(
         0.0f,  0.6f, 0.0f,   // 상단
        -0.6f, -0.3f, 0.0f,   // 좌측 하단
         0.6f, -0.3f, 0.0f    // 우측 하단
    )
    private val color = floatArrayOf(0.0f, 1.0f, 0.8f, 1.0f) // 청록색

    private var program = 0
    private lateinit var vertexBuffer: FloatBuffer

    override fun onSurfaceCreated(gl: GL10?, config: EGLConfig?) {
        GLES20.glClearColor(0.1f, 0.1f, 0.14f, 1.0f)

        // 버텍스 버퍼 생성
        vertexBuffer = ByteBuffer.allocateDirect(triangleCoords.size * 4)
            .order(ByteOrder.nativeOrder())
            .asFloatBuffer()
            .apply {
                put(triangleCoords)
                position(0)
            }

        // 셰이더 컴파일 및 프로그램 링크
        val vertexShader = loadShader(GLES20.GL_VERTEX_SHADER, vertexShaderCode)
        val fragmentShader = loadShader(GLES20.GL_FRAGMENT_SHADER, fragmentShaderCode)
        program = GLES20.glCreateProgram().also {
            GLES20.glAttachShader(it, vertexShader)
            GLES20.glAttachShader(it, fragmentShader)
            GLES20.glLinkProgram(it)
        }
    }

    override fun onSurfaceChanged(gl: GL10?, width: Int, height: Int) {
        GLES20.glViewport(0, 0, width, height)
    }

    override fun onDrawFrame(gl: GL10?) {
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT)
        GLES20.glUseProgram(program)

        val positionHandle = GLES20.glGetAttribLocation(program, "vPosition")
        GLES20.glEnableVertexAttribArray(positionHandle)
        GLES20.glVertexAttribPointer(positionHandle, 3, GLES20.GL_FLOAT, false, 12, vertexBuffer)

        val colorHandle = GLES20.glGetUniformLocation(program, "vColor")
        GLES20.glUniform4fv(colorHandle, 1, color, 0)

        GLES20.glDrawArrays(GLES20.GL_TRIANGLES, 0, 3)
        GLES20.glDisableVertexAttribArray(positionHandle)
    }

    private fun loadShader(type: Int, code: String): Int =
        GLES20.glCreateShader(type).also { shader ->
            GLES20.glShaderSource(shader, code)
            GLES20.glCompileShader(shader)
        }
}

// 2. Activity에서 GLSurfaceView 설정
class OpenGLActivity : AppCompatActivity() {

    private lateinit var glSurfaceView: GLSurfaceView

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        glSurfaceView = GLSurfaceView(this).apply {
            setEGLContextClientVersion(2)          // OpenGL ES 2.0 사용
            setRenderer(TriangleRenderer())
            renderMode = GLSurfaceView.RENDERMODE_CONTINUOUSLY  // 매 프레임 그리기
        }
        setContentView(glSurfaceView)
    }

    // 반드시 생명주기를 GLSurfaceView에 전달해야 함
    override fun onResume() { super.onResume(); glSurfaceView.onResume() }
    override fun onPause()  { super.onPause();  glSurfaceView.onPause()  }
}
```

> **`RENDERMODE_WHEN_DIRTY`**: 애니메이션이 없는 정적 GL 씬(지도, 3D 모델 뷰어 등)에는 `RENDERMODE_WHEN_DIRTY`를 사용하고 `requestRender()`로 명시적 갱신을 트리거하세요. 불필요한 GPU 사용을 크게 줄일 수 있습니다.

---

## 5. SurfaceView vs TextureView vs GLSurfaceView 완전 비교

| 항목 | SurfaceView | TextureView | GLSurfaceView |
|---|---|---|---|
| **렌더링 스레드** | 임의 스레드 | 임의 스레드 | 전용 GL 스레드 (자동) |
| **합성 방식** | SurfaceFlinger 직접 합성 | View 계층 GPU 텍스처 복사 | SurfaceFlinger 직접 합성 |
| **Alpha/애니메이션** | ❌ (기본 불가) | ✅ | ❌ (기본 불가) |
| **추가 GPU 복사** | 없음 | 있음 (비용 발생) | 없음 |
| **EGL 관리** | 직접 설정 필요 | 직접 설정 필요 | 자동 관리 |
| **소비 전력** | 낮음 | 보통~높음 | 낮음 |
| **주요 사용처** | 카메라, 게임, 영상 | 카드형 비디오, 애니메이션 | OpenGL ES 렌더링 |

---

## 6. 주의사항 및 실전 팁

### 6.1 SurfaceView: Z-order와 투명도 함정

SurfaceView는 기본적으로 View 계층보다 **아래(Z-order 하단)**에 그려집니다. 게임 UI(버튼, HUD)를 SurfaceView 위에 올리려면 UI를 별도 ViewGroup으로 올려야 합니다.

```kotlin
// 투명한 SurfaceView를 맨 위에 올리는 경우
surfaceView.setZOrderOnTop(true)
surfaceView.holder.setFormat(PixelFormat.TRANSPARENT)
```

`setZOrderOnTop(true)`를 사용하면 SurfaceView가 UI보다 위에 그려지므로, 이 경우 Surface 배경을 명시적으로 투명하게 설정해야 합니다.

### 6.2 surfaceDestroyed 후 렌더링 스레드 안전 종료

Surface가 파괴된 이후 `holder.lockCanvas()`를 호출하면 `IllegalArgumentException`이 발생합니다. `isRunning` 플래그 외에도 `holder.surface.isValid` 체크를 추가하는 것이 안전합니다.

```kotlin
while (isRunning) {
    if (!holder.surface.isValid) continue
    val canvas = holder.lockHardwareCanvas() ?: continue
    // ...
}
```

### 6.3 GLSurfaceView 생명주기를 Activity에 전달해야 하는 이유

`onPause()`/`onResume()`을 전달하지 않으면 앱이 백그라운드로 가도 GL 렌더링 스레드가 계속 동작하거나, 포그라운드 복귀 시 EGL 컨텍스트가 복구되지 않아 화면이 검게 됩니다.

### 6.4 TextureView의 "Surface가 아직 준비되지 않음" 문제

`TextureView`의 Surface는 뷰가 레이아웃된 이후에 준비됩니다. `SurfaceTextureListener`로 준비 완료를 감지해야 합니다.

```kotlin
textureView.surfaceTextureListener = object : TextureView.SurfaceTextureListener {
    override fun onSurfaceTextureAvailable(surface: SurfaceTexture, w: Int, h: Int) {
        // Surface 준비 완료 → 카메라 프리뷰 시작 등
        startCameraPreview(Surface(surface))
    }
    override fun onSurfaceTextureSizeChanged(s: SurfaceTexture, w: Int, h: Int) {}
    override fun onSurfaceTextureDestroyed(s: SurfaceTexture): Boolean = true
    override fun onSurfaceTextureUpdated(s: SurfaceTexture) {}
}
```

### 6.5 Compose와 SurfaceView 연동

Jetpack Compose에서는 `AndroidView`로 `SurfaceView`를 임베드할 수 있습니다.

```kotlin
@Composable
fun GameScreen() {
    AndroidView(
        factory = { context -> GameSurfaceView(context) },
        modifier = Modifier.fillMaxSize()
    )
}
```

Compose의 리컴포지션이 일어나도 `factory` 블록은 한 번만 실행되므로, 게임 루프는 영향을 받지 않습니다.

---

## 7. 언제 무엇을 선택할 것인가?

- **전체 화면 게임, 카메라 프리뷰(전통적)**: `SurfaceView` — 최소 오버헤드, 별도 스레드 렌더링
- **리스트나 카드 안의 비디오/애니메이션**: `TextureView` — View 변환 지원, 성능 트레이드오프 감수
- **OpenGL ES / Vulkan 커스텀 그래픽**: `GLSurfaceView` — EGL 관리 자동화, 검증된 패턴
- **CameraX 사용**: `PreviewView` 내부적으로 최적 뷰를 선택하므로 직접 선택 불필요
- **최신 고성능 요구사항**: `SurfaceControl` API(API 29+)로 더 세밀한 합성 제어 가능

SurfaceView의 "hole-punch" 특성과 TextureView의 "추가 GPU 복사" 비용을 이해하면, 자신의 앱 시나리오에 맞는 올바른 뷰를 자신 있게 선택할 수 있습니다. 고성능 렌더링은 올바른 도구 선택에서 시작됩니다.

---

## 참고 자료

- [SurfaceView — Android API Reference](https://developer.android.com/reference/android/view/SurfaceView)
- [TextureView — Android API Reference](https://developer.android.com/reference/android/view/TextureView)
- [GLSurfaceView — Android API Reference](https://developer.android.com/reference/android/opengl/GLSurfaceView)
- [OpenGL ES 시작하기 — Android Developers](https://developer.android.com/develop/ui/views/graphics/opengl/about-opengl)
