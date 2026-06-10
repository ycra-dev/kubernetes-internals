# 20.15 · Watch 구현 (watchableStore)

**근거**: `etcd/server/storage/mvcc/watchable_store.go`
(`watchableStore` `:56`, `unsynced` `:68`, `synced`, `victims` `:64`)

[05-lease-watch.md](05-lease-watch.md)에서 watch를 "revision 이후 변경 스트림"으로 개요했다. 이 문서는
etcd가 **수많은 watcher를 효율적으로 먹이는** 내부 구조 — synced/unsynced 분리와 victim 처리 — 를 본다.
apiserver cacher([10-apiserver/12](../10-apiserver/12-watch-cache-internals.md))가 의존하는 바로 그 etcd
watch의 밑바닥이다.

## watchableStore — MVCC 위의 watch 계층

`watchable_store.go:56`의 `watchableStore`는 MVCC store([03-mvcc.md](03-mvcc.md))를 감싸, 변경이 일어날
때 관심 있는 watcher에게 통지한다. 핵심은 watcher를 **두 그룹**으로 나누는 것이다.

## synced vs unsynced — 따라잡았나 아닌가

| 그룹 | 의미 | 코드 |
|------|------|------|
| **synced** | 현재 revision까지 따라잡은 watcher | `synced watcherGroup` |
| **unsynced** | 과거 revision부터 시작해 아직 따라잡는 중 | `unsynced` (`watchable_store.go:68`) |

### synced watcher
이미 최신인 watcher다. 새 쓰기가 일어나면([06-server-flow.md](06-server-flow.md)의 apply), 그 변경
이벤트를 synced 그룹 중 **그 키에 관심 있는** watcher에게 즉시 보낸다. 빠른 경로.

### unsynced watcher
클라이언트가 "RV=과거시점 이후"로 watch를 열면, 그 시점부터 현재까지의 변경을 따라잡아야 한다. 이런
watcher는 unsynced 그룹에 들어가고, 백그라운드 동기 루프(`syncWatchers`)가 MVCC에서 과거 이벤트를 읽어
보내준다. 다 따라잡으면 **synced 그룹으로 이동**한다.

```
watch(RV=과거) → unsynced 그룹
   syncWatchers: MVCC에서 과거~현재 이벤트 읽어 전송
   따라잡음 → synced 그룹으로 이동 → 이후 실시간 통지
```

이 분리 덕분에, 대부분(synced) watcher는 새 이벤트만 빠르게 받고, 따라잡아야 하는 소수(unsynced)만
무거운 과거 조회를 한다.

## victim — 느린 watcher 처리

watcher의 전송 채널이 가득 차면(클라이언트가 느림) 통지를 못 보낸다. 그런 watcher 배치를 **victims**
(`watchable_store.go:64`)에 모아둔다:

- 채널이 막힌 watcher의 이벤트를 victim 버퍼에 보관.
- 별도 루프(`victimc` 신호)가 나중에 재전송 시도.
- 이로써 한 느린 watcher가 전체 통지 루프를 막지 못하게 한다.

> 이는 apiserver cacher의 `blockedWatchers`([10-apiserver/12](../10-apiserver/12-watch-cache-internals.md))와
> **같은 문제·같은 해법**이다 — 한 계층 아래에서 etcd가 똑같이 "느린 소비자 격리"를 한다. watch 팬아웃이
> etcd(victim) → apiserver cacher(blocked) → 클라이언트 Informer로 이어지며 각 계층이 같은 패턴을 쓴다.

## 전체 그림 — watch 팬아웃의 밑바닥

```
쓰기(apply, 06-server-flow) → watchableStore
   ├─ synced watcher: 즉시 통지 (빠름)
   ├─ unsynced watcher: syncWatchers가 과거 따라잡기
   └─ 느린 watcher: victims에 보관 후 재시도
        │
   etcd watch (revision 기반)
        → apiserver cacher (10-apiserver/12)
        → 수많은 client-go Informer (00-foundations/04)
```

apiserver cacher가 etcd에 watch **하나**를 걸므로, 보통 etcd watcher 수는 적다(리소스 종류 수준). 진짜
팬아웃은 cacher가 한다. 하지만 그 한 watcher가 정확·효율적으로 동작하는 것이 이 watchableStore다.

## 더 읽을 곳
- [05-lease-watch.md](05-lease-watch.md) — watch 개요
- [03-mvcc.md](03-mvcc.md) — unsynced가 읽는 과거 이벤트(revision)
- [10-apiserver/12](../10-apiserver/12-watch-cache-internals.md) — 위 계층(cacher)의 같은 패턴
