# 30 · containerd

**근거 레포**: `containerd` (`github.com/containerd/containerd/v2`, 버전 2.3.0)

containerd는 **컨테이너 런타임**이다. kubelet이 CRI로 "이 컨테이너를 띄워라"고 하면
([13-kubelet/02](../13-kubelet/02-cri.md)), 이미지를 받아 풀고, 파일시스템을 준비하고, OCI 런타임(runc)으로
실제 프로세스를 띄우는 일을 한다. Kubernetes 전용이 아닌 범용 런타임이지만, 가장 널리 쓰이는 CRI 구현이다.

## 역할

1. **이미지 관리**: 레지스트리에서 이미지 pull, 레이어 저장(content store), 압축 해제(unpack).
2. **컨테이너 실행**: 루트 파일시스템 준비(snapshot), OCI 번들 생성, **shim**을 통해 runc로 프로세스 기동.
3. **CRI 서버**: kubelet의 CRI 요청을 위 기능들로 변환.

## 아키텍처: client → daemon → plugins

containerd는 **플러그인 조합**으로 구성된 데몬이다(`plugins/`). 핵심 서비스들이 플러그인으로 등록되고
gRPC로 노출된다:

```
kubelet ──CRI(gRPC)──► containerd 데몬
ctr/client ──gRPC───►   │
                        ├─ CRI plugin (internal/cri)     ── CRI를 내부 서비스로 번역
                        ├─ images / content / snapshots  ── 이미지·레이어·파일시스템
                        ├─ runtime (core/runtime/v2)     ── shim 관리
                        └─ ...                            ── leases, diff, events, sandbox
                                     │
                              containerd-shim-runc-v2 (컨테이너마다)
                                     │
                                   runc (OCI)  ──► 컨테이너 프로세스
```

- **core/**: 각 서브시스템의 핵심 로직(`containers`, `content`, `images`, `snapshots`, `diff`, `mount`,
  `runtime`, `sandbox`, `leases`...).
- **plugins/**: 그 core 기능을 플러그인으로 등록/배선(`plugins/cri`, `plugins/snapshots` 등).
- **internal/cri/**: CRI 서버 구현(kubelet 연결점).
- **cmd/**: 바이너리 — `containerd`(데몬), `containerd-shim-runc-v2`(shim), `ctr`(디버그 CLI).

## 컨테이너 하나가 뜨는 경로

```
CRI: RunPodSandbox        ─► sandbox 생성(네트워크 NS) + pause
CRI: PullImage            ─► content store에 레이어 저장 → unpack → snapshot
CRI: CreateContainer      ─► snapshotter.Prepare로 rootfs 마운트 + OCI 스펙 생성
CRI: StartContainer       ─► shim 기동 → runc create/start → 프로세스 실행
```

## 문서

| # | 문서 | 내용 |
|---|------|------|
| 01 | [cri-plugin.md](01-cri-plugin.md) | kubelet 연결점(CRI 서버) |
| 02 | [content-snapshots.md](02-content-snapshots.md) | 이미지 content store, snapshot으로 rootfs |
| 03 | [runtime-shim-oci.md](03-runtime-shim-oci.md) | shim, runc(OCI) 호출 |
| 04 | [ctr-client.md](04-ctr-client.md) | ctr CLI와 client 라이브러리 |
| 05 | [sandbox-nri.md](05-sandbox-nri.md) | Sandbox API, NRI 확장, 이벤트 |
| 06 | [image-verify-transfer.md](06-image-verify-transfer.md) | 이미지 서명 검증, Transfer 서비스 |
| 07 | [metadata-gc.md](07-metadata-gc.md) | 메타데이터(bbolt), tricolor GC, lease |
| 08 | [snapshotters.md](08-snapshotters.md) | overlayfs/native/btrfs 등 백엔드 |
| 09 | [oci-image-format.md](09-oci-image-format.md) | OCI 이미지 포맷(index/manifest/config/layer) |
| 10 | [plugin-system.md](10-plugin-system.md) | 플러그인 등록/의존성 그래프/초기화 |
| 11 | [content-store.md](11-content-store.md) | content store(ingester/digest 검증/local) |

## 진입점
- 데몬: `containerd/cmd/containerd/`
- shim: `containerd/cmd/containerd-shim-runc-v2/`
- CRI 서버: `containerd/internal/cri/server/`

## 더 읽을 곳
- [13-kubelet/02](../13-kubelet/02-cri.md) — CRI 클라이언트 쪽(kubelet)
- [42-cilium](../42-cilium/) — sandbox에 네트워크를 붙이는 CNI
