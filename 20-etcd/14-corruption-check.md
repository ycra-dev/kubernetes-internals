# 20.14 · 데이터 손상 감지

**근거**: `etcd/server/etcdserver/corrupt.go` (`corruptionChecker` `:45`, `PeerHashByRev` `:58`,
`newCorruptionChecker` `:63`), `etcd/server/storage/mvcc/hash.go`

etcd는 Kubernetes의 유일한 영속 상태다([10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md)). 만약
디스크 비트 부패나 버그로 **멤버 간 데이터가 달라지면**(silent corruption) 클러스터가 일관성을 잃는다 —
어떤 apiserver는 객체 X를, 다른 apiserver는 Y를 본다. etcd는 이를 **능동적으로 감지**한다.

## 문제 — 조용한 불일치

Raft는 "합의된 로그가 같은 순서로 적용됨"을 보장한다([01-raft.md](01-raft.md)). 하지만 적용 *이후*의
저장 계층(bbolt, [04-backend-bbolt.md](04-backend-bbolt.md))에서 디스크 손상/버그가 생기면, 같은 로그를
적용했는데도 멤버 간 실제 데이터가 달라질 수 있다. Raft는 이걸 못 잡는다 — 로그는 같으니까.

## 해시 기반 일관성 검사

`corrupt.go`의 `corruptionChecker`(`:45`)가 이를 감지한다. 핵심은 **각 멤버가 자기 데이터의 해시를 계산해
비교**하는 것:

- `mvcc/hash.go`가 특정 revision 시점까지의 KV 데이터 해시(`HashKV`)를 계산한다([03-mvcc.md](03-mvcc.md)의
  revision 기준).
- **같은 revision이면 모든 정상 멤버의 해시가 같아야 한다.** 다르면 누군가 손상됐다.

`PeerHashByRev`(`corrupt.go:58`)가 다른 멤버들에게 "이 revision의 해시를 알려달라"고 물어 비교한다.

## 두 가지 검사 시점

1. **시작 시 검사**(`CheckInitialHashKV` 류): 멤버가 합류/재시작할 때, 자기 해시가 기존 멤버들과 맞는지
   확인. 안 맞으면 손상된 데이터로 합류하지 않는다.
2. **주기적 검사**(periodic): 운영 중 일정 주기로 해시를 비교해 런타임 손상을 감지.

손상이 감지되면 etcd는 **alarm(CORRUPT)** 을 올리고([04-backend-bbolt.md](04-backend-bbolt.md)의 NOSPACE
alarm과 유사) 해당 멤버를 격리하거나 경고한다 — 손상된 데이터가 클라이언트에 응답되는 것을 막는다.

## 왜 Kubernetes에 중요한가

```
디스크 비트 부패/버그 → 멤버 A의 데이터 손상
   │  Raft는 못 잡음(로그는 동일)
   ▼
corruptionChecker: A의 해시 ≠ B,C의 해시 → CORRUPT alarm
   → A 격리 → 손상 데이터가 apiserver를 거쳐 클러스터에 퍼지는 것 차단
```

apiserver가 손상된 etcd 멤버에서 객체를 읽으면, 잘못된 상태로 컨트롤러가 동작해 연쇄 오류가 난다. 손상
감지가 이를 1차 차단한다. 손상된 멤버는 정상 멤버의 스냅샷으로 복구한다([02-wal-snapshot.md](02-wal-snapshot.md),
[08-operations.md](08-operations.md)).

## 운영

- `etcdctl endpoint hashkv`로 멤버 해시를 직접 비교할 수 있다.
- `--experimental-corrupt-check-time`(또는 안정화된 플래그)로 주기 검사 간격을 설정.
- alarm 발생 시 `etcdctl alarm list`로 확인, 손상 멤버를 교체/복구.

## 더 읽을 곳
- [03-mvcc.md](03-mvcc.md) — 해시 계산의 기준(revision)
- [04-backend-bbolt.md](04-backend-bbolt.md) — 손상이 일어나는 저장 계층
- [08-operations.md](08-operations.md) — 손상 멤버 복구
