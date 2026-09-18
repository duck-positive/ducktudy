---
layout: post
title: "데이터베이스 정규화 완전 정복: 1NF부터 BCNF까지, 이상 현상 없는 스키마 설계"
date: 2026-09-18
categories: [cs, computer-science]
tags: [database, normalization, 1NF, 2NF, 3NF, BCNF, functional-dependency, relational-database, schema-design, SQL]
---

데이터베이스를 설계할 때 가장 흔히 마주하는 문제는 **이상 현상(Anomaly)**입니다. 데이터를 삽입할 때 불필요한 정보를 함께 넣어야 하거나, 한 데이터를 수정할 때 여러 행을 바꿔야 하거나, 데이터를 삭제했더니 다른 중요한 정보까지 사라지는 현상들입니다. **정규화(Normalization)**는 이러한 이상 현상을 제거하기 위해 테이블을 체계적으로 분해하는 과정입니다. 1970년 에드가 F. 코드(Edgar F. Codd)가 제안한 이래 관계형 데이터베이스 설계의 핵심 원칙으로 자리 잡았습니다.

## 이상 현상이란 무엇인가?

다음과 같은 `수강_교수` 테이블을 생각해봅시다:

| 학번 | 학생명 | 과목코드 | 과목명 | 교수명 | 교수실 |
|------|--------|----------|--------|--------|--------|
| 101 | 김철수 | CS101 | 자료구조 | 이교수 | 302호 |
| 101 | 김철수 | CS201 | 알고리즘 | 박교수 | 401호 |
| 102 | 이영희 | CS101 | 자료구조 | 이교수 | 302호 |
| 103 | 박민준 | CS101 | 자료구조 | 이교수 | 302호 |

이 테이블에서 발생하는 이상 현상:

- **삽입 이상(Insertion Anomaly)**: 새 교수(정교수, CS301)를 등록하려면 수강 학생 없이는 삽입 불가능
- **삭제 이상(Deletion Anomaly)**: 학번 102 학생이 자료구조 수강을 취소하면 이교수의 교수실 정보가 삭제될 위험
- **갱신 이상(Update Anomaly)**: 이교수의 교수실이 변경되면 세 행을 모두 수정해야 함 — 일부만 수정 시 데이터 불일치

이 문제들의 근본 원인은 **함수 종속성(Functional Dependency)**이 제대로 분리되지 않았기 때문입니다.

## 함수 종속성 (Functional Dependency)

함수 종속성 `X → Y`는 "X의 값이 Y의 값을 결정한다"는 의미입니다. 위 테이블에서:

- `학번 → 학생명` (학번이 같으면 학생명이 같다)
- `과목코드 → 과목명, 교수명` (과목코드가 같으면 과목명과 교수명이 같다)
- `교수명 → 교수실` (교수명이 같으면 교수실이 같다)
- `(학번, 과목코드) → (모든 속성)` (복합 기본키)

**암스트롱 공리(Armstrong's Axioms)**는 함수 종속성을 추론하는 기본 규칙입니다:

1. **반사율(Reflexivity)**: `Y ⊆ X → X → Y`
2. **증가율(Augmentation)**: `X → Y → XZ → YZ`
3. **추이율(Transitivity)**: `X → Y, Y → Z → X → Z`

---

## 제1정규형 (1NF: First Normal Form)

**원칙**: 모든 속성은 원자 값(Atomic Value)을 가져야 한다. 반복 그룹이나 다중 값 속성이 없어야 한다.

**위반 예시**:

| 학번 | 학생명 | 수강과목 |
|------|--------|----------|
| 101 | 김철수 | CS101, CS201 |

`수강과목` 컬럼에 여러 값이 들어있어 1NF 위반입니다.

**1NF 변환 후**:

| 학번 | 학생명 | 수강과목 |
|------|--------|----------|
| 101 | 김철수 | CS101 |
| 101 | 김철수 | CS201 |

### 코드 예제 1: SQL로 정규화 전/후 비교

```sql
-- 1NF 위반 테이블 (정규화 전)
CREATE TABLE 수강_비정규 (
    학번 INT,
    학생명 VARCHAR(50),
    수강과목 VARCHAR(200)  -- 'CS101, CS201, CS301' 형태로 저장
);

-- 1NF를 만족하는 테이블
CREATE TABLE 수강_1nf (
    학번 INT,
    학생명 VARCHAR(50),
    과목코드 VARCHAR(10),
    PRIMARY KEY (학번, 과목코드)
);

-- 비정규 데이터를 1NF로 변환하는 쿼리 (PostgreSQL)
INSERT INTO 수강_1nf (학번, 학생명, 과목코드)
SELECT 
    학번,
    학생명,
    TRIM(unnest(string_to_array(수강과목, ','))) AS 과목코드
FROM 수강_비정규;
```

---

## 제2정규형 (2NF: Second Normal Form)

**원칙**: 1NF를 만족하고, 모든 비주요 속성이 기본키에 **완전 함수 종속(Fully Functionally Dependent)**이어야 한다. 즉, 부분 함수 종속(Partial Dependency)이 없어야 한다.

부분 함수 종속은 복합 기본키에서 기본키의 일부에만 종속되는 경우 발생합니다.

**위반 예시** (기본키: `(학번, 과목코드)`):

| 학번 | 과목코드 | 학생명 | 과목명 | 성적 |
|------|----------|--------|--------|------|
| 101 | CS101 | 김철수 | 자료구조 | A |

- `학번 → 학생명`: 학번만으로 학생명이 결정됨 → **부분 종속** (위반!)
- `과목코드 → 과목명`: 과목코드만으로 과목명이 결정됨 → **부분 종속** (위반!)
- `(학번, 과목코드) → 성적`: 완전 종속 (정상)

**2NF 변환**: 부분 종속 속성들을 분리합니다.

```sql
-- 2NF를 만족하는 분리된 테이블들
CREATE TABLE 학생 (
    학번 INT PRIMARY KEY,
    학생명 VARCHAR(50)
);

CREATE TABLE 과목 (
    과목코드 VARCHAR(10) PRIMARY KEY,
    과목명 VARCHAR(100)
);

CREATE TABLE 수강 (
    학번 INT REFERENCES 학생(학번),
    과목코드 VARCHAR(10) REFERENCES 과목(과목코드),
    성적 CHAR(2),
    PRIMARY KEY (학번, 과목코드)
);
```

---

## 제3정규형 (3NF: Third Normal Form)

**원칙**: 2NF를 만족하고, 비주요 속성이 기본키에 **이행적 함수 종속(Transitive Dependency)**을 가지지 않아야 한다.

이행적 종속: `기본키 → A → B` 형태의 종속 (A가 비주요 속성인데 B가 A에 종속됨)

**위반 예시** (기본키: `과목코드`):

| 과목코드 | 교수번호 | 교수명 | 교수실 |
|----------|----------|--------|--------|
| CS101 | P001 | 이교수 | 302호 |

- `과목코드 → 교수번호 → 교수실`: 이행적 종속 발생
- `교수번호 → 교수명, 교수실`: 교수번호로 교수 정보가 결정됨

**3NF 변환**:

```sql
-- 이행적 종속 제거
CREATE TABLE 과목_교수 (
    과목코드 VARCHAR(10) PRIMARY KEY,
    교수번호 CHAR(4),
    FOREIGN KEY (교수번호) REFERENCES 교수(교수번호)
);

CREATE TABLE 교수 (
    교수번호 CHAR(4) PRIMARY KEY,
    교수명 VARCHAR(50),
    교수실 VARCHAR(20)
);
```

---

## BCNF (Boyce-Codd Normal Form)

**원칙**: 3NF보다 엄격한 형태로, 모든 **결정자(Determinant)**가 **슈퍼키(Superkey)**이어야 한다. 즉, `X → Y`인 모든 비자명 함수 종속에서 X는 슈퍼키여야 한다.

3NF를 만족하지만 BCNF를 만족하지 않는 경우가 존재합니다:

**예시**: 학생이 여러 과목을 수강하고, 각 과목마다 한 명의 지도교수가 배정됩니다. 교수는 하나의 과목만 담당합니다.

| 학번 | 과목명 | 교수명 |
|------|--------|--------|
| 101 | 자료구조 | 이교수 |
| 101 | 알고리즘 | 박교수 |
| 102 | 자료구조 | 이교수 |

- 기본키 후보: `(학번, 과목명)` 또는 `(학번, 교수명)`
- `교수명 → 과목명`: 교수가 결정자이지만 슈퍼키가 아님 → **BCNF 위반**

**BCNF 변환**:

```sql
-- 교수_과목 관계 분리
CREATE TABLE 교수_과목 (
    교수명 VARCHAR(50) PRIMARY KEY,
    과목명 VARCHAR(100)
);

CREATE TABLE 학생_교수 (
    학번 INT,
    교수명 VARCHAR(50) REFERENCES 교수_과목(교수명),
    PRIMARY KEY (학번, 교수명)
);
```

### 코드 예제 2: Python으로 함수 종속성 검증 및 BCNF 분해

```python
from itertools import combinations


def closure(attributes, fds):
    """함수 종속성 집합에서 속성 집합의 폐포(Closure)를 계산"""
    closure_set = set(attributes)
    changed = True
    while changed:
        changed = False
        for lhs, rhs in fds:
            if lhs.issubset(closure_set):
                new_attrs = rhs - closure_set
                if new_attrs:
                    closure_set |= new_attrs
                    changed = True
    return closure_set


def is_superkey(attributes, all_attrs, fds):
    """주어진 속성 집합이 슈퍼키인지 확인"""
    return closure(attributes, fds) == all_attrs


def find_bcnf_violations(all_attrs, fds):
    """BCNF 위반 함수 종속성 탐색"""
    violations = []
    for lhs, rhs in fds:
        non_trivial_rhs = rhs - lhs
        if non_trivial_rhs and not is_superkey(lhs, all_attrs, fds):
            violations.append((lhs, rhs))
    return violations


def bcnf_decompose(relation, all_attrs, fds):
    """BCNF 분해 알고리즘"""
    violations = find_bcnf_violations(all_attrs, fds)
    if not violations:
        print(f"  {relation}: BCNF 만족 ✓")
        return [relation]
    
    lhs, rhs = violations[0]
    lhs_closure = closure(lhs, fds)
    
    r1 = lhs_closure  # lhs의 폐포
    r2 = all_attrs - (lhs_closure - lhs)  # 나머지 속성 + lhs
    
    print(f"  {relation} 분해:")
    print(f"    위반 FD: {lhs} → {rhs}")
    print(f"    R1 = {r1}")
    print(f"    R2 = {r2}")
    
    # 각 부분 스키마에 해당하는 FD 추출
    fds_r1 = [(l, r) for l, r in fds if l.issubset(r1) and r.issubset(r1)]
    fds_r2 = [(l, r) for l, r in fds if l.issubset(r2) and r.issubset(r2)]
    
    result = []
    result.extend(bcnf_decompose('R1', r1, fds_r1))
    result.extend(bcnf_decompose('R2', r2, fds_r2))
    return result


# 예시: 학번(S), 과목명(C), 교수명(P)
# FDs: {S,C} → P, {P} → C (교수는 하나의 과목만 담당)
all_attrs = {'S', 'C', 'P'}
fds = [
    (frozenset({'S', 'C'}), frozenset({'P'})),
    (frozenset({'P'}), frozenset({'C'})),
]

print("BCNF 분해 과정:")
result = bcnf_decompose('R(S,C,P)', all_attrs, fds)
print(f"\n최종 분해 결과: {result}")

# 슈퍼키 검증
for key_size in range(1, len(all_attrs) + 1):
    for combo in combinations(all_attrs, key_size):
        key_set = frozenset(combo)
        if is_superkey(key_set, all_attrs, fds):
            print(f"슈퍼키: {set(key_set)}")
```

---

## 역정규화 (Denormalization)와 실전 팁

정규화가 항상 최선은 아닙니다. 지나친 정규화는 **조인(JOIN) 연산 증가**로 조회 성능이 떨어질 수 있습니다. 실제 서비스에서는 다음을 고려합니다:

**역정규화 적용 상황:**
- 읽기가 쓰기보다 압도적으로 많은 경우
- 조인 비용이 너무 높아 성능 문제가 발생하는 경우
- OLAP 분석용 데이터 웨어하우스 (Star Schema, Snowflake Schema)

```sql
-- 역정규화 예: 주문 테이블에 고객명 캐싱 (조인 없이 빠른 조회)
CREATE TABLE 주문 (
    주문번호 BIGINT PRIMARY KEY,
    고객번호 INT NOT NULL,
    고객명 VARCHAR(50),  -- 역정규화: 고객 테이블에서 복사
    주문금액 DECIMAL(15, 2),
    주문일시 TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (고객번호) REFERENCES 고객(고객번호)
);

-- 트리거로 고객명 변경 시 동기화
CREATE OR REPLACE FUNCTION sync_고객명()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE 주문 SET 고객명 = NEW.고객명 WHERE 고객번호 = NEW.고객번호;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER 고객명_동기화
AFTER UPDATE OF 고객명 ON 고객
FOR EACH ROW EXECUTE FUNCTION sync_고객명();
```

**정규화 단계 요약:**

| 정규형 | 제거 대상 | 핵심 조건 |
|--------|-----------|-----------|
| 1NF | 반복 그룹 / 다중 값 | 원자 값 보장 |
| 2NF | 부분 함수 종속 | 완전 함수 종속 |
| 3NF | 이행적 함수 종속 | 비주요 속성이 기본키에만 종속 |
| BCNF | 슈퍼키가 아닌 결정자 | 모든 결정자가 슈퍼키 |

---

## 마무리

데이터베이스 정규화는 이상 현상을 제거하고 데이터 무결성을 보장하는 체계적인 방법입니다. 대부분의 OLTP 시스템은 3NF 또는 BCNF까지 정규화하는 것이 일반적이며, 그 이상(4NF, 5NF)은 매우 특수한 경우에만 적용합니다. 실제 시스템에서는 정규화와 역정규화 사이의 균형점을 찾는 것이 중요하며, 이 결정은 항상 측정된 성능 데이터를 기반으로 해야 합니다.

## 참고 자료
- [Database Normalization: 1NF, 2NF, 3NF & BCNF Examples - DigitalOcean](https://www.digitalocean.com/community/tutorials/database-normalization)
- [Normalization in DBMS: 1NF, 2NF, 3NF and BCNF - BeginnersBook](https://beginnersbook.com/2015/05/normalization-in-dbms/)
- [Database Normalization Tutorial - SoftwareTestingHelp](https://www.softwaretestinghelp.com/database-normalization-tutorial/)
- [Functional Dependencies and Normalization - GeeksforGeeks](https://www.geeksforgeeks.org/normal-forms-in-dbms/)
