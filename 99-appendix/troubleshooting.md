# 99 · 트러블슈팅 — 증상에서 코드까지

이 문서는 흔한 장애 증상을 출발점으로, **그 증상이 이 문서 모음의 어느 메커니즘에서 비롯되는지**를
역추적한다. 디버깅은 결국 "어느 단계에서 멈췄나"를 좁히는 일이고, 각 단계는 앞 문서들이 설명한 그
컴포넌트다.

## Pod가 안 뜬다 — 단계별 추적

Pod 생성 흐름([80-scenarios/01](../80-scenarios/01-pod-create.md))의 어디서 막혔는지로 좁힌다.
`kubectl describe pod`의 **Events**([18-observability/02](../18-observability/02-events-logging.md))가 1차
단서다.

| 증상(STATUS) | 막힌 단계 | 원인/볼 곳 |
|--------------|-----------|------------|
| `Pending`, "FailedScheduling" | 스케줄러 | 자원 부족/affinity/taint → [12-scheduler](../12-scheduler/). 노드 추가 필요면 CA([70-autoscaling](../70-autoscaling/)) |
| `Pending`, 이벤트 없음 | 스케줄러 큐/어드미션 | 쿼터 초과([45-policy/01](../45-policy/01-quota-limitrange.md)), 어드미션 거부([10-apiserver/04](../10-apiserver/04-admission.md)) |
| `ContainerCreating` 지속 | kubelet/CRI/CNI | 볼륨 mount 실패([13-kubelet/05](../13-kubelet/05-volume-manager.md)), CNI IP 할당 실패([42-cilium](../42-cilium/)), 이미지 pull |
| `ImagePullBackOff` | 런타임 | 레지스트리 인증/이미지 없음/서명 검증([30-containerd/06](../30-containerd/06-image-verify-transfer.md)) |
| `CrashLoopBackOff` | 앱/probe | 컨테이너가 죽음, liveness 실패([13-kubelet/07](../13-kubelet/07-probes.md)) — `kubectl logs`로 |
| `CreateContainerConfigError` | kubelet | 없는 ConfigMap/Secret 참조([17-security/01](../17-security/01-serviceaccount-secrets.md)) |

핵심: `Pending`이면 **컨트롤 플레인**(스케줄/어드미션), `ContainerCreating`/`CrashLoop`이면 **노드**
(kubelet/런타임/앱)다. 이 경계가 추적을 반으로 줄인다.

## 서비스에 연결이 안 된다

요청 경로([80-scenarios/02](../80-scenarios/02-service-request.md))를 거꾸로 짚는다:

| 확인 | 볼 곳 |
|------|-------|
| DNS가 이름을 푸나? (`nslookup svc`) | [40-networking/03](../40-networking/03-dns.md) — CoreDNS Pod 상태 |
| EndpointSlice에 백엔드가 있나? | [40-networking/06](../40-networking/06-endpointslice-controller.md) — ready Pod가 있나(readiness, [13-kubelet/07](../13-kubelet/07-probes.md)) |
| 데이터플레인 규칙이 있나? | kube-proxy([14-kube-proxy/03](../14-kube-proxy/03-iptables-rules.md)) / cilium([42-cilium/04](../42-cilium/04-kube-proxy-replacement.md)) |
| NetworkPolicy가 막나? | [42-cilium/03](../42-cilium/03-networkpolicy.md) — Hubble로 드롭 확인([42-cilium/05](../42-cilium/05-hubble.md)) |

"엔드포인트가 비었다"면 보통 **readiness probe 실패**다 — Pod는 Running인데 NotReady라 트래픽에서 빠진
것([13-kubelet/07](../13-kubelet/07-probes.md), [40-networking/02](../40-networking/02-service-discovery.md)).

## 클러스터 전체가 느리다/멈췄다

컨트롤 플레인의 병목은 대개 **apiserver ↔ etcd**다:

| 증상 | 원인 | 볼 곳 |
|------|------|-------|
| API 요청이 느림/`429` | APF 과부하, 폭주 클라이언트 | [10-apiserver/10](../10-apiserver/10-priority-fairness.md) — APF 메트릭 |
| 모든 쓰기가 멈춤 | etcd `NOSPACE` alarm | [20-etcd/04](../20-etcd/04-backend-bbolt.md) — compact+defrag([20-etcd/08](../20-etcd/08-operations.md)) |
| watch가 자꾸 끊김/relist 폭증 | etcd 느림/compaction | [10-apiserver/12](../10-apiserver/12-watch-cache-internals.md), [20-etcd/11](../20-etcd/11-mvcc-index.md) |
| 컨트롤러가 반응 안 함 | 리더 상실/informer 미동기 | [00-foundations/06](../00-foundations/06-leader-election.md), [00-foundations/04](../00-foundations/04-list-watch-informer.md) |

etcd 건강(`etcdctl endpoint status`: 리더 안정성, DB 크기)이 컨트롤 플레인 건강의 근본이다
([20-etcd/08](../20-etcd/08-operations.md)).

## 노드가 NotReady

| 확인 | 볼 곳 |
|------|-------|
| kubelet 하트비트(Lease)가 오나? | [13-kubelet/06](../13-kubelet/06-node-registration.md) — kubelet 동작/네트워크 |
| 자원 압박으로 Pod 축출 중? | [13-kubelet/04](../13-kubelet/04-eviction.md) — disk/memory 신호 |
| NotReady가 지속되면 | NodeLifecycle이 taint→eviction([11-controller-manager/05](../11-controller-manager/05-node-lifecycle.md)) → Pod 재배치 |

## 세 가지 관측 도구 (재확인)

[18-observability](../18-observability/)의 세 축이 모든 추적의 도구다:

```
1. 메트릭  → 어디가 이상한지 범위 좁히기 (apiserver 지연, workqueue 적체, etcd)
2. 이벤트  → kubectl describe → 무슨 사건이 있었나 (Failed/Evicted/Scheduled)
3. 로그    → kubectl logs / 컴포넌트 로그(--v) → 구체적 원인
```

계층별 직접 점검:
- CRI: `crictl ps/logs` / containerd: `ctr -n k8s.io ...`([30-containerd/04](../30-containerd/04-ctr-client.md))
- 네트워크: `cilium status`, `hubble observe`([42-cilium/05](../42-cilium/05-hubble.md))
- 저장: `etcdctl endpoint status/health`([20-etcd/08](../20-etcd/08-operations.md))

## 일반 원칙 — 선언형 디버깅

Kubernetes는 선언형([00-foundations/02](../00-foundations/02-architecture.md))이라, 디버깅도 **"원하는
상태 vs 현재 상태"의 차이가 왜 안 좁혀지나**를 묻는 것이다:

```
원하는 상태(spec)는 무엇인가?  →  현재 상태(status)는 무엇인가?  →  그 차이를 좁혀야 할
컨트롤러/컴포넌트가 왜 못 하고 있나? (그 컴포넌트의 이벤트/로그/메트릭)
```

이 질문이 항상 "어느 컴포넌트의 어느 단계"로 데려간다 — 그게 이 문서 모음의 한 챕터다.

## 더 읽을 곳
- [reading-guide.md](reading-guide.md) — 목적별 코드 진입점
- [80-scenarios](../80-scenarios/) — 정상 흐름(고장 추적의 기준)
- [18-observability](../18-observability/) — 관측 도구
