# 20.04 · 백엔드 — bbolt B+tree

**근거**: `etcd/server/storage/backend/` (`backend.go`, `batch_tx.go`, `read_tx.go`)

MVCC가 매긴 `revision → 값`을 실제 디스크에 영속화하는 것이 **백엔드**다. etcd는 백엔드로 **bbolt**
(BoltDB의 후속, `go.etcd.io/bbolt`)를 쓴다 — 단일 파일에 B+tree로 키-값을 저장하는 임베디드 DB.

```go
// backend.go:30, :108
import bolt "go.etcd.io/bbolt"
...
db *bolt.DB   // 단일 파일(보통 member/snap/db)
```

## bbolt의 특성

- **단일 파일 + mmap**: 전체 DB가 하나의 파일이고 메모리 매핑으로 읽는다. 읽기는 매우 빠르다.
- **B+tree**: 정렬된 키 순회(range query)에 적합 — etcd의 prefix 조회/list에 잘 맞는다.
- **MVCC(쓰기 1 / 읽기 N)**: 한 시점에 쓰기 트랜잭션은 하나, 읽기 트랜잭션은 여럿 가능. 읽기는 쓰기를
  막지 않는다(읽기 일관성 스냅샷).
- **fully serializable 쓰기**: 쓰기는 직렬화돼 커밋된다.

## 트랜잭션 — 배치 쓰기

etcd는 모든 MVCC 변경을 bbolt 트랜잭션으로 감싼다:

- **batch_tx**(`batch_tx.go`): 쓰기 트랜잭션. 여러 변경을 모아 한 번에 커밋해 fsync 횟수를 줄인다
  (성능). 일정 간격/건수마다 커밋.
- **read_tx**(`read_tx.go`): 읽기 트랜잭션. 일관된 스냅샷에서 읽는다.

bbolt 안에서 데이터는 **bucket**(네임스페이스)으로 나뉜다. etcd는 `key` bucket에 `revision → 값`을,
별도 메타 bucket에 클러스터/멤버 정보를 저장한다.

## 디스크 회수 — compaction과 defrag

bbolt는 데이터를 지워도 파일 크기를 자동으로 줄이지 않는다. 빈 페이지는 **freelist**로 재사용되지만
파일 자체는 그대로다. 그래서 두 단계가 필요하다:

1. **compaction**(MVCC 계층, [03-mvcc.md](03-mvcc.md)): 오래된 revision을 논리적으로 제거 → bbolt에
   빈 페이지 발생.
2. **defrag**(`etcdctl defrag`): bbolt 파일을 다시 써서 빈 페이지를 제거하고 **실제 파일 크기를 축소**.
   defrag 중에는 그 멤버가 잠시 멈춘다(블로킹).

## 공간 할당량 — quota

etcd는 DB 크기 상한(`--quota-backend-bytes`, 기본 보통 2~8GB)을 둔다. 초과하면 **alarm(NOSPACE)** 을
올리고 쓰기를 거부한다([06-server-flow.md](06-server-flow.md)의 `Alarm`). Kubernetes에서 이 alarm이
뜨면 클러스터 전체 쓰기가 멈추므로, compaction+defrag로 공간을 관리하는 것이 운영의 핵심이다.

## 더 읽을 곳
- [03-mvcc.md](03-mvcc.md) — 백엔드에 무엇을(revision→값) 저장하나
- [08-operations.md](08-operations.md) — defrag/백업 운영
