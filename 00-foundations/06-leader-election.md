# 00.06 · 리더 선출 (고가용성)

**근거 레포**: `client-go` (`tools/leaderelection/`)

컨트롤 플레인 컴포넌트(controller-manager, scheduler 등)는 보통 여러 복제본으로 떠 있다. 하지만
**한 번에 하나만** 일해야 한다 — 둘이 동시에 같은 객체를 reconcile하면 충돌하기 때문이다
([05](05-controller-pattern.md) 참조). 그래서 "활성 1 + 대기 N(active-standby)" 구조를 쓰고,
누가 활성인지는 **리더 선출**로 정한다.

## 핵심 아이디어: 공유 객체에 임차(lease)를 건다

리더 선출은 별도 합의 프로토콜을 새로 구현하지 않는다. 대신 **apiserver에 저장된 하나의 객체(Lease)를
임차 기록으로 삼는다**. 모든 후보가 같은 Lease 객체를 보고:

- 비어 있거나 임차가 만료됐으면 → 자기가 리더로 갱신(획득).
- 자기가 리더면 → 주기적으로 갱신(갱신).
- 남이 리더면 → 대기.

apiserver의 낙관적 동시성(resourceVersion 기반 조건부 갱신)이 "두 후보가 동시에 획득"을 막는다.

> 솔직한 한계: 패키지 주석(`leaderelection.go:19`)이 명시한다 —
> *"This implementation does not guarantee that only one client is acting as a leader (a.k.a. fencing)."*
> 즉 클럭 스큐/지연으로 짧은 순간 둘이 리더라 믿을 수 있다. 그래서 리더가 하는 작업은
> "한쪽이 잠깐 더 돌아도 치명적이지 않게" 멱등적으로 설계된다.

## 시간 파라미터 세 개

`LeaderElectionConfig`(`leaderelection.go`)의 세 값이 동작을 결정한다:

| 필드 | 의미 |
|------|------|
| `LeaseDuration` | 리더가 아닌 쪽이 "리더가 죽었다"고 간주하기까지 기다리는 임차 기간 |
| `RenewDeadline` | 현재 리더가 이 시간 안에 갱신 못 하면 리더십을 포기 |
| `RetryPeriod` | 획득/갱신 시도 간격 |

검증 규칙(`leaderelection.go:77`)이 관계를 강제한다: `LeaseDuration > RenewDeadline`,
`RenewDeadline > JitterFactor * RetryPeriod`. 이 비율이 허용 가능한 클럭 스큐율을 결정한다(주석 `:31`).

## 동작 흐름

진입점은 `RunOrDie`(`leaderelection.go:227`) 또는 `LeaderElector.Run`(`:211`):

```
Run:
  acquire(ctx)                      // [1] 리더가 될 때까지 RetryPeriod 마다 시도
      └─ tryAcquireOrRenew()  ──► 성공하면 리더
  OnStartedLeading() 콜백 실행       // [2] 여기서 실제 컨트롤러 로직 시작
  renew(ctx)                        // [3] 리더인 동안 계속 갱신
      └─ 갱신 실패(RenewDeadline 초과) → 루프 종료
  OnStoppedLeading() 콜백 실행       // [4] 리더십 상실 → 작업 중단
```

- `acquire`(`:252`): `RetryPeriod` 간격으로 `tryAcquireOrRenew`(`:432`)를 호출, 성공할 때까지 반복.
- `tryAcquireOrRenew`(`:432`): Lease를 Get → 비었으면 Create, 만료됐으면 자기로 Update,
  남이 갖고 있고 유효하면 실패 반환. **로컬 시계로 측정한 변화 시점만** 신뢰한다(주석 `:22` — 기록된
  타임스탬프 자체는 믿지 않음, 클럭 스큐 내성).
- `renew`(`:279`): 리더가 된 뒤 계속 `tryAcquireOrRenew`로 갱신. `RenewDeadline` 안에 못 하면 포기.
- 콜백 `OnStartedLeading`/`OnStoppedLeading`이 "리더가 됐을 때 시작 / 잃었을 때 중단"할 실제 작업이다.
  controller-manager는 여기서 모든 빌트인 컨트롤러를 켜고 끈다.

## 잠금 자원: Lease 객체

선출 상태를 담는 객체는 `resourcelock.Interface`(`resourcelock/interface.go:81`)로 추상화돼 있고,
Get/Create/Update 세 연산만 요구한다(`:83`~`:89`). 표준 구현은 **`LeaseLock`**
(`resourcelock/leaselock.go:31`)으로, `coordination.k8s.io/v1` 의 **Lease** 리소스를 사용한다:

```go
// leaselock.go:43
lease, err := ll.Client.Leases(ll.LeaseMeta.Namespace).Get(ctx, ll.LeaseMeta.Name, ...)
```

> 과거에는 Endpoints/ConfigMap 객체의 annotation을 잠금으로 썼고(패키지 주석 `:18`이 그 흔적),
> 호환을 위한 `multilock.go`도 있다. 현재 권장은 전용 `Lease` 리소스다.

## 어디서 쓰이나

- `kube-controller-manager`, `kube-scheduler`, `cloud-controller-manager` — HA 배포 시 활성 1개만 일함.
- 사용자 오퍼레이터도 controller-runtime의 Manager 옵션으로 동일 메커니즘을 켤 수 있다
  ([60-controller-runtime](../60-controller-runtime/)).

## 더 읽을 곳
- [05-controller-pattern.md](05-controller-pattern.md) — 리더가 켜는 reconcile 루프
- [11-controller-manager/](../11-controller-manager/) — 실제 적용 컴포넌트
- [20-etcd/05-lease-watch.md](../20-etcd/) — etcd 자체의 lease(이름은 같지만 다른 계층)
