---
layout: post
title: "OpenTelemetry 완전 정복: 분산 추적·메트릭·로깅을 통합하는 현대 관측 가능성의 표준"
date: 2026-10-19
categories: [cs, computer-science]
tags: [opentelemetry, distributed-tracing, observability, metrics, logging, span, jaeger, prometheus]
---

마이크로서비스 시대에는 하나의 요청이 수십 개의 서비스를 거칩니다. 어느 서비스에서 지연이 발생했는지, 에러가 어디서 시작됐는지 파악하는 일은 수백 개의 로그를 손으로 뒤지는 고통스러운 과정이었습니다. OpenTelemetry는 이 문제를 해결하기 위해 CNCF(Cloud Native Computing Foundation)가 표준화한 관측 가능성(Observability) 프레임워크입니다. 이 글에서는 OpenTelemetry의 핵심 개념과 내부 동작 원리, 그리고 실제 구현 방법을 깊이 있게 다룹니다.

## 관측 가능성의 세 기둥

관측 가능성은 세 가지 신호(signal)로 구성됩니다:

- **Traces (추적)**: 요청이 시스템을 통과하는 인과 관계 정보. "이 HTTP 요청이 어떤 서비스, 어떤 함수, 어떤 DB 쿼리를 거쳤는가?"
- **Metrics (메트릭)**: 시계열 집계 데이터. "초당 요청 수, 평균 응답 시간, 에러율은 얼마인가?"
- **Logs (로그)**: 이벤트의 상세 기록. "구체적으로 어떤 일이 일어났는가?"

OpenTelemetry(OTel)의 혁신은 이 세 가지를 **하나의 통합된 API·SDK·프로토콜(OTLP)**로 표준화한 것입니다.

## Trace와 Span의 데이터 모델

### Span 구조

Span은 분산 추적의 기본 단위로, 하나의 작업(operation)을 나타냅니다.

```json
{
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "00f067aa0ba902b7",
  "parentSpanId": "b7ad6b7169203331",
  "traceState": "rojo=00f067aa0ba902b7",
  "name": "HTTP GET /api/users/{id}",
  "kind": "SERVER",
  "startTimeUnixNano": "1544712660000000000",
  "endTimeUnixNano": "1544712661000000000",
  "attributes": [
    {"key": "http.method", "value": {"stringValue": "GET"}},
    {"key": "http.status_code", "value": {"intValue": 200}},
    {"key": "http.url", "value": {"stringValue": "https://api.example.com/api/users/42"}},
    {"key": "db.system", "value": {"stringValue": "postgresql"}},
    {"key": "db.statement", "value": {"stringValue": "SELECT * FROM users WHERE id = $1"}}
  ],
  "events": [
    {
      "timeUnixNano": "1544712660500000000",
      "name": "cache.miss",
      "attributes": [{"key": "cache.key", "value": {"stringValue": "user:42"}}]
    }
  ],
  "status": {"code": "OK"},
  "links": []
}
```

### Trace 구성

여러 Span이 `traceId`로 연결되어 하나의 Trace를 형성합니다:

```
Trace: 4bf92f3577b34da6a3ce929d0e0e4736
│
└─ [0ms~120ms] API Gateway (SERVER span)
   │  spanId: aaaaaa
   │
   ├─ [5ms~80ms] UserService.getUser (CLIENT span)
   │  │  spanId: bbbbbb, parentSpanId: aaaaaa
   │  │
   │  ├─ [6ms~15ms] Redis.get (CLIENT span)
   │  │    spanId: cccccc, parentSpanId: bbbbbb
   │  │    → cache.miss 이벤트
   │  │
   │  └─ [20ms~75ms] PostgreSQL.query (CLIENT span)
   │       spanId: dddddd, parentSpanId: bbbbbb
   │
   └─ [85ms~115ms] AuditService.log (CLIENT span)
        spanId: eeeeee, parentSpanId: aaaaaa
```

## 컨텍스트 전파(Context Propagation)

분산 추적의 핵심 메커니즘은 서비스 간 **컨텍스트 전파**입니다. `traceId`와 `spanId`가 HTTP 헤더를 통해 다음 서비스로 전달됩니다.

### W3C Trace Context 표준

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^ ^^
             버전  traceId (16바이트 hex)          spanId (8바이트) 플래그
```

플래그 비트:
- `01` = sampled (이 트레이스를 수집함)
- `00` = not sampled (수집 안 함)

### Python 구현 예제

```python
from opentelemetry import trace, baggage
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.propagate import inject, extract
import requests

# 1. TracerProvider 설정
exporter = OTLPSpanExporter(endpoint="http://jaeger:4317")
provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)

# 2. 자동 계측: requests 라이브러리 자동 추적
RequestsInstrumentor().instrument()

tracer = trace.get_tracer("my-service", "1.0.0")

def get_user(user_id: int) -> dict:
    with tracer.start_as_current_span(
        "UserService.getUser",
        kind=trace.SpanKind.SERVER,
        attributes={
            "user.id": user_id,
            "service.name": "user-service",
        }
    ) as span:
        with tracer.start_as_current_span("redis.get") as cache_span:
            cache_key = f"user:{user_id}"
            cache_span.set_attribute("cache.key", cache_key)
            result = redis_client.get(cache_key)
            
            if result is None:
                cache_span.add_event("cache.miss", attributes={"key": cache_key})
                response = requests.get(f"http://db-service/users/{user_id}")
                user_data = response.json()
                redis_client.setex(cache_key, 300, str(user_data))
            else:
                cache_span.add_event("cache.hit")
                user_data = eval(result)
        
        span.set_attribute("user.found", True)
        return user_data

def handle_request(request):
    ctx = extract(request.headers)
    with tracer.start_as_current_span(
        "handle_request",
        context=ctx,
        kind=trace.SpanKind.SERVER
    ) as span:
        user_id = int(request.params.get("id"))
        return get_user(user_id)
```

## 메트릭(Metrics) 시스템

OTel 메트릭은 세 가지 기본 계측기(instrument)를 제공합니다:

| 계측기 | 용도 | 예시 |
|---|---|---|
| Counter | 단조 증가 카운터 | HTTP 요청 수, 에러 수 |
| Gauge | 순간적 측정값 | 메모리 사용량, 큐 크기 |
| Histogram | 분포 측정 | 응답 시간, 페이로드 크기 |

### Java 구현 예제

```java
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.metrics.*;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.metrics.SdkMeterProvider;
import io.opentelemetry.exporter.prometheus.PrometheusHttpServer;
import java.util.concurrent.atomic.AtomicLong;

public class MetricsExample {
    private final Meter meter;
    private final LongCounter requestCounter;
    private final LongCounter errorCounter;
    private final DoubleHistogram latencyHistogram;
    private final AtomicLong activeConnCount = new AtomicLong(0);
    
    public MetricsExample(OpenTelemetry openTelemetry) {
        this.meter = openTelemetry.getMeter("my-service", "1.0.0");
        
        this.requestCounter = meter
            .counterBuilder("http.server.requests.total")
            .setDescription("Total number of HTTP requests")
            .setUnit("requests")
            .build();
        
        this.errorCounter = meter
            .counterBuilder("http.server.errors.total")
            .setDescription("Total number of HTTP errors")
            .build();
        
        this.latencyHistogram = meter
            .histogramBuilder("http.server.request.duration")
            .setDescription("HTTP request latency")
            .setUnit("ms")
            .setExplicitBucketBoundariesAdvice(
                List.of(5.0, 10.0, 25.0, 50.0, 100.0, 250.0, 500.0, 1000.0)
            )
            .build();
        
        meter.gaugeBuilder("http.server.active_connections")
            .ofLongs()
            .buildWithCallback(obs -> obs.record(activeConnCount.get()));
    }
    
    public void handleRequest(String method, String path, Runnable handler) {
        long start = System.currentTimeMillis();
        activeConnCount.incrementAndGet();
        
        Attributes attrs = Attributes.of(
            AttributeKey.stringKey("http.method"), method,
            AttributeKey.stringKey("http.route"), path
        );
        
        try {
            handler.run();
            requestCounter.add(1, attrs);
        } catch (Exception e) {
            errorCounter.add(1, attrs.toBuilder()
                .put("error.type", e.getClass().getSimpleName())
                .build());
            throw e;
        } finally {
            long duration = System.currentTimeMillis() - start;
            latencyHistogram.record(duration, attrs);
            activeConnCount.decrementAndGet();
        }
    }
    
    public static void main(String[] args) throws Exception {
        PrometheusHttpServer prometheusServer = PrometheusHttpServer
            .builder().setPort(9090).build();
        
        SdkMeterProvider meterProvider = SdkMeterProvider.builder()
            .registerMetricReader(prometheusServer).build();
        
        OpenTelemetry otel = OpenTelemetrySdk.builder()
            .setMeterProvider(meterProvider).build();
        
        MetricsExample app = new MetricsExample(otel);
        System.out.println("Prometheus 메트릭 노출: http://localhost:9090/metrics");
    }
}
```

## OTLP: OpenTelemetry Protocol

OTel은 벤더 중립적인 자체 프로토콜 **OTLP(OpenTelemetry Protocol)**를 정의합니다. Protobuf + gRPC 또는 HTTP/JSON으로 전송되며, 모든 백엔드(Jaeger, Zipkin, Prometheus, Datadog 등)가 OTLP를 지원합니다.

```
애플리케이션 SDK
      ↓  OTLP/gRPC
  [OpenTelemetry Collector]  ← 수집·필터링·라우팅·집계 담당
      ↓ 팬아웃 export
  ┌─────────────────────────────────┐
  │  Jaeger (traces)                │
  │  Prometheus (metrics) ← /scrape │
  │  Elasticsearch (logs)           │
  │  Grafana Tempo (traces)         │
  └─────────────────────────────────┘
```

### OpenTelemetry Collector 설정

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: "0.0.0.0:4317"
      http:
        endpoint: "0.0.0.0:4318"

processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
    spike_limit_mib: 128
  batch:
    send_batch_size: 1000
    timeout: 10s
  resource:
    attributes:
      - key: environment
        value: production
        action: insert
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors-policy
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: slow-traces
        type: latency
        latency: {threshold_ms: 1000}
      - name: probabilistic-10pct
        type: probabilistic
        probabilistic: {sampling_percentage: 10}

exporters:
  jaeger:
    endpoint: "jaeger:14250"
    tls:
      insecure: true
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: myapp
  logging:
    loglevel: warn

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, tail_sampling]
      exporters: [jaeger, logging]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch, resource]
      exporters: [prometheus]
```

## 샘플링 전략

고트래픽 환경에서는 모든 트레이스를 수집하면 스토리지와 처리 비용이 폭발합니다. 샘플링 전략이 중요합니다:

- **Head Sampling (헤드 샘플링)**: 요청 시작 시 샘플링 여부 결정. 구현이 간단하지만 에러 트레이스를 놓칠 수 있음.
- **Tail Sampling (테일 샘플링)**: 전체 트레이스가 완성된 후 샘플링 여부 결정. Collector에서 구현. 에러·느린 요청은 100% 보존 가능.

## 주의사항과 실전 팁

### 1. 계측 과잉(Over-instrumentation) 주의
모든 함수에 Span을 붙이면 오버헤드가 급증합니다. I/O 경계(DB 쿼리, HTTP 호출, 큐 발행)와 비즈니스 핵심 경로에만 집중하세요.

### 2. 민감 정보 필터링
HTTP 헤더, 쿼리 파라미터, 요청 본문에는 민감 정보가 포함될 수 있습니다. Collector의 `attributes` 프로세서로 `Authorization`, `password` 등을 필터링하세요.

### 3. 카디널리티 폭발 방지
`user_id`를 메트릭 레이블로 쓰면 메트릭 수가 사용자 수만큼 늘어납니다. 레이블값은 소수의 고정 값(상태 코드, 라우트 패턴)으로 제한하세요.

### 4. 자동 계측 활용
대부분의 프레임워크(Spring, Django, Express, gRPC, JDBC 등)는 자동 계측 라이브러리를 제공합니다. 코드 변경 없이 계측을 시작하고, 이후 비즈니스 로직에 수동 계측을 추가하는 접근이 효과적입니다.

## 정리

OpenTelemetry는 단순한 추적 라이브러리가 아니라 Trace·Metric·Log를 통합하는 관측 가능성 표준입니다. W3C Trace Context로 서비스 간 컨텍스트를 전파하고, OTLP로 벤더에 종속되지 않는 데이터 파이프라인을 구축하며, Collector로 유연한 라우팅과 샘플링을 구현합니다. 마이크로서비스 아키텍처에서 관측 가능성을 갖추는 것은 선택이 아닌 필수이며, OpenTelemetry는 그 표준 답안입니다.

## 참고 자료
- [OpenTelemetry 공식 스펙 개요](https://opentelemetry.io/docs/specs/otel/overview/)
- [OpenTelemetry Spans 심화 가이드 (SigNoz)](https://signoz.io/blog/opentelemetry-spans)
- [W3C Trace Context 표준](https://www.w3.org/TR/trace-context/)
- [OpenTelemetry Collector 공식 문서](https://opentelemetry.io/docs/collector/)
