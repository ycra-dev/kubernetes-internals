# 11.06 · StatefulSet 컨트롤러

**근거**: `kubernetes/pkg/controller/statefulset/` (`stateful_set_control.go`, `stateful_set.go`)

StatefulSet은 **고정된 신원과 순서가 필요한** 워크로드(데이터베이스, 분산 시스템)를 위한 컨트롤러다.
ReplicaSet([00-foundations/05](../00-foundations/05-controller-pattern.md))이 "N개의 교체 가능한 Pod"를
보장한다면, StatefulSet은 "**순서가 있고 신원이 고정된** N개의 Pod"를 보장한다.

## ReplicaSet과의 결정적 차이

| | ReplicaSet | StatefulSet |
|--|-----------|-------------|
| Pod 이름 | 랜덤(`web-x7k2p`) | **순번 고정**(`web-0`, `web-1`, `web-2`) |
| 생성/삭제 순서 | 임의·병렬 | **순서대로**(0→1→2 생성, 역순 삭제) |
| 스토리지 | 공유/없음 | **Pod별 고유 PVC**(`web-0`은 항상 같은 볼륨) |
| 네트워크 신원 | 없음 | Headless Service로 **안정적 DNS**(`web-0.svc...`) |

## ordinal — 신원의 핵심

각 Pod는 **ordinal**(순번)을 가지며, 그 순번이 곧 신원이다(`stateful_set_control.go`의 ordinal 로직).
`web-0`이 죽으면 **같은 이름·같은 PVC·같은 DNS**로 다시 생성된다 — 랜덤 새 Pod가 아니다. 이로써 클러스터
멤버십(예: etcd가 `web-0`을 특정 멤버로 인식)이 유지된다.

## 순서 보장 — monotonic 업데이트

핵심 진입점은 `UpdateStatefulSet`(`stateful_set_control.go:86`)이다. 기본 전략은 **monotonic**
(`:81` 주석): 스케일 업은 ordinal 순서로 진행되며, **이전 Pod가 Running & Ready가 되기 전엔 다음 Pod를
만들지 않는다**(`:471`). 스케일 다운은 역순.

```
스케일 업 (0→3):  web-0 생성 → Ready 대기 → web-1 생성 → Ready 대기 → web-2
스케일 다운(3→0):  web-2 삭제 → web-1 삭제 → web-0 삭제 (역순)
```

이 순서 보장 덕분에 "리더가 먼저 떠야 팔로워가 붙는" 류의 시스템을 안전하게 운영한다.

## identity / storage 매칭

`UpdateStatefulSet`은 각 ordinal 위치의 Pod가 **올바른 신원과 스토리지를 갖는지** 검사한다
(`stateful_set_control.go:494`의 `identityMatches && storageMatches`):

- **identityMatches**: Pod 이름/호스트네임/subdomain이 ordinal에 맞나.
- **storageMatches**: Pod가 자기 ordinal의 PVC에 묶여 있나.

어긋나면 Pod를 삭제·재생성해 맞춘다. PVC는 `volumeClaimTemplates`로 Pod마다 자동 생성되고, Pod가 죽어도
PVC는 보존돼 데이터가 유지된다(retention 정책으로 조정 가능, `retentionMatch` `:494`).

## 롤링 업데이트 — partition

업데이트(`updateStatefulSet`, `stateful_set_control.go:560`)도 순서대로다. **partition**으로 카나리
배포가 가능하다(`:557` 주석): `Partition.Ordinal`보다 작은 ordinal은 현재 리비전 유지, 그 이상만 새
리비전으로 업데이트. partition을 점점 낮춰 단계적으로 굴린다.

리비전 추적에는 **ControllerRevision**(`history/`)을 쓴다 — Pod 템플릿의 각 버전을 객체로 보존해
업데이트/롤백의 기준으로 삼는다(DaemonSet도 공유, [07](07-daemonset.md)).

## 더 읽을 곳
- [00-foundations/05](../00-foundations/05-controller-pattern.md) — 기본 컨트롤러 패턴
- [50-storage](../50-storage/) — Pod별 PVC와 볼륨
- [40-networking/03](../40-networking/03-dns.md) — Headless Service의 안정적 DNS
