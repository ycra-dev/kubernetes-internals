# 12.06 · 점수 계산 (Scoring)

**근거**: `kubernetes/pkg/scheduler/framework/plugins/noderesources/`
(`least_allocated.go`, `most_allocated.go`, `balanced_allocation.go`, `requested_to_capacity_ratio.go`),
`kubernetes/pkg/scheduler/schedule_one.go`(`prioritizeNodes` `:937`)

Filter를 통과한 노드들 중 **"가장 좋은"** 노드를 어떻게 고르나. 각 Score 플러그인이 노드마다 점수를
매기고, 가중 합산해 1등을 뽑는다. 이 문서는 그 점수 계산을 본다.

## 점수 합산 구조

`prioritizeNodes`(`schedule_one.go:937`)의 흐름:

1. 각 Score 플러그인이 feasible 노드마다 **0~100 범위 점수**를 낸다.
2. **NormalizeScore**(선택): 플러그인이 자기 점수를 0~100으로 정규화.
3. 플러그인별 **가중치(weight)** 를 곱한다(프로파일 설정, [02-plugins.md](02-plugins.md)).
4. 노드별로 모든 플러그인 점수를 **합산** → 최고점 노드 선택(동점이면 무작위 분산해 핫스팟 방지).

즉 최종 점수 = Σ(플러그인 점수 × 가중치). 가중치를 바꿔 "자원 균형 우선"인지 "이미지 지역성 우선"인지
정책을 조정한다.

## NodeResources 점수 전략 — bin packing vs spreading

가장 중요한 점수는 자원 기반이고, `noderesources/`에 여러 **전략**이 있다:

| 전략 | 파일 | 동작 | 효과 |
|------|------|------|------|
| **LeastAllocated** | `least_allocated.go` | 덜 찬 노드에 높은 점수 | **분산**(spread) — 부하를 고르게 |
| **MostAllocated** | `most_allocated.go` | 더 찬 노드에 높은 점수 | **빽빽이**(bin packing) — 노드 수 절약(오토스케일 다운 유리) |
| **RequestedToCapacityRatio** | `requested_to_capacity_ratio.go` | 사용률 구간별 점수 곡선 지정 | 커스텀 bin packing (`:29` 주석) |
| **BalancedAllocation** | `balanced_allocation.go` | CPU·메모리 사용률이 **균형**잡힌 노드 선호 | 한 자원만 고갈되는 것 방지 |

- 기본은 보통 **LeastAllocated**(부하 분산). 클러스터를 빽빽이 채워 노드를 줄이고 싶으면(비용 절감 +
  Cluster Autoscaler 연계, [70-autoscaling](../70-autoscaling/)) **MostAllocated**로 바꾼다.
- `requested_to_capacity_ratio.go:31`의 `buildRequestedToCapacityRatioScorerFunction`은 사용률→점수
  곡선(shape)을 사용자가 정의하게 해, 임의의 bin packing 정책을 표현한다.

## 다른 점수 플러그인

자원 외 점수들([02-plugins.md](02-plugins.md))도 합산에 들어간다:

- `imagelocality`: 이미지가 캐시된 노드 가점(이미지 pull 비용 절감).
- `interpodaffinity`/`podtopologyspread` Score: 선호 배치 충족 가점([05-affinity-topology.md](05-affinity-topology.md)).
- `tainttoleration` Score: PreferNoSchedule taint 회피.
- `nodeaffinity` Score: preferred affinity 충족.

## 정규화의 필요성

플러그인마다 점수 척도가 다를 수 있어(어떤 건 0~10, 어떤 건 노드 수에 비례), **NormalizeScore**로 0~100에
맞춘 뒤 가중 합산한다. 안 그러면 척도가 큰 플러그인이 결과를 지배한다.

## 더 읽을 곳
- [01-framework.md](01-framework.md) — Score 확장점
- [05-affinity-topology.md](05-affinity-topology.md) — 배치 관계 점수
- [70-autoscaling](../70-autoscaling/) — bin packing과 노드 스케일다운의 연계
