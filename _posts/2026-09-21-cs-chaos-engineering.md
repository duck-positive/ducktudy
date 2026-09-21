---
layout: post
title: "카오스 엔지니어링 완전 정복: 장애를 먼저 일으켜 복원력을 키우는 기술"
date: 2026-09-21
categories: [cs, computer-science]
tags: [chaos-engineering, resilience, netflix, fault-injection, distributed-systems, sre, devops]
---

## 개념 설명

**카오스 엔지니어링(Chaos Engineering)**은 "프로덕션 환경에서 시스템이 혼돈 상황을 견딜 수 있다는 자신감을 쌓기 위해 시스템을 의도적으로 실험하는 규율"이다. 이 정의는 *Principles of Chaos Engineering* 문서에서 가져온 것으로, Netflix가 2010년 Chaos Monkey를 도입하며 개척한 분야다.

전통적인 품질 보증은 **예상된 시나리오**만 검증한다. 하지만 복잡한 분산 시스템은 아무도 예측하지 못한 방식으로 실패한다 — 네트워크 지연이 증폭되거나, 데이터베이스가 특정 시간대에만 느려지거나, 의존 서비스가 응답을 늦게 돌려보내거나. 카오스 엔지니어링은 **통제된 실험**을 통해 이러한 알 수 없는 약점을 사전에 발견한다.

### 카오스 엔지니어링의 4대 원칙

1. **정상 상태 행동(Steady State)을 가설로 세운다**  
   처리량, 오류율, p99 레이턴시 등 시스템 출력 지표를 측정하여 정상 상태를 정의한다.

2. **현실적인 사건을 변수로 도입한다**  
   서버 크래시, 네트워크 파티션, 디스크 가득 참, 외부 의존 서비스 지연 등 실제로 발생하는 장애를 주입한다.

3. **프로덕션에서 실험한다**  
   스테이징 환경의 결과는 프로덕션과 다를 수 있다. 폭발 반경(blast radius)을 최소화하면서 실제 트래픽으로 검증한다.

4. **실험을 자동화하여 지속적으로 실행한다**  
   CI/CD 파이프라인에 통합하거나 정기적으로 실행하여 시스템이 시간이 지나도 복원력을 유지하는지 확인한다.

---

## 왜 필요한가

### 분산 시스템의 복잡성

마이크로서비스 아키텍처에서 서비스 A가 B에 의존하고, B가 C·D에 의존하는 체인이 형성되면 어디서든 장애가 전파될 수 있다. 작은 결함이 연쇄 장애(cascading failure)로 이어져 전체 시스템을 마비시키는 사례는 드물지 않다.

Netflix는 2011년 AWS us-east-1 장애로 수 시간 서비스 중단을 경험한 후 카오스 엔지니어링에 전사적으로 투자했다. 이후 수천 번의 카오스 실험을 통해 "두려움 없이 배포할 수 있는" 시스템을 만들었다.

### 숨겨진 가정을 드러낸다

"이 서비스는 절대 다운되지 않을 거야", "재시도 로직이 있으니 괜찮아" 같은 **암묵적 가정**이 팀 내에 쌓인다. 카오스 실험은 이를 즉시 검증하여 거짓 가정을 제거한다.

---

## 실전 구현 예제

### 예제 1 — Python으로 구현하는 장애 주입 라이브러리

간단한 카오스 데코레이터를 만들어 지연·예외를 주입한다.

```python
import time
import random
import functools
from typing import Callable, Optional


class ChaosConfig:
    def __init__(
        self,
        failure_rate: float = 0.0,        # 예외 발생 확률 (0~1)
        latency_ms: Optional[int] = None, # 추가 지연 (ms)
        latency_jitter_ms: int = 0,       # 지연 흔들림 (ms)
        enabled: bool = True,
    ):
        self.failure_rate = failure_rate
        self.latency_ms = latency_ms
        self.latency_jitter_ms = latency_jitter_ms
        self.enabled = enabled


def chaos(config: ChaosConfig):
    """함수에 카오스를 주입하는 데코레이터"""
    def decorator(func: Callable):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            if not config.enabled:
                return func(*args, **kwargs)

            # 지연 주입
            if config.latency_ms is not None:
                jitter = random.randint(0, config.latency_jitter_ms)
                delay = (config.latency_ms + jitter) / 1000
                time.sleep(delay)

            # 예외 주입
            if random.random() < config.failure_rate:
                raise RuntimeError(
                    f"[Chaos] {func.__name__} 에서 의도적 장애 주입"
                )

            return func(*args, **kwargs)
        return wrapper
    return decorator


# 사용 예 — 10% 확률로 실패, 200ms + 최대 50ms 지터 지연
payment_chaos = ChaosConfig(
    failure_rate=0.1,
    latency_ms=200,
    latency_jitter_ms=50,
    enabled=True,  # 프로덕션: True, 로컬 개발: False
)


@chaos(payment_chaos)
def process_payment(order_id: str, amount: float) -> dict:
    # 실제 결제 처리 로직
    return {"status": "success", "order_id": order_id, "amount": amount}


# 실험 실행
SUCCESS, FAILURE = 0, 0
for _ in range(100):
    try:
        result = process_payment("ORD-001", 50_000)
        SUCCESS += 1
    except RuntimeError as e:
        FAILURE += 1

print(f"성공: {SUCCESS}, 실패: {FAILURE}")
# 예: 성공: 91, 실패: 9
```

---

### 예제 2 — 카오스 실험 프레임워크 (Python)

가설 → 실험 → 검증 흐름을 구조화하는 미니 프레임워크다.

```python
import time
import statistics
from dataclasses import dataclass, field
from typing import Callable, List


@dataclass
class Measurement:
    latency_ms: float
    success: bool
    error: Optional[str] = None


@dataclass
class ExperimentResult:
    hypothesis: str
    measurements: List[Measurement] = field(default_factory=list)

    @property
    def success_rate(self) -> float:
        if not self.measurements:
            return 0.0
        return sum(1 for m in self.measurements if m.success) / len(self.measurements)

    @property
    def p99_latency_ms(self) -> float:
        latencies = [m.latency_ms for m in self.measurements]
        return statistics.quantiles(latencies, n=100)[98] if latencies else 0.0

    def is_hypothesis_valid(self, min_success_rate: float, max_p99_ms: float) -> bool:
        return (
            self.success_rate >= min_success_rate
            and self.p99_latency_ms <= max_p99_ms
        )


class ChaosExperiment:
    def __init__(self, hypothesis: str, target: Callable, chaos_fn: Callable):
        self.hypothesis = hypothesis
        self.target = target
        self.chaos_fn = chaos_fn  # 장애를 주입하는 함수

    def run(self, iterations: int = 100) -> ExperimentResult:
        result = ExperimentResult(hypothesis=self.hypothesis)

        for _ in range(iterations):
            self.chaos_fn()  # 장애 주입
            start = time.perf_counter()
            try:
                self.target()
                elapsed = (time.perf_counter() - start) * 1000
                result.measurements.append(Measurement(elapsed, success=True))
            except Exception as e:
                elapsed = (time.perf_counter() - start) * 1000
                result.measurements.append(
                    Measurement(elapsed, success=False, error=str(e))
                )

        return result


# ---- 실제 실험 예시 ----

import requests
import random as _random

def inject_network_delay():
    """20% 확률로 네트워크 지연 시뮬레이션"""
    if _random.random() < 0.2:
        time.sleep(0.5)  # 500ms 추가 지연

def call_payment_api():
    """결제 API 호출 (여기서는 예시로 로컬에서 실행)"""
    if _random.random() < 0.05:
        raise ConnectionError("결제 API 연결 실패")
    time.sleep(_random.uniform(0.01, 0.05))  # 정상 10~50ms

experiment = ChaosExperiment(
    hypothesis="결제 API는 네트워크 불안정 상황에서도 성공률 95% 이상, p99 지연 300ms 이하를 유지한다",
    target=call_payment_api,
    chaos_fn=inject_network_delay,
)

result = experiment.run(iterations=200)
print(f"가설: {result.hypothesis}")
print(f"성공률: {result.success_rate:.1%}")
print(f"p99 레이턴시: {result.p99_latency_ms:.1f}ms")
print(f"가설 검증: {'통과' if result.is_hypothesis_valid(0.95, 300) else '실패 — 시스템 개선 필요'}")
```

---

## 카오스 도구 생태계

| 도구 | 특징 | 타깃 환경 |
|---|---|---|
| **Netflix Chaos Monkey** | 무작위 EC2 인스턴스 종료 | AWS |
| **Chaos Toolkit** | 오픈소스, YAML 기반 실험 정의 | 범용 |
| **LitmusChaos** | CNCF, Kubernetes 네이티브 | K8s |
| **Gremlin** | 상용, 세밀한 장애 제어 | 범용 |
| **AWS Fault Injection Simulator** | AWS 관리형 서비스 | AWS |
| **Chaos Mesh** | CNCF, 쿠버네티스 장애 주입 | K8s |

---

## GameDay — 팀 훈련 프로세스

GameDay는 팀 전체가 참여하는 카오스 훈련이다.

1. **준비**: 실험 범위와 롤백 계획을 문서화한다
2. **모니터링 대시보드 준비**: Grafana·Datadog 등으로 정상 상태 기준선을 잡는다
3. **장애 주입**: 정의된 시나리오(서버 다운, DB 느림, 디스크 가득 참 등)를 실행한다
4. **팀 대응 관찰**: 알람이 제때 울리는지, 런북대로 대응하는지 기록한다
5. **복기(Retrospective)**: 발견된 약점을 정리하고 개선 태스크를 추적한다

---

## 주의사항과 팁

**1. 폭발 반경을 최소화하라**  
첫 실험은 단일 서비스·소규모 사용자 그룹으로 시작한다. 성공적으로 검증한 후 범위를 점진적으로 확장한다.

**2. 관측 가능성(Observability)이 선행되어야 한다**  
메트릭·로그·트레이싱이 없으면 실험 결과를 해석할 수 없다. Prometheus + Grafana, ELK 스택, OpenTelemetry 등을 먼저 갖춰라.

**3. 자동 중단 조건을 반드시 설정하라**  
실험 중 오류율이 임계치(예: 1%)를 초과하면 즉시 자동으로 실험을 중단하고 시스템을 복구하는 **안전 스위치(kill switch)**가 필수다.

**4. 비즈니스 이해관계자와 소통하라**  
카오스 실험 일정을 사전에 공유하고, 피크 타임을 피하며, 법무·보안팀의 승인을 받아야 하는 경우가 있다.

**5. 프로덕션 없이 시작하라**  
스테이징 환경에서 실험 방법론을 익힌 후 프로덕션으로 점진적으로 확대한다. "스테이징에서만 발생하는 문제"도 카오스 엔지니어링의 학습 결과다.

**6. 복원력을 코드에 녹여라**  
Circuit Breaker(Resilience4j, Hystrix), 타임아웃, 재시도(Exponential Backoff), Bulkhead 패턴이 없다면 카오스 실험 전에 먼저 구현하라.

---

## 참고 자료
- [Principles of Chaos Engineering](https://principlesofchaos.org/)
- [Netflix Chaos Monkey — InfoQ](https://www.infoq.com/news/2015/09/netflix-chaos-engineering)
- [Awesome Chaos Engineering — GitHub](https://github.com/xcaspar/awesome-chaos-engineering)
- [Chaos Engineering 101 — Educative](https://educative.io/blog/chaos-engineering-process-principles)
