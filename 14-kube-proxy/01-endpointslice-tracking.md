# 14.01 · Service/EndpointSlice 변경 추적

**근거**: `kubernetes/pkg/proxy/servicechangetracker.go`, `endpointschangetracker.go`,
`endpointslicecache.go`

kube-proxy는 두 종류의 객체를 watch한다: **Service**(가상 IP와 포트)와 **EndpointSlice**(실제 백엔드
Pod IP 목록). 이 둘의 변화를 추적해 데이터플레인 규칙을 갱신한다.

## 왜 EndpointSlice인가 (Endpoints 아니고)

초기엔 한 Service의 모든 백엔드가 하나의 **Endpoints** 객체에 담겼다. 백엔드가 수천 개면 그 객체가
거대해지고, Pod 하나만 바뀌어도 전체 객체가 갱신돼 watch 트래픽이 폭증했다.

**EndpointSlice**는 이를 **여러 조각(slice)으로 분할**한다(슬라이스당 기본 ~100개 엔드포인트). Pod 하나가
바뀌면 그 Pod가 든 슬라이스만 갱신되므로 변경 전파가 가볍다. EndpointSlice 컨트롤러
([11-controller-manager/01](../11-controller-manager/01-controller-catalog.md))가 이를 유지한다.

## 변경 추적기

kube-proxy 내부에 두 트래커가 있다:

- **ServiceChangeTracker** (`servicechangetracker.go:33`): Service 추가/수정/삭제를 누적.
- **EndpointsChangeTracker** (`endpointschangetracker.go:33`): EndpointSlice 변화를 누적. 여러
  슬라이스를 합쳐 "이 Service의 전체 엔드포인트 집합"을 재구성하는 캐시가 `endpointslicecache.go`다.

이 트래커들은 watch 이벤트를 받아 **"무엇이 바뀌었는지"를 모아 두고**, 다음 sync 때 한꺼번에 반영한다.

## ready/serving/terminating

엔드포인트는 단순히 "있다/없다"가 아니다. 각 엔드포인트는 상태를 가진다:

- **ready**: 트래픽을 받을 수 있음(readiness probe 통과 → [13-kubelet/01](../13-kubelet/01-pod-lifecycle.md)).
- **serving** / **terminating**: 종료 중이지만 기존 연결은 처리하는 상태 등.

kube-proxy는 보통 **ready 엔드포인트**로만 라우팅한다. 그래서 readiness probe가 실패한 Pod는 자동으로
서비스 트래픽에서 빠진다 — 무중단 배포의 핵심.

## 토폴로지 인지 라우팅

EndpointSlice는 엔드포인트의 **존(zone)/노드** 정보를 담는다. 이를 이용해 "같은 존의 백엔드를 우선"하는
토폴로지 인지 라우팅(traffic distribution)을 할 수 있어, 존 간 트래픽 비용을 줄인다.

## 더 읽을 곳
- [02-backends.md](02-backends.md) — 추적된 변경이 커널 규칙이 되는 과정
- [40-networking/02-service-discovery.md](../40-networking/) — Service 타입 전체
