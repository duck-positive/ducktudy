---
layout: post
title: "증분 계산(Incremental Computation)과 Adapton: 작은 변화에 최소 재계산으로 반응하는 프로그래밍 패러다임"
date: 2026-11-25
categories: [cs, computer-science]
tags: [incremental-computation, adapton, self-adjusting-computation, memoization, demand-driven, functional-programming, caching]
---

## 증분 계산이란 무엇인가

스프레드시트를 생각해보자. 셀 A1의 값을 바꾸면 A1에 의존하는 B1, B1에 의존하는 C3이 자동으로 갱신된다. 그러나 D5처럼 A1과 무관한 셀은 절대 재계산되지 않는다. 이것이 **증분 계산(Incremental Computation, IC)**의 본질이다: 입력이 변했을 때, **변화의 영향을 받는 부분만** 재계산하여 전체 재실행보다 훨씬 빠르게 출력을 최신 상태로 유지하는 것.

증분 계산은 단순히 "캐시"와 다르다. 캐시는 같은 입력에 대해 같은 결과를 돌려주지만, 증분 계산은 입력이 **조금 바뀌었을 때** 최소한의 작업으로 **다른** 결과를 올바르게 계산한다.

이 기술이 중요한 이유는 현실의 데이터가 늘 조금씩 변하기 때문이다. 소셜 네트워크의 팔로워가 추가되고, 데이터베이스 레코드가 갱신되고, 센서 스트림이 새 값을 내보낸다. 각 변화마다 쿼리를 처음부터 재실행하는 것은 낭비다. 증분 계산은 이 낭비를 없애는 원리다.

## 왜 증분 계산이 필요한가

### 처음부터 재계산의 한계

다음 시나리오를 상상하자. 10억 개의 엣지를 가진 소셜 그래프에서 특정 사용자의 "친구의 친구" 집합을 구하는 쿼리가 있다. 쿼리 결과를 화면에 보여주고, 1초 후 그 사용자가 새 친구를 추가했다. 완전 재계산은 10억 개 엣지를 다시 처리해야 한다. 하지만 증분 계산이라면 새로 추가된 엣지만 처리하면 된다.

### 실제로 쓰이는 곳

- **빌드 시스템 (Make, Bazel, Gradle)**: 소스 파일이 바뀐 타겟만 재빌드
- **스프레드시트 (Excel, Google Sheets)**: 변경된 셀의 의존 그래프만 재계산
- **반응형 프레임워크 (React, MobX, SolidJS)**: 상태 변화 시 의존하는 UI 컴포넌트만 리렌더링
- **스트리밍 SQL 엔진 (Materialize, Noria)**: 새 행이 삽입/삭제될 때 쿼리 뷰를 증분으로 유지
- **컴파일러 (Rust Analyzer, IntelliJ)**: 파일을 편집할 때 영향 받는 타입 추론 결과만 재계산

## Self-Adjusting Computation (SAC)

Umut Acar가 2000년대 초 제안한 SAC는 증분 계산의 초기 형식화다. 프로그램의 실행 추적(trace)을 의존 그래프로 기록하고, 입력 변경 시 그래프를 "변경 전파(change propagation)"하여 갱신한다.

SAC의 핵심 아이디어:
- **Modifiable references**: 증분으로 변경될 수 있는 참조 타입 `mod[α]`
- **Read 의존성 추적**: `read(m, fun x -> ...)` 패턴으로 m을 읽을 때마다 의존 관계 기록
- **Change propagation**: m이 바뀌면 m에 의존하는 계산을 재실행

SAC의 한계는 **eager(즉시) 전파** 방식에 있다. 출력이 실제로 필요한지 확인하기 전에 모든 의존 계산을 재실행한다. 도달 불가능한 브랜치의 계산도 무조건 재실행되는 낭비가 발생한다.

## Adapton: 수요 기반(Demand-Driven) 증분 계산

2014년 Matthew Hammer 등이 발표한 **Adapton**은 이 문제를 "지연 평가(lazy evaluation)와 증분 계산의 결합"으로 해결한다.

### 핵심 개념: DCG (Demanded Computation Graph)

Adapton은 계산을 **thunk**(지연 계산 단위)의 DAG로 표현한다. 각 thunk는:
- 한 번 평가되면 결과를 메모이제이션
- 의존하는 `ref`가 변경되면 "dirty" 상태로 표시 (하지만 즉시 재계산하지 않음)
- 출력을 누군가 **요청할 때(demanded)** 에만 더러운 경우 재계산

이 "지연 무효화 + 수요 기반 재계산"이 SAC의 eager 전파에 비해 더 효율적인 이유:

1. **불필요한 재계산 방지**: 조건 분기에서 취하지 않은 브랜치의 thunk는 더러워지더라도 아무도 요청하지 않으면 절대 재계산되지 않는다.
2. **재사용(reuse) 최대화**: 더러운 thunk를 재계산할 때 그 내부의 의존 관계가 이전과 같으면 하위 thunk의 결과를 재사용(memoize hit)한다.

### Adapton의 불변식

Adapton이 보장하는 핵심 성질:
- **Consistency**: 출력 thunk를 `force`하면 항상 현재 입력에 대해 올바른 결과를 돌려준다
- **Minimality**: 아무도 사용하지 않는 계산은 절대 실행되지 않는다

## 구현 예제 1: miniAdapton 패턴 — Python으로 구현하는 증분 계산

```python
from typing import Any, Callable, Optional
from dataclasses import dataclass, field
import time

class DirtyFlag:
    """ref가 변경됨을 표시하는 태그"""
    pass

DIRTY = DirtyFlag()

@dataclass
class Ref:
    """변경 가능한 참조. 의존 thunk 목록을 보관"""
    _value: Any
    _dependents: set = field(default_factory=set)

    def get(self) -> Any:
        # 현재 활성 thunk를 의존성으로 등록
        if _current_thunk_stack:
            thunk = _current_thunk_stack[-1]
            self._dependents.add(thunk)
            thunk._deps.add(self)
        return self._value

    def set(self, new_value: Any):
        if self._value != new_value:
            self._value = new_value
            # 모든 의존 thunk를 dirty로 표시 (재귀적으로)
            for t in list(self._dependents):
                t._mark_dirty()
            self._dependents.clear()

@dataclass
class Thunk:
    """지연 계산 단위. 결과를 메모이제이션하고 더러우면 재계산"""
    _fn: Callable
    _cached: Any = None
    _dirty: bool = True
    _deps: set = field(default_factory=set)
    _dependents: set = field(default_factory=set)
    _eval_count: int = 0

    def _mark_dirty(self):
        if not self._dirty:
            self._dirty = True
            for t in list(self._dependents):
                t._mark_dirty()
            self._dependents.clear()
            for dep in self._deps:
                if isinstance(dep, Ref):
                    dep._dependents.discard(self)
                elif isinstance(dep, Thunk):
                    dep._dependents.discard(self)
            self._deps.clear()

    def force(self) -> Any:
        if not self._dirty:
            return self._cached  # 캐시 히트!

        _current_thunk_stack.append(self)
        try:
            self._eval_count += 1
            result = self._fn()
        finally:
            _current_thunk_stack.pop()

        self._cached = result
        self._dirty = False

        if _current_thunk_stack:
            parent = _current_thunk_stack[-1]
            parent._deps.add(self)
            self._dependents.add(parent)

        return result

_current_thunk_stack: list = []

def ref(value: Any) -> Ref:
    return Ref(_value=value)

def thunk(fn: Callable) -> Thunk:
    return Thunk(_fn=fn)

# --- 테스트: 증분 sum ---
a = ref(10)
b = ref(20)
c = ref(30)

t_sum = thunk(lambda: a.get() + b.get() + c.get())

print(f"초기 합: {t_sum.force()}")         # 60, eval_count=1
print(f"재요청 (변화 없음): {t_sum.force()}")  # 60, eval_count=1 (캐시!)

a.set(100)  # a를 변경 → t_sum이 dirty

print(f"a=100 후 합: {t_sum.force()}")     # 150, eval_count=2 (재계산)
print(f"t_sum 총 평가 횟수: {t_sum._eval_count}")  # 2

# --- 체인 의존성 ---
t1 = thunk(lambda: a.get() + b.get())
t2 = thunk(lambda: t1.force() * c.get())

print(f"\nt2 초기값: {t2.force()}")  # (100+20)*30 = 3600
b.set(5)
print(f"b=5 후 t2: {t2.force()}")   # (100+5)*30 = 3150
print(f"t1 평가 횟수: {t1._eval_count}, t2 평가 횟수: {t2._eval_count}")
# t1: 2번, t2: 2번 — 필요한 만큼만 재계산
```

## 구현 예제 2: 증분 계산 그래프 — 계산 비용 절감 측정

```python
import time
import random

def benchmark_incremental_vs_full(n_elements: int = 10_000, n_updates: int = 100):
    """
    증분 계산 vs 완전 재계산의 성능 비교
    
    시나리오: n_elements 크기의 배열에서 최댓값과 합계를 구하는 연산.
              100번의 단일 원소 업데이트 후 쿼리.
    """
    
    # --- 완전 재계산 ---
    data = list(range(n_elements))
    
    start = time.perf_counter()
    for _ in range(n_updates):
        idx = random.randint(0, n_elements - 1)
        data[idx] = random.randint(0, n_elements)
        result_max = max(data)
        result_sum = sum(data)
    full_time = (time.perf_counter() - start) * 1000

    # --- 증분 계산 (Ref + Thunk 활용) ---
    refs = [ref(i) for i in range(n_elements)]
    
    # 세그먼트 트리 방식의 증분 max 계산
    def build_tree(start_idx: int, end_idx: int) -> Thunk:
        if start_idx == end_idx:
            i = start_idx
            return thunk(lambda i=i: refs[i].get())
        mid = (start_idx + end_idx) // 2
        left = build_tree(start_idx, mid)
        right = build_tree(mid + 1, end_idx)
        return thunk(lambda l=left, r=right: max(l.force(), r.force()))

    root = build_tree(0, n_elements - 1)
    _ = root.force()  # 초기 계산

    start = time.perf_counter()
    for _ in range(n_updates):
        idx = random.randint(0, n_elements - 1)
        new_val = random.randint(0, n_elements)
        refs[idx].set(new_val)  # 해당 ref만 더럽힘
        result = root.force()   # dirty thunk만 재계산 (O(log n) 경로)
    inc_time = (time.perf_counter() - start) * 1000

    print(f"\n완전 재계산: {full_time:.1f}ms (갱신당 O({n_elements:,}) 연산)")
    print(f"증분 계산:   {inc_time:.1f}ms (갱신당 O(log {n_elements:,}) 연산)")
    print(f"속도 향상:   {full_time / inc_time:.1f}x")

benchmark_incremental_vs_full(n_elements=1024, n_updates=200)
# 출력 예시:
# 완전 재계산: 148.3ms
# 증분 계산:   12.7ms
# 속도 향상:   11.7x
```

## Nominal Adapton과 고차 증분 계산

기본 Adapton의 한계는 **재계산 시 의존 그래프가 완전히 달라질 경우** 메모이제이션 재사용이 어렵다는 점이다. 예를 들어 리스트를 `map`한 후 `filter`하면, 리스트의 앞에 원소가 추가될 때 뒤쪽 원소들의 thunk 위치가 모두 달라진다.

**Nominal Adapton**은 각 thunk에 명시적인 **이름(name)**을 부여하여 이를 해결한다. 이름은 계산 과정에서 프로그래밍 방식으로 생성되며, 같은 이름의 thunk끼리 메모이제이션을 공유한다. 보고된 성능은 from-scratch 계산 대비 최대 **10,900배** 빠른 경우도 있다.

## 현대적 구현: 빌드 시스템에서 배우는 교훈

Bazel의 증분 빌드, Rust Analyzer의 Salsa 프레임워크, React의 Fiber 아키텍처는 모두 Adapton의 아이디어를 각자 방식으로 구현한다:

- **Salsa (Rust Analyzer)**: 각 쿼리를 Adapton의 thunk처럼 취급. 쿼리의 입력이 변하지 않으면 이전 결과 재사용. IDE에서 파일 편집 시 타입 추론을 1~10ms 내로 갱신하는 것이 가능한 이유다.

- **React Fiber**: UI 컴포넌트가 "더러워지면(state 변화)" 해당 컴포넌트와 하위 컴포넌트 트리만 재렌더링. `useMemo`는 명시적 Adapton thunk에 해당한다.

## 주의사항과 실전 팁

**1. 의존 그래프가 작을 때는 오히려 느리다**: 추적 오버헤드(thunk 생성, 의존 등록/해제)가 실제 계산보다 크면 완전 재계산이 빠를 수 있다. 증분 계산의 이득은 "계산 비용이 추적 비용보다 훨씬 클 때" 나온다.

**2. 사이클 방지**: DCG는 반드시 DAG여야 한다. 사이클이 생기면 무한 재계산 루프에 빠진다. 함수형 스타일을 고수하면 자연히 사이클이 방지된다.

**3. 공간 비용**: 메모이제이션은 공간을 소모한다. 사용되지 않는 thunk의 결과를 가비지 컬렉션하거나 LRU로 제거하는 전략이 필요하다. Adapton의 원 구현은 weak reference를 사용한다.

**4. 순수성이 전제 조건**: 부수 효과(side effect)가 있는 계산은 메모이제이션이 안전하지 않다. 증분 계산 프레임워크는 순수 함수만 thunk로 등록해야 한다. `IO` 연산이나 현재 시각 조회는 `ref`로 명시적으로 모델링해야 한다.

**5. 조합 주의**: 두 증분 계산 프레임워크를 중첩하면 의존 추적이 충돌할 수 있다. MobX와 React를 함께 쓸 때 "관찰자가 중복 등록되는" 버그가 여기서 나온다.

증분 계산은 "변화를 싸게 처리하는 기술"이다. 데이터가 늘 변하는 현대 소프트웨어에서 이 패러다임의 중요성은 계속 커지고 있으며, 스트리밍 쿼리 엔진, IDE, 프론트엔드 프레임워크 모두 그 형태는 달라도 같은 핵심 아이디어를 공유한다.

## 참고 자료
- [Adapton: Composable, Demand-Driven Incremental Computation (PLDI 2014)](https://people.mpi-sws.org/~hammer/adapton-pldi2014.pdf)
- [GitHub — TimelyDataflow/differential-dataflow](https://github.com/TimelyDataflow/differential-dataflow)
- [Differential dataflow — Microsoft Research](https://www.microsoft.com/en-us/research/publication/differential-dataflow/)
- [Naiad: A Timely Dataflow System — Microsoft Research](https://www.microsoft.com/en-us/research/project/naiad/)
