# 20.01 · Raft 합의

**근거**: `etcd/server/etcdserver/raft.go`, raft 알고리즘 모듈 `go.etcd.io/raft/v3`
(`server/go.mod`에 `v3.7.0-rc.1`)

Raft는 etcd가 **여러 노드에 복제되면서도 모두가 같은 데이터에 동의**하게 하는 합의 알고리즘이다. etcd는
Raft 구현을 별도 모듈(`go.etcd.io/raft/v3`)로 두고, 서버는 그것을 `raftNode`
(`raft.go:81`)로 감싸 사용한다.

## Raft가 푸는 문제

3대(또는 5대)의 etcd가 있다고 하자. 클라이언트가 어디에 써도 모두가 같은 결과를 봐야 하고, 일부가
죽어도 동작해야 한다. Raft는 이를 **로그 복제(replicated log)** 로 푼다: 모든 쓰기를 "로그 엔트리"로
만들고, **과반(quorum)** 이 같은 순서로 그 로그를 가질 때만 "확정(committed)"한다.

## 세 가지 핵심

### 1) 리더 선출
한 시점에 **리더 하나**만 있다. 리더가 죽으면(하트비트 끊김) 팔로워들이 타임아웃 후 후보가 돼 투표를
요청하고, 과반 표를 얻으면 새 리더가 된다. **term**(임기 번호)이 단조 증가해 오래된 리더를 구분한다.
모든 쓰기는 리더를 통한다.

### 2) 로그 복제
리더는 클라이언트 쓰기를 로그 엔트리로 만들어 팔로워에 보낸다(AppendEntries). 과반이 그 엔트리를
받았다고 확인하면 리더는 그것을 **committed**로 표시한다. committed 엔트리는 절대 사라지지 않는다.
이후 각 노드가 그것을 상태기계에 **apply**한다(→ [06-server-flow.md](06-server-flow.md)).

### 3) 안전성
같은 인덱스의 엔트리는 모든 노드에서 같은 내용이어야 한다. Raft는 로그 일치 속성과 리더 완전성으로
이를 보장한다 — 한 번 committed된 것은 모든 미래 리더의 로그에 존재한다.

## etcd 쪽 배선: raftNode

`raftNode`(`raft.go:81`)는 raft 라이브러리와 etcd 서버를 잇는다:

- `tick()`(`raft.go:159`): 논리 시계를 한 칸 진행(선거/하트비트 타이머).
- `start()`(`raft.go:174`): raft의 **Ready 루프**를 돈다 — raft가 "이제 이 엔트리를 디스크에 쓰고, 이
  메시지를 보내고, 이 committed 엔트리를 apply하라"고 알려주는 `Ready` 구조를 처리한다.
- `apply()`(`raft.go:404`): committed 엔트리를 서버의 apply 루프로 넘기는 채널을 돌려준다.
- `processMessages()`(`raft.go:357`): 팔로워/리더 간 메시지를 네트워크(peer transport)로 송신 준비.

**중요한 순서**: raft Ready 루프는 committed 엔트리를 apply하기 **전에** WAL에 먼저 기록한다
([02-wal-snapshot.md](02-wal-snapshot.md)). 그래야 노드가 적용 직후 죽어도, 재시작 시 WAL을 재생해
같은 상태로 복구된다.

## quorum과 가용성

| 클러스터 크기 | quorum | 견딜 수 있는 장애 |
|---------------|--------|-------------------|
| 3 | 2 | 1대 |
| 5 | 3 | 2대 |

짝수는 권장하지 않는다(가용성 이득 없이 quorum만 커짐). 그래서 etcd는 보통 3 또는 5대로 운영한다.

## 멤버십 변경

노드를 추가/제거하는 것도 Raft 로그를 통한 특수 엔트리(ConfChange)로 처리돼, 멤버 구성 변경마저
합의된 순서로 일어난다. 이로써 클러스터 재구성 중에도 일관성이 깨지지 않는다.

## 더 읽을 곳
- [02-wal-snapshot.md](02-wal-snapshot.md) — committed 엔트리의 디스크 내구성
- [06-server-flow.md](06-server-flow.md) — committed → apply → 응답
