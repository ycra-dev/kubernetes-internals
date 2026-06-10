# 13.10 · Pod 스펙 세부 — init/sidecar/hooks/종료

**근거**: `kubernetes/pkg/kubelet/kuberuntime/kuberuntime_manager.go`(`:1717` restartable init = sidecar),
core API의 `Lifecycle`(PostStart/PreStop)

Pod 안에는 여러 종류의 컨테이너와 라이프사이클 훅이 있다. SyncPod([01-pod-lifecycle.md](01-pod-lifecycle.md))가
이들을 정해진 순서로 다룬다.

## 컨테이너 종류와 실행 순서

```
[init 컨테이너들] ── 순차 실행, 각자 성공해야 다음으로
   │  (그중 restartable init = "네이티브 사이드카")
[app 컨테이너들]  ── 병렬 시작
```

### init 컨테이너
앱 컨테이너 **전에 순서대로** 실행돼 완료되어야 한다. 셋업(마이그레이션, 설정 다운로드, 의존성 대기)에
쓴다. 하나라도 실패하면 (정책에 따라) 재시도하고, app 컨테이너는 시작되지 않는다.

### 네이티브 사이드카 (restartable init container)
`kuberuntime_manager.go:1717`/`:1800`의 "restartable init containers (sidecars)"가 그것이다. init
컨테이너에 `restartPolicy: Always`를 주면 **사이드카**가 된다:

- init 단계에서 시작되지만 **계속 살아 있다**(보통 init과 달리 끝나지 않음).
- 그다음 init/app 컨테이너와 **함께** 동작한다(로그 수집기, 프록시 등).
- app 컨테이너가 끝나면 사이드카도 종료된다.

기존엔 사이드카를 일반 컨테이너로 넣어 종료 순서/Job 완료 문제가 있었는데, 네이티브 사이드카가 이를 푼다.

## 라이프사이클 훅 — PostStart / PreStop

컨테이너의 `Lifecycle`(core API)은 두 훅을 제공한다:

- **PostStart**: 컨테이너 시작 직후 실행(시작과 비동기). 초기화 트리거 등.
- **PreStop**: 컨테이너 종료 **직전** 실행. graceful 종료 준비(연결 드레이닝, "이제 트래픽 그만" 신호)에
  쓴다.

훅은 Exec 또는 HTTP로 실행된다.

## Pod 종료 흐름 (graceful termination)

Pod가 삭제되면 즉시 죽이지 않는다:

```
1. Pod에 DeletionTimestamp 설정 + EndpointSlice에서 제외(트래픽 차단)
2. PreStop 훅 실행 (있으면)
3. 컨테이너에 SIGTERM 전송
4. terminationGracePeriodSeconds 대기 (앱이 정리할 시간)
5. 시간 초과 시 SIGKILL (강제 종료)
```

이 유예(grace period) 덕분에 진행 중 요청을 끝내고 깔끔히 빠질 수 있다. 노드 전체 종료 시의 조율은
[08-static-pods-shutdown.md](08-static-pods-shutdown.md).

## downward API — 자기 정보 주입

컨테이너는 **downward API**로 자기 메타데이터(Pod 이름/네임스페이스/IP, 노드 이름, 자기 requests/limits,
라벨/어노테이션)를 환경변수나 파일로 받을 수 있다. 앱이 자기가 어디서 도는지 알아야 할 때
(예: 자기 Pod IP를 클러스터에 등록) 쓴다.

## 더 읽을 곳
- [01-pod-lifecycle.md](01-pod-lifecycle.md) — SyncPod의 컨테이너 시작 순서
- [07-probes.md](07-probes.md) — 컨테이너 건강 검사
- [08-static-pods-shutdown.md](08-static-pods-shutdown.md) — 노드 종료 시 조율
