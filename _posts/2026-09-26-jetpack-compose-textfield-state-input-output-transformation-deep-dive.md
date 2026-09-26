---
layout: post
title: "Jetpack Compose TextField 심화: TextFieldState, InputTransformation, OutputTransformation 완전 정복"
date: 2026-09-26
categories: [android, flutter]
tags: [android, jetpack-compose, textfield, textfieldstate, inputtransformation, outputtransformation, kotlin]
---

Jetpack Compose가 도입된 이후 `TextField`는 끊임없이 진화해왔습니다. 초기의 `value`/`onValueChange` 기반 API는 소프트웨어 키보드와의 동기화 문제, 복잡한 오프셋 매핑 등 여러 고질적인 문제를 안고 있었습니다. Compose Foundation 1.6.0부터 공식적으로 도입된 **상태 기반(state-based) TextField** 패러다임은 이러한 문제들을 근본적으로 해결하며, 커스텀 텍스트 입력 경험을 구현하는 방식을 완전히 바꾸어 놓았습니다. 이 글에서는 `TextFieldState`, `InputTransformation`, `OutputTransformation`을 깊이 파고들어 실무에 바로 적용 가능한 수준의 이해를 쌓겠습니다.

---

## 1. 왜 기존 방식이 문제였는가

### value/onValueChange의 구조적 한계

기존 `TextField`는 다음과 같은 패턴으로 사용했습니다.

```kotlin
var text by remember { mutableStateOf("") }
TextField(
    value = text,
    onValueChange = { text = it }
)
```

이 방식의 가장 큰 문제는 **비동기(async) 업데이트**입니다. 사용자가 타이핑하면 IME(Input Method Engine, 소프트웨어 키보드)가 텍스트를 변경하고, 이를 `onValueChange`가 받아 상태를 업데이트한 뒤, Compose 리컴포지션을 통해 `value`를 다시 TextField에 전달합니다. 이 과정에서 한 프레임이라도 지연이 발생하면 키보드와 화면의 텍스트가 일시적으로 불일치하는 **글리치(glitch)**가 나타납니다. 특히 텍스트를 필터링하거나 포맷팅하는 로직을 `onValueChange` 안에 넣으면 이 문제가 심각해집니다.

또한 `VisualTransformation`을 사용해 텍스트를 시각적으로 변환할 경우, **오프셋 매핑(offset mapping)**을 직접 구현해야 했습니다. 예를 들어 `1234-5678-9012-3456` 형태의 카드 번호를 표시할 때, 원본 인덱스와 변환된 인덱스 사이의 매핑을 `OffsetMapping` 인터페이스로 수동으로 계산해야 했는데, 이 계산이 미묘하게 틀리면 커서가 엉뚱한 위치로 점프하는 버그가 발생했습니다.

---

## 2. 새로운 패러다임: TextFieldState

### TextFieldState란

`TextFieldState`는 TextField의 텍스트 내용과 선택 영역(selection), 조합 중인 텍스트(composition)를 하나의 객체에서 완전히 관리하는 상태 홀더입니다. 상태가 단일 진실 공급원(single source of truth) 역할을 하며, IME와의 동기화를 프레임워크 레벨에서 처리합니다.

```kotlin
@Composable
fun BasicTextFieldStateExample() {
    val textState = rememberTextFieldState()

    Column(modifier = Modifier.padding(16.dp)) {
        BasicTextField(
            state = textState,
            modifier = Modifier
                .fillMaxWidth()
                .border(1.dp, MaterialTheme.colorScheme.outline, RoundedCornerShape(8.dp))
                .padding(12.dp)
        )
        Spacer(modifier = Modifier.height(8.dp))
        Text(
            text = "입력된 텍스트: ${textState.text}",
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
        Text(
            text = "글자 수: ${textState.text.length}",
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
    }
}
```

`rememberTextFieldState()`는 상태 객체를 생성하고 화면 회전이나 프로세스 재시작에서도 자동으로 복원되도록 `rememberSaveable`과 동일한 수준의 생명주기 관리를 제공합니다.

### TextFieldState를 ViewModel과 통합하기

실무에서는 상태를 ViewModel에서 관리해야 합니다. 공식적으로 권장하는 방법은 `TextFieldState`를 ViewModel에 직접 보관하는 것입니다.

```kotlin
class SignUpViewModel : ViewModel() {
    val emailState = TextFieldState()
    val passwordState = TextFieldState()

    fun submit() {
        val email = emailState.text.toString()
        val password = passwordState.text.toString()
        // 유효성 검사 및 API 호출
    }
}

@Composable
fun SignUpScreen(viewModel: SignUpViewModel = viewModel()) {
    Column(modifier = Modifier.padding(24.dp)) {
        OutlinedTextField(
            state = viewModel.emailState,
            label = { Text("이메일") },
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Email,
                imeAction = ImeAction.Next
            ),
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(12.dp))
        OutlinedTextField(
            state = viewModel.passwordState,
            label = { Text("비밀번호") },
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Password,
                imeAction = ImeAction.Done
            ),
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(24.dp))
        Button(
            onClick = { viewModel.submit() },
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("가입하기")
        }
    }
}
```

ViewModel에 `TextFieldState`를 직접 두면, 더 이상 `snapshotFlow`나 `collectAsState`로 상태를 변환할 필요가 없습니다. Compose가 직접 `TextFieldState`를 관찰합니다.

---

## 3. InputTransformation: 입력 필터링

`InputTransformation`은 사용자가 타이핑하는 순간, 텍스트가 상태에 저장되기 **전에** 개입하여 내용을 필터링하거나 변환합니다. 즉, 상태 자체를 변환하는 역할입니다.

### 최대 글자수 제한

가장 간단한 예는 최대 글자수 제한입니다. `InputTransformation.maxLength(n)`이라는 내장 변환이 이미 제공됩니다.

```kotlin
BasicTextField(
    state = rememberTextFieldState(),
    inputTransformation = InputTransformation.maxLength(10)
)
```

### 숫자만 허용하는 InputTransformation

```kotlin
val digitsOnlyTransformation = InputTransformation { _, valueWithChanges ->
    // valueWithChanges는 TextFieldBuffer이며, 변경을 취소하거나 수정할 수 있습니다
    val filteredText = valueWithChanges.toString().filter { it.isDigit() }
    if (filteredText != valueWithChanges.toString()) {
        valueWithChanges.replace(0, valueWithChanges.length, filteredText)
    }
}
```

### 여러 변환 체이닝

`InputTransformation`은 `then` 연산자로 체이닝할 수 있습니다. 예를 들어 숫자만 허용하면서 최대 10자리로 제한하고 싶다면:

```kotlin
val phoneInputTransformation = digitsOnlyTransformation
    .then(InputTransformation.maxLength(11))
```

이렇게 하면 두 변환이 순서대로 적용됩니다.

---

## 4. OutputTransformation: 시각적 포맷팅

`OutputTransformation`은 상태에 저장된 텍스트는 그대로 두고, 화면에 표시할 때만 포맷을 변환합니다. 기존 `VisualTransformation`과 달리 **오프셋 매핑을 자동으로 처리**해주기 때문에 훨씬 간결하게 구현할 수 있습니다.

### 전화번호 포맷팅: OutputTransformation 실전 예제

```kotlin
/**
 * 숫자 11자리를 010-1234-5678 형태로 시각적으로 변환합니다.
 * 실제 저장되는 값은 01012345678 (숫자만) 입니다.
 */
val phoneNumberOutputTransformation = OutputTransformation { buffer ->
    // buffer: TextFieldBuffer — 변환된 텍스트를 이 버퍼에 작성
    if (buffer.length >= 3) {
        buffer.insert(3, "-")
    }
    if (buffer.length >= 8) {
        buffer.insert(8, "-")
    }
}

@Composable
fun PhoneNumberTextField() {
    val phoneState = rememberTextFieldState()

    Column(modifier = Modifier.padding(16.dp)) {
        OutlinedTextField(
            state = phoneState,
            label = { Text("전화번호") },
            inputTransformation = InputTransformation.maxLength(11)
                .then(InputTransformation { _, changes ->
                    // 숫자만 허용
                    val digits = changes.toString().filter { it.isDigit() }
                    if (digits != changes.toString()) {
                        changes.replace(0, changes.length, digits)
                    }
                }),
            outputTransformation = phoneNumberOutputTransformation,
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Phone
            ),
            modifier = Modifier.fillMaxWidth()
        )
        Text(
            text = "저장된 값: ${phoneState.text}",  // 01012345678
            style = MaterialTheme.typography.bodySmall
        )
    }
}
```

`outputTransformation`은 `TextFieldBuffer`를 받아 `insert`, `replace`, `append` 등의 메서드로 표시용 텍스트를 조작합니다. 오프셋 매핑은 프레임워크가 알아서 계산하므로, 구현자는 "어디에 어떤 문자를 삽입할지"만 신경 쓰면 됩니다.

### 신용카드 번호 포맷팅: 완성형 예제

```kotlin
/**
 * 16자리 숫자를 1234 5678 9012 3456 형태로 표시합니다.
 */
val creditCardOutputTransformation = OutputTransformation { buffer ->
    var i = 4
    while (i < buffer.length) {
        buffer.insert(i, " ")
        i += 5  // 공백 삽입 후 다음 그룹 시작 위치
    }
}

val creditCardInputTransformation = InputTransformation { _, changes ->
    val digits = changes.toString().filter { it.isDigit() }
    if (digits != changes.toString()) {
        changes.replace(0, changes.length, digits)
    }
}.then(InputTransformation.maxLength(16))

@Composable
fun CreditCardTextField() {
    val cardState = rememberTextFieldState()

    OutlinedTextField(
        state = cardState,
        label = { Text("카드 번호") },
        inputTransformation = creditCardInputTransformation,
        outputTransformation = creditCardOutputTransformation,
        keyboardOptions = KeyboardOptions(
            keyboardType = KeyboardType.Number,
            imeAction = ImeAction.Next
        ),
        placeholder = { Text("0000 0000 0000 0000") },
        modifier = Modifier.fillMaxWidth()
    )
}
```

---

## 5. TextLayoutResult와 커서 포지셔닝

고급 시나리오에서는 `TextLayoutResult`를 활용해 텍스트의 레이아웃 정보(각 문자의 위치, 줄 높이 등)를 얻을 수 있습니다. `BasicTextField`의 `onTextLayout` 파라미터로 접근합니다.

```kotlin
@Composable
fun HighlightableTextField() {
    val state = rememberTextFieldState("검색어를 입력하세요")
    var textLayoutResult by remember { mutableStateOf<TextLayoutResult?>(null) }
    val searchQuery = "입력"

    Box(
        modifier = Modifier
            .fillMaxWidth()
            .border(1.dp, Color.Gray, RoundedCornerShape(4.dp))
            .padding(8.dp)
    ) {
        BasicTextField(
            state = state,
            onTextLayout = { textLayoutCallback ->
                textLayoutResult = textLayoutCallback()
            },
            decorator = { innerTextField ->
                Box {
                    // 검색어 하이라이트 오버레이
                    textLayoutResult?.let { layout ->
                        val text = state.text.toString()
                        var startIndex = text.indexOf(searchQuery)
                        while (startIndex != -1) {
                            val endIndex = startIndex + searchQuery.length
                            if (endIndex <= text.length) {
                                val boundingBoxes = layout.getPathForRange(startIndex, endIndex)
                                Canvas(modifier = Modifier.matchParentSize()) {
                                    drawPath(
                                        path = boundingBoxes,
                                        color = Color.Yellow.copy(alpha = 0.4f)
                                    )
                                }
                            }
                            startIndex = text.indexOf(searchQuery, startIndex + 1)
                        }
                    }
                    innerTextField()
                }
            }
        )
    }
}
```

---

## 6. 주의사항과 실전 팁

### 1. value/onValueChange API 지원 중단 예고

Compose Foundation 1.13.0-alpha03부터 `value, onValueChange` 파라미터를 받는 `BasicTextField` 오버로드가 공식적으로 deprecated 되었습니다. 새 프로젝트는 반드시 `TextFieldState` 기반 API를 사용하고, 기존 프로젝트는 마이그레이션 가이드(`developer.android.com/develop/ui/compose/text/migrate-state-based`)를 참고해 이전하세요.

### 2. InputTransformation은 상태를 바꾼다

`InputTransformation`은 실제 `TextFieldState`에 저장되는 텍스트를 변환합니다. 따라서 필터링 로직이 잘못되면 `textState.text`의 값 자체가 의도하지 않게 바뀔 수 있습니다. 반면 `OutputTransformation`은 시각적 표현만 바꾸므로, 저장 값을 보존하고 표시 형식만 다르게 하고 싶다면 반드시 `OutputTransformation`을 사용해야 합니다.

### 3. ViewModel의 TextFieldState는 직렬화되지 않는다

`TextFieldState`는 `rememberTextFieldState()`를 통해 Composable에서 선언하면 자동 저장 복원이 되지만, ViewModel에 직접 선언하면 프로세스 재시작 시 내용이 사라집니다. 중요한 입력값은 ViewModel의 `SavedStateHandle`과 연동하거나, `snapshotFlow`를 통해 외부 저장소에 백업하는 로직을 추가하세요.

### 4. SecureTextField 사용

비밀번호 필드에는 Compose Foundation 1.13+에서 `SecureTextField`를 사용하세요. `TextObfuscationMode.System`이 기본값으로 설정되어 있어, 플랫폼의 보안 입력 정책을 자동으로 따릅니다. 별도의 `VisualTransformation`으로 마스킹을 구현할 필요가 없습니다.

### 5. 성능: 불필요한 리컴포지션 방지

`TextFieldState.text`는 `AnnotatedString`을 반환합니다. 화면에서 이 값을 다른 곳에 바인딩할 때 `snapshotFlow { state.text.toString() }`를 사용하면, 텍스트가 실제로 변경될 때만 흐름이 방출되어 불필요한 리컴포지션을 방지할 수 있습니다.

```kotlin
LaunchedEffect(textState) {
    snapshotFlow { textState.text.toString() }
        .debounce(300.milliseconds)
        .distinctUntilChanged()
        .collect { query ->
            viewModel.search(query)
        }
}
```

---

## 마치며

Jetpack Compose의 TextField는 `TextFieldState` 기반의 새 API로 전환되면서, 개발자가 흔히 겪던 글리치와 오프셋 계산의 복잡성이 크게 해소되었습니다. `InputTransformation`으로 입력 자체를 제어하고, `OutputTransformation`으로 시각적 표현을 분리하는 패턴을 익히면, 전화번호, 카드번호, OTP 입력 등 다양한 커스텀 텍스트 필드를 훨씬 안정적이고 간결하게 구현할 수 있습니다. 새 프로젝트는 처음부터 `TextFieldState`를 채택하고, 기존 프로젝트도 단계적으로 마이그레이션하기를 강력히 권장합니다.

## 참고 자료
- [Configure text fields | Jetpack Compose | Android Developers](https://developer.android.com/develop/ui/compose/text/user-input)
- [Migrate to state-based text fields | Jetpack Compose | Android Developers](https://developer.android.com/develop/ui/compose/text/migrate-state-based)
- [Compose Foundation 릴리즈 노트 | Android Developers](https://developer.android.com/jetpack/androidx/releases/compose-foundation)
