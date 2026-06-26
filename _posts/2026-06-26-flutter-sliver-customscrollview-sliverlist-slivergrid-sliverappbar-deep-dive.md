---
layout: post
title: "Flutter Sliver 완전 정복: CustomScrollView, SliverList, SliverGrid, SliverAppBar 심화"
date: 2026-06-26
categories: [flutter]
tags: [flutter, sliver, customscrollview, sliverlist, slivergrid, sliverappbar, scroll, performance]
---

앱을 개발하다 보면 단순한 `ListView`나 `GridView`로는 구현하기 어려운 복잡한 스크롤 UI가 필요할 때가 있습니다. 헤더가 스크롤에 따라 축소되거나, 리스트와 그리드가 한 화면에 자연스럽게 이어지거나, 특정 섹션이 화면 상단에 고정되는 등의 동작이 그 예입니다. Flutter에서는 이 모든 것을 **Sliver**라는 개념으로 해결합니다.

이 글에서는 Sliver의 내부 원리부터 시작하여 실전에서 바로 쓸 수 있는 구현 패턴까지 단계적으로 살펴보겠습니다.

---

## Sliver란 무엇인가?

 Flutter의 레이아웃 프로토콜은 크게 두 가지입니다. 하나는 우리가 흔히 쓰는 **박스 프로토콜(Box Protocol)**이고, 다른 하나가 스크롤에 특화된 **슬리버 프로토콜(Sliver Protocol)**입니다.

일반 위젯(`RenderBox`)은 주어진 제약(constraints) 안에서 자신의 크기를 결정하고 그립니다. 반면 Sliver(`RenderSliver`)는 스크롤 위치를 포함한 `SliverConstraints`를 입력받아 `SliverGeometry`를 반환합니다. `SliverGeometry`에는 스크롤 가능한 전체 길이(scrollExtent), 현재 화면에 실제로 그려지는 영역(paintExtent), 레이아웃에 영향을 주는 영역(layoutExtent) 등의 정보가 담깁니다.

이러한 구조 덕분에 Sliver는 뷰포트(Viewport)와 협력하여 현재 화면에 보이는 아이템만 렌더링하는 **지연 렌더링(Lazy Rendering)**을 구현합니다. `ListView`를 비롯한 대부분의 스크롤 위젯은 내부적으로 이 Sliver 프로토콜을 사용합니다.

```
[ScrollView]
  └── [Viewport]  ← 스크롤 위치, 픽셀 오프셋 관리
        ├── [SliverAppBar]    → SliverGeometry { scrollExtent: 200, paintExtent: 120 ... }
        ├── [SliverList]      → SliverGeometry { scrollExtent: 3000, paintExtent: 600 ... }
        └── [SliverGrid]      → SliverGeometry { scrollExtent: 1200, paintExtent: 0 ... }
```

---

## 왜 Sliver가 필요한가?

`ListView` + `Column`으로 복잡한 UI를 만들려고 하면 곧 한계에 부딪힙니다.

- **`ListView` 안에 `ListView`**: 스크롤 충돌이 발생하며, `shrinkWrap: true`를 쓰면 모든 아이템을 미리 렌더링해 성능이 급격히 저하됩니다.
- **콜랩시블 앱바와 리스트 조합**: `NestedScrollView`를 쓰더라도 세밀한 제어가 어렵습니다.
- **이질적인 섹션 혼합**: 헤더, 리스트, 그리드, 배너 등 서로 다른 형태의 콘텐츠를 하나의 스크롤로 연결하기 어렵습니다.

`CustomScrollView`와 Sliver 위젯들을 조합하면 이 모든 문제를 **단일 스크롤 컨텍스트** 안에서 우아하게 해결할 수 있습니다. 각 Sliver는 독립적으로 동작하면서도 하나의 스크롤 흐름을 공유합니다.

---

## 핵심 Sliver 위젯 해부

### CustomScrollView

`slivers` 프로퍼티에 Sliver 위젯 목록을 받아 단일 스크롤 영역을 구성하는 루트 위젯입니다.

```dart
CustomScrollView(
  slivers: [
    // Sliver 위젯들...
  ],
)
```

### SliverAppBar

스크롤에 반응하는 앱바입니다. 세 가지 주요 모드가 있습니다.

- `pinned: true` — 스크롤해도 앱바가 화면 상단에 고정됩니다.
- `floating: true` — 위로 스크롤하면 앱바가 즉시 나타납니다.
- `snap: true` — `floating`과 함께 사용하며, 절반만 보인 상태로 멈추지 않고 완전히 펼쳐지거나 접힙니다.
- `expandedHeight` — 완전히 펼쳐졌을 때의 높이를 지정합니다.

### SliverList

세로 방향의 리스트를 지연 렌더링합니다. `SliverChildBuilderDelegate`를 사용하면 아이템이 실제로 화면에 나타날 때만 위젯을 생성합니다.

### SliverGrid

그리드를 지연 렌더링합니다. `SliverGridDelegateWithFixedCrossAxisCount`(열 수 고정) 또는 `SliverGridDelegateWithMaxCrossAxisExtent`(최대 너비 고정) 두 가지 그리드 레이아웃 전략을 사용합니다.

### SliverToBoxAdapter

일반 Box 위젯을 Sliver 컨텍스트 안에 삽입할 때 사용하는 어댑터입니다. 헤더 배너, 섹션 타이틀 등 단일 위젯을 슬리버 목록 사이에 끼워 넣을 때 필수입니다.

### SliverPersistentHeader

커스텀 접히는 헤더를 만들 때 사용합니다. `SliverPersistentHeaderDelegate`를 구현하여 최대/최소 높이와 그에 따른 UI 변화를 직접 정의합니다.

---

## 실전 구현 예제 1: 쇼핑 앱 홈 화면

`SliverAppBar`로 콜랩시블 배너를 만들고, 그 아래에 카테고리 그리드와 상품 리스트를 연결하는 전형적인 쇼핑 앱 홈 화면 패턴입니다.

```dart
import 'package:flutter/material.dart';

class ShoppingHomePage extends StatelessWidget {
  const ShoppingHomePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: CustomScrollView(
        slivers: [
          // 1. 콜랩시블 앱바 (배너 이미지 포함)
          SliverAppBar(
            expandedHeight: 220,
            pinned: true,
            flexibleSpace: FlexibleSpaceBar(
              title: const Text(
                'Ducktudy Shop',
                style: TextStyle(shadows: [
                  Shadow(blurRadius: 4, color: Colors.black54),
                ]),
              ),
              background: Image.network(
                'https://picsum.photos/800/400',
                fit: BoxFit.cover,
              ),
              collapseMode: CollapseMode.parallax,
            ),
          ),

          // 2. 섹션 타이틀 (SliverToBoxAdapter로 일반 위젯 삽입)
          const SliverToBoxAdapter(
            child: Padding(
              padding: EdgeInsets.fromLTRB(16, 20, 16, 8),
              child: Text(
                '카테고리',
                style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
              ),
            ),
          ),

          // 3. 카테고리 그리드 (고정 높이)
          SliverGrid(
            gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
              crossAxisCount: 4,
              mainAxisSpacing: 8,
              crossAxisSpacing: 8,
              childAspectRatio: 0.85,
            ),
            delegate: SliverChildBuilderDelegate(
              (context, index) {
                final categories = ['전자', '패션', '식품', '뷰티', '스포츠', '가구', '도서', '여행'];
                final icons = [Icons.devices, Icons.checkroom, Icons.restaurant,
                  Icons.face, Icons.sports_basketball, Icons.chair,
                  Icons.menu_book, Icons.flight];
                return Padding(
                  padding: const EdgeInsets.symmetric(horizontal: 4),
                  child: Column(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      CircleAvatar(
                        radius: 28,
                        backgroundColor: Colors.indigo.shade50,
                        child: Icon(icons[index], color: Colors.indigo, size: 26),
                      ),
                      const SizedBox(height: 6),
                      Text(categories[index], style: const TextStyle(fontSize: 12)),
                    ],
                  ),
                );
              },
              childCount: 8,
            ),
          ),

          // 4. 상품 리스트 섹션 타이틀
          const SliverToBoxAdapter(
            child: Padding(
              padding: EdgeInsets.fromLTRB(16, 24, 16, 8),
              child: Text(
                '추천 상품',
                style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
              ),
            ),
          ),

          // 5. 상품 리스트 (지연 렌더링)
          SliverList(
            delegate: SliverChildBuilderDelegate(
              (context, index) {
                return Card(
                  margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 6),
                  child: ListTile(
                    leading: Container(
                      width: 56,
                      height: 56,
                      decoration: BoxDecoration(
                        color: Colors.grey.shade200,
                        borderRadius: BorderRadius.circular(8),
                      ),
                      child: const Icon(Icons.shopping_bag_outlined, color: Colors.grey),
                    ),
                    title: Text('상품 #${index + 1}'),
                    subtitle: Text('${(index + 1) * 9900}원'),
                    trailing: IconButton(
                      icon: const Icon(Icons.add_shopping_cart),
                      onPressed: () {},
                    ),
                  ),
                );
              },
              childCount: 30,
            ),
          ),

          // 6. 하단 여백
          const SliverToBoxAdapter(child: SizedBox(height: 32)),
        ],
      ),
    );
  }
}
```

이 예제에서 핵심은 그리드와 리스트가 **동일한 `CustomScrollView`의 `slivers` 배열 안에** 존재한다는 점입니다. 스크롤이 그리드를 모두 지나면 자연스럽게 리스트로 이어지며, 앱바는 스크롤에 따라 부드럽게 축소됩니다.

---

## 실전 구현 예제 2: 커스텀 접히는 헤더 (SliverPersistentHeader)

`SliverPersistentHeader`를 사용하면 `SliverAppBar`보다 훨씬 세밀한 헤더 애니메이션을 구현할 수 있습니다. 스크롤 비율(shrinkOffset / maxExtent)을 활용해 색상, 크기, 투명도 등을 자유롭게 조작합니다.

```dart
import 'package:flutter/material.dart';

// SliverPersistentHeaderDelegate 구현체
class _ProfileHeaderDelegate extends SliverPersistentHeaderDelegate {
  const _ProfileHeaderDelegate();

  @override
  double get minExtent => kToolbarHeight + 16; // 축소 시 최소 높이

  @override
  double get maxExtent => 280; // 완전히 펼쳐졌을 때 높이

  @override
  Widget build(BuildContext context, double shrinkOffset, bool overlapsContent) {
    // 0.0 (완전히 펼침) ~ 1.0 (완전히 접힘)
    final progress = shrinkOffset / maxExtent;
    // 아바타 크기: 72 → 32
    final avatarRadius = 36.0 - (36.0 - 16.0) * progress;
    // 배경 색상: 딥퍼플 → 흰색
    final bgColor = Color.lerp(Colors.deepPurple, Colors.white, progress)!;
    // 타이틀 색상: 흰색 → 검정색
    final titleColor = Color.lerp(Colors.white, Colors.black87, progress)!;

    return Container(
      color: bgColor,
      padding: const EdgeInsets.symmetric(horizontal: 16),
      child: Stack(
        children: [
          // 펼쳐진 상태에서만 보이는 설명 텍스트
          if (progress < 0.5)
            Positioned(
              bottom: 32,
              left: 0,
              right: 0,
              child: Opacity(
                opacity: (1 - progress * 2).clamp(0, 1),
                child: const Text(
                  'Flutter 개발을 공유합니다 🦆',
                  textAlign: TextAlign.center,
                  style: TextStyle(color: Colors.white70, fontSize: 14),
                ),
              ),
            ),

          // 항상 표시되는 아바타 + 이름 행
          Positioned(
            bottom: 8,
            left: 0,
            right: 0,
            child: Row(
              children: [
                CircleAvatar(
                  radius: avatarRadius,
                  backgroundColor: Colors.amber,
                  child: Text(
                    '🦆',
                    style: TextStyle(fontSize: avatarRadius * 0.9),
                  ),
                ),
                const SizedBox(width: 12),
                Expanded(
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    mainAxisSize: MainAxisSize.min,
                    children: [
                      Text(
                        'Ducktudy Dev',
                        style: TextStyle(
                          color: titleColor,
                          fontSize: 16 - 2 * progress,
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                      if (progress < 0.7)
                        Opacity(
                          opacity: (1 - progress / 0.7).clamp(0, 1),
                          child: Text(
                            '@ducktudy',
                            style: TextStyle(
                              color: titleColor.withOpacity(0.7),
                              fontSize: 12,
                            ),
                          ),
                        ),
                    ],
                  ),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }

  @override
  bool shouldRebuild(covariant _ProfileHeaderDelegate oldDelegate) => false;
}

// 사용 예시
class ProfilePage extends StatelessWidget {
  const ProfilePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: CustomScrollView(
        slivers: [
          // 커스텀 접히는 헤더
          const SliverPersistentHeader(
            pinned: true,
            delegate: _ProfileHeaderDelegate(),
          ),

          // 탭 바 (고정)
          SliverPersistentHeader(
            pinned: true,
            delegate: _StickyTabBarDelegate(
              tabBar: const TabBar(
                tabs: [Tab(text: '포스트'), Tab(text: '좋아요'), Tab(text: '저장됨')],
              ),
            ),
          ),

          // 콘텐츠 리스트
          SliverList(
            delegate: SliverChildBuilderDelegate(
              (context, index) => ListTile(
                leading: const Icon(Icons.article_outlined),
                title: Text('포스트 #${index + 1}'),
                subtitle: const Text('Flutter 팁 & 트릭'),
              ),
              childCount: 20,
            ),
          ),
        ],
      ),
    );
  }
}

// 탭바를 SliverPersistentHeader로 고정하기 위한 delegate
class _StickyTabBarDelegate extends SliverPersistentHeaderDelegate {
  const _StickyTabBarDelegate({required this.tabBar});
  final TabBar tabBar;

  @override
  double get minExtent => tabBar.preferredSize.height;

  @override
  double get maxExtent => tabBar.preferredSize.height;

  @override
  Widget build(BuildContext context, double shrinkOffset, bool overlapsContent) {
    return ColoredBox(
      color: Theme.of(context).scaffoldBackgroundColor,
      child: tabBar,
    );
  }

  @override
  bool shouldRebuild(covariant _StickyTabBarDelegate oldDelegate) =>
      tabBar != oldDelegate.tabBar;
}
```

이 패턴은 소셜 미디어 앱의 프로필 화면, 뉴스 앱의 카테고리 탭 화면 등에서 광범위하게 활용됩니다.

---

## 성능 최적화: SliverFixedExtentList와 SliverPrototypeExtentList

아이템 높이가 일정할 때 `SliverList` 대신 `SliverFixedExtentList`를 사용하면 성능이 크게 향상됩니다. Flutter가 스크롤 오프셋에서 어떤 아이템이 화면에 보여야 하는지를 **O(1)** 시간에 계산할 수 있기 때문입니다.

```dart
// 아이템 높이가 72px로 고정인 경우
SliverFixedExtentList(
  itemExtent: 72,
  delegate: SliverChildBuilderDelegate(
    (context, index) => ListTile(
      title: Text('아이템 $index'),
    ),
    childCount: 10000, // 1만 개여도 성능 이슈 없음
  ),
)
```

아이템 높이가 고정이 아니지만 프로토타입 위젯으로 높이를 측정할 수 있을 때는 `SliverPrototypeExtentList`를 사용합니다. `SliverList`보다 빠르고 `SliverFixedExtentList`와 달리 동적 콘텐츠에도 대응할 수 있습니다.

---

## 주의사항 및 실전 팁

### 1. `shrinkWrap: true`는 Sliver의 적

`ListView(shrinkWrap: true)`는 모든 자식 위젯을 한 번에 레이아웃하고 렌더링합니다. 아이템이 많을수록 **프레임 드롭과 jank**의 원인이 됩니다. `shrinkWrap: true`가 필요한 상황이라면 대부분 `SliverList`나 `SliverToBoxAdapter`로 대체할 수 있습니다.

### 2. `SliverToBoxAdapter` 안에서 무한 높이 위젯 사용 금지

`SliverToBoxAdapter`는 자식 위젯에게 무한 높이 제약을 전달하지 않습니다. 하지만 자식 위젯이 자체 크기를 결정할 수 없으면 오류가 발생합니다. `Column` 안에 `Expanded`를 쓰거나, 명시적 높이 없이 `ListView`를 중첩하면 이 오류가 발생합니다.

### 3. `SliverPersistentHeader.shouldRebuild` 최적화

`shouldRebuild`는 `SliverPersistentHeaderDelegate`의 상태가 변경됐을 때만 `true`를 반환하도록 작성하세요. 불필요한 리빌드를 방지하여 스크롤 성능을 유지할 수 있습니다.

### 4. 복잡한 Sliver 조합 — sliver_tools 패키지 활용

여러 Sliver를 하나의 논리적 그룹으로 묶어야 할 때(예: 섹션 헤더와 리스트를 함께 접기), `sliver_tools` 패키지의 `MultiSliver`를 활용하면 코드 구조가 훨씬 명확해집니다.

```dart
// sliver_tools: MultiSliver로 섹션 단위 Sliver 묶기
MultiSliver(
  pushPinnedChildren: true,
  children: [
    SliverPinnedHeader(child: _SectionHeader(title: '오늘의 추천')),
    SliverList(delegate: SliverChildBuilderDelegate(...)),
  ],
)
```

### 5. `CustomScrollView`의 `controller`와 `physics` 활용

프로그래매틱 스크롤(`animateTo`, `jumpTo`)이 필요하면 `ScrollController`를 연결하세요. iOS/Android의 기본 물리 효과(`BouncingScrollPhysics`, `ClampingScrollPhysics`)를 직접 지정하면 플랫폼 간 일관된 UX를 보장할 수 있습니다.

---

## 마치며

Flutter의 Sliver 시스템은 처음에는 다소 낯설지만, 한번 익히고 나면 복잡한 스크롤 UI를 놀라울 정도로 선언적이고 성능 효율적으로 구현할 수 있게 됩니다. `CustomScrollView` + `SliverAppBar` + `SliverList`/`SliverGrid`의 조합은 실무에서 가장 자주 쓰이는 패턴이며, `SliverPersistentHeader`를 마스터하면 디자인 팀이 요구하는 거의 모든 스크롤 인터랙션을 소화할 수 있습니다.

다음 단계로는 커스텀 `RenderSliver`를 직접 작성하거나, `SliverCrossAxisGroup`(Flutter 3.7+)으로 가로축 Sliver를 조합하는 것에 도전해 보세요.

---

## 참고 자료
- [flutter_staggered_grid_view — pub.dev](https://pub.dev/packages/flutter_staggered_grid_view)
- [sliver_tools — pub.dev](https://pub.dev/packages/sliver_tools)
