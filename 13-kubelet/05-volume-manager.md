# 13.05 · 볼륨 매니저

**근거**: `kubernetes/pkg/kubelet/volumemanager/`, `kubernetes/pkg/volume/csi/`

Pod가 볼륨(PVC, ConfigMap, Secret, emptyDir 등)을 요구하면, 컨테이너가 시작되기 전에 그 볼륨이 노드에
**attach + mount** 돼야 한다. 이를 책임지는 것이 kubelet의 volume manager다.

## 원하는 상태 / 실제 상태 모델

volume manager는 두 캐시를 둔다(`volumemanager/cache/`):

- **desiredStateOfWorld**: "이 노드의 Pod들이 요구하는 볼륨"(populator가 Pod 목록에서 채움,
  `volumemanager/populator/`).
- **actualStateOfWorld**: "지금 실제로 attach/mount된 볼륨".

**reconciler**(`volumemanager/reconciler/reconciler.go`)가 둘을 비교해 차이를 메운다 — 또 하나의
reconcile 루프다. `reconcile`(`reconciler.go:33`)이 `mountOrAttachVolumes`(`:48`)로:

- desired엔 있는데 actual엔 없으면 → **attach**(노드에 디바이스 연결) → **mount**(파일시스템 마운트).
- actual엔 있는데 desired엔 없으면(Pod 삭제됨) → **unmount** → **detach**.

컨테이너 시작([01](01-pod-lifecycle.md)의 SyncPod)은 필요한 볼륨이 mount될 때까지 기다린다.

## 볼륨 플러그인과 CSI

볼륨 종류별 동작은 플러그인으로 구현된다(`pkg/volume/`). 오늘날 외부 스토리지는 거의 **CSI(Container
Storage Interface)** 로 통합된다(`pkg/volume/csi/`):

- CSI는 CRI/CNI와 같은 철학 — 스토리지 벤더가 **별도 드라이버**(CSI 노드 플러그인)를 제공하고, kubelet은
  표준 gRPC로 호출한다(NodeStageVolume/NodePublishVolume 등).
- in-tree 레거시 플러그인은 **CSI migration**(`pkg/volume/csimigration/`)으로 점차 CSI 드라이버로
  리디렉션된다.

## attach는 누구 일인가

- **node 수준 mount**: kubelet volume manager(이 문서).
- **attach/detach**: 노드의 kubelet이 할 수도, 컨트롤 플레인의 **AttachDetach 컨트롤러**
  ([11-controller-manager/01](../11-controller-manager/01-controller-catalog.md))가 할 수도 있다
  (`--enable-controller-attach-detach`). 클라우드 디스크 attach는 보통 컨트롤러가 담당.
- **PVC↔PV 바인딩과 동적 프로비저닝**: 컨트롤 플레인의 PV Binder 컨트롤러 → [50-storage](../50-storage/).

## 스케줄러와의 연계

볼륨이 특정 노드(존)에만 있을 수 있으므로, 스케줄러의 `volumebinding` 플러그인
([12-scheduler/02](../12-scheduler/02-plugins.md))이 "이 노드에 이 PVC를 붙일 수 있나"를 미리 확인하고,
PreBind 단계에서 바인딩을 확정한다.

## 더 읽을 곳
- [50-storage](../50-storage/) — PV/PVC/StorageClass와 CSI 프로비저닝 전체
- [01-pod-lifecycle.md](01-pod-lifecycle.md) — 볼륨 mount를 기다리는 컨테이너 시작
