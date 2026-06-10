# 10.03 · 인가 (Authorization)

**근거**: `staging/src/k8s.io/apiserver/pkg/authorization/`, `kubernetes/plugin/pkg/auth/authorizer/`

인가는 "이 신원이 **이 동작을 할 권한이 있나**"를 답한다. 입력은 인증이 만든 `user.Info`와 요청 속성
(verb·resource·namespace·name 등)이고, 출력은 허용/거부/기권 결정이다.

## 인터페이스와 결정 모델

`authorization/authorizer/interfaces.go:100`:

```go
type Authorizer interface {
    Authorize(ctx context.Context, a Attributes) (authorized Decision, reason string, err error)
}
```

`Decision`은 세 값을 가진다:

| Decision | 의미 |
|----------|------|
| `DecisionAllow` | 허용 (체인 종료) |
| `DecisionDeny` | 명시적 거부 (체인 종료) |
| `DecisionNoOpinion` | 기권 → 다음 인가기로 |

`Attributes`에는 user, verb(get/list/create/...), apiGroup, resource, subresource, namespace, name,
또는 비-리소스 경로(`/healthz` 등)가 담긴다.

## Union — 인가기 체인

여러 인가 모드를 union(`authorization/union/union.go`)으로 묶어 순서대로 묻는다. 누군가 `Allow` 또는
`Deny`를 내면 거기서 끝, 모두 `NoOpinion`이면 최종 거부. 대표 모드:

### RBAC
`kubernetes/plugin/pkg/auth/authorizer/rbac/rbac.go:78`의 `RBACAuthorizer.Authorize`.
Role/ClusterRole가 정의한 규칙(어떤 apiGroup의 어떤 resource에 어떤 verb 허용)과,
RoleBinding/ClusterRoleBinding이 그것을 사용자/그룹/ServiceAccount에 연결한 것을 평가한다.
RBAC는 **허용 목록(allow-list)** 모델이다 — 명시적으로 허용된 것만 가능, 명시적 deny는 없다(union의
다른 인가기가 deny할 수는 있음). 부트스트랩 기본 역할은 `rbac/bootstrappolicy/`에 있다.

### Node authorizer
`kubernetes/plugin/pkg/auth/authorizer/node/node_authorizer.go`. kubelet(`system:node:<name>` 신원)이
**자기 노드에 관련된 객체만** 접근하도록 제한하는 특수 인가기다. 노드↔파드↔시크릿/컨피그맵/볼륨의
관계 그래프(`graph.go`, `graph_populator.go`)를 만들어, 그 노드의 파드가 실제로 참조하는 리소스만
허용한다. 이는 한 노드가 탈취돼도 다른 노드의 비밀에 접근 못 하게 하는 핵심 격리다.

### 그 외
Webhook(외부 인가 서버에 위임), ABAC(정책 파일, 레거시), AlwaysAllow/AlwaysDeny(테스트/특수).

## 핸들러 체인에서의 위치

`WithAuthorization`(`server/config.go:1040`)은 인증·APF 뒤, 실제 REST 핸들러 직전에 실행된다
([01](01-request-pipeline.md)). 거부 시 403을 반환한다.

## 인가 ≠ 어드미션

인가는 "이 동작 자체가 허용되는가"(verb/resource 수준)를 본다. 객체의 **내용**(예: 특권 컨테이너 금지,
리소스 쿼터 초과)을 보는 것은 다음 단계인 **어드미션**이다 → [04](04-admission.md).

## 더 읽을 곳
- [02-authentication.md](02-authentication.md) — 인가의 입력(신원)을 만드는 단계
- [13-kubelet/06-node-registration.md](../13-kubelet/) — Node authorizer가 제한하는 kubelet 신원
