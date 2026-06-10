# 12.05 · Affinity와 Topology Spread

**근거**: `kubernetes/pkg/scheduler/framework/plugins/interpodaffinity/`,
`kubernetes/pkg/scheduler/framework/plugins/podtopologyspread/`

스케줄러는 "자원이 맞나"([02-plugins.md](02-plugins.md))를 넘어, **Pod들 사이의 배치 관계**도 제어한다.
"이 Pod들은 같이 둬라"(affinity), "떨어뜨려라"(anti-affinity), "고르게 펴라"(topology spread).

## Node Affinity — 노드 선택

가장 단순한 형태. nodeSelector/nodeAffinity로 "이런 라벨의 노드에만"을 지정한다(`nodeaffinity` 플러그인).
필수(`requiredDuringScheduling`, Filter에서 거름)와 선호(`preferred`, Score에서 가점) 두 강도가 있다.

## Pod Affinity / Anti-Affinity — Pod 간 관계

`interpodaffinity/` 플러그인. "어떤 라벨의 Pod가 **있는/없는** 토폴로지 도메인에 둬라":

- **podAffinity**: "캐시 Pod와 같은 노드/존에" — 지연 줄이기.
- **podAntiAffinity**: "같은 앱의 replica를 서로 다른 노드에" — 가용성(한 노드 죽어도 살아남게).

**topologyKey**가 핵심이다 — 관계를 적용할 도메인 경계(`kubernetes.io/hostname`=노드,
`topology.kubernetes.io/zone`=존). "이 topologyKey가 같은 도메인 안에서" 매칭 Pod의 유무를 본다.

> 비용 주의: podAffinity는 모든 후보 노드에 대해 "그 도메인에 매칭 Pod가 있나"를 계산해야 해 대규모
> 클러스터에서 비싸다. 그래서 가능하면 topology spread를 권장한다.

## Pod Topology Spread — 고르게 펴기

`podtopologyspread/` 플러그인. anti-affinity보다 표현력 있고 효율적인 분산 제어다. "replica를 존/노드에
**고르게**" 펼친다. 핵심 파라미터(`filtering.go`):

| 파라미터 | 의미 |
|----------|------|
| `topologyKey` | 분산 단위(존/노드) |
| `maxSkew` | 도메인 간 Pod 수 차이 허용치 (`filtering.go:338`) |
| `whenUnsatisfiable` | 못 맞추면 `DoNotSchedule`(필수)/`ScheduleAnyway`(선호) |

`filtering.go:338`의 계산이 본질이다: `'그 도메인의 매칭 수' + 'self(0/1)' - '전역 최소' <= maxSkew`.
즉 어느 도메인이든 가장 적은 도메인보다 `maxSkew`를 초과해 많아지지 않게 한다. 그 노드에 둬도 skew가
유지되면 통과, 아니면 거른다(또는 감점).

```
maxSkew=1, zone 기준:
  zone-a: 2개, zone-b: 1개, zone-c: 1개  → 다음 Pod는 b나 c로 (a로 가면 skew=2 위반)
```

## Filter와 Score 양쪽에서

이 플러그인들은 [01-framework.md](01-framework.md)의 **Filter**(필수 제약 위반 노드 제거)와 **Score**
(선호 충족 노드 가점) 양쪽을 구현한다. 그래서 "반드시"와 "가급적"을 모두 표현할 수 있다.

## 더 읽을 곳
- [06-scoring.md](06-scoring.md) — 점수가 합쳐지는 방식
- [02-plugins.md](02-plugins.md) — 전체 플러그인 목록
