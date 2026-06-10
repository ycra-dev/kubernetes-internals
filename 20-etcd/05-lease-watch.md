# 20.05 · Lease와 Watch

**근거**: `etcd/server/lease/` (`lessor.go`), `etcd/server/storage/mvcc/` (watch)

etcd가 단순 KV를 넘어 분산 시스템의 조율 도구가 되게 하는 두 기능: **lease**(TTL 기반 만료)와
**watch**(변경 스트리밍).

## Lease — TTL을 가진 키

**lease**는 만료 시간(TTL)을 가진 객체다. 키를 lease에 묶으면, lease가 만료될 때 그 키들이 **자동
삭제**된다.

`server/lease/lessor.go:145`의 `lessor`가 lease를 관리한다:

- **Grant**: TTL을 가진 새 lease 생성.
- **KeepAlive**: 클라이언트가 주기적으로 갱신해 만료를 미룬다. 갱신이 끊기면 만료.
- **Revoke**: 만료/명시적 취소 시 lease에 묶인 모든 키를 삭제.

> lease 만료/취소도 **Raft를 거친다**(`v3_server.go`의 `LeaseGrant`/`LeaseRevoke`,
> [06-server-flow.md](06-server-flow.md)) — 모든 멤버가 같은 시점에 같은 키를 지우도록.
> TTL 잔여 시간은 **checkpoint**(`lessor.go:54` 부근)로 합의 로그에 기록돼, 리더가 바뀌어도 만료
> 타이밍이 유지된다.

### Kubernetes에서의 쓰임
- **Event** 객체처럼 일시적 데이터를 TTL로 자동 정리(apiserver의 `lease_manager`가 etcd lease를 재사용,
  [10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md)).
- 단, Kubernetes의 **노드 하트비트 Lease**나 **리더 선출 Lease**는 *Kubernetes API의 Lease 리소스*이지
  etcd lease가 아니다 — 이름이 같지만 다른 계층이다([00-foundations/06](../00-foundations/06-leader-election.md)).

## Watch — revision 기반 변경 스트림

**watch**는 "어떤 키(또는 prefix)에 대해, 주어진 revision 이후의 모든 변경"을 순서대로 받는 스트림이다.
[03-mvcc.md](03-mvcc.md)의 revision이 이것의 기반이다.

- 클라이언트는 `watch(key, startRevision)`을 연다.
- etcd는 그 revision부터 현재까지의 변경을 보내고, 이후 새 변경을 실시간으로 push한다.
- 연결이 끊겨도 마지막으로 받은 revision+1부터 다시 watch하면 **놓친 변경만** 받는다.

### 왜 이게 Kubernetes의 핵심인가
apiserver의 watch cacher가 etcd watch **하나**를 열고, 그것을 수많은 클라이언트 Informer로 팬아웃한다
([10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md),
[00-foundations/04](../00-foundations/04-list-watch-informer.md)). 즉:

```
etcd watch (revision 기반)
   └─► apiserver cacher (revision = resourceVersion)
         └─► 수많은 client-go Informer (watch "이 RV 이후")
               └─► 컨트롤러 reconcile
```

watch의 startRevision이 너무 오래돼 compaction됐으면([03-mvcc.md](03-mvcc.md)) etcd가 거부하고,
상위에서 relist로 복구한다.

## 더 읽을 곳
- [03-mvcc.md](03-mvcc.md) — watch가 의존하는 revision
- [00-foundations/06](../00-foundations/06-leader-election.md) — Kubernetes Lease(다른 계층)
