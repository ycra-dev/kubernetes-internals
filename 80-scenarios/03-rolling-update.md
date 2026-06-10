# 80.03 · 시나리오 — 롤링 업데이트

`kubectl set image deployment/web nginx=nginx:1.26` 처럼 Deployment의 이미지를 바꿀 때, 무중단으로 새
버전이 배포되는 과정을 추적한다. 여러 컨트롤러가 **공유 객체를 매개로** 협력하는 대표 사례다.

## 전체 흐름

```
[1] Deployment.spec.template 변경 → apiserver → etcd
[2] Deployment 컨트롤러: 새 템플릿용 ReplicaSet(RS-new) 생성, RS-old 유지
[3] 점진적 전환 루프:
       RS-new.replicas 를 조금 ↑ (maxSurge 한도)
       새 Pod 들이 Ready 가 되면
       RS-old.replicas 를 조금 ↓ (maxUnavailable 한도)
       ... 반복 until RS-new=원하는수, RS-old=0
[4] 각 RS 변경마다 ReplicaSet 컨트롤러가 Pod 생성/삭제
[5] Pod Ready/종료에 따라 EndpointSlice·데이터플레인 갱신 → 트래픽 전환
```

## 단계별 추적

### [1]~[2] 새 ReplicaSet 등장
Deployment의 `spec.template`이 바뀌면 그 변경이 etcd에 저장되고, Deployment 컨트롤러가 watch로 받는다.
`syncDeployment`가 새 템플릿 해시에 해당하는 ReplicaSet이 없음을 보고 **RS-new를 만든다**(replicas=0
에서 시작), RS-old는 그대로 둔다([11-controller-manager/03](../11-controller-manager/03-deployment-replicaset.md)).

### [3] 점진 전환 — Deployment는 RS의 replicas만 조정
`rolloutRolling`이 핵심이다([11-controller-manager/03](../11-controller-manager/03-deployment-replicaset.md)):

- **maxSurge**: 원하는 수보다 몇 개까지 더 띄울 수 있나 → RS-new를 그만큼 올린다.
- **maxUnavailable**: 동시에 몇 개까지 부족해도 되나 → 새 Pod가 Ready가 되면 RS-old를 그만큼 내린다.

Deployment 컨트롤러는 **직접 Pod를 만들지 않는다.** RS의 `spec.replicas` 숫자만 조정한다.

### [4] ReplicaSet 컨트롤러가 실제 Pod를 움직인다
RS-new.replicas가 올라가면, ReplicaSet 컨트롤러의 `manageReplicas`가 `diff = 현재 - 원하는`을 보고
부족분 Pod를 만든다([00-foundations/05](../00-foundations/05-controller-pattern.md)). RS-old.replicas가
내려가면 초과분 Pod를 삭제한다. 새 Pod는 스케줄러→kubelet을 거쳐 뜬다([01-pod-create.md](01-pod-create.md)).

> 두 컨트롤러는 서로를 호출하지 않는다. Deployment 컨트롤러가 RS 객체를 바꾸면, ReplicaSet 컨트롤러가
> 그 변화를 watch해 반응한다 — **공유 객체(ReplicaSet)를 통한 협력**.

### [5] 트래픽이 무중단으로 넘어가는 이유
이 전환이 다운타임 없이 되는 것은 네트워크 계층 덕분이다:

- 새 Pod는 **readiness probe를 통과해야** EndpointSlice에 ready로 들어간다
  ([13-kubelet/01](../13-kubelet/01-pod-lifecycle.md)). 그전엔 트래픽을 안 받는다.
- 종료되는 옛 Pod는 EndpointSlice에서 먼저 빠져 신규 연결을 안 받는다
  ([40-networking/02](../40-networking/02-service-discovery.md)).
- kube-proxy/cilium이 이 변화를 watch해 데이터플레인을 갱신하므로, 트래픽이 항상 ready Pod로만 간다
  ([80-scenarios/02](02-service-request.md)).

## 롤백

문제가 생기면 `kubectl rollout undo`로 되돌린다. Deployment는 옛 RS들을 `revisionHistoryLimit`까지
**0-replica로 보존**하므로, 롤백은 "이전 RS의 템플릿을 다시 적용"하는 같은 전환 루프일 뿐이다
([11-controller-manager/03](../11-controller-manager/03-deployment-replicaset.md)).

## 핵심 관찰

- **위임 계층**(Deployment → ReplicaSet → Pod)이 무중단 전환을 가능케 한다. 각 계층은 한 가지만 한다.
- 무중단의 핵심은 **readiness + EndpointSlice + 데이터플레인 watch**의 협력이다.

## 더 읽을 곳
- [11-controller-manager/03](../11-controller-manager/03-deployment-replicaset.md) — Deployment/RS 코드
- [04-scale-and-failure.md](04-scale-and-failure.md) — 스케일링과 장애
