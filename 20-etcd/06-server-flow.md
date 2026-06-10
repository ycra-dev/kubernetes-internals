# 20.06 · 서버 흐름 — 쓰기, apply 루프, 선형화 읽기

**근거**: `etcd/server/etcdserver/` (`v3_server.go`, `server.go`, `apply/`)

앞 문서들의 부품(Raft·WAL·MVCC·backend)을 `EtcdServer`가 어떻게 엮는지 본다. 핵심은 **쓰기는 Raft를
거치고, 읽기는 선형성을 보장하는 방식으로 처리**한다는 점이다.

## 쓰기 — 모두 Raft를 통한다

쓰기 연산(`Put`, `DeleteRange`, `Txn`, `LeaseGrant`...)은 전부 같은 패턴이다
(`v3_server.go`):

```go
// v3_server.go:295
func (s *EtcdServer) Put(ctx, r *pb.PutRequest) (*pb.PutResponse, error) {
    resp, err := s.raftRequest(ctx, &pb.InternalRaftRequest{Put: r})  // :303
    ...
}
```

`Range`(읽기, `:106`)를 제외한 거의 모든 연산이 `InternalRaftRequest`로 포장돼 `raftRequest`로 간다
(`Put` `:303`, `DeleteRange` `:318`, `Txn` `:375`, `LeaseGrant` `:451`, `Alarm` `:805`...). 즉:

1. 연산을 `InternalRaftRequest`로 직렬화해 Raft에 **propose**.
2. Raft가 로그로 복제·commit([01-raft.md](01-raft.md)).
3. apply 루프가 commit된 엔트리를 상태기계에 적용(아래).
4. 적용 결과를 propose한 고루틴에 돌려줘 클라이언트에 응답.

`processInternalRaftRequestOnce`가 propose와 "적용 완료 통지" 사이를 잇는다(요청마다 고유 ID로 결과를
기다림).

## apply 루프 — commit을 상태로

`server.go`의 apply 경로(`applyAll`, `server.go:972`)는 Raft가 넘긴 committed 엔트리를 **순서대로**
상태기계에 적용한다:

- **applier**(`server/etcdserver/apply/apply.go`, `uber_applier.go`): 엔트리 종류별로 분기해 MVCC store
  ([03-mvcc.md](03-mvcc.md))에 Put/Delete/Txn을 수행하고 새 revision을 매긴다. auth/lease/quota도 여기서
  적용(`apply/auth.go`, `apply/quota.go`).
- **applySnapshot**(`server.go:995`): 너무 뒤처진 노드가 리더에게 받은 스냅샷을 통째로 적용해 따라잡기.

모든 노드가 **같은 순서**로 apply하므로 결국 같은 상태에 도달한다. 이것이 복제의 일관성을 만든다.

## 읽기 — 선형화(linearizable) vs 직렬화(serializable)

읽기는 두 종류다:

- **선형화 읽기(기본)**: "이 읽기 시점까지 commit된 모든 쓰기를 반영"함을 보장. 단순히 로컬에서 읽으면
  팔로워가 뒤처져 옛 값을 줄 수 있다. 그래서 etcd는 **ReadIndex** 기법을 쓴다: 읽기 전에 리더에게
  "지금 commit 인덱스"를 확인하고, 로컬 apply가 그 인덱스까지 따라잡은 뒤 읽는다. Raft 합의 라운드 없이도
  최신성을 보장한다.
- **직렬화 읽기(`WithSerializable`)**: 로컬 상태에서 즉시 읽음. 빠르지만 살짝 옛 값일 수 있다.

Kubernetes apiserver는 일관성이 중요한 읽기에 선형화 읽기를, 캐시 가능한 list에는 상황에 따라 다른
일관성 모드를 쓴다([10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md)의 cacher).

## Txn — 조건부 트랜잭션

`Txn`(`v3_server.go:325`)은 "compare → success ops / failure ops"의 원자적 조건 실행이다. 예:
"키 foo의 mod revision이 R이면 PUT, 아니면 실패". 이것이 **낙관적 동시성**의 etcd 측 메커니즘이고,
apiserver의 `GuaranteedUpdate`([10-apiserver/05](../10-apiserver/05-registry-storage.md))가 이 위에서
구현된다.

## 더 읽을 곳
- [01-raft.md](01-raft.md) — propose→commit
- [03-mvcc.md](03-mvcc.md) — apply가 쓰는 store
- [07-api-client.md](07-api-client.md) — 이 연산들의 gRPC API
