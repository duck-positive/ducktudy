---
layout: post
title: "Free Monad 완전 정복: 함수형 DSL과 인터프리터를 완전히 분리하는 고급 패턴"
date: 2026-11-23
categories: [cs, computer-science]
tags: [free-monad, functional-programming, haskell, dsl, monad, interpreter-pattern, scala]
---

## 개념 설명

Free Monad는 함수형 프로그래밍에서 **프로그램의 구조(AST)와 실행 방식(인터프리터)을 완전히 분리**할 수 있게 해주는 고급 패턴입니다. 임의의 Functor `f`에 대해 최소한의 모나드 구조를 추가하여, `f`로 표현된 연산들을 do-notation이나 flatMap으로 조합할 수 있게 합니다.

### 수학적 정의

Free Monad의 Haskell 정의는 다음과 같습니다.

```haskell
data Free f a
  = Pure a               -- 계산이 완료된 순수한 값
  | Roll (f (Free f a))  -- Functor f로 감싼 계속(continuation)
```

`Free f a`는 `f`가 Functor일 때 자동으로 Monad가 됩니다.

```haskell
instance Functor f => Functor (Free f) where
  fmap g (Pure a)  = Pure (g a)
  fmap g (Roll fa) = Roll (fmap (fmap g) fa)

instance Functor f => Applicative (Free f) where
  pure = Pure
  (<*>) = ap

instance Functor f => Monad (Free f) where
  return = Pure
  Pure a  >>= k = k a
  Roll fa >>= k = Roll (fmap (>>= k) fa)
```

이 구조는 사실상 **연산들의 트리(AST)** 를 만들어냅니다. `Pure`는 트리의 잎(leaf)이고, `Roll`은 내부 노드로서 다음 단계의 계산을 담은 `f`를 포장합니다.

### "Free"의 의미

대수학에서 "자유(free)"는 "최소한의 법칙만 만족하는"을 의미합니다. Free Monad는 모나드 법칙을 만족하는 가장 단순한 구조로, 연산의 의미(semantics)를 나중에 인터프리터가 결정합니다. 이는 프리 그룹(free group)이 군 법칙만 만족하는 것과 같은 개념입니다.

---

## 왜 필요한가

### 문제 상황: IO와 비즈니스 로직의 결합

전통적인 명령형 코드에서는 비즈니스 로직이 IO 연산과 뒤섞여 테스트가 어렵습니다.

```python
# 테스트하기 어려운 코드
def process_user(user_id: int):
    user = db.query(f"SELECT * FROM users WHERE id={user_id}")  # DB 접근
    if user['age'] >= 18:
        email_service.send(user['email'], "Welcome!")           # 이메일 전송
        log_service.log(f"User {user_id} welcomed")            # 로그 기록
    return user
```

### Free Monad의 해법

연산을 **대수(algebra)**로 정의하고, 실행을 나중으로 미룹니다.

1. **프로그램 정의**: 연산들을 조합하여 AST 구축
2. **인터프리터**: AST를 실제 실행으로 변환 (실제 DB, 테스트 목업, 로깅 등)

---

## 실제 구현 예제

### 예제 1: Haskell - 파일시스템 DSL

```haskell
{-# LANGUAGE DeriveFunctor #-}
module FileSystemDSL where

import Control.Monad.Free

-- 1단계: 연산(Functor) 정의
data FileSystemF next
  = ReadFile FilePath (String -> next)   -- 파일 읽기: 결과를 next에 전달
  | WriteFile FilePath String next        -- 파일 쓰기
  | DeleteFile FilePath next             -- 파일 삭제
  | ListDir FilePath ([FilePath] -> next) -- 디렉토리 목록
  deriving (Functor)

-- 2단계: Free Monad 타입 별칭
type FileSystem a = Free FileSystemF a

-- 3단계: 스마트 생성자 (lift 함수들)
readFile' :: FilePath -> FileSystem String
readFile' path = liftF $ ReadFile path id

writeFile' :: FilePath -> String -> FileSystem ()
writeFile' path content = liftF $ WriteFile path content ()

deleteFile' :: FilePath -> FileSystem ()
deleteFile' path = liftF $ DeleteFile path ()

listDir' :: FilePath -> FileSystem [FilePath]
listDir' path = liftF $ ListDir path id

-- 4단계: 프로그램 작성 (do-notation)
processConfig :: FileSystem String
processConfig = do
  files <- listDir' "/etc/myapp"
  let configFile = "/etc/myapp/config.json"
  if configFile `elem` files
    then do
      content <- readFile' configFile
      writeFile' "/tmp/config_backup.json" content
      return content
    else do
      let defaultConfig = "{\"version\": 1}"
      writeFile' configFile defaultConfig
      return defaultConfig

-- 5단계: 인터프리터 1 - 실제 IO 실행
interpretIO :: FileSystem a -> IO a
interpretIO (Pure a) = return a
interpretIO (Roll (ReadFile path k)) = do
  content <- readFile path
  interpretIO (k content)
interpretIO (Roll (WriteFile path content next)) = do
  writeFile path content
  interpretIO next
interpretIO (Roll (DeleteFile path next)) = do
  removeFile path
  interpretIO next
interpretIO (Roll (ListDir path k)) = do
  entries <- listDirectory path
  interpretIO (k entries)

-- 5단계: 인터프리터 2 - 테스트용 순수 실행
import qualified Data.Map.Strict as Map

type MockFS = Map.Map FilePath String

interpretPure :: FileSystem a -> MockFS -> (a, MockFS, [String])
interpretPure (Pure a) fs = (a, fs, [])
interpretPure (Roll (ReadFile path k)) fs =
  let content = Map.findWithDefault "" path fs
  in interpretPure (k content) fs
interpretPure (Roll (WriteFile path content next)) fs =
  let (a, fs', logs) = interpretPure next (Map.insert path content fs)
  in (a, fs', ("WRITE: " ++ path) : logs)
interpretPure (Roll (DeleteFile path next)) fs =
  let (a, fs', logs) = interpretPure next (Map.delete path fs)
  in (a, fs', ("DELETE: " ++ path) : logs)
interpretPure (Roll (ListDir path k)) fs =
  let entries = Map.keys $ Map.filterWithKey (\k _ -> path `isPrefixOf` k) fs
  in interpretPure (k entries) fs

-- 테스트 실행
testProcessConfig :: IO ()
testProcessConfig = do
  let mockFS = Map.fromList [("/etc/myapp/config.json", "{\"version\": 2}")]
      (result, finalFS, logs) = interpretPure processConfig mockFS
  putStrLn $ "Result: " ++ result
  putStrLn $ "Logs: " ++ show logs
  putStrLn $ "Final FS: " ++ show finalFS
```

이 코드에서 `processConfig`는 **순수한 값**입니다. `IO` 모나드에 묶이지 않고, 실제 파일 시스템을 전혀 건드리지 않습니다. 같은 프로그램을 `interpretIO`로 실제 실행하거나 `interpretPure`로 테스트할 수 있습니다.

### 예제 2: Python으로 구현하는 Free Monad 패턴

Python에는 타입 클래스가 없지만, OOP로 Free Monad의 핵심 개념을 구현할 수 있습니다.

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Any, Callable, Generic, TypeVar

A = TypeVar('A')
B = TypeVar('B')

# Free Monad 구조 (Python OOP 버전)
class Free(ABC, Generic[A]):
    @abstractmethod
    def flat_map(self, f: Callable[[A], 'Free[B]']) -> 'Free[B]':
        ...

    def map(self, f: Callable[[A], B]) -> 'Free[B]':
        return self.flat_map(lambda a: Pure(f(a)))

    def __rshift__(self, other: Callable[[A], 'Free[B]']) -> 'Free[B]':
        return self.flat_map(other)


@dataclass
class Pure(Free[A]):
    value: A

    def flat_map(self, f: Callable[[A], Free[B]]) -> Free[B]:
        return f(self.value)


@dataclass
class Roll(Free[A]):
    operation: Any  # Functor 역할

    def flat_map(self, f: Callable[[A], Free[B]]) -> Free[B]:
        # operation 안의 Free를 flat_map으로 감싸기
        return Roll(self.operation.map(lambda next_free: next_free.flat_map(f)))


# DB DSL 연산 정의
class DbOp(ABC, Generic[A]):
    @abstractmethod
    def map(self, f: Callable) -> 'DbOp':
        ...

@dataclass
class Query(DbOp[A]):
    sql: str
    cont: Callable[[list], A]

    def map(self, f):
        return Query(self.sql, lambda rows: f(self.cont(rows)))

@dataclass
class Execute(DbOp[A]):
    sql: str
    next: A

    def map(self, f):
        return Execute(self.sql, f(self.next))


# 스마트 생성자
def query(sql: str) -> Free[list]:
    return Roll(Query(sql, lambda rows: Pure(rows)))

def execute(sql: str) -> Free[None]:
    return Roll(Execute(sql, Pure(None)))


# 프로그램 구성
def get_active_users_and_notify() -> Free[list]:
    return (
        query("SELECT id, email FROM users WHERE active = 1")
        >> (lambda users:
            execute(f"INSERT INTO logs VALUES ('fetched {len(users)} users')")
            >> (lambda _:
                Pure(users)
            )
        )
    )


# 인터프리터 1: 실제 DB (psycopg2 등 사용 가정)
def interpret_real(program: Free, conn) -> Any:
    if isinstance(program, Pure):
        return program.value
    op = program.operation
    if isinstance(op, Query):
        cursor = conn.cursor()
        cursor.execute(op.sql)
        rows = cursor.fetchall()
        return interpret_real(op.cont(rows), conn)
    elif isinstance(op, Execute):
        cursor = conn.cursor()
        cursor.execute(op.sql)
        conn.commit()
        return interpret_real(op.next, conn)


# 인터프리터 2: 테스트용 목업
def interpret_test(program: Free, mock_db: dict) -> tuple[Any, list[str]]:
    logs = []

    def go(prog):
        if isinstance(prog, Pure):
            return prog.value
        op = prog.operation
        if isinstance(op, Query):
            rows = mock_db.get(op.sql, [])
            logs.append(f"QUERY: {op.sql} -> {len(rows)} rows")
            return go(op.cont(rows))
        elif isinstance(op, Execute):
            logs.append(f"EXECUTE: {op.sql}")
            return go(op.next)

    result = go(program)
    return result, logs


# 테스트
if __name__ == "__main__":
    mock_db = {
        "SELECT id, email FROM users WHERE active = 1": [
            (1, "alice@example.com"),
            (2, "bob@example.com"),
        ]
    }
    
    program = get_active_users_and_notify()
    result, logs = interpret_test(program, mock_db)
    
    print("Result:", result)
    print("Execution log:")
    for log in logs:
        print(" -", log)
```

출력 결과:
```
Result: [(1, 'alice@example.com'), (2, 'bob@example.com')]
Execution log:
 - QUERY: SELECT id, email FROM users WHERE active = 1 -> 2 rows
 - EXECUTE: INSERT INTO logs VALUES ('fetched 2 users')
```

---

## 주의사항과 팁

### 1. 스택 오버플로우 위험

순진한 Free Monad 구현에서 `>>=`는 왼쪽 연관적으로 쌓이면 `O(n)`의 중첩된 `fmap` 호출이 발생합니다.

```
((m >>= f) >>= g) >>= h
```

이런 경우 Codensity Monad, Church-encoded Free Monad, 또는 `freer-simple` 패키지를 사용하면 `O(1)` `>>=`를 달성할 수 있습니다.

### 2. Tagless Final과의 비교

Free Monad의 대안으로 **Tagless Final** 패턴이 있습니다.

| 항목 | Free Monad | Tagless Final |
|------|-----------|---------------|
| AST 형태 | 명시적 데이터 타입 | 타입 클래스 메서드 |
| 인터프리터 추가 | 새 인터프리터 함수 | 새 인스턴스 선언 |
| 연산 추가 | 새 Functor 생성자 | 타입 클래스 확장 |
| 성능 | 중간 | 더 빠름 (JIT 최적화 용이) |
| 학습 곡선 | 중간 | 더 가파름 |

대규모 프로덕션 코드에서는 Tagless Final이 더 선호되는 추세이지만, Free Monad는 AST 직접 접근이 필요한 경우(최적화, 직렬화 등)에 여전히 유용합니다.

### 3. Operational Monad

Free Monad의 실용적 변형으로 `operational` 패키지의 Operational Monad가 있습니다. 타입 위트니스(type witness)를 사용하여 더 타입 안전한 인터프리터를 작성할 수 있습니다.

### 4. Scala에서의 활용

Scala의 `cats` 라이브러리는 `cats.free.Free`를 제공합니다. `tailRecM`을 통해 스택 안전성을 보장하며, 실제 프로덕션 코드에서 많이 사용됩니다.

---

## 참고 자료

- [ekmett/free: Free Monads in Haskell](https://github.com/ekmett/free)
- [typelevel/cats: Lightweight, modular Scala functional programming library](https://github.com/typelevel/cats)
