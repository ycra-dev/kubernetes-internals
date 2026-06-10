# 42.05 · Hubble — 관측성

**근거**: `cilium/hubble/`, `cilium/hubble-relay/`

Hubble은 cilium의 **관측성 계층**이다. eBPF 데이터패스([01](01-ebpf-datapath.md))가 패킷을 처리하면서
만들어내는 **흐름(flow) 이벤트**를 수집해, "누가 누구와 통신했고, 무엇이 정책으로 드롭됐는지"를 보여준다.

## eBPF가 곧 관측 지점

전통적 네트워크 관측은 별도 패킷 캡처/프록시가 필요했다. cilium은 이미 모든 패킷이 eBPF 프로그램을
지나므로, **그 지점에서 흐름 메타데이터를 그대로 방출**한다 — 추가 데이터패스 오버헤드가 작다.

흐름 이벤트에 담기는 것:
- 출발/도착 **identity**([02](02-agent.md))와 IP/포트.
- L7 정보(HTTP 메서드/경로, DNS 질의 등, L7 정책이 켜진 경우 [03](03-networkpolicy.md)).
- **판정**: forwarded / dropped(드롭 사유: 정책 거부 등).

## 구성

- **Hubble**(`hubble/`): 각 노드의 cilium-agent에 내장돼 로컬 흐름을 수집·노출.
- **Hubble Relay**(`hubble-relay/`): 여러 노드의 Hubble을 모아 **클러스터 전체** 흐름을 한 곳에서 보게
  한다. CLI(`hubble observe`)나 UI가 여기에 붙는다.

## 무엇에 쓰나

- **연결성 디버깅**: "왜 frontend가 backend에 못 붙나" → 드롭된 흐름과 정책 사유를 즉시 확인.
- **보안 가시성**: 어떤 identity가 어떤 외부 도메인과 통신하는지.
- **서비스 의존성 맵**: 실제 트래픽 기반으로 서비스 간 호출 관계를 시각화.

## 정책과의 연결

[03](03-networkpolicy.md)의 정책이 트래픽을 드롭하면, Hubble이 그 드롭을 사유와 함께 기록한다. 그래서
NetworkPolicy를 작성/디버깅할 때 Hubble로 "이 정책이 의도대로 막고 허용하는지"를 관찰하는 것이 표준
워크플로다.

## 더 읽을 곳
- [03-networkpolicy.md](03-networkpolicy.md) — 관측되는 정책 판정
- [01-ebpf-datapath.md](01-ebpf-datapath.md) — 흐름 이벤트의 출처
