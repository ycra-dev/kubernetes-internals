# 10.17 · Egress Selector와 Konnectivity

**근거**: `kubernetes/staging/src/k8s.io/apiserver/pkg/server/egressselector/`
(`EgressSelector` `:53`, `ControlPlane`/`Etcd`/`Cluster` EgressType `:62`~)

apiserver는 보통 클라이언트의 요청을 **받는다**. 하지만 어떤 동작은 apiserver가 **나가서** 다른 곳에
연결해야 한다 — `kubectl logs`/`exec`는 apiserver가 **kubelet/Pod로** 연결하고, Aggregated API
([07-crd-aggregation.md](07-crd-aggregation.md))는 외부 API 서버로 프록시한다. 네트워크가 분리된
환경(컨트롤 플레인과 워커가 다른 네트워크)에서 이 egress를 통제하는 것이 egress selector와 konnectivity다.

## 문제 — 컨트롤 플레인이 워커 네트워크에 못 닿을 수 있다

보안 격리를 위해 컨트롤 플레인(apiserver)과 워커 노드를 **다른 네트워크**에 두는 구성이 흔하다(매니지드
Kubernetes 등). 이때:

- `kubectl exec`는 apiserver → kubelet(워커 네트워크)로 연결해야 하는데, apiserver가 워커 네트워크에
  직접 못 닿을 수 있다.
- 방화벽이 "워커 → 컨트롤 플레인"만 허용하고 역방향을 막을 수도 있다.

## EgressSelector — 목적지별 egress 경로

`egressselector/egress_selector.go:53`의 **EgressSelector**가 "이 연결은 어디로 가는가"에 따라 다른
**egress 경로**를 고른다. egress 종류(`EgressType`, `:62`~):

| EgressType | 목적지 | 예 |
|------------|--------|-----|
| `ControlPlane` (`:63`) | 컨트롤 플레인 내부 | aggregated API, 웹훅(인-클러스터) |
| `Etcd` (`:65`) | etcd 저장소 | storage 계층([06-etcd-storage-layer.md](06-etcd-storage-layer.md)) |
| `Cluster` (`:66`) | 관리 대상 클러스터(워커) | kubelet logs/exec, Pod 연결 |

apiserver는 나가는 연결마다 `Lookup`으로 그 종류에 맞는 다이얼러(직접 연결 / 프록시 경유 / konnectivity
터널)를 선택한다. `--egress-selector-config-file`로 설정한다.

## Konnectivity — 역방향 터널

`Cluster` egress가 핵심 문제다(apiserver → 워커). **Konnectivity**(별도 프로젝트: konnectivity-server +
konnectivity-agent)가 이를 푼다:

```
컨트롤 플레인:  konnectivity-server (apiserver 옆)
워커 노드:      konnectivity-agent (DaemonSet)
   │
agent가 server로 **나가는** 연결을 맺어 터널을 미리 연다 (워커→컨트롤플레인 방향, 방화벽 허용)
   │
apiserver가 kubelet에 연결할 때 → konnectivity-server → (열린 터널) → agent → kubelet
```

핵심 발상: **연결을 워커가 먼저 건다(reverse tunnel)**. 그래서 방화벽이 "워커→컨트롤 플레인"만 허용해도,
그 터널을 통해 apiserver가 워커에 도달할 수 있다. apiserver는 `Cluster` egress를 konnectivity-server로
보내고, server가 적절한 agent의 터널로 중계한다.

## 무엇을 통제하나

egress selector로:
- **네트워크 격리 강제**: apiserver가 워커 네트워크에 직접 라우팅 없이도 동작.
- **감사/정책**: 모든 egress가 정해진 경로(konnectivity)를 거치게 해 통제.
- etcd/control-plane/cluster 트래픽을 **분리된 경로**로 보내 보안 도메인을 나눔
  ([17-security/03](../17-security/03-pki-certificates.md)의 CA 분리와 같은 정신).

## kubectl exec의 전체 경로 (재확인)

[15-cli/02](../15-cli/02-kubectl-internals.md)의 exec 스트림이 egress까지 포함하면:

```
kubectl exec ─► apiserver
   │  Cluster egress → (konnectivity 터널) → kubelet
   ▼
kubelet ─CRI Exec─► containerd streaming ─► 컨테이너 셸
   (13-kubelet/02, 30-containerd/01)
```

konnectivity가 그림의 "apiserver → kubelet" 구간을 네트워크 분리 환경에서도 가능케 한다.

## 더 읽을 곳
- [15-cli/02](../15-cli/02-kubectl-internals.md) — exec 스트리밍(egress 대상)
- [07-crd-aggregation.md](07-crd-aggregation.md) — aggregated API(ControlPlane egress)
- [06-etcd-storage-layer.md](06-etcd-storage-layer.md) — Etcd egress
