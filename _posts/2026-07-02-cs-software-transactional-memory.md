---
layout: post
title: "소프트웨어 트랜잭셔널 메모리(STM) 완전 정복: 락 없는 낙관적 동시성 제어"
date: 2026-07-02
categories: [cs, computer-science]
tags: [stm, concurrency, transactional-memory, haskell, clojure, multithread, lock-free, mvcc]
---

## 소프트웨어 트랜잭셔널 메모리란?

소프트웨어 트랜잭셔널 메모리(STM, Software Transactional Memory)는 **데이터베이스 트랜잭션의 ACID 개념을 공유 메모리 연산에 적용한 동시성 제어 메커니즘**입니다. 1993년 Nir Shavit과 Dan Touitou가 논문에서 처음 제안했으며, Haskell의 GHC, Clojure의 Ref, Groovy GPars, Scala Akka STM 등에서 구현되었습니다.

STM의 핵심 아이디어는 간단합니다: 공유 메모리에 대한 여러 연산을 **트랜잭션 블록으로 묶어 원자적(Atomic)으로 실행**합니다. 충돌이 없으면 그대로 커밋되고, 충돌이 감지되면 전체를 롤백하고 재시도합니다.

```
atomically $ do
    x <- readTVar account1
    y <- readTVar account2
    writeTVar account1 (x - 100)
    writeTVar account2 (y + 100)
-- 이 블록은 원자적으로 실행됨: 전부 성공하거나 전부 실패
```

## 왜 STM이 필요한가?

### 락 기반 동시성의 고통

전통적인 뮤텍스/락 기반 동시성에는 근본적인 문제가 있습니다.

**데드락(Deadlock)**:
두 스레드가 서로 상대방의 락을 기다리면 영원히 멈춥니다.

```java
// Thread A: lock1 획득 후 lock2 대기
synchronized (lock1) {
    synchronized (lock2) { /* ... */ }
}

// Thread B: lock2 획득 후 lock1 대기  ← 데드락!
synchronized (lock2) {
    synchronized (lock1) { /* ... */ }
}
```

**컴포지션 불가능성(Non-composability)**:
스레드 세이프한 두 함수 `f()`와 `g()`를 결합한 `f()+g()`가 스레드 세이프하지 않을 수 있습니다. 내부 락을 외부에서 합성하는 방법이 없기 때문입니다.

**우선순위 역전(Priority Inversion)**:
낮은 우선순위 스레드가 락을 보유하면 높은 우선순위 스레드가 블록됩니다.

**락 세분화 딜레마**:
코어스 그레인 락: 안전하지만 성능 병목  
파인 그레인 락: 성능은 좋지만 데드락과 버그 위험 급증

### STM이 제공하는 것

STM은 **낙관적 동시성 제어(Optimistic Concurrency Control)**를 사용합니다:
- 충돌이 드물다고 가정하고 락 없이 진행
- 커밋 시점에만 충돌 검사
- 충돌 시 롤백 후 재시도 (프로그래머 개입 불필요)
- **컴포지션 가능**: 두 트랜잭션을 `atomically`로 감싸면 자동으로 하나의 원자 연산이 됨

## STM의 핵심 구현 메커니즘

### TVar (Transactional Variable)

STM의 기본 단위는 **TVar(Transactional Variable)**입니다. 일반 변수 대신 TVar를 사용하면 STM 런타임이 모든 읽기/쓰기를 추적합니다.

### 트랜잭션 로그 (Transaction Log)

각 트랜잭션은 다음 두 집합을 관리합니다:

- **읽기 집합 (Read-Set)**: 읽은 TVar와 해당 시점의 버전(또는 값)
- **쓰기 집합 (Write-Set)**: 쓰기를 원하는 TVar와 새 값 (아직 실제 메모리에 반영 안 됨)

트랜잭션 내 모든 쓰기는 **로컬 Write-Set에만** 기록됩니다. 커밋 시점에 read-set의 버전이 여전히 유효한지 확인하고, 유효하면 write-set을 원자적으로 플러시합니다.

```
트랜잭션 실행 중:
  readTVar x  → Read-Set에 {x: version=5, value=100} 추가, 100 반환
  writeTVar x 200 → Write-Set에 {x: 200} 추가 (실제 메모리는 그대로 100)

커밋 시도:
  x의 현재 버전이 여전히 5인가?
  YES → Write-Set 플러시: x = 200, version = 6   [성공]
  NO  → 롤백, 트랜잭션 전체 재실행               [재시도]
```

### retry와 orElse: STM의 강력한 프리미티브

**`retry`**: 트랜잭션을 즉시 중단하고, Read-Set의 어느 TVar라도 변경될 때까지 대기 후 재시도합니다. 조건부 대기(conditional wait)를 락 없이 구현할 수 있게 해줍니다.

**`orElse`**: 첫 번째 트랜잭션이 `retry`하면 두 번째 트랜잭션을 시도합니다. 논블로킹 폴링 패턴을 쉽게 구성할 수 있습니다.

## 실제 구현 예제

### 예제 1: Haskell STM으로 은행 계좌 이체 구현

Haskell의 `Control.Concurrent.STM` 모듈은 STM의 가장 완성도 높은 구현으로 꼽힙니다.

```haskell
-- bank_account.hs
-- 컴파일: ghc -threaded -o bank bank_account.hs
-- 필요: cabal install stm

import Control.Concurrent.STM
import Control.Concurrent (forkIO, threadDelay)
import Control.Monad (replicateM_, forM_)
import Data.List (foldl')

-- 계좌 타입: TVar로 감싼 정수 잔액
type Account = TVar Int

newAccount :: Int -> IO Account
newAccount balance = newTVarIO balance

-- 안전한 이체: STM 트랜잭션으로 원자적 실행
transfer :: Account -> Account -> Int -> STM ()
transfer from to amount = do
    fromBal <- readTVar from
    if fromBal < amount
        then retry  -- 잔액 부족 시 from이 변경될 때까지 대기
        else do
            writeTVar from (fromBal - amount)
            toBal <- readTVar to
            writeTVar to   (toBal + amount)

-- 잔액 확인 (읽기 전용 트랜잭션)
getBalance :: Account -> IO Int
getBalance acc = readTVarIO acc

-- 여러 계좌의 총합이 불변인지 확인
totalBalance :: [Account] -> IO Int
totalBalance accounts = atomically $ do
    balances <- mapM readTVar accounts
    return (sum balances)

-- 데모: 여러 스레드가 동시에 이체
main :: IO ()
main = do
    -- 3개 계좌 생성: 각 1,000원
    acc1 <- newAccount 1000
    acc2 <- newAccount 1000
    acc3 <- newAccount 1000
    let accounts = [acc1, acc2, acc3]

    let initialTotal = 3000
    putStrLn $ "초기 총합: " ++ show initialTotal

    -- 1,000번 병렬 이체 (각 스레드가 랜덤 계좌 간 이체)
    let transfers =
            [ (acc1, acc2, 100), (acc2, acc3, 200)
            , (acc3, acc1, 150), (acc1, acc3, 50)
            , (acc2, acc1, 300)
            ]

    -- 병렬로 트랜잭션 실행
    forM_ (replicate 200 transfers) $ \txList ->
        mapM_ (\(f, t, a) -> forkIO $ atomically (transfer f t a)) txList

    -- 완료 대기
    threadDelay 500000  -- 0.5초

    -- 불변성 검증: 총합이 항상 3,000이어야 함
    finalTotal <- totalBalance accounts
    bal1 <- getBalance acc1
    bal2 <- getBalance acc2
    bal3 <- getBalance acc3

    putStrLn $ "최종 잔액: acc1=" ++ show bal1
                ++ ", acc2=" ++ show bal2
                ++ ", acc3=" ++ show bal3
    putStrLn $ "최종 총합: " ++ show finalTotal
    putStrLn $ "불변성 유지: " ++ show (finalTotal == initialTotal)
    -- 출력:
    -- 초기 총합: 3000
    -- 최종 잔액: acc1=..., acc2=..., acc3=...
    -- 최종 총합: 3000
    -- 불변성 유지: True
```

`retry`의 우아함이 핵심입니다: `fromBal < amount`일 때 그냥 `retry`를 호출하면 됩니다. STM 런타임이 `from` TVar가 변경될 때까지 스레드를 자동으로 블록시키고, 변경되면 트랜잭션을 재실행합니다. 명시적인 `wait()`/`notifyAll()` 없이 조건 변수를 구현한 것입니다.

### 예제 2: Python으로 STM 구현 (교육용)

Python에는 내장 STM이 없으므로, 핵심 메커니즘을 직접 구현해 원리를 이해합니다.

```python
"""
stm_demo.py — Python으로 STM 핵심 메커니즘 구현 (교육 목적)
실제 프로덕션에서는 Haskell STM, Clojure Ref, 또는 PySTM 라이브러리 사용 권장
"""
import threading
from typing import Any, Dict, Set
from contextlib import contextmanager

_transaction_local = threading.local()

class TVar:
    """STM의 기본 단위 — Transactional Variable"""
    _id_counter = 0
    _lock = threading.Lock()

    def __init__(self, initial_value: Any):
        with TVar._lock:
            self.id = TVar._id_counter
            TVar._id_counter += 1
        self._value = initial_value
        self._version = 0
        self._lock = threading.Lock()

    def _read_committed(self):
        """커밋된 값과 버전 읽기 (락 보유 필요)"""
        return self._value, self._version

    def _commit_write(self, new_value: Any) -> None:
        """새 값 커밋 (전역 커밋 락 보유 상태에서 호출)"""
        self._value = new_value
        self._version += 1


class STMTransaction:
    """단일 STM 트랜잭션 — read-set과 write-set 관리"""

    def __init__(self):
        self.read_set: Dict[int, tuple] = {}   # tvar.id -> (version, value)
        self.write_set: Dict[int, Any] = {}    # tvar.id -> new_value
        self._tvars: Dict[int, TVar] = {}

    def read(self, tvar: TVar) -> Any:
        """TVar 읽기 — write-set 우선, 없으면 커밋된 값"""
        self._tvars[tvar.id] = tvar
        if tvar.id in self.write_set:
            return self.write_set[tvar.id]
        with tvar._lock:
            value, version = tvar._read_committed()
        self.read_set[tvar.id] = (version, value)
        return value

    def write(self, tvar: TVar, value: Any) -> None:
        """TVar 쓰기 — write-set에만 기록 (아직 실제 메모리 반영 안 됨)"""
        self._tvars[tvar.id] = tvar
        self.write_set[tvar.id] = value

    def try_commit(self) -> bool:
        """
        커밋 시도:
        1. read-set의 모든 TVar 버전이 변하지 않았는지 확인
        2. 유효하면 write-set을 원자적으로 플러시
        3. 버전 충돌이면 False 반환
        """
        # 모든 관련 TVar의 락을 정렬된 순서로 획득 (데드락 방지)
        sorted_ids = sorted(self._tvars.keys())
        locks = [self._tvars[tid]._lock for tid in sorted_ids]

        # 모든 락 획득
        for lock in locks:
            lock.acquire()

        try:
            # 유효성 검사: read-set 버전 확인
            for tvar_id, (recorded_version, _) in self.read_set.items():
                tvar = self._tvars[tvar_id]
                if tvar._version != recorded_version:
                    return False  # 충돌 감지

            # write-set 커밋
            for tvar_id, new_value in self.write_set.items():
                self._tvars[tvar_id]._commit_write(new_value)

            return True
        finally:
            for lock in reversed(locks):
                lock.release()


_GLOBAL_COMMIT_LOCK = threading.Lock()

def atomically(func, *args, **kwargs):
    """
    func를 STM 트랜잭션으로 실행.
    충돌 시 자동 재시도 (최대 100회).
    """
    max_retries = 100
    for attempt in range(max_retries):
        txn = STMTransaction()
        _transaction_local.current = txn
        try:
            result = func(txn, *args, **kwargs)
        finally:
            _transaction_local.current = None

        if txn.try_commit():
            return result
        # 충돌 → 재시도 (지수 백오프 생략, 교육용 단순화)

    raise RuntimeError(f"트랜잭션이 {max_retries}회 재시도 후 실패")


# 사용 예시: 은행 이체
def transfer(txn: STMTransaction, from_acc: TVar, to_acc: TVar, amount: int):
    from_bal = txn.read(from_acc)
    to_bal = txn.read(to_acc)
    if from_bal < amount:
        raise ValueError(f"잔액 부족: {from_bal} < {amount}")
    txn.write(from_acc, from_bal - amount)
    txn.write(to_acc,   to_bal + amount)
    return from_bal - amount, to_bal + amount


if __name__ == "__main__":
    acc_a = TVar(5000)
    acc_b = TVar(3000)

    def do_transfer(from_acc, to_acc, amount):
        atomically(transfer, from_acc, to_acc, amount)

    threads = []
    for i in range(50):
        t = threading.Thread(target=do_transfer, args=(acc_a, acc_b, 10))
        threads.append(t)
    for i in range(30):
        t = threading.Thread(target=do_transfer, args=(acc_b, acc_a, 15))
        threads.append(t)

    for t in threads:
        t.start()
    for t in threads:
        t.join()

    final_a = acc_a._value
    final_b = acc_b._value
    print(f"최종 잔액: A={final_a}, B={final_b}")
    print(f"총합 불변: {final_a + final_b == 8000} (초기 총합: 8000)")
    # 기대: 총합이 항상 8000 유지
```

## 주의사항과 실전 팁

### 1. 재시도 폭탄 (Retry Storm) 방지

트랜잭션 충돌이 많으면 재시도가 기하급수적으로 늘어납니다. **짧고 빠른 트랜잭션**을 작성하고, 트랜잭션 내에서 I/O나 긴 연산을 피해야 합니다.

```haskell
-- 나쁜 예: 트랜잭션 내 I/O
atomically $ do
    result <- readTVar sharedData
    unsafePerformIO (writeFile "log.txt" result)  -- 절대 금지!

-- 좋은 예: I/O를 트랜잭션 밖으로
result <- atomically (readTVar sharedData)
writeFile "log.txt" result
```

### 2. 부작용(Side Effects) 금지

STM 트랜잭션은 실패 시 재실행됩니다. 트랜잭션 내 부작용(파일 I/O, 네트워크, 프린트)은 여러 번 실행될 수 있습니다. Haskell의 타입 시스템(`STM` 모나드)이 이를 컴파일 타임에 강제합니다.

### 3. STM vs 락 성능 비교

| 시나리오 | STM | 락 |
|----------|-----|-----|
| 충돌 없음 (읽기 위주) | 우수 | 보통 |
| 충돌 적음 | 우수 | 보통 |
| 충돌 많음 | 나쁨 (재시도 폭풍) | 보통 |
| 쓰기 집약적, 고경합 | 나쁨 | 좋음 |

STM은 **읽기 위주 워크로드**나 **충돌이 드문 환경**에서 가장 효과적입니다.

### 4. Clojure Ref와 dosync

Clojure는 MVCC 기반 STM을 내장합니다:

```clojure
(def account-a (ref 5000))
(def account-b (ref 3000))

;; dosync 내에서만 ref 수정 가능
(defn transfer [from to amount]
  (dosync
    (when (< @from amount)
      (throw (Exception. "잔액 부족")))
    (alter from - amount)
    (alter to   + amount)))

;; 병렬 이체 (충돌 시 자동 재시도)
(pmap #(transfer account-a account-b %) (repeat 100 10))
(println "총합:" (+ @account-a @account-b))  ; 항상 8000
```

### 5. 실무 선택 기준

- **Haskell**: 가장 완성도 높은 STM (`Control.Concurrent.STM`)
- **Clojure**: JVM 생태계 + STM이 필요할 때
- **Scala (Akka STM)**: 분산 STM이 필요할 때 (단, Akka STM은 현재 별도 라이브러리)
- **C++ (Intel TBB)**: 하드웨어 트랜잭셔널 메모리(HTM)와 연계 가능

## 참고 자료

- [Haskell STM 공식 위키](https://wiki.haskell.org/Software_transactional_memory)
- [Wikipedia: Software transactional memory](https://en.wikipedia.org/wiki/Software_transactional_memory)
- [STM 심화 — 논문 (Shavit & Touitou, 1995)](https://dl.acm.org/doi/10.1145/224964.224987)
- [Clojure Refs와 STM 공식 문서](https://clojure.org/reference/refs)
