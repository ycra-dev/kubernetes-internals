# 70 · autoscaler

**근거 레포**: `autoscaler` (cluster-autoscaler, vertical-pod-autoscaler, multidimensional-pod-autoscaler)

`autoscaler` 레포는 **클러스터 위에서 도는 별도 오토스케일링 컴포넌트들**의 모음이다. 빌트인 HPA와
혼동하지 않는 것이 중요하다 — 세 종류의 "확장"은 서로 다른 차원을 조정한다.

## 세 가지 오토스케일링 — 무엇을 늘리나

| 종류 | 조정 대상 | 어디에 있나 |
|------|-----------|-------------|
| **HPA** (Horizontal Pod Autoscaler) | Pod **개수** | **빌트인**: controller-manager ([11-controller-manager/01](../11-controller-manager/01-controller-catalog.md)) |
| **VPA** (Vertical Pod Autoscaler) | Pod의 **리소스 요청/제한** | 이 레포 `vertical-pod-autoscaler/` |
| **Cluster Autoscaler (CA)** | **노드 개수** | 이 레포 `cluster-autoscaler/` |

```
부하 증가
  ├─ HPA: replica 3 → 10 (Pod 더 띄움)
  │        그런데 노드에 자리가 없으면 Pod가 Pending(unschedulable)
  ├─ CA:  Pending Pod 감지 → 노드 추가 (클라우드 API로)
  └─ VPA: Pod가 자주 OOM → 메모리 요청 상향 추천/적용
```

HPA와 CA는 **협력**한다: HPA가 Pod를 늘려 자리가 부족해지면, CA가 노드를 늘려 그 Pod를 수용한다.

## 문서

| # | 문서 | 내용 |
|---|------|------|
| (이 문서) | cluster-autoscaler 동작 루프 | |
| 01 | [vpa.md](01-vpa.md) | VPA와 multidimensional |
| 02 | [hpa.md](02-hpa.md) | HPA 스케일 알고리즘(수식/tolerance/안정화) |

## Cluster Autoscaler — 노드 수 조정 루프

진입점은 `cluster-autoscaler/core/static_autoscaler.go`의 `StaticAutoscaler`다. 주기적으로 도는 루프:

```
RunOnce (static_autoscaler.go):
  1. 스케줄 안 된(unschedulable) Pod가 있나?  ── 자원 부족으로 Pending
       → 있으면 ScaleUp: 어떤 노드 그룹을 키우면 그 Pod가 들어갈지 계산
                         → 클라우드 API로 노드 추가 (cluster-autoscaler/core/scaleup/)
  2. 오랫동안 저활용(underutilized)인 노드가 있나?
       → 있으면 ScaleDown: 그 노드의 Pod를 다른 곳으로 옮길 수 있으면
                          노드를 비우고(drain) 제거 (core/scaledown/)
```

핵심: CA는 **스케줄러의 결정을 시뮬레이션**한다. "이 노드를 추가하면 Pending Pod가 스케줄될까?",
"이 노드를 제거하면 그 Pod들이 다른 노드에 들어갈까?"를 스케줄러 로직으로 판단한다
([12-scheduler](../12-scheduler/)). 그래서 스케줄 제약(taint/affinity/리소스)을 존중한다.

> CA는 노드 사이 시간 버퍼를 둔다(`unschedulablePodTimeBuffer`, `static_autoscaler.go:81` 등) —
> 일시적 Pending에 과민 반응해 노드를 추가했다 곧 제거하는 플래핑을 막는다.

## cloudprovider — 어떻게 노드를 만드나

CA는 클라우드별 **cloudprovider**(`cluster-autoscaler/cloudprovider/`: aws, azure, gce 등 수십 종)로
실제 노드 그룹(ASG/MIG 등)을 키우고 줄인다. CA 핵심 로직은 클라우드 중립이고, provider 인터페이스가
"노드 그룹 크기 조정"을 추상화한다.

## 더 읽을 곳
- [01-vpa.md](01-vpa.md) — 리소스 요청 조정
- [11-controller-manager/01](../11-controller-manager/01-controller-catalog.md) — HPA(빌트인)
- [12-scheduler](../12-scheduler/) — CA가 시뮬레이션하는 스케줄 로직
