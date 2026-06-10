# 90 · enhancements (KEP)

**근거 레포**: `enhancements` (`k8s.io/enhancements`)

`enhancements` 레포는 코드가 아니라 **설계 문서 저장소**다. Kubernetes의 모든 중요한 변경은 **KEP
(Kubernetes Enhancement Proposal)** 라는 표준 문서를 거친다. 이 문서들이 "왜 이 기능이 이렇게
설계됐는가"의 1차 사료다 — 코드를 읽다 의도가 궁금할 때 찾아갈 곳이다.

## KEP이란

KEP은 기능 하나의 동기·설계·대안·위험·단계적 출시 계획을 담은 구조화된 제안서다. 작은 버그 수정이
아니라 **사용자에게 보이는 변화, API 변경, 아키텍처 영향**이 있는 모든 것이 KEP을 요구한다.

## 디렉토리 구조 — SIG별

KEP은 담당 **SIG(Special Interest Group)** 별로 나뉜다(`keps/sig-*`):

```
enhancements/keps/
├─ NNNN-kep-template/      ← 표준 템플릿(README.md + kep.yaml)
├─ sig-api-machinery/      ← apiserver/apimachinery 관련 ([10], [00-foundations/03])
├─ sig-apps/               ← 워크로드 컨트롤러 ([11])
├─ sig-scheduling/         ← 스케줄러 ([12])
├─ sig-node/               ← kubelet/런타임 ([13], [30])
├─ sig-network/            ← 네트워킹 ([40], [14])
├─ sig-storage/            ← 스토리지/CSI ([50])
├─ sig-auth/               ← 인증/인가 ([10-apiserver/02,03])
├─ sig-autoscaling/        ← HPA/VPA ([70])
└─ ... (그 외 다수 SIG)
```

각 KEP은 `keps/<sig>/<번호>-<이름>/` 디렉토리에 `README.md`(본문)와 `kep.yaml`(메타데이터)로 있다.

## KEP의 두 파일

### kep.yaml — 메타데이터
`keps/NNNN-kep-template/kep.yaml`이 표준 양식이다. 핵심 필드:

| 필드 | 의미 |
|------|------|
| `title`, `kep-number` | 제목/번호 |
| `owning-sig`, `participating-sigs` | 담당/참여 SIG |
| `status` | `provisional` → `implementable` → `implemented` (또는 `deferred`/`rejected`/`withdrawn`/`replaced`) |
| `stage` | `alpha` → `beta` → `stable` (성숙 단계) |
| `replaces` / `see-also` | 대체/관련 KEP |

### README.md — 본문
템플릿(`NNNN-kep-template/README.md`)이 정한 표준 섹션을 따른다: Summary, Motivation, Goals/Non-Goals,
Proposal, Design Details, **Test Plan**, **Graduation Criteria**(alpha→beta→stable 조건),
**Production Readiness Review**(prod-readiness/), Risks and Mitigations, Alternatives 등.

## 단계적 출시 — feature gate와의 연결

KEP의 `stage`(alpha/beta/stable)는 코드의 **feature gate**와 직접 연결된다. 새 기능은 보통:

```
alpha(기본 off, feature gate 로 opt-in)
   → beta(기본 on/off, 안정화)
   → stable(GA, gate 제거)
```

코드에서 `utilfeature.DefaultFeatureGate.Enabled(features.SomeFeature)` 같은 분기를 봤다면
([13-kubelet/01](13-kubelet/01-pod-lifecycle.md)의 EventedPLEG 등), 그 기능의 동기·설계·졸업 기준은
해당 KEP에 있다.

## 코드 ↔ 설계 추적법

1. 코드에서 feature gate 이름이나 새 API 필드를 발견.
2. 그 이름으로 `enhancements/keps/sig-*/`를 검색.
3. KEP README에서 **왜 이렇게 했는지**, 어떤 대안을 버렸는지, 어떤 위험을 고려했는지 확인.

이것이 "코드만 봐서는 알 수 없는 설계 의도"를 메우는 길이다.

## 더 읽을 곳
- [99-appendix/reading-guide.md](99-appendix/reading-guide.md) — 목적별 코드/문서 진입점
- 각 컴포넌트 문서의 "더 읽을 곳"에서 관련 SIG를 추론
