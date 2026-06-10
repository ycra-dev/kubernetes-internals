# 99 · staging 라이브러리 지도

**근거**: `kubernetes/staging/src/k8s.io/` (33개 모듈)

`kubernetes` 레포의 `staging/src/k8s.io/`에는 **독립 모듈로 배포되는 라이브러리 33개**가 있다. 모두
여기서 개발되고 자동으로 별도 레포(`k8s.io/<name>`)로 미러된다([00-foundations/01](../00-foundations/01-overview.md)).
이 모음의 `apimachinery/`·`client-go/`가 바로 그렇게 나온 것이다. 이 문서는 33개의 지도를 제공한다.

## API / 타입 시스템

| 모듈 | 역할 | 문서 |
|------|------|------|
| `api` | 빌트인 API 타입 정의(Pod/Service/Deployment...) | [00-foundations/03](../00-foundations/03-api-object-model.md) |
| `apimachinery` | GVK/Scheme/직렬화/메타 타입 | [00-foundations/03](../00-foundations/03-api-object-model.md) |
| `apiextensions-apiserver` | CRD 서버 | [10-apiserver/07](../10-apiserver/07-crd-aggregation.md) |
| `kube-aggregator` | API Aggregation 서버 | [10-apiserver/07](../10-apiserver/07-crd-aggregation.md) |
| `code-generator` | deepcopy/client 등 코드 생성 도구 | - |

## 서버 (apiserver)

| 모듈 | 역할 | 문서 |
|------|------|------|
| `apiserver` | 제네릭 API 서버(파이프라인/저장/어드미션) | [10-apiserver](../10-apiserver/) |
| `pod-security-admission` | Pod Security Admission | [17-security/02](../17-security/02-pod-security.md) |
| `sample-apiserver` | Aggregation 예제 | [10-apiserver/07](../10-apiserver/07-crd-aggregation.md) |
| `externaljwt`, `kms` | 외부 JWT 서명, 비밀 암호화 KMS | [17-security/01](../17-security/01-serviceaccount-secrets.md) |

## 클라이언트 / 컨트롤러 작성

| 모듈 | 역할 | 문서 |
|------|------|------|
| `client-go` | Go 클라이언트(informer/lister/workqueue) | [00-foundations/04](../00-foundations/04-list-watch-informer.md) |
| `component-base` | 공용 기반(featuregate/logs/metrics/cli) | [00-foundations/07](../00-foundations/07-component-base.md) |
| `component-helpers` | 컴포넌트 공용 헬퍼 | - |
| `controller-manager` | 컨트롤러 매니저 공용 프레임워크 | [11-controller-manager](../11-controller-manager/) |
| `sample-controller` | 컨트롤러 작성 예제 | [60-controller-runtime](../60-controller-runtime/) |

## CLI

| 모듈 | 역할 | 문서 |
|------|------|------|
| `kubectl` | kubectl 로직 | [15-cli](../15-cli/) |
| `cli-runtime` | kubectl 공용(Builder/Visitor/printers) | [15-cli](../15-cli/) |
| `sample-cli-plugin` | kubectl 플러그인 예제 | - |
| `cluster-bootstrap` | 부트스트랩 토큰 | [15-cli/01](../15-cli/01-kubeadm.md) |

## 컴포넌트별 API/설정 (타입 전용)

`kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `kube-proxy`, `kubelet` 모듈은 각
컴포넌트의 **설정 API 타입**(`KubeSchedulerConfiguration` 등)과 일부 공개 인터페이스를 담는다(컴포넌트
본체 코드는 `kubernetes/pkg/`와 `cmd/`에 있음).

| 모듈 | 문서 |
|------|------|
| `kube-scheduler` | 스케줄러 프레임워크 인터페이스 → [12-scheduler/01](../12-scheduler/01-framework.md) |
| `kubelet` | kubelet 설정/API | [13-kubelet](../13-kubelet/) |
| `kube-proxy` | proxy 설정 | [14-kube-proxy](../14-kube-proxy/) |
| `kube-controller-manager` | CM 설정 | [11-controller-manager](../11-controller-manager/) |

## 노드 / 런타임 / 스토리지 / 네트워크

| 모듈 | 역할 | 문서 |
|------|------|------|
| `cri-api` | CRI gRPC 인터페이스 정의 | [13-kubelet/02](../13-kubelet/02-cri.md) |
| `cri-client`, `cri-streaming` | CRI 클라이언트, exec/attach 스트리밍 | [13-kubelet/02](../13-kubelet/02-cri.md) |
| `mount-utils` | 볼륨 마운트 유틸 | [13-kubelet/05](../13-kubelet/05-volume-manager.md) |
| `dynamic-resource-allocation` | DRA(장치 할당) 공용 | [12-scheduler/02](../12-scheduler/02-plugins.md) |
| `csi-translation-lib` | in-tree 볼륨 → CSI 변환 | [50-storage](../50-storage/) |
| `endpointslice` | EndpointSlice 유틸 | [14-kube-proxy/01](../14-kube-proxy/01-endpointslice-tracking.md) |
| `cloud-provider` | 클라우드 추상화 + CCM 컨트롤러 | [16-cloud-controller-manager](../16-cloud-controller-manager.md) |

## 관측성

| 모듈 | 역할 | 문서 |
|------|------|------|
| `metrics` | 리소스 메트릭 API(`metrics.k8s.io`) | [18-observability/01](../18-observability/01-metrics-monitoring.md) |

## 왜 staging인가

이 라이브러리들은 Kubernetes 코어와 **함께 개발/버전 관리**되어야 정합성이 유지된다(예: client-go가
apimachinery의 타입을 정확히 알아야 함). 그래서 한 레포(`kubernetes`)의 staging에서 개발하고, 외부
사용자를 위해 독립 모듈로 publish한다. 외부 코드는 `k8s.io/client-go` 등으로 임포트하지만, 진실의
원천은 `kubernetes/staging/`이다.

## 더 읽을 곳
- [00-foundations/01](../00-foundations/01-overview.md) — staging 미러 개념
- [reading-guide.md](reading-guide.md) — 목적별 진입점
