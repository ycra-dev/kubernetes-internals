# 11.01 · 컨트롤러 카탈로그

**근거**: `kubernetes/pkg/controller/` (38개 서브패키지),
등록 목록 `cmd/kube-controller-manager/app/controller_descriptor.go`

빌트인 컨트롤러를 책임별로 정리한다. 모두 동일한 reconcile 패턴
([00-foundations/05](../00-foundations/05-controller-pattern.md))을 따르므로, "무엇을 원하는 상태로
유지하는가"만 알면 된다.

## 워크로드 컨트롤러 (Pod를 만드는 것들)

| 컨트롤러 | 패키지 | 원하는 상태 |
|----------|--------|-------------|
| Deployment | `deployment/` | 선언한 ReplicaSet/롤아웃 상태 유지 → [03](03-deployment-replicaset.md) |
| ReplicaSet | `replicaset/` | N개의 Pod 유지 |
| ReplicationController | `replication/` | (구버전) N개 Pod, RS의 전신 |
| StatefulSet | `statefulset/` | 순서·고정 ID를 가진 Pod 집합 |
| DaemonSet | `daemon/` | 모든(또는 선택된) 노드마다 Pod 1개 |
| Job | `job/` | 완료까지 Pod 실행 |
| CronJob | `cronjob/` | 일정에 따라 Job 생성 |

## 네트워킹/서비스 컨트롤러

| 컨트롤러 | 패키지 | 역할 |
|----------|--------|------|
| Endpoints | `endpoint/` | Service ↔ 백엔드 Pod IP 목록(레거시 Endpoints) |
| EndpointSlice | `endpointslice/` | 확장형 엔드포인트 분할 → [14-kube-proxy](../14-kube-proxy/) |
| EndpointSliceMirroring | `endpointslicemirroring/` | 수동 Endpoints를 EndpointSlice로 미러 |
| ServiceCIDRs | `servicecidrs/` | 서비스 IP 대역 관리 |
| NodeIPAM | `nodeipam/` | 노드별 Pod CIDR 할당 |

## 노드/스케줄 관련

| 컨트롤러 | 패키지 | 역할 |
|----------|--------|------|
| NodeLifecycle | `nodelifecycle/` | 노드 건강 추적, NotReady 시 taint → [05](05-node-lifecycle.md) |
| TaintEviction | `tainteviction/` | NoExecute taint에 맞춰 Pod 축출 |
| PodGC | `podgc/` | 종료/고아 Pod 정리 |
| TTLAfterFinished | `ttlafterfinished/` | 완료된 Job을 TTL 후 삭제 |

## 스토리지 컨트롤러

| 컨트롤러 | 패키지 | 역할 |
|----------|--------|------|
| PV Binder / AttachDetach / Expander | `volume/` | PVC↔PV 바인딩, 볼륨 attach/detach, 확장 → [50-storage](../50-storage/) |
| EphemeralVolume | `volume/ephemeral/` | 임시 볼륨용 PVC 생성 |
| ResourceClaim | `resourceclaim/` | 동적 리소스 할당(DRA) |

## 보안/정책/시스템

| 컨트롤러 | 패키지 | 역할 |
|----------|--------|------|
| GarbageCollector | `garbagecollector/` | ownerReference 그래프 기반 연쇄 삭제 → [04](04-garbage-collector.md) |
| ResourceQuota | `resourcequota/` | 네임스페이스 사용량 집계(쿼터 시행은 어드미션) |
| Namespace | `namespace/` | 네임스페이스 삭제 시 내부 객체 정리 |
| ServiceAccount + Token | `serviceaccount/` | SA와 토큰 생성/발급 |
| CSR (Signing/Approving/Cleaner) | `certificates/` | 인증서 서명 요청 처리 |
| ClusterRoleAggregation | `clusterroleaggregation/` | 라벨로 ClusterRole 규칙 집계 |
| Disruption | `disruption/` | PodDisruptionBudget 상태 계산 |

## 오토스케일링

| 컨트롤러 | 패키지 | 역할 |
|----------|--------|------|
| HorizontalPodAutoscaler | `podautoscaler/` | 메트릭 기반 replica 수 조정 |

> **주의**: HPA는 여기(controller-manager) 안에 있다. **Cluster Autoscaler(노드 수 조정)** 와
> **VPA(리소스 요청 조정)** 는 별도 레포(`autoscaler/`)의 독립 컴포넌트다 →
> [70-autoscaling](../70-autoscaling/).

## 공통점

모든 컨트롤러가 같은 골격이다: *공유 Informer로 watch → 변경 시 workqueue에 enqueue → 워커가
sync(key) → spec과 status의 차이를 apiserver 쓰기로 좁힘.* 공통 머시너리는
[02](02-common-machinery.md).

## 더 읽을 곳
- [03-deployment-replicaset.md](03-deployment-replicaset.md) — 대표 컨트롤러 심화
