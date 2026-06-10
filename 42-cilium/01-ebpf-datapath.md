# 42.01 · eBPF 데이터패스

**근거**: `cilium/bpf/` (`bpf_lxc.c`, `bpf_host.c`, `bpf_overlay.c`, `bpf_sock.c`, `bpf_xdp.c`)

cilium의 실제 패킷 처리는 커널의 **eBPF 프로그램**이 한다. `bpf/`의 C 소스가 컴파일돼 커널의 여러
훅 지점(tc, XDP, socket)에 붙고, 에이전트가 채워 넣은 **eBPF 맵**을 참조해 라우팅/정책/로드밸런싱을
결정한다.

## eBPF 프로그램과 훅 지점

| 파일 | 붙는 곳 | 역할 |
|------|---------|------|
| `bpf_lxc.c` | Pod의 veth(엔드포인트) | Pod에서 나가고/들어오는 패킷에 정책/라우팅 적용 |
| `bpf_host.c` | 노드의 물리 인터페이스 | 노드 경계를 넘나드는 트래픽 처리 |
| `bpf_overlay.c` | 오버레이 터널(VXLAN/Geneve) | 노드 간 캡슐화 트래픽 |
| `bpf_sock.c` | 소켓(cgroup) | **소켓 수준 로드밸런싱**(연결 시점에 백엔드 선택) → [04](04-kube-proxy-replacement.md) |
| `bpf_xdp.c` | XDP(드라이버 최하단) | 가장 빠른 경로(DDoS 필터링, 일부 LB) |

각 프로그램은 패킷 메타데이터와 eBPF 맵을 보고 "통과/드롭/리다이렉트/캡슐화"를 결정한다. iptables처럼
규칙 체인을 선형 순회하지 않고, 맵 조회(O(1))로 판단한다.

## eBPF 맵 — 사용자공간과 커널의 공유 상태

eBPF 프로그램은 상태를 **맵**(`pkg/maps/`)에 둔다. cilium-agent가 Kubernetes 객체를 watch해 이 맵을
채우고, 커널 프로그램이 읽는다:

- **엔드포인트 맵**: Pod IP → 엔드포인트/identity.
- **정책 맵**: identity 쌍 → 허용/거부([03](03-networkpolicy.md)).
- **서비스/백엔드 맵**: ClusterIP:port → 백엔드 목록([04](04-kube-proxy-replacement.md)).
- **conntrack/NAT 맵**: 연결 추적.

즉 **제어 평면(agent, 사용자공간)이 맵을 갱신 → 데이터 평면(eBPF, 커널)이 맵을 보고 패킷 처리**라는
분리가 핵심이다.

## 노드 간 전달: 오버레이 vs 네이티브

- **오버레이**(`bpf_overlay.c`): Pod 트래픽을 VXLAN/Geneve로 캡슐화해 노드 간 전달. 인프라 라우팅에
  의존하지 않아 설치가 쉽다.
- **네이티브 라우팅**: 캡슐화 없이 노드 라우팅으로 직접 전달(오버헤드 적음, 인프라가 Pod CIDR을 라우팅해야 함).

둘 다 [40-networking](../40-networking/)의 "평평한 IP" 모델을 만족한다.

## 컴파일과 로드

cilium-agent는 노드/엔드포인트별 설정(`bpf/ep_config.h`, `node_config.h` 등 헤더)을 생성해 BPF를
컴파일하고, 해당 인터페이스에 로드한다. 엔드포인트가 생기거나 정책이 바뀌면 관련 프로그램/맵을 갱신한다
([02](02-agent.md)).

## 더 읽을 곳
- [02-agent.md](02-agent.md) — 맵을 채우는 제어 평면
- [04-kube-proxy-replacement.md](04-kube-proxy-replacement.md) — bpf_sock.c의 서비스 LB
