# 17.04 · 저장 암호화 (Encryption at Rest)

**근거**: `kubernetes/staging/src/k8s.io/apiserver/pkg/storage/value/encrypt/`
(`aes/`, `secretbox/`, `identity/`, `envelope/`, `envelope/kmsv2/`),
`apiserver/pkg/apis/apiserver/types_encryption.go`(`EncryptionConfiguration` `:28`)

[17-security/01](01-serviceaccount-secrets.md)에서 "Secret은 기본적으로 etcd에 base64일 뿐, 진짜 보호는
저장 암호화가 필요"라 했다. 이 문서는 그 **저장 암호화(encryption at rest)** — apiserver가 etcd에 쓰기 전
값을 암호화하는 메커니즘 — 를 본다. etcd 디스크/백업이 유출돼도 Secret이 평문으로 노출되지 않게 한다.

## 어디서 암호화하나 — storage 계층

암호화는 apiserver의 **storage 계층**에서 일어난다([10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md)).
객체를 직렬화한 뒤([00-foundations/10](../00-foundations/10-serialization.md)) etcd에 쓰기 직전 **value
transformer**가 암호화하고, 읽을 때 복호화한다:

```
객체 → 직렬화(protobuf) → [value transformer: 암호화] → etcd 저장
etcd 읽기 → [value transformer: 복호화] → 역직렬화 → 객체
```

etcd는 **암호문**만 본다 — etcd 자체는 평문을 모른다([20-etcd](../20-etcd/)).

## EncryptionConfiguration — 무엇을 어떻게

apiserver는 `--encryption-provider-config`로 **EncryptionConfiguration**(`types_encryption.go:28`)를
받는다. "어떤 리소스를, 어떤 provider로 암호화할지"를 선언한다:

```yaml
resources:
- resources: [secrets, configmaps]      # 이 리소스들을
  providers:                            # 순서대로 시도
  - aescbc: { keys: [...] }             # 쓰기: 첫 provider로 암호화
  - identity: {}                        # 읽기: 매칭되는 provider로 복호화
```

핵심 규칙:
- **쓰기**: 목록의 **첫 provider**로 암호화.
- **읽기**: 암호문의 prefix로 어느 provider가 썼는지 알아 그것으로 복호화(여러 provider 공존 → 키 회전
  가능).
- **`identity`**(`encrypt/identity/`): 암호화 안 함(평문). 목록 첫째면 그 리소스는 평문 저장.

## Provider 종류

`encrypt/` 아래 transformer 구현:

| Provider | 디렉토리 | 키 관리 |
|----------|----------|---------|
| `identity` | `identity/` | 암호화 없음(평문) |
| `aescbc` / `aesgcm` | `aes/` | 설정 파일에 키 직접(로컬) |
| `secretbox` | `secretbox/` | 설정 파일에 키(NaCl) |
| `kms` (envelope) | `envelope/`, `kmsv2/` | **외부 KMS**가 키 관리 |

`aescbc`/`secretbox`는 키가 **설정 파일에 평문**으로 있어, apiserver 노드가 뚫리면 키도 노출된다(부분적
보호). 진짜 보호는 KMS다.

## Envelope 암호화 — KMS

`envelope/`(특히 `kmsv2/`)가 권장 방식이다. **봉투(envelope) 암호화**:

```
DEK(Data Encryption Key): 실제 데이터를 암호화하는 키 (apiserver가 생성)
KEK(Key Encryption Key):  DEK를 암호화하는 키 (외부 KMS가 보관, 절대 안 나옴)

저장: 데이터 ─DEK로 암호화→ 암호문
      DEK   ─KEK로 암호화(KMS 호출)→ 암호화된 DEK
      etcd에 [암호화된 DEK + 암호문] 저장
읽기: 암호화된 DEK ─KMS가 KEK로 복호화→ DEK → 데이터 복호화
```

- **KEK는 외부 KMS**(클라우드 KMS, HSM 등)에만 있고 apiserver/etcd에 절대 안 들어온다 — etcd 디스크가
  통째로 유출돼도 KMS 없이는 복호화 불가.
- apiserver는 KMS provider 플러그인(`grpc_service.go`)을 gRPC로 호출해 DEK를 감싸고/푼다.

### KMS v2 개선
`kmsv2/`는 v1 대비 성능/보안을 개선했다. DEK를 매번 KMS로 푸는 대신 **로컬 캐시**(`kmsv2/cache.go`)로
KMS 호출을 줄이고, 키 ID 추적으로 회전을 명확히 한다.

## 키 회전

provider 목록에 새 키를 첫째로 추가하면, 이후 쓰기는 새 키로 암호화되고 옛 키는 읽기용으로 남는다. 모든
기존 객체를 새 키로 다시 쓰려면 **Storage Version Migrator**([11-controller-manager/01](../11-controller-manager/01-controller-catalog.md))나
`kubectl get secrets -A -o yaml | kubectl replace`로 재암호화한다.

## 무엇을 암호화하나

최소한 **Secret**은 암호화해야 한다([17-security/01](01-serviceaccount-secrets.md)). ServiceAccount 토큰,
일부 ConfigMap도 대상이 될 수 있다. Event 같은 비민감 데이터는 `identity`로 두어 성능을 아낀다
(`types_encryption.go:45` 예시).

## 더 읽을 곳
- [01-serviceaccount-secrets.md](01-serviceaccount-secrets.md) — Secret 보호
- [10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md) — 암호화가 끼는 storage 계층
- [03-pki-certificates.md](03-pki-certificates.md) — 다른 보안 계층(전송 mTLS)
