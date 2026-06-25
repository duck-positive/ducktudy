---
layout: post
title: "레이트 리미팅 알고리즘 완전 정복: Token Bucket, Leaky Bucket, Sliding Window 구현과 비교"
date: 2026-06-25
categories: [cs, computer-science]
tags: [rate-limiting, token-bucket, leaky-bucket, sliding-window, system-design, api, 시스템설계]
---

API를 운영하다 보면 특정 클라이언트가 초당 수천 건의 요청을 쏟아내는 상황을 마주치게 됩니다. 이를 막지 않으면 서버는 과부하로 다운되고, 다른 사용자는 서비스를 이용할 수 없게 됩니다. **레이트 리미팅(Rate Limiting)**은 이런 상황을 방어하는 핵심 메커니즘입니다. 오늘은 대표적인 네 가지 알고리즘을 깊이 있게 살펴봅니다.

## 레이트 리미팅이란 무엇인가

레이트 리미팅은 일정 시간 동안 클라이언트가 서버에 보낼 수 있는 요청 수를 제한하는 기법입니다. 단순히 DoS 공격을 막는 것을 넘어, 다음과 같은 목적으로 사용됩니다.

- **공정한 자원 배분**: 특정 사용자가 서버 자원을 독점하지 못하도록 제한
- **비용 제어**: 외부 API 호출이나 데이터베이스 쿼리 비용을 예측 가능하게 유지
- **SLA 보장**: 전체 사용자에게 일정 수준의 응답 속도를 보장
- **보안 강화**: 브루트포스 공격, 자격증명 스터핑 등 자동화 공격 방어

레이트 리미팅은 보통 **API 게이트웨이**, **리버스 프록시(Nginx, Envoy)**, 또는 **애플리케이션 미들웨어** 레이어에서 구현됩니다.

## 알고리즘 1: 고정 윈도우 카운터 (Fixed Window Counter)

가장 단순한 방식입니다. 시간을 고정된 윈도우(예: 1분)로 나누고, 각 윈도우 안에서 요청 횟수를 카운트합니다.

```
[0s ~ 60s]   카운터: 0
요청 → 카운터: 1
요청 → 카운터: 2
...
요청 → 카운터: 100  (한도 초과 시 429 반환)
[60s ~ 120s] 카운터: 0 (리셋)
```

**장점**: 구현이 매우 단순하고 메모리 사용량이 적습니다.

**치명적 단점**: 경계 폭발(Boundary Burst) 문제가 있습니다. 59초에 100개, 61초에 100개를 요청하면 2초 안에 200개의 요청이 처리됩니다.

## 알고리즘 2: 슬라이딩 윈도우 로그 (Sliding Window Log)

각 요청의 타임스탬프를 저장하고, 새 요청이 올 때마다 현재 시각 기준으로 1분 이내의 요청 수를 세는 방식입니다.

```python
import time
from collections import deque

class SlidingWindowLog:
    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window = window_seconds
        self.log: deque = deque()

    def allow(self) -> bool:
        now = time.time()
        # 윈도우 밖의 오래된 타임스탬프 제거
        while self.log and self.log[0] <= now - self.window:
            self.log.popleft()
        
        if len(self.log) < self.limit:
            self.log.append(now)
            return True
        return False

# 사용 예시: 1분에 5회 제한
limiter = SlidingWindowLog(limit=5, window_seconds=60)
for i in range(7):
    result = limiter.allow()
    print(f"요청 {i+1}: {'허용' if result else '거부'}")
```

**장점**: 경계 폭발 문제가 없습니다. 항상 직전 N초 동안의 실제 요청 수를 정확히 측정합니다.

**단점**: 요청마다 타임스탬프를 저장하므로, 트래픽이 많을수록 메모리 사용량이 커집니다. 100만 req/s라면 저장해야 할 타임스탬프도 어마어마합니다.

## 알고리즘 3: 토큰 버킷 (Token Bucket)

현실에서 가장 널리 쓰이는 알고리즘입니다. 버킷에 일정 속도로 토큰이 채워지고, 요청이 올 때마다 토큰을 소비합니다. 버킷이 비면 요청을 거부합니다.

```
버킷 용량: 10개
토큰 충전 속도: 1초당 2개
현재 토큰: 10개

요청 1개 → 토큰 9개
요청 1개 → 토큰 8개
...
요청 1개 → 토큰 0개
요청 → 거부 (토큰 없음)
1초 대기 → 토큰 2개 충전
```

```python
import time
import threading

class TokenBucket:
    def __init__(self, capacity: int, refill_rate: float):
        """
        capacity: 버킷 최대 용량 (버스트 허용량)
        refill_rate: 초당 충전되는 토큰 수
        """
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.tokens = float(capacity)
        self.last_refill = time.monotonic()
        self.lock = threading.Lock()

    def _refill(self):
        now = time.monotonic()
        elapsed = now - self.last_refill
        new_tokens = elapsed * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + new_tokens)
        self.last_refill = now

    def consume(self, tokens: int = 1) -> bool:
        with self.lock:
            self._refill()
            if self.tokens >= tokens:
                self.tokens -= tokens
                return True
            return False

# AWS API Gateway, Stripe, GitHub API 등이 이 방식을 사용
bucket = TokenBucket(capacity=10, refill_rate=2.0)

# 순간 버스트: 10개 즉시 처리
for i in range(12):
    allowed = bucket.consume()
    print(f"요청 {i+1}: {'허용 ✓' if allowed else '거부 ✗'}")
```

**핵심 특성**: 평균 처리율은 `refill_rate`로 제한되지만, 버킷이 가득 찬 상태에서는 `capacity`만큼 순간적으로 버스트를 허용합니다. 실제 사용자 행동(주기적인 사용, 가끔씩의 폭발적 요청)을 잘 반영합니다.

## 알고리즘 4: 누출 버킷 (Leaky Bucket)

토큰 버킷과 반대 관점입니다. 요청이 버킷(큐)에 들어오고, 버킷에서는 **일정한 속도로** 요청이 처리됩니다. 버킷이 가득 차면 새 요청을 버립니다.

```python
import time
import threading
from collections import deque

class LeakyBucket:
    def __init__(self, capacity: int, leak_rate: float):
        """
        capacity: 큐 최대 크기
        leak_rate: 초당 처리되는 요청 수
        """
        self.capacity = capacity
        self.leak_rate = leak_rate
        self.queue: deque = deque()
        self.last_leak = time.monotonic()
        self.lock = threading.Lock()

    def _leak(self):
        now = time.monotonic()
        elapsed = now - self.last_leak
        leaked = int(elapsed * self.leak_rate)
        if leaked > 0:
            for _ in range(min(leaked, len(self.queue))):
                self.queue.popleft()
            self.last_leak = now

    def add_request(self, request) -> bool:
        with self.lock:
            self._leak()
            if len(self.queue) < self.capacity:
                self.queue.append(request)
                return True  # 큐에 추가됨 (나중에 처리됨)
            return False  # 버킷 가득 참 → 버림

# 누출 버킷은 출력이 항상 일정 → 다운스트림 서비스를 보호
bucket = LeakyBucket(capacity=5, leak_rate=1.0)
for i in range(8):
    result = bucket.add_request(f"req-{i}")
    print(f"요청 {i+1}: {'큐 추가' if result else '드롭'}")
```

**토큰 버킷 vs 누출 버킷의 핵심 차이**:
- 토큰 버킷: 버스트를 허용, 평균 속도를 제한
- 누출 버킷: 버스트를 거부, 출력 속도를 일정하게 유지

누출 버킷은 비디오 스트리밍, 네트워크 트래픽 쉐이핑처럼 **일정한 처리율이 중요한** 상황에 적합합니다.

## 알고리즘 비교 요약

| 알고리즘 | 버스트 허용 | 메모리 | 구현 복잡도 | 대표 사용처 |
|--------|-----------|-------|----------|-----------|
| 고정 윈도우 | △ (경계 문제) | 낮음 | 매우 쉬움 | 단순 API 보호 |
| 슬라이딩 윈도우 로그 | ✗ | 높음 | 보통 | 정확한 요청 추적 |
| 토큰 버킷 | ✓ | 낮음 | 보통 | AWS, GitHub API |
| 누출 버킷 | ✗ | 보통 | 보통 | 네트워크 장비, 스트리밍 |

## 분산 환경에서의 레이트 리미팅

단일 서버에서는 메모리 내 카운터로 충분하지만, 여러 서버가 클러스터를 이루는 분산 환경에서는 **공유 저장소**가 필요합니다. Redis가 사실상 표준입니다.

```python
import redis
import time

class RedisTokenBucket:
    def __init__(self, redis_client, key_prefix: str, 
                 capacity: int, refill_rate: float):
        self.redis = redis_client
        self.key_prefix = key_prefix
        self.capacity = capacity
        self.refill_rate = refill_rate

    def consume(self, user_id: str) -> bool:
        key = f"{self.key_prefix}:{user_id}"
        now = time.time()
        
        # Lua 스크립트로 원자적 실행 (경쟁 조건 방지)
        lua_script = """
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local refill_rate = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])
        
        local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
        local tokens = tonumber(bucket[1]) or capacity
        local last_refill = tonumber(bucket[2]) or now
        
        local elapsed = now - last_refill
        tokens = math.min(capacity, tokens + elapsed * refill_rate)
        
        if tokens >= 1 then
            tokens = tokens - 1
            redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
            redis.call('EXPIRE', key, 3600)
            return 1
        end
        return 0
        """
        result = self.redis.eval(lua_script, 1, key, 
                                  self.capacity, self.refill_rate, now)
        return result == 1
```

Redis의 **원자적 Lua 스크립트** 실행이 핵심입니다. HMGET → 계산 → HMSET 사이에 다른 요청이 끼어들면 토큰이 중복 소비될 수 있기 때문입니다.

## 주의사항 및 실전 팁

**1. 식별자 설계**: IP 기반 제한은 NAT 뒤의 다수 사용자를 한 명으로 묶어버립니다. API 키, 사용자 ID, 조직 ID를 복합적으로 사용하세요.

**2. 헤더로 상태 노출**: 클라이언트가 남은 할당량을 알 수 있도록 응답 헤더에 정보를 담으세요.
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1719302400
Retry-After: 30
```

**3. 429 응답 코드 사용**: "Too Many Requests"를 의미하는 HTTP 429를 반환하고, `Retry-After` 헤더로 재시도 시점을 알려주세요.

**4. 계층별 제한 고려**: 요청 수뿐 아니라 데이터량(바이트), 연산 비용(credits) 기준 제한도 검토하세요. LLM API의 토큰 기반 과금이 좋은 예입니다.

**5. 화이트리스트/블랙리스트**: 내부 서비스나 모니터링 시스템은 제한에서 제외하고, 알려진 악성 IP는 즉시 차단하세요.

레이트 리미팅은 단순한 방어 수단이 아니라 **공정한 서비스 운영**을 위한 핵심 인프라입니다. 서비스 특성에 맞는 알고리즘을 선택하고, 분산 환경에서의 원자성을 반드시 보장하세요.

## 참고 자료
- [From Token Bucket to Sliding Window: Pick the Perfect Rate Limiting Algorithm - API7.ai](https://api7.ai/blog/rate-limiting-guide-algorithms-best-practices)
- [Rate Limiting Algorithms: Token Bucket vs Sliding Window vs Fixed Window - Arcjet](https://blog.arcjet.com/rate-limiting-algorithms-token-bucket-vs-sliding-window-vs-fixed-window/)
- [Diagramming System Design: Rate Limiters - Codesmith](https://codesmith.io/blog/diagramming-system-design-rate-limiters)
- [Rate Limiting Algorithms Compared - Medium](https://medium.com/@erwindev/rate-limiting-algorithms-compared-token-bucket-leaky-bucket-and-sliding-window-log-acd9c44bc86f)
