# 99 · Well-Known 라벨·Taint·어노테이션 참조

**근거**: `kubernetes/staging/src/k8s.io/api/core/v1/well_known_labels.go`,
`well_known_taints.go`

Kubernetes는 시스템이 의미를 부여하는 **예약된 라벨/taint/어노테이션**을 정의한다. 코드를 읽거나
디버깅할 때 자주 마주치므로, 그 의미와 "어느 컴포넌트가 다루는지"를 한곳에 모았다. 개념은 본문에, 여기선
빠른 참조다.

## 토폴로지 라벨 (노드)

`well_known_labels.go`. 노드가 어디 있는지를 나타내며, 스케줄링/라우팅에 쓰인다.

| 라벨 | 의미 | 쓰는 곳 |
|------|------|---------|
| `kubernetes.io/hostname` (`:20`) | 노드 이름 | topology spread/affinity의 노드 단위 ([12-scheduler/05](../12-scheduler/05-affinity-topology.md)) |
| `topology.kubernetes.io/zone` (`:26`) | 가용 영역(존) | 존 분산, 토폴로지 인지 라우팅 ([14-kube-proxy/01](../14-kube-proxy/01-endpointslice-tracking.md)) |
| `topology.kubernetes.io/region` (`:27`) | 리전 | 광역 분산 |
| `kubernetes.io/os`, `kubernetes.io/arch` | OS/아키텍처 | nodeSelector(멀티 아키, [30-containerd/09](../30-containerd/09-oci-image-format.md)) |
| `node.kubernetes.io/instance-type` | 인스턴스 타입 | 워크로드 배치 |

> `failure-domain.beta.kubernetes.io/*`(`:33`)는 **deprecated** — `topology.kubernetes.io/*`로 대체됨
> ([91-versioning-skew](../91-versioning-skew.md)의 deprecation). 이 노드 토폴로지 라벨은 보통
> cloud-controller-manager나 kubelet이 채운다([16-cloud-controller-manager](../16-cloud-controller-manager.md),
> [13-kubelet/06](../13-kubelet/06-node-registration.md)).

## Well-Known Taint (노드)

`well_known_taints.go`. 노드 상태가 나빠지면 시스템이 자동으로 붙이는 `NoExecute`/`NoSchedule` taint다
([11-controller-manager/05](../11-controller-manager/05-node-lifecycle.md), [13-kubelet/04](../13-kubelet/04-eviction.md)).

| Taint | 값 | 추가 주체 |
|-------|-----|-----------|
| NotReady | `node.kubernetes.io/not-ready` (`:22`) | NodeLifecycle(하트비트 끊김) |
| Unreachable | `node.kubernetes.io/unreachable` (`:27`) | NodeLifecycle(도달 불가) |
| Unschedulable | `node.kubernetes.io/unschedulable` (`:31`) | cordon/drain |
| MemoryPressure | `node.kubernetes.io/memory-pressure` (`:35`) | kubelet(메모리 압박) |
| DiskPressure | `node.kubernetes.io/disk-pressure` (`:39`) | kubelet(디스크 압박) |
| PIDPressure | `node.kubernetes.io/pid-pressure` | kubelet(PID 압박) |
| NetworkUnavailable | `node.kubernetes.io/network-unavailable` | CNI 미설정 |

이들은 [11-controller-manager/05](../11-controller-manager/05-node-lifecycle.md)(NodeLifecycle→taint→eviction)와
[13-kubelet/04](../13-kubelet/04-eviction.md)(자원 압박)에서 동작한다. Pod는 톨러레이션으로 일부를 견딜 수
있다([12-scheduler/02](../12-scheduler/02-plugins.md)의 tainttoleration).

## 시스템 사용자/그룹

신원 관련 예약 문자열([10-apiserver/02](../10-apiserver/02-authentication.md), [03](../10-apiserver/03-authorization.md)):

| 문자열 | 의미 |
|--------|------|
| `system:node:<name>` | kubelet 신원([13-kubelet/06](../13-kubelet/06-node-registration.md)) |
| `system:nodes` | 노드 그룹 |
| `system:serviceaccount:<ns>:<name>` | ServiceAccount([17-security/01](../17-security/01-serviceaccount-secrets.md)) |
| `system:serviceaccounts` | 모든 SA 그룹 |
| `system:authenticated` / `system:unauthenticated` | 인증/비인증 그룹 |
| `system:masters` | 클러스터 관리자(부트스트랩 RBAC) |

## 흔한 finalizer

삭제 제어용 예약 finalizer([00-foundations/05](../00-foundations/05-controller-pattern.md),
[11-controller-manager/04](../11-controller-manager/04-garbage-collector.md)):

| finalizer | 의미 |
|-----------|------|
| `foregroundDeletion` | foreground GC(dependent 먼저) |
| `orphan` | dependent를 고아로 |
| `kubernetes.io/pvc-protection`, `pv-protection` | 사용 중 PVC/PV 삭제 보호 |

## 사용 규칙

- `kubernetes.io/`, `k8s.io/` 접두사는 **예약**됨 — 사용자가 임의로 쓰면 안 된다.
- `*.beta.kubernetes.io/`는 대개 deprecated(stable로 이전, [91-versioning-skew](../91-versioning-skew.md)).
- 사용자 정의 라벨/어노테이션은 자기 도메인 접두사(`example.com/...`)를 권장.

## 더 읽을 곳
- [00-foundations/11](../00-foundations/11-selectors.md) — 라벨 셀렉터
- [glossary.md](glossary.md) — 용어집
- [troubleshooting.md](troubleshooting.md) — taint/상태 기반 장애 추적
