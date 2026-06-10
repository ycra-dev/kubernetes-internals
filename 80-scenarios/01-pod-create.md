# 80.01 · 시나리오 — 파드 생성 전 과정

`kubectl run nginx --image=nginx` 한 줄이 노드에서 도는 컨테이너가 되기까지, 앞 문서들의 메커니즘이
어떻게 한 사건으로 꿰이는지 시간순으로 추적한다. 각 단계 옆 `→`는 상세 문서다.

## 전체 시퀀스

```
[1] kubectl ─POST /api/v1/.../pods─► kube-apiserver
[2] 핸들러 체인: 인증 → 인가 → (APF) 
[3] REST 핸들러: 디코딩 → 어드미션(mutating→validating) → 검증
[4] storage: etcd 에 Pod 기록 (spec 채워짐, nodeName="")
[5] etcd: Raft commit → MVCC revision 부여 → watch 통지
[6] scheduler: nodeName 없는 Pod 수신 → 필터/점수 → Binding (nodeName 설정)
[7] kubelet(해당 노드): 자기 Pod 수신 → SyncPod
[8]   sandbox 생성 → CNI(cilium) 네트워크 → 이미지 pull → 컨테이너 시작 (CRI→containerd→runc)
[9] kubelet: Pod.status=Running 보고
[10] kube-proxy/cilium: (Service에 속하면) EndpointSlice 갱신 반영
```

## 단계별 추적

### [1]~[4] apiserver 진입
`kubectl`이 kubeconfig의 자격증명으로 apiserver에 POST한다([15-cli](../15-cli/)). 요청은 핸들러
체인을 지난다 — **인증**(누구), **인가**(RBAC로 pods create 권한 확인), APF
→ [10-apiserver/01](../10-apiserver/01-request-pipeline.md). REST 핸들러에서 **어드미션**이
ServiceAccount 토큰 볼륨 주입 등 변형을 하고 검증한다([10-apiserver/04](../10-apiserver/04-admission.md)).
이 시점 Pod에는 `spec.nodeName`이 **비어 있다**.

### [4]~[5] etcd 저장
storage 계층이 Pod를 etcd에 쓴다([10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md)).
etcd는 이를 Raft로 commit하고([20-etcd/01](../20-etcd/01-raft.md)) MVCC revision을 매긴다
([20-etcd/03](../20-etcd/03-mvcc.md)). 그 revision이 Pod의 `resourceVersion`이 된다. 저장 직후
**watch 통지**가 apiserver cacher를 통해 구독자에게 퍼진다([20-etcd/05](../20-etcd/05-lease-watch.md)).

### [6] 스케줄링
scheduler의 Informer가 "nodeName 없는 Pod"를 watch로 받아 activeQ에 넣는다
([12-scheduler/03](../12-scheduler/03-queue-eventhandlers.md)). 스케줄링 사이클이 **필터**(자원/affinity/
taint로 후보 노드 거름)와 **점수**(가장 좋은 노드 선택)를 돌리고
([12-scheduler/01](../12-scheduler/01-framework.md)), **Binding**을 apiserver에 생성한다 = Pod의
`spec.nodeName`이 채워진다. (자리가 없으면 preemption, [12-scheduler/04](../12-scheduler/04-preemption-extender.md).)

### [7]~[8] kubelet이 띄운다
이제 Pod에 nodeName이 있으므로, **그 노드의 kubelet**이 watch로 받는다(Node authorizer가 자기 노드 것만
허용, [10-apiserver/03](../10-apiserver/03-authorization.md)). SyncLoop이 SyncPod를 부른다
([13-kubelet/01](../13-kubelet/01-pod-lifecycle.md)):

1. **sandbox 생성**(CRI `RunPodSandbox`) → 이때 **CNI(cilium)** 가 네트워크 네임스페이스에 IP를 붙인다
   ([40-networking/01](../40-networking/01-cni.md), [42-cilium/02](../42-cilium/02-agent.md)).
2. **이미지 pull**(CRI `PullImage`) → containerd가 레이어를 content store에 받아 snapshot으로 unpack
   ([30-containerd/02](../30-containerd/02-content-snapshots.md)).
3. **컨테이너 생성/시작**(CRI `CreateContainer`/`StartContainer`) → containerd가 shim을 통해 runc로
   프로세스를 띄운다([30-containerd/03](../30-containerd/03-runtime-shim-oci.md)).

볼륨이 있으면 컨테이너 시작 전에 volume manager가 mount한다([13-kubelet/05](../13-kubelet/05-volume-manager.md)).

### [9]~[10] 상태 보고와 네트워크 반영
kubelet이 Pod.status를 Running으로 apiserver에 보고한다([13-kubelet/01](../13-kubelet/01-pod-lifecycle.md)).
이 Pod가 Service 셀렉터에 맞고 readiness를 통과하면, EndpointSlice 컨트롤러가 EndpointSlice에 추가하고
([40-networking/02](../40-networking/02-service-discovery.md)), kube-proxy/cilium이 그 변경을 watch해
데이터플레인에 반영한다([14-kube-proxy](../14-kube-proxy/), [42-cilium/04](../42-cilium/04-kube-proxy-replacement.md)).

## 핵심 관찰

- **누구도 서로를 직접 호출하지 않았다.** kubectl→apiserver만 직접이고, 나머지는 전부 **etcd에 저장된
  Pod 객체의 변화를 watch**해 반응했다([00-foundations/02](../00-foundations/02-architecture.md)의 공유
  상태 모델).
- **nodeName 한 필드**가 스케줄러→kubelet의 바통이었다. 그 필드가 비면 스케줄러의 일, 채워지면
  kubelet의 일.
- 각 단계는 **level-triggered** — 어디서 끊겨도 재시작 후 현재 상태를 보고 이어간다.

## 더 읽을 곳
- [02-service-request.md](02-service-request.md) — 이 Pod에 트래픽이 닿는 과정
