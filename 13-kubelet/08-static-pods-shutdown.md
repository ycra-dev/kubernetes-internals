# 13.08 · Static Pod와 Graceful Node Shutdown

**근거**: `kubernetes/pkg/kubelet/`(static pod 경로, `config/`),
`kubernetes/pkg/kubelet/nodeshutdown/`

kubelet이 apiserver 없이도 Pod를 띄우는 경우(static pod)와, 노드가 꺼질 때 Pod를 안전히 정리하는 경우
(graceful shutdown)를 다룬다. 둘 다 SyncLoop([01-pod-lifecycle.md](01-pod-lifecycle.md))의 특수 입력이다.

## Static Pod — apiserver 없이 뜨는 Pod

보통 Pod는 apiserver watch로 받는다. **static pod**는 다르다 — kubelet이 **로컬 디렉토리의 매니페스트**
(보통 `/etc/kubernetes/manifests/`)를 직접 읽어 띄운다. SyncLoop의 `configCh`
([01-pod-lifecycle.md](01-pod-lifecycle.md))가 받는 입력 중 하나(`pkg/kubelet/config/`의 파일 소스)다.

특징:
- **apiserver가 없어도 동작** — 그래서 컨트롤 플레인 부트스트랩의 닭-달걀 문제를 푼다
  ([15-cli/01](../15-cli/01-kubeadm.md)의 kubeadm이 apiserver/etcd를 static pod로 띄움).
- 스케줄러를 거치지 않는다 — 그 노드에 고정.
- kubelet이 apiserver에 **mirror Pod**를 만들어, `kubectl get pods`에 보이게 한다(읽기 전용 반영).

```
/etc/kubernetes/manifests/kube-apiserver.yaml
   │  kubelet이 파일 watch
   ▼
컨테이너 기동 (CRI→containerd)  +  apiserver에 mirror Pod 생성(보이기용)
```

## Graceful Node Shutdown — 노드가 꺼질 때

노드가 갑자기 꺼지면 그 위 Pod는 정리 없이 사라진다(SIGKILL). **graceful node shutdown**
(`nodeshutdown/nodeshutdown_manager.go`, 리눅스는 `_linux.go`)은 노드 종료 신호를 받아 Pod를 순서대로
정상 종료시킨다:

- **systemd inhibitor**(`nodeshutdown/systemd/`)로 OS 종료를 잠시 지연시킨다.
- 그동안 Pod들에 **PreStop 훅 + SIGTERM**을 보내 graceful termination을 수행
  ([10-pod-spec.md](10-pod-spec.md)).
- **우선순위 기반 순서**: 일반 워크로드를 먼저 종료하고, 시스템 크리티컬 Pod를 나중에(설정된 유예 시간
  배분).

이로써 노드 재부팅/스케일다운 시 진행 중 요청을 흘리지 않고 깔끔히 종료할 수 있다. 상태는 `storage.go`에
기록해 재시작 시 일관성을 유지한다.

> Pod 종료 자체의 graceful 흐름(SIGTERM → grace period → SIGKILL)은 [10-pod-spec.md](10-pod-spec.md)에서
> 다룬다. 여기선 "노드 전체가 꺼질 때 그것을 조율"하는 부분이다.

## 더 읽을 곳
- [15-cli/01](../15-cli/01-kubeadm.md) — static pod로 부팅되는 컨트롤 플레인
- [10-pod-spec.md](10-pod-spec.md) — PreStop/종료 유예
- [11-controller-manager/05](../11-controller-manager/05-node-lifecycle.md) — 노드 장애(graceful 아닌 경우)
