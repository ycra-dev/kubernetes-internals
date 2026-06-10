# 00.10 · 직렬화와 콘텐츠 협상

**근거**: `kubernetes/staging/src/k8s.io/apimachinery/pkg/runtime/serializer/`
(`json/`, `protobuf/`, `cbor/`, `yaml/`, `codec_factory.go`, `negotiated_codec.go`),
`pkg/runtime/negotiate.go`

[03-api-object-model.md](03-api-object-model.md)에서 "객체는 JSON/protobuf/YAML로 직렬화된다"고 했다. 이
문서는 그 **직렬화기들과 콘텐츠 협상(content negotiation)** 을 본다. 클라이언트와 apiserver가 "어떤
형식으로 주고받을지"를 정하는 메커니즘이다 — in-memory 객체는 하나지만 와이어 형식은 여럿이다.

## 하나의 객체, 여러 형식

`serializer/` 아래에 형식별 직렬화기가 있다:

| 형식 | 디렉토리 | 용도 |
|------|----------|------|
| **JSON** | `json/` | 사람이 읽음, kubectl/YAML 호환, 기본 외부 형식 |
| **Protobuf** | `protobuf/` | apiserver↔컴포넌트 내부 통신(작고 빠름) |
| **YAML** | `yaml/` | 매니페스트(JSON 위의 표현) |
| **CBOR** | `cbor/` | 이진 형식(JSON 데이터 모델의 효율적 인코딩) |

핵심: **내부 표현(in-memory Go struct)은 하나**이고, 직렬화기가 그것을 형식 간에 변환한다. protobuf 태그는
[03-api-object-model.md](03-api-object-model.md)의 `types.go` 필드에 박혀 있다(`protobuf:"..."`).

## 왜 protobuf인가

JSON은 사람이 읽기 좋지만 크고 파싱이 느리다. apiserver와 컴포넌트(스케줄러/kubelet 등)는 초당 수많은
객체를 주고받으므로, 기본 내부 통신은 **protobuf**를 쓴다:

- 더 작은 바이트 → etcd 저장/네트워크 절약([10-apiserver/06](../10-apiserver/06-etcd-storage-layer.md)은
  보통 protobuf로 저장).
- 더 빠른 (역)직렬화.

`kubectl`은 사람이 보는 용도라 JSON/YAML을 쓴다 — 같은 객체를 다른 형식으로 받는 것뿐이다.

## 콘텐츠 협상 — Accept / Content-Type

클라이언트와 서버가 형식을 합의하는 것이 **콘텐츠 협상**(`pkg/runtime/negotiate.go`,
`serializer/negotiated_codec.go`):

```
클라이언트 요청 헤더:
  Content-Type: application/json        ← 내가 보내는 형식
  Accept: application/vnd.kubernetes.protobuf  ← 받고 싶은 형식
   │
apiserver: 디코딩(JSON) → 처리 → 인코딩(protobuf로 응답)
```

- 컴포넌트들은 보통 `Accept: protobuf`로 효율을 택한다.
- protobuf를 모르는(또는 CRD처럼 protobuf 스키마가 없는) 경우 JSON으로 폴백한다 — CRD는 protobuf 미지원이라
  JSON으로 저장/전송된다([10-apiserver/07](../10-apiserver/07-crd-aggregation.md)).

## CodecFactory — 코덱 조립

`serializer/codec_factory.go`의 **CodecFactory**가 "이 형식 + 이 버전"에 맞는 코덱을 만든다. 코덱은
직렬화기 + 버전 변환([09-conversion.md](09-conversion.md))을 묶는다:

```
codec = 직렬화기(형식) + versioning(버전 변환)
  디코딩: 바이트 → (형식 파싱) → (버전 변환) → internal 객체
  인코딩: internal 객체 → (버전 변환) → (형식 직렬화) → 바이트
```

`serializer/versioning/`이 그 버전 변환 단계로, [09-conversion.md](09-conversion.md)의 Scheme.Convert를
호출한다. 즉 "v1beta1 JSON으로 받아 internal로 디코딩, v1 protobuf로 인코딩해 저장"이 코덱 한 번에 일어난다.

## streaming — watch 응답

`serializer/streaming/`은 watch 같은 **스트리밍 응답**을 다룬다. watch는 단일 객체가 아니라 이벤트의
연속이므로([00-foundations/04](04-list-watch-informer.md)), 각 이벤트를 프레임으로 직렬화해 흘려보낸다
(chunked). apiserver의 watch cacher([10-apiserver/12](../10-apiserver/12-watch-cache-internals.md))가 이를
통해 클라이언트에 이벤트를 스트리밍한다.

## 정리

```
in-memory 객체 1개  ↔  코덱(형식 직렬화기 + 버전 변환)  ↔  와이어 바이트(JSON/protobuf/...)
   콘텐츠 협상(Accept/Content-Type)이 형식을, RESTMapper/버전이 버전을 정함
```

이 계층이 "효율적 내부 통신(protobuf) + 사람 친화 인터페이스(YAML) + 버전 호환"을 동시에 가능케 한다.

## 더 읽을 곳
- [03-api-object-model.md](03-api-object-model.md) — 직렬화되는 객체 구조
- [09-conversion.md](09-conversion.md) — 코덱이 호출하는 버전 변환
- [08-client-go-clients.md](08-client-go-clients.md) — 협상을 수행하는 RESTClient
