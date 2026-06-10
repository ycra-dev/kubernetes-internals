# 45 · 정책과 멀티테넌시

**근거 레포**: `kubernetes` (`plugin/pkg/admission/{resourcequota,limitranger}`, `pkg/quota/`,
`pkg/controller/{resourcequota,disruption}`, `api/scheduling/v1`)

한 클러스터를 여러 팀/사용자가 공유할 때(멀티테넌시), 누군가 자원을 독식하거나 남의 워크로드를 망치지
못하게 하는 **정책 장치**들이 필요하다. 이 폴더는 자원 한도(쿼터/리밋), 가용성 보호(PDB), 우선순위,
네임스페이스 격리를 다룬다.

## 격리의 단위 — Namespace

**Namespace**는 멀티테넌시의 기본 경계다:
- 이름 유일성의 범위([00-foundations/03](../00-foundations/03-api-object-model.md)의 ObjectMeta).
- **RBAC**의 적용 범위(이 네임스페이스의 이 리소스만, [10-apiserver/03](../10-apiserver/03-authorization.md)).
- **ResourceQuota/LimitRange**의 적용 범위.
- 삭제 시 내부 객체 일괄 정리(Namespace 컨트롤러,
  [11-controller-manager/01](../11-controller-manager/01-controller-catalog.md)).

단, Namespace는 **네트워크/노드 격리가 아니다** — 그건 NetworkPolicy([42-cilium/03](../42-cilium/03-networkpolicy.md))와
스케줄 제약의 몫이다. Namespace는 "API 수준의 논리적 경계"다.

## 문서

| # | 문서 | 내용 |
|---|------|------|
| 01 | [quota-limitrange.md](01-quota-limitrange.md) | ResourceQuota(총량), LimitRange(개별 기본/한도) |
| 02 | [pdb-priority.md](02-pdb-priority.md) | PodDisruptionBudget, PriorityClass/선점 |
| 03 | [eviction-api.md](03-eviction-api.md) | Eviction API(PDB 시행, drain 동작) |

## 정책 시행은 어디서?

| 정책 | 시행 지점 |
|------|-----------|
| ResourceQuota | 어드미션(초과 거부) + 컨트롤러(사용량 집계) |
| LimitRange | 어드미션(기본값/한도 적용) |
| PodDisruptionBudget | Eviction API + Disruption 컨트롤러 |
| PriorityClass | 어드미션(priority 해석) + 스케줄러(선점) |
| RBAC | 인가([10-apiserver/03](../10-apiserver/03-authorization.md)) |
| Pod Security | 어드미션([17-security/02](../17-security/02-pod-security.md)) |

대부분 **어드미션**([10-apiserver/04](../10-apiserver/04-admission.md))에서 1차로 시행되고, 일부는
컨트롤러/스케줄러가 보조한다.

## 더 읽을 곳
- [17-security](../17-security/) — 보안 정책(인증/인가/Pod Security)
- [10-apiserver/04](../10-apiserver/04-admission.md) — 정책 시행의 주 무대(어드미션)
