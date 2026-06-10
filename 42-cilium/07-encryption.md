# 42.07 · 전송 암호화 (WireGuard / IPsec)

**근거**: `cilium/pkg/wireguard/` (`agent/`, `types/`)

기본적으로 Pod 간 트래픽은 노드 네트워크 위로 **평문**으로 흐른다([01-ebpf-datapath.md](01-ebpf-datapath.md)).
신뢰할 수 없는 네트워크(여러 데이터센터, 공용 클라우드)에서는 노드 간 트래픽을 암호화해야 한다. cilium은
**투명한 전송 암호화**를 제공한다.

## WireGuard

`cilium/pkg/wireguard/`. 커널 WireGuard를 이용해 노드 간 Pod 트래픽을 암호화한다:

- 각 노드가 WireGuard 키 쌍을 갖고, cilium-agent가 노드들의 공개키를 교환·구성(`wireguard/agent/`).
- 노드를 떠나는 Pod 트래픽이 WireGuard 터널로 암호화돼 상대 노드로 가고, 거기서 복호화돼 목적지 Pod로.
- **투명하다** — 애플리케이션은 암호화를 의식하지 않는다(데이터패스가 자동 처리).

WireGuard는 현대적이고 설정이 단순해 cilium에서 권장되는 암호화 방식이다.

## IPsec

cilium은 IPsec 기반 암호화도 지원한다(대안). 기존 IPsec 인프라/규정 요구가 있는 환경에서 쓴다.
키 관리·갱신을 cilium이 조율한다.

## 무엇이 암호화되나

- **노드 간(node-to-node) Pod 트래픽**: 노드 경계를 넘는 트래픽이 대상. 같은 노드 안 Pod 간 통신은
  네트워크를 안 타므로 대상이 아니다.
- 오버레이/네이티브 라우팅([01-ebpf-datapath.md](01-ebpf-datapath.md)) 어느 모드든 적용 가능.

## NetworkPolicy와의 차이

혼동 주의:
- **암호화**(이 문서): 트래픽을 **도청으로부터 보호**(전송 보안).
- **NetworkPolicy**([03-networkpolicy.md](03-networkpolicy.md)): 어떤 통신을 **허용/차단**(접근 제어).

둘은 직교한다 — 정책으로 허용된 트래픽을 암호화해 보낼 수 있다. 함께 쓰면 "허용된 대상에게만, 도청 없이"를
달성한다.

## 더 읽을 곳
- [01-ebpf-datapath.md](01-ebpf-datapath.md) — 암호화가 얹히는 데이터패스
- [03-networkpolicy.md](03-networkpolicy.md) — 접근 제어(직교 개념)
