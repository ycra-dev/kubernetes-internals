# 10.07 · API 확장 — CRD와 Aggregation

**근거**: `staging/src/k8s.io/apiextensions-apiserver/`, `staging/src/k8s.io/kube-aggregator/`

Kubernetes API는 고정돼 있지 않다. 운영자/개발자가 **새 리소스 타입**을 추가할 수 있고, 두 가지 길이 있다:
**CRD**(가볍게 새 타입 선언)와 **API Aggregation**(별도 API 서버를 통째로 연결). apiserver 프로세스는
이 둘을 위해 세 서버가 위임 체인으로 묶인 구조다([README](README.md)).

```
요청 → kube-aggregator → (등록된 APIService면 외부 서버로 프록시)
                       → 아니면 KubeAPIServer(내장 그룹)
                       → 그것도 아니면 apiextensions-apiserver(CRD)
```

## CRD — CustomResourceDefinition

`apiextensions-apiserver`가 담당. 사용자는 **CRD 객체** 하나를 만들어 "새 종류"를 선언한다:
그룹/버전/Kind, 스키마(OpenAPI v3 검증), 스코프(namespaced/cluster) 등.

- 등록되면, **새 REST 엔드포인트가 동적으로 생긴다**. 그 타입의 객체는 보통 객체처럼 etcd에 저장되고
  list/watch/CRUD가 된다 — 별도 코드 없이.
- 동적 핸들러: `apiextensions-apiserver/pkg/apiserver/customresource_handler.go`. CRD가 생기거나
  바뀌면 이 핸들러가 해당 GVR의 저장/검증/서빙 파이프라인을 즉석에서 구성한다.
- 디스커버리도 동적으로 갱신된다(`customresource_discovery_controller.go`).
- **검증**: CRD에 박힌 OpenAPI 스키마(+ CEL `x-kubernetes-validations`)로 객체를 검증.
- **버전 변환**: 여러 버전을 선언하면, "none"(변환 안 함) 또는 **conversion webhook**으로 버전 간
  변환을 한다(`apiextensions-apiserver/pkg/apiserver/conversion/`).

CRD는 "데이터만 있는 새 타입"을 만든다. 그 타입을 **실제로 동작하게** 하려면 그것을 watch해서 reconcile
하는 컨트롤러가 따로 필요하다 → 그게 [60-controller-runtime](../60-controller-runtime/),
[61-kubebuilder](../61-kubebuilder.md)가 돕는 일이다.

## API Aggregation — APIService

CRD로는 부족할 때(예: 커스텀 저장소, 커스텀 비즈니스 로직, 임의 서브리소스)는 **별도 API 서버**를
띄우고 `APIService` 객체로 특정 group/version을 그 서버에 위임할 수 있다. `kube-aggregator`가 담당.

- `APIService`가 등록되면, 그 group/version으로 오는 요청을 aggregator가 **해당 백엔드 서버로 프록시**
  한다(`kube-aggregator/pkg/apiserver/handler_proxy.go`).
- 등록 상태를 추적하는 컨트롤러: `apiservice_controller.go`. 백엔드 가용성/디스커버리 병합을 관리한다
  (`handler_apis.go`, `handler_discovery.go`).
- 실제 예: `metrics.k8s.io`(metrics-server) 같은 API가 이 방식으로 제공된다.

## CRD vs Aggregation — 선택 기준

| 기준 | CRD | Aggregation |
|------|-----|-------------|
| 구현 부담 | 거의 없음(객체 하나) | 별도 API 서버 운영 |
| 저장 | etcd(공용) | 자유(자체 저장 가능) |
| 커스텀 로직 | 제한적(스키마/CEL) | 임의 |
| 대부분의 경우 | **권장** | 특수 요구 시 |

오늘날 대부분의 확장은 CRD + 컨트롤러로 충분하다. Aggregation은 metrics처럼 특별한 저장/계산이 필요한
소수 경우에 쓴다.

## 더 읽을 곳
- [60-controller-runtime](../60-controller-runtime/) · [61-kubebuilder](../61-kubebuilder.md) — CRD를 동작시키는 컨트롤러 작성
- [00-foundations/03](../00-foundations/03-api-object-model.md) — CRD도 따르는 GVK/객체 모델
