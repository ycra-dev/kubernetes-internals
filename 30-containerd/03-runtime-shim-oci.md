# 30.03 · 런타임 — shim과 OCI(runc)

**근거**: `containerd/core/runtime/v2/`, `containerd/cmd/containerd-shim-runc-v2/`

준비된 rootfs([02](02-content-snapshots.md)) 위에서 실제 프로세스를 띄우는 단계다. containerd는 컨테이너
프로세스를 직접 자식으로 갖지 않고, **shim**이라는 중간 프로세스를 거쳐 **OCI 런타임(runc)** 을 호출한다.

## 왜 shim이 필요한가

containerd 데몬이 컨테이너의 부모 프로세스라면, **데몬을 업그레이드/재시작할 때 모든 컨테이너가
죽는다.** 이를 피하려고 컨테이너마다 가벼운 **shim**(`containerd-shim-runc-v2`)을 두고, shim이
컨테이너 프로세스의 부모가 된다. 그래서:

- containerd 데몬을 재시작해도 실행 중인 컨테이너는 살아 있다(shim이 붙들고 있음).
- shim이 컨테이너의 stdio, exit code 수집, 라이프사이클을 관리하고 containerd와 gRPC/ttRPC로 통신한다.

## Runtime v2 (shim API)

`core/runtime/v2/`가 shim과 데몬 사이의 표준(Runtime v2)이다:

- `binary.go`, `manager_unix.go`: shim 바이너리를 찾아 띄우고 관리.
- `bundle.go`(+`bundle_linux.go`): **OCI 번들** 생성 — rootfs 마운트 정보 + `config.json`(OCI 런타임
  스펙: 명령, 환경변수, 마운트, cgroup, capabilities, 네임스페이스)을 디렉토리로 구성.

`config.json`(OCI 스펙)에는 [13-kubelet/03](../13-kubelet/03-resource-mgmt.md)에서 정한 cgroup 제한,
보안 컨텍스트(특권/capabilities), 마운트(볼륨, [13-kubelet/05](../13-kubelet/05-volume-manager.md))가
반영된다.

## runc — OCI 런타임

shim은 OCI 번들을 받아 **runc**(또는 다른 OCI 런타임)를 호출한다. runc는 OCI 스펙을 읽어 리눅스
원시 기능으로 컨테이너를 만든다:

- **namespaces**: PID/네트워크/마운트/IPC/UTS 격리.
- **cgroups**: 자원 제한 적용.
- **capabilities / seccomp / AppArmor**: 권한 축소.
- pivot_root로 rootfs를 바꿔 컨테이너 파일시스템으로 진입.

runc는 컨테이너 프로세스를 만든 뒤 자신은 빠진다("create then exit") — 컨테이너 프로세스는 shim의
자식으로 남는다.

```
containerd 데몬
   └─ shim (컨테이너당, 계속 살아있음)
        └─ runc create/start  (잠깐 실행 후 종료)
             └─ 컨테이너 프로세스 (shim이 reparent해 관리)
```

## 다른 런타임 — RuntimeClass

shim API는 runc 외의 런타임도 지원한다. 예: **gVisor(runsc)**, **Kata Containers**(경량 VM). 이들은
각자의 shim을 제공하고, Kubernetes의 **RuntimeClass**([13-kubelet/02](../13-kubelet/02-cri.md))로
"이 Pod는 더 강한 격리(VM)로"처럼 선택할 수 있다.

## 더 읽을 곳
- [02-content-snapshots.md](02-content-snapshots.md) — rootfs 준비(앞 단계)
- [13-kubelet/03](../13-kubelet/03-resource-mgmt.md) — cgroup 제한의 출처
