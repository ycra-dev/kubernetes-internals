# 40.02 · 서비스 디스커버리

**근거**: `kubernetes/staging/src/k8s.io/api/core/v1/types.go` (`ServiceSpec`, `:5966`),
`kubernetes/pkg/controller/endpointslice/`

Pod는 수시로 죽고 IP가 바뀐다. **Service**는 그 위에 안정적인 이름과 가상 IP를 얹어, 클라이언트가
백엔드의 변화를 몰라도 되게 한다.

## Service 타입

`ServiceSpec`(`core/v1/types.go:5966`)의 `type` 필드로 정해진다:

| 타입 | 동작 |
|------|------|
| **ClusterIP** (기본) | 클러스터 내부 가상 IP. 외부에서 접근 불가 |
| **NodePort** | 모든 노드의 특정 포트를 열어 외부 노출(ClusterIP 포함) |
| **LoadBalancer** | 클라우드 LB를 프로비저닝해 외부 노출(NodePort 포함). LB 생성은 cloud-controller-manager |
| **ExternalName** | DNS CNAME만 반환(프록시 없음) → [03-dns.md](03-dns.md) |
| **Headless** (`clusterIP: None`) | 가상 IP 없이 DNS가 백엔드 Pod IP들을 직접 반환 |

## 셀렉터 → EndpointSlice

대부분의 Service는 **라벨 셀렉터**로 백엔드 Pod를 고른다. 그 매칭 결과(ready Pod들의 IP/포트)를
**EndpointSlice** 객체로 유지하는 것이 EndpointSlice 컨트롤러다
(`pkg/controller/endpointslice/`, [11-controller-manager/01](../11-controller-manager/01-controller-catalog.md)):

```
Service(셀렉터: app=web)
   │  EndpointSlice 컨트롤러가 셀렉터에 맞는 ready Pod 추적
   ▼
EndpointSlice(여러 조각): [10.1.1.5:8080, 10.1.2.7:8080, ...]
   │  kube-proxy/cilium 이 watch
   ▼
ClusterIP 로 온 트래픽 → 이 IP들로 분산
```

EndpointSlice를 여러 조각으로 나누는 이유(대규모 확장)는
[14-kube-proxy/01](../14-kube-proxy/01-endpointslice-tracking.md)에서 다뤘다.

## ready만 트래픽 받는다

readiness probe([13-kubelet/01](../13-kubelet/01-pod-lifecycle.md))를 통과한 Pod만 EndpointSlice에
ready로 들어간다. 그래서:

- 롤링 업데이트 중 새 Pod는 ready가 되기 전까지 트래픽을 안 받는다.
- 종료 중인 Pod는 EndpointSlice에서 빠져 신규 연결을 안 받는다.

이것이 무중단 배포의 네트워크 측 기반이다.

## Service를 넘어 — Ingress / Gateway API

Service는 L4(연결)다. L7(HTTP 경로/호스트 기반 라우팅, TLS 종료)은 상위 개념이 다룬다:

- **Ingress**: HTTP 라우팅 규칙(경로/호스트 → Service). Ingress 컨트롤러(별도 설치)가 구현.
- **Gateway API**: Ingress의 후계로, 더 표현력 있는 L7/L4 라우팅 모델. 역시 별도 컨트롤러가 구현.

이들은 클러스터 외부 트래픽을 내부 Service로 연결하는 진입점이다.

## 더 읽을 곳
- [14-kube-proxy](../14-kube-proxy/) — ClusterIP 데이터플레인
- [03-dns.md](03-dns.md) — 이름 해석
