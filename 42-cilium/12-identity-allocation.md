# 42.12 · Identity 할당

**근거**: `cilium/pkg/identity/` (`cache/allocator.go` `:31`~`:47`, `numericidentity.go`, `reserved.go`)

[02-agent.md](02-agent.md)에서 "cilium은 정책을 IP가 아니라 identity로 표현한다"고 했다. 이 문서는 그
**identity가 어떻게 할당·합의되는지** — 라벨 집합 → 숫자 ID 매핑이 클러스터 전체에서 일관되게 정해지는
메커니즘 — 를 본다. identity가 일관되지 않으면 정책이 노드마다 다르게 적용돼 보안이 깨진다.

## identity = 라벨 집합의 숫자 표현

핵심 개념(재확인, [02-agent.md](02-agent.md)): **같은 보안 라벨 집합을 가진 모든 Pod는 같은 숫자
identity를 공유**한다. 정책은 "identity A → identity B 허용"으로 컴파일돼 eBPF policymap에 들어간다
([10-maps-conntrack.md](10-maps-conntrack.md)).

라벨 집합 → identity 변환은 `pkg/identity/key/`(라벨을 정규화한 키)와 `numericidentity.go`(숫자 ID
타입)가 다룬다.

## 두 종류의 identity

### Reserved identity
`reserved.go`. 미리 정해진 고정 ID들 — 동적 할당이 필요 없는 특수 대상:

- `host`(노드 자신), `world`(클러스터 밖 모든 것), `remote-node`, `health`, `init`, `unmanaged` 등.
- 예: "world로 나가는 egress 허용"은 reserved identity로 표현된다.

### 동적(보안) identity
일반 워크로드의 라벨 집합마다 할당되는 ID. "이 라벨 조합 = ID 12345"를 **클러스터 전체에서 합의**해야
한다 — 노드 A와 노드 B가 같은 라벨의 Pod에 같은 ID를 줘야 정책이 일관된다.

## 할당의 합의 — 분산 allocator

`cache/allocator.go`가 동적 identity를 클러스터 전체에서 유일하게 할당한다. 두 백엔드 모드:

| 모드 | 저장 | 근거 |
|------|------|------|
| **kvstore** | etcd/Consul 등 KV에 "라벨키 → ID" 기록 | `allocator.go:31` `kvstore`, `IdentitiesPath` `:47` |
| **CRD** | Kubernetes의 `CiliumIdentity` CRD로 | apiserver를 KV처럼 사용(별도 KV 불필요) |
| **double-write** | 전환기: 둘 다 기록 | `:33` `doublewrite` |

`IdentitiesPath`(`allocator.go:47` = `.../state/identities/v1`)가 kvstore 모드에서 identity들이 저장되는
경로다. allocator는 "이 라벨키에 이미 ID가 있나? 없으면 새 ID를 원자적으로 할당"을 분산 환경에서
보장한다(경쟁 시 한 ID로 수렴).

```
노드 A: Pod(app=web) 생성 → 라벨키 계산 → allocator에 "이 키의 ID?" 
   ├─ 이미 있음(다른 노드가 먼저 할당) → 그 ID 재사용
   └─ 없음 → 새 ID 원자적 할당 → kvstore/CRD에 기록
모든 노드가 같은 키 → 같은 ID  (정책 일관성)
```

## identity의 수명

- Pod가 생기면 그 라벨의 identity가 참조된다(없으면 할당).
- 그 identity를 쓰는 마지막 Pod가 사라지면, identity는 **GC**된다(cilium-operator,
  [02-agent.md](02-agent.md)) — 안 쓰는 ID를 회수해 ID 공간을 재사용.
- identity가 바뀌면(라벨 변경) 엔드포인트 재생성이 트리거된다([11-endpoint-regeneration.md](11-endpoint-regeneration.md)).

## CIDR/외부 대상 identity

클러스터 밖 대상(특정 CIDR, DNS 이름)도 identity를 받아 정책에 쓸 수 있다([03-networkpolicy.md](03-networkpolicy.md)의
egress 정책). `ipcache`([10-maps-conntrack.md](10-maps-conntrack.md))가 IP↔identity 매핑을 데이터패스에
제공한다.

## 왜 이 설계인가

IP는 휘발적이다(Pod 재생성마다 바뀜). identity는 라벨 기반이라 안정적이다 — 그래서:
- 정책을 IP로 쓰면 Pod 교체마다 갱신해야 하지만, identity로 쓰면 라벨이 같은 한 그대로다.
- ID는 작은 정수라 eBPF 맵 조회가 빠르다([01-ebpf-datapath.md](01-ebpf-datapath.md)).

## 더 읽을 곳
- [02-agent.md](02-agent.md) — identity 개념과 정책 시행
- [03-networkpolicy.md](03-networkpolicy.md) — identity 기반 정책
- [10-maps-conntrack.md](10-maps-conntrack.md) — ipcache/policymap
