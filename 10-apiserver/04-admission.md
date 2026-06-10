# 10.04 · 어드미션 (Admission Control)

**근거**: `staging/src/k8s.io/apiserver/pkg/admission/`, `kubernetes/plugin/pkg/admission/`

어드미션은 인증·인가를 통과한 **변경 요청(create/update/delete)** 에 대해, 객체의 **내용**을 보고
변형(mutate)하거나 거부(validate)한다. REST 핸들러 안에서(핸들러 체인이 아니라) 실행된다
([01](01-request-pipeline.md)).

## 두 종류, 두 단계

어드미션 플러그인은 두 인터페이스로 나뉜다(`admission/interfaces.go`):

1. **Mutating** (`MutationInterface`): 객체를 **고친다**. 예) 기본 톨러레이션 추가, SA 토큰 마운트 주입.
2. **Validating** (`ValidationInterface`): 객체를 **검사만** 한다(거부 가능, 변형 불가).

실행 순서는 항상 **모든 mutating 먼저 → 그다음 모든 validating**. 이유: 검증은 *최종 형태*의 객체를
봐야 하므로 변형이 끝난 뒤여야 한다. 여러 플러그인은 `chainAdmissionHandler`
(`admission/chain.go:23`)로 묶여 순차 실행된다.

> Mutating 단계 후 다른 플러그인이 또 변형할 수 있으므로 **reinvocation**(`admission/reinvocation.go`)
> 으로 일부 mutating 플러그인을 한 번 더 호출해 수렴시킨다.

## 빌트인 플러그인

컴파일된 어드미션 플러그인은 `kubernetes/plugin/pkg/admission/` 아래에 있다. 일부:

| 플러그인 | 역할 |
|----------|------|
| `serviceaccount` | 파드에 SA와 토큰 볼륨을 주입 |
| `limitranger` | LimitRange에 따라 기본 리소스 요청/제한 채움·검증 |
| `resourcequota` | 네임스페이스 ResourceQuota 초과 요청 거부 |
| `noderestriction` | kubelet이 자기 노드/파드 외 객체를 못 바꾸게 |
| `podtolerationrestriction`, `podnodeselector` | 네임스페이스 기본 스케줄 제약 적용 |
| `priority` | PriorityClass → Pod.spec.priority 해석 |
| `security`, `podsecurity` | 파드 보안 표준 시행 |
| `defaulttolerationseconds`, `nodetaint` 등 | 각종 기본값/제약 |

어떤 플러그인을 켜는지는 apiserver 플래그(`--enable-admission-plugins`)와 기본 목록으로 정해진다.

## 동적 어드미션 — 웹훅

빌트인 외에, 클러스터 운영자가 **웹훅**으로 어드미션을 확장할 수 있다
(`admission/plugin/webhook/`). 두 가지:

- **MutatingAdmissionWebhook** (`webhook/mutating/`)
- **ValidatingAdmissionWebhook** (`webhook/validating/`)

apiserver가 매칭되는 요청을 외부 HTTPS 서버(`AdmissionReview` 요청)로 보내고, 응답으로 허용/거부 또는
patch를 받는다. 어떤 요청에 어떤 웹훅을 부를지는 `MutatingWebhookConfiguration`/
`ValidatingWebhookConfiguration` 객체로 등록한다. 매칭 술어는 `webhook/predicates/`,
조건은 `webhook/matchconditions/`.

## CEL 기반 정책 — 웹훅 없는 검증

외부 서버 없이 apiserver 내부에서 정책을 평가하는 길도 있다(`admission/plugin/policy/`, `cel/`).
**ValidatingAdmissionPolicy**(및 Mutating 버전)는 CEL(Common Expression Language) 식으로 검증 규칙을
객체로 선언한다. 웹훅의 네트워크 왕복·운영 부담 없이 정책을 표현할 수 있다.

## 거부의 의미

어떤 validating 단계든 거부하면 요청 전체가 실패하고(에러 응답), etcd에 아무것도 쓰이지 않는다.
mutating 단계의 변형은 최종 저장될 객체에 반영된다.

## 더 읽을 곳
- [05-registry-storage.md](05-registry-storage.md) — 어드미션 후 저장 경로
- [00-foundations/03](../00-foundations/03-api-object-model.md) — 어드미션이 다루는 객체 구조
