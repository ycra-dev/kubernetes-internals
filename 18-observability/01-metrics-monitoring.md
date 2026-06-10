# 18.01 · 메트릭과 모니터링

**근거**: `kubernetes/staging/src/k8s.io/component-base/metrics/`,
`kubernetes/staging/src/k8s.io/metrics/`

## 컴포넌트 메트릭 — /metrics

apiserver·scheduler·controller-manager·kubelet·kube-proxy 등은 모두 **Prometheus 형식** `/metrics`
HTTP 엔드포인트를 노출한다. 공용 라이브러리는 `component-base/metrics`다:

| 타입 (`metrics/`) | 의미 | 예 |
|-------------------|------|-----|
| `counter.go` | 단조 증가 카운터 | 처리한 요청 수, reconcile 횟수 |
| `gauge.go` | 오르내리는 현재값 | workqueue 길이, 현재 노드 수 |
| `histogram.go` | 분포(버킷) | 요청 지연, reconcile 소요시간 |

`registry.go`가 메트릭을 등록/노출한다. Kubernetes는 메트릭에 **안정성 등급**(ALPHA/STABLE)과
deprecation을 부여해, 모니터링 대시보드가 깨지지 않도록 변경을 관리한다.

### 무엇을 보나 (운영 핵심 신호)
- **apiserver**: 요청률/지연/에러, watch 수, **APF**(과부하) 상태([10-apiserver/01](../10-apiserver/01-request-pipeline.md)).
- **컨트롤러/스케줄러**: workqueue 깊이·지연(밀림 감지), reconcile 에러율.
- **kubelet**: Pod/컨테이너 시작 지연, PLEG 상태([13-kubelet/01](../13-kubelet/01-pod-lifecycle.md)).
- **etcd**: 리더 변경, DB 크기, fsync/commit 지연([20-etcd/08](../20-etcd/08-operations.md)).

## Prometheus가 수집한다

Prometheus(별도 설치되는 모니터링 시스템)가 각 컴포넌트의 `/metrics`를 주기적으로 **scrape**해 시계열로
저장하고, 질의/알람을 제공한다. 이것은 Kubernetes 코어가 아니라 생태계 표준 도구이며, 컴포넌트는 그저
표준 형식으로 노출만 한다.

## 리소스 사용량 — 별도 파이프라인

CPU/메모리 **실측 사용량**은 위 컴포넌트 메트릭과 다른 경로다:

```
컨테이너 사용량 (cAdvisor, kubelet 내장)
   │  kubelet Summary API 로 노출
   ▼
metrics-server (별도 컴포넌트) 가 노드들에서 수집·집계
   │  metrics.k8s.io API 로 제공 (API Aggregation, 10-apiserver/07)
   ▼
소비자: kubectl top,  HPA (11-controller-manager/01)
```

- API 타입(`NodeMetrics`, `PodMetrics`)은 `staging/src/k8s.io/metrics/`에 정의.
- metrics-server는 **단기 사용량**만 메모리에 들고 있다(장기 저장은 Prometheus의 몫).
- **HPA**가 "CPU 사용률이 목표 초과"를 판단하는 입력이 바로 이 API다
  ([80-scenarios/04](../80-scenarios/04-scale-and-failure.md)).

## 커스텀/외부 메트릭

HPA는 리소스 메트릭 외에 **custom metrics**(애플리케이션 지표)와 **external metrics**(큐 길이 등)로도
스케일할 수 있다. 이들은 각각 `custom.metrics.k8s.io`/`external.metrics.k8s.io` API를 Aggregation으로
제공하는 어댑터(예: Prometheus Adapter)가 구현한다.

## 추적(Tracing)

`component-base/tracing`은 OpenTelemetry 기반 분산 추적을 지원한다. apiserver 요청 처리
([10-apiserver/01](../10-apiserver/01-request-pipeline.md)의 `WithTracing`)나 etcd 연산에 스팬을 붙여,
한 요청이 컴포넌트들을 지나는 경로를 추적할 수 있다.

## 더 읽을 곳
- [02-events-logging.md](02-events-logging.md) — 사건/로그 축
- [70-autoscaling](../70-autoscaling/) — 메트릭 소비자
