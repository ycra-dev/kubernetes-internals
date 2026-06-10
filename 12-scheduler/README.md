# 12 · kube-scheduler

**근거 레포**: `kubernetes` (`cmd/kube-scheduler`, `pkg/scheduler/`,
`staging/src/k8s.io/kube-scheduler/framework/`)

스케줄러는 **아직 노드가 정해지지 않은 Pod(`spec.nodeName`이 빈)** 에게 가장 알맞은 노드를 골라준다.
실제로 컨테이너를 띄우는 것은 kubelet이고, 스케줄러는 "어디에"만 결정한다 — 그 결정을 **Binding**으로
apiserver에 기록한다.

## 역할

- 스케줄 대기 Pod를 큐에서 꺼내, 후보 노드들을 **필터링(filter)** → **점수화(score)** → 최고점 노드 선택.
- 노드가 없으면 **선점(preemption)** 을 시도(낮은 우선순위 Pod를 비워 자리 확보).
- 선택을 `Pod.spec.nodeName`을 설정하는 **Binding** API 호출로 확정.

## 두 사이클: scheduling cycle + binding cycle

진입점은 `pkg/scheduler/schedule_one.go`. `scheduleOnePod`(`:93`)이 Pod 하나를 처리하며,
**스케줄링 사이클**(동기, 결정)과 **바인딩 사이클**(비동기, 확정)로 나뉜다:

```
scheduleOnePod (:93)
  │
  ├─ schedulingCycle (:169)   ── 동기, 한 번에 하나씩
  │    SchedulePod (:264)
  │      ├─ findNodesThatFitPod (:622)   필터: 못 받는 노드 제거
  │      └─ prioritizeNodes (:937)       점수: 남은 노드에 점수 매겨 1등 선택
  │    (실패 시 PostFilter = preemption 시도, :277)
  │
  └─ bindingCycle (:391)      ── 비동기(고루틴), 여러 개 병행
       Reserve → Permit → PreBind → Bind(Binding 생성) → PostBind
```

스케줄링 사이클은 **직렬**(한 번에 한 Pod)로 돌아 결정 일관성을 보장하고, 느릴 수 있는 바인딩
(볼륨 프로비저닝 대기 등)은 **병렬**로 떼어내 처리량을 높인다.

## Scheduling Framework — 확장점 플러그인

스케줄러의 모든 로직은 **플러그인**으로 구현된다. Framework는 사이클의 정해진 지점("확장점")마다
등록된 플러그인을 호출한다. 확장점 인터페이스는
`staging/src/k8s.io/kube-scheduler/framework/interface.go`에 정의돼 있다:

| 확장점 | 인터페이스(`interface.go`) | 역할 |
|--------|----------------------------|------|
| QueueSort | `QueueSortPlugin` (`:459`) | 큐에서 Pod 처리 순서 결정 |
| PreFilter | `PreFilterPlugin` (`:513`) | 필터 전 사전 계산/조기 거부 |
| Filter | `FilterPlugin` (`:542`) | 이 노드가 Pod를 받을 수 있나(가부) |
| PostFilter | `PostFilterPlugin` (`:571`) | 후보 0개일 때(예: preemption) |
| PreScore / Score | `PreScorePlugin`(`:598`)/`ScorePlugin`(`:619`) | 노드에 점수 매기기 |
| Reserve | `ReservePlugin` (`:636`) | 자원 예약(낙관적) |
| Permit | `PermitPlugin` (`:680`) | 바인딩 승인 보류/지연(gang scheduling 등) |
| PreBind / Bind / PostBind | (`:652`/`:693`/`:669`) | 바인딩 실행 |

이 플러그인 모델 덕분에 스케줄링 정책을 **코드 수정 없이 조합·교체**할 수 있다(프로파일 설정).

## 문서

| # | 문서 | 내용 |
|---|------|------|
| 01 | [framework.md](01-framework.md) | 확장점/사이클/CycleState 상세 |
| 02 | [plugins.md](02-plugins.md) | filter/score 빌트인 플러그인 |
| 03 | [queue-eventhandlers.md](03-queue-eventhandlers.md) | activeQ/backoffQ/unschedulable, 이벤트 |
| 04 | [preemption-extender.md](04-preemption-extender.md) | 선점과 외부 extender |
| 05 | [affinity-topology.md](05-affinity-topology.md) | affinity/anti-affinity, topology spread |
| 06 | [scoring.md](06-scoring.md) | 점수 합산, bin packing vs spreading |
| 07 | [internals.md](07-internals.md) | 노드 스냅샷, assume, percentageOfNodes |
| 08 | [queueing-hints.md](08-queueing-hints.md) | QueueingHint: 똑똑한 재시도 |
| 09 | [gang-scheduling.md](09-gang-scheduling.md) | Gang 스케줄링(PodGroup/MinMember/Permit) |

## 진입점
- 바이너리: `kubernetes/cmd/kube-scheduler/scheduler.go`
- 사이클: `kubernetes/pkg/scheduler/schedule_one.go`
- 큐: `kubernetes/pkg/scheduler/backend/queue/`

## 더 읽을 곳
- [11-controller-manager/03](../11-controller-manager/03-deployment-replicaset.md) — 스케줄 대상 Pod를 만드는 쪽
- [13-kubelet](../13-kubelet/) — Binding 후 실제로 Pod를 띄우는 쪽
