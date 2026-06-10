# 95.01 · 코드 생성

**근거**: `kubernetes/staging/src/k8s.io/code-generator/cmd/`, `kubernetes/hack/update-codegen.sh`

Kubernetes API의 보일러플레이트는 `code-generator`의 생성기들이 만든다. 각 생성기는 **타입 정의 + marker
주석**(`// +k8s:...`, `// +genclient` 등)을 읽어 코드를 뽑는다. `hack/update-codegen.sh`가 이들을
일괄 실행한다.

## 생성기 목록 (`code-generator/cmd/`)

| 생성기 | 만드는 것 | 왜 |
|--------|-----------|-----|
| **deepcopy-gen** | `DeepCopyObject()` 구현(`zz_generated.deepcopy.go`) | 모든 runtime.Object가 깊은 복사 필요([00-foundations/03](../00-foundations/03-api-object-model.md)) |
| **client-gen** | typed clientset(`Pods().Get()` 등) | 타입 안전한 API 클라이언트([00-foundations/04](../00-foundations/04-list-watch-informer.md)) |
| **informer-gen** | typed Informer | list/watch 캐시([00-foundations/04](../00-foundations/04-list-watch-informer.md)) |
| **lister-gen** | typed Lister | 캐시 읽기 |
| **conversion-gen** | 버전 간 변환 함수 | `v1beta1`↔`v1`([91-versioning-skew.md](../91-versioning-skew.md)) |
| **defaulter-gen** | 기본값 채우기 함수 | 어드미션/registry의 defaulting |
| **go-to-protobuf** | protobuf 직렬화 코드(`generated.pb.go`) | 내부 통신용 protobuf([00-foundations/03](../00-foundations/03-api-object-model.md)) |
| **register-gen** | Scheme 등록 코드 | GVK↔타입 등록 |
| **applyconfiguration-gen** | Server-Side Apply용 빌더 | SSA([10-apiserver/08](../10-apiserver/08-server-side-apply.md)) |
| **prerelease-lifecycle-gen** | API 버전 생애주기 메타 | deprecation([91-versioning-skew.md](../91-versioning-skew.md)) |
| **validation-gen** | 선언적 검증 코드 | 타입 검증 |

## marker 주석으로 제어

생성은 코드 안의 **marker 주석**으로 제어된다. 예:

```go
// +genclient                    ← client-gen: 이 타입의 클라이언트 생성
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
type MyResource struct { ... }   ← deepcopy-gen: DeepCopyObject 생성
```

[61-kubebuilder](../61-kubebuilder.md)의 `// +kubebuilder:...` marker도 같은 발상이다 — 주석이 생성을
지시한다.

## 생성된 코드는 커밋된다

생성 결과(`zz_generated.*`, `generated.pb.go`)는 빌드 때마다 만드는 게 아니라 **레포에 커밋**돼 있다.
`hack/verify-codegen.sh`가 CI에서 "생성 결과가 최신인가"(타입을 바꿨는데 재생성을 안 했나)를 검증한다
([02-build-test.md](02-build-test.md)).

## 이게 왜 중요한가 (코드 읽을 때)

코드베이스를 탐색하다 `zz_generated.deepcopy.go`(수천 줄의 DeepCopy)나 `generated.pb.go`(거대한
protobuf 코드)를 보면 — **읽지 않아도 된다.** 기계 생성물이다. 의미 있는 로직은 타입 정의(`types.go`)와
손으로 짠 파일에 있다. 이를 알면 1.7GB 코드베이스에서 "읽을 가치 있는 부분"을 빠르게 가린다.

## 더 읽을 곳
- [00-foundations/03](../00-foundations/03-api-object-model.md) — 생성 코드가 구현하는 규약
- [02-build-test.md](02-build-test.md) — 생성 결과 검증(CI)
