# 11.05 · 노드 라이프사이클

**근거**: `kubernetes/pkg/controller/nodelifecycle/`, `kubernetes/pkg/controller/tainteviction/`

노드가 죽거나 도달 불가능해졌을 때, 그 위의 Pod를 어떻게 처리할지 결정하는 컨트롤러다. kubelet이
보내는 **노드 하트비트**를 감시하고, 끊기면 노드를 taint하고 결국 Pod를 축출한다.

## 입력: 노드 하트비트

각 노드의 kubelet은 주기적으로 자기 상태를 apiserver에 보고한다(Node 객체의 condition 및 Lease 갱신
→ [13-kubelet](../13-kubelet/)). NodeLifecycle 컨트롤러는 이를 watch한다.

## monitorNodeHealth — 건강 추적

핵심 루프는 `node_lifecycle_controller.go`의 노드 건강 모니터링이다. 하트비트가 일정 시간 안 오면
노드의 `Ready` condition을 `Unknown`으로 보고, 두 가지 taint 중 하나를 단다
(`node_lifecycle_controller.go:74`~`:93`):

| 상황 | Taint |
|------|-------|
| 노드가 NotReady (Ready=False) | `NotReadyTaintTemplate` (`node.kubernetes.io/not-ready`, `:82`) |
| 노드 도달 불가 (Ready=Unknown) | `UnreachableTaintTemplate` (`node.kubernetes.io/unreachable`, `:74`) |

둘 다 `NoExecute` 효과의 taint다 — "이 노드의 Pod를 (톨러레이션 없으면) 쫓아내라"는 의미.

## taint → eviction

NodeLifecycle 컨트롤러는 내부에 **TaintEviction 컨트롤러**(`tainteviction.Controller`)를 들고 있다
(`node_lifecycle_controller.go:225`, 시작은 `Run`에서 `nc.taintManager.Run(ctx)`, `:477`). 동작:

- `NoExecute` taint가 붙은 노드의 Pod를 검사.
- Pod가 그 taint를 **톨러레이트하지 않으면** 축출(삭제)한다. 톨러레이션에 `tolerationSeconds`가 있으면
  그 시간만큼 유예 후 축출.

이렇게 축출된 Pod는, 그 Pod를 소유한 컨트롤러(ReplicaSet 등)가 다시 만들어 다른 건강한 노드에
스케줄되게 한다 — 즉 "노드 장애 → 자동 재배치"가 여러 컨트롤러의 협력으로 일어난다.

## 점진적 축출 (rate limiting)

대규모 노드가 동시에 NotReady가 되면(예: 네트워크 분할) 모든 Pod를 한꺼번에 쫓아내는 것은 위험하다.
컨트롤러는 존(zone)별 상태를 보고 축출 속도를 제한한다 — 다수가 동시에 죽으면 오히려 축출을 늦춰
"전체가 비워지는" 사태를 막는다.

## 관련 컨트롤러

- **NodeIPAM**(`nodeipam/`): 새 노드에 Pod CIDR를 할당.
- **PodGC**(`podgc/`): 노드가 사라진 뒤 남은 고아 Pod 객체를 정리.

## 엔드투엔드

노드 장애 → taint → eviction → 재스케줄의 시간순 추적은
[80-scenarios/04-scale-and-failure.md](../80-scenarios/).

## 더 읽을 곳
- [13-kubelet/06-node-registration.md](../13-kubelet/) — 하트비트를 보내는 쪽
- [12-scheduler](../12-scheduler/) — 축출된 Pod가 재배정되는 곳
