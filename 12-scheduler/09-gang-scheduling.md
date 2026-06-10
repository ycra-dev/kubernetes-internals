# 12.09 · Gang 스케줄링 (전부 아니면 전무)

**근거**: `kubernetes/pkg/scheduler/framework/plugins/gangscheduling/gangscheduling.go`
(`PodGroupLister` `:50`, `permitTimeoutDuration` `:41`, PodGroup `:47`)

일부 워크로드(분산 학습, MPI, Spark)는 **Pod들이 동시에 다 떠야** 의미가 있다. 절반만 스케줄되면 나머지를
기다리며 자원만 점유한다(데드락/낭비). **Gang 스케줄링**은 "한 그룹의 최소 멤버가 모두 배치 가능할 때만
함께 배치"하는 정책이다.

## 문제 — 부분 스케줄의 함정

기본 스케줄러는 Pod를 **하나씩 독립** 스케줄한다([01-framework.md](01-framework.md)). 8개 Pod가 필요한
학습 작업에서 자원이 5개분만 있으면:

- 5개가 스케줄돼 자원을 점유하고,
- 나머지 3개는 Pending,
- 작업은 8개가 다 있어야 시작하므로 **아무것도 진행 안 되면서 자원 5개분이 묶인다**.

게다가 다른 작업도 그 5개 때문에 자원이 부족해 — 둘 다 못 도는 교착이 생길 수 있다.

## PodGroup과 MinMember

Gang 스케줄링은 Pod들을 **PodGroup**으로 묶는다(`gangscheduling.go:47`, `:50` `PodGroupLister`). PodGroup은
**MinMember**(최소 동시 멤버 수)를 가진다 — "이만큼이 함께 배치될 수 있을 때만 배치하라".

```
PodGroup "training-job" { MinMember: 8 }
   → 8개가 모두 배치 가능할 때만 바인딩
   → 5개만 가능하면 아무도 바인딩 안 함 (자원 점유 없음)
```

## Permit 확장점으로 구현

Gang 스케줄링은 [01-framework.md](01-framework.md)의 **Permit** 확장점으로 구현된다. Permit은 "스케줄
결정은 났지만 **바인딩을 보류**"하는 지점이다:

```
각 Pod가 스케줄 사이클 통과(노드 결정) → Permit:
   "내 PodGroup의 MinMember만큼 Permit에 도달했나?"
   ├─ 아직 부족 → wait (바인딩 보류, 동료 Pod 기다림)
   └─ MinMember 충족 → 그룹 전체를 한꺼번에 allow → 바인딩 진행
```

- 각 Pod는 자기 노드를 정한 뒤 **Permit에서 대기**한다(자원은 assume로 예약,
  [07-internals.md](07-internals.md)).
- 그룹의 충분한 멤버가 모두 Permit에 도달하면 함께 풀려 바인딩된다("전부").
- `permitTimeoutDuration`(`gangscheduling.go:41`) 안에 모이지 못하면 **타임아웃 → 모두 거부**하고
  자원을 반환("전무") — 부분 점유를 막는다.

## assume와의 상호작용

Permit 대기 중에도 그 Pod들은 [07-internals.md](07-internals.md)의 **assume**로 노드 자원을 예약하고
있다. 그래서 동료를 기다리는 동안 다른 Pod가 그 자리를 가로채지 않는다. 타임아웃되면 assume를 취소
(`ForgetPod`)해 자원을 돌려준다.

## 빌트인 vs 외부 코스케줄러

`gangscheduling` 플러그인이 빌트인으로 제공되지만, 더 정교한 배치 정책(큐잉, 공정성, 선점 연계)이
필요하면 **외부 코스케줄러**(예: Volcano, Coscheduling 플러그인)를 별도 스케줄러 프로파일로 쓰기도 한다
([02-plugins.md](02-plugins.md)의 프로파일).

## 더 읽을 곳
- [01-framework.md](01-framework.md) — Permit 확장점
- [07-internals.md](07-internals.md) — assume(Permit 대기 중 자원 예약)
- [04-preemption-extender.md](04-preemption-extender.md) — 자리 부족 시 선점
