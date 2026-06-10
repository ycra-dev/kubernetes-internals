# 40.04 · Ingress와 Gateway API

**근거**: `kubernetes/staging/src/k8s.io/api/networking/v1/types.go`(IngressSpec), Gateway API(별도 프로젝트)

Service는 L4(연결) 추상화다([02-service-discovery.md](02-service-discovery.md)). 그런데 "호스트/경로 기반
HTTP 라우팅", "TLS 종료", "하나의 외부 IP로 여러 서비스 노출" 같은 **L7** 요구는 Service만으론 부족하다.
**Ingress**와 그 후계 **Gateway API**가 이를 담당한다.

## Ingress — HTTP 라우팅 규칙

**Ingress**(`networking/v1/types.go`의 IngressSpec)는 "어떤 호스트/경로의 HTTP 요청을 어떤 Service로
보낼지"를 선언하는 객체다:

```
ingress 규칙 예:
  host: shop.example.com
    /api    → api-service:8080
    /static → static-service:80
  TLS: shop.example.com → secretName: shop-tls
```

핵심: **Ingress 객체 자체는 아무것도 하지 않는다.** 그것을 watch해 실제 L7 프록시(nginx, Envoy,
HAProxy 등)를 설정하는 **Ingress 컨트롤러**가 별도로 설치돼야 한다. Ingress 컨트롤러도 결국
reconcile 루프다([00-foundations/05](../00-foundations/05-controller-pattern.md)) — Ingress/Service/
EndpointSlice를 watch해 프록시 설정을 갱신한다.

```
외부 HTTP ─► Ingress 컨트롤러(프록시) ─► 규칙 매칭 ─► Service ─► Pod
            (Ingress 객체를 watch해 설정됨)        (14-kube-proxy/42-cilium)
```

## Ingress의 한계

Ingress는 HTTP/HTTPS에 치우쳐 있고, 고급 기능(트래픽 분할, 헤더 기반 라우팅, gRPC/TCP/UDP)을 표준으로
표현하기 어렵다. 그래서 벤더들이 어노테이션으로 확장했고, 이식성이 깨졌다.

## Gateway API — Ingress의 후계

**Gateway API**(별도 프로젝트, CRD로 제공)는 이를 역할 기반으로 재설계했다:

| 리소스 | 소유자 | 역할 |
|--------|--------|------|
| **GatewayClass** | 인프라 제공자 | 구현체 종류(어떤 컨트롤러) |
| **Gateway** | 클러스터 운영자 | 리스너(포트/프로토콜/TLS), 진입점 |
| **HTTPRoute / TCPRoute / GRPCRoute** | 앱 개발자 | 라우팅 규칙 |

- **역할 분리**: 인프라/운영/개발 책임이 다른 리소스로 나뉜다.
- **표현력**: 트래픽 분할(카나리), 헤더 매칭, 다양한 프로토콜을 표준으로.
- **이식성**: 어노테이션 의존을 줄여 구현체 간 호환.

Gateway API도 구현은 컨트롤러(cilium, Istio, Envoy Gateway 등)가 한다 — cilium은 Gateway API를
자체 데이터패스로 구현할 수 있다([42-cilium/09](../42-cilium/09-l7-envoy.md)).

## Service / Ingress / Gateway 정리

```
L4(연결)            Service        ── ClusterIP/NodePort/LoadBalancer
L7(HTTP) 단순       Ingress        ── 호스트/경로 라우팅
L7 풍부/다중 역할    Gateway API    ── Gateway + HTTPRoute 등
```

## 더 읽을 곳
- [02-service-discovery.md](02-service-discovery.md) — 하위 L4 Service
- [42-cilium/09](../42-cilium/09-l7-envoy.md) — L7 구현(Envoy)
