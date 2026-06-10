# 13.12 · 노드 수준 Pod Admission

**근거**: `kubernetes/pkg/kubelet/lifecycle/` (`predicate.go` `predicateAdmitHandler` `:101`,
`PodAdmitHandler`, `handlers.go`, `interfaces.go`)

[10-apiserver/04](../10-apiserver/04-admission.md)의 어드미션은 apiserver에서 객체를 검증한다. 하지만
스케줄러가 노드를 정한 뒤에도([12-scheduler](../12-scheduler/)), **그 노드의 kubelet이 Pod를 거부할 수
있다.** 노드의 실시간 상태는 스케줄 시점과 다를 수 있기 때문이다. 이것이 **노드 수준 pod admission**이다.

## 왜 노드에서 또 검사하나

스케줄러는 캐시(nodeInfo 스냅샷, [12-scheduler/07](../12-scheduler/07-internals.md))를 보고 배정한다.
하지만 배정과 실제 시작 사이에 노드 상태가 바뀔 수 있다:

- 그사이 다른 Pod가 자원을 더 써서 실제로는 자리가 없어짐.
- 노드의 토폴로지/장치 상태가 바뀜([13-kubelet/09](09-topology-cadvisor.md)).
- static pod처럼 스케줄러를 안 거친 Pod([08-static-pods-shutdown.md](08-static-pods-shutdown.md)).

그래서 kubelet이 **컨테이너를 띄우기 직전** 마지막으로 검사한다.

## PodAdmitHandler 체인

`pkg/kubelet/lifecycle/`의 **`PodAdmitHandler`** 인터페이스(`interfaces.go`)가 핵심이다. SyncLoop
([01-pod-lifecycle.md](01-pod-lifecycle.md))이 Pod를 받으면, 여러 admit handler를 순서대로 통과시킨다
(`handlers.go`). 하나라도 거부하면 Pod는 시작되지 않고 **Rejected** 상태가 된다.

대표 핸들러 — `predicateAdmitHandler`(`predicate.go:101`):

- 스케줄러의 노드 적합성 술어(predicate) 일부를 **노드의 실제 현재 상태로** 다시 평가한다.
- 자원이 실제로 충분한가, 노드 셀렉터/affinity가 맞는가, 장치가 있는가 등.
- 실패 시 거부 사유를 status에 기록(`predicate.go:70`~의 거부 이유 처리).

다른 핸들러: cgroup/자원 매니저의 admission([13-kubelet/03](03-resource-mgmt.md)), Topology Manager의
NUMA 정렬 검사([13-kubelet/09](09-topology-cadvisor.md), `restricted`/`single-numa-node` 정책이 여기서
거부), eviction 임박 시 거부 등.

## 거부되면 어떻게 되나

```
스케줄러: Pod를 노드 N에 배정 (nodeName=N)
   │
노드 N의 kubelet: PodAdmitHandler 체인 평가
   ├─ 모두 통과 → SyncPod로 컨테이너 시작 (01-pod-lifecycle)
   └─ 거부 → Pod status=Failed (Rejected), 컨테이너 안 뜸
              │
소유 컨트롤러(ReplicaSet 등)가 새 Pod 생성 → 스케줄러가 다른 노드로
```

거부된 Pod는 그 노드에서 실패로 끝나고, 소유 컨트롤러가 대체 Pod를 만들어 재스케줄된다
([00-foundations/05](../00-foundations/05-controller-pattern.md), [11-controller-manager/03](../11-controller-manager/03-deployment-replicaset.md)).
즉 "스케줄러의 낙관적 배정 + 노드의 최종 거부 + 컨트롤러의 재시도"가 협력해 결국 적합한 노드에 안착한다.

## apiserver 어드미션과의 구분

| | apiserver 어드미션 | 노드 pod admission |
|--|-------------------|---------------------|
| 시점 | 객체 저장 전 | 컨테이너 시작 직전 |
| 기준 | API 객체 내용/정책 | 노드의 실시간 상태(자원/토폴로지) |
| 거부 결과 | 요청 거부(객체 안 생김) | Pod Failed(Rejected) → 재스케줄 |
| 코드 | `apiserver/pkg/admission` | `kubelet/pkg/lifecycle` |

## 더 읽을 곳
- [01-pod-lifecycle.md](01-pod-lifecycle.md) — admit 통과 후 SyncPod
- [12-scheduler/07](../12-scheduler/07-internals.md) — 스케줄러의 낙관적 배정(assume)
- [03-resource-mgmt.md](03-resource-mgmt.md) — 자원 매니저 admission
