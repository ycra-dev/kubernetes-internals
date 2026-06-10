# 20.02 · WAL과 스냅샷 (내구성)

**근거**: `etcd/server/storage/wal/`, `etcd/server/storage/`(snap)

Raft가 "엔트리를 committed로 정하기" 전에, 각 노드는 그 엔트리를 **디스크에 먼저 기록**해야 한다.
전원이 갑자기 꺼져도 합의된 로그가 사라지지 않게 하는 것이 **WAL(Write-Ahead Log)** 이다.

## WAL — 먼저 쓰고 나서 적용

원칙은 이름 그대로다: **상태기계에 적용하기 전에 로그에 먼저 쓴다.** etcd의 Raft Ready 루프는 각 라운드
에서 (1) 새 엔트리/하드스테이트를 WAL에 fsync로 기록한 뒤 (2) 메시지를 보내고 (3) committed 엔트리를
apply한다([01-raft.md](01-raft.md)).

`server/storage/wal/`의 구성:

- `encoder.go` / `decoder.go`: 엔트리를 WAL 레코드로 직렬화/역직렬화(CRC로 무결성 검증).
- `file_pipeline.go`: WAL 세그먼트 파일을 미리 할당(preallocate)해 쓰기 지연을 줄인다.
- `repair.go`: 비정상 종료로 마지막 레코드가 잘렸을 때 복구.

WAL은 **순차 추가(append-only)** 라 디스크에 친화적이다. 그래서 etcd 성능은 fsync 지연에 크게 좌우된다
— 빠른 디스크(SSD)가 권장되는 이유.

## 재시작 복구

노드가 죽었다 살아나면:

1. 가장 최근 **스냅샷**을 로드해 그 시점의 상태를 복원.
2. 스냅샷 이후의 **WAL 엔트리를 재생(replay)** 해 마지막 committed 상태까지 따라잡는다.

WAL은 "무엇이 어떤 순서로 합의됐는가"의 완전한 기록이므로, 이 재생으로 노드는 죽기 직전 상태로 정확히
복구된다.

## 스냅샷 — WAL이 무한히 자라지 않게

WAL을 영원히 보관하면 디스크가 가득 차고 재생이 느려진다. 그래서 주기적으로(엔트리 N개마다)
**스냅샷**을 찍는다: 현재 전체 상태(백엔드 DB와 메타)를 저장하고, 그보다 오래된 WAL을 폐기한다.

- 스냅샷 인덱스보다 오래된 로그는 더 이상 필요 없다(스냅샷이 그 상태를 담고 있으므로).
- 느린 팔로워가 너무 뒤처져 리더의 로그에서 필요한 엔트리가 이미 폐기됐으면, 리더는 로그 대신
  **스냅샷 전체를 전송**해 따라잡게 한다(`applySnapshot`, `server/etcdserver/server.go:995`).

## 두 종류의 "스냅샷"을 혼동하지 말 것

- **Raft 스냅샷**(이 문서): 로그 절단/복구용 etcd 내부 메커니즘.
- **백업 스냅샷**(`etcdctl snapshot save`): 운영자가 재해 복구용으로 뜨는 DB 사본 →
  [08-operations.md](08-operations.md).

## 더 읽을 곳
- [01-raft.md](01-raft.md) — WAL이 보호하는 committed 로그
- [04-backend-bbolt.md](04-backend-bbolt.md) — 스냅샷이 담는 실제 데이터 저장소
