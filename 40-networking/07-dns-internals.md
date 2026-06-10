# 40.07 · DNS 내부 (kube-dns / NodeLocal)

**근거 레포**: `dns` — `pkg/dns/dns.go`(`endpointsStore`/`servicesStore` `:77`~, `treecache/`),
`cmd/node-cache/`

[03-dns.md](03-dns.md)에서 클러스터 DNS를 개요로 봤다. 이 문서는 `dns` 레포의 **kube-dns 내부 동작**과
**NodeLocal DNSCache**를 코드 수준에서 본다. DNS 서버가 "Service를 watch해 레코드를 만드는 평범한
컨트롤러"임을 확인한다.

> 오늘날 기본은 CoreDNS(별도 프로젝트)지만, 동작 원리는 동일하다 — 이 레포의 kube-dns가 그 메커니즘을
> 가장 명확히 보여준다.

## DNS도 결국 watch 기반 컨트롤러

`dns/pkg/dns/dns.go`의 핵심은 **client-go 캐시**다(`dns.go:31` `k8s.io/client-go/tools/cache`):

```go
// dns.go:77
endpointsStore kcache.Store   // EndpointSlice/Endpoints watch 캐시
servicesStore  kcache.Store   // Service watch 캐시
```

즉 DNS 서버는 Service와 EndpointSlice를 **Informer로 watch**하고
([00-foundations/04](../00-foundations/04-list-watch-informer.md)), 그 변화에 맞춰 DNS 레코드를 갱신한다.
새 Service가 생기면 그 ClusterIP의 A 레코드가, Headless Service면 백엔드 Pod IP들이 레코드가 된다
([03-dns.md](03-dns.md)).

## treecache — 계층적 레코드 저장

DNS 이름은 계층적이다(`<svc>.<ns>.svc.cluster.local`). kube-dns는 이를 **treecache**(`dns/pkg/dns/treecache/`)
라는 트리 구조에 저장한다 — 도메인 레이블을 트리 노드로. 조회 시 이름을 레이블로 쪼개 트리를 내려가며
매칭한다. Service/Endpoint 변경 시 해당 트리 노드만 갱신해 효율적이다.

```
cluster.local
└── svc
    └── default
        ├── web    → ClusterIP 10.96.0.10
        └── db     → (headless) Pod IPs [10.244.1.5, 10.244.2.7]
```

## 질의 처리

Pod가 이름을 조회하면([03-dns.md](03-dns.md)의 resolv.conf):

1. **클러스터 도메인**(`*.cluster.local`)이면 treecache에서 응답.
2. **외부 도메인**이면 **upstream**(노드의 resolv.conf 또는 설정된 상위 DNS)으로 포워딩.
3. **stub domain / custom upstream**: 특정 도메인을 다른 DNS 서버로 보내는 설정(예: 사내 도메인).

## NodeLocal DNSCache (node-cache)

`dns/cmd/node-cache/`는 [03-dns.md](03-dns.md)에서 언급한 노드 로컬 캐시다. 각 노드에 DaemonSet으로
띄워:

- Pod의 DNS 질의를 **노드 로컬에서** 가로채 캐시 응답(히트 시 클러스터 DNS까지 안 감).
- 노드에 **로컬 DNS 리스너**를 구성한다(`pkg/netif/`로 인터페이스/iptables 설정).
- TCP로 클러스터 DNS에 연결(UDP conntrack 부하/타임아웃 회피).

왜 필요한가: 대규모 클러스터에서 모든 Pod의 UDP DNS 질의가 중앙 CoreDNS로 가면 **conntrack 테이블 고갈**과
간헐적 DNS 지연/실패가 생긴다([14-kube-proxy/02](../14-kube-proxy/02-backends.md)의 conntrack). NodeLocal
캐시가 대부분을 노드에서 처리해 이를 완화한다.

```
Pod DNS 질의 → 노드 로컬 캐시(node-cache)
   ├─ 캐시 히트 → 즉시 응답 (중앙 DNS 안 감)
   └─ 미스 → TCP로 중앙 CoreDNS → 캐시에 저장
```

## DNS 가용성

DNS는 모든 통신의 진입점이라 죽으면 클러스터가 마비된다. 그래서:
- CoreDNS를 **여러 복제본**으로(HPA로 스케일).
- **NodeLocal 캐시**로 중앙 의존을 줄임.
- DNS Pod가 readiness probe로 관리됨([13-kubelet/07](../13-kubelet/07-probes.md)).

## 더 읽을 곳
- [03-dns.md](03-dns.md) — DNS 개요
- [02-service-discovery.md](02-service-discovery.md) — DNS가 해석하는 Service
- [00-foundations/04](../00-foundations/04-list-watch-informer.md) — DNS 서버의 watch 기반
