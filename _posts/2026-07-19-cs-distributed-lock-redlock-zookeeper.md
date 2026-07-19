---
layout: post
title: "분산 락(Distributed Lock) 완전 정복: Redlock, ZooKeeper, Fencing Token으로 분산 시스템 상호 배제 구현하기"
date: 2026-07-19
categories: [cs, computer-science]
tags: [distributed-lock, redlock, zookeeper, fencing-token, redis, distributed-systems, concurrency]
---

## 개념 설명

분산 시스템에서는 여러 노드가 동시에 같은 자원(DB 레코드, 파일, API 엔드포인트)에 접근하는 상황이 발생합니다. 단일 프로세스라면 `synchronized`, `mutex`, `ReentrantLock` 등으로 해결할 수 있지만, 서로 다른 서버에서 실행되는 프로세스들은 공유 메모리가 없어 이런 기본 동기화 도구를 사용할 수 없습니다.

**분산 락(Distributed Lock)** 은 여러 노드가 공유 자원에 동시에 접근하지 못하도록 제어하는 메커니즘입니다.

### 올바른 분산 락이 만족해야 하는 세 가지 속성

1. **상호 배제(Mutual Exclusion)**: 임의의 시점에 오직 하나의 클라이언트만 락을 보유할 수 있습니다.
2. **데드락 프리(Deadlock-free)**: 락을 획득한 클라이언트가 크래시되더라도 다른 클라이언트가 결국 락을 획득할 수 있어야 합니다.
3. **내결함성(Fault-tolerant)**: 락 서버 일부가 다운되더라도 락이 동작해야 합니다.

### 분산 락이 어려운 이유: GC 멈춤과 네트워크 지연

분산 락이 단순해 보이지만 실제로는 까다롭습니다. 아래 시나리오를 살펴보세요.

```
Client 1이 락 획득
  → Client 1이 GC 멈춤으로 수십 초간 정지
  → 락의 TTL이 만료, 락 해제
  → Client 2가 락 획득
  → Client 1이 깨어나 자신이 락을 보유한다고 착각하고 자원에 쓰기
  → Client 1과 Client 2 동시에 자원 변경!
```

이 문제를 해결하는 핵심 개념이 **Fencing Token**입니다.

---

## 왜 필요한가?

### 분산 락이 필요한 실제 시나리오

1. **결제 처리 중복 방지**: 동일한 주문이 두 번 결제되지 않도록
2. **재고 감소 원자성 보장**: 재고 조회와 감소를 하나의 원자적 연산으로
3. **크론 잡(Cron Job) 중복 실행 방지**: 다중 서버 환경에서 동일한 배치 작업이 한 번만 실행되도록
4. **리더 선출(Leader Election)**: 분산 시스템에서 단 하나의 리더 노드 선정

### Redis 기반 단순 락의 문제점

Redis의 `SET key value NX PX <ttl>` 명령으로 간단히 락을 구현할 수 있지만, **단일 Redis 인스턴스**를 사용하면 Redis가 장애 시 전체 락 시스템이 마비됩니다.

---

## 실제 구현 예제

### 예제 1: Redis 단일 인스턴스 분산 락 (Python)

```python
import redis
import uuid
import time
from contextlib import contextmanager

class RedisLock:
    """
    Redis 단일 인스턴스 기반 분산 락.
    단일 Redis 장애 시 락이 동작하지 않으므로
    프로덕션 환경에서는 Redlock 또는 ZooKeeper를 권장.
    """

    LOCK_SCRIPT = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    """

    def __init__(self, redis_client: redis.Redis, key: str, ttl_ms: int = 30000):
        self.redis = redis_client
        self.key = f"lock:{key}"
        self.ttl_ms = ttl_ms
        self.token = None
        self._unlock_script = redis_client.register_script(self.LOCK_SCRIPT)

    def acquire(self, retry_count: int = 3, retry_delay_ms: int = 200) -> bool:
        self.token = str(uuid.uuid4())
        for attempt in range(retry_count):
            # NX: 키가 없을 때만 설정, PX: 만료 시간(밀리초)
            result = self.redis.set(
                self.key, self.token,
                nx=True, px=self.ttl_ms
            )
            if result:
                return True
            if attempt < retry_count - 1:
                time.sleep(retry_delay_ms / 1000)
        return False

    def release(self) -> bool:
        if self.token is None:
            return False
        # Lua 스크립트로 get + delete를 원자적으로 실행
        # 자신의 토큰인 경우에만 삭제 (다른 클라이언트의 락을 실수로 해제하지 않음)
        result = self._unlock_script(keys=[self.key], args=[self.token])
        self.token = None
        return bool(result)

    @contextmanager
    def lock(self, **kwargs):
        acquired = self.acquire(**kwargs)
        if not acquired:
            raise RuntimeError(f"Failed to acquire lock: {self.key}")
        try:
            yield self
        finally:
            self.release()


# 사용 예시
r = redis.Redis(host='localhost', port=6379, db=0)
lock = RedisLock(r, "order:12345", ttl_ms=10000)

with lock.lock():
    # 임계 구역: 이 블록은 한 번에 하나의 프로세스만 실행
    print("결제 처리 중...")
    # process_payment(order_id=12345)
```

### 예제 2: Fencing Token 기반 안전한 분산 락 패턴

Fencing Token은 락 획득 시마다 단조 증가하는 숫자를 발급해, 오래된 클라이언트의 요청을 자원 서버(스토리지)가 직접 거부할 수 있게 합니다.

```python
import threading
import time
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class LockToken:
    token_id: str
    fence: int          # 단조 증가하는 fencing 번호
    expires_at: float

class FencedResourceServer:
    """
    Fencing Token을 검증하는 자원 서버 예시.
    저장소 레이어에서 오래된 클라이언트 요청을 거부한다.
    """
    def __init__(self):
        self._data: dict[str, str] = {}
        self._max_fence: dict[str, int] = {}  # key -> 지금까지 본 최대 fence

    def write(self, key: str, value: str, lock_token: LockToken) -> bool:
        current_max = self._max_fence.get(key, -1)

        # 오래된 fence 번호는 거부
        if lock_token.fence <= current_max:
            print(f"[거부] fence={lock_token.fence} <= max_seen={current_max}")
            return False

        # 만료된 토큰도 거부
        if time.time() > lock_token.expires_at:
            print(f"[거부] 토큰 만료")
            return False

        self._max_fence[key] = lock_token.fence
        self._data[key] = value
        print(f"[성공] fence={lock_token.fence}으로 '{key}'에 '{value}' 기록")
        return True


class FencingLockManager:
    """단조 증가하는 fence 번호를 발급하는 락 매니저"""
    def __init__(self):
        self._counter = 0
        self._lock = threading.Lock()

    def acquire(self, ttl_seconds: float = 30.0) -> LockToken:
        with self._lock:
            self._counter += 1
            return LockToken(
                token_id=f"token-{self._counter}",
                fence=self._counter,
                expires_at=time.time() + ttl_seconds
            )


# 시뮬레이션: GC 멈춤 후 오래된 클라이언트가 쓰기를 시도하는 경우
manager = FencingLockManager()
server = FencedResourceServer()

# Client 1이 락 획득
token1 = manager.acquire(ttl_seconds=5.0)
print(f"Client1 락 획득: fence={token1.fence}")

# Client 1이 GC로 멈추는 동안 락 만료, Client 2가 락 획득
time.sleep(6)  # 락 만료 시뮬레이션
token2 = manager.acquire(ttl_seconds=30.0)
print(f"Client2 락 획득: fence={token2.fence}")

# Client 2가 먼저 쓰기
server.write("inventory:item-1", "qty=99", token2)  # 성공

# Client 1이 깨어나 (오래된 fence로) 쓰기 시도
server.write("inventory:item-1", "qty=100", token1)  # 거부!
```

### 예제 3: Redlock 알고리즘 (5개 Redis 인스턴스)

```python
import redis
import uuid
import time
from typing import Optional

class Redlock:
    """
    Redis 공식 Redlock 알고리즘 구현.
    N개의 독립 Redis 인스턴스 중 과반수(N/2+1) 이상에서
    락을 획득해야 유효한 락으로 간주한다.
    """
    CLOCK_DRIFT_FACTOR = 0.01  # 클록 드리프트 보정

    def __init__(self, redis_nodes: list[redis.Redis]):
        self.nodes = redis_nodes
        self.quorum = len(redis_nodes) // 2 + 1

    def _acquire_on_node(self, node: redis.Redis, key: str,
                         token: str, ttl_ms: int) -> bool:
        try:
            return bool(node.set(key, token, nx=True, px=ttl_ms))
        except redis.RedisError:
            return False

    def _release_on_node(self, node: redis.Redis, key: str, token: str):
        script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """
        try:
            node.eval(script, 1, key, token)
        except redis.RedisError:
            pass

    def acquire(self, key: str, ttl_ms: int) -> Optional[dict]:
        token = str(uuid.uuid4())
        start_time = time.time() * 1000  # ms

        acquired_count = 0
        for node in self.nodes:
            if self._acquire_on_node(node, key, token, ttl_ms):
                acquired_count += 1

        elapsed = time.time() * 1000 - start_time
        drift = self.CLOCK_DRIFT_FACTOR * ttl_ms + 2  # ms
        validity = ttl_ms - elapsed - drift

        if acquired_count >= self.quorum and validity > 0:
            return {"key": key, "token": token, "validity_ms": validity}

        # 과반수 획득 실패: 이미 획득한 락 모두 해제
        for node in self.nodes:
            self._release_on_node(node, key, token)
        return None

    def release(self, lock_info: dict):
        for node in self.nodes:
            self._release_on_node(node, lock_info["key"], lock_info["token"])


# 사용 예시
nodes = [
    redis.Redis(host='redis1', port=6379),
    redis.Redis(host='redis2', port=6379),
    redis.Redis(host='redis3', port=6379),
    redis.Redis(host='redis4', port=6379),
    redis.Redis(host='redis5', port=6379),
]
redlock = Redlock(nodes)

lock = redlock.acquire("payment:order-9999", ttl_ms=10000)
if lock:
    try:
        print(f"락 획득 성공, 유효 시간: {lock['validity_ms']:.0f}ms")
        # 임계 구역 실행
    finally:
        redlock.release(lock)
else:
    print("락 획득 실패")
```

---

## 주의사항 및 팁

### 1. Redlock은 완벽하지 않다

Martin Kleppmann은 Redlock이 다음 가정에 의존하기 때문에 **정확성(correctness)** 이 중요한 경우 사용하면 안 된다고 경고합니다.

- NTP 클록 동기화가 완벽하게 동작한다는 가정
- 네트워크 지연이 TTL 대비 작다는 가정
- GC 멈춤이 TTL보다 짧다는 가정

이 가정들이 깨지면 두 클라이언트가 동시에 락을 보유할 수 있습니다. **금전 거래, 결제처럼 정확성이 최우선인 시스템**에서는 ZooKeeper나 etcd(Raft 기반) 락을 사용하고 Fencing Token을 함께 적용하세요.

### 2. ZooKeeper 기반 락은 왜 더 안전한가

ZooKeeper는 Sequential Ephemeral 노드와 Watch 메커니즘을 제공합니다.

```
/locks/resource_A/lock-0000000001  (Client 1 보유)
/locks/resource_A/lock-0000000002  (Client 2 대기)
/locks/resource_A/lock-0000000003  (Client 3 대기)
```

- 순번이 가장 낮은 노드의 소유자가 락을 보유합니다.
- Client 1 세션이 끊기면 ZooKeeper가 자동으로 Ephemeral 노드를 삭제합니다.
- 노드 번호가 Fencing Token의 역할을 합니다(단조 증가).

### 3. 재진입(Reentrant) 락 구현 시 주의

같은 클라이언트가 이미 보유한 락을 다시 획득하려 할 때 데드락이 발생하지 않도록, 토큰에 클라이언트 ID와 스레드 ID를 포함시켜 재진입을 허용하세요.

### 4. 락 갱신(Watchdog) 패턴

임계 구역 작업이 TTL보다 오래 걸릴 수 있다면, 별도 스레드가 주기적으로 락의 TTL을 연장(extend)하는 **워치독 패턴**을 사용하세요. Redisson(Java Redis 클라이언트)이 이 기능을 내장 지원합니다.

---

## 참고 자료

- [How to do distributed locking — Martin Kleppmann](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- [Distributed Locks with Redis — Redis 공식 문서](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
- [The Fencing Gap: Why Your Distributed Lock Isn't Safe — HackerNoon](https://hackernoon.com/the-fencing-gap-why-your-distributed-lock-isnt-safe-and-how-to-fix-it)
- [Distributed Locking: A Practical Guide — Architecture Weekly](https://www.architecture-weekly.com/p/distributed-locking-a-practical-guide)
