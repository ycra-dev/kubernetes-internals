# 00 · Foundations — 횡단 기반 개념

이 폴더는 개별 컴포넌트를 읽기 전에 알아야 하는 **공통 작동 원리**를 다룬다. Kubernetes의 거의 모든
컴포넌트(apiserver를 제외한)는 동일한 패턴 위에서 돈다: *API 객체를 정의하고 → list/watch로 캐시하고 →
원하는 상태로 수렴시키는 컨트롤러를 돌린다.* 이 토대를 먼저 잡으면 나머지 문서가 빠르게 읽힌다.

이 파트의 근거 코드는 주로 두 라이브러리 레포에 있다:

- `apimachinery` — API 객체의 타입 시스템 (GVK, Scheme, 직렬화)
- `client-go` — 그 객체를 list/watch 하고 캐시·재시도하는 클라이언트 머시너리

> 원본은 `kubernetes/staging/src/k8s.io/{apimachinery,client-go}/` 에 있다(자동 미러되어
> 독립 레포 `apimachinery/`, `client-go/`로 배포된다). 본 문서는 원본 staging 경로를 인용한다.

## 문서

| # | 문서 | 내용 |
|---|------|------|
| 01 | [overview.md](01-overview.md) | 11개 레포 구성, core vs 독립, staging 미러 관계 |
| 02 | [architecture.md](02-architecture.md) | 3계층 구조, 요청 흐름, 데이터 흐름 큰 그림 |
| 03 | [api-object-model.md](03-api-object-model.md) | GVK/GVR, Scheme, RESTMapper, 객체 공통 구조, 직렬화 |
| 04 | [list-watch-informer.md](04-list-watch-informer.md) | Reflector→DeltaFIFO→Informer→Indexer→Workqueue |
| 05 | [controller-pattern.md](05-controller-pattern.md) | reconcile 루프, ownerReference, finalizer, GC |
| 06 | [leader-election.md](06-leader-election.md) | lease 기반 고가용성 |
| 07 | [component-base.md](07-component-base.md) | 공용 기반(featuregate/logs/metrics/cli) |
| 08 | [client-go-clients.md](08-client-go-clients.md) | Clientset/Dynamic/Discovery/RESTClient/transport |
| 09 | [conversion.md](09-conversion.md) | 버전 변환(hub-and-spoke)·기본값 |
| 10 | [serialization.md](10-serialization.md) | 직렬화(JSON/protobuf/CBOR)·콘텐츠 협상 |
| 11 | [selectors.md](11-selectors.md) | 라벨/필드 셀렉터(느슨한 결합) |

## 한눈에 보는 핵심 사상

- **선언형(declarative)**: 사용자는 "원하는 상태(spec)"를 적고, 시스템이 "현재 상태(status)"를
  거기에 맞춰 끊임없이 수렴시킨다. 명령형 "이걸 해라"가 아니라 "이렇게 되어 있어라"이다.
- **단일 진실 공급원**: 모든 상태는 etcd에 있고, 오직 apiserver만 etcd에 접근한다. 다른 모든
  컴포넌트는 apiserver를 통해 list/watch 한다.
- **수준 기반(level-based) 제어**: 컨트롤러는 "변경 이벤트"가 아니라 "현재 전체 상태"를 보고 동작한다.
  이벤트를 놓쳐도 다음 동기화에서 복구된다(edge-triggered가 아니라 level-triggered).
