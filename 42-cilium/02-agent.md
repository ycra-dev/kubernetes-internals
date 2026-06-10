# 42.02 · cilium-agent — 제어 평면

**근거**: `cilium/daemon/`, `cilium/pkg/{endpoint,endpointmanager,identity,datapath,maps}/`

cilium-agent는 각 노드에서 DaemonSet으로 도는 핵심 컴포넌트다. Kubernetes 객체를 watch해
([00-foundations/04](../00-foundations/04-list-watch-informer.md)), 그것을 [01](01-ebpf-datapath.md)의
eBPF 맵/프로그램으로 컴파일·로드한다. "선언된 네트워크 상태 → 커널 데이터패스"를 수렴시키는 reconcile
루프의 cilium 버전이다.

## 엔드포인트 — Pod의 cilium 측 표현

cilium에서 각 Pod(정확히는 네트워크 네임스페이스)는 **엔드포인트**다(`pkg/endpoint/`,
관리 `pkg/endpointmanager/`). CNI ADD([40-networking/01](../40-networking/01-cni.md))가 오면 agent는:

1. 엔드포인트를 생성하고 IP를 할당.
2. veth를 만들고 `bpf_lxc.c`를 그 veth에 로드.
3. 엔드포인트별 설정(헤더)을 생성해 BPF를 컴파일.

엔드포인트는 재시작 후에도 복원된다(`daemon/cmd/endpoint_restore.go`) — agent가 재시작해도 Pod 네트워크가
유지되도록.

## identity — 정책의 단위

핵심 개념: cilium은 정책을 **IP가 아니라 identity**로 표현한다(`pkg/identity/`). 같은 라벨 집합을 가진
Pod들은 **하나의 보안 identity**(숫자 ID)를 공유한다.

- 왜? Pod IP는 수시로 바뀌지만 라벨(=identity)은 안정적이다. IP 기반 정책은 Pod가 재생성될 때마다
  갱신해야 하지만, identity 기반은 라벨이 같으면 그대로다.
- agent는 "라벨 → identity" 매핑을 클러스터 전체에서 합의(KV 저장: cilium 자체 CRD 또는 외부 KV)하고,
  엔드포인트의 identity를 eBPF 맵에 반영한다.

정책 시행은 "identity A가 identity B로 가는 것이 허용되나"를 eBPF 정책 맵에서 조회하는 것이다 →
[03](03-networkpolicy.md).

## datapath 추상화

`pkg/datapath/`는 "원하는 네트워크 상태"를 실제 커널 구성(BPF 로드, 라우팅, 맵 쓰기)으로 옮기는 계층이다
(`pkg/datapath/linux/`가 리눅스 구현). agent의 상위 로직은 datapath 인터페이스에 "이렇게 되어야 한다"고
선언하고, datapath가 그것을 커널에 시행한다.

## IPAM

Pod IP 풀 관리(IPAM)는 모드에 따라 agent와 **cilium-operator**(`operator/`)가 나눠 맡는다. operator는
클러스터 단위로 노드별 CIDR 할당/회수, 안 쓰는 identity GC 등을 처리한다.

## 무엇을 watch하나

agent는 Kubernetes의 Pod, Service, EndpointSlice, NetworkPolicy, Namespace, Node와 cilium 자체 CRD
(CiliumNetworkPolicy, CiliumEndpoint 등)를 watch한다(`pkg/k8s/`). 변경이 오면 해당 eBPF 맵/프로그램을
갱신한다 — 다시 말해 cilium도 결국 apiserver를 watch하는 컨트롤러다.

## 더 읽을 곳
- [01-ebpf-datapath.md](01-ebpf-datapath.md) — agent가 채우는 데이터패스
- [03-networkpolicy.md](03-networkpolicy.md) — identity 기반 정책
