# 17.01 · 워크로드 신원과 Secret

**근거**: `kubernetes/pkg/serviceaccount/`, `staging/.../api/authentication/v1/types.go`(TokenRequest `:143`),
`plugin/pkg/admission/serviceaccount/`

Pod 안에서 도는 애플리케이션도 apiserver에 접근하려면 **신원**이 필요하다. 그 신원이 **ServiceAccount(SA)**
이고, 증명 수단이 **토큰**이다. 이 문서는 SA 토큰과 Secret이 어떻게 만들어지고 Pod에 전달되는지를 본다.

## ServiceAccount — Pod의 신원

- 모든 Pod는 하나의 ServiceAccount에 묶인다(지정 안 하면 namespace의 `default`).
- SA로 인증하면 사용자 이름은 `system:serviceaccount:<namespace>:<name>`, 그룹은 `system:serviceaccounts`
  등이 된다([10-apiserver/02](../10-apiserver/02-authentication.md)).
- 이 신원에 RBAC RoleBinding을 걸어 "이 SA는 이 namespace의 ConfigMap만 읽기" 식으로 권한을 준다
  ([10-apiserver/03](../10-apiserver/03-authorization.md)).

## bound ServiceAccount 토큰 — 현재 방식

과거에는 SA마다 영구 토큰을 Secret으로 만들어 Pod에 마운트했다. 문제: 토큰이 만료되지 않고, 유출되면
계속 유효하다.

현재는 **bound token**(`pkg/serviceaccount/`, TokenRequest API `authentication/v1/types.go:143`)을 쓴다:

- **TokenRequest API**로 발급되는 JWT로, **특정 Pod·시간·대상(audience)에 묶인다(bound)**.
- **수명이 짧고 자동 갱신**된다 — kubelet이 만료 전에 새 토큰을 받아 갱신.
- Pod가 사라지면 토큰도 무효(Pod에 bound).

이 토큰은 **projected 볼륨**으로 Pod에 주입된다(보통 `/var/run/secrets/kubernetes.io/serviceaccount/`).
주입을 트리거하는 것은 어드미션 플러그인 `serviceaccount`
(`plugin/pkg/admission/serviceaccount/`, [10-apiserver/04](../10-apiserver/04-admission.md))다.

> `pkg/serviceaccount/legacy.go`는 구식 영구 토큰 호환 경로다. 새 클러스터는 bound 토큰을 기본으로 쓴다.

## projected 볼륨

projected 볼륨은 여러 소스(SA 토큰, ConfigMap, Secret, downward API)를 한 디렉토리로 합쳐 주입한다.
SA 토큰뿐 아니라 CA 인증서, namespace 정보도 함께 들어가 Pod가 apiserver를 신뢰하며 통신할 수 있게 한다.
마운트는 kubelet volume manager가 한다([13-kubelet/05](../13-kubelet/05-volume-manager.md)).

## Secret — 비밀 데이터

**Secret**은 비밀번호/토큰/키 등을 담는 객체다. 특징과 주의:

- **저장**: 다른 객체처럼 etcd에 저장된다([10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md)).
  기본은 base64 인코딩일 뿐 암호화가 아니므로, **etcd 저장 시 암호화(encryption at rest)** 를 켜야 진짜
  보호된다(apiserver의 EncryptionConfiguration, 외부 KMS 연동은 `staging/.../kms`).
- **전달**: Secret을 볼륨으로 마운트하거나 환경변수로 주입. 볼륨 마운트는 보통 tmpfs(메모리)라 디스크에
  안 남는다.
- **접근 제한**: 어떤 Pod가 어떤 Secret을 쓰는지를 **Node authorizer**가 추적해, 한 노드의 kubelet은
  자기 노드 Pod가 실제 참조하는 Secret만 받을 수 있다([13-kubelet/06](../13-kubelet/06-node-registration.md)).
  이것이 한 노드 탈취가 전체 비밀로 번지지 않게 하는 핵심.

## 외부 워크로드 신원 — 토큰 연합

bound 토큰은 OIDC 호환 JWT라, 클라우드 IAM 등 **외부 시스템**이 이를 검증해 클라우드 권한을 줄 수 있다
(workload identity federation). `staging/.../externaljwt`가 외부 JWT 서명자 연동을 지원한다. 이로써 Pod가
정적 클라우드 자격증명 없이 클라우드 리소스에 접근할 수 있다.

## 더 읽을 곳
- [10-apiserver/02](../10-apiserver/02-authentication.md) — SA 토큰 검증
- [02-pod-security.md](02-pod-security.md) — Pod 권한 제한
- [13-kubelet/06](../13-kubelet/06-node-registration.md) — Node authorizer의 Secret 격리
