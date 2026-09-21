---
layout: post
title: "서비스 디스커버리 완전 정복: Consul·etcd·Kubernetes DNS로 구현하는 자동 서비스 등록과 탐색"
date: 2026-09-21
categories: [cs, computer-science]
tags: [service-discovery, consul, etcd, kubernetes, microservices, distributed-systems, dns, health-check]
---

## 개념 설명

마이크로서비스 환경에서 서비스 인스턴스는 오토스케일링, 장애 복구, 롤링 배포 등으로 **IP 주소와 포트가 지속적으로 바뀐다**. 이 동적인 환경에서 서비스가 서로를 어떻게 찾아낼 것인가? 이것이 **서비스 디스커버리(Service Discovery)**가 해결하는 문제다.

전통적인 모놀리식 시스템에서는 로드밸런서의 고정 IP를 환경변수에 저장해 두면 충분했다. 하지만 수십~수백 개의 마이크로서비스가 수시로 스케일 인·아웃하는 환경에서는 그 방법이 동작하지 않는다.

### 핵심 구성 요소

1. **서비스 레지스트리(Service Registry)**: 서비스 인스턴스의 네트워크 위치(IP:Port)를 저장하는 데이터베이스. Consul, etcd, Eureka, ZooKeeper 등이 이 역할을 한다.

2. **서비스 등록(Service Registration)**: 인스턴스가 시작될 때 레지스트리에 자신을 등록하고, 종료될 때 등록을 해제한다.

3. **서비스 탐색(Service Lookup)**: 클라이언트가 레지스트리에서 목적지 서비스의 인스턴스 목록을 조회한다.

4. **헬스 체크(Health Check)**: 레지스트리가 주기적으로 인스턴스 상태를 확인하여 비정상 인스턴스를 목록에서 제외한다.

### 두 가지 패턴

**클라이언트 사이드 디스커버리(Client-Side Discovery)**  
클라이언트가 레지스트리에서 인스턴스 목록을 직접 조회한 후 로드밸런싱 로직을 수행한다. Netflix Eureka + Ribbon 조합이 대표적이다. 클라이언트에 로드밸런싱 로직이 있어 언어별 구현이 필요하다.

**서버 사이드 디스커버리(Server-Side Discovery)**  
클라이언트는 로드밸런서(또는 API Gateway)에만 요청한다. 로드밸런서가 레지스트리를 조회하여 라우팅한다. AWS ALB, Nginx, HAProxy, Kubernetes Service가 여기에 속한다. 클라이언트는 단순해지지만 로드밸런서가 단일 장애점(SPOF)이 될 수 있다.

---

## 왜 필요한가

컨테이너 오케스트레이션 플랫폼(Kubernetes, Nomad, ECS)에서 파드(Pod)는 재시작될 때마다 새 IP를 받는다. 이 환경에서 서비스 디스커버리 없이 하드코딩된 IP로 통신하면:

- 배포 시마다 모든 클라이언트의 설정 파일을 수동으로 업데이트해야 한다
- 장애로 재시작된 인스턴스는 다른 IP를 받아 트래픽을 받지 못한다
- A/B 테스트나 카나리 배포가 불가능하다

자동화된 서비스 디스커버리는 이 과정을 투명하게 처리하여 **운영 부담을 0에 가깝게 줄인다**.

---

## 실전 구현 예제

### 예제 1 — Consul을 활용한 서비스 등록·탐색 (Python)

`python-consul` 라이브러리로 서비스를 등록하고 헬스 체크를 설정한다.

```python
import consul
import socket
import uuid
import time
import threading

# Consul 클라이언트 초기화
c = consul.Consul(host="127.0.0.1", port=8500)

# 고유한 서비스 ID 생성 (여러 인스턴스 운영 시 필요)
SERVICE_ID   = f"payment-service-{uuid.uuid4().hex[:8]}"
SERVICE_NAME = "payment-service"
SERVICE_PORT = 8080
SERVICE_IP   = socket.gethostbyname(socket.gethostname())


def register_service() -> None:
    """Consul에 서비스 등록 + 헬스 체크 설정"""
    c.agent.service.register(
        name=SERVICE_NAME,
        service_id=SERVICE_ID,
        address=SERVICE_IP,
        port=SERVICE_PORT,
        tags=["v2", "production"],
        check=consul.Check.http(
            url=f"http://{SERVICE_IP}:{SERVICE_PORT}/health",
            interval="10s",   # 10초마다 헬스 체크
            timeout="2s",     # 2초 초과 시 비정상 처리
            deregister="30s", # 30초 연속 실패 시 자동 해제
        ),
    )
    print(f"[Consul] 등록 완료: {SERVICE_ID} @ {SERVICE_IP}:{SERVICE_PORT}")


def deregister_service() -> None:
    """서비스 종료 시 Consul에서 해제"""
    c.agent.service.deregister(SERVICE_ID)
    print(f"[Consul] 해제 완료: {SERVICE_ID}")


def discover_service(name: str) -> list[dict]:
    """
    건강한 서비스 인스턴스 목록 조회
    반환값 예: [{"address": "10.0.1.5", "port": 8080}, ...]
    """
    index, services = c.health.service(name, passing=True)  # passing=True: 정상만
    return [
        {
            "address": s["Service"]["Address"],
            "port":    s["Service"]["Port"],
            "tags":    s["Service"]["Tags"],
        }
        for s in services
    ]


def round_robin_select(instances: list[dict], counter: list[int]) -> dict | None:
    if not instances:
        return None
    instance = instances[counter[0] % len(instances)]
    counter[0] += 1
    return instance


# ---- 시뮬레이션 ----
register_service()

try:
    counter = [0]
    for _ in range(5):
        instances = discover_service("payment-service")
        selected = round_robin_select(instances, counter)
        if selected:
            print(f"요청 대상: {selected['address']}:{selected['port']} (tags: {selected['tags']})")
        time.sleep(1)
finally:
    deregister_service()
```

Consul은 **gRPC 스트리밍**을 이용한 Watch 기능도 제공하므로, 인스턴스 목록이 바뀔 때마다 클라이언트가 즉시 통보받을 수 있다.

---

### 예제 2 — etcd를 이용한 서비스 등록 (Go)

etcd는 분산 키-값 저장소로, 리스(Lease) 기능을 활용하면 인스턴스가 죽으면 자동으로 키가 만료된다.

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	clientv3 "go.etcd.io/etcd/client/v3"
)

const (
	etcdEndpoint = "localhost:2379"
	serviceName  = "order-service"
	instanceAddr = "10.0.2.10:9090"
	ttlSeconds   = 10 // 리스 TTL: 10초 갱신 없으면 자동 삭제
)

func registerWithEtcd(ctx context.Context, cli *clientv3.Client) error {
	// 1. 리스 생성 (TTL = 10초)
	leaseResp, err := cli.Grant(ctx, ttlSeconds)
	if err != nil {
		return fmt.Errorf("리스 생성 실패: %w", err)
	}
	leaseID := leaseResp.ID

	// 2. 서비스 키 등록 (/services/order-service/10.0.2.10:9090)
	key := fmt.Sprintf("/services/%s/%s", serviceName, instanceAddr)
	_, err = cli.Put(ctx, key, instanceAddr, clientv3.WithLease(leaseID))
	if err != nil {
		return fmt.Errorf("키 등록 실패: %w", err)
	}
	log.Printf("[etcd] 등록: %s → %s", key, instanceAddr)

	// 3. KeepAlive 고루틴: TTL 주기마다 갱신
	keepAliveCh, err := cli.KeepAlive(ctx, leaseID)
	if err != nil {
		return fmt.Errorf("keepalive 실패: %w", err)
	}

	go func() {
		for {
			select {
			case <-ctx.Done():
				log.Println("[etcd] KeepAlive 중단, 서비스 해제")
				cli.Revoke(context.Background(), leaseID)
				return
			case ka, ok := <-keepAliveCh:
				if !ok {
					log.Println("[etcd] KeepAlive 채널 종료")
					return
				}
				log.Printf("[etcd] TTL 갱신 완료: %d초 남음", ka.TTL)
			}
		}
	}()

	return nil
}

func discoverServices(cli *clientv3.Client) ([]string, error) {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	prefix := fmt.Sprintf("/services/%s/", serviceName)
	resp, err := cli.Get(ctx, prefix, clientv3.WithPrefix())
	if err != nil {
		return nil, err
	}

	var addrs []string
	for _, kv := range resp.Kvs {
		addrs = append(addrs, string(kv.Value))
	}
	return addrs, nil
}

func main() {
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{etcdEndpoint},
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	if err := registerWithEtcd(ctx, cli); err != nil {
		log.Fatal(err)
	}

	// Watch: 서비스 목록 변경 감지
	watchCh := cli.Watch(ctx, "/services/"+serviceName+"/", clientv3.WithPrefix())
	go func() {
		for watchResp := range watchCh {
			for _, event := range watchResp.Events {
				log.Printf("[Watch] %s: %s", event.Type, event.Kv.Key)
			}
		}
	}()

	// 주기적으로 서비스 목록 조회
	for i := 0; i < 3; i++ {
		addrs, _ := discoverServices(cli)
		fmt.Printf("현재 %s 인스턴스: %v\n", serviceName, addrs)
		time.Sleep(3 * time.Second)
	}
}
```

etcd의 리스(Lease) + KeepAlive 패턴은 **TTL 기반 자동 해제**의 핵심이다. 프로세스가 죽으면 KeepAlive가 중단되고, TTL이 만료된 후 키가 자동으로 삭제된다.

---

## Kubernetes 내장 서비스 디스커버리

Kubernetes는 서비스 디스커버리를 **기본 기능**으로 제공한다.

```yaml
# Service 정의 — 파드 집합에 대한 안정적인 엔드포인트
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: production
spec:
  selector:
    app: payment
  ports:
    - protocol: TCP
      port: 80         # 서비스 포트
      targetPort: 8080  # 파드 컨테이너 포트
  type: ClusterIP
```

```python
import requests

# Kubernetes DNS로 서비스 탐색
# 형식: <service-name>.<namespace>.svc.cluster.local
PAYMENT_URL = "http://payment-service.production.svc.cluster.local/api/pay"

# 같은 네임스페이스라면 단순히:
PAYMENT_URL_SHORT = "http://payment-service/api/pay"

def call_payment(order_id: str, amount: float) -> dict:
    response = requests.post(
        PAYMENT_URL_SHORT,
        json={"order_id": order_id, "amount": amount},
        timeout=5,
    )
    response.raise_for_status()
    return response.json()
```

Kubernetes CoreDNS는 `<service-name>.<namespace>.svc.cluster.local` 형식의 DNS 레코드를 자동으로 관리한다. 파드가 재시작되어 IP가 바뀌어도 Service IP(ClusterIP)는 고정이며, kube-proxy가 iptables/IPVS 규칙으로 실제 파드로 트래픽을 분산한다.

---

## 도구 비교

| 항목 | Consul | etcd | Kubernetes DNS | Eureka |
|---|---|---|---|---|
| 헬스 체크 | 내장 (HTTP/TCP/Script) | 직접 구현 필요 | Liveness/Readiness Probe | 클라이언트 heartbeat |
| KV 저장소 | 지원 | 주 기능 | 미지원 | 미지원 |
| 다중 데이터센터 | 내장 지원 | 외부 페더레이션 필요 | 클러스터 내부 | 지역별 독립 |
| DNS 인터페이스 | 지원 | 미지원 | 기본 | 미지원 |
| 서비스 메시 통합 | Consul Connect | - | Istio/Linkerd | - |
| 주요 사용처 | HashiCorp 생태계 | Kubernetes 내부 스토어 | Kubernetes 환경 | Spring Cloud 생태계 |

---

## 주의사항과 팁

**1. 레지스트리 자체의 고가용성**  
Consul과 etcd는 Raft 합의 알고리즘으로 클러스터를 구성한다. 최소 3개(권장 5개) 노드로 운영해야 단일 장애점이 없다.

**2. 클라이언트 캐싱**  
레지스트리에 모든 요청마다 조회하면 레지스트리 과부하가 발생한다. 클라이언트에서 인스턴스 목록을 로컬에 캐시하고, Watch/롱폴링으로 변경 시에만 갱신한다.

**3. 헬스 체크는 정교하게 설계하라**  
단순 HTTP 200 응답만이 아니라 DB 연결 가능 여부, 메모리 사용량, 처리 큐 상태 등 **실제 서비스 가능 여부**를 반영해야 한다.

**4. 그레이스풀 셧다운(Graceful Shutdown)**  
SIGTERM 수신 시 즉시 종료하지 말고: ① 레지스트리에서 해제 → ② 진행 중인 요청 완료 대기 → ③ 종료 순서를 지켜야 요청 손실을 방지한다.

**5. 서비스 디스커버리와 API Gateway의 결합**  
Kong, NGINX, Envoy 같은 API Gateway가 레지스트리를 동적으로 구독하면 업스트림 설정 파일을 수동으로 편집하지 않아도 된다. Consul-Template, Confd 등이 이 역할을 한다.

**6. 멀티 클라우드/멀티 클러스터**  
서비스가 여러 클러스터에 분산된 환경에서는 Consul의 WAN 페더레이션이나 Kubernetes Federation, 또는 서비스 메시(Istio)의 멀티클러스터 기능을 활용한다.

---

## 참고 자료
- [Service Discovery Explained — HashiCorp Consul Docs](https://developer.hashicorp.com/consul/docs/use-case/service-discovery)
- [Advanced Service Discovery: Consul, etcd, Zookeeper — Medium](https://ahmettsoner.medium.com/advanced-service-discovery-in-microservices-consul-etcd-and-zookeeper-b8860dce8363)
- [Service Discovery in 2026: Consul, etcd, Kubernetes — DEV Community](https://dev.to/gabrielanhaia/service-discovery-in-2026-consul-etcd-and-kubernetes-which-wins-when-2931)
- [Consul vs etcd Service Discovery Comparison](https://slickfinch.com/blog/consul-vs-etcd-service-discovery-tools-comparison/)
