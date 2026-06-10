# 13.14 · Device Plugin

**근거**: `kubernetes/pkg/kubelet/cm/devicemanager/` (`manager.go` `healthyDevices` `:84`,
`podDevices` `:93`, `endpoint.go` `allocate` `:109`, `topology_hints.go`), `pkg/kubelet/pluginmanager/`

[03-resource-mgmt.md](03-resource-mgmt.md)에서 device manager를 한 줄로 언급했다. 이 문서는 그
**Device Plugin** 프레임워크 — GPU/FPGA 같은 특수 장치를 Pod에 할당하는 방식 — 를 본다. DRA
([19-dra.md](../19-dra.md))의 전신이자 현재도 널리 쓰이는 개수 기반 장치 할당이다.

## 문제 — 커널이 모르는 자원

CPU/메모리는 kubelet이 cgroup으로 직접 관리한다([11-cgroups.md](11-cgroups.md)). 하지만 GPU 같은 장치는
벤더 드라이버가 필요하고 kubelet이 모른다. **Device Plugin**은 벤더가 "이 노드에 이런 장치가 N개 있다"를
kubelet에 알리고 할당을 처리하는 플러그인 인터페이스다.

## 등록 — plugin watcher

벤더 device plugin(DaemonSet)은 노드의 kubelet 플러그인 소켓에 자기를 **등록**한다. kubelet의
**pluginmanager**(`pkg/kubelet/pluginmanager/`)가 그 소켓을 watch해 등록을 감지한다 — CSI 드라이버 등록
([50-storage/02](../50-storage/02-csi.md))과 같은 메커니즘.

등록 시 plugin은 **resource name**(예: `nvidia.com/gpu`)을 선언한다. 이후 그 자원은 Pod의
`resources.limits`에 정수로 요청된다(`nvidia.com/gpu: 2`).

## ListAndWatch — 장치 목록과 건강

device plugin은 `ListAndWatch` gRPC 스트림으로 **자기 장치 목록과 건강 상태**를 계속 보고한다. device
manager가 이를 받아 추적한다(`manager.go`):

- `healthyDevices`(`manager.go:84`): 자원별 사용 가능한 건강한 장치 ID 집합.
- `unhealthyDevices`(`:87`): 고장난 장치(할당 대상에서 제외).

장치가 고장나면 plugin이 ListAndWatch로 알리고, device manager가 그만큼 가용량을 줄인다 — 노드의
allocatable 자원에 반영돼 스케줄러가 본다([12-scheduler/02](../12-scheduler/02-plugins.md)).

## Allocate — Pod에 장치 주입

Pod가 그 노드에 배정되고 장치를 요구하면, device manager가 plugin의 **`Allocate`**(`endpoint.go:109`)를
호출한다. plugin은 응답으로 컨테이너에 주입할 것을 돌려준다:

- **장치 노드**(`/dev/nvidia0` 등): 컨테이너에 마운트할 디바이스.
- **환경변수/마운트**: 드라이버 라이브러리 경로 등.

device manager는 이를 OCI 스펙([30-containerd/03](../30-containerd/03-runtime-shim-oci.md))에 반영해
컨테이너가 장치에 접근하게 한다. 어떤 Pod가 어떤 장치를 받았는지는 `podDevices`(`manager.go:93`)로 추적해
중복 할당을 막는다.

```
Pod{limits: nvidia.com/gpu: 2}가 노드에 배정
   │  kubelet device manager
   ▼
plugin.Allocate(["GPU-uuid-1","GPU-uuid-2"]) → /dev/nvidia0,1 + 드라이버 마운트
   → OCI 스펙에 주입 → 컨테이너가 GPU 2개 사용
```

## Topology 정렬

`topology_hints.go`. device manager는 **Topology Manager**([09-topology-cadvisor.md](09-topology-cadvisor.md))에
"이 장치를 어느 NUMA 노드에서 줄 수 있나"를 hint로 제공한다. 그래서 GPU와 그것에 가까운 CPU/메모리를
같은 NUMA에 정렬할 수 있다 — 고성능 워크로드의 지연 최소화.

## Device Plugin vs DRA

| | Device Plugin | DRA ([19-dra](../19-dra.md)) |
|--|---------------|------------------------------|
| 모델 | **정수 개수**(`gpu: 2`) | **속성 기반 구조적 요구** |
| 표현력 | 불투명한 카운트 | 공유/분할/토폴로지 제약 |
| 할당 결정 | kubelet device manager | 스케줄러 + 드라이버 |

DRA가 더 유연하지만, 단순 개수 할당은 device plugin이 여전히 표준이다. 둘 다 kubelet cm이 조율한다
([03-resource-mgmt.md](03-resource-mgmt.md)).

## 더 읽을 곳
- [03-resource-mgmt.md](03-resource-mgmt.md) — cm과 자원 매니저
- [09-topology-cadvisor.md](09-topology-cadvisor.md) — NUMA 정렬
- [19-dra.md](../19-dra.md) — 차세대 장치 할당
