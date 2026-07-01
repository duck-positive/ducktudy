---
layout: post
title: "gRPC와 Protocol Buffers 완전 정복: HTTP/2 기반 고성능 RPC 내부 구조"
date: 2026-07-01
categories: [cs, computer-science]
tags: [gRPC, protobuf, protocol-buffers, HTTP2, RPC, serialization, microservices, streaming]
---

마이크로서비스 아키텍처가 보편화되면서 서비스 간 통신의 효율성이 시스템 성능의 핵심 변수가 되었습니다. REST/JSON은 개발자 친화적이지만 직렬화 오버헤드와 스키마 부재라는 약점이 있습니다. **gRPC**와 **Protocol Buffers**는 이 문제를 정면돌파합니다. Google이 내부 마이크로서비스에서 사용하던 Stubby RPC를 오픈소스화한 gRPC는 현재 CNCF(Cloud Native Computing Foundation)의 주요 프로젝트로, Kubernetes, Envoy, Istio 등 클라우드 네이티브 생태계에 깊이 통합되어 있습니다.

## gRPC란 무엇인가?

gRPC는 **g**oogle **R**emote **P**rocedure **C**all의 약자입니다. 클라이언트가 마치 로컬 함수를 호출하듯 원격 서버의 메서드를 호출할 수 있게 해주는 프레임워크입니다. 세 가지 핵심 기술이 결합되어 있습니다:

| 기술 | 역할 |
|------|------|
| **Protocol Buffers** | IDL(인터페이스 정의 언어) + 직렬화 포맷 |
| **HTTP/2** | 전송 프로토콜 (멀티플렉싱, 헤더 압축) |
| **코드 생성** | protoc 컴파일러가 클라이언트/서버 스텁 자동 생성 |

## Protocol Buffers: 이진 직렬화의 핵심

### varint 인코딩

Protobuf의 성능 비결은 **varint** (Variable-length integer) 인코딩입니다. 작은 숫자는 1바이트로, 큰 숫자는 최대 10바이트로 인코딩합니다.

```
숫자 1    → 0x01          (1바이트)
숫자 300  → 0xAC 0x02     (2바이트)
숫자 1000 → 0xE8 0x07     (2바이트)
```

각 바이트의 최상위 비트(MSB)는 "다음 바이트가 있는가" 여부를 나타내는 continuation bit입니다. 나머지 7비트가 실제 데이터입니다.

### 필드 인코딩

```
field_number << 3 | wire_type
```

각 필드는 (field number, wire type) 쌍으로 식별됩니다:

| Wire Type | 값 | 사용처 |
|-----------|---|--------|
| Varint | 0 | int32, int64, bool, enum |
| 64-bit | 1 | fixed64, double |
| Length-delimited | 2 | string, bytes, embedded messages |
| 32-bit | 5 | fixed32, float |

타입 정보 없이 필드 번호로만 식별하므로, 스키마가 없으면 디코딩이 어렵습니다 — 보안 면에서도 장점입니다.

## HTTP/2가 가져온 혁명

gRPC가 HTTP/2를 선택한 이유는 단순히 "새 버전이라서"가 아닙니다.

### 멀티플렉싱

HTTP/1.1에서는 하나의 TCP 연결에서 동시에 하나의 요청/응답만 처리할 수 있습니다(HOL blocking). HTTP/2는 **스트림(Stream)** 개념을 도입하여 하나의 TCP 연결에서 수백 개의 요청을 동시에 처리합니다.

```
HTTP/1.1:  [요청1] → [응답1] → [요청2] → [응답2]  (순차)
HTTP/2:    Stream 1: [요청1] ←→ [응답1]
           Stream 3: [요청2] ←→ [응답2]   (동시)
           Stream 5: [요청3] ←→ [응답3]
           모두 같은 TCP 연결에서!
```

### gRPC 스트리밍 4가지 패턴

```protobuf
service ChatService {
  // Unary: 요청 1개, 응답 1개
  rpc GetUser (UserRequest) returns (UserResponse);
  
  // Server Streaming: 요청 1개, 응답 여러 개
  rpc ListPosts (ListRequest) returns (stream PostResponse);
  
  // Client Streaming: 요청 여러 개, 응답 1개
  rpc UploadChunks (stream ChunkRequest) returns (UploadResult);
  
  // Bidirectional Streaming: 요청/응답 모두 스트림
  rpc Chat (stream Message) returns (stream Message);
}
```

## 실제 구현 예제

### 예제 1: .proto 정의와 Python 서버/클라이언트

```protobuf
// user_service.proto
syntax = "proto3";

package user;

option go_package = "./pb;user";

message User {
  int64  id       = 1;
  string username = 2;
  string email    = 3;
  int32  age      = 4;
  repeated string roles = 5;  // 반복 필드
}

message GetUserRequest {
  int64 user_id = 1;
}

message SearchUsersRequest {
  string  query    = 1;
  int32   page     = 2;
  int32   page_size = 3;
}

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc SearchUsers (SearchUsersRequest) returns (stream User);
  rpc CreateUsers (stream User) returns (CreateResult);
}

message CreateResult {
  int32 created_count = 1;
  repeated string errors = 2;
}
```

```python
# server.py
from concurrent import futures
import grpc
import time
import user_service_pb2 as pb
import user_service_pb2_grpc as pb_grpc

# 인메모리 데이터베이스
USERS = {
    1: pb.User(id=1, username="alice", email="alice@example.com", age=30, roles=["admin"]),
    2: pb.User(id=2, username="bob",   email="bob@example.com",   age=25, roles=["user"]),
    3: pb.User(id=3, username="carol", email="carol@example.com", age=28, roles=["user", "editor"]),
}

class UserServicer(pb_grpc.UserServiceServicer):
    def GetUser(self, request, context):
        """Unary RPC: 단일 사용자 조회"""
        user = USERS.get(request.user_id)
        if user is None:
            context.abort(grpc.StatusCode.NOT_FOUND, f"User {request.user_id} not found")
        return user
    
    def SearchUsers(self, request, context):
        """Server Streaming RPC: 검색 결과를 스트림으로 전송"""
        query = request.query.lower()
        for user in USERS.values():
            if query in user.username.lower() or query in user.email.lower():
                time.sleep(0.1)  # DB 쿼리 시뮬레이션
                yield user
    
    def CreateUsers(self, request_iterator, context):
        """Client Streaming RPC: 여러 사용자를 한 번에 생성"""
        created = 0
        errors = []
        for user in request_iterator:
            if user.id in USERS:
                errors.append(f"User {user.id} already exists")
            else:
                USERS[user.id] = user
                created += 1
        return pb.CreateResult(created_count=created, errors=errors)

def serve():
    server = grpc.server(
        futures.ThreadPoolExecutor(max_workers=10),
        options=[
            ('grpc.max_send_message_length', 50 * 1024 * 1024),  # 50MB
            ('grpc.max_receive_message_length', 50 * 1024 * 1024),
            ('grpc.keepalive_time_ms', 10000),
        ]
    )
    pb_grpc.add_UserServiceServicer_to_server(UserServicer(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    print("gRPC server started on :50051")
    server.wait_for_termination()

if __name__ == '__main__':
    serve()
```

```python
# client.py
import grpc
import user_service_pb2 as pb
import user_service_pb2_grpc as pb_grpc

def run():
    # 채널 생성 — 실제 TCP 연결은 필요할 때 lazy하게 생성됨
    with grpc.insecure_channel(
        'localhost:50051',
        options=[('grpc.enable_retries', 1)]
    ) as channel:
        stub = pb_grpc.UserServiceStub(channel)
        
        # 1) Unary RPC
        print("=== Unary RPC: GetUser ===")
        try:
            user = stub.GetUser(pb.GetUserRequest(user_id=1))
            print(f"Got: {user.username} ({user.email}), roles: {list(user.roles)}")
        except grpc.RpcError as e:
            print(f"Error: {e.code()} - {e.details()}")
        
        # 2) Server Streaming RPC
        print("\n=== Server Streaming: SearchUsers ===")
        for user in stub.SearchUsers(pb.SearchUsersRequest(query="a", page=1, page_size=10)):
            print(f"  Found: {user.username}")
        
        # 3) Client Streaming RPC
        print("\n=== Client Streaming: CreateUsers ===")
        new_users = [
            pb.User(id=4, username="dave",  email="dave@example.com",  age=35),
            pb.User(id=5, username="eve",   email="eve@example.com",   age=22),
        ]
        result = stub.CreateUsers(iter(new_users))
        print(f"Created: {result.created_count}, Errors: {list(result.errors)}")

if __name__ == '__main__':
    run()
```

### 예제 2: Protobuf 직렬화 성능 비교

Protobuf와 JSON의 직렬화 크기 및 속도를 직접 비교합니다.

```python
import json
import time
import struct
import sys

# 순수 Python으로 Protobuf varint 인코딩 구현
def encode_varint(value: int) -> bytes:
    bits = value & 0x7f
    value >>= 7
    result = []
    while value:
        result.append(0x80 | bits)
        bits = value & 0x7f
        value >>= 7
    result.append(bits)
    return bytes(result)

def encode_string(field_num: int, s: str) -> bytes:
    encoded = s.encode('utf-8')
    tag = encode_varint((field_num << 3) | 2)  # wire_type=2 (length-delimited)
    length = encode_varint(len(encoded))
    return tag + length + encoded

def encode_int32(field_num: int, value: int) -> bytes:
    tag = encode_varint((field_num << 3) | 0)  # wire_type=0 (varint)
    return tag + encode_varint(value)

# 테스트 데이터
users = [
    {"id": i, "username": f"user{i}", "email": f"user{i}@example.com", "age": 20 + (i % 50)}
    for i in range(1000)
]

# JSON 직렬화
start = time.perf_counter()
json_payloads = [json.dumps(u).encode() for u in users]
json_time = time.perf_counter() - start
json_total_size = sum(len(p) for p in json_payloads)

# 수동 Protobuf 직렬화 (필드 1~4)
def serialize_user_proto(user: dict) -> bytes:
    payload = b""
    payload += encode_int32(1, user["id"])           # field 1: id
    payload += encode_string(2, user["username"])     # field 2: username
    payload += encode_string(3, user["email"])        # field 3: email
    payload += encode_int32(4, user["age"])           # field 4: age
    return payload

start = time.perf_counter()
proto_payloads = [serialize_user_proto(u) for u in users]
proto_time = time.perf_counter() - start
proto_total_size = sum(len(p) for p in proto_payloads)

print("=== 직렬화 크기 비교 (1000 users) ===")
print(f"JSON:   {json_total_size:,} bytes ({json_total_size / 1000:.1f} bytes/user)")
print(f"Proto:  {proto_total_size:,} bytes ({proto_total_size / 1000:.1f} bytes/user)")
print(f"압축률: {(1 - proto_total_size/json_total_size)*100:.1f}% 더 작음")

print(f"\n=== 직렬화 속도 비교 ===")
print(f"JSON:   {json_time*1000:.2f}ms")
print(f"Proto:  {proto_time*1000:.2f}ms")
print(f"속도:   Proto가 {json_time/proto_time:.1f}x 빠름")

# varint 크기 예시
print("\n=== Varint 인코딩 예시 ===")
for n in [1, 127, 128, 300, 16383, 16384]:
    encoded = encode_varint(n)
    print(f"  {n:6d} → {encoded.hex():12s} ({len(encoded)} bytes)")
```

## gRPC 채널과 연결 관리

```python
# interceptor를 이용한 메타데이터(헤더), 재시도, 타임아웃 처리
import grpc
from grpc import UnaryUnaryClientInterceptor

class LoggingInterceptor(UnaryUnaryClientInterceptor):
    def intercept_unary_unary(self, continuation, client_call_details, request):
        # 요청에 인증 토큰 추가 (HTTP/2 헤더로 전송)
        metadata = list(client_call_details.metadata or [])
        metadata.append(('authorization', 'Bearer my-jwt-token'))
        metadata.append(('x-request-id', 'trace-12345'))
        
        new_details = client_call_details._replace(metadata=metadata)
        
        import time
        start = time.monotonic()
        response = continuation(new_details, request)
        elapsed = time.monotonic() - start
        
        print(f"  RPC {client_call_details.method}: {elapsed*1000:.1f}ms")
        return response

# 채널에 인터셉터 적용
channel = grpc.intercept_channel(
    grpc.insecure_channel('localhost:50051'),
    LoggingInterceptor()
)
```

## 주의사항 및 팁

**1. `.proto` 파일 하위 호환성 유지**  
필드를 삭제하면 안 됩니다. 삭제하면 해당 필드 번호를 `reserved`로 표시하세요. 기존 클라이언트가 알 수 없는 필드를 보내도 Protobuf는 이를 무시(unknown fields)하므로 기존 서버는 정상 동작합니다.

**2. gRPC 헬스 체크**  
표준 `grpc.health.v1.Health` 프로토콜을 구현하세요. Kubernetes liveness/readiness probe가 이를 인식합니다.

**3. deadline/timeout 반드시 설정**  
gRPC 호출에는 항상 deadline을 설정하세요. 없으면 서버가 무한 대기하여 리소스를 고갈시킵니다:
```python
response = stub.GetUser(request, timeout=5.0)  # 5초 타임아웃
```

**4. 에러 코드 활용**  
HTTP 상태 코드 대신 gRPC Status Code를 사용하세요: `NOT_FOUND`, `PERMISSION_DENIED`, `RESOURCE_EXHAUSTED`, `DEADLINE_EXCEEDED` 등. 클라이언트가 에러 유형을 명확히 구분할 수 있습니다.

**5. REST Gateway**  
`grpc-gateway`를 사용하면 동일한 .proto에서 REST API와 gRPC API를 동시에 제공할 수 있습니다. 브라우저 클라이언트는 REST로, 서비스 간 통신은 gRPC로.

## 참고 자료

- [gRPC 공식 문서: Introduction to gRPC](https://grpc.io/docs/what-is-grpc/introduction/)
- [Protocol Buffers 공식 문서: Language Guide (proto3)](https://protobuf.dev/programming-guides/proto3/)
- [Protocol Buffers Overview](https://protobuf.dev/overview/)
- [gRPC Services with C# — Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/grpc/basics)
