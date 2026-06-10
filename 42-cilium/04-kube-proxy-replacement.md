# 42.04 · kube-proxy 대체 (eBPF 서비스 로드밸런싱)

**근거**: `cilium/pkg/loadbalancer/`, `cilium/pkg/kpr/`, `cilium/bpf/bpf_sock.c`

cilium은 [14-kube-proxy](../14-kube-proxy/)를 **완전히 대체**할 수 있다("kube-proxy replacement", KPR).
Service의 ClusterIP→백엔드 변환을 iptables 대신 **eBPF**로 한다.

## 무엇이 달라지나

kube-proxy는 Service/EndpointSlice를 watch해 iptables/IPVS 규칙을 만든다
([14-kube-proxy/02](../14-kube-proxy/02-backends.md)). cilium KPR은 같은 객체를 watch하되, 결과를
**eBPF 서비스/백엔드 맵**에 넣고 데이터패스가 그 맵으로 로드밸런싱한다(`pkg/loadbalancer/`, 설정
`pkg/kpr/`).

```
Service + EndpointSlice (watch)
   │  cilium-agent
   ▼
eBPF 서비스 맵: ClusterIP:port → [백엔드 IP들]
   │  데이터패스가 조회
   ▼
패킷 DNAT (커널 eBPF)
```

## 소켓 수준 로드밸런싱 — bpf_sock.c

가장 효율적인 부분이다. `bpf/bpf_sock.c`는 **연결을 맺는 시점(connect/sendmsg)** 에 소켓 훅에서 동작한다:

- 애플리케이션이 ClusterIP에 `connect()`하면, eBPF가 **그 자리에서 목적지 주소를 백엔드 Pod IP로 바꾼다.**
- 즉 패킷마다 DNAT하는 게 아니라, **연결 설정 시 한 번** 목적지를 백엔드로 정한다. 이후 패킷은 이미
  백엔드를 향하므로 per-packet 변환/역변환과 conntrack 부담이 준다.

iptables의 per-packet DNAT보다 가볍고, 서비스/백엔드가 많아도 맵 조회라 확장성이 좋다.

## 장점

- **확장성**: 규칙이 선형으로 늘지 않는다(맵 기반).
- **지연**: 소켓 수준 LB로 데이터패스 홉/변환이 적다.
- **kube-proxy 제거**: kube-proxy DaemonSet 자체가 불필요해져 노드 구성이 단순해진다.

## 부분 대체도 가능

전부 대체하지 않고 일부만 cilium이 맡는 모드도 있다(예: NodePort만 eBPF, 나머지는 kube-proxy). KPR
설정(`pkg/kpr/`)으로 범위를 정한다.

## NodePort / LoadBalancer / HostPort

cilium은 ClusterIP뿐 아니라 NodePort(`bpf_host.c`/`bpf_xdp.c`에서 외부 진입 처리), LoadBalancer,
HostPort, 외부 트래픽 정책(`externalTrafficPolicy`,
[14-kube-proxy/02](../14-kube-proxy/02-backends.md))도 eBPF로 구현한다.

## 더 읽을 곳
- [14-kube-proxy](../14-kube-proxy/) — 대체되는 전통 방식
- [01-ebpf-datapath.md](01-ebpf-datapath.md) — bpf_sock.c의 위치
