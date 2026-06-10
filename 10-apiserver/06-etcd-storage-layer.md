# 10.06 · etcd 저장 계층 (etcd3 store + cacher)

**근거**: `staging/src/k8s.io/apiserver/pkg/storage/{etcd3,cacher}/`

[05](05-registry-storage.md)의 `storage.Interface`가 실제 etcd와 만나는 곳이다. 두 겹으로 구현된다:
밑단의 **etcd3 store**(진짜 etcd 호출)와, 그 위의 **cacher**(watch 캐시).

## etcd3 store — 진짜 etcd와 말하는 층

`storage/etcd3/store.go:77`의 `store` 구조체가 `storage.Interface`를 etcd v3 클라이언트로 구현한다.

### 쓰기 — Create
`store.Create`(`store.go:269`)의 흐름:

1. 객체를 직렬화(보통 protobuf)하고, `PrepareObjectForStorage`로 resourceVersion 등을 정리
   (`storage/api_object_versioner.go:63`).
2. etcd에 **트랜잭션**으로 쓴다 — "이 키가 아직 없으면 PUT"(존재 검사 + 쓰기를 원자적으로).
3. etcd가 돌려준 **mod revision**을 객체의 `resourceVersion`으로 설정해 반환
   (`APIObjectVersioner.UpdateObject`, `:33`).

### resourceVersion = etcd revision
이것이 핵심 연결고리다. **객체의 `ObjectMeta.ResourceVersion`은 etcd의 전역 revision에서 온다**
(`APIObjectVersioner`, `api_object_versioner.go`). etcd는 모든 변경에 단조 증가하는 revision을 매기므로,
RV는 클러스터 전체에서 "이 시점"을 가리키는 논리 시계가 된다.

### 낙관적 동시성 — GuaranteedUpdate
갱신은 `GuaranteedUpdate`로 한다: 객체를 읽어 RV를 기억하고, 수정 함수를 적용한 뒤, etcd 트랜잭션에
"mod revision이 그대로일 때만 PUT" 조건을 건다. 그사이 누가 바꿔서 revision이 달라졌으면 트랜잭션이
실패하고, store가 다시 읽어 재시도한다. 이것이 "lost update" 없이 동시 수정을 안전하게 만든다.
사용자 입장에서는 `409 Conflict`로 나타날 수 있다.

### Watch
`store.Watch`(`store.go:965`)는 etcd watch를 열어 "주어진 키(접두사) 아래, 주어진 revision 이후의 변경"을
받아 Kubernetes `watch.Event`(Added/Modified/Deleted)로 변환한다(`watcher.go`, `event.go`).

## cacher — etcd 부하를 줄이는 watch 캐시

수천 개의 클라이언트가 같은 리소스를 watch하면, 매번 etcd에 watch를 거는 것은 부담이다. **cacher**
(`storage/cacher/cacher.go`)는 리소스마다 **etcd에 watch를 딱 하나만** 걸고, 그 결과를 메모리에 캐싱한
뒤 모든 클라이언트 watch를 그 캐시에서 팬아웃한다.

- 내부적으로 client-go의 Reflector/캐시 머시너리를 재사용한다(`cacher.go` 임포트에 `client-go/tools/cache`).
- **List**와 **Watch**의 상당수를 etcd 대신 메모리 캐시에서 응답한다.
- 클라이언트가 `resourceVersion`을 지정해 watch하면, cacher가 그 시점부터의 이벤트를 캐시 버퍼에서
  되짚어 보내준다.

> 즉 [00-foundations/04](../00-foundations/04-list-watch-informer.md)의 "클라이언트 쪽 Informer"와,
> 여기 apiserver 쪽 "서버 캐시(cacher)"는 같은 watch 머시너리를 양쪽에서 쓰는 셈이다. etcd의 watch
> 하나가 → apiserver cacher → 수많은 클라이언트 Informer로 부채꼴처럼 퍼진다.

## resourceVersion의 일생 (정리)

```
etcd revision (전역 단조 증가)
   │  store.Create/Update 시 객체에 박힘
   ▼
ObjectMeta.ResourceVersion
   │  watch 시 "이 RV 이후" 요청에 사용 (client-go Reflector)
   ▼
컴포넌트가 끊겼다 재연결해도 놓친 변경만 수신; 너무 오래되면 relist
```

## 주요 디렉토리

| 경로 | 책임 |
|------|------|
| `storage/etcd3/store.go` | etcd v3 클라이언트로 CRUD/watch 구현 |
| `storage/etcd3/watcher.go`, `event.go` | etcd watch → k8s 이벤트 변환 |
| `storage/etcd3/lease_manager.go` | TTL 객체(이벤트 등)용 etcd lease 재사용 |
| `storage/cacher/` | watch 캐시 팬아웃 |
| `storage/api_object_versioner.go` | RV ↔ etcd revision 변환 |

## 더 읽을 곳
- [20-etcd](../20-etcd/) — revision/MVCC/watch가 etcd 내부에서 어떻게 구현되나
- [00-foundations/04](../00-foundations/04-list-watch-informer.md) — 클라이언트 쪽 watch 소비
