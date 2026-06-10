# 60 · controller-runtime

**근거 레포**: `controller-runtime` (`sigs.k8s.io/controller-runtime`)

controller-runtime은 **커스텀 컨트롤러/오퍼레이터를 만들기 위한 프레임워크**다.
[00-foundations/05](../00-foundations/05-controller-pattern.md)에서 본 reconcile 패턴과
[04](../00-foundations/04-list-watch-informer.md)의 informer 머시너리를, 사용자가 매번 직접 배선하지
않도록 깔끔한 추상화로 감싼다. 빌트인 컨트롤러([11-controller-manager](../11-controller-manager/))와
같은 원리지만, 외부 개발자가 CRD를 동작시킬 때 쓰는 라이브러리다.

## 핵심: Reconcile만 작성하면 된다

controller-runtime의 약속은 단순하다. 개발자는 **`Reconcile` 함수 하나**만 구현하면 된다
(`pkg/reconcile/reconcile.go:126`):

```go
type Reconciler interface {
    Reconcile(context.Context, Request) (Result, error)
}
```

- `Request`: 변경된 객체의 `namespace/name`(객체 자체가 아님 — 캐시에서 최신본을 읽으라는 의미,
  level-triggered).
- `Result`: 재큐 여부/지연(`Requeue`, `RequeueAfter`). 에러를 반환하면 백오프 후 재시도.

informer, workqueue, 캐시, 재시도는 전부 프레임워크가 처리한다. 개발자는 "이 객체가 원하는 상태대로
되어 있는지 확인하고 맞추는" 로직만 쓴다.

## 구성요소

```
Manager (pkg/manager)  ── 모든 것을 들고 Start()
  ├─ Client + Cache (pkg/client, pkg/cache)  ── 읽기는 캐시, 쓰기는 apiserver
  ├─ Controller (pkg/controller)             ── workqueue + 워커
  │    └─ Source → Handler → Predicate (pkg/source, handler, predicate)
  │         이벤트를 받아 Request를 큐에 적재
  └─ Webhook (pkg/webhook)                    ── admission/conversion 웹훅 서버
```

## 문서

| # | 문서 | 내용 |
|---|------|------|
| 01 | [manager-reconciler.md](01-manager-reconciler.md) | Manager 수명주기, Reconciler, Builder |
| 02 | [client-cache.md](02-client-cache.md) | 캐시 읽기 / apiserver 쓰기 클라이언트 |
| 03 | [event-pipeline.md](03-event-pipeline.md) | Source→Handler→Predicate 이벤트 파이프라인 |
| 04 | [webhook-envtest.md](04-webhook-envtest.md) | admission/conversion 웹훅, envtest |
| 05 | [finalizers.md](05-finalizers.md) | finalizer 기반 삭제 처리 |
| 06 | [manager-extras.md](06-manager-extras.md) | healthz/metrics/leader/certwatcher |

## 진입점
- Reconciler 인터페이스: `controller-runtime/pkg/reconcile/reconcile.go`
- Manager: `controller-runtime/pkg/manager/manager.go`
- Builder: `controller-runtime/pkg/builder/controller.go`

## 더 읽을 곳
- [00-foundations/05](../00-foundations/05-controller-pattern.md) — 추상화 이전의 raw 패턴
- [61-kubebuilder.md](../61-kubebuilder.md) — 이 프레임워크를 쓰는 코드를 스캐폴딩
- [10-apiserver/07](../10-apiserver/07-crd-aggregation.md) — 컨트롤러가 동작시키는 CRD
