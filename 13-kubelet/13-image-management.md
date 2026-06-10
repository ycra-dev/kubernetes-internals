# 13.13 · 이미지 관리

**근거**: `kubernetes/pkg/kubelet/images/` (`image_manager.go` `EnsureImageExists` `:168`, `backOff` `:62`,
`image_gc_manager.go`, `pullmanager/`), `kubernetes/pkg/credentialprovider/`

[01-pod-lifecycle.md](01-pod-lifecycle.md)의 SyncPod가 컨테이너 시작 전 이미지를 보장한다고 했다. 이
문서는 kubelet의 **이미지 관리** — pull 정책, 자격증명, 중복 pull 제어, GC — 를 본다. CRI로 containerd에
pull을 위임하지만([02-cri.md](02-cri.md)), "언제·어떻게 pull할지"는 kubelet이 정한다.

## EnsureImageExists — pull 정책 적용

`image_manager.go:168`의 `EnsureImageExists`가 진입점이다. 컨테이너의 `imagePullPolicy`에 따라:

| 정책 | 동작 |
|------|------|
| `Always` | 매번 레지스트리에서 최신 확인(pull 또는 다이제스트 검증) |
| `IfNotPresent` | 노드에 없으면만 pull(기본, 태그 기준) |
| `Never` (`:157`) | pull 안 함 — 없으면 실패 |

정책 평가 후 필요하면 CRI `PullImage`를 호출한다([02-cri.md](02-cri.md)) → containerd가 실제로 받는다
([30-containerd/02](../30-containerd/02-content-snapshots.md)).

## pull 백오프와 중복 제거

`backOff`(`image_manager.go:62`)가 실패한 pull을 지수 백오프로 재시도한다 — 레지스트리 장애 시 무한
재시도로 폭주하지 않게. 이것이 `ImagePullBackOff` 상태의 출처다([99-appendix/troubleshooting](../99-appendix/troubleshooting.md)).

`pullmanager/`는 **같은 이미지를 동시에 요구하는 여러 Pod**의 pull을 중복 제거한다 — 한 번만 받아 공유.
containerd의 content store가 콘텐츠 주소로 중복을 막는 것([30-containerd/02](../30-containerd/02-content-snapshots.md))과
함께, kubelet 수준에서도 동시 pull을 합친다.

## 자격증명 — credential provider

프라이빗 레지스트리는 인증이 필요하다. `pkg/credentialprovider/`가 이미지별 자격증명을 찾는다:

- **imagePullSecrets**: Pod/ServiceAccount에 붙은 레지스트리 Secret([17-security/01](../17-security/01-serviceaccount-secrets.md)).
- **credential provider plugin**(`credentialprovider/plugin/`): 클라우드 레지스트리(ECR/GCR/ACR)의
  자격증명을 **외부 플러그인**으로 동적으로 받아온다 — 정적 Secret 없이 클라우드 IAM으로
  ([17-security/01](../17-security/01-serviceaccount-secrets.md)의 워크로드 신원 연계).
- `keyring.go`가 여러 소스의 자격증명을 합쳐 레지스트리별로 매칭한다.

찾은 자격증명을 CRI `PullImage`의 `AuthConfig`로 전달한다([02-cri.md](02-cri.md)의 `PullImage`).

## 이미지 GC

`image_gc_manager.go`. 노드 디스크가 차면 안 쓰는 이미지를 정리한다([04-eviction.md](04-eviction.md)에서
언급):

- 디스크 사용률이 **HighThresholdPercent**를 넘으면, **LowThresholdPercent**까지 이미지를 지운다.
- **LRU**: 가장 오래 안 쓴(마지막 사용 시각 기준) 이미지부터.
- 현재 컨테이너가 쓰는 이미지는 보호.

eviction manager가 imagefs 압박 시 이 GC를 트리거한다([04-eviction.md](04-eviction.md)의
`reclaimNodeLevelResources`). containerd 수준의 content/snapshot GC([30-containerd/07](../30-containerd/07-metadata-gc.md))와는
별개 계층 — kubelet이 "어떤 이미지를 지울지" 정하면 containerd가 실제 정리.

## 정리

```
SyncPod → EnsureImageExists (pull 정책)
   ├─ 자격증명 찾기(credentialprovider) → CRI PullImage(AuthConfig)
   ├─ 동시 pull 중복 제거(pullmanager), 실패 시 백오프(ImagePullBackOff)
   └─ 디스크 압박 시 image GC(LRU)로 회수
```

## 더 읽을 곳
- [02-cri.md](02-cri.md) — pull을 위임하는 CRI
- [30-containerd/02](../30-containerd/02-content-snapshots.md) — 받은 이미지 저장
- [17-security/01](../17-security/01-serviceaccount-secrets.md) — 레지스트리 자격증명
- [04-eviction.md](04-eviction.md) — 디스크 압박과 GC 트리거
