# 30.11 · Content Store 내부

**근거**: `containerd/core/content/content.go`(`Ingester`/`Writer`/`Commit` `:34`~`:91`, `digest`),
`containerd/plugins/content/local/`

[02-content-snapshots.md](02-content-snapshots.md)에서 content store를 "콘텐츠 주소(digest) 저장소"로
개요했다. 이 문서는 그 **내부 동작** — 어떻게 blob을 받아 digest로 검증·저장하는지 — 를 본다. 이미지
pull의 무결성 보장이 여기서 일어난다.

## 콘텐츠 주소 = digest

content store의 모든 객체는 **digest(sha256:...)** 로 식별된다(`content.go:24` `go-digest`). 핵심 성질:

- **불변·검증 가능**: digest는 내용의 해시라, 받은 바이트를 다시 해시해 digest와 비교하면 변조를
  감지한다. "받은 게 진짜 그 내용"임을 수학적으로 보장.
- **자동 중복 제거**: 같은 내용 = 같은 digest = 한 번만 저장. 여러 이미지가 공유하는 레이어가 중복되지
  않는다([02-content-snapshots.md](02-content-snapshots.md)).

## Ingester / Writer — 받아서 검증

새 blob을 저장하는 흐름이 `content.go`의 **Ingester → Writer → Commit**이다(`:34`~`:69`):

```
PullImage: 레이어 blob 다운로드 (30-containerd/06)
   │
Ingester.Writer(ref): 미완성 ingestion 시작 (임시 영역에 기록)
   │  레이어 바이트를 Writer에 스트리밍 기록 (다운로드하며)
   ▼
Writer.Commit(size, expectedDigest):
   - 기록된 내용의 digest 계산
   - expectedDigest와 비교 → 다르면 거부(변조/손상)
   - 일치하면 digest 주소로 영구 저장
```

`Commit`(`content.go:69`)이 **무결성 게이트**다 — 매니페스트가 약속한 digest와 실제 받은 바이트의 digest가
일치할 때만 저장한다. 다운로드 중 손상되거나 변조되면 여기서 걸린다.

## Provider / ReaderAt — 읽기

저장된 blob은 **Provider**(`content.go:34` 주석)로 digest를 줘서 읽는다(`ReaderAt`). unpack
([02-content-snapshots.md](02-content-snapshots.md))이 레이어를 읽어 snapshot으로 풀 때, GC가 참조를
추적할 때([07-metadata-gc.md](07-metadata-gc.md)) 이 인터페이스를 쓴다.

## local 구현 — 파일시스템

`plugins/content/local/`이 표준 구현이다. blob을 **digest 기반 경로**의 파일로 저장한다:

```
content/
  blobs/sha256/<digest의 앞부분>/<digest>   ← 레이어/매니페스트/config 파일
  ingest/<ref>/                              ← 진행 중(미완성) ingestion
```

ingest 디렉토리는 다운로드 중인 미완성 blob을 담고, Commit되면 blobs로 옮겨진다. 중단된 ingestion은
재개하거나 GC된다.

## status / 재개

큰 레이어 다운로드가 중단되면 처음부터 다시 받는 건 낭비다. content store는 ingestion **status**를 추적해
(어디까지 받았나) **재개(resumable)** 를 지원한다. ref(참조 키)로 진행 중 ingestion을 식별한다.

## lease로 보호

[02-content-snapshots.md](02-content-snapshots.md), [07-metadata-gc.md](07-metadata-gc.md)에서 봤듯, 막
받은 blob은 아직 이미지에 연결 전이라 GC 대상이다. **lease**로 ingestion 중 blob을 보호한다 — pull이
끝나 이미지가 참조하면 lease 해제.

## 무결성 체인 (요약)

```
레지스트리 ─바이트→ Writer ─digest 검증(Commit)→ content store(불변 blob)
   │                                              ↑
매니페스트가 약속한 digest와 대조 → 변조/손상 차단     서명 검증(imageverifier, 30-containerd/06)
```

digest 기반 저장이 "받은 바이트의 무결성"을, imageverifier가 "출처의 신뢰성"을 보장한다
([06-image-verify-transfer.md](06-image-verify-transfer.md)). 둘이 합쳐 공급망 보안의 토대가 된다.

## 더 읽을 곳
- [02-content-snapshots.md](02-content-snapshots.md) — content store 개요
- [06-image-verify-transfer.md](06-image-verify-transfer.md) — 서명 검증
- [07-metadata-gc.md](07-metadata-gc.md) — content GC
- [09-oci-image-format.md](09-oci-image-format.md) — digest 그래프(이미지 포맷)
