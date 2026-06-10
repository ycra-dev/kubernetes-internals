# 50.02 · CSI 드라이버 프로토콜

**근거**: `kubernetes/pkg/volume/csi/` (`csi_attacher.go`, `csi_mounter.go` `SetUp`/`SetUpAt` `:99/:103`,
`csi_client.go`, `csi_plugin.go`)

CSI(Container Storage Interface)는 스토리지 벤더가 **별도 드라이버**로 Kubernetes에 스토리지를 제공하는
표준 gRPC 인터페이스다. CRI/CNI와 같은 철학 — Kubernetes는 인터페이스만 정하고 구현은 드라이버에
맡긴다.

## CSI 드라이버의 두 부분

CSI 드라이버는 두 컴포넌트로 배포된다:

| 부분 | 어디서 도나 | gRPC 서비스 | 역할 |
|------|-------------|-------------|------|
| **Controller 플러그인** | 클러스터에 1개(Deployment) | Controller Service | 볼륨 생성/삭제, attach/detach |
| **Node 플러그인** | 모든 노드(DaemonSet) | Node Service | 노드에서 stage/mount |

Kubernetes의 in-tree 코드(`pkg/volume/csi/`)는 이 드라이버들을 호출하는 **클라이언트**다(`csi_client.go`).

## 볼륨이 컨테이너에 붙는 3단계

CSI는 볼륨 사용을 세 단계로 나눈다. PV가 Pod에 도달하기까지:

```
[1] CreateVolume        (Controller)  ─ 실제 스토리지(디스크) 생성  → 동적 프로비저닝 (50.01)
[2] ControllerPublishVolume (Controller) ─ 디스크를 노드에 attach   → 클라우드 attach
[3] NodeStageVolume     (Node)        ─ 노드에 전역 mount(포맷 포함)
[4] NodePublishVolume   (Node)        ─ 컨테이너 경로에 bind mount
```

- **[2] attach**: VolumeAttachment 객체를 통해 일어난다. `csi_attacher.go`가 이를 다룬다 —
  AttachDetach 컨트롤러([11-controller-manager/01](../11-controller-manager/01-controller-catalog.md))가
  VolumeAttachment를 만들면, 노드의 CSI가 실제 attach를 수행.
- **[3]~[4] mount**: kubelet volume manager([13-kubelet/05](../13-kubelet/05-volume-manager.md))가
  `csi_mounter.go`의 `SetUp`/`SetUpAt`(`:99`/`:103`)을 호출 → Node 플러그인의 NodeStage/NodePublish
  를 gRPC로 부른다.

`SetUpAt`(`csi_mounter.go:103`)이 VolumeAttachment를 찾아(`:196`) attach 완료를 확인한 뒤 mount를
진행하는 흐름이 코드에 보인다.

## 드라이버 등록 — kubelet plugin watcher

노드의 CSI Node 플러그인은 kubelet의 **plugin 디렉토리**(유닉스 소켓)에 자기를 등록한다
(`csi_plugin.go`, `csi_drivers_store.go`). kubelet은 이 소켓을 watch해 어떤 CSI 드라이버가 노드에
있는지 안다. 등록되면 그 드라이버가 담당하는 볼륨을 위 단계로 호출할 수 있다.

## CSI migration — in-tree에서 CSI로

과거 in-tree 볼륨 플러그인(awsEBS, gcePD 등)은 코어에 박혀 있었다. **CSI migration**
(`pkg/volume/csimigration/`)은 이 레거시 볼륨 요청을 투명하게 해당 **CSI 드라이버 호출로 리디렉션**한다.
사용자 매니페스트는 그대로 두고 내부적으로 CSI 경로를 쓴다 — 코어에서 벤더 코드를 제거하기 위한 전환
([16-cloud-controller-manager](../16-cloud-controller-manager.md)의 out-of-tree화와 같은 동기).
변환 규칙은 `staging/.../csi-translation-lib`.

## 부가 기능

- **확장(expand)**: PVC 용량을 키우면 `expander.go`가 ControllerExpandVolume/NodeExpandVolume 호출.
- **블록 볼륨**: `csi_block.go`로 파일시스템 없는 raw 블록 디바이스 제공.
- **스냅샷/클론**: VolumeSnapshot CRD + CSI 스냅샷 기능(드라이버 지원 시).

## 더 읽을 곳
- [01-pv-binding.md](01-pv-binding.md) — 동적 프로비저닝(CreateVolume 호출처)
- [13-kubelet/05](../13-kubelet/05-volume-manager.md) — mount를 호출하는 노드 측
