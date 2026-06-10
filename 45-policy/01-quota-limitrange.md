# 45.01 · ResourceQuota와 LimitRange

**근거**: `kubernetes/plugin/pkg/admission/resourcequota/`, `kubernetes/plugin/pkg/admission/limitranger/`,
`kubernetes/pkg/quota/`, `kubernetes/pkg/controller/resourcequota/`

두 객체가 네임스페이스의 자원 사용을 통제한다: **ResourceQuota**(네임스페이스 **총량** 제한)와
**LimitRange**(**개별 객체**의 기본값/한도).

## ResourceQuota — 네임스페이스 총량

ResourceQuota는 "이 네임스페이스가 쓸 수 있는 자원의 합"을 제한한다:

- **컴퓨트**: CPU/메모리의 requests/limits 총합.
- **객체 수**: Pod/Service/ConfigMap/PVC 등의 개수.
- **스토리지**: PVC 용량 총합.

### 두 부분이 협력
1. **어드미션**(`plugin/pkg/admission/resourcequota/`): 새 객체 생성 시 "이걸 추가하면 쿼터를 넘나"를
   검사해 초과면 **거부**([10-apiserver/04](../10-apiserver/04-admission.md)).
2. **컨트롤러**(`pkg/controller/resourcequota/`): 현재 사용량을 집계해 ResourceQuota.status에 기록
   (`pkg/quota/`의 계산 로직). 어드미션이 이 status를 기준으로 판단.

> 중요: ResourceQuota가 있는 네임스페이스에서는 **모든 Pod가 requests/limits를 명시해야** 한다(쿼터
> 계산이 가능하도록). 그래서 LimitRange와 짝지어 쓴다(아래).

## LimitRange — 개별 객체의 기본/한도

LimitRange는 네임스페이스 안 **개별 Pod/컨테이너/PVC**에 적용된다(`plugin/pkg/admission/limitranger/`):

- **기본값(default)**: requests/limits를 안 적은 컨테이너에 기본값 주입 — ResourceQuota가 작동하도록.
- **최소/최대**: 컨테이너가 요청할 수 있는 자원의 상·하한.
- **비율**: limits/requests 비율 제한.

LimitRange는 **mutating + validating 어드미션**으로 동작한다 — 기본값을 채우고(mutate), 범위를
벗어나면 거부(validate).

## 둘의 조합 (전형적 패턴)

```
LimitRange:   requests/limits 미설정 컨테이너에 기본값 주입 + 상하한 검증
   │
ResourceQuota: 네임스페이스 총합이 한도 내인지 검증 (LimitRange가 채운 값 기준)
```

LimitRange가 "모든 Pod가 자원을 명시하도록" 보장하고, ResourceQuota가 "그 총합이 한도 내인지" 보장한다.
함께 쓰면 한 팀이 네임스페이스를 받아 그 안에서 자유롭게 쓰되 **총량은 못 넘게** 할 수 있다 — 멀티테넌시의
핵심.

## QoS와의 연결

LimitRange가 채운 requests/limits 관계가 Pod의 **QoS 클래스**를 결정한다
([13-kubelet/03](../13-kubelet/03-resource-mgmt.md)) — 이는 노드에서의 축출 순서로 이어진다
([13-kubelet/04](../13-kubelet/04-eviction.md)). 즉 정책(API)이 노드 동작(런타임)으로 연결된다.

## 더 읽을 곳
- [10-apiserver/04](../10-apiserver/04-admission.md) — 쿼터/리밋이 도는 어드미션
- [13-kubelet/03](../13-kubelet/03-resource-mgmt.md) — requests/limits의 노드 측 시행
