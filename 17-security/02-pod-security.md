# 17.02 · Pod Security Admission

**근거**: `kubernetes/staging/src/k8s.io/pod-security-admission/`
(`api/constants.go`의 `LevelPrivileged`/`LevelBaseline`/`LevelRestricted`)

Pod는 호스트를 위협할 수 있는 강력한 기능을 요청할 수 있다 — 특권 모드, 호스트 네임스페이스 공유,
임의 볼륨, root 실행. **Pod Security Admission(PSA)** 은 이를 표준 등급으로 제한하는 빌트인 어드미션이다
([10-apiserver/04](../10-apiserver/04-admission.md)).

## 세 가지 표준 레벨

`pod-security-admission/api/constants.go`에 정의된 `Level`(`LevelPrivileged`/`LevelBaseline`/
`LevelRestricted`, `helpers_test.go:59`에서 셋이 함께 쓰임):

| 레벨 | 의미 |
|------|------|
| **privileged** | 제한 없음. 모든 것 허용(시스템/인프라 워크로드용) |
| **baseline** | 알려진 권한 상승을 막는 최소 제한(특권 컨테이너, 호스트 네임스페이스 등 금지) |
| **restricted** | 강력히 제한(non-root 실행 강제, capabilities 대폭 축소, seccomp 등 모범사례 강제) |

레벨은 점점 엄격해진다: privileged ⊃ baseline ⊃ restricted.

## 세 가지 모드 — 같은 정책, 다른 결과

PSA는 **namespace 라벨**로 적용한다. 한 namespace에 레벨과 모드를 라벨로 지정:

| 모드 | 라벨 | 동작 |
|------|------|------|
| **enforce** | `pod-security.kubernetes.io/enforce=<level>` | 위반 Pod **거부** |
| **audit** | `.../audit=<level>` | 허용하되 감사 로그에 기록 |
| **warn** | `.../warn=<level>` | 허용하되 사용자에게 경고 표시 |

세 모드를 함께 쓸 수 있다 — 예: `enforce=baseline` + `warn=restricted`(baseline은 강제, restricted는
경고만)로 점진적으로 조여간다.

## 어떻게 동작하나

PSA는 **validating admission**으로 구현된다([10-apiserver/04](../10-apiserver/04-admission.md)). Pod
생성/수정 시:

1. 그 Pod가 속한 namespace의 PSA 라벨을 읽어 적용 레벨/모드를 결정.
2. Pod 스펙(securityContext, 볼륨, 호스트 네임스페이스 등)을 레벨 기준과 대조(`api/`의 검사 로직).
3. enforce면 위반 시 거부, audit/warn이면 기록/경고.

웹훅이 아니라 apiserver 내장이므로 네트워크 왕복이 없다(PodSecurityPolicy(PSP)의 후계 — PSP는 제거됨).

## PSA로 부족할 때

PSA는 **표준 등급**만 제공한다. 더 세밀하거나 조직 특화 정책이 필요하면:
- **ValidatingAdmissionPolicy(CEL)** 로 커스텀 규칙([10-apiserver/04](../10-apiserver/04-admission.md)).
- 외부 정책 엔진(OPA/Gatekeeper, Kyverno 등)을 validating 웹훅으로 연동.

## 런타임 격리와의 관계

PSA는 "어떤 Pod 스펙을 허용하나"(입장 시점)를 본다. 실제 격리 **강제**는 런타임이 한다 — runc의
namespaces/cgroups/seccomp/capabilities([30-containerd/03](../30-containerd/03-runtime-shim-oci.md)).
더 강한 격리가 필요하면 RuntimeClass로 gVisor/Kata 같은 샌드박스 런타임을 선택한다
([13-kubelet/02](../13-kubelet/02-cri.md)).

## 더 읽을 곳
- [10-apiserver/04](../10-apiserver/04-admission.md) — PSA가 도는 어드미션 단계
- [30-containerd/03](../30-containerd/03-runtime-shim-oci.md) — 실제 격리 강제(runc)
