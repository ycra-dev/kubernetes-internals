# 14 · kube-proxy

**근거 레포**: `kubernetes` (`cmd/kube-proxy`, `pkg/proxy/`)

kube-proxy는 각 노드에서 도는 컴포넌트로, **Service 추상화를 실제 패킷 라우팅으로** 구현한다.
"`my-service:80`으로 보낸 트래픽을 살아 있는 백엔드 Pod들에 분산"하는 것이 그 일이다.

## Service라는 추상화

Pod는 죽고 다시 뜨며 IP가 바뀐다. **Service**는 그 위에 **안정적인 가상 IP(ClusterIP)** 와 이름을 얹는다.
클라이언트는 Service의 IP/이름으로만 통신하고, kube-proxy가 그것을 현재 살아 있는 Pod IP들로 변환한다.

- 어떤 Pod가 백엔드인지는 **EndpointSlice** 객체로 표현된다(Service 셀렉터에 맞는 ready Pod들).
  EndpointSlice는 controller-manager가 유지한다
  ([11-controller-manager/01](../11-controller-manager/01-controller-catalog.md)).
- kube-proxy는 Service와 EndpointSlice를 **watch**해, 변화가 생기면 노드의 패킷 규칙을 갱신한다.

## 역할

- Service/EndpointSlice를 watch → 노드의 데이터플레인 규칙(iptables/IPVS/nftables) 동기화.
- ClusterIP, NodePort, LoadBalancer, ExternalName 등 Service 타입별 라우팅.
- 세션 어피니티, 헬스 체크 응답.

> kube-proxy는 **L4(연결) 수준**에서 동작하며, 패킷 자체를 매번 프록시하지 않는다 — 커널의 규칙
> (iptables 등)을 설정해 두면 커널이 알아서 DNAT 한다. (cilium처럼 eBPF로 kube-proxy를 **대체**할 수도
> 있다 → [42-cilium/04](../42-cilium/).)

## 동작: watch → 변경 추적 → syncProxyRules

```
apiserver watch (Service, EndpointSlice)
   │
   ▼
ServiceChangeTracker / EndpointsChangeTracker  (변경 누적)
   │  (pkg/proxy/servicechangetracker.go:33, endpointschangetracker.go:33)
   ▼
syncProxyRules()  ─ 현재 원하는 규칙 전체를 계산해 커널에 반영
   (iptables/proxier.go:631, ipvs/proxier.go:658, nftables/proxier.go:1056)
```

`syncProxyRules`는 **전체 원하는 규칙 집합을 한 번에 계산해 적용**하는 level-triggered 방식이다 — 개별
변경을 추적해 부분 수정하지 않고, "지금 있어야 할 규칙"을 통째로 만든 뒤 동기화한다. 잦은 변경을 묶어
처리(throttle)해 커널 호출을 줄인다.

## 백엔드 — 데이터플레인 구현

`pkg/proxy/` 아래에 백엔드 구현이 여럿 있다(노드 OS/모드에 따라 선택):

| 백엔드 | 디렉토리 | 특징 |
|--------|----------|------|
| iptables | `iptables/` | 전통적 기본. 규칙 수가 많아지면 갱신이 느려질 수 있음 |
| IPVS | `ipvs/` | 커널 IPVS 사용. 대규모 서비스에서 더 효율적 |
| nftables | `nftables/` | iptables 후継. 성능/확장성 개선 |
| winkernel | `winkernel/` | Windows 노드 |

상세는 [02-backends.md](02-backends.md).

## 문서

| # | 문서 | 내용 |
|---|------|------|
| 01 | [endpointslice-tracking.md](01-endpointslice-tracking.md) | Service/EndpointSlice 변경 추적 |
| 02 | [backends.md](02-backends.md) | iptables/IPVS/nftables, conntrack |
| 03 | [iptables-rules.md](03-iptables-rules.md) | KUBE-* 체인 구조(코드 레벨) |
| 04 | [ipvs.md](04-ipvs.md) | IPVS 모드(가상서버/ipset/graceful) |

## 진입점
- 바이너리: `kubernetes/cmd/kube-proxy/proxy.go`
- 핵심: `kubernetes/pkg/proxy/`

## 더 읽을 곳
- [40-networking](../40-networking/) — Service/CIDR 모델 전체
- [42-cilium/04](../42-cilium/) — eBPF로 kube-proxy 대체
