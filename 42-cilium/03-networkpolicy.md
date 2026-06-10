# 42.03 · NetworkPolicy 시행

**근거**: `cilium/pkg/policy/`, `cilium/pkg/identity/`

[40-networking/01](../40-networking/01-cni.md)에서 봤듯, NetworkPolicy는 Kubernetes API로 **선언**되지만
**시행은 CNI**가 한다. cilium은 이를 eBPF로, 그리고 identity 기반으로 시행한다.

## identity 기반 정책

핵심은 [02](02-agent.md)의 **identity**다. 정책은 "어떤 라벨의 Pod가 어떤 라벨의 Pod와 통신 가능한가"로
표현되고, cilium은 이를 "identity A → identity B 허용" 형태로 컴파일한다.

```
NetworkPolicy(YAML):  "app=frontend 는 app=backend:8080 으로만"
   │  agent가 라벨 → identity 로 변환
   ▼
정책 맵(eBPF):  identity(frontend) → identity(backend):8080 = 허용
   │  bpf_lxc.c 가 패킷마다 조회
   ▼
허용/드롭 결정 (커널에서)
```

`pkg/policy/`가 선언된 정책을 계산해(어떤 identity 쌍이 어떤 포트로 허용되는지) eBPF 정책 맵으로
내려보낸다. 패킷이 올 때 데이터패스([01](01-ebpf-datapath.md))는 출발/도착 identity로 맵을 조회해
즉시 판정한다.

## 두 종류의 정책

- **Kubernetes NetworkPolicy**: 표준 API. L3/L4(IP/포트) 수준. namespace/라벨 셀렉터 기반.
- **CiliumNetworkPolicy / CiliumClusterwideNetworkPolicy**(cilium CRD): 확장 기능 —
  - **L7 정책**: HTTP 메서드/경로, gRPC, Kafka 등 애플리케이션 프로토콜 수준 제어(프록시 경유, `pkg/proxy/`).
  - **DNS 기반**: 도메인 이름으로 egress 허용(`*.example.com`).
  - **엔티티/CIDR**: 클러스터 외부 대상 정책.

## 기본 동작: 허용 → 격리

기본적으로 모든 Pod는 서로 통신 가능하다(평평한 모델). 어떤 Pod에 NetworkPolicy가 하나라도 적용되면,
그 Pod는 **격리 모드**가 되어 *명시적으로 허용된 트래픽만* 통과한다(allow-list). 이 전환은
EndpointSlice/identity 갱신처럼 eBPF 맵 갱신으로 즉시 반영된다.

## L7 시행 — 프록시 경유

L3/L4는 eBPF가 직접 처리하지만, L7(HTTP 등)은 패킷 페이로드 파싱이 필요해 eBPF가 트래픽을 노드 로컬
**Envoy 프록시**(`pkg/proxy/`)로 리다이렉트하고, 거기서 HTTP 규칙을 검사한 뒤 통과시킨다. L7 정책이
없는 트래픽은 프록시를 거치지 않아 빠른 경로를 유지한다.

## 더 읽을 곳
- [02-agent.md](02-agent.md) — identity 개념
- [05-hubble.md](05-hubble.md) — 정책으로 드롭된 흐름 관측
