# 14.03 · iptables 규칙 구조 (코드 레벨)

**근거**: `kubernetes/pkg/proxy/iptables/proxier.go`
(`kubeServicesChain` `:56`, `kubeNodePortsChain` `:62`, 체인 연결 `:373`~)

[02-backends.md](02-backends.md)에서 iptables 백엔드가 "DNAT 규칙으로 ClusterIP를 백엔드로 바꾼다"고
했다. 이 문서는 그 **실제 체인 구조**를 본다. kube-proxy가 만드는 iptables 체인은 디버깅 시 직접 보게
되므로 구조를 알아두면 유용하다.

## 커스텀 체인 — KUBE-*

kube-proxy는 기본 iptables 체인을 직접 건드리지 않고 **자기 체인을 만들어 거기에 점프**시킨다
(`proxier.go`의 상수들):

| 체인 | 상수 | 역할 |
|------|------|------|
| `KUBE-SERVICES` | `kubeServicesChain` (`:56`) | 모든 서비스 트래픽의 진입점 |
| `KUBE-NODEPORTS` | `kubeNodePortsChain` (`:62`) | NodePort 처리 |
| `KUBE-SVC-<hash>` | (서비스별 생성) | 한 서비스 → 백엔드들로 분산 |
| `KUBE-SEP-<hash>` | (엔드포인트별 생성) | 한 백엔드(Service EndPoint)로 DNAT |
| `KUBE-MARK-MASQ` | | SNAT(masquerade) 표시 |

기본 체인 연결(`proxier.go:373`~): `PREROUTING`/`OUTPUT`의 NAT 테이블이 `KUBE-SERVICES`로 점프하고,
`FORWARD`/`OUTPUT`의 filter 테이블도 연결된다(`:374`는 conntrack `--ctstate NEW`로 새 연결만).

## 패킷이 흐르는 경로

ClusterIP `10.96.0.10:80`(백엔드 3개)으로 가는 새 연결:

```
패킷 → KUBE-SERVICES
   │  "목적지 10.96.0.10:80 이면" → KUBE-SVC-XXXX 로 점프
   ▼
KUBE-SVC-XXXX  (서비스별 체인)
   │  확률적 분산: 1/3 확률로 KUBE-SEP-A, 아니면 1/2로 KUBE-SEP-B, 아니면 KUBE-SEP-C
   ▼
KUBE-SEP-B  (엔드포인트별 체인)
   │  DNAT --to-destination 10.244.2.7:8080  (백엔드 Pod IP)
   ▼
패킷이 백엔드로
```

- **KUBE-SVC-** 체인이 백엔드 분산을 한다 — iptables `statistic` 모듈의 확률로 균등 분배
  ([02-backends.md](02-backends.md)).
- **KUBE-SEP-** 체인이 실제 DNAT(목적지를 Pod IP로 변경)을 한다.
- 응답 패킷은 커널 conntrack이 자동 역변환(DNAT 되돌림).

## 왜 선형 확장이 문제인가

서비스 S개, 평균 엔드포인트 E개면 KUBE-SVC 체인 S개 + KUBE-SEP 체인 S×E개가 생긴다. `KUBE-SERVICES`는
들어온 패킷을 맞는 KUBE-SVC로 보내기 위해 규칙을 **순회**하므로, 서비스가 수천이면:

- 첫 패킷 매칭 지연이 커진다(규칙 선형 순회).
- `syncProxyRules`([02-backends.md](02-backends.md))가 전체 규칙을 다시 쓰는 시간이 길어진다.

이것이 IPVS/nftables/cilium-eBPF로 넘어가는 동기다([02-backends.md](02-backends.md),
[42-cilium/04](../42-cilium/04-kube-proxy-replacement.md)) — 이들은 해시/맵 기반이라 O(1)에 가깝다.

## SNAT — KUBE-MARK-MASQ

특정 경우(예: 노드 밖에서 온 트래픽, hairpin) 소스 IP를 노드 IP로 바꿔야(SNAT) 응답이 돌아온다.
kube-proxy는 그런 패킷에 `KUBE-MARK-MASQ`로 마크를 찍고, POSTROUTING에서 masquerade 한다.
`externalTrafficPolicy: Local`은 이 SNAT를 피해 클라이언트 소스 IP를 보존한다
([02-backends.md](02-backends.md)).

## restore 방식 — 원자적 적용

kube-proxy는 규칙을 한 줄씩 추가하지 않고 `iptables-restore`로 **전체 규칙 세트를 원자적으로** 적용한다.
`syncProxyRules`가 "지금 있어야 할 모든 규칙"을 텍스트로 생성해 한 번에 restore — level-triggered
([02-backends.md](02-backends.md))이고 중간 상태가 없다.

## 더 읽을 곳
- [02-backends.md](02-backends.md) — iptables/IPVS/nftables 비교
- [42-cilium/10](../42-cilium/10-maps-conntrack.md) — eBPF 맵 대안
