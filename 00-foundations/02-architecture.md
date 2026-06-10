# 00.02 · 아키텍처 — 큰 그림

## 3계층

Kubernetes 클러스터는 크게 세 묶음으로 나뉜다.

1. **컨트롤 플레인 (control plane)** — 클러스터의 두뇌. 보통 마스터 노드에서 돈다.
   - `kube-apiserver` — 모든 요청의 관문, 유일한 etcd 접근자
   - `etcd` — 모든 상태가 저장되는 단일 진실 공급원
   - `kube-controller-manager` — 빌트인 컨트롤러들의 reconcile 루프
   - `kube-scheduler` — 아직 노드가 안 정해진 Pod에 노드를 배정
2. **노드 (node)** — 실제 워크로드가 도는 워커. 각 노드마다:
   - `kubelet` — 그 노드의 Pod 라이프사이클을 책임지는 에이전트
   - `kube-proxy` (또는 cilium) — 서비스 추상화를 노드 네트워크에 구현
   - `containerd` — 컨테이너를 실제로 띄우는 런타임
3. **네트워킹/애드온**
   - `cilium` — Pod 간 연결과 정책을 eBPF로 구현하는 CNI
   - `dns` — 서비스 이름 → IP 디스커버리

## 핵심 규칙 두 가지

이 두 규칙이 전체 아키텍처를 결정한다.

1. **오직 apiserver만 etcd에 접근한다.** scheduler도, controller-manager도, kubelet도 etcd를 직접
   읽거나 쓰지 않는다. 전부 apiserver의 REST API를 통한다. 따라서 인증·인가·검증·버전 변환이 한 곳에
   집중된다.
2. **컴포넌트 간 직접 호출이 거의 없다.** scheduler는 kubelet을 호출하지 않는다. 대신
   *apiserver를 통해 객체를 바꾸고*, 상대 컴포넌트가 그 변경을 watch 해서 반응한다. 즉 모든 협력은
   **etcd에 저장된 객체 상태를 매개로** 일어난다(이른바 "허브-앤-스포크"가 아니라 "공유 상태" 모델).

## 요청 흐름 — `kubectl apply -f pod.yaml` 한 줄의 여정

```
[1] kubectl ──HTTP POST /api/v1/namespaces/default/pods──► kube-apiserver
                                                              │
[2] 인증(Authentication): 누구인가?  ─────────────────────────┤
[3] 인가(Authorization):  권한이 있나? (RBAC/Node)  ──────────┤
[4] 어드미션(Admission):  변형/검증 (mutating→validating, webhook, CEL)
[5] 스킴 검증 + 버전 변환 → 내부 표현                          │
[6] storage 계층 ──► etcd 에 Pod 객체 기록 (spec 채워짐, nodeName 비어있음)
                                                              │
        ┌─────────────────── etcd watch 알림 ─────────────────┘
        │
[7] kube-scheduler 가 "nodeName 없는 Pod" 를 watch 로 받음
        → 필터링/스코어링으로 노드 선택
        → apiserver 에 Binding 생성 (= Pod.spec.nodeName 설정)
        │
[8] 해당 노드의 kubelet 이 "내 노드에 배정된 Pod" 를 watch 로 받음
        → CRI(gRPC) 로 containerd 에 컨테이너 생성 요청
        → CNI(cilium) 로 Pod 네트워크 연결
        → Pod.status 를 Running 으로 apiserver 에 보고
        │
[9] kube-proxy/cilium 이 Service/EndpointSlice 변경을 watch 로 받아
        노드의 패킷 라우팅을 갱신
```

각 단계의 상세는 다음 문서로 이어진다:

| 단계 | 담당 | 코드 진입점 | 문서 |
|------|------|-------------|------|
| [2]~[6] | apiserver | `kubernetes/cmd/kube-apiserver`, `.../apiserver/pkg/endpoints` | [10-apiserver](../10-apiserver/) |
| [6] 저장 | apiserver→etcd | `.../apiserver/pkg/storage/etcd3` | [10-apiserver/06](../10-apiserver/) · [20-etcd](../20-etcd/) |
| [7] | scheduler | `kubernetes/pkg/scheduler` | [12-scheduler](../12-scheduler/) |
| [8] | kubelet | `kubernetes/pkg/kubelet/kubelet.go` | [13-kubelet](../13-kubelet/) |
| [8] 컨테이너 | containerd | `containerd/cmd/containerd` | [30-containerd](../30-containerd/) |
| [8] 네트워크 | cilium | `cilium/daemon` | [42-cilium](../42-cilium/) |
| [9] | kube-proxy | `kubernetes/pkg/proxy` | [14-kube-proxy](../14-kube-proxy/) |

전 과정의 한 단계씩 코드 추적은 [80-scenarios/01-pod-create.md](../80-scenarios/)에 있다.

## 데이터 흐름 — 상태는 어떻게 동기화되나

요청 흐름이 "한 번의 쓰기"라면, 데이터 흐름은 "지속적 동기화"다. 핵심 메커니즘은 **list/watch + 로컬 캐시
+ reconcile 루프**다.

```
        etcd (권위 있는 상태)
          ▲          │
   쓰기   │          │ watch 스트림 (변경 통지)
          │          ▼
     kube-apiserver  ────────────────────────┐
          ▲                                   │
          │ REST(list/watch)                  │ list/watch
          │                                   ▼
  ┌───────┴────────┐                  각 컨트롤러/컴포넌트 안의
  │  컨트롤러 A     │                  client-go Informer
  │  - Informer 캐시 (로컬, 읽기 빠름)
  │  - Workqueue   │   변경 → 큐에 키 적재 → Reconcile(key) 호출
  │  - Reconcile() │   "원하는 상태 != 현재 상태" 면 apiserver 에 쓰기
  └────────────────┘
```

- **Informer**: apiserver를 한 번 list 한 뒤 watch로 증분 갱신하며, 객체들을 로컬 캐시(Indexer)에
  보관한다. 컨트롤러는 etcd/apiserver를 매번 때리지 않고 로컬 캐시를 읽는다.
- **Workqueue**: 변경된 객체의 키를 큐에 넣어 중복 제거·재시도·rate limiting을 처리한다.
- **Reconcile**: 큐에서 키를 꺼내 "그 객체가 원하는 상태대로 되어 있는지" 확인하고, 아니면 맞춘다.

이 메커니즘의 코드 수준 설명은 [04-list-watch-informer.md](04-list-watch-informer.md)와
[05-controller-pattern.md](05-controller-pattern.md)에 있다.

## 왜 이렇게 설계했나 — 선언형과 수렴

사용자는 명령("컨테이너를 켜라")이 아니라 **목표 상태**("이 Pod가 3개 떠 있어야 한다")를 선언한다.
컨트롤러는 현재 상태를 목표와 비교해 **차이를 줄이는 방향으로** 계속 행동한다. 이 "수렴 루프" 덕분에:

- 컴포넌트가 잠깐 죽었다 살아나도, 다시 현재 상태를 보고 목표로 수렴하면 된다(자가 치유).
- 이벤트를 놓쳐도 다음 동기화에서 전체 상태를 다시 보므로 복구된다(level-triggered).

## 더 읽을 곳
- [03-api-object-model.md](03-api-object-model.md) — 위에서 오간 "객체"의 타입 시스템
- [04-list-watch-informer.md](04-list-watch-informer.md) — watch/캐시 메커니즘
- [05-controller-pattern.md](05-controller-pattern.md) — reconcile 루프
