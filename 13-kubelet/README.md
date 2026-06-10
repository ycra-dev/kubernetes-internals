# 13 · kubelet

**근거 레포**: `kubernetes` (`cmd/kubelet`, `pkg/kubelet/`)

kubelet은 **각 노드에서 도는 에이전트**다. 컨트롤 플레인이 "이 Pod를 이 노드에 둬라"고 결정하면
([12-scheduler](../12-scheduler/)의 Binding), 그것을 실제 컨테이너로 만드는 것이 kubelet의 일이다.
노드당 하나씩 돈다.

## 역할

- **자기 노드에 배정된 Pod**를 watch해 컨테이너를 생성/유지/삭제(컨테이너 런타임을 CRI로 호출).
- Pod의 **현재 상태(status)** 를 apiserver에 보고.
- 노드 자체를 등록하고 **하트비트**를 보낸다([11-controller-manager/05](../11-controller-manager/05-node-lifecycle.md)가 감시).
- cgroup으로 **자원을 관리**하고, 자원이 부족하면 Pod를 **축출(eviction)**.
- 볼륨을 attach/mount하고, probe(liveness/readiness)를 실행.

> kubelet은 etcd나 다른 노드를 직접 보지 않는다. **오직 apiserver**를 통해 자기 노드의 Pod만 list/watch
> 한다(`spec.nodeName == 자기노드`). Node authorizer가 이를 강제한다
> ([10-apiserver/03](../10-apiserver/03-authorization.md)).

## 심장: SyncLoop

kubelet의 본체는 `pkg/kubelet/kubelet.go`의 **SyncLoop**(`syncLoop`, `:2630`)다. 여러 채널에서
이벤트를 받아, 각 Pod를 "원하는 상태"로 수렴시킨다 — 컨트롤러 패턴의 노드 버전이다.

`syncLoopIteration`(`:2705`)의 주석(`:2673`~`:2704`)이 입력 채널을 명시한다:

| 채널 | 의미 |
|------|------|
| `configCh` | Pod 설정 변경(apiserver watch, static pod 파일 등) → 타입별 핸들러로 디스패치 |
| `plegCh` | **PLEG**(Pod Lifecycle Event Generator)가 감지한 컨테이너 상태 변화 → 런타임 캐시 갱신 후 sync |
| `syncCh` | 주기적 전체 동기화(놓친 것 복구) |
| `housekeepingCh` | 정리(사라진 Pod의 잔여 자원 회수) |
| health manager | probe 실패한 Pod 재동기 |

> 주석(`:2688`)이 강조하듯, `select`의 case는 **무작위 순서**로 평가된다 — 어떤 채널이 먼저 처리될지
> 가정하면 안 된다.

각 이벤트는 결국 해당 Pod에 대해 **SyncPod**를 부르고, SyncPod가 "이 Pod의 컨테이너들이 spec대로
떠 있는지" 확인해 차이를 메운다.

## Pod 한 개가 뜨는 경로 (개요)

```
apiserver watch ─► configCh ─► SyncLoop
                                  │  SyncPod(pod)
                                  ▼
                kuberuntime.SyncPod (kuberuntime_manager.go:1450)
                  ├─ (필요시) sandbox(파드 네트워크 네임스페이스) 생성 → CNI(cilium)
                  ├─ 이미지 pull
                  └─ 각 컨테이너 생성/시작  ──CRI(gRPC)──► containerd
                                  │
                Pod.status 갱신 ─► apiserver 에 보고
```

## 문서

| # | 문서 | 내용 |
|---|------|------|
| 01 | [pod-lifecycle.md](01-pod-lifecycle.md) | SyncLoop, PLEG, SyncPod 상태 수렴 |
| 02 | [cri.md](02-cri.md) | 런타임 경계(CRI gRPC) → containerd |
| 03 | [resource-mgmt.md](03-resource-mgmt.md) | cgroup, cm, QoS, cpu/memory/device manager |
| 04 | [eviction.md](04-eviction.md) | 자원 압박 시 축출, 이미지 GC |
| 05 | [volume-manager.md](05-volume-manager.md) | 볼륨 attach/mount, CSI 연계 |
| 06 | [node-registration.md](06-node-registration.md) | 노드 등록, 하트비트, 인증서 |
| 07 | [probes.md](07-probes.md) | liveness/readiness/startup probe |
| 08 | [static-pods-shutdown.md](08-static-pods-shutdown.md) | static pod, graceful node shutdown |
| 09 | [topology-cadvisor.md](09-topology-cadvisor.md) | Topology Manager(NUMA), cAdvisor |
| 10 | [pod-spec.md](10-pod-spec.md) | init/sidecar 컨테이너, 훅, 종료 흐름 |
| 11 | [cgroups.md](11-cgroups.md) | cgroup v1/v2 계층, requests/limits 시행 |
| 12 | [pod-admission.md](12-pod-admission.md) | 노드 수준 Pod admission(predicate) |
| 13 | [image-management.md](13-image-management.md) | pull 정책/자격증명/백오프/이미지 GC |
| 14 | [device-plugins.md](14-device-plugins.md) | GPU/장치 할당(ListAndWatch/Allocate) |
| 15 | [mounts-ephemeral.md](15-mounts-ephemeral.md) | subPath/마운트 전파, 임시(디버그) 컨테이너 |
| 16 | [runtimeclass-overhead.md](16-runtimeclass-overhead.md) | RuntimeClass(런타임 선택), Pod overhead |

## 진입점
- 바이너리: `kubernetes/cmd/kubelet/kubelet.go`
- 본체: `kubernetes/pkg/kubelet/kubelet.go`
- 런타임: `kubernetes/pkg/kubelet/kuberuntime/`

## 더 읽을 곳
- [30-containerd](../30-containerd/) — CRI 너머의 런타임
- [42-cilium](../42-cilium/) — sandbox 네트워크를 붙이는 CNI
- [80-scenarios/01-pod-create.md](../80-scenarios/) — 전 과정 통합
