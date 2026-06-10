# 20.10 · Raft Ready 루프 (코드 레벨)

**근거**: `etcd/server/etcdserver/raft.go` (`start` `:174`, Ready 루프 `:185`~)

[01-raft.md](01-raft.md)에서 raft 라이브러리가 "Ready 구조로 할 일을 알려준다"고 했다. 이 문서는 그
**Ready 루프**를 실제 코드로 따라간다 — etcd가 합의·내구성·적용·전송의 순서를 어떻게 지키는지가 여기 있다.

## raft 라이브러리와의 계약

`go.etcd.io/raft`는 순수 상태기계다 — 네트워크/디스크를 모른다. 대신 `Ready` 구조로 "이것들을 처리하라"고
알려주고, 처리가 끝나면 `Advance()`로 다음으로 넘어간다. `raftNode.start`(`raft.go:174`)의 고루틴이 이
루프를 돈다.

## Ready 한 사이클의 순서

`rd := <-r.Ready()`(`raft.go:185`)로 받은 한 묶음을 처리하는 순서가 **정확성의 핵심**이다:

```
[1] SoftState 처리 (:186)      ─ 리더십 변화 감지 (누가 리더인지, 메트릭)
[2] ReadStates (:209)          ─ 선형화 읽기(ReadIndex) 응답 → readStateC (06-server-flow)
[3] toApply 구성 + applyc 전송 (:222~232) ─ committed 엔트리/스냅샷을 apply 루프로 넘김
[4] (리더면) 메시지 전송 (:240) ─ 팔로워에 엔트리 복제
[5] WAL 기록                   ─ 엔트리/하드스테이트를 디스크에 fsync
[6] r.Advance() (:331)         ─ raft에 "처리 완료" 통지 → 다음 Ready
```

### 리더의 병렬 최적화 (`:237` 주석)
코드 주석이 명시한다 — *"the leader can write to its disk in parallel with replicating to the
followers"*. 리더는 자기 디스크 기록(WAL)과 팔로워 복제를 **병렬로** 한다(`:240`에서 먼저 Send). 이는
Raft 논문 10.2.1의 최적화로, 리더가 자기 fsync를 기다리지 않고 복제를 시작해 지연을 줄인다.

### notifyc — apply와 WAL 기록의 순서 조율
`toApply`에 `notifyc` 채널이 들어간다(`raft.go:225`). apply 루프와 raft 루프가 이 채널로 동기화해,
**"committed 엔트리가 WAL/백엔드에 안전히 기록된 뒤 apply가 진행"** 되도록 순서를 맞춘다 — 스냅샷이
관련될 때 특히 중요하다(아래).

## committedIndex와 apply 분리

`updateCommittedIndex`(`raft.go:229`)로 committed 인덱스를 갱신하고, 엔트리를 `applyc` 채널로 보낸다
(`:232`). **raft 루프는 "합의·전송·기록"만 하고, 실제 상태기계 적용은 별도 apply 루프**가 한다
([06-server-flow.md](06-server-flow.md)의 `applyAll`). 이 분리로:

- raft 루프가 느린 apply에 막히지 않는다(`:218` 부근, apply가 raft보다 느릴 수 있음을 고려).
- 두 루프가 채널(`applyc`, `notifyc`)로 순서만 조율.

## 스냅샷 처리

`raftSnap`(`raft.go:221`)이 비어 있지 않으면 — 너무 뒤처진 팔로워가 리더에게 스냅샷을 받은 경우다
([02-wal-snapshot.md](02-wal-snapshot.md)). 이때는 **스냅샷을 백엔드에 먼저 적용한 뒤** raft를 Advance
해야 한다. `:326` 주석("leader already processed 'MsgSnap'")과 notifyc 동기화가 이 순서를 보장한다 —
순서가 틀리면 복구 시 데이터가 어긋난다.

## tick — 논리 시계

`r.tick()`(`raft.go:184`)는 일정 주기로 raft의 논리 시계를 한 칸 진행한다. 이것이 선거 타임아웃과
하트비트 간격의 기준이다([01-raft.md](01-raft.md)) — 리더는 tick마다 하트비트를, 팔로워는 tick을 세다
타임아웃되면 선거를 시작한다.

## 정리 — 왜 이 순서인가

```
합의(committed) → WAL 기록(내구성) → apply(상태) → Advance(다음)
                  ↑ 이 순서를 어기면 "적용했는데 재시작 후 사라지는" 데이터 손실
```

WAL 기록이 apply보다 먼저여야 하고, 스냅샷은 적용 후 Advance여야 한다. Ready 루프가 이 불변식을 채널
동기화로 강제한다 — etcd가 "전원이 언제 꺼져도 합의된 데이터를 잃지 않는" 비결이다.

## 더 읽을 곳
- [01-raft.md](01-raft.md) — Raft 개념
- [02-wal-snapshot.md](02-wal-snapshot.md) — WAL/스냅샷 내구성
- [06-server-flow.md](06-server-flow.md) — applyc가 도착하는 apply 루프
