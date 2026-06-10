# 10.16 · 읽기 일관성 의미론 (resourceVersion)

**근거**: `kubernetes/staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/types.go`
(`ResourceVersionMatch` `:504`, `NotOlderThan` `:509`, `Exact` `:512`)

[06-etcd-storage-layer.md](06-etcd-storage-layer.md), [12-watch-cache-internals.md](12-watch-cache-internals.md)에서
cacher가 List/Watch를 캐시에서 응답한다고 했다. 그런데 "얼마나 최신인 데이터를 받는가"는 클라이언트가
**resourceVersion 옵션**으로 정한다. 이 의미론은 캐시와 etcd 중 어디서 읽을지, 얼마나 기다릴지를 결정해
성능과 정확성의 트레이드오프를 만든다.

## List의 RV 옵션

List 요청은 `resourceVersion`과 `resourceVersionMatch`(`types.go:504`)로 일관성 수준을 고른다:

| 요청 | 의미 | 어디서 읽나 |
|------|------|-------------|
| RV 미지정 (또는 빈값) | **가장 최신**(consistent read) | etcd(또는 cacher가 최신 보장) |
| `RV="0"` | "아무거나, 캐시면 됨"(가장 빠름) | cacher 캐시(약간 옛 데이터 가능) |
| `RV=X`, match=`NotOlderThan` (`:509`) | "X 이상으로 최신" | cacher가 X까지 따라잡으면 캐시 |
| `RV=X`, match=`Exact` (`:512`) | "정확히 X 시점" | etcd(과거 시점, [20-etcd/03](../20-etcd/03-mvcc.md)) |

핵심 트레이드오프:
- **consistent read**(RV 미지정): 항상 최신이지만 etcd를 거쳐(또는 cacher가 최신성을 보장하느라 대기)
  비싸다.
- **RV="0"**: 캐시에서 즉시 응답, 빠르지만 살짝 옛 데이터일 수 있다. Informer 초기 List가 이걸 쓸 수
  있다([00-foundations/04](../00-foundations/04-list-watch-informer.md)).

## 왜 이게 부하에 중요한가

대규모 클러스터에서 **모든 List가 consistent read면 etcd가 폭주**한다. 그래서 캐시 가능한 List는 RV="0"
또는 NotOlderThan으로 cacher에서 응답한다([12-watch-cache-internals.md](12-watch-cache-internals.md)).
컨트롤러는 보통 캐시 일관성으로 충분하다(level-triggered라 약간 옛 데이터도 다음 sync에서 수렴,
[00-foundations/02](../00-foundations/02-architecture.md)).

반대로 "방금 쓴 것을 반드시 읽어야"(read-after-write)하면 consistent read가 필요하다. `GuaranteedUpdate`
([05-registry-storage.md](05-registry-storage.md))의 내부 읽기가 그렇다.

## consistent read도 캐시로 — progress notify

과거엔 consistent read가 무조건 etcd를 거쳤다. 근래엔 cacher가 etcd의 **progress notify**로 "내 캐시가
이 RV까지 최신"임을 알면, consistent read도 캐시에서 응답할 수 있다(`cacher/progress/`,
[12-watch-cache-internals.md](12-watch-cache-internals.md)). 이로써 정확성을 지키면서 etcd 부하를 더 줄인다.

## Watch의 RV

Watch도 RV로 시작점을 정한다([20-etcd/05](../20-etcd/05-lease-watch.md)):
- `RV=X`: "X 이후의 변경만". 재연결 시 놓친 것만 받음.
- RV 미지정/`RV="0"`: 현재 상태부터(cacher가 초기 상태 + 이후 이벤트).
- 너무 오래된 X: `Expired` → relist([00-foundations/04](../00-foundations/04-list-watch-informer.md)).

## Streaming List (WatchList)

거대한 List(수만 객체)는 한 번에 받으면 apiserver 메모리와 응답 지연이 크다. **WatchList**(streaming list)는
List를 **watch 스트림처럼** 청크로 흘려보낸다 — Informer가 초기 상태를 watch 프레임으로 받아 메모리
스파이크를 줄인다. cacher의 watch 머시너리([12-watch-cache-internals.md](12-watch-cache-internals.md))를
재사용해, "초기 List + 이후 watch"를 하나의 스트림으로 통합한다.

## 정리

```
"최신이 꼭 필요" → consistent read(RV 미지정) → etcd 또는 progress-notify 캐시
"빠른 게 좋아"   → RV="0" → cacher 캐시 (약간 옛 데이터)
"이 시점 이상"   → NotOlderThan → 캐시가 따라잡으면 캐시
"정확히 그 시점" → Exact → etcd 과거 revision
거대 List        → WatchList(스트리밍)로 메모리 절감
```

이 손잡이들이 "정확성 vs etcd 부하 vs 지연"을 클라이언트가 상황에 맞게 고르게 한다.

## 더 읽을 곳
- [12-watch-cache-internals.md](12-watch-cache-internals.md) — cacher가 RV를 처리하는 법
- [20-etcd/03](../20-etcd/03-mvcc.md) — RV=etcd revision
- [00-foundations/04](../00-foundations/04-list-watch-informer.md) — Informer의 List/Watch
