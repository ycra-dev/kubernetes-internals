# 95 · 개발·빌드·테스트 인프라

**근거 레포**: `kubernetes` (`staging/.../code-generator`, `hack/`, `test/`, `build/`)

이 폴더는 Kubernetes 코드가 **어떻게 생성·빌드·검증되는지**를 다룬다. 런타임 동작이 아니라 개발자
워크플로지만, 코드를 읽다 보면 "이건 손으로 안 짠 것 같은데?"(맞다, 생성된 코드다) 같은 의문이 생기므로
알아둘 가치가 있다.

## 핵심 사실: 많은 코드가 생성된다

Kubernetes 코드의 상당 부분은 **코드 생성기**가 만든다. `zz_generated.*.go`, `generated.pb.go`,
`*/clientset/`, `*/informers/` 같은 파일은 손으로 짜지 않는다. API 타입 하나를 정의하면, 그에 딸린
deepcopy·client·informer·protobuf가 자동 생성된다.

## 문서

| # | 문서 | 내용 |
|---|------|------|
| 01 | [code-generation.md](01-code-generation.md) | deepcopy/client/informer/conversion 등 생성기 |
| 02 | [build-test.md](02-build-test.md) | 빌드 시스템, e2e/통합/conformance 테스트 |

## 왜 생성하나

[00-foundations/03](../00-foundations/03-api-object-model.md)에서 봤듯, 모든 API 타입은 같은 규약
(deepcopy, Scheme 등록, client, informer)을 따라야 한다. 타입이 수백 개인데 이를 손으로 짜면:
- 실수가 생기고,
- 규약이 일관되지 않으며,
- 새 타입 추가가 번거롭다.

그래서 "타입 정의 + marker 주석"에서 보일러플레이트를 **생성**한다. 개발자는 타입과 비즈니스 로직만 쓴다.
이는 [61-kubebuilder](../61-kubebuilder.md)가 오퍼레이터에 해주는 일을, 코어가 자기 자신에게 하는 것이다.

## 더 읽을 곳
- [00-foundations/03](../00-foundations/03-api-object-model.md) — 생성된 코드가 따르는 규약
- [61-kubebuilder](../61-kubebuilder.md) — 같은 생성 도구의 오퍼레이터 버전
