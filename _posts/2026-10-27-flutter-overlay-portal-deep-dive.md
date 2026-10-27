---
layout: post
title: "Flutter OverlayPortal 심화: OverlayEntry·CompositedTransformFollower·OverlayPortal로 전역 플로팅 UI 완전 정복"
date: 2026-10-27
categories: [android, flutter]
tags: [flutter, overlay, overlayportal, overlayentry, compositedtransform, dart, dropdown, tooltip]
---

Flutter 앱에서 Dialog, Snackbar, BottomSheet, Tooltip은 모두 **Overlay** 라는 공통 기반 위에서 작동한다. Overlay는 MaterialApp이 내부적으로 생성하는 특수한 Stack 위젯으로, 앱의 가장 상단 레이어에 위젯을 삽입할 수 있게 해준다. 일반 Stack이 특정 부모 위젯의 경계 안에서만 동작하는 것과 달리, Overlay는 앱 전체 화면을 캔버스로 사용할 수 있다.

## 핵심 컴포넌트 개념 설명

Flutter의 오버레이 시스템을 이루는 핵심 컴포넌트는 세 가지다.

### Overlay & OverlayEntry

`Overlay`는 `Navigator`와 함께 `MaterialApp` 또는 `WidgetsApp` 안에 자동으로 생성된다. `OverlayEntry`는 이 Overlay에 삽입할 레이어 하나를 나타내는 객체다. **명령형(imperative) 방식**으로 동작한다.

```dart
// 삽입
final entry = OverlayEntry(builder: (context) => MyOverlayWidget());
Overlay.of(context).insert(entry);

// 업데이트 (setState 대신 markNeedsBuild 사용)
entry.markNeedsBuild();

// 제거
entry.remove();
```

`OverlayEntry`의 핵심 특징:
- `builder`가 반환하는 위젯은 Overlay의 전체 화면 크기를 기본으로 사용한다
- 리빌드가 필요할 때 `markNeedsBuild()`를 수동으로 호출해야 한다
- 생명주기를 직접 관리해야 하므로 `dispose()`에서 `remove()` 호출이 필수다

### OverlayPortal (Flutter 3.7+)

`OverlayPortal`은 Flutter 3.7에서 도입된 **선언형(declarative) 오버레이 API**다. `OverlayPortalController`로 show/hide 상태를 제어하고, `overlayChildBuilder`로 오버레이 위젯을 선언한다. `OverlayEntry`와 달리 위젯 트리 안에 자연스럽게 포함되므로, 부모의 `Theme`, `Provider`, `InheritedWidget` 등을 자동으로 상속받는다.

가장 중요한 성능 특징은 **오버레이가 보이지 않을 때 `overlayChildBuilder`가 호출되지 않는다는 점**이다. 기존 `OverlayEntry`는 builder가 항상 존재하지만 `OverlayPortal`은 `isShowing == false`일 때 빌드 자체를 건너뛴다.

### CompositedTransformTarget & CompositedTransformFollower

드롭다운, 툴팁, 팝오버처럼 **특정 위젯의 위치에 따라 오버레이가 따라다녀야 하는 경우**, `LayerLink`와 두 위젯을 함께 사용한다.

- **`CompositedTransformTarget`**: "여기를 기준으로 삼아라" 역할. 추적될 위젯을 감싼다
- **`CompositedTransformFollower`**: "저 Target을 따라다녀라" 역할. 오버레이 위젯을 감싼다
- **`LayerLink`**: Target과 Follower를 연결하는 공유 키 객체

이 세 가지를 조합하면 오버레이가 스크롤, 애니메이션, 레이아웃 변경에도 자동으로 올바른 위치를 유지한다. RenderObject 레이어 수준에서 동작하기 때문에 `setState` 없이 GPU 레벨에서 부드럽게 위치를 추적한다.

---

## 왜 필요한가?

### 일반 Stack의 한계

Widget 트리 안의 일반 `Stack`은 자신의 부모 범위를 벗어날 수 없다.

```dart
Stack(
  clipBehavior: Clip.none, // 시각적으로는 벗어나도...
  children: [
    MyButton(),
    Positioned(
      top: -100, // 부모 Stack 영역 밖
      child: DropdownMenu(), // 형제 위젯에 가려질 수 있음
    ),
  ],
)
```

`Clip.none`으로 시각적으로는 부모 경계를 벗어나더라도, Z-order 문제로 다른 형제 위젯에 덮일 수 있다. 또한 실제 Overlay 레이어가 아니므로 ModalBarrier와 같은 상호작용 차단도 동작하지 않는다.

### Overlay가 해결하는 시나리오

| 사용 사례 | 설명 |
|-----------|------|
| 커스텀 드롭다운 | 버튼 아래에 위치하며 다른 위젯 위에 렌더링 |
| 앱 내 툴팁 | 특정 위젯에 앵커된 말풍선 |
| 전역 로딩 스피너 | 화면 전환과 무관하게 유지 |
| 컨텍스트 메뉴 | 롱프레스 시 터치 위치에 메뉴 표시 |
| 온보딩 하이라이트 | 특정 위젯을 강조하고 설명 추가 |

---

## 실제 구현 예제

### 예제 1: OverlayPortal + CompositedTransformFollower로 커스텀 드롭다운 구현

`CompositedTransformTarget`과 `CompositedTransformFollower`를 결합해 버튼 바로 아래 표시되는 드롭다운을 구현한다.

```dart
class AnchoredDropdown extends StatefulWidget {
  final List<String> items;
  final void Function(String) onSelected;

  const AnchoredDropdown({
    super.key,
    required this.items,
    required this.onSelected,
  });

  @override
  State<AnchoredDropdown> createState() => _AnchoredDropdownState();
}

class _AnchoredDropdownState extends State<AnchoredDropdown> {
  final _controller = OverlayPortalController();
  final _layerLink = LayerLink();
  String _selected = '항목 선택';

  @override
  Widget build(BuildContext context) {
    return CompositedTransformTarget(
      link: _layerLink,
      child: OverlayPortal(
        controller: _controller,
        overlayChildBuilder: (context) {
          return CompositedTransformFollower(
            link: _layerLink,
            // Target의 좌측 하단 → Follower의 좌측 상단 정렬
            targetAnchor: Alignment.bottomLeft,
            followerAnchor: Alignment.topLeft,
            child: Align(
              alignment: AlignmentDirectional.topStart,
              child: Material(
                elevation: 8,
                borderRadius: BorderRadius.circular(8),
                child: SizedBox(
                  width: 200,
                  child: Column(
                    mainAxisSize: MainAxisSize.min,
                    children: widget.items.map((item) {
                      return ListTile(
                        dense: true,
                        title: Text(item),
                        selected: item == _selected,
                        onTap: () {
                          setState(() => _selected = item);
                          _controller.hide();
                          widget.onSelected(item);
                        },
                      );
                    }).toList(),
                  ),
                ),
              ),
            ),
          );
        },
        child: OutlinedButton.icon(
          onPressed: () {
            _controller.isShowing
                ? _controller.hide()
                : _controller.show();
          },
          icon: Icon(
            _controller.isShowing
                ? Icons.arrow_drop_up
                : Icons.arrow_drop_down,
            size: 20,
          ),
          label: Text(_selected),
        ),
      ),
    );
  }
}
```

`CompositedTransformFollower`의 `targetAnchor`는 Target 위젯의 어느 지점을 기준점으로 사용할지, `followerAnchor`는 Follower 위젯의 어느 지점을 해당 기준점에 맞출지를 지정한다. `Alignment.bottomLeft` → `Alignment.topLeft` 조합은 버튼의 좌측 하단에 드롭다운의 좌측 상단을 맞추는 전형적인 패턴이다.

### 예제 2: GlobalKey를 활용한 전역 로딩 오버레이 (OverlayEntry)

앱 어디서나 호출 가능한 전역 로딩 인디케이터를 구현한다. `Navigator`의 최상위 Overlay에 삽입하므로 화면 전환과 무관하게 유지된다.

```dart
class GlobalOverlayManager {
  static OverlayEntry? _loadingEntry;
  static final _navigatorKey = GlobalKey<NavigatorState>();

  static GlobalKey<NavigatorState> get navigatorKey => _navigatorKey;

  static void showLoading({String? message}) {
    final overlayState = _navigatorKey.currentState?.overlay;
    if (overlayState == null || _loadingEntry != null) return;

    _loadingEntry = OverlayEntry(
      builder: (context) => Stack(
        children: [
          // 배경 터치 이벤트 차단
          const ModalBarrier(
            color: Color(0x66000000),
            dismissible: false,
          ),
          Center(
            child: Card(
              shape: RoundedRectangleBorder(
                borderRadius: BorderRadius.circular(16),
              ),
              child: Padding(
                padding: const EdgeInsets.symmetric(
                  horizontal: 32,
                  vertical: 24,
                ),
                child: Column(
                  mainAxisSize: MainAxisSize.min,
                  children: [
                    const CircularProgressIndicator(),
                    if (message != null) ...[
                      const SizedBox(height: 16),
                      Text(
                        message,
                        style: const TextStyle(fontSize: 14),
                      ),
                    ],
                  ],
                ),
              ),
            ),
          ),
        ],
      ),
    );

    overlayState.insert(_loadingEntry!);
  }

  static void hideLoading() {
    _loadingEntry?.remove();
    _loadingEntry = null;
  }
}
```

`main.dart`에서 `navigatorKey`를 연결하면 앱 어디서든 `GlobalOverlayManager`로 로딩 오버레이를 제어할 수 있다.

```dart
// main.dart
MaterialApp(
  navigatorKey: GlobalOverlayManager.navigatorKey,
  home: const MyHomePage(),
);

// 비동기 작업에서 사용:
Future<void> submitForm() async {
  GlobalOverlayManager.showLoading(message: '저장 중...');
  try {
    await repository.save(data);
  } finally {
    GlobalOverlayManager.hideLoading();
  }
}
```

---

## 주의사항 및 팁

### 1. BuildContext와 Theme/Provider 상속 문제

`OverlayEntry`의 builder는 **Overlay의 context에서 실행되므로**, 현재 위젯 트리의 Theme이나 Provider를 직접 상속하지 않는다. 이를 해결하려면 두 가지 방법이 있다.

```dart
// 방법 1: OverlayPortal 사용 (위젯 트리 내 context 자동 상속 — 권장)

// 방법 2: OverlayEntry에서 현재 context 값을 캡처
final theme = Theme.of(context); // 삽입 전에 미리 가져오기
final provider = context.read<MyProvider>();

_entry = OverlayEntry(
  builder: (_) => Provider.value(
    value: provider,
    child: Theme(
      data: theme, // 캡처한 theme 명시적 전달
      child: MyOverlay(),
    ),
  ),
);
```

### 2. OverlayEntry 메모리 누수 방지

`StatefulWidget`의 `dispose()`에서 반드시 `OverlayEntry.remove()`를 호출해야 한다. Route가 pop될 때 자동으로 제거되지 않으므로, 위젯이 destroy되어도 오버레이가 화면에 남는 문제가 발생할 수 있다.

```dart
@override
void dispose() {
  _overlayEntry?.remove(); // 반드시 remove 호출
  _overlayEntry = null;
  super.dispose();
}
```

### 3. CompositedTransformFollower의 위치 제약

`CompositedTransformFollower`는 Overlay 레이어 또는 최소한 `CompositedTransformTarget`보다 상위 레이어에 있어야 한다. Follower를 일반 위젯 트리에 배치하면 Target보다 먼저 렌더링되어 위치 계산이 올바르게 동작하지 않는다. 항상 `OverlayPortal` 또는 `OverlayEntry`의 builder 안에서 사용해야 한다.

### 4. OverlayPortal vs OverlayEntry 선택 기준

| 상황 | 권장 API |
|------|---------|
| 특정 위젯에 연결된 팝업(드롭다운, 툴팁) | `OverlayPortal` + `CompositedTransformFollower` |
| 전역 싱글톤 오버레이(로딩, 에러 토스트) | `OverlayEntry` + `GlobalKey<NavigatorState>` |
| Provider·Theme 상속이 중요한 경우 | `OverlayPortal` (트리 내 context 자동 상속) |
| Flutter 3.7 미만 지원 필요 | `OverlayEntry` |
| 복잡한 선언형 오버레이 정렬 | `flutter_portal` 패키지 |

### 5. 화면 전환 시 오버레이 처리

`OverlayPortal`의 오버레이는 해당 Widget이 unmount되면 자동으로 사라진다. 반면 `OverlayEntry`로 Navigator의 최상위 Overlay에 직접 삽입한 경우, Route 전환과 무관하게 유지된다. 전역 로딩 인디케이터는 이 특성을 의도적으로 활용하지만, 특정 화면에 종속된 오버레이라면 `RouteAware`를 구현하거나 `dispose()`에서 제거해야 한다.

```dart
class _MyPageState extends State<MyPage> with RouteAware {
  @override
  void didPushNext() {
    // 다음 화면으로 이동 시 오버레이 숨기기
    _overlayController.hide();
  }
}
```

---

## 참고 자료
- [OverlayPortal class - Flutter API](https://api.flutter.dev/flutter/widgets/OverlayPortal-class.html)
- [Overlay class - Flutter API](https://api.flutter.dev/flutter/widgets/Overlay-class.html)
- [flutter_portal 패키지 - pub.dev](https://pub.dev/packages/flutter_portal)
- [flutter_portal API 문서](https://pub.dev/documentation/flutter_portal/latest/)
