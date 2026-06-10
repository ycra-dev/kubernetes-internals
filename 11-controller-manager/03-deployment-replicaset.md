# 11.03 · 심화: Deployment → ReplicaSet → Pod

**근거**: `kubernetes/pkg/controller/deployment/`, `kubernetes/pkg/controller/replicaset/`

가장 흔한 워크로드 흐름이다. **Deployment는 직접 Pod를 만들지 않는다.** 대신 ReplicaSet을 만들고,
ReplicaSet이 Pod를 만든다. 이 2단계 위임이 무중단 롤링 업데이트를 가능케 한다.

```
Deployment ──소유──► ReplicaSet(현재) ──소유──► Pod × N
            (업데이트 시)
           └─소유──► ReplicaSet(이전) ──소유──► Pod × M  (점점 0으로)
```

## ReplicaSet 컨트롤러 — "N개 유지"

[00-foundations/05](../00-foundations/05-controller-pattern.md)에서 추적한 그대로다. 핵심은
`manageReplicas`의 `diff := len(activePods) - int(*(rs.Spec.Replicas))`로 부족/초과를 계산해
생성/삭제하는 것. ReplicaSet은 롤아웃을 모른다 — 그냥 자기 replica 수만 맞춘다.

## Deployment 컨트롤러 — ReplicaSet들을 조율

진입점 `syncDeployment`(`deployment/deployment_controller.go:574`). 핵심 단계:

1. **ReplicaSet 수집과 리비전 동기화**: `getAllReplicaSetsAndSyncRevision`(`deployment/sync.go:124`)이
   이 Deployment 소유의 모든 RS를 모으고, 현재 Pod 템플릿에 해당하는 **새 RS**를 찾거나 만든다
   (`getNewReplicaSet`, `sync.go:146`). 템플릿 해시로 RS를 구분한다.
2. **전략에 따라 롤아웃**:
   - **RollingUpdate**: `rolloutRolling`(`deployment/rolling.go:31`). 새 RS를 점진적으로 키우고
     (`maxSurge` 한도), 이전 RS를 점진적으로 줄인다(`scaleDownOldReplicaSetsForRollingUpdate`,
     `rolling.go:193`, `maxUnavailable` 한도). 한 번에 모두 바꾸지 않으므로 가용성이 유지된다.
   - **Recreate**: 이전 RS를 0으로 줄인 뒤 새 RS를 키운다(다운타임 있음).
3. **status 갱신**: 사용 가능/업데이트된 replica 수 등을 Deployment.status에 보고.

각 단계에서 Deployment는 **RS의 `spec.replicas`만 조정**한다. 실제 Pod 생성/삭제는 ReplicaSet
컨트롤러가 자기 루프에서 한다. 즉 두 컨트롤러가 **공유 객체(ReplicaSet)를 매개로 협력**한다
— 직접 호출이 아니다([00-foundations/02](../00-foundations/02-architecture.md)의 공유 상태 모델).

## 롤백과 히스토리

이전 RS들은 즉시 삭제되지 않고 `revisionHistoryLimit`까지 0-replica 상태로 보존된다. 롤백은 "이전 RS의
템플릿을 다시 새 RS로" 적용하는 것이다(`deployment/rollback.go`, `history/`).

## 엔드투엔드 시나리오

이 흐름의 시간순 추적(트리거 → RS 전환 → Pod 교체)은
[80-scenarios/03-rolling-update.md](../80-scenarios/)에 있다.

## 더 읽을 곳
- [00-foundations/05](../00-foundations/05-controller-pattern.md) — ReplicaSet 코드 추적
- [12-scheduler](../12-scheduler/) — 새로 생긴 Pod가 노드에 배정되는 다음 단계
