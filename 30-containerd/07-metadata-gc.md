# 30.07 · 메타데이터 저장소와 가비지 컬렉션

**근거**: `containerd/core/metadata/` (`db.go` `:32` bbolt, `gc.go` `ResourceContent`/`Snapshot`/`Lease`
`:38`~), `containerd/plugins/gc/scheduler/`, `containerd/core/leases/`

containerd의 모든 객체(컨테이너, 이미지, 스냅샷 참조, content 참조)는 **메타데이터 저장소**에 기록되고,
참조가 없어진 것은 **GC**가 정리한다. [02-content-snapshots.md](02-content-snapshots.md)의 content/snapshot이
어떻게 추적·회수되는지가 여기 있다.

## 메타데이터 저장소 — bbolt

`core/metadata/db.go:32`가 흥미롭다 — containerd도 **bbolt**(`go.etcd.io/bbolt`)를 메타데이터 저장에
쓴다(etcd와 같은 임베디드 KV, [20-etcd/04](../20-etcd/04-backend-bbolt.md)):

- 컨테이너/이미지/스냅샷/content/lease 메타데이터를 bbolt 버킷에 저장(`containers.go`, `images.go`,
  `content.go`, `snapshots`, `leases.go`).
- **namespace 격리**(`db.go:38` `namespaces`, `:60` `WithPolicyIsolated`): containerd 자체 namespace
  ([04-ctr-client.md](04-ctr-client.md))마다 객체를 분리. `k8s.io` namespace의 Kubernetes 객체와 `default`가
  섞이지 않는다.

> 즉 content store(실제 blob, [02-content-snapshots.md](02-content-snapshots.md))는 콘텐츠 주소 파일시스템에,
> 그것을 **참조하는 메타데이터**는 bbolt에 — 분리돼 있다.

## 왜 GC가 필요한가

[02-content-snapshots.md](02-content-snapshots.md)에서 content store는 digest로 blob을 저장하고, snapshot은
레이어 위에 쌓인다고 했다. 이미지를 지우거나 컨테이너를 삭제하면:

- 그 이미지가 참조하던 레이어 blob, snapshot이 **다른 것도 참조하지 않으면** 디스크 낭비.
- 하지만 **공유 레이어**(여러 이미지가 쓰는)는 지우면 안 된다.

그래서 "무엇이 무엇을 참조하는가"의 그래프를 추적해, 도달 불가능한 것만 지워야 한다.

## tricolor mark-and-sweep GC

`core/metadata/gc.go`의 GC는 **삼색(tricolor) mark-and-sweep** 알고리즘이다. 자원 종류(`gc.go:38`~):

- `ResourceContent`(blob), `ResourceSnapshot`(레이어), `ResourceImage`, `ResourceLease`, `ResourceContainer`.

동작:

```
[mark]  루트(컨테이너, 이미지, lease)에서 시작해 참조를 따라가며 도달 가능한 자원을 표시
[sweep] 표시 안 된(도달 불가) content/snapshot 메타데이터를 삭제 → 실제 blob/레이어도 제거
```

도달 가능성: 컨테이너 → snapshot → 부모 snapshot → ... , 이미지 → 매니페스트 → 레이어 blob. 이 그래프에서
어떤 루트에서도 못 닿는 노드가 garbage다.

> 이는 Kubernetes의 ownerReference GC([11-controller-manager/04](../11-controller-manager/04-garbage-collector.md))와
> **같은 발상**(참조 그래프 기반 도달 불가 객체 제거)이지만, 대상이 API 객체가 아니라 런타임 자원이다.

## lease — GC로부터 보호

[02-content-snapshots.md](02-content-snapshots.md)에서 언급한 **lease**(`core/leases/`)가 여기서 역할을
한다. 작업 중인 content/snapshot은 아직 컨테이너/이미지에 연결되지 않아 GC의 루트에서 도달 불가다 —
그냥 두면 mark-sweep이 지운다.

**lease를 루트로** 삼아 보호한다: 이미지를 pull하는 동안 lease를 만들어 받은 레이어들을 lease에 묶으면,
GC가 그것을 도달 가능으로 보고 안 지운다. pull이 끝나 이미지가 완성되면 lease를 풀어도 이미지가 루트가
된다.

```
PullImage 시작 → lease 생성 → 레이어들을 lease에 묶음 (GC 보호)
   다운로드/unpack ...
이미지 완성 → 이미지가 루트가 됨 → lease 해제 가능
```

> etcd의 lease(TTL 키, [20-etcd/05](../20-etcd/05-lease-watch.md))와 이름은 같지만 다른 개념 — 여기선
> "GC로부터의 보호 참조"다.

## GC 스케줄링

`plugins/gc/scheduler/scheduler.go`가 GC 실행 시점을 정한다 — 매 삭제마다 돌리면 비싸므로, 변경량/시간
기준으로 배치 실행한다(딜리션이 쌓이면 또는 주기적으로). etcd의 compaction+defrag
([20-etcd/08](../20-etcd/08-operations.md))처럼 "정리를 모아서" 한다.

## 더 읽을 곳
- [02-content-snapshots.md](02-content-snapshots.md) — GC가 정리하는 content/snapshot
- [11-controller-manager/04](../11-controller-manager/04-garbage-collector.md) — 비교: K8s ownerReference GC
- [20-etcd/04](../20-etcd/04-backend-bbolt.md) — 같은 bbolt 백엔드
