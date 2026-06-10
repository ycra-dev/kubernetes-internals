# 50.04 · 임시(Ephemeral) 볼륨

**근거**: `kubernetes/staging/src/k8s.io/api/core/v1/types.go`(`EmptyDir` `:63`, `CSI` `:177`,
`Ephemeral` `:204`), `kubernetes/pkg/controller/volume/ephemeral/controller.go`(`syncHandler` `:221`)

[README](README.md), [01-pv-binding.md](01-pv-binding.md)의 PV/PVC는 **영속(persistent)** 볼륨이다.
하지만 많은 워크로드는 **Pod 수명만큼만** 필요한 임시 저장을 쓴다. 이 문서는 그 임시 볼륨들 — emptyDir,
generic ephemeral, CSI inline — 을 본다. 공통점: **Pod가 사라지면 볼륨도 사라진다**.

## 세 종류

Pod의 `volumes`에 직접 선언되는 볼륨 소스들(`core/v1/types.go`):

| 종류 | 필드 | 수명 | 용도 |
|------|------|------|------|
| **emptyDir** | `EmptyDir` (`:63`) | Pod 수명 | 컨테이너 간 공유 임시 공간, 캐시, 스크래치 |
| **CSI inline** | `CSI` (`:177`) | Pod 수명 | CSI 드라이버가 제공하는 경량 임시 볼륨(시크릿 주입 등) |
| **generic ephemeral** | `Ephemeral` (`:204`) | Pod 수명 | 완전한 스토리지 기능(스냅샷/크기/StorageClass)이 필요한 임시 볼륨 |

## emptyDir — 가장 단순

`EmptyDir`(`types.go:63`)는 Pod 시작 시 빈 디렉토리로 생성돼, **같은 Pod의 컨테이너들이 공유**하고, Pod가
삭제되면 사라진다. kubelet volume manager([13-kubelet/05](../13-kubelet/05-volume-manager.md))가 노드
디스크(또는 `medium: Memory`면 tmpfs)에 만든다. PV/PVC가 필요 없다 — 컨트롤 플레인을 안 거치는 순수 노드
로컬 볼륨.

```
용도: 다단계 처리의 중간 산출물, 사이드카가 공유하는 소켓/파일, 임시 캐시
```

## CSI inline 볼륨

`CSI`(`types.go:177`)는 **CSI 드라이버가 제공하는 임시 볼륨**을 Pod 스펙에 **직접** 선언한다(PVC 없이).
드라이버가 "임시 inline"을 지원해야 한다([02-csi.md](02-csi.md)). 예: 시크릿 관리 드라이버가 외부
볼트에서 비밀을 가져와 Pod에 마운트(영속 PVC 없이 Pod마다).

PVC를 안 만들므로 가볍지만, 스냅샷/크기 조정 같은 PVC 기능은 없다.

## Generic Ephemeral 볼륨 — PVC의 임시 버전

`Ephemeral`(`types.go:204`)은 **PVC의 모든 기능**(StorageClass, 동적 프로비저닝, 스냅샷, 크기)을 갖되
**Pod 수명에 묶인** 볼륨이다. 핵심: Pod 스펙 안의 **volumeClaimTemplate**으로 선언하면, 컨트롤러가 Pod별
PVC를 자동 생성한다.

### EphemeralVolume 컨트롤러
`pkg/controller/volume/ephemeral/controller.go`([11-controller-manager/01](../11-controller-manager/01-controller-catalog.md))가
이를 처리한다. `syncHandler`(`controller.go:221`):

```
Pod{ volumes: [ephemeral: { volumeClaimTemplate: {storageClass: fast, size: 10Gi} }] }
   │  ephemeral 컨트롤러가 Pod를 watch
   ▼
template으로 PVC 생성 (controller.go:290~293)
   - ownerReference = Pod  ← 핵심
   - 일반 PVC처럼 동적 프로비저닝 (50.01)
   │
Pod 삭제 → ownerReference로 PVC도 GC (11-controller-manager/04)
   → 동적 PV도 reclaim
```

**ownerReference가 Pod**인 것이 핵심([00-foundations/05](../00-foundations/05-controller-pattern.md))이다 —
Pod가 사라지면 GC가 그 PVC를 자동 삭제하고, 동적 PV도 회수된다([01-pv-binding.md](01-pv-binding.md)).
그래서 "PVC의 풍부한 기능 + 임시 수명"을 동시에 얻는다.

## 영속 vs 임시 정리

| | PVC(영속) | generic ephemeral | emptyDir |
|--|-----------|-------------------|----------|
| 수명 | PVC 삭제까지 | Pod 수명 | Pod 수명 |
| 생성 | 사용자가 PVC | Pod 스펙 template → 자동 PVC | Pod 스펙(노드 로컬) |
| 기능 | 풀(스냅샷/크기/SC) | 풀 | 없음(단순 디렉토리) |
| GC | 수동/reclaim | Pod ownerRef로 자동 | Pod와 함께 |

선택 기준: 데이터를 Pod 너머 보존 → PVC. 임시지만 스토리지 기능 필요 → generic ephemeral. 단순 스크래치 →
emptyDir.

## 더 읽을 곳
- [01-pv-binding.md](01-pv-binding.md) — 영속 PVC 바인딩
- [02-csi.md](02-csi.md) — CSI 드라이버
- [13-kubelet/05](../13-kubelet/05-volume-manager.md) — 볼륨 mount
- [11-controller-manager/04](../11-controller-manager/04-garbage-collector.md) — ownerRef GC(임시 PVC 정리)
