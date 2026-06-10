# 10.05 · Registry와 저장 전략

**근거**: `staging/src/k8s.io/apiserver/pkg/registry/`

어드미션을 통과한 객체는 **registry** 계층으로 넘어간다. registry는 "리소스별 비즈니스 규칙(전략)"과
"제네릭 저장 로직"을 잇는 층이다. apiserver가 수십 종의 리소스를 거의 같은 코드로 다룰 수 있는 비결이다.

## generic registry Store

대부분의 리소스는 `registry/generic/registry/store.go:101`의 `Store`를 재사용한다. 이 구조체는 함수
필드들로 리소스별 동작을 주입받는 "틀"이다:

| 필드 | 역할 |
|------|------|
| `NewFunc` / `NewListFunc` | 빈 객체/리스트 생성(역직렬화 대상 타입) |
| `KeyFunc` / `KeyRootFunc` | 객체 ↔ etcd 키 매핑 (예: `/pods/<ns>/<name>`) |
| `CreateStrategy` | 생성 시 기본값/검증/필드 정규화 |
| `UpdateStrategy` | 갱신 시 규칙(불변 필드 보호 등) |
| `DeleteStrategy` | 삭제 시 규칙(finalizer/graceful) |
| `Storage` | 실제 저장 백엔드(`storage.Interface`) |

즉 "Pod를 저장하라"는 요청은 PodStorage가 채운 전략으로 검증·정규화된 뒤, 동일한 `Store.Create`
경로로 흐른다.

## 전략(Strategy) — 리소스별 규칙

전략은 `registry/rest/`의 인터페이스로 정의된다(`create.go`, `update.go`, `delete.go`). 핵심 책임:

- **기본값(defaulting)**: 비어 있는 필드에 기본값.
- **검증(validation)**: 객체가 유효한지(어드미션과 별개의, 타입 고유 검증).
- **정규화/필드 보호**: 사용자가 못 바꾸는 필드(예: status 분리, immutable 필드) 처리.
- **이름 생성**: `generateName` 처리([00-foundations/03](../00-foundations/03-api-object-model.md)).

각 리소스 패키지(예: `kubernetes/pkg/registry/core/pod/`)가 자기 전략을 구현해 generic Store에 끼운다.

## storage.Interface — 백엔드 추상화

registry가 호출하는 저장 인터페이스는 `storage/interfaces.go`에 있다. 백엔드(etcd)를 추상화한
연산들(`:176`~`:243`):

```go
Create(ctx, key, obj, out, ttl)                 // :176
Delete(ctx, key, ...)                           // :183
Watch(ctx, key, opts) (watch.Interface, error)  // :194
Get(ctx, key, opts, objPtr)                     // :201
GetList(ctx, key, opts, listObj)                // :209
GuaranteedUpdate(ctx, key, ...)                 // :243  ← 낙관적 동시성 갱신
```

`GuaranteedUpdate`가 중요하다: "객체를 읽고 → 수정 함수를 적용하고 → resourceVersion이 그대로면 저장,
바뀌었으면 재시도"하는 read-modify-write 루프다. 이것이 낙관적 동시성의 핵심
([06](06-etcd-storage-layer.md)).

## 이 인터페이스의 두 구현

`storage.Interface`는 두 겹으로 구현된다:

1. **etcd3 store** (`storage/etcd3/store.go`) — 실제로 etcd와 말하는 구현.
2. **cacher** (`storage/cacher/`) — etcd3 위에 watch 캐시를 얹은 래퍼. 읽기/watch를 메모리에서 처리해
   etcd 부하를 줄인다.

둘 다 [06](06-etcd-storage-layer.md)에서 다룬다.

## 더 읽을 곳
- [06-etcd-storage-layer.md](06-etcd-storage-layer.md) — storage.Interface의 실제 etcd 구현
- [20-etcd](../20-etcd/) — etcd 자체
