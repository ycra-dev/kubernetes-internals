# 11 · kube-controller-manager

**근거 레포**: `kubernetes` (`cmd/kube-controller-manager`, `pkg/controller/`)

kube-controller-manager는 **수십 개의 빌트인 컨트롤러를 한 프로세스에 모아 돌리는** 컴포넌트다.
각 컨트롤러는 [00-foundations/05](../00-foundations/05-controller-pattern.md)에서 본 reconcile 루프를
구현해, 자기가 책임지는 리소스를 "원하는 상태"로 수렴시킨다.

## 역할

- Deployment→ReplicaSet→Pod, Job, StatefulSet, DaemonSet, CronJob 등 **워크로드 컨트롤러** 실행.
- Node 라이프사이클, EndpointSlice, ServiceAccount/토큰, GC, PV 바인딩 등 **시스템 컨트롤러** 실행.
- 모두 apiserver를 통해 list/watch 하고 apiserver에 쓴다. etcd 직접 접근 없음.

## 컨트롤러 등록 — descriptor 맵

모든 컨트롤러는 `cmd/kube-controller-manager/app/controller_descriptor.go:138`의
`NewControllerDescriptors()`에서 이름→생성자(`ControllerDescriptor`)로 등록된다(`register(...)`,
`:174`~`:225`). 약 50개가 등록돼 있다. 발췌:

```
ServiceAccountToken (특수: 가장 먼저 시작)  // :174 — 이후 컨트롤러용 SA 토큰 보장
Endpoints / EndpointSlice / EndpointSliceMirroring
ReplicationController / ReplicaSet / Deployment / StatefulSet / DaemonSet
Job / CronJob / TTLAfterFinished
HorizontalPodAutoscaler                     // :190 (HPA는 여기, CA는 autoscaler 레포)
GarbageCollector / PodGarbageCollector
ResourceQuota / Namespace
NodeIpam / NodeLifecycle / TaintEviction
PersistentVolume {Binder,AttachDetach,Expander,Protection} / EphemeralVolume
CertificateSigningRequest {Signing,Approving,Cleaner}
ServiceAccount / ClusterRoleAggregation / BootstrapSigner ...
```

> `register`는 중복 이름/별칭, nil 생성자를 panic으로 막는다(`:143`~`:166`) — 등록 일관성을 강제.
> `ServiceAccountTokenController`만 "특수"로 표시돼 가장 먼저 시작된다(`:171` 주석): 이후 컨트롤러들이
> 쓸 SA 토큰이 먼저 존재해야 하기 때문.

## 실행과 고가용성

`controllermanager.go`(`app/`)가 부팅 시:

1. 공유 InformerFactory를 만들어 모든 컨트롤러가 watch를 공유하게 한다([04](../00-foundations/04-list-watch-informer.md)의 SharedInformer).
2. **리더 선출**을 통과한 인스턴스에서만 컨트롤러들을 시작한다
   ([00-foundations/06](../00-foundations/06-leader-election.md)). HA로 여러 개 떠도 활성은 하나.
3. 각 컨트롤러의 `Run(ctx, workers)`를 고루틴으로 띄운다.

## 문서

| # | 문서 | 내용 |
|---|------|------|
| 01 | [controller-catalog.md](01-controller-catalog.md) | 어떤 컨트롤러가 무엇을 하나 |
| 02 | [common-machinery.md](02-common-machinery.md) | 공통 유틸·Informer·workqueue·expectations |
| 03 | [deployment-replicaset.md](03-deployment-replicaset.md) | 깊게: Deployment→RS→Pod |
| 04 | [garbage-collector.md](04-garbage-collector.md) | ownerReference 기반 GC |
| 05 | [node-lifecycle.md](05-node-lifecycle.md) | 노드 건강 추적·taint·eviction |
| 06 | [statefulset.md](06-statefulset.md) | 순서·고정 신원·Pod별 PVC |
| 07 | [daemonset.md](07-daemonset.md) | 노드마다 Pod 1개 |
| 08 | [job-cronjob.md](08-job-cronjob.md) | 완료형 작업·일정 실행 |
| 09 | [namespace.md](09-namespace.md) | 네임스페이스 삭제(discovery 기반 전체 정리) |

## 더 읽을 곳
- [00-foundations/05](../00-foundations/05-controller-pattern.md) — 컨트롤러 일반 패턴
- [70-autoscaling](../70-autoscaling/) — HPA(여기) vs cluster-autoscaler(별도 레포) 구분
