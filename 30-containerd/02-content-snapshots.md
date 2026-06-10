# 30.02 · 이미지 — content store와 snapshot

**근거**: `containerd/core/content/`, `containerd/core/images/`, `containerd/core/snapshots/`,
`containerd/core/unpack/`, `containerd/core/diff/`

컨테이너 이미지가 어떻게 받아지고, 어떻게 컨테이너의 루트 파일시스템이 되는지가 이 문서다. 핵심은
**불변 레이어(content)** 와 **그 위에 쓰는 작업 공간(snapshot)** 의 분리다.

## content store — 콘텐츠 주소 저장소

`core/content/`는 이미지의 모든 조각(매니페스트, config, 레이어 tar)을 **콘텐츠 주소(digest, sha256)** 로
저장한다. 같은 digest는 한 번만 저장되므로:

- 여러 이미지가 공유하는 레이어는 중복 저장되지 않는다.
- digest로 무결성이 검증된다(받은 내용이 변조되지 않았음).

`PullImage`([01](01-cri-plugin.md))는 레지스트리(`core/remotes/`)에서 매니페스트를 받아, 참조된 레이어
blob들을 content store에 내려받는다.

## images — 이미지 메타데이터

`core/images/`는 "이름(태그) → 매니페스트 digest" 매핑과 이미지 구조(매니페스트/인덱스)를 다룬다.
멀티 아키텍처 이미지에서 현재 플랫폼에 맞는 매니페스트를 고르는 일도 여기서.

## unpack — 레이어를 snapshot으로

받은 레이어(압축 tar)는 그대로 쓸 수 없다. `core/unpack/`이 레이어를 순서대로 풀어
([core/diff/](../30-containerd/) 의 apply) **snapshot**으로 쌓는다. 각 레이어가 이전 레이어 위에 올라가
최종 이미지 rootfs를 이룬다.

## snapshotter — 컨테이너의 쓰기 가능한 rootfs

`core/snapshots/snapshotter.go`의 **Snapshotter** 인터페이스가 핵심이다. 컨테이너마다 이미지 레이어
위에 **얇은 쓰기 계층**을 얹어 rootfs를 만든다. 인터페이스 주석(`snapshotter.go:157`~)이 설명하는 흐름:

```
Prepare(key, parent)  ─► active snapshot 생성, Mounts 로 마운트 정보 획득
   (컨테이너가 이 위에서 파일을 읽고 쓴다 — 이미지 레이어는 불변, 변경은 이 계층에)
Commit(name, key)     ─► (이미지 빌드 시) active → committed 레이어로 확정
```

- **active** 스냅샷: 쓰기 가능(컨테이너 실행 중 rootfs).
- **committed** 스냅샷: 읽기 전용(이미지 레이어).

컨테이너를 띄울 때 CRI는 `Prepare`로 active snapshot을 만들고, 그 `Mounts`를 OCI 번들의 rootfs로 쓴다
([03](03-runtime-shim-oci.md)). 컨테이너가 지워지면 그 active snapshot만 버리면 되고, 공유 이미지
레이어는 그대로 남는다.

### snapshotter 구현
overlayfs(가장 흔함), btrfs, zfs, devmapper 등 여러 백엔드가 플러그인으로 제공된다(`plugins/snapshots`).
overlayfs는 "하위(이미지 레이어, 읽기전용) + 상위(쓰기 계층)"를 합쳐 보여주는 커널 기능이라, 컨테이너
시작이 빠르고 디스크 효율적이다.

## leases — GC로부터 보호

content/snapshot은 참조가 없으면 GC된다(`core/leases/`, `plugins/gc`). 작업 중인 콘텐츠가 갑자기
수집되지 않도록 **lease**로 보호한다(etcd lease와 무관, 같은 이름의 다른 개념).

## 더 읽을 곳
- [03-runtime-shim-oci.md](03-runtime-shim-oci.md) — 준비된 rootfs로 프로세스 띄우기
- [01-cri-plugin.md](01-cri-plugin.md) — 이 과정을 부르는 CRI
