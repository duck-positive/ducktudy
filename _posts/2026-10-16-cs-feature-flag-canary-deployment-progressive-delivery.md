---
layout: post
title: "Feature Flag와 점진적 배포 전략: 프로덕션에서 안전하게 변경을 출시하는 기술"
date: 2026-10-16
categories: [cs, computer-science]
tags: [feature-flag, canary-deployment, progressive-delivery, blue-green, A/B-testing, DevOps, 점진적배포]
---

## Feature Flag란 무엇인가

**Feature Flag**(또는 Feature Toggle, Feature Switch)는 코드 배포와 기능 출시를 분리하는 기술입니다. 새 기능을 담은 코드를 이미 배포했더라도, 플래그가 꺼져 있는 동안은 사용자에게 노출되지 않습니다. 플래그를 켜는 순간 즉각적으로 기능이 활성화됩니다.

```
전통적 배포:  코드 작성 → 코드 배포 → 기능 출시  (배포 = 출시)
Feature Flag: 코드 작성 → 코드 배포 → (필요할 때) 플래그 ON → 기능 출시
```

이 분리가 가져오는 이점은 막강합니다.

- **즉시 롤백**: 문제 발생 시 플래그를 끄면 배포 없이 수 초 만에 기능 비활성화
- **점진적 출시**: 1% → 10% → 100% 순으로 트래픽을 조금씩 늘려 안전하게 확장
- **A/B 테스트**: 동일 코드베이스에서 두 가지 변형을 동시에 테스트
- **킬 스위치**: 과부하 상황에서 비필수 기능을 즉시 끄는 서킷 브레이커 역할

## 왜 Feature Flag가 필요한가

### 배포의 고통을 없애다

전통적인 빅뱅 배포(하루 또는 주 단위로 많은 변경을 한꺼번에 배포)는 매우 위험합니다. 무엇이 장애를 일으켰는지 파악하기 어렵고, 롤백 비용도 큽니다. 현대 팀은 **지속적 배포(CD)** 를 통해 하루에도 수십 번 배포하지만, 그만큼 작은 변경 단위로 리스크를 분산시킵니다.

Feature Flag는 이 모델을 한 단계 더 발전시킵니다. 출시하지 않은 기능을 메인 브랜치에 병합(trunk-based development)할 수 있어 장기 피처 브랜치의 병합 충돌 문제도 해소됩니다.

### 다양한 적용 시나리오

| 시나리오 | 설명 |
|---------|------|
| **출시 플래그(Release Flag)** | 개발 완료 전까지 기능 숨김, 출시 준비 완료 시 ON |
| **실험 플래그(Experiment Flag)** | A/B 테스트, 통계적 유의성 달성 후 승자 선택 |
| **운영 플래그(Ops Flag)** | 성능 최적화 기능 토글, 부하 분산 목적 킬 스위치 |
| **권한 플래그(Permission Flag)** | 베타 사용자·특정 기업·특정 지역에만 기능 노출 |

## Feature Flag 직접 구현하기

### 단순 파일 기반 구현

```python
# feature_flags.py - 간단한 Feature Flag 시스템
import json
import time
import hashlib
from pathlib import Path
from functools import lru_cache


class FeatureFlagManager:
    def __init__(self, config_path: str = "flags.json"):
        self.config_path = Path(config_path)
        self._cache_time = 0
        self._cache_ttl = 30  # 30초마다 설정 파일 재읽기
        self._flags = {}

    def _load_flags(self):
        now = time.time()
        if now - self._cache_time > self._cache_ttl:
            try:
                with open(self.config_path) as f:
                    self._flags = json.load(f)
                self._cache_time = now
            except (FileNotFoundError, json.JSONDecodeError):
                pass

    def is_enabled(self, flag_name: str, user_id: str = None) -> bool:
        """
        플래그 활성화 여부 판단.
        percentage 필드가 있으면 사용자 ID 해시로 일관된 롤아웃 결정.
        """
        self._load_flags()
        flag = self._flags.get(flag_name, {})

        if not flag.get("enabled", False):
            return False

        # 퍼센트 롤아웃
        percentage = flag.get("percentage", 100)
        if percentage < 100 and user_id:
            # 사용자 ID 해시를 기반으로 0~99 버킷 배정 (일관성 보장)
            bucket = int(hashlib.md5(
                f"{flag_name}:{user_id}".encode()
            ).hexdigest(), 16) % 100
            return bucket < percentage

        # 특정 사용자 화이트리스트
        whitelist = flag.get("whitelist", [])
        if whitelist and user_id:
            return user_id in whitelist

        return True

    def get_variant(self, flag_name: str, user_id: str) -> str:
        """A/B 테스트용 변형(variant) 반환"""
        self._load_flags()
        flag = self._flags.get(flag_name, {})
        variants = flag.get("variants", ["control", "treatment"])

        bucket = int(hashlib.md5(
            f"{flag_name}:{user_id}".encode()
        ).hexdigest(), 16) % len(variants)
        return variants[bucket]


# flags.json 예시:
FLAGS_CONFIG_EXAMPLE = {
    "new_checkout_flow": {
        "enabled": True,
        "percentage": 10,       # 10% 사용자에게만 출시
        "description": "신규 결제 UX 테스트"
    },
    "dark_mode": {
        "enabled": True,
        "percentage": 100
    },
    "ai_recommendations": {
        "enabled": True,
        "whitelist": ["user_001", "user_beta_tester"],  # 특정 사용자만
        "description": "AI 추천 기능 베타"
    },
    "heavy_feature": {
        "enabled": False,       # 완전 비활성화 (킬 스위치)
        "description": "DB 과부하 시 즉시 비활성화"
    }
}

# 사용 예시
flags = FeatureFlagManager("flags.json")

def checkout(user_id: str):
    if flags.is_enabled("new_checkout_flow", user_id):
        return new_checkout_flow(user_id)
    else:
        return legacy_checkout_flow(user_id)

def new_checkout_flow(user_id): return f"새 결제 ({user_id})"
def legacy_checkout_flow(user_id): return f"기존 결제 ({user_id})"

# 테스트
import json
with open("flags.json", "w") as f:
    json.dump(FLAGS_CONFIG_EXAMPLE, f)

for uid in ["user_100", "user_200", "user_300"]:
    result = checkout(uid)
    print(f"{uid}: {result}")
```

### Redis 기반 실시간 업데이트 구현

파일 기반 구현은 배포 재시작이 필요합니다. **Redis**를 백엔드로 쓰면 실시간으로 플래그를 바꿀 수 있습니다.

```python
import redis
import json
import hashlib

class RedisFeatureFlagManager:
    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis = redis.from_url(redis_url, decode_responses=True)
        self.prefix = "ff:"

    def set_flag(self, flag_name: str, config: dict):
        """플래그 설정 저장 (즉시 반영)"""
        key = f"{self.prefix}{flag_name}"
        self.redis.set(key, json.dumps(config))
        print(f"플래그 '{flag_name}' 업데이트: {config}")

    def is_enabled(self, flag_name: str, user_id: str = None,
                   context: dict = None) -> bool:
        key = f"{self.prefix}{flag_name}"
        raw = self.redis.get(key)
        if not raw:
            return False

        flag = json.loads(raw)

        if not flag.get("enabled", False):
            return False

        # 퍼센트 롤아웃
        percentage = flag.get("percentage", 100)
        if percentage < 100 and user_id:
            bucket = int(hashlib.sha256(
                f"{flag_name}:{user_id}".encode()
            ).hexdigest(), 16) % 100
            if bucket >= percentage:
                return False

        # 컨텍스트 기반 규칙 (예: 특정 국가, 플랜, 역할)
        rules = flag.get("rules", [])
        if rules and context:
            for rule in rules:
                attr = rule.get("attribute")
                values = rule.get("values", [])
                if attr and context.get(attr) in values:
                    return True
            return False  # 규칙이 있지만 매칭 없으면 비활성화

        return True

    def gradual_rollout(self, flag_name: str, target_pct: int,
                        step: int = 10, interval_sec: int = 60):
        """점진적 롤아웃 스케줄 생성 (시연용)"""
        import time
        key = f"{self.prefix}{flag_name}"
        raw = self.redis.get(key)
        current = json.loads(raw) if raw else {"enabled": True, "percentage": 0}

        current_pct = current.get("percentage", 0)
        print(f"점진적 롤아웃 시작: {current_pct}% → {target_pct}%")

        while current_pct < target_pct:
            current_pct = min(current_pct + step, target_pct)
            current["percentage"] = current_pct
            self.redis.set(key, json.dumps(current))
            print(f"  → {current_pct}% 롤아웃 완료")
            if current_pct < target_pct:
                time.sleep(interval_sec)  # 실제 환경에서는 메트릭 모니터링 후 진행


# 사용 예
# manager = RedisFeatureFlagManager()
# manager.set_flag("new_search_ui", {"enabled": True, "percentage": 0})
# manager.gradual_rollout("new_search_ui", target_pct=100, step=10, interval_sec=300)
```

## 카나리 배포(Canary Deployment)

Feature Flag가 **코드 레벨**에서 동작한다면, 카나리 배포는 **인프라 레벨**에서 트래픽을 분리합니다. 새 버전을 소수의 서버(카나리 인스턴스)에만 배포하고, 나머지는 안정 버전을 유지합니다.

```yaml
# Kubernetes Argo Rollouts를 사용한 카나리 배포 예시
# rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: checkout-service
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 5          # 5% 트래픽을 카나리로
        - pause: {duration: 5m} # 5분 대기 후 메트릭 분석
        - setWeight: 20         # 20%로 증가
        - pause: {duration: 10m}
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 100        # 전체 전환
      # 자동 롤백 기준: 에러율 5% 초과 시 자동 롤백
      analysis:
        templates:
          - templateName: error-rate-check
        startingStep: 1
        args:
          - name: service-name
            value: checkout-service
---
# analysis-template.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-check
spec:
  metrics:
    - name: error-rate
      interval: 1m
      successCondition: result[0] < 0.05  # 에러율 5% 미만
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{
              job="{{args.service-name}}",
              status=~"5.."
            }[1m])) /
            sum(rate(http_requests_total{
              job="{{args.service-name}}"
            }[1m]))
```

### Nginx를 이용한 가중 트래픽 분산

```nginx
# nginx.conf - 카나리 트래픽 분산
upstream checkout_stable {
    server checkout-stable:8080 weight=95;
}

upstream checkout_canary {
    server checkout-canary:8080 weight=5;
}

# split_clients를 사용한 결정론적 분배 (같은 사용자는 항상 같은 버전)
split_clients "${remote_addr}${request_uri}" $upstream_pool {
    5%    checkout_canary;
    *     checkout_stable;
}

server {
    listen 80;
    location /checkout {
        proxy_pass http://$upstream_pool;
        proxy_set_header X-Canary $upstream_pool;

        # 카나리 트래픽에 헤더 추가 (로깅/모니터링용)
        add_header X-Version $upstream_pool;
    }
}
```

## 블루-그린 배포 vs 카나리 배포

| 구분 | 블루-그린 | 카나리 |
|------|---------|-------|
| **트래픽 전환** | 즉각적 0% → 100% | 점진적 1% → 100% |
| **리소스** | 인프라 2배 필요 | 일부만 신규 버전 사용 |
| **롤백 속도** | 즉각 (DNS 전환 또는 LB 전환) | 단계별 (자동/수동) |
| **위험 노출** | 전체 전환 순간 위험 | 소수 사용자만 먼저 노출 |
| **적합 상황** | DB 스키마 변경, 데이터 마이그레이션 | 기능 변경, UI 업데이트 |

## 주의사항 및 팁

### 1. 기술 부채: 플래그 청소
플래그가 쌓이면 코드가 복잡해집니다. 모든 플래그에 **만료일**을 설정하고, 100% 롤아웃 완료된 플래그는 빠르게 코드에서 제거하세요.

```python
# 플래그 사용 시 항상 만료일 기록
FLAGS_CONFIG_EXAMPLE["new_checkout_flow"]["expires"] = "2026-11-01"
FLAGS_CONFIG_EXAMPLE["new_checkout_flow"]["owner"] = "checkout-team"
# CI 파이프라인에서 만료된 플래그 경고 알림 자동화 권장
```

### 2. 플래그 평가 일관성
같은 사용자가 서로 다른 서버에서 다른 변형을 경험하면 UX가 깨집니다. 사용자 ID 해시로 결정론적 버킷팅을 구현해야 합니다(위 예시 참고).

### 3. 데이터베이스 호환성
새 기능이 DB 스키마 변경을 동반할 때는 반드시 **하위 호환 방식(expand-contract 패턴)**으로 마이그레이션하세요:
1. 새 컬럼 추가 (nullable)
2. 코드에서 두 컬럼 모두 지원 (플래그로 전환)
3. 구 컬럼 데이터 마이그레이션
4. 구 컬럼 제거

### 4. 분산 시스템에서의 플래그 전파 지연
모든 서버에 플래그 변경이 전파되는 데 수 초가 걸릴 수 있습니다. 완전 전파 전까지 일부 서버에서는 구 동작, 일부에서는 신 동작이 공존합니다. 이에 강건한 코드를 작성해야 합니다.

### 5. 관찰 가능성(Observability)
Feature Flag와 연동된 메트릭을 대시보드로 모니터링하세요. 플래그를 켰을 때의 에러율, 레이턴시, 전환율 변화를 실시간으로 비교해야 빠른 의사결정이 가능합니다.

```python
# 메트릭 기록 예시 (Prometheus + Python)
from prometheus_client import Counter, Histogram

flag_evaluations = Counter(
    'feature_flag_evaluations_total',
    'Feature flag evaluation count',
    ['flag_name', 'result', 'variant']
)

def tracked_is_enabled(flags, flag_name, user_id):
    result = flags.is_enabled(flag_name, user_id)
    flag_evaluations.labels(
        flag_name=flag_name,
        result=str(result),
        variant="enabled" if result else "disabled"
    ).inc()
    return result
```

## 참고 자료
- [LaunchDarkly: Feature Flags 개요](https://launchdarkly.com/featureflags.html)
- [OpenFeature: CNCF 표준 Feature Flag API](https://openfeature.dev/)
- [Martin Fowler: Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [Argo Rollouts: 카나리 배포 문서](https://argo-rollouts.readthedocs.io/)
