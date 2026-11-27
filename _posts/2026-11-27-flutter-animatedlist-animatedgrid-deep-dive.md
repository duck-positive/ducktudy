---
layout: post
title: "Flutter AnimatedList & AnimatedGrid 심화: 삽입·삭제 애니메이션으로 살아 숨쉬는 목록 UI 구현하기"
date: 2026-11-27
categories: [android, flutter]
tags: [flutter, animatedlist, animatedgrid, animation, dart, ui]
---

Flutter 앱에서 목록에 항목을 추가하거나 삭제할 때, 항목이 그냥 "툭" 나타나거나 사라지면 사용자는 순간적으로 맥락을 잃게 됩니다. `AnimatedList`와 `AnimatedGrid`는 이 문제를 해결하기 위해 Flutter가 제공하는 위젯으로, 항목의 삽입·삭제 순간에 개발자가 정의한 애니메이션을 자연스럽게 삽입해 줍니다. 단순한 `ListView.builder`로는 불가능한 "살아 움직이는" 목록 경험을 만들 수 있습니다.

## 개념 설명: AnimatedList와 AnimatedGrid란?

`AnimatedList`는 `ListView.builder`와 유사하게 작동하지만, 항목이 추가되거나 제거될 때 자동으로 전환 애니메이션을 실행하는 스크롤 가능한 위젯입니다. 핵심은 `AnimatedListState`를 통해 항목의 삽입과 삭제를 직접 제어한다는 점입니다.

`AnimatedGrid`는 Flutter 3.7에서 도입된 위젯으로, `AnimatedList`의 그리드(격자) 버전입니다. `GridView.builder`처럼 열과 행으로 구성된 레이아웃에서 항목 애니메이션을 지원합니다. 두 위젯 모두 스크롤 가능한 뷰 안에 녹아드는 Sliver 버전(`SliverAnimatedList`, `SliverAnimatedGrid`)도 존재하므로, `CustomScrollView`와 조합하면 더욱 복잡한 레이아웃도 구성할 수 있습니다.

### 핵심 API 구조

```
AnimatedList / AnimatedGrid
  ├── key: GlobalKey<AnimatedListState> / GlobalKey<AnimatedGridState>
  ├── initialItemCount: 초기 항목 수
  ├── itemBuilder: (context, index, animation) => Widget
  └── [gridDelegate: SliverGridDelegate]  ← AnimatedGrid 전용

AnimatedListState / AnimatedGridState
  ├── insertItem(index, {duration})    ← 삽입 시 호출
  └── removeItem(index, builder, {duration})  ← 삭제 시 호출
```

중요한 점은 `insertItem`과 `removeItem`을 호출하기 **전에** 실제 데이터 모델(List)을 먼저 수정해야 한다는 것입니다. 위젯 내부적으로 항목 수를 추적하므로 모델과 동기화가 어긋나면 `RangeError`가 발생합니다.

---

## 왜 필요한가?

### 사용자 경험 측면

목록이 갑자기 변경되면 사용자는 "무슨 일이 일어났는지" 파악하기 어렵습니다. 쇼핑 카트에서 항목을 삭제할 때 슬라이드 아웃 애니메이션이 있다면, 사용자는 자신의 행동과 결과를 직접 연결합니다. 이는 단순한 미적 요소가 아니라 UX의 핵심 원칙인 **피드백**의 구현입니다.

### 기존 방식의 한계

`ListView.builder`는 `setState()`로 데이터를 바꾸면 목록을 즉시 다시 그립니다. 중간 상태(애니메이션)를 표현할 방법이 없습니다. `AnimatedList`는 삽입/삭제 시 `Animation<double>` 객체를 `itemBuilder`에 전달하여, 개발자가 원하는 어떤 전환 효과든 적용할 수 있게 합니다.

---

## 실제 구현 예제

### 예제 1: AnimatedList를 활용한 쇼핑 카트

가장 흔한 패턴입니다. `GlobalKey<AnimatedListState>`를 선언해 State에 직접 접근하고, 항목 추가/삭제 시 각각 `insertItem`과 `removeItem`을 호출합니다.

```dart
import 'package:flutter/material.dart';

class ShoppingCartPage extends StatefulWidget {
  const ShoppingCartPage({super.key});

  @override
  State<ShoppingCartPage> createState() => _ShoppingCartPageState();
}

class _ShoppingCartPageState extends State<ShoppingCartPage> {
  final GlobalKey<AnimatedListState> _listKey = GlobalKey<AnimatedListState>();
  final List<String> _items = ['사과', '바나나', '오렌지'];

  void _addItem() {
    final newItem = '상품 ${_items.length + 1}';
    // 1. 모델에 먼저 추가
    _items.add(newItem);
    // 2. AnimatedListState에 알림 (마지막 인덱스)
    _listKey.currentState?.insertItem(
      _items.length - 1,
      duration: const Duration(milliseconds: 400),
    );
  }

  void _removeItem(int index) {
    final removedItem = _items[index];
    // 1. AnimatedListState에 먼저 알림 (builder는 사라지는 애니메이션용)
    _listKey.currentState?.removeItem(
      index,
      (context, animation) => _buildItem(removedItem, animation),
      duration: const Duration(milliseconds: 300),
    );
    // 2. 모델에서 제거
    _items.removeAt(index);
  }

  Widget _buildItem(String item, Animation<double> animation) {
    return SizeTransition(
      sizeFactor: animation,
      child: Card(
        margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
        child: ListTile(
          leading: const Icon(Icons.shopping_bag_outlined, color: Colors.deepOrange),
          title: Text(item, style: const TextStyle(fontWeight: FontWeight.w500)),
          trailing: IconButton(
            icon: const Icon(Icons.delete_outline, color: Colors.red),
            onPressed: () => _removeItem(_items.indexOf(item)),
          ),
        ),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('쇼핑 카트'),
        backgroundColor: Colors.deepOrange,
        foregroundColor: Colors.white,
      ),
      body: AnimatedList(
        key: _listKey,
        initialItemCount: _items.length,
        itemBuilder: (context, index, animation) {
          return _buildItem(_items[index], animation);
        },
      ),
      floatingActionButton: FloatingActionButton.extended(
        onPressed: _addItem,
        label: const Text('상품 추가'),
        icon: const Icon(Icons.add),
        backgroundColor: Colors.deepOrange,
      ),
    );
  }
}
```

**포인트 해설**:

- `SizeTransition`은 `animation`을 `sizeFactor`로 넘겨 항목이 위아래로 접히고 펼쳐지는 효과를 냅니다. `FadeTransition`이나 `SlideTransition`으로 교체하면 완전히 다른 느낌을 줄 수 있습니다.
- `removeItem`의 `builder` 콜백은 삭제 애니메이션 도중 화면에 그릴 위젯을 반환합니다. 실제 모델에서는 이미 제거됐지만 애니메이션이 끝날 때까지 화면에 잔류하는 "유령 위젯"입니다.
- 삭제 순서: `removeItem` 호출 → 애니메이션 시작 → `_items.removeAt(index)` 순서를 지켜야 인덱스 계산 오류가 없습니다.

---

### 예제 2: AnimatedGrid를 활용한 사진 갤러리

`AnimatedGrid`는 Flutter 3.7+ 에서만 사용 가능합니다. `gridDelegate`를 반드시 지정해야 한다는 점이 `AnimatedList`와 다릅니다.

```dart
import 'package:flutter/material.dart';

class PhotoGalleryPage extends StatefulWidget {
  const PhotoGalleryPage({super.key});

  @override
  State<PhotoGalleryPage> createState() => _PhotoGalleryPageState();
}

class _PhotoGalleryPageState extends State<PhotoGalleryPage> {
  final GlobalKey<AnimatedGridState> _gridKey = GlobalKey<AnimatedGridState>();

  // 색상으로 사진을 모의
  final List<Color> _photos = [
    Colors.red.shade200,
    Colors.blue.shade200,
    Colors.green.shade200,
    Colors.amber.shade200,
    Colors.purple.shade200,
    Colors.teal.shade200,
  ];

  void _addPhoto() {
    final colors = [
      Colors.pink.shade200,
      Colors.indigo.shade200,
      Colors.lime.shade200,
      Colors.cyan.shade200,
    ];
    final newColor = colors[_photos.length % colors.length];
    _photos.add(newColor);
    _gridKey.currentState?.insertItem(
      _photos.length - 1,
      duration: const Duration(milliseconds: 500),
    );
  }

  void _removePhoto(int index) {
    final removedColor = _photos[index];
    _gridKey.currentState?.removeItem(
      index,
      (context, animation) => _buildPhotoCell(removedColor, animation, index),
      duration: const Duration(milliseconds: 400),
    );
    _photos.removeAt(index);
  }

  Widget _buildPhotoCell(Color color, Animation<double> animation, int index) {
    return ScaleTransition(
      scale: CurvedAnimation(parent: animation, curve: Curves.easeOut),
      child: GestureDetector(
        onLongPress: () => _removePhoto(index),
        child: Container(
          decoration: BoxDecoration(
            color: color,
            borderRadius: BorderRadius.circular(12),
            boxShadow: [
              BoxShadow(
                color: color.withOpacity(0.4),
                blurRadius: 8,
                offset: const Offset(0, 4),
              ),
            ],
          ),
          child: Center(
            child: Icon(
              Icons.photo,
              size: 40,
              color: Colors.white.withOpacity(0.8),
            ),
          ),
        ),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('갤러리 (${_photos.length}장)'),
        backgroundColor: Colors.indigo,
        foregroundColor: Colors.white,
      ),
      body: Padding(
        padding: const EdgeInsets.all(12),
        child: AnimatedGrid(
          key: _gridKey,
          initialItemCount: _photos.length,
          gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
            crossAxisCount: 3,
            mainAxisSpacing: 10,
            crossAxisSpacing: 10,
          ),
          itemBuilder: (context, index, animation) {
            return _buildPhotoCell(_photos[index], animation, index);
          },
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: _addPhoto,
        backgroundColor: Colors.indigo,
        child: const Icon(Icons.add_photo_alternate),
      ),
    );
  }
}
```

**포인트 해설**:

- `ScaleTransition`에 `CurvedAnimation`을 조합하면 "탄력 있는" 등장 효과를 줄 수 있습니다. `Curves.elasticOut`을 써도 재미있는 느낌이 납니다.
- `onLongPress`로 삭제를 트리거하는 패턴은 iOS 사진 앱의 편집 모드와 비슷한 UX를 제공합니다.
- `gridDelegate`의 `crossAxisCount`를 `MediaQuery`로 반응형으로 설정하면 태블릿에서도 자연스럽게 작동합니다.

---

## SliverAnimatedList: CustomScrollView와의 조합

앱 상단에 `SliverAppBar`나 헤더를 붙이고 싶다면 `SliverAnimatedList`를 사용하세요. `AnimatedList` 대신 Sliver 버전을 쓰면 `CustomScrollView` 내부에서 다른 Sliver 위젯들과 자연스럽게 조합됩니다.

```dart
CustomScrollView(
  slivers: [
    const SliverAppBar(
      title: Text('알림'),
      floating: true,
    ),
    SliverAnimatedList(
      key: _sliverListKey,
      initialItemCount: _notifications.length,
      itemBuilder: (context, index, animation) {
        return FadeTransition(
          opacity: animation,
          child: NotificationTile(item: _notifications[index]),
        );
      },
    ),
  ],
)
```

`SliverAnimatedList`의 State 접근 방식은 `AnimatedList`와 동일합니다. `GlobalKey<SliverAnimatedListState>`를 선언하고 `insertItem`/`removeItem`을 호출합니다.

---

## 주의사항 및 팁

### 1. 모델과 상태 동기화 순서를 반드시 지켜라

- **삽입**: 데이터 추가 → `insertItem()`
- **삭제**: `removeItem()` → 데이터 제거

이 순서를 지키지 않으면 `AnimatedList`가 추적하는 인덱스와 실제 데이터의 인덱스가 어긋나 `RangeError`나 의도치 않은 항목이 화면에 표시됩니다.

### 2. removeItem의 builder는 반드시 구현하라

`removeItem`의 두 번째 인자인 `AnimatedRemovedItemBuilder`는 nullable이 아닙니다. 삭제 애니메이션 도중 화면에 보여줄 위젯을 반드시 반환해야 합니다. `SizedBox.shrink()`를 반환하면 애니메이션 없이 즉시 사라지는 것처럼 보입니다.

### 3. 복수 항목 삽입 시 인덱스 주의

루프로 여러 항목을 한 번에 삽입할 때는 순서에 주의하세요:

```dart
// 올바른 방법: 삽입할 항목들을 먼저 모두 모델에 추가
final startIndex = _items.length;
_items.addAll(newItems);
for (int i = 0; i < newItems.length; i++) {
  _listKey.currentState?.insertItem(startIndex + i);
}
```

### 4. AnimatedGrid는 Flutter 3.7+ 필수

`AnimatedGrid`와 `SliverAnimatedGrid`는 Flutter 3.7.0에서 도입되었습니다. `pubspec.yaml`의 `environment.sdk`와 Flutter 버전을 반드시 확인하세요. 그 이하 버전에서는 `AnimatedList` + `Wrap` 조합으로 유사한 효과를 구현해야 합니다.

### 5. 성능: 애니메이션 duration 조절

항목이 많을 때 삭제 duration이 너무 길면 사용자가 답답함을 느낄 수 있습니다. 일반적으로 삽입은 300~500ms, 삭제는 200~350ms가 적당합니다. 또한 `AnimatedList`는 모든 가시 항목에 대해 `itemBuilder`를 호출하므로, `itemBuilder` 내부를 가볍게 유지하는 것이 중요합니다.

### 6. 스크롤 위치 복원

`AnimatedList`는 `ScrollController`를 `controller` 프로퍼티로 받습니다. 항목 추가 후 맨 아래로 스크롤하고 싶다면 `insertItem` 직후에 `scrollController.animateTo(scrollController.position.maxScrollExtent, ...)`를 호출하면 됩니다. 단, `insertItem` 직후는 레이아웃이 완성되지 않았으므로 `WidgetsBinding.instance.addPostFrameCallback`으로 다음 프레임에서 실행하세요.

```dart
void _addItemAndScroll() {
  _items.add('새 항목');
  _listKey.currentState?.insertItem(_items.length - 1);
  WidgetsBinding.instance.addPostFrameCallback((_) {
    _scrollController.animateTo(
      _scrollController.position.maxScrollExtent,
      duration: const Duration(milliseconds: 300),
      curve: Curves.easeOut,
    );
  });
}
```

---

## 마무리

`AnimatedList`와 `AnimatedGrid`는 단순해 보이지만, 올바르게 사용하려면 데이터 모델과 위젯 상태의 동기화 순서를 정확히 이해해야 합니다. 핵심 규칙은 **삽입 시에는 모델 먼저, 삭제 시에는 removeItem 먼저**입니다. 이 원칙을 지키면서 `SizeTransition`, `FadeTransition`, `ScaleTransition` 등을 조합하면 사용자가 "어디서 왔고 어디로 갔는지" 직관적으로 이해할 수 있는 목록 UI를 만들 수 있습니다. `CustomScrollView`가 필요한 복잡한 화면에서는 `SliverAnimatedList`와 `SliverAnimatedGrid`로 자연스럽게 확장하세요.

## 참고 자료
- [AnimatedList class — Flutter API](https://api.flutter.dev/flutter/widgets/AnimatedList-class.html)
- [AnimatedGrid class — Flutter API](https://api.flutter.dev/flutter/widgets/AnimatedGrid-class.html)
- [SliverAnimatedList class — Flutter API](https://api.flutter.dev/flutter/widgets/SliverAnimatedList-class.html)
- [Flutter Animation & Motion Widgets](https://docs.flutter.dev/ui/widgets/animation)
