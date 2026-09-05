---
layout: post
title: "성질 기반 테스팅(Property-Based Testing) 완전 정복: QuickCheck부터 Hypothesis까지 자동으로 버그를 찾는 법"
date: 2026-09-05
categories: [cs, computer-science]
tags: [property-based-testing, quickcheck, hypothesis, testing, fuzzing, shrinking, python, haskell]
---

소프트웨어 테스트를 작성할 때 우리는 대부분 **예시 기반 테스트(Example-Based Testing)**를 사용한다. `add(2, 3) == 5`, `sort([3,1,2]) == [1,2,3]` 같이 특정 입력과 그에 대한 기대 출력을 직접 명시하는 방식이다. 그러나 이 방식은 근본적인 한계가 있다. 개발자가 떠올린 케이스만 검사할 뿐, **미처 생각하지 못한 경계 조건**은 영원히 테스트되지 않는다. 성질 기반 테스팅(Property-Based Testing, PBT)은 이 한계를 정면으로 해결한다.

## 성질 기반 테스팅이란 무엇인가

PBT는 1999년 Haskell 생태계에서 등장한 **QuickCheck** 라이브러리에서 시작됐다. 핵심 아이디어는 간단하다: "이 함수는 어떤 입력에 대해서도 이 성질(property)을 만족해야 한다"고 선언하면, 테스팅 프레임워크가 **수백~수천 개의 입력을 자동으로 생성**하여 해당 성질이 깨지는 경우를 찾아낸다.

예시 기반 테스트와의 차이를 명확히 하자:

```
# 예시 기반 테스트 — 개발자가 직접 케이스를 나열
assert sort([3, 1, 2]) == [1, 2, 3]
assert sort([]) == []
assert sort([1]) == [1]

# 성질 기반 테스트 — "어떤 리스트라도 정렬하면 이 성질이 성립해야 한다"
for any list xs:
    sorted_xs = sort(xs)
    # 성질 1: 길이가 보존된다
    assert len(sorted_xs) == len(xs)
    # 성질 2: 인접 원소는 오름차순이다
    assert all(sorted_xs[i] <= sorted_xs[i+1] for i in range(len(sorted_xs)-1))
    # 성질 3: 같은 원소들로 구성된다 (순열)
    assert sorted(sorted_xs) == sorted(xs)  # 또는 multiset equality
```

성질(property)이란 입력의 구체적인 값에 의존하지 않는 **불변 조건(invariant)**이다. 우리는 "특정 입력 → 특정 출력"이 아니라, "임의의 입력 → 반드시 참이어야 하는 관계"를 명세한다.

## 왜 성질 기반 테스팅이 필요한가

### 경계 조건(Edge Case)의 자동 발견

개발자가 직접 테스트를 작성할 때는 머릿속에 있는 "정상 케이스"와 "기억하는 경계 케이스"만 다룬다. PBT 프레임워크는 체계적으로 다음을 시도한다:

- **정수**: 0, -1, 최솟값(`INT_MIN`), 최댓값(`INT_MAX`), 대용량 수
- **문자열**: 빈 문자열, 한 글자, 유니코드, 개행/탭, 매우 긴 문자열
- **리스트**: 빈 리스트, 단일 원소, 중복 원소, 정렬된/역정렬된 리스트
- **수치**: NaN, Infinity, 음수 부동소수점

이처럼 프레임워크가 직접 "고약한 입력"을 탐색하므로, 개발자가 상상하지 못한 버그를 찾아낸다.

### 축소(Shrinking)로 최소 재현 사례 제공

PBT의 또 다른 핵심 기능은 **shrinking**이다. 버그를 유발하는 입력을 찾으면, 프레임워크는 여전히 버그를 일으키는 **가장 작은(단순한) 반례**를 자동으로 찾아준다.

예를 들어, 정수 리스트에서 버그를 유발하는 입력이 처음에 `[1347, -92, 0, 4567, -1, 99, 234]`였다면, shrinking 후에는 `[-1]` 또는 `[0, -1]`처럼 최소화된 케이스를 보고해 준다. 디버깅 시간이 극적으로 줄어든다.

### 명세(Specification)로서의 테스트

PBT를 작성하는 과정 자체가 **함수의 계약(contract)을 명확히 사고하도록 강제**한다. "이 함수가 만족해야 할 성질이 무엇인가?"를 묻는 것은 구현과 독립적인 정확한 명세를 정의하는 행위다. 이는 코드 리뷰, 문서화, 리팩터링 안전망으로도 작용한다.

## Python Hypothesis로 구현하기

Hypothesis는 Python 생태계의 대표적인 PBT 라이브러리다. `pip install hypothesis`로 설치한다.

### 예제 1: 인코딩/디코딩 라운드트립 성질

```python
import base64
from hypothesis import given, settings
from hypothesis import strategies as st

def encode(data: bytes) -> str:
    return base64.b64encode(data).decode("ascii")

def decode(encoded: str) -> bytes:
    return base64.b64decode(encoded)

# 성질: 어떤 바이트 시퀀스든 encode → decode 하면 원래 값이 된다
@given(st.binary())
def test_roundtrip(data: bytes):
    assert decode(encode(data)) == data

# 성질: encode 결과는 항상 ASCII 영역의 유효한 base64 문자만 포함한다
@given(st.binary())
def test_encode_is_valid_base64(data: bytes):
    encoded = encode(data)
    valid_chars = set("ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/=")
    assert all(c in valid_chars for c in encoded)

# 더 많은 예시를 테스트하려면 settings 조절
@settings(max_examples=500)
@given(st.binary(min_size=1, max_size=10_000))
def test_roundtrip_large(data: bytes):
    assert decode(encode(data)) == data
```

### 예제 2: 정렬 알고리즘의 성질 테스트

```python
from hypothesis import given
from hypothesis import strategies as st
from typing import List

def my_sort(lst: List[int]) -> List[int]:
    # 개발자가 직접 구현한 정렬 — 버그가 있을 수 있음
    if len(lst) <= 1:
        return lst[:]
    pivot = lst[len(lst) // 2]
    left  = [x for x in lst if x < pivot]
    mid   = [x for x in lst if x == pivot]
    right = [x for x in lst if x > pivot]
    return my_sort(left) + mid + my_sort(right)

@given(st.lists(st.integers()))
def test_sort_length_preserved(lst):
    """정렬 전후 길이가 보존된다."""
    assert len(my_sort(lst)) == len(lst)

@given(st.lists(st.integers()))
def test_sort_is_ordered(lst):
    """정렬 결과는 오름차순이다."""
    result = my_sort(lst)
    for i in range(len(result) - 1):
        assert result[i] <= result[i + 1]

@given(st.lists(st.integers()))
def test_sort_is_permutation(lst):
    """정렬 결과는 원래 리스트와 같은 원소로 구성된다."""
    assert sorted(my_sort(lst)) == sorted(lst)

@given(st.lists(st.integers()))
def test_sort_idempotent(lst):
    """이미 정렬된 리스트를 다시 정렬해도 결과가 같다."""
    sorted_once = my_sort(lst)
    sorted_twice = my_sort(sorted_once)
    assert sorted_once == sorted_twice
```

### 예제 3: 상태 기반 테스트 (Stateful Testing)

Hypothesis는 단순 함수뿐 아니라 **상태를 가진 시스템**도 테스트할 수 있다. `RuleBasedStateMachine`을 사용하여 임의의 연산 시퀀스로 시스템을 검증한다.

```python
from hypothesis.stateful import RuleBasedStateMachine, rule, invariant
from hypothesis import strategies as st
from collections import deque

class BoundedQueue:
    """최대 크기 제한이 있는 큐"""
    def __init__(self, max_size: int):
        self.max_size = max_size
        self._data = deque()

    def enqueue(self, item) -> bool:
        if len(self._data) >= self.max_size:
            return False  # 꽉 찼으면 실패
        self._data.append(item)
        return True

    def dequeue(self):
        if not self._data:
            raise IndexError("Queue is empty")
        return self._data.popleft()

    def size(self) -> int:
        return len(self._data)

    def is_empty(self) -> bool:
        return len(self._data) == 0

class BoundedQueueTest(RuleBasedStateMachine):
    def __init__(self):
        super().__init__()
        self.model = BoundedQueue(max_size=5)
        self.expected_size = 0

    @rule(item=st.integers())
    def do_enqueue(self, item):
        result = self.model.enqueue(item)
        if self.expected_size < 5:
            assert result is True
            self.expected_size += 1
        else:
            assert result is False

    @rule()
    def do_dequeue(self):
        if self.expected_size > 0:
            self.model.dequeue()
            self.expected_size -= 1
        else:
            try:
                self.model.dequeue()
                assert False, "빈 큐에서 dequeue 시 IndexError여야 함"
            except IndexError:
                pass

    @invariant()
    def size_is_consistent(self):
        assert self.model.size() == self.expected_size
        assert (self.model.size() == 0) == self.model.is_empty()

QueueTest = BoundedQueueTest.TestCase
```

## JavaScript: fast-check로 PBT 구현하기

JavaScript 생태계에서는 **fast-check** 라이브러리가 대표적이다. `npm install fast-check`로 설치한다.

```javascript
import fc from "fast-check";

// 성질 1: reverse(reverse(arr)) === arr
fc.assert(
  fc.property(fc.array(fc.integer()), (arr) => {
    const reversed = [...arr].reverse();
    const doubleReversed = [...reversed].reverse();
    // 배열 비교를 JSON으로 단순화
    return JSON.stringify(doubleReversed) === JSON.stringify(arr);
  })
);

// 성질 2: 두 정렬된 배열을 병합하면 결과도 정렬된다
function mergeSorted(a, b) {
  const result = [];
  let i = 0, j = 0;
  while (i < a.length && j < b.length) {
    if (a[i] <= b[j]) result.push(a[i++]);
    else result.push(b[j++]);
  }
  return result.concat(a.slice(i)).concat(b.slice(j));
}

fc.assert(
  fc.property(
    fc.array(fc.integer()).map(arr => [...arr].sort((a, b) => a - b)),
    fc.array(fc.integer()).map(arr => [...arr].sort((a, b) => a - b)),
    (sortedA, sortedB) => {
      const merged = mergeSorted(sortedA, sortedB);
      // 병합 결과가 정렬되어 있는가?
      for (let i = 0; i < merged.length - 1; i++) {
        if (merged[i] > merged[i + 1]) return false;
      }
      // 길이가 보존되는가?
      return merged.length === sortedA.length + sortedB.length;
    }
  )
);

// 성질 3: 문자열 직렬화 라운드트립
fc.assert(
  fc.property(
    fc.record({
      name: fc.string(),
      age: fc.nat(120),
      active: fc.boolean(),
    }),
    (obj) => {
      const serialized = JSON.stringify(obj);
      const deserialized = JSON.parse(serialized);
      return (
        deserialized.name === obj.name &&
        deserialized.age === obj.age &&
        deserialized.active === obj.active
      );
    }
  )
);
```

## 유용한 성질 패턴

PBT에서 자주 사용하는 성질 유형을 정리해 두면 테스트 작성이 쉬워진다:

| 패턴 | 설명 | 예시 |
|---|---|---|
| **라운드트립** | `decode(encode(x)) == x` | 직렬화/역직렬화, 암호화/복호화 |
| **항등원** | `f(identity) == x` | `x + 0 == x`, `x * 1 == x` |
| **교환법칙** | `f(a, b) == f(b, a)` | 덧셈, 집합 합집합 |
| **결합법칙** | `f(f(a,b),c) == f(a,f(b,c))` | 문자열 연결, 리스트 병합 |
| **멱등성** | `f(f(x)) == f(x)` | 정렬, 중복 제거 |
| **오라클** | `fast_impl(x) == slow_impl(x)` | 최적화 전후 동일성 검증 |
| **불변 조건** | 연산 후에도 성질이 유지됨 | BST 성질, 힙 성질 |
| **단조성** | `a <= b → f(a) <= f(b)` | 단조 함수 검증 |

## 주의사항과 팁

### 성질 선택의 어려움

PBT의 가장 큰 도전은 "어떤 성질을 작성할 것인가"이다. 나쁜 성질의 예:

```python
# 너무 약한 성질 — 거의 모든 구현이 통과함
@given(st.lists(st.integers()))
def test_bad_sort(lst):
    result = my_sort(lst)
    assert len(result) == len(lst)  # 역방향 정렬도 통과!
```

좋은 성질은 구현의 핵심 계약을 명확히 포착해야 한다. **오라클 기법**(느리지만 명확히 올바른 참조 구현과 비교)이 가장 강력한 성질이다.

### 생성 전략(Strategy) 설계

무작위 입력이 도메인 제약을 위반하면 테스트 자체가 의미 없다. Hypothesis의 `assume()`이나 커스텀 전략으로 유효한 입력만 생성해야 한다:

```python
from hypothesis import given, assume
from hypothesis import strategies as st

@given(st.integers(), st.integers())
def test_division(a, b):
    assume(b != 0)  # 0으로 나누기 제외
    result = a / b
    assert isinstance(result, float)
```

### 실행 속도 조절

PBT는 기본적으로 100개(Hypothesis 기준)의 예시를 생성한다. CI 파이프라인에서는 빠른 검증을 위해 줄이고, 밤새 돌리는 검증에서는 늘리는 식으로 조절한다:

```python
from hypothesis import given, settings, HealthCheck
from hypothesis import strategies as st

@settings(
    max_examples=1000,          # 더 많은 예시
    deadline=None,              # 타임아웃 비활성화
    suppress_health_check=[HealthCheck.too_slow],
)
@given(st.text())
def test_extensive(s):
    ...
```

### 데이터베이스 활용

Hypothesis는 이전 테스트 실행에서 발견한 실패 케이스를 **로컬 DB에 저장**하여 회귀 테스트로 재사용한다. 한 번 발견된 버그는 코드가 수정될 때까지 계속 재현을 시도한다.

## 성질 기반 테스팅의 한계

PBT가 만능은 아니다:
- **성질 작성의 인지 비용**: "이 코드가 만족해야 할 수학적 성질"을 공식화하는 것 자체가 어렵다.
- **도메인 복잡도**: 입력이 복잡한 비즈니스 규칙을 만족해야 할 때 유효한 입력 생성이 까다롭다.
- **비결정적 시스템**: 외부 의존성이나 시간에 의존하는 코드는 테스트하기 어렵다.
- **예시 기반 테스트와 병행**: PBT는 예시 기반 테스트를 **대체**하는 것이 아니라 **보완**하는 도구다.

성질 기반 테스팅은 "내가 생각하지 못한 버그"를 찾는 데 탁월하다. 특히 파서, 직렬화 라이브러리, 자료구조, 암호화 코드처럼 수학적 불변 조건이 명확한 코드에서 강력한 효과를 발휘한다. 오늘 당장 기존 코드베이스의 유틸리티 함수 하나에 PBT를 적용해 보라 — 예상치 못한 버그를 발견할 확률이 놀랍도록 높다.

## 참고 자료
- [Hypothesis 공식 문서](https://hypothesis.readthedocs.io/)
- [What is Property-Based Testing? — hypothesis.works](https://hypothesis.works/articles/what-is-property-based-testing/)
- [QuickCheck in Every Language — hypothesis.works](https://hypothesis.works/articles/quickcheck-in-every-language/)
- [fast-check 공식 문서](https://fast-check.dev/)
