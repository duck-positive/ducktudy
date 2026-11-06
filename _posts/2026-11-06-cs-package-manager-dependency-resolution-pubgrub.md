---
layout: post
title: "패키지 매니저 의존성 해결 알고리즘 심화: PubGrub와 SAT Solving"
date: 2026-11-06
categories: [cs, computer-science]
tags: [package-manager, dependency-resolution, pubgrub, sat-solver, cdcl, npm, cargo, semver]
---

`npm install` 명령 하나가 수백 개의 패키지를 순식간에 설치한다. 그 내부에서는 복잡한 버전 제약 조건을 풀어내는 알고리즘이 동작하고 있다. 이 과정은 본질적으로 NP-완전 문제이며, 초기 패키지 매니저들은 그리디(Greedy) 접근으로 인한 "의존성 지옥(Dependency Hell)"이라는 고질적 문제를 안고 있었다.

## 의존성 해결이란 무엇인가

**의존성 해결(Dependency Resolution)**은 프로젝트가 요구하는 패키지들과 그 패키지들이 각자 요구하는 패키지들을 모두 만족시키는 버전 집합을 찾는 과정이다.

### 왜 어려운가?

다음과 같은 시나리오를 생각해 보자.

```
app 1.0
  ├── logger >= 1.0, < 2.0
  └── http >= 2.0

logger 1.5
  └── utils >= 0.5

http 2.1
  └── utils >= 0.5

http 2.0
  └── utils >= 0.3
```

여기서 `app`이 `http >= 2.0`을 요구하고, `logger 1.5`와 `http 2.1` 모두 `utils >= 0.5`를 요구한다면 `utils 0.5+`를 설치하면 된다. 그러나 패키지 수가 수백~수천 개로 늘어나면, 버전 선택 순서에 따라 나중에 충돌이 발생할 수 있다. 이를 **다이아몬드 의존성(Diamond Dependency)** 문제라 한다.

버전 선택 문제는 **Boolean Satisfiability Problem(SAT)** 으로 환원할 수 있으며, 최악의 경우 지수 시간이 걸린다. 실용적인 해결을 위해 현대 패키지 매니저들은 SAT 솔버의 아이디어를 차용한다.

## 의존성 지옥의 역사

### npm v2의 중첩 설치 방식

npm v2는 충돌을 단순히 각 패키지 디렉터리 내에 필요한 버전을 중첩 설치하는 방식으로 "해결"했다.

```
node_modules/
  app/
    node_modules/
      logger@1.5/
        node_modules/
          utils@0.6/
      http@2.1/
        node_modules/
          utils@0.6/   ← 중복 설치!
```

이 방식은 디스크 낭비와 번들 크기 폭증을 초래했다. npm v3은 가능하면 최상위로 호이스팅(hoisting)하는 방식을 도입했으나, 이로 인해 `phantom dependencies`(명시적으로 의존성에 추가하지 않았지만 사용 가능한 패키지) 문제가 생겼다.

### PubGrub의 등장

2018년 Natalie Weizenbaum이 Dart 패키지 매니저인 `pub`을 위해 **PubGrub** 알고리즘을 발표했다. PubGrub는 CDCL(Conflict-Driven Clause Learning) SAT 솔버의 아이디어를 패키지 의존성 해결에 적용한다. 현재 Cargo(Rust), uv(Python)에서도 채택되었다.

## PubGrub 알고리즘 심화

### 핵심 개념

**Incompatibility(비호환성)**은 PubGrub의 핵심 추상이다. "X 패키지의 버전 집합 V1과 Y 패키지의 버전 집합 V2가 동시에 성립하면 불가능하다"는 제약을 표현한다.

```
Incompatibility 예시:
  - { app: 1.0.0, logger: not >=1.0.0 }
    → app 1.0.0이 선택됐을 때, logger가 1.0.0 미만이면 불가
  - { logger: >=2.0.0 }
    → logger 2.0+ 버전은 존재하지 않음 (저장소에 없는 경우)
```

**Partial Solution**은 현재까지 결정된 패키지 버전들의 집합이다. 각 결정(decision)은 "이 버전으로 확정"이고, 각 도출(derivation)은 "이 버전 범위는 불가"다.

### 알고리즘의 4단계

```
1. Unit Propagation
   현재 partial solution에서 각 incompatibility를 검토한다.
   모든 term이 거짓이거나 하나만 미결정인 경우, 그 term은
   반드시 거짓이어야 한다 → derivation 추가

2. Conflict Resolution
   Unit propagation 중 충돌 발생 시, incompatibility들을
   역추적하여 근본 원인(root cause)을 나타내는 새로운
   incompatibility를 생성 → backjump 후 계속

3. Decision Making
   아직 선택되지 않은 패키지 중 하나를 고르고,
   가능한 버전 범위에서 최선의 버전을 선택

4. 반복
   모든 패키지가 결정되면 성공,
   근본 incompatibility 도달 시 실패 + 사람이 읽을 수 있는 오류 메시지 출력
```

## 실제 구현 예제

### 예제 1: 그리디 의존성 해결기 (문제 시연)

```python
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Tuple
from packaging.version import Version
from packaging.specifiers import SpecifierSet

@dataclass
class PackageVersion:
    name: str
    version: str
    deps: Dict[str, str] = field(default_factory=dict)

    def version_obj(self) -> Version:
        return Version(self.version)

# 가상 패키지 저장소 (최신 버전이 먼저)
REGISTRY: Dict[str, List[PackageVersion]] = {
    "app": [
        PackageVersion("app", "1.0.0", {
            "logger": ">=1.0.0,<2.0.0",
            "http-client": ">=2.0.0"
        }),
    ],
    "logger": [
        PackageVersion("logger", "2.0.0", {"utils": ">=1.0.0"}),  # 호환 안됨
        PackageVersion("logger", "1.5.0", {"utils": ">=0.5.0"}),
        PackageVersion("logger", "1.0.0", {"utils": ">=0.4.0"}),
    ],
    "http-client": [
        PackageVersion("http-client", "2.2.0", {"utils": ">=0.5.0"}),
        PackageVersion("http-client", "2.1.0", {"utils": ">=0.5.0"}),
        PackageVersion("http-client", "2.0.0", {"utils": ">=0.4.0"}),
    ],
    "utils": [
        PackageVersion("utils", "0.6.0", {}),
        PackageVersion("utils", "0.5.0", {}),
        PackageVersion("utils", "0.4.0", {}),
    ],
}

class GreedyResolver:
    """그리디 해결기: npm v3 방식의 단순화된 버전"""

    def resolve(self, root: str) -> Dict[str, str]:
        resolved: Dict[str, str] = {}
        queue: List[Tuple[str, str]] = [(root, ">=0.0.0")]
        visited = set()

        while queue:
            pkg_name, constraint_str = queue.pop(0)
            spec = SpecifierSet(constraint_str)

            if pkg_name in resolved:
                # 이미 해결됨 → 버전 충돌 검사
                if resolved[pkg_name] not in spec:
                    raise RuntimeError(
                        f"[충돌] {pkg_name}: "
                        f"이미 {resolved[pkg_name]}이 선택됐지만 "
                        f"{constraint_str}을 만족하지 않음"
                    )
                continue

            # 최신 호환 버전 선택
            candidates = REGISTRY.get(pkg_name, [])
            selected: Optional[PackageVersion] = None
            for candidate in candidates:
                if candidate.version in spec:
                    selected = candidate
                    break

            if not selected:
                raise RuntimeError(
                    f"[해결 불가] {pkg_name} {constraint_str}를 "
                    f"만족하는 버전이 없음"
                )

            resolved[pkg_name] = selected.version
            # 하위 의존성 추가
            for dep_name, dep_constraint in selected.deps.items():
                if (dep_name, dep_constraint) not in visited:
                    queue.append((dep_name, dep_constraint))
                    visited.add((dep_name, dep_constraint))

        return resolved

# 실행
resolver = GreedyResolver()
try:
    result = resolver.resolve("app")
    print("의존성 해결 성공:")
    for pkg, ver in sorted(result.items()):
        print(f"  {pkg} = {ver}")
except RuntimeError as e:
    print(f"실패: {e}")

# 출력:
# 의존성 해결 성공:
#   app = 1.0.0
#   http-client = 2.2.0
#   logger = 1.5.0
#   utils = 0.6.0
```

### 예제 2: PubGrub 핵심 — CDCL 충돌 해결 구현

```python
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Tuple, Set
from enum import Enum, auto

class DecisionType(Enum):
    DECISION   = auto()   # 명시적 버전 선택
    DERIVATION = auto()   # incompatibility에서 도출

@dataclass
class Assignment:
    """Partial Solution의 단위: 패키지에 대한 결정/도출"""
    package: str
    version: Optional[str]      # 확정 버전 (DECISION)
    excluded: Set[str]          # 제외된 버전들 (DERIVATION)
    decision_type: DecisionType
    cause: Optional['Incompatibility'] = None  # 도출의 원인

@dataclass
class Incompatibility:
    """동시에 성립할 수 없는 조건들의 집합"""
    # {패키지명: (포함 버전 집합, 제외 버전 집합)}
    terms: Dict[str, Tuple[Set[str], bool]]
    # True = 이 버전들 중 하나여야 함, False = 이 버전들이 아니어야 함
    cause: str = ""

    def __repr__(self) -> str:
        parts = []
        for pkg, (versions, positive) in self.terms.items():
            prefix = "" if positive else "not "
            parts.append(f"{prefix}{pkg}∈{versions}")
        return f"Incompatible({' ∧ '.join(parts)}) [{self.cause}]"

class PubGrubSolver:
    """
    PubGrub 알고리즘의 핵심 구조 구현
    (실제 pubgrub-rs 구현의 단순화된 버전)
    """

    def __init__(self, registry: Dict[str, List[str]]):
        """
        registry: {패키지명: [버전 목록 (최신순)]}
        """
        self.registry = registry
        self.incompatibilities: List[Incompatibility] = []
        self.partial_solution: List[Assignment] = []
        self.resolution_log: List[str] = []

    def add_incompatibility(self, incompat: Incompatibility):
        self.incompatibilities.append(incompat)
        self.resolution_log.append(f"  추가 incompatibility: {incompat}")

    def _get_decided(self) -> Dict[str, str]:
        return {
            a.package: a.version
            for a in self.partial_solution
            if a.decision_type == DecisionType.DECISION and a.version
        }

    def _get_excluded(self) -> Dict[str, Set[str]]:
        excluded: Dict[str, Set[str]] = {}
        for a in self.partial_solution:
            if a.decision_type == DecisionType.DERIVATION:
                excluded.setdefault(a.package, set()).update(a.excluded)
        return excluded

    def unit_propagate(self) -> Optional[Incompatibility]:
        """
        Unit propagation: incompatibility를 검토하여 도출(derivation) 생성
        Returns: 충돌 incompatibility (없으면 None)
        """
        changed = True
        while changed:
            changed = False
            decided = self._get_decided()
            excluded = self._get_excluded()

            for incompat in self.incompatibilities:
                unsatisfied_count = 0
                unsatisfied_term = None

                for pkg, (versions, positive) in incompat.terms.items():
                    if pkg in decided:
                        # 이미 결정된 패키지
                        if positive and decided[pkg] not in versions:
                            # incompat의 조건을 만족하지 않음 → 이 incompat 무관
                            break
                        elif not positive and decided[pkg] in versions:
                            break
                    else:
                        unsatisfied_count += 1
                        unsatisfied_term = (pkg, versions, positive)
                else:
                    # 모든 term이 partial solution과 일치
                    if unsatisfied_count == 0:
                        # 충돌!
                        self.resolution_log.append(f"  충돌 감지: {incompat}")
                        return incompat
                    elif unsatisfied_count == 1:
                        # Unit: 하나의 term만 미결정 → 도출
                        pkg, versions, positive = unsatisfied_term
                        if positive:
                            # 이 버전들 중 하나여야 하는데 → 나머지는 제외
                            all_vers = set(self.registry.get(pkg, []))
                            excluded_vers = all_vers - versions
                        else:
                            # 이 버전들이 아니어야 함
                            excluded_vers = versions

                        derivation = Assignment(
                            package=pkg,
                            version=None,
                            excluded=excluded_vers,
                            decision_type=DecisionType.DERIVATION,
                            cause=incompat
                        )
                        self.partial_solution.append(derivation)
                        self.resolution_log.append(
                            f"  도출: {pkg}는 {excluded_vers}를 제외"
                        )
                        changed = True

        return None  # 충돌 없음

    def make_decision(self) -> bool:
        """패키지 하나를 선택하고 최선의 버전으로 확정"""
        decided = self._get_decided()
        excluded = self._get_excluded()

        # 아직 결정되지 않은 패키지 선택
        # (실제 구현에서는 의존성 그래프 분석으로 순서 최적화)
        for pkg, versions in self.registry.items():
            if pkg in decided:
                continue
            # 제외된 버전을 뺀 가능한 버전 선택
            pkg_excluded = excluded.get(pkg, set())
            for ver in versions:
                if ver not in pkg_excluded:
                    decision = Assignment(
                        package=pkg,
                        version=ver,
                        excluded=set(),
                        decision_type=DecisionType.DECISION
                    )
                    self.partial_solution.append(decision)
                    self.resolution_log.append(f"  결정: {pkg} = {ver}")
                    return True
            # 가능한 버전이 없음 → 충돌
            return False

        return False  # 모든 패키지 결정됨

    def solve(self) -> Optional[Dict[str, str]]:
        """의존성 해결 실행"""
        print("PubGrub 의존성 해결 시작")

        max_iterations = 100
        for i in range(max_iterations):
            # 1. Unit Propagation
            conflict = self.unit_propagate()
            if conflict:
                # 2. Conflict Resolution (백점프)
                print(f"\n충돌 발생 (iteration {i+1}):")
                print(f"  {conflict}")
                # 실제 구현에서는 derivation graph 역추적으로 근본 원인 도출
                # 여기서는 단순히 실패 처리
                return None

            # 3. Decision Making
            if not self.make_decision():
                break  # 모든 패키지 결정 완료

        decided = self._get_decided()
        print(f"\n해결 완료 ({len(decided)}개 패키지):")
        for pkg, ver in sorted(decided.items()):
            print(f"  {pkg} = {ver}")

        print("\n해결 과정 로그:")
        for log in self.resolution_log:
            print(log)

        return decided

# 실행 예시
registry = {
    "app":         ["1.0.0"],
    "logger":      ["1.5.0", "1.0.0"],
    "http-client": ["2.1.0", "2.0.0"],
    "utils":       ["0.6.0", "0.5.0", "0.4.0"],
}

solver = PubGrubSolver(registry)

# app 1.0.0 선택 강제
solver.add_incompatibility(Incompatibility(
    {"app": ({"1.0.0"}, False)},  # app은 1.0.0이 아니면 불가
    "root dependency"
))

# app 1.0.0은 logger >= 1.0.0 필요
solver.add_incompatibility(Incompatibility(
    {
        "app":    ({"1.0.0"}, True),
        "logger": ({"1.5.0", "1.0.0"}, False),  # logger가 이 중 하나가 아니면
    },
    "app@1.0.0 requires logger >=1.0.0"
))

result = solver.solve()
```

## 실제 패키지 매니저의 차이

### npm / pnpm

npm은 자체 SAT 솔버 없이 호이스팅 기반 그리디 해결을 사용한다. pnpm은 패키지를 중앙 저장소에서 심볼릭 링크로 연결하여 중복 설치를 제거하되, 버전 충돌 시에는 각 패키지별로 격리된 `node_modules`를 유지한다.

### Cargo (Rust)

Cargo는 `pubgrub` 크레이트를 사용한 PubGrub 기반 해결기를 채택 중이다. `Cargo.lock` 파일이 존재하면 정확히 그 버전을 사용하고, 없으면 허용 범위 내 최신 버전을 선택한다.

### Poetry / uv (Python)

Poetry는 PubGrub를 사용한다. uv는 Rust로 구현된 초고속 Python 패키지 설치 도구로, `pubgrub-rs`를 사용하여 기존 pip 대비 10~100배 빠른 의존성 해결을 보여준다.

## 주의사항과 실전 팁

### 1. Lock 파일은 반드시 커밋하라

재현 가능한 빌드(Reproducible Build)를 위해 `Cargo.lock`, `package-lock.json`, `poetry.lock`은 반드시 버전 관리에 포함해야 한다. CI/CD에서 `--frozen` 또는 `--locked` 옵션을 사용하여 lock 파일과 다른 버전이 설치되는 것을 방지한다.

```bash
# Cargo: lock 파일과 완전히 일치하지 않으면 실패
cargo build --locked

# npm: package-lock.json을 절대 변경하지 않음
npm ci

# poetry: lock 파일 기반 설치, 변경 불허
poetry install --frozen
```

### 2. 의존성 범위를 정확히 명세하라

```toml
# Cargo.toml: 너무 느슨한 범위
some-crate = "*"           # 위험! 메이저 버전 업그레이드로 호환성 깨짐

# 권장: 메이저 버전 고정, 패치는 허용
some-crate = "^1.2.3"     # >= 1.2.3, < 2.0.0
some-crate = "~1.2.3"     # >= 1.2.3, < 1.3.0 (보수적)
```

### 3. 충돌 오류 메시지 읽는 법

PubGrub의 최대 강점은 충돌의 근본 원인을 설명하는 오류 메시지다.

```
error: package A 1.0 requires package B ^2.0
       package C 3.0 requires package B ^1.0
       Because you require A and C, B cannot be satisfied.
```

이 메시지는 `A`와 `C`가 `B`의 서로 다른 메이저 버전을 요구하기 때문에 충돌함을 명확히 알려준다.

### 4. Monorepo에서의 의존성 관리

대규모 Monorepo에서는 패키지 간 내부 의존성이 많아져 해결 복잡도가 급증한다. `pnpm workspaces`, `Cargo workspace`, `Bazel`의 `rules_pkg`는 이를 위한 도구들이다.

의존성 해결은 소프트웨어 엔지니어링에서 가장 과소평가된 문제 중 하나다. PubGrub의 등장으로 "의존성 지옥"은 과거의 이야기가 되어 가고 있지만, 여전히 패키지 생태계에서 호환성 계약을 지키는 것은 개발자의 책임이다.

## 참고 자료
- [PubGrub Rust 구현체 (pubgrub-rs)](https://github.com/pubgrub-rs/pubgrub)
- [Dart Pub 솔버 알고리즘 문서](https://github.com/dart-lang/pub/blob/master/doc/solver.md)
