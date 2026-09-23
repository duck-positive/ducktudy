---
layout: post
title: "SLI·SLO·SLA와 에러 버짓: Google SRE가 설계한 신뢰성 측정 프레임워크 완전 정복"
date: 2026-09-23
categories: [cs, computer-science]
tags: [sre, sli, slo, sla, error-budget, reliability, monitoring, prometheus, alerting, observability]
---

"우리 서비스는 얼마나 신뢰할 수 있어야 하는가?" 이 질문에 명확하게 답할 수 없는 팀은 언제나 두 가지 함정에 빠집니다. 무한한 신뢰성을 추구하다 혁신을 멈추거나, 아무런 기준 없이 장애를 반복하거나. Google SRE(Site Reliability Engineering)는 이 딜레마를 **SLI, SLO, SLA, 에러 버짓** 프레임워크로 해결합니다.

## 개념 설명

### SLI — 무엇을 측정하는가

**SLI(Service Level Indicator)**는 서비스 동작의 특정 측면을 정량적으로 측정하는 지표입니다. 대부분 **비율(ratio)**로 표현합니다:

```
SLI = 좋은 이벤트 수 / 전체 이벤트 수
```

좋은 SLI를 정의하는 원칙은 **사용자가 실제로 경험하는 것을 측정**하는 것입니다. 서버 CPU 사용률이 50%라도 사용자에게 에러를 반환하면 서비스는 좋지 않습니다.

주요 SLI 유형:
- **가용성(Availability)**: `성공 응답 수 / 전체 요청 수`
- **지연시간(Latency)**: `200ms 이내 응답 수 / 전체 요청 수` (임계값을 넘으면 나쁜 이벤트)
- **처리량(Throughput)**: `초당 처리된 작업 수` (일정 수준 아래면 나쁜 이벤트)
- **정확성(Freshness/Correctness)**: `최신 데이터로 응답한 읽기 수 / 전체 읽기 수` (데이터 파이프라인)
- **포화도(Saturation)**: `큐 길이, 디스크 사용률` — 일반적으로 용량 계획에 사용

### SLO — 어떤 수준을 목표로 하는가

**SLO(Service Level Objective)**는 SLI의 목표값입니다. 팀 내부의 약속이며, "우리 서비스는 이 정도는 되어야 한다"는 기준입니다.

```
가용성 SLO: 가용성 SLI ≥ 99.9% (월간 윈도우 기준)
지연시간 SLO: P99 200ms 이내 요청 비율 ≥ 99%
```

SLO의 수치는 "사용자를 만족시킬 수 있는 최솟값"으로 설정해야 합니다. 99.99%처럼 과도하게 높은 SLO는:
- 연간 52.6분의 다운타임만 허용 → 모든 배포가 위험해짐
- 엔지니어가 신규 기능 개발 대신 안정성에만 집중하게 됨
- 달성해도 사용자가 그 차이를 못 느낌

### SLA — 외부와의 법적 약속

**SLA(Service Level Agreement)**는 SLO를 기반으로 한 고객과의 법적 계약입니다. SLO보다 낮게 설정하여 **여유(Buffer)**를 확보합니다:

| 구분 | 성격 | 위반 결과 |
|------|------|-----------|
| SLO  | 내부 목표 | 팀 내부 정책 실행 (배포 동결 등) |
| SLA  | 외부 계약 | 환불, 크레딧, 계약 해지 |

SLO: 99.9% → SLA: 99.5% 로 설정하면, SLO를 위반하더라도 SLA 위반 전에 복구할 시간적 여유가 생깁니다.

### 에러 버짓 — 혁신과 신뢰성의 균형추

**에러 버짓(Error Budget)**은 SLO의 반대입니다:

```
에러 버짓 = 1 - SLO 목표값

예) SLO = 99.9%  →  에러 버짓 = 0.1%
```

30일 기준으로 99.9% SLO는 **43.2분**의 다운타임을 허용합니다. 이 시간이 바로 에러 버짓입니다.

에러 버짓이 중요한 이유는 **배포 의사결정**에 명확한 기준을 제공하기 때문입니다:
- 에러 버짓이 **충분히 남아 있으면**: 새 기능 배포, 실험, 성능 테스트 진행 가능
- 에러 버짓이 **소진되면**: 모든 신규 배포 중단, 안정화에만 집중

이것은 개발팀과 운영팀이 서로를 탓하는 대신, **공통의 수치로 대화**할 수 있게 해줍니다. "에러 버짓이 75% 남았으니 이번 배포는 진행하자", "에러 버짓이 10% 밖에 안 남았으니 위험한 마이그레이션은 다음 달로 미루자."

## 왜 필요한가

### "100% 가용성"이 왜 잘못된 목표인가

100% 가용성을 추구하면:
1. **변경이 불가능해집니다**: 모든 배포는 다운타임 위험이 있습니다.
2. **비용이 폭발합니다**: 4-9's(99.99%)와 5-9's(99.999%) 사이 인프라 비용은 수십 배 차이납니다.
3. **사용자는 차이를 못 느낍니다**: 사용자의 인터넷 연결 자체가 99.9%보다 불안정합니다.

### Burn Rate — 에러 버짓이 얼마나 빠르게 소진되는가

단순히 에러 버짓 잔여량만 보는 것은 충분하지 않습니다. **소진 속도(Burn Rate)**도 중요합니다.

```
Burn Rate = 현재 에러 발생률 / 에러 버짓 허용률

Burn Rate = 1.0: 정확히 SLO를 달성하는 속도로 에러 발생
Burn Rate = 2.0: 에러 버짓의 2배 속도로 소진 → 15일 후 소진
Burn Rate = 14.4: 에러 버짓이 1시간 내에 2% 소진 → 즉시 대응 필요
```

## 실제 구현 예제

### 예제 1: Python으로 에러 버짓 계산기 구현

```python
from __future__ import annotations
from dataclasses import dataclass, field
from datetime import datetime, timedelta
import statistics

@dataclass
class SLOConfig:
    """SLO 목표 설정"""
    service_name: str
    window_days: int          = 30      # 측정 윈도우 (일)
    avail_target: float       = 0.999   # 가용성 목표 (99.9%)
    latency_threshold_ms: float = 200.0 # 지연시간 임계값 (ms)
    latency_target: float     = 0.99    # 해당 임계값 이내 비율 목표 (99%)


@dataclass
class RequestEvent:
    """개별 요청 이벤트"""
    timestamp: datetime
    latency_ms: float
    status_code: int

    @property
    def is_successful(self) -> bool:
        # 5xx는 서버 오류, 4xx는 클라이언트 오류 (SLI에서 제외 가능)
        return self.status_code < 500

    def is_fast(self, threshold_ms: float) -> bool:
        return self.latency_ms < threshold_ms


class ErrorBudgetTracker:
    """에러 버짓 실시간 추적기"""

    def __init__(self, config: SLOConfig):
        self.config = config
        self._events: list[RequestEvent] = []

    def record(self, event: RequestEvent) -> None:
        self._events.append(event)
        # 측정 윈도우 바깥의 오래된 이벤트 제거
        cutoff = datetime.now() - timedelta(days=self.config.window_days)
        self._events = [e for e in self._events if e.timestamp >= cutoff]

    # ── SLI 계산 ────────────────────────────────────────────────────────────

    def availability_sli(self) -> float:
        if not self._events:
            return 1.0
        good = sum(1 for e in self._events if e.is_successful)
        return good / len(self._events)

    def latency_sli(self) -> float:
        if not self._events:
            return 1.0
        fast = sum(1 for e in self._events
                   if e.is_fast(self.config.latency_threshold_ms))
        return fast / len(self._events)

    def p99_latency_ms(self) -> float:
        if not self._events:
            return 0.0
        sorted_lat = sorted(e.latency_ms for e in self._events)
        idx = int(len(sorted_lat) * 0.99)
        return sorted_lat[min(idx, len(sorted_lat) - 1)]

    # ── 에러 버짓 계산 ───────────────────────────────────────────────────────

    def availability_budget(self) -> dict:
        sli        = self.availability_sli()
        target     = self.config.avail_target
        budget_pct = 1.0 - target                  # 허용 오류 비율

        consumed   = max(0.0, target - sli)         # 소진된 비율
        remaining  = budget_pct - consumed           # 잔여 비율
        burn_rate  = consumed / budget_pct if budget_pct > 0 else 0.0

        total_min  = self.config.window_days * 24 * 60
        return {
            "sli":              f"{sli:.5%}",
            "target":           f"{target:.5%}",
            "budget_total":     f"{budget_pct:.5%}",
            "budget_consumed":  f"{consumed:.5%}",
            "budget_remaining": f"{remaining:.5%}",
            "burn_rate":        f"{burn_rate:.2f}x",
            "downtime_allowed": f"{total_min * budget_pct:.1f}분",
            "downtime_used":    f"{total_min * consumed:.1f}분",
        }

    def latency_budget(self) -> dict:
        sli       = self.latency_sli()
        target    = self.config.latency_target
        budget    = 1.0 - target
        consumed  = max(0.0, target - sli)
        remaining = budget - consumed
        burn_rate = consumed / budget if budget > 0 else 0.0
        return {
            "sli":              f"{sli:.5%}",
            "target":           f"{target:.5%}",
            "p99_ms":           f"{self.p99_latency_ms():.1f}",
            "threshold_ms":     f"{self.config.latency_threshold_ms:.0f}",
            "budget_remaining": f"{remaining:.5%}",
            "burn_rate":        f"{burn_rate:.2f}x",
        }

    def policy(self) -> str:
        """에러 버짓 잔여량에 따른 운영 정책 결정"""
        avail_b = self.availability_budget()
        lat_b   = self.latency_budget()

        avail_remaining = float(avail_b["budget_remaining"].rstrip("%")) / 100
        lat_remaining   = float(lat_b["budget_remaining"].rstrip("%")) / 100
        min_remaining   = min(avail_remaining, lat_remaining)

        budget_pct = 1.0 - self.config.avail_target
        consumed_ratio = (budget_pct - min_remaining) / budget_pct if budget_pct > 0 else 1.0

        if consumed_ratio >= 1.0:
            return "🚨 FREEZE   : 에러 버짓 소진. 모든 변경 중단 — 안정화 집중"
        elif consumed_ratio >= 0.8:
            return "⚠️  CAUTION : 80% 소진. 위험 배포 금지 — 인시던트 리뷰 필요"
        elif consumed_ratio >= 0.5:
            return "🟡 MODERATE : 50% 소진. 주의하며 배포 진행"
        else:
            return "✅ HEALTHY  : 에러 버짓 충분 — 정상 배포 및 실험 가능"

    def report(self) -> None:
        a = self.availability_budget()
        l = self.latency_budget()
        print(f"\n{'═'*55}")
        print(f"  에러 버짓 대시보드 — {self.config.service_name}")
        print(f"{'═'*55}")
        print(f"\n  [가용성]")
        print(f"    SLI          : {a['sli']}  (목표: {a['target']})")
        print(f"    에러 버짓    : 소진 {a['budget_consumed']} / 허용 {a['budget_total']}")
        print(f"    소진률       : {a['burn_rate']}")
        print(f"    다운타임     : {a['downtime_used']} 사용 / {a['downtime_allowed']} 허용")
        print(f"\n  [지연시간]")
        print(f"    P99          : {l['p99_ms']}ms  (임계값: {l['threshold_ms']}ms)")
        print(f"    SLI          : {l['sli']}  (목표: {l['target']})")
        print(f"    버짓 잔여    : {l['budget_remaining']}  (소진률: {l['burn_rate']})")
        print(f"\n  ▶ 운영 정책  : {self.policy()}")
        print(f"{'═'*55}\n")


# ── 사용 예시 ─────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    import random

    config  = SLOConfig(service_name="payment-api", window_days=30,
                        avail_target=0.999, latency_threshold_ms=200.0,
                        latency_target=0.99)
    tracker = ErrorBudgetTracker(config)

    # 30일치 요청 시뮬레이션 (총 100,000건)
    for _ in range(100_000):
        ts      = datetime.now() - timedelta(minutes=random.randint(0, 43_200))
        # 0.15% 서버 오류
        code    = 500 if random.random() < 0.0015 else 200
        # P99 ≈ 220ms (SLO를 약간 위반)
        latency = max(1.0, random.lognormvariate(4.5, 0.5))

        tracker.record(RequestEvent(timestamp=ts, latency_ms=latency,
                                    status_code=code))

    tracker.report()
```

### 예제 2: Prometheus 기록 규칙과 멀티 윈도우 알림 설정

실제 운영 환경에서는 Prometheus의 Recording Rule로 SLI를 사전 계산하고, Multi-Window 알림으로 오탐을 줄입니다.

```yaml
# prometheus/slo_recording_rules.yml
# SLI를 미리 계산하는 기록 규칙 (Recording Rules)
# 쿼리 성능을 위해 반드시 기록 규칙을 사용할 것

groups:
  - name: slo:availability:recording
    interval: 30s
    rules:
      # 가용성 SLI — 1분 단위 (실시간 모니터링용)
      - record: sli:availability:ratio_rate1m
        expr: |
          sum by (service) (
            rate(http_requests_total{status!~"5.."}[1m])
          )
          /
          sum by (service) (
            rate(http_requests_total[1m])
          )

      # 가용성 SLI — 5분 단위 (안정적 측정용)
      - record: sli:availability:ratio_rate5m
        expr: |
          sum by (service) (
            rate(http_requests_total{status!~"5.."}[5m])
          )
          /
          sum by (service) (
            rate(http_requests_total[5m])
          )

      # 가용성 SLI — 30일 단위 (SLO 컴플라이언스용)
      - record: sli:availability:ratio_rate30d
        expr: |
          sum by (service) (
            rate(http_requests_total{status!~"5.."}[30d])
          )
          /
          sum by (service) (
            rate(http_requests_total[30d])
          )

  - name: slo:latency:recording
    interval: 30s
    rules:
      # 지연시간 SLI: 200ms 이내 응답 비율
      - record: sli:latency_under200ms:ratio_rate5m
        expr: |
          sum by (service) (
            rate(http_request_duration_seconds_bucket{le="0.2"}[5m])
          )
          /
          sum by (service) (
            rate(http_request_duration_seconds_count[5m])
          )

  - name: slo:error_budget:recording
    rules:
      # 가용성 에러 버짓 소진률 (Burn Rate)
      # Burn Rate = 1.0: 정확히 SLO 속도, 2.0: 2배 빠르게 소진
      - record: error_budget:availability:burn_rate1h
        expr: |
          (1 - sli:availability:ratio_rate1m) / (1 - 0.999)

      - record: error_budget:availability:burn_rate6h
        expr: |
          (1 - sli:availability:ratio_rate5m) / (1 - 0.999)

      # 30일 기준 에러 버짓 잔여량 (0~1, 1이면 버짓 전부 남음)
      - record: error_budget:availability:remaining30d
        expr: |
          clamp_min(
            (sli:availability:ratio_rate30d - 0.999) / (1 - 0.999),
            0
          )


# prometheus/slo_alerting_rules.yml
# 멀티 윈도우(Multi-Window) 알림: 짧은 + 긴 윈도우 동시 확인으로 오탐 최소화
#
# 원칙: 빠르게 감지 + 느리게 확정
# - 짧은 윈도우(1h): 빠른 문제 감지
# - 긴 윈도우(6h, 3d): 노이즈 필터링
#
# Burn Rate 임계값 설계 (30일 윈도우 기준):
# | Alert     | Burn Rate | 감지 기준 | 에러 버짓 소진 속도 |
# |-----------|-----------|-----------|---------------------|
# | Critical  | 14.4x     | 1h        | 1시간에 2% 소진     |
# | High      | 6.0x      | 6h        | 6시간에 5% 소진     |
# | Medium    | 3.0x      | 3d        | 3일에 10% 소진      |
# | Low       | 1.0x      | 30d       | 30일에 100% 소진    |

groups:
  - name: slo:availability:alerting
    rules:
      # [CRITICAL] 즉시 호출(Page): 1시간 내 에러 버짓 2% 소진 예상
      - alert: AvailabilitySLOBurnRateCritical
        expr: |
          error_budget:availability:burn_rate1h > 14.4
          and
          error_budget:availability:burn_rate6h > 14.4
        for: 2m
        labels:
          severity: critical
          slo: availability
        annotations:
          summary: >-
            {{ $labels.service }} 가용성 SLO 심각 위반
          description: >-
            현재 Burn Rate: {{ $value | humanize }}x
            이 속도라면 에러 버짓이 1시간 내에 2% 소진됩니다.
            즉각 인시던트 대응을 시작하세요.
          runbook: https://wiki.internal/slo/availability-critical

      # [HIGH] 티켓 생성: 6시간 내 에러 버짓 5% 소진 예상
      - alert: AvailabilitySLOBurnRateHigh
        expr: |
          error_budget:availability:burn_rate6h > 6.0
          and
          error_budget:availability:burn_rate1h > 6.0
        for: 15m
        labels:
          severity: high
          slo: availability
        annotations:
          summary: >-
            {{ $labels.service }} 가용성 SLO 빠른 소진
          description: >-
            Burn Rate: {{ $value | humanize }}x
            에러 버짓 5%가 6시간 내 소진됩니다.
            비즈니스 시간 내 조사 필요.

      # [LOW] 에러 버짓 90% 소진 경보 (이달 말 위험)
      - alert: ErrorBudgetAlmostExhausted
        expr: error_budget:availability:remaining30d < 0.10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: >-
            {{ $labels.service }} 이번 달 에러 버짓 90% 소진
          description: >-
            잔여 에러 버짓: {{ $value | humanizePercentage }}
            이번 달 배포 및 실험에 주의가 필요합니다.
```

## 주의사항 및 팁

**1. SLO는 고객 관점에서 정의하라**

CPU 사용률, 메모리 같은 내부 지표는 SLI가 아닙니다. "HTTP 200 응답"처럼 사용자가 실제로 경험하는 것을 측정하세요. 가능하면 합성 모니터링(Synthetic Monitoring)으로 실제 사용자 시나리오를 주기적으로 테스트하세요.

**2. 처음에는 SLO 하나로 시작하라**

처음부터 가용성, 지연시간, 처리량, 정확성 등 여러 SLO를 동시에 정의하고 관리하면 복잡해집니다. 가장 중요한 SLI 하나로 시작해서 팀이 익숙해지면 확장하세요.

**3. 에러 버짓 정책을 문서화하고 자동화하라**

에러 버짓이 소진됐을 때 무엇을 할 것인지를 **사전에 팀과 합의**하고 문서화하세요. 이상적으로는 CI/CD 파이프라인에서 에러 버짓 소진 시 배포를 자동으로 차단하는 게이트를 구현하세요.

**4. 멀티 윈도우 알림으로 오탐을 줄여라**

단일 윈도우 알림은 일시적인 스파이크에 오탐이 많습니다. 짧은 윈도우(빠른 감지)와 긴 윈도우(안정적 확인)를 AND 조건으로 결합하면 오탐을 크게 줄일 수 있습니다. Google SRE Workbook에서 권장하는 Burn Rate 임계값을 시작점으로 사용하세요.

**5. SLO는 100%가 되어서는 안 된다**

100% SLO는 어떤 변경도 허용하지 않습니다. 99.9%와 99.99%의 차이는 월간 43분 vs 4.3분입니다. 사용자가 실제로 그 차이를 느끼는지 먼저 검증하세요. 더 높은 SLO는 더 많은 비용과 혁신 속도 저하를 의미합니다.

**6. 측정할 수 없는 것은 SLO로 만들지 마라**

"응답이 정확해야 한다"처럼 측정 방법이 명확하지 않은 지표는 SLI가 될 수 없습니다. SLI는 반드시 자동으로 측정 가능하고, 수치로 표현 가능해야 합니다.

## 참고 자료

- [Google SRE Book — Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
- [Google SRE Workbook — Implementing SLOs](https://sre.google/workbook/implementing-slos/)
- [Google SRE Workbook — Error Budget Policy](https://sre.google/workbook/error-budget-policy/)
- [Prometheus — Alerting 가이드](https://prometheus.io/docs/practices/alerting/)
