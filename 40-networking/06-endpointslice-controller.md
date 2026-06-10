# 40.06 · EndpointSlice 컨트롤러

**근거**: `kubernetes/pkg/controller/endpointslice/`
(`endpointslice_controller.go` `maxEndpointsPerSlice` `:89`, `reconciler.go`)

[02-service-discovery.md](02-service-discovery.md)에서 EndpointSlice가 "Service 셀렉터에 맞는 ready
Pod들"을 담는다고 했다. 이 문서는 그것을 **유지하는 컨트롤러**가 어떻게 동작하는지 — 특히 슬라이스
**분할/배치(placement)** 알고리즘 — 를 본다. kube-proxy/cilium이 소비하는 데이터의 생산자다.

## 무엇을 reconcile하나

EndpointSlice 컨트롤러는 셀렉터 있는 **Service마다** "현재 EndpointSlice들"이 "있어야 할 엔드포인트
집합"과 일치하는지 맞춘다([00-foundations/05](../00-foundations/05-controller-pattern.md)):

```
입력: Service(셀렉터) + 매칭되는 Pod들 + Node 정보
   │  컨트롤러가 watch
   ▼
원하는 상태: ready Pod들의 (IP, 포트, 토폴로지, 상태)를 슬라이스들로 분할
   │  현재 EndpointSlice들과 비교
   ▼
diff만큼 슬라이스 생성/수정/삭제
```

Service/Pod/Node 변경이 오면 해당 Service를 재동기화한다(`endpointslice_controller.go`의 syncService).

## 왜 분할하나 — maxEndpointsPerSlice

핵심 파라미터는 **`maxEndpointsPerSlice`**(`endpointslice_controller.go:89`, `:253`, 기본 100). 한
슬라이스에 최대 이만큼만 담는다. 이유는 [02-service-discovery.md](02-service-discovery.md)에서 본 대로 —
거대한 단일 객체(레거시 Endpoints)는 Pod 하나만 바뀌어도 전체가 갱신돼 watch 트래픽이 폭증하기 때문.

```
백엔드 250개, maxEndpointsPerSlice=100:
  → EndpointSlice 3개 (100 + 100 + 50)
  → Pod 하나 바뀌면 그것이 든 슬라이스 1개만 갱신 (전체 250 아님)
```

## placement — 어느 슬라이스에 넣나

`reconciler.go`의 배치 알고리즘이 까다로운 부분이다. 단순히 채우는 게 아니라 **변경을 최소화**하려 한다:

1. **기존 슬라이스 재사용**: 새 엔드포인트는 가능하면 빈자리 있는 기존 슬라이스에 넣는다(새 슬라이스
   생성을 줄임).
2. **떠난 엔드포인트 제거**: 사라진 Pod를 그 슬라이스에서 뺀다.
3. **압축(compaction)**: 슬라이스들이 너무 비면 합쳐 슬라이스 수를 줄인다(과도한 조각화 방지).

목표: **건드리는 슬라이스 수를 최소화**해 watch 갱신을 줄인다. 한 Pod 변경이 슬라이스 하나의 갱신으로
끝나는 것이 이상적.

## 담기는 정보

각 엔드포인트에 들어가는 것([14-kube-proxy/01](../14-kube-proxy/01-endpointslice-tracking.md)):

- **주소**(Pod IP, 듀얼스택이면 패밀리별, [05-dual-stack.md](05-dual-stack.md)).
- **conditions**: ready / serving / terminating — readiness probe 결과
  ([13-kubelet/07](../13-kubelet/07-probes.md))가 ready를 정한다.
- **topology**: 노드 이름, 존 — 토폴로지 인지 라우팅용
  ([14-kube-proxy/01](../14-kube-proxy/01-endpointslice-tracking.md)).
- **targetRef**: 어느 Pod인지.

## 소비자

생산된 EndpointSlice는 여럿이 소비한다:
- **kube-proxy**([14-kube-proxy](../14-kube-proxy/))/**cilium**([42-cilium/04](../42-cilium/04-kube-proxy-replacement.md)):
  데이터플레인 규칙.
- **클러스터 DNS**: Headless Service의 Pod IP 레코드([03-dns.md](03-dns.md)).
- **Ingress/Gateway 컨트롤러**([04-ingress-gateway.md](04-ingress-gateway.md)).

즉 EndpointSlice 컨트롤러는 "Service↔Pod 매핑"의 **단일 생산자**이고, 네트워킹 컴포넌트 다수가 이를 watch
한다.

## 더 읽을 곳
- [02-service-discovery.md](02-service-discovery.md) — Service/EndpointSlice 모델
- [14-kube-proxy/01](../14-kube-proxy/01-endpointslice-tracking.md) — 소비자(kube-proxy)
- [13-kubelet/07](../13-kubelet/07-probes.md) — ready 상태의 출처
