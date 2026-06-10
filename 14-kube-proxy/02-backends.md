# 14.02 · 데이터플레인 백엔드 (iptables / IPVS / nftables)

**근거**: `kubernetes/pkg/proxy/{iptables,ipvs,nftables}/proxier.go`, `pkg/proxy/conntrack/`

추적된 Service/EndpointSlice([01](01-endpointslice-tracking.md))는 `syncProxyRules`에서 **커널 규칙**으로
변환된다. 백엔드마다 메커니즘이 다르지만 목표는 같다: *ClusterIP로 온 패킷을 살아 있는 백엔드 Pod IP로
DNAT 분산.*

## 공통 진입점: syncProxyRules

각 백엔드의 `Proxier.syncProxyRules`(iptables `proxier.go:631`, ipvs `:658`, nftables `:1056`)가
**"지금 있어야 할 규칙 전체"를 계산해 커널에 반영**한다. 부분 갱신이 아니라 전체 재계산(level-triggered)
이라 멱등적이고, 어떤 상태에서 시작해도 올바른 규칙으로 수렴한다. 잦은 변경은 최소 간격으로 묶어 반영한다.

## iptables (`iptables/`)

- Service마다 가상 체인을 만들고, **DNAT 규칙**으로 ClusterIP:port → 백엔드 중 하나로 변환.
- 백엔드 분산은 통계적 확률(`statistic` 모듈)로 균등 분배.
- 단점: 서비스/엔드포인트가 수천이면 규칙이 선형으로 늘어 `syncProxyRules` 시간과 첫 패킷 지연이 커진다.
- 그래도 가장 검증되고 호환성 높은 기본값.

## IPVS (`ipvs/`)

- 커널의 **IPVS(IP Virtual Server)** 를 사용. 서비스를 해시 테이블 기반 가상 서버로 등록.
- 규칙 조회가 O(1)에 가까워 **대규모 서비스에서 iptables보다 빠르다**.
- 다양한 분산 알고리즘(rr, lc, dh 등) 지원.
- 일부 부수 규칙(예: 패킷 마킹)에 여전히 iptables를 함께 쓴다.

## nftables (`nftables/`)

- iptables의 후계인 **nftables**를 사용. 맵/셋 자료구조로 규칙 수에 덜 민감.
- iptables 대비 갱신/조회 확장성 개선. 점진적으로 기본 권장 방향.

## conntrack — 연결 추적 정리

`pkg/proxy/conntrack/`는 커널의 **conntrack(연결 추적)** 테이블을 관리한다. 백엔드가 사라지거나 바뀌면,
그 백엔드로 향하던 기존 conntrack 항목을 정리해야 한다 — 안 그러면 죽은 Pod로 가는 연결이 남는다.
특히 UDP처럼 상태 없는 프로토콜에서 중요하다.

## Service 타입별 처리

| 타입 | 처리 |
|------|------|
| ClusterIP | 클러스터 내부 가상 IP로 DNAT |
| NodePort | 모든 노드의 특정 포트로 들어온 트래픽도 백엔드로 |
| LoadBalancer | 클라우드 LB → NodePort/ClusterIP 경로 (LB 자체는 cloud-controller-manager) |
| ExternalName | DNS CNAME (kube-proxy 규칙 없음, DNS가 처리 → [40-networking/03](../40-networking/)) |

## externalTrafficPolicy / internalTrafficPolicy

- `externalTrafficPolicy: Local` — 외부 트래픽을 **그 노드의 로컬 백엔드로만** 보내 클라이언트 소스 IP를
  보존하고 추가 홉을 없앤다.
- `Cluster`(기본) — 어느 노드로 들어와도 전체 백엔드로 분산.

## 더 읽을 곳
- [42-cilium/04](../42-cilium/) — eBPF로 이 데이터플레인을 대체하는 방식
- [40-networking/02-service-discovery.md](../40-networking/) — Service 모델
