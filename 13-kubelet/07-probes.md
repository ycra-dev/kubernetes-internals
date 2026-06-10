# 13.07 · Probe — liveness / readiness / startup

**근거**: `kubernetes/pkg/kubelet/prober/` (`prober_manager.go`, `worker.go`, `prober.go`)

kubelet은 컨테이너가 "살아 있나", "트래픽 받을 준비됐나"를 **probe**로 주기적으로 검사한다. probe 결과가
재시작·트래픽 차단·기동 판정을 좌우한다.

## 세 가지 probe

`prober_manager`(`prober/prober_manager.go`)가 컨테이너마다 probe worker를 띄운다(`worker.go`). 세 종류:

| probe | 실패 시 동작 | 용도 |
|-------|--------------|------|
| **liveness** | 컨테이너 **재시작** | 데드락/멈춤 감지 → 살려내기 |
| **readiness** | EndpointSlice에서 **제외**(트래픽 차단) | 일시적으로 못 받을 때 |
| **startup** | (기동 중) liveness/readiness 보류 | 느리게 뜨는 앱 보호 |

`prober_manager.go:165`에 "Readiness" 등 종류가 보이고, `:80`의 `StopLivenessAndStartup`은 종료 시
probe를 멈춘다.

## probe 방식

각 probe는 핸들러로 검사한다:
- **HTTP GET**: 지정 경로에 GET, 2xx/3xx면 성공.
- **TCP**: 포트 연결 성공이면 OK.
- **Exec**: 컨테이너 안에서 명령 실행, 종료코드 0이면 OK.
- **gRPC**: gRPC 헬스 체크.

`periodSeconds`(주기), `failureThreshold`(연속 실패 몇 번에 실패 판정), `timeoutSeconds`,
`initialDelaySeconds` 등으로 민감도를 조절한다.

## readiness가 무중단 배포의 핵심

readiness probe 결과는 Pod의 **Ready 컨디션**이 되고, 그게 EndpointSlice의 ready 여부를 결정한다
([40-networking/02](../40-networking/02-service-discovery.md), [14-kube-proxy/01](../14-kube-proxy/01-endpointslice-tracking.md)):

```
readiness 통과 → Pod Ready → EndpointSlice에 ready → kube-proxy/cilium이 트래픽 보냄
readiness 실패 → Pod NotReady → EndpointSlice에서 빠짐 → 트래픽 안 감 (재시작은 안 함)
```

그래서 새 Pod가 완전히 뜨기 전엔 트래픽을 안 받고, 일시 과부하 시 자동으로 트래픽에서 빠진다
([80-scenarios/03](../80-scenarios/03-rolling-update.md)).

## startup probe — 느린 앱 보호

레거시 앱은 기동에 몇 분이 걸릴 수 있다. liveness만 있으면 그사이 "죽었다"고 오판해 재시작 루프에 빠진다.
**startup probe**가 통과할 때까지 liveness/readiness를 보류해, 기동 시간을 넉넉히 준 뒤 정상 감시로
전환한다.

## liveness 남용 주의

liveness 실패는 재시작을 부르므로, 외부 의존성(DB 연결 등)을 liveness로 검사하면 DB 일시 장애에 멀쩡한
Pod들이 줄줄이 재시작되는 연쇄 장애가 난다. 외부 의존성은 readiness로, liveness는 "자기 자신이
데드락인가"만 검사하는 것이 권장이다.

## 더 읽을 곳
- [01-pod-lifecycle.md](01-pod-lifecycle.md) — probe가 SyncLoop과 맞물리는 곳
- [40-networking/02](../40-networking/02-service-discovery.md) — readiness → EndpointSlice
