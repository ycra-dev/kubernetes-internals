# 12.08 · QueueingHint — 똑똑한 재시도

**근거**: `kubernetes/staging/src/k8s.io/kube-scheduler/framework/interface.go`
(`EventsToRegister` `:495`, `ClusterEventWithHint`), `kubernetes/pkg/scheduler/eventhandlers.go`
(`MoveAllToActiveOrBackoffQueue` `:63`)

[03-queue-eventhandlers.md](03-queue-eventhandlers.md)에서 "unschedulable Pod를 클러스터 변화 시 다시
시도한다"고 했다. 이 문서는 그 **"어떤 변화가 어떤 Pod를 재시도하게 하나"** 를 정밀하게 거르는
QueueingHint 메커니즘을 본다. 무의미한 재시도(다시 실패할 게 뻔한)를 없애 스케줄러 효율을 크게 높인다.

## 문제 — 둔한 재시도

순진한 방식: 클러스터에 **아무 변화**라도 생기면 모든 unschedulable Pod를 activeQ로 옮겨 재시도. 문제:

- ConfigMap 하나 바뀐 것과 "CPU 부족으로 막힌 Pod"는 무관한데도 재시도 → 또 실패 → 낭비.
- 대규모 클러스터에서 변화는 끊임없으니, unschedulable Pod들이 계속 헛돈다.

## EventsToRegister — 플러그인이 관심 이벤트를 선언

각 플러그인은 "내 결정을 **바꿀 수 있는** 이벤트는 무엇인가"를 선언한다
(`interface.go:495`의 `EventsToRegister`가 `[]ClusterEventWithHint` 반환):

| 플러그인 | 관심 이벤트(예) |
|----------|-----------------|
| `noderesources/fit` | Node 추가, Pod 삭제(자원 반환), Node allocatable 증가 |
| `nodename` | Node 추가 |
| `interpodaffinity` | Pod 추가/삭제/라벨 변경 |
| `tainttoleration` | Node taint 변경 |
| `dynamicresources` | ResourceSlice/ResourceClaim 변경([19-dra](../19-dra.md)) |

`noderesources`로 막힌 Pod는 "ConfigMap 변경"엔 관심 없고 "Node 추가/Pod 삭제"에만 관심 있다. 무관한
이벤트엔 아예 재시도되지 않는다.

## QueueingHint — 한 단계 더 정밀하게

이벤트 종류만으로도 부족할 때가 있다. "Pod가 삭제됐다"고 모든 자원 부족 Pod를 재시도하면, 작은 Pod가
삭제됐는데 큰 Pod를 재시도하는 헛수고가 생긴다.

**QueueingHint 함수**는 *구체적 이벤트 객체*를 보고 판정한다:

```
이벤트: "Pod X 삭제됨"
   │  unschedulable Pod P의 noderesources 플러그인 QueueingHint 호출
   ▼
"삭제된 X가 P가 막힌 노드에 있었고, 충분한 자원을 반환하나?"
   ├─ Queue      → activeQ로 (재시도 가치 있음)
   └─ QueueSkip  → 그대로 둠 (재시도해도 또 실패)
```

즉 플러그인이 "이 변화가 *이 특정 Pod*를 스케줄 가능하게 만들 수 있나"를 판단해, **될 가망이 있을 때만**
재시도시킨다.

## 이벤트가 큐를 움직이는 경로

`eventhandlers.go`가 Informer 이벤트를 받아 큐를 움직인다([03-queue-eventhandlers.md](03-queue-eventhandlers.md)):

- Node 추가(`:63`), Node 업데이트(`:90`), Node 삭제(`:118`) 등이 `MoveAllToActiveOrBackoffQueue`를
  호출하며 이벤트를 전달.
- 큐는 각 unschedulable Pod에 대해 등록된 플러그인의 QueueingHint를 돌려, `Queue`인 것만 activeQ/backoffQ로
  옮긴다.

## 효과

```
순진한 방식:  아무 변화 → 모든 unschedulable Pod 재시도 → 대량 헛수고
QueueingHint: 관련 이벤트 → 그 이벤트로 풀릴 가망 있는 Pod만 재시도
```

대규모 클러스터에서 스케줄러 CPU와 지연을 크게 줄인다 — "막힌 Pod를 정확히 풀릴 때 깨우는" 것이다.

## 더 읽을 곳
- [03-queue-eventhandlers.md](03-queue-eventhandlers.md) — 세 큐와 이벤트 핸들러
- [01-framework.md](01-framework.md) — 플러그인 확장점
