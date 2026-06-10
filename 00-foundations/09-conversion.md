# 00.09 · 버전 변환과 기본값 (Conversion / Defaulting)

**근거**: `kubernetes/staging/src/k8s.io/apimachinery/pkg/runtime/scheme.go`
(`Convert` `:396`, `AddConversionFunc` `:327`), `pkg/conversion/converter.go`

[03-api-object-model.md](03-api-object-model.md)에서 "같은 객체가 여러 버전을 가지고 Scheme이 변환을
보유한다"고 했다. 이 문서는 그 **변환(conversion)과 기본값(defaulting)** 메커니즘을 본다. 이것이 apiserver가
`v1beta1`로 받은 요청을 `v1`로 저장하고 다시 `v1beta1`로 응답할 수 있게 하는 핵심이다.

## 왜 변환이 필요한가

같은 Kind라도 버전이 여럿이다(`apps/v1beta1` Deployment, `apps/v1` Deployment). 하지만:
- **etcd에는 하나의 버전(저장 버전)만** 저장한다([10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md)).
- 클라이언트는 자기가 원하는 버전으로 읽고 쓴다.

따라서 apiserver는 요청/응답/저장 사이에서 **버전을 끊임없이 변환**해야 한다.

## hub-and-spoke — 내부 버전

모든 버전이 서로 직접 변환하면 N² 개의 변환 함수가 필요하다. Kubernetes는 **internal version(허브)** 을
두어 이를 N개로 줄인다:

```
v1beta1 ──┐                    ┌── v1beta1
v1beta2 ──┼──► __internal__ ──┼── v1beta2
v1      ──┘   (허브, 메모리만)  └── v1
```

- 외부 버전 ↔ internal 변환만 정의하면, 임의의 두 외부 버전은 internal을 거쳐 변환된다.
- internal 버전은 **메모리 내부 표현**일 뿐 etcd/와이어로 나가지 않는다. 컨트롤러/registry 로직은 보통
  internal 타입으로 동작한다.

## Scheme.Convert — 변환 실행

`scheme.go:396`의 `Scheme.Convert(in, out, context)`가 변환의 진입점이다. Scheme에 등록된 변환 함수
(`AddConversionFunc`, `:327`)를 찾아 실행한다. 대부분의 변환 함수는 **conversion-gen**이 생성한다
([95-development/01](../95-development/01-code-generation.md)) — 필드가 같으면 자동 복사, 다르면 손으로 짠
변환을 호출.

`pkg/conversion/converter.go`가 그 하부 엔진으로, 리플렉션/생성 코드로 필드를 옮긴다.

## defaulting — 기본값 채우기

변환과 짝을 이루는 것이 **defaulting**이다. 객체가 들어오면 비어 있는 필드에 기본값을 채운다
(defaulter-gen이 생성, [95-development/01](../95-development/01-code-generation.md)). 예: Deployment에
`replicas`를 안 적으면 기본 1.

**중요한 순서**: defaulting은 **버전별**로 다를 수 있어, **외부 버전에서 internal로 변환하기 전에** 그
외부 버전의 기본값을 적용한다. 그래야 버전마다 다른 기본값 정책을 정확히 반영한다.

## apiserver에서의 전체 흐름

요청 처리에서 변환/기본값이 끼는 지점([10-apiserver/01](../10-apiserver/01-request-pipeline.md)):

```
요청(v1beta1 JSON)
  → 디코딩 → v1beta1 객체
  → defaulting (v1beta1 기본값)
  → 변환: v1beta1 → internal
  → 어드미션/검증/registry (internal로)
  → 변환: internal → 저장버전(v1) → etcd 저장
응답:
  → etcd(v1) → internal → 클라이언트 요청 버전(v1beta1) → 인코딩
```

## CRD의 변환

빌트인은 컴파일된 변환 함수를 쓰지만, CRD는 코드가 없다. CRD의 다중 버전은:
- **None**: 변환 안 함(필드 호환 가정).
- **Webhook**: conversion webhook으로 변환([10-apiserver/07](../10-apiserver/07-crd-aggregation.md),
  [60-controller-runtime/04](../60-controller-runtime/04-webhook-envtest.md)).

## 왜 이게 중요한가

이 변환 계층 덕분에:
- 오래된 클라이언트(`v1beta1`)가 새 클러스터에서도 동작한다([91-versioning-skew](../91-versioning-skew.md)).
- API를 깨지 않고 새 버전으로 진화시킬 수 있다.
- 저장 버전을 바꿔도(storage version migration) 클라이언트는 영향받지 않는다.

## 더 읽을 곳
- [03-api-object-model.md](03-api-object-model.md) — GVK/Scheme 개요
- [91-versioning-skew.md](../91-versioning-skew.md) — 버전 생애주기/저장 버전
- [95-development/01](../95-development/01-code-generation.md) — conversion-gen/defaulter-gen
