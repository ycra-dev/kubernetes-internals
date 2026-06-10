# 91 · 버전, 스큐, API deprecation

**근거 레포**: `kubernetes` (`staging/.../component-base/{version,compatibility,featuregate}/`,
`staging/.../apiserver/pkg/storageversion`)

Kubernetes는 끊임없이 진화하지만, **컴포넌트들이 서로 다른 버전으로 떠 있어도** 동작해야 하고(업그레이드
중), **오래된 클라이언트/매니페스트도** 깨지지 않아야 한다. 이 문서는 그 호환성 장치들을 모은다.

## API 버전 생애주기 — alpha → beta → GA

같은 리소스도 여러 API 버전을 가진다(예: `apps/v1beta1` → `apps/v1`,
[00-foundations/03](00-foundations/03-api-object-model.md)). 버전은 성숙 단계를 가진다:

| 단계 | 안정성 | 기본 |
|------|--------|------|
| **Alpha** (`v1alpha1`) | 실험적, 호환 미보장, 삭제 가능 | 보통 비활성(feature gate) |
| **Beta** (`v1beta1`) | 안정화 중, 마이그레이션 경로 제공 | 정책에 따라 |
| **GA/Stable** (`v1`) | 안정, 장기 지원 | 활성 |

[19-dra.md](19-dra.md)의 `resource.k8s.io`가 v1alpha3~v1로 공존하는 것이 이 생애주기의 실례다.

## 같은 객체, 여러 버전 — 저장 버전과 변환

여러 버전이 공존해도 **etcd에는 하나의 "저장 버전(storage version)"** 으로만 저장된다. apiserver가
요청/응답 시 클라이언트가 원하는 버전으로 **변환(conversion)** 한다:

- 변환 함수는 Scheme이 보유([00-foundations/03](00-foundations/03-api-object-model.md)).
- 저장 버전 추적/마이그레이션: `apiserver/pkg/storageversion`,
  controller-manager의 StorageVersionMigrator/GC 컨트롤러([11-controller-manager/01](11-controller-manager/01-controller-catalog.md)).
- CRD의 다중 버전은 conversion webhook으로 변환([10-apiserver/07](10-apiserver/07-crd-aggregation.md)).

그래서 클라이언트는 `v1beta1`로 만든 객체를 `v1`로 읽을 수 있다 — 내부적으론 같은 객체다.

## API deprecation 정책

API는 함부로 사라지지 않는다. Kubernetes의 **deprecation 정책**:

- GA API는 매우 오래(여러 마이너 릴리스) 유지된 뒤에야 제거된다.
- deprecated API를 쓰면 apiserver가 **경고 헤더**를 보낸다([10-apiserver/01](10-apiserver/01-request-pipeline.md)의
  `WithWarningRecorder`) — `kubectl`이 이를 사용자에게 표시.
- 제거 일정은 사전 공지되고 마이그레이션 경로가 제공된다.

이 덕분에 오래된 매니페스트/도구가 갑자기 깨지지 않는다.

## 컴포넌트 버전 스큐

클러스터 업그레이드는 한 번에 모든 컴포넌트를 같은 버전으로 올리지 않는다 — apiserver를 먼저, 그다음
컨트롤러/스케줄러, 마지막에 노드(kubelet) 식으로 **롤링**한다. 그동안 컴포넌트 버전이 어긋난다(skew).
Kubernetes는 **버전 스큐 정책**으로 허용 범위를 정한다:

- **apiserver가 가장 최신**이어야 한다(다른 컴포넌트는 apiserver보다 같거나 낮은 버전).
- kubelet은 apiserver보다 몇 마이너 낮은 것까지 허용(정해진 범위 내).
- HA에서 여러 apiserver의 버전도 제한된 범위만 어긋날 수 있다.

이를 코드로 뒷받침하는 것이 `component-base/version`(버전 정보)과
`component-base/compatibility`(`version.go` — 호환성 버전 관리)다. feature gate도 호환성 버전에 따라
기본값을 달리한다(`featuregate/feature_gate.go:104`, [00-foundations/07](00-foundations/07-component-base.md)) —
스큐 중에 "새 컴포넌트가 옛 클러스터를 깨뜨리는" 것을 막는다.

## emulated/min-compatibility 버전

최신 바이너리가 **옛 버전처럼 행동**하도록 강제할 수 있다(`--emulated-version`/`--min-compatibility-version`).
업그레이드 시 새 바이너리를 깔되 동작은 옛 버전으로 묶어 두었다가, 모든 컴포넌트가 올라간 뒤 새 동작을
켜는 안전한 전환을 가능케 한다.

## 정리 — 왜 이 모든 장치가 필요한가

```
무중단 업그레이드(롤링)  →  컴포넌트 버전 스큐 발생  →  스큐 정책 + 호환성 버전으로 안전
새 기능 도입            →  alpha/beta/GA + feature gate 로 점진 출시
API 진화               →  다중 버전 + 변환 + deprecation 정책 으로 옛 클라이언트 보호
```

이 모두가 "거대한 분산 시스템을 멈추지 않고 진화시킨다"는 목표를 떠받친다. 각 기능의 단계 전환 기준과
설계 배경은 KEP에 있다([90-enhancements](90-enhancements.md)).

## 더 읽을 곳
- [00-foundations/07](00-foundations/07-component-base.md) — feature gate / 호환성 버전
- [90-enhancements](90-enhancements.md) — 단계 전환 기준(KEP)
- [10-apiserver/07](10-apiserver/07-crd-aggregation.md) — CRD 버전 변환
