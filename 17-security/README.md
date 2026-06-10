# 17 · 보안 모델

**근거 레포**: `kubernetes` (`staging/.../apiserver/pkg/authentication`,`authorization`,
`plugin/pkg/auth`, `pkg/serviceaccount`, `staging/.../pod-security-admission`)

보안은 한 컴포넌트에 있지 않고 여러 계층을 **횡단**한다. 이 폴더는 흩어진 보안 메커니즘(인증·인가·어드미션·
노드 격리·워크로드 신원·Pod 보안)을 **신뢰 경계(trust boundary)** 관점에서 하나로 꿴다. 개별 메커니즘의
상세는 각 컴포넌트 문서에 있고, 여기선 "전체 그림"을 본다.

## 신뢰 경계 한눈에

```
사용자/외부 ──► apiserver  ◄── 모든 정책 시행의 단일 지점
   인증(누구) → 인가(권한) → 어드미션(내용)
        │
        ├─ 워크로드 신원: ServiceAccount + bound 토큰 ([01])
        ├─ 노드 격리: Node authorizer + NodeRestriction ([10-apiserver/03])
        ├─ 비밀: Secret (etcd 저장, 마운트) ([01])
        └─ Pod 보안: Pod Security Admission ([02])
```

핵심 원칙: **apiserver가 유일한 정책 시행 지점**이다([00-foundations/02](../00-foundations/02-architecture.md)).
모든 접근이 인증→인가→어드미션을 거치므로, 보안 정책을 한 곳에 집중할 수 있다.

## 세 계층의 방어

| 질문 | 메커니즘 | 문서 |
|------|----------|------|
| 너는 누구인가 | 인증(토큰/인증서/SA) | [10-apiserver/02](../10-apiserver/02-authentication.md) |
| 이 동작을 할 수 있나 | 인가(RBAC/Node) | [10-apiserver/03](../10-apiserver/03-authorization.md) |
| 이 내용이 허용되나 | 어드미션(PSA/웹훅/쿼터) | [10-apiserver/04](../10-apiserver/04-admission.md) |

## 격리의 축들

- **사용자 격리**: RBAC로 namespace/resource 수준 권한 분리.
- **노드 격리**: Node authorizer + NodeRestriction으로 탈취된 노드의 영향 제한
  ([13-kubelet/06](../13-kubelet/06-node-registration.md)).
- **워크로드 격리**: ServiceAccount로 Pod별 신원 부여, NetworkPolicy로 통신 제한
  ([42-cilium/03](../42-cilium/03-networkpolicy.md)).
- **Pod 권한 격리**: Pod Security Admission으로 특권 컨테이너 등 제한 → [02](02-pod-security.md).

## 문서

| # | 문서 | 내용 |
|---|------|------|
| 01 | [serviceaccount-secrets.md](01-serviceaccount-secrets.md) | 워크로드 신원(SA·bound 토큰)과 Secret |
| 02 | [pod-security.md](02-pod-security.md) | Pod Security Admission(privileged/baseline/restricted) |
| 03 | [pki-certificates.md](03-pki-certificates.md) | 다중 CA, 컴포넌트 인증서, TLS bootstrap/rotation |
| 04 | [encryption-at-rest.md](04-encryption-at-rest.md) | etcd 저장 암호화(provider/KMS envelope) |

## 더 읽을 곳
- [10-apiserver/02~04](../10-apiserver/) — 인증/인가/어드미션 상세
- [13-kubelet/06](../13-kubelet/06-node-registration.md) — 노드 신원과 격리
