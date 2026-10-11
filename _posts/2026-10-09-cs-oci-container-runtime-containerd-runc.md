---
layout: post
title: "OCI Container Runtime 심화: containerd와 runc 내부 동작 분석"
date: 2026-10-09
categories: [cs, computer-science]
tags: [container, OCI, containerd, runc, docker, kubernetes, runtime, linux]
---

## 개요

"도커 컨테이너 실행"이라는 간단한 명령 뒤에는 놀랍도록 정교한 소프트웨어 계층 구조가 존재합니다. `docker run nginx` 한 줄을 입력하면, 요청은 Docker 데몬(dockerd) → containerd → containerd-shim → runc → Linux 커널 순으로 전달되며, 각 계층은 명확하게 분리된 책임을 집니다.

이 글은 OCI(Open Container Initiative) 표준이 정의하는 컨테이너 런타임의 계층 구조와, 실제 구현체인 **containerd**와 **runc**의 내부 동작을 분석합니다.

---

## OCI 표준: 컨테이너 생태계의 공통 언어

2015년 Docker와 CoreOS가 갈등을 빚던 시절, 컨테이너 포맷과 런타임에 대한 통일된 표준이 필요했습니다. Linux Foundation 산하에 **OCI(Open Container Initiative)**가 설립되어 두 가지 핵심 명세를 정의했습니다.

### OCI 이미지 명세 (Image Specification)
컨테이너 이미지를 어떻게 패키징하고 배포할지 정의합니다.
- **레이어 기반 파일시스템**: 각 레이어는 변경분(diff)만 저장
- **매니페스트(Manifest)**: 이미지 메타데이터와 레이어 목록
- **인덱스**: 멀티 플랫폼 이미지 지원

### OCI 런타임 명세 (Runtime Specification)
컨테이너를 **어떻게 실행**할지 정의합니다. 핵심은 `config.json`이라는 JSON 파일입니다.

```json
// config.json: OCI 런타임 명세의 핵심 파일 구조
{
  "ociVersion": "1.0.2",
  "process": {
    "terminal": false,
    "user": { "uid": 0, "gid": 0 },
    "args": ["/bin/sh", "-c", "echo hello"],
    "env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
      "TERM=xterm"
    ],
    "cwd": "/",
    "capabilities": {
      "bounding": ["CAP_AUDIT_WRITE", "CAP_KILL", "CAP_NET_BIND_SERVICE"],
      "effective": ["CAP_AUDIT_WRITE", "CAP_KILL"],
      "permitted": ["CAP_AUDIT_WRITE", "CAP_KILL", "CAP_NET_BIND_SERVICE"]
    }
  },
  "root": {
    "path": "rootfs",        // 컨테이너 루트 파일시스템 경로
    "readonly": true
  },
  "mounts": [
    { "destination": "/proc", "type": "proc", "source": "proc" },
    { "destination": "/dev", "type": "tmpfs", "source": "tmpfs",
      "options": ["nosuid", "strictatime", "mode=755", "size=65536k"] }
  ],
  "linux": {
    "namespaces": [
      { "type": "pid" },
      { "type": "network" },
      { "type": "ipc" },
      { "type": "uts" },
      { "type": "mount" }
    ],
    "resources": {
      "memory": { "limit": 536870912 },   // 512MB 메모리 제한
      "cpu": { "shares": 1024 }
    },
    "seccomp": { /* 허용된 시스템 콜 목록 */ }
  }
}
```

이 명세 덕분에 Docker, Podman, CRI-O 등 다양한 컨테이너 플랫폼이 동일한 이미지를 실행할 수 있습니다.

---

## 런타임 계층 구조

컨테이너 런타임은 **High-Level Runtime**과 **Low-Level Runtime**으로 구분됩니다.

```
┌─────────────────────────────────────────────────────────────┐
│                    사용자/오케스트레이터                         │
│                 (kubectl, docker CLI 등)                      │
└─────────────────────────┬───────────────────────────────────┘
                          │ gRPC / REST API
┌─────────────────────────▼───────────────────────────────────┐
│              High-Level Runtime (Docker, CRI-O)              │
│   이미지 관리, 네트워킹, 볼륨, 레지스트리 통신, API 제공              │
└─────────────────────────┬───────────────────────────────────┘
                          │ gRPC (CRI)
┌─────────────────────────▼───────────────────────────────────┐
│               containerd (High-Level Core)                   │
│   컨테이너 생명주기, 스냅샷, 이미지 스토리지, 플러그인 시스템           │
└─────────────────────────┬───────────────────────────────────┘
                          │ stdin/stdout/stderr + API
┌─────────────────────────▼───────────────────────────────────┐
│                  containerd-shim-runc-v2                     │
│   프로세스 격리, 재시작 내성, 로그 수집, 이벤트 발행                   │
└─────────────────────────┬───────────────────────────────────┘
                          │ OCI Runtime Spec (config.json)
┌─────────────────────────▼───────────────────────────────────┐
│                runc (Low-Level Runtime)                      │
│   네임스페이스, cgroup, 마운트, seccomp, capabilities 설정         │
└─────────────────────────┬───────────────────────────────────┘
                          │ syscall
┌─────────────────────────▼───────────────────────────────────┐
│                     Linux 커널                               │
│   실제 격리 메커니즘: namespaces, cgroups, seccomp, LSM         │
└─────────────────────────────────────────────────────────────┘
```

---

## containerd 심층 분석

### 아키텍처

containerd는 **플러그인 기반 아키텍처**를 채택한 데몬으로, `/run/containerd/containerd.sock` UNIX 소켓을 통해 gRPC API를 노출합니다.

```
containerd 내부 구조:

┌──────────────────────────────────────────────────┐
│                    gRPC API                      │
│  (ContainerService, ImageService, SnapshotService│
│   TaskService, EventService, ContentService ...)  │
└──────────────────────┬───────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────┐
│                 Plugin System                     │
│  ┌──────────┐ ┌──────────┐ ┌────────────────┐   │
│  │ Runtime  │ │Snapshotter│ │  Content Store │   │
│  │ Plugin   │ │  Plugin   │ │    Plugin      │   │
│  │(runc,    │ │(overlayfs,│ │(local storage) │   │
│  │ kata,    │ │  btrfs)   │ │                │   │
│  │ gvisor)  │ └──────────┘ └────────────────┘   │
│  └──────────┘                                    │
└──────────────────────────────────────────────────┘
```

### 핵심 컴포넌트

**1. Content Store**
이미지 레이어와 설정 파일을 내용 주소화(content-addressable) 방식으로 저장합니다.

```bash
# containerd content store 구조
/var/lib/containerd/io.containerd.content.v1.content/
└── blobs/
    └── sha256/
        ├── abc123...  # 이미지 매니페스트 (JSON)
        ├── def456...  # 설정 레이어 (JSON)
        └── ghi789...  # 파일시스템 레이어 (tar.gz)
```

**2. Snapshotter**
컨테이너 루트 파일시스템을 관리합니다. 기본값은 `overlayfs`입니다.

```bash
# overlayfs 스냅샷 실제 구조
/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/
└── snapshots/
    ├── 1/  # 베이스 레이어 스냅샷
    │   ├── fs/   # 실제 파일들
    │   └── work/ # overlayfs 워크 디렉토리
    ├── 2/  # 두 번째 레이어
    └── 3/  # 컨테이너 쓰기 가능 레이어
```

**3. Task와 Shim**

containerd가 컨테이너를 실행할 때의 흐름:

```go
// containerd 내부 Task 생성 흐름 (개념적 코드)
func (r *runtime) Create(ctx context.Context, id string, opts ...containerd.NewTaskOpts) (containerd.Task, error) {
    // 1. 스냅샷에서 번들 디렉토리 준비
    bundle, err := r.newBundle(id)

    // 2. OCI spec(config.json) 생성
    spec, err := generateSpec(container, opts...)
    bundle.SaveSpec(spec)

    // 3. containerd-shim 프로세스 생성
    shim, err := r.startShim(ctx, bundle)

    // 4. Shim을 통해 runc 실행 (gRPC 또는 ttrpc)
    response, err := shim.Create(ctx, &task.CreateTaskRequest{
        ID:         id,
        Bundle:     bundle.Path,
        Runtime:    "io.containerd.runc.v2",
        Rootfs:     mounts,
        Terminal:   opts.Terminal,
        Stdin:      opts.Stdin,
        Stdout:     opts.Stdout,
        Stderr:     opts.Stderr,
    })

    return &task{shim: shim, ...}, nil
}
```

### containerd-shim의 역할

**Shim**은 containerd 데몬과 실제 컨테이너 프로세스 사이의 **중개자**입니다. 이 설계의 핵심 이점은 **데몬 재시작 내성(restart resilience)**입니다.

```
데몬 없이도 컨테이너 계속 실행:

containerd 재시작
    │
    ▼
shim 프로세스는 계속 살아있음 (고아 프로세스가 됨)
    │
    ▼
컨테이너 프로세스도 계속 실행됨
    │
    ▼
containerd 재기동 시 shim 재연결
```

```bash
# 실제 프로세스 트리 예시 (pstree 출력)
systemd
└── containerd
    └── containerd-shim-runc-v2
        └── [container init process]
            └── [application process]
```

---

## runc 심층 분석

runc는 OCI Runtime Specification의 **참조 구현체(reference implementation)**로, 순수 Go로 작성된 약 300KB의 바이너리입니다.

### 컨테이너 생명주기 상태 기계

OCI 명세는 컨테이너 상태를 엄격하게 정의합니다:

```
[creating] → [created] → [running] → [stopped]
                ↑
           (hooks 실행)
```

```bash
# runc 명령어로 직접 컨테이너 조작
mkdir -p /mycontainer/rootfs

# busybox 루트 파일시스템 준비
docker export $(docker create busybox) | tar -C /mycontainer/rootfs -xvf -

# OCI spec 생성
cd /mycontainer
runc spec  # config.json 자동 생성

# 컨테이너 생성 (running 상태 전)
runc create mycontainer-1

# 컨테이너 상태 확인
runc state mycontainer-1
# {
#   "ociVersion": "1.0.2",
#   "id": "mycontainer-1",
#   "status": "created",    ← created 상태
#   "pid": 4321,
#   "bundle": "/mycontainer"
# }

# 컨테이너 시작 (init 프로세스 unblock)
runc start mycontainer-1

# 실행 중인 컨테이너에 명령 실행
runc exec mycontainer-1 ls /

# 컨테이너 삭제
runc delete mycontainer-1
```

### runc의 내부 실행 흐름

`runc run` 명령이 실행될 때의 내부 흐름:

```go
// runc 내부 실행 순서 (libcontainer 기반, 단순화)
func (c *linuxContainer) Start(process *Process) error {

    // 1. 자식 프로세스 생성 (clone 시스템 콜)
    //    새로운 네임스페이스 플래그 포함
    //    CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWMNT | CLONE_NEWUTS | CLONE_NEWIPC
    cmd := exec.Command("/proc/self/exe", "init")
    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWPID |
                    syscall.CLONE_NEWNET |
                    syscall.CLONE_NEWNS  | // Mount namespace
                    syscall.CLONE_NEWUTS |
                    syscall.CLONE_NEWIPC,
    }

    // 2. 부모-자식 간 파이프 생성 (동기화용)
    parentPipe, childPipe, _ := os.Pipe()

    // 3. 자식 프로세스 시작 (새 네임스페이스에서 "init" 서브커맨드 실행)
    cmd.Start()

    // 4. cgroup 설정 (v1: cgroupfs, v2: unified hierarchy)
    c.cgroupManager.Apply(cmd.Process.Pid)
    c.cgroupManager.Set(c.config.Resources)

    // 5. 네트워크 설정 (네임스페이스 안에서 veth 페어 구성 등)
    c.networkManager.Initialize(cmd.Process.Pid)

    // 6. 자식 프로세스에 계속 진행 신호 (파이프 통해)
    parentPipe.Write([]byte("continue"))

    // [자식 프로세스 "init" 측]
    // 7. 루트 파일시스템 마운트 (pivot_root 또는 chroot)
    // 8. seccomp 프로파일 적용
    // 9. capabilities 조정
    // 10. 실제 컨테이너 프로세스 execve()
    return nil
}
```

### 네임스페이스와 cgroup 설정 상세

```bash
# 네임스페이스 확인 (호스트에서)
ls -la /proc/$(docker inspect --format '{{.State.Pid}}' mycontainer)/ns/
# lrwxrwxrwx ... cgroup -> cgroup:[4026531835]
# lrwxrwxrwx ... ipc    -> ipc:[4026532267]    ← 격리된 IPC
# lrwxrwxrwx ... mnt    -> mnt:[4026532265]    ← 격리된 마운트
# lrwxrwxrwx ... net    -> net:[4026532270]    ← 격리된 네트워크
# lrwxrwxrwx ... pid    -> pid:[4026532268]    ← 격리된 PID
# lrwxrwxrwx ... uts    -> uts:[4026532266]    ← 격리된 호스트명

# cgroup v2 리소스 제한 확인
cat /sys/fs/cgroup/system.slice/docker-abc123.scope/memory.max
# 536870912  (512MB)

cat /sys/fs/cgroup/system.slice/docker-abc123.scope/cpu.max
# 50000 100000  (CPU 50% 제한)
```

---

## 대안 런타임: kata-containers와 gVisor

OCI 명세의 강점은 pluggable runtime을 허용한다는 점입니다.

```yaml
# containerd 설정에서 대안 런타임 지정
# /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes]

  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    runtime_type = "io.containerd.runc.v2"

  # Kata Containers: VM 기반 강력한 격리
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
    runtime_type = "io.containerd.kata.v2"

  # gVisor: 사용자 공간 커널으로 syscall 가로채기
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
    runtime_type = "io.containerd.runsc.v1"
```

| 런타임 | 격리 방식 | 성능 오버헤드 | 보안 강도 | 사용 사례 |
|--------|---------|------------|---------|---------|
| runc | 네임스페이스/cgroup | 매우 낮음 | 보통 | 일반 워크로드 |
| kata-containers | 경량 VM (QEMU/Firecracker) | 중간 | 높음 | 멀티테넌트, 금융 |
| gVisor | 사용자 공간 커널 | 낮음-중간 | 높음 | 신뢰할 수 없는 코드 |

---

## 실전 디버깅: 컨테이너 런타임 문제 진단

```bash
# containerd 상태 확인
systemctl status containerd
journalctl -u containerd --since "1 hour ago"

# containerd 이벤트 스트리밍 (실시간 모니터링)
ctr events

# 실행 중인 태스크 목록
ctr task ls

# 특정 컨테이너의 shim 프로세스 확인
ps aux | grep containerd-shim

# runc 직접 상태 조회
runc --root /run/containerd/runc/k8s.io list

# 컨테이너 네임스페이스로 진입 (디버깅)
nsenter -t <PID> --mount --pid --net -- /bin/sh

# cgroup 자원 사용량 실시간 확인
systemd-cgtop
```

---

## 주의사항 및 실전 팁

1. **Shim 프로세스 누수**: containerd 업그레이드 시 구 shim이 남아있을 수 있음. `ps aux | grep shim`으로 확인 후 정리 필요.

2. **cgroup v1 vs v2**: 최신 배포판은 cgroup v2 통합 계층구조(unified hierarchy)를 기본 사용. 레거시 도구와의 호환성 주의.

3. **seccomp 기본 프로파일**: Docker/containerd의 기본 seccomp 프로파일은 300+ 시스템 콜 중 약 40개를 차단. 필요 시 커스텀 프로파일 작성.

4. **rootless 컨테이너**: 루트 권한 없이 컨테이너 실행(user namespace 활용). 보안 강화이나 일부 기능 제한됨.

---

## 참고 자료

- [runc 공식 GitHub 저장소](https://github.com/opencontainers/runc/blob/main/README.md)
- [containerd 공식 GitHub 저장소](https://github.com/containerd/containerd/blob/main/README.md)
- [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec/blob/main/spec.md)
- [containerd 아키텍처 설계 문서](https://github.com/containerd/containerd/blob/main/docs/design/architecture.md)
