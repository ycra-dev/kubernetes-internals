# 10.02 · 인증 (Authentication)

**근거**: `staging/src/k8s.io/apiserver/pkg/authentication/`

인증은 "요청자가 **누구인가**"만 답한다(권한은 인가의 몫). 결과는 사용자 신원(이름/UID/그룹/extra)이다.
핸들러 체인에서 `WithAuthentication`(`server/config.go:1075`)이 호출한다.

## 인터페이스

핵심 인터페이스는 단순하다(`authentication/authenticator/interfaces.go:35`):

```go
AuthenticateRequest(req *http.Request) (*Response, bool, error)
```

- 반환 `(응답, ok, err)`: `ok=true`면 인증 성공(`Response.User`에 신원). `ok=false`면 "이 인증기는
  이 요청을 모름"(다음 인증기로). `err`면 오류.

## Union — 여러 인증기를 순차 시도

apiserver는 여러 인증 방식을 동시에 켜고, **union**(`authentication/request/union/`)으로 묶어
하나씩 시도하다 처음 성공하는 것을 채택한다. 켤 수 있는 인증기(`authentication/request/` 하위):

| 디렉토리 | 방식 |
|----------|------|
| `bearertoken` | `Authorization: Bearer <token>` 헤더에서 토큰 추출 |
| `x509` | 클라이언트 TLS 인증서의 subject(CN/O)를 사용자/그룹으로 |
| `headerrequest` | 신뢰된 프록시가 넣은 `X-Remote-User` 등 헤더 |
| `websocket` | 웹소켓 핸드셰이크 토큰 |
| `anonymous` | 위에서 아무도 인증 못 하면 `system:anonymous`로 (켜져 있을 때) |

`bearertoken`이 추출한 토큰은 다시 토큰 인증기 체인(`authentication/token/`)으로 검증된다:
ServiceAccount 토큰(JWT), OIDC, 정적 토큰, webhook 등.

## ServiceAccount 토큰

파드 안에서 도는 워크로드는 **ServiceAccount** 토큰으로 인증한다
(`authentication/serviceaccount/`). 토큰은 JWT이고, apiserver가 서명 키로 검증해
`system:serviceaccount:<ns>:<name>` 사용자와 관련 그룹을 부여한다. 토큰 발급/검증과 파드 주입은
TokenRequest API 및 kubelet과 연계된다.

## 출력 — 이후 단계가 쓰는 신원

성공 시 `user.Info`(이름/UID/그룹/extra)가 요청 컨텍스트에 실린다. 이것이:
- **인가**의 입력이 되고([03](03-authorization.md)),
- **어드미션**과 **감사 로그**에서도 쓰인다.

> 인증 실패는 핸들러 체인의 `failedHandler`(`server/config.go:1067`)로 빠져 401을 반환한다.

## 더 읽을 곳
- [03-authorization.md](03-authorization.md) — 이 신원으로 권한을 판단
- [01-request-pipeline.md](01-request-pipeline.md) — 체인에서의 위치
