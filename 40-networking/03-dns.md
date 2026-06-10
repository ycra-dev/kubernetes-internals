# 40.03 · 클러스터 DNS

**근거 레포**: `dns` (`k8s.io/dns`) — `cmd/kube-dns`, `cmd/node-cache`, `cmd/sidecar`,
`cmd/dnsmasq-nanny`

Service에 안정적 IP가 있어도, 애플리케이션은 보통 IP가 아니라 **이름**으로 통신한다
(`my-svc.my-ns.svc.cluster.local`). 그 이름을 ClusterIP로 바꿔주는 것이 클러스터 DNS다.

## DNS 이름 규칙

- **Service**: `<service>.<namespace>.svc.cluster.local` → Service의 ClusterIP.
- **Headless Service**: 같은 이름이 백엔드 **Pod IP들**을 직접 반환(가상 IP 없음 →
  [02-service-discovery.md](02-service-discovery.md)).
- **Pod**: `<pod-ip-dashed>.<namespace>.pod.cluster.local` 형태.
- **ExternalName**: CNAME으로 외부 도메인 반환.

kubelet이 각 Pod의 `/etc/resolv.conf`에 클러스터 DNS 서버 주소와 search 도메인을 주입하므로, Pod 안에선
`my-svc`만 써도 `my-svc.<현재ns>.svc.cluster.local`로 확장돼 해석된다.

## DNS 서버 — CoreDNS와 kube-dns

DNS 서버는 클러스터 안에서 도는 일반 Pod(보통 Deployment + Service)다. 그것이 Service/EndpointSlice를
watch해 레코드를 생성한다.

- **CoreDNS**: 오늘날 **기본** 클러스터 DNS. 별도 프로젝트(이 모음의 `dns` 레포가 아님)이며, kubeadm이
  애드온으로 설치한다([15-cli/01](../15-cli/01-kubeadm.md)).
- **kube-dns**: 이 `dns` 레포(`cmd/kube-dns`)의 구버전 DNS. CoreDNS 이전의 기본이었다. 보조 컴포넌트:
  - `cmd/dnsmasq-nanny`(`dnsmasq` 캐시 관리), `cmd/sidecar`(헬스/메트릭).

둘 다 같은 역할(Service 이름 → IP)을 하지만, CoreDNS가 플러그인 구조로 더 유연해 표준이 됐다.

## NodeLocal DNSCache — node-cache

`dns` 레포의 `cmd/node-cache`는 **NodeLocal DNSCache**다. 각 노드에 DNS 캐시를 DaemonSet으로 띄워:

- Pod의 DNS 질의를 **로컬 노드에서** 먼저 처리(캐시 히트 시 클러스터 DNS까지 안 감).
- conntrack 부하와 DNS 지연/타임아웃을 줄인다(특히 대규모 클러스터의 UDP DNS 이슈 완화).

`pkg/netif/`(네트워크 인터페이스 설정) 등으로 노드 로컬에 DNS 리스너를 구성한다.

## DNS는 별도 컴포넌트일 뿐

핵심: 클러스터 DNS는 마법이 아니라 **Service/EndpointSlice를 watch하는 평범한 워크로드**다
([00-foundations/04](../00-foundations/04-list-watch-informer.md)). 따라서 DNS Pod가 죽으면 이름 해석이
멈추고, 그래서 보통 여러 복제본 + NodeLocal 캐시로 가용성을 높인다.

## 더 읽을 곳
- [02-service-discovery.md](02-service-discovery.md) — DNS가 해석하는 Service 모델
- [80-scenarios/02-service-request.md](../80-scenarios/) — DNS→Service→Pod 전체 경로
