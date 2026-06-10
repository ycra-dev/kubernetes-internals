# 18 · 관측성 (Observability)

**근거 레포**: `kubernetes` (`staging/.../component-base/metrics`, `staging/.../metrics`,
`api/core/v1`·`api/events/v1`의 Event)

클러스터가 무엇을 하고 있는지 어떻게 아는가? Kubernetes의 관측성은 세 축이다: **메트릭**(수치),
**이벤트**(사건), **로그**(상세). 이들은 오토스케일링([70-autoscaling](../70-autoscaling/))과 운영
모니터링의 토대다.

## 세 축

| 축 | 무엇 | 형식 |
|----|------|------|
| **메트릭** | 시간에 따른 수치(요청 수, 큐 길이, 자원 사용량) | Prometheus `/metrics` |
| **이벤트** | 객체에 일어난 사건(스케줄됨, 실패함, 축출됨) | Event API 객체 |
| **로그** | 컴포넌트/컨테이너의 상세 출력 | klog(컴포넌트), stdout(컨테이너) |

## 메트릭 — Prometheus 모델

모든 핵심 컴포넌트는 `/metrics` 엔드포인트로 Prometheus 형식 메트릭을 노출한다. 공통 라이브러리는
`component-base/metrics`([00-foundations/07](../00-foundations/07-component-base.md)):

- 메트릭 타입: `counter.go`, `gauge.go`, `histogram.go`(누적/현재값/분포), 레지스트리 `registry.go`.
- 안정성 등급과 deprecation을 메트릭마다 관리(컴포넌트 메트릭 정책).

Prometheus(별도 설치)가 이 엔드포인트들을 주기적으로 scrape해 저장/질의한다. 상세는
[01-metrics-monitoring.md](01-metrics-monitoring.md).

## 리소스 메트릭 — metrics-server

CPU/메모리 **사용량**(요청값이 아니라 실측)은 별도 경로다:

- 각 노드 kubelet이 cAdvisor로 컨테이너 사용량을 수집해 **Summary API**로 노출.
- **metrics-server**(별도 컴포넌트)가 이를 모아 `metrics.k8s.io` API로 제공 — 이것은 **API
  Aggregation**으로 붙는다([10-apiserver/07](../10-apiserver/07-crd-aggregation.md)). API 타입은
  `staging/src/k8s.io/metrics/`에 정의.
- `kubectl top`과 **HPA**([11-controller-manager/01](../11-controller-manager/01-controller-catalog.md))가
  이 API를 소비한다.

## 이벤트 — 사건의 기록

**Event**(`api/core/v1/types.go:7570`, 신형 `api/events/v1/types.go:34`)는 "무슨 일이 있었나"를 담는
객체다: 어떤 객체에, 무슨 이유로(Scheduled/Pulled/Failed/Evicted), 언제, 몇 번. `kubectl describe`나
`kubectl get events`로 본다. 상세는 [02-events-logging.md](02-events-logging.md).

## 문서

| # | 문서 | 내용 |
|---|------|------|
| 01 | [metrics-monitoring.md](01-metrics-monitoring.md) | /metrics, Prometheus, metrics-server |
| 02 | [events-logging.md](02-events-logging.md) | Event 객체, klog, 컨테이너 로그 |

## 더 읽을 곳
- [00-foundations/07](../00-foundations/07-component-base.md) — metrics/logs 공용 라이브러리
- [70-autoscaling](../70-autoscaling/) — 메트릭을 소비하는 오토스케일러
