# 30.04 · ctr CLI와 client 라이브러리

**근거**: `containerd/cmd/ctr/`, `containerd/client/`

containerd를 직접 다루는 방법 두 가지: 디버그용 CLI **ctr**와, Go 프로그램용 **client 라이브러리**.

## ctr — 디버그/저수준 CLI

`cmd/ctr/`의 `ctr`은 containerd에 직접 gRPC로 접속하는 저수준 CLI다(`cmd/ctr/commands/`). 예:

- `ctr images pull/list` — 이미지 관리([02](02-content-snapshots.md)).
- `ctr containers create` / `ctr tasks start` — 컨테이너/태스크 실행([03](03-runtime-shim-oci.md)).
- `ctr snapshots ls` — 스냅샷 조회.

> ctr은 **kubelet/CRI를 우회**한다. 그래서 노드 디버깅에 유용하지만, ctr로 만든 컨테이너는 Kubernetes가
> 모른다. Kubernetes 컨텍스트에서 CRI 수준을 보려면 `crictl`(별도 도구)이 CRI 네임스페이스(`k8s.io`)로
> 접속한다. 둘 다 같은 containerd를 보지만 추상화 수준이 다르다.

## client 라이브러리

`containerd/client/`는 containerd를 임베드/제어하는 Go API다. 이미지 pull, 컨테이너 생성, 태스크 실행,
snapshot 관리 등을 타입 안전한 함수로 제공한다. containerd를 백엔드로 쓰는 다른 프로젝트(빌드 도구 등)가
이 라이브러리를 쓴다.

## namespaces — 멀티 테넌트 격리

containerd는 자체 **namespace** 개념으로 객체(이미지/컨테이너/스냅샷)를 격리한다(리눅스 네임스페이스와
무관). Kubernetes 워크로드는 `k8s.io` 네임스페이스를, ctr 기본은 `default`를 쓴다. 그래서 ctr 기본으로
보면 Kubernetes 컨테이너가 안 보일 수 있다(`ctr -n k8s.io ...` 필요).

## 더 읽을 곳
- [README.md](README.md) — containerd 전체 구조
- [13-kubelet/02](../13-kubelet/02-cri.md) — 운영 경로(CRI)
