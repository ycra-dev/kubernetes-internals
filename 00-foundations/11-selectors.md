# 00.11 · 라벨과 셀렉터

**근거**: `kubernetes/staging/src/k8s.io/apimachinery/pkg/labels/`(`selector.go`),
`kubernetes/staging/src/k8s.io/apimachinery/pkg/fields/`(`requirements.go`)

[03-api-object-model.md](03-api-object-model.md)에서 ObjectMeta의 `Labels`를 봤다. 이 문서는 그 라벨로
객체를 **선택(select)** 하는 메커니즘을 본다. 셀렉터는 Kubernetes의 **느슨한 결합(loose coupling)** 을
가능케 하는 핵심 — Service가 어떤 Pod를 백엔드로 삼을지, ReplicaSet이 어떤 Pod를 자기 것으로 셀지 모두
셀렉터로 정해진다.

## 두 종류의 셀렉터

| 셀렉터 | 대상 | 패키지 |
|--------|------|--------|
| **Label selector** | 객체의 `metadata.labels` | `apimachinery/pkg/labels/` |
| **Field selector** | 객체의 특정 필드(`spec.nodeName`, `status.phase` 등) | `apimachinery/pkg/fields/` |

## Label Selector — 라벨로 묶기

`labels/selector.go`. 라벨(키-값)으로 객체 집합을 선택한다. 두 형태:

- **equality 기반**: `app=web,tier=frontend` (모두 만족).
- **set 기반**: `app in (web,api)`, `tier notin (db)`, `env`(키 존재), `!debug`(키 부재).

이것이 Kubernetes의 연결 방식이다:

```
Service { selector: app=web }  ──선택──►  Pod{labels: app=web} × N
ReplicaSet { selector: app=web } ──소유──►  같은 라벨 Pod
NetworkPolicy { podSelector: app=web } ──적용──► 그 Pod들 (42-cilium/03)
```

라벨이 같으면 자동으로 묶인다 — Service는 Pod 이름을 몰라도 되고, Pod가 늘고 줄어도 셀렉터가 알아서
따라간다([40-networking/02](../40-networking/02-service-discovery.md)). 이 **간접 결합**이 선언형 모델의
유연성을 만든다.

## Field Selector — 필드로 거르기

`fields/requirements.go`. 라벨이 아닌 **객체 필드**로 선택한다. 대표 용도:

- `spec.nodeName=node-1`: 그 노드의 Pod만. **kubelet이 자기 노드 Pod만 watch**할 때 쓴다
  ([13-kubelet/README](../13-kubelet/README.md)).
- `status.phase=Running`: 실행 중 Pod만.
- `metadata.namespace=x`.

단, **모든 필드가 셀렉터로 지원되진 않는다** — 리소스마다 인덱싱된 필드만 가능하다. apiserver가 그 필드로
효율적으로 거를 수 있어야 하기 때문(임의 필드 셀렉터는 전체 스캔이라 막혀 있다).

## List/Watch에서의 셀렉터

셀렉터는 List/Watch 요청에 실린다([00-foundations/04](04-list-watch-informer.md)):

```
GET /api/v1/pods?labelSelector=app=web&fieldSelector=spec.nodeName=node-1
```

- **서버 측 필터링**: apiserver/cacher가 셀렉터로 걸러 매칭 객체만 보낸다 — 네트워크/클라이언트 부하
  감소([10-apiserver/12](../10-apiserver/12-watch-cache-internals.md)).
- cacher의 인덱스([10-apiserver/12](../10-apiserver/12-watch-cache-internals.md))나 Informer의 Indexer
  ([00-foundations/04](04-list-watch-informer.md))가 자주 쓰는 셀렉터를 빠르게 처리한다.

## 컨트롤러가 셀렉터를 쓰는 법

ReplicaSet 컨트롤러([00-foundations/05](05-controller-pattern.md))는 자기 셀렉터로 "내가 책임질 Pod"를
찾는다. 셀렉터에 맞지만 ownerReference 없는 Pod를 **입양**하고, 안 맞게 된 Pod를 **포기**한다
([11-controller-manager/02](../11-controller-manager/02-common-machinery.md)의 ControllerRefManager). 즉
셀렉터가 "소유 범위"를 정한다.

> 주의: ReplicaSet/Deployment의 `selector`는 **생성 후 불변**이다(전략이 강제,
> [10-apiserver/13](../10-apiserver/13-strategy-status.md)) — 셀렉터를 바꾸면 소유 관계가 통째로 흔들리기
> 때문.

## 더 읽을 곳
- [03-api-object-model.md](03-api-object-model.md) — 라벨이 사는 ObjectMeta
- [40-networking/02](../40-networking/02-service-discovery.md) — Service 셀렉터
- [05-controller-pattern.md](05-controller-pattern.md) — 컨트롤러의 소유 셀렉터
