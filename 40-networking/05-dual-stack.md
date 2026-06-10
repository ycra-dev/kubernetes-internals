# 40.05 · IPv4/IPv6 듀얼 스택

**근거**: `kubernetes` (Pod/Service의 IP 필드 다중화, `pkg/proxy`, `pkg/controller/nodeipam`)

Kubernetes는 **IPv4와 IPv6를 동시에** 지원할 수 있다(dual-stack). Pod와 Service가 두 종류의 IP를 함께
가질 수 있다.

## 무엇이 듀얼 스택이 되나

- **Pod**: IPv4 주소와 IPv6 주소를 둘 다 받는다(`Pod.status.podIPs`가 복수 IP).
- **Service**: `ipFamilies`로 IPv4/IPv6 중 하나 또는 둘 다, `ipFamilyPolicy`로
  SingleStack/PreferDualStack/RequireDualStack을 지정. ClusterIP가 패밀리별로 하나씩.
- **노드**: 두 패밀리의 Pod CIDR을 할당받는다(NodeIPAM 컨트롤러,
  [11-controller-manager/01](../11-controller-manager/01-controller-catalog.md)).

## 주소 공간이 패밀리마다

[README](README.md)의 Pod/Service CIDR이 **패밀리마다 따로** 존재한다:

```
Pod CIDR:     IPv4 10.244.0.0/16  +  IPv6 fd00::/48
Service CIDR: IPv4 10.96.0.0/12   +  IPv6 fd00:svc::/108
```

NodeIPAM이 각 노드에 두 패밀리의 Pod CIDR을 분할하고, CNI(cilium)가 Pod에 두 IP를 부여한다
([01-cni.md](01-cni.md), [42-cilium](../42-cilium/)).

## 데이터플레인의 처리

kube-proxy/cilium은 **각 패밀리별로 규칙을 따로** 만든다 — IPv4 Service IP는 IPv4 백엔드로, IPv6는 IPv6
백엔드로([14-kube-proxy/02](../14-kube-proxy/02-backends.md), [42-cilium/04](../42-cilium/04-kube-proxy-replacement.md)).
한 Service의 두 ClusterIP가 같은 백엔드 Pod들(의 각 패밀리 IP)로 향한다.

## DNS

클러스터 DNS는 Service의 패밀리에 따라 A(IPv4)와 AAAA(IPv6) 레코드를 반환한다
([03-dns.md](03-dns.md)). 클라이언트는 자기 스택에 맞는 레코드를 골라 쓴다.

## 왜 필요한가

- IPv4 주소 고갈 대응(대규모 클러스터에서 Pod IP 부족).
- IPv6 네이티브 환경/외부 시스템과의 연동.
- 점진적 마이그레이션(둘 다 지원하다 한쪽으로).

## 더 읽을 곳
- [README.md](README.md) — Pod/Service CIDR 모델
- [02-service-discovery.md](02-service-discovery.md) — Service IP 패밀리
