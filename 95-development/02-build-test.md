# 95.02 · 빌드와 테스트

**근거**: `kubernetes/build/`, `kubernetes/hack/`, `kubernetes/test/`(`e2e`, `integration`, `conformance`)

Kubernetes가 어떻게 빌드·검증되는지를 본다. 이 모음의 각 레포는 독립 모듈이라 빌드 방식이 다르지만
([99-appendix/reading-guide](../99-appendix/reading-guide.md)), 여기선 `kubernetes` 코어 중심으로 다룬다.

## 빌드

- 진입점: 루트 `Makefile`. `make`로 모든 바이너리(`cmd/`의 kube-apiserver 등)를 빌드.
- `build/`: 컨테이너 이미지 빌드, 릴리스 아티팩트 생성. 재현 가능한 빌드를 위해 빌드 컨테이너
  (`build/build-image/`)에서 컴파일.
- 의존성: `vendor/`에 vendoring([00-foundations/01](../00-foundations/01-overview.md)), `go.work`로
  staging 모듈을 워크스페이스로 묶음.

## hack/ — 개발 자동화

`hack/`은 코드 생성·검증·빌드 스크립트 모음이다:

- `hack/update-codegen.sh`: 코드 생성 실행([01-code-generation.md](01-code-generation.md)).
- `hack/verify-*.sh`: CI 검증. 예: `verify-codegen.sh`(생성 결과 최신?), `verify-boilerplate.sh`(라이선스
  헤더), `verify-api-groups.sh`. `verify-all.sh`가 전부 실행.
- `hack/make-rules/`: Makefile이 호출하는 실제 빌드/테스트 규칙.

## 테스트 — 세 층위

`test/` 디렉토리는 충실도(fidelity)가 다른 세 종류를 담는다:

| 종류 | 위치 | 무엇 | 충실도/비용 |
|------|------|------|-------------|
| **단위(unit)** | 각 패키지 `*_test.go` | 함수/타입 단위 | 빠름/낮음 |
| **통합(integration)** | `test/integration/` | 실제 apiserver+etcd 띄우고 컴포넌트 상호작용 검증 | 중간 |
| **e2e** | `test/e2e/`, `test/e2e_node/` | 진짜 클러스터에서 전체 시나리오 | 느림/높음 |

- **통합 테스트**는 [60-controller-runtime/04](../60-controller-runtime/04-webhook-envtest.md)의 envtest와
  같은 발상 — 가짜 목이 아니라 **실제 apiserver/etcd 바이너리**를 띄워 진짜 API 동작(어드미션/검증/watch)을
  상대로 테스트한다.
- **e2e**는 Ginkgo 기반으로 실제 클러스터에 워크로드를 띄워 동작을 검증. `test/e2e_node/`는 노드 단위
  (kubelet) e2e, `test/e2e_dra/`는 DRA([19-dra.md](../19-dra.md)) 등 영역별로 나뉜다.

## Conformance — "진짜 Kubernetes인가"

`test/conformance/`. **conformance 테스트**는 "이 클러스터가 표준 Kubernetes 동작을 만족하는가"를
검증하는 e2e의 부분집합이다. 벤더(EKS/GKE/AKS 등)가 이를 통과해야 "Certified Kubernetes" 인증을 받는다.
이로써 어느 배포판이든 같은 API 동작을 보장한다([91-versioning-skew.md](../91-versioning-skew.md)의 호환성
정신과 연결).

## CI 게이트

PR은 머지 전 위 검증들(verify-* + 단위/통합/일부 e2e)을 통과해야 한다. 특히 **API 변경**은 생성 코드
재생성, 호환성, conformance 영향까지 검사받는다 — 거대한 분산 시스템의 API를 깨지 않고 진화시키기 위한
안전망이다.

## 더 읽을 곳
- [01-code-generation.md](01-code-generation.md) — CI가 검증하는 생성 코드
- [60-controller-runtime/04](../60-controller-runtime/04-webhook-envtest.md) — envtest(통합 테스트와 같은 발상)
- [99-appendix/reading-guide](../99-appendix/reading-guide.md) — 레포별 빌드 진입점
