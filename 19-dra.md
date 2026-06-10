# 19 · DRA — 동적 리소스 할당 (Dynamic Resource Allocation)

**근거 레포**: `kubernetes` (`staging/.../api/resource/`, `staging/.../dynamic-resource-allocation/`,
`pkg/scheduler/framework/plugins/dynamicresources/`, `pkg/kubelet/cm/dra/`,
`pkg/controller/resourceclaim/`)

DRA는 **GPU·FPGA·특수 네트워크 카드 같은 장치를 유연하게 할당**하는 차세대 모델이다. 기존
device plugin([13-kubelet/03](13-kubelet/03-resource-mgmt.md))이 "이 노드에 GPU가 N개" 같은
**개수 기반 정수 카운트**만 다뤘다면, DRA는 "공유 가능한 GPU의 특정 파티션", "두 장치가 같은 NUMA에"
같은 **구조적·파라미터화된 요구**를 표현한다.

## 왜 기존 방식으론 부족했나

device plugin 모델의 한계:
- 장치를 **불투명한 정수**(`nvidia.com/gpu: 2`)로만 셈 — 어떤 GPU인지, 공유/분할 가능한지 표현 불가.
- 스케줄러가 장치 속성을 모르고 그냥 개수만 봄 — 토폴로지/공유 제약을 고려 못 함.

DRA는 장치를 **속성을 가진 1급 API 객체**로 만들어, 스케줄러가 그 속성을 보고 배정하게 한다.

## API 객체 (resource.k8s.io)

`staging/src/k8s.io/api/resource/v1/types.go`. 핵심 타입:

| 객체 | 역할 |
|------|------|
| **ResourceSlice** (`types.go:95`) | 노드(또는 드라이버)가 광고하는 **가용 장치 풀**. "이 노드에 이런 GPU들이 있다"를 드라이버가 게시 |
| **DeviceClass** (`types.go:1942`) | 장치 종류/선택 기준의 템플릿(관리자/벤더 제공) |
| **ResourceClaim** (`types.go:887`) | "이런 장치가 필요하다"는 **요청**(PVC와 유사한 위치). CEL 식으로 장치 셀렉션 표현 |
| **ResourceClaimTemplate** (`types.go:2030`) | Pod마다 claim을 생성하기 위한 틀(PVC 템플릿과 유사) |

PV/PVC 모델([50-storage](50-storage/))과 구조가 닮았다: **ResourceSlice(공급) ↔ ResourceClaim(요구)**,
그리고 그 사이를 스케줄러/컨트롤러가 잇는다.

## 동작 흐름 — 여러 컴포넌트의 협력

```
[1] 드라이버(노드): ResourceSlice 게시 — "내 노드의 장치 목록/속성"
[2] 사용자: Pod 가 ResourceClaim(Template) 참조 — "이런 장치 필요"
[3] resourceclaim 컨트롤러: Template 으로 Pod별 ResourceClaim 생성
       (pkg/controller/resourceclaim/)
[4] 스케줄러(dynamicresources 플러그인): ResourceSlice 를 보고
       claim 을 만족하는 장치가 있는 노드로 Pod 배정 + 장치 할당 결정
       (pkg/scheduler/framework/plugins/dynamicresources/)
[5] kubelet(cm/dra): 노드에서 DRA 드라이버를 호출해 장치를 컨테이너에 준비/주입
       (pkg/kubelet/cm/dra/)
```

각 단계의 코드:
- **[3] resourceclaim 컨트롤러**(`pkg/controller/resourceclaim/controller.go`): Pod의
  ResourceClaimTemplate을 보고 실제 ResourceClaim을 만들고, Pod 삭제 시 정리한다 — 또 하나의 reconcile
  루프([00-foundations/05](00-foundations/05-controller-pattern.md)).
- **[4] dynamicresources 스케줄러 플러그인**(`dynamicresources.go`, `dra_manager.go`): Filter/Reserve
  확장점([12-scheduler/01](12-scheduler/01-framework.md))에서 ResourceSlice를 조회해, claim을 만족하는
  장치가 있는 노드만 통과시키고 구체 장치를 예약한다. `structured`(구조적 파라미터)로 속성 기반 매칭.
- **[5] kubelet DRA 매니저**(`pkg/kubelet/cm/dra/manager.go`, `plugin/`): 노드의 DRA 드라이버(kubelet
  플러그인)를 gRPC로 호출해 장치를 컨테이너 환경에 prepare/주입. claim 상태는 `claiminfo.go`/`state/`로 추적.

## 공용 라이브러리

`staging/src/k8s.io/dynamic-resource-allocation/`이 드라이버 개발용 공용 코드다: `resourceslice/`(슬라이스
게시 헬퍼), `kubeletplugin/`(노드 드라이버 골격), `cel/`(claim 셀렉션 식 평가),
`structured/`(구조적 파라미터). 벤더는 이를 써서 자기 장치용 DRA 드라이버를 만든다.

## device plugin과의 관계

DRA는 device plugin을 **대체하는 방향**이지만, 단순 개수 할당은 device plugin이 여전히 쓰인다. DRA는
공유/분할/토폴로지 같은 복잡한 요구가 있을 때 쓴다. 두 메커니즘 다 최종적으로는 kubelet cm이
조율한다([13-kubelet/03](13-kubelet/03-resource-mgmt.md)).

## 성숙도 — feature gate

DRA는 발전 중인 기능이라 여러 API 버전(`resource.k8s.io` v1alpha3/v1beta1/v1beta2/v1)이 공존하고,
feature gate로 단계 관리된다([00-foundations/07](00-foundations/07-component-base.md),
[91-versioning-skew.md](91-versioning-skew.md)). 설계 배경은 sig-node 계열 KEP에 있다
([90-enhancements](90-enhancements.md)).

## 더 읽을 곳
- [12-scheduler/02](12-scheduler/02-plugins.md) — dynamicresources 플러그인
- [13-kubelet/03](13-kubelet/03-resource-mgmt.md) — device manager / 자원 관리
- [50-storage](50-storage/) — 유사 구조의 공급/요구 모델(PV/PVC)
