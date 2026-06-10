# 10.14 · 어드미션 웹훅 내부

**근거**: `kubernetes/staging/src/k8s.io/apiserver/pkg/admission/plugin/webhook/`
(`generic/webhook.go` matchConditions `:326`, failing closed `:337`, `matchconditions/`, `predicates/`)

[04-admission.md](04-admission.md)에서 동적 어드미션 웹훅을 개요로 봤다. 이 문서는 apiserver가 웹훅을
**어떻게 안전하게 호출**하는지 — 매칭, 실패 정책, 순서, 부작용, 재호출 — 를 코드 수준에서 본다. 웹훅은
모든 변경 요청 경로에 끼어들므로, 잘못 설정하면 클러스터 전체가 멈출 수 있어 이 안전장치들이 중요하다.

## 어떤 요청에 어떤 웹훅을 부르나 — 매칭

웹훅 설정(`MutatingWebhookConfiguration`/`ValidatingWebhookConfiguration`)은 매칭 규칙을 가진다. apiserver는
요청이 들어오면 호출할 웹훅을 거른다(`webhook/predicates/`):

- **rules**: apiGroup/version/resource/operation 매칭(예: pods의 CREATE만).
- **namespaceSelector / objectSelector**: 네임스페이스/객체 라벨로 범위 제한.
- **matchConditions**(`generic/webhook.go:326`): **CEL 식**으로 더 정밀한 조건. 예: "특정 사용자의
  요청은 제외". 매칭 조건이 안 맞으면 웹훅을 건너뛴다.

이 거름이 중요한 이유: 모든 요청에 웹훅을 부르면 부하·지연이 크고, kube-system 같은 핵심 네임스페이스에
웹훅이 끼면 부트스트랩이 막힌다. selector로 범위를 좁힌다.

## failurePolicy — 웹훅이 응답 못 하면

웹훅 서버가 다운/타임아웃이면? `failurePolicy`가 결정한다:

| 정책 | 동작 |
|------|------|
| **Fail** (기본) | 요청을 **거부**(fail closed) — 안전하지만, 웹훅 서버 장애 시 해당 리소스 변경 전체가 막힘 |
| **Ignore** | 웹훅을 무시하고 진행(fail open) — 가용성 우선, 정책이 누락될 수 있음 |

`generic/webhook.go:337`의 "failing closed" 로그가 matchConditions 평가 실패 시의 처리를 보여준다 —
조건 평가조차 실패하면 안전하게 닫는다(Fail 정책일 때).

> 운영 함정: `failurePolicy: Fail` 웹훅의 서버가 죽으면 그 리소스(예: pods)의 모든 생성이 막혀 클러스터가
> 마비될 수 있다. 그래서 웹훅 서버는 HA로 띄우고, 자기 자신을 매칭에서 제외하며(부트스트랩 데드락 방지),
> `timeoutSeconds`를 짧게 둔다.

## 순서와 재호출 (reinvocation)

[04-admission.md](04-admission.md)에서 "mutating 먼저, validating 나중"이라 했다. 웹훅 안에서도:

- **mutating 웹훅**은 설정된 순서로 호출되지만, 한 웹훅이 객체를 바꾸면 **앞선 웹훅이 그 변경을 못 봤을**
  수 있다.
- **reinvocationPolicy**: `IfNeeded`면, 다른 웹훅이 객체를 바꾼 경우 그 웹훅을 **한 번 더** 호출해
  수렴시킨다([04-admission.md](04-admission.md)의 reinvocation).
- validating 웹훅은 모든 mutating이 끝난 **최종 객체**를 본다.

## 부작용(side effects)

웹훅이 외부 상태를 바꾸면(side effect) 위험하다 — 특히 apiserver가 **dry-run**(실제 저장 없이 검증)일 때
웹훅이 진짜 부작용을 일으키면 안 된다. `sideEffects` 선언:

- **None**: 부작용 없음(권장). dry-run 안전.
- **NoneOnDryRun**: 평소엔 부작용 있지만 dry-run 땐 없음.

apiserver는 dry-run 요청 시 `None`/`NoneOnDryRun`이 아닌 웹훅을 호출하지 않는다.

## AdmissionReview — 요청/응답 형식

apiserver는 매칭된 웹훅에 `AdmissionReview`(요청 객체 + 메타)를 HTTPS POST하고, 응답으로:
- **allowed: true/false**(+거부 사유), 그리고 mutating이면 **patch**(JSONPatch)를 받는다.
- apiserver가 그 patch를 객체에 적용한다(`webhook/mutating/`).

## CEL 대안 — 웹훅 없는 정책

웹훅의 네트워크 왕복·운영 부담을 피하려면 **ValidatingAdmissionPolicy**(CEL)
([04-admission.md](04-admission.md))를 쓴다 — apiserver 내부에서 CEL 식으로 검증해 웹훅 서버가 필요 없다.
matchConditions가 CEL인 것처럼, 정책 자체도 CEL로 표현하는 방향이다.

## 더 읽을 곳
- [04-admission.md](04-admission.md) — 어드미션 개요
- [60-controller-runtime/04](../60-controller-runtime/04-webhook-envtest.md) — 웹훅 서버 작성
- [17-security/02](../17-security/02-pod-security.md) — 빌트인 정책 어드미션(PSA)
