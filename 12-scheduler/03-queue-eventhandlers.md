# 12.03 · 스케줄 큐와 이벤트 핸들러

**근거**: `kubernetes/pkg/scheduler/backend/queue/`, `kubernetes/pkg/scheduler/eventhandlers.go`

스케줄러는 어떤 Pod를 다음에 처리할지 **세 개의 큐**로 관리한다. 그리고 클러스터 변화(노드 추가 등)를
watch해, "전에 스케줄 못 했던 Pod를 다시 시도할지"를 판단한다.

## 세 개의 큐

`backend/queue/scheduling_queue.go:20`의 주석이 구조를 명시한다 — activeQ, backoffQ,
unschedulableEntities:

| 큐 | 의미 | 코드 |
|----|------|------|
| **activeQ** | 지금 스케줄을 시도할 Pod들 (우선순위 정렬) | `active_queue.go` |
| **backoffQ** | 실패해서 백오프 중인 Pod, 시간이 차면 activeQ로 | `backoff_queue.go` |
| **unschedulableEntities** | 시도했지만 둘 곳이 없던 Pod, 클러스터가 바뀌면 재시도 | `unschedulable_entities.go` |

스케줄러 메인 루프는 activeQ에서 Pod를 하나 꺼내(`Pop`) `scheduleOnePod`에 넘긴다. 실패하면 백오프 후
backoffQ로, 또는 unschedulable로 보낸다.

### activeQ 정렬
QueueSort 플러그인([02](02-plugins.md)의 `queuesort`)이 순서를 정한다. 기본은 **Pod 우선순위 내림차순,
같으면 큐 진입 시각**. 즉 높은 우선순위 Pod가 먼저 스케줄된다.

## 왜 unschedulable을 따로 두나 — 무의미한 재시도 방지

자리가 없어 실패한 Pod를 곧장 activeQ에 다시 넣으면, 클러스터가 그대로인데 계속 실패만 반복한다(busy
loop). 대신 unschedulable에 보관하고, **"이 Pod가 스케줄 가능해질 만한 클러스터 변화"가 생겼을 때만**
다시 activeQ로 옮긴다. 그 "변화"를 판단하는 게 이벤트 핸들러다.

## 이벤트 핸들러 — 클러스터 변화를 큐로 연결

`eventhandlers.go`는 Pod/Node/PV/PVC/StorageClass/CSINode 등 여러 리소스에 Informer 핸들러를 단다
([00-foundations/04](../00-foundations/04-list-watch-informer.md)). 변화가 생기면:

- **새 노드 추가/노드 업데이트** → unschedulable Pod 중 그 변화로 스케줄 가능해질 수 있는 것들을
  activeQ로 이동(`MoveAllToActiveOrBackoffQueue`/이벤트 기반 이동).
- **새 Pod 생성(nodeName 없음)** → activeQ에 추가.
- **Pod 삭제(자원 반환)** → 자원 부족으로 막혔던 Pod들을 재시도 대상으로.

각 플러그인은 "어떤 이벤트가 자기 결과를 바꿀 수 있는지"를 선언하므로(QueueingHint), 관련 없는 변화로는
재시도하지 않아 효율적이다.

## 더 읽을 곳
- [01-framework.md](01-framework.md) — Pop된 Pod가 들어가는 사이클
- [00-foundations/04](../00-foundations/04-list-watch-informer.md) — 이벤트 핸들러의 기반
