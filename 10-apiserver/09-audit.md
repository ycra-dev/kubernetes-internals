# 10.09 · 감사 로깅 (Audit)

**근거**: `kubernetes/staging/src/k8s.io/apiserver/pkg/audit/` (`types.go`, `policy/`, `evaluator.go`)

감사(audit)는 "**누가, 언제, 무엇을, 어떻게** 요청했고 결과가 무엇이었나"를 기록한다. 보안 사고 조사와
규정 준수의 토대다. apiserver가 유일한 관문이므로([00-foundations/02](../00-foundations/02-architecture.md)),
모든 행위를 한 곳에서 감사할 수 있다.

## 감사 레벨 — 얼마나 자세히

`apis/audit/types.go:42`의 `Level`이 기록 상세도를 정한다:

| Level | 기록 내용 |
|-------|-----------|
| `None` (`:47`) | 감사 안 함 |
| `Metadata` (`:49`) | 메타데이터만(누가/언제/무엇/verb), 바디 제외 |
| `Request` (`:52`) | Metadata + 요청 바디 |
| `RequestResponse` (`:55`) | Request + 응답 바디 |

`RequestResponse`는 가장 자세하지만 양이 많다(특히 list/watch). 그래서 리소스별로 레벨을 다르게 준다.

## 감사 정책 — 규칙으로 레벨 결정

`audit/policy/`(평가 `evaluator.go`)의 **감사 정책**이 "어떤 요청에 어떤 레벨을 적용할지"를 규칙 목록으로
정한다. 예: Secret은 메타데이터만(바디에 비밀이 있으므로), 일반 리소스 변경은 RequestResponse, 읽기는
None. 첫 번째로 매칭되는 규칙의 레벨이 적용된다.

## 감사 단계(Stage)

한 요청은 처리되며 여러 단계에서 이벤트를 낼 수 있다:

- **RequestReceived**: 요청 도착 시점.
- **ResponseStarted**: (긴 요청/watch) 응답 시작.
- **ResponseComplete**: 처리 완료.
- **Panic**: 처리 중 패닉.

긴 watch 같은 요청은 시작과 끝에 각각 이벤트를 남긴다.

## 핸들러 체인에서의 위치

감사는 핸들러 체인의 `WithAudit`([01-request-pipeline.md](01-request-pipeline.md)의 `config.go:1064`)에서
**인증 직후** 시작된다 — 누구의 요청인지 안 뒤 기록해야 의미가 있기 때문. 인증 실패도 별도로 기록된다
(`WithFailedAuthenticationAudit`).

## 백엔드 — 어디에 쓰나

감사 이벤트는 백엔드로 보내진다(`audit/`의 union으로 여러 곳 동시 가능):

- **로그 백엔드**: 파일/stdout에 JSON으로.
- **웹훅 백엔드**: 외부 감사 수집 시스템에 HTTP로 전송.

각 이벤트는 사용자(impersonation 포함), verb, 리소스, 소스 IP, 응답 코드, (정책에 따라) 요청/응답 바디를
담는다.

## 더 읽을 곳
- [01-request-pipeline.md](01-request-pipeline.md) — 감사가 시작되는 지점
- [17-security/README](../17-security/README.md) — 감사가 뒷받침하는 보안 모델
