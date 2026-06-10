# 42.15 · 패킷 경로 (bpf_lxc.c)

**근거**: `cilium/bpf/bpf_lxc.c` (`handle_ipv4_from_lxc`, tail call `CILIUM_CALL_IPV4_*` `:250`~`:289`)

[01-ebpf-datapath.md](01-ebpf-datapath.md)에서 "`bpf_lxc.c`가 Pod의 veth에 붙어 패킷을 처리한다"고 했다.
이 문서는 그 **패킷 한 개가 eBPF 프로그램 안에서 거치는 단계**를 본다. cilium 데이터패스의 가장 낮은
수준이다 — 앞 문서들의 맵/정책/identity가 패킷마다 어떻게 적용되는지 여기서 합쳐진다.

## from-container / to-container — 두 방향

Pod의 veth에는 두 방향의 처리가 붙는다:

- **from-container(egress)**: Pod가 **보내는** 패킷. Pod를 떠날 때 정책·서비스 LB·라우팅 결정.
- **to-container(ingress)**: Pod로 **들어오는** 패킷. 도착 정책 검사 후 Pod에 전달.

`bpf_lxc.c`의 `handle_ipv4_from_lxc`(egress 핵심)가 한 패킷을 처리하는 흐름을 본다.

## tail call로 단계를 잇는다

eBPF 프로그램은 스택/명령 수 제한이 있어, 하나의 거대한 함수로 못 짠다. 대신 **tail call**로 단계를
이어붙인다(`bpf_lxc.c:250` `tail_call_internal(ctx, CILIUM_CALL_IPV4_NO_SERVICE, ...)`,
`:289` `CILIUM_CALL_IPV4_CT_EGRESS`). 각 단계가 별도 eBPF 프로그램이고, tail call로 다음으로 점프한다:

```
from-container (egress) 패킷:
  [1] 서비스 LB 확인: 목적지가 ClusterIP인가?
        예 → 백엔드 선택(소켓 LB가 안 했으면) (04-kube-proxy-replacement)
        아니오 → CILIUM_CALL_IPV4_NO_SERVICE (:250)
  [2] CT(conntrack) 조회/생성: CILIUM_CALL_IPV4_CT_EGRESS (:289)
        새 연결인가 기존인가? (ctmap, 10-maps-conntrack)
  [3] 정책 확인: 출발 identity → 목적지 identity 허용? (policymap, 03-networkpolicy)
        새 연결만 정책 평가, 기존 연결은 CT 항목으로 빠르게 통과
  [4] 라우팅: 목적지가 로컬 Pod / 원격 노드 / 외부?
        ipcache로 목적지 위치 결정 (02-agent)
  [5] 전달: 로컬이면 직접, 원격이면 오버레이 캡슐화/네이티브 라우팅, 외부면 SNAT
```

각 `[n]`이 tail call로 연결된 별도 프로그램이다. 이로써 eBPF 제약 안에서 복잡한 처리를 조립한다.

## conntrack이 빠른 경로를 만든다

`[2]`의 CT 조회가 성능의 핵심이다([10-maps-conntrack.md](10-maps-conntrack.md)):

- **새 연결**: 전체 정책 평가(`[3]`) → 통과하면 ctmap에 항목 생성.
- **기존 연결**: ctmap에 이미 항목이 있으면 정책 재평가 없이 빠르게 통과(established).
- **응답(reply)**: 나간 연결의 응답은 CT로 "허용된 연결의 reply"임을 알아 통과.

그래서 연결당 한 번만 정책을 평가하고, 이후 패킷은 맵 조회로 끝난다 — iptables의 규칙 순회와 대조된다
([14-kube-proxy/03](../14-kube-proxy/03-iptables-rules.md)).

## 맵이 모이는 지점

이 한 함수가 앞 문서들의 맵을 모두 조회한다([10-maps-conntrack.md](10-maps-conntrack.md)):

| 단계 | 조회 맵 |
|------|---------|
| 서비스 LB | 서비스/백엔드 맵 |
| CT | ctmap |
| 정책 | policymap |
| 목적지 위치/identity | ipcache, lxcmap |
| 암호화 대상? | encrypt ([07-encryption](07-encryption.md)) |

agent가 채운 맵([02-agent.md](02-agent.md))을 패킷마다 읽어 **커널에서 O(1)에 가깝게** 결정한다 —
이것이 cilium이 "규모에 덜 민감"한 근본 이유다.

## 왜 tail call인가 (요약)

```
거대한 단일 프로그램(불가) → tail call 체인으로 단계 분할
  각 단계는 작고 검증 가능 → 커널 verifier 통과
  맵으로 단계 간 상태 전달 → 복잡한 L3/L4 처리 조립
```

## 더 읽을 곳
- [01-ebpf-datapath.md](01-ebpf-datapath.md) — bpf 프로그램과 훅 지점
- [10-maps-conntrack.md](10-maps-conntrack.md) — 조회되는 맵들
- [11-endpoint-regeneration.md](11-endpoint-regeneration.md) — 이 프로그램의 컴파일/로드
