# 15.02 · kubectl 내부 — Builder, apply, 스트리밍

**근거**: `kubernetes/staging/src/k8s.io/cli-runtime/pkg/resource/`(Builder/Visitor),
`kubernetes/staging/src/k8s.io/client-go/tools/remotecommand/`(`spdy.go`, `v1.go`, `v2.go`, `fallback.go`)

[README](README.md)에서 kubectl이 "apiserver REST 클라이언트 래퍼"라 했다. 이 문서는 그 핵심 메커니즘
세 가지 — 입력을 객체로 바꾸는 Builder, apply의 병합, exec/logs의 스트리밍 — 을 코드 수준에서 본다.

## Builder/Visitor — 다양한 입력을 하나로

`cli-runtime/pkg/resource/`의 **Builder**가 kubectl의 입력 처리 심장이다([README](README.md)). 다양한
입력 형태를 **공통 Visitor 스트림**으로 변환한다:

```
입력: -f file.yaml | -k kustomize/ | stdin | "deploy/web" | -l app=x
   │  Builder가 각각을 해석
   ▼
Visitor 체인: 각 객체에 대해 동작 수행 (Visit(fn))
   │  내부에서 RESTMapper로 GVR 해석, RESTClient로 요청
   ▼
get/apply/delete/... 동작
```

- 파일/디렉토리/URL/stdin → 디코딩 Visitor.
- 라벨 셀렉터/이름 → list/get Visitor(apiserver 조회,
  [00-foundations/08](../00-foundations/08-client-go-clients.md)).
- 여러 Visitor를 합성/필터링해 균일하게 처리 — 그래서 `kubectl delete -f`와 `kubectl delete deploy web`이
  같은 코드 경로를 탄다.

## apply — 3-way merge와 Server-Side Apply

`kubectl apply`는 단순 update가 아니다([README](README.md)):

- **클라이언트 측(레거시)**: 세 입력 — 이전 적용본(last-applied annotation), 현재 클러스터 상태, 새
  매니페스트 — 으로 **3-way merge** patch를 계산해 보낸다. 사용자가 지운 필드를 감지해 제거.
- **Server-Side Apply(권장)**: 병합을 서버가 한다. kubectl은 자기 소유 필드만 보내고, 서버의
  fieldmanager가 충돌을 판정한다([10-apiserver/08](../10-apiserver/08-server-side-apply.md)).

SSA로 옮겨가는 이유: 클라이언트 측 3-way merge는 annotation에 의존해 깨지기 쉽고, 여러 actor(HPA 등)와의
충돌을 표현 못 했다.

## exec / attach / port-forward — 양방향 스트리밍

`kubectl exec`/`logs`/`port-forward`는 일반 REST(요청-응답)가 아니라 **양방향 스트림**이 필요하다(터미널
입출력, 포트 터널). `client-go/tools/remotecommand/`가 이를 구현한다:

- **SPDY**(`spdy.go`): 전통적 다중화 스트림 프로토콜(stdin/stdout/stderr/resize를 한 연결에 다중화).
- **WebSocket**(`v1.go`/`v2.go`): 최신 대안. 프록시/방화벽 친화적.
- **fallback**(`fallback.go`): WebSocket 시도 후 안 되면 SPDY로 폴백.

흐름: kubectl → apiserver → (kubelet) → CRI streaming → 컨테이너
([13-kubelet/02](../13-kubelet/02-cri.md)의 Exec/Attach). apiserver가 스트림을 kubelet으로 프록시하고,
kubelet이 containerd의 streaming 서버로 잇는다([30-containerd/01](../30-containerd/01-cri-plugin.md)).

```
kubectl exec ─SPDY/WS─► apiserver ─프록시─► kubelet ─CRI Exec─► containerd ─► 컨테이너 셸
```

## 플러그인 — kubectl 확장

`kubectl`은 `kubectl-<name>` 실행 파일을 PATH에서 찾아 서브커맨드로 노출한다(krew 생태계). `kubectl
foo`는 `kubectl-foo` 바이너리를 실행 — 코어를 안 건드리고 CLI를 확장하는 방식. 자격증명/디스커버리는
공유한다.

## 출력 — printers

조회 결과는 `cli-runtime`의 printers로 형식화된다: 기본 테이블(서버가 보내는 Table 형식), `-o yaml/json`,
`-o jsonpath`, `-o custom-columns`, `-o wide`. 서버 측 Table 변환
([10-apiserver/05](../10-apiserver/05-registry-storage.md)의 `table.go`)이 `kubectl get`의 컬럼을 정한다.

## 더 읽을 곳
- [00-foundations/08](../00-foundations/08-client-go-clients.md) — Builder가 쓰는 클라이언트들
- [10-apiserver/08](../10-apiserver/08-server-side-apply.md) — apply의 서버 측
- [13-kubelet/02](../13-kubelet/02-cri.md) — exec 스트림의 종착지
