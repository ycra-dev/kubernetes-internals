# 61 · kubebuilder

**근거 레포**: `kubebuilder` (`sigs.k8s.io/kubebuilder/v4`)

kubebuilder는 **오퍼레이터 프로젝트를 스캐폴딩(생성)** 하는 SDK/CLI다. [60-controller-runtime](60-controller-runtime/)이
*런타임 라이브러리*라면, kubebuilder는 그 라이브러리를 쓰는 *코드와 매니페스트를 생성*하는 도구다.
"CRD를 정의하고 컨트롤러 골격을 만들어줘"가 그 일이다.

## 무엇을 생성하나

`kubebuilder init` / `kubebuilder create api`를 실행하면:

- **API 타입 골격**: `api/v1/<kind>_types.go` — Spec/Status 구조체([00-foundations/03](00-foundations/03-api-object-model.md)).
- **컨트롤러 골격**: `internal/controller/<kind>_controller.go` — controller-runtime의 `Reconcile`
  스텁([60-controller-runtime/01](60-controller-runtime/01-manager-reconciler.md)).
- **main.go**: Manager를 세팅하고 컨트롤러/웹훅을 등록.
- **CRD/RBAC/웹훅 매니페스트**: `config/` 아래 YAML(아래 marker 기반 생성).
- 테스트 골격(envtest, [60-controller-runtime/04](60-controller-runtime/04-webhook-envtest.md)).

개발자는 생성된 `Reconcile`에 로직만 채우면 된다.

## 스캐폴딩 엔진 — machinery

`kubebuilder/pkg/machinery/`가 코드 생성 엔진이다. 템플릿(`scaffold.go`, `file.go`, `injector.go`)을
조합해 파일을 만들거나, 기존 파일의 특정 마커 지점에 코드를 주입한다(`marker.go`). 즉 단순 템플릿이 아니라
**기존 프로젝트에 점진적으로 코드를 끼워 넣는** 도구다(api 추가 시 main.go에 등록 코드 주입 등).

## 플러그인 구조

생성 방식은 **플러그인**으로 갈린다(`kubebuilder/pkg/plugin/`, `pkg/plugins/`):

- `plugins/golang/v4/` — 현재 표준 Go 오퍼레이터 레이아웃.
- `plugins/golang/deploy-image/` — 특정 이미지를 배포하는 오퍼레이터를 더 완성형으로 생성.

플러그인이 "어떤 레이아웃/관례로 스캐폴딩할지"를 결정하므로, 새 관례를 플러그인으로 추가할 수 있다.

## marker 주석 → 매니페스트 생성

생성된 Go 타입에는 **marker 주석**(`// +kubebuilder:...`)이 붙는다. 예:

```go
// +kubebuilder:validation:Minimum=1
Replicas int32
// +kubebuilder:printcolumn:name="Ready",type=string,JSONPath=`.status.ready`
```

`make manifests`(controller-gen 도구)가 이 marker를 읽어 **CRD의 OpenAPI 스키마, RBAC Role, 웹훅
설정**을 자동 생성한다. 즉 "Go 타입이 진실의 원천"이고, 매니페스트는 거기서 파생된다.

## kubebuilder ↔ controller-runtime ↔ apiserver

```
kubebuilder (생성)
   └─► controller-runtime 기반 코드 (런타임)
         └─► apiserver의 CRD/Aggregation (10-apiserver/07)에 등록된 타입을
             watch·reconcile
```

세 계층의 관계: kubebuilder가 만들고, controller-runtime이 돌리고, 그 결과 CRD가 동작한다. CRD 자체의
서빙은 apiserver(apiextensions, [10-apiserver/07](10-apiserver/07-crd-aggregation.md))가 한다.

> 참고: kubebuilder와 비슷한 도구로 **Operator SDK**가 있으며, 내부적으로 같은 controller-runtime/
> controller-gen을 공유한다.

## 더 읽을 곳
- [60-controller-runtime/](60-controller-runtime/) — 생성된 코드가 쓰는 런타임
- [10-apiserver/07](10-apiserver/07-crd-aggregation.md) — CRD가 서빙되는 곳
