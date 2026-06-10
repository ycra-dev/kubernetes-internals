# 20.03 · MVCC — 다중 버전과 revision

**근거**: `etcd/server/storage/mvcc/` (`kvstore.go`, `index.go`, `key_index.go`)

etcd는 키를 덮어쓰지 않고 **버전을 쌓는다(Multi-Version Concurrency Control)**. 모든 변경에 단조 증가하는
**revision**을 매겨, "과거 시점의 상태"와 "이 시점 이후의 변경"을 조회할 수 있다. 이 revision이 곧
Kubernetes의 `resourceVersion`이다([10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md)).

## revision — 전역 논리 시계

`mvcc.store`(`kvstore.go:53`)는 두 핵심 값을 관리한다:

```go
// kvstore.go:71
currentRev int64      // 마지막으로 완료된 트랜잭션의 revision
compactMainRev int64  // 마지막 compaction의 main revision
```

- 쓰기 트랜잭션마다 `currentRev`가 1 증가한다(초기값 1, `kvstore.go:104`).
- revision은 사실 `(main, sub)` 쌍이다: `main`은 트랜잭션 번호, `sub`는 한 트랜잭션 안의 연산 순번.
  한 Txn에서 여러 키를 바꾸면 같은 main에 sub가 늘어난다.

즉 revision은 **클러스터 전체에 단일한, 절대 되돌아가지 않는 시계**다. 두 노드가 같은 순서로 적용하므로
같은 revision에서 같은 상태를 본다([01-raft.md](01-raft.md)).

## 두 개의 색인

MVCC는 두 단계로 키를 찾는다:

1. **인메모리 B-tree 색인**(`index.go`, `key_index.go`): `키 → 그 키의 revision들` 매핑. "키 foo의
   revision 5에서의 값은 어느 backend revision에 있나"를 빠르게 찾는다.
2. **백엔드(bbolt)**: 실제 값은 `revision → 값`으로 bbolt에 저장된다([04-backend-bbolt.md](04-backend-bbolt.md)).

키로 읽으면: 인메모리 색인에서 해당 revision을 찾고 → 그 revision으로 bbolt에서 값을 꺼낸다.

## 과거 읽기와 watch의 기반

revision이 보존되므로:

- **과거 조회**: "revision R 시점의 foo 값"을 읽을 수 있다(Range with revision).
- **watch**: "revision R 이후 foo에 일어난 모든 변경"을 순서대로 받을 수 있다 →
  [05-lease-watch.md](05-lease-watch.md). 이것이 apiserver/Informer가 끊겼다 재연결해도 놓친 변경만
  받는 비결이다([00-foundations/04](../00-foundations/04-list-watch-informer.md)).

## Compaction — 오래된 버전 청소

버전을 영원히 쌓으면 디스크가 무한히 커진다. **compaction**은 특정 revision 이전의 **오래된 버전을
제거**한다(`kvstore_compaction.go`, `compactMainRev`). compaction 후에는 그 revision 이전을 watch/조회할
수 없다 — etcd가 `mvcc: required revision has been compacted` 오류를 낸다.

> 이 오류가 apiserver/Informer에 닿으면 "RV too old"가 되어 **전체 relist**로 복구된다
> ([00-foundations/04](../00-foundations/04-list-watch-informer.md)의 Reflector). 즉 etcd compaction과
> Kubernetes의 relist는 직접 연결돼 있다.

Kubernetes는 보통 apiserver의 compaction 옵션으로 etcd를 주기적으로 compact하고, 그 뒤 **defrag**으로
디스크를 실제로 회수한다([08-operations.md](08-operations.md)).

## 더 읽을 곳
- [04-backend-bbolt.md](04-backend-bbolt.md) — revision→값이 실제로 저장되는 곳
- [05-lease-watch.md](05-lease-watch.md) — revision 기반 watch
