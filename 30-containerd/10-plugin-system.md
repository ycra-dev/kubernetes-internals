# 30.10 · 플러그인 시스템

**근거**: `containerd/plugins/` (각 `*/plugin.go`의 `registry.Register`),
`containerd/core/plugin/`, `containerd/plugins/types.go`

[README](README.md)에서 "containerd는 플러그인 조합으로 구성된 데몬"이라 했다. 이 문서는 그 **플러그인
시스템** — 모든 서브시스템이 어떻게 플러그인으로 등록·초기화·연결되는지 — 를 본다. containerd의 확장성과
모듈성의 근간이다.

## 거의 모든 것이 플러그인

content store, snapshotter, CRI 서버, metadata, GC, events, NRI, runtime... 전부 **플러그인**이다
(`plugins/` 아래 각 `plugin.go`). 데몬 자체는 작고, 플러그인들을 로드·연결하는 골격이다. 그래서:

- 필요한 기능만 켤 수 있다(예: CRI 안 쓰면 끔).
- 서드파티가 새 snapshotter/runtime을 플러그인으로 추가할 수 있다.

## Registration — 플러그인 선언

각 플러그인은 `init()`에서 자기를 등록한다(`plugins/*/plugin.go`의 `registry.Register(...)`). 등록 정보
(`Registration`)에 담기는 것:

| 필드 | 의미 |
|------|------|
| **Type** | 플러그인 종류(예: `content`, `snapshot`, `grpc`, `service`, `gc`) |
| **ID** | 같은 Type 내 식별자(예: snapshot의 `overlayfs`/`btrfs`) |
| **Requires** | 의존하는 다른 플러그인 Type들 |
| **InitFn** | 초기화 함수 — 의존성을 받아 플러그인 인스턴스 생성 |

플러그인 Type 상수는 `plugins/types.go`에 정의된다.

## 의존성 그래프와 초기화 순서

`Requires`가 **플러그인 간 의존성 그래프**를 만든다. 예: CRI 플러그인은 content/snapshot/metadata/runtime
플러그인을 필요로 한다([01-cri-plugin.md](01-cri-plugin.md)). containerd는 이 그래프를 **위상 정렬**해:

```
초기화 순서(의존성 먼저):
  content, metadata(bbolt) → snapshotter → runtime → services → CRI plugin → grpc
```

각 플러그인의 `InitFn`은 자기가 `Requires`한 플러그인 인스턴스를 주입받아 초기화한다 — Kubernetes의
controller-manager가 공유 informer를 컨트롤러들에 주입하는 것
([11-controller-manager/README](../11-controller-manager/README.md))과 유사한 의존성 주입이다.

## 플러그인 Type 예시

| Type | 역할 | 문서 |
|------|------|------|
| `content` | blob 저장 | [02-content-snapshots](02-content-snapshots.md) |
| `snapshot` | rootfs 스냅샷(overlayfs 등) | [08-snapshotters](08-snapshotters.md) |
| `metadata` | bbolt 메타 저장 | [07-metadata-gc](07-metadata-gc.md) |
| `gc` | 가비지 컬렉션 | [07-metadata-gc](07-metadata-gc.md) |
| `runtime` | shim 관리 | [03-runtime-shim-oci](03-runtime-shim-oci.md) |
| `service`/`grpc` | gRPC API 노출 | [README](README.md) |
| `cri` | Kubernetes CRI 서버 | [01-cri-plugin](01-cri-plugin.md) |
| `nri` | 노드 자원 인터페이스 | [05-sandbox-nri](05-sandbox-nri.md) |
| `event` | 이벤트 버스 | [05-sandbox-nri](05-sandbox-nri.md) |

## 설정으로 켜고 끄기

containerd 설정(`config.toml`)에서 플러그인별로 활성/비활성/파라미터를 정한다. 예: 기본 snapshotter를
`overlayfs`에서 `btrfs`로([08-snapshotters.md](08-snapshotters.md)), CRI 플러그인의 sandbox 이미지/CNI
경로 설정([01-cri-plugin.md](01-cri-plugin.md)). 비활성 플러그인은 로드되지 않아 의존하는 다른 플러그인도
영향받는다(그래서 의존성 그래프가 중요).

## 왜 이 설계인가

```
작은 코어 + 플러그인 그래프
  ├─ 필요한 것만 로드(경량)
  ├─ 의존성 자동 해결(위상 정렬)
  └─ 서드파티 확장(새 snapshotter/runtime을 플러그인으로)
```

이는 Kubernetes의 "인터페이스 + 플러그인"(CRI/CNI/CSI, scheduler framework) 철학을 containerd 내부에도
적용한 것이다 — 핵심은 작게, 확장은 플러그인으로.

## 더 읽을 곳
- [README.md](README.md) — containerd 아키텍처 개요
- [01-cri-plugin.md](01-cri-plugin.md) — 의존성이 많은 대표 플러그인
