# 20.08 · 운영 — etcdctl / etcdutl / 백업·복구

**근거**: `etcd/etcdctl/`, `etcd/etcdutl/`, `etcd/etcdutl/snapshot/v3_snapshot.go`

etcd는 Kubernetes의 **유일한 영속 상태**이므로, etcd 운영이 곧 클러스터 재해 복구의 핵심이다. 두
도구가 있다:

- **etcdctl**(`etcd/etcdctl/`): 살아 있는 etcd에 gRPC로 접속하는 클라이언트 CLI(put/get/watch/멤버 관리/
  defrag/snapshot save 등).
- **etcdutl**(`etcd/etcdutl/`): etcd에 접속하지 않고 **데이터 파일을 직접 다루는** 오프라인 도구
  (snapshot restore, WAL/DB 분석).

## 백업 — snapshot save

`etcdctl snapshot save backup.db`는 현재 etcd 상태의 일관된 사본을 뜬다(Maintenance API의 Snapshot,
`etcdctl/ctlv3/command/snapshot_command.go`). 이것이 클러스터 전체를 되살릴 수 있는 단일 백업이다.

> 주의: 이 "백업 스냅샷"은 [02-wal-snapshot.md](02-wal-snapshot.md)의 "Raft 스냅샷"과 다르다. 전자는
> 운영자 재해 복구용, 후자는 etcd 내부 로그 절단용이다.

## 복구 — snapshot restore

`etcdutl snapshot restore backup.db`(`etcdutl/snapshot/v3_snapshot.go`,
`etcdutl/etcdutl/snapshot_command.go`)는 백업 파일로부터 **새 데이터 디렉토리**를 만든다. 이때 새 멤버
구성/클러스터 ID를 부여하므로, 복구된 etcd는 깨끗한 새 클러스터로 뜬다. 이 데이터로 etcd를 시작하면
백업 시점의 모든 Kubernetes 객체가 돌아온다.

전형적 재해 복구:
```
1. (사고 발생) etcd 데이터 손상/소실
2. etcdutl snapshot restore <백업>  → 새 data-dir 생성
3. 그 data-dir 로 etcd 멤버(들) 재시작
4. apiserver 가 복구된 etcd 를 가리키도록
```

## 디스크 관리 — compaction + defrag

[03-mvcc.md](03-mvcc.md), [04-backend-bbolt.md](04-backend-bbolt.md)에서 본 대로:

1. **compact**: `etcdctl compact <revision>` — 그 revision 이전 버전 제거(논리 삭제). Kubernetes는 보통
   apiserver가 주기적으로 자동 수행.
2. **defrag**: `etcdctl defrag` — bbolt 파일을 다시 써 실제 디스크 회수(멤버별로, 잠시 블로킹).

이를 게을리하면 DB가 quota를 넘어 **NOSPACE alarm**이 뜨고 **클러스터 전체 쓰기가 멈춘다**
([04-backend-bbolt.md](04-backend-bbolt.md)). alarm 해제는 `etcdctl alarm disarm`.

## 상태 점검

- `etcdctl endpoint status` / `endpoint health`: 각 멤버의 리더 여부, DB 크기, raft 인덱스, 건강 상태.
- `etcdctl member list`: 멤버 구성.

운영에서 가장 중요한 신호는 **리더 안정성**(잦은 리더 변경 = 디스크/네트워크 문제)과 **DB 크기**다.

## Kubernetes 운영 관점 요약

| 작업 | 도구 | 빈도 |
|------|------|------|
| 정기 백업 | `etcdctl snapshot save` | 주기적(자동화 권장) |
| 디스크 관리 | compact(자동) + `etcdctl defrag` | 주기적 |
| 재해 복구 | `etcdutl snapshot restore` | 사고 시 |
| 건강 모니터링 | `etcdctl endpoint status/health` | 상시 |

## 더 읽을 곳
- [02-wal-snapshot.md](02-wal-snapshot.md) — 내부 스냅샷과의 구분
- [15-cli/01-kubeadm.md](../15-cli/01-kubeadm.md) — kubeadm의 스택드 etcd
