# 60.04 · Webhook와 envtest

**근거**: `controller-runtime/pkg/webhook/`, `pkg/conversion/`, `pkg/envtest/`

오퍼레이터는 컨트롤러(비동기 reconcile)뿐 아니라, **동기적 어드미션/변환 웹훅**도 제공할 수 있다.
controller-runtime은 그 웹훅 서버와, 테스트용 실제 apiserver(envtest)를 제공한다.

## Admission Webhook

[10-apiserver/04](../10-apiserver/04-admission.md)에서 본 동적 어드미션의 **서버 쪽**을 쉽게 구현하게
한다(`pkg/webhook/admission/`):

- **Defaulting(Mutating) webhook**: 객체에 기본값을 채운다(예: CRD의 빈 필드).
- **Validating webhook**: CRD의 복잡한 검증(스키마/CEL로 표현 못 하는 규칙).

개발자는 `Default(obj)`/`ValidateCreate(obj)` 같은 메서드만 구현하고, controller-runtime이 HTTPS
`AdmissionReview` 서버, 인증서, 라우팅을 처리한다. Manager의 Runnable로 등록돼 컨트롤러와 함께 뜬다
([01](01-manager-reconciler.md)).

## Conversion Webhook

CRD가 여러 버전(v1alpha1, v1)을 가지면, 버전 간 변환이 필요하다
([10-apiserver/07](../10-apiserver/07-crd-aggregation.md)). `pkg/conversion/`이 변환 웹훅을 구현하게
돕는다 — apiserver가 저장/응답 시 호출한다.

## 컨트롤러 vs 웹훅 — 타이밍의 차이

| | 컨트롤러(Reconcile) | 웹훅(Admission) |
|--|--------------------|-----------------|
| 시점 | 객체가 저장된 **후**, 비동기 | 객체가 저장되기 **전**, 동기 |
| 실패 시 | 재시도(결국 수렴) | 요청 자체가 거부됨 |
| 용도 | 원하는 상태로 수렴(하위 객체 생성 등) | 즉시 검증/기본값/거부 |

오퍼레이터는 보통 둘을 함께 쓴다: 웹훅으로 입력을 검증/정규화하고, 컨트롤러로 실제 리소스를 만든다.

## envtest — 실제 apiserver로 테스트

`pkg/envtest/`는 테스트에서 **진짜 apiserver + etcd 바이너리**를 띄운다(스케줄러/kubelet 없이).
그래서:

- 가짜 클라이언트가 아니라 **실제 API 동작**(어드미션, 검증, watch, RBAC)을 상대로 reconcile을 테스트.
- CRD를 실제로 등록하고 객체를 만들어 컨트롤러가 올바르게 반응하는지 검증.

이는 mock보다 충실도가 높아 controller-runtime 기반 오퍼레이터의 표준 테스트 방식이다.

## 더 읽을 곳
- [10-apiserver/04](../10-apiserver/04-admission.md) — 웹훅을 호출하는 apiserver 측
- [61-kubebuilder.md](../61-kubebuilder.md) — 웹훅/테스트 스캐폴딩 생성
