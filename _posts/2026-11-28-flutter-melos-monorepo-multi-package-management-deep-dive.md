---
layout: post
title: "Flutter Monorepo 심화: melos로 멀티 패키지 프로젝트 관리하기"
date: 2026-11-28
categories: [android, flutter]
tags: [flutter, melos, monorepo, dart, package, workspace, ci-cd]
---

앱 규모가 커질수록 단일 저장소에서 여러 패키지를 함께 관리해야 하는 상황이 생깁니다. 이 글에서는 Flutter/Dart 프로젝트를 위한 모노레포 도구인 **melos**를 심층적으로 다루고, 실전 환경에서 멀티 패키지 프로젝트를 체계적으로 관리하는 방법을 알아봅니다.

## 모노레포란 무엇이고 왜 필요한가

**모노레포(Monorepo)**는 여러 개의 독립적인 프로젝트나 패키지를 하나의 Git 저장소에서 관리하는 방식입니다. Flutter 생태계에서는 다음과 같은 상황에서 모노레포가 큰 도움이 됩니다.

- 여러 앱이 공통 비즈니스 로직, 디자인 시스템, 네트워킹 레이어를 공유할 때
- 플러그인을 플랫폼별로 분리해 연합(federated) 플러그인 구조로 개발할 때
- 디자인 시스템 패키지와 앱을 함께 개발하면서 즉각적인 피드백이 필요할 때
- 의존성 버전을 한 곳에서 일관되게 관리해야 할 때

단순히 `path` 의존성을 사용해 연결할 수도 있지만, 패키지 수가 늘어나면 부트스트랩, 버전 관리, 스크립트 실행 등이 금방 복잡해집니다. 이 문제를 해결하는 것이 **melos**입니다.

## melos 개요

[melos](https://pub.dev/packages/melos)는 JavaScript 생태계의 Lerna에서 영감을 받아 Invertase팀이 만든 Dart/Flutter 전용 모노레포 관리 도구입니다. 최신 버전(8.x)은 Dart 3.9.0 이상과 Dart pub workspaces를 활용해 로컬 패키지 연결을 처리합니다.

주요 기능:
- **워크스페이스 연결**: pub workspaces를 활용해 로컬 패키지를 의존성 오버라이드 없이 연결
- **자동 버전 관리**: Conventional Commits 기반 자동 버전 범핑과 CHANGELOG 생성
- **커스텀 스크립트**: 루트에서 여러 패키지에 걸쳐 명령을 일괄 실행
- **패키지 필터링**: 이름, glob, 변경된 파일, 의존 관계 기준으로 패키지 선택
- **CI/CD 통합**: 변경된 패키지만 테스트·배포하는 증분 빌드 지원

## 프로젝트 구조 설계

실전에서 자주 사용하는 디렉토리 구조는 다음과 같습니다.

```
my_app_monorepo/
├── pubspec.yaml          # 루트 pubspec (pub workspace 선언)
├── melos.yaml            # melos 설정 (스크립트, 필터 등)
├── apps/
│   ├── customer_app/     # 고객용 Flutter 앱
│   └── driver_app/       # 기사용 Flutter 앱
└── packages/
    ├── core/             # 비즈니스 로직, 도메인 모델
    ├── ui_components/    # 공유 디자인 시스템
    ├── network/          # API 클라이언트
    └── analytics/        # 이벤트 추적
```

**핵심 원칙**: 앱은 패키지에 의존할 수 있지만, 패키지는 앱에 의존하면 안 됩니다. 이 단방향 의존성이 모노레포 아키텍처를 건강하게 유지합니다.

## melos 설치 및 초기 설정

먼저 melos를 전역으로 설치합니다.

```bash
dart pub global activate melos
```

루트 `pubspec.yaml`에 pub workspace를 선언합니다.

```yaml
# pubspec.yaml (루트)
name: my_app_monorepo
environment:
  sdk: ">=3.9.0 <4.0.0"

workspace:
  - apps/customer_app
  - apps/driver_app
  - packages/core
  - packages/ui_components
  - packages/network
  - packages/analytics
```

각 멤버 패키지의 `pubspec.yaml`에는 `resolution: workspace`를 추가합니다.

```yaml
# packages/core/pubspec.yaml
name: core
environment:
  sdk: ">=3.9.0 <4.0.0"
resolution: workspace

dependencies:
  freezed_annotation: ^2.4.1
  json_annotation: ^4.9.0
```

이제 `melos.yaml`을 작성합니다.

```yaml
# melos.yaml
name: my_app_monorepo

packages:
  - apps/**
  - packages/**

scripts:
  analyze:
    run: dart analyze .
    description: 전체 워크스페이스 정적 분석

  test:all:
    run: melos exec -- dart test
    description: 모든 패키지 테스트 실행
    packageFilters:
      dirExists: test

  test:changed:
    run: melos exec -- dart test
    description: 변경된 패키지만 테스트
    packageFilters:
      dirExists: test
      diff: origin/main

  format:
    run: dart format .
    description: 코드 포매팅

  build:runner:
    run: dart run build_runner build --delete-conflicting-outputs
    description: build_runner 실행
    packageFilters:
      dependsOn: build_runner

  clean:
    run: melos exec -- flutter clean
    description: 모든 패키지 clean
    packageFilters:
      flutter: true
```

워크스페이스를 초기화하려면 루트에서 실행합니다.

```bash
melos bootstrap
```

이 명령은 모든 패키지의 의존성을 설치하고, pub workspaces를 통해 로컬 패키지 간 연결을 설정합니다.

## 실전 코드 예제 1: 공유 패키지 생성 및 연결

`packages/core`에 공유 도메인 모델을 만들어 보겠습니다.

```dart
// packages/core/lib/src/models/user.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user.freezed.dart';
part 'user.g.dart';

@freezed
class User with _$User {
  const factory User({
    required String id,
    required String name,
    required String email,
    @Default(UserRole.customer) UserRole role,
  }) = _User;

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}

enum UserRole { customer, driver, admin }
```

```dart
// packages/core/lib/src/repositories/user_repository.dart
abstract class UserRepository {
  Future<User> fetchUser(String id);
  Future<void> updateUser(User user);
  Stream<User> watchUser(String id);
}
```

```dart
// packages/core/lib/core.dart
export 'src/models/user.dart';
export 'src/repositories/user_repository.dart';
```

이제 `customer_app`에서 `core` 패키지를 사용합니다.

```yaml
# apps/customer_app/pubspec.yaml
name: customer_app
environment:
  sdk: ">=3.9.0 <4.0.0"
resolution: workspace

dependencies:
  flutter:
    sdk: flutter
  core: any         # pub workspace가 로컬 경로로 자동 연결
  ui_components: any
```

```dart
// apps/customer_app/lib/features/profile/profile_page.dart
import 'package:core/core.dart';
import 'package:flutter/material.dart';

class ProfilePage extends StatelessWidget {
  final User user;

  const ProfilePage({super.key, required this.user});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('프로필')),
      body: Column(
        children: [
          Text(user.name),
          Text(user.email),
          Text(user.role.name),
        ],
      ),
    );
  }
}
```

## 실전 코드 예제 2: Conventional Commits 기반 자동 버전 관리

melos의 버전 관리는 Git 커밋 메시지 규칙을 따릅니다.

| 커밋 접두어 | 버전 범핑 |
|-------------|-----------|
| `fix:` | patch (1.0.0 → 1.0.1) |
| `feat:` | minor (1.0.0 → 1.1.0) |
| `feat!:` 또는 `BREAKING CHANGE:` | major (1.0.0 → 2.0.0) |

버전을 범핑하고 CHANGELOG를 생성하는 명령입니다.

```bash
# 모든 변경된 패키지를 분석해 버전 범핑
melos version

# dry-run으로 미리 확인
melos version --dry-run

# 특정 패키지만
melos version --scope="core,ui_components"
```

melos가 각 패키지의 `CHANGELOG.md`를 자동 업데이트합니다.

```markdown
# core

## 1.3.0 - 2026-11-28

### Features

 - **feat**: 사용자 역할(role) 필드 추가 (#42)
 - **feat**: 프로필 이미지 URL 지원 (#38)

### Bug Fixes

 - **fix**: 직렬화 오류 수정 (#40)
```

pub.dev에 게시할 때는 다음을 실행합니다.

```bash
# 버전 범핑 + 태깅 + 게시
melos version && melos publish
```

태그는 `{패키지명}-v{버전}` 형식으로 생성됩니다. 예: `core-v1.3.0`

## CI/CD에서 변경된 패키지만 테스트하기

가장 강력한 기능 중 하나는 변경된 패키지만 처리하는 증분 빌드입니다. GitHub Actions 예시입니다.

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 전체 히스토리 필요

      - uses: dart-lang/setup-dart@v1
        with:
          sdk: "3.9.0"

      - name: melos 설치
        run: dart pub global activate melos

      - name: 의존성 설치
        run: melos bootstrap

      - name: 변경된 패키지만 분석
        run: |
          melos exec \
            --diff=origin/main \
            -- dart analyze .

      - name: 변경된 패키지만 테스트
        run: |
          melos exec \
            --diff=origin/main \
            --dir-exists=test \
            -- dart test --reporter=github
```

`--diff=origin/main` 플래그 덕분에 수정된 패키지와 그 패키지에 의존하는 패키지만 선택적으로 테스트합니다. 100개의 패키지 중 2개만 변경됐다면, 나머지 98개는 CI에서 건너뜁니다.

## melos exec의 고급 활용

`melos exec`는 여러 패키지에 걸쳐 임의의 명령을 실행합니다.

```bash
# 모든 Flutter 패키지에서 flutter pub upgrade 실행
melos exec --flutter -- flutter pub upgrade

# 동시에 4개씩 테스트 (기본값은 5)
melos exec --concurrency=4 -- dart test

# 하나라도 실패하면 즉시 중단
melos exec --fail-fast -- dart test

# 특정 의존성을 가진 패키지만
melos exec --depends-on=flutter -- flutter build apk

# 환경 변수 주입
melos exec --flutter -- \
  flutter test --dart-define=ENV=ci
```

melos.yaml의 스크립트에 `dependsOn`으로 의존성도 선언할 수 있습니다.

```yaml
scripts:
  generate:
    run: melos exec -- dart run build_runner build
    packageFilters:
      dependsOn: build_runner

  test:
    run: melos exec -- dart test
    description: 모든 패키지 테스트
    dependsOn:
      - generate  # generate 먼저 실행
    packageFilters:
      dirExists: test
```

## 주의사항과 팁

**1. 루트 pubspec.yaml 의존성 관리**

melos 8.x는 pub workspaces를 사용하므로, 공통 도구 의존성은 루트 `pubspec.yaml`의 `dev_dependencies`에 한 번만 선언하고 하위 패키지에서는 `any`를 사용해 버전 충돌을 방지합니다.

```yaml
# 루트 pubspec.yaml
dev_dependencies:
  build_runner: ^2.4.0
  freezed: ^2.5.0
  json_serializable: ^6.8.0
  melos: ^8.10.1
```

**2. IDE 설정**

VS Code 사용자라면 melos VS Code 확장을 설치하면 워크스페이스 스크립트를 사이드바에서 바로 실행할 수 있습니다. IntelliJ/Android Studio는 `melos ide-setup` 명령으로 run configuration을 자동 생성합니다.

**3. pub workspace 없이 사용하는 경우**

Dart SDK < 3.9.0 환경이라면 `melos.yaml`에서 `command.bootstrap.usePubWorkspaces: false`로 비활성화하고, 이전 방식(dependency overrides)을 사용합니다.

**4. 순환 의존성 주의**

패키지 간 순환 의존성이 생기면 pub workspaces가 오류를 발생시킵니다. 도메인 레이어 패키지는 외부 프레임워크 의존성을 최소화하고, 인터페이스만 노출하도록 설계하세요.

**5. 모노레포 전환 시기**

패키지가 2~3개를 넘어서거나, 여러 팀이 같은 비즈니스 로직을 별도로 복사하고 있다면 모노레포 전환을 고려하세요. `melos bootstrap`의 마이그레이션 과정은 비교적 간단합니다.

## 정리

melos는 Flutter 멀티 패키지 프로젝트를 위한 강력한 도구입니다. pub workspaces 기반의 로컬 패키지 연결, Conventional Commits 기반 자동 버전 관리, 그리고 변경된 패키지만 처리하는 증분 CI/CD 파이프라인을 조합하면, 대규모 Flutter 프로젝트에서도 개발 속도를 유지할 수 있습니다.

## 참고 자료

- [melos on pub.dev](https://pub.dev/packages/melos)
- [melos GitHub 저장소](https://github.com/invertase/melos)
