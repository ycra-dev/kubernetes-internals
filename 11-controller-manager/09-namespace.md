# 11.09 · Namespace 컨트롤러

**근거**: `kubernetes/pkg/controller/namespace/` (`namespace_controller.go` `discoverResourcesFn` `:70`,
`ResourcesRemainingError` `:157`, `deletion/namespaced_resources_deleter.go`)

`kubectl delete namespace x`를 하면 그 안의 **모든 리소스**(Pod, Service, ConfigMap, Secret, 그리고
CRD까지)가 함께 사라진다. 이를 처리하는 것이 Namespace 컨트롤러다. "한 종류가 아니라 모든 종류의
리소스를 지워야" 하는 점이 다른 컨트롤러와 다른 흥미로운 부분이다.

## 문제 — 무엇을 지워야 하는지 모른다

ReplicaSet 컨트롤러는 "Pod"만 안다. 하지만 Namespace 컨트롤러는 그 네임스페이스 안에 **어떤 종류의
리소스가 있는지** 미리 알 수 없다 — 빌트인 타입뿐 아니라 사용자가 만든 CRD
([10-apiserver/07](../10-apiserver/07-crd-aggregation.md))도 있을 수 있다.

## discovery로 리소스 종류 발견

해결: **discovery**([10-apiserver/11](../10-apiserver/11-discovery-openapi.md))로 "현재 클러스터에 존재하는
네임스페이스 스코프 리소스 종류"를 동적으로 알아낸다(`namespace_controller.go:70` `discoverResourcesFn`).
CRD가 추가/제거되면 discovery가 갱신되므로, 컨트롤러는 항상 최신 리소스 종류 목록을 본다.

```
namespace 삭제 요청 → DeletionTimestamp + finalizer (00-foundations/05)
   │
discoverResources: 이 클러스터의 모든 namespaced 리소스 종류 나열 (CRD 포함)
   │  각 GVR에 대해
   ▼
deleteAllContent: 그 종류의, 이 네임스페이스 안 객체를 모두 삭제 (metadata client, dynamic)
   │  모두 사라졌나?
   ├─ 아직 남음 → ResourcesRemainingError(:157) → 백오프 후 재시도
   └─ 다 사라짐 → finalizer 제거 → 네임스페이스 실제 삭제
```

## finalizer 기반 — 정리 후 삭제

Namespace 객체에는 finalizer가 있다([00-foundations/05](../00-foundations/05-controller-pattern.md)). 삭제
요청 시 apiserver는 `DeletionTimestamp`만 찍고, 컨트롤러가 **내부 모든 리소스를 지운 뒤** finalizer를
제거해야 비로소 네임스페이스가 사라진다. 그래서 네임스페이스가 한동안 **Terminating** 상태에 머문다.

`namespaced_resources_deleter.go`가 각 리소스 종류를 순회하며 그 네임스페이스의 객체를 삭제한다. dynamic/
metadata 클라이언트([00-foundations/08](../00-foundations/08-client-go-clients.md))를 써서 **타입을 몰라도**
임의 GVR의 객체를 지운다 — CRD 객체도 같은 코드로 정리.

## ResourcesRemainingError — 재시도

`ResourcesRemainingError`(`namespace_controller.go:157`)는 "아직 남은 객체가 있다"는 신호다. 객체에
자체 finalizer가 있으면(예: PVC protection, 사용자 CRD의 finalizer) 즉시 안 사라지므로, 컨트롤러는
추정 시간(`:158`)만큼 기다렸다 재시도한다. 모든 객체가 정리될 때까지 반복 — 그래서 어떤 리소스의
finalizer가 영영 안 풀리면 네임스페이스가 Terminating에 갇힐 수 있다(흔한 운영 이슈).

## 왜 Terminating에 갇히나 (트러블슈팅)

[99-appendix/troubleshooting](../99-appendix/troubleshooting.md)의 흔한 사례:

- 네임스페이스 안 객체에 **풀리지 않는 finalizer**가 있음(그 finalizer를 책임지는 컨트롤러가 죽었거나
  외부 의존성 장애).
- **Aggregated API**([10-apiserver/07](../10-apiserver/07-crd-aggregation.md))가 다운되어 그 그룹의 리소스를
  나열/삭제 못 함 → discovery 실패 → 정리 진행 불가.

해결은 막힌 finalizer를 제거하거나, 다운된 APIService를 복구하는 것이다.

## 다른 GC와의 관계

| | GarbageCollector ([04](04-garbage-collector.md)) | Namespace 컨트롤러 (이 문서) |
|--|--------------------------------------------------|------------------------------|
| 기준 | ownerReference 그래프 | namespace 소속 |
| 트리거 | owner 삭제 | namespace 삭제 |
| 범위 | 소유 관계 | 네임스페이스 전체 |

네임스페이스를 지우면 그 안 객체가 사라지고, 그 객체를 owner로 하던 dependent는 다시 GC가 정리한다 —
두 메커니즘이 협력한다.

## 더 읽을 곳
- [04-garbage-collector.md](04-garbage-collector.md) — ownerReference GC
- [00-foundations/05](../00-foundations/05-controller-pattern.md) — finalizer
- [10-apiserver/11](../10-apiserver/11-discovery-openapi.md) — 리소스 종류 발견(discovery)
- [45-policy/README](../45-policy/README.md) — 네임스페이스(멀티테넌시 경계)
