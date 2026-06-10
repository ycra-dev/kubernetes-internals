# 13.11 · cgroup 계층 (v1/v2)

**근거**: `kubernetes/pkg/kubelet/cm/cgroup_manager_linux.go`(`IsCgroup2UnifiedMode` `:158`),
`cgroup_v1_manager_linux.go`, `cgroup_v2_manager_linux.go`, `qos_container_manager_linux.go`(`:50`)

[03-resource-mgmt.md](03-resource-mgmt.md)에서 "cm이 노드 자원을 계층적 cgroup으로 나눈다"고 했다. 이
문서는 그 **cgroup 계층 구조**와 **v1/v2 차이**를 코드 수준에서 본다. requests/limits가 실제로 커널에
시행되는 마지막 단계다.

## cgroup 계층 — 노드를 나무로 나누기

kubelet은 노드 자원을 **트리 구조**로 분할한다(`cm/`의 cgroup manager):

```
/ (루트, 노드 전체)
├── /kubepods                    ← 모든 Pod의 총 예산 (노드 - 시스템 예약)
│   ├── /kubepods/guaranteed     ← Guaranteed QoS Pod들
│   │   └── /kubepods/.../pod<UID>     ← Pod별 cgroup
│   │       └── <container>            ← 컨테이너별 cgroup
│   ├── /kubepods/burstable      ← Burstable QoS
│   └── /kubepods/besteffort     ← BestEffort QoS
├── /system.slice                ← 시스템 데몬(--system-reserved)
└── /kube.slice (등)             ← kubelet 등(--kube-reserved)
```

- **`/kubepods`** 가 워크로드 전체 예산이다 — 노드 capacity에서 시스템/kube 예약을 뺀 값
  ([03-resource-mgmt.md](03-resource-mgmt.md)).
- 그 아래 **QoS별 cgroup**(`qos_container_manager_linux.go:50` `QOSContainerManager`)이
  Guaranteed/Burstable/BestEffort를 나눈다.
- 다시 **Pod cgroup → 컨테이너 cgroup** 으로 내려간다.

## requests/limits → cgroup 값

각 cgroup에 컨테이너의 requests/limits가 변환돼 들어간다:

| Pod 스펙 | cgroup 제어 |
|----------|-------------|
| CPU `requests` | `cpu.shares`(v1) / `cpu.weight`(v2) — 경쟁 시 비례 배분(보장 최소) |
| CPU `limits` | `cpu.cfs_quota_us`(v1) / `cpu.max`(v2) — 절대 상한(스로틀) |
| memory `limits` | `memory.limit_in_bytes`(v1) / `memory.max`(v2) — 초과 시 OOM kill |
| memory `requests` | (v2) `memory.min`/`memory.low` — 보호 |

`requests`는 **보장(경쟁 시 최소 몫)**, `limits`는 **상한(넘으면 스로틀/OOM)**. 스케줄러는 requests로
배치를 약속하고([12-scheduler/02](../12-scheduler/02-plugins.md)), cgroup이 그 약속을 커널에서 강제한다.

## cgroup v1 vs v2

코드는 두 버전을 분기 처리한다(`cgroup_manager_linux.go:158` `IsCgroup2UnifiedMode`):

| | cgroup v1 | cgroup v2 |
|--|-----------|-----------|
| 구조 | 자원별(cpu, memory...) **별도 계층** | **단일 통합 계층**(unified) |
| 코드 | `cgroup_v1_manager_linux.go` | `cgroup_v2_manager_linux.go` |
| 메모리 보호 | 제한적 | `memory.min`/`low`로 정교 |
| 장점 | 호환성 | 일관성, 압력 정보(PSI), 더 정확한 제어 |

v2가 권장 방향이다 — 자원 관리가 일관되고, **PSI(Pressure Stall Information)** 로 자원 압박을 더 정확히
감지해 eviction([13-kubelet/04](../13-kubelet/04-eviction.md))에 활용할 수 있다.

## QoS와 cgroup의 연결

QoS 등급([03-resource-mgmt.md](03-resource-mgmt.md))이 cgroup 트리의 위치를 정하고, 그것이 자원 경쟁 시
동작을 결정한다:

- **Guaranteed**(requests==limits): 전용 보장. 메모리 압박 시 가장 나중에 영향.
- **Burstable**: requests 보장 + limits까지 버스트. 남는 자원을 빌려 쓸 수 있다.
- **BestEffort**: 보장 없음. 남는 것만, 압박 시 가장 먼저 스로틀/축출.

이 cgroup 위치가 노드 자원 압박 시 **OOM 우선순위**와 **eviction 순서**([13-kubelet/04](../13-kubelet/04-eviction.md))의
근거다.

## 컨테이너 런타임과의 분담

kubelet cm은 **Pod/QoS cgroup 계층**을 만들고, 개별 **컨테이너 cgroup**의 실제 생성은 런타임(runc)이
OCI 스펙([30-containerd/03](../30-containerd/03-runtime-shim-oci.md))의 cgroup 설정으로 한다. 즉 kubelet이
"틀"을 잡고, runc가 그 안에 컨테이너 cgroup을 채운다.

## 더 읽을 곳
- [03-resource-mgmt.md](03-resource-mgmt.md) — QoS와 cm 개요
- [04-eviction.md](04-eviction.md) — cgroup 압박 → 축출
- [30-containerd/03](../30-containerd/03-runtime-shim-oci.md) — runc의 컨테이너 cgroup 적용
