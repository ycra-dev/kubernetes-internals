# 30.01 · CRI 플러그인 — kubelet 연결점

**근거**: `containerd/internal/cri/`, `containerd/plugins/cri/`

CRI 플러그인은 containerd가 Kubernetes와 만나는 지점이다. kubelet이 보내는 **CRI gRPC 요청**
([13-kubelet/02](../13-kubelet/02-cri.md))을 받아 containerd 내부 서비스(images/snapshots/runtime)로
번역한다.

## 두 서비스 구현

CRI 플러그인은 kubelet이 기대하는 두 CRI 서비스를 구현한다(`internal/cri/server/`):

- **RuntimeService**: sandbox/컨테이너 라이프사이클.
  - `sandbox_run.go` — `RunPodSandbox`: Pod의 네트워크 네임스페이스(sandbox)를 만들고 CNI로 네트워크를
    붙인 뒤 pause 컨테이너를 띄운다.
  - `container_create.go` — `CreateContainer`: 이미지의 snapshot으로 rootfs를 준비하고 OCI 스펙을
    구성([02](02-content-snapshots.md), [03](03-runtime-shim-oci.md)).
  - 그 외 Start/Stop/Remove/List/Status, Exec/Attach/PortForward(`streaming/`).
- **ImageManagerService**: `PullImage`, `ListImages`, `RemoveImage`(이미지 처리는 [02](02-content-snapshots.md)).

## sandbox와 CNI

`RunPodSandbox`(`sandbox_run.go`)는 Pod 단위 격리 단위를 만든다. 이때:

1. 네트워크 네임스페이스를 생성.
2. **CNI**를 호출해 그 네임스페이스에 인터페이스/IP를 부여([42-cilium](../42-cilium/)). cilium 같은
   CNI 플러그인이 여기서 동작한다.
3. pause 컨테이너로 그 네임스페이스를 붙들어, 이후 app 컨테이너들이 합류할 수 있게 한다.

즉 **Pod의 네트워크가 정해지는 시점**이 바로 이 sandbox 생성이다.

## 설정

CRI 플러그인 동작은 containerd 설정(`internal/cri/config/`)으로 제어된다 — 기본 런타임 핸들러(runc),
sandbox(pause) 이미지, CNI 설정 경로, 레지스트리 인증 등. RuntimeClass
([13-kubelet/02](../13-kubelet/02-cri.md))의 핸들러가 이 설정의 런타임에 매핑된다.

## NRI — Node Resource Interface

`internal/cri/nri/`(및 `plugins/nri`)는 컨테이너 생성/시작 시점에 외부 플러그인이 끼어들어 OCI 스펙을
조정하게 하는 확장점이다(예: 토폴로지 인지 자원 할당). kubelet 어드미션과는 별개의, 런타임 측 확장이다.

## 더 읽을 곳
- [02-content-snapshots.md](02-content-snapshots.md) — 이미지를 rootfs로 만드는 과정
- [03-runtime-shim-oci.md](03-runtime-shim-oci.md) — 실제 프로세스 기동
