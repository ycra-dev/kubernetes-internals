# 10.11 · Discovery와 OpenAPI

**근거**: `kubernetes/staging/src/k8s.io/apiserver/pkg/endpoints/discovery/`,
`.../endpoints/openapi/`

클라이언트(kubectl 등)는 클러스터가 **어떤 API를 제공하는지**를 어떻게 알까? 하드코딩하지 않는다 —
apiserver가 **discovery**와 **OpenAPI** 엔드포인트로 자기 능력을 알려준다. CRD/Aggregation으로 API가
동적으로 늘어도([07-crd-aggregation.md](07-crd-aggregation.md)) 클라이언트가 따라갈 수 있는 비결이다.

## Discovery — "어떤 그룹/버전/리소스가 있나"

apiserver는 discovery 엔드포인트(`endpoints/discovery/`)로 API 목록을 노출한다:

- `/api`, `/apis` — 사용 가능한 **API 그룹과 버전** 목록(`root.go`, `group.go`).
- `/apis/<group>/<version>` — 그 버전의 **리소스 목록**(이름, Kind, namespaced 여부, 지원 verb, 짧은
  이름).

`kubectl`은 이를 받아 **RESTMapper**를 구성한다([00-foundations/03](../00-foundations/03-api-object-model.md)) —
`kubectl get deploy`의 `deploy`를 `apps/v1/deployments`로 해석하는 근거가 이 discovery다.

### Aggregated discovery
리소스가 많으면 그룹마다 따로 조회하는 게 느리다. **aggregated discovery**(`discovery/aggregated/`)는
전체 디스커버리를 **한 번의 요청**으로 묶어 제공해 kubectl 시작을 빠르게 한다.

## OpenAPI — "각 타입의 스키마는 무엇인가"

discovery가 "무엇이 있나"라면, **OpenAPI**(`endpoints/openapi/`)는 "각 타입의 **필드 구조/검증 규칙**"을
제공한다:

- `/openapi/v2`, `/openapi/v3` — 모든 타입의 OpenAPI 스키마.
- `kubectl explain pod.spec.containers`가 이 스키마를 읽어 필드를 설명한다.
- **client-side 검증**, **kubectl apply의 필드 인식**, **CRD 스키마 노출**의 근거.

CRD의 OpenAPI 스키마([07-crd-aggregation.md](07-crd-aggregation.md))도 여기 합쳐져, 사용자 정의 타입도
`kubectl explain`이 된다.

## 동적 확장과의 연결

CRD가 등록되거나 APIService가 추가되면, discovery/OpenAPI가 **자동 갱신**된다
([07-crd-aggregation.md](07-crd-aggregation.md)의 동적 핸들러). 그래서 새 CRD를 만들면 즉시
`kubectl get <newkind>`가 동작한다 — kubectl이 갱신된 discovery를 받아 RESTMapper를 다시 만들기 때문.

## storageversionhash

discovery 응답에는 리소스의 **storage version hash**(`discovery/storageversionhash.go`)도 포함될 수 있다 —
저장 버전이 바뀌었는지 클라이언트가 감지하는 단서로, 버전 마이그레이션([91-versioning-skew.md](../91-versioning-skew.md))과
연관된다.

## 더 읽을 곳
- [00-foundations/03](../00-foundations/03-api-object-model.md) — discovery로 만드는 RESTMapper
- [07-crd-aggregation.md](07-crd-aggregation.md) — 동적 API 확장
- [15-cli](../15-cli/) — discovery/OpenAPI를 소비하는 kubectl
