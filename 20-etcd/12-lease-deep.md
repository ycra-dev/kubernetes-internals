# 20.12 · Lease 내부 (lessor)

**근거**: `etcd/server/lease/lessor.go`(`Grant` `:281`, `expiredC` `:176`, `SetCheckpointer` `:274`,
`leaseCheckpointRate` `:49`)

[05-lease-watch.md](05-lease-watch.md)에서 lease(TTL 키)를 개요로 봤다. 이 문서는 **lessor**가 lease를
어떻게 추적·만료·체크포인트하는지를 코드 수준에서 본다. lease 만료가 Raft를 거치면서도 정확한 타이밍을
유지하는 메커니즘이 핵심이다.

## lessor — lease들의 관리자

`lessor`(`lessor.go:145`, [05-lease-watch.md](05-lease-watch.md))가 모든 lease를 관리한다. 주요 연산:

- **Grant**(`lessor.go:281`): TTL을 가진 새 lease 생성. 만료 시각을 계산해 추적 큐에 넣는다.
- **Renew**: KeepAlive로 만료를 미룸(만료 시각 재계산).
- **Revoke**: lease에 묶인 모든 키를 삭제.
- **Attach/Detach**: 키를 lease에 묶거나 푼다.

## 만료 감지 — expiredC

lessor는 만료된 lease를 주기적으로 검사한다. 만료된 것을 찾으면 **`expiredC`** 채널
(`lessor.go:176`, `:237` 버퍼 16)로 내보낸다:

```
lessor: 만료된 lease 발견 → expiredC 채널로
   │
EtcdServer: expiredC 수신 → LeaseRevoke 를 Raft에 제안 (v3_server)
   │
Raft commit → apply: lease에 묶인 키들 삭제 (모든 멤버에서)
```

**핵심**: lease 만료는 lessor가 로컬에서 감지하지만, **실제 삭제는 Raft를 거친다**
([01-raft.md](01-raft.md), [06-server-flow.md](06-server-flow.md)). 그래야 모든 멤버가 같은 revision에서
같은 키를 지운다 — 한 멤버만 지우면 일관성이 깨진다.

## 리더만 만료시킨다

오직 **리더**의 lessor만 만료를 처리한다. 팔로워는 자기 시계로 만료를 정하지 않는다 — 그러면 멤버마다
시계 차이로 다른 시점에 만료시켜 불일치가 생긴다. 리더가 만료를 감지해 Revoke를 제안하면, 그 결과가
로그로 모든 멤버에 동일하게 적용된다.

## checkpoint — 리더 교체 시 TTL 보존

문제: lease의 "남은 TTL"은 리더의 메모리에만 있다. 리더가 바뀌면 새 리더는 lease의 **원래 TTL**부터
다시 센다 — 곧 만료될 lease가 갑자기 수명이 연장되는 버그.

**checkpoint**(`lessor.go:274` `SetCheckpointer`, `:49` `leaseCheckpointRate=1000`)가 이를 푼다:

- 리더가 주기적으로 각 lease의 **남은 TTL을 Raft 로그에 기록**(checkpoint)한다.
- 리더가 바뀌어도 새 리더는 마지막 checkpoint의 남은 TTL부터 이어 센다.
- `leaseCheckpointRate`(`:49`)로 초당 checkpoint 수를 제한(로그 폭증 방지).

```
리더 A: lease(TTL 100s) → 30s 지남 → checkpoint("남은 70s")를 로그에
   │  리더 교체
리더 B: 마지막 checkpoint 봄 → "남은 70s"부터 이어 셈 (100s로 리셋 안 함)
```

이로써 리더 교체가 lease 만료 타이밍을 망치지 않는다.

## Kubernetes에서의 영향

[05-lease-watch.md](05-lease-watch.md)에서 봤듯, apiserver는 Event 같은 TTL 객체에 etcd lease를 재사용한다
([10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md)의 `lease_manager`). lease 만료 정확성이
보장되므로, TTL 객체가 제때 정리되고 리더 교체에도 흔들리지 않는다.

> 다시 강조: 이 etcd lease는 Kubernetes API의 Lease 리소스(노드 하트비트/리더 선출,
> [00-foundations/06](../00-foundations/06-leader-election.md))와 **다른 계층**이다.

## 더 읽을 곳
- [05-lease-watch.md](05-lease-watch.md) — lease/watch 개요
- [06-server-flow.md](06-server-flow.md) — Revoke가 거치는 Raft/apply
- [01-raft.md](01-raft.md) — 만료가 합의되는 이유
