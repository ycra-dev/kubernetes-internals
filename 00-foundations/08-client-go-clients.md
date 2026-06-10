# 00.08 · client-go 클라이언트 종류

**근거**: `kubernetes/staging/src/k8s.io/client-go/`
(`kubernetes/`(clientset), `dynamic/`, `discovery/`, `rest/`, `restmapper/`, `scale/`, `transport/`)

[04-list-watch-informer.md](04-list-watch-informer.md)는 client-go의 **캐시 머시너리**를 다뤘다. 이
문서는 그 아래의 **실제 API 클라이언트들** — 타입드/동적/디스커버리 — 을 본다. 컨트롤러도, kubectl도,
오퍼레이터도 결국 이 클라이언트로 apiserver와 말한다.

## 계층 구조

```
RESTClient (rest/)        ─ 가장 낮은 층: HTTP 요청 빌더(verb/path/body/RV)
   ├─ Clientset (kubernetes/)   ─ 타입 안전: clientset.CoreV1().Pods(ns).Get(...)
   ├─ Dynamic (dynamic/)        ─ 타입 비의존: unstructured.Unstructured 로 임의 GVR
   ├─ Discovery (discovery/)    ─ "어떤 API가 있나" 조회
   └─ Scale (scale/)            ─ /scale 서브리소스 전용
```

## RESTClient — 최하층

`rest/`의 `RESTClient`는 apiserver와의 HTTP를 추상화한다: `Get().Namespace(ns).Resource("pods").
Name("x").Do()` 같은 fluent 빌더로 verb/경로/바디/쿼리(RV, 셀렉터)를 조립한다. 직렬화(JSON/protobuf
협상, [03-api-object-model.md](03-api-object-model.md))와 인증 주입을 처리한다. 위 모든 클라이언트가
내부적으로 이것을 쓴다.

## Clientset — 타입 안전 클라이언트

`kubernetes/`의 **Clientset**이 가장 흔히 쓰인다. 각 GVK마다 타입드 메서드가 **생성**돼 있다
([95-development/01](../95-development/01-code-generation.md)의 client-gen):

```go
clientset.AppsV1().Deployments("default").Get(ctx, "web", metav1.GetOptions{})
```

컴파일 타임 타입 검사가 되고 IDE 자동완성이 된다. 빌트인 타입에 최적. 단, 새 CRD 타입은 따로 코드 생성을
해야 쓸 수 있다.

## Dynamic Client — 타입 비의존

`dynamic/`의 **Dynamic Client**는 컴파일 시 타입을 몰라도 **GVR만으로** 임의 객체를 다룬다. 객체는
`unstructured.Unstructured`(map 기반)로 표현된다:

```go
dynamicClient.Resource(gvr).Namespace(ns).Get(ctx, name, ...)  // 어떤 CRD든
```

- **CRD/임의 타입**을 다루는 도구(GC([11-controller-manager/04](../11-controller-manager/04-garbage-collector.md)),
  kubectl, controller-runtime 일부)가 이걸 쓴다 — 모든 타입의 코드를 미리 생성할 수 없으므로.
- 타입 안전성은 포기하는 대신 범용성을 얻는다.

## Discovery Client

`discovery/`는 apiserver의 discovery 엔드포인트([10-apiserver/11](../10-apiserver/11-discovery-openapi.md))를
조회해 "어떤 그룹/버전/리소스가 있나"를 알아낸다. **RESTMapper**(`restmapper/`)가 이를 이용해 GVK↔GVR을
해석한다([03-api-object-model.md](03-api-object-model.md)). kubectl이 `deploy`를 `apps/v1/deployments`로
바꾸는 근거.

## transport — 인증과 연결

`transport/`가 HTTP 전송 계층을 담당한다: TLS, 클라이언트 인증서/토큰 주입, **exec credential plugin**
(클라우드 IAM 토큰을 외부 명령으로 받아오는 `kubectl`의 인증 방식), 연결 재사용. kubeconfig의 인증 정보가
여기서 요청에 실린다([10-apiserver/02](../10-apiserver/02-authentication.md)).

## 어떤 걸 언제 쓰나

| 상황 | 클라이언트 |
|------|-----------|
| 빌트인 타입, 타입 안전 필요 | Clientset |
| CRD/임의 타입, 범용 도구 | Dynamic |
| "어떤 API가 있나" | Discovery + RESTMapper |
| 저수준 커스텀 요청 | RESTClient |
| 오퍼레이터 작성 | controller-runtime client(이들 위의 추상화, [60-controller-runtime/02](../60-controller-runtime/02-client-cache.md)) |

## 더 읽을 곳
- [04-list-watch-informer.md](04-list-watch-informer.md) — 이 클라이언트 위의 캐시 머시너리
- [60-controller-runtime/02](../60-controller-runtime/02-client-cache.md) — 상위 추상화 클라이언트
- [95-development/01](../95-development/01-code-generation.md) — clientset 생성
