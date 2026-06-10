# 10.15 · 제네릭 서버와 위임 체인

**근거**: `kubernetes/staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go`
(`delegationTarget` `:223`, `NextDelegate` `:328`, `PrepareRun`/`Run`)

[README](README.md)에서 apiserver가 "세 서버(aggregator/KubeAPIServer/apiextensions)가 한 프로세스에
합쳐진 구조"라 했다. 이 문서는 그것들이 **어떻게 한 체인으로 묶이고, 어떻게 시작/종료**하는지를 본다.
세 서버를 잇는 것이 **delegation chain**이다.

## GenericAPIServer — 공통 골격

세 서버는 모두 `GenericAPIServer`(`server/genericapiserver.go`)를 기반으로 만들어진다. 제네릭 서버가
제공하는 공통 기능: 핸들러 체인([01-request-pipeline.md](01-request-pipeline.md)), API 등록, 디스커버리
([11-discovery-openapi.md](11-discovery-openapi.md)), 헬스체크, PostStartHook, graceful shutdown. 각
서버는 여기에 자기 API를 얹는다.

## delegation chain — 못 받으면 다음에게

세 서버는 **위임 체인**으로 연결된다(`genericapiserver.go:223` `delegationTarget`, `:328` `NextDelegate`):

```
요청 → kube-aggregator
         │ 내가 처리할 수 있나? (등록된 APIService?)
         ├─ 예 → 외부 API 서버로 프록시 (07-crd-aggregation)
         └─ 아니오 → delegationTarget(다음)에게 위임
                      ↓
              KubeAPIServer
                │ 내장 그룹(pods/deployments...)인가?
                ├─ 예 → 처리
                └─ 아니오 → delegationTarget에게 위임
                             ↓
                     apiextensions-apiserver
                       │ CRD인가?
                       ├─ 예 → 처리 (07-crd-aggregation)
                       └─ 아니오 → 404 (체인 끝)
```

`delegationTarget`(`:223`, "never nil")은 "내가 못 처리하면 넘길 다음 서버"다. 체인 끝은 빈 위임(404
반환). 이 구조 덕분에 **하나의 HTTP 포트**로 내장 API + CRD + Aggregated API가 모두 서빙된다.

## 부팅 — PrepareRun → Run

서버 시작은 두 단계다:

- **PrepareRun**: 디스커버리/OpenAPI 집계, 헬스체크 설치, 라우팅 확정. 위임 체인 전체를 준비한다.
- **Run**: HTTPS 리스너를 열고 **PostStartHook**들을 실행. PostStartHook은 "서버가 뜬 뒤 해야 할 초기화"
  (예: 기본 RBAC 역할 생성, 시스템 네임스페이스 보장)다.

## graceful shutdown — 안전한 종료

apiserver 종료는 진행 중 요청을 끊지 않도록 단계적이다(`genericapiserver.go`의 lifecycle signals,
[01-request-pipeline.md](01-request-pipeline.md)의 `WithWaitGroup`/`WithRetryAfter`):

```
1. NotAcceptingNewRequest 신호 → 새 요청에 Retry-After (LB가 다른 인스턴스로)
2. 진행 중 요청(non-long-running) 완료 대기 (WaitGroup)
3. watch 등 long-running 연결 종료 유예 (ShutdownWatchTerminationGracePeriod)
4. 리스너 종료
```

이로써 롤링 업데이트/재시작 시 클라이언트가 끊김 없이 다른 apiserver로 넘어간다(HA). 이는 워크로드의
graceful 종료([13-kubelet/10](../13-kubelet/10-pod-spec.md))와 같은 정신을 컨트롤 플레인에 적용한 것.

## 조립 위치

세 서버를 만들어 체인으로 엮는 코드는 `kubernetes/pkg/controlplane/instance.go`다([README](README.md)).
순서: apiextensions(맨 뒤) → KubeAPIServer(가운데) → aggregator(맨 앞)로 위임 체인을 구성한다.

## 더 읽을 곳
- [README.md](README.md) — 세 서버 개요
- [07-crd-aggregation.md](07-crd-aggregation.md) — CRD/Aggregation 위임
- [01-request-pipeline.md](01-request-pipeline.md) — 각 서버의 핸들러 체인
