# 70.01 · VPA (Vertical Pod Autoscaler)

**근거**: `autoscaler/vertical-pod-autoscaler/pkg/`

VPA는 Pod의 **개수**가 아니라 **크기(리소스 요청/제한)** 를 조정한다. "이 컨테이너는 메모리를 더(또는 덜)
줘야 한다"를 관측 기반으로 추천하고 적용한다. HPA가 가로(replica)라면 VPA는 세로(per-Pod 자원)다.

## 세 컴포넌트

VPA는 세 부분으로 나뉜다(`vertical-pod-autoscaler/pkg/`):

| 컴포넌트 | 디렉토리 | 역할 |
|----------|----------|------|
| **Recommender** | `recommender/` | 실제 사용량(메트릭)을 보고 적정 요청값을 추천 |
| **Updater** | `updater/` | 추천과 크게 어긋난 Pod를 축출해 새 값으로 재생성되게 |
| **Admission Controller** | `admission-controller/` | 새로 생기는 Pod에 추천값을 주입(어드미션 웹훅) |

```
Recommender: 사용량 히스토리 분석 → VPA 객체.status 에 추천값 기록
                                        │
Updater: 추천과 동떨어진 Pod 를 축출 (evict)
                                        │
(Pod 재생성 시) Admission Controller: 새 Pod 스펙에 추천 요청값 주입
```

## Recommender — 통계 기반 추천

`recommender/`가 핵심 지능이다. 컨테이너의 CPU/메모리 사용량을 시계열로 수집(`recommender/input/`)하고,
통계 모델(`recommender/logic/`, `model/`)로 적정 요청값을 계산한다(예: 사용량 분포의 특정 백분위수).
결과는 VPA 리소스의 status에 "target/lowerBound/upperBound"로 기록된다. 체크포인트(`recommender/checkpoint/`)로
히스토리를 영속화해 재시작 후에도 추천을 유지한다.

## 왜 축출이 필요한가

전통적으로 **실행 중인 Pod의 리소스 요청은 변경 불가**였다. 그래서 VPA Updater는 값을 바꾸려면 Pod를
**축출(재생성)** 해야 했다 — 새 Pod가 뜰 때 Admission Controller가 추천값을 주입한다. (이는 가용성에
영향을 주므로 PodDisruptionBudget을 존중한다.)

> 근래에는 **in-place pod resize**(Pod를 재시작하지 않고 자원 조정) 기능이 발전 중이며, 이를 통해 축출
> 없이 세로 확장이 가능해지는 방향이다. 관련 어드미션은 빌트인 `podresize` 플러그인
> ([10-apiserver/04](../10-apiserver/04-admission.md))에서도 보인다.

## HPA와 함께 쓸 때 주의

같은 리소스 메트릭(CPU/메모리)으로 HPA와 VPA를 동시에 돌리면 충돌한다(둘 다 그 메트릭에 반응). 그래서
보통 HPA는 커스텀/다른 메트릭으로, VPA는 리소스로 역할을 나눈다.

## multidimensional-pod-autoscaler

`autoscaler/multidimensional-pod-autoscaler/`는 **가로(HPA)와 세로(VPA)를 함께** 조정하려는
컴포넌트다(설계 문서 `AEP.md` 중심). replica 수와 per-Pod 자원을 동시에 최적화하는 것을 목표로 한다.

## 더 읽을 곳
- [README.md](README.md) — CA/HPA와의 구분
- [13-kubelet/03](../13-kubelet/03-resource-mgmt.md) — 조정되는 리소스 요청이 시행되는 곳
