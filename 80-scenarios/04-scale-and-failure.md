# 80.04 · 시나리오 — 스케일링과 노드 장애

부하 증가에 따른 자동 확장과, 노드가 죽었을 때의 자동 복구를 추적한다. 둘 다 "여러 컨트롤러가 현재
상태를 보고 수렴"하는 모습을 보여준다.

## A. 부하 증가 → 자동 스케일링

```
[1] 부하 ↑ → Pod CPU 사용률 ↑
[2] HPA: 사용률이 목표 초과 → Deployment.replicas ↑ (예: 3→10)
[3] ReplicaSet 컨트롤러: 부족분 Pod 생성
[4] scheduler: 새 Pod 배정 시도 → 노드 자원 부족 → 일부 Pod Pending(unschedulable)
[5] Cluster Autoscaler: Pending Pod 감지 → 노드 추가(클라우드 API)
[6] 새 노드 Ready → Pending Pod 가 그 노드에 스케줄 → 실행
```

- **[2] HPA**(빌트인, [11-controller-manager/01](../11-controller-manager/01-controller-catalog.md)):
  메트릭을 보고 Deployment의 replica 수를 조정. Pod **개수**를 늘린다.
- **[3]~[4]**: replica가 늘면 ReplicaSet이 Pod를 만들고
  ([00-foundations/05](../00-foundations/05-controller-pattern.md)), 스케줄러가 배정을 시도한다. 자원이
  부족하면 일부는 **Pending**(unschedulable)으로 남는다([12-scheduler/03](../12-scheduler/03-queue-eventhandlers.md)).
- **[5] Cluster Autoscaler**(별도 레포, [70-autoscaling](../70-autoscaling/)): Pending Pod를 감지하고
  "노드를 추가하면 이 Pod가 스케줄될까"를 시뮬레이션해, 맞으면 클라우드 노드 그룹을 키운다.
- **[6]**: 새 노드의 kubelet이 등록/Ready 되면([13-kubelet/06](../13-kubelet/06-node-registration.md)),
  스케줄러가 Pending Pod를 그 노드에 배정한다. 이벤트 핸들러가 "새 노드 추가"를 보고 unschedulable Pod를
  activeQ로 옮긴 결과다([12-scheduler/03](../12-scheduler/03-queue-eventhandlers.md)).

**HPA와 CA의 협력**: HPA는 Pod를, CA는 노드를 늘린다. 둘은 직접 통신하지 않고 "Pending Pod"라는 신호로
연결된다.

## B. 노드 장애 → 자동 복구

```
[1] 노드 다운 → kubelet 하트비트(Lease) 끊김
[2] NodeLifecycle 컨트롤러: 일정 시간 후 노드 NotReady/Unreachable → NoExecute taint
[3] TaintEviction: taint 를 톨러레이트 못 하는 Pod 들을 축출(삭제)
[4] 소유 컨트롤러(ReplicaSet 등): 사라진 Pod 만큼 새 Pod 생성
[5] scheduler: 새 Pod 를 건강한 노드에 배정 → 실행
```

- **[1]~[2]**: 노드의 kubelet은 Lease를 주기적으로 갱신한다
  ([13-kubelet/06](../13-kubelet/06-node-registration.md)). 끊기면 NodeLifecycle 컨트롤러가 노드를
  NotReady/Unreachable로 보고 `NoExecute` taint를 단다
  ([11-controller-manager/05](../11-controller-manager/05-node-lifecycle.md)).
- **[3] TaintEviction**: 그 taint를 견디지 못하는 Pod를 축출한다(`tolerationSeconds` 유예 후). 대규모
  동시 장애 시엔 축출 속도를 제한해 폭주를 막는다.
- **[4]~[5]**: 축출로 Pod가 사라지면, ReplicaSet 등 소유 컨트롤러가 "현재 N개 < 원하는 수"를 보고 새
  Pod를 만든다([00-foundations/05](../00-foundations/05-controller-pattern.md)). 죽은 노드는 후보에서
  빠졌으므로 스케줄러가 **건강한 노드**에 배정한다.

## 핵심 관찰

- 두 시나리오 모두 **중앙 오케스트레이터가 없다.** HPA·CA·ReplicaSet·NodeLifecycle·scheduler가 각자
  자기 객체의 "원하는 상태 vs 현재 상태"만 보고 독립적으로 행동했는데, 그 합이 "자동 확장"과 "자가 치유"가
  된다.
- 연결 고리는 항상 **객체 상태**다: Pending Pod, replica 수, 노드 taint. 컴포넌트는 그것을 watch할 뿐이다
  ([00-foundations/02](../00-foundations/02-architecture.md)).
- **level-triggered**라서 견고하다 — 어떤 컨트롤러가 잠깐 죽어도, 살아나면 현재 상태를 보고 이어간다.

## 더 읽을 곳
- [70-autoscaling](../70-autoscaling/) — HPA/VPA/CA 구분
- [11-controller-manager/05](../11-controller-manager/05-node-lifecycle.md) — 노드 장애 처리
