# 00.03 · API 객체 모델

**근거 레포**: `apimachinery` (원본: `kubernetes/staging/src/k8s.io/apimachinery`)

Kubernetes의 모든 것은 **API 객체**다 — Pod, Service, Deployment, Node, 심지어 사용자가 정의한 CRD까지.
이 객체들이 어떻게 식별·표현·변환·직렬화되는지가 `apimachinery`가 정의하는 타입 시스템이다.

## 객체의 좌표: GVK와 GVR

모든 객체 타입은 세 좌표 **Group / Version / Kind (GVK)** 로 식별된다.

- 정의: `apimachinery/pkg/runtime/schema/group_version.go`

```go
// group_version.go:128 부근
type GroupVersionKind struct {
    Group   string
    Version string
    Kind    string
}
```

- **Group**: API 그룹. 예) `apps`, `batch`, `networking.k8s.io`. 코어 그룹은 빈 문자열("")이다(Pod, Service 등).
- **Version**: `v1`, `v1beta1` 등. 같은 Kind라도 버전이 여럿일 수 있다.
- **Kind**: 타입 이름. `Pod`, `Deployment` 등 CamelCase.

예: `apps/v1` 그룹·버전의 `Deployment` Kind → `apps/v1, Kind=Deployment`.

REST URL에서는 Kind 대신 **Resource**(복수형 소문자)를 쓴다. 이것이 **GroupVersionResource (GVR)** 다
(`group_version.go:96`):

```go
type GroupVersionResource struct {
    Group    string
    Version  string
    Resource string  // 예: "deployments", "pods"
}
```

| 개념 | 예시 | 쓰이는 곳 |
|------|------|-----------|
| GVK | `apps/v1, Kind=Deployment` | 코드 내부 타입 식별 |
| GVR | `apps/v1, Resource=deployments` | REST 경로 `/apis/apps/v1/namespaces/x/deployments` |

GVK ↔ GVR 변환은 **RESTMapper**가 담당한다(아래).

## 객체 공통 구조: TypeMeta + ObjectMeta + Spec/Status

거의 모든 객체는 동일한 골격을 가진다. 정의는 `apimachinery/pkg/apis/meta/v1/types.go`.

```go
// 개념적 구조
type SomeResource struct {
    metav1.TypeMeta   // Kind, APIVersion          (types.go:42)
    metav1.ObjectMeta // Name, Namespace, UID, ...  (types.go: ObjectMeta)
    Spec   SomeSpec   // 원하는 상태 (사용자가 작성)
    Status SomeStatus // 현재 상태 (시스템이 채움)
}
```

- **`TypeMeta`** (`types.go:42`): `Kind`, `APIVersion`. 객체가 자기 타입을 자기 안에 들고 있다.
- **`ObjectMeta`**: 모든 객체의 공통 메타데이터. 핵심 필드(`types.go`):
  - `Name`, `Namespace` — 식별. (Name은 네임스페이스 내에서 유일)
  - `UID` — 시공간 유일 식별자. 같은 이름을 지우고 다시 만들어도 UID는 다르다.
  - `ResourceVersion` — 낙관적 동시성 제어용 버전. 변경마다 바뀐다 → [04](04-list-watch-informer.md), [10-apiserver/06](../10-apiserver/) 참조.
  - `Generation` — spec이 바뀔 때 증가(상태 변경과 구분).
  - `Labels` / `Annotations` — 셀렉터용 라벨 / 임의 메타데이터.
  - `OwnerReferences` — 소유 관계 (GC의 근거). → [05](05-controller-pattern.md)
  - `Finalizers` / `DeletionTimestamp` — 삭제 훅과 삭제 예약. → [05](05-controller-pattern.md)
- **`Spec`/`Status` 분리**: spec은 *원하는 상태*(사용자 작성), status는 *현재 상태*(컨트롤러가 보고).
  이 분리가 선언형 모델의 핵심이다.

### runtime.Object 인터페이스
모든 API 타입이 만족해야 하는 최소 인터페이스 (`apimachinery/pkg/runtime/interfaces.go:337`):

```go
type Object interface {
    GetObjectKind() schema.ObjectKind  // 자신의 GVK 접근
    DeepCopyObject() Object             // 깊은 복사 (캐시 안전성)
}
```

`DeepCopyObject`이 인터페이스에 박혀 있는 이유: Informer 캐시의 객체를 컨트롤러가 직접 수정하면 캐시가
오염되므로, 항상 깊은 복사본을 받아 다룬다.

## Scheme — 타입 레지스트리

**Scheme**(`apimachinery/pkg/runtime/scheme.go`)은 "GVK ↔ Go 타입" 매핑과 변환 함수들의 중앙 레지스트리다.
역할:

1. **타입 등록**: 어떤 GVK가 어떤 Go struct인지 안다 → 역직렬화 시 올바른 타입 생성.
2. **버전 변환(conversion)**: 같은 객체의 `v1beta1` ↔ `v1` 표현 간 변환 함수를 보유
   (`pkg/runtime/converter.go`). apiserver가 저장/응답 시 버전을 바꿀 때 사용.
3. **기본값 채우기(defaulting)**.

각 API 그룹은 `install`/`register.go`에서 자기 타입을 Scheme에 등록한다.

## RESTMapper — GVK ↔ GVR ↔ REST 경로

**RESTMapper**(`apimachinery/pkg/api/meta/`)는 GVK를 GVR로, 그리고 REST 스코프(네임스페이스 여부)로
매핑한다. `kubectl`이 `deployment`라는 짧은 이름을 받아 `apps/v1/deployments` 경로를 찾아갈 때 이걸 쓴다.

## 직렬화 — JSON / YAML / Protobuf

객체는 와이어로 나갈 때 직렬화된다. apimachinery는 여러 코덱을 제공한다(`pkg/runtime/serializer/`):

- **JSON** — 사람이 읽는 기본, `kubectl`/YAML과 호환.
- **Protobuf** — apiserver ↔ 컴포넌트 간 내부 통신의 기본(더 작고 빠름). 위 `types.go` 필드의
  `protobuf:"..."` 태그가 그 근거다.
- **YAML** — JSON 위의 표현.

직렬화 형식과 무관하게 내부 표현(in-memory Go struct)은 하나이며, 코덱이 그 사이를 오간다.

## 왜 이게 모든 것의 기반인가

- 컨트롤러는 이 객체들을 list/watch 하고([04](04-list-watch-informer.md)),
- 그 spec과 status의 차이를 좁히며([05](05-controller-pattern.md)),
- apiserver는 이 타입 시스템으로 검증·변환·저장한다([10-apiserver](../10-apiserver/)).

즉 GVK/Scheme/직렬화는 클러스터 안에서 오가는 "공용 언어"다.

## 더 읽을 곳
- [04-list-watch-informer.md](04-list-watch-informer.md) — 이 객체들을 캐싱하는 법
- [10-apiserver/05-registry-storage.md](../10-apiserver/) — 객체가 저장되는 법
- [10-apiserver/07-crd-aggregation.md](../10-apiserver/) — 사용자 정의 객체(CRD)
