# 60.01 · Manager와 Reconciler

**근거**: `controller-runtime/pkg/manager/manager.go`, `pkg/reconcile/reconcile.go`,
`pkg/builder/controller.go`

## Manager — 모든 것을 들고 도는 컨테이너

**Manager**(`pkg/manager/manager.go:54`)는 오퍼레이터의 최상위 객체다. 공유 자원(클라이언트, 캐시,
스킴)을 들고, 등록된 컨트롤러/웹훅을 함께 시작·종료한다.

```go
type Manager interface {           // manager.go:54
    GetClient() client.Client      // 캐시 읽기 + apiserver 쓰기 ([02])
    Start(ctx context.Context) error  // :90 모든 Runnable 시작
    ...
}
```

- **공유 캐시/클라이언트**: 여러 컨트롤러가 한 Manager 아래에서 informer 캐시를 공유한다
  ([00-foundations/04](../00-foundations/04-list-watch-informer.md)의 SharedInformer와 같은 절약).
- **Runnable**: 컨트롤러·웹훅 서버·헬스체크 등이 `Runnable`(`Start(ctx)`, `manager.go:308`)로 등록돼
  `mgr.Start()` 한 번에 함께 뜨고, 컨텍스트 취소 시 함께 정리된다(graceful shutdown).
- **리더 선출**: Manager 옵션으로 켜면, 여러 복제본 중 리더만 컨트롤러를 돌린다
  ([00-foundations/06](../00-foundations/06-leader-election.md)와 같은 메커니즘).

## Reconciler — 개발자가 쓰는 단 하나

`Reconcile`(`reconcile.go:126`)이 비즈니스 로직의 전부다. 전형적 구현:

```go
func (r *MyReconciler) Reconcile(ctx, req Request) (Result, error) {
    // 1. 캐시에서 객체를 읽는다 (req는 name/namespace만 줌)
    var obj MyKind
    if err := r.Get(ctx, req.NamespacedName, &obj); err != nil {
        return Result{}, client.IgnoreNotFound(err)  // 삭제됨 → 무시
    }
    // 2. 원하는 상태(obj.Spec)와 현재 상태를 비교
    // 3. 차이를 apiserver 쓰기로 좁힌다 (하위 객체 생성/갱신)
    // 4. 상태 보고 (obj.Status 갱신)
    return Result{}, nil
}
```

`Request`가 객체가 아니라 **이름**인 것이 핵심 — 항상 캐시에서 최신본을 다시 읽으므로 오래된 이벤트로
잘못 판단하지 않는다(level-triggered, [00-foundations/02](../00-foundations/02-architecture.md)).

## Builder — 선언적 배선

직접 informer/source를 엮는 대신, **Builder**(`pkg/builder/controller.go`)로 선언한다:

```go
ctrl.NewControllerManagedBy(mgr).
    For(&MyKind{}).        // controller.go:93  주 리소스를 watch
    Owns(&v1.Pod{}).       // controller.go:123 소유한 하위 리소스도 watch
    Complete(r)            // controller.go:260 Reconciler 연결
```

- `For(&MyKind{})`: 이 종류의 변경 시 Reconcile.
- `Owns(&Pod{})`: 내가 소유(ownerReference)한 Pod가 바뀌면, 그 owner(MyKind)를 Reconcile로 큐잉 —
  [00-foundations/05](../00-foundations/05-controller-pattern.md)의 역참조를 자동화.
- `Complete(r)`: workqueue·워커·이벤트 핸들러를 모두 배선하고 Manager에 등록.

이 몇 줄이 [00-foundations/04](../00-foundations/04-list-watch-informer.md)의 Reflector→Informer→
Workqueue 전체 배선을 대신한다.

## 더 읽을 곳
- [02-client-cache.md](02-client-cache.md) — Reconcile이 읽고 쓰는 클라이언트
- [03-event-pipeline.md](03-event-pipeline.md) — For/Owns 뒤의 이벤트 흐름
