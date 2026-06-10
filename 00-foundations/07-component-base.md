# 00.07 · component-base — 공용 컴포넌트 기반

**근거**: `kubernetes/staging/src/k8s.io/component-base/`

apiserver·scheduler·controller-manager·kubelet은 서로 다른 일을 하지만, **부팅·설정·로깅·메트릭·기능
플래그**는 똑같은 방식으로 처리해야 한다(일관성). 그 공통 골격을 모아 둔 라이브러리가 **component-base**다.
모든 핵심 바이너리가 이것을 임포트한다.

## 무엇이 들어 있나

`component-base/`의 주요 패키지:

| 패키지 | 역할 |
|--------|------|
| `cli/` | 컴포넌트 공통 CLI 부팅(플래그 파싱, 시그널 처리, graceful 종료) |
| `featuregate/` | **feature gate** — alpha/beta/GA 기능 토글 (아래) |
| `logs/` | 구조화 로깅(klog/v2 기반, `logs.go:33`) |
| `metrics/` | Prometheus 메트릭 등록/노출 → [18-observability/01](../18-observability/01-metrics-monitoring.md) |
| `config/`, `codec/` | 컴포넌트 설정 객체와 직렬화 |
| `configz/`, `zpages/` | 런타임 설정/진단 노출 엔드포인트 |
| `tracing/` | 분산 추적(OpenTelemetry) |
| `version/`, `compatibility/` | 버전 정보와 호환성 정책 → [90-enhancements](../90-enhancements.md) |
| `term/` | 터미널 유틸 |

## feature gate — 기능의 생애주기 토글

`featuregate/feature_gate.go`. 새 기능은 **feature gate**로 켜고 끈다. 게이트는 성숙 단계(prerelease)를
가진다(`feature_gate.go:133`~):

| 단계 | 기본 | 의미 |
|------|------|------|
| `Alpha` (`:135`) | off | 실험적, opt-in |
| `Beta` | on/off | 안정화 중 |
| `GA` (`:110`) | on(제거 예정) | 정식 |
| `Deprecated` / `PreAlpha` | - | 폐기/미출시 |

코드 곳곳의 `utilfeature.DefaultFeatureGate.Enabled(features.SomeFeature)` 분기가 이것이다(예:
[13-kubelet/01](../13-kubelet/01-pod-lifecycle.md)의 EventedPLEG). 게이트는 `--feature-gates` 플래그로
운영자가 조정하고, 단계 전환은 KEP의 졸업 기준을 따른다([90-enhancements](../90-enhancements.md)).

> `feature_gate.go:104`의 주석은 `--min-compatibility-version`에 따라 같은 기능이 다르게 기본값을
> 갖는 **버전 호환성**까지 다룬다(`compatibility/`). 컴포넌트 간 버전 스큐를 안전하게 하는 장치다.

## 왜 별도 라이브러리인가

만약 각 컴포넌트가 플래그/로깅/메트릭/게이트를 제각각 구현하면, 운영자는 컴포넌트마다 다른 방식을
배워야 하고 일관성이 깨진다. component-base는 이를 한 곳에 모아:

- 모든 컴포넌트가 **같은 플래그 관례**(`--feature-gates`, `--v`(로그 레벨) 등)를 갖고,
- 같은 형식의 `/metrics`를 노출하며([18-observability](../18-observability/)),
- 같은 방식으로 기능을 단계적으로 출시한다.

## staging 위치

component-base는 [00-foundations/01](01-overview.md)의 staging 미러 중 하나다 — 원본은
`kubernetes/staging/src/k8s.io/component-base/`이고 독립 모듈로 배포된다. 사용자 오퍼레이터도 같은 feature
gate/로깅 관례를 쓰고 싶으면 이것을 임포트할 수 있다.

## 더 읽을 곳
- [99-appendix/staging-map.md](../99-appendix/staging-map.md) — component-base를 포함한 staging 전체 지도
- [18-observability/01](../18-observability/01-metrics-monitoring.md) — metrics 패키지 활용
- [90-enhancements](../90-enhancements.md) — feature gate ↔ KEP 단계
