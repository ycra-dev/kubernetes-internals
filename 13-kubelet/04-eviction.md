# 13.04 · 축출(Eviction)과 이미지 GC

**근거**: `kubernetes/pkg/kubelet/eviction/`, `kubernetes/pkg/kubelet/images/`

노드의 메모리/디스크가 부족해지면, kubelet은 노드를 보호하기 위해 **Pod를 능동적으로 축출**한다. 이는
스케줄러의 일이 아니라 노드 로컬 결정이다(이미 떠 있는 Pod를 쫓아냄).

## 신호(signal)와 임계값

eviction manager(`eviction/eviction_manager.go`)는 자원 신호를 감시한다. 대표 신호
(`eviction_manager.go:196` 등):

| 신호 | 의미 |
|------|------|
| `memory.available` (`SignalMemoryAvailable`) | 가용 메모리 |
| `nodefs.available` (`SignalNodeFsAvailable`) | 노드 파일시스템 여유 |
| `imagefs.available` (`SignalImageFsAvailable`) | 이미지/컨테이너 저장소 여유 |
| `pid.available` | 가용 PID |

운영자가 `--eviction-hard`/`--eviction-soft`로 임계값을 정한다. hard는 즉시, soft는 유예 후 축출.

## synchronize — 축출 결정 루프

`synchronize`(`eviction_manager.go:256`)가 주기적으로:

1. 현재 신호 값을 임계값과 비교.
2. 임계값을 넘었으면 먼저 **노드 수준 회수**를 시도(`reclaimNodeLevelResources`, `:481`) — 예: 안 쓰는
   이미지/죽은 컨테이너 정리로 디스크 확보. 이걸로 충분하면 Pod 축출 없이 끝.
3. 그래도 부족하면 **Pod를 골라 축출**한다. 선택 순서:
   - **QoS**: BestEffort → Burstable → Guaranteed ([03](03-resource-mgmt.md)).
   - 같은 등급 안에서는 **요청 대비 초과 사용량**이 큰 Pod부터.
   - Pod 우선순위(`priority`)도 고려.

축출된 Pod는 status가 Failed가 되고, 소유 컨트롤러가 다른 노드에 다시 만든다. 즉 노드 보호와 워크로드
복원이 분리돼 협력한다.

## 이미지 GC

`pkg/kubelet/images/`의 이미지 GC는 디스크 사용률이 상한을 넘으면 **안 쓰는 이미지**를 LRU로 지운다
(하한까지). 위 `reclaimNodeLevelResources`가 imagefs 압박 시 이를 트리거한다. 컨테이너 GC도 죽은
컨테이너를 정리한다.

## 축출 vs API eviction

이름이 같지만 다른 두 가지:
- **kubelet eviction**(이 문서): 노드 자원 압박으로 kubelet이 로컬에서 Pod를 죽임.
- **API-initiated eviction**: `kubectl drain` 등이 Eviction API로 요청, PodDisruptionBudget을 존중하며
  컨트롤 플레인이 처리(노드 비우기). → [11-controller-manager/01](../11-controller-manager/01-controller-catalog.md)의 Disruption 컨트롤러.

## 더 읽을 곳
- [03-resource-mgmt.md](03-resource-mgmt.md) — QoS 등급(축출 순서의 근거)
- [11-controller-manager/05](../11-controller-manager/05-node-lifecycle.md) — 노드 장애 기반 축출(다른 경로)
