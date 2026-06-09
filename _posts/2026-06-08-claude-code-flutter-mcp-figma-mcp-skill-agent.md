---
layout: post
title: "Claude Code에서 Dart MCP + Figma MCP 연동하기 — Skill과 Agent 활용 구조"
date: 2026-06-08
categories: [ai, flutter]
tags: [claude, mcp, flutter, figma, dart, fvm, skill, agent, workflow]
---

## 왜 MCP인가

Claude Code는 MCP(Model Context Protocol) 서버를 통해 외부 도구와 직접 연결할 수 있습니다. Figma 디자인을 읽고, Flutter 앱에 즉시 hot reload 하는 워크플로우를 MCP 없이 구현하려면 스크린샷을 직접 캡처하고, 코드를 작성하고, 터미널에서 `r`을 눌러야 합니다. MCP를 쓰면 이 모든 단계를 Claude가 자동으로 처리합니다.

---

## 1. Dart MCP 연결 — FVM 환경의 함정

가장 먼저 만나는 문제가 **`-32000` 에러**입니다.

```
Failed to reconnect to dart: -32000
```

원인은 단순합니다. Claude Code의 MCP 설정에서 `dart mcp-server` 명령을 실행할 때 시스템 전역 `dart` 바이너리를 사용하는데, FVM으로 관리되는 프로젝트의 경우 전역 Dart 버전이 낮아 `mcp-server` 서브커맨드 자체가 없습니다.

`dart mcp-server`는 **Dart 3.8+ (Flutter 3.44.1)** 부터 지원됩니다.

### 해결 방법

전역 `dart` 대신 FVM 버전의 절대 경로를 직접 지정합니다.

```bash
# 기존 설정 제거
claude mcp remove "dart" -s local

# FVM 버전 경로로 다시 추가
claude mcp add dart -s local -- \
  /Users/사용자명/fvm/versions/3.44.1/bin/dart mcp-server
```

설정 후 `~/.claude.json`에서 확인:

```json
{
  "mcpServers": {
    "dart": {
      "type": "stdio",
      "command": "/Users/사용자명/fvm/versions/3.44.1/bin/dart",
      "args": ["mcp-server"]
    }
  }
}
```

연결되면 `hot_reload`, `analyze_files`, `widget_inspector` 등의 도구가 활성화됩니다.

---

## 2. Figma MCP 활용

Figma MCP는 설치 후 두 가지 도구를 핵심으로 사용합니다.

| 도구 | 역할 |
|------|------|
| `get_design_context` | 레이아웃, 색상, 타이포그래피, 컴포넌트 구조를 JSON으로 추출 |

URL에서 `node-id`를 파싱할 때 주의할 점이 있습니다. URL의 `-`를 `:`로 변환해야 합니다.

```
figma.com/design/f63mhUKaUED9sjh5Z0DuUC/예시?node-id=38-10295
→ fileKey: f63mhUKaUED9sjh5Z0DuUC
→ nodeId:  38:10295  ← 하이픈을 콜론으로
```

---

## 3. Skill vs Agent — 어떤 걸 써야 하나

Claude Code에서 반복 워크플로우를 자동화하는 방법은 두 가지입니다.

### Skill (권장 — 단일 화면 구현 시)

Skill은 **메인 컨텍스트**에서 실행됩니다. 즉, 이미 인증된 Figma MCP와 DTD가 연결된 Dart MCP를 그대로 사용할 수 있습니다.

```
/figma-flutter https://www.figma.com/design/...?node-id=38-10295
```

Skill을 호출하면:
1. Figma에서 디자인 분석
2. Flutter 코드 생성 및 파일 저장
3. Dart MCP로 즉시 hot reload

### Agent + Workflow (권장 — 다수 화면 동시 구현 시)

여러 화면을 한번에 구현할 때는 순수 Skill만으로는 두 가지 문제가 생깁니다.

- **Figma API 응답이 너무 커서 메인 컨텍스트를 오염** — 레이아웃 트리가 수천 줄에 달할 수 있음
- **화면을 순차적으로 구현** — 5개 화면이면 5배 시간 소요

해결 구조:

```
Phase 0: 스냅샷 조회
  → .claude/figma_snapshot.json에서 nodeId 조회
  → 없으면 "new" (전체 구현) / 있으면 "diff" (변경 부분만 수정)

Phase 1: Explore 에이전트 (Figma 분석)
  → get_design_context 호출
  → "diff" 모드일 때 이전 스냅샷과 비교해 변경 항목만 식별
  → 구조화된 요약만 반환 (원문은 에이전트 컨텍스트에서 소비)

Phase 2: Workflow pipeline (코드 병렬 생성)
  → "new": 화면 전체 코드 생성
  → "diff": 변경된 항목만 기존 파일에 반영
  → 벽시계 시간 = 가장 느린 화면 하나 기준

Phase 3: 메인 컨텍스트 (hot reload + 스냅샷 갱신)
  → DTD 연결은 세션에 묶임 → 반드시 메인에서 실행
  → hot reload 성공 후 .claude/figma_snapshot.json 갱신
```

---

## 4. /figma-flutter Skill 설정

`.claude/skills/figma-flutter/SKILL.md`를 생성합니다.

```markdown
---
name: figma-flutter
description: Figma 디자인을 Flutter 코드로 구현하고 hot reload
---

## Phase 0: 스냅샷 조회
.claude/figma_snapshot.json에서 nodeId 조회 → mode: "new" / "diff" 결정

## Phase 1: Figma 분석 (Explore 에이전트)
Explore 에이전트를 spawn → get_design_context 호출
"diff"이면 이전 스냅샷 전달 → 변경 항목만 식별
→ { screens: [{ name, nodeId, isNew, changes, layout, colors, textStyles, spacing }] } 반환

## Phase 2: 코드 생성 (Workflow)
pipeline(screens, screen =>
  screen.isNew
    ? agent(전체 구현)
    : agent(변경 항목만 기존 파일에 반영)
)

## Phase 3: hot reload + 스냅샷 갱신 (메인 컨텍스트)
dtd(listDtdUris) → connect → hot_reload(clearRuntimeErrors: true)
.claude/figma_snapshot.json에 nodeId 키로 갱신 저장
```

---

## 5. Dart MCP의 DTD 연결 플로우

Hot reload가 실패하는 경우 대부분 DTD 연결 순서 문제입니다.

```bash
# 1. 사용 가능한 DTD URI 목록
dtd(listDtdUris)

# 2. 프로젝트 경로가 일치하는 URI 선택 후 연결
dtd(connect, uri: "ws://127.0.0.1:61459/xxxx=")

# 3. hot reload
hot_reload(clearRuntimeErrors: true)

# 4. 런타임 에러 확인
get_runtime_errors()
```

앱이 실행 중이지 않으면 DTD URI가 없습니다. 이 경우 `flutter run`으로 앱을 먼저 실행해야 합니다.

---

## 6. 스냅샷 기반 diff — Figma 버전 관리의 한계를 우회하기

처음 기대했던 워크플로우는 "Figma에서 수정 → Claude가 변경점 자동 감지 → hot reload"였습니다. 하지만 Figma MCP는 **현재 상태만 읽을 수 있고 버전 히스토리에는 접근하지 못합니다.** 매 실행마다 전체 화면을 새로 분석하고 코드 전체를 재생성하게 됩니다.

이를 보완하기 위해 Explore 에이전트의 분석 결과를 로컬 JSON 파일(`.claude/figma_snapshot.json`)로 저장하고, 다음 실행 때 비교하는 방식을 도입했습니다.

```json
{
  "2001:16": {
    "layout": "Two-column staggered grid...",
    "colors": "AppColors.gray05 (Color(0xFFE3E5E8))",
    "textStyles": "none",
    "spacing": "horizontal padding 20, top padding 8...",
    "components": "HomeScreen > SingleChildScrollView > Row > 2x Expanded columns, _PostCard h=120",
    "lastSynced": "2026-06-09"
  }
}
```

Phase 0에서 이 파일을 읽어 해당 nodeId의 스냅샷이 있으면 `mode: "diff"`, 없으면 `mode: "new"`로 분기합니다. `diff` 모드에서는 Explore 에이전트에게 이전 스냅샷을 함께 전달해 변경된 항목만 식별하도록 지시합니다.

### 성능 비교 — 스냅샷 있음 vs 없음

동일 디자인(이미 동기화된 상태)으로 두 방식을 병렬 실행해 측정했습니다.

| 지표 | 스냅샷 있음 | 스냅샷 없음 |
|------|:-----------:|:-----------:|
| 총 소요 시간 | **39초** | **104초** |
| 에이전트 수 | 1 | 2 |
| 토큰 사용량 | 25,719 | 25,931 |
| Figma 도구 호출 | 15회 | 8회 |
| 코드 생성 에이전트 | 0개 | 1개 |
| 생성된 코드 라인 | 0줄 | 58줄 |

속도는 2.7배 차이가 났지만 토큰 사용량은 거의 동일했습니다. Figma API를 호출하는 Phase 1 비용은 두 방식 모두 피할 수 없기 때문입니다. 오히려 스냅샷이 있는 쪽이 변경 여부를 꼼꼼히 검증하느라 Figma 도구 호출이 더 많았습니다(15 vs 8회).

속도 차이는 전적으로 Phase 2에서 발생합니다. 변경 없음이 확인되면 코드 생성 에이전트 자체를 생략하기 때문이며, 화면이 많을수록 차이는 선형으로 커집니다.

핵심 가치는 토큰 절약보다 **불필요한 코드 재생성 방지**입니다. 스냅샷 없이 실행하면 이미 최신 상태인 파일을 매번 새로 덮어씁니다. 스냅샷은 이를 막아주면서 응답 속도도 함께 올려줍니다.

---

## 마치며

Flutter + Claude Code 조합에서 MCP는 단순한 편의 기능이 아닙니다. Figma 디자인에서 실제 앱 화면까지의 피드백 루프를 분 단위로 줄여주는 핵심 인프라입니다. FVM 환경이라면 반드시 절대 경로로 Dart MCP를 설정하고, 다수 화면 작업 시엔 Workflow 병렬 구조를 활용하세요. Figma MCP의 버전 히스토리 미지원은 로컬 스냅샷으로 우회할 수 있으며, 변경이 없을 때는 2.7배 빠른 실행 속도를 기대할 수 있습니다.
