# 42.09 · L7 처리와 Envoy

**근거**: `cilium/pkg/envoy/` (`accesslog.go`, `manager.go`)

eBPF는 L3/L4(IP/포트)를 커널에서 빠르게 처리하지만([01-ebpf-datapath.md](01-ebpf-datapath.md)), **L7**
(HTTP 메서드/경로, gRPC, Kafka)은 패킷 페이로드를 파싱해야 한다. cilium은 이를 위해 노드 로컬 **Envoy
프록시**를 활용한다.

## 왜 Envoy인가

L7 정책(예: "GET /api/public만 허용, POST는 금지")이나 L7 관측은 애플리케이션 프로토콜 파싱이 필요하다.
이것은 eBPF만으로는 무겁다. 그래서 cilium은:

1. L7 정책이 적용되는 트래픽만 **eBPF가 노드 로컬 Envoy로 리다이렉트**한다.
2. **Envoy**(`cilium/pkg/envoy/`)가 HTTP/gRPC/Kafka를 파싱해 규칙을 적용.
3. 통과한 트래픽은 목적지로, 거부는 차단.

L7 정책이 **없는** 트래픽은 Envoy를 거치지 않고 eBPF 빠른 경로를 유지한다 — 필요한 곳에만 L7 비용을 낸다.

## L7 NetworkPolicy

[03-networkpolicy.md](03-networkpolicy.md)의 CiliumNetworkPolicy는 L7 규칙을 표현한다:

```
"app=frontend 는 app=backend 의 GET /api/* 만 호출 가능"
   │  cilium-agent가 Envoy에 L7 규칙 구성
   ▼
eBPF: frontend→backend 트래픽을 Envoy로 리다이렉트
Envoy: HTTP 파싱 → GET /api/* 면 통과, 아니면 403
```

identity 기반([02-agent.md](02-agent.md))으로 "누가" 보냈는지 판단하고, Envoy가 "무엇을" 요청했는지
검사한다.

## L7 관측 — access log

`cilium/pkg/envoy/accesslog.go`. Envoy가 처리한 L7 요청(메서드/경로/응답코드)을 access log로 남기고,
이것이 **Hubble**([05-hubble.md](05-hubble.md))의 L7 흐름 데이터가 된다. 그래서 "어떤 HTTP 경로가 얼마나
호출되고 무엇이 거부됐는지"를 관측할 수 있다.

## Gateway API / Ingress 구현

cilium은 이 Envoy 통합으로 **Ingress와 Gateway API**도 구현할 수 있다
([40-networking/04](../40-networking/04-ingress-gateway.md)) — 외부 L7 트래픽을 Envoy로 라우팅/TLS
종료해 내부 Service로 보낸다. 별도 Ingress 컨트롤러 없이 cilium이 L7 진입점 역할을 한다.

## 더 읽을 곳
- [03-networkpolicy.md](03-networkpolicy.md) — L7 정책
- [05-hubble.md](05-hubble.md) — L7 관측
- [40-networking/04](../40-networking/04-ingress-gateway.md) — Ingress/Gateway
