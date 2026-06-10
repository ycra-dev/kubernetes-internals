# 11.08 · Job / CronJob 컨트롤러

**근거**: `kubernetes/pkg/controller/job/job_controller.go`,
`kubernetes/pkg/controller/cronjob/cronjob_controllerv2.go`

Deployment/StatefulSet/DaemonSet이 **계속 도는** 서비스를 위한 것이라면, Job은 **끝나는 작업**(배치
처리, 마이그레이션)을 위한 컨트롤러다. CronJob은 그 Job을 **일정에 따라** 만든다.

## Job — 완료까지 실행

진입점 `syncJob`(`job_controller.go:911`). Job은 "Pod가 **성공적으로 끝날 때까지**" 실행한다. 핵심
파라미터:

| 필드 | 의미 |
|------|------|
| `completions` | 성공해야 할 총 횟수 |
| `parallelism` | 동시에 돌릴 Pod 수 |
| `backoffLimit` | 실패 재시도 한도(넘으면 Job 실패) |
| `activeDeadlineSeconds` | 전체 시간 제한 |
| `completionMode` | NonIndexed(아무거나 N개 성공) / **Indexed**(0..N-1 각 인덱스가 성공) |

`manageJob`이 "현재 active Pod 수"와 "필요한 수(parallelism, 남은 completions)"의 차이를 보고 Pod를
만들거나 정리한다 — reconcile 루프([00-foundations/05](../00-foundations/05-controller-pattern.md))인데
**목표가 "N개 유지"가 아니라 "N번 성공"** 이다.

### Indexed Job
`completionMode == Indexed`(`job_controller.go:990`)면 각 Pod가 **고유 인덱스(0..completions-1)** 를
받는다(환경변수/호스트네임으로 노출). 데이터를 인덱스로 분할 처리하는 병렬 작업에 쓴다 — Indexed Job이
StatefulSet처럼 "신원 있는" 배치 워크로드를 가능케 한다.

## 완료/실패 판정 — finishedCondition

Job의 종료는 `finishedCondition`(`job_controller.go:161`, 계산 `:1067~1087`)으로 결정된다:

- **Complete**: 성공 기준 충족(`completions`만큼 성공, 또는 success policy).
- **Failed**: `backoffLimit` 초과(`JobReasonBackoffLimitExceeded`, `:1087`) 또는 Pod failure policy
  발동(`JobReasonPodFailurePolicy`, `:1079`), 또는 deadline 초과.

**Pod failure policy**로 "이 종료 코드는 재시도, 저 코드는 즉시 Job 실패"처럼 실패를 세밀하게 다룰 수 있다.

### finalizer로 정확한 카운팅
Job은 추적 중인 Pod에 finalizer를 달아([00-foundations/05](../00-foundations/05-controller-pattern.md)),
Pod가 컨트롤러가 세기 전에 사라져 **성공/실패 카운트가 누락되는 것**을 막는다(`trackJobStatusAndRemoveFinalizers`).

## 완료된 Job 정리 — TTL

끝난 Job은 자동으로 안 사라진다(로그/상태 확인용). `ttlSecondsAfterFinished`를 주면 **TTLAfterFinished
컨트롤러**([01-controller-catalog.md](01-controller-catalog.md))가 그 시간 뒤 삭제한다.

## CronJob — 일정에 따라 Job 생성

진입점 `syncCronJob`(`cronjob_controllerv2.go:446`). CronJob은 직접 Pod를 만들지 않고, **cron 스케줄에
맞춰 Job 객체를 생성**한다(Deployment→RS처럼 위임). 핵심:

- **스케줄 계산**: `getNextScheduleTime`으로 "지금 실행해야 할 시점이 지났나"를 판단.
- **concurrencyPolicy**: 이전 Job이 아직 도는데 다음 시간이 오면? `Allow`(동시 허용)/`Forbid`(건너뜀)/
  `Replace`(이전 취소).
- **startingDeadlineSeconds**: 컨트롤러가 멈췄다 살아나 시점을 놓쳤을 때, 이 시간 안의 놓친 실행만 보충.
- **히스토리 제한**: `successfulJobsHistoryLimit`/`failedJobsHistoryLimit`로 오래된 Job 정리.

```
CronJob ──(스케줄 도달)──► Job 생성 ──► Pod 실행 ──► 완료 ──(TTL)──► 정리
```

## 더 읽을 곳
- [00-foundations/05](../00-foundations/05-controller-pattern.md) — finalizer 기반 정확한 카운팅
- [01-controller-catalog.md](01-controller-catalog.md) — TTLAfterFinished 컨트롤러
