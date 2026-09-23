---
layout: post
title: "Bazel 빌드 시스템 완전 정복: 선언적·재현 가능한 빌드로 대규모 모노레포를 정복하는 법"
date: 2026-09-23
categories: [cs, computer-science]
tags: [bazel, build-system, monorepo, starlark, hermetic-build, remote-execution, incremental-build, ci-cd]
---

구글은 수십억 줄의 코드를 단일 저장소(Monorepo)에서 관리하며 하루에도 수만 번 빌드를 실행합니다. 이것을 가능하게 하는 비밀이 바로 **Bazel**입니다. 이 글에서는 Bazel의 핵심 설계 철학과 내부 작동 원리를 깊이 파헤칩니다.

## 개념 설명: Bazel이란?

**Bazel**은 Google이 개발하여 2015년 오픈소스로 공개한 빌드·테스트 시스템입니다. Google 내부의 **Blaze**를 외부 공개용으로 포팅한 것이며, 대규모 멀티 언어 모노레포(Monorepo)를 위해 설계되었습니다. Facebook의 Buck, Twitter의 Pants도 같은 계열의 도구입니다.

기존 빌드 도구와의 결정적 차이는 **세 가지 철학**에 있습니다:

**1. 정확성(Correctness) — 재현 가능한 빌드**

같은 소스 코드 + 같은 의존성 = 반드시 같은 결과물. 이것을 **Hermetic Build(격리된 빌드)**라고 합니다. Makefile은 `$(CC)`가 로컬 환경에 따라 달라지는 문제가 있지만, Bazel은 컴파일러 자체를 의존성으로 선언하여 환경 차이를 원천 차단합니다.

**2. 속도(Speed) — 진정한 증분 빌드**

Bazel은 각 빌드 액션(Action)의 입력 파일, 환경 변수, 명령어를 해시로 추적합니다. 변경이 없으면 캐시된 결과를 즉시 사용합니다. `src/main.c`를 수정하면 그 파일에만 의존하는 액션만 재실행됩니다.

**3. 확장성(Scalability) — 분산 빌드**

로컬 캐시를 넘어 **Remote Cache**(빌드 결과를 팀 전체가 공유)와 **Remote Execution**(빌드 액션을 클라우드 서버에서 병렬 실행)을 지원합니다. CI가 빌드한 결과를 개발자의 로컬에서 재사용할 수 있습니다.

### 핵심 개념 정리

| 개념 | 설명 |
|------|------|
| **WORKSPACE / MODULE.bazel** | 저장소 루트에 위치. 외부 의존성 정의 |
| **BUILD / BUILD.bazel** | 패키지(디렉토리)별 빌드 규칙 정의 파일 |
| **Rule** | 빌드 작업 유형. `cc_binary`, `java_library`, `py_test` 등 |
| **Target** | 빌드 가능한 단위. `//path/to/package:target_name` 형식의 레이블로 참조 |
| **Action** | 실제로 실행되는 명령어 (컴파일, 링크, 코드 생성 등) |
| **Artifact** | 빌드 액션의 입력/출력 파일 |
| **Sandbox** | 각 액션이 실행되는 격리된 환경 (허가된 파일만 접근 가능) |

### Bazel의 빌드 3단계

**1. 로딩(Loading)**: WORKSPACE와 BUILD 파일을 읽어 의존성 그래프를 구성합니다. 각 패키지의 타겟과 그 의존 관계를 파악합니다.

**2. 분석(Analysis)**: 각 타겟 Rule의 구현 함수(Starlark)를 실행하여 어떤 Action을 실행해야 하는지 결정합니다. 이 단계의 결과가 **Action Graph**입니다.

**3. 실행(Execution)**: Action Graph를 위상 정렬하여 의존 관계 없는 액션들을 병렬 실행합니다. 캐시를 확인하여 이미 결과가 있는 액션은 건너뜁니다.

## 왜 필요한가

현대 소프트웨어 프로젝트가 커지면 기존 빌드 도구의 한계가 드러납니다:

- **Maven/Gradle**: JVM 생태계에 특화. 멀티 언어, 분산 빌드 지원 미흡.
- **Make**: 파일 타임스탬프 기반의 변경 감지로 부정확한 증분 빌드.
- **npm scripts**: 의존성 격리 없음, 환경에 따라 빌드 결과 달라짐.

특히 **모노레포** 환경에서 10만 개의 소스 파일 중 하나만 바꿨을 때 전체를 다시 빌드하는 것은 재앙입니다. Bazel은 정확한 의존성 추적으로 **변경된 파일에만 영향받는 최소한의 작업만** 실행합니다.

## 실제 구현 예제

### 예제 1: Python 모노레포 프로젝트 BUILD 파일 작성

```
# 디렉토리 구조
myapp/
├── MODULE.bazel          # Bzlmod 의존성 관리 (Bazel 6+ 권장 방식)
├── .bazelrc              # 팀 공통 Bazel 설정
├── common/
│   ├── BUILD.bazel
│   ├── config.py
│   └── utils.py
├── services/
│   ├── payment/
│   │   ├── BUILD.bazel
│   │   ├── server.py
│   │   └── handler.py
│   └── notification/
│       ├── BUILD.bazel
│       └── sender.py
└── tools/
    └── BUILD.bazel
```

```python
# MODULE.bazel — 외부 의존성을 중앙 관리 (pip, Go 모듈 등)
module(
    name = "myapp",
    version = "1.0.0",
)

# rules_python: Python 빌드 지원
bazel_dep(name = "rules_python", version = "0.31.0")

# Python 툴체인 설정 (Bazel이 지정된 버전의 Python을 관리)
python = use_extension("@rules_python//python/extensions:python.bzl", "python")
python.toolchain(python_version = "3.11")

# pip 의존성 관리 (requirements.txt 기반)
pip = use_extension("@rules_python//python/extensions:pip.bzl", "pip")
pip.parse(
    hub_name = "pip",
    python_version = "3.11",
    requirements_lock = "//:requirements.lock.txt",
)
use_repo(pip, "pip")
```

```python
# common/BUILD.bazel — 공유 라이브러리 정의
load("@rules_python//python:defs.bzl", "py_library", "py_test")

py_library(
    name = "utils",
    srcs = ["utils.py"],
    # visibility: 이 라이브러리를 다른 패키지에서 사용할 수 있는 범위 제어
    # public으로 열면 저장소 전체에서 참조 가능
    visibility = ["//visibility:public"],
)

py_library(
    name = "config",
    srcs = ["config.py"],
    deps = [
        ":utils",           # 같은 패키지의 타겟
        "@pip//pydantic",   # 외부 pip 패키지
    ],
    visibility = ["//services/..."],  # services/ 이하에서만 접근 허용
)

py_test(
    name = "utils_test",
    srcs = ["utils_test.py"],
    deps = [":utils"],
    # size: 테스트 실행 시간 SLO (small < 1분, medium < 5분, large < 15분)
    size = "small",
)
```

```python
# services/payment/BUILD.bazel — 결제 서비스 바이너리
load("@rules_python//python:defs.bzl", "py_binary", "py_library", "py_test")

py_library(
    name = "handler_lib",
    srcs = ["handler.py"],
    deps = [
        "//common:config",   # 절대 레이블: 저장소 루트 기준 경로
        "//common:utils",
        "@pip//stripe",
        "@pip//grpcio",
    ],
)

py_binary(
    name = "server",
    srcs = ["server.py"],
    deps = [":handler_lib"],
    # Python 바이너리는 .runfiles/ 구조로 모든 의존성을 포함
)

py_test(
    name = "handler_test",
    srcs = ["handler_test.py"],
    deps = [
        ":handler_lib",
        "@pip//pytest",
    ],
    env = {
        "STRIPE_TEST_KEY": "sk_test_placeholder",
    },
)
```

```bash
# 주요 Bazel 명령어

# 특정 타겟 빌드
bazel build //services/payment:server

# 저장소 전체 빌드
bazel build //...

# 변경된 파일에 영향받는 타겟만 테스트
# (CI에서 PR의 변경 사항만 테스트할 때 유용)
CHANGED=$(git diff --name-only origin/main)
AFFECTED=$(bazel query "rdeps(//..., set($CHANGED))" 2>/dev/null)
bazel test $AFFECTED

# 의존성 쿼리
bazel query "deps(//services/payment:server)"       # 모든 의존 타겟
bazel query "rdeps(//..., //common:utils)"          # utils를 참조하는 모든 타겟
bazel query "kind(py_test, //...)"                  # 모든 py_test 타겟

# 원격 캐시 활용 (팀 전체 빌드 결과 공유)
bazel build //... \
  --remote_cache=grpc://buildcache.example.internal:9090 \
  --remote_upload_local_results=true

# 원격 실행 (빌드 클러스터에서 병렬 빌드)
bazel build //... \
  --remote_executor=grpc://buildexec.example.internal:8980 \
  --jobs=200
```

### 예제 2: 커스텀 Starlark 규칙 — 프로토콜 버퍼 코드 생성기

Bazel의 가장 강력한 기능은 Starlark(Python 방언)로 커스텀 빌드 규칙을 작성할 수 있다는 점입니다.

```python
# tools/proto_rules.bzl — 커스텀 빌드 규칙 정의

# ── Rule Implementation ───────────────────────────────────────────────────────

def _py_proto_impl(ctx):
    """
    .proto 파일에서 Python gRPC 코드를 생성하는 커스텀 규칙.
    Bazel의 Action API를 사용하여 hermetic하게 실행합니다.
    """
    proto_src = ctx.file.src

    # 출력 파일 선언: Bazel이 경로와 생명주기를 완전히 관리
    out_pb2      = ctx.actions.declare_file(
        proto_src.basename.replace(".proto", "_pb2.py")
    )
    out_pb2_grpc = ctx.actions.declare_file(
        proto_src.basename.replace(".proto", "_pb2_grpc.py")
    )

    # Action 정의: 실제 실행될 명령어
    # - inputs: 이 액션이 읽는 파일 목록 (변경 감지에 사용)
    # - outputs: 이 액션이 생성하는 파일 목록
    # - executable: 실행할 바이너리 (Bazel 의존성으로 관리)
    ctx.actions.run(
        inputs     = depset(
            [proto_src],
            transitive = [d[ProtoInfo].transitive_sources for d in ctx.attr.deps],
        ),
        outputs    = [out_pb2, out_pb2_grpc],
        executable = ctx.executable._protoc,
        arguments  = [
            "--python_out="      + out_pb2.dirname,
            "--grpc_python_out=" + out_pb2_grpc.dirname,
            "-I" + proto_src.dirname,
            proto_src.path,
        ],
        tools      = [ctx.executable._grpc_python_plugin],
        mnemonic   = "PyProtoGen",
        progress_message = "Generating Python proto stubs for %s" % proto_src.basename,
        # execution_requirements: 샌드박스 설정
        execution_requirements = {
            "no-remote": "",        # 로컬에서만 실행 (네트워크 필요 시 제거)
            "no-sandbox": "",       # 필요한 경우만 사용 (재현성 낮아짐)
        } if False else {},         # 기본: 원격 실행 + 샌드박스 활성화
    )

    # Provider 반환: 다운스트림 규칙에 전달할 정보
    return [
        DefaultInfo(files = depset([out_pb2, out_pb2_grpc])),
        PyInfo(
            transitive_sources = depset(
                [out_pb2, out_pb2_grpc],
                transitive = [
                    d[PyInfo].transitive_sources
                    for d in ctx.attr.deps
                    if PyInfo in d
                ],
            ),
            uses_shared_libraries = False,
            has_py2_only_sources  = False,
            has_py3_only_sources  = True,
        ),
    ]

# ── Rule 선언 ────────────────────────────────────────────────────────────────

py_proto_library = rule(
    implementation = _py_proto_impl,
    attrs = {
        "src": attr.label(
            allow_single_file = [".proto"],
            mandatory         = True,
            doc               = "Proto 소스 파일",
        ),
        "deps": attr.label_list(
            providers = [ProtoInfo],
            doc       = "의존하는 proto 라이브러리",
        ),
        # _ 접두사: private 속성 (BUILD에서 override 불가)
        "_protoc": attr.label(
            default    = "@com_google_protobuf//:protoc",
            executable = True,
            cfg        = "exec",  # 빌드 도구는 exec(빌드 머신) configuration에서 실행
        ),
        "_grpc_python_plugin": attr.label(
            default    = "@grpc//tools:grpc_python_plugin",
            executable = True,
            cfg        = "exec",
        ),
    },
    doc = ".proto 파일에서 Python gRPC stub을 생성합니다.",
)

# ── Macro: 규칙 조합으로 편의성 제공 ────────────────────────────────────────

def py_grpc_library(name, srcs, deps = [], **kwargs):
    """
    여러 proto 파일에서 Python gRPC 라이브러리를 생성하는 매크로.
    내부적으로 py_proto_library + py_library를 조합합니다.
    """
    gen_targets = []
    for src in srcs:
        gen_name = name + "_gen_" + src.replace(".proto", "").replace("/", "_")
        py_proto_library(
            name = gen_name,
            src  = src,
            deps = deps,
        )
        gen_targets.append(":" + gen_name)

    native.py_library(
        name = name,
        srcs = gen_targets,
        **kwargs
    )
```

```bash
# .bazelrc — 팀 공통 설정 파일 (저장소에 커밋)

# 공통 설정
common --enable_bzlmod                      # Bzlmod 의존성 관리 활성화
build  --incompatible_strict_action_env     # 환경변수 격리 강화

# 로컬 개발 최적화
build:local --disk_cache=~/.cache/bazel-disk
build:local --jobs=HOST_CPUS * 0.75        # CPU 75% 활용

# CI 환경 설정
build:ci --remote_cache=grpc://cache.internal:9090
build:ci --remote_upload_local_results=true
build:ci --remote_executor=grpc://exec.internal:8980
build:ci --jobs=100

# 빌드 이벤트 스트리밍 (BuildBuddy 등 모니터링 도구와 연동)
build:ci --bes_backend=grpc://bes.internal:1985
build:ci --bes_results_url=https://bes.internal/invocation/

# 테스트 설정
test --test_output=errors                   # 실패한 테스트 출력만 표시
test --test_timeout=60                      # 기본 타임아웃 60초

# 특정 컴파일러 경고를 에러로 처리
build --per_file_copt=//...:@-Werror

# .bazelrc.local (gitignore에 추가, 개발자 개인 설정)
# try-import %workspace%/.bazelrc.local
```

## 주의사항 및 팁

**1. Hermeticity를 절대로 깨뜨리지 마라**

빌드 규칙에서 `os.environ`, `/usr/bin/curl` 같은 시스템 의존을 직접 사용하면 재현성이 깨집니다. 모든 도구는 Bazel 의존성으로 선언하세요. `ctx.actions.run_shell`에서 `PATH`를 직접 사용하는 것도 위험합니다.

**2. `visibility`로 아키텍처 경계를 강제하라**

`//visibility:public`을 남용하면 의도치 않은 의존 관계가 생깁니다. `//services/payment:__pkg__`처럼 세밀하게 제어하여 레이어 간 경계를 코드 레벨에서 강제하세요. `bazel query "rdeps"` 커맨드로 의존성 역추적을 확인하세요.

**3. Gazelle로 BUILD 파일 자동 생성하라**

Go, Python 프로젝트에서 BUILD 파일을 수동으로 유지하는 것은 고통스럽습니다. [Gazelle](https://github.com/bazelbuild/bazel-gazelle)을 사용하면 소스 파일을 분석하여 BUILD 파일을 자동 생성·업데이트합니다.

```bash
gazelle update //...        # BUILD 파일 자동 업데이트
gazelle update-repos -from_file=go.mod  # go.mod에서 외부 의존성 동기화
```

**4. Remote Cache 히트율을 모니터링하라**

Remote Cache를 도입했다면 `--profile=bazel_profile.json`으로 프로파일링 데이터를 수집하고, BuildBuddy나 EngFlow 같은 도구로 캐시 히트율을 모니터링하세요. 히트율이 낮다면 환경 변수 누수(non-hermetic) 문제일 수 있습니다.

**5. `query`, `cquery`, `aquery`를 활용하라**

- `bazel query`: 로딩 단계 결과. 타겟 그래프 탐색.
- `bazel cquery`: 분석 단계 결과. 설정(configuration)을 고려한 의존성.
- `bazel aquery`: 액션 그래프 탐색. 실제 실행될 명령어 확인.

## 참고 자료

- [Bazel 공식 GitHub 저장소](https://github.com/bazelbuild/bazel)
- [Bazel 공식 사이트 — 개념 문서](https://bazel.build/concepts)
- [rules_python — Python 빌드 규칙](https://github.com/bazelbuild/rules_python)
- [Gazelle — BUILD 파일 자동화](https://github.com/bazelbuild/bazel-gazelle)
