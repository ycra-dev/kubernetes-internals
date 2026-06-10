# 42.13 · cilium-operator

**근거**: `cilium/operator/` (`pkg/{ipam,ciliumidentity,ciliumendpointslice,bgp,gateway-api,ingress}`,
`cmd/leader_election.go`, `cmd/lifecycle.go`)

[README](README.md)에서 cilium-operator를 "클러스터 단위 작업 담당"으로 소개했다. agent가 **노드마다**
도는 데이터패스 담당이라면, operator는 **클러스터에 하나**(리더 선출) 도는 조율자다. 노드 로컬로는
못 하는 클러스터 전역 결정을 맡는다.

## agent vs operator — 역할 분담

| | cilium-agent | cilium-operator |
|--|--------------|-----------------|
| 배포 | DaemonSet(노드마다) | Deployment(클러스터 1개, 리더 선출) |
| 책임 | 노드의 eBPF 데이터패스, CNI, 엔드포인트 | 클러스터 전역 IPAM/GC/CRD 조율 |
| 근거 | [02-agent.md](02-agent.md) | 이 문서 |

operator는 **리더 선출**(`cmd/leader_election.go`,
[00-foundations/06](../00-foundations/06-leader-election.md))로 복제본 중 하나만 활성화한다 — 전역
결정이 중복되면 안 되므로.

## 주요 책임 (`operator/pkg/`)

| 컴포넌트 | 디렉토리 | 역할 |
|----------|----------|------|
| **IPAM** | `ipam/` | 노드별 Pod CIDR 할당/회수(클러스터 풀 관리). 클라우드 IPAM 모드 연동 |
| **identity GC** | `ciliumidentity/` | 안 쓰는 보안 identity 회수([12-identity-allocation.md](12-identity-allocation.md)) |
| **endpoint GC** | (ciliumendpoint) | 죽은 Pod의 CiliumEndpoint 객체 정리 |
| **CiliumEndpointSlice** | `ciliumendpointslice/` | 대규모에서 엔드포인트 정보를 슬라이스로 묶어 watch 부하 감소 |
| **BGP** | `bgp/` | BGP 제어 평면 조율([08-bgp-egress.md](08-bgp-egress.md)) |
| **Gateway API / Ingress** | `gateway-api/`, `ingress/` | L7 라우팅 리소스 조정([09-l7-envoy.md](09-l7-envoy.md), [40-networking/04](../40-networking/04-ingress-gateway.md)) |

## 왜 operator가 필요한가 — IPAM 예

IPAM이 대표적이다. 각 노드 agent가 독립적으로 Pod IP를 할당하면, **노드 간 CIDR이 겹칠** 수 있다(평평한
IP 모델 위반, [40-networking](../40-networking/)). operator가 클러스터 풀에서 **각 노드에 겹치지 않는
CIDR을 배정**해 이를 막는다:

```
operator(IPAM): 클러스터 Pod CIDR 풀 → 노드 A에 10.244.1.0/24, 노드 B에 10.244.2.0/24
   │
각 노드 agent: 배정받은 CIDR 안에서만 Pod IP 할당 (겹침 없음)
```

이는 Kubernetes 빌트인 NodeIPAM 컨트롤러([11-controller-manager/01](../11-controller-manager/01-controller-catalog.md))와
유사한 역할을, cilium IPAM 모드에서 operator가 한다.

## identity GC — 또 하나의 GC

[12-identity-allocation.md](12-identity-allocation.md)에서 "안 쓰는 identity를 회수한다"고 했다. 그
주체가 operator의 `ciliumidentity/`다. 어떤 identity를 쓰는 엔드포인트가 모두 사라지면 그 identity를
삭제해 ID 공간을 재사용한다 — etcd compaction([20-etcd/03](../20-etcd/03-mvcc.md))이나 containerd GC
([30-containerd/07](../30-containerd/07-metadata-gc.md))와 같은 "참조 없는 것 정리" 패턴.

## operator도 컨트롤러다

operator의 모든 책임은 reconcile 루프다([00-foundations/05](../00-foundations/05-controller-pattern.md)) —
Kubernetes/Cilium CRD 객체를 watch해 클러스터 상태를 원하는 대로 수렴시킨다. 일부는 controller-runtime
([60-controller-runtime](../60-controller-runtime/))을 쓴다(`operator/pkg/controller-runtime`). 즉 cilium은
"노드 데이터패스(agent) + 클러스터 컨트롤러(operator)"의 조합이다.

## 더 읽을 곳
- [02-agent.md](02-agent.md) — 노드 측 agent
- [12-identity-allocation.md](12-identity-allocation.md) — operator가 GC하는 identity
- [00-foundations/06](../00-foundations/06-leader-election.md) — operator의 리더 선출
