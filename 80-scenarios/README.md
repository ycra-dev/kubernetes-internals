# 80 · 엔드투엔드 시나리오

앞 파트들이 컴포넌트를 **하나씩** 설명했다면, 이 폴더는 그것들을 **한 사건**으로 꿰어 본다. 흩어진
메커니즘(API 파이프라인, 스케줄링, kubelet, 런타임, 네트워킹)이 실제로 어떻게 협력하는지를 시간순으로
추적한다. "구조·동작 설명"의 결론에 해당한다.

## 공통 주제 — 누구도 서로를 직접 호출하지 않는다

모든 시나리오에서 반복되는 핵심:

- 컴포넌트들은 **etcd에 저장된 객체 상태를 매개로** 협력한다(직접 호출 X) —
  [00-foundations/02](../00-foundations/02-architecture.md)의 공유 상태 모델.
- 각 단계는 **level-triggered** — 어디서 끊겨도 재시작 후 현재 상태를 보고 이어간다.
- 한 필드(예: `nodeName`)나 한 객체(예: EndpointSlice)가 컴포넌트 간 **바통**이 된다.

## 문서

| # | 시나리오 | 꿰는 메커니즘 |
|---|----------|---------------|
| 01 | [pod-create.md](01-pod-create.md) | `kubectl run` → apiserver → etcd → scheduler → kubelet → containerd → cilium |
| 02 | [service-request.md](02-service-request.md) | DNS → Service(ClusterIP) → 데이터플레인 → 백엔드 Pod |
| 03 | [rolling-update.md](03-rolling-update.md) | Deployment → ReplicaSet 전환 → 무중단 트래픽 이동 |
| 04 | [scale-and-failure.md](04-scale-and-failure.md) | HPA/CA 스케일링 · 노드 장애 → eviction → 재스케줄 |

## 읽는 법

각 시나리오는 **정상 흐름**을 보여준다. 그래서 [99-appendix/troubleshooting.md](../99-appendix/troubleshooting.md)의
"증상 → 코드" 역추적과 짝을 이룬다 — 정상 경로를 알면 어디서 막혔는지 좁힐 수 있다.

## 더 읽을 곳
- [00-foundations/02](../00-foundations/02-architecture.md) — 요청/데이터 흐름 큰 그림
- [99-appendix/troubleshooting.md](../99-appendix/troubleshooting.md) — 고장 시 역추적
