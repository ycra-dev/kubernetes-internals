# 30.08 · Snapshotter 백엔드 (overlayfs 외)

**근거**: `containerd/plugins/snapshots/` (`overlay/`, `native/`, `btrfs/`, `devmapper/`, `erofs/`,
`blockfile/`, `windows/`), `containerd/core/snapshots/snapshotter.go`

[02-content-snapshots.md](02-content-snapshots.md)에서 snapshotter가 "이미지 레이어 위에 쓰기 계층을 얹어
컨테이너 rootfs를 만든다"고 했다. 이 문서는 그 **Snapshotter 인터페이스의 여러 구현**을 본다. 같은
인터페이스 뒤에 다른 파일시스템 기술이 끼워진다 — CRI/CNI/CSI와 같은 플러그인 철학.

## 같은 인터페이스, 다른 백엔드

`core/snapshots/snapshotter.go`의 `Snapshotter` 인터페이스([02-content-snapshots.md](02-content-snapshots.md)의
`Prepare`/`Mounts`/`Commit`)를 여러 구현이 만족한다(`plugins/snapshots/`):

| 백엔드 | 디렉토리 | 메커니즘 |
|--------|----------|----------|
| **overlayfs** | `overlay/` | 커널 overlay fs(lower=이미지 레이어, upper=쓰기) — 가장 흔함 |
| **native** | `native/` | 레이어를 실제 복사(느림, 호환성 최고) |
| **btrfs** | `btrfs/` | btrfs 서브볼륨/스냅샷 |
| **devmapper** | `devmapper/` | device-mapper 씬 프로비저닝(블록 수준) |
| **erofs** | `erofs/` | 읽기 전용 압축 fs(EROFS) |
| **blockfile** | `blockfile/` | 블록 디바이스 파일 |
| **windows** | `windows/` | Windows 레이어 |

## overlayfs — 기본 동작

가장 널리 쓰이는 overlayfs(`overlay/`)의 원리:

```
overlay 마운트:
  lowerdir = [이미지 레이어들, 읽기 전용]   ← content store에서 unpack된 committed snapshot
  upperdir = 컨테이너 쓰기 계층               ← Prepare로 만든 active snapshot
  merged   = 컨테이너가 보는 rootfs           ← lower+upper 합쳐 보임
```

- 컨테이너가 파일을 읽으면 upper에 없으면 lower에서.
- 쓰면 upper에만 기록(**copy-up**: lower의 파일을 수정하면 upper로 복사 후 수정).
- 컨테이너 삭제 시 upper(active snapshot)만 버리면 되고, 공유 lower(이미지 레이어)는 그대로
  ([02-content-snapshots.md](02-content-snapshots.md)).

이 덕분에 같은 이미지로 컨테이너 100개를 띄워도 이미지 레이어는 **한 번만** 디스크에 있고, 각 컨테이너는
얇은 upper만 갖는다 — 빠른 시작과 디스크 효율.

## 왜 여러 백엔드인가

- **overlayfs**: 대부분의 리눅스에서 최선(빠르고 커널 내장).
- **devmapper/btrfs**: overlay가 안 맞는 환경, 블록 수준 격리/스냅샷이 필요할 때.
- **native**: 특수 파일시스템 기능이 없어도 동작(복사 기반, 최후 수단).
- **erofs**: 읽기 전용 압축으로 이미지 크기/메모리 절감.

운영자는 노드의 파일시스템/요구에 맞는 snapshotter를 containerd 설정으로 선택한다.

## 스냅샷 트리

snapshot들은 **부모-자식 트리**를 이룬다 — 각 이미지 레이어가 부모 위에 쌓이고
([02-content-snapshots.md](02-content-snapshots.md)), 컨테이너의 active snapshot이 최상위 이미지 레이어를
부모로 갖는다. 이 트리가 [07-metadata-gc.md](07-metadata-gc.md)의 GC가 추적하는 참조 그래프의 일부다 —
부모 snapshot은 자식이 있는 한 GC되지 않는다.

## 더 읽을 곳
- [02-content-snapshots.md](02-content-snapshots.md) — snapshot으로 rootfs 만들기
- [07-metadata-gc.md](07-metadata-gc.md) — snapshot GC
- [03-runtime-shim-oci.md](03-runtime-shim-oci.md) — snapshot을 OCI 번들 rootfs로
