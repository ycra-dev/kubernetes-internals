# 12.01 · Scheduling Framework

**근거**: `staging/src/k8s.io/kube-scheduler/framework/interface.go`,
`kubernetes/pkg/scheduler/schedule_one.go`, `pkg/scheduler/framework/`

Framework는 스케줄링 사이클의 뼈대다. 각 확장점에서 등록된 플러그인들을 순서대로 호출하고, 그 결과를
모아 "이 Pod를 어느 노드에"를 결정한다. 정책은 전부 플러그인에 있고, Framework는 그 호출 순서와
상태 전달만 책임진다.

## 스케줄링 사이클의 확장점 순서

`schedulingCycle`(`schedule_one.go:169`) 안에서 확장점이 다음 순서로 실행된다:

```
QueueSort      ─ (큐 단계) Pod 처리 순서 결정
   │
PreFilter      ─ 사전 계산, 불가능하면 조기 종료
   │
Filter         ─ findNodesThatFitPod (:622): 각 노드에 대해 가/부
   │             → "feasible nodes"(받을 수 있는 노드) 집합
   ▼ (feasible == 0)
PostFilter     ─ 보통 preemption 시도 (:277)
   │ (feasible >= 1)
PreScore / Score ─ prioritizeNodes (:937): 노드마다 0~100 점, 가중합으로 1등 선택
   │
Reserve        ─ 선택 노드에 자원을 낙관적으로 예약 (실패 시 Unreserve로 롤백)
   │
Permit         ─ 바인딩을 허용/지연/거부 (gang scheduling 등)
```

여기까지가 동기 **스케줄링 사이클**. 결과(선택된 노드)는 메모리에 반영되고, 실제 확정은 비동기
**바인딩 사이클**로 넘어간다.

## 바인딩 사이클

`bindingCycle`(`schedule_one.go:391`)은 고루틴으로 실행돼 여러 Pod의 바인딩이 병행된다:

```
PreBind   ─ 바인딩 전 준비 (예: 볼륨 프로비저닝/attach 대기 → volumebinding 플러그인)
Bind      ─ 실제 Binding 객체 생성 = apiserver에 Pod.spec.nodeName 설정 (defaultbinder)
PostBind  ─ 정리/통지
```

`Bind`가 성공하면 그 Pod는 더 이상 스케줄러의 일이 아니다 — 해당 노드의 kubelet이 watch로 받아
띄운다([13-kubelet](../13-kubelet/)).

## 필터 — "받을 수 있나" (Predicate)

`findNodesThatFitPod`(`schedule_one.go:622`)는 모든(또는 표본) 노드에 대해 Filter 플러그인을 돌려
**하나라도 거부하면 그 노드 탈락**. 결과는 "feasible nodes". 병렬로 평가된다(`framework/parallelize`).
대표 필터: 노드에 남은 CPU/메모리가 충분한가, nodeSelector/affinity가 맞는가, taint를 톨러레이트하나,
포트가 비었나 등 → [02-plugins.md](02-plugins.md).

## 점수 — "얼마나 좋은가" (Priority)

`prioritizeNodes`(`schedule_one.go:937`)는 남은 feasible 노드마다 Score 플러그인을 돌려 점수를 매기고,
플러그인별 가중치로 합산해 **최고점 노드**를 고른다(동점이면 무작위 분산). 대표 점수: 자원이 고르게
분산되는가, 이미지가 노드에 캐시돼 있는가(`imagelocality`), affinity 선호가 맞는가 등.

## CycleState — 플러그인 간 상태 공유

한 사이클 동안 플러그인들이 계산 결과를 공유하는 스크래치패드가 `CycleState`
(`pkg/scheduler/framework/cycle_state.go`)다. 예: PreFilter가 계산한 값을 Filter가 읽어 재계산을
피한다. CycleState는 Pod마다 새로 만들어지고 사이클이 끝나면 버려진다.

## 더 읽을 곳
- [02-plugins.md](02-plugins.md) — 빌트인 필터/점수 플러그인
- [03-queue-eventhandlers.md](03-queue-eventhandlers.md) — QueueSort 이전, 큐 동작
- [04-preemption-extender.md](04-preemption-extender.md) — PostFilter의 선점
