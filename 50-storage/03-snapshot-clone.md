# 50.03 · 볼륨 스냅샷과 클론

**근거**: `kubernetes/pkg/apis/core/types.go`(PVC `DataSource`/`DataSourceRef`),
`kubernetes/pkg/volume/csi/`(CSI 스냅샷 연동), external-snapshotter(별도 컨트롤러, CRD)

[02-csi.md](02-csi.md)에서 CSI 드라이버가 볼륨 생성/마운트를 한다고 했다. 이 문서는 그 위의 **데이터
관리 기능** — 스냅샷(특정 시점 백업)과 클론(기존 볼륨 복제) — 을 본다. 스테이트풀 워크로드의 백업/복구에
쓰인다.

## PVC의 dataSource — "무엇으로부터 만드나"

핵심은 PVC의 `DataSource`/`DataSourceRef` 필드(`pkg/apis/core/types.go`)다. PVC를 만들 때 "빈 볼륨"이
아니라 **무언가로부터** 채워진 볼륨을 요청할 수 있다:

| dataSource | 의미 |
|------------|------|
| (없음) | 빈 볼륨 (기본) |
| **VolumeSnapshot** | 스냅샷에서 복원한 볼륨 |
| **PersistentVolumeClaim** | 기존 PVC를 복제(클론)한 볼륨 |

```
PVC { dataSource: VolumeSnapshot "db-backup-monday" }
   → 그 스냅샷 시점의 데이터를 담은 새 볼륨 생성
```

## VolumeSnapshot — CRD + 외부 컨트롤러

**VolumeSnapshot**은 빌트인이 아니라 **CRD**다([10-apiserver/07](../10-apiserver/07-crd-aggregation.md)).
`external-snapshotter`(별도 프로젝트의 컨트롤러 + CRD)가 제공한다:

- `VolumeSnapshotClass`: 스냅샷 방법(어느 CSI 드라이버/파라미터). StorageClass의 스냅샷 버전.
- `VolumeSnapshot`: "이 PVC의 스냅샷을 떠라"는 요청.
- `VolumeSnapshotContent`: 실제 스냅샷(PV에 대응).

external-snapshotter 컨트롤러가 이를 watch해([00-foundations/05](../00-foundations/05-controller-pattern.md))
CSI 드라이버의 **CreateSnapshot**을 호출한다. PV/PVC 모델([01-pv-binding.md](01-pv-binding.md))과 똑같은
구조 — VolumeSnapshot(요구) ↔ VolumeSnapshotContent(공급)를 바인딩.

```
VolumeSnapshot (요구)  ↔  VolumeSnapshotContent (공급)   ← PVC↔PV와 동형
   external-snapshotter 컨트롤러 + CSI CreateSnapshot
```

## CSI가 실제 작업

스냅샷/클론의 실제 수행은 CSI 드라이버다([02-csi.md](02-csi.md)):

- **CreateSnapshot / DeleteSnapshot**(Controller 서비스): 스토리지 백엔드에서 스냅샷 생성/삭제.
- 스냅샷에서 복원/클론: `CreateVolume`에 소스(스냅샷/볼륨)를 지정해 새 볼륨을 그 데이터로 채운다.

드라이버가 이 기능을 지원해야 한다(모든 CSI 드라이버가 스냅샷/클론을 지원하진 않음 — capability로 광고).

## 클론 vs 스냅샷

| | 스냅샷 | 클론 |
|--|--------|------|
| 무엇 | 특정 시점의 백업(별도 객체로 보존) | 기존 볼륨을 즉시 복제한 새 볼륨 |
| 용도 | 백업/복구, 시점 복원 | 테스트 데이터 복제, 빠른 환경 복제 |
| dataSource | VolumeSnapshot | PersistentVolumeClaim |

## 왜 별도(out-of-tree)인가

VolumeSnapshot이 CRD + 외부 컨트롤러인 이유는 [16-cloud-controller-manager](../16-cloud-controller-manager.md),
[02-csi.md](02-csi.md)의 CSI migration과 같은 철학이다 — **스토리지 기능을 코어에서 분리**해, 벤더가 독립
배포하고 코어 릴리스에 묶이지 않게 한다. 코어는 PVC의 dataSource 필드라는 **연결점**만 제공한다.

## 더 읽을 곳
- [01-pv-binding.md](01-pv-binding.md) — PV/PVC 바인딩(같은 구조)
- [02-csi.md](02-csi.md) — CSI 드라이버(스냅샷 수행)
- [11-controller-manager/06](../11-controller-manager/06-statefulset.md) — 스냅샷이 유용한 스테이트풀 워크로드
