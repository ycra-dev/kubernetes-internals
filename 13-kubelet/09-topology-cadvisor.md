# 13.09 · Topology Manager와 cAdvisor

**근거**: `kubernetes/pkg/kubelet/cm/topologymanager/`, `kubernetes/pkg/kubelet/cadvisor/`

[03-resource-mgmt.md](03-resource-mgmt.md)에서 본 CPU/메모리/장치 매니저들을 **NUMA 정렬**로 조율하는
Topology Manager와, 컨테이너 사용량을 수집하는 cAdvisor를 다룬다.

## Topology Manager — NUMA 정렬

현대 서버는 **NUMA(Non-Uniform Memory Access)** 구조다 — CPU 코어와 메모리/장치가 NUMA 노드로 묶여 있고,
**같은 NUMA 노드 안의 자원**에 접근하는 것이 다른 노드보다 훨씬 빠르다. 지연 민감/고성능 워크로드(예:
GPU + 그 GPU에 가까운 CPU)는 자원이 **같은 NUMA 노드에 정렬**돼야 한다.

문제: CPU Manager, Memory Manager, Device Manager가 **따로따로** 할당하면 CPU는 NUMA 0, GPU는 NUMA 1처럼
어긋날 수 있다. **Topology Manager**(`cm/topologymanager/`)가 이들의 결정을 **조율**한다 — 각 매니저에게
"가능한 NUMA 조합(hint)"을 받아, 모두를 만족하는 정렬을 고른다(`bitmask/`로 NUMA 집합 연산).

### 정책 (`policy_*.go`)

| 정책 | 파일 | 동작 |
|------|------|------|
| `none` | `policy_none.go` | 토폴로지 무시(기본) |
| `best-effort` | `policy_best_effort.go` | 정렬 시도, 안 되면 그냥 배치 |
| `restricted` | `policy_restricted.go` | 정렬 못 하면 **Pod 입장 거부** |
| `single-numa-node` | `policy_single_numa_node.go` | 단일 NUMA 노드 정렬 강제, 못 하면 거부 |

`restricted`/`single-numa-node`는 노드 admission([03-resource-mgmt.md](03-resource-mgmt.md)의 cm admission)
에서 정렬 불가 Pod를 거부한다 — 그 Pod는 다른 노드로 가거나 Pending.

## cAdvisor — 컨테이너 자원 사용량 수집

**cAdvisor**(`cm` 외부 `pkg/kubelet/cadvisor/`)는 kubelet에 내장된 컨테이너 모니터링이다. 각 컨테이너의
cgroup에서 **실측 사용량**(CPU, 메모리, 파일시스템, 네트워크)을 수집한다.

이 데이터가 흘러가는 곳:
- kubelet **Summary API** → metrics-server → `kubectl top`, HPA
  ([18-observability/01](../18-observability/01-metrics-monitoring.md)).
- eviction manager가 자원 압박 판단에 사용([04-eviction.md](04-eviction.md)).

즉 "이 컨테이너가 실제로 얼마나 쓰고 있나"의 1차 출처가 cAdvisor다(스케줄러가 보는 requests는 *약속*,
cAdvisor는 *실측*).

## 더 읽을 곳
- [03-resource-mgmt.md](03-resource-mgmt.md) — Topology Manager가 조율하는 CPU/Memory/Device 매니저
- [18-observability/01](../18-observability/01-metrics-monitoring.md) — cAdvisor → metrics-server → HPA
- [19-dra.md](../19-dra.md) — 차세대 장치 할당(NUMA 인지 포함)
