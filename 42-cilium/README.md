# 42 · cilium

**근거 레포**: `cilium` (`github.com/cilium/cilium`, 버전 1.20-dev)

cilium은 **eBPF 기반 CNI**다. [40-networking/01](../40-networking/01-cni.md)의 CNI 규약을 구현해 Pod에
네트워크를 붙이고, 나아가 **NetworkPolicy 시행**, **kube-proxy 대체**, **관측성(Hubble)** 까지 제공한다.
전통적 방식(iptables)과 달리, 패킷 처리를 **커널 eBPF 프로그램**으로 한다.

## 왜 eBPF인가

iptables 기반 네트워킹([14-kube-proxy](../14-kube-proxy/))은 서비스/정책이 많아지면 규칙이 선형으로 늘어
느려진다. **eBPF**는 커널에 작은 프로그램을 안전하게 주입해, 패킷이 지나는 지점(소켓, 네트워크 인터페이스,
XDP)에서 직접 라우팅/정책/로드밸런싱을 수행한다 — 해시맵 기반이라 규모에 덜 민감하고, 컨텍스트 스위치가
적다.

## 구성요소

```
┌─────────────────────────────────────────────────────────────┐
│ cilium-agent (각 노드, DaemonSet)   daemon/                   │
│   - CNI 요청 처리, 엔드포인트 관리                            │
│   - 정책/서비스를 eBPF 맵에 컴파일·로드                       │
│   └─► eBPF 프로그램 (bpf/)  ── 커널에서 패킷 처리             │
├─────────────────────────────────────────────────────────────┤
│ cilium-operator (클러스터 1, operator/)  ── IPAM, GC, CRD 등 │
│ Hubble (hubble/, hubble-relay/)  ── 흐름 관측/디버깅         │
│ clustermesh-apiserver  ── 멀티 클러스터 연결                 │
└─────────────────────────────────────────────────────────────┘
```

- **cilium-agent**(`daemon/`): 노드마다 도는 핵심. CNI를 처리하고, Kubernetes 객체(Pod/Service/정책)를
  watch해 eBPF 맵/프로그램으로 변환·로드한다.
- **eBPF 데이터패스**(`bpf/`): 실제 패킷을 처리하는 C 프로그램들(`bpf_lxc.c`, `bpf_host.c`,
  `bpf_sock.c`, `bpf_overlay.c`, `bpf_xdp.c`).
- **cilium-operator**(`operator/`): 클러스터 단위 작업(IPAM 조율, 가비지 컬렉션 등).
- **Hubble**(`hubble/`, `hubble-relay/`): eBPF가 만든 흐름 이벤트를 모아 관측성을 제공.

## 문서

| # | 문서 | 내용 |
|---|------|------|
| 01 | [ebpf-datapath.md](01-ebpf-datapath.md) | bpf/ 프로그램들, 패킷 경로 |
| 02 | [agent.md](02-agent.md) | cilium-agent: 엔드포인트/identity/맵 |
| 03 | [networkpolicy.md](03-networkpolicy.md) | identity 기반 정책 시행 |
| 04 | [kube-proxy-replacement.md](04-kube-proxy-replacement.md) | eBPF 서비스 로드밸런싱 |
| 05 | [hubble.md](05-hubble.md) | 흐름 관측성 |
| 06 | [clustermesh.md](06-clustermesh.md) | 멀티 클러스터 |
| 07 | [encryption.md](07-encryption.md) | WireGuard/IPsec 전송 암호화 |
| 08 | [bgp-egress.md](08-bgp-egress.md) | BGP, L2 광고, Egress Gateway |
| 09 | [l7-envoy.md](09-l7-envoy.md) | L7 정책/관측(Envoy), Gateway 구현 |
| 10 | [maps-conntrack.md](10-maps-conntrack.md) | eBPF 맵 카탈로그, conntrack(ctmap) |
| 11 | [endpoint-regeneration.md](11-endpoint-regeneration.md) | 엔드포인트 재생성(BPF 컴파일/로드) |
| 12 | [identity-allocation.md](12-identity-allocation.md) | identity 할당(reserved/동적, kvstore/CRD) |
| 13 | [operator.md](13-operator.md) | cilium-operator(IPAM/identity GC/BGP/Gateway) |
| 14 | [bandwidth-l2.md](14-bandwidth-l2.md) | 대역폭(EDT), L2 광고(리더 선출 ARP) |
| 15 | [packet-path.md](15-packet-path.md) | bpf_lxc.c 패킷 경로(tail call/CT) |
| 16 | [ipam-modes.md](16-ipam-modes.md) | IPAM 모드(kubernetes/cluster-pool/ENI) |

## 진입점
- 에이전트: `cilium/daemon/main.go`, `cilium/daemon/cmd/`
- 데이터패스: `cilium/bpf/`
- 핵심 로직: `cilium/pkg/{endpoint,policy,identity,datapath,maps,loadbalancer}/`

## 더 읽을 곳
- [40-networking/01](../40-networking/01-cni.md) — CNI 규약(cilium이 구현)
- [14-kube-proxy](../14-kube-proxy/) — cilium이 대체할 수 있는 전통 방식
