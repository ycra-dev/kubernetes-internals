# 42.14 · 대역폭 관리(EDT)와 L2 광고

**근거**: `cilium/pkg/datapath/linux/bandwidth/`, `cilium/pkg/datapath/tables/bandwidth_qdisc.go`,
`cilium/pkg/maps/bwmap/`, `cilium/pkg/l2announcer/l2announcer.go`(리더 선출 `:25`)

cilium의 두 보조 기능: Pod별 **대역폭 제한**(EDT)과, BGP 없이 LoadBalancer IP를 알리는 **L2 광고**.
[08-bgp-egress.md](08-bgp-egress.md)와 [10-maps-conntrack.md](10-maps-conntrack.md)를 보강한다.

## 대역폭 관리 — EDT 기반

Pod에 `kubernetes.io/egress-bandwidth` 어노테이션으로 egress 대역폭 상한을 줄 수 있다. cilium은 이를
**EDT(Earliest Departure Time)** 방식으로 구현한다(`pkg/datapath/linux/bandwidth/`):

- 전통적 TBF(Token Bucket Filter) 큐잉 대신, 각 패킷에 **출발 시각(departure time)** 을 계산해 부여한다.
- eBPF가 패킷의 출발 시각을 **bwmap**([10-maps-conntrack.md](10-maps-conntrack.md))의 Pod별 속도 정보로
  계산하고, 커널 qdisc(`bandwidth_qdisc.go`, FQ/MQ)가 그 시각에 맞춰 내보낸다.
- EDT는 큐 잠금 경합이 적어 **고속·다중 코어에서 더 효율적**이다(전통 TBF 대비).

```
Pod egress 패킷 → eBPF가 bwmap의 속도로 departure time 계산 → 패킷에 타임스탬프
   → FQ qdisc가 그 시각에 송출 → 결과적으로 대역폭 상한 준수
```

즉 "언제 보낼지"를 패킷마다 정해 속도를 제어한다 — Pod별 QoS를 노드 네트워크 수준에서 시행.

## L2 Announcements — BGP 없는 LB IP

[08-bgp-egress.md](08-bgp-egress.md)에서 언급한 L2 광고를 코드로 본다(`pkg/l2announcer/`). BGP 라우터가
없는 환경에서 **LoadBalancer Service IP**를 로컬 네트워크에 알리는 방법이다(MetalLB L2 모드와 유사).

### 리더 선출로 IP 소유권 결정
핵심 문제: 같은 LB IP를 여러 노드가 동시에 ARP 응답하면 충돌한다. 그래서 **IP(서비스)마다 한 노드만**
응답해야 한다. l2announcer는 **Kubernetes 리더 선출**로 이를 정한다(`l2announcer.go:25`
`leaderelection`, `:84` 주석):

- 각 LB IP(정책에 매칭되는 서비스)마다 **Lease를 잡아**(리더 선출,
  [00-foundations/06](../00-foundations/06-leader-election.md)) 소유 노드를 정한다.
- 리더 노드만 그 IP에 대해 **ARP(IPv4)/NDP(IPv6) 응답**(gratuitous ARP 포함)을 보내, 로컬 스위치가 그
  IP→리더 노드 MAC을 학습하게 한다.
- 리더 노드가 죽으면 Lease가 다른 노드로 넘어가 IP 소유권이 이동(failover).

```
LoadBalancer Service IP 1.2.3.4
   │  l2announcer: 이 IP의 Lease를 노드 N이 획득(리더)
   ▼
노드 N만 1.2.3.4에 ARP 응답 → 스위치가 "1.2.3.4 = N의 MAC" 학습
   → 외부 트래픽이 N으로 → eBPF 서비스 LB로 백엔드 분산 (04-kube-proxy-replacement)
```

응답할 IP/MAC 정보는 `l2respondermap`([10-maps-conntrack.md](10-maps-conntrack.md))에 들어가 eBPF가
ARP에 응답한다.

## 정리

| 기능 | 메커니즘 | 맵 |
|------|----------|-----|
| 대역폭 제한 | EDT(출발 시각) + FQ qdisc | bwmap |
| L2 광고 | 리더 선출 + ARP/NDP 응답 | l2respondermap |

둘 다 cilium이 "Kubernetes 객체(어노테이션/Service) → eBPF 데이터패스"로 시행하는
([02-agent.md](02-agent.md)) 또 다른 예다.

## 더 읽을 곳
- [08-bgp-egress.md](08-bgp-egress.md) — BGP 기반 광고(대안)
- [10-maps-conntrack.md](10-maps-conntrack.md) — bwmap/l2respondermap
- [00-foundations/06](../00-foundations/06-leader-election.md) — L2 광고의 리더 선출
