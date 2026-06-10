# 42.10 · eBPF 맵 카탈로그와 conntrack

**근거**: `cilium/pkg/maps/` (`ctmap`, `lxcmap`, `ipcache`, `policymap`, `nat`, `egressmap`, `bwmap` 등)

[01-ebpf-datapath.md](01-ebpf-datapath.md)에서 "eBPF 프로그램은 상태를 맵에 두고, agent가 맵을 채운다"고
했다. 이 문서는 그 **맵들의 카탈로그**와, 그중 가장 중요한 **conntrack(연결 추적)** 을 본다. 제어 평면
(agent, 사용자공간)과 데이터 평면(eBPF, 커널)이 공유하는 자료구조가 곧 cilium의 "상태"다.

## 주요 맵 (`pkg/maps/`)

| 맵 | 디렉토리 | 담는 것 | 쓰는 곳 |
|----|----------|---------|---------|
| **lxcmap** | `lxcmap/` | 로컬 엔드포인트(Pod) IP → 엔드포인트 정보 | 패킷이 어느 로컬 Pod로 가나 ([02-agent](02-agent.md)) |
| **ipcache** | `ipcache/` | IP → identity + 위치(로컬/원격 노드) | identity 결정, 라우팅 ([02-agent](02-agent.md)) |
| **policymap** | `policymap/` | (identity, 포트) → 허용/거부 | NetworkPolicy 시행 ([03-networkpolicy](03-networkpolicy.md)) |
| **ctmap** | `ctmap/` | 연결 추적(conntrack) 항목 | 상태 추적 (아래) |
| **nat** | `nat/` | NAT 매핑 | 서비스 LB/SNAT ([04-kube-proxy-replacement](04-kube-proxy-replacement.md)) |
| **egressmap** | `egressmap/` | egress 정책 → 게이트웨이 IP | Egress Gateway ([08-bgp-egress](08-bgp-egress.md)) |
| **encrypt** | `encrypt/` | 암호화 키/상태 | WireGuard/IPsec ([07-encryption](07-encryption.md)) |
| **bwmap** | `bwmap/` | 대역폭 제한 | bandwidth manager(EDT) |
| **nodemap** | `nodemap/` | 노드 ID | 노드 간 라우팅 |
| **l2respondermap** | `l2respondermap/` | L2 광고 IP | L2 announcements ([08-bgp-egress](08-bgp-egress.md)) |

각 맵은 **agent가 Kubernetes 객체를 watch해 채우고**([02-agent](02-agent.md)), **eBPF 프로그램이
패킷마다 조회**한다([01-ebpf-datapath](01-ebpf-datapath.md)). 예: Pod가 생기면 agent가 lxcmap/ipcache에
그 IP→identity를 넣고, 정책이 바뀌면 policymap을 갱신한다.

## conntrack — 연결 상태 추적 (ctmap)

**ctmap**(`ctmap/`)은 cilium의 연결 추적 테이블이다. iptables가 커널 conntrack에 의존하는 것
([14-kube-proxy/02](../14-kube-proxy/02-backends.md))을 cilium은 자체 eBPF 맵으로 대체한다.

왜 필요한가:
- **상태 기반 정책**: "나가는 연결의 응답은 허용"(reply 트래픽). 새 연결만 정책 검사하고, 기존 연결의
  후속 패킷은 conntrack 항목으로 빠르게 허용 — 패킷마다 전체 정책 평가를 안 한다.
- **서비스 LB 일관성**: [04-kube-proxy-replacement.md](04-kube-proxy-replacement.md)에서 연결 시점에
  백엔드를 정하면, 그 연결의 모든 패킷이 같은 백엔드로 가야 한다. ctmap이 "이 연결은 이 백엔드"를 기억.
- **NAT 역변환**: 응답 패킷의 주소를 원래대로 되돌리려면 정방향 매핑을 알아야 한다(`nat/`와 연계).

conntrack 항목은 TTL이 있어 만료되고, GC가 오래된 항목을 정리한다(백엔드가 사라지면 관련 항목도 정리 —
[14-kube-proxy/02](../14-kube-proxy/02-backends.md)의 conntrack 정리와 같은 문제).

## 맵의 수명과 복원

eBPF 맵은 **커널에 pin**되어 agent 재시작과 무관하게 유지될 수 있다. 그래서 cilium-agent를 업그레이드해도
데이터패스(맵 + 프로그램)는 계속 동작하고, 새 agent가 맵을 이어받아 reconcile한다
([02-agent.md](02-agent.md)의 엔드포인트 복원과 같은 원리).

## 정리 — 맵이 곧 상태

```
제어 평면(agent): Kubernetes watch → 맵에 쓰기 (Pod/Service/Policy → lxcmap/ipcache/policymap...)
데이터 평면(eBPF): 패킷마다 맵 조회 → 라우팅/정책/LB/conntrack 판정 (커널, O(1))
```

iptables가 "규칙 체인"으로 상태를 표현했다면, cilium은 "해시맵"으로 표현한다 — 그래서 규모에 덜 민감하다
([01-ebpf-datapath.md](01-ebpf-datapath.md)).

## 더 읽을 곳
- [01-ebpf-datapath.md](01-ebpf-datapath.md) — 맵을 조회하는 eBPF 프로그램
- [02-agent.md](02-agent.md) — 맵을 채우는 제어 평면
- [14-kube-proxy/02](../14-kube-proxy/02-backends.md) — 비교: 커널 conntrack
