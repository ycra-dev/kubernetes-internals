# 00.01 · 레포 구성 개요

## 이 디렉토리의 정체

이 작업 공간은 하나의 빌드 가능한 프로젝트가 아니라, **서로 독립적으로 개발/배포되는
레포지토리 11개를 나란히 펼쳐 둔 작업 공간**이다. 각 레포는 자체 `go.mod`(또는 서브모듈)를 가지며 별도로
빌드된다.

```
kubernetes/   ─ 컨트롤 플레인 + 노드 컴포넌트 본체 (가장 큼, ~1.7GB)
apimachinery/ ─┐  kubernetes/staging 에서 자동 미러됨
client-go/    ─┘  (읽기 전용, 직접 기여 대상 아님)
etcd/         ─ 분산 KV 저장소
containerd/   ─ 컨테이너 런타임
cilium/       ─ eBPF CNI (~865MB)
dns/          ─ 클러스터 DNS
controller-runtime/ ─ 오퍼레이터 프레임워크
kubebuilder/  ─ 오퍼레이터 스캐폴딩 SDK
autoscaler/   ─ 오토스케일링 컴포넌트 모음
enhancements/ ─ KEP 설계 문서 (코드 아님)
```

## Core 레포 vs 독립 레포

### Core (`kubernetes`)
`kubernetes` 레포 하나에 다음 바이너리들의 소스가 모두 들어 있다 (`kubernetes/cmd/`):

| 바이너리 | 역할 | 본 문서 |
|----------|------|---------|
| `kube-apiserver` | API 게이트웨이 + etcd 접근 | [10-apiserver](../10-apiserver/) |
| `kube-controller-manager` | 빌트인 컨트롤러 집합 | [11-controller-manager](../11-controller-manager/) |
| `kube-scheduler` | Pod→Node 배정 | [12-scheduler](../12-scheduler/) |
| `kubelet` | 노드 에이전트 | [13-kubelet](../13-kubelet/) |
| `kube-proxy` | 서비스 네트워킹 | [14-kube-proxy](../14-kube-proxy/) |
| `kubectl`, `kubeadm` | CLI/부트스트랩 | [15-cli](../15-cli/) |
| `cloud-controller-manager` | 클라우드 연동 컨트롤러 | (간략) |
| `gen*`, `*check`, `import-boss` 등 | 코드 생성/검증 내부 도구 | - |

> 나머지 `kubernetes/cmd/` 항목(`gendocs`, `genman`, `dependencycheck`, `prune-junit-xml` 등)은
> 빌드/문서/검증을 위한 개발용 도구이며 런타임 컴포넌트가 아니다.

### 독립 레포
`etcd`, `containerd`, `cilium`, `dns`는 Kubernetes와 **별개 프로젝트**지만 클러스터를 이루는 데 필수다.
이들은 표준 인터페이스를 통해 결합한다:

- `etcd` ↔ apiserver: etcd v3 gRPC API (`kubernetes/.../apiserver/pkg/storage/etcd3`)
- `containerd` ↔ kubelet: **CRI** (Container Runtime Interface, gRPC)
- `cilium`, `dns` ↔ kubelet/노드: **CNI** (Container Network Interface) 및 DNS 프로토콜

### 확장 도구
`controller-runtime`, `kubebuilder`는 **사용자가 직접 컨트롤러/CRD를 만들 때** 쓰는 SDK이고,
`autoscaler`는 클러스터 위에서 도는 별도 컨트롤러 모음이다.

## staging 미러 관계

`apimachinery`와 `client-go`는 독립 레포로 보이지만, 실제 개발은
`kubernetes/staging/src/k8s.io/{apimachinery,client-go}/` 에서 이뤄지고 거기서 자동으로 독립 레포로
배포(publish)된다. 두 레포 README 상단에 다음과 같이 명시돼 있다:

> ⚠️ This is an automatically published staged repository for Kubernetes... This repository is
> read-only for importing, and not used for direct contributions.

그래서 이 문서 모음은 **원본 경로(`kubernetes/staging/src/k8s.io/...`)** 를 우선 인용한다.
(`kubernetes/vendor/k8s.io/...` 에도 동일 코드의 vendored 사본이 있다.)

## 공통점: 전부 Go

모든 런타임 레포는 Go로 작성됐다. `kubernetes/go.work`는 staging 모듈들을 워크스페이스로 묶고,
각 레포는 `vendor/` 디렉토리에 의존성을 vendoring 한다.

## 더 읽을 곳
- [02-architecture.md](02-architecture.md) — 이 레포들이 런타임에 어떻게 맞물리는가
