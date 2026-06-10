# 13.01 · Pod 라이프사이클과 SyncLoop

**근거**: `kubernetes/pkg/kubelet/kubelet.go`, `kubernetes/pkg/kubelet/pleg/`,
`kubernetes/pkg/kubelet/kuberuntime/kuberuntime_manager.go`

kubelet은 Pod를 "원하는 상태(spec)"로 끊임없이 수렴시킨다. 컨트롤 플레인의 reconcile 루프
([00-foundations/05](../00-foundations/05-controller-pattern.md))와 같은 사상이지만, 대상이 *로컬
컨테이너*다.

## SyncLoop — 이벤트를 받아 Pod를 동기화

`syncLoop`(`kubelet.go:2630`)와 `syncLoopIteration`(`:2705`)이 핵심이다. 입력 채널(configCh / plegCh /
syncCh / housekeepingCh / health)에서 이벤트를 받아 영향받는 Pod를 **pod worker**에 디스패치한다
([README](README.md)의 채널 표 참조). 각 Pod는 자기 전용 worker(고루틴)가 직렬로 처리해 한 Pod에 대한
동기화가 겹치지 않게 한다.

## PLEG — 컨테이너 상태 변화를 감지

apiserver는 "원하는 상태"만 알려준다. "컨테이너가 실제로 죽었다" 같은 **현재 상태 변화**는 누가 알려주나?
**PLEG (Pod Lifecycle Event Generator)** 다(`pkg/kubelet/pleg/`):

- **Generic PLEG**(`generic.go`): 주기적으로 런타임에 "지금 어떤 컨테이너가 떠 있나"를 relist 해
  이전 상태와 비교, 차이를 `PodLifecycleEvent`로 SyncLoop의 `plegCh`에 보낸다
  (`kubelet.go:867`에서 생성).
- **Evented PLEG**(`evented.go`): CRI 런타임이 이벤트를 push하면 relist 빈도를 낮춰 부하를 줄인다
  (feature gate, `kubelet.go:875`). 오류 시 Generic으로 폴백.

즉 PLEG가 "컨테이너가 죽었다"를 감지 → SyncLoop이 SyncPod 호출 → 죽은 컨테이너를 재시작.

## SyncPod — 차이를 메우는 핵심

한 Pod를 동기화하는 실제 작업은 `kuberuntime.SyncPod`
(`kuberuntime/kuberuntime_manager.go:1450`)다. 개념적 단계:

1. **현재 상태 계산**: 런타임에서 이 Pod의 sandbox/컨테이너 현황을 읽는다.
2. **변경 계산**: 무엇을 죽이고/만들고/재시작할지 결정(`computePodActions` 류).
3. **sandbox 보장**: Pod의 네트워크 네임스페이스(sandbox/pause 컨테이너)가 없으면 생성 →
   이때 **CNI**로 네트워크를 붙인다([42-cilium](../42-cilium/)).
4. **이미지 pull** → **init 컨테이너 순차 실행** → **app 컨테이너 시작**. 전부 **CRI**로 런타임에 요청
   ([02-cri.md](02-cri.md)).
5. **재시작 백오프**: 죽은 컨테이너를 즉시 무한 재시작하지 않도록 지수 백오프 적용.

SyncPod는 멱등적이다 — 이미 원하는 대로면 아무것도 안 한다. 그래서 SyncLoop이 같은 Pod를 여러 번
sync해도 안전하다(level-triggered).

## status 보고

SyncPod 후 kubelet은 Pod의 **status**(phase: Pending/Running/Succeeded/Failed, 컨테이너별 상태,
조건)를 계산해 apiserver에 보고한다(status manager). 이 status가 [00-foundations/03](../00-foundations/03-api-object-model.md)의
spec/status 분리에서 "현재 상태"에 해당한다. 다른 컴포넌트(스케줄러, 컨트롤러, 사용자)는 이걸 보고 판단한다.

## probe — liveness / readiness / startup

prober(`pkg/kubelet/prober/`)가 컨테이너에 주기적으로 probe를 실행한다:
- **liveness** 실패 → 컨테이너 재시작.
- **readiness** 실패 → EndpointSlice에서 제외(서비스 트래픽 차단, [14-kube-proxy](../14-kube-proxy/)).
- **startup** → 느리게 뜨는 앱의 초기 유예.

## 더 읽을 곳
- [02-cri.md](02-cri.md) — SyncPod가 컨테이너를 만드는 메커니즘
- [80-scenarios/01-pod-create.md](../80-scenarios/) — 전 과정 시간순
