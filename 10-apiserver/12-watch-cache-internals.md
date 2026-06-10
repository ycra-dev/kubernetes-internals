# 10.12 · Watch 캐시 내부 (cacher)

**근거**: `kubernetes/staging/src/k8s.io/apiserver/pkg/storage/cacher/`
(`cacher.go` `Watch` `:513`, `dispatchEvents` `:874`, `dispatchEvent` `:966`, `blockedWatchers` `:332`,
`cache_watcher.go`)

[06-etcd-storage-layer.md](06-etcd-storage-layer.md)에서 cacher가 "etcd watch 하나를 수많은 클라이언트로
팬아웃한다"고 했다. 이 문서는 그 내부 — **watchCache(공유 버퍼) → dispatch → 클라이언트별 cacheWatcher**
— 를 코드로 본다. 수천 개의 Informer가 apiserver에 watch를 걸어도 etcd가 멀쩡한 비결이다.

## 구조: 하나의 watchCache, 다수의 cacheWatcher

```
etcd watch (리소스당 1개)
   │
watchCache: 최근 이벤트들을 RV 순서로 담은 순환 버퍼(공유)
   │  dispatchEvents (cacher.go:874)
   ├─► cacheWatcher #1 (클라이언트 A의 Informer)
   ├─► cacheWatcher #2 (클라이언트 B)
   └─► ... 수천 개
```

- **watchCache**: etcd에서 받은 변경을 RV 순서로 보관하는 **공유 순환 버퍼**. List/Watch 요청 다수를
  여기서 응답해 etcd 접근을 없앤다.
- **cacheWatcher**(`cache_watcher.go:53`): **클라이언트 하나당** 생성되는 watcher. 자기 입력 채널과
  버퍼를 갖는다.

## Watch 시작 — 과거 RV부터 재생

`Cacher.Watch`(`cacher.go:513`)는 클라이언트가 준 `resourceVersion`을 본다:

- 클라이언트가 "RV=X 이후"를 요청하면, cacher는 watchCache 버퍼에서 **X 이후의 이벤트를 먼저 재생**한
  뒤, 이후 새 이벤트를 실시간 전달한다.
- 그래서 Informer가 끊겼다 재연결해도 놓친 변경만 받는다([00-foundations/04](../00-foundations/04-list-watch-informer.md)).
- 요청한 RV가 버퍼보다 오래됐으면(버퍼는 유한) → `Expired` → 클라이언트는 전체 relist로 복구
  ([20-etcd/03](../20-etcd/03-mvcc.md)의 compaction과 같은 결과).

## dispatch — 한 이벤트를 모두에게

새 이벤트가 오면 `dispatchEvents`(`cacher.go:874`) → `dispatchEvent`(`:966`)가 **모든 cacheWatcher**에
복제 전달한다. 각 watcher는 자기 셀렉터(네임스페이스/라벨/필드)로 필터링해 관심 있는 것만 클라이언트에
보낸다.

### 느린 클라이언트 처리 — blockedWatchers
문제: 한 클라이언트가 느리면(네트워크 지연 등) 그 watcher의 버퍼가 차서 전체 dispatch가 막힐 수 있다.
cacher는 이를 `blockedWatchers`(`cacher.go:332`)로 다룬다:

- 버퍼가 가득 찬 watcher를 blocked 목록에 모아두고(`:999`),
- 논블로킹으로 먼저 보낸 뒤, 막힌 것들만 따로 재시도(`:1003`~`:1016`).
- 끝내 못 따라오는 watcher는 **연결을 끊는다** — 한 느린 클라이언트가 전체를 막지 못하게. 끊긴
  클라이언트는 relist로 복구한다.

이 격리가 APF([10-priority-fairness.md](10-priority-fairness.md))와 같은 정신 — 하나의 나쁜 시민이
전체를 망치지 못하게 한다.

## Bookmark — "여기까지 봤다" 신호

`cache_watcher.go:39~81`의 bookmark 로직. 문제: 어떤 네임스페이스에 변화가 오래 없으면, 그 watch의 RV가
정체돼 클라이언트가 가진 RV가 낡는다. 나중에 재연결 시 그 낡은 RV가 만료돼 비싼 relist가 필요해진다.

**bookmark 이벤트**는 변화가 없어도 주기적으로 "현재 RV는 여기까지"를 알려주는 빈 신호다:

- 클라이언트의 RV를 최신으로 유지해, 재연결 시 relist를 피한다.
- `bookmarkAfterResourceVersion`(`cache_watcher.go:81`)으로 "특정 RV 이후 bookmark를 보내라"는 요청도
  지원(예: 정확한 시점 동기화).

## 일관성 — consistency/progress

cacher는 List가 "충분히 최신인지"를 보장해야 한다(클라이언트가 RV를 지정하면 그 RV까지 반영된 결과를
줘야 함). `cacher/progress/`, `cacher/consistency/`, watchCache의 `waitUntilFresh` 류 메커니즘이
"캐시가 요청 RV까지 따라잡을 때까지 대기"해 일관성을 지킨다. 너무 최신을 요구하면 잠깐 블록될 수 있다.

## 정리

```
etcd watch 1개
   → watchCache(공유 버퍼, RV 순서)
   → dispatch → cacheWatcher 수천 개(클라이언트별, 셀렉터 필터)
       - 느리면 blockedWatchers → 끝내 못 따라오면 끊고 relist
       - 조용하면 bookmark로 RV 최신 유지
```

이 구조 덕분에 apiserver는 etcd watch 하나로 수만 개의 Informer를 먹인다.

## 더 읽을 곳
- [06-etcd-storage-layer.md](06-etcd-storage-layer.md) — cacher의 상위 개요
- [00-foundations/04](../00-foundations/04-list-watch-informer.md) — 클라이언트 쪽 소비
- [20-etcd/05](../20-etcd/05-lease-watch.md) — etcd watch(상류)
