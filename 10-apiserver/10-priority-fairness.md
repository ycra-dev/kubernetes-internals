# 10.10 · API Priority and Fairness (APF)

**근거**: `kubernetes/staging/src/k8s.io/apiserver/pkg/util/flowcontrol/`
(`apf_controller.go`, `apf_filter.go`, `fairqueuing/`)

apiserver는 클러스터의 단일 관문이라([00-foundations/02](../00-foundations/02-architecture.md)), 한 명의
폭주하는 클라이언트(예: 버그난 컨트롤러가 초당 수천 요청)가 전체를 마비시킬 수 있다. **APF**는 요청을
**공정하게 큐잉·분리**해 이를 막는다. 핸들러 체인의 `WithPriorityAndFairness`
([01-request-pipeline.md](01-request-pipeline.md)의 `config.go:1048`)가 이것이다.

## 단순 동시성 제한의 한계

APF 이전엔 `--max-requests-inflight` 같은 전역 동시성 한도만 있었다. 문제: 한 클라이언트가 그 한도를 다
먹으면 다른 모두가 굶는다. APF는 "전역 한 줄"을 **여러 우선순위·격리된 큐**로 나눈다.

## 두 설정 객체

APF는 두 API 객체로 구성된다(`apf_controller.go:67` 주석):

| 객체 | 역할 |
|------|------|
| **FlowSchema** | "어떤 요청을 어느 우선순위로 분류하나" — 요청자(사용자/SA/그룹)와 리소스를 매칭 |
| **PriorityLevelConfiguration** | 우선순위 레벨별로 동시성 몫(concurrency shares)과 큐잉 방식 정의 |

흐름: 요청 도착 → FlowSchema가 매칭해 **flow distinguisher**(예: 네임스페이스, 사용자)로 식별 →
해당 PriorityLevel의 큐로.

## 공정 큐잉 — fair queuing

`fairqueuing/`이 핵심 알고리즘이다. 한 우선순위 레벨 안에서도 여러 클라이언트가 경쟁하므로:

- **shuffle sharding**: 각 flow를 여러 큐에 해싱 분산해, 한 폭주 flow가 모든 큐를 막지 못하게 한다.
- 큐들에서 **공정하게(round-robin 류)** 요청을 꺼내 처리 — 폭주 클라이언트는 자기 큐에서 적체되고,
  조용한 클라이언트는 영향을 덜 받는다.

동시성 몫은 레벨 간에 배분되고(`conc_alloc.go`), 부족하면 큐잉하거나(대기) 거절한다.

## 격리 효과

- **시스템 요청 보호**: leader election, 노드 하트비트 같은 핵심 트래픽을 별도 고우선순위 레벨에 둬,
  사용자 워크로드 폭주에도 컨트롤 플레인이 살아남게 한다.
- **공정성**: 한 테넌트/컨트롤러가 apiserver를 독점하지 못한다.

## 거부와 재시도

큐가 차면 APF는 `429 Too Many Requests`를 반환한다. 클라이언트(client-go)는 이를 받아 백오프 후
재시도한다([00-foundations/04](../00-foundations/04-list-watch-informer.md)의 rate limit과 맞물림).
거절된 요청 추적은 `dropped_requests_tracker.go`.

## 더 읽을 곳
- [01-request-pipeline.md](01-request-pipeline.md) — 체인에서 APF 위치
- [18-observability/01](../18-observability/01-metrics-monitoring.md) — APF 상태 메트릭(과부하 감지)
