# 12.04 · 선점(Preemption)과 Extender

**근거**: `kubernetes/pkg/scheduler/framework/plugins/defaultpreemption/`,
`kubernetes/pkg/scheduler/extender.go`

## 선점 — 자리를 만들어 고우선순위 Pod를 넣기

필터 결과 feasible 노드가 0개면, 높은 우선순위 Pod는 그냥 포기하지 않는다. **PostFilter** 확장점
([01](01-framework.md))에서 `defaultpreemption` 플러그인이 선점을 시도한다(`schedule_one.go:277`에서
PostFilter 호출).

동작 개념:

1. **후보 노드 탐색**: 어떤 노드에서 **더 낮은 우선순위의 Pod들을 비우면** 이 Pod가 들어갈 수 있는지
   계산한다.
2. **최소 비용 선택**: 가능한 노드 중, 쫓아낼 Pod의 수/우선순위/PDB 위반이 가장 적은 노드를 고른다.
   PodDisruptionBudget을 가급적 존중한다.
3. **victim 축출**: 선택된 노드의 victim Pod들을 삭제(graceful)하고, 선점하는 Pod에 그 노드를
   **nominated node**로 지정한다(`backend/queue/nominator.go`).
4. victim이 비워지면 다음 스케줄 시도에서 그 노드에 배정된다.

> 선점은 우선순위(`PriorityClass`)가 있어야 의미가 있다. PriorityClass → `Pod.spec.priority` 변환은
> 어드미션 플러그인 `priority`가 한다([10-apiserver/04](../10-apiserver/04-admission.md)).

## nominated node

선점한 Pod는 즉시 바인딩되지 않는다(victim 종료를 기다려야 함). 그 사이 다른 Pod가 비워진 자리를
가로채지 못하도록 "이 노드를 예약 중"이라고 표시하는 것이 nominated node다. 다음 사이클에서 우선 고려된다.

## Extender — 외부 스케줄 로직(레거시 확장)

`extender.go`는 Framework 플러그인이 등장하기 전의 확장 방식이다. 스케줄러가 **외부 HTTP 서버**에
필터/점수/바인딩을 위임할 수 있다:

- Filter 단계 후 feasible 노드 목록을 extender에 보내 추가 필터링.
- Score 단계에서 extender의 점수를 가중 합산(`prioritizeNodes`가 `sched.Extenders`를 받는다,
  `schedule_one.go:600`).

오늘날 새 확장은 **in-process 플러그인(Scheduling Framework)** 이 권장된다 — 네트워크 왕복이 없고 더
풍부한 확장점을 제공하기 때문. Extender는 스케줄러를 재빌드할 수 없는 환경 등 특수 경우에 남아 있다.

## 더 읽을 곳
- [01-framework.md](01-framework.md) · [02-plugins.md](02-plugins.md)
- [80-scenarios/04-scale-and-failure.md](../80-scenarios/) — 자원 압박 시 동작
