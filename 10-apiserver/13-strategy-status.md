# 10.13 · REST 전략과 서브리소스 (status/scale)

**근거**: `kubernetes/pkg/registry/apps/deployment/storage/storage.go`
(`REST`/`StatusREST`/`RollbackREST` `:53`~`:116`), `kubernetes/pkg/apis/apps/.../strategy.go`

[05-registry-storage.md](05-registry-storage.md)에서 generic registry Store와 전략을 개요로 봤다. 이
문서는 그 전략이 **리소스마다 어떻게 구현**되는지와, **서브리소스**(`/status`, `/scale`)가 왜 별도
엔드포인트인지를 Deployment 예로 본다.

## 한 리소스, 여러 REST 핸들러

Deployment의 storage 구성(`storage.go:94` `NewREST`)은 **세 개의 REST 핸들러**를 만든다:

```go
// storage.go:53~116
type DeploymentStorage struct {
    Deployment *REST          // /deployments        — 본체
    Status     *StatusREST    // /deployments/status — status 서브리소스
    Rollback   *RollbackREST  // /deployments/rollback
    Scale      *ScaleREST     // /deployments/scale  — scale 서브리소스
}
```

같은 객체(같은 etcd 키)지만 **다른 엔드포인트 = 다른 전략 = 다른 권한**으로 노출된다.

## 전략 두 개 — main vs status

`storage.go:102`와 `:114`가 핵심이다:

```go
store.UpdateStrategy       = deployment.Strategy        // 본체: spec 수정
statusStore.UpdateStrategy = deployment.StatusStrategy  // status: status만 수정
```

- **`Strategy`**: 본체 PUT/PATCH. **spec은 바꿀 수 있지만 status는 무시**한다(사용자가 status를 못
  바꾸게).
- **`StatusStrategy`**: `/status` 엔드포인트. **status는 바꾸지만 spec은 무시**한다(컨트롤러가 status만
  보고하게).

이것이 [00-foundations/03](../00-foundations/03-api-object-model.md)의 **spec/status 분리**를 강제하는
실제 메커니즘이다 — 사용자는 spec을, 컨트롤러는 status를 쓰되 서로의 영역을 침범 못 한다. RBAC로
`/status` 권한을 따로 줄 수도 있다.

## 왜 서브리소스인가

### /status
컨트롤러가 status를 갱신할 때 본체 PUT을 쓰면 사용자의 spec 변경과 충돌한다. `/status` 서브리소스로
**status만** 갱신하면 충돌이 없다. controller-runtime의 `Status().Update()`
([60-controller-runtime/02](../60-controller-runtime/02-client-cache.md))가 이걸 친다.

### /scale
`/scale` 서브리소스는 **replicas만** 노출하는 작은 뷰(`autoscaling/v1` Scale 객체)다. 이로써:
- **HPA**([70-autoscaling/02](../70-autoscaling/02-hpa.md))가 Deployment의 전체 스펙을 몰라도 replicas만
  조정할 수 있다 — `/scale`만 PATCH.
- `kubectl scale`도 이걸 쓴다.
- Deployment/ReplicaSet/StatefulSet이 **공통 Scale 인터페이스**를 가져, HPA가 종류를 몰라도 스케일한다
  (`client-go/scale`, [00-foundations/08](../00-foundations/08-client-go-clients.md)).

## 전략이 하는 일 (재확인)

각 전략은 [05-registry-storage.md](05-registry-storage.md)의 단계를 리소스별로 구현한다
(`pkg/apis/apps/validation`, `strategy.go`):

- **PrepareForCreate/Update**: 사용자가 못 바꾸는 필드 보호(status 클리어, 불변 필드 복원).
- **Validate/ValidateUpdate**: 타입 고유 검증(예: Deployment의 selector 불변성).
- **Canonicalize**: 정규화.

`RollbackREST`(`storage.go`)는 Deployment 전용 특수 서브리소스로, 롤백 요청을 처리한다
([11-controller-manager/03](../11-controller-manager/03-deployment-replicaset.md)).

## 정리

```
/deployments         → Strategy       (spec 쓰기, status 무시)
/deployments/status  → StatusStrategy (status 쓰기, spec 무시)  ← 컨트롤러용
/deployments/scale   → Scale 뷰       (replicas만)             ← HPA/kubectl scale용
모두 같은 etcd 객체, 다른 엔드포인트/전략/권한
```

## 더 읽을 곳
- [05-registry-storage.md](05-registry-storage.md) — generic registry 개요
- [70-autoscaling/02](../70-autoscaling/02-hpa.md) — /scale을 쓰는 HPA
- [00-foundations/03](../00-foundations/03-api-object-model.md) — spec/status 분리
