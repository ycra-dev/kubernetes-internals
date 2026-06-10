# 80.02 · 시나리오 — 서비스 요청 경로

한 Pod의 애플리케이션이 `http://my-svc/`를 호출할 때, 그 패킷이 실제 백엔드 Pod에 닿기까지를 추적한다.
DNS 해석과 데이터플레인 라우팅 두 단계다.

## 전체 경로

```
[1] 앱: "my-svc" 에 연결 시도
[2] DNS 조회: my-svc.<ns>.svc.cluster.local → ClusterIP (10.96.0.10)
       └─ 클러스터 DNS(CoreDNS) 가 Service watch 로 만든 레코드
[3] 앱: ClusterIP:80 으로 connect
[4] 데이터플레인: ClusterIP → 백엔드 Pod IP 로 변환(DNAT/소켓 LB)
       └─ kube-proxy(iptables/IPVS) 또는 cilium(eBPF)
[5] 패킷이 백엔드 Pod 에 도달 (CNI 가 깐 네트워크 위로)
```

## 단계별 추적

### [2] DNS 해석
앱이 `my-svc`를 조회하면, kubelet이 주입한 `/etc/resolv.conf`의 search 도메인으로
`my-svc.<ns>.svc.cluster.local`로 확장돼 **클러스터 DNS**(CoreDNS)에 질의된다
([40-networking/03](../40-networking/03-dns.md)). CoreDNS는 Service를 watch해 만든 레코드에서 그 Service의
**ClusterIP**를 반환한다. (NodeLocal DNSCache가 있으면 노드 로컬 캐시에서 먼저 응답.)

> Headless Service면 ClusterIP 대신 백엔드 **Pod IP들**을 직접 반환하고, [4]를 건너뛴다
> ([40-networking/02](../40-networking/02-service-discovery.md)).

### [3]~[4] ClusterIP → 백엔드 변환
ClusterIP는 **가상 IP**다 — 어떤 인터페이스에도 없다([40-networking](../40-networking/)). 패킷이
ClusterIP로 향하면 노드의 데이터플레인이 가로채 살아 있는 백엔드 중 하나로 바꾼다:

- **kube-proxy**: 미리 깔아둔 iptables/IPVS 규칙으로 DNAT
  ([14-kube-proxy/02](../14-kube-proxy/02-backends.md)). 백엔드 목록은 EndpointSlice에서 온다
  ([14-kube-proxy/01](../14-kube-proxy/01-endpointslice-tracking.md)).
- **cilium(eBPF)**: `connect()` 시점에 소켓 훅(`bpf_sock.c`)이 목적지를 백엔드 Pod IP로 교체
  ([42-cilium/04](../42-cilium/04-kube-proxy-replacement.md)). per-packet 변환보다 가볍다.

백엔드 선택은 **ready 엔드포인트**들 중에서다 — readiness probe를 통과한 Pod만 후보
([13-kubelet/01](../13-kubelet/01-pod-lifecycle.md), [40-networking/02](../40-networking/02-service-discovery.md)).

### [5] 백엔드 도달
바뀐 목적지(Pod IP)로 패킷이 전달된다. 이 Pod-to-Pod 도달성은 CNI가 구축한 "평평한 IP" 네트워크가
보장한다([40-networking/01](../40-networking/01-cni.md), [42-cilium/01](../42-cilium/01-ebpf-datapath.md)).
다른 노드면 오버레이/네이티브 라우팅으로 노드 경계를 넘는다.

## NetworkPolicy가 끼어들면

출발 Pod와 도착 Pod 사이에 NetworkPolicy가 있으면, 데이터플레인이 **identity 기반**으로 허용 여부를
판정한다([42-cilium/03](../42-cilium/03-networkpolicy.md)). 거부되면 패킷이 드롭되고, Hubble로 그 드롭을
관측할 수 있다([42-cilium/05](../42-cilium/05-hubble.md)).

## 외부에서 들어오는 경우

클러스터 밖에서 들어오면 진입점이 다르다:
- **NodePort/LoadBalancer**: 노드 포트/클라우드 LB → 같은 데이터플레인으로
  ([14-kube-proxy/02](../14-kube-proxy/02-backends.md)).
- **Ingress/Gateway**: L7 라우팅으로 내부 Service에 연결([40-networking/02](../40-networking/02-service-discovery.md)).

## 핵심 관찰

- DNS와 데이터플레인은 **둘 다 Service/EndpointSlice를 watch**하는 컴포넌트가 만든 결과다. Service
  하나가 바뀌면 CoreDNS의 레코드와 proxy/cilium의 규칙이 각자 갱신된다.
- ClusterIP는 추상화일 뿐, 실제 일은 노드 로컬 데이터플레인이 한다.

## 더 읽을 곳
- [01-pod-create.md](01-pod-create.md) — 백엔드 Pod가 생기는 과정
- [03-rolling-update.md](03-rolling-update.md) — 백엔드가 교체되는 중의 트래픽
