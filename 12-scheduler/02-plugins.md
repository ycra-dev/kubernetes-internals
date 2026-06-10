# 12.02 · 빌트인 플러그인

**근거**: `kubernetes/pkg/scheduler/framework/plugins/`

스케줄링 정책은 전부 이 디렉토리의 플러그인으로 구현된다. 한 플러그인이 여러 확장점을 동시에 구현하는
경우가 많다(예: `noderesources`는 Filter와 Score 둘 다). 디렉토리 이름이 곧 책임이다.

## 필터(Filter) 위주 — "이 노드에 둘 수 있나"

| 플러그인 | 검사 |
|----------|------|
| `noderesources` | 노드에 남은 CPU/메모리/확장 리소스가 Pod 요청을 감당하나 (Fit) |
| `nodename` | `Pod.spec.nodeName`이 지정됐으면 그 노드만 |
| `nodeports` | Pod가 요구하는 hostPort가 노드에서 비어 있나 |
| `nodeaffinity` | nodeSelector / nodeAffinity 매칭 |
| `nodeunschedulable` | 노드가 `unschedulable`로 표시되지 않았나 |
| `tainttoleration` | 노드 taint를 Pod가 톨러레이트하나 |
| `interpodaffinity` | Pod 간 affinity/anti-affinity 충족 |
| `podtopologyspread` | 토폴로지(존/노드) 분산 제약 충족 |
| `nodevolumelimits`, `volumebinding`, `volumezone`, `volumerestrictions` | 볼륨 관련 제약(노드의 볼륨 한도, PVC 바인딩 가능성, 존 일치 등) |

## 점수(Score) 위주 — "얼마나 좋은가"

| 플러그인 | 선호 |
|----------|------|
| `noderesources` (Score 모드) | 자원 균형/여유(전략에 따라 LeastAllocated/MostAllocated 등) |
| `imagelocality` | Pod 이미지가 이미 캐시된 노드 선호(당겨받기 비용 절감) |
| `interpodaffinity` (Score) | affinity 선호 충족 노드 |
| `podtopologyspread` (Score) | 더 고르게 분산되는 노드 |
| `nodeaffinity` (Score) | preferred affinity 충족 노드 |
| `tainttoleration` (Score) | PreferNoSchedule taint 회피 |

## 큐/바인딩/특수

| 플러그인 | 역할 |
|----------|------|
| `queuesort` | activeQ 정렬 기준(기본: 우선순위 → 생성시각) |
| `defaultbinder` | Bind 확장점: Binding 객체 생성 |
| `defaultpreemption` | PostFilter: 자리 없을 때 선점 → [04](04-preemption-extender.md) |
| `schedulinggates` | `spec.schedulingGates`가 남아 있으면 스케줄 보류 |
| `dynamicresources` | DRA(동적 리소스 할당) 처리 |
| `tainttoleration` | (위 양쪽 모두) |

## 프로파일 — 플러그인 조합

어떤 플러그인을 어느 확장점에 어떤 가중치로 켤지는 **스케줄러 프로파일**(`KubeSchedulerConfiguration`)로
설정한다. 기본 프로파일은 위 플러그인들을 합리적 가중치로 켠다. 여러 프로파일을 정의해 Pod의
`spec.schedulerName`으로 골라 쓸 수도 있다.

## 더 읽을 곳
- [01-framework.md](01-framework.md) — 이 플러그인들이 호출되는 확장점
- [50-storage](../50-storage/) — `volumebinding`이 연계하는 PV/PVC
