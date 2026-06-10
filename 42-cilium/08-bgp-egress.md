# 42.08 · BGP와 Egress Gateway

**근거**: `cilium/pkg/bgpv1/`, `cilium/pkg/egressgateway/`, `cilium/pkg/l2announcer`

클러스터 네트워크를 **외부 네트워크와 통합**하는 두 기능: BGP(라우팅 광고)와 Egress Gateway(외부로
나가는 트래픽의 소스 IP 제어).

## BGP — 외부 라우터에 경로 광고

`cilium/pkg/bgpv1/`. cilium-agent가 **BGP 스피커**가 되어, 클러스터의 경로(Pod CIDR, LoadBalancer
Service IP 등)를 데이터센터의 물리 라우터에 광고한다:

- 외부 라우터가 "이 Pod CIDR / 이 LB IP는 이 노드로 보내라"를 BGP로 학습.
- **오버레이 없이** Pod 네트워크를 물리 네트워크에 통합(네이티브 라우팅,
  [01-ebpf-datapath.md](01-ebpf-datapath.md)).
- **LoadBalancer Service**를 클라우드 LB 없이 온프레미스에서 구현(서비스 IP를 BGP로 광고).

이로써 베어메탈/온프레미스 환경에서 클라우드 같은 네트워킹을 얻는다.

## L2 Announcements — BGP 없는 LB IP 광고

`l2announcer`. BGP 라우터가 없는 소규모/단순 환경에서는 **L2(ARP/NDP) 광고**로 LoadBalancer IP를
로컬 네트워크에 알린다(MetalLB의 L2 모드와 유사). 같은 L2 세그먼트에서 LB IP를 특정 노드가 응답하게 한다.

## Egress Gateway — 나가는 트래픽의 소스 IP 고정

`cilium/pkg/egressgateway/`. 문제: Pod가 클러스터 **밖**(레거시 시스템, 방화벽 뒤 DB)으로 통신할 때,
소스 IP가 Pod IP(또는 노드 IP)라 변동적이라 외부 방화벽 규칙을 걸기 어렵다.

**Egress Gateway**는 선택된 Pod들의 egress 트래픽을 **지정된 게이트웨이 노드를 통해** 내보내, **고정된
소스 IP**(게이트웨이의 egress IP)로 SNAT한다:

```
Pod (app=payments) ─► egress gateway 노드 ─SNAT─► 고정 IP 1.2.3.4 ─► 외부 DB
                                                   (외부 방화벽이 1.2.3.4만 허용)
```

이로써 외부 시스템은 "이 고정 IP에서 오는 트래픽만 허용"하는 안정적 방화벽 규칙을 걸 수 있다.

## 공통점 — 외부 통합

세 기능 모두 "클러스터의 동적 네트워크"와 "외부의 정적 네트워크/정책"을 잇는다:
- BGP/L2: 외부가 클러스터로 **들어오는** 경로.
- Egress Gateway: 클러스터가 외부로 **나가는** 소스 IP.

## 더 읽을 곳
- [01-ebpf-datapath.md](01-ebpf-datapath.md) — 네이티브 라우팅
- [04-kube-proxy-replacement.md](04-kube-proxy-replacement.md) — LoadBalancer Service 처리
