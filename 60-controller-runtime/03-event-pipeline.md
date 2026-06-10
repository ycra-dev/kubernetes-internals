# 60.03 · 이벤트 파이프라인 (Source → Handler → Predicate)

**근거**: `controller-runtime/pkg/source/`, `pkg/handler/`, `pkg/predicate/`, `pkg/controller/`

[01](01-manager-reconciler.md)의 `For`/`Owns`는 내부적으로 이 파이프라인을 배선한다. "어떤 변화가
어떤 Reconcile 요청이 되는가"를 결정하는 단계들이다.

```
Source ──► Predicate(필터) ──► Handler(매핑) ──► Workqueue ──► Reconcile
(이벤트 발생) (관심 있는가)    (어떤 객체를 큐에)
```

## Source — 이벤트의 출처

`pkg/source/`. 가장 흔한 것은 informer 기반(`Kind`) 소스 — 특정 종류의 객체 변화를 watch한다
([00-foundations/04](../00-foundations/04-list-watch-informer.md)). 그 외:

- **Channel**: 외부 이벤트(예: 비-Kubernetes 시스템)를 reconcile로 흘려보내는 소스.
- `For(&MyKind{})`는 MyKind에 대한 Kind 소스를 만든다.

## Predicate — 관심 필터

`pkg/predicate/`. 모든 변경을 reconcile할 필요는 없다. Predicate는 이벤트를 **걸러낸다**:

- `GenerationChangedPredicate`: spec이 바뀐 경우만(status-only 변경 무시). status 갱신이 다시 reconcile을
  부르는 무한 루프를 막는 데 흔히 쓰인다.
- `LabelChangedPredicate`, 커스텀 술어 등.

Predicate를 통과하지 못한 이벤트는 큐에 들어가지 않는다 — 불필요한 reconcile을 줄인다.

## Handler — 이벤트 → Request 매핑

`pkg/handler/`. "이 이벤트가 *어떤 객체*의 reconcile을 유발하는가"를 정한다:

- **EnqueueRequestForObject** (`For`의 기본): 바뀐 객체 자신을 큐에. MyKind가 바뀌면 그 MyKind를.
- **EnqueueRequestForOwner** (`Owns`의 기본): 바뀐 객체의 **owner**를 큐에. 소유한 Pod가 바뀌면 그
  Pod의 owner(MyKind)를 reconcile — [00-foundations/05](../00-foundations/05-controller-pattern.md)의
  역참조.
- **EnqueueRequestsFromMapFunc**: 임의 매핑. 예: ConfigMap이 바뀌면 그것을 참조하는 모든 MyKind를 큐에.
  여러 객체로의 fan-out에 쓴다.

## Controller — 워커와 큐

`pkg/controller/`가 workqueue와 워커 고루틴을 관리한다([00-foundations/04](../00-foundations/04-list-watch-informer.md)의
Workqueue). 큐에서 Request를 꺼내 Reconcile을 부르고, 에러/`Requeue` 시 백오프로 다시 넣는다. 동시
워커 수(`MaxConcurrentReconciles`)로 처리량을 조절한다.

## 전체를 다시 보면

```
informer watch (Source)
   → "spec 바뀐 것만"(Predicate)
   → "이 객체/그 owner를 큐에"(Handler)
   → workqueue (중복제거/재시도)
   → Reconcile(name) → 캐시에서 읽고 차이를 좁힘
```

이것이 빌트인 컨트롤러([00-foundations/05](../00-foundations/05-controller-pattern.md))가 손으로 짜던
배선과 정확히 같은 구조를, 선언적으로 제공하는 것이다.

## 더 읽을 곳
- [01-manager-reconciler.md](01-manager-reconciler.md) — For/Owns가 이 파이프라인을 만든다
- [00-foundations/04](../00-foundations/04-list-watch-informer.md) — informer/workqueue 기반
