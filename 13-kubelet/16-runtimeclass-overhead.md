# 13.16 · RuntimeClass와 Pod Overhead

**근거**: `kubernetes/staging/src/k8s.io/api/node/v1/types.go`(`RuntimeClass` `:36`, `Overhead` `:63`,
`scheduling` `:65`), `kubernetes/pkg/kubelet/runtimeclass/`,
`kubernetes/pkg/scheduler/framework/plugins/noderesources/`

[13-kubelet/02](02-cri.md)와 [30-containerd/03](../30-containerd/03-runtime-shim-oci.md)에서 RuntimeClass로
runc 외의 런타임(gVisor/Kata)을 고를 수 있다고 했다. 이 문서는 **RuntimeClass 객체**와, 그 런타임이
소비하는 자원을 스케줄링에 반영하는 **Pod overhead**를 본다.

## RuntimeClass — 런타임 선택

**RuntimeClass**(`node/v1/types.go:36`)는 "이 Pod를 어떤 런타임 핸들러로 실행할지"를 정하는 클러스터
스코프 객체다:

```
RuntimeClass "gvisor" { handler: "runsc" }
   │  Pod.spec.runtimeClassName: gvisor
   ▼
kubelet → CRI RunPodSandbox(runtimeHandler="runsc") → containerd가 gVisor shim 선택
   (13-kubelet/02, 30-containerd/03)
```

- `handler`: CRI 런타임 핸들러 이름. containerd 설정의 런타임에 매핑([30-containerd/01](../30-containerd/01-cri-plugin.md)).
- kubelet의 `runtimeclass/runtimeclass_manager.go`가 RuntimeClass를 watch해, Pod의
  `runtimeClassName`을 핸들러로 해석한다.

이로써 "이 워크로드는 강한 격리(VM/gVisor)로, 저 워크로드는 일반(runc)으로"를 Pod 단위로 정한다.

## scheduling — 적합한 노드로 제한

모든 노드가 모든 런타임을 지원하진 않는다(gVisor가 설치된 노드만 gVisor를 돌릴 수 있다). RuntimeClass의
`scheduling`(`types.go:65`)이 이를 처리한다:

- **nodeSelector/tolerations**: "이 런타임을 지원하는 노드"의 라벨/taint를 지정.
- 이 RuntimeClass를 쓰는 Pod에 그 제약이 **자동으로 합쳐진다**(어드미션이 주입).
- 그래서 스케줄러([12-scheduler](../12-scheduler/))가 런타임을 지원하는 노드에만 배정한다.

`scheduling`이 nil이면 모든 노드가 지원한다고 가정(`types.go:67` 주석).

## Pod Overhead — 런타임의 숨은 비용

핵심 문제: gVisor/Kata 같은 런타임은 **샌드박스 자체가 자원을 쓴다**(VM/추가 프로세스). 컨테이너의
requests/limits만 보면 이 오버헤드가 누락돼, 노드에 과배치(overcommit)된다.

**Overhead**(`node/v1/types.go:63`)가 이를 푼다 — RuntimeClass가 "이 런타임은 Pod당 CPU/메모리를 이만큼
추가로 쓴다"를 선언한다:

```
RuntimeClass "kata" { overhead: { cpu: 250m, memory: 160Mi } }
   │  이 RuntimeClass를 쓰는 Pod에 어드미션이 overhead 주입 → Pod.spec.overhead
   ▼
스케줄러: Pod 자원 계산 시 컨테이너 requests + overhead 합산
   (noderesources/resource_allocation.go가 Overhead 포함)
   → 노드에 "컨테이너 + 샌드박스" 만큼의 자리가 있어야 배정
```

- **스케줄링**: noderesources 필터/점수([12-scheduler/02](../12-scheduler/02-plugins.md))가 overhead를
  더한 총량으로 노드 적합성을 본다 — 샌드박스 비용까지 고려해 과배치를 막는다.
- **cgroup**: kubelet cm이 Pod cgroup 예산에 overhead를 반영([11-cgroups.md](11-cgroups.md)).
- **eviction/쿼터**: overhead가 Pod의 실효 자원에 포함돼 ResourceQuota([45-policy/01](../45-policy/01-quota-limitrange.md))와
  eviction([04-eviction.md](04-eviction.md)) 계산에도 들어간다.

overhead 주입은 어드미션 플러그인(`runtimeclass`,
[10-apiserver/04](../10-apiserver/04-admission.md))이 RuntimeClass를 보고 한다.

## 정리

```
RuntimeClass:
  handler    → 어떤 런타임(runc/gVisor/Kata)  (13-kubelet/02, 30-containerd/03)
  scheduling → 그 런타임 지원 노드로 제한        (12-scheduler)
  overhead   → 샌드박스 자원을 스케줄/cgroup/쿼터에 반영  (정확한 자원 회계)
```

이로써 "강한 격리 런타임을 쓰되, 그 비용을 정확히 회계해 안전하게 배치"한다.

## 더 읽을 곳
- [02-cri.md](02-cri.md) — RuntimeClass 핸들러 → CRI
- [30-containerd/03](../30-containerd/03-runtime-shim-oci.md) — gVisor/Kata shim
- [11-cgroups.md](11-cgroups.md) — overhead의 cgroup 반영
- [12-scheduler/02](../12-scheduler/02-plugins.md) — overhead 포함 자원 회계
