---
layout: post
title: "서비스 메시(Service Mesh) 완전 정복: Envoy·Istio 사이드카 프록시와 데이터 플레인의 내부 구조"
date: 2026-10-19
categories: [cs, computer-science]
tags: [service-mesh, envoy, istio, sidecar, data-plane, control-plane, xds, mtls, kubernetes]
---

마이크로서비스 아키텍처가 수십~수백 개의 서비스로 확장되면, 서비스 간 통신을 관리하는 것이 새로운 도전이 됩니다. 서킷 브레이커, 재시도 정책, mTLS 암호화, 트래픽 분산, 관측 가능성 등을 각 서비스마다 직접 구현하는 것은 비효율적이고 일관성을 보장하기 어렵습니다. **서비스 메시(Service Mesh)**는 이런 네트워크 기능을 인프라 레이어로 분리하여 애플리케이션 코드 변경 없이 제공합니다. 이 글에서는 서비스 메시의 핵심 원리와 Envoy·Istio의 내부 구조를 심층 분석합니다.

## 왜 서비스 메시가 필요한가

전통적인 마이크로서비스에서는 각 서비스가 네트워크 로직을 직접 구현해야 했습니다:

```javascript
// 각 서비스마다 반복되는 코드
async function callUserService(userId) {
    const circuitBreaker = new CircuitBreaker({ threshold: 50, timeout: 5000 });
    const retryPolicy = new RetryPolicy({ maxRetries: 3, backoff: 'exponential' });
    const timeout = new Timeout({ ms: 1000 });
    
    return retryPolicy.execute(() =>
        circuitBreaker.execute(() =>
            timeout.execute(() =>
                axios.get(`http://user-service/users/${userId}`, {
                    headers: { 'X-Trace-ID': getTraceId() }
                })
            )
        )
    );
}
```

서비스 메시는 이 로직을 **사이드카 프록시**로 외부화합니다:

```javascript
// 서비스 메시 사용 후:
const user = await axios.get(`http://user-service/users/${userId}`);
// 재시도, 서킷브레이커, mTLS, 추적은 프록시가 자동 처리
```

## 아키텍처 개요

서비스 메시는 **데이터 플레인(Data Plane)**과 **컨트롤 플레인(Control Plane)**으로 분리됩니다.

```
┌─────────────────────────────────────────────────────────────┐
│                    컨트롤 플레인 (Istiod)                     │
│   ┌──────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│   │  Pilot   │  │    Citadel   │  │       Galley         │ │
│   │ (라우팅) │  │  (인증서/ID) │  │  (설정 검증/배포)    │ │
│   └─────┬────┘  └──────┬───────┘  └──────────────────────┘ │
└─────────┼───────────────┼───────────────────────────────────┘
          │ xDS API       │ 인증서
┌─────────▼───────────────▼───────────────────────────────────┐
│                    데이터 플레인                               │
│   Pod A                         Pod B                       │
│ ┌─────────────────┐           ┌─────────────────┐           │
│ │  App Container  │           │  App Container  │           │
│ ├─────────────────┤           ├─────────────────┤           │
│ │  Envoy Sidecar  │──mTLS────▶│  Envoy Sidecar  │           │
│ └─────────────────┘           └─────────────────┘           │
└─────────────────────────────────────────────────────────────┘
```

## Envoy 프록시 내부 구조

Envoy는 C++로 작성된 고성능 L4/L7 프록시로, 서비스 메시의 핵심 데이터 플레인입니다.

### 핵심 개념

| 개념 | 설명 |
|---|---|
| **Listener** | 연결을 수신하는 소켓 (포트 바인딩) |
| **Filter Chain** | Listener가 연결을 처리하는 필터 파이프라인 |
| **Router** | 요청을 어느 Cluster로 보낼지 결정 |
| **Cluster** | 업스트림 서비스의 엔드포인트 집합 (로드밸런싱 포함) |
| **Endpoint** | 실제 서비스 인스턴스의 IP:Port |

### 요청 처리 파이프라인

```
클라이언트 요청
      ↓
[Listener: :15001 아웃바운드]
      ↓
[Network Filter: TCP Proxy / HTTP Connection Manager]
      ↓
[HTTP Filter 체인]
  1. jwt_authn        — JWT 토큰 검증
  2. rbac             — 역할 기반 접근 제어
  3. fault            — 장애 주입 (카오스 엔지니어링)
  4. cors             — CORS 정책 적용
  5. router           — 라우팅 결정
      ↓
[Cluster 선택] — Virtual Host 기반 라우팅
      ↓
[Load Balancing] — Round-robin / Least Connections / Ring Hash
      ↓
[Connection Pool] — HTTP/2 멀티플렉싱, keep-alive 관리
      ↓
업스트림 서비스 (Envoy 사이드카)
```

### xDS API: 동적 설정 프로토콜

Envoy의 설정은 **xDS(Extendable Discovery Service)** API를 통해 컨트롤 플레인에서 동적으로 배포됩니다.

```
xDS API 종류:
  LDS (Listener DS)   — 리스너 설정 동적 업데이트
  RDS (Route DS)      — HTTP 라우팅 규칙 동적 업데이트
  CDS (Cluster DS)    — 클러스터 (업스트림 서비스) 동적 업데이트
  EDS (Endpoint DS)   — 엔드포인트 목록 동적 업데이트 (서비스 디스커버리)
  SDS (Secret DS)     — TLS 인증서·키 동적 갱신
  RTDS (Runtime DS)   — 런타임 기능 플래그
```

```python
# xDS를 통한 Envoy 설정 조회 (관리 API)
import requests
import json

ENVOY_ADMIN = "http://localhost:15000"

def get_clusters():
    """현재 클러스터 설정 조회"""
    response = requests.get(f"{ENVOY_ADMIN}/clusters?format=json")
    clusters = response.json()
    for name, data in clusters.get("cluster_statuses", {}).items():
        print(f"\nCluster: {name}")
        for host in data.get("host_statuses", []):
            addr = host["address"]["socket_address"]
            health = host["health_status"]["eds_health_status"]
            stats = host.get("stats", [])
            rq_success = next((s["value"] for s in stats
                               if s["name"] == "rq_success"), 0)
            print(f"  {addr['address']}:{addr['port_value']} "
                  f"[{health}] success={rq_success}")

def get_listeners():
    """현재 리스너 설정 조회"""
    response = requests.get(f"{ENVOY_ADMIN}/listeners?format=json")
    for listener in response.json().get("listener_statuses", []):
        name = listener["name"]
        addr = listener.get("local_address", {}).get("socket_address", {})
        print(f"Listener: {name} → {addr.get('address')}:{addr.get('port_value')}")

get_clusters()
get_listeners()
```

## iptables 트래픽 인터셉션

Istio의 `istio-init` 초기화 컨테이너는 **iptables 규칙**을 설정하여 Pod의 모든 트래픽을 Envoy 사이드카로 리디렉션합니다.

```bash
# istio-init이 설정하는 iptables 규칙 (단순화)
# 아웃바운드 트래픽: 앱의 모든 아웃바운드를 Envoy 15001 포트로
iptables -t nat -A OUTPUT -p tcp \
  ! -d 127.0.0.1/32 \
  -j REDIRECT --to-port 15001

# 인바운드 트래픽: 외부에서 오는 트래픽을 Envoy 15006 포트로
iptables -t nat -A PREROUTING -p tcp \
  -j REDIRECT --to-port 15006

# Envoy 자신의 트래픽은 우회 (무한 루프 방지)
iptables -t nat -A OUTPUT -m owner \
  --uid-owner 1337 -j RETURN
```

```
트래픽 흐름:
App (포트 8080) → 아웃바운드 TCP
  → iptables NAT → Envoy :15001
  → Envoy가 목적지 분석 → mTLS 연결 수립
  → 원격 Pod의 Envoy :15006
  → iptables가 App :8080으로 전달 → 상대 앱이 수신
```

## mTLS: 상호 인증 TLS

서비스 메시의 핵심 보안 기능은 서비스 간 **mTLS(mutual TLS)**입니다.

```
인증서 구조 (SPIFFE ID):
  spiffe://cluster.local/ns/default/sa/payment-service

Istiod(CA) → Pod A의 Envoy (인증서 발급)
Istiod(CA) → Pod B의 Envoy (인증서 발급)

Pod A Envoy ──[mTLS 핸드셰이크]──▶ Pod B Envoy
  A가 B의 인증서 검증 + B가 A의 인증서 검증
  → 암호화된 채널로 통신
```

```yaml
# PeerAuthentication: mTLS 정책
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT

---
# AuthorizationPolicy: 서비스 간 접근 제어
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-service-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-service
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/production/sa/order-service"]
      to:
        - operation:
            methods: ["POST"]
            paths: ["/api/payments/*"]
```

## 트래픽 관리: VirtualService와 DestinationRule

```yaml
# DestinationRule: 버전별 서브셋 정의 + 서킷 브레이커
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: user-service
spec:
  host: user-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http2MaxRequests: 1000
        pendingRequests: 100
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: v1
      labels:
        version: v1
      trafficPolicy:
        loadBalancer:
          simple: ROUND_ROBIN
    - name: v2
      labels:
        version: v2
      trafficPolicy:
        loadBalancer:
          simple: LEAST_CONN

---
# VirtualService: 카나리 배포 (90% v1, 10% v2)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: user-service
spec:
  hosts:
    - user-service
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: user-service
            subset: v2
    - route:
        - destination:
            host: user-service
            subset: v1
          weight: 90
        - destination:
            host: user-service
            subset: v2
          weight: 10
      timeout: 3s
      retries:
        attempts: 3
        perTryTimeout: 1s
        retryOn: "5xx,reset,connect-failure"
      fault:
        delay:
          percentage:
            value: 0.1
          fixedDelay: 500ms
```

## Ambient Mesh: 사이드카 없는 미래

Istio 1.18부터 도입된 **Ambient Mesh**는 사이드카 프록시 없이 서비스 메시를 구현합니다.

```
사이드카 모델:
  Pod 1: App + Envoy (200MB RAM)
  Pod 2: App + Envoy (200MB RAM)
  총 메모리 = 서비스 수 × 200MB

Ambient Mesh 모델:
  Pod 1: App만 (메모리 절약)
  Pod 2: App만
  노드당: ztunnel (20MB) — L4 mTLS
  네임스페이스당: waypoint (150MB) — L7 정책 (선택적)
  총 메모리 = 노드 수 × 20MB + 정책 필요 네임스페이스 × 150MB
```

## 주의사항과 실전 팁

### 1. 레이턴시 오버헤드 측정
사이드카를 거치면 일반적으로 요청당 1~3ms의 지연이 추가됩니다. SLA가 엄격한 서비스는 반드시 실제 환경에서 측정하세요.

### 2. 리소스 사이징
Envoy 사이드카는 Pod당 50~200MB 메모리와 0.1~0.5 CPU를 사용합니다. 수백 개의 Pod를 운영하면 클러스터 리소스 계획에 포함해야 합니다.

### 3. 헤더 전파 필수
분산 추적이 작동하려면 `x-b3-traceid`, `x-b3-spanid`, `traceparent` 등의 헤더를 서비스 간 요청에서 수동으로 전파해야 합니다.

### 4. 점진적 도입
전체 클러스터에 한 번에 적용하기보다, 네임스페이스 단위로 점진적으로 적용하고 트래픽·에러율 변화를 모니터링하세요.

### 5. Egress 트래픽 제어
기본적으로 Istio는 외부 서비스 호출을 허용합니다. `outboundTrafficPolicy: REGISTRY_ONLY`로 설정하면 명시적으로 등록된 ServiceEntry만 외부 통신이 가능해져 보안이 강화됩니다.

## 정리

서비스 메시는 마이크로서비스 운영의 복잡성을 애플리케이션 코드에서 인프라 레이어로 이동시킵니다. Envoy의 강력한 L7 프록시 기능과 Istio의 컨트롤 플레인이 결합되어 mTLS, 서킷 브레이커, 트래픽 분산, 관측 가능성을 코드 변경 없이 제공합니다. xDS API를 통한 동적 설정, SPIFFE 기반 서비스 ID, iptables 인터셉션의 원리를 이해하면 서비스 메시를 더 효과적으로 구성하고 문제를 진단할 수 있습니다.

## 참고 자료
- [Istio 공식 아키텍처 문서](https://istio.io/latest/docs/ops/deployment/architecture/)
- [Envoy Proxy 공식 문서](https://www.envoyproxy.io/docs/envoy/latest/)
- [Solo.io — Istio 아키텍처 심화](https://www.solo.io/topics/istio-architecture/)
- [SPIFFE 공식 표준](https://spiffe.io/docs/latest/spiffe-about/overview/)
