# 50 · 스토리지 모델

**근거 레포**: `kubernetes` (`pkg/controller/volume/`, `pkg/volume/`, `pkg/kubelet/volumemanager/`)

Kubernetes 스토리지는 **"무엇이 필요한가(PVC)"와 "무엇이 있는가(PV)"를 분리**하고, 그 사이를 컨트롤러가
잇는 선언형 모델이다. 컨테이너가 디스크를 직접 다루지 않고, 추상화된 볼륨을 마운트한다.

## 세 개의 객체

| 객체 | 의미 | 누가 만드나 |
|------|------|-------------|
| **PersistentVolume (PV)** | 실제 스토리지 조각(디스크, NFS 공유 등) | 운영자 또는 동적 프로비저닝 |
| **PersistentVolumeClaim (PVC)** | "이만큼의 스토리지가 필요하다"는 요청 | 사용자 |
| **StorageClass** | 동적 프로비저닝 방법(어떤 프로비저너/파라미터) | 운영자 |

사용자는 PVC를 만들고, Pod는 그 PVC를 참조한다. PVC가 적절한 PV에 **바인딩**되면 Pod가 그 볼륨을 쓴다.

## 동적 프로비저닝 흐름

```
사용자: PVC 생성 (storageClassName: fast)
   │
PV Binder 컨트롤러 (pkg/controller/volume/persistentvolume/)
   ├─ 맞는 기존 PV 있으면 → 바인딩
   └─ 없으면 → StorageClass의 프로비저너에게 PV 생성 요청 (CSI)
                → 새 PV 생성 후 PVC에 바인딩
   │
스케줄러: Pod를 PVC가 붙을 수 있는 노드에 배정 (volumebinding 플러그인)
   │
kubelet: 볼륨 attach + mount → 컨테이너 시작
   (pkg/kubelet/volumemanager/, → 13-kubelet/05)
```

## 컨트롤 플레인 vs 노드 — 역할 분담

스토리지는 두 계층이 협력한다:

| 계층 | 컴포넌트 | 책임 |
|------|----------|------|
| 컨트롤 플레인 | **PV Binder** 컨트롤러 ([11-controller-manager](../11-controller-manager/01-controller-catalog.md)) | PVC↔PV 바인딩, 동적 프로비저닝 |
| 컨트롤 플레인 | **AttachDetach** 컨트롤러 | (클라우드 디스크 등) 노드에 볼륨 attach |
| 노드 | **kubelet volume manager** ([13-kubelet/05](../13-kubelet/05-volume-manager.md)) | mount/unmount, 컨테이너에 노출 |
| 스케줄러 | **volumebinding** 플러그인 ([12-scheduler/02](../12-scheduler/02-plugins.md)) | "이 노드에 이 PVC를 붙일 수 있나" |

## CSI — Container Storage Interface

오늘날 외부 스토리지는 거의 **CSI**로 통합된다(`pkg/volume/csi/`). CRI/CNI와 같은 철학 — 스토리지
벤더가 **CSI 드라이버**를 제공하고, Kubernetes는 표준 gRPC로 호출한다:

- **Controller 플러그인**: 볼륨 생성/삭제/attach(CreateVolume, ControllerPublishVolume).
- **Node 플러그인**: 노드에서 stage/mount(NodeStageVolume, NodePublishVolume) — kubelet이 호출.

in-tree 레거시 볼륨 플러그인은 **CSI migration**(`pkg/volume/csimigration/`)으로 점차 CSI 드라이버
호출로 리디렉션된다.

## 볼륨의 다른 종류

PV/PVC만 볼륨이 아니다. Pod는 여러 볼륨 타입을 마운트한다:
- **ConfigMap / Secret**: 설정/비밀을 파일로 주입.
- **emptyDir**: Pod 수명 동안의 임시 공간.
- **projected**: SA 토큰 등 여러 소스를 한 디렉토리로.
- **hostPath**: 노드 경로(보안 주의).

이들도 kubelet volume manager가 mount한다([13-kubelet/05](../13-kubelet/05-volume-manager.md)).

## 문서

| # | 문서 | 내용 |
|---|------|------|
| 01 | [pv-binding.md](01-pv-binding.md) | PV Binder 컨트롤러, 동적 프로비저닝, 회수 |
| 02 | [csi.md](02-csi.md) | CSI 드라이버 프로토콜(attach/stage/publish), migration |
| 03 | [snapshot-clone.md](03-snapshot-clone.md) | 볼륨 스냅샷/클론(dataSource, external-snapshotter) |
| 04 | [ephemeral-volumes.md](04-ephemeral-volumes.md) | emptyDir/CSI inline/generic ephemeral |

## 더 읽을 곳
- [13-kubelet/05](../13-kubelet/05-volume-manager.md) — 노드 측 mount
- [12-scheduler/02](../12-scheduler/02-plugins.md) — volumebinding 스케줄 연계
