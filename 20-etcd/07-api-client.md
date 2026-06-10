# 20.07 · v3 API와 클라이언트

**근거**: `etcd/api/etcdserverpb/` (`rpc.proto`), `etcd/client/v3/`

etcd v3는 **gRPC API**다. 스키마는 `etcd/api/etcdserverpb/rpc.proto`에 정의되고, 코드 생성으로
`rpc.pb.go`/`rpc_grpc.pb.go`가 만들어진다. apiserver는 이 gRPC API의 클라이언트다.

## 서비스 묶음

`rpc.proto`는 연산을 서비스로 묶는다:

| 서비스 | 연산 | 본 문서 |
|--------|------|---------|
| **KV** | `Put`, `Range`, `DeleteRange`, `Txn`, `Compact` | [06-server-flow.md](06-server-flow.md) |
| **Watch** | `Watch`(양방향 스트림) | [05-lease-watch.md](05-lease-watch.md) |
| **Lease** | `LeaseGrant`, `LeaseRevoke`, `LeaseKeepAlive` | [05-lease-watch.md](05-lease-watch.md) |
| **Auth** | 사용자/역할/권한 | - |
| **Cluster** | 멤버 추가/제거/목록 | [01-raft.md](01-raft.md) |
| **Maintenance** | `Status`, `Defragment`, `Snapshot`, `Alarm` | [08-operations.md](08-operations.md) |

키는 바이트열이고, **prefix(접두사) 조회**가 1급 기능이다(`Range`에 range_end 지정). Kubernetes가
`/registry/pods/<ns>/` 같은 계층 키를 쓰고 "네임스페이스 내 모든 Pod"를 prefix로 list 하는 것이 이
기능이다.

## 데이터 모델 메시지

- `mvccpb`(`api/mvccpb/`): `KeyValue`(키, 값, **create_revision**, **mod_revision**, version, lease)와
  `Event`(PUT/DELETE). mod_revision이 [03-mvcc.md](03-mvcc.md)의 revision이며 apiserver의
  resourceVersion이 된다.
- `authpb`, `membershippb`: 인증/멤버십 데이터.

## Go 클라이언트 — client/v3

`etcd/client/v3/`가 공식 Go 클라이언트다. 기능별 파일:

- `kv.go` — Put/Get/Delete/Txn/Compact.
- `watch.go` — watch 스트림(자동 재연결, 마지막 revision부터 재개).
- `lease.go` — lease grant/keepalive.
- `txn.go` — fluent Txn 빌더(`If(...).Then(...).Else(...)`).
- `concurrency/` — 이 위에 구현된 **분산 락/선출** 유틸(Mutex, Election). lease+watch+Txn의 조합으로
  만들어진다.

apiserver의 etcd3 store([10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md))가 바로 이
`client/v3`를 사용한다.

## 엔드포인트와 장애 조치

클라이언트는 여러 etcd 엔드포인트를 받아, 하나가 죽으면 다른 멤버로 재연결한다. 쓰기는 어느 멤버로
보내도 내부적으로 리더에게 전달돼 처리된다([06-server-flow.md](06-server-flow.md)).

## 더 읽을 곳
- [10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md) — 이 클라이언트의 가장 중요한 사용자
- [08-operations.md](08-operations.md) — Maintenance API 운영
