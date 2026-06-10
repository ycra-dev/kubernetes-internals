# 15.01 · kubeadm — 클러스터 부트스트랩

**근거**: `kubernetes/cmd/kubeadm/app/`

kubeadm은 **클러스터 자체를 처음 세우는** 도구다. kubectl이 "돌아가는 클러스터와 대화"한다면, kubeadm은
"컨트롤 플레인을 부팅"한다. 핵심 명령은 `init`(첫 컨트롤 플레인 노드)와 `join`(노드 추가)이다
(`cmd/kubeadm/app/cmd/`).

## phase 기반 설계

`kubeadm init`은 하나의 거대한 동작이 아니라 **순서 있는 phase들의 파이프라인**이다
(`cmd/kubeadm/app/cmd/phases/init/`). 각 phase는 독립 실행도 가능하다. 주요 phase(파일명):

| phase | 파일 | 하는 일 |
|-------|------|---------|
| preflight | `preflight.go` | 환경 점검(포트/권한/스왑/런타임 존재) |
| certs | `certs.go` | CA와 각 컴포넌트용 인증서 생성([13-kubelet/06](../13-kubelet/06-node-registration.md)) |
| kubeconfig | `kubeconfig.go` | admin/컨트롤러/스케줄러용 kubeconfig 생성 |
| etcd | `etcd.go` | (스택드) etcd를 static pod로 띄움([20-etcd](../20-etcd/)) |
| control-plane | `controlplane.go` | apiserver/controller-manager/scheduler를 **static pod**로 |
| kubelet | `kubelet.go` | kubelet 설정/시작 |
| bootstrap-token | `bootstraptoken.go` | 노드 join용 부트스트랩 토큰 |
| upload-config | `uploadconfig.go` | 클러스터 설정을 ConfigMap으로 보존 |
| mark-control-plane | `markcontrolplane.go` | 컨트롤 플레인 노드에 라벨/taint |
| addons | `addons.go` | CoreDNS([40-networking/03](../40-networking/)), kube-proxy 설치 |

## static pod — 닭과 달걀 문제

apiserver를 띄우려면 보통 apiserver가 필요하다(순환). kubeadm은 이를 **static pod**로 푼다: kubelet이
특정 디렉토리(`/etc/kubernetes/manifests/`)의 매니페스트를 직접 읽어, **apiserver 없이도** 컨테이너를
띄운다. 그래서 컨트롤 플레인 컴포넌트(apiserver/controller-manager/scheduler/etcd)가 static pod로
부팅되고, 그 apiserver가 떠야 나머지 클러스터가 동작한다.

> static pod는 [13-kubelet/01](../13-kubelet/01-pod-lifecycle.md)의 `configCh`가 받는 입력 중 하나다
> (apiserver watch가 아니라 로컬 파일).

## join

`kubeadm join`(`cmd/join.go`)은 새 노드를:
1. preflight 점검,
2. 부트스트랩 토큰으로 CA를 검증하고 TLS bootstrap으로 kubelet 인증서 발급
   ([13-kubelet/06](../13-kubelet/06-node-registration.md)),
3. kubelet을 시작해 컨트롤 플레인에 등록.

컨트롤 플레인 노드로 join하면 추가 컨트롤 플레인 컴포넌트도 올린다(HA).

## kubeadm은 무엇을 안 하나

CNI(네트워크 플러그인)는 설치하지 않는다 — 사용자가 cilium 등을 따로 설치해야 노드가 Ready가 된다
([42-cilium](../42-cilium/)). 클라우드 통합/로드밸런서/스토리지도 범위 밖이다. kubeadm의 목표는
"호환되는 최소 클러스터"를 표준 방식으로 세우는 것이다.

## 더 읽을 곳
- [10-apiserver](../10-apiserver/) — 부트스트랩되는 핵심 컴포넌트
- [20-etcd](../20-etcd/) — 스택드 etcd
