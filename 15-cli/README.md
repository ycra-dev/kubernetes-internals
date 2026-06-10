# 15 · kubectl

**근거 레포**: `kubernetes` (`staging/src/k8s.io/kubectl/`, `staging/src/k8s.io/cli-runtime/`)

kubectl은 사용자가 클러스터와 대화하는 주요 CLI다. 본질은 **apiserver의 REST API에 대한 클라이언트
래퍼**다 — kubectl이 하는 거의 모든 일은 [10-apiserver](../10-apiserver/)로 가는 HTTP 요청으로 환원된다.

## 명령 구조

각 서브커맨드는 `staging/src/k8s.io/kubectl/pkg/cmd/` 아래 한 패키지씩이다(cobra 기반). 분류:

| 분류 | 명령 | 하는 일 |
|------|------|---------|
| 조회 | `get`, `describe`, `explain`, `events`, `logs` | apiserver에서 객체를 읽어 표시 |
| 변경 | `apply`, `create`, `edit`, `patch`, `delete`, `label`, `annotate` | 객체를 만들고/고치고/지움 |
| 워크로드 | `scale`, `rollout`, `autoscale`, `expose`, `run` | 워크로드/서비스 조작 |
| 디버그 | `exec`, `attach`, `port-forward`, `debug`, `cp` | 컨테이너에 접속(스트리밍) |
| 노드 | `drain`, `cordon`, `taint` | 노드 운영 |
| 메타 | `config`, `auth`, `api-resources`, `cluster-info` | kubeconfig/권한/디스커버리 |

## 핵심 메커니즘: Builder와 Resource

kubectl이 "파일이든, 라벨 셀렉터든, 이름이든" 다양한 입력을 일관되게 다루는 비결은
`cli-runtime`의 **resource.Builder**(`cli-runtime/pkg/resource/builder.go`)다.

- 입력(파일 경로, stdin, `-l` 셀렉터, 리소스/이름)을 받아 **Visitor** 체인
  (`cli-runtime/pkg/resource/visitor.go`)을 만든다.
- Visitor를 순회하며 각 객체에 대해 동작(get/apply/delete)을 수행한다.
- RESTMapper([00-foundations/03](../00-foundations/03-api-object-model.md))로 짧은 이름(`deploy`)을
  올바른 GVR/REST 경로로 해석한다.

즉 `kubectl get pods -l app=x -o yaml`은 Builder가 셀렉터를 REST list 요청으로 바꾸고, 응답을 원하는
출력 형식으로 직렬화하는 흐름이다.

## apply — 선언형의 클라이언트 측

`kubectl apply`는 단순 create/update가 아니다. "사용자가 선언한 상태"를 기존 객체에 **3-way merge**
(이전 적용본 / 현재 클러스터 상태 / 새 매니페스트)로 병합한다. 오늘날은 서버 측에서 필드 소유권을
추적하는 **Server-Side Apply**(apiserver의 fieldmanager,
`apiserver/pkg/endpoints/handlers/fieldmanager`)로 처리해 충돌을 명확히 한다.

## 인증/접속 — kubeconfig

kubectl은 `~/.kube/config`(kubeconfig)에서 클러스터 주소·인증 정보(인증서/토큰/exec 플러그인)를 읽어
apiserver에 연결한다. 인증은 [10-apiserver/02](../10-apiserver/02-authentication.md)의 인증기들이 받는다.

## 문서

| # | 문서 | 내용 |
|---|------|------|
| 01 | [kubeadm.md](01-kubeadm.md) | 클러스터 부트스트랩 도구 |
| 02 | [kubectl-internals.md](02-kubectl-internals.md) | Builder/Visitor, apply, exec 스트리밍 |
| 03 | [kubeadm-upgrade-reset.md](03-kubeadm-upgrade-reset.md) | 클러스터 업그레이드/리셋 |

## 더 읽을 곳
- [10-apiserver](../10-apiserver/) — kubectl 요청을 받는 서버
- [00-foundations/03](../00-foundations/03-api-object-model.md) — 짧은 이름↔GVR 해석(RESTMapper)
