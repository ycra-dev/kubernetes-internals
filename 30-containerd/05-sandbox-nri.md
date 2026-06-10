# 30.05 · Sandbox API와 NRI

**근거**: `containerd/core/sandbox/` (`controller.go`, `bridge.go`), `containerd/plugins/nri/`,
`containerd/core/events/`

containerd가 sandbox(Pod 격리 단위)를 다루는 방식과, 컨테이너 생성에 끼어드는 확장점 NRI, 그리고 내부
이벤트 시스템을 다룬다.

## Sandbox API — Pod sandbox의 추상화

[01-cri-plugin.md](01-cri-plugin.md)에서 sandbox는 "pause 컨테이너로 네트워크 네임스페이스를 붙드는 것"
이라 했다. containerd 2.x는 이를 **Sandbox API**(`core/sandbox/`)로 1급 추상화했다:

- **Sandbox Controller**(`sandbox/controller.go`): sandbox의 생성/시작/중지/상태를 관리하는 인터페이스.
- 기본 구현은 여전히 pause 컨테이너지만, **VM 기반 sandbox**(Kata Containers처럼 Pod를 경량 VM으로
  격리)를 같은 API로 끼울 수 있다.

이로써 "Pod = 프로세스 격리(runc)"와 "Pod = VM 격리(Kata)"를 동일한 sandbox 추상화로 다룬다 —
RuntimeClass([13-kubelet/02](../13-kubelet/02-cri.md))로 선택.

## NRI — Node Resource Interface

`containerd/plugins/nri/`(및 cri 통합). NRI는 컨테이너/sandbox 라이프사이클 지점에 **외부 플러그인이
끼어들어 OCI 스펙을 조정**하게 하는 확장점이다:

- 컨테이너 생성 직전에 NRI 플러그인이 호출돼, OCI 스펙(cgroup, 장치, 마운트, 환경)을 수정할 수 있다.
- 용도: 토폴로지 인지 자원 할당, 커스텀 장치 주입, 보안 정책 적용 등.

> NRI는 **런타임 수준** 확장이다. Kubernetes 어드미션([10-apiserver/04](../10-apiserver/04-admission.md))이
> "API 객체"를 조정한다면, NRI는 "실제 컨테이너 스펙"을 노드에서 조정한다. 둘은 다른 계층이다.

## 이벤트 시스템

`containerd/core/events/`. containerd 내부는 **이벤트 버스**로 동작한다 — 이미지 pull 완료, 컨테이너
시작/종료, sandbox 변경 등이 이벤트로 발행된다. CRI 플러그인은 이 이벤트를 구독해 상태를 추적하고,
kubelet의 **PLEG**([13-kubelet/01](../13-kubelet/01-pod-lifecycle.md))가 그 변화를 감지하는 토대가 된다
(Evented PLEG는 CRI 이벤트를 push 받음).

```
컨테이너 종료 → containerd 이벤트 → CRI 플러그인 → (CRI 이벤트) → kubelet PLEG → SyncPod
```

## 더 읽을 곳
- [01-cri-plugin.md](01-cri-plugin.md) — sandbox 생성(CRI)
- [03-runtime-shim-oci.md](03-runtime-shim-oci.md) — OCI 스펙(NRI가 조정)
- [13-kubelet/01](../13-kubelet/01-pod-lifecycle.md) — 이벤트를 받는 PLEG
