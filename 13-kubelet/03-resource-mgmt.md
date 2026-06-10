# 13.03 · 자원 관리 (cgroup / cm)

**근거**: `kubernetes/pkg/kubelet/cm/`

kubelet은 노드의 CPU/메모리/장치를 Pod들에 나눠주고 격리한다. 리눅스의 **cgroup**으로 강제하며, 이
로직은 `pkg/kubelet/cm/`(Container Manager)에 있다.

## QoS 클래스 — 요청과 한계로 등급 매기기

Pod의 컨테이너는 `requests`(보장)와 `limits`(상한)를 선언한다. 이 둘의 관계로 **QoS 클래스**가 정해진다
(`cm/qos/`):

| QoS | 조건 | 자원 압박 시 |
|-----|------|--------------|
| **Guaranteed** | 모든 컨테이너 requests == limits | 가장 나중에 축출 |
| **Burstable** | requests < limits (일부라도 설정) | 중간 |
| **BestEffort** | requests/limits 미설정 | 가장 먼저 축출 |

이 등급이 cgroup 계층과 [04-eviction.md](04-eviction.md)의 축출 순서를 결정한다.

## cgroup 계층

cm은 노드 자원을 계층적 cgroup으로 나눈다: 노드 전체 → kube/system 예약 → Pod별 cgroup → 컨테이너.
`requests`/`limits`가 각 cgroup의 cpu.shares/quota, memory.limit로 변환돼 커널이 강제한다. 노드의
일부는 kubelet/시스템 데몬용으로 예약된다(`--kube-reserved`, `--system-reserved`).

## 전문 매니저들

`cm/` 아래에 자원 종류별 매니저가 있다:

| 매니저 | 역할 |
|--------|------|
| `cpumanager/` | Guaranteed Pod에 **CPU 코어 전용 할당**(pinning) — 캐시 지역성/지연 민감 워크로드 |
| `memorymanager/` | NUMA 인지 메모리 할당 |
| `devicemanager/` | GPU 등 **장치 플러그인**을 통한 확장 리소스 할당 |
| `dra/` | Dynamic Resource Allocation(차세대 장치 할당) |
| `admission/` | 노드 수준 입장 검사(자원이 안 맞으면 Pod 거부) |

이들은 **Topology Manager**로 조율돼, CPU/메모리/장치를 같은 NUMA 노드에 정렬할 수 있다.

## apiserver 인가와의 관계

스케줄러는 "노드에 자원이 충분한가"를 `noderesources` 필터로 본다
([12-scheduler/02](../12-scheduler/02-plugins.md)). 하지만 **실제 강제**는 여기 kubelet cm의 cgroup이
한다. 스케줄러의 계산은 requests 기반의 약속이고, cm은 그 약속을 커널 수준에서 시행한다.

## 더 읽을 곳
- [04-eviction.md](04-eviction.md) — 자원이 부족해질 때
- [12-scheduler/02](../12-scheduler/02-plugins.md) — 스케줄 단계의 자원 필터
