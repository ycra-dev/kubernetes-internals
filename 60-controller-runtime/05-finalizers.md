# 60.05 · Finalizer와 삭제 처리

**근거**: `controller-runtime/pkg/finalizer/finalizer.go`(`finalizers` `:25`, `NewFinalizers` `:38`)

오퍼레이터가 만든 외부 리소스(클라우드 LB, DNS 레코드, 외부 DB)는 커스텀 리소스가 삭제될 때 **함께
정리**되어야 한다. 그냥 두면 누수된다. 이를 안전하게 하는 것이 **finalizer**다. 개념은
[00-foundations/05](../00-foundations/05-controller-pattern.md)에 있고, 여기선 controller-runtime의
처리 패턴을 본다.

## 문제 — 삭제를 가로채야 한다

커스텀 리소스 `MyApp`이 외부에 클라우드 버킷을 만든다고 하자. 사용자가 `kubectl delete myapp x`하면:
- API 객체는 즉시 사라질 수 있지만,
- **외부 버킷은 그대로 남는다** — Reconcile이 "지워야 한다"는 걸 알기도 전에 객체가 없어졌으므로.

**finalizer**가 이를 푼다([00-foundations/05](../00-foundations/05-controller-pattern.md)): finalizer가
있으면 apiserver는 객체를 실제로 지우지 않고 **`DeletionTimestamp`만 찍는다**. 컨트롤러가 정리를 끝내고
finalizer를 제거해야 비로소 삭제된다.

## Reconcile에서의 표준 패턴

controller-runtime 기반 컨트롤러의 전형적 삭제 처리:

```go
func (r *MyAppReconciler) Reconcile(ctx, req) (Result, error) {
    var app MyApp
    r.Get(ctx, req.NamespacedName, &app)

    if app.DeletionTimestamp.IsZero() {
        // 살아 있음: finalizer가 없으면 추가
        if !containsFinalizer(&app, myFinalizer) {
            addFinalizer(&app, myFinalizer); r.Update(ctx, &app)
        }
        // ... 정상 reconcile (외부 리소스 생성/유지) ...
    } else {
        // 삭제 중(DeletionTimestamp 있음):
        if containsFinalizer(&app, myFinalizer) {
            cleanupExternalResources()           // 외부 버킷 삭제
            removeFinalizer(&app, myFinalizer)    // 정리 끝 → finalizer 제거
            r.Update(ctx, &app)                   // 이제 apiserver가 객체를 실제 삭제
        }
    }
    return Result{}, nil
}
```

핵심: **`DeletionTimestamp`의 유무로 "생성/유지"와 "정리"를 분기**한다. 정리가 끝나야 finalizer를 떼고,
그때 비로소 객체가 사라진다. 정리가 실패하면 finalizer가 남아 객체가 "Terminating"에 머문다(재시도).

## Finalizers 헬퍼

매번 위 패턴을 손으로 짜는 대신, controller-runtime의 `finalizer` 패키지(`finalizer.go:25`
`finalizers map`, `:38` `NewFinalizers`)가 여러 finalizer를 등록·실행하는 헬퍼를 제공한다:

- 각 finalizer를 키로 등록(`Register`).
- `Finalize(ctx, obj)` 호출 시, 객체가 삭제 중이면 등록된 finalizer들의 정리 로직을 실행하고 끝난
  것의 표식을 제거한다.
- 여러 관심사(예: LB 정리 + DNS 정리)를 독립 finalizer로 나눠 관리할 수 있다.

## 주의점

- **finalizer 누수**: 컨트롤러가 영영 정리를 못 하면(외부 의존성 영구 장애) 객체가 Terminating에 갇힌다.
  타임아웃/강제 제거 전략이 필요할 수 있다.
- **멱등성**: 정리 로직은 여러 번 호출돼도 안전해야 한다(Reconcile은 재시도되므로,
  [01-manager-reconciler.md](01-manager-reconciler.md)).
- **GC와의 관계**: 빌트인 GC([11-controller-manager/04](../11-controller-manager/04-garbage-collector.md))의
  foreground 삭제도 finalizer(`foregroundDeletion`)를 쓴다 — 같은 메커니즘이다.

## 더 읽을 곳
- [00-foundations/05](../00-foundations/05-controller-pattern.md) — finalizer 개념
- [01-manager-reconciler.md](01-manager-reconciler.md) — Reconcile 패턴
- [11-controller-manager/04](../11-controller-manager/04-garbage-collector.md) — GC의 finalizer
