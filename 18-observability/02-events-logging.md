# 18.02 · 이벤트와 로깅

**근거**: `kubernetes/staging/src/k8s.io/api/core/v1/types.go`(Event `:7570`),
`api/events/v1/types.go`(`:34`), `staging/.../component-base/logs/`(klog/v2)

메트릭이 "수치 추세"라면, 이벤트와 로그는 "무슨 일이 있었나"의 서사다.

## Event — 객체에 일어난 사건

**Event**는 다른 객체에 무슨 일이 있었는지를 기록하는 API 객체다(`core/v1/types.go:7570`, 신형
`events/v1/types.go:34`). 담기는 것:

- **involvedObject / regarding**: 어떤 객체에 대한 사건인가(예: Pod nginx).
- **reason / note(message)**: 무슨 일(예: `Scheduled`, `Pulled`, `Failed`, `Evicted`)과 설명.
- **type**: Normal / Warning.
- **count / first·lastTimestamp**: 같은 사건이 반복되면 묶어서 횟수로.

### 누가 만드나
컴포넌트들이 reconcile/처리 중 **EventRecorder**로 이벤트를 낸다. 예:
- 스케줄러: Pod를 노드에 배정하면 `Scheduled`([12-scheduler](../12-scheduler/)).
- kubelet: 이미지 pull `Pulling`/`Pulled`, 실패 `Failed`, 축출 `Evicted`([13-kubelet/04](../13-kubelet/04-eviction.md)).
- 컨트롤러: 생성/실패 등.

`kubectl describe pod nginx`의 하단 Events 섹션이나 `kubectl get events`가 이것이다. **디버깅의 1차
단서** — "왜 Pod가 안 뜨나"의 답이 보통 여기 있다.

### Event는 일시적이다
Event도 etcd에 저장되지만, 무한히 쌓이면 부담이므로 **TTL**로 자동 삭제된다(apiserver의 event용 lease
재사용, [10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md), [20-etcd/05](../20-etcd/05-lease-watch.md)).
그래서 오래된 사건은 사라진다 — 장기 보존이 필요하면 외부로 내보내야 한다.

## 로깅 — klog

컴포넌트(apiserver/kubelet 등)는 **klog/v2** 기반으로 로그를 남긴다(`component-base/logs/logs.go:33`).
공통 관례:

- **로그 레벨** `--v=<n>`: 숫자가 클수록 상세(코드의 `klog.V(2).Info(...)` 등). 운영 시 낮게, 디버깅 시
  높게.
- **구조화 로깅**: 키-값 형태(`klog.InfoS("msg", "key", val)`)로 기계 파싱이 쉽다. cilium 등에서 본
  `logger.V(2).Info(..., "replicaSet", ...)`이 그 예([00-foundations/05](../00-foundations/05-controller-pattern.md)).
- `component-base/logs`가 모든 컴포넌트의 로그 플래그/포맷을 통일한다([00-foundations/07](../00-foundations/07-component-base.md)).

## 컨테이너 로그

워크로드(컨테이너)의 로그는 다른 경로다:

```
컨테이너 stdout/stderr
   │  런타임(containerd)이 노드 디스크에 기록
   ▼
kubelet 이 CRI 로 읽어 제공 → kubectl logs
   │
(클러스터 차원 수집은) 로그 에이전트(Fluentd/Fluent Bit 등, 별도)가
노드 로그를 수집해 외부 저장소로
```

`kubectl logs pod`는 kubelet이 런타임에서 읽어 스트리밍하는 것이다([13-kubelet/02](../13-kubelet/02-cri.md)의
CRI). 노드가 사라지면 그 로그도 사라지므로, 영속 보관은 외부 로그 수집기가 담당한다.

## 세 축을 함께 쓰기 (디버깅 흐름)

```
1. 메트릭: "에러율이 올랐다" → 어디가 문제인지 범위 좁히기
2. 이벤트: kubectl describe → "Failed: ImagePullBackOff" 같은 사건 확인
3. 로그:   kubectl logs / 컴포넌트 로그 → 구체적 원인
```

## 더 읽을 곳
- [01-metrics-monitoring.md](01-metrics-monitoring.md) — 수치 축
- [13-kubelet/01](../13-kubelet/01-pod-lifecycle.md) — 이벤트를 내는 주요 출처
