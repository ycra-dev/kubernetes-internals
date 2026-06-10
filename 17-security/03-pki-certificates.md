# 17.03 · PKI와 인증서

**근거**: `kubernetes/cmd/kubeadm/app/phases/certs/`(`certlist.go`, 상수 `constants.go:55`~),
`kubernetes/pkg/kubelet/certificate/`, `kubernetes/pkg/controller/certificates/signer/`

Kubernetes 컴포넌트 간 통신은 거의 모두 **mTLS**(상호 TLS)다. 이 문서는 그 신뢰의 뿌리인 **PKI(인증서
체계)** — 여러 CA, 컴포넌트별 인증서, 자동 갱신 — 를 한곳에 모은다. 여러 문서(kubeadm, kubelet, etcd,
apiserver)에 흩어진 인증서 이야기를 꿰는 횡단 문서다.

## 여러 개의 CA — 신뢰 도메인 분리

하나의 CA로 모든 것을 서명하지 않는다. kubeadm은 **여러 CA**를 만든다(`constants.go`):

| CA | 상수 | 서명 대상 |
|----|------|-----------|
| **Kubernetes CA** | `CACertAndKeyBaseName="ca"` (`:56`) | apiserver 서버 인증서, 클라이언트(kubelet/admin/컨트롤러) 인증서 |
| **etcd CA** | `EtcdCACertAndKeyBaseName="etcd/ca"` (`:81`) | etcd 서버/피어, apiserver↔etcd 클라이언트 |
| **front-proxy CA** | `FrontProxyCA...` | API Aggregation 프록시([10-apiserver/07](../10-apiserver/07-crd-aggregation.md)) |

왜 분리하나: **신뢰 도메인 격리**. etcd CA로 서명된 인증서는 etcd 접근만, Kubernetes CA는 apiserver
접근만. 한 CA가 유출돼도 다른 도메인은 안전하다. 특히 etcd CA 분리는 "apiserver만 etcd에 접근"
([00-foundations/02](../00-foundations/02-architecture.md))을 암호학적으로 강제한다.

## ServiceAccount 키 — 별도

위 CA들과 별개로 **ServiceAccount 서명 키**가 있다(`constants.go`의 `ServiceAccountKey`). 이건 X.509
인증서가 아니라 **JWT 서명 키**다 — Pod의 SA 토큰을 서명/검증하는 데 쓴다
([17-security/01](01-serviceaccount-secrets.md)). apiserver가 이 키로 토큰을 발급하고 검증한다.

## 컴포넌트별 인증서

`kubeadm certs`([15-cli/01](../15-cli/01-kubeadm.md))가 생성하는 인증서들(`phases/certs/certlist.go`):

```
Kubernetes CA
├── apiserver (서버)          ─ 클라이언트가 apiserver를 신뢰
├── apiserver-kubelet-client  ─ apiserver가 kubelet에 접근(로그/exec)
├── kubelet client            ─ kubelet이 apiserver에 접근 (system:node:<name>)
├── admin.conf                ─ kubectl 관리자
├── controller-manager.conf   ─ 컨트롤러 매니저
└── scheduler.conf            ─ 스케줄러
etcd CA
├── etcd server/peer          ─ etcd 멤버 간
└── apiserver-etcd-client     ─ apiserver → etcd
```

각 인증서의 **subject**가 신원이 된다 — 예: kubelet 클라이언트 인증서의 CN=`system:node:<name>`,
O=`system:nodes`가 인증/인가의 입력이다([10-apiserver/02](../10-apiserver/02-authentication.md),
[10-apiserver/03](../10-apiserver/03-authorization.md)의 Node authorizer).

## 자동 갱신 — TLS bootstrap과 rotation

인증서는 만료된다. 수동 재발급은 운영 부담이라 **자동화**한다:

### kubelet TLS bootstrap
[13-kubelet/06](../13-kubelet/06-node-registration.md)에서 봤듯, kubelet은 처음에 부트스트랩 토큰으로
**CSR(CertificateSigningRequest)** 을 제출해 클라이언트 인증서를 발급받는다.

### CSR 서명 컨트롤러
`pkg/controller/certificates/signer/signer.go`가 CSR을 **자동 승인·서명**한다
([11-controller-manager/01](../11-controller-manager/01-controller-catalog.md)의 certificate 컨트롤러):

- 노드의 CSR이 적절한지(요청자가 그 노드인지) 검증 후 CA로 서명.
- `ca_provider.go`가 서명에 쓸 CA를 제공.

### 자동 rotation
`pkg/kubelet/certificate/kubelet.go`가 인증서 만료 전 새 CSR로 **갱신(rotation)** 한다(`:104`의
`certificate_manager_server_rotation_seconds` 메트릭이 그 추적). 서버/클라이언트 인증서 모두 자동 갱신돼
운영자 개입 없이 mTLS가 유지된다.

```
부팅: 부트스트랩 토큰 → CSR 제출 → signer 컨트롤러가 서명 → 인증서 수령
주기: 만료 전 → 새 CSR → 갱신  (rotation)
```

## 신뢰의 부트스트랩 (닭-달걀)

apiserver가 떠야 CSR을 처리하는데, apiserver 인증서는 누가 서명하나? kubeadm이 **클러스터 시작 전에**
CA와 핵심 인증서를 로컬에서 생성한다([15-cli/01](../15-cli/01-kubeadm.md)의 certs phase). 그 인증서로
static pod apiserver가 뜨고, 이후 노드들은 그 apiserver를 통해 CSR로 인증서를 받는다.

## 정리

```
신뢰의 뿌리: 여러 CA (Kubernetes/etcd/front-proxy) — 도메인 격리
   ├─ 컴포넌트마다 CA가 서명한 인증서 → subject가 신원(인증/인가 입력)
   ├─ SA 토큰은 별도 JWT 서명 키
   └─ TLS bootstrap + CSR signer + rotation → 자동 발급/갱신
```

이 PKI가 "컴포넌트 간 통신은 누구인지 증명된 mTLS"라는 보안 기반을 떠받친다.

## 더 읽을 곳
- [13-kubelet/06](../13-kubelet/06-node-registration.md) — kubelet TLS bootstrap
- [15-cli/01](../15-cli/01-kubeadm.md) — kubeadm certs phase
- [10-apiserver/02](../10-apiserver/02-authentication.md) — 인증서 기반 인증
- [01-serviceaccount-secrets.md](01-serviceaccount-secrets.md) — SA 토큰(JWT)
