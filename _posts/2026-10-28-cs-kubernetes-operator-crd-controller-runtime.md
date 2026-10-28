---
layout: post
title: "Kubernetes Operator 패턴과 CRD: 선언적 자동화의 구현 원리"
date: 2026-10-28
categories: [cs, computer-science]
tags: [kubernetes, operator, CRD, custom-resource, controller-runtime, kubebuilder, reconciliation, Go]
---

Kubernetes는 Deployment, StatefulSet, Service 같은 기본 리소스를 제공한다. 그런데 PostgreSQL 클러스터를 운영하거나, Kafka 브로커를 자동 스케일링하거나, 인증서를 자동 갱신하려면 어떻게 해야 할까? 이러한 복잡한 운영 지식을 쿠버네티스 안에 코드로 인코딩하는 패턴이 바로 **Operator 패턴**이다.

2016년 CoreOS가 처음 제안한 이 패턴은 오늘날 cert-manager, Prometheus Operator, Strimzi(Kafka), Vitess(MySQL), CloudNativePG(PostgreSQL) 등 수백 개의 프로덕션 시스템을 구동한다. Operator의 핵심은 공식처럼 간단하다:

```
Operator = CRD + Custom Controller
```

## Custom Resource Definition(CRD) 이해

CRD는 쿠버네티스 API를 확장하는 수단이다. `kubectl apply -f my-crd.yaml`을 실행하면 쿠버네티스 API 서버에 새로운 REST 엔드포인트가 생긴다. 예를 들어 `PostgresCluster`라는 CRD를 등록하면 다음과 같이 사용할 수 있다:

```yaml
# PostgresCluster CRD 정의
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresclusters.db.example.com
spec:
  group: db.example.com
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: ["replicas", "version"]
              properties:
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
                  description: "클러스터 레플리카 수"
                version:
                  type: string
                  pattern: "^[0-9]+\\.[0-9]+$"
                  description: "PostgreSQL 버전 (예: 15.4)"
                storageClass:
                  type: string
                  default: "standard"
                resources:
                  type: object
                  properties:
                    memory:
                      type: string
                      default: "1Gi"
                    cpu:
                      type: string
                      default: "500m"
                backup:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: false
                    schedule:
                      type: string
                      description: "크론 표현식 (예: 0 2 * * *)"
            status:
              type: object
              properties:
                phase:
                  type: string
                  enum: ["Pending", "Running", "Degraded", "Failed"]
                readyReplicas:
                  type: integer
                primaryEndpoint:
                  type: string
                conditions:
                  type: array
                  items:
                    type: object
                    properties:
                      type:
                        type: string
                      status:
                        type: string
                      message:
                        type: string
                      lastTransitionTime:
                        type: string
                        format: date-time
      subresources:
        status: {}  # status 서브리소스 활성화: spec과 status를 별도로 업데이트 가능
      additionalPrinterColumns:
        - name: Phase
          type: string
          jsonPath: .status.phase
        - name: Ready
          type: integer
          jsonPath: .status.readyReplicas
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
  scope: Namespaced  # 또는 Cluster
  names:
    plural: postgresclusters
    singular: postgrescluster
    kind: PostgresCluster
    shortNames:
      - pgc
```

이 YAML을 적용하면:
- `kubectl get postgresclusters` 명령이 동작한다
- `kubectl apply -f my-postgres.yaml`로 PostgresCluster 인스턴스를 생성할 수 있다
- OpenAPI v3 스키마로 입력값이 자동 검증된다
- `status` 서브리소스로 spec 변경과 status 변경을 독립적으로 관리한다

## Custom Controller: 조정 루프 구현

CRD는 데이터 스키마일 뿐이다. 실제 동작을 구현하는 것은 **Custom Controller**다. 컨트롤러는 쿠버네티스의 핵심 패턴인 **조정 루프(Reconciliation Loop)**를 구현한다:

```
관찰(Observe) → 분석(Analyze) → 행동(Act) → 반복
```

Go 언어와 `controller-runtime` 라이브러리를 사용한 완전한 Operator 구현:

```go
package controller

import (
    "context"
    "fmt"
    "time"

    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/api/resource"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/types"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
    "sigs.k8s.io/controller-runtime/pkg/log"

    dbv1alpha1 "example.com/postgres-operator/api/v1alpha1"
)

const (
    postgresClusterFinalizer = "db.example.com/finalizer"
    requeueAfter             = 30 * time.Second
)

// PostgresClusterReconciler는 PostgresCluster 리소스를 조정한다
type PostgresClusterReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// Reconcile는 controller-runtime이 이벤트 발생 시 호출하는 핵심 함수다.
// 항등적(idempotent)이어야 한다: 같은 상태로 여러 번 호출해도 결과가 동일해야 한다.
func (r *PostgresClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    logger.Info("조정 시작", "namespace", req.Namespace, "name", req.Name)

    // 1단계: 현재 상태(desired state) 읽기
    cluster := &dbv1alpha1.PostgresCluster{}
    if err := r.Get(ctx, req.NamespacedName, cluster); err != nil {
        if errors.IsNotFound(err) {
            // 리소스가 삭제됨 — 이미 클린업 완료
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, fmt.Errorf("PostgresCluster 읽기 실패: %w", err)
    }

    // 2단계: 삭제 처리 (Finalizer 패턴)
    if !cluster.DeletionTimestamp.IsZero() {
        return r.handleDeletion(ctx, cluster)
    }

    // Finalizer 등록 (삭제 시 클린업 보장)
    if !controllerutil.ContainsFinalizer(cluster, postgresClusterFinalizer) {
        controllerutil.AddFinalizer(cluster, postgresClusterFinalizer)
        if err := r.Update(ctx, cluster); err != nil {
            return ctrl.Result{}, err
        }
    }

    // 3단계: 원하는 상태(desired state)와 현재 상태(actual state) 비교 및 조정
    if err := r.reconcileStatefulSet(ctx, cluster); err != nil {
        r.setCondition(cluster, "StatefulSetReady", "False", err.Error())
        _ = r.Status().Update(ctx, cluster)
        return ctrl.Result{}, err
    }

    if err := r.reconcileService(ctx, cluster); err != nil {
        r.setCondition(cluster, "ServiceReady", "False", err.Error())
        _ = r.Status().Update(ctx, cluster)
        return ctrl.Result{}, err
    }

    // 4단계: 실제 상태 관찰 및 status 업데이트
    if err := r.updateStatus(ctx, cluster); err != nil {
        return ctrl.Result{}, err
    }

    // requeueAfter: 주기적으로 재조정 (드리프트 감지)
    return ctrl.Result{RequeueAfter: requeueAfter}, nil
}

// reconcileStatefulSet은 PostgreSQL StatefulSet을 조정한다.
// "있어야 한다면 만들고, 다르다면 수정하고, 불필요하면 삭제한다"
func (r *PostgresClusterReconciler) reconcileStatefulSet(
    ctx context.Context,
    cluster *dbv1alpha1.PostgresCluster,
) error {
    logger := log.FromContext(ctx)

    desired := r.buildStatefulSet(cluster)

    // Owner Reference 설정: cluster가 삭제되면 StatefulSet도 자동 삭제됨 (GC)
    if err := controllerutil.SetControllerReference(cluster, desired, r.Scheme); err != nil {
        return err
    }

    existing := &appsv1.StatefulSet{}
    err := r.Get(ctx, types.NamespacedName{
        Name:      desired.Name,
        Namespace: desired.Namespace,
    }, existing)

    if errors.IsNotFound(err) {
        logger.Info("StatefulSet 생성", "name", desired.Name)
        return r.Create(ctx, desired)
    }
    if err != nil {
        return err
    }

    // 레플리카 수 또는 이미지가 변경된 경우만 업데이트
    if *existing.Spec.Replicas != *desired.Spec.Replicas ||
        existing.Spec.Template.Spec.Containers[0].Image !=
            desired.Spec.Template.Spec.Containers[0].Image {
        existing.Spec.Replicas = desired.Spec.Replicas
        existing.Spec.Template.Spec.Containers[0].Image =
            desired.Spec.Template.Spec.Containers[0].Image
        logger.Info("StatefulSet 업데이트", "name", existing.Name)
        return r.Update(ctx, existing)
    }

    return nil
}

func (r *PostgresClusterReconciler) buildStatefulSet(
    cluster *dbv1alpha1.PostgresCluster,
) *appsv1.StatefulSet {
    replicas := int32(cluster.Spec.Replicas)
    image := fmt.Sprintf("postgres:%s", cluster.Spec.Version)

    return &appsv1.StatefulSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:      cluster.Name + "-postgres",
            Namespace: cluster.Namespace,
            Labels: map[string]string{
                "app.kubernetes.io/name":       "postgres",
                "app.kubernetes.io/instance":   cluster.Name,
                "app.kubernetes.io/managed-by": "postgres-operator",
            },
        },
        Spec: appsv1.StatefulSetSpec{
            Replicas:    &replicas,
            ServiceName: cluster.Name + "-headless",
            Selector: &metav1.LabelSelector{
                MatchLabels: map[string]string{
                    "app.kubernetes.io/instance": cluster.Name,
                },
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: map[string]string{
                        "app.kubernetes.io/instance": cluster.Name,
                    },
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name:  "postgres",
                            Image: image,
                            Ports: []corev1.ContainerPort{
                                {Name: "postgres", ContainerPort: 5432},
                            },
                            Resources: corev1.ResourceRequirements{
                                Requests: corev1.ResourceList{
                                    corev1.ResourceCPU:    resource.MustParse(cluster.Spec.Resources.CPU),
                                    corev1.ResourceMemory: resource.MustParse(cluster.Spec.Resources.Memory),
                                },
                            },
                            ReadinessProbe: &corev1.Probe{
                                ProbeHandler: corev1.ProbeHandler{
                                    Exec: &corev1.ExecAction{
                                        Command: []string{
                                            "pg_isready", "-U", "postgres",
                                        },
                                    },
                                },
                                InitialDelaySeconds: 5,
                                PeriodSeconds:       10,
                            },
                        },
                    },
                },
            },
        },
    }
}

func (r *PostgresClusterReconciler) reconcileService(
    ctx context.Context,
    cluster *dbv1alpha1.PostgresCluster,
) error {
    svc := &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      cluster.Name + "-primary",
            Namespace: cluster.Namespace,
        },
        Spec: corev1.ServiceSpec{
            Selector: map[string]string{
                "app.kubernetes.io/instance": cluster.Name,
                "role": "primary",
            },
            Ports: []corev1.ServicePort{
                {Name: "postgres", Port: 5432},
            },
        },
    }
    _ = controllerutil.SetControllerReference(cluster, svc, r.Scheme)

    existing := &corev1.Service{}
    err := r.Get(ctx, types.NamespacedName{Name: svc.Name, Namespace: svc.Namespace}, existing)
    if errors.IsNotFound(err) {
        return r.Create(ctx, svc)
    }
    return err
}

func (r *PostgresClusterReconciler) updateStatus(
    ctx context.Context,
    cluster *dbv1alpha1.PostgresCluster,
) error {
    sts := &appsv1.StatefulSet{}
    err := r.Get(ctx, types.NamespacedName{
        Name: cluster.Name + "-postgres", Namespace: cluster.Namespace,
    }, sts)
    if err != nil {
        return err
    }

    cluster.Status.ReadyReplicas = int(sts.Status.ReadyReplicas)
    if sts.Status.ReadyReplicas == sts.Status.Replicas {
        cluster.Status.Phase = "Running"
    } else if sts.Status.ReadyReplicas > 0 {
        cluster.Status.Phase = "Degraded"
    } else {
        cluster.Status.Phase = "Pending"
    }
    cluster.Status.PrimaryEndpoint = fmt.Sprintf(
        "%s-primary.%s.svc.cluster.local:5432",
        cluster.Name, cluster.Namespace,
    )

    return r.Status().Update(ctx, cluster)
}

func (r *PostgresClusterReconciler) handleDeletion(
    ctx context.Context,
    cluster *dbv1alpha1.PostgresCluster,
) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    if controllerutil.ContainsFinalizer(cluster, postgresClusterFinalizer) {
        logger.Info("클린업 수행 (파이널라이저)")
        // 백업 완료 확인, 외부 리소스 정리 등 수행
        // ...

        controllerutil.RemoveFinalizer(cluster, postgresClusterFinalizer)
        if err := r.Update(ctx, cluster); err != nil {
            return ctrl.Result{}, err
        }
    }
    return ctrl.Result{}, nil
}

func (r *PostgresClusterReconciler) setCondition(
    cluster *dbv1alpha1.PostgresCluster,
    condType, status, message string,
) {
    now := metav1.Now()
    for i, c := range cluster.Status.Conditions {
        if c.Type == condType {
            cluster.Status.Conditions[i].Status = status
            cluster.Status.Conditions[i].Message = message
            cluster.Status.Conditions[i].LastTransitionTime = now.Format(time.RFC3339)
            return
        }
    }
    cluster.Status.Conditions = append(cluster.Status.Conditions,
        dbv1alpha1.Condition{
            Type:               condType,
            Status:             status,
            Message:            message,
            LastTransitionTime: now.Format(time.RFC3339),
        })
}

// SetupWithManager는 컨트롤러를 Manager에 등록한다
func (r *PostgresClusterReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&dbv1alpha1.PostgresCluster{}).  // 주 감시 리소스
        Owns(&appsv1.StatefulSet{}).          // 소유 리소스 변경도 감시
        Owns(&corev1.Service{}).
        Complete(r)
}
```

## Kubebuilder로 Operator 프로젝트 초기화

실제 Operator 개발은 Kubebuilder를 사용해 보일러플레이트를 자동 생성한다:

```bash
# Kubebuilder 프로젝트 초기화
kubebuilder init --domain example.com --repo example.com/postgres-operator

# API 및 컨트롤러 생성
kubebuilder create api \
  --group db \
  --version v1alpha1 \
  --kind PostgresCluster

# 생성되는 파일 구조:
# api/v1alpha1/
#   postgrescluster_types.go     ← Go 타입 정의 (CRD 스키마)
#   zz_generated.deepcopy.go     ← 자동 생성 DeepCopy
# config/
#   crd/bases/                   ← 자동 생성 CRD YAML
#   rbac/                        ← RBAC 권한 설정
#   manager/                     ← 컨트롤러 배포 설정
# internal/controller/
#   postgrescluster_controller.go ← 조정 로직 구현 위치

# CRD를 클러스터에 설치
make install

# 컨트롤러를 로컬에서 실행 (개발/디버깅)
make run

# 실제 배포 (Docker 이미지 빌드 후 클러스터에 배포)
make docker-build docker-push IMG=my-registry/postgres-operator:v0.1.0
make deploy IMG=my-registry/postgres-operator:v0.1.0

# PostgresCluster 인스턴스 생성
cat <<EOF | kubectl apply -f -
apiVersion: db.example.com/v1alpha1
kind: PostgresCluster
metadata:
  name: my-postgres
  namespace: production
spec:
  replicas: 3
  version: "16.2"
  storageClass: "fast-ssd"
  resources:
    memory: "4Gi"
    cpu: "2000m"
  backup:
    enabled: true
    schedule: "0 2 * * *"
EOF

# 상태 확인
kubectl get postgrescluster my-postgres -o wide
# NAME          PHASE     READY   AGE
# my-postgres   Running   3       5m

kubectl describe postgrescluster my-postgres
```

## Operator 성숙도 레벨

Operator Framework는 Operator의 성숙도를 5단계로 정의한다:

| 레벨 | 기능 |
|------|------|
| Level 1 - Basic Install | 설치 및 설정 자동화 |
| Level 2 - Seamless Upgrades | 업그레이드 자동화 (롤링 업데이트 등) |
| Level 3 - Full Lifecycle | 백업/복구, 실패 감지 및 자동 복구 |
| Level 4 - Deep Insights | 메트릭, 알림, 로그 분석 |
| Level 5 - Auto Pilot | 수평/수직 자동 스케일링, 자동 튜닝 |

프로덕션 Operator(CloudNativePG, Strimzi, Prometheus Operator)는 대부분 Level 3~4 수준을 목표로 한다.

## 주의사항과 실무 팁

**1. 조정 루프는 항상 항등적(idempotent)이어야 한다.** 컨트롤러는 같은 이벤트를 여러 번 받을 수 있다. "이미 원하는 상태라면 아무것도 하지 말라"는 원칙을 철저히 지켜야 한다. `Create or Update(Apply)` 패턴을 사용하라.

**2. status와 spec을 명확히 분리하라.** spec은 "원하는 상태", status는 "현재 관찰된 상태"다. spec에 관찰값을 쓰거나 status를 의사결정에 직접 사용하면 일관성 문제가 생긴다. status 서브리소스를 활성화하면 etcd에서 두 필드의 충돌이 방지된다.

**3. 리더 선출(Leader Election)을 반드시 활성화하라.** 컨트롤러가 여러 레플리카로 배포될 때, 단 하나만 실제로 조정 작업을 수행해야 한다. `controller-runtime`의 리더 선출은 쿠버네티스 Lease 리소스를 사용한다.

**4. Finalizer를 남용하지 마라.** Finalizer가 있는 리소스는 kubectl delete 후에도 클린업이 완료될 때까지 삭제되지 않는다. 컨트롤러가 죽으면 리소스가 영원히 삭제되지 않는 상황이 발생할 수 있다. 외부 리소스(AWS S3 버킷, 클라우드 DNS 레코드 등) 클린업에만 사용하라.

**5. 이벤트와 메트릭을 풍성하게 기록하라.** `r.Recorder.Event(cluster, corev1.EventTypeNormal, "Created", "StatefulSet 생성됨")` 형태로 쿠버네티스 이벤트를 기록하면 `kubectl describe`에서 디버깅이 훨씬 쉬워진다. Prometheus 메트릭도 함께 노출하라.

Operator 패턴은 쿠버네티스가 단순한 컨테이너 오케스트레이터를 넘어 범용 자동화 플랫폼으로 발전한 핵심 기제다. 복잡한 스테이트풀 애플리케이션의 운영 지식을 선언적 코드로 만들 수 있다는 것 — 이것이 Operator의 가치다.

## 참고 자료
- [Kubernetes Custom Resource Definition 공식 문서](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
- [controller-runtime GitHub](https://github.com/kubernetes-sigs/controller-runtime)
- [Kubebuilder Book](https://book.kubebuilder.io/)
- [Operator Framework: Operator Maturity Model](https://operatorframework.io/operator-capabilities/)
