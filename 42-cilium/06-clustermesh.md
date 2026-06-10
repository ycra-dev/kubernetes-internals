# 42.06 · ClusterMesh — 멀티 클러스터

**근거**: `cilium/clustermesh-apiserver/`, `cilium/pkg/clustermesh/`

ClusterMesh는 **여러 Kubernetes 클러스터를 하나처럼** 연결하는 cilium 기능이다. 클러스터 경계를 넘어
Pod 통신, 서비스 디스커버리, 정책을 가능케 한다.

## 무엇을 푸는가

기본적으로 클러스터는 격리돼 있다 — A 클러스터의 Pod는 B 클러스터의 Service를 모른다. ClusterMesh는:

- **글로벌 서비스**: 같은 이름의 Service를 여러 클러스터에 두고, 백엔드를 클러스터 간에 공유/페일오버.
- **클러스터 간 Pod 통신**: 한 클러스터 Pod가 다른 클러스터 Pod에 직접 도달.
- **클러스터 간 정책**: NetworkPolicy를 클러스터 경계 너머로 확장([03](03-networkpolicy.md)).

## 동작 개념

각 클러스터는 자신의 상태(엔드포인트, identity, 서비스) 일부를 **clustermesh-apiserver**
(`clustermesh-apiserver/`)를 통해 노출한다. 다른 클러스터의 cilium-agent가 이를 읽어, 원격 클러스터의
엔드포인트/서비스를 자기 eBPF 맵에 반영한다(`pkg/clustermesh/`).

```
클러스터 A의 agent ──watch──► 클러스터 B의 clustermesh-apiserver
   │                            (B의 서비스/엔드포인트/identity 노출)
   ▼
A의 eBPF 맵에 B의 백엔드 추가  → A의 Pod가 B의 Pod로 라우팅 가능
```

핵심 전제: **identity가 클러스터 간에 일관**돼야 정책이 의미를 가진다. ClusterMesh는 identity를 클러스터
경계 너머로 정렬한다.

## 전제 조건

- 클러스터 간 Pod CIDR이 겹치지 않아야 한다(평평한 IP 모델을 멀티 클러스터로 확장하므로).
- 노드 간(또는 게이트웨이 간) 네트워크 도달성.

## 더 읽을 곳
- [02-agent.md](02-agent.md) — identity와 엔드포인트(멀티 클러스터로 확장됨)
- [README.md](README.md) — cilium 전체 구조
