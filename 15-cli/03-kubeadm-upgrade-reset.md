# 15.03 · kubeadm — 업그레이드와 리셋

**근거**: `kubernetes/cmd/kubeadm/app/cmd/upgrade/`, `cmd/reset.go`,
`cmd/kubeadm/app/phases/upgrade/`(`compute.go`, `staticpods.go`, `health.go`, `postupgrade.go`)

[01-kubeadm.md](01-kubeadm.md)에서 kubeadm의 init/join을 봤다. 이 문서는 클러스터의 **수명주기 후반** —
버전 업그레이드와 노드 리셋 — 을 본다. 업그레이드는 [91-versioning-skew.md](../91-versioning-skew.md)의
버전 스큐 정책을 실제로 적용하는 절차다.

## 업그레이드 — 버전 스큐를 지키며 굴리기

`kubeadm upgrade`는 컨트롤 플레인을 새 버전으로 올린다. **한 번에 다 올리지 않고** 순서를 지킨다
([91-versioning-skew.md](../91-versioning-skew.md)의 "apiserver가 가장 최신"):

```
1. upgrade plan/compute (phases/upgrade/compute.go)
     현재 버전 + 가능한 목표 버전 계산, 스큐 정책 위반 여부 검사 (versiongetter.go)
2. upgrade apply
     a. preflight (preflight.go): 건강/전제 점검
     b. etcd 업그레이드 (필요시)
     c. 컨트롤 플레인 static pod 교체 (staticpods.go):
          apiserver → controller-manager → scheduler 매니페스트를 새 버전으로
          ([01-kubeadm.md]의 static pod, kubelet이 파일 변경 감지해 재시작)
     d. health 확인 (health.go): 새 컴포넌트가 건강한지
     e. postupgrade (postupgrade.go): 애드온(CoreDNS/kube-proxy) 갱신, 설정 마이그레이션
3. 노드별로 kubeadm upgrade node + kubelet 업그레이드 (마지막)
```

### static pod 교체가 핵심
컨트롤 플레인 업그레이드의 실체는 **static pod 매니페스트 교체**다(`staticpods.go`). kubeadm이
`/etc/kubernetes/manifests/`의 매니페스트 이미지 태그를 새 버전으로 바꾸면, kubelet이 파일 변경을 감지해
([13-kubelet/08](../13-kubelet/08-static-pods-shutdown.md)) 컨테이너를 새 버전으로 재시작한다. 교체 전
백업을 떠 롤백 가능하게 한다.

### 순서의 이유
apiserver를 먼저, kubelet(노드)을 마지막에 올린다 — 그래야 업그레이드 도중에도 버전 스큐가 허용 범위
안에 있다([91-versioning-skew.md](../91-versioning-skew.md)). 거꾸로 하면(노드가 apiserver보다 최신) 스큐
정책 위반.

## 리셋 — 노드를 깨끗이 되돌리기

`kubeadm reset`(`cmd/reset.go`)은 노드를 kubeadm 적용 전 상태로 되돌린다 — init/join의 역연산:

- kubelet 중지, static pod 매니페스트 제거([01-kubeadm.md](01-kubeadm.md)).
- etcd 멤버면 클러스터에서 제거(멤버십 변경, [20-etcd/09](../20-etcd/09-auth-membership.md)).
- 생성한 인증서/설정/데이터 디렉토리 정리([17-security/03](../17-security/03-pki-certificates.md)).
- (일부) iptables/CNI 설정 잔여물 안내(완전 정리는 수동일 수 있음).

리셋 후 그 노드는 다시 init/join할 수 있다.

## 업그레이드 안전성

kubeadm 업그레이드가 무중단에 가까운 이유:
- HA 컨트롤 플레인이면 한 노드씩 굴려 apiserver 가용성 유지(graceful shutdown,
  [10-apiserver/15](../10-apiserver/15-server-lifecycle.md)).
- 워크로드 Pod는 컨트롤 플레인 재시작과 무관하게 계속 돈다(노드의 kubelet/컨테이너는 그대로,
  [00-foundations/02](../00-foundations/02-architecture.md)).
- 노드 업그레이드 시 `kubectl drain`으로 PDB를 지키며 Pod를 비운다
  ([45-policy/02](../45-policy/02-pdb-priority.md)).

## 더 읽을 곳
- [01-kubeadm.md](01-kubeadm.md) — init/join
- [91-versioning-skew.md](../91-versioning-skew.md) — 버전 스큐 정책
- [10-apiserver/15](../10-apiserver/15-server-lifecycle.md) — apiserver graceful shutdown
