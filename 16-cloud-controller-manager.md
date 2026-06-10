# 16 · cloud-controller-manager

**근거 레포**: `kubernetes` (`cmd/cloud-controller-manager/`,
`staging/src/k8s.io/cloud-provider/`)

cloud-controller-manager(CCM)는 **클라우드 제공자(AWS/GCP/Azure 등)와 연동하는 컨트롤러들**을 모아
돌리는 컴포넌트다. 원래 이 로직은 kube-controller-manager 안에 있었지만, 클라우드 의존 코드를 핵심에서
**분리**하기 위해 별도 바이너리로 떼어냈다.

## 왜 분리했나

kube-controller-manager([11-controller-manager](11-controller-manager/))에 클라우드별 코드가 박혀 있으면:
- Kubernetes 코어 릴리스가 모든 클라우드 SDK에 묶인다.
- 새 클라우드를 추가하려면 코어를 수정해야 한다.

그래서 클라우드 의존 컨트롤러를 CCM으로 옮기고, 클라우드별 구현은 **`cloud.Interface`** 뒤로 추상화했다.
이로써 클라우드 제공자는 코어를 건드리지 않고 **외부(out-of-tree)** 에서 자기 CCM을 만들 수 있다.

## cloud.Interface — 클라우드 추상화

`staging/src/k8s.io/cloud-provider/cloud.go:43`의 `Interface`가 경계다. 제공자가 구현하는 하위 인터페이스:

| 메서드 | 반환 | 역할 |
|--------|------|------|
| `LoadBalancer()` (`cloud.go`) | LoadBalancer | Service type=LoadBalancer용 클라우드 LB 생성/관리 |
| `Instances()` / `InstancesV2()` | Instances | 노드↔클라우드 VM 매핑, 노드 메타데이터 |
| `Routes()` | Routes | Pod CIDR용 클라우드 라우트 설정 |
| `Zones()` | Zones | 노드의 존/리전 |

`InstancesV2`(`cloud.go` 주석)는 외부 제공자용 최적화 버전으로, Zones 호출을 줄이도록 설계됐다.

## CCM의 컨트롤러들

`staging/src/k8s.io/cloud-provider/controllers/`:

| 컨트롤러 | 역할 |
|----------|------|
| `node/` | 새 노드에 클라우드 메타데이터(존/인스턴스 타입/주소) 채우고, 클라우드에서 사라진 VM의 노드 정리 |
| `nodelifecycle/` | 클라우드 기준으로 노드가 실제 삭제됐는지 확인(kube-controller-manager의 NodeLifecycle와 협력, [11-controller-manager/05](11-controller-manager/05-node-lifecycle.md)) |
| `route/` | 노드별 Pod CIDR을 클라우드 라우팅 테이블에 등록(오버레이 없는 네트워킹) |
| `service/` | **Service type=LoadBalancer** → 클라우드 LB 프로비저닝, EndpointSlice 변화에 맞춰 갱신 |

이들도 전부 reconcile 루프다([00-foundations/05](00-foundations/05-controller-pattern.md)) — apiserver를
watch하고, 차이를 클라우드 API 호출로 좁힌다.

## Service LoadBalancer의 흐름

[40-networking/02](40-networking/02-service-discovery.md)에서 미뤄둔 부분이다:

```
사용자: Service type=LoadBalancer 생성
   │  CCM service 컨트롤러가 watch
   ▼
클라우드 API: 외부 LB 프로비저닝 → 외부 IP 할당
   │  Service.status.loadBalancer 에 IP 기록
   ▼
LB → 노드들의 NodePort → kube-proxy/cilium → 백엔드 Pod
   ([14-kube-proxy/02], [42-cilium/04])
```

즉 클라우드 LB는 "외부 → 노드"까지만 책임지고, "노드 → Pod"는 여전히 kube-proxy/cilium 데이터플레인이
한다.

## in-tree vs out-of-tree

과거에는 클라우드 코드가 코어에 in-tree로 있었으나, 지금은 대부분 **out-of-tree CCM**(제공자가 별도 배포)
으로 이전됐다. `cloud-provider` 라이브러리가 그 공통 골격(컨트롤러 + 인터페이스)을 제공하고, 제공자는
`cloud.Interface`만 구현한다.

## 더 읽을 곳
- [11-controller-manager](11-controller-manager/) — 클라우드 비의존 컨트롤러(여기서 분리됨)
- [40-networking/02](40-networking/02-service-discovery.md) — LoadBalancer Service 모델
