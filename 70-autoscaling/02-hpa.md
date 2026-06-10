# 70.02 · HPA 알고리즘

**근거**: `kubernetes/pkg/controller/podautoscaler/`
(`horizontal.go`, `replica_calculator.go` — `GetResourceReplicas` `:80`, `Tolerances` `:46`)

HorizontalPodAutoscaler(HPA)는 [README](README.md)에서 본 "Pod 개수 조정"의 빌트인 구현이다. 여기선
**실제 스케일 계산 수식**을 본다. HPA는 controller-manager 안의 컨트롤러다
([11-controller-manager/01](../11-controller-manager/01-controller-catalog.md)).

## 기본 수식 — 사용률 비례

핵심은 단순한 비례식이다(`replica_calculator.go`의 계산):

```
desiredReplicas = ceil( currentReplicas × (currentMetricValue / desiredMetricValue) )
```

예: 현재 3 replica, 평균 CPU 사용률이 90%인데 목표가 50%라면:
```
desired = ceil(3 × 90/50) = ceil(5.4) = 6
```

즉 "지금 목표보다 1.8배 쓰고 있으니 replica를 1.8배로". `GetResourceReplicas`(`replica_calculator.go:80`)가
리소스 메트릭에 대해 이 계산을 한다.

## tolerance — 미세 진동 방지

매번 정확히 맞추면 메트릭이 조금만 흔들려도 스케일이 출렁인다(thrashing). 그래서 **tolerance**
(`replica_calculator.go:46` `Tolerances`, `:112` `isWithin`)를 둔다:

- `currentMetric / desiredMetric`의 비율이 1.0 ± tolerance(기본 ~10%) 안이면 **아무것도 안 한다.**
- 즉 사용률 45~55%(목표 50% 기준)면 그대로 둔다 — 작은 변동을 무시.

## 메트릭 소스

HPA가 보는 메트릭은 세 종류([18-observability/01](../18-observability/01-metrics-monitoring.md)):

| 종류 | 출처 |
|------|------|
| **Resource**(CPU/메모리) | metrics-server (`metrics.k8s.io`) |
| **Custom**(앱 지표, 예: QPS) | `custom.metrics.k8s.io` 어댑터 |
| **External**(외부, 예: 큐 길이) | `external.metrics.k8s.io` 어댑터 |

여러 메트릭을 지정하면 각각으로 desired를 계산해 **가장 큰 값**을 택한다(가장 부하 높은 신호를 따름).

## 안정화와 속도 제한

급격한 스케일을 막는 장치들(`horizontal.go`, `rate_limiters.go`):

- **stabilization window**: 스케일 **다운**은 최근 일정 시간의 최댓값을 기준으로 해, 메트릭이 잠깐
  떨어졌다고 곧장 줄이지 않는다(다시 오를 수 있으므로). 스케일 업은 더 민감하게.
- **scaling policy**: 한 번에 늘리거나 줄일 수 있는 비율/개수 제한(`behavior` 설정).

이로써 "빠르게 늘리고, 천천히 줄이는" 비대칭으로 안정성을 확보한다.

## HPA → CA 연쇄

HPA가 replica를 늘려 노드 자원이 부족해지면, 새 Pod는 Pending이 되고 **Cluster Autoscaler**가 노드를
추가한다([README](README.md), [80-scenarios/04](../80-scenarios/04-scale-and-failure.md)). HPA(Pod 수)와
CA(노드 수)는 "Pending Pod" 신호로 협력한다.

## 더 읽을 곳
- [README.md](README.md) — HPA/VPA/CA 구분
- [18-observability/01](../18-observability/01-metrics-monitoring.md) — 메트릭 파이프라인
- [80-scenarios/04](../80-scenarios/04-scale-and-failure.md) — 스케일링 시나리오
