---
layout: post
title: "서비스 메시 내부 구조 완전 정복: Envoy와 Istio가 마이크로서비스를 제어하는 방법"
date: 2026-07-13
categories: [cs, computer-science]
tags: [service-mesh, envoy, istio, kubernetes, sidecar, xds, mtls, traffic-management, distributed-systems]
---

수십 개의 마이크로서비스가 서로 통신하는 환경에서 타임아웃, 재시도, 서킷 브레이커, TLS 인증, 트래픽 분배를 각 서비스 코드에 직접 구현하는 것은 악몽이다. 언어마다 다른 SDK, 버전 불일치, 비즈니스 로직과 인프라 로직의 혼재. **서비스 메시(Service Mesh)**는 이 모든 네트워크 로직을 애플리케이션에서 분리해 인프라 계층으로 내려보내는 해법이다.

## 서비스 메시란

서비스 메시는 마이크로서비스 간의 모든 네트워크 통신을 제어하는 **전용 인프라 레이어**다. 핵심 특징은 **애플리케이션 코드 수정 없이** 다음을 제공한다는 점이다.

- **트래픽 관리**: 로드 밸런싱, 가중치 기반 라우팅, A/B 테스트, 카나리 배포
- **복원력**: 타임아웃, 재시도, 서킷 브레이커, 결함 주입
- **보안**: 서비스 간 mTLS 자동 암호화, 인증/인가 정책
- **가시성**: 분산 트레이싱, 메트릭, 액세스 로그

서비스 메시는 **데이터 플레인(Data Plane)**과 **컨트롤 플레인(Control Plane)**으로 구성된다.

```
┌─────────────────────────────────────────────────────────┐
│                   Control Plane (Istiod)                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │  Pilot   │  │ Citadel  │  │  Galley  │              │
│  │(라우팅)   │  │(인증서)   │  │(설정검증) │              │
│  └──────────┘  └──────────┘  └──────────┘              │
│              xDS API (gRPC)                              │
└─────────────────────┬───────────────────────────────────┘
                      │ 설정 푸시
┌─────────────────────▼───────────────────────────────────┐
│                   Data Plane                             │
│  ┌────────────────────────────────────────────────────┐ │
│  │  Pod A                                             │ │
│  │  ┌──────────────┐   ┌──────────────────────────┐  │ │
│  │  │ 앱 컨테이너   │◄──│   Envoy Sidecar Proxy    │  │ │
│  │  │ (port 8080)  │   │ (port 15001 outbound      │  │ │
│  │  └──────────────┘   │  port 15006 inbound)      │  │ │
│  │                     └──────────────────────────┘  │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

## Envoy: 데이터 플레인의 심장

Envoy는 Lyft가 C++로 개발한 고성능 L7 프록시로, 현재 CNCF 프로젝트다. Istio의 모든 기능은 Envoy 위에 구현된다.

### Envoy의 핵심 개념

Envoy의 내부 아키텍처는 4가지 핵심 개념으로 구성된다.

**Listener**: 특정 포트에서 연결을 수락하는 소켓. `0.0.0.0:15001`에서 outbound 트래픽을 수신하는 방식으로 동작한다.

**Filter Chain**: 연결/요청을 처리하는 파이프라인. 각 필터가 순서대로 실행된다.
- `envoy.filters.network.http_connection_manager`: HTTP/gRPC 처리
- `envoy.filters.http.router`: 라우팅 결정

**Cluster**: 업스트림 서비스의 논리적 그룹. 로드 밸런싱 정책, 헬스 체크 설정 포함.

**Endpoint**: 클러스터를 구성하는 실제 서비스 인스턴스(IP:Port).

### Envoy 설정 예제 (YAML)

```yaml
# envoy-config.yaml - 기본 HTTP 프록시 설정 예시
static_resources:
  listeners:
    - name: listener_0
      address:
        socket_address:
          protocol: TCP
          address: 0.0.0.0
          port_value: 10000   # 인바운드 수신 포트
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ingress_http
                access_log:
                  - name: envoy.access_loggers.stdout
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: backend
                      domains: ["*"]
                      routes:
                        - match:
                            prefix: "/api/v1"
                          route:
                            cluster: service_api_v1
                            timeout: 5s
                            retry_policy:
                              retry_on: "5xx,reset"  # 500 에러나 연결 리셋 시 재시도
                              num_retries: 3
                              per_try_timeout: 2s
                        - match:
                            prefix: "/"
                          route:
                            cluster: service_default
                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
    - name: service_api_v1
      connect_timeout: 1s
      type: STRICT_DNS        # DNS 조회로 엔드포인트 해석
      lb_policy: ROUND_ROBIN
      load_assignment:
        cluster_name: service_api_v1
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: api-service
                      port_value: 8080
      health_checks:
        - timeout: 1s
          interval: 10s
          unhealthy_threshold: 3
          healthy_threshold: 2
          http_health_check:
            path: /health

    - name: service_default
      connect_timeout: 1s
      type: STRICT_DNS
      lb_policy: LEAST_REQUEST   # 가장 적은 요청을 받은 인스턴스 선택
      load_assignment:
        cluster_name: service_default
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: default-service
                      port_value: 8080

admin:
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 9901  # 관리 API: /stats, /clusters, /listeners 등
```

## Istio: 컨트롤 플레인의 구조

Istio 1.5 이후로 Pilot, Citadel, Galley가 **Istiod**라는 단일 바이너리로 통합됐다.

| 컴포넌트 | 역할 |
|---|---|
| **Pilot** | 서비스 레지스트리(kube-apiserver)를 읽어 라우팅 설정을 xDS API로 Envoy에 배포 |
| **Citadel** | X.509 인증서 발급/갱신, SVID(SPIFFE Verifiable Identity Document) 관리 |
| **Galley** | 설정 유효성 검사 및 배포 (MeshConfig, IstioOperator 처리) |

### xDS: 동적 설정 프로토콜

Istiod와 Envoy는 **xDS(Extensible Discovery Service) API**를 통해 gRPC 스트리밍으로 통신한다. Envoy가 설정 변경을 위해 프로세스를 재시작할 필요 없이 **핫 리로드**가 가능한 이유다.

| xDS API | 약어 | 관리 대상 |
|---|---|---|
| Listener Discovery Service | LDS | Listener 설정 |
| Route Discovery Service | RDS | HTTP 라우팅 규칙 |
| Cluster Discovery Service | CDS | 업스트림 클러스터 |
| Endpoint Discovery Service | EDS | 클러스터 내 실제 엔드포인트 |
| Secret Discovery Service | SDS | TLS 인증서/키 (보안 채널로 전송) |

### Istio VirtualService / DestinationRule 예제 (YAML)

```yaml
# VirtualService: 트래픽 라우팅 규칙 (Pilot → Envoy의 RDS에 해당)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-vs
  namespace: bookinfo
spec:
  hosts:
    - reviews          # 이 규칙이 적용될 서비스
  http:
    # 카나리 배포: 헤더 기반 라우팅
    - match:
        - headers:
            end-user:
              exact: "tester"
      route:
        - destination:
            host: reviews
            subset: v3   # 테스터는 v3로
          weight: 100

    # 가중치 기반 라우팅: v1 90%, v2 10%
    - route:
        - destination:
            host: reviews
            subset: v1
          weight: 90
        - destination:
            host: reviews
            subset: v2
          weight: 10
      timeout: 3s              # 요청 타임아웃
      retries:
        attempts: 3            # 최대 3회 재시도
        perTryTimeout: 1s
        retryOn: "5xx,reset,connect-failure"
      fault:                   # 결함 주입 (카오스 테스트)
        delay:
          percentage:
            value: 0.1         # 0.1%의 요청에
          fixedDelay: 5s       # 5초 지연 주입

---
# DestinationRule: 업스트림 정책 (Pilot → Envoy의 CDS/EDS에 해당)
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-dr
  namespace: bookinfo
spec:
  host: reviews
  trafficPolicy:
    # 서킷 브레이커 설정
    outlierDetection:
      consecutiveGatewayErrors: 5   # 연속 5회 오류 시
      interval: 10s                  # 10초 주기로 확인
      baseEjectionTime: 30s          # 30초 동안 트래픽 차단
      maxEjectionPercent: 50         # 최대 50%의 인스턴스 차단
    # 연결 풀 설정
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
      trafficPolicy:              # 서브셋별 개별 정책
        loadBalancer:
          simple: LEAST_CONN
    - name: v3
      labels:
        version: v3
```

## 사이드카 주입 메커니즘

Istio는 Kubernetes의 **MutatingAdmissionWebhook**을 사용해 Pod 생성 시 자동으로 Envoy 컨테이너를 주입한다.

주입 과정:
1. 네임스페이스에 `istio-injection: enabled` 레이블 부착
2. Pod 생성 요청 → kube-apiserver → Istio의 Mutating Webhook 호출
3. 웹훅이 Pod 스펙에 `istio-proxy` 컨테이너(Envoy)와 `istio-init` 초기화 컨테이너 추가
4. `istio-init`이 iptables 규칙을 설정해 모든 인바운드/아웃바운드 트래픽을 Envoy로 리다이렉트

iptables 리다이렉션 규칙의 핵심:
```bash
# 아웃바운드: 앱이 보내는 모든 TCP 트래픽 → Envoy 15001 포트로
iptables -t nat -A OUTPUT -p tcp -j REDIRECT --to-ports 15001

# 인바운드: 외부에서 들어오는 모든 TCP 트래픽 → Envoy 15006 포트로
iptables -t nat -A PREROUTING -p tcp -j REDIRECT --to-ports 15006

# Envoy 자신의 트래픽은 루프 방지를 위해 제외 (UID 1337 = envoy 프로세스)
iptables -t nat -A OUTPUT -m owner --uid-owner 1337 -j RETURN
```

앱은 `localhost:8080`으로 요청을 보내면, iptables가 이를 `15001`의 Envoy로 리다이렉트한다. 앱은 Envoy의 존재를 전혀 모른다.

## mTLS: 서비스 간 상호 인증

Istio의 Citadel(Istiod 내부)은 각 서비스 계정(ServiceAccount)에 X.509 인증서를 발급한다. 인증서는 **SPIFFE URI**(예: `spiffe://cluster.local/ns/default/sa/reviews`) 형식의 ID를 포함한다.

mTLS 핸드셰이크:
1. 클라이언트 Envoy → 서버 Envoy: "안녕, 나는 `spiffe://…/reviews`야"
2. 서버 Envoy → 클라이언트 Envoy: "나는 `spiffe://…/ratings`야"
3. 양측이 Citadel의 루트 CA를 신뢰해 서로 검증
4. 암호화된 채널로 통신

인증서는 기본 24시간마다 자동 갱신되며, SDS를 통해 Envoy에 전달되므로 재시작 없이 핫 스왑된다.

## 주의사항 및 실전 팁

**성능 오버헤드**
사이드카 프록시는 레이턴시를 추가한다. 단순 HTTP 요청에 수 ms의 오버헤드가 발생할 수 있다. Latency-sensitive한 서비스는 프록시리스(proxyless) gRPC 또는 Ambient Mesh(사이드카 없이 노드 레벨 프록시 사용) 옵션을 검토하라.

**디버깅 방법**
```bash
# Envoy 현재 설정 덤프
kubectl exec -it <pod> -c istio-proxy -- pilot-agent request GET config_dump | jq .

# xDS 동기화 상태 확인
istioctl proxy-status

# Envoy 관리 API (포트 포워딩 후 접근)
kubectl port-forward <pod> 15000:15000
curl http://localhost:15000/clusters  # 알려진 클러스터 목록
curl http://localhost:15000/stats     # 상세 메트릭
```

**Ambient Mesh**
Istio 1.22부터 정식 릴리스된 Ambient Mesh는 사이드카 없이 **ztunnel**(노드 레벨 L4 프록시)과 **waypoint proxy**(네임스페이스 레벨 L7 프록시)를 사용한다. 사이드카 대비 메모리 사용량이 약 90% 감소하고 레이턴시도 개선된다.

**PeerAuthentication으로 mTLS 강제화**
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT   # 네임스페이스 전체에 mTLS 강제
```

## 참고 자료
- [Istio 공식 아키텍처 문서](https://istio.io/latest/docs/ops/deployment/architecture/)
- [Envoy Proxy 공식 문서](https://www.envoyproxy.io/docs/envoy/latest/)
- [Tetrate: Envoy 아키텍처 상세 해설](https://tetrate.io/learn/envoy/envoy-architecture)
- [Red Hat: Service Mesh Architecture](https://docs.redhat.com/en/documentation/openshift_container_platform/4.2/html/service_mesh/service-mesh-architecture)
