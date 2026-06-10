# 00.04 · list/watch/informer 메커니즘

**근거 레포**: `client-go` (원본: `kubernetes/staging/src/k8s.io/client-go/tools/cache/`)

컨트롤러는 apiserver를 매 reconcile마다 직접 호출하지 않는다. 대신 **Informer**가 객체들을 한 번 list 하고
이후 watch로 증분 갱신하며 **로컬 캐시**를 유지한다. 컨트롤러는 그 캐시를 읽고, 변경 통지를 받아 동작한다.
이 문서는 그 데이터 파이프라인을 코드 수준에서 따라간다.

## 전체 파이프라인

```
apiserver ──(List + Watch)──► Reflector ──push──► DeltaFIFO ──Pop──► HandleDeltas
                              (생산자)            (변경 큐)            │
                                                                      ├─► Indexer (로컬 캐시 갱신)
                                                                      └─► sharedProcessor ─► 각 핸들러(컨트롤러)
                                                                                              └─► Workqueue
```

`sharedIndexInformer`의 세 구성요소는 코드 주석에 명시돼 있다
(`client-go/tools/cache/shared_informer.go:575`):

> `*sharedIndexInformer` ... has three main components. One is an indexed local cache, `indexer Indexer`.
> The second ... is a Controller that pulls objects/notifications using the ListerWatcher and pushes them
> into a DeltaFIFO ... while concurrently Popping Deltas values from that fifo and processing them with
> `sharedIndexInformer::HandleDeltas`. ... The third main component is that sharedProcessor, which is
> responsible for relaying those notifications to each of the informer's clients.

## [1] Reflector — list 한 번, 그다음 watch

`Reflector`(`reflector.go`)가 apiserver와 직접 말하는 부분이다. 진입점은
`ListAndWatch`(`reflector.go:463`):

1. **초기 List**: `relistResourceVersion()`을 `ListOptions`에 넣어 전체 목록을 가져온다
   (`reflector.go:676`, 페이지네이션은 `pager.ListWithAlloc`, `:728`).
2. **store 동기화**: 받은 목록으로 큐의 내용을 통째로 교체한다(`syncWith`, `reflector.go:910`),
   그리고 마지막으로 본 `resourceVersion`을 기록한다(`setLastSyncResourceVersion`, `:780`).
3. **Watch**: 그 resourceVersion **이후의 변경만** watch 스트림으로 받는다. 변경마다 store에
   Add/Update/Delete 하고 RV를 갱신한다(`reflector.go:633`).
4. **재동기(relist)**: watch가 끊기거나 RV가 너무 오래돼 서버가 거부하면(`Expired`), 다시 List부터
   시작한다. 이 "List → Watch → (끊김) → 다시 List" 루프가 캐시를 권위 있는 상태에 *결국 일치*시킨다.

> Reflector가 store에 쓸 때 쓰는 인터페이스는 `ReflectorStore`(`reflector.go:70`)로 좁혀져 있다
> (Add/Update/Delete/Replace/Resync).

### resourceVersion의 의미
`resourceVersion`(RV)은 [03](03-api-object-model.md)에서 본 `ObjectMeta` 필드로, etcd의 변경 순번에서
온다. watch는 "이 RV 이후"를 요청하므로, 컴포넌트가 잠깐 끊겼다 재연결해도 놓친 변경분만 받으면 된다.
RV가 너무 오래돼 etcd 히스토리에서 사라지면 서버가 거부하고, Reflector는 전체 relist로 복구한다.

## [2] DeltaFIFO — "객체별 변경 누적" 큐

Reflector가 밀어 넣는 곳은 일반 큐가 아니라 `DeltaFIFO`(`delta_fifo.go`)다. 주석이 그 목적을 명확히 한다:

> DeltaFIFO is a producer-consumer queue, where a Reflector is intended to be the producer, and the
> consumer is whatever calls the Pop() method.
> - You want to process every object change (delta) at most once.
> - When you process an object, you want to see everything that's happened to it since you last processed it.

즉 같은 객체에 대한 여러 변경(Added, Updated, Deleted)을 **객체별로 누적(Deltas)** 해 두고,
소비자가 `Pop()` 할 때 "지난번 처리 이후 그 객체에 일어난 일 전부"를 한 번에 본다. 그래서 변경을
빠짐없이, 그러나 객체당 최소 횟수로 처리할 수 있다.

## [3] HandleDeltas — 캐시 갱신 + 통지 분배

`Pop()` 된 Deltas는 `HandleDeltas`(`shared_informer.go`)가 처리한다. fifo 락을 쥔 채 각 Delta마다:

1. **Indexer(로컬 캐시) 갱신** — Add/Update/Delete를 캐시에 반영.
2. **sharedProcessor에 통지 전달** — 등록된 모든 이벤트 핸들러에 알린다.

### Indexer — 인덱싱된 로컬 캐시
`Indexer`(`index.go`, 구현 `thread_safe_store.go`)는 객체를 키(보통 `namespace/name`)로 저장하면서
**보조 인덱스**(예: 라벨, 노드 이름)도 유지한다. 컨트롤러는 `Lister`를 통해 이 캐시를 O(1)~O(인덱스)로
읽으므로 apiserver를 때리지 않는다.

> 캐시 안전성: 캐시의 객체는 공유되므로 직접 수정하면 안 된다. `cacheMutationDetector`
> (`shared_informer.go:597`)가 디버그 모드에서 캐시 변형을 잡아낸다. 그래서 컨트롤러는
> `DeepCopy()` 후 다룬다([03](03-api-object-model.md)의 `DeepCopyObject` 참조).

### sharedProcessor — 여러 컨트롤러가 하나의 watch를 공유
"Shared" Informer인 이유: 같은 리소스를 여러 컨트롤러가 봐야 할 때 watch 연결을 하나만 열고,
`sharedProcessor`가 그 통지를 **등록된 모든 핸들러**에 복제 전달한다. apiserver의 watch 부하를 줄인다.

## [4] Workqueue — reconcile로 넘기는 완충 장치

이벤트 핸들러는 보통 "변경된 객체의 키를 Workqueue에 넣는" 일만 한다. 실제 reconcile은 큐를 소비하는
워커가 한다. Workqueue(`client-go/util/workqueue/`)가 제공하는 것:

- **중복 제거**: 같은 키가 처리 대기 중이면 다시 안 쌓인다(`queue.go`).
- **지연 큐**: 일정 시간 뒤 처리(`delaying_queue.go`).
- **rate limiting + 재시도**: 실패한 키를 지수 백오프로 다시 큐에 넣는다
  (`rate_limiting_queue.go`, `default_rate_limiters.go`).

이 분리(이벤트 → 키 적재 → 워커가 reconcile) 덕분에 컨트롤러는 "지금 이 키의 객체가 원하는 상태인가"만
보면 되고, 어떤 이벤트가 왔는지는 신경 쓸 필요가 없다(level-triggered). 자세한 reconcile 패턴은
[05](05-controller-pattern.md).

## 일관성 보장 — "eventually consistent"

`SharedInformer` 주석(`shared_informer.go:45`)이 명시하듯, 캐시는 권위 있는 상태에 대해
**결국 일관(eventually consistent)** 하다: 통신 장애가 지속되지 않는 한, 객체 X가 상태 S에 있으면
informer 캐시도 결국 X를 S(또는 더 이후 상태)로 갖는다. 단 *서로 다른 객체 간의 순서는 보장하지 않는다.*

## 주요 디렉토리

| 경로 | 책임 |
|------|------|
| `client-go/tools/cache/reflector.go` | List+Watch 루프, RV 관리 |
| `client-go/tools/cache/delta_fifo.go` | 객체별 변경 누적 큐 |
| `client-go/tools/cache/shared_informer.go` | Informer 본체, HandleDeltas, sharedProcessor |
| `client-go/tools/cache/thread_safe_store.go`, `index.go` | Indexer(로컬 캐시) |
| `client-go/util/workqueue/` | 중복제거·재시도·rate limit 큐 |

## 더 읽을 곳
- [05-controller-pattern.md](05-controller-pattern.md) — 이 캐시/큐 위에서 도는 reconcile 루프
- [10-apiserver/06-etcd-storage-layer.md](../10-apiserver/) — watch가 apiserver→etcd에서 어떻게 구현되나
- [60-controller-runtime](../60-controller-runtime/) — 이 머시너리를 감싼 상위 프레임워크
