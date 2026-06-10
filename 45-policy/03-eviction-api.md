# 45.03 · Eviction API (PDB 시행 지점)

**근거**: `kubernetes/pkg/registry/core/pod/storage/eviction.go`
(`EvictionREST` `:67`, `podDisruptionBudgetClient` `:74`, `MaxDisruptedPodSize` `:49`)

[02-pdb-priority.md](02-pdb-priority.md)에서 PodDisruptionBudget(PDB)이 "자발적 중단으로부터 최소
가용성을 보장한다"고 했다. 이 문서는 그것이 **실제로 시행되는 지점** — `kubectl drain`이 호출하는 Eviction
API — 를 코드 레벨로 본다. PDB가 "선언"이라면 Eviction API는 "집행"이다.

## Eviction은 Pod의 서브리소스

`kubectl delete pod`와 `kubectl drain`(내부적으로 Eviction)은 다르다:

- **Delete**: Pod를 그냥 지운다 — PDB를 무시.
- **Eviction**: Pod의 **`eviction` 서브리소스**([10-apiserver/13](../10-apiserver/13-strategy-status.md)의
  서브리소스 패턴)로 "이 Pod를 쫓아내도 되나?"를 묻는다 — **PDB를 검사**.

`pkg/registry/core/pod/storage/eviction.go`의 **EvictionREST**(`:67`)가 이 서브리소스를 구현한다.
`drain`/`descheduler`/오토스케일러 등 "협조적" 컴포넌트는 이 API를 써서 PDB를 존중한다.

## PDB 검사 — checkAndDecrement

Eviction 요청이 오면 EvictionREST가 그 Pod에 적용되는 PDB를 찾아 검사한다(`eviction.go`의
`podDisruptionBudgetClient` `:74`):

```
Eviction 요청 (Pod X)
   │  X에 매칭되는 PDB가 있나?
   ├─ 없음 → 바로 삭제 허용
   └─ 있음 → PDB.status.disruptionsAllowed > 0 인가?
        ├─ 예 → disruptionsAllowed 를 1 감소시키고 Pod 삭제 허용
        └─ 아니오 → 429 Too Many Requests (거부)
```

- **disruptionsAllowed**는 Disruption 컨트롤러([02-pdb-priority.md](02-pdb-priority.md))가 "지금 몇 개를
  더 쫓아내도 minAvailable을 지키나"로 계산해 PDB.status에 기록한 값이다.
- Eviction은 이 값을 **원자적으로 감소**시키며 허용한다(낙관적 동시성,
  [10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md)) — 동시에 여러 eviction이 와도 PDB를 초과해
  쫓아내지 않는다.
- 거부 시 **429**를 반환 → 클라이언트(drain)는 백오프 후 재시도.

## kubectl drain의 동작

`kubectl drain node-1`의 전체 흐름([15-cli](../15-cli/)):

```
1. 노드를 cordon (unschedulable taint, 99-appendix/well-known)
2. 노드의 각 Pod에 Eviction 요청
     ├─ PDB 허용 → Pod 삭제 → 소유 컨트롤러가 다른 노드에 재생성 (11-controller-manager/03)
     │              새 Pod가 Ready 될 때까지 disruptionsAllowed 회복 대기
     └─ PDB 거부(429) → 대기 후 재시도
3. 모든 Pod가 빠지면 drain 완료
```

그래서 drain은 **PDB를 지키며 한 번에 허용된 만큼만** Pod를 옮긴다 — 업그레이드/노드 교체 중에도 서비스
최소 가용성이 유지된다([15-cli/03](../15-cli/03-kubeadm-upgrade-reset.md)의 업그레이드).

## MaxDisruptedPodSize — 안전 한계

`eviction.go:49`의 `MaxDisruptedPodSize`는 PDB.status가 추적하는 "방금 쫓겨난 Pod" 목록의 최대 크기다.
이 추적으로 "막 evict했지만 아직 사라지지 않은 Pod"를 이중 계산하지 않게 한다 —
[11-controller-manager/02](../11-controller-manager/02-common-machinery.md)의 expectations와 같은
"캐시 지연 보정" 발상.

## 두 종류의 eviction (재확인)

| | API Eviction (이 문서) | kubelet eviction |
|--|------------------------|-------------------|
| 주체 | 컨트롤 플레인(drain 등) | 노드 kubelet |
| 트리거 | 자발적(노드 비우기/업그레이드) | 노드 자원 압박 |
| PDB | **존중** | 무시(노드 보호 우선) |
| 코드 | `registry/core/pod/storage/eviction.go` | `pkg/kubelet/eviction` ([13-kubelet/04](../13-kubelet/04-eviction.md)) |

이름이 같지만 완전히 다른 메커니즘이다 — API eviction은 협조적·PDB 존중, kubelet eviction은 강제·노드 보호.

## 더 읽을 곳
- [02-pdb-priority.md](02-pdb-priority.md) — PDB 개념과 Disruption 컨트롤러
- [13-kubelet/04](../13-kubelet/04-eviction.md) — 다른 eviction(노드 압박)
- [15-cli/03](../15-cli/03-kubeadm-upgrade-reset.md) — drain을 쓰는 업그레이드
