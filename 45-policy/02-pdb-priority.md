# 45.02 · PodDisruptionBudget과 PriorityClass

**근거**: `kubernetes/pkg/controller/disruption/`, `kubernetes/staging/src/k8s.io/api/scheduling/v1/types.go`(PriorityClass),
`kubernetes/plugin/pkg/admission/priority/`

워크로드의 **가용성을 보호**(PDB)하고, 자원 경쟁 시 **우선순위**(PriorityClass)를 정하는 두 정책이다.

## PodDisruptionBudget — 자발적 중단으로부터 보호

중단(disruption)에는 두 종류가 있다:
- **비자발적**: 노드 하드웨어 장애, 커널 패닉 등(막을 수 없음).
- **자발적**: `kubectl drain`(노드 비우기), 클러스터 업그레이드, 스케일다운 — **관리자가 통제 가능**.

**PodDisruptionBudget(PDB)** 은 자발적 중단 시 "최소 몇 개는 살아 있어야 한다"를 보장한다:

- `minAvailable: 2` 또는 `maxUnavailable: 1` 처럼 지정.
- `kubectl drain` 등이 **Eviction API**로 Pod를 쫓아내려 할 때, PDB를 위반하면 **거부**된다.
- 그래서 drain은 PDB를 지키며 한 번에 허용된 만큼만 축출하고, 대체 Pod가 Ready가 될 때까지 기다린다.

### Disruption 컨트롤러
`pkg/controller/disruption/`이 각 PDB의 현재 상태(허용 가능한 중단 수 `disruptionsAllowed`)를 계산해
status에 기록한다. Eviction API가 이 값을 보고 축출을 허용/거부한다.

```
kubectl drain node-1
   │  각 Pod에 Eviction 요청
   ▼
PDB 검사: minAvailable 위반? → 거부(429), 대기 → 안전하면 축출
   │
소유 컨트롤러가 다른 노드에 새 Pod 생성 → Ready → 다음 Pod 축출
```

> PDB는 **자발적** 중단만 막는다. 노드가 갑자기 죽는 비자발적 중단은 못 막는다(그건
> [11-controller-manager/05](../11-controller-manager/05-node-lifecycle.md)의 eviction이 처리).

## PriorityClass — 무엇이 더 중요한가

**PriorityClass**(`scheduling/v1` API)는 Pod에 **우선순위 숫자**를 부여한다. 자원이 부족할 때 누가
우선인지 정한다:

1. 사용자가 PriorityClass를 만들고(예: `critical: 1000000`, `batch: 100`), Pod가 `priorityClassName`으로
   참조.
2. 어드미션 플러그인 `priority`(`plugin/pkg/admission/priority/`,
   [10-apiserver/04](../10-apiserver/04-admission.md))가 이를 `Pod.spec.priority` 숫자로 해석.
3. 스케줄러가 이 숫자를 두 곳에서 쓴다:
   - **큐 정렬**: 높은 우선순위 Pod를 먼저 스케줄([12-scheduler/03](../12-scheduler/03-queue-eventhandlers.md)).
   - **선점(preemption)**: 자리가 없으면 낮은 우선순위 Pod를 쫓아내 자리 확보
     ([12-scheduler/04](../12-scheduler/04-preemption-extender.md)).

### 노드 압박 시에도
우선순위는 kubelet **eviction** 순서에도 영향을 준다([13-kubelet/04](../13-kubelet/04-eviction.md)) —
자원 부족 시 낮은 우선순위 Pod가 먼저 축출된다.

## 두 정책의 협력

```
PriorityClass: "이 워크로드가 더 중요" → 우선 스케줄 + 선점 우선권 + 축출 후순위
PDB:           "이 워크로드는 한 번에 N개 이상 죽이지 마" → 자발적 중단 제한
```

함께 쓰면 "중요한 서비스는 우선 배치되고, 업그레이드 중에도 최소 가용성을 유지"한다.

## 더 읽을 곳
- [12-scheduler/04](../12-scheduler/04-preemption-extender.md) — 우선순위 기반 선점
- [13-kubelet/04](../13-kubelet/04-eviction.md) — 우선순위와 축출
