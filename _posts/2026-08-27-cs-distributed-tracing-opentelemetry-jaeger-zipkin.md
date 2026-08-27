---
layout: post
title: "분산 추적 심화 — OpenTelemetry, Jaeger, Zipkin으로 마이크로서비스 가시성 확보하기"
date: 2026-08-27
categories: [cs, computer-science]
tags: [distributed-tracing, opentelemetry, jaeger, zipkin, observability, microservices, spans, context-propagation]
---

현대의 마이크로서비스 아키텍처에서 요청 하나가 수십 개의 서비스를 통과하는 상황은 흔한 일이 되었다. 사용자가 응답 지연을 겪을 때, 수백 개의 로그 파일을 뒤지는 방식으로는 병목지점을 찾기가 거의 불가능하다. 이때 **분산 추적(Distributed Tracing)** 이 그 해답을 제공한다.

## 개념 설명: 분산 추적이란?

분산 추적은 단일 사용자 요청이 분산 시스템의 다양한 서비스를 거치는 과정 전체를 추적하고 시각화하는 기술이다. 이를 이해하려면 세 가지 핵심 개념이 필요하다.

### Trace, Span, Context

**Trace**는 하나의 사용자 요청에 대한 완전한 여정이다. 예를 들어 사용자가 "상품 조회" 버튼을 눌렀을 때 API 게이트웨이 → 상품 서비스 → 재고 서비스 → 결제 서비스를 거치는 전체 흐름이 하나의 Trace가 된다. 각 Trace는 전역적으로 유일한 **TraceID**를 가진다.

**Span**은 하나의 단위 작업(operation)을 나타낸다. "상품 서비스의 DB 쿼리"가 하나의 Span이고, "재고 서비스 HTTP 호출"이 또 다른 Span이 된다. Span은 다음 정보를 포함한다:

- `SpanID`: 해당 Span을 식별하는 고유 ID
- `ParentSpanID`: 부모 Span의 ID (트리 구조 형성)
- `TraceID`: 속한 Trace의 ID
- `Operation Name`: 작업 이름 (예: "GET /products/{id}")
- `Start/End Timestamp`: 시작·종료 시각
- `Status`: 성공/실패
- `Attributes(Tags)`: 추가 메타데이터 (예: HTTP 상태 코드, DB 쿼리)
- `Events(Logs)`: Span 내 이벤트 로그

**Context Propagation**은 Span 간에 Trace 정보를 전파하는 메커니즘이다. 서비스 A가 서비스 B를 HTTP로 호출할 때, 요청 헤더에 TraceID와 SpanID를 심어 보내고, 서비스 B는 이 정보를 꺼내어 자신의 Span을 생성할 때 부모로 설정한다.

W3C가 표준화한 **Trace Context** 헤더 형식은 다음과 같다:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             버전 TraceID(16B hex)         SpanID(8B hex)      플래그
tracestate:  vendor=specific,custom=data
```

### OpenTelemetry란?

OpenTelemetry(OTel)는 CNCF(Cloud Native Computing Foundation) 산하의 오픈소스 프로젝트로, 분산 추적·메트릭·로그를 수집하는 표준 SDK와 API를 제공한다. Jaeger, Zipkin, Datadog, New Relic 등 다양한 백엔드로 데이터를 전송할 수 있는 벤더-중립적 레이어다.

OTel의 주요 구성요소:
- **API**: 계측(instrumentation) 인터페이스 정의
- **SDK**: API 구현체 (샘플링·익스포터 포함)
- **Collector**: 수집-변환-전송 파이프라인 역할의 독립 프로세스
- **Exporters**: Jaeger, Zipkin, OTLP 등 백엔드로 데이터 전송

## 왜 필요한가?

단일 모놀리식 애플리케이션에서는 스택 트레이스 하나로 에러의 근본 원인을 파악할 수 있다. 그러나 마이크로서비스 환경에서는:

- **로그의 파편화**: 서비스마다 별도의 로그 파일이 존재하며 상관관계 파악이 어렵다.
- **지연 원인 불명**: 응답 시간이 느려졌을 때 어느 서비스에서 지연이 발생했는지 알기 어렵다.
- **캐스케이딩 장애**: 서비스 A의 타임아웃이 서비스 B, C의 연쇄 장애를 일으키는 상황에서 인과관계 추적이 필요하다.
- **비동기 요청 추적**: 메시지 큐를 통한 비동기 처리에서는 요청 흐름이 더욱 불투명해진다.

분산 추적은 "이 요청이 얼마나 걸렸는가?"를 넘어, "어떤 서비스의 어떤 함수에서 왜 느렸는가?"를 시각적으로 답해준다.

## 실제 구현 예제

### 예제 1: Python OpenTelemetry 기본 계측

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource

# 1. TracerProvider 초기화
resource = Resource.create({"service.name": "product-service", "service.version": "1.0.0"})
provider = TracerProvider(resource=resource)

# 2. OTLP Exporter 설정 (Jaeger/Zipkin Collector로 전송)
exporter = OTLPSpanExporter(endpoint="http://otel-collector:4317", insecure=True)
provider.add_span_processor(BatchSpanProcessor(exporter))

# 3. 전역 TracerProvider 등록
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("product-service")

# 4. 실제 비즈니스 로직 계측
def get_product(product_id: str):
    with tracer.start_as_current_span("get_product") as span:
        # Span에 속성 추가
        span.set_attribute("product.id", product_id)
        span.set_attribute("db.system", "postgresql")

        try:
            # 하위 작업을 별도 Span으로 분리
            with tracer.start_as_current_span("db.query") as db_span:
                db_span.set_attribute("db.statement", f"SELECT * FROM products WHERE id='{product_id}'")
                result = db_query(product_id)  # 실제 DB 호출

            with tracer.start_as_current_span("cache.set") as cache_span:
                cache_span.set_attribute("cache.key", f"product:{product_id}")
                cache_set(product_id, result)

            span.set_attribute("product.name", result["name"])
            return result

        except Exception as e:
            # 에러 시 Span 상태를 ERROR로 설정
            span.record_exception(e)
            span.set_status(trace.Status(trace.StatusCode.ERROR, str(e)))
            raise

def db_query(product_id: str):
    # 실제 DB 쿼리 시뮬레이션
    import time
    time.sleep(0.05)  # 50ms 지연
    return {"id": product_id, "name": "Sample Product", "price": 9900}

def cache_set(key, value):
    pass  # Redis 캐시 저장 시뮬레이션

# 실행
product = get_product("P-12345")
print(f"상품명: {product['name']}")
```

이 코드는 `get_product` 함수 호출 전체를 하나의 Span으로 계측하고, 내부의 DB 쿼리와 캐시 저장을 별도 자식 Span으로 분리한다. Jaeger UI에서 이를 열면 **워터폴 차트** 형태로 각 작업의 소요 시간을 한눈에 볼 수 있다.

### 예제 2: HTTP 서비스 간 컨텍스트 전파 (FastAPI)

```python
import httpx
from fastapi import FastAPI, Request
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry import trace, context, propagate

app = FastAPI()

# 자동 계측 활성화 (모든 FastAPI 요청을 자동으로 Span으로 감쌈)
FastAPIInstrumentor.instrument_app(app)
HTTPXClientInstrumentor().instrument()  # HTTPX 클라이언트도 자동 계측

tracer = trace.get_tracer("order-service")

@app.post("/orders")
async def create_order(request: Request, item_id: str, quantity: int):
    """주문 생성 — 재고 서비스와 결제 서비스 호출"""

    with tracer.start_as_current_span("create_order") as span:
        span.set_attribute("order.item_id", item_id)
        span.set_attribute("order.quantity", quantity)

        # HTTPXClientInstrumentor가 자동으로 traceparent 헤더를 주입함
        async with httpx.AsyncClient() as client:
            # 재고 서비스 호출 — 컨텍스트가 자동으로 전파됨
            inventory_resp = await client.get(
                f"http://inventory-service/stock/{item_id}"
            )
            inventory = inventory_resp.json()

            if inventory["quantity"] < quantity:
                span.set_attribute("order.result", "out_of_stock")
                span.set_status(trace.Status(trace.StatusCode.ERROR, "재고 부족"))
                return {"error": "재고 부족"}

            # 결제 서비스 호출
            payment_resp = await client.post(
                "http://payment-service/charge",
                json={"item_id": item_id, "quantity": quantity}
            )
            payment = payment_resp.json()

            span.set_attribute("order.payment_id", payment["payment_id"])
            span.set_attribute("order.result", "success")
            return {"order_id": payment["payment_id"], "status": "confirmed"}

# 수동 컨텍스트 전파 예시 (메시지 큐 사용 시)
def publish_to_queue(queue_client, message: dict):
    """Kafka 등 메시지 큐에 컨텍스트 헤더를 직접 심는 예시"""
    headers = {}
    propagate.inject(headers)  # 현재 Span 컨텍스트를 헤더 딕셔너리에 직렬화
    queue_client.send("orders", value=message, headers=list(headers.items()))

def consume_from_queue(raw_headers: dict):
    """Kafka 메시지를 소비하며 컨텍스트를 복원하는 예시"""
    ctx = propagate.extract(raw_headers)  # 헤더에서 컨텍스트 역직렬화
    with tracer.start_as_current_span("process_order_event", context=ctx) as span:
        span.set_attribute("messaging.system", "kafka")
        # 이 Span은 publish_to_queue의 Span을 부모로 연결됨
        process_order()

def process_order():
    pass
```

HTTP 호출 시 `HTTPXClientInstrumentor`가 자동으로 `traceparent` 헤더를 요청에 추가하므로, 서비스 간 Trace 연결이 투명하게 이루어진다. Kafka처럼 헤더 자동 전파가 없는 경우에는 `propagate.inject`/`propagate.extract`로 수동으로 처리한다.

## 주의사항과 팁

### 1. 샘플링 전략 선택

모든 요청을 추적하면 오버헤드가 크다. 세 가지 샘플링 전략이 있다:

- **Head-based Sampling (확률 샘플링)**: 요청이 시작될 때 추적 여부를 결정. 단순하지만 에러 요청을 놓칠 수 있음.
- **Tail-based Sampling**: 요청이 완료된 후 에러·지연 여부를 보고 추적 여부 결정. 정확하지만 메모리 사용 증가.
- **Parent-based Sampling**: 부모 서비스가 추적 중이면 자식도 추적. 분산 환경에서 일관성 유지.

프로덕션에서는 에러 요청 100% + 정상 요청 1~5% 혼합 전략을 권장한다.

### 2. Span 세분화 주의

너무 많은 Span을 생성하면 오히려 가시성을 해친다. 일반적으로:
- 외부 서비스 호출(HTTP, gRPC, 메시지 큐)
- DB 쿼리 (각 쿼리별)
- 중요한 비즈니스 로직 단위

위 세 가지 수준에서 Span을 생성하는 것이 적절하다. 단순한 함수 호출 하나하나를 Span으로 만들 필요는 없다.

### 3. Cardinality 관리

Span 속성(attribute)의 카디널리티가 높으면 백엔드 저장소에 부하가 생긴다. `user.id`처럼 값이 무한히 다양한 속성은 신중하게 사용하고, `http.status_code`처럼 값이 제한적인 속성을 우선시한다.

### 4. Baggage와 CorrelationID 활용

Baggage는 Span 컨텍스트와 함께 전파되는 key-value 페어로, 사용자 ID나 요청 ID 같은 비즈니스 컨텍스트를 서비스 간에 전달할 때 유용하다. 단, 모든 Span에 첨부되므로 크기를 최소화해야 한다.

```python
from opentelemetry.baggage import set_baggage, get_baggage
from opentelemetry import context

# Baggage 설정 (A 서비스)
ctx = set_baggage("user.id", "user-42")
ctx = set_baggage("request.id", "req-abc123", context=ctx)

# B 서비스에서 컨텍스트 전파 후 읽기
user_id = get_baggage("user.id")  # "user-42"
```

### 5. Jaeger vs Zipkin 선택 가이드

| 항목 | Jaeger | Zipkin |
|------|--------|--------|
| 개발사 | Uber (CNCF 졸업) | Twitter |
| 저장소 | Cassandra, Elasticsearch, OpenSearch | MySQL, Elasticsearch, Cassandra |
| UI | 풍부한 필터·비교 기능 | 단순하고 경량 |
| OTLP 지원 | 완전 지원 | 지원 (Zipkin 형식 선호) |
| 적합한 규모 | 중대형 마이크로서비스 환경 | 소규모 빠른 시작 |

두 백엔드 모두 OTel Collector를 통해 데이터를 수신하므로, SDK 코드 변경 없이 백엔드만 교체할 수 있다.

## 참고 자료
- [OpenTelemetry 공식 문서 — Traces 개념](https://opentelemetry.io/docs/concepts/signals/traces/)
- [W3C Trace Context 표준](https://www.w3.org/TR/trace-context/)
- [Jaeger 공식 문서](https://www.jaegertracing.io/docs/)
- [OpenTelemetry Context Propagation 가이드](https://opentelemetry.io/docs/concepts/context-propagation/)
