# 10.01 · 요청 파이프라인 (핸들러 체인)

**근거**: `staging/src/k8s.io/apiserver/pkg/server/config.go`, `.../endpoints/handlers/`

모든 HTTP 요청은 두 단계를 지난다: (1) 공통 **핸들러 체인**(인증/인가 등 횡단 필터), 그다음
(2) 리소스별 **REST 핸들러**(실제 생성/조회/삭제). 어드미션은 (2) 안에서 일어난다.

## 핸들러 체인 — `DefaultBuildHandlerChain`

`config.go:1036`의 `DefaultBuildHandlerChain`이 필터들을 **역순으로 래핑**한다. 즉 코드에서 나중에
감싼 것이 바깥에 있어 **먼저 실행**된다. 코드(`config.go:1039`~`1114`)를 실행 순서로 풀면:

```
[바깥 = 먼저 실행]
  WithPanicRecovery            (:1113) 패닉 복구
  WithRequestInfo              (:1110) URL 파싱 → group/version/resource/verb 추출
  WithMuxAndDiscoveryComplete  (:1112)
  WithHTTPLogging / WithLatencyTrackers
  WithRetryAfter / WithHSTS / WithCacheControl
  WithTimeoutForNonLongRunningRequests (:1086) 타임아웃
  WithWarningRecorder          (:1082)
  WithCORS                     (:1078)
  WithAuthentication           (:1075)  ◄── 누구인가? (실패 시 failedHandler→401)
  WithTracing                  (:1072)  인증 후 추적 (인증 클라이언트가 샘플링 영향)
  WithAudit                    (:1064)  감사 로그 시작
  WithImpersonation            (:1059)  사용자 가장(--as) 처리
  WithPriorityAndFairness      (:1048)  APF: 공정 큐잉/과부하 보호 (또는 WithMaxInFlightLimit)
  WithAuthorization            (:1040)  ◄── 권한이 있나? (실패 시 403)
[안쪽 = 마지막]
  apiHandler                           실제 REST 핸들러 (아래)
```

핵심 순서 보장:
- **RequestInfo가 가장 먼저** 수준에서 실행돼야 한다 — 이후 모든 필터가 "이 요청이 어떤
  group/resource/verb인지" 알아야 하기 때문(인가도 이걸 본다).
- **인증 → 인가** 순서. 인증 실패는 `failedHandler`(`config.go:1067`, `Unauthorized`)로 빠진다.
- **Priority & Fairness(APF)** 가 인가 직전에 있어 과부하 시 공정하게 큐잉/거절한다.

> 이 체인은 `genericapifilters`/`genericfilters` 패키지의 작은 미들웨어들로 구성된다. 각 `WithXxx`는
> `http.Handler`를 받아 감싼 `http.Handler`를 돌려주는 표준 데코레이터다.

## REST 핸들러 — 리소스별 동작

체인을 통과하면 `RequestInfo`의 verb에 따라 리소스별 핸들러로 라우팅된다
(`endpoints/handlers/` — `create.go`, `get.go`, `delete.go`, `update.go`, `patch.go`, `watch` 등).

예를 들어 `POST .../pods`(생성)는 `endpoints/handlers/create.go`로 가고, 거기서:

1. **디코딩**: 요청 바디를 내부 객체로 역직렬화([00-foundations/03](../00-foundations/03-api-object-model.md)의 Scheme/코덱).
2. **어드미션(mutating)**: 변형 어드미션 플러그인/웹훅이 객체를 고친다 → [04](04-admission.md).
3. **검증(validation)** + **어드미션(validating)**: 검증 어드미션이 거부할 수 있다.
4. **registry로 전달**: 전략(strategy)이 기본값/이름생성/추가검증 후 storage에 저장 → [05](05-registry-storage.md).

watch 요청은 별도 경로로, storage 계층의 watch를 HTTP 스트리밍(chunked/`watch=true`)으로 중계한다
→ [06](06-etcd-storage-layer.md).

## 왜 데코레이터 체인인가

각 관심사(인증/감사/타임아웃/과부하)를 독립 미들웨어로 분리해 **순서를 명시적으로** 조립한다. 순서가
곧 정책이다(예: 감사는 인증 뒤, 타임아웃은 바깥). 이 명시성이 보안상 중요하다 — 인증 전에 비싼 작업이
실행되지 않도록 보장한다.

## 더 읽을 곳
- [02-authentication.md](02-authentication.md) · [03-authorization.md](03-authorization.md) · [04-admission.md](04-admission.md)
- [05-registry-storage.md](05-registry-storage.md) — 체인 다음의 저장 경로
