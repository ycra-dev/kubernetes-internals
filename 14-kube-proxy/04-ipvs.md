# 14.04 · IPVS 모드 내부

**근거**: `kubernetes/pkg/proxy/ipvs/` (`proxier.go`, `ipset.go`, `netlink_linux.go`,
`graceful_termination.go`)

[02-backends.md](02-backends.md)에서 IPVS가 "대규모에서 iptables보다 효율적"이라 했다. 이 문서는 IPVS
모드의 **내부 구조** — 가상 서버, ipset, netlink — 를 본다. iptables 모드([03-iptables-rules.md](03-iptables-rules.md))와의
차이가 왜 확장성을 만드는지 보인다.

## IPVS = 커널 L4 로드밸런서

**IPVS(IP Virtual Server)** 는 리눅스 커널에 내장된 L4 로드밸런서다. iptables가 규칙 체인을 순회하는
것과 달리, IPVS는 **해시 테이블 기반 가상 서버**로 동작한다:

```
Virtual Server (ClusterIP:port)
  └── Real Server × N (백엔드 Pod IP:port)
       스케줄링 알고리즘으로 분산: rr(round-robin), lc(least-conn), dh, sh 등
```

각 Service가 하나의 Virtual Server, 각 ready 엔드포인트가 Real Server다. 조회가 해시 기반이라 서비스가
수천 개여도 **O(1)에 가깝게** 백엔드를 찾는다 — iptables의 선형 순회와 대조
([03-iptables-rules.md](03-iptables-rules.md)).

## netlink로 IPVS 제어

kube-proxy는 IPVS 규칙을 **netlink**(`netlink_linux.go`)로 커널에 설정한다. `syncProxyRules`
([02-backends.md](02-backends.md))가 Service/EndpointSlice를 보고:

- Service마다 Virtual Server를 만들고,
- ready 엔드포인트마다 Real Server를 추가/제거,
- 스케줄링 알고리즘(설정값)을 적용한다.

엔드포인트가 바뀌면 해당 Virtual Server의 Real Server 목록만 갱신 — 전체 규칙을 다시 쓰지 않아 갱신도
가볍다.

## ipset — iptables를 짧게 유지

IPVS 모드도 일부 iptables 규칙이 필요하다(SNAT 마킹, NodePort, 헬스체크 등). 그런데 서비스마다 iptables
규칙을 만들면 다시 선형으로 늘어난다. 그래서 **ipset**(`ipset.go`, `ipset/`)을 쓴다:

- ipset은 IP/포트의 **집합(set)** 을 커널 해시로 관리한다.
- iptables 규칙이 개별 IP가 아니라 "이 ipset에 속하면"으로 한 줄에 매칭한다.
- 그래서 서비스가 수천이어도 iptables 규칙은 **상수 개수**로 유지된다.

```
iptables: "목적지가 KUBE-CLUSTER-IP ipset에 속하면 ..." (한 줄)
   ↑ ipset에 모든 ClusterIP를 담아 → 규칙 수는 서비스 수와 무관
```

이 "IPVS(분산) + ipset(짧은 iptables)" 조합이 대규모에서 iptables 모드보다 빠른 이유다.

## graceful termination — 종료 중 연결 보존

`graceful_termination.go`. 백엔드 Pod가 종료될 때, 그 Real Server를 즉시 제거하면 진행 중 연결이 끊긴다.
IPVS 모드는 종료 중 백엔드의 **weight를 0으로** 만들어:

- 새 연결은 안 받지만,
- 기존 연결(established)은 계속 처리하다가,
- 연결이 다 빠지면 Real Server를 제거한다.

이로써 롤링 업데이트([80-scenarios/03](../80-scenarios/03-rolling-update.md)) 중 진행 중 요청이 끊기지
않는다. readiness 기반 제외([01-endpointslice-tracking.md](01-endpointslice-tracking.md))와 함께 무중단
배포를 돕는다.

## iptables vs IPVS 정리

| | iptables | IPVS |
|--|----------|------|
| 자료구조 | 규칙 체인(선형) | 해시 가상 서버 |
| 첫 패킷 매칭 | O(규칙 수) | O(1) 근접 |
| 규칙 갱신 | 전체 재작성 | 변경분만 |
| 대규모 | 느려짐 | 효율적 |
| 보조 iptables | (자체) | ipset로 상수화 |

더 나아간 대안은 cilium의 eBPF다([42-cilium/04](../42-cilium/04-kube-proxy-replacement.md),
[42-cilium/15](../42-cilium/15-packet-path.md)) — 커널 맵 + 소켓 LB로 한 단계 더.

## 더 읽을 곳
- [02-backends.md](02-backends.md) — 백엔드 비교
- [03-iptables-rules.md](03-iptables-rules.md) — iptables 모드(대조)
- [42-cilium/04](../42-cilium/04-kube-proxy-replacement.md) — eBPF 대안
