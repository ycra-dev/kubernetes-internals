# 50.01 · PV 바인딩과 동적 프로비저닝

**근거**: `kubernetes/pkg/controller/volume/persistentvolume/pv_controller.go`
(`syncClaim` `:238`, `provisionClaim` `:377`, `bind` `:396`)

[README](README.md)에서 본 PVC↔PV 바인딩을 컨트롤러 코드 수준에서 추적한다. PV Binder 컨트롤러는 **두
객체(PVC, PV)를 양쪽에서 동기화**하는 reconcile 루프다([00-foundations/05](../00-foundations/05-controller-pattern.md)).

## 두 개의 동기화 루프

컨트롤러는 PVC와 PV를 각각 watch하며 둘을 맞춘다(`pv_controller_base.go`):

- **syncClaim**(`pv_controller.go:238`): PVC 하나를 보고 "맞는 PV에 바인딩됐나"를 확인.
- **syncVolume**: PV 하나를 보고 "올바른 PVC에 묶였나, 해제됐으면 회수해야 하나"를 확인.

두 루프가 협력해 "PVC ↔ PV"가 양방향으로 일관되게 바인딩되도록 한다.

## syncClaim — PVC의 여정

`syncClaim`(`pv_controller.go:238`)의 핵심 분기:

```
PVC가 아직 안 묶였나(Pending)?
  ├─ 맞는 기존 PV 있음 → bind(volume, claim)   (:396)
  └─ 없음 + StorageClass 있음 → provisionClaim(claim)  (:377)  ← 동적 생성
PVC가 이미 묶였나(Bound)?
  └─ 바인딩이 유효한지 확인(PV가 사라졌나 등)
```

### 매칭
바인딩 가능한 기존 PV를 찾을 때는 **용량(요청 이상)**, **접근 모드**(RWO/RWX/ROX), **StorageClass**,
**셀렉터/볼륨모드**가 맞아야 한다(`index.go`의 인덱스로 후보 탐색).

### bind — 양방향 연결
`bind`(`pv_controller.go:396`)는 PVC.spec.volumeName ← PV, PV.spec.claimRef ← PVC를 **양쪽 다 설정**하고
둘의 상태를 Bound로 만든다. 양방향 참조라 한쪽만 보고도 짝을 안다.

## provisionClaim — 동적 프로비저닝

맞는 PV가 없고 PVC에 StorageClass가 있으면 **동적 생성**한다(`provisionClaim`, `pv_controller.go:377`):

- StorageClass의 **provisioner**가 in-tree면 컨트롤러가 직접, **CSI**면 외부 프로비저너에게 위임한다
  ([02-csi.md](02-csi.md)).
- 비동기 작업으로 새 PV를 만들고(클라우드 디스크 생성 등), 완료되면 PVC에 바인딩.

`:373` 주석대로 프로비저닝은 **비동기**라 시간이 걸릴 수 있고, 그동안 PVC는 Pending으로 남는다.

## 회수(Reclaim)

PVC가 삭제되면 PV의 `persistentVolumeReclaimPolicy`에 따라:
- **Delete**: 실제 스토리지(클라우드 디스크 등)까지 삭제.
- **Retain**: PV를 Released 상태로 보존(데이터 유지, 수동 정리 필요).

## 스케줄러와의 협력 — WaitForFirstConsumer

`volumeBindingMode: WaitForFirstConsumer`면 바인딩을 **Pod가 스케줄될 때까지 미룬다.** 이유: 볼륨이
특정 존에만 있을 수 있어, Pod가 어느 노드에 갈지 정해진 뒤 그 노드에서 접근 가능한 PV에 바인딩해야 한다.
이 조율은 스케줄러의 `volumebinding` 플러그인([12-scheduler/02](../12-scheduler/02-plugins.md))이 PreBind
단계에서 한다([13-kubelet/05](../13-kubelet/05-volume-manager.md)).

## 더 읽을 곳
- [02-csi.md](02-csi.md) — 외부 프로비저너/드라이버 프로토콜
- [13-kubelet/05](../13-kubelet/05-volume-manager.md) — 바인딩 후 노드 mount
