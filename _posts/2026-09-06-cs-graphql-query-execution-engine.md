---
layout: post
title: "GraphQL 쿼리 실행 엔진 완전 정복: AST 파싱부터 DataLoader N+1 해결까지"
date: 2026-09-06
categories: [cs, computer-science]
tags: [graphql, query-execution, dataloader, n-plus-one, resolver, ast, api-design]
---

## GraphQL이란 무엇인가

GraphQL은 Facebook이 2012년에 내부 개발하고 2015년에 공개한 **쿼리 언어 겸 런타임**입니다. REST API가 엔드포인트 중심이라면, GraphQL은 **타입 시스템으로 정의된 단일 엔드포인트**에서 클라이언트가 원하는 데이터 형태를 명시적으로 요청하는 패러다임입니다.

```graphql
# 클라이언트가 원하는 데이터만 정확히 요청
query {
  user(id: "42") {
    name
    email
    posts {
      title
      createdAt
    }
  }
}
```

이 단순해 보이는 쿼리 뒤에는 파싱, 검증, 실행이라는 복잡한 엔진이 동작합니다.

## GraphQL 실행 엔진의 4단계

### 1단계: 파싱(Parsing) — 텍스트 → AST

GraphQL 서버는 받은 쿼리 문자열을 **추상 구문 트리(AST)**로 변환합니다.

```
쿼리 텍스트
    ↓ 렉싱(Lexing)
토큰 스트림 [QUERY, NAME:user, LPAREN, ...]
    ↓ 파싱(Parsing)
DocumentNode
  └─ OperationDefinitionNode (query)
      └─ SelectionSetNode
          └─ FieldNode (user)
              ├─ ArgumentNode (id: "42")
              └─ SelectionSetNode
                  ├─ FieldNode (name)
                  ├─ FieldNode (email)
                  └─ FieldNode (posts)
                      └─ SelectionSetNode
                          ├─ FieldNode (title)
                          └─ FieldNode (createdAt)
```

### 2단계: 검증(Validation)

AST가 스키마에 부합하는지 검사합니다. 존재하지 않는 필드, 잘못된 인자 타입, 순환 참조 등을 이 단계에서 걸러냅니다. GraphQL 명세는 약 30가지 검증 규칙을 정의합니다.

### 3단계: 실행(Execution)

검증된 AST와 스키마를 바탕으로 **Resolver** 함수들을 호출하며 결과를 조립합니다.

### 4단계: 직렬화(Serialization)

결과 맵을 JSON으로 직렬화해 클라이언트에 응답합니다.

## Resolver 함수와 실행 모델

Resolver는 특정 필드의 값을 반환하는 함수입니다. GraphQL 실행 엔진은 **깊이 우선으로 AST를 순회**하며 각 필드의 Resolver를 호출합니다.

```javascript
// Node.js + GraphQL.js 예시
const { graphql, buildSchema } = require('graphql');

const schema = buildSchema(`
  type User {
    id: ID!
    name: String!
    email: String!
    posts: [Post!]!
  }
  type Post {
    id: ID!
    title: String!
    authorId: ID!
  }
  type Query {
    user(id: ID!): User
    users: [User!]!
  }
`);

// 가상 데이터베이스
const usersDB = {
  '1': { id: '1', name: 'Alice', email: 'alice@example.com' },
  '2': { id: '2', name: 'Bob',   email: 'bob@example.com' },
};

const postsDB = [
  { id: 'p1', title: 'Hello World', authorId: '1' },
  { id: 'p2', title: 'GraphQL Tips', authorId: '1' },
  { id: 'p3', title: 'REST vs GraphQL', authorId: '2' },
];

const resolvers = {
  Query: {
    user: (parent, { id }) => usersDB[id],
    users: () => Object.values(usersDB),
  },
  User: {
    // 기본 필드(id, name, email)는 parent 객체에서 자동 추출
    posts: (parent) => {
      // parent는 상위 Resolver가 반환한 User 객체
      console.log(`Fetching posts for user ${parent.id}`);
      return postsDB.filter(p => p.authorId === parent.id);
    },
  },
};

async function main() {
  const result = await graphql({
    schema,
    source: `
      query {
        users {
          name
          posts { title }
        }
      }
    `,
    rootValue: resolvers.Query,
    contextValue: { resolvers },
  });
  console.log(JSON.stringify(result, null, 2));
}
```

### Resolver 실행 순서

```
Query.users()                  → [User1, User2]
  User.posts(User1)           → [Post p1, Post p2]
  User.posts(User2)           → [Post p3]
```

여기서 핵심 문제가 드러납니다. User가 100명이라면 User.posts()가 100번 호출되어 **100번의 DB 쿼리**가 발생합니다 — 이것이 **N+1 문제**입니다.

## N+1 문제와 DataLoader 해결책

### N+1 문제란

```
1번 쿼리: SELECT * FROM users          → 100명 반환
100번 쿼리: SELECT * FROM posts WHERE authorId = 1
           SELECT * FROM posts WHERE authorId = 2
           ...
           SELECT * FROM posts WHERE authorId = 100
```

총 101번의 쿼리가 발생합니다. 이는 GraphQL의 구조적 특성으로, Resolver가 독립적으로 실행되기 때문입니다.

### DataLoader — 배치와 캐시로 해결

DataLoader는 Facebook이 오픈소스로 공개한 라이브러리로, **이벤트 루프 틱(tick) 내에서 개별 요청을 배치로 묶어** 한 번의 DB 쿼리로 처리합니다.

```javascript
const DataLoader = require('dataloader');

// 배치 함수: authorId 배열을 받아 posts 배열의 배열을 반환
async function batchLoadPosts(authorIds) {
  console.log(`Batch loading posts for authors: ${authorIds}`);
  // SELECT * FROM posts WHERE authorId IN (1, 2, ..., 100)
  const posts = await db.query(
    'SELECT * FROM posts WHERE authorId = ANY($1)',
    [authorIds]
  );
  
  // authorId별로 그루핑 후 authorIds 순서에 맞게 반환
  const postsByAuthor = {};
  for (const post of posts) {
    if (!postsByAuthor[post.authorId]) {
      postsByAuthor[post.authorId] = [];
    }
    postsByAuthor[post.authorId].push(post);
  }
  return authorIds.map(id => postsByAuthor[id] || []);
}

// DataLoader 인스턴스는 요청(context)당 한 번 생성
function createLoaders() {
  return {
    posts: new DataLoader(batchLoadPosts, {
      maxBatchSize: 100,    // 배치 최대 크기
      cache: true,          // 같은 키 재요청 시 캐시 사용
    }),
  };
}

// Resolver에서 DataLoader 사용
const resolversWithDataLoader = {
  User: {
    posts: (parent, args, context) => {
      // load()는 즉시 반환되지 않고 배치 큐에 추가됨
      // 이벤트 루프 다음 틱에 batchLoadPosts가 한 번 호출됨
      return context.loaders.posts.load(parent.id);
    },
  },
};

// Apollo Server 설정
const { ApolloServer } = require('@apollo/server');

const server = new ApolloServer({ typeDefs, resolvers: resolversWithDataLoader });

// context 함수는 매 요청마다 호출됨 — DataLoader 인스턴스를 요청별로 생성
await startStandaloneServer(server, {
  context: async ({ req }) => ({
    loaders: createLoaders(),
  }),
});
```

### DataLoader 동작 원리

```
이벤트 루프 틱 N:
  User1 Resolver → posts.load('user1')  (큐에 추가)
  User2 Resolver → posts.load('user2')  (큐에 추가)
  ...
  User100 Resolver → posts.load('user100') (큐에 추가)

이벤트 루프 틱 N+1:
  batchLoadPosts(['user1', 'user2', ..., 'user100'])
  → SELECT * FROM posts WHERE authorId IN (...)
  → 1번의 DB 쿼리로 해결!
```

## 스키마 설계 패턴

### Cursor-based Pagination

```graphql
type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type PostEdge {
  node: Post!
  cursor: String!   # base64(post.id + timestamp)
}

type PageInfo {
  hasNextPage: Boolean!
  endCursor: String
}

type Query {
  posts(first: Int, after: String): PostConnection!
}
```

```javascript
// Relay Cursor Connections Specification 구현
const postsResolver = async (_, { first = 10, after }) => {
  const decodedCursor = after ? Buffer.from(after, 'base64').toString() : null;
  
  const posts = await db.query(`
    SELECT * FROM posts
    WHERE ($1::text IS NULL OR id > $1)
    ORDER BY id ASC
    LIMIT $2
  `, [decodedCursor, first + 1]);

  const hasNextPage = posts.length > first;
  const edges = posts.slice(0, first).map(post => ({
    node: post,
    cursor: Buffer.from(String(post.id)).toString('base64'),
  }));

  return {
    edges,
    pageInfo: {
      hasNextPage,
      endCursor: edges.length > 0 ? edges[edges.length - 1].cursor : null,
    },
    totalCount: await db.one('SELECT COUNT(*) FROM posts'),
  };
};
```

## 성능 최적화와 주의사항

### 1. Persisted Queries

매 요청마다 쿼리 문자열 전체를 전송하면 네트워크 비용이 높습니다. **Persisted Queries**는 쿼리의 SHA256 해시만 전송하고 서버에서 캐시된 쿼리를 실행합니다.

```javascript
// Automatic Persisted Queries (APQ) 플로우
// 1차 요청: { extensions: { persistedQuery: { sha256Hash: "abc123" } } }
// 서버: 모름 → 404 반환
// 2차 요청: { query: "...", extensions: { persistedQuery: { sha256Hash: "abc123" } } }
// 서버: 저장 후 실행 → 이후 해시만으로 처리 가능
```

### 2. 쿼리 복잡도 제한

GraphQL은 클라이언트가 임의의 깊이로 쿼리를 중첩할 수 있어 악의적이거나 실수로 인한 폭발적 복잡도 쿼리가 발생할 수 있습니다.

```javascript
const depthLimit = require('graphql-depth-limit');
const { createComplexityLimitRule } = require('graphql-validation-complexity');

const server = new ApolloServer({
  validationRules: [
    depthLimit(7),  // 최대 깊이 7
    createComplexityLimitRule(1000, {
      // 필드별 복잡도 가중치
      scalarCost: 1,
      objectCost: 2,
      listFactor: 10,
    }),
  ],
});
```

### 3. 필드 레벨 권한 제어

REST는 엔드포인트별로 권한을 제어하지만 GraphQL은 필드별 제어가 필요합니다.

```javascript
const resolvers = {
  User: {
    email: (parent, args, context) => {
      // 본인 또는 관리자만 이메일 조회 가능
      if (context.user.id !== parent.id && !context.user.isAdmin) {
        return null;  // 또는 throw new ForbiddenError()
      }
      return parent.email;
    },
  },
};
```

### 4. N+1이 DataLoader로도 해결되지 않는 경우

DataLoader는 같은 배치 내에서만 효과적입니다. **순차 의존성**이 있는 경우(결과 A를 받아야 쿼리 B를 만들 수 있는 경우)는 배치가 불가능합니다. 이 경우 쿼리 계획(Query Planning) 레이어를 별도로 구현하거나 스키마를 재설계해야 합니다.

## GraphQL vs REST 선택 기준

| 기준 | GraphQL | REST |
|------|---------|------|
| 유연한 데이터 요청 | 강점 | 여러 엔드포인트 필요 |
| 캐싱 | 복잡 (POST 기반) | HTTP 캐시 활용 쉬움 |
| 파일 업로드 | 별도 처리 필요 | 단순 |
| 학습 곡선 | 높음 | 낮음 |
| 모바일 클라이언트 | 최적 | 과다 전송 발생 |
| 마이크로서비스 BFF | 적합 | 조율 복잡 |

## 마무리

GraphQL의 쿼리 실행 엔진은 파싱 → 검증 → 실행 → 직렬화의 4단계로 동작합니다. Resolver 기반의 재귀적 실행 모델은 강력하지만, 구조적으로 N+1 문제가 발생하기 쉽습니다. DataLoader는 이벤트 루프 틱을 활용한 배치와 캐시로 이 문제를 우아하게 해결합니다. Cursor 기반 페이지네이션, 쿼리 복잡도 제한, Persisted Queries를 함께 적용하면 프로덕션에서 안전하고 효율적인 GraphQL API를 운영할 수 있습니다.

## 참고 자료
- [GraphQL 공식 명세](https://spec.graphql.org/)
- [DataLoader GitHub 저장소](https://github.com/graphql/dataloader)
- [Apollo Server 공식 문서](https://www.apollographql.com/docs/apollo-server/)
- [Relay Cursor Connections Specification](https://relay.dev/graphql/connections.htm)
