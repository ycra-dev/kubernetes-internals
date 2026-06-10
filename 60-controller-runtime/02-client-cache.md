# 60.02 · Client와 Cache

**근거**: `controller-runtime/pkg/client/`, `controller-runtime/pkg/cache/`

controller-runtime의 `Client`는 읽기와 쓰기를 영리하게 분리한다: **읽기는 로컬 캐시에서(빠름),
쓰기는 apiserver로(권위)**. 개발자는 같은 `Get/List/Create/Update` 인터페이스만 쓰면 된다.

## 분리된 읽기/쓰기

기본 Manager 클라이언트(`mgr.GetClient()`)는:

- **Get / List** → **Cache**(`pkg/cache/`)에서 읽는다. Cache는 informer 기반 로컬 저장소다
  ([00-foundations/04](../00-foundations/04-list-watch-informer.md)). apiserver/etcd를 매번 때리지 않는다.
- **Create / Update / Delete / Patch / Status().Update** → **apiserver**로 직접 보낸다.

이 분리 덕분에 컨트롤러가 reconcile마다 자유롭게 `Get`해도 부하가 작다(전부 캐시 히트).

## 캐시의 함정과 대응

캐시는 **결국 일관(eventually consistent)** 하다([00-foundations/04](../00-foundations/04-list-watch-informer.md)).
방금 Create한 객체가 즉시 캐시에 안 보일 수 있다. 그래서:

- level-triggered로 설계하면(다음 reconcile에서 다시 확인) 자연히 수렴한다.
- 꼭 최신이 필요하면 캐시를 우회하는 직접 클라이언트(APIReader)를 쓸 수 있다.

이는 [11-controller-manager/02](../11-controller-manager/02-common-machinery.md)의 expectations가
풀던 문제와 같은 근원(캐시 지연)이다.

## 인덱스

캐시에 **필드 인덱스**를 추가하면(`mgr.GetFieldIndexer().IndexField(...)`), 특정 필드로 List를 O(인덱스)로
조회할 수 있다([00-foundations/04](../00-foundations/04-list-watch-informer.md)의 Indexer). 예: "이 노드의
Pod들"을 spec.nodeName 인덱스로.

## Scheme — 타입 등록

Client/Cache가 객체를 (역)직렬화하려면 그 타입이 Scheme에 등록돼야 한다
([00-foundations/03](../00-foundations/03-api-object-model.md)). 오퍼레이터는 자기 CRD 타입과 의존하는
빌트인 타입을 Manager의 Scheme에 등록한다(`pkg/scheme`). kubebuilder가 이 등록 코드를 생성한다
([61-kubebuilder](../61-kubebuilder.md)).

## Status 서브리소스

`Status().Update()`는 객체의 status만 갱신하는 별도 경로다. spec(사용자 소유)과 status(컨트롤러 소유)를
분리해 갱신하므로, 컨트롤러가 status를 쓰는 동안 사용자의 spec 변경과 충돌하지 않는다
([00-foundations/03](../00-foundations/03-api-object-model.md)의 spec/status 분리).

## 더 읽을 곳
- [01-manager-reconciler.md](01-manager-reconciler.md) — 이 클라이언트를 제공하는 Manager
- [00-foundations/04](../00-foundations/04-list-watch-informer.md) — 캐시의 기반
