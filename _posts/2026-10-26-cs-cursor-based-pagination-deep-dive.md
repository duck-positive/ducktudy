---
layout: post
title: "커서 기반 페이지네이션(Cursor-Based Pagination) 심화: 대용량 데이터 조회 최적화"
date: 2026-10-26
categories: [cs, computer-science]
tags: [pagination, cursor, database, api-design, sql, performance, scalability]
---

## 페이지네이션이 왜 중요한가

수천만 건의 데이터를 보유한 서비스에서 목록 조회 API를 설계할 때 가장 먼저 맞닥뜨리는 난제가 페이지네이션입니다. 단순히 전체 결과를 반환하면 응답이 너무 크고, 페이지 단위로 잘라 반환해야 합니다. 그러나 어떤 방식으로 자르느냐에 따라 성능과 정확성이 극단적으로 달라집니다.

크게 두 가지 방식이 존재합니다:
- **오프셋 기반(Offset-Based) 페이지네이션**: `LIMIT n OFFSET m` SQL 구문 활용
- **커서 기반(Cursor-Based) 페이지네이션**: 마지막으로 조회한 레코드의 포인터(커서)를 기준으로 다음 페이지를 조회

대부분의 초보 개발자는 오프셋 방식으로 시작하지만, 규모가 커질수록 반드시 커서 방식으로 전환해야 합니다.

## 오프셋 페이지네이션의 근본적인 문제

### 문제 1: 대용량 데이터에서의 성능 저하

```sql
-- 1페이지 조회 (빠름)
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 0;

-- 50,000페이지 조회 (매우 느림)
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 999980;
```

`OFFSET 999980`은 데이터베이스가 실제로 100만 번째 레코드까지 **모두 읽고** 그 중 999,980개를 버린다는 의미입니다. 인덱스를 사용하더라도 인덱스 탐색을 100만 번 수행해야 합니다. 페이지 번호가 클수록 쿼리가 선형적으로 느려집니다.

### 문제 2: 데이터 변경 시 레코드 누락/중복

오프셋 방식은 데이터가 변하면 정확성을 보장하지 못합니다.

```
페이지 1 조회: [A, B, C, D, E] (id=5~1, 내림차순)
→ 이 사이에 새 레코드 F(id=6)가 삽입됨
페이지 2 조회: [E, F, A, B, C...] ← E가 두 번 등장!
```

소셜 미디어 피드, 실시간 데이터 조회처럼 데이터가 자주 변하는 환경에서는 오프셋 방식이 부적합합니다.

## 커서 기반 페이지네이션의 원리

커서는 **데이터셋 내의 특정 위치를 가리키는 불투명한 포인터**입니다. 클라이언트는 커서의 내부 구조를 알 필요가 없으며, 다음 페이지를 요청할 때 이전 응답에서 받은 커서를 그대로 전달합니다.

### 핵심 SQL 패턴: Keyset Pagination

커서 기반 페이지네이션의 SQL 구현은 `WHERE` 조건에 마지막 레코드의 값을 사용하는 것입니다:

```sql
-- 첫 페이지 조회
SELECT id, title, created_at 
FROM posts 
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- 이전 결과의 마지막 레코드가 (created_at='2026-10-25 12:00:00', id=1234) 일 때
-- 다음 페이지 조회
SELECT id, title, created_at 
FROM posts 
WHERE (created_at, id) < ('2026-10-25 12:00:00', 1234)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

`OFFSET`을 사용하지 않으므로 데이터베이스는 조건에 맞는 레코드부터 탐색을 시작합니다. 인덱스 `(created_at, id)`가 있다면 **O(log N + page_size)** 복잡도로, 전체 데이터 크기에 무관한 일정한 성능을 보장합니다.

## 실제 구현 예제

### 예제 1: REST API 커서 페이지네이션 (Python + PostgreSQL)

```python
import base64
import json
from datetime import datetime
from typing import Optional
import psycopg2
from psycopg2.extras import RealDictCursor

def encode_cursor(created_at: datetime, record_id: int) -> str:
    """커서를 불투명한 Base64 문자열로 인코딩"""
    cursor_data = {
        'created_at': created_at.isoformat(),
        'id': record_id
    }
    encoded = base64.urlsafe_b64encode(
        json.dumps(cursor_data).encode()
    ).decode()
    return encoded

def decode_cursor(cursor_str: str) -> tuple[datetime, int]:
    """Base64 커서를 디코딩하여 실제 값 추출"""
    decoded = json.loads(
        base64.urlsafe_b64decode(cursor_str.encode())
    )
    return datetime.fromisoformat(decoded['created_at']), decoded['id']

def get_posts_paginated(
    conn,
    limit: int = 20,
    after_cursor: Optional[str] = None
) -> dict:
    """커서 기반 페이지네이션으로 게시글 목록 조회"""
    
    with conn.cursor(cursor_factory=RealDictCursor) as cur:
        if after_cursor:
            # 커서 이후 데이터 조회
            after_created_at, after_id = decode_cursor(after_cursor)
            
            query = """
                SELECT id, title, created_at, author_id
                FROM posts
                WHERE (created_at, id) < (%s, %s)
                ORDER BY created_at DESC, id DESC
                LIMIT %s
            """
            cur.execute(query, (after_created_at, after_id, limit + 1))
        else:
            # 첫 페이지 조회
            query = """
                SELECT id, title, created_at, author_id
                FROM posts
                ORDER BY created_at DESC, id DESC
                LIMIT %s
            """
            cur.execute(query, (limit + 1,))
        
        rows = cur.fetchall()
    
    # limit+1개를 조회해서 다음 페이지 존재 여부 판단
    has_next_page = len(rows) > limit
    items = rows[:limit]
    
    # 다음 페이지 커서 생성
    next_cursor = None
    if has_next_page and items:
        last_item = items[-1]
        next_cursor = encode_cursor(
            last_item['created_at'],
            last_item['id']
        )
    
    return {
        'items': [dict(row) for row in items],
        'pagination': {
            'has_next_page': has_next_page,
            'next_cursor': next_cursor,
            'count': len(items)
        }
    }

# 사용 예시
# conn = psycopg2.connect(...)
# page1 = get_posts_paginated(conn, limit=20)
# page2 = get_posts_paginated(conn, limit=20, after_cursor=page1['pagination']['next_cursor'])
```

`limit + 1`개를 조회하여 다음 페이지 존재 여부를 판단하는 트릭에 주목하세요. 별도의 `COUNT` 쿼리 없이 다음 페이지가 있는지 효율적으로 확인합니다.

### 예제 2: GraphQL Relay 커넥션 스펙 구현 (TypeScript)

Facebook이 제안한 [Relay Cursor Connections Specification](https://relay.dev/graphql/connections.htm)은 커서 기반 페이지네이션의 표준입니다.

```typescript
import { GraphQLObjectType, GraphQLString, GraphQLInt, GraphQLBoolean, GraphQLList, GraphQLNonNull } from 'graphql';

// Relay 스펙의 Edge와 Connection 타입 정의
const PostEdgeType = new GraphQLObjectType({
  name: 'PostEdge',
  fields: {
    node: { type: PostType },    // 실제 데이터
    cursor: { type: GraphQLString }  // 이 노드의 커서
  }
});

const PageInfoType = new GraphQLObjectType({
  name: 'PageInfo',
  fields: {
    hasNextPage: { type: new GraphQLNonNull(GraphQLBoolean) },
    hasPreviousPage: { type: new GraphQLNonNull(GraphQLBoolean) },
    startCursor: { type: GraphQLString },
    endCursor: { type: GraphQLString }
  }
});

const PostConnectionType = new GraphQLObjectType({
  name: 'PostConnection',
  fields: {
    edges: { type: new GraphQLList(PostEdgeType) },
    pageInfo: { type: new GraphQLNonNull(PageInfoType) },
    totalCount: { type: GraphQLInt }  // 선택적 - 전체 개수
  }
});

// 리졸버 구현
async function resolvePostConnection(
  _: unknown,
  args: { first?: number; after?: string; before?: string; last?: number },
  ctx: Context
) {
  const { first = 20, after } = args;
  
  let whereClause = '';
  const params: any[] = [];
  
  if (after) {
    const { createdAt, id } = decodeCursor(after);
    whereClause = 'WHERE (created_at, id) < ($1, $2)';
    params.push(createdAt, id);
  }
  
  const rows = await ctx.db.query(
    `SELECT id, title, created_at 
     FROM posts 
     ${whereClause}
     ORDER BY created_at DESC, id DESC 
     LIMIT $${params.length + 1}`,
    [...params, first + 1]
  );
  
  const hasNextPage = rows.length > first;
  const nodes = rows.slice(0, first);
  
  const edges = nodes.map(node => ({
    node,
    cursor: encodeCursor(node.created_at, node.id)
  }));
  
  return {
    edges,
    pageInfo: {
      hasNextPage,
      hasPreviousPage: !!after,
      startCursor: edges[0]?.cursor ?? null,
      endCursor: edges[edges.length - 1]?.cursor ?? null
    }
  };
}
```

## 커서 설계 시 고려사항

### 정렬 키의 유일성 보장

커서의 기준이 되는 컬럼은 **유일하게 위치를 특정**할 수 있어야 합니다. `created_at`만으로는 동일 타임스탬프를 가진 레코드들 사이에서 위치를 특정할 수 없으므로, `(created_at, id)` 복합 키처럼 고유성을 보장하는 조합을 사용해야 합니다.

### 커서 만료 처리

커서는 특정 시점의 데이터 위치를 가리키므로, 너무 오래된 커서는 의미 없거나 보안 문제를 일으킬 수 있습니다. 커서에 만료 시간을 포함하거나 서버 측에서 커서 유효성을 검증하는 방안을 고려하세요.

### 양방향 페이지네이션

Relay 스펙은 `after/first`(앞으로 이동)와 `before/last`(뒤로 이동)를 모두 지원합니다. 양방향 커서 구현 시 정렬 방향을 반전해야 하며, `(created_at, id) > (cursor_created_at, cursor_id)` 조건을 사용합니다.

### 보안: 커서 내 민감 정보 보호

커서를 단순 Base64로 인코딩하면 디코딩이 쉬워 내부 데이터 구조가 노출됩니다. 민감한 시스템에서는 HMAC으로 커서를 서명하거나 서버 측 세션에 커서 상태를 저장하세요.

```python
import hmac, hashlib

SECRET_KEY = b'super-secret-key'

def sign_cursor(cursor_data: str) -> str:
    sig = hmac.new(SECRET_KEY, cursor_data.encode(), hashlib.sha256).hexdigest()
    return f"{cursor_data}.{sig}"

def verify_cursor(signed_cursor: str) -> str:
    parts = signed_cursor.rsplit('.', 1)
    if len(parts) != 2:
        raise ValueError("Invalid cursor")
    data, sig = parts
    expected = hmac.new(SECRET_KEY, data.encode(), hashlib.sha256).hexdigest()
    if not hmac.compare_digest(sig, expected):
        raise ValueError("Cursor signature invalid")
    return data
```

## 오프셋 vs 커서: 언제 무엇을 쓸까

| 기준 | 오프셋 | 커서 |
|------|--------|------|
| 구현 복잡도 | 낮음 | 중간 |
| 대용량 성능 | O(N) 저하 | O(log N) 일정 |
| 데이터 변경 안정성 | 누락/중복 가능 | 안정적 |
| 임의 페이지 이동 | 가능 | 불가능 |
| 전체 개수 제공 | 쉬움 | 별도 쿼리 필요 |
| 적합한 상황 | 소규모, 정적 데이터 | 대규모, 동적 피드 |

인스타그램, Twitter, GitHub 등 대규모 서비스의 피드 API는 모두 커서 기반입니다. 데이터가 100만 건을 넘거나 실시간으로 변하는 환경이라면 커서 기반 페이지네이션 전환을 강력히 권장합니다.

## 참고 자료
- [Relay Cursor Connections Specification](https://relay.dev/graphql/connections.htm)
- [PostgreSQL LIMIT/OFFSET Documentation](https://www.postgresql.org/docs/current/queries-limit.html)
- [GitHub REST API Pagination Guide](https://docs.github.com/en/rest/using-the-rest-api/using-pagination-in-the-rest-api)
- [Stripe API Pagination Design](https://stripe.com/docs/api/pagination)
