# 20 · etcd

**근거 레포**: `etcd` (`go.etcd.io/etcd/v3`), raft는 별도 모듈 `go.etcd.io/raft/v3`

etcd는 **분산 키-값 저장소**이고, Kubernetes의 **단일 진실 공급원**이다. 모든 클러스터 상태(Pod,
Service, Secret...)가 여기 저장되며, 오직 apiserver만 접근한다([10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md)).
etcd가 제공하는 두 가지 핵심 보장이 Kubernetes를 떠받친다:

1. **합의(consensus)**: 여러 노드에 복제돼도 모두가 같은 데이터에 동의한다(Raft). 일부 노드가 죽어도
   과반이 살아 있으면 동작.
2. **MVCC + watch**: 모든 변경에 단조 증가 **revision**을 매기고, "이 revision 이후의 변경"을 watch로
   스트리밍한다. 이것이 apiserver/Informer의 list/watch를 가능케 한다
   ([00-foundations/04](../00-foundations/04-list-watch-informer.md)).

## 쓰기 1건의 전체 여정

이 폴더의 나머지 문서가 각 단계를 깊게 판다. 먼저 전체를 꿴다 — `Put` 한 번이 디스크에 안전히
박히기까지:

```
클라이언트 ──gRPC Put──► EtcdServer.Put (v3_server.go:295)
   │
[1] raftRequest: InternalRaftRequest{Put} 으로 포장해 Raft에 제안(propose)
   │                                              → [01-raft.md]
[2] Raft: 리더가 로그 엔트리로 만들어 팔로워에 복제, 과반 확인되면 "committed"
   │       (복제와 동시에 각 노드는 WAL에 먼저 기록)  → [02-wal-snapshot.md]
[3] applyAll (server.go:972): committed 엔트리를 상태기계에 적용
   │                                              → [06-server-flow.md]
[4] MVCC store: 새 revision 을 매겨 키-값을 기록    → [03-mvcc.md]
[5] backend(bbolt): B+tree 디스크 트랜잭션으로 영속화 → [04-backend-bbolt.md]
[6] watch: 이 변경을 구독자(apiserver cacher)에게 통지 → [05-lease-watch.md]
   │
응답 ◄── 적용된 revision 을 담아 클라이언트로
```

핵심 통찰: **쓰기는 "Raft 로그에 commit"과 "상태기계에 apply"의 두 단계**다. Raft는 *순서에 대한 합의*만
하고, 그 순서대로 각 노드가 독립적으로 MVCC에 적용한다. 모두 같은 순서로 적용하므로 결국 같은 상태가 된다.

## 문서

| # | 문서 | 내용 | 근거 경로 |
|---|------|------|-----------|
| 01 | [raft.md](01-raft.md) | 합의: 리더 선출·로그 복제·멤버십 | `etcd/server/etcdserver/raft.go`, `raft/v3` |
| 02 | [wal-snapshot.md](02-wal-snapshot.md) | 내구성: WAL·스냅샷 | `server/storage/wal/` |
| 03 | [mvcc.md](03-mvcc.md) | 다중버전·revision·compaction | `server/storage/mvcc/` |
| 04 | [backend-bbolt.md](04-backend-bbolt.md) | bbolt B+tree 백엔드·트랜잭션 | `server/storage/backend/` |
| 05 | [lease-watch.md](05-lease-watch.md) | lease(TTL)·watch 스트림 | `server/lease/`, `server/storage/mvcc/` |
| 06 | [server-flow.md](06-server-flow.md) | apply 루프·선형화 읽기·txn | `server/etcdserver/` |
| 07 | [api-client.md](07-api-client.md) | v3 gRPC API·client | `etcd/api/`, `etcd/client/` |
| 08 | [operations.md](08-operations.md) | etcdctl/etcdutl·백업·복구 | `etcd/etcdctl/`, `etcd/etcdutl/` |
| 09 | [auth-membership.md](09-auth-membership.md) | etcd 인증/RBAC·learner·gRPC 프록시 | `etcd/server/auth/`, `.../membership/` |
| 10 | [raft-ready-loop.md](10-raft-ready-loop.md) | Raft Ready 루프 코드 레벨(합의→WAL→apply→Advance) | `etcd/server/etcdserver/raft.go` |
| 11 | [mvcc-index.md](11-mvcc-index.md) | treeIndex/keyIndex/generation, compaction 내부 | `etcd/server/storage/mvcc/` |
| 12 | [lease-deep.md](12-lease-deep.md) | lessor 내부(만료/checkpoint/Revoke) | `etcd/server/lease/lessor.go` |
| 13 | [bootstrap-embed.md](13-bootstrap-embed.md) | 부팅/복원(embed/bootstrap), 클러스터 형성 | `etcd/server/embed/`, `.../bootstrap.go` |
| 14 | [corruption-check.md](14-corruption-check.md) | 해시 기반 데이터 손상 감지 | `etcd/server/etcdserver/corrupt.go` |
| 15 | [watch-impl.md](15-watch-impl.md) | watchableStore(synced/unsynced/victim) | `etcd/server/storage/mvcc/watchable_store.go` |

## 진입점
- 바이너리: `etcd/server/main.go` → `server/etcdmain/`
- 임베드 가능 서버: `server/embed/`
- 핵심 서버: `server/etcdserver/server.go`, `v3_server.go`

## 더 읽을 곳
- [10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md) — apiserver가 etcd를 쓰는 법
- [00-foundations/06](../00-foundations/06-leader-election.md) — Kubernetes 리더 선출(etcd lease와는 다른 계층)
