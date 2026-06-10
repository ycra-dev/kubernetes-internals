# 11.07 · DaemonSet 컨트롤러

**근거**: `kubernetes/pkg/controller/daemon/` (`daemon_controller.go`, `update.go`)

DaemonSet은 **모든(또는 선택된) 노드마다 Pod를 정확히 하나씩** 띄운다. 노드 에이전트형 워크로드 — 로그
수집기, 모니터링, CNI(cilium), kube-proxy, 스토리지 드라이버 — 가 이 방식이다.

## ReplicaSet과의 차이: 노드가 곧 replica 수

ReplicaSet은 "N개"를 만들고 스케줄러가 어디 둘지 정한다. DaemonSet은 다르다:

- replica 수를 사람이 정하지 않는다 — **노드 수가 곧 Pod 수**.
- 노드가 추가되면 그 노드에 자동으로 Pod가 생기고, 노드가 빠지면 그 Pod도 사라진다.

## syncDaemonSet — 노드별로 맞추기

진입점 `syncDaemonSet`(`daemon_controller.go:1270`) → `manage`(`:998`). 동작:

1. **노드 목록을 순회**하며, 각 노드에 대해 `nodeShouldRunDaemonPod`로 "이 노드에 이 DS의 Pod가 떠
   있어야 하나"를 판단한다 — nodeSelector, affinity, **taint 톨러레이션**을 본다.
2. 떠 있어야 하는데 없으면 → 그 노드에 Pod 생성. 떠 있으면 안 되는데 있으면 → 삭제.

즉 "원하는 상태 = 적격 노드마다 Pod 1개"와 현재 상태의 차이를 좁히는 reconcile 루프
([00-foundations/05](../00-foundations/05-controller-pattern.md))인데, **기준이 replica 수가 아니라
노드 집합**이다.

## 스케줄링 — 스케줄러를 쓰되 노드 고정

DaemonSet Pod도 일반 Pod처럼 스케줄러가 바인딩하지만, DaemonSet 컨트롤러가 Pod 생성 시 **nodeAffinity로
대상 노드를 못박는다**. 그래서 스케줄러는 사실상 그 노드에만 배정한다. taint가 있는 노드(예: 컨트롤
플레인)에도 띄우려면 DaemonSet에 톨러레이션을 넣는다 — 그래서 cilium/kube-proxy 같은 시스템 DaemonSet은
대개 광범위한 톨러레이션을 갖는다.

## 롤링 업데이트

업데이트(`update.go`, `updateDaemonSet` `:968`)도 노드별로 점진적이다. `maxUnavailable`(과 `maxSurge`)로
한 번에 몇 노드의 Pod를 교체할지 제한해, 모든 노드의 에이전트가 동시에 죽지 않게 한다. StatefulSet과 같이
**ControllerRevision**(`history/`)으로 리비전을 추적해 롤백을 지원한다.

## 상태 보고

`updateDaemonSetStatus`(`daemon_controller.go:1202`)가 desired/current/ready/available 노드 수 등을
status에 기록한다. "몇 개 노드에 떠야 하고 몇 개가 실제로 Ready인지"를 보고한다.

## 더 읽을 곳
- [11-controller-manager/05](05-node-lifecycle.md) — 노드 추가/삭제(DaemonSet의 트리거)
- [06-statefulset.md](06-statefulset.md) — ControllerRevision을 공유하는 또 다른 컨트롤러
