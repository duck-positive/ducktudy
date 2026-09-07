---
layout: post
title: "쿠버네티스 스케줄러 내부 구조 완전 정복: Pod가 노드에 배치되는 모든 과정"
date: 2026-09-07
categories: [cs, computer-science]
tags: [kubernetes, scheduler, cloud-native, distributed-systems, scheduling-framework]
---

쿠버네티스(Kubernetes)는 컨테이너 오케스트레이션 플랫폼의 사실상 표준이 되었다. 수백 개의 서비스가 수천 개의 Pod로 나뉘어 실행되는 환경에서, **어느 Pod가 어느 노드에서 실행될지를 결정하는 컴포넌트**가 바로 `kube-scheduler`이다. 스케줄러는 단순해 보이지만 내부적으로는 정교한 플러그인 아키텍처와 두 단계의 사이클로 구성된다. 이 아티클에서는 스케줄링 프레임워크의 모든 확장 포인트를 코드와 함께 분석한다.

---

## 쿠버네티스 스케줄러란 무엇인가

`kube-scheduler`는 쿠버네티스 컨트롤 플레인의 핵심 컴포넌트다. 역할은 간단하다: **아직 노드가 배정되지 않은 Pod를 관찰하고, 최적의 노드를 선택해 바인딩(binding)하는 것**이다.

"최적"이라는 기준은 단순한 CPU/메모리 여유분이 아니다. 스케줄러는 다음 요소들을 종합적으로 평가한다.

- **리소스 요청량**: CPU, 메모리, GPU, 확장 리소스
- **하드웨어/소프트웨어 제약**: `nodeSelector`, `nodeName`
- **친화성/반친화성(Affinity/Anti-Affinity)**: Pod 간 혹은 Pod-노드 간 배치 규칙
- **테인트(Taint)와 톨러레이션(Toleration)**: 특정 노드를 오염시켜 일반 Pod 차단
- **토폴로지 분산 제약(Topology Spread Constraints)**: Pod를 가용 영역(AZ)에 고르게 분산
- **우선순위(Priority)**: 높은 우선순위 Pod를 위해 낮은 우선순위 Pod를 퇴거(Evict)

---

## 스케줄링이 왜 어려운가

스케줄링은 본질적으로 **NP-완전에 가까운 최적 배치 문제**다. 노드가 N개이고 Pod가 M개라면 모든 가능한 배치 조합은 N^M에 달한다. 클러스터 규모가 커질수록 전체 탐색은 불가능하다.

이를 해결하기 위해 쿠버네티스 스케줄러는 **휴리스틱(heuristic) 기반의 두 단계 파이프라인**을 사용한다.

### 두 단계: Scheduling Cycle + Binding Cycle

```
┌────────────────── Scheduling Cycle (직렬) ──────────────────┐
│  PreFilter → Filter → PostFilter → PreScore → Score → Reserve │
└────────────────────────────────────────────────────────────────┘
                               │
                    (최적 노드 선택 완료)
                               │
┌────────────────── Binding Cycle (병렬 가능) ────────────────┐
│  PreBind → Bind → PostBind                                     │
└────────────────────────────────────────────────────────────────┘
```

**Scheduling Cycle**은 단일 고루틴에서 직렬로 실행되며 가장 적합한 노드를 선택한다. **Binding Cycle**은 네트워크 I/O가 포함되므로 비동기로 실행될 수 있다. 스케줄러는 다음 Pod의 Scheduling Cycle을 Binding이 끝나기 전에 시작할 수 있어 처리량을 높인다.

---

## 확장 포인트(Extension Points) 상세 분석

### 1. QueueSort
Pod를 대기 큐에서 꺼내는 순서를 결정한다. 기본 구현은 `PrioritySort`로 우선순위 값과 생성 시각을 비교한다.

### 2. PreFilter
Pod와 전체 클러스터 상태를 검증한다. 예를 들어 `NodeResourcesFit` 플러그인은 여기서 Pod가 요청하는 리소스 합계를 계산해 캐싱한다.

### 3. Filter
각 노드에 대해 Pod를 실행할 수 있는지 **Boolean 판단**을 내린다. 통과하지 못한 노드는 후보에서 제외된다. 이것이 **Feasibility Check**다.

주요 Filter 플러그인:
- `NodeResourcesFit`: CPU, 메모리, GPU 여유 확인
- `NodeAffinity`: nodeAffinity 규칙 적용
- `TaintToleration`: 테인트/톨러레이션 매칭
- `VolumeBinding`: PersistentVolume 바인딩 가능 여부
- `TopologySpreadConstraints`: 분산 제약 위반 노드 제외

### 4. PostFilter
Filter 단계를 통과한 노드가 **0개**일 때 호출된다. 기본 플러그인은 `DefaultPreemption`으로, 우선순위가 낮은 Pod를 퇴거시켜 공간을 만드는 선점(Preemption)을 시도한다.

### 5. PreScore
Score 단계 전에 공유 상태를 계산하고 캐싱한다. 성능 최적화 목적이다.

### 6. Score
Filter를 통과한 각 노드에 **0~100점** 사이의 점수를 매긴다. 여러 Score 플러그인의 점수는 가중합으로 최종 점수가 산출된다.

주요 Score 플러그인:
- `LeastAllocated`: 자원을 적게 쓴 노드 선호 (분산 전략)
- `MostAllocated`: 자원을 많이 쓴 노드 선호 (빈 패킹 전략)
- `ImageLocality`: 필요한 컨테이너 이미지가 이미 있는 노드 우대
- `InterPodAffinity`: 선호하는 Pod와 같은 노드에 배치

### 7. Reserve
선택된 노드의 리소스를 예약한다. 바인딩이 실패하면 `Unreserve`로 롤백된다.

### 8. Permit
바인딩 직전 마지막 게이트. 플러그인은 Allow, Deny, Wait 중 하나를 반환할 수 있다. **Gang Scheduling**(그룹 내 모든 Pod가 준비될 때까지 대기)에 활용된다.

### 9. Bind
실제로 Pod의 `spec.nodeName`을 API 서버에 기록한다. 기본 구현은 `DefaultBinder`이며, 하나의 Bind 플러그인만 실행된다.

---

## 구현 예제 1: 커스텀 스케줄러 플러그인 (Go)

가장 최근에 사용된 노드를 피하는 "LeastRecentlyUsed" Score 플러그인을 구현해보자.

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"

	"k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/runtime"
	framework "k8s.io/kubernetes/pkg/scheduler/framework"
)

const PluginName = "LeastRecentlyUsed"

type LRUPlugin struct {
	mu          sync.RWMutex
	lastUsed    map[string]time.Time // nodeName → last scheduled time
}

var _ framework.ScorePlugin = &LRUPlugin{}

func (p *LRUPlugin) Name() string { return PluginName }

// Score: 오래 전에 사용된 노드일수록 높은 점수
func (p *LRUPlugin) Score(
	ctx context.Context,
	state *framework.CycleState,
	pod *v1.Pod,
	nodeName string,
) (int64, *framework.Status) {
	p.mu.RLock()
	lastTime, exists := p.lastUsed[nodeName]
	p.mu.RUnlock()

	if !exists {
		// 한 번도 사용 안 된 노드 → 최고 점수
		return 100, nil
	}

	// 마지막 사용 후 경과 시간 기반 점수 (최대 100분 기준 정규화)
	elapsed := time.Since(lastTime).Minutes()
	score := int64(elapsed)
	if score > 100 {
		score = 100
	}
	return score, nil
}

func (p *LRUPlugin) ScoreExtensions() framework.ScoreExtensions { return nil }

// Bind 후 호출되어 사용 시각 기록 (PostBind 훅 활용)
func (p *LRUPlugin) PostBind(
	ctx context.Context,
	state *framework.CycleState,
	pod *v1.Pod,
	nodeName string,
) {
	p.mu.Lock()
	defer p.mu.Unlock()
	p.lastUsed[nodeName] = time.Now()
	fmt.Printf("[LRU] Node %s used at %v\n", nodeName, time.Now())
}

func New(obj runtime.Object, h framework.Handle) (framework.Plugin, error) {
	return &LRUPlugin{lastUsed: make(map[string]time.Time)}, nil
}
```

플러그인을 등록하려면 scheduler 바이너리에 컴파일 시점에 포함시켜야 한다:

```go
// cmd/scheduler/main.go
import (
	"k8s.io/kubernetes/pkg/scheduler/app"
	lru "mycompany/scheduler-plugins/lru"
)

func main() {
	command := app.NewSchedulerCommand(
		app.WithPlugin(lru.PluginName, lru.New),
	)
	// ...
}
```

---

## 구현 예제 2: Python으로 스케줄링 이벤트 관찰하기

쿠버네티스 Python 클라이언트를 이용해 스케줄링 이벤트를 실시간 모니터링하는 예제다.

```python
from kubernetes import client, config, watch
import json
from datetime import datetime

def watch_scheduling_events():
    """Pod 스케줄링 이벤트를 실시간으로 모니터링한다."""
    config.load_kube_config()  # ~/.kube/config 사용
    v1 = client.CoreV1Api()
    w = watch.Watch()

    print(f"[{datetime.now()}] 스케줄링 이벤트 모니터링 시작...")

    for event in w.stream(v1.list_event_for_all_namespaces,
                           field_selector="reason=Scheduled"):
        obj = event["object"]
        event_type = event["type"]

        if obj.reason != "Scheduled":
            continue

        pod_name = obj.involved_object.name
        namespace = obj.involved_object.namespace
        message = obj.message  # "Successfully assigned default/nginx to node-1"
        timestamp = obj.event_time or obj.last_timestamp

        print(f"[{timestamp}] {event_type}: {namespace}/{pod_name}")
        print(f"  → {message}")
        print()

def analyze_scheduling_latency():
    """Pod 생성부터 스케줄링 완료까지의 지연 시간을 분석한다."""
    config.load_kube_config()
    v1 = client.CoreV1Api()

    pods = v1.list_pod_for_all_namespaces(watch=False)
    latencies = []

    for pod in pods.items:
        creation = pod.metadata.creation_timestamp
        if not pod.status.conditions:
            continue

        scheduled_time = None
        for cond in pod.status.conditions:
            if cond.type == "PodScheduled" and cond.status == "True":
                scheduled_time = cond.last_transition_time
                break

        if creation and scheduled_time:
            latency_ms = (scheduled_time - creation).total_seconds() * 1000
            latencies.append({
                "pod": pod.metadata.name,
                "namespace": pod.metadata.namespace,
                "node": pod.spec.node_name,
                "latency_ms": round(latency_ms, 2),
            })

    # 지연 시간 정렬 및 출력
    latencies.sort(key=lambda x: x["latency_ms"], reverse=True)
    print(f"{'Pod':<40} {'Namespace':<20} {'Node':<20} {'Latency(ms)'}")
    print("-" * 95)
    for item in latencies[:20]:  # 상위 20개
        print(f"{item['pod']:<40} {item['namespace']:<20} "
              f"{item['node'] or 'N/A':<20} {item['latency_ms']}")

if __name__ == "__main__":
    analyze_scheduling_latency()
```

---

## 스케줄러 성능 튜닝

### 1. `percentageOfNodesToScore` 활용
대규모 클러스터(500+ 노드)에서 모든 노드를 Score하면 성능이 저하된다. 이 설정으로 샘플링 비율을 조정할 수 있다.

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
percentageOfNodesToScore: 30  # 30%만 Score 단계 진행
profiles:
  - schedulerName: default-scheduler
    plugins:
      score:
        disabled:
          - name: PodTopologySpread  # 불필요하면 비활성화
```

### 2. 스케줄러 프로파일 멀티플렉싱
하나의 클러스터에서 서로 다른 스케줄링 정책을 적용해야 할 때 **멀티 프로파일**을 사용한다.

```yaml
profiles:
  - schedulerName: high-priority-scheduler
    plugins:
      score:
        enabled:
          - name: MostAllocated
            weight: 3
  - schedulerName: batch-scheduler
    plugins:
      score:
        enabled:
          - name: LeastAllocated
            weight: 1
```

Pod는 `spec.schedulerName` 필드로 원하는 프로파일을 지정한다.

---

## 주의사항과 팁

**1. 스케줄러 병목 모니터링**
스케줄링 처리량이 부족하면 `scheduler_pending_pods` 메트릭이 증가한다. Prometheus와 Grafana를 통해 다음 메트릭을 주시하자.
- `scheduler_scheduling_algorithm_duration_seconds`: 알고리즘 소요 시간
- `scheduler_pod_scheduling_attempts_total`: 스케줄링 시도 횟수
- `scheduler_queue_incoming_pods_total`: 큐 유입 속도

**2. 선점(Preemption)의 부작용**
선점이 활성화되면 낮은 우선순위 Pod가 갑자기 퇴거될 수 있다. 배치 작업에는 `PodDisruptionBudget(PDB)`을 설정해 최소 실행 인스턴스 수를 보장하자.

**3. 컨텍스트 스위칭 비용**
Scheduling Cycle은 단일 고루틴에서 실행된다. 커스텀 플러그인에서 블로킹 I/O를 수행하면 전체 스케줄링이 지연된다. 반드시 비동기 처리나 캐시를 활용하라.

**4. 클러스터 오토스케일러와의 상호작용**
Pending 상태의 Pod가 지속되면 Cluster Autoscaler가 노드를 추가한다. 커스텀 Filter 플러그인이 너무 엄격하면 AutoScaler가 적절한 노드를 추가해도 스케줄이 되지 않는 현상이 발생할 수 있다.

**5. Gang Scheduling 구현 시 데드락 주의**
Permit 플러그인의 Wait를 사용하는 Gang Scheduling에서 두 Pod 그룹이 서로 상대방의 완료를 기다리는 데드락이 발생할 수 있다. 반드시 타임아웃을 설정하라.

---

## 참고 자료

- [Scheduling Framework | Kubernetes Official Docs](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
- [Cluster Architecture | Kubernetes](https://kubernetes.io/docs/concepts/architecture/)
- [Internals of the Kubernetes Scheduler - Clusters and Coffee](https://clustersandcoffee.substack.com/p/internals-of-the-kubernetes-scheduler)
- [How the Kubernetes Scheduler Works Under the Hood](https://blog.devops.dev/how-the-kubernetes-scheduler-works-under-the-hood-kube-scheduler-internals-%EF%B8%8F-2ec27473cede)
