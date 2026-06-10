# 40 · 네트워킹 모델

**근거 레포**: `kubernetes`(`pkg/proxy`, `pkg/kubelet/network`), `dns`, (+ CNI는 [42-cilium](../42-cilium/))

Kubernetes 네트워킹은 몇 가지 단순한 약속 위에 서 있다. 이 폴더는 그 모델과 두 축(서비스 디스커버리,
DNS)을 다루고, 데이터플레인 구현은 [14-kube-proxy](../14-kube-proxy/)와 [42-cilium](../42-cilium/)으로
이어진다.

## 네트워크 모델의 세 약속

Kubernetes는 네트워크 구현을 강제하지 않는 대신, CNI가 반드시 만족해야 할 **모델**을 정한다:

1. **모든 Pod는 고유 IP를 가진다.** 노드 경계와 무관하게 클러스터 전체에서 라우팅 가능.
2. **Pod끼리는 NAT 없이 직접 통신**한다(같은 노드든 다른 노드든).
3. **노드의 에이전트(kubelet 등)도 Pod와 통신**할 수 있다.

이 "평평한(flat) IP" 모델 덕분에, 애플리케이션은 컨테이너가 어느 노드에 있는지 신경 쓰지 않아도 된다.
이 모델을 실제로 구현하는 것이 **CNI 플러그인**(cilium 등)이다.

## 주소 공간

| 대역 | 의미 | 누가 할당 |
|------|------|-----------|
| **Pod CIDR** | Pod에 줄 IP 풀 | 노드별로 NodeIPAM 컨트롤러가 분할 ([11-controller-manager](../11-controller-manager/01-controller-catalog.md)) |
| **Service CIDR** | ClusterIP에 줄 가상 IP 풀 | apiserver/ServiceCIDRs 컨트롤러 |
| **노드 IP** | 노드 자체 IP | 인프라 |

Pod IP는 진짜 인터페이스에 붙는 라우팅 가능한 주소지만, **Service의 ClusterIP는 가상**이다 — 어떤
인터페이스에도 없고, kube-proxy/cilium이 만든 규칙이 그 IP로 온 패킷을 백엔드 Pod로 DNAT한다
([14-kube-proxy/02](../14-kube-proxy/02-backends.md)).

## 두 축

```
이름으로 찾기                          IP로 도달하기
   │                                      │
   ▼                                      ▼
DNS (이 폴더 03)                      Service → kube-proxy / cilium
"my-svc.ns.svc.cluster.local"         ClusterIP → 백엔드 Pod IP들로 분산
   → ClusterIP                         (EndpointSlice 기반)
```

1. **서비스 디스커버리**: "이름 → 안정적 가상 IP" → [02-service-discovery.md](02-service-discovery.md).
2. **DNS**: 그 이름을 IP로 해석 → [03-dns.md](03-dns.md).

그리고 그 ClusterIP로 온 트래픽을 실제 Pod로 보내는 데이터플레인이
[14-kube-proxy](../14-kube-proxy/)와 [42-cilium](../42-cilium/)이다.

## 문서

| # | 문서 | 내용 |
|---|------|------|
| 01 | [cni.md](01-cni.md) | CNI 규약, kubelet↔CNI 경계 |
| 02 | [service-discovery.md](02-service-discovery.md) | Service 타입, EndpointSlice, Ingress/Gateway |
| 03 | [dns.md](03-dns.md) | 클러스터 DNS(CoreDNS/kube-dns), NodeLocal DNSCache |
| 04 | [ingress-gateway.md](04-ingress-gateway.md) | Ingress, Gateway API(L7 라우팅) |
| 05 | [dual-stack.md](05-dual-stack.md) | IPv4/IPv6 듀얼 스택 |
| 06 | [endpointslice-controller.md](06-endpointslice-controller.md) | EndpointSlice 생산자(분할/배치) |
| 07 | [dns-internals.md](07-dns-internals.md) | kube-dns 내부(treecache), NodeLocal DNSCache |
| 08 | [service-ip-allocation.md](08-service-ip-allocation.md) | ClusterIP/NodePort 할당, ServiceCIDR API |

## 더 읽을 곳
- [14-kube-proxy](../14-kube-proxy/) — Service 데이터플레인
- [42-cilium](../42-cilium/) — CNI 구현체 + 정책 + proxy 대체
