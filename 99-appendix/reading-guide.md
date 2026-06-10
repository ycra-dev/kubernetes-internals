# 99 · 목적별 읽기 가이드

목적에 따라 어디서부터 코드를 읽으면 되는지, 그리고 각 레포의 빌드/실행 진입점을 모았다.

## 처음 읽는다면

순서대로:
1. [00-foundations/02-architecture.md](../00-foundations/02-architecture.md) — 큰 그림
2. [00-foundations/03~05](../00-foundations/) — API 객체 / list-watch / reconcile (필수 어휘)
3. [80-scenarios/01-pod-create.md](../80-scenarios/01-pod-create.md) — 모든 것이 한 사건으로

## "이게 어떻게 동작하나" — 주제별 시작점

| 알고 싶은 것 | 시작 문서 | 코드 진입점 |
|--------------|-----------|-------------|
| API 요청 처리 | [10-apiserver/01](../10-apiserver/01-request-pipeline.md) | `kubernetes/staging/src/k8s.io/apiserver/pkg/server/config.go` (DefaultBuildHandlerChain) |
| 컨트롤러 작성 패턴 | [00-foundations/05](../00-foundations/05-controller-pattern.md) | `kubernetes/pkg/controller/replicaset/replica_set.go` |
| 스케줄링 | [12-scheduler/01](../12-scheduler/01-framework.md) | `kubernetes/pkg/scheduler/schedule_one.go` |
| 파드가 뜨는 법 | [13-kubelet/01](../13-kubelet/01-pod-lifecycle.md) | `kubernetes/pkg/kubelet/kubelet.go` (syncLoop) |
| 컨테이너 실행 | [30-containerd/03](../30-containerd/03-runtime-shim-oci.md) | `containerd/internal/cri/server/`, `core/runtime/v2/` |
| 서비스 라우팅 | [14-kube-proxy/02](../14-kube-proxy/02-backends.md) | `kubernetes/pkg/proxy/iptables/proxier.go` (syncProxyRules) |
| 네트워킹/eBPF | [42-cilium/01](../42-cilium/01-ebpf-datapath.md) | `cilium/daemon/`, `cilium/bpf/` |
| 저장(etcd) | [20-etcd/06](../20-etcd/06-server-flow.md) | `etcd/server/etcdserver/v3_server.go` |
| 인증/인가 | [10-apiserver/02](../10-apiserver/02-authentication.md) | `kubernetes/plugin/pkg/auth/authorizer/` |
| CRD/오퍼레이터 | [60-controller-runtime](../60-controller-runtime/) | `controller-runtime/pkg/{manager,reconcile,builder}/` |
| 스토리지/CSI | [50-storage](../50-storage/) | `kubernetes/pkg/controller/volume/`, `pkg/kubelet/volumemanager/` |
| 오토스케일링 | [70-autoscaling](../70-autoscaling/) | `autoscaler/cluster-autoscaler/core/static_autoscaler.go` |

## "왜 이렇게 설계했나"

코드의 의도/대안/위험은 KEP에 있다 → [90-enhancements.md](../90-enhancements.md). feature gate 이름으로
`enhancements/keps/sig-*/`를 검색한다.

## 레포별 빌드/실행 진입점

| 레포 | 빌드 | 주요 바이너리 진입점 |
|------|------|----------------------|
| kubernetes | `make` (루트) | `cmd/kube-apiserver/`, `cmd/kube-scheduler/`, `cmd/kube-controller-manager/`, `cmd/kubelet/`, `cmd/kube-proxy/`, `cmd/kubectl/`, `cmd/kubeadm/` |
| etcd | `make build` | `etcd/server/main.go`, `etcd/etcdctl/`, `etcd/etcdutl/` |
| containerd | `make` (`BUILDING.md` 참고) | `containerd/cmd/containerd/`, `cmd/containerd-shim-runc-v2/`, `cmd/ctr/` |
| cilium | `make` (`Makefile`) | `cilium/daemon/main.go`, `cilium/operator/`, `cilium/cilium-dbg/` |
| dns | `make` | `dns/cmd/kube-dns/`, `dns/cmd/node-cache/` |
| controller-runtime | (라이브러리) | `controller-runtime/pkg/` (임포트해서 사용) |
| kubebuilder | `make build` | `kubebuilder/cmd/` |
| autoscaler | 서브모듈별 `make` | `autoscaler/cluster-autoscaler/`, `autoscaler/vertical-pod-autoscaler/` |
| enhancements | (문서) | `enhancements/keps/` |

> 빌드 전 각 레포의 README/BUILDING/CONTRIBUTING과 Go 버전 요구사항을 확인할 것. 이 작업 공간의 레포들은
> 각자 독립 모듈이며 함께 빌드되지 않는다([00-foundations/01](../00-foundations/01-overview.md)).

## 실제로 띄워 보려면

- **로컬 단일 노드**: `kind`(kubernetes-in-docker) 또는 `minikube`로 클러스터를 띄우고
  `kubectl`로 관찰. CNI/CSI가 포함돼 있어 [80-scenarios](../80-scenarios/)를 실습할 수 있다.
- **컴포넌트 디버깅**: 노드에서 `crictl`(CRI), `ctr -n k8s.io`(containerd, [30-containerd/04](../30-containerd/04-ctr-client.md)),
  `cilium status`/`hubble observe`(cilium), `etcdctl endpoint status`(etcd,
  [20-etcd/08](../20-etcd/08-operations.md))로 각 계층을 직접 들여다본다.

## 더 읽을 곳
- [glossary.md](glossary.md) — 용어집
- [../README.md](../README.md) — 전체 인덱스
