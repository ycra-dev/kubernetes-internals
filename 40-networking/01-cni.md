# 40.01 · CNI — Container Network Interface

**근거**: `kubernetes/pkg/kubelet/network/`, `kubernetes/pkg/kubelet/kubelet_network.go`,
CNI 구현은 [42-cilium](../42-cilium/)

CNI는 "컨테이너에 네트워크를 붙이는 방법"의 표준 규약이다. CRI(런타임)·CSI(스토리지)와 같은 철학 —
Kubernetes는 인터페이스만 정하고, 실제 구현은 플러그인(cilium, Calico 등)에 맡긴다.

## 누가 CNI를 부르나 — sandbox 생성 시점

[13-kubelet/02](../13-kubelet/02-cri.md), [30-containerd/01](../30-containerd/01-cri-plugin.md)에서 봤듯,
Pod의 네트워크는 **sandbox(pod 네트워크 네임스페이스)가 만들어질 때** 설정된다. 흐름:

```
kubelet: SyncPod → RunPodSandbox (CRI)
   │
런타임(containerd): 네트워크 네임스페이스 생성
   │
CNI ADD 호출 ─► CNI 플러그인(cilium)이
   │             - 네임스페이스에 veth/인터페이스 생성
   │             - Pod CIDR에서 IP 할당
   │             - 라우팅/정책 설정
   ▼
Pod 가 IP 를 갖고 통신 가능
```

> 오늘날 CNI는 주로 **런타임(containerd)** 이 호출한다(CRI 플러그인의 sandbox 설정). kubelet에도
> 네트워크 관련 코드(`kubelet_network.go`, `pkg/kubelet/network/`)가 있으나, dockershim 제거 이후
> CNI 실행의 중심은 런타임으로 옮겨졌다.

## CNI 동작(ADD/DEL/CHECK)

CNI 플러그인은 표준 동작을 구현한다:

- **ADD**: 네임스페이스에 인터페이스를 만들고 IP를 부여(Pod 시작 시).
- **DEL**: 인터페이스/IP 회수(Pod 종료 시).
- **CHECK**: 설정 검증.

설정은 노드의 CNI 설정 디렉토리(보통 `/etc/cni/net.d/`)의 conflist로 주어지고, 플러그인 바이너리는
`/opt/cni/bin/`에 놓인다. cilium 등은 이 표준 위에서 자체 데이터패스를 구현한다([42-cilium/01](../42-cilium/)).

## 모델만 정하고 구현은 자유

CNI는 [README](README.md)의 "평평한 IP 모델"만 만족하면 **어떻게** 구현하든 자유다:

- **오버레이**(VXLAN/Geneve 등): Pod 트래픽을 캡슐화해 노드 간 전달.
- **네이티브 라우팅**: 노드 간 라우팅 테이블로 직접 전달(캡슐화 없음).
- **eBPF**(cilium): 커널 eBPF로 패킷 처리/정책 시행.

같은 모델을 만족하므로, 애플리케이션은 어떤 CNI를 쓰든 동일하게 동작한다.

## NetworkPolicy는 CNI의 몫

Kubernetes의 **NetworkPolicy**(어떤 Pod가 어떤 Pod와 통신 가능한가)는 API 객체로 선언되지만, **실제
시행은 CNI**가 한다. kube-proxy/kubelet은 정책을 시행하지 않는다. 그래서 NetworkPolicy를 쓰려면 그것을
지원하는 CNI(cilium 등)가 필요하다 → [42-cilium/03](../42-cilium/).

## 더 읽을 곳
- [42-cilium/01](../42-cilium/) — eBPF 기반 CNI 구현
- [30-containerd/01](../30-containerd/01-cri-plugin.md) — CNI를 호출하는 런타임
