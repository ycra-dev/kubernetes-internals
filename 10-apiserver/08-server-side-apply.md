# 10.08 · Server-Side Apply (필드 관리)

**근거**: `kubernetes/staging/src/k8s.io/apiserver/pkg/endpoints/handlers/fieldmanager/`,
`ObjectMeta.ManagedFields`([00-foundations/03](../00-foundations/03-api-object-model.md))

여러 주체(사용자, kubectl, 컨트롤러, HPA)가 같은 객체를 동시에 수정하면 누가 어떤 필드의 주인인지가
모호해진다. **Server-Side Apply(SSA)** 는 **필드별 소유권**을 서버가 추적해 이 충돌을 명확히 한다.

## 문제 — 누가 이 필드의 주인인가

예: 사용자가 Deployment에 `replicas: 3`을 선언했는데, HPA가 `replicas: 10`으로 올린다. 다음에 사용자가
`apply`하면 3으로 되돌려야 하나? SSA 이전에는 클라이언트 측 3-way merge(last-applied annotation)로
처리했지만 깨지기 쉬웠다.

## managedFields — 필드 소유권 기록

SSA는 `ObjectMeta.ManagedFields`에 **"어떤 manager가 어떤 필드를 소유하는가"** 를 기록한다. 각 필드
경로마다 그것을 마지막으로 설정한 manager(예: `kubectl`, `kube-controller-manager`,
`hpa-controller`)가 적힌다.

- `apply` 요청 시 클라이언트는 자기가 **소유를 주장하는 필드만** 보낸다.
- 서버는 그 필드들을 그 manager 소유로 표시하고 값을 병합한다.
- 어떤 manager가 자기 소유 필드를 빼고 apply하면, 그 필드 소유를 **포기**한 것으로 본다(필드 제거).

구현은 `endpoints/handlers/fieldmanager/`. `pod.yaml`/`node.yaml` 등은 각 타입의 필드 구조(어떤 필드가
원자적/리스트 키인지)를 정의한다.

## 충돌(conflict)

두 manager가 **같은 필드**를 서로 다른 값으로 소유하려 하면 **충돌**이 난다. SSA는:

- 기본적으로 충돌 시 요청을 거부한다(`Conflict` 에러) — 모르고 남의 필드를 덮는 것을 막는다.
- `force=true`로 강제하면 소유권을 가져오며 덮어쓴다.

이로써 "HPA가 관리하는 replicas를 사용자가 실수로 덮는" 상황을 명시적 충돌로 드러낸다.

## apply는 별도 verb

SSA는 `PATCH`의 한 형태(`application/apply-patch+yaml` content type)로 들어오며, 어드미션/검증
([01-request-pipeline.md](01-request-pipeline.md))을 똑같이 거친다. 즉 SSA도 일반 쓰기 경로 위에 얹힌
필드 병합 의미론이다.

## 클라이언트 측과의 관계

[15-cli](../15-cli/)의 `kubectl apply`는 점점 SSA를 기본으로 쓴다. controller-runtime 기반 오퍼레이터도
SSA로 자기 소유 필드만 패치해, 다른 컨트롤러/사용자와 충돌 없이 협력할 수 있다
([60-controller-runtime/02](../60-controller-runtime/02-client-cache.md)).

## 더 읽을 곳
- [00-foundations/03](../00-foundations/03-api-object-model.md) — ManagedFields가 사는 ObjectMeta
- [15-cli](../15-cli/) — kubectl apply
