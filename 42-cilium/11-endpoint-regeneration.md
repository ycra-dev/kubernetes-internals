# 42.11 · 엔드포인트 재생성 (BPF 빌드)

**근거**: `cilium/pkg/endpoint/bpf.go`(`regenerateBPF` `:360`, `realizeBPFState` `:568`)

[02-agent.md](02-agent.md)에서 "정책/엔드포인트가 바뀌면 agent가 eBPF를 갱신한다"고 했다. 이 문서는 그
**재생성(regeneration)** 과정 — 헤더 생성 → BPF 컴파일 → 맵 갱신 → 로드 — 을 코드로 본다. cilium의
"제어 평면이 데이터 평면을 reconcile하는" 핵심 루프다.

## 재생성이 트리거되는 때

엔드포인트(Pod)의 데이터패스를 다시 만들어야 하는 상황:

- 새 Pod 생성(CNI ADD, [40-networking/01](../40-networking/01-cni.md)).
- 그 Pod에 영향을 주는 **NetworkPolicy 변경**([03-networkpolicy.md](03-networkpolicy.md)).
- **identity 변경**(라벨이 바뀜, [02-agent.md](02-agent.md)).
- 설정/CIDR 변경.

이들은 엔드포인트의 **regeneration 이벤트**로 큐잉돼 직렬 처리된다(한 엔드포인트의 재생성이 겹치지 않게).

## regenerateBPF — 재생성의 단계

`regenerateBPF`(`bpf.go:360`)가 핵심이다. 주석(`:351`)대로 *"모든 헤더를 다시 쓰고 모든 BPF 맵을
갱신"* 한다:

```
regenerateBPF (bpf.go:360):
  [1] 정책 계산: 이 엔드포인트의 identity가 누구와 통신 가능한지 → policymap 항목
  [2] 헤더 생성: 엔드포인트별 C 헤더(ep_config.h 등)에 ID/IP/옵션 기록
  [3] BPF 맵 갱신: policymap, ipcache 등 (42-cilium/10)
  [4] realizeBPFState (:568): 실제 컴파일 + 로드
```

### realizeBPFState — 컴파일과 로드
`realizeBPFState`(`bpf.go:568`)가 무거운 부분이다:

- 엔드포인트별 헤더로 **`bpf_lxc.c`를 컴파일**([01-ebpf-datapath.md](01-ebpf-datapath.md))해 그
  엔드포인트 전용 eBPF 오브젝트를 만든다.
- 그것을 Pod의 veth 인터페이스에 **로드**(tc/네트워크 훅).
- 성공하면 엔드포인트의 **revision 번호**를 올린다(`regenerateBPF`가 revnum 반환, `:360`) — "이 엔드포인트는
  이 버전의 정책/데이터패스를 반영했다"는 표식.

## 왜 엔드포인트마다 컴파일하나

cilium은 정책/설정을 **컴파일 시점에 헤더로 박아** 최적화된 eBPF를 만든다(런타임 분기 최소화). 그래서
엔드포인트마다, 변경마다 재컴파일이 필요하다. 비용이 있지만:

- 재생성은 **비동기·증분**이다 — 바뀐 엔드포인트만 재생성, 나머지는 그대로.
- 정책 맵 갱신만으로 되는 변경은 전체 재컴파일 없이 **맵만** 갱신(빠른 경로).

> 근래 cilium은 정책을 맵으로 더 많이 옮겨, 정책 변경 시 재컴파일 없이 policymap 갱신만으로 처리하는
> 방향으로 발전했다(재생성 비용 감소).

## 재생성 = reconcile

이 과정은 결국 reconcile 루프다([00-foundations/05](../00-foundations/05-controller-pattern.md))의 cilium
데이터패스 버전:

```
원하는 상태(Kubernetes 객체: Pod/Policy/identity)
   │  agent가 watch (02-agent)
   ▼
regenerateBPF: 차이를 BPF 헤더/맵/프로그램으로 realize
   │
실제 상태(커널의 eBPF 데이터패스)가 원하는 상태로 수렴
```

revision 번호로 "어디까지 반영됐나"를 추적해, 실패 시 재시도하고 성공 시 확정한다.

## 더 읽을 곳
- [02-agent.md](02-agent.md) — 재생성을 트리거하는 agent
- [01-ebpf-datapath.md](01-ebpf-datapath.md) — 컴파일되는 BPF 프로그램
- [10-maps-conntrack.md](10-maps-conntrack.md) — 갱신되는 맵들
