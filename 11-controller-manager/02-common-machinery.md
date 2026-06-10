# 11.02 · 공통 머시너리

**근거**: `kubernetes/pkg/controller/controller_utils.go`, `controller_ref_manager.go`

빌트인 컨트롤러들은 같은 보조 코드를 공유한다. 핵심 세 가지: **PodControl**(객체 생성/삭제 헬퍼),
**Expectations**(캐시 지연 보정), **ControllerRefManager**(소유 기반 입양/포기).

## RealPodControl — 생성/삭제 헬퍼

`controller_utils.go:540`의 `RealPodControl.CreatePods`(및 `CreatePodsWithGenerateName`, `:544`)는
모든 워크로드 컨트롤러가 Pod를 만들 때 쓰는 공통 경로다. 생성 시 **controllerRef(ownerReference)** 를
반드시 박는다 — 그래야 나중에 역참조와 GC가 가능하다([00-foundations/05](../00-foundations/05-controller-pattern.md)).

## slow start — 폭주 방지

`SlowStartInitialBatchSize`(`controller_utils.go:87`)와 `slowStartBatch`는 한 번에 모든 Pod를 만들지
않고 1개 → 2개 → 4개 ... 지수적으로 배치를 키운다. 초기 배치가 실패하면(예: 쿼터 초과) 일찍 멈춰
apiserver에 무더기 실패 요청을 쏟지 않는다.

## Expectations — level-triggered 루프의 보정 장치

문제: 컨트롤러가 Pod 5개 생성을 요청했는데, Informer 캐시에는 아직 안 보인다. level-triggered 루프가
다시 돌면 "여전히 0개네, 또 5개 만들자"고 **과잉 생성**할 수 있다.

해결: `ControllerExpectations`(`controller_utils.go:169`). 컨트롤러는 생성/삭제를 요청하면서 "이만큼
생기길/사라지길 기대한다"고 기록(`ExpectCreations`)하고, 캐시에서 그만큼 관측될 때까지 다음 sync를
건너뛴다. `ControlleeExpectations`(`:273`)/`UIDTrackingControllerExpectations`(`:340`)가 카운트를
관리한다. 즉 **캐시 지연으로 인한 중복 행동을 막는** 장치다.

> [00-foundations/05](../00-foundations/05-controller-pattern.md)에서 ReplicaSet의
> `expectations.ExpectCreations`가 바로 이것이다.

## ControllerRefManager — 입양과 포기

`controller_ref_manager.go`. 셀렉터에 맞지만 ownerReference가 없는 Pod를 **입양(adopt)** 하고,
셀렉터에 안 맞게 된 Pod를 **포기(release)** 한다. 예: ReplicaSet의 라벨 셀렉터에 맞는 고아 Pod를
발견하면 자기 소유로 삼는다. 이로써 "라벨이 맞는 N개" 불변식을 유지한다.

## 더 읽을 곳
- [03-deployment-replicaset.md](03-deployment-replicaset.md) — 이 머시너리가 실제로 쓰이는 흐름
