# 20.09 · etcd 인증·멤버십·프록시

**근거**: `etcd/server/auth/`, `etcd/server/etcdserver/api/membership/`(`member.go:33` IsLearner),
`etcd/server/proxy/grpcproxy/`

etcd 자체의 보안(인증/RBAC)과, 클러스터 멤버십을 안전하게 바꾸는 메커니즘, 부하 분산용 프록시를 다룬다.
Kubernetes에서 etcd는 보통 apiserver만 접근하므로([10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md)),
etcd 인증은 그 한 클라이언트(apiserver)와의 mTLS가 핵심이다.

## etcd 인증·RBAC

`etcd/server/auth/`. etcd는 자체 인증/인가를 가진다(Kubernetes RBAC와 별개):

- **인증**: 사용자/비밀번호 또는 클라이언트 인증서. 토큰은 simple token 또는 JWT(`auth/jwt.go`,
  `simple_token.go`).
- **RBAC**: 사용자→역할→키 범위 권한(`auth/range_perm_cache.go` — 어떤 키 범위에 read/write 가능한지).

Kubernetes 컨텍스트에서는 보통 **클라이언트 인증서 기반 mTLS**로 apiserver만 접근하게 한다. etcd가 직접
노출되면 클러스터 전체 비밀이 노출되므로, etcd는 네트워크적으로 격리하고 인증서로 보호하는 것이 필수다.

## 멤버십 변경 — learner 노드

[01-raft.md](01-raft.md)에서 멤버십 변경도 Raft 로그를 거친다고 했다. 새 멤버를 바로 투표권 있는 멤버로
추가하면 위험하다 — 아직 데이터를 못 따라잡은 새 멤버가 quorum 계산에 끼면 가용성이 흔들린다.

**learner 노드**(`membership/member.go:33` `IsLearner`)가 이를 푼다:

- 새 멤버를 먼저 **learner**(투표권 없음, 데이터만 수신)로 추가.
- learner가 리더의 로그/스냅샷을 따라잡아 동기화되면([02-wal-snapshot.md](02-wal-snapshot.md)),
- 그때 **투표 멤버로 승격**한다.

이로써 멤버 추가 중 quorum이 흔들리지 않는다 — etcd 클러스터 확장의 안전장치다.

## gRPC 프록시

`etcd/server/proxy/grpcproxy/`. 많은 클라이언트가 같은 watch/read를 하면 etcd에 부담이다. gRPC 프록시는
etcd 앞에 서서:

- **watch 통합(coalescing)**: 같은 키를 watch하는 여러 클라이언트를 etcd의 watch 하나로 합친다
  ([05-lease-watch.md](05-lease-watch.md)).
- **read 캐시**: 읽기를 캐싱해 etcd 부하 감소.

> 흥미롭게도 이것은 apiserver의 watch cacher([10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md))와
> 같은 발상이다 — "watch 하나를 여럿에게 팬아웃". Kubernetes는 보통 apiserver cacher로 이미 해결하므로
> etcd gRPC 프록시를 추가로 쓰는 경우는 드물다.

## 더 읽을 곳
- [01-raft.md](01-raft.md) — 멤버십 변경이 거치는 Raft
- [08-operations.md](08-operations.md) — 멤버 추가/제거 운영
- [10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md) — apiserver의 유사한 watch 팬아웃
