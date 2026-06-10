# 13.06 · 노드 등록·하트비트·인증서

**근거**: `kubernetes/pkg/kubelet/`(`kubelet_node_status.go`), `kubernetes/pkg/kubelet/certificate/`

kubelet은 부팅 시 자기를 클러스터에 **등록**하고, 살아 있음을 **하트비트**로 알리며, apiserver와 mTLS로
통신하기 위한 **인증서**를 관리한다.

## 노드 등록

부팅 시 kubelet은 자기 노드 정보를 담은 **Node 객체**를 apiserver에 만든다(또는 갱신)
(`kubelet_node_status.go`): 노드 이름, 라벨, capacity(CPU/메모리/장치), 주소, 운영체제 등. 스케줄러는
이 Node 객체의 capacity/allocatable을 보고 스케줄을 결정한다([12-scheduler](../12-scheduler/)).

## 하트비트 — 두 가지 방식

NodeLifecycle 컨트롤러([11-controller-manager/05](../11-controller-manager/05-node-lifecycle.md))가
"이 노드가 살아 있나"를 판단하려면 신호가 필요하다. kubelet은 두 가지를 보낸다:

1. **Node status 업데이트**: 노드의 condition(Ready 등)과 상태를 주기적으로 갱신. 정보량이 많다.
2. **Lease 갱신**: `coordination.k8s.io` 의 **Lease** 객체를 짧은 주기로 갱신. 가볍다(작은 객체).

대규모 클러스터에서 무거운 status를 자주 보내면 apiserver/etcd 부하가 크므로, 평소엔 가벼운 **Lease**로
생존을 알리고 status는 변화가 있을 때/드물게 보낸다. NodeLifecycle 컨트롤러는 Lease 갱신이 끊기면
노드를 NotReady/Unreachable로 보고 taint한다.

## 인증서 — TLS bootstrap과 자동 갱신

kubelet은 apiserver와 mTLS로 통신하고, 자기 신원은 `system:node:<노드이름>`
+ `system:nodes` 그룹이다(이게 Node authorizer의 입력 →
[10-apiserver/03](../10-apiserver/03-authorization.md)).

- **TLS bootstrap**: 처음엔 부트스트랩 토큰으로 **CertificateSigningRequest(CSR)** 를 제출해 클라이언트
  인증서를 발급받는다. CSR 승인/서명은 controller-manager의 certificate 컨트롤러
  ([11-controller-manager/01](../11-controller-manager/01-controller-catalog.md))가 한다.
- **자동 갱신**: `pkg/kubelet/certificate/`가 인증서 만료 전에 새 CSR로 갱신(rotation)한다.

## Node authorizer가 강제하는 격리

`system:node:<name>` 신원은 Node authorizer에 의해 **자기 노드에 관련된 객체만** 접근할 수 있다
([10-apiserver/03](../10-apiserver/03-authorization.md)). 즉 한 노드가 탈취돼도 다른 노드의 Pod가 쓰는
Secret 등에 접근하지 못한다. NodeRestriction 어드미션
([10-apiserver/04](../10-apiserver/04-admission.md))은 한 발 더 나아가, kubelet이 자기 Node/Pod status
외의 것을 바꾸지 못하게 한다.

## 더 읽을 곳
- [11-controller-manager/05](../11-controller-manager/05-node-lifecycle.md) — 하트비트를 받는 쪽
- [10-apiserver/03](../10-apiserver/03-authorization.md) — Node authorizer
