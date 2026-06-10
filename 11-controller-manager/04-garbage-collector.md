# 11.04 · 가비지 컬렉터

**근거**: `kubernetes/pkg/controller/garbagecollector/`

GC 컨트롤러는 **ownerReference 그래프**를 추적해, owner가 사라진 dependent를 정리한다. "Deployment를
지우면 그 Pod도 사라진다"가 이 컨트롤러 덕분이다. 개념적 토대는
[00-foundations/05](../00-foundations/05-controller-pattern.md)에 있고, 여기선 구현을 본다.

## 전체 그래프를 watch로 구축

GC는 한 종류 리소스만 보지 않는다. **클러스터의 거의 모든 리소스**를 watch하며 owner→dependent
관계 그래프를 메모리에 만든다:

- `graph_builder.go` — 모든 리소스를 모니터링해 객체가 생기고/바뀌고/지워질 때 그래프 노드(`node`,
  `graph.go`)를 갱신. 각 노드는 자기 owner들과 dependent들을 안다.
- 디스커버리로 "현재 클러스터에 어떤 리소스 종류가 있는지" 알아내 모니터를 동적으로 추가한다(CRD 포함).

## 삭제 정책 세 가지

owner가 삭제될 때 dependent를 어떻게 할지 세 정책이 있고, finalizer로 구분된다
(`garbagecollector.go`):

| 정책 | 동작 | 표식 |
|------|------|------|
| **Background** | owner를 즉시 지우고, GC가 dependent를 비동기로 정리 | (기본) |
| **Foreground** | dependent를 먼저 다 지운 뒤 owner를 지움 | `metav1.FinalizerDeleteDependents` (`garbagecollector.go:668`) |
| **Orphan** | owner만 지우고 dependent는 ownerReference만 떼어 고아로 | `metav1.FinalizerOrphanDependents` (`:769`) |

`removeFinalizer`(`:668`, `:769`)가 정리 완료 후 finalizer를 떼어 실제 삭제가 진행되게 한다.

## 동작 루프

1. **attemptToDelete 큐**: owner가 없어진(또는 없어질) 노드를 찾아 dependent 삭제를 시도.
2. **attemptToOrphan 큐**: Orphan 정책이면 dependent의 ownerReference를 제거.
3. Foreground의 경우, dependent가 다 지워졌는지 그래프로 확인한 뒤 owner의 finalizer를 제거.

GC도 결국 reconcile 루프다: "그래프 상태"를 보고 "있어선 안 되는 dependent"를 지우는 방향으로 수렴한다.

## 왜 별도 컨트롤러인가

연쇄 삭제를 각 컨트롤러가 제각각 구현하면 일관성이 깨진다. ownerReference라는 **공통 메타데이터**
([00-foundations/03](../00-foundations/03-api-object-model.md))와 그것을 해석하는 **단일 GC**로
연쇄 삭제 규칙을 한 곳에 모았다. CRD가 만든 커스텀 리소스도 ownerReference만 박으면 자동으로 GC 대상이
된다.

## 더 읽을 곳
- [00-foundations/05](../00-foundations/05-controller-pattern.md) — ownerReference/finalizer 개념
