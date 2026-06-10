# 00.05 · 컨트롤러 / reconcile 루프

**근거 레포**: `kubernetes` (`pkg/controller/`), `client-go`

컨트롤러는 Kubernetes의 일꾼이다. 각 컨트롤러는 한 종류의 리소스를 책임지고, **"원하는 상태(spec)와
현재 상태(status)의 차이를 줄이는"** 단순한 루프를 끝없이 돈다. 이 문서는 그 보편 패턴을 실제
ReplicaSet 컨트롤러 코드로 따라가고, 객체 소유/삭제(ownerReference, finalizer, GC)를 설명한다.

## 보편 패턴

```
                ┌──────────── Informer (로컬 캐시, [04] 참조)
                │  변경 통지
                ▼
   이벤트 핸들러: 변경된 객체의 key 를 Workqueue 에 enqueue
                │
                ▼
   워커 루프: key 를 Pop → sync(key) 호출
                │
                ▼
   sync(key):
     1. 캐시(Lister)에서 객체를 읽는다 (없으면 = 삭제됨)
     2. "원하는 상태" 와 "현재 상태" 를 비교한다
     3. 차이가 있으면 apiserver 에 쓰기(생성/삭제/갱신)로 차이를 줄인다
     4. 실패하면 에러 반환 → Workqueue 가 재시도(백오프)
```

이 루프는 **level-triggered** 다: "무슨 이벤트가 왔는지"가 아니라 "지금 전체 상태가 어떤지"를 보고
판단한다. 그래서 이벤트를 놓쳐도, 컨트롤러가 재시작해도, 다음 sync에서 복구된다.

## 실제 코드: ReplicaSet 컨트롤러

`kubernetes/pkg/controller/replicaset/replica_set.go`. "ReplicaSet은 N개의 Pod가 떠 있게 한다"는
한 문장이 코드로는 이렇게 된다.

### 1) 배선 — Informer + Workqueue
생성자에서(`replica_set.go:214` 부근):

```go
queue: workqueue.NewTypedRateLimitingQueueWithConfig(
    workqueue.DefaultTypedControllerRateLimiter[string](), ...)   // 재시도/백오프 큐
```

ReplicaSet과 Pod 두 Informer에 이벤트 핸들러를 단다(`:223`, `:251`). 핸들러는 변경 시 관련 RS의 키를
큐에 넣는다(`enqueueRS`, `:367`). Pod 변경도 그 Pod의 소유 RS를 찾아 enqueue 한다 — Pod가 죽으면
ReplicaSet을 다시 sync해서 새 Pod를 띄워야 하므로.

### 2) 워커 — 큐를 소비
`Run`(`:275`)이 N개의 워커를 띄우고, 각 워커는 큐에서 키를 꺼내 `syncReplicaSet(key)`(`:755`)를 부른다.

### 3) reconcile 핵심 — 차이를 계산해 맞춘다
`manageReplicas`(`:649`)의 첫 줄이 이 패턴의 정수다:

```go
diff := len(activePods) - int(*(rs.Spec.Replicas))   // 현재 - 원하는
```

- `diff < 0` (부족): 부족분만큼 Pod를 만든다. 단 한 번에 폭주하지 않도록 `burstReplicas`로 제한하고,
  `slowStartBatch`로 점진 생성한다(`:649` 이하). 생성 시 `metav1.NewControllerRef(rs, ...)`로
  **소유 참조(ownerReference)** 를 박는다(아래 참조).
- `diff > 0` (초과): 초과분 Pod를 삭제한다.
- `diff == 0`: 아무것도 안 한다 — 이미 원하는 상태.

> `expectations`(`ExpectCreations`)는 "방금 생성을 요청했지만 아직 캐시에 안 보이는 Pod"를 추적해
> 같은 sync가 중복 생성하는 것을 막는 보정 장치다. level-triggered 루프가 캐시 지연 때문에
> 과잉 반응하지 않게 한다.

이 한 컨트롤러를 이해하면 나머지 빌트인 컨트롤러
(`pkg/controller/{deployment,job,statefulset,daemon,...}`)도 같은 골격임을 알 수 있다.
전체 카탈로그는 [11-controller-manager/01-controller-catalog.md](../11-controller-manager/).

## 객체 소유 — ownerReferences

컨트롤러가 만든 객체에는 `ObjectMeta.OwnerReferences`로 "누가 나를 소유하는가"가 기록된다
(위 `NewControllerRef`). 이것이 두 가지를 가능케 한다:

1. **역참조**: Pod가 변하면 그 Pod의 owner(RS)를 찾아 다시 sync. ReplicaSet 컨트롤러는 이를 위해
   Pod에 owner 기반 인덱서를 단다(`controller.AddPodControllerIndexer`, `:267`).
2. **가비지 컬렉션**: owner가 삭제되면 그 dependent들도 정리된다(아래).

## 삭제 — finalizer와 가비지 컬렉션

### Finalizer
객체를 지워도 즉시 사라지지 않을 수 있다. `ObjectMeta.Finalizers`에 항목이 있으면, apiserver는 객체를
실제로 지우는 대신 `DeletionTimestamp`만 찍는다([03](03-api-object-model.md) 참조). 그 finalizer를
책임지는 컨트롤러가 정리 작업을 끝내고 finalizer를 제거해야 비로소 객체가 삭제된다. = "지우기 전 훅".

### Garbage Collector
`kubernetes/pkg/controller/garbagecollector/`는 ownerReference 그래프를 관리하는 특수 컨트롤러다.
owner가 사라진 dependent를 어떻게 처리할지 세 정책이 있고, 각각 finalizer 상수로 표현된다
(`garbagecollector.go`, `metav1` 상수):

| 정책 | 의미 | finalizer |
|------|------|-----------|
| Background | owner 즉시 삭제, dependent는 비동기 정리 | (기본) |
| Foreground | dependent 먼저 삭제, 그다음 owner | `metav1.FinalizerDeleteDependents` (`:668`) |
| Orphan | owner만 삭제, dependent는 고아로 남김 | `metav1.FinalizerOrphanDependents` (`:769`) |

GC는 클러스터 전체의 owner→dependent 그래프(`graph_builder.go`, `graph.go`)를 watch로 구축하고,
owner가 없어진 노드를 찾아 정책에 따라 삭제/고아 처리한다.

## 고가용성과의 관계

controller-manager는 여러 인스턴스로 떠도 **한 번에 하나만** 활성화돼야 한다(같은 객체를 둘이 동시에
reconcile하면 충돌). 이를 위해 리더 선출을 쓴다 → [06-leader-election.md](06-leader-election.md).

## 주요 디렉토리

| 경로 | 책임 |
|------|------|
| `pkg/controller/replicaset/` | 본문에서 추적한 예시 컨트롤러 |
| `pkg/controller/controller_utils.go`, `controller_ref_manager.go` | 공통 유틸·소유 관리 |
| `pkg/controller/garbagecollector/` | ownerReference 기반 GC |
| `client-go/util/workqueue/` | 재시도 큐 |

## 더 읽을 곳
- [04-list-watch-informer.md](04-list-watch-informer.md) — 이 루프가 읽는 캐시/큐
- [11-controller-manager/](../11-controller-manager/) — 빌트인 컨트롤러 전체
- [60-controller-runtime/](../60-controller-runtime/) — 이 패턴을 추상화한 `Reconcile()` 프레임워크
