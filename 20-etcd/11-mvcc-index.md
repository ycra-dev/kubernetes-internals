# 20.11 · MVCC 인덱스 내부 (treeIndex/keyIndex)

**근거**: `etcd/server/storage/mvcc/index.go`(`treeIndex` `:39`, `Compact` `:205`),
`etcd/server/storage/mvcc/key_index.go`(`keyIndex` `:73`, `generation` `:346`, `tombstone` `:131`,
`doCompact` `:258`)

[03-mvcc.md](03-mvcc.md)에서 "인메모리 B-tree 색인이 키→revision을 찾고, bbolt가 revision→값을 저장
한다"고 했다. 이 문서는 그 **인메모리 색인의 자료구조** — treeIndex, keyIndex, generation — 를 코드로
파고든다. 과거 읽기와 compaction이 정확히 어떻게 동작하는지가 여기 있다.

## treeIndex — 키들의 B-tree

`index.go:39`의 `treeIndex`는 모든 키를 담는 B-tree다. 각 키는 **keyIndex** 하나에 매핑된다. 키로
조회하면 treeIndex에서 그 keyIndex를 찾고, keyIndex가 "어떤 revision에 어떤 값이 있었나"를 안다.

> treeIndex는 **키와 revision 메타만** 메모리에 둔다 — 실제 값은 bbolt에 있다
> ([04-backend-bbolt.md](04-backend-bbolt.md)). 그래서 메모리는 키 수에 비례하지, 값 크기에 비례하지
> 않는다.

## keyIndex — 한 키의 일생

`key_index.go:73`의 `keyIndex`는 **한 키가 거쳐온 모든 revision**을 기록한다. 구조는 **generation들의
리스트**다(`generation` `:346`):

```
keyIndex("foo")
  generation[0]: [rev2, rev5, rev9(tombstone)]   ← 첫 생성~삭제까지
  generation[1]: [rev12, rev15]                  ← 다시 생성된 후
  ...
```

- 한 **generation**은 "키가 생성돼 살아 있다가 삭제(tombstone)될 때까지"의 revision 묶음이다.
- 키를 **삭제하면 tombstone**(`key_index.go:131`)이 그 generation을 닫는다. 같은 키를 다시 만들면 새
  generation이 시작된다.

이 구조 덕분에 "키 foo의 revision 7 시점의 값"을 찾을 수 있다: keyIndex에서 rev7 이하의 가장 큰 revision
(rev5)을 찾아 그 backend revision으로 bbolt에서 값을 읽는다([03-mvcc.md](03-mvcc.md)).

## tombstone — 삭제의 표현

etcd는 키를 삭제해도 즉시 지우지 않고 **tombstone revision**을 추가한다(`key_index.go:131`). 왜?

- **watch**가 "삭제됨" 이벤트를 그 revision에 전달해야 한다([05-lease-watch.md](05-lease-watch.md)).
- **과거 읽기**가 "그 시점엔 있었다/없었다"를 정확히 답해야 한다.

tombstone은 compaction 때 비로소 제거된다(아래).

## compaction의 실제 동작

[03-mvcc.md](03-mvcc.md)의 compaction을 코드로 보면 — `treeIndex.Compact`(`index.go:205`)가 모든
keyIndex에 대해 `doCompact`(`key_index.go:258`)를 호출한다:

1. 주어진 revision보다 **오래된 generation/revision을 제거**한다.
2. 단, 각 generation의 **마지막 살아 있는 revision은 남긴다** — 그게 "현재 값"이거나 "그 시점 이후
   조회에 필요"하기 때문.
3. tombstone으로 완전히 닫힌 generation은 통째로 제거. keyIndex에 남는 게 없으면 treeIndex에서 키 자체를
   제거.
4. 반환된 "제거 가능한 revision 집합"(`Compact`의 반환)을 bbolt에서도 지운다 → 실제 디스크 빈 공간 발생
   ([04-backend-bbolt.md](04-backend-bbolt.md), defrag로 회수).

```
compact(rev=10):
  foo gen[0]: [2,5,9(tomb)]  → 9 이전 다 제거, 닫힌 generation이므로 통째 제거
  foo gen[1]: [12,15]        → 10 이후라 유지
  bar gen[0]: [3,7]          → 7만 남김(현재 값), 3 제거
```

## 왜 이게 Kubernetes에 중요한가

- compaction된 revision 이전을 watch하면 etcd가 거부 → apiserver가 `Expired` → 클라이언트 relist
  ([10-apiserver/12](../10-apiserver/12-watch-cache-internals.md), [00-foundations/04](../00-foundations/04-list-watch-informer.md)).
  즉 keyIndex의 generation 구조가 "RV too old"의 근본이다.
- 키가 많고(대규모 클러스터) 변경이 잦으면 keyIndex 리스트가 길어져 메모리/조회 비용이 는다 — 그래서
  주기적 compaction이 필수다([08-operations.md](08-operations.md)).

## 더 읽을 곳
- [03-mvcc.md](03-mvcc.md) — MVCC 개요
- [04-backend-bbolt.md](04-backend-bbolt.md) — revision→값 저장
- [10-apiserver/12](../10-apiserver/12-watch-cache-internals.md) — compaction이 watch에 미치는 영향
