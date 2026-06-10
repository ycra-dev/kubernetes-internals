# 40.08 · Service IP/포트 할당

**근거**: `kubernetes/pkg/registry/core/service/ipallocator/`(`bitmap.go`, `cidrallocator.go`, `ipallocator.go`),
`kubernetes/pkg/registry/core/service/portallocator/`, `kubernetes/pkg/controller/servicecidrs/`,
`networking/v1` ServiceCIDR/IPAddress

[02-service-discovery.md](02-service-discovery.md)에서 "ClusterIP는 Service CIDR 풀에서 온다"고 했다. 이
문서는 그 **할당 메커니즘** — apiserver가 Service 생성 시 어떻게 유일한 ClusterIP/NodePort를 배정하는지,
그리고 새 ServiceCIDR API — 를 본다. ClusterIP는 가상이지만 클러스터 전체에서 **유일**해야 하므로
중앙 할당이 필요하다.

## ClusterIP는 apiserver가 할당

ClusterIP는 가상 IP라([02-service-discovery.md](02-service-discovery.md)) 실제 인터페이스에 없지만,
**중복되면 안 된다**(두 Service가 같은 IP면 라우팅이 깨짐). 그래서 **apiserver의 registry 계층**
([10-apiserver/05](../10-apiserver/05-registry-storage.md))이 Service 생성 시 IP를 할당한다 — Service의
전략(strategy)이 ipallocator를 호출.

- 사용자가 IP를 지정하면(`spec.clusterIP`): 그게 비어 있는지 확인 후 예약.
- 안 지정하면(`ClusterIP: ""`): 풀에서 빈 IP를 골라 배정(`AllocateNext`).

## 비트맵 할당자 (전통 방식)

`ipallocator/bitmap.go`. Service CIDR(예: `10.96.0.0/12`)의 각 IP를 **비트 하나**로 표현하는 비트맵으로
할당 상태를 추적한다:

- 할당: 빈 비트(0)를 찾아 1로 — O(범위) 스캔 또는 랜덤.
- 해제: Service 삭제 시 그 비트를 0으로.

문제: 비트맵 전체가 **하나의 객체로 etcd에 저장**돼, CIDR이 크거나 변경이 잦으면 부담이고, **CIDR을
런타임에 못 바꾼다**(고정 풀).

## ServiceCIDR / IPAddress API (새 방식)

이 한계를 풀기 위해 새 API가 도입됐다(`networking/v1`의 **ServiceCIDR**, **IPAddress**):

- **ServiceCIDR**: 서비스 IP 풀을 **API 객체**로 선언. 여러 개 둘 수 있고 **동적으로 추가**할 수 있다
  (풀 확장).
- **IPAddress**: 할당된 개별 IP를 **객체 하나**로 표현. "이 IP는 이 Service가 씀".

`pkg/registry/core/service/ipallocator/cidrallocator.go`가 이 방식의 할당자다. 할당이 **IPAddress 객체
생성**이 되므로:

- 비트맵 단일 객체 경합이 사라진다(IP마다 별도 객체).
- ServiceCIDR을 추가해 풀을 **무중단 확장**할 수 있다(IP 고갈 대응).
- `pkg/controller/servicecidrs/servicecidrs_controller.go`가 ServiceCIDR의 상태(준비/사용 중)를 관리한다.

```
Service 생성(ClusterIP="") 
   → cidrallocator: ServiceCIDR 풀에서 빈 IP 선택 → IPAddress 객체 생성(예약)
   → Service.spec.clusterIP 에 기록
Service 삭제 → IPAddress 객체 삭제(IP 반환)
```

## NodePort 할당

NodePort도 비슷하다(`portallocator/`). NodePort Service는 노드 포트 범위(기본 30000–32767)에서 포트를
할당받는다. 같은 비트맵/할당자 패턴 — 중복 없는 포트를 배정하고, 삭제 시 반환한다.

## 일관성 — 재시작 시 복구

apiserver가 재시작하면 할당 상태를 etcd에서 복원한다. 또한 **repair 컨트롤러**(`ipallocator/controller/`,
`portallocator/controller/`)가 주기적으로 "실제 Service들이 쓰는 IP/포트"와 "할당자 기록"을 대조해
누수(할당됐지만 안 쓰는)나 충돌을 고친다 — 또 하나의 reconcile 루프
([00-foundations/05](../00-foundations/05-controller-pattern.md)).

## 데이터플레인과의 연결

할당된 ClusterIP는 [02-service-discovery.md](02-service-discovery.md)의 모델대로 kube-proxy/cilium이
데이터플레인 규칙으로 구현한다([14-kube-proxy/03](../14-kube-proxy/03-iptables-rules.md),
[42-cilium/04](../42-cilium/04-kube-proxy-replacement.md)). 즉 apiserver가 "가상 IP를 배정"하고,
데이터플레인이 "그 IP를 백엔드로 변환"한다 — 할당과 시행의 분리.

## 더 읽을 곳
- [02-service-discovery.md](02-service-discovery.md) — Service/ClusterIP 모델
- [10-apiserver/05](../10-apiserver/05-registry-storage.md) — 할당이 일어나는 registry
- [14-kube-proxy/03](../14-kube-proxy/03-iptables-rules.md) — ClusterIP 데이터플레인
