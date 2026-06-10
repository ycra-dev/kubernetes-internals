# 30.09 · OCI 이미지 포맷

**근거**: `containerd/core/images/` (`mediatypes.go` `:34`~, `image.go`, `handlers.go`, `diffid.go`)

[02-content-snapshots.md](02-content-snapshots.md)에서 "content store가 매니페스트/config/레이어를 digest로
저장한다"고 했다. 이 문서는 그 **이미지 포맷 자체** — 무엇이 어떤 구조로 저장되는지 — 를 본다. 이미지가
콘텐츠 주소 그래프임을 이해하면 pull/공유/검증이 명확해진다.

## 이미지 = 콘텐츠 주소 그래프

컨테이너 이미지는 단일 파일이 아니라, **digest로 서로를 가리키는 객체들의 그래프**다:

```
Index (멀티 아키 매니페스트 목록)
  └─► Manifest (한 아키텍처)
        ├─► Config (이미지 메타: 명령, 환경, rootfs diff_id 목록)
        └─► Layer × N (rootfs 변경분, tar+gzip/zstd)
```

각 노드는 `{mediaType, digest, size}` 형태의 **디스크립터**로 참조된다. digest(sha256)가 콘텐츠 주소라,
같은 digest는 어디서든 같은 내용 — 무결성과 공유의 기반([02-content-snapshots.md](02-content-snapshots.md)).

## Media Type — 무엇인지 알려주는 꼬리표

`core/images/mediatypes.go`가 각 조각의 종류를 정의한다(`:34`~). 두 계열이 공존한다:

| 종류 | OCI | Docker(schema2) |
|------|-----|-----------------|
| 레이어 | `application/vnd.oci.image.layer.v1.tar+gzip` | `application/vnd.docker.image.rootfs.diff.tar.gzip` (`:36`) |
| config | `application/vnd.oci.image.config.v1+json` | `application/vnd.docker.container.image.v1+json` (`:39`) |
| 매니페스트 | `application/vnd.oci.image.manifest.v1+json` | `application/vnd.docker.distribution.manifest.v2+json` |
| 인덱스 | `application/vnd.oci.image.index.v1+json` | `...manifest.list.v2+json` |

containerd는 **둘 다** 다룬다(`mediatypes.go`의 Docker/OCI 상수 병존) — Docker가 만든 이미지와 OCI 표준
이미지를 모두 받는다. 압축도 gzip/zstd 선택지가 있다(`:37` zstd).

## Index — 멀티 아키텍처

하나의 이미지 태그(`nginx:latest`)가 여러 CPU 아키텍처(amd64/arm64)를 담을 수 있다. **Index**가 "이
아키텍처면 이 매니페스트"의 목록이다. pull 시 containerd가 노드 플랫폼에 맞는 매니페스트를 골라 그것의
레이어만 받는다([02-content-snapshots.md](02-content-snapshots.md)).

## Config와 diff_id

**Config**(`vnd...config`)는 이미지의 실행 메타데이터(ENTRYPOINT, ENV, 작업 디렉토리, 노출 포트)와
**rootfs.diff_ids**(레이어들의 압축 해제 후 digest 목록)를 담는다. `core/images/diffid.go`가 이를 다룬다 —
레이어의 "압축된 digest"와 "압축 해제된 diff_id"를 구분해 무결성을 검증한다.

## handlers — 그래프 순회

`core/images/handlers.go`는 이미지 그래프를 **순회(walk)** 하는 메커니즘이다. pull/검증/GC가 "매니페스트
→ config → 레이어들"을 따라 내려가며 각 노드를 처리한다(다운로드, unpack, 참조 등록). 이 순회가:

- **pull**: 디스크립터를 따라 모든 레이어를 content store에 받음([02-content-snapshots.md](02-content-snapshots.md)).
- **GC**: 이미지에서 도달 가능한 content/snapshot을 mark([07-metadata-gc.md](07-metadata-gc.md)).
- **검증**: 각 노드의 digest 무결성 확인([06-image-verify-transfer.md](06-image-verify-transfer.md)).

## 왜 이 구조인가

- **공유**: 여러 이미지가 같은 베이스 레이어를 digest로 공유 — 한 번만 저장
  ([02-content-snapshots.md](02-content-snapshots.md)).
- **증분 pull**: 이미 있는 레이어는 다시 안 받음(digest로 존재 확인).
- **무결성/서명**: digest 그래프라 변조 감지가 쉽고, 매니페스트에 서명하면 전체가 보호됨
  ([06-image-verify-transfer.md](06-image-verify-transfer.md)).

## 더 읽을 곳
- [02-content-snapshots.md](02-content-snapshots.md) — 받은 이미지를 rootfs로
- [06-image-verify-transfer.md](06-image-verify-transfer.md) — 이미지 검증/전송
- [07-metadata-gc.md](07-metadata-gc.md) — 이미지 그래프 GC
