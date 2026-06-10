# 99 · 용어집

이 문서 모음에서 반복되는 핵심 용어를 한 줄로 정리한다. 각 항목은 상세 문서로 링크한다.

## API / 객체 모델
- **GVK / GVR** — Group/Version/Kind(타입 식별) / Group/Version/Resource(REST 경로). → [00-foundations/03](../00-foundations/03-api-object-model.md)
- **Scheme** — GVK↔Go타입 매핑과 변환의 레지스트리. → [00-foundations/03](../00-foundations/03-api-object-model.md)
- **spec / status** — 원하는 상태(사용자) / 현재 상태(시스템). 선언형의 핵심.
- **ObjectMeta** — 모든 객체 공통 메타데이터(name/namespace/uid/labels/ownerReferences...).
- **resourceVersion (RV)** — 객체의 버전. etcd의 revision에서 온다. 낙관적 동시성·watch의 기준. → [10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md)
- **CRD** — CustomResourceDefinition. 새 API 타입을 선언. → [10-apiserver/07](../10-apiserver/07-crd-aggregation.md)
- **finalizer** — 삭제 전 정리 훅. → [00-foundations/05](../00-foundations/05-controller-pattern.md)
- **ownerReference** — 소유 관계. GC와 역참조의 근거. → [00-foundations/05](../00-foundations/05-controller-pattern.md)

## 제어 / 동기화
- **reconcile** — 원하는 상태와 현재 상태의 차이를 좁히는 루프. → [00-foundations/05](../00-foundations/05-controller-pattern.md)
- **level-triggered** — 이벤트가 아니라 현재 전체 상태를 보고 동작(놓쳐도 복구). → [00-foundations/02](../00-foundations/02-architecture.md)
- **Informer / Reflector / DeltaFIFO / Indexer** — list/watch 캐시 파이프라인 구성요소. → [00-foundations/04](../00-foundations/04-list-watch-informer.md)
- **Workqueue** — 중복제거·재시도·rate limit 큐. → [00-foundations/04](../00-foundations/04-list-watch-informer.md)
- **watch** — "이 RV 이후의 변경"을 받는 스트림. → [20-etcd/05](../20-etcd/05-lease-watch.md)
- **leader election** — HA에서 활성 1개만 일하게. → [00-foundations/06](../00-foundations/06-leader-election.md)

## 표준 인터페이스 (플러그인 경계)
- **CRI** — Container Runtime Interface. kubelet↔런타임(gRPC). → [13-kubelet/02](../13-kubelet/02-cri.md)
- **CNI** — Container Network Interface. 컨테이너 네트워크 설정. → [40-networking/01](../40-networking/01-cni.md)
- **CSI** — Container Storage Interface. 외부 스토리지 드라이버. → [50-storage](../50-storage/)
- **OCI** — Open Container Initiative. 컨테이너 런타임/이미지 표준(runc 등). → [30-containerd/03](../30-containerd/03-runtime-shim-oci.md)

## apiserver
- **인증/인가/어드미션** — 누구/권한/내용 검증·변형. → [10-apiserver/02](../10-apiserver/02-authentication.md), [03](../10-apiserver/03-authorization.md), [04](../10-apiserver/04-admission.md)
- **RBAC** — Role 기반 인가(allow-list). → [10-apiserver/03](../10-apiserver/03-authorization.md)
- **APF** — API Priority and Fairness. 과부하 시 공정 큐잉. → [10-apiserver/01](../10-apiserver/01-request-pipeline.md)
- **cacher** — apiserver의 watch 캐시(etcd watch 1개를 팬아웃). → [10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md)

## 노드 / 런타임
- **SyncLoop / SyncPod** — kubelet의 Pod 수렴 루프. → [13-kubelet/01](../13-kubelet/01-pod-lifecycle.md)
- **PLEG** — Pod Lifecycle Event Generator(컨테이너 상태 변화 감지). → [13-kubelet/01](../13-kubelet/01-pod-lifecycle.md)
- **sandbox / pause** — Pod의 공유 네트워크 네임스페이스. → [13-kubelet/02](../13-kubelet/02-cri.md)
- **QoS** — Guaranteed/Burstable/BestEffort(축출 순서). → [13-kubelet/03](../13-kubelet/03-resource-mgmt.md)
- **eviction** — 노드 자원 압박 시 Pod 축출. → [13-kubelet/04](../13-kubelet/04-eviction.md)
- **shim** — containerd와 컨테이너 사이의 영속 중간 프로세스. → [30-containerd/03](../30-containerd/03-runtime-shim-oci.md)
- **snapshot (containerd)** — 이미지 레이어 위 쓰기 가능 rootfs. → [30-containerd/02](../30-containerd/02-content-snapshots.md)

## 네트워킹
- **ClusterIP / NodePort / LoadBalancer** — Service 타입. → [40-networking/02](../40-networking/02-service-discovery.md)
- **EndpointSlice** — Service의 백엔드 Pod 목록(분할형). → [14-kube-proxy/01](../14-kube-proxy/01-endpointslice-tracking.md)
- **NetworkPolicy** — Pod 간 통신 규칙(시행은 CNI). → [42-cilium/03](../42-cilium/03-networkpolicy.md)
- **identity (cilium)** — 라벨 집합에 부여된 보안 ID(정책 단위). → [42-cilium/02](../42-cilium/02-agent.md)
- **eBPF** — 커널에 안전히 주입하는 프로그램(cilium 데이터패스). → [42-cilium/01](../42-cilium/01-ebpf-datapath.md)
- **kube-proxy replacement** — cilium이 eBPF로 서비스 LB를 대체. → [42-cilium/04](../42-cilium/04-kube-proxy-replacement.md)

## etcd
- **Raft** — 복제 로그 합의(리더/과반). → [20-etcd/01](../20-etcd/01-raft.md)
- **WAL** — Write-Ahead Log(적용 전 디스크 기록). → [20-etcd/02](../20-etcd/02-wal-snapshot.md)
- **MVCC / revision** — 다중 버전 저장, 단조 증가 순번. → [20-etcd/03](../20-etcd/03-mvcc.md)
- **bbolt** — etcd의 B+tree 백엔드(단일 파일). → [20-etcd/04](../20-etcd/04-backend-bbolt.md)
- **compaction / defrag** — 오래된 버전 제거 / 디스크 회수. → [20-etcd/04](../20-etcd/04-backend-bbolt.md)
- **lease (etcd)** — TTL 키. → [20-etcd/05](../20-etcd/05-lease-watch.md)
- **선형화 읽기 (ReadIndex)** — 최신성을 보장하는 읽기. → [20-etcd/06](../20-etcd/06-server-flow.md)

## 오토스케일링 (혼동 주의)
- **HPA** — Pod **개수** 조정(빌트인). → [11-controller-manager/01](../11-controller-manager/01-controller-catalog.md)
- **VPA** — Pod **리소스 요청** 조정(별도 레포). → [70-autoscaling/01](../70-autoscaling/01-vpa.md)
- **Cluster Autoscaler** — **노드 개수** 조정(별도 레포). → [70-autoscaling](../70-autoscaling/)

## 확장 개발
- **controller-runtime** — 오퍼레이터 프레임워크(Manager/Reconciler). → [60-controller-runtime](../60-controller-runtime/)
- **kubebuilder** — 오퍼레이터 스캐폴딩. → [61-kubebuilder](../61-kubebuilder.md)
- **KEP** — Kubernetes Enhancement Proposal(설계 문서). → [90-enhancements](../90-enhancements.md)
