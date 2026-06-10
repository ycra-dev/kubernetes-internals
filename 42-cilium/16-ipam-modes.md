# 42.16 · IPAM 모드

**근거**: `cilium/pkg/ipam/` (`option/option.go`의 `IPAMKubernetes="kubernetes"` `:9`,
`IPAMCRD="crd"` `:13`, `IPAMENI="eni"` `:16`, `crd.go`, `eni.go`, `cidrset/`)

[13-operator.md](13-operator.md)에서 "operator가 노드별 Pod CIDR을 배정한다(IPAM)"고 했다. 이 문서는
cilium의 **IPAM 모드들** — Pod에 IP를 주는 방식의 선택지 — 를 본다. 환경(온프레미스/클라우드)에 따라
Pod IP가 어디서 오는지가 달라진다.

## IPAM이 푸는 문제

[40-networking](../40-networking/)의 "모든 Pod는 고유·라우팅 가능 IP" 모델을 만족하려면, 누군가
**노드 간 겹치지 않게** Pod IP를 나눠줘야 한다. 그 "누가, 어디서"가 IPAM 모드다.

## 주요 모드 (`pkg/ipam/option/option.go`)

| 모드 | 상수 | Pod IP 출처 | 환경 |
|------|------|-------------|------|
| **kubernetes** | `IPAMKubernetes` (`:9`) | Kubernetes Node의 `spec.podCIDR`(NodeIPAM 컨트롤러가 할당) | 기본/단순 |
| **cluster-pool** | (cluster-pool) | cilium operator가 클러스터 풀에서 노드별 CIDR 배정 | cilium 표준(operator 관리) |
| **CRD** | `IPAMCRD` (`:13`) | `CiliumNode` CRD에 기록된 풀 | 유연/커스텀 |
| **ENI** | `IPAMENI` (`:16`) | AWS ENI의 보조 IP(VPC IP) | AWS 네이티브 |
| **Azure / GKE** | (azure/...) | 클라우드 네이티브 IP | 해당 클라우드 |

### kubernetes 모드
Kubernetes 빌트인 NodeIPAM 컨트롤러([11-controller-manager/01](../11-controller-manager/01-controller-catalog.md))가
각 Node에 `podCIDR`을 할당하고, cilium은 그걸 따른다. 가장 단순하지만 cilium이 IP 관리를 주도하지 않는다.

### cluster-pool 모드 (cilium 표준)
cilium **operator**([13-operator.md](13-operator.md))가 클러스터 IP 풀을 관리하며 노드마다 CIDR을 배정
(`cidrset/`). cilium이 IPAM을 주도해 더 유연하다(여러 풀, 동적 확장).

### ENI 모드 (AWS)
`eni.go`. Pod에 **VPC의 실제 IP**(ENI 보조 IP)를 직접 부여한다. 그래서 Pod가 VPC 네이티브로 라우팅되어
오버레이가 불필요하고, AWS 보안 그룹/라우팅과 자연스럽게 통합된다. operator가 ENI를 관리하며 노드에
IP 풀을 붙인다.

## CiliumNode CRD — IP 풀의 표현

CRD/cluster-pool/ENI 모드에서 노드의 IP 풀 상태는 **CiliumNode** CRD(`crd.go`)에 기록된다. operator가
이를 갱신하고(풀 할당/보충), 노드 agent가 읽어 Pod에 IP를 준다 — 또 하나의 watch 기반 reconcile
([00-foundations/05](../00-foundations/05-controller-pattern.md), [02-agent.md](02-agent.md)).

```
operator: 클러스터 풀 → CiliumNode CRD에 노드별 CIDR/IP 풀 기록
   │  agent가 watch
   ▼
CNI ADD 시: 노드 풀에서 Pod IP 할당 (40-networking/01)
   └─ 풀이 떨어지면 agent가 operator에 보충 요청 → operator가 더 배정
```

## 모드 선택의 영향

| 측면 | kubernetes/cluster-pool | ENI/클라우드 네이티브 |
|------|-------------------------|------------------------|
| Pod IP | 클러스터 내부 대역(오버레이/라우팅) | VPC 실제 IP |
| 외부 통합 | 오버레이 필요할 수 있음 | 클라우드 라우팅/보안그룹 네이티브 |
| IP 소비 | 클러스터 대역만 | VPC IP 소비(한계 주의) |

오버레이 vs 네이티브 라우팅([01-ebpf-datapath.md](01-ebpf-datapath.md))의 선택과 맞물린다 — ENI 모드는
보통 네이티브 라우팅과 함께 쓴다.

## 더 읽을 곳
- [13-operator.md](13-operator.md) — IPAM을 관리하는 operator
- [40-networking/README](../40-networking/README.md) — Pod CIDR 모델
- [01-ebpf-datapath.md](01-ebpf-datapath.md) — 오버레이 vs 네이티브 라우팅
