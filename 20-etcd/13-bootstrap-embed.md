# 20.13 · etcd 부팅 (embed / bootstrap)

**근거**: `etcd/server/embed/etcd.go`(`Etcd` `:70`, `StartEtcd` `:110`),
`etcd/server/etcdserver/bootstrap.go`(`bootstrap` `:53`), `etcd/server/etcdmain/`

이 폴더의 다른 문서들이 "도는 etcd"를 설명했다면, 이 문서는 **etcd가 어떻게 시작하는가** — 설정 →
스토리지 복원 → Raft 초기화 → 서빙 — 를 본다. 재시작 시 데이터를 어떻게 복구하는지가 여기 모인다.

## 두 진입점: etcdmain과 embed

| 방식 | 위치 | 용도 |
|------|------|------|
| **etcdmain** | `server/etcdmain/` | 독립 `etcd` 바이너리(`etcd/server/main.go`) |
| **embed** | `server/embed/` | 다른 Go 프로그램에 etcd를 **내장** |

`embed`의 `StartEtcd(cfg)`(`embed/etcd.go:110`)가 핵심 — `Etcd` 구조체(`:70`)를 만들어 모든 하위 시스템을
조립한다. 독립 바이너리도 내부적으로 이 embed 경로를 쓴다. (Kubernetes는 보통 독립 etcd를 static pod로
띄운다, [15-cli/01](../15-cli/01-kubeadm.md).)

## bootstrap — 시작 시 복원

`etcdserver/bootstrap.go:53`의 `bootstrap(cfg)`가 서버를 세운다. 핵심은 **기존 데이터가 있으면 복원,
없으면 새로 시작**이다:

```
bootstrap (bootstrap.go:53):
  데이터 디렉토리에 WAL/스냅샷이 있나?
  ├─ 있음(재시작/재합류): 
  │    최신 스냅샷 로드 → WAL 재생 → 마지막 committed 상태 복원 (02-wal-snapshot)
  │    → 그 상태로 Raft 노드 재개
  └─ 없음(최초 시작):
       클러스터 멤버 구성 초기화 → 빈 Raft 노드 시작
```

이것이 [02-wal-snapshot.md](02-wal-snapshot.md)의 "재시작 복구"가 실제로 일어나는 지점이다 —
스냅샷+WAL로 죽기 직전 상태를 정확히 되살린다.

## 조립되는 하위 시스템

bootstrap이 세우는 것들(앞 문서들의 부품):

| 부품 | 문서 |
|------|------|
| backend(bbolt) | [04-backend-bbolt.md](04-backend-bbolt.md) |
| MVCC store + 인덱스 | [03-mvcc.md](03-mvcc.md), [11-mvcc-index.md](11-mvcc-index.md) |
| WAL | [02-wal-snapshot.md](02-wal-snapshot.md) |
| Raft 노드 | [01-raft.md](01-raft.md), [10-raft-ready-loop.md](10-raft-ready-loop.md) |
| lessor | [05-lease-watch.md](05-lease-watch.md), [12-lease-deep.md](12-lease-deep.md) |
| auth | [09-auth-membership.md](09-auth-membership.md) |

`Close`(`bootstrap.go:159`/`:168`)가 역순으로 정리한다(graceful shutdown).

## 클러스터 형성 — 세 가지 방식

새 etcd 클러스터를 시작하는 방법:

- **static**: 멤버 목록을 설정에 박아(`--initial-cluster`). kubeadm 스택드 etcd가 보통 이 방식.
- **discovery**: 디스커버리 서비스로 멤버를 동적 발견.
- **기존 클러스터 합류**: `--initial-cluster-state=existing`으로 기존 클러스터에 learner로 합류
  ([09-auth-membership.md](09-auth-membership.md)).

## serve — 클라이언트/피어 리스너

`embed/serve.go`가 두 종류의 gRPC 리스너를 연다:

- **클라이언트**(보통 2379): apiserver 등이 접속([07-api-client.md](07-api-client.md)). etcd CA로 mTLS
  ([17-security/03](../17-security/03-pki-certificates.md)).
- **피어**(보통 2380): 멤버 간 Raft 통신([01-raft.md](01-raft.md)).

설정(`embed/config.go`)이 주소/TLS/데이터 디렉토리/Raft 파라미터를 정한다.

## 정리

```
StartEtcd(cfg)
  → bootstrap: WAL/스냅샷 있으면 복원, 없으면 초기화
  → backend/MVCC/WAL/Raft/lessor 조립
  → serve: 클라이언트(2379)+피어(2380) 리스너
  → Ready 루프 시작 (10-raft-ready-loop)
```

## 더 읽을 곳
- [02-wal-snapshot.md](02-wal-snapshot.md) — 부팅 시 복원의 근거
- [09-auth-membership.md](09-auth-membership.md) — 멤버 합류(learner)
- [15-cli/01](../15-cli/01-kubeadm.md) — 스택드 etcd 부팅
