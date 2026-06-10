# 13.15 · 마운트 시맨틱과 임시 컨테이너

**근거**: `kubernetes/staging/src/k8s.io/api/core/v1/types.go`(`SubPath` `:2409`, `MountPropagation`,
`RecursiveReadOnly` `:2414`, `EphemeralContainers` `:4184`), `kubernetes/pkg/kubelet/volumemanager/`

[05-volume-manager.md](05-volume-manager.md)의 볼륨 마운트와 [10-pod-spec.md](10-pod-spec.md)의 컨테이너를
세부적으로 보강한다 — 볼륨 마운트의 미묘한 옵션들과, 실행 중 Pod를 디버깅하는 임시 컨테이너.

## 마운트 시맨틱

컨테이너의 `volumeMounts`에는 단순 경로 외에 여러 옵션이 있다(`core/v1/types.go`의 VolumeMount):

### subPath
**`subPath`**(`types.go:2409`)는 볼륨의 **하위 경로만** 마운트한다. 예: 하나의 PVC를 여러 컨테이너가
각자 다른 하위 디렉토리로 쓸 때(`subPath: app1`, `subPath: app2`). ConfigMap의 특정 키만 특정 파일로
마운트할 때도 쓴다. kubelet volume manager가 마운트 시 그 하위 경로를 바인드 마운트한다.

### mountPropagation
**`mountPropagation`**(`types.go:2412`)은 마운트가 컨테이너↔호스트 간 **전파되는 방향**을 정한다:

| 값 | 의미 |
|----|------|
| `None`(기본) | 격리 — 한쪽 마운트가 다른 쪽에 안 보임 |
| `HostToContainer` | 호스트의 새 마운트가 컨테이너에 보임 |
| `Bidirectional` | 양방향 전파(컨테이너 마운트가 호스트에도) — **특권 필요** |

`Bidirectional`은 CSI 드라이버 Pod처럼 "컨테이너 안에서 마운트해 호스트/다른 Pod가 보게" 하는 특수
워크로드용이다([50-storage/02](../50-storage/02-csi.md)). 보안상 위험해 특권 컨테이너에만 허용된다
([17-security/02](../17-security/02-pod-security.md)).

### recursiveReadOnly
**`recursiveReadOnly`**(`types.go:2414`)는 readonly 마운트를 **하위 마운트까지 재귀적으로** 읽기 전용으로
만든다. 일반 readonly는 최상위만 ro라 하위 마운트는 쓰기 가능할 수 있었는데, 이를 막아 보안을 강화한다
(커널 지원 필요).

## 임시 컨테이너 (Ephemeral Containers)

**`EphemeralContainers`**(`types.go:4184`)는 **이미 실행 중인 Pod에 디버깅용 컨테이너를 추가**하는
기능이다. `kubectl debug`가 이를 쓴다.

### 왜 필요한가
프로덕션 이미지는 보통 셸/디버깅 도구가 없다(distroless 등). 컨테이너가 죽거나 이상할 때 들어가 볼
방법이 없다. 임시 컨테이너는 **디버깅 도구가 든 별도 이미지**를 그 Pod에 추가해, 같은 네임스페이스
(네트워크/프로세스)를 공유하며 조사하게 한다.

### 특징
- **subresource로 추가**: Pod의 `ephemeralcontainers` 서브리소스로만 추가
  ([10-apiserver/13](../10-apiserver/13-strategy-status.md)의 서브리소스 패턴) — 일반 spec 수정이 아님.
- **재시작 안 함**: 일반 컨테이너와 달리 죽어도 재시작되지 않는다(probe/리소스 보장 없음).
- **제거 불가**: 한 번 추가하면 Pod에서 못 뺀다(Pod가 사라질 때까지).
- **프로세스 네임스페이스 공유**(`shareProcessNamespace` 또는 `targetContainerName`): 대상 컨테이너의
  프로세스/파일시스템을 들여다볼 수 있다.

```
kubectl debug -it pod/x --image=busybox --target=app
   │  ephemeralcontainers 서브리소스에 추가
   ▼
kubelet이 그 Pod에 디버깅 컨테이너 시작(같은 네임스페이스)
   → app 컨테이너의 프로세스/네트워크를 같은 시야로 조사
```

이로써 프로덕션 distroless Pod도 재배포 없이 라이브 디버깅할 수 있다.

## 더 읽을 곳
- [05-volume-manager.md](05-volume-manager.md) — 볼륨 마운트 일반
- [10-pod-spec.md](10-pod-spec.md) — 컨테이너 종류
- [02-cri.md](02-cri.md) — exec(다른 디버깅 경로)
