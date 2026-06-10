# 30.06 · 이미지 검증과 Transfer 서비스

**근거**: `containerd/plugins/imageverifier/`, `containerd/core/transfer/`(`image/`, `local/`),
`containerd/core/remotes/`

이미지를 받을 때의 **신뢰**(서명 검증)와, 이미지 전송을 추상화한 **Transfer 서비스**를 다룬다.
[02-content-snapshots.md](02-content-snapshots.md)의 이미지 처리를 보안/구조 관점에서 보강한다.

## 이미지 검증 — imageverifier

`containerd/plugins/imageverifier/`. 받은 이미지가 **신뢰할 수 있는지** 검증하는 플러그인 지점이다:

- **서명 검증**: 이미지가 신뢰된 키로 서명됐는지(예: Sigstore/cosign, Notation). 공급망 공격(변조된
  이미지 주입)을 막는다.
- pull 후 unpack/실행 전에 검증해, 검증 실패 이미지로 컨테이너가 뜨는 것을 차단.

> Kubernetes 측에도 이미지 정책 어드미션(`imagepolicy`,
> [10-apiserver/04](../10-apiserver/04-admission.md))이 있지만, 그것은 "어떤 이미지를 허용하나"(API
> 수준)이고, imageverifier는 "받은 바이트가 진짜인가"(런타임 수준)다. 공급망 보안은 두 계층을 함께 쓴다.

## Transfer 서비스 — 이미지 이동의 추상화

`containerd/core/transfer/`. 이미지 pull/push/import/export를 **소스→대상 전송**이라는 일반 모델로
추상화한다:

- `transfer/image/`, `transfer/local/`, `transfer/archive/`: 레지스트리·로컬·tar 아카이브 간 전송.
- 클라이언트는 "이 소스에서 저 대상으로"만 지정하고, containerd가 레이어 다운로드·content store
  저장·unpack을 조율([02-content-snapshots.md](02-content-snapshots.md)).

이로써 "레지스트리에서 pull", "tar에서 import", "다른 레지스트리로 push"가 같은 API로 처리된다.

## remotes — 레지스트리 통신

`containerd/core/remotes/`가 실제 레지스트리(OCI Distribution API)와 말하는 계층이다: 매니페스트 조회,
blob 다운로드, 인증(토큰), 미러 폴백. Transfer 서비스가 이를 사용해 이미지를 가져온다.

## 전체 그림 (보안 포함)

```
PullImage (CRI)
   │
Transfer 서비스
   ├─ remotes: 레지스트리에서 매니페스트/레이어 다운로드
   ├─ imageverifier: 서명 검증 (실패 시 차단)
   ├─ content store: 콘텐츠 주소로 저장 (digest 무결성)
   └─ unpack: snapshot으로 rootfs 구성
```

digest 기반 저장([02-content-snapshots.md](02-content-snapshots.md))이 "받은 바이트의 무결성"을,
imageverifier가 "출처의 신뢰성"을 보장한다.

## 더 읽을 곳
- [02-content-snapshots.md](02-content-snapshots.md) — content store/snapshot
- [17-security/02](../17-security/02-pod-security.md) — Pod 보안과의 연계
