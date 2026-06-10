# Kubernetes 생태계 코드베이스 뜯어보기 — 공부 기록

Kubernetes 생태계 핵심 레포 11개의 코드를 따라가며 정리한 공부 기록이다. 단일 프로젝트가 아니라 서로 독립적으로 개발되면서도 함께 클러스터를 이루는 레포들의 모음이다.

> 모든 설명은 실재하는 코드 경로에 묶어 둔다. 경로는 `repo/path/to/file.go` 형식으로 인용하고,
> 직접 확인하지 않은 추측은 적지 않는다.

## 대상 레포와 버전

| 레포 | Go 모듈 | 역할 | 확인된 버전대 |
|------|---------|------|---------------|
| `kubernetes` | `k8s.io/kubernetes` | 컨트롤 플레인 + 노드 컴포넌트 본체 | ~1.36 (go 1.26) |
| `apimachinery` | `k8s.io/apimachinery` | API 타입/스킴/직렬화 공용 기반 (staging 미러) | - |
| `client-go` | `k8s.io/client-go` | Go용 API 클라이언트 (staging 미러) | - |
| `etcd` | `go.etcd.io/etcd/v3` | 분산 KV 저장소 = 클러스터 백엔드 | v3 |
| `containerd` | `github.com/containerd/containerd/v2` | 컨테이너 런타임 (CRI 구현) | 2.3.0 |
| `cilium` | `github.com/cilium/cilium` | eBPF 기반 CNI/네트워킹 | 1.20-dev |
| `dns` | `k8s.io/dns` | 클러스터 DNS 컴포넌트 | - |
| `controller-runtime` | `sigs.k8s.io/controller-runtime` | 커스텀 컨트롤러 프레임워크 | - |
| `kubebuilder` | `sigs.k8s.io/kubebuilder/v4` | CRD/오퍼레이터 스캐폴딩 | v4 |
| `autoscaler` | (서브모듈 다수) | cluster-autoscaler / VPA 등 | - |
| `enhancements` | `k8s.io/enhancements` | KEP 설계 문서 | - |

> `apimachinery`, `client-go`는 `kubernetes/staging/src/k8s.io/` 에서 자동 배포되는 읽기 전용 미러다.
> 따라서 이 문서들은 원본인 `kubernetes/staging/src/k8s.io/...` 경로를 우선 인용한다.

## 전체 지도

```
                          ┌─────────────────────────────────────────────┐
   사용자/kubectl ──────► │   kube-apiserver  (유일한 etcd 접근 게이트)   │
                          │   인증 → 인가 → 어드미션 → registry → storage │
                          └───────────────┬─────────────────────────────┘
                                          │ (저장/watch)
                                   ┌──────▼──────┐
                                   │    etcd     │  Raft + MVCC, 단일 진실 공급원
                                   └──────▲──────┘
                                          │ list/watch
      ┌───────────────────────────┬───────┴───────────┬──────────────────────┐
      │                           │                   │                      │
┌─────▼──────────┐      ┌─────────▼────────┐   ┌──────▼───────┐     ┌─────────▼────────┐
│ controller-mgr │      │  kube-scheduler  │   │   kubelet    │ ... │   kube-proxy     │
│ reconcile 루프 │      │  Pod→Node 배정   │   │ (각 노드)    │     │ (각 노드, 서비스)│
└────────────────┘      └──────────────────┘   └──────┬───────┘     └──────────────────┘
                                                      │ CRI(gRPC)
                                               ┌──────▼───────┐
                                               │  containerd  │ ─► runc(OCI) 컨테이너
                                               └──────────────┘
                  네트워킹: cilium(CNI/eBPF) · dns(서비스 디스커버리)
                  확장:     controller-runtime / kubebuilder(오퍼레이터) · autoscaler
```

## 문서 구성과 읽는 순서

> **모든 문서(151개)의 전체 목차**는 [TOC.md](TOC.md)에 있다. 아래 표는 폴더 단위 개요다.

번호 순서가 곧 권장 독해 순서다. `00-foundations`가 나머지 모든 문서의 어휘·개념 토대이므로 먼저 읽는다.

| # | 폴더/문서 | 내용 | 근거 레포 |
|---|-----------|------|-----------|
| 00 | [foundations/](00-foundations/) | 횡단 기반: API 객체 모델, list/watch/informer, reconcile, 리더 선출 | apimachinery, client-go |
| 10 | [apiserver/](10-apiserver/) | 요청 파이프라인, 인증/인가/어드미션, 저장 계층, CRD/aggregation | kubernetes |
| 11 | [controller-manager/](11-controller-manager/) | 컨트롤러 카탈로그, 공통 머시너리, GC, 노드 라이프사이클 | kubernetes |
| 12 | [scheduler/](12-scheduler/) | 스케줄링 사이클, Framework 플러그인, 큐, preemption | kubernetes |
| 13 | [kubelet/](13-kubelet/) | SyncLoop, 파드 라이프사이클, CRI, cgroup, eviction, 볼륨 | kubernetes |
| 14 | [kube-proxy/](14-kube-proxy/) | 서비스 추상화, EndpointSlice 추적, iptables/IPVS/nftables | kubernetes |
| 15 | [cli/](15-cli/) | kubectl 구조, kubeadm 부트스트랩 | kubernetes |
| 16 | [cloud-controller-manager.md](16-cloud-controller-manager.md) | 클라우드 연동 컨트롤러(노드/라우트/LB) | kubernetes |
| 17 | [security/](17-security/) | 신뢰 경계, SA/토큰/Secret, Pod Security Admission | kubernetes |
| 18 | [observability/](18-observability/) | 메트릭/Prometheus, metrics-server, Event, 로깅 | kubernetes |
| 19 | [dra.md](19-dra.md) | 동적 리소스 할당(GPU/장치), ResourceClaim/Slice | kubernetes |
| 20 | [etcd/](20-etcd/) | Raft, WAL/스냅샷, MVCC, bbolt, lease/watch, 서버 흐름, API | etcd |
| 30 | [containerd/](30-containerd/) | 아키텍처, CRI 플러그인, content/snapshots, shim/OCI | containerd |
| 40 | [networking/](40-networking/) | Pod/Service CIDR 모델, CNI, 서비스 디스커버리, DNS | kubernetes, dns |
| 42 | [cilium/](42-cilium/) | eBPF 데이터패스, 에이전트, NetworkPolicy, proxy 대체, Hubble, 암호화/BGP/L7 | cilium |
| 45 | [policy/](45-policy/) | ResourceQuota/LimitRange, PDB, PriorityClass, 멀티테넌시 | kubernetes |
| 50 | [storage/](50-storage/) | PV/PVC/StorageClass, CSI 흐름 | kubernetes |
| 60 | [controller-runtime/](60-controller-runtime/) | Manager/Reconciler, client/cache, 이벤트 파이프라인, webhook | controller-runtime |
| 61 | [kubebuilder.md](61-kubebuilder.md) | 스캐폴딩 엔진, 플러그인 | kubebuilder |
| 70 | [autoscaling/](70-autoscaling/) | cluster-autoscaler 루프, VPA, HPA 구분 | autoscaler |
| 80 | [scenarios/](80-scenarios/) | 엔드투엔드: 파드 생성/서비스 요청/롤링업데이트/장애복구 | (통합) |
| 90 | [enhancements.md](90-enhancements.md) | KEP 프로세스, 설계↔코드 추적 | enhancements |
| 91 | [versioning-skew.md](91-versioning-skew.md) | API 버전 생애주기, 버전 스큐, deprecation 정책 | kubernetes |
| 95 | [development/](95-development/) | 코드 생성, 빌드, e2e/통합/conformance 테스트 | kubernetes |
| 99 | [appendix/](99-appendix/) | 용어집, 읽기 가이드, staging 지도, **트러블슈팅** | (통합) |

## 목적별 빠른 진입

- **클러스터 동작 큰 그림** → [00-foundations/02-architecture.md](00-foundations/02-architecture.md)
- **API 요청이 어떻게 처리되나** → [10-apiserver/01-request-pipeline.md](10-apiserver/)
- **컨트롤러는 어떻게 도나** → [00-foundations/05-controller-pattern.md](00-foundations/05-controller-pattern.md)
- **파드가 뜨는 전 과정** → [80-scenarios/01-pod-create.md](80-scenarios/)
- **저장 계층(etcd)** → [20-etcd/README.md](20-etcd/)
- **오퍼레이터를 직접 만들려면** → [60-controller-runtime/](60-controller-runtime/) + [61-kubebuilder.md](61-kubebuilder.md)
- **보안 경계가 궁금하면** → [17-security/](17-security/)
- **모니터링/디버깅** → [18-observability/](18-observability/)
- **장애가 났을 때(증상→코드)** → [99-appendix/troubleshooting.md](99-appendix/troubleshooting.md)

## 각 문서의 공통 틀

1. **역할** — 무엇을 책임지는가
2. **진입점** — 코드 어디서 시작하는가 (`main`/생성자/인터페이스)
3. **핵심 동작** — 실제 코드 흐름을 단계로 추적
4. **주요 디렉토리** — 경로 ↔ 책임 표
5. **더 읽을 곳** — 관련 문서 링크
