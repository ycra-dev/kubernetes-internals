# 60.06 · Manager 부가 기능 (healthz / metrics / leader / 인증서)

**근거**: `controller-runtime/pkg/` (`healthz/`, `metrics/`, `leaderelection/`, `certwatcher/`)

[01-manager-reconciler.md](01-manager-reconciler.md)에서 Manager가 "공유 자원을 들고 Runnable을 함께
시작/종료한다"고 했다. 이 문서는 Manager가 컨트롤러 외에 함께 제공하는 **운영 기능들** — 헬스체크, 메트릭,
리더 선출, 인증서 감시 — 을 본다. 오퍼레이터를 프로덕션에서 돌리는 데 필요한 것들이다.

## healthz / readyz — 헬스 엔드포인트

`pkg/healthz/`. Manager는 `/healthz`(살아있나)와 `/readyz`(준비됐나) HTTP 엔드포인트를 노출한다. 표준
컴포넌트와 같은 관례([00-foundations/07](../00-foundations/07-component-base.md)). Kubernetes가 오퍼레이터
Pod의 liveness/readiness probe([13-kubelet/07](../13-kubelet/07-probes.md))로 이를 찔러:

- 캐시 동기화 완료, 리더십 획득 등을 readiness 체크로 등록할 수 있다.
- 죽으면 재시작(liveness), 준비 안 되면 트래픽/리더십 보류.

## metrics — Prometheus 노출

`pkg/metrics/`. Manager는 `/metrics`로 Prometheus 메트릭을 노출한다([18-observability/01](../18-observability/01-metrics-monitoring.md)).
controller-runtime이 기본 제공하는 메트릭:

- reconcile **횟수/지연/에러율**(컨트롤러별).
- **workqueue 깊이/지연**([00-foundations/04](../00-foundations/04-list-watch-informer.md)) — 컨트롤러가
  밀리는지 감지.
- 캐시/클라이언트 메트릭.

이로써 오퍼레이터의 건강을 빌트인 컴포넌트와 동일한 방식으로 모니터링한다 — "reconcile 큐가 계속 쌓이면
컨트롤러가 못 따라잡는 것"([99-appendix/troubleshooting](../99-appendix/troubleshooting.md)).

## leaderelection — HA 오퍼레이터

`pkg/leaderelection/`. Manager 옵션으로 리더 선출을 켜면, 오퍼레이터 복제본 중 **하나만 컨트롤러를
돌린다**([00-foundations/06](../00-foundations/06-leader-election.md)). 빌트인 controller-manager
([11-controller-manager/README](../11-controller-manager/README.md))와 같은 메커니즘(Lease 리소스 기반).

- 리더가 되면 `Start`가 컨트롤러를 켜고, 리더십을 잃으면 graceful하게 종료한다
  ([01-manager-reconciler.md](01-manager-reconciler.md)의 Runnable 수명주기).
- 오퍼레이터를 2~3개 복제본으로 띄워도 같은 객체를 둘이 reconcile하는 충돌이 없다.

## certwatcher — 웹훅 인증서 갱신

`pkg/certwatcher/`. 오퍼레이터가 admission/conversion 웹훅 서버를 제공하면
([04-webhook-envtest.md](04-webhook-envtest.md)), 그 TLS 인증서가 갱신될 때(cert-manager 등이 Secret을
교체) **재시작 없이 새 인증서를 로드**해야 한다. certwatcher가 인증서 파일을 watch해 변경 시 핫 리로드한다 —
[17-security/03](../17-security/03-pki-certificates.md)의 rotation을 웹훅 서버 측에서 무중단으로 반영.

## 모두 Runnable로 통합

이 기능들은 전부 Manager의 **Runnable**([01-manager-reconciler.md](01-manager-reconciler.md))로 등록돼
`mgr.Start()` 한 번에 컨트롤러와 함께 뜨고, 컨텍스트 취소 시 함께 정리된다. 즉 오퍼레이터 개발자는
"reconcile 로직만" 짜면, 헬스/메트릭/HA/인증서 갱신은 프레임워크가 표준 방식으로 제공한다 — 이것이
controller-runtime이 "프로덕션 오퍼레이터의 보일러플레이트"를 없애주는 부분이다.

## 더 읽을 곳
- [01-manager-reconciler.md](01-manager-reconciler.md) — Manager/Runnable
- [00-foundations/06](../00-foundations/06-leader-election.md) — 리더 선출
- [18-observability/01](../18-observability/01-metrics-monitoring.md) — 메트릭
- [04-webhook-envtest.md](04-webhook-envtest.md) — 웹훅(certwatcher 대상)
