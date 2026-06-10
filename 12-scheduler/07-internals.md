# 12.07 · 스케줄러 내부 — 스냅샷·assume·percentageOfNodes

**근거**: `kubernetes/pkg/scheduler/schedule_one.go`(`UpdateSnapshot` `:177`, `ForgetPod` `:369`),
`kubernetes/pkg/scheduler/backend/cache/`

[01-framework.md](01-framework.md)는 스케줄링 사이클의 확장점을 다뤘다. 이 문서는 그것을 빠르고 정확하게
만드는 **내부 메커니즘** — 노드 스냅샷, 낙관적 assume, 노드 표본 추출 — 을 본다.

## nodeInfo 스냅샷 — 일관된 시야

스케줄링 사이클은 **직렬**(한 번에 한 Pod)로 돈다([01-framework.md](01-framework.md)). 하지만 그동안에도
노드/Pod는 계속 변한다. 매 필터·점수마다 최신 캐시를 읽으면 사이클 중간에 상태가 바뀌어 결정이
일관되지 않을 수 있다.

해결: 사이클 시작 시 **스냅샷을 뜬다**. `schedule_one.go:177`의 `sched.Cache.UpdateSnapshot(...,
nodeInfoSnapshot)`이 스케줄러 캐시(`backend/cache/`)에서 **그 시점의 모든 노드 상태(nodeInfo)** 를 일관된
스냅샷으로 만든다. 사이클은 이 스냅샷만 보고 결정하므로, 한 Pod를 스케줄하는 동안 일관된 시야를 갖는다.

`nodeInfo`는 한 노드의 "현재 떠 있는 Pod들, 남은 자원, 포트, 토폴로지"를 집계한 구조다 — 필터/점수가
노드를 평가할 때 보는 단위다.

## assume — 낙관적 예약

[01-framework.md](01-framework.md)에서 스케줄링 사이클(동기)과 바인딩 사이클(비동기)이 나뉜다고 했다.
문제: 바인딩이 끝나기 전에 다음 Pod를 스케줄하면, 방금 결정한 Pod의 자원이 아직 반영 안 돼 같은 자리에
또 배정할 수 있다.

해결: **assume**. 스케줄 결정 직후 Pod를 "그 노드에 있다고 가정(assume)"해 캐시/스냅샷에 즉시 반영한다.
그래서 다음 Pod 스케줄 시 그 자원이 이미 차 있는 것으로 보인다. 바인딩이 실제로 성공하면 확정,
**실패하면 `ForgetPod`**(`schedule_one.go:369`)로 가정을 취소해 자원을 되돌린다.

```
스케줄 결정 → assume(Pod를 노드에 가정) → 다음 Pod는 이 자원을 찬 것으로 봄
   │
바인딩 성공 → 확정
바인딩 실패 → ForgetPod(:369) → 가정 취소, 자원 복구, Pod 재큐
```

이 낙관적 동시성 덕분에 직렬 스케줄링 사이클과 병렬 바인딩이 충돌 없이 공존한다.

## percentageOfNodesToScore — 표본 추출

수천 노드 클러스터에서 **모든** 노드를 필터·점수하면 느리다. 게다가 보통은 일부만 봐도 충분히 좋은 노드를
찾는다. 그래서 스케줄러는 **percentageOfNodesToScore**(기본값
`schedulerapi.DefaultPercentageOfNodesToScore`)로 **feasible 노드를 일정 수만 찾으면 멈춘다**:

- 노드 목록을 라운드로빈으로 순회하며 필터를 돌리다, 충분한 수의 feasible 노드를 찾으면 중단
  ([01-framework.md](01-framework.md)의 `findNodesThatFitPod`).
- 큰 클러스터일수록 비율을 낮춰(예: 5%) 지연을 줄인다 — 최적은 아닐 수 있지만 "충분히 좋은" 노드를 빠르게.
- 라운드로빈 시작점을 옮겨가며 특정 노드에 쏠리지 않게 한다.

이것이 "스케줄 품질 vs 지연"의 트레이드오프 손잡이다. 작은 클러스터는 전부 평가, 큰 클러스터는 표본.

## 병렬화

필터/점수 자체는 노드별로 독립이라 **병렬 실행**된다(`framework/parallelize`,
[01-framework.md](01-framework.md)). 스냅샷이 불변이라 락 없이 병렬 평가가 안전하다 — 직렬 사이클 안에서
노드 평가만 병렬화해 처리량을 높인다.

## 정리

```
사이클 시작: UpdateSnapshot (일관된 노드 시야)
   → 필터(병렬, percentageOfNodes로 조기 종료) → 점수(병렬)
   → assume(낙관적 예약) → 바인딩(비동기)
       성공: 확정 / 실패: ForgetPod로 롤백
```

이 메커니즘들이 "직렬 결정의 일관성"과 "대규모 클러스터의 처리량"을 동시에 달성한다.

## 더 읽을 곳
- [01-framework.md](01-framework.md) — 스케줄링/바인딩 사이클
- [06-scoring.md](06-scoring.md) — 점수 계산
- [03-queue-eventhandlers.md](03-queue-eventhandlers.md) — 실패 Pod 재큐
