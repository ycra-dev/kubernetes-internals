# 10 · kube-apiserver

**근거 레포**: `kubernetes` (`cmd/kube-apiserver`, `staging/src/k8s.io/apiserver`, `pkg/controlplane`)

kube-apiserver는 클러스터의 **유일한 관문**이다. 모든 읽기/쓰기가 여기를 통하고, **오직 이 컴포넌트만
etcd에 접근한다**([00-foundations/02](../00-foundations/02-architecture.md)). 그래서 인증·인가·검증·버전
변환·저장이 전부 여기 집중된다.

## 역할

1. **REST API 서버**: 모든 API 객체에 대해 `GET/POST/PUT/PATCH/DELETE/WATCH`를 제공.
2. **정책 시행 지점**: 인증(누구) → 인가(권한) → 어드미션(변형/검증)을 모든 변경 요청에 적용.
3. **저장 게이트웨이**: 검증된 객체를 etcd에 저장하고, watch 스트림을 팬아웃.
4. **확장 지점**: CRD(동적 타입)와 Aggregation(별도 API 서버 연결)을 호스팅.

## 코드의 세 갈래

apiserver는 사실 세 개의 서버가 한 프로세스에 합쳐진 구조다(`pkg/controlplane`):

```
                  ┌──────────────────────────────────────────┐
요청 ─► (handler chain: authn/authz/...) ─►  요청 경로에 따라 라우팅
                  │                                            │
   ┌──────────────┼───────────────────────┬───────────────────┤
   ▼              ▼                        ▼                   
kube-aggregator   KubeAPIServer            apiextensions-apiserver
(/apis/<agg>)     (코어/내장 그룹: pods,    (CRD: /apis/<custom>)
별도 서버 위임     deployments, ...)
```

- **KubeAPIServer** — 코어 그룹과 내장 API 그룹(apps, batch 등)을 제공. 본체.
- **apiextensions-apiserver** — CRD로 정의된 사용자 타입을 동적으로 제공.
- **kube-aggregator** — `APIService`로 등록된 외부 API 서버에 요청을 위임(프록시).

상세는 [07-crd-aggregation.md](07-crd-aggregation.md).

## 요청 한 건의 내부 경로

```
HTTP 요청
  │
  ▼ [handler chain]  (server/config.go: DefaultBuildHandlerChain)
  │   panic recovery → RequestInfo → authentication → audit
  │   → impersonation → priority&fairness → authorization
  │
  ▼ [REST handler]   (endpoints/handlers/create.go 등)
  │   디코딩 → admission(mutating) → validation → admission(validating)
  │
  ▼ [registry/strategy]  (registry/generic/registry/store.go)
  │   기본값/검증/이름 생성 → storage 호출
  │
  ▼ [storage 계층]   (apiserver/pkg/storage → storage/etcd3)
      직렬화 → etcd 트랜잭션 → watch 팬아웃
```

## 문서

| # | 문서 | 내용 |
|---|------|------|
| 01 | [request-pipeline.md](01-request-pipeline.md) | 핸들러 체인 순서, REST 핸들러 진입 |
| 02 | [authentication.md](02-authentication.md) | 인증기들(토큰/SA/x509/anonymous)과 union |
| 03 | [authorization.md](03-authorization.md) | RBAC, Node authorizer, Decision 모델 |
| 04 | [admission.md](04-admission.md) | mutating/validating 체인, webhook, CEL |
| 05 | [registry-storage.md](05-registry-storage.md) | REST 전략, generic registry Store |
| 06 | [etcd-storage-layer.md](06-etcd-storage-layer.md) | etcd3 store, watch cacher, resourceVersion |
| 07 | [crd-aggregation.md](07-crd-aggregation.md) | apiextensions(CRD)와 kube-aggregator |
| 08 | [server-side-apply.md](08-server-side-apply.md) | 필드 소유권(managedFields), 충돌 |
| 09 | [audit.md](09-audit.md) | 감사 로깅(레벨/정책/단계) |
| 10 | [priority-fairness.md](10-priority-fairness.md) | APF: 공정 큐잉/과부하 보호 |
| 11 | [discovery-openapi.md](11-discovery-openapi.md) | discovery/OpenAPI 엔드포인트 |
| 12 | [watch-cache-internals.md](12-watch-cache-internals.md) | cacher 내부(watchCache/dispatch/bookmark) |
| 13 | [strategy-status.md](13-strategy-status.md) | REST 전략, status/scale 서브리소스 |
| 14 | [webhook-internals.md](14-webhook-internals.md) | 어드미션 웹훅(failurePolicy/매칭/재호출/부작용) |
| 15 | [server-lifecycle.md](15-server-lifecycle.md) | 제네릭 서버, 위임 체인, graceful shutdown |
| 16 | [consistency-reads.md](16-consistency-reads.md) | RV 읽기 일관성(Exact/NotOlderThan/WatchList) |
| 17 | [egress-konnectivity.md](17-egress-konnectivity.md) | egress selector, konnectivity 역터널 |

## 진입점

- 바이너리: `kubernetes/cmd/kube-apiserver/apiserver.go`
- 서버 조립: `kubernetes/pkg/controlplane/instance.go`
- 제네릭 서버: `kubernetes/staging/src/k8s.io/apiserver/pkg/server/`

## 더 읽을 곳
- [20-etcd](../20-etcd/) — 저장 백엔드 자체
- [00-foundations/04](../00-foundations/04-list-watch-informer.md) — 클라이언트 쪽 watch 소비
