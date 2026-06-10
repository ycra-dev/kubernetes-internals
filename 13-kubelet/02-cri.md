# 13.02 · CRI — 런타임 경계

**근거**: `kubernetes/staging/src/k8s.io/cri-api/pkg/apis/services.go`,
`kubernetes/pkg/kubelet/kuberuntime/`

kubelet은 컨테이너를 **직접** 만들지 않는다. **CRI(Container Runtime Interface)** 라는 gRPC 인터페이스
너머의 런타임(containerd 등)에 위임한다. 이 경계 덕분에 kubelet은 어떤 런타임을 쓰든 동일한 코드로
동작한다.

## 두 서비스

CRI는 두 gRPC 서비스로 나뉜다(`cri-api/pkg/apis/services.go`):

### RuntimeService (`services.go:114`)
컨테이너/sandbox 라이프사이클을 다룬다. 핵심 메서드:

| 메서드 | 역할 |
|--------|------|
| `RunPodSandbox` (`:72`) | Pod 단위 sandbox(네트워크 네임스페이스 + pause 컨테이너) 생성/시작 |
| `CreateContainer` (`:36`) | sandbox 안에 컨테이너 생성 |
| `StartContainer` (`:38`) | 컨테이너 시작 |
| `StopContainer` / `RemoveContainer` | 중지/삭제 |
| `ListContainers` / `ContainerStatus` | 상태 조회 (PLEG가 사용 → [01](01-pod-lifecycle.md)) |
| `Exec` / `Attach` / `PortForward` | `kubectl exec` 등 스트리밍 |

### ImageManagerService (`services.go:133`)
이미지를 다룬다: `PullImage`(`:139`), `ListImages`, `ImageStatus`, `RemoveImage`.

## sandbox 모델 — 왜 pause 컨테이너인가

Pod는 여러 컨테이너가 **네트워크/IPC 네임스페이스를 공유**한다. CRI는 먼저 `RunPodSandbox`로 그 공유
네임스페이스를 담는 **sandbox**를 만든다(보통 아무 일도 안 하는 작은 "pause" 컨테이너가 그 네임스페이스를
붙들고 있다). 이후 app 컨테이너들은 이 sandbox의 네임스페이스에 합류한다. **CNI 네트워크 설정은 이
sandbox 생성 시점에** 일어난다([42-cilium](../42-cilium/)).

## kubelet 쪽 구현

`pkg/kubelet/kuberuntime/`의 `kubeGenericRuntimeManager`가 CRI 클라이언트를 들고 SyncPod
([01](01-pod-lifecycle.md))를 구현한다. 내부적으로 `RuntimeService`/`ImageManagerService`
(`internalapi`)를 호출하며, 계측 래퍼(`instrumented_services.go`)로 감싸 메트릭을 수집한다.
gRPC 원격 클라이언트는 `pkg/kubelet/cri/`에 있고, 유닉스 소켓(예: `/run/containerd/containerd.sock`)으로
런타임에 연결한다.

## SyncPod가 CRI를 쓰는 순서 (재확인)

```
SyncPod
  ├─ RunPodSandbox        (sandbox + CNI 네트워크)
  ├─ PullImage            (이미지가 없으면)
  ├─ CreateContainer + StartContainer  (init → app, 순서대로)
  └─ (죽은 컨테이너) StopContainer/RemoveContainer 후 재생성
```

## RuntimeClass

노드에 런타임이 여러 개일 수 있다(예: runc, gVisor). **RuntimeClass** 객체가 어떤 핸들러를 쓸지
지정하고, `RunPodSandbox`의 `runtimeHandler` 인자(`services.go:72`)로 전달된다. 어드미션 플러그인
`runtimeclass`가 이를 검증한다([10-apiserver/04](../10-apiserver/04-admission.md)).

## 더 읽을 곳
- [30-containerd](../30-containerd/) — CRI 서버 쪽 구현
- [30-containerd/01-cri-plugin.md](../30-containerd/) — containerd의 CRI 플러그인
