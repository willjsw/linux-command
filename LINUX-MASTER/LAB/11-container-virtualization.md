---
title: LAB 11 — 컨테이너(Docker)·가상화(libvirt)
type: exam-lab
part: 11
tags:
  - exam/linux-master
  - exam/lab
  - linux/container
  - linux/virtualization
  - linux/package
  - task/configure
  - task/verify
related: ["[[README]]", "[[10-security-firewall-selinux]]", "[[12-backup-recovery-review]]", "[[../THEORY/network-service]]", "[[../THEORY/package-software]]", "[[../../CONTAINER/docker]]", "[[../../DATABASE/redis-cli]]"]
distro: Rocky Linux 9 (aarch64, UTM)
updated: 2026-09-03
---

# LAB 11 — 컨테이너(Docker)·가상화(libvirt)

- 가상화 4계층(전가상화·반가상화·컨테이너·하이퍼바이저 Type1/2)을 표로 고정하고, 컨테이너 기반 기술(namespace·cgroup·overlayfs)을 **호스트에서 직접 눈으로 확인**
- `docker-ce` 저장소 등록 → 설치 → `daemon.json` 튜닝 → `docker` 그룹 권한 위임(및 그 위험)까지 운영 관점으로 구성
- Part 09 인트라넷 웹이 쓸 캐시 `lab-redis` 를 볼륨·루프백 바인딩·인증 포함으로 기동하고, **컨테이너를 지웠다 다시 만들어도 데이터가 남는지**로 볼륨을 증명
- Ubuntu 컨테이너 `lab-ubuntu` 안에서 `apt`·`dpkg` 전 범위를 실습(기출 필수) → `docker commit` 으로 이미지화. 마지막에 `libvirt`/`virsh` 는 **KVM 불가 환경임을 증거와 함께 기록**하고 조회 수준으로 진행

> **이 파트의 시나리오**: Part 09 에서 인트라넷 웹(`intranet.lab.local`)을, Part 10 에서 방화벽·SELinux 를 갖췄다. 웹이 세션·조회 결과를 캐싱할 Redis 가 필요한데 RPM 저장소 버전에 묶이기 싫어 **Docker 컨테이너**로 올린다. 또 팀 일부가 Ubuntu 를 쓰므로 같은 서버 안에 Ubuntu 컨테이너를 띄워 `apt`/`dpkg` 를 익힌다. 끝으로 "왜 이 서버에서는 KVM 가상머신을 못 만드는가" 를 명령 출력으로 남겨 둔다.

- 선행 자원: 사용자 `dev1`(Part 03), `/srv/devteam/proj`(Part 04), `/srv/share`(Part 05), firewalld 커스텀 서비스 `lab-redis`(Part 10), SELinux enforcing(Part 10)
- 이 파트에서 만든 `lab-redis` 컨테이너와 `redis-data` 볼륨은 **[[12-backup-recovery-review]] 의 백업 대상** → 마지막 정리 단계에서 지우지 않음
- ⚠️ 9절 `docker system prune` 은 "미사용" 판정 범위가 넓음 — 실행 전 반드시 대상 목록 확인

---

## 1. 개념 정리 — 가상화와 컨테이너

### 1-1. 가상화 유형 — 전가상화 · 반가상화 · 컨테이너

> **상황**: 필기 기출에서 가장 자주 나오는 3분류다. 명령 실습에 들어가기 전에 "게스트 커널을 수정하는가", "하드웨어를 흉내 내는가" 두 축으로 표를 고정한다.

| 구분 | 전가상화 (Full) | 반가상화 (Para) | 컨테이너 (OS 수준) |
| --- | --- | --- | --- |
| 게스트 OS 수정 | **불필요** (수정 없이 그대로 부팅) | **필요** (커널이 하이퍼콜 사용) | 게스트 OS 자체가 없음 |
| 하드웨어 처리 | 하이퍼바이저가 완전 에뮬레이션 | 게스트가 하이퍼바이저에 직접 요청(hypercall) | 호스트 커널 시스템콜 직접 사용 |
| 게스트의 인지 | 가상 환경임을 **인지 못함** | 가상 환경임을 **인지함** | 격리된 프로세스 |
| CPU 확장 기능 | VT-x/AMD-V 있으면 성능 급상승(HVM) | 필수 아님 | 무관 |
| 성능 | 에뮬레이션 오버헤드 존재 | 전가상화보다 오버헤드 적음 | 거의 네이티브 |
| 예 | KVM+QEMU, VMware, Hyper-V | Xen PV, **virtio** 드라이버 | Docker, Podman, LXC |

- **virtio** : 전가상화 VM 안에서 디스크·NIC 만 반가상화 방식으로 처리해 I/O 성능을 올리는 드라이버 → 이 실습 VM 의 `/dev/vda`·`enp0s1` 이 바로 virtio 장치 (Part 01·05 의 `vd*` 표기 이유)
- 하드웨어 지원 가상화(HVM) : CPU 의 VT-x(Intel `vmx`)·AMD-V(`svm`) 확장으로 전가상화의 에뮬레이션 비용을 줄이는 방식 — **반가상화 전용 기술이 아님**(기출 오답 선지 단골)

```bash
# x86_64 에서의 확인 방법 (참고 — 이 aarch64 VM 에는 해당 플래그 없음)
grep -oE 'vmx|svm' /proc/cpuinfo | sort -u
lsmod | grep -E '^kvm'
```

**검증**

```bash
uname -m                       # 아키텍처 — aarch64
grep -c -E 'vmx|svm' /proc/cpuinfo   # x86 가상화 플래그 개수 (aarch64 는 0)
lsmod | grep -c '^kvm'         # kvm 모듈 적재 개수
```

```text
# uname -m
aarch64
# grep -c -E 'vmx|svm' /proc/cpuinfo
0
# lsmod | grep -c '^kvm'
0
```

> 📝 **시험 포인트**: "전가상화 = 게스트 수정 불필요", "반가상화 = 커널 수정 + 하이퍼콜", "VT-x/AMD-V 는 전가상화 성능 보조" 세 문장이 그대로 정답 선지. `virtio` 를 반가상화 개념의 응용으로 묶는 문항(필기 R10 #87)도 출제됨.

### 1-2. 하이퍼바이저 Type 1 / Type 2 와 제품 분류

> **상황**: "베어메탈이냐 호스트형이냐" 는 제품 이름과 함께 묻는다. KVM 처럼 분류가 애매한 것을 정리해 둔다.

| 구분 | Type 1 (베어메탈, Native) | Type 2 (호스트형, Hosted) |
| --- | --- | --- |
| 위치 | 하드웨어 **바로 위**에 하이퍼바이저 | 호스트 OS 위에 **응용 프로그램처럼** 설치 |
| 성능 | 높음 (중간 계층 없음) | 상대적으로 낮음 |
| 용도 | 서버·데이터센터 | 데스크톱·개발·학습 |
| 예 | VMware **ESXi**, Xen, Microsoft **Hyper-V**, KVM(*) | **VirtualBox**, VMware **Workstation/Player/Fusion**, QEMU(단독), Parallels |

- (*) **KVM** : 리눅스 커널 자체가 하이퍼바이저가 되는 커널 모듈(`kvm.ko` + `kvm_intel.ko`/`kvm_amd.ko`) → 통상 **Type 1 로 분류**. "호스트 OS 위" 처럼 보이지만 커널에 내장되므로 베어메탈에 준함
- **QEMU** : 순수 에뮬레이터(다른 아키텍처도 흉내 가능, 느림). KVM 과 결합하면 QEMU 가 장치 모델을, KVM 이 CPU/메모리 가상화를 담당 → `qemu-kvm`
- **Xen** : Type 1. Dom0(관리 도메인) + DomU(게스트). PV(반가상화)와 HVM(전가상화) 모두 지원, CLI 는 `xl`
- **libvirt** : 하이퍼바이저 종류에 상관없이 같은 API 로 다루는 **관리 계층** — 하이퍼바이저가 아님. 데몬 `libvirtd`(RHEL 9 는 모듈식 `virtqemud` 등), CLI `virsh`, GUI `virt-manager`
- 클라우드 서비스 모델 : **IaaS**(가상 서버·스토리지·네트워크 = 인프라 제공) / **PaaS**(실행 플랫폼) / **SaaS**(완성 소프트웨어) / **FaaS**(함수 단위 실행)

```bash
# 이 실습 VM 이 어떤 하이퍼바이저 위에 있는지 확인
systemd-detect-virt          # 감지된 가상화 기술 이름
hostnamectl | grep -i virt   # Virtualization 행
```

- `systemd-detect-virt` : 실행 환경의 가상화 종류 출력 — 물리 머신이면 `none` 이고 종료 코드 1
	- `-c` : 컨테이너만 판정 (**c**ontainer), `-v` : VM 만 판정 (**v**m), `-q` : 출력 없이 종료 코드만 (**q**uiet)

**검증**

```bash
systemd-detect-virt; echo "rc=$?"
systemd-detect-virt -c || echo "컨테이너 아님"
```

```text
# systemd-detect-virt
apple            ← UTM 의 Apple Virtualization 백엔드. QEMU 백엔드면 qemu 또는 kvm
rc=0
# systemd-detect-virt -c
none
컨테이너 아님
```

> 📝 **시험 포인트**: Type 2 를 고르는 문항(필기 R02 #87)의 정답은 **VirtualBox**. ESXi·Hyper-V·Xen 은 Type 1. "커널 모듈 형태 + VT-x/AMD-V 활용" 서술은 무조건 **KVM**(필기 R01 #89, R06 #89).

### 1-3. 컨테이너 vs 가상 머신

> **상황**: Redis 를 VM 이 아니라 컨테이너로 올리기로 한 근거를 정리한다. 격리 수준 비교는 필기 단골이다.

| 항목 | 가상 머신 (VM) | 컨테이너 |
| --- | --- | --- |
| 커널 | 게스트마다 **별도 커널 부팅** | 호스트 커널 **공유** |
| 격리 수단 | 하이퍼바이저(하드웨어 수준) | **namespace**(격리) + **cgroup**(자원 제한) |
| 격리 강도 | 강함 — 게스트 커널 취약점이 다른 VM 에 직접 전이되지 않음 | 상대적으로 약함 — **호스트 커널 취약점이 전 컨테이너에 영향** |
| 부팅 시간 | 수십 초 (BIOS→커널→init) | 수십 ms ~ 수 초 (프로세스 기동) |
| 이미지 크기 | GB 단위 (OS 전체) | MB 단위 (애플리케이션 + 최소 런타임) |
| 오버헤드 | CPU·메모리 예약, 에뮬레이션 비용 | 거의 없음 (호스트 프로세스와 동일) |
| 배치 밀도 | 낮음 | 높음 (한 호스트에 수십~수백) |
| 게스트 OS 선택 | 호스트와 다른 OS 가능 (Windows 등) | **호스트 커널을 쓰는 OS 만** (리눅스 호스트 → 리눅스 배포판) |
| 대표 도구 | KVM, VMware, Hyper-V, VirtualBox | Docker, Podman, LXC, containerd |

- Ubuntu 컨테이너가 Rocky 호스트에서 도는 원리 : 컨테이너 이미지는 **커널을 포함하지 않고 사용자 공간(userland)만** 담음 → `apt`·`dpkg`·glibc 는 Ubuntu 것, 커널은 호스트 Rocky 것 (6절에서 직접 확인)
- 그래서 컨테이너는 "가벼운 VM" 이 아니라 **격리된 프로세스**

**검증**

```bash
uptime -p                                   # 호스트 가동 시간
systemd-analyze | head -1                   # 호스트 부팅에 걸린 시간 (비교 기준)
```

```text
# uptime -p
up 2 hours, 13 minutes
# systemd-analyze
Startup finished in 4.1s (kernel) + 21.3s (initrd) + 33.4s (userspace) = 58.9s
```

- 위 58.9초와 4절에서 측정할 컨테이너 기동 시간(1초 미만)을 비교 → 부팅 시간 차이의 실체

> 📝 **시험 포인트**: "namespace 가 격리, cgroup 이 자원 제한" 을 **뒤집어 놓은 선지**가 오답으로 자주 나옴(필기 R10 #88 ④). "컨테이너가 VM 보다 항상 강한 격리" 도 오답(필기 R09 #89).

### 1-4. 리눅스 컨테이너의 기반 기술 — namespace · cgroup · chroot · overlayfs

> **상황**: 컨테이너는 Docker 가 발명한 것이 아니라 커널 기능의 조합이다. 각 기능이 무엇을 격리하는지 표로 잡고, 호스트에서 실제 파일로 확인한다.

**namespace 6종(+2)** — "무엇이 따로 보이는가"

| 네임스페이스 | 격리 대상 | 컨테이너에서의 효과 |
| --- | --- | --- |
| `mnt` (Mount) | 마운트 지점 목록 | 컨테이너만의 루트 파일시스템 |
| `pid` (PID) | 프로세스 ID 공간 | 컨테이너 내부 첫 프로세스가 **PID 1** |
| `net` (Network) | NIC·IP·라우팅·포트 | 컨테이너 전용 `eth0`, 독립 포트 공간 |
| `ipc` (IPC) | System V IPC, POSIX 메시지 큐 | 공유 메모리·세마포어 분리 |
| `uts` (UTS) | 호스트명·도메인명 | `--hostname` 으로 별도 호스트명 |
| `user` (User) | UID/GID 매핑 | 컨테이너 root(0) ↔ 호스트 비특권 UID (rootless 의 핵심) |
| `cgroup` | cgroup 루트 시야 | 컨테이너가 자기 cgroup 만 봄 |
| `time` | 시스템 시각 오프셋 | 커널 5.6+, Docker 미사용 |

- **cgroup**(control group) v2 : CPU·메모리·PID 수·블록 I/O **사용량 제한과 계측**. 격리가 아니라 **자원 통제**가 역할 → `/sys/fs/cgroup` (7-6 에서 파일 직접 확인)
- **chroot** : 프로세스의 루트 디렉터리를 바꾸는 원시적 격리. 프로세스·네트워크는 그대로 보이므로 **컨테이너가 아님**(탈출 기법도 알려짐) → 컨테이너는 `pivot_root` 로 옛 루트를 언마운트해 되돌아갈 경로 자체를 제거
- **overlayfs** : 읽기 전용 하위 레이어(`lowerdir`) + 쓰기 가능 상위 레이어(`upperdir`)를 합쳐 하나로 보여 주는 유니온 파일시스템 → 이미지 레이어 공유와 컨테이너 쓰기 레이어의 실체. Docker 스토리지 드라이버 `overlay2`

```bash
ls /sys/fs/cgroup/cgroup.controllers && cat /sys/fs/cgroup/cgroup.controllers   # cgroup v2 컨트롤러 목록
stat -fc %T /sys/fs/cgroup                    # cgroup2fs 이면 v2 단일 계층
ls -l /proc/1/ns/                             # PID 1(systemd) 의 네임스페이스 목록
lsns | head                                   # 시스템 전체 네임스페이스 목록
grep -E 'overlay' /proc/filesystems           # overlayfs 커널 지원 여부
```

- `stat -fc %T` : 파일시스템 유형 문자열 (**f**ilesystem, **c** 사용자 지정 형식, `%T` = 유형)
- `lsns` : 네임스페이스 목록 (**l**i**s**t **n**ame**s**paces, `util-linux`)
	- `-t` : 유형 필터 (**t**ype, 예 `-t net`), `-p` : 특정 PID 의 네임스페이스 (**p**id)

**검증**

```bash
stat -fc %T /sys/fs/cgroup
readlink /proc/1/ns/pid /proc/1/ns/net        # PID 1 의 pid·net 네임스페이스 inode
lsns -t net | wc -l                           # 현재 network 네임스페이스 개수(호스트만이면 2줄)
```

```text
# stat -fc %T /sys/fs/cgroup
cgroup2fs
# cat /sys/fs/cgroup/cgroup.controllers
cpuset cpu io memory hugetlb pids rdma misc
# readlink /proc/1/ns/pid /proc/1/ns/net
pid:[4026531836]
net:[4026531840]
# ls -l /proc/1/ns/
cgroup -> 'cgroup:[4026531835]'  ipc -> 'ipc:[4026531839]'  mnt -> 'mnt:[4026531841]'
net -> 'net:[4026531840]'  pid -> 'pid:[4026531836]'  user -> 'user:[4026531837]'  uts -> 'uts:[4026531838]'
```

- 위 inode 번호를 기억해 둘 것 — 4-11 에서 컨테이너 프로세스의 같은 파일과 비교하면 어떤 네임스페이스가 갈렸는지 즉시 보임

> 📝 **시험 포인트**: namespace 6종 이름과 격리 대상 짝짓기, "chroot 는 컨테이너가 아니다", "cgroup = 자원 제한" 세 가지가 출제 축. 필기 R01 #89 의 오답 선지에 `chroot` 가 들어감.

### 1-5. Docker 아키텍처와 구성 요소

> **상황**: `docker` 를 쳤을 때 실제로 무엇이 무엇을 호출하는지 알아야 "데몬이 죽었는데 CLI 는 살아 있는" 상황을 이해한다.

```
docker (CLI)  ──REST/Unix 소켓──▶  dockerd (데몬)
   /var/run/docker.sock                 │  이미지·네트워크·볼륨 관리
                                        ▼
                                   containerd (컨테이너 런타임 관리)
                                        │  이미지 pull, 스냅샷, 수명주기
                                        ▼
                                   containerd-shim ─▶ runc (OCI 런타임)
                                                        │ namespace·cgroup 생성
                                                        ▼
                                                    컨테이너 프로세스
```

| 구성 요소 | 역할 |
| --- | --- |
| `docker` (CLI) | 사용자 명령을 REST API 요청으로 변환 — **클라이언트일 뿐** |
| `dockerd` | 데몬. 이미지·컨테이너·볼륨·네트워크 전체 관리, API 제공 |
| `containerd` | 표준 컨테이너 런타임 관리자. 이미지 배포·스냅샷·수명주기 |
| `containerd-shim` | 컨테이너별 중개 프로세스 — dockerd 재시작에도 컨테이너 생존(`live-restore`) |
| `runc` | OCI 런타임. 실제로 namespace·cgroup 을 만들고 프로세스 exec |
| `/var/run/docker.sock` | CLI ↔ 데몬 통신 유닉스 소켓 (권한 = 사실상 root) |

**오브젝트 5종**

| 오브젝트 | 정의 | 주요 명령 |
| --- | --- | --- |
| 이미지 (image) | **읽기 전용 레이어의 집합** + 메타데이터. 실행 템플릿 | `pull` `images` `build` `rmi` |
| 컨테이너 (container) | 이미지 위에 **쓰기 가능 레이어**를 얹은 실행 인스턴스 | `run` `ps` `start` `stop` `rm` |
| 레지스트리 (registry) | 이미지 저장·배포 서버 (Docker Hub, Harbor, ECR) | `search` `pull` `push` `login` |
| 볼륨 (volume) | 컨테이너 **수명과 독립된** 데이터 저장소 | `volume create/ls/rm` |
| 네트워크 (network) | 컨테이너 간 통신 위한 가상 네트워크 | `network create/connect/ls` |

**docker 서브명령 총람** (관리 명령 + 단축형)

| 그룹 | 명령 |
| --- | --- |
| 이미지 | `images` `pull` `push` `build` `tag` `rmi` `save` `load` `import` `history` `search` `image <sub>` |
| 컨테이너 | `run` `create` `start` `stop` `restart` `pause` `unpause` `kill` `rm` `exec` `attach` `logs` `ps` `top` `stats` `port` `cp` `diff` `commit` `export` `rename` `update` `wait` `container <sub>` |
| 볼륨·네트워크 | `volume <sub>` `network <sub>` |
| 시스템 | `info` `version` `events` `system df` `system prune` `login` `logout` `inspect` `context` |
| 확장(플러그인) | `compose` `buildx` |

**검증**

```bash
which docker; ls -l /usr/bin/docker      # 설치 후 확인용 (2절 이후)
docker --help | head -20
```

```text
Usage:  docker [OPTIONS] COMMAND
Common Commands:
  run         Create and run a new container from an image
  exec        Execute a command in a running container
  ps          List containers
  build       Build an image from a Dockerfile
  pull        Download an image from a registry
  images      List images
  ...
```

> 📝 **시험 포인트**: "이미지는 읽기 전용 레이어 집합, 컨테이너는 그 위의 쓰기 레이어", "볼륨은 컨테이너 생명주기와 독립" 이 정답 문장(필기 R03 #85). 쿠버네티스 문항(R03 #87)에서는 **Pod = 최소 배포 단위**, **Service = Pod 집합에 대한 고정 접근 지점**.

### 1-6. OCI 표준과 Docker vs Podman

> **상황**: Rocky 9 는 기본으로 Podman 을 제공하는데도 이 실습은 `docker-ce` 를 쓴다. 시험이 Docker 명령을 묻기 때문이며, 두 도구의 차이를 알아 두면 설치 단계의 충돌도 이해된다.

- **OCI**(Open Container Initiative) 표준 3종
	- **Image Spec** : 이미지 레이어·매니페스트 형식 → 그래서 Docker 로 만든 이미지를 Podman 이 그대로 실행
	- **Runtime Spec** : 컨테이너 실행 방법 → `runc`, `crun`, `kata-runtime` 이 모두 호환
	- **Distribution Spec** : 레지스트리 API

| 항목 | Docker | Podman |
| --- | --- | --- |
| 데몬 | `dockerd` **상주 데몬 필요** | **데몬리스** — 명령이 곧 컨테이너의 부모 프로세스 |
| 권한 | 기본 root 데몬, `docker` 그룹 = 사실상 root | **rootless** 기본 지원 (user namespace 활용) |
| 런타임 | `containerd` + `runc` | `conmon` + `crun`/`runc` |
| systemd 연동 | `--restart` 정책 | `podman generate systemd` 로 유닛 생성 |
| 묶음 실행 | `docker compose` | `podman-compose`, `podman play kube` |
| 배포판 | 별도 저장소(`docker-ce`) | RHEL/Rocky **기본 저장소 포함** |
| 명령 문법 | 기준 | Docker 와 **거의 동일** (`alias docker=podman` 가능) |

```bash
dnf list --available podman buildah skopeo 2>/dev/null | tail -5   # Rocky 기본 저장소에 있는지
rpm -q podman || echo "podman 미설치"
```

**검증**

```bash
dnf info podman 2>/dev/null | grep -E '^(Name|Version|Repository)' 
```

```text
Name         : podman
Version      : 4.9.4
Repository   : appstream
```

> 📝 **시험 포인트**: 1급 필기는 Docker 명령 중심. Podman 은 "데몬리스·rootless" 두 키워드만 기억. OCI 는 "이미지·런타임 표준화 단체" 로 서술형에 등장 가능.

### 1-7. 현재 환경 확인 — 중첩 가상화 미지원 증거

> **상황**: 8절에서 `virsh` 로 VM 을 만들려고 하면 KVM 가속을 못 쓴다. 나중에 "왜 안 되지?" 로 헤매지 않도록 **지금 증거를 남긴다.**

```bash
ls -l /dev/kvm                    # KVM 문자 장치 — 있으면 하드웨어 가속 가능
lscpu | grep -iE 'virt|hypervisor|model name|architecture'
systemd-detect-virt               # 어떤 하이퍼바이저 위인지
dnf install -y virt-what          # 가상화 종류 판별 도구
virt-what                         # 감지 결과(여러 줄 가능)
dmidecode -s system-product-name  # SMBIOS 제품명 (UTM/QEMU 값)
dmidecode -s system-manufacturer
```

- `lscpu` : CPU 아키텍처 정보 (**l**i**s**t **cpu**) — x86 이면 `Virtualization: VT-x` 행이 나오지만 **aarch64 에는 해당 행이 없음**
- `virt-what` : 실행 중인 가상화 기술 이름 출력 (패키지 `virt-what`) — root 권한 필요
- `dmidecode -s <문자열>` : DMI/SMBIOS 단일 항목 조회 (**s**tring) — aarch64 UTM 에서는 SMBIOS 미제공으로 오류가 날 수 있음

**검증**

```bash
ls /dev/kvm 2>&1 || echo ">> /dev/kvm 없음 = 하드웨어 가속 불가"
lsmod | grep -E 'kvm' || echo ">> kvm 커널 모듈 미적재"
modprobe kvm 2>&1 | head -1 || true
```

```text
# ls -l /dev/kvm
ls: cannot access '/dev/kvm': No such file or directory
>> /dev/kvm 없음 = 하드웨어 가속 불가
# lsmod | grep -E 'kvm'
>> kvm 커널 모듈 미적재
# lscpu | grep -iE 'architecture|model name'
Architecture:            aarch64
Model name:              -
# systemd-detect-virt
apple
# virt-what
...
# dmidecode -s system-product-name
...        ← QEMU 백엔드면 "QEMU Virtual Machine", Apple 백엔드는 SMBIOS 미제공으로 빈 값/오류
```

- 결론 : **UTM(Apple Silicon)은 게스트에 중첩 가상화를 노출하지 않음** → 이 VM 안에서 KVM 가속 사용 불가
- 대안 : `libvirt` 도메인을 `<domain type='qemu'>`(TCG 소프트웨어 에뮬레이션)로 정의하면 **정의·조회·상태 전환**은 실습 가능. 실제 OS 설치는 극도로 느려 `※ 미실행`

> 📝 **시험 포인트**: "KVM 사용 전제 = CPU 가상화 확장(`vmx`/`svm`) + `kvm` 모듈 적재 + `/dev/kvm` 존재". 필기 R07 #86 의 절차 문제에서 **가상화 지원 확인이 가장 먼저**(ㄴ→ㄱ→ㄹ→ㄷ)인 이유가 이것.

---

## 2. Docker 설치와 데몬 설정

### 2-1. 충돌 패키지 확인·정리

> **상황**: Rocky 9 는 Podman 계열 도구를 기본 제공한다. `containerd.io` 는 배포판의 `runc` 와 파일이 겹쳐 설치가 거부될 수 있으므로 먼저 상태를 조사한다.

```bash
rpm -q podman buildah skopeo runc containerd docker             # 설치 여부 일괄 확인
rpm -qa | grep -E 'podman|buildah|runc|container' | sort        # 관련 패키지 전체
```

- `rpm -q <패키지>...` : 설치 여부·버전 질의 (**q**uery) — 미설치면 `package X is not installed`
- `rpm -qa` : 설치된 전체 패키지 (**a**ll)

⚠️ 아래 제거는 Podman 으로 만든 컨테이너·이미지가 있으면 함께 못 쓰게 됨. 이 실습 VM 에는 없으므로 진행

```bash
# 위 질의에서 "installed" 로 나온 것만 제거 (없으면 이 단계 생략)
dnf remove -y podman buildah runc
```

- `dnf remove` : 패키지와 그에 의존하는 패키지 제거
- 대안 — 제거 대신 설치 시 `dnf install --allowerasing …` 으로 충돌 패키지를 자동 교체 가능 (**allow erasing**)

**검증**

```bash
rpm -q runc podman
ls /usr/bin/runc 2>&1
```

```text
# rpm -q runc podman
package runc is not installed
package podman is not installed
# ls /usr/bin/runc
ls: cannot access '/usr/bin/runc': No such file or directory
```

> 📝 **시험 포인트**: RHEL 계열에서 `containerd.io` ↔ `runc` 충돌은 실무 단골. 시험에서는 "저수준 도구(`rpm`)는 의존성 자동 해결 불가, 고수준(`dnf`)은 가능" 개념으로 출제 → [[../THEORY/package-software]] 1-2.

### 2-2. docker-ce 저장소 등록

> **상황**: Docker Engine 은 Rocky 기본 저장소(BaseOS/AppStream)에 없다. Docker 공식 저장소를 추가해야 한다.

```bash
dnf install -y dnf-plugins-core                                                  # config-manager 플러그인
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
cat /etc/yum.repos.d/docker-ce.repo | head -12
dnf repolist | grep -i docker
```

- `dnf-plugins-core` : `config-manager`, `copr`, `download` 등 dnf 확장 플러그인 묶음
- `dnf config-manager --add-repo <URL>` : `.repo` 파일을 내려받아 `/etc/yum.repos.d/` 에 배치 (**add repo**)
	- `--set-enabled <repo>` / `--set-disabled <repo>` : 저장소 활성·비활성 전환
- **Rocky 가 `linux/centos` 경로를 쓰는 이유** : Docker 는 RHEL 파생 배포판용 빌드를 `centos` 디렉터리로 배포하고, `.repo` 안의 `$releasever` 가 Rocky 9 에서 `9` 로 치환되어 `centos/9/aarch64/stable` 을 가리킴 → RHEL 9 호환 바이너리이므로 그대로 사용 가능 (`rhel` 경로는 RHEL 정품 구독용)
- `$basearch` : 아키텍처 치환 변수 — 이 VM 에서는 `aarch64`

**검증**

```bash
grep -E '^\[|baseurl|enabled|gpgkey' /etc/yum.repos.d/docker-ce.repo | head -8
dnf repolist enabled | grep docker-ce-stable
dnf list --available docker-ce | tail -2         # 실제로 패키지가 보이는지
```

```text
[docker-ce-stable]
baseurl=https://download.docker.com/linux/centos/$releasever/$basearch/stable
enabled=1
gpgkey=https://download.docker.com/linux/centos/gpg
# dnf repolist enabled | grep docker-ce-stable
docker-ce-stable        Docker CE Stable - aarch64
# dnf list --available docker-ce | tail -2
docker-ce.aarch64        3:27....-1.el9        docker-ce-stable
```

> 📝 **시험 포인트**: `.repo` 파일의 필수 지시자 `[ID]` `name=` `baseurl=`(또는 `mirrorlist=`) `enabled=` `gpgcheck=` `gpgkey=` 와 위치 `/etc/yum.repos.d/` 는 필기 빈출 → Part 02 와 동일 개념.

### 2-3. 설치와 기동

> **상황**: 엔진·CLI·런타임·빌드/컴포즈 플러그인을 한 번에 설치하고 부팅 시 자동 시작까지 걸어 둔다.

```bash
dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker
systemctl status docker --no-pager | head -8
```

| 패키지 | 내용 |
| --- | --- |
| `docker-ce` | 데몬 `dockerd` 본체 |
| `docker-ce-cli` | `docker` 클라이언트 명령 (단독 설치로 원격 데몬 제어도 가능) |
| `containerd.io` | `containerd` 런타임 + `runc` |
| `docker-buildx-plugin` | `docker buildx` — 확장 빌더(멀티 아키텍처 빌드) |
| `docker-compose-plugin` | `docker compose` — 다중 컨테이너 정의 실행 (v2, 하이픈 없음) |

- `systemctl enable --now` : 부팅 시 자동 시작 등록 + 즉시 기동 → [[07-boot-systemd-log]] 2절
- 데몬 소켓 활성화 : `docker.socket` 유닛이 함께 설치됨 — 소켓에 접근이 오면 데몬을 깨우는 방식

**검증**

```bash
systemctl is-enabled docker; systemctl is-active docker
ss -xl | grep docker.sock                         # 유닉스 소켓 리슨 확인
ps -ef | grep -E 'dockerd|containerd' | grep -v grep | awk '{print $1, $2, $8}'
docker run --rm hello-world | head -4             # 설치 확인용 공식 테스트 이미지
```

```text
# systemctl is-enabled docker; systemctl is-active docker
enabled
active
# ss -xl | grep docker.sock
u_str  LISTEN  0  4096  /run/docker.sock 12345  * 0
# ps -ef | grep -E 'dockerd|containerd'
root 1234 /usr/bin/dockerd
root 1200 /usr/bin/containerd
# docker run --rm hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

- `docker run --rm hello-world` 가 성공하면 **pull → 컨테이너 생성 → 실행 → 종료 → 자동 삭제** 전 경로가 검증된 것

> 📝 **시험 포인트**: `docker.service` 는 systemd 유닛이므로 `systemctl start/enable/status` 대상. 필기에서 "서비스 상시 기동 등록" 은 `enable`, "즉시 시작" 은 `start` — `--now` 는 둘의 결합.

### 2-4. docker version · docker info 읽기

> **상황**: 클라이언트만 살아 있고 데몬이 죽은 상태를 구분할 줄 알아야 한다. `docker info` 는 스토리지 드라이버·cgroup 버전 같은 운영 핵심 값을 한 번에 보여 준다.

```bash
docker version                                   # Client / Server 두 블록
docker version --format '{{.Client.Version}} / {{.Server.Version}}'
docker info                                      # 데몬 전체 상태
docker info --format '{{.ServerVersion}} {{.Driver}} {{.CgroupDriver}} {{.CgroupVersion}} {{.Architecture}}'
docker info | grep -E 'Storage Driver|Cgroup|Architecture|Docker Root Dir|Logging Driver|Live Restore|Containers|Images'
```

- `docker version` : **클라이언트와 서버(데몬) 버전을 각각** 출력 — 데몬 미기동 시 Client 블록만 나오고 `Cannot connect to the Docker daemon` 오류
- `--format '{{...}}'` : Go 템플릿 출력 — 스크립트에서 특정 필드만 추출할 때 사용 → [[../../CONTAINER/docker]]
- `docker info` 주요 항목
	- `Storage Driver: overlay2` : 유니온 파일시스템 드라이버 (레이어 구현체)
	- `Cgroup Driver: systemd` : cgroup 을 systemd 가 관리 (RHEL 계열 권장값)
	- `Cgroup Version: 2` : RHEL 9 기본 통합 계층
	- `Docker Root Dir: /var/lib/docker` : 이미지·컨테이너·볼륨 저장 위치
	- `Logging Driver: json-file` : 로그 저장 방식 (2-6 에서 회전 정책 부여)

**검증**

```bash
docker info --format '{{.ServerVersion}} {{.Driver}} {{.CgroupDriver}} {{.CgroupVersion}} {{.Architecture}}'
systemctl stop docker; docker version 2>&1 | tail -3; systemctl start docker    # 데몬 정지 시 동작 확인
```

```text
# docker info --format ...
27.... overlay2 systemd 2 aarch64
# (데몬 정지 상태)
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

> 📝 **시험 포인트**: `docker version` 이 Client/Server 두 블록인 이유 = **CLI 와 데몬이 분리된 구조**. "docker 명령은 되는데 컨테이너 목록이 안 나온다" → 데몬 상태 확인이 정답 흐름.

### 2-5. dev1 을 docker 그룹에 — 그리고 그 위험

> **상황**: 개발자 `dev1`(Part 03) 이 매번 `sudo docker` 를 치지 않도록 권한을 위임한다. 다만 이 권한이 무엇을 의미하는지 반드시 못 박아 둔다.

```bash
ls -l /var/run/docker.sock                # 소켓 소유자·그룹·권한
getent group docker                        # 설치 시 생성된 docker 그룹
usermod -aG docker dev1                    # 보조 그룹 추가
id dev1
```

- `usermod -aG <그룹> <사용자>` : 보조 그룹 **추가** (**a**ppend + **G**roups) — `-a` 없이 `-G` 만 쓰면 **기존 보조 그룹이 모두 교체됨**(기출 함정) → [[03-user-group-permission]]
- 그룹 변경은 **새 로그인 세션부터** 반영 → 기존 세션에서는 `newgrp docker` 로 임시 적용

⚠️ **`docker` 그룹 = 사실상 root 권한**

- `docker.sock` 은 `root:docker 660` → 이 그룹 구성원은 데몬 API 를 전부 호출 가능
- 데몬은 **root 로 동작**하므로, 호스트 루트 디렉터리를 컨테이너에 마운트(`-v /:/host`)하거나 `--privileged` 컨테이너를 만들면 호스트 파일 전체를 읽고 쓸 수 있음
- 즉 **`sudo` 권한을 주지 않은 사용자에게 `docker` 그룹을 주는 것은 우회 경로를 여는 것** → 감사 대상. 최소 권한이 필요하면 rootless Docker 또는 Podman rootless 검토
- 컨테이너 안에 `docker.sock` 을 마운트하는 구성(`-v /var/run/docker.sock:/var/run/docker.sock`)도 같은 이유로 침해 시 **컨테이너 탈출 경로** → [[../../CONTAINER/docker]] `docker inspect` 절

**검증**

```bash
id -nG dev1 | tr ' ' '\n' | grep -x docker      # 그룹 반영 확인
su - dev1 -c 'id -nG'                            # 새 로그인 세션의 그룹 목록
su - dev1 -c 'docker ps'                         # sudo 없이 데몬 접근 성공 여부
su - dev1 -c 'docker info --format "{{.ServerVersion}}"'
```

```text
# id -nG dev1 | tr ' ' '\n' | grep -x docker
docker
# su - dev1 -c 'id -nG'
devteam docker
# su - dev1 -c 'docker ps'
CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES
# (오류 없이 헤더만 나오면 성공. 아래는 그룹 미반영 시 메시지)
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock
```

> 📝 **시험 포인트**: `usermod -aG` 의 `-a` 누락 = 보조 그룹 초기화. 보안 문항으로는 "특정 그룹 소속만으로 root 권한에 준하는 접근이 가능한 사례" 로 `docker` 그룹이 언급됨.

### 2-6. /etc/docker/daemon.json — 로그 회전·저장 경로·무중단 재시작

> **상황**: 기본 설정은 컨테이너 로그가 무한히 커진다. Part 07 에서 logrotate 로 시스템 로그를 다뤘듯, 컨테이너 로그도 상한을 건다.

```bash
mkdir -p /etc/docker
cat > /etc/docker/daemon.json <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "data-root": "/var/lib/docker",
  "live-restore": true
}
EOF
python3 -m json.tool /etc/docker/daemon.json      # JSON 문법 검사 (오타 시 데몬 기동 실패)
systemctl restart docker
```

| 키 | 의미 |
| --- | --- |
| `log-driver` | 로그 저장 방식 — `json-file`(기본), `journald`, `local`, `syslog`, `none` |
| `log-opts.max-size` | 로그 파일 1개 최대 크기 (초과 시 회전) |
| `log-opts.max-file` | 보관할 회전 파일 개수 (총 상한 = max-size × max-file) |
| `data-root` | 이미지·컨테이너·볼륨 저장 루트 (기본 `/var/lib/docker`) |
| `live-restore` | **데몬 재시작 중에도 실행 중 컨테이너 유지** — shim 이 컨테이너를 붙들고 있어 가능 |

⚠️ `data-root` 를 다른 경로로 바꾸려면 **데몬 정지 → 기존 데이터 이동 → 재시작** 순서가 필요. 값만 바꾸고 재시작하면 기존 이미지·볼륨이 안 보임. 이 실습에서는 기본값 그대로 명시만 함

**검증**

```bash
systemctl is-active docker
docker info | grep -E 'Logging Driver|Live Restore|Docker Root Dir'
docker info --format '{{.LoggingDriver}} {{.LiveRestoreEnabled}} {{.DockerRootDir}}'
journalctl -u docker -n 5 --no-pager             # 재시작 로그(설정 오류 시 여기 표시)
```

```text
# docker info | grep -E 'Logging Driver|Live Restore|Docker Root Dir'
 Logging Driver: json-file
 Docker Root Dir: /var/lib/docker
 Live Restore Enabled: true
# docker info --format '{{.LoggingDriver}} {{.LiveRestoreEnabled}} {{.DockerRootDir}}'
json-file true /var/lib/docker
```

- `log-opts` 는 **새로 만드는 컨테이너부터** 적용 — 기존 컨테이너는 재생성해야 반영
- 개별 컨테이너에서 덮어쓰려면 `docker run --log-driver=... --log-opt max-size=...`

> 📝 **시험 포인트**: 설정 파일 경로 `/etc/docker/daemon.json` 과 "JSON 문법 오류 시 데몬이 기동하지 않는다" 는 실무형 서술. 로그 폭주 대응 = `max-size`/`max-file`.

### 2-7. /var/lib/docker 구조

> **상황**: 디스크가 찼을 때 무엇이 용량을 먹는지 알아야 한다. 저장 루트의 하위 구조를 확인한다.

```bash
ls -l /var/lib/docker/
du -sh /var/lib/docker/* 2>/dev/null | sort -h
df -h /var/lib/docker
```

| 디렉터리 | 내용 |
| --- | --- |
| `overlay2/` | **이미지 레이어 + 컨테이너 쓰기 레이어**의 실제 파일 (보통 최대 용량) |
| `image/overlay2/` | 이미지 메타데이터·레이어 DB |
| `containers/<ID>/` | 컨테이너별 설정(`config.v2.json`)과 **로그 파일** `<ID>-json.log` |
| `volumes/<이름>/_data` | 이름 있는 볼륨의 실제 데이터 |
| `network/` | 네트워크 상태 DB |
| `buildkit/` | 빌드 캐시 |
| `tmp/`, `runtimes/`, `plugins/` | 임시·런타임·플러그인 |

- `du -sh` : 디렉터리별 합계 크기 (**s**ummarize + **h**uman-readable), `sort -h` : 사람이 읽는 크기 단위로 정렬
- ⚠️ `/var/lib/docker` 하위 파일을 직접 `rm` 하지 말 것 — 메타데이터 DB 와 불일치가 생겨 데몬이 깨짐. 정리는 반드시 `docker system prune`/`volume rm` 사용(9절)

**검증**

```bash
du -sh /var/lib/docker | tail -1
ls /var/lib/docker/containers | wc -l           # 컨테이너 개수와 일치해야 함
docker ps -aq | wc -l
```

```text
# du -sh /var/lib/docker/* | sort -h
0	/var/lib/docker/plugins
4.0K	/var/lib/docker/network
...
120M	/var/lib/docker/overlay2
# ls /var/lib/docker/containers | wc -l
1
# docker ps -aq | wc -l
1
```

> 📝 **시험 포인트**: 컨테이너 로그의 실제 파일 경로 `/var/lib/docker/containers/<ID>/<ID>-json.log` 는 "컨테이너를 지우면 로그도 사라진다" 는 사고 대응 논점과 함께 출제될 수 있음.

### 2-8. firewalld · iptables 와 Docker

> **상황**: Part 10 에서 firewalld 를 세밀하게 잡아 두었다. Docker 는 자기 규칙을 직접 넣으므로 둘의 관계를 확인해 둔다.

```bash
firewall-cmd --get-active-zones                     # docker 존이 생겼는지
firewall-cmd --info-zone=docker 2>/dev/null
iptables -t nat -L DOCKER -n                        # docker 가 만든 nat 체인
iptables -L DOCKER-USER -n                          # 관리자가 규칙을 넣는 자리
ip -br a show docker0                                # 기본 브리지 주소
sysctl net.ipv4.ip_forward                          # docker 가 1 로 켬
```

- Docker 는 기동 시 **`docker0` 브리지 생성 + IP 포워딩 활성 + iptables `DOCKER`/`DOCKER-USER`/`DOCKER-ISOLATION` 체인 삽입**
- firewalld 가 동작 중이면 Docker 가 `docker` 존을 만들고 `docker0` 를 그 존에 넣음 → 존 정책이 컨테이너 트래픽에 영향
- **핵심 함정** : `-p 0.0.0.0:포트` 로 게시한 컨테이너 포트는 nat `DOCKER` 체인의 DNAT 규칙이 **firewalld 의 존 규칙보다 먼저 처리**되는 경로가 있어, `firewall-cmd` 로 막았다고 생각해도 외부에서 접근될 수 있음
	- 대응 1 — 이번 `lab-redis` 처럼 **`-p 127.0.0.1:6379:6379` 루프백에만 바인딩** (가장 확실)
	- 대응 2 — `DOCKER-USER` 체인에 관리자 규칙 삽입 (Docker 가 지우지 않는 유일한 체인)
- Part 10 에서 정의한 firewalld 커스텀 서비스 `lab-redis`(6379/tcp) 는 **나중에 외부 노출로 전환할 때**를 위한 준비물 — 루프백 바인딩 상태에서는 개방 없이도 동작

```bash
# 참고 — 컨테이너 트래픽을 관리자 규칙으로 통제하는 자리 (지금은 실행 불필요)
# iptables -I DOCKER-USER -s 192.168.64.0/24 -j ACCEPT
# iptables -I DOCKER-USER -j DROP
```

**검증**

```bash
ip -br a show docker0
sysctl -n net.ipv4.ip_forward
iptables -t nat -S DOCKER | head -3
firewall-cmd --get-active-zones | head -6
```

```text
# ip -br a show docker0
docker0   DOWN   172.17.0.1/16
# sysctl -n net.ipv4.ip_forward
1
# iptables -t nat -S DOCKER
-N DOCKER
-A DOCKER -i docker0 -j RETURN
# firewall-cmd --get-active-zones
docker
  interfaces: docker0
internal
  sources: 192.168.64.0/24
public
  interfaces: enp0s1
```

- 실행 중 컨테이너가 없으면 `docker0` 상태는 `DOWN` 이 정상 (컨테이너 연결 시 UP)

> 📝 **시험 포인트**: `iptables` 체인 이름과 테이블 구분은 Part 10 범위. 여기서는 "포트 게시(`-p`)는 nat 테이블 DNAT 로 구현된다" 와 "DOCKER-USER 가 관리자 규칙 자리" 두 가지만 기억.

---

## 3. 이미지 다루기

### 3-1. 검색과 내려받기 — search · pull

> **상황**: 캐시로 쓸 Redis 와 패키지 실습용 Ubuntu 이미지를 확보한다. 태그를 명시해 버전을 고정하는 습관을 들인다.

```bash
docker search redis --limit 5                     # Docker Hub 검색
docker search redis --limit 5 --filter is-official=true
docker pull redis:7-alpine                        # 캐시용 (alpine 기반 = 경량)
docker pull ubuntu:24.04                          # apt/dpkg 실습용
docker pull httpd:2.4-alpine                      # 7절 Dockerfile 베이스
```

- `docker search <키워드>` : 레지스트리(Docker Hub) 이미지 검색 — **로컬 이미지 검색이 아님**
	- `--limit N` : 결과 개수 제한 (기본 25, 최대 100)
	- `--filter is-official=true` : 공식 이미지만 (**filter**), `--filter stars=100` : 별점 하한
	- `--format` : 출력 형식 지정
- `docker pull <이미지>[:<태그>]` : 레지스트리에서 로컬로 **내려받기**. 태그 생략 시 `:latest`
	- `-a` : 저장소의 모든 태그 (**a**ll-tags) ⚠️ 용량 폭증
	- `-q` : 진행 출력 억제 (**q**uiet)
	- `--platform linux/amd64` : 아키텍처 지정 (3-5 참고)
- 이미지 이름 구조 : `[레지스트리/][네임스페이스/]이름[:태그|@다이제스트]`
	- `redis:7-alpine` = `docker.io/library/redis:7-alpine` 의 축약 — `library` 는 공식 이미지 네임스페이스

**검증**

```bash
docker images
docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.ID}}\t{{.Size}}'
docker image inspect redis:7-alpine --format '{{.Architecture}} {{.Os}}'
```

```text
# docker images
REPOSITORY    TAG          IMAGE ID       CREATED        SIZE
redis         7-alpine     ...            ... ago        41.2MB
ubuntu        24.04        ...            ... ago        97.1MB
httpd         2.4-alpine   ...            ... ago        ...MB
hello-world   latest       ...            ... ago        ...kB
# docker image inspect redis:7-alpine --format '{{.Architecture}} {{.Os}}'
arm64 linux
```

- `arm64 linux` 확인 = **호스트 아키텍처(aarch64)에 맞는 이미지**가 자동 선택됨 (멀티 아키텍처 매니페스트 덕분)

> 📝 **시험 포인트**: `docker pull` = 레지스트리 → 로컬(**다운로드**), `docker push` = 로컬 → 레지스트리(업로드). "pull 이 업로드" 선지는 오답(필기 R01 #90 ③).

### 3-2. 목록·상세·이력 — images · image inspect · history

> **상황**: 받아 둔 이미지가 어떤 레이어로 이루어졌고 기본 실행 명령이 무엇인지 확인한다. 이미지의 `CMD` 를 알아야 4절에서 덮어쓸 수 있다.

```bash
docker images                                    # = docker image ls
docker image ls -a --digests                     # 중간 이미지 포함 + 다이제스트 표시
docker images -q                                 # ID 만 (스크립트용)
docker images --filter 'dangling=true'           # 태그 없는(<none>) 이미지
docker image inspect redis:7-alpine | head -30
docker image inspect redis:7-alpine --format '{{.Config.Cmd}} / {{.Config.Entrypoint}} / {{.Config.ExposedPorts}}'
docker image inspect redis:7-alpine --format '{{range .RootFS.Layers}}{{println .}}{{end}}' | wc -l
docker history redis:7-alpine
docker history --no-trunc --format 'table {{.Size}}\t{{.CreatedBy}}' redis:7-alpine | head -5
```

- `docker image ls` : `docker images` 와 동일 (관리 명령 체계)
	- `-a` : 중간 레이어 이미지 포함 (**a**ll)
	- `--digests` : `sha256:...` 콘텐츠 다이제스트 열 추가
	- `--filter dangling=true` : 어떤 태그도 가리키지 않는 이미지 (빌드 후 남은 찌꺼기)
	- `--no-trunc` : ID·명령을 자르지 않고 전체 출력 (**no truncate**)
- `docker history <이미지>` : **레이어별 생성 명령과 크기** — Dockerfile 의 각 지시자가 한 레이어
- `docker image inspect` : 이미지 메타데이터 전체 (JSON). `Config.Cmd`, `Config.Entrypoint`, `Config.Env`, `RootFS.Layers`

**검증**

```bash
docker image inspect redis:7-alpine --format '{{.Config.Entrypoint}} {{.Config.Cmd}}'
docker history redis:7-alpine | wc -l
docker images --format '{{.Repository}}:{{.Tag}}' | sort
```

```text
# docker image inspect redis:7-alpine --format '{{.Config.Entrypoint}} {{.Config.Cmd}}'
[docker-entrypoint.sh] [redis-server]
# docker history redis:7-alpine
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
...            ... ago       CMD ["redis-server"]                            0B        buildkit.dockerfile.v0
<missing>      ... ago       ENTRYPOINT ["docker-entrypoint.sh"]             0B        buildkit.dockerfile.v0
<missing>      ... ago       COPY docker-entrypoint.sh /usr/local/bin/ ...   ...       buildkit.dockerfile.v0
...
```

- `ENTRYPOINT ["docker-entrypoint.sh"]` + `CMD ["redis-server"]` 구조 → 4-2 에서 `redis-server --appendonly yes …` 로 **CMD 만 덮어씀**
- `<missing>` : 로컬에 중간 이미지 ID 가 없는 레이어(원격 빌드분) — 오류가 아님

> 📝 **시험 포인트**: `docker images` 는 **로컬 이미지 목록**, `docker ps` 는 **컨테이너 목록**. 둘을 바꿔 놓은 선지가 필기 R01 #90, R03 #86, R04 #88 에 반복 등장.

### 3-3. 태그·삭제·정리 — tag · rmi · prune

> **상황**: 같은 이미지에 사내 표기용 이름을 하나 더 붙이고, 필요 없어진 `hello-world` 를 정리한다.

```bash
docker tag redis:7-alpine lab/redis:cache          # 같은 이미지에 별칭 추가
docker images | grep -E 'redis'
docker image inspect lab/redis:cache --format '{{.Id}}'
docker image inspect redis:7-alpine --format '{{.Id}}'   # 동일해야 함

docker rmi lab/redis:cache                         # 태그만 제거 (이미지 본체는 유지)
docker rmi hello-world                             # 참조하는 태그가 하나뿐이면 실제 삭제
```

- `docker tag <원본> <새이름[:태그]>` : **참조(이름)만 추가** — 디스크 복사 아님. IMAGE ID 동일
- `docker rmi <이미지>` : 이미지(태그) 제거 (**r**e**m**ove **i**mage)
	- 태그가 여러 개면 해당 **태그만 제거**되고 `Untagged:` 만 출력 / 마지막 태그면 `Deleted: sha256:...` 로 레이어까지 제거
	- `-f` : 컨테이너가 참조 중이어도 강제 (**f**orce) ⚠️ 실행 중 컨테이너가 있으면 기본 거부
	- ⚠️ **`docker rmi` 는 컨테이너를 지우는 명령이 아님** — 컨테이너 삭제는 `docker rm`
- `docker image prune` : 태그 없는(dangling) 이미지 일괄 삭제
	- `-a` : **사용 중이 아닌 모든 이미지** 삭제 (**a**ll) ⚠️ 범위 넓음
	- `-f` : 확인 프롬프트 생략 (**f**orce)
	- `--filter 'until=24h'` : 생성 24시간 이전 것만

⚠️ `docker image prune -a` 는 컨테이너가 쓰지 않는 이미지를 전부 지움 → 재다운로드 비용 발생. 대상 확인 후 실행

```bash
docker images --filter 'dangling=true' -q          # 먼저 대상 확인
docker image prune -f                              # dangling 만 정리 (안전)
```

**검증**

```bash
docker images | grep -c 'hello-world' || echo "hello-world 삭제됨"
docker images --format '{{.Repository}}:{{.Tag}} {{.ID}}' | sort
docker system df                                    # 이미지가 차지한 총량
```

```text
# docker rmi lab/redis:cache
Untagged: lab/redis:cache
# docker rmi hello-world
Untagged: hello-world:latest
Deleted: sha256:...
# docker images | grep -c 'hello-world'
0
hello-world 삭제됨
```

> 📝 **시험 포인트**: "`docker rmi web` 은 실행 중인 컨테이너 web 을 삭제한다" 는 **틀린 설명**(필기 R04 #88 ④). `rmi` = 이미지, `rm` = 컨테이너.

### 3-4. 반출·반입 — save/load vs export/import

> **상황**: 인터넷이 없는 망분리 서버로 Redis 이미지를 옮겨야 한다고 가정하고, 이미지를 파일로 만들었다가 되살린다.

```bash
cd /root
docker save -o redis-7-alpine.tar redis:7-alpine     # 이미지 → tar
ls -lh redis-7-alpine.tar
tar -tf redis-7-alpine.tar | head -5                  # 내부 구조(레이어 + manifest)

docker rmi redis:7-alpine                             # 로컬에서 지운 뒤
docker images | grep redis || echo "로컬에 redis 없음"
docker load -i redis-7-alpine.tar                     # tar → 이미지 복원
docker images | grep redis
```

- `docker save -o <파일> <이미지>...` : 이미지를 **레이어·태그·메타데이터 포함** tar 로 저장 (**o**utput)
	- `-o` 대신 리다이렉션도 가능 : `docker save redis:7-alpine | gzip > redis.tar.gz`
- `docker load -i <파일>` : `save` 산출물 복원 (**i**nput) — **태그·이력이 그대로 살아남**
- `docker export <컨테이너>` / `docker import <파일> <이미지명>` : **컨테이너 파일시스템 스냅샷**을 단일 레이어 tar 로 반출/반입

| 구분 | `save` / `load` | `export` / `import` |
| --- | --- | --- |
| 대상 | **이미지** | **컨테이너**(실행 중이어도 가능) |
| 레이어 | 전부 보존 (다층) | **단일 레이어로 평탄화** |
| 태그·이력 | 보존 (`history` 유지) | 소실 (`import` 시 새 이름 부여) |
| 메타데이터 | `CMD`·`ENTRYPOINT`·`ENV` 보존 | **소실** → `import --change 'CMD …'` 로 재지정 |
| 볼륨 데이터 | 포함 안 됨 | 포함 안 됨 (마운트 지점은 빈 디렉터리) |
| 용도 | 이미지 배포·백업 | 컨테이너 상태 평탄화, 이미지 슬림화 |

```bash
# export/import 비교 실습 (임시 컨테이너로)
docker run -d --name tmp-exp redis:7-alpine
docker export -o tmp-exp.tar tmp-exp
docker import tmp-exp.tar lab/from-export:1.0
docker image inspect lab/from-export:1.0 --format 'Cmd={{.Config.Cmd}} Entry={{.Config.Entrypoint}}'
docker history lab/from-export:1.0                    # 레이어 1개
docker rm -f tmp-exp; docker rmi lab/from-export:1.0; rm -f tmp-exp.tar
```

**검증**

```bash
ls -lh /root/redis-7-alpine.tar
docker images --format '{{.Repository}}:{{.Tag}}' | grep redis
docker image inspect redis:7-alpine --format '{{.Config.Entrypoint}}'   # load 후에도 보존
```

```text
# ls -lh /root/redis-7-alpine.tar
-rw------- 1 root root 42M ... redis-7-alpine.tar
# docker load -i redis-7-alpine.tar
Loaded image: redis:7-alpine
# docker image inspect redis:7-alpine --format '{{.Config.Entrypoint}}'
[docker-entrypoint.sh]
# docker image inspect lab/from-export:1.0 --format 'Cmd={{.Config.Cmd}} Entry={{.Config.Entrypoint}}'
Cmd=[] Entry=[]            ← export/import 는 실행 메타데이터 소실
```

> 📝 **시험 포인트**: `save`↔`load` 는 **이미지** 쌍, `export`↔`import` 는 **컨테이너** 쌍. 짝을 섞은 선지(`save`↔`import`)가 오답으로 나옴. 백업 관점은 [[12-backup-recovery-review]] 와 연계.

### 3-5. 레이어 공유·다이제스트·멀티 아키텍처

> **상황**: 같은 베이스(alpine)를 쓰는 이미지를 하나 더 받아 레이어가 재사용되는지 확인한다. 태그와 다이제스트의 차이도 정리한다.

```bash
docker images --format '{{.Repository}}:{{.Tag}} {{.Size}}'
docker pull alpine:3.20                                  # redis:7-alpine 의 베이스와 같은 계열
docker pull redis:7.2-alpine                             # 같은 계열 다른 태그 → 공유 레이어 관찰
docker image inspect redis:7-alpine redis:7.2-alpine --format '{{index .RootFS.Layers 0}}'
docker system df -v | head -20                           # 실제 점유량(SHARED SIZE)
```

- pull 로그의 `Already exists` 줄 = **이미 로컬에 있는 레이어를 재사용** → 두 이미지의 `SIZE` 합보다 실제 디스크 사용량이 작음
- `docker system df -v` 의 `SHARED SIZE` 열이 공유분

**태그 vs 다이제스트**

| 구분 | 태그 (`redis:7-alpine`) | 다이제스트 (`redis@sha256:...`) |
| --- | --- | --- |
| 성질 | **가변** — 같은 태그가 새 이미지로 갱신될 수 있음 | **불변** — 콘텐츠 해시 |
| 재현성 | 낮음 (`latest` 최악) | 높음 — 운영 배포 권장 |
| 확인 | `docker images` | `docker images --digests` |

```bash
docker images --digests | grep redis
# 다이제스트로 고정 pull (참고)
# docker pull redis@sha256:<64자리해시>
```

**멀티 아키텍처**

```bash
docker image inspect redis:7-alpine --format '{{.Architecture}}/{{.Os}} {{.Variant}}'
docker buildx version                                     # buildx 플러그인 존재 확인
# ※ 참고 — 다른 아키텍처 이미지 강제 지정 (실행하려면 QEMU 사용자 모드 에뮬레이션 필요)
# docker pull --platform linux/amd64 redis:7-alpine
# docker run --rm --platform linux/amd64 redis:7-alpine redis-server --version
```

- 하나의 태그가 아키텍처별 이미지를 묶는 **매니페스트 리스트**를 가리킴 → 호스트에 맞는 것이 자동 선택
- `--platform linux/amd64` 를 aarch64 호스트에서 **실행**하려면 `qemu-user-static` + binfmt 등록이 필요 → 이 실습 환경 미구성이므로 `※ 미실행`. 지정만 하고 실행하면 `exec format error`

**검증**

```bash
docker pull redis:7.2-alpine 2>&1 | grep -c 'Already exists'    # 재사용된 레이어 수
docker system df
docker rmi alpine:3.20 redis:7.2-alpine                          # 비교 실습 정리
```

```text
# docker pull redis:7.2-alpine
7-alpine: Pulling from library/redis
Already exists          ← 베이스 레이어 재사용
Already exists
...
# docker system df
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          4         1         ...MB     ...MB (..%)
Containers      1         1         ...kB     0B (0%)
Local Volumes   1         1         ...MB     0B (0%)
Build Cache     0         0         0B        0B
```

> 📝 **시험 포인트**: `docker images` 출력에서 **IMAGE ID 가 같으면 태그만 다른 동일 이미지** — 용량을 두 번 더하는 선지는 오답(필기 R08 #84). `latest` 는 "최신" 을 보장하지 않는 **단순 기본 태그명**.

---

## 4. 컨테이너 운영 — lab-redis

### 4-1. docker run 옵션 총람

> **상황**: `docker run` 은 시험·실무 모두에서 가장 많이 나오는 명령이다. 기동 전에 옵션 표를 한 번에 정리한다.

`docker run` = **`docker create`(컨테이너 생성) + `docker start`(실행)** 를 합친 명령. 이미지가 없으면 `pull` 까지 자동 수행

| 옵션 | 의미 (원어) |
| --- | --- |
| `-d` | 백그라운드 실행 (**d**etach) — 컨테이너 ID 만 출력하고 프롬프트 복귀 |
| `-i` | 표준입력 유지 (**i**nteractive) |
| `-t` | 의사 터미널 할당 (**t**ty) — `-it` 로 묶어 대화형 셸에 사용 |
| `--rm` | 종료 시 컨테이너 **자동 삭제** — 일회성 작업용 |
| `--name <이름>` | 컨테이너 이름 지정 (미지정 시 무작위 이름) |
| `-p <호스트>:<컨테이너>` | 포트 **게시**(publish) — `-p 127.0.0.1:6379:6379` 처럼 바인딩 주소 지정 가능 |
| `-P` | `EXPOSE` 된 모든 포트를 호스트 임의 포트에 자동 게시 (**P**ublish all) |
| `-v <이름\|경로>:<컨테이너경로>[:opt]` | 볼륨/바인드 마운트 (**v**olume). 옵션 `ro`, `rw`, `z`, `Z` |
| `--mount type=...` | 명시적 마운트 문법 (5-3 비교표) |
| `-e KEY=VALUE` | 환경변수 주입 (**e**nvironment) |
| `--env-file <파일>` | 환경변수를 파일에서 읽음 — **비밀값을 명령행에 노출하지 않는 방법** |
| `--network <이름>` | 연결할 네트워크 (`bridge` 기본, `host`, `none`, 사용자 정의) |
| `--hostname <이름>` | 컨테이너 내부 호스트명 (UTS 네임스페이스) |
| `-w <경로>` | 작업 디렉터리 (**w**orking dir) |
| `-u <UID[:GID]>` | 실행 사용자 (**u**ser) |
| `--restart <정책>` | 재시작 정책 — `no`(기본) / `on-failure[:N]` / `always` / `unless-stopped` |
| `--cpus <n>` | CPU 코어 환산 상한 (예 `0.5` = 코어 절반) |
| `--memory <크기>` | 메모리 상한 (`256m`, `1g`) — `--memory-swap` 과 짝 |
| `--privileged` | ⚠️ 거의 모든 커널 권한 부여 — **컨테이너 격리 사실상 해제** |
| `--cap-add` / `--cap-drop` | 리눅스 capability 개별 추가·제거 (권장: `--cap-drop=ALL` 후 필요분만 추가) |
| `--read-only` | 컨테이너 루트 파일시스템을 읽기 전용으로 |
| `--health-cmd` | 헬스체크 명령 (`--health-interval`, `--health-retries` 와 함께) |
| `--log-driver` / `--log-opt` | 컨테이너 단위 로그 드라이버·옵션 |
| `--entrypoint` | 이미지의 `ENTRYPOINT` 덮어쓰기 |
| `--label` | 메타데이터 라벨 부여 (`--filter label=` 로 검색) |

- `--restart` 정책 차이
	- `always` : 데몬 재시작 시에도 **항상** 기동 (사용자가 `docker stop` 한 것도 데몬 재시작 후 다시 뜸)
	- `unless-stopped` : `always` 와 같으나 **사용자가 명시적으로 stop 한 것은 다시 띄우지 않음** → 운영 서비스 권장값

**검증**

```bash
docker run --help | grep -cE '^\s+-'          # 지원 옵션 개수(참고)
docker create --name tmp-create redis:7-alpine && docker ps -a --filter name=tmp-create
docker start tmp-create && docker ps --filter name=tmp-create --format '{{.Names}} {{.Status}}'
docker rm -f tmp-create
```

```text
# docker ps -a --filter name=tmp-create
CONTAINER ID   IMAGE            COMMAND                  CREATED   STATUS    PORTS   NAMES
...            redis:7-alpine   "docker-entrypoint.s…"   ...       Created           tmp-create
# docker start tmp-create
tmp-create
# docker ps --filter name=tmp-create --format '{{.Names}} {{.Status}}'
tmp-create Up 2 seconds
```

- `Created` 상태(생성만 하고 미실행)를 직접 만들어 봄으로써 `run = create + start` 가 확인됨

> 📝 **시험 포인트**: `-p 8080:80` 은 **호스트 8080 → 컨테이너 80**(앞이 호스트). 순서를 뒤집은 선지가 필기 R04 #87·R06 #87·R08 #83 에 반복. `-d` 는 백그라운드이지 "삭제" 가 아님(R03 #86 ②).

### 4-2. lab-redis 기동

> **상황**: 인트라넷 웹이 쓸 캐시를 올린다. 인증 없는 Redis 는 인터넷에 노출되면 즉시 침해되는 대표 사례이므로, **루프백 바인딩 + 비밀번호 + 영속화** 세 가지를 처음부터 건다.

⚠️ **인증 없는 Redis 외부 노출은 즉시 침해로 이어짐**

- Redis 는 기본 인증이 없고 `CONFIG SET dir` 등으로 임의 파일 쓰기가 가능 → 공개 노출 시 SSH 키 삽입·크립토 마이너 설치 사례 다수
- 그래서 이 실습은 ① `-p 127.0.0.1:6379:6379` **루프백에만 게시**(같은 서버의 httpd 만 접속) ② `--requirepass` 로 인증 ③ `--appendonly yes` 로 영속화
- 웹과 Redis 가 같은 호스트이므로 외부 게시 자체가 불필요 — **필요 없는 노출은 만들지 않는다**가 원칙

```bash
docker volume create redis-data                       # 데이터용 이름 있는 볼륨 선생성
docker run -d \
  --name lab-redis \
  --restart unless-stopped \
  -p 127.0.0.1:6379:6379 \
  -v redis-data:/data \
  redis:7-alpine \
  redis-server --appendonly yes --requirepass '<비밀번호>'
```

- 이미지 뒤의 `redis-server --appendonly yes --requirepass …` : 이미지의 `CMD`(`redis-server`)를 **덮어쓴 실행 인자** — `ENTRYPOINT`(`docker-entrypoint.sh`)는 그대로 유지
- `--appendonly yes` : AOF(Append Only File) 영속화 활성 → `/data/appendonlydir/` 에 기록
- `--requirepass <비밀번호>` : 접속 시 `AUTH` 요구
- `<비밀번호>` 는 자리표시자 — 실제로는 `openssl rand -base64 24` 등으로 생성한 값 사용

⚠️ **명령행에 비밀번호를 넣으면 호스트 `ps` 와 `docker inspect` 에 그대로 노출됨**(4-11 에서 직접 확인). 운영에서는 아래처럼 파일로 주입

```bash
# 권장 방식 (참고) — 파일 권한 600 으로 두고 환경변수로 주입
# printf 'REDIS_PASSWORD=<비밀번호>\n' > /root/redis.env && chmod 600 /root/redis.env
# docker run -d --name lab-redis --env-file /root/redis.env ... redis:7-alpine \
#   sh -c 'exec redis-server --appendonly yes --requirepass "$REDIS_PASSWORD"'
```

**검증**

```bash
docker ps --filter name=lab-redis --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
docker inspect lab-redis --format '{{.State.Status}} {{.HostConfig.RestartPolicy.Name}}'
ss -tlnp | grep 6379
docker port lab-redis
```

```text
# docker ps --filter name=lab-redis --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
NAMES       STATUS         PORTS
lab-redis   Up 8 seconds   127.0.0.1:6379->6379/tcp
# docker inspect lab-redis --format '{{.State.Status}} {{.HostConfig.RestartPolicy.Name}}'
running unless-stopped
# ss -tlnp | grep 6379
LISTEN 0 4096 127.0.0.1:6379 0.0.0.0:*  users:(("docker-proxy",pid=...,fd=4))
# docker port lab-redis
6379/tcp -> 127.0.0.1:6379
```

- `PORTS` 가 `127.0.0.1:6379->6379/tcp` (앞에 `0.0.0.0` 이 **아님**) → 외부 대역에서 접근 불가
- 리스닝 프로세스가 `docker-proxy` : 호스트 포트를 받아 컨테이너로 넘겨 주는 Docker 의 사용자 공간 프록시

> 📝 **시험 포인트**: `docker ps` 의 `PORTS` 열에 `0.0.0.0:5432->5432/tcp` 면 **전 대역 공개**, 주소가 `127.0.0.1` 이면 루프백 한정. 침해 사고 문항에서 노출 경로 판정 근거로 쓰임 → [[../../CONTAINER/docker]].

### 4-3. 상태 조회 — ps · filter · format

> **상황**: 컨테이너가 몇 개 있고 무엇이 죽었는지 한눈에 봐야 한다. `docker ps` 의 조회 옵션을 전부 익힌다.

```bash
docker ps                                              # 실행 중만
docker ps -a                                           # 정지·생성 상태 포함 전체
docker ps -q                                           # ID 만 (스크립트용)
docker ps -aq                                          # 전체 ID
docker ps -l                                           # 가장 최근 생성 1개 (latest)
docker ps -n 3                                         # 최근 3개
docker ps -s                                           # 쓰기 레이어 크기 표시 (size)
docker ps --filter status=running
docker ps --filter status=exited --filter name=lab
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
docker ps -a --format '{{.Names}}\t{{.Image}}\t{{.Status}}'
```

- `--filter <key>=<value>` : `status`(created/running/paused/restarting/exited/dead), `name`, `id`, `label`, `ancestor=<이미지>`, `volume`, `network`, `before`/`since`
- `--format` : Go 템플릿. `table` 접두어를 붙이면 헤더 포함 표 형식
	- 사용 가능 필드 : `.ID` `.Names` `.Image` `.Command` `.CreatedAt` `.RunningFor` `.Status` `.Ports` `.Size` `.Labels` `.Mounts` `.Networks`
- `docker ps` 열 의미
	- `CONTAINER ID` : 축약 ID(12자) — 전체는 64자
	- `COMMAND` : 컨테이너가 실행 중인 명령 (`ENTRYPOINT` + `CMD`)
	- `STATUS` : `Up <시간>` / `Exited (<코드>) <시간> ago` / `Up ... (healthy|unhealthy)`
	- `PORTS` : `<호스트바인딩>-><컨테이너포트>/<프로토콜>`

**검증**

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
docker ps -aq | wc -l
docker ps --filter status=running -q | wc -l
docker ps -a --filter 'ancestor=redis:7-alpine' --format '{{.Names}}'
```

```text
# docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
NAMES       STATUS          PORTS
lab-redis   Up 3 minutes    127.0.0.1:6379->6379/tcp
# docker ps -aq | wc -l
1
# docker ps -a --filter 'ancestor=redis:7-alpine' --format '{{.Names}}'
lab-redis
```

> 📝 **시험 포인트**: `docker ps` = 실행 중, `docker ps -a` = 정지 포함 전체. `--all` 없이 정지 컨테이너가 보인다는 선지는 오답. `STATUS` 의 `Up 2 hours` 는 "2시간 전에 중지" 가 아니라 **2시간째 실행 중**(필기 R08 #83 ③ 오답).

### 4-4. 내부 접속 검증 — exec · redis-cli

> **상황**: 포트가 열렸다고 서비스가 정상인 것은 아니다. 컨테이너 안의 `redis-cli` 로 실제 응답을 받아 본다.

```bash
docker exec -it lab-redis redis-cli -a '<비밀번호>' ping
docker exec -it lab-redis redis-cli -a '<비밀번호>' info server | head -6
docker exec lab-redis redis-cli -a '<비밀번호>' config get appendonly
docker exec -it lab-redis sh                       # 컨테이너 셸 진입 (alpine 은 bash 없음 → sh)
```

- `docker exec [옵션] <컨테이너> <명령>` : **실행 중** 컨테이너 안에서 새 프로세스 실행
	- `-i` : 표준입력 유지, `-t` : TTY 할당 → 대화형은 `-it`
	- `-e KEY=VAL` : 이 명령에만 환경변수 주입 (컨테이너 `Config.Env` 에는 기록되지 않음)
	- `-u <사용자>` : 실행 사용자 지정 (예 `-u root`)
	- `-w <경로>` : 작업 디렉터리 지정
	- `-d` : 컨테이너 안에서 백그라운드 실행
- 파이프·글로브·리다이렉션은 컨테이너 셸이 처리하도록 `sh -c '...'` 로 감쌈 → [[../../CONTAINER/docker]]
- `redis-cli -a <비밀번호>` : 인증 지정 — **명령행 노출 경고가 출력됨**. 대안 `REDISCLI_AUTH` 환경변수 또는 접속 후 `AUTH <비밀번호>`

```bash
# 경고 없이 인증하는 방법 (권장)
docker exec -e REDISCLI_AUTH='<비밀번호>' -it lab-redis redis-cli ping
```

- 이미지별 셸 차이 : alpine 계열은 `/bin/sh`(busybox), Ubuntu/Debian 계열은 `/bin/bash` → `docker exec -it lab-ubuntu bash`

**검증**

```bash
docker exec lab-redis redis-cli -e -a '<비밀번호>' ping; echo "rc=$?"
docker exec lab-redis redis-cli -a '<비밀번호>' config get requirepass | tail -1 | head -c 4
docker exec lab-redis sh -c 'ls -l /data'
docker exec lab-redis hostname
docker exec lab-redis cat /etc/os-release | head -2
cat /etc/os-release | head -2                      # 호스트와 비교
```

```text
# docker exec -it lab-redis redis-cli -a '<비밀번호>' ping
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
PONG
# docker exec lab-redis sh -c 'ls -l /data'
drwxr-xr-x 2 redis redis 4096 ... appendonlydir
# docker exec lab-redis hostname
<컨테이너 ID 12자>          ← --hostname 미지정 시 컨테이너 ID
# docker exec lab-redis cat /etc/os-release | head -2
NAME="Alpine Linux"
ID=alpine
# cat /etc/os-release | head -2                    ← 호스트
NAME="Rocky Linux"
ID="rocky"
```

- **같은 커널 위에서 Alpine 과 Rocky 의 사용자 공간이 동시에 동작** — 1-3 표의 "커널 공유" 가 여기서 실증됨

> 📝 **시험 포인트**: "실행 중 컨테이너 내부에서 대화식 셸 실행" = `docker exec -it <이름> bash`(필기 R02 #89, R04 #88 ③). `docker run` 은 **새 컨테이너**를 만들므로 오답.

### 4-5. 데이터 영속성 검증 — stop/start, rm -f 후 재생성

> **상황**: 볼륨이 정말로 컨테이너와 독립인지 증명한다. "컨테이너를 지워도 데이터가 남는다" 를 직접 확인해야 백업 대상(Part 12)을 볼륨으로 잡는 근거가 선다.

```bash
# ① 데이터 기록
docker exec lab-redis redis-cli -a '<비밀번호>' set lab:part11 'container-cache-ok'
docker exec lab-redis redis-cli -a '<비밀번호>' set lab:owner 'dev1'
docker exec lab-redis redis-cli -a '<비밀번호>' dbsize
docker exec lab-redis redis-cli -a '<비밀번호>' get lab:part11

# ② 정지 → 시작 (컨테이너 유지)
docker stop lab-redis
docker ps -a --filter name=lab-redis --format '{{.Names}} {{.Status}}'
docker start lab-redis
docker exec lab-redis redis-cli -a '<비밀번호>' get lab:part11        # 유지되어야 함

# ③ 컨테이너 강제 삭제 → 같은 볼륨으로 재생성
docker rm -f lab-redis
docker ps -a --filter name=lab-redis | wc -l                          # 헤더만 = 1
docker volume ls | grep redis-data                                     # 볼륨은 남아 있음
docker run -d --name lab-redis --restart unless-stopped \
  -p 127.0.0.1:6379:6379 -v redis-data:/data \
  redis:7-alpine redis-server --appendonly yes --requirepass '<비밀번호>'
docker exec lab-redis redis-cli -a '<비밀번호>' get lab:part11        # 여전히 유지
```

- `docker stop` : `SIGTERM` 전송 후 유예(기본 10초) 뒤 `SIGKILL` — `-t <초>` 로 유예 조정
- `docker rm -f` : 실행 중이어도 강제 삭제 (**f**orce) ⚠️ `-v` 를 함께 쓰면 **익명 볼륨까지 삭제**되므로 주의
- 이름 있는 볼륨(named volume)은 `docker rm` 으로 **삭제되지 않음** → `docker volume rm` 을 명시해야 사라짐

**검증**

```bash
docker exec lab-redis redis-cli -a '<비밀번호>' dbsize
docker exec lab-redis redis-cli -a '<비밀번호>' get lab:owner
ls -l /var/lib/docker/volumes/redis-data/_data
docker inspect lab-redis --format '{{range .Mounts}}{{.Type}} {{.Name}} -> {{.Destination}}{{end}}'
```

```text
# docker exec lab-redis redis-cli -a '<비밀번호>' dbsize
(integer) 2
# docker exec lab-redis redis-cli -a '<비밀번호>' get lab:part11
"container-cache-ok"
# ls -l /var/lib/docker/volumes/redis-data/_data
drwxr-xr-x 2 ... appendonlydir
# docker inspect lab-redis --format '{{range .Mounts}}{{.Type}} {{.Name}} -> {{.Destination}}{{end}}'
volume redis-data -> /data
```

- 컨테이너를 **완전히 지웠다가 새로 만들었는데도** `lab:part11` 이 살아 있음 = 볼륨이 컨테이너 수명과 독립임을 증명
- 이 `redis-data` 볼륨이 [[12-backup-recovery-review]] 의 백업 대상

> 📝 **시험 포인트**: "컨테이너를 삭제하면 named volume 데이터도 함께 삭제된다" 는 **틀린 설명**(필기 R03 #85 ③). 볼륨은 명시적으로 지워야 사라짐.

### 4-6. 로그 — docker logs

> **상황**: Redis 가 AOF 를 제대로 열었는지, 인증 설정 경고는 없는지 로그로 확인한다.

```bash
docker logs lab-redis                                  # 전체
docker logs --tail 20 lab-redis                        # 마지막 20줄
docker logs --tail 20 --timestamps lab-redis           # 시각 포함
docker logs --since 10m lab-redis                      # 최근 10분
docker logs --since 2026-09-03T00:00:00 --until 2026-09-03T23:59:59 lab-redis
docker logs -f --tail 5 lab-redis                      # 실시간 추적 (Ctrl+C 종료)
docker logs lab-redis 2>&1 | grep -iE 'ready|error|warning'
```

- `-f` : 실시간 추적 (**f**ollow) — `tail -f` 와 동일 개념
- `--tail N` : 마지막 N 줄 (기본 `all`)
- `--since` / `--until` : RFC3339 절대시각 또는 `10m`·`1h` 상대시각
- `-t`, `--timestamps` : 각 줄 앞에 수집 시각 표시
- `--details` : 로그 드라이버가 붙인 추가 속성 표시
- ⚠️ **`docker logs` 는 컨테이너를 삭제하면 함께 사라짐** → 사고 조사 시 초동에 파일로 수집
- 표준출력(stdout)·표준에러(stderr)만 수집 — 애플리케이션이 파일에 직접 쓰는 로그는 안 잡힘

```bash
docker logs lab-redis > /root/lab-redis.log 2>&1       # 파일로 보존
wc -l /root/lab-redis.log
```

**검증**

```bash
docker logs --tail 6 lab-redis
docker logs lab-redis 2>&1 | grep -c 'Ready to accept connections'
ls -l /var/lib/docker/containers/$(docker inspect -f '{{.Id}}' lab-redis)/*-json.log
```

```text
1:M ... * Server initialized
1:M ... * Ready to accept connections tcp
# docker logs lab-redis 2>&1 | grep -c 'Ready to accept connections'
1
# ls -l /var/lib/docker/containers/<64자ID>/<64자ID>-json.log
-rw-r----- 1 root root ... <64자ID>-json.log
```

- 로그의 프로세스 표기가 `1:M` = **컨테이너 안에서 PID 1** (PID 네임스페이스 효과)

> 📝 **시험 포인트**: `docker logs` 는 컨테이너 표준출력 조회. 시스템 서비스 로그는 `journalctl -u`(Part 07)와 대비되는 개념으로 함께 물어볼 수 있음.

### 4-7. 상태·자원·프로세스 — inspect · stats · top · port · diff

> **상황**: 운영 중 컨테이너의 IP·PID·자원 사용량·변경 파일을 조사한다. 침해 조사 시 초동 수집 항목과 동일하다.

```bash
docker inspect lab-redis | head -30
docker inspect lab-redis --format '{{.State.Status}} {{.NetworkSettings.IPAddress}}'
docker inspect lab-redis --format 'Pid={{.State.Pid}} Started={{.State.StartedAt}} Restarts={{.RestartCount}}'
docker inspect lab-redis --format 'Priv={{.HostConfig.Privileged}} Caps={{.HostConfig.CapAdd}} Net={{.HostConfig.NetworkMode}}'
docker inspect lab-redis --format '{{json .Mounts}}' | python3 -m json.tool
docker inspect lab-redis --format '{{range .Config.Cmd}}{{println .}}{{end}}'    # ⚠️ 비밀번호 노출 확인

docker stats --no-stream
docker stats --no-stream --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.NetIO}}'
docker top lab-redis
docker top lab-redis -eo pid,ppid,user,comm            # ps 옵션 그대로 전달 가능
docker port lab-redis
docker port lab-redis 6379
docker diff lab-redis
```

- `docker inspect` : 컨테이너·이미지·볼륨·네트워크 **모든 오브젝트**의 메타데이터 조회 (JSON)
	- `--format`/`-f` : Go 템플릿. `{{json .필드}}` 로 하위 구조를 JSON 문자열로
	- `--type container|image|volume|network` : 이름이 겹칠 때 대상 종류 지정
	- `--size`/`-s` : 쓰기 레이어 크기 포함
- `docker stats` : 컨테이너별 CPU·메모리·네트워크·블록 I/O 실시간
	- `--no-stream` : 1회 스냅샷 후 종료 (스크립트 필수)
	- `-a` : 정지 컨테이너 포함
	- `MEM USAGE / LIMIT` 의 LIMIT 이 곧 cgroup 메모리 상한 (7-6)
- `docker top <컨테이너> [ps 옵션]` : 컨테이너 안 프로세스를 **호스트 시점 PID 로** 표시
- `docker port <컨테이너> [포트]` : 게시된 포트 매핑 조회
- `docker diff <컨테이너>` : **이미지 대비 쓰기 레이어 변경 목록**
	- `A` 추가(**A**dded), `C` 변경(**C**hanged), `D` 삭제(**D**eleted)
	- 볼륨·바인드 마운트 영역은 쓰기 레이어가 아니므로 **여기에 안 잡힘** → 별도 확인 필요
	- 파일 시각(mtime)이 조작돼도 검출됨 → `find -newermt` 의 보완 수단 → [[../../CONTAINER/docker]]

**검증**

```bash
docker inspect lab-redis --format '{{.State.Status}} {{.NetworkSettings.IPAddress}} Pid={{.State.Pid}}'
docker stats --no-stream --format '{{.Name}} {{.CPUPerc}} {{.MemUsage}}'
docker top lab-redis | head -2
docker diff lab-redis | wc -l
```

```text
# docker inspect lab-redis --format '{{.State.Status}} {{.NetworkSettings.IPAddress}} Pid={{.State.Pid}}'
running 172.17.0.2 Pid=12345
# docker stats --no-stream --format '{{.Name}} {{.CPUPerc}} {{.MemUsage}}'
lab-redis 0.15% 9.5MiB / 3.8GiB
# docker top lab-redis
UID    PID    PPID   C   STIME   TTY   TIME       CMD
999    12345  12300  0   14:02   ?     00:00:01   redis-server *:6379
# docker diff lab-redis
C /run
A /run/... 
```

- 볼륨에 쓴 Redis 데이터가 `docker diff` 에 **안 나오는 것이 정상** — `/data` 는 볼륨이라 쓰기 레이어 밖

> 📝 **시험 포인트**: `docker inspect --format` 은 실기 서술형보다 실무형. `docker stats` 는 "CPU 폭주 컨테이너 특정" 맥락으로 Part 06 의 `top` 과 짝지어 나올 수 있음.

### 4-8. 파일 복사 — docker cp

> **상황**: 컨테이너 안의 Redis 설정 샘플을 호스트로 꺼내고, 호스트의 점검 스크립트를 컨테이너로 넣어 본다.

```bash
# 컨테이너 → 호스트
docker cp lab-redis:/usr/local/bin/docker-entrypoint.sh /root/redis-entrypoint.sh
ls -l /root/redis-entrypoint.sh

# 호스트 → 컨테이너
echo 'echo "hello from host"' > /root/hello.sh
docker cp /root/hello.sh lab-redis:/tmp/hello.sh
docker exec lab-redis sh /tmp/hello.sh

# 디렉터리 통째 복사
docker cp lab-redis:/data /root/redis-data-copy
ls -R /root/redis-data-copy | head
```

- `docker cp <컨테이너>:<경로> <호스트경로>` / `docker cp <호스트경로> <컨테이너>:<경로>`
	- `-a` : 소유자·그룹 보존 (**a**rchive)
	- `-L` : 심볼릭 링크를 따라 실제 파일 복사 (**L**ink dereference)
	- 컨테이너가 **정지 상태여도 동작** (`docker exec` 는 실행 중에만 가능)
- ⚠️ 목적지 쓰기는 **명령을 실행한 사용자 권한**으로 처리 → `/root` 로 바로 못 넣는 경우 홈 경유 후 `sudo mv` → [[../../CONTAINER/docker]]

**검증**

```bash
head -1 /root/redis-entrypoint.sh
docker exec lab-redis ls -l /tmp/hello.sh
docker exec lab-redis sh /tmp/hello.sh
docker diff lab-redis | grep hello.sh          # 쓰기 레이어에 잡히는지
rm -rf /root/redis-data-copy /root/hello.sh
```

```text
# head -1 /root/redis-entrypoint.sh
#!/bin/sh
# docker exec lab-redis sh /tmp/hello.sh
hello from host
# docker diff lab-redis | grep hello.sh
A /tmp/hello.sh
```

- 방금 넣은 파일이 `docker diff` 에 `A` 로 잡힘 → 4-7 의 변경 추적이 실제로 동작함을 확인

> 📝 **시험 포인트**: `docker cp` 는 양방향. 정지 컨테이너에도 쓸 수 있다는 점이 `exec` 와의 결정적 차이.

### 4-9. 생명주기 제어 — pause · restart · kill · rename · update · wait

> **상황**: 컨테이너 상태 전이 명령을 모두 한 번씩 실행해 상태값이 어떻게 바뀌는지 확인한다.

```bash
docker pause lab-redis;    docker ps --format '{{.Names}} {{.Status}}'
docker unpause lab-redis;  docker ps --format '{{.Names}} {{.Status}}'

docker restart lab-redis                                   # stop + start
docker restart -t 3 lab-redis                              # SIGTERM 후 3초 뒤 KILL

docker kill -s TERM lab-redis                              # 시그널 직접 전송
docker ps -a --format '{{.Names}} {{.Status}}'
docker start lab-redis

docker rename lab-redis lab-redis-old
docker ps --format '{{.Names}}'
docker rename lab-redis-old lab-redis                      # 원복

docker update --memory 256m --memory-swap 256m lab-redis   # 실행 중 자원 한도 변경
docker inspect lab-redis --format '{{.HostConfig.Memory}}'

docker run -d --name tmp-wait redis:7-alpine sh -c 'sleep 3; exit 7'
docker wait tmp-wait                                        # 종료까지 대기 → 종료 코드 출력
docker rm tmp-wait
```

| 명령 | 동작 | 결과 상태 |
| --- | --- | --- |
| `docker pause` | cgroup freezer 로 **전 프로세스 동결** (메모리 유지) | `paused` |
| `docker unpause` | 동결 해제 | `running` |
| `docker stop [-t N]` | SIGTERM → N초 후 SIGKILL (기본 10초) | `exited (0)` |
| `docker kill [-s SIG]` | 즉시 시그널 전송 (기본 **SIGKILL**) | `exited (137)` 등 |
| `docker restart [-t N]` | stop 후 start | `running` |
| `docker start [-a] [-i]` | 정지 컨테이너 재시작 (`-a` 출력 연결, `-i` 입력 연결) | `running` |
| `docker rename <구> <신>` | 이름 변경 (ID·데이터 불변) | 변화 없음 |
| `docker update <옵션>` | 자원 한도·재시작 정책 변경 (재생성 불필요) | 변화 없음 |
| `docker wait` | 종료될 때까지 **블로킹** 후 종료 코드 출력 | — |

- 종료 코드 관례 : `0` 정상, `137` = 128+9(SIGKILL), `143` = 128+15(SIGTERM), `125` docker 자체 오류, `126` 실행 불가, `127` 명령 없음
- `docker update` 로 바꿀 수 있는 것 : `--cpus` `--memory` `--memory-swap` `--pids-limit` `--restart` 등. **포트·볼륨·환경변수는 변경 불가** → 재생성해야 함

**검증**

```bash
docker inspect lab-redis --format '{{.State.Status}} Restarts={{.RestartCount}} Mem={{.HostConfig.Memory}}'
docker ps --format 'table {{.Names}}\t{{.Status}}'
docker exec lab-redis redis-cli -a '<비밀번호>' get lab:part11     # 재시작 후에도 데이터 유지
```

```text
# docker pause 직후
lab-redis Up 12 minutes (Paused)
# docker kill -s TERM 직후
lab-redis Exited (0) 2 seconds ago
# docker inspect ... 
running Restarts=0 Mem=268435456
# docker exec ... get lab:part11
"container-cache-ok"
```

> 📝 **시험 포인트**: 컨테이너 라이프사이클 순서 `pull → create → start → stop → rm`(필기 R07 #88). `pause` 는 정지가 아니라 **동결**, `kill` 기본 시그널은 SIGKILL(호스트 `kill` 의 기본은 SIGTERM 인 것과 대비).

### 4-10. attach vs exec

> **상황**: 둘 다 "컨테이너 안으로 들어가는" 명령처럼 보이지만 성격이 다르다. 잘못 쓰면 컨테이너를 죽인다.

| 구분 | `docker attach` | `docker exec` |
| --- | --- | --- |
| 대상 | **PID 1(메인 프로세스)의 표준입출력에 연결** | **새 프로세스를 추가 실행** |
| 여러 개 | 여러 터미널이 붙으면 화면 공유 | 각자 독립 세션 |
| Ctrl+C | 메인 프로세스에 SIGINT → **컨테이너 종료** ⚠️ | 방금 띄운 프로세스만 종료 |
| 빠져나오기 | `Ctrl+P` `Ctrl+Q` (detach 키) | `exit` |
| 셸 필요 | 메인 프로세스가 셸이어야 의미 있음 | 이미지에 셸만 있으면 됨 |
| 용도 | 포그라운드 로그 확인, 대화형 메인 프로세스 | 점검·디버깅 (일반적 선택) |

```bash
# ⚠️ attach 후 Ctrl+C 를 누르면 lab-redis 가 종료됨 — 반드시 Ctrl+P, Ctrl+Q 로 빠져나올 것
docker attach lab-redis
# (로그가 흐름) → Ctrl+P 이어서 Ctrl+Q

# 안전한 대안 두 가지
docker logs -f --tail 5 lab-redis        # 출력만 볼 때
docker exec -it lab-redis sh             # 안에서 명령을 칠 때
```

- `docker attach --no-stdin` : 표준입력을 붙이지 않아 실수로 시그널을 보낼 위험 감소
- `docker attach --sig-proxy=false` : Ctrl+C 가 컨테이너로 전달되지 않게 함

**검증**

```bash
docker ps --filter name=lab-redis --format '{{.Names}} {{.Status}}'   # attach 후에도 Up 이어야 정상
docker exec lab-redis redis-cli -a '<비밀번호>' ping
```

```text
lab-redis Up 20 minutes
PONG
```

> 📝 **시험 포인트**: "실행 중 컨테이너 내부에서 셸 실행" 의 정답은 `exec`, `attach -it web bash` 는 **문법 자체가 틀린** 오답 선지(필기 R02 #89 ③) — `attach` 는 명령 인자를 받지 않음.

### 4-11. 커널 공유 체감 — 호스트 PID · 네임스페이스

> **상황**: 컨테이너가 "가벼운 VM" 이 아니라 **호스트의 프로세스**임을 눈으로 확인한다. 동시에 4-2 에서 경고한 비밀번호 노출도 실제로 드러난다.

```bash
ps -ef | grep redis-server | grep -v grep                     # 호스트 프로세스 목록에 그대로 보임
PID=$(docker inspect lab-redis --format '{{.State.Pid}}')
echo "container PID on host = $PID"
ps -p "$PID" -o pid,ppid,user,args
docker exec lab-redis ps                                      # 컨테이너 안에서는 PID 1
ls -l /proc/"$PID"/ns/                                        # 컨테이너 프로세스의 네임스페이스
ls -l /proc/1/ns/                                             # 호스트 PID 1 과 비교
cat /proc/"$PID"/cgroup                                       # 소속 cgroup 경로
```

- 호스트 `ps` 에 보이는 UID `999` = 컨테이너 안의 `redis` 사용자 UID. **user 네임스페이스를 안 쓰면 UID 가 그대로 호스트 UID**
- `/proc/<PID>/ns/*` 의 inode 를 호스트 PID 1 과 비교
	- `mnt`·`pid`·`net`·`ipc`·`uts` : **다름** → 격리됨
	- `user` : **같음** → user 네임스페이스 미사용(rootless 아님)
- ⚠️ `ps -ef` 출력에 `--requirepass <비밀번호>` 가 **평문으로 노출** → 4-2 에서 `--env-file` 을 권장한 이유

```bash
# 컨테이너 네트워크 네임스페이스 안에서 호스트 도구 실행 (참고)
nsenter -t "$PID" -n ss -tlnp                 # 컨테이너 시점 리스닝 포트
nsenter -t "$PID" -u hostname                 # 컨테이너 시점 호스트명
nsenter -t "$PID" -m ls /data                 # 컨테이너 시점 마운트
```

- `nsenter` : 다른 프로세스의 네임스페이스로 진입해 명령 실행 (**n**ame**s**pace **enter**, `util-linux`)
	- `-t <PID>` : 대상 프로세스 (**t**arget), `-n` : network, `-m` : mount, `-u` : uts, `-p` : pid, `-a` : 전부
	- 컨테이너에 진단 도구(`ss`, `tcpdump`)가 없을 때 **호스트 도구를 컨테이너 네임스페이스에서** 쓰는 실무 기법

**검증**

```bash
PID=$(docker inspect lab-redis --format '{{.State.Pid}}')
ps -p "$PID" -o pid,user,args --no-headers
readlink /proc/"$PID"/ns/pid /proc/1/ns/pid          # 다름
readlink /proc/"$PID"/ns/user /proc/1/ns/user        # 같음
docker exec lab-redis ps -o pid,comm | head -3
nsenter -t "$PID" -n ss -tln | grep 6379
```

```text
# ps -ef | grep redis-server
999  12345  12300 ... redis-server *:6379            ← 호스트에서 그대로 보임
# ps -p 12345 -o pid,user,args --no-headers
12345 999 redis-server *:6379
# readlink /proc/12345/ns/pid /proc/1/ns/pid
pid:[4026532xxx]         ← 컨테이너
pid:[4026531836]         ← 호스트 (다름 = 격리)
# readlink /proc/12345/ns/user /proc/1/ns/user
user:[4026531837]
user:[4026531837]        ← 같음 = user 네임스페이스 미사용
# docker exec lab-redis ps -o pid,comm | head -3
PID   COMMAND
    1 redis-server         ← 컨테이너 안에서는 PID 1
# nsenter -t 12345 -n ss -tln | grep 6379
LISTEN 0 511 0.0.0.0:6379 0.0.0.0:*
```

- **같은 프로세스가 호스트에서는 12345, 컨테이너에서는 1** → PID 네임스페이스의 실체
- 컨테이너 네임스페이스 안에서는 `0.0.0.0:6379` 리슨이지만, 호스트 쪽 게시는 `127.0.0.1:6379` → 외부 노출은 **호스트 바인딩이 결정**

> 📝 **시험 포인트**: "컨테이너는 호스트 커널을 공유하므로 오버헤드가 작다"(필기 R07 #87 ㄷ)의 근거가 바로 이 화면. VM 이라면 호스트 `ps` 에 게스트 프로세스가 보이지 않음.

### 4-12. 삭제와 정리 — rm · container prune

> **상황**: 실습 중 만든 임시 컨테이너를 정리한다. 삭제 범위를 명확히 구분한다.

```bash
docker ps -a --format 'table {{.Names}}\t{{.Status}}'
docker rm tmp-wait 2>/dev/null || true                # 정지 컨테이너 삭제
docker rm -f <이름|ID>                                 # ⚠️ 실행 중이어도 강제 삭제
docker rm -v <이름>                                    # ⚠️ 익명 볼륨까지 함께 삭제
docker container prune                                 # 정지된 컨테이너 일괄 삭제 (확인 프롬프트)
docker container prune -f --filter 'until=1h'          # 1시간 이상 지난 것만, 확인 없이
docker ps -aq | wc -l
```

- `docker rm` : **정지된** 컨테이너 삭제가 기본. 실행 중이면 `Error: You cannot remove a running container`
	- `-f` : 강제 (**f**orce) — 내부적으로 SIGKILL 후 삭제
	- `-v` : 이 컨테이너에 붙은 **익명 볼륨**도 삭제 (**v**olume) — 이름 있는 볼륨은 대상 아님
	- `-l` : 링크만 제거 (**l**ink, 레거시)
- `docker container prune` : 정지 상태 컨테이너 전부 삭제 → ⚠️ 조사 대상이 있으면 먼저 `docker diff`·`docker logs` 수집

**검증**

```bash
docker ps -a --format '{{.Names}} {{.Status}}'
docker volume ls                                       # named volume 은 남아 있어야 함
```

```text
# docker ps -a --format '{{.Names}} {{.Status}}'
lab-redis Up 25 minutes
# docker volume ls
DRIVER    VOLUME NAME
local     redis-data
```

> 📝 **시험 포인트**: `docker rm` = 컨테이너, `docker rmi` = 이미지, `docker volume rm` = 볼륨. 세 명령의 대상 구분이 곧 오답 선지의 축.

---

## 5. 볼륨과 네트워크

### 5-1. 볼륨 관리 — create · ls · inspect · rm · prune

> **상황**: 4절에서 만든 `redis-data` 볼륨이 호스트 어디에 있는지 확인하고, 볼륨 관리 명령을 전부 익힌다.

```bash
docker volume ls
docker volume ls -q
docker volume ls --filter dangling=true                # 어떤 컨테이너도 안 쓰는 볼륨
docker volume inspect redis-data
docker volume inspect redis-data --format '{{.Mountpoint}} {{.Driver}} {{.CreatedAt}}'
ls -l /var/lib/docker/volumes/redis-data/_data          # 실제 데이터 경로
du -sh /var/lib/docker/volumes/redis-data/_data

docker volume create lab-tmpvol                         # 생성
docker volume create --label purpose=lab --driver local lab-tmpvol2
docker volume ls --filter label=purpose=lab
docker volume rm lab-tmpvol lab-tmpvol2                 # 삭제 (사용 중이면 거부)
docker volume prune -f                                  # ⚠️ 미사용 볼륨 일괄 삭제
```

- `docker volume create [이름]` : 이름 생략 시 임의 해시 이름의 **익명 볼륨**
	- `--driver`/`-d` : 볼륨 드라이버 (기본 `local`, 플러그인으로 NFS·클라우드 스토리지 연결 가능)
	- `--opt`/`-o` : 드라이버 옵션 (예 `-o type=nfs -o device=:/export`)
	- `--label` : 메타데이터 라벨
- `docker volume inspect` 주요 필드 : `Mountpoint`(호스트 실제 경로), `Driver`, `Scope`, `Labels`
- `docker volume rm` : **사용 중인 볼륨은 삭제 거부** → 컨테이너를 먼저 지워야 함
- ⚠️ `docker volume prune` 은 **컨테이너가 참조하지 않는 모든 볼륨**을 지움 — 잠시 컨테이너를 지운 상태에서 실행하면 데이터가 날아감. 실행 전 `docker volume ls --filter dangling=true` 로 대상 확인 필수

**검증**

```bash
docker volume inspect redis-data --format '{{.Mountpoint}}'
ls /var/lib/docker/volumes/redis-data/_data
docker volume ls --format 'table {{.Driver}}\t{{.Name}}'
docker ps -a --filter volume=redis-data --format '{{.Names}}'    # 이 볼륨을 쓰는 컨테이너
```

```text
# docker volume inspect redis-data --format '{{.Mountpoint}}'
/var/lib/docker/volumes/redis-data/_data
# ls /var/lib/docker/volumes/redis-data/_data
appendonlydir
# docker ps -a --filter volume=redis-data --format '{{.Names}}'
lab-redis
```

- 볼륨 3형태 비교
	- **이름 있는 볼륨** (`-v redis-data:/data`) : Docker 가 경로를 관리, 백업·이전 용이, **권장**
	- **익명 볼륨** (`-v /data`) : 이름이 해시, `docker rm -v` 로 사라짐 → 관리 어려움
	- **바인드 마운트** (`-v /srv/share/redis:/data`) : 호스트 경로 직접 지정, SELinux·권한 문제 발생 지점(5-2)

> 📝 **시험 포인트**: "볼륨은 컨테이너 생명주기와 독립적으로 데이터를 보존한다" 가 정답 문장(필기 R03 #85 ④). 호스트 실제 경로 `/var/lib/docker/volumes/<이름>/_data` 도 기억.

### 5-2. 바인드 마운트와 SELinux — :z / :Z

> **상황**: 백업 스크립트가 접근하기 쉽게 Redis 데이터를 `/srv/share` 밑에 두고 싶다. 그런데 Part 10 에서 SELinux 를 enforcing 으로 두었기 때문에 그냥은 안 된다. **일부러 실패를 재현**하고 라벨로 해결한다.

```bash
getenforce                                        # Enforcing 확인 (Part 10)
mkdir -p /srv/share/redis
ls -Zd /srv/share/redis                           # 현재 SELinux 라벨
```

⚠️ 아래 첫 시도는 **실패하는 것이 정상** — 실패 화면을 확인하는 것이 목적

```bash
docker run --rm --name t-bind -v /srv/share/redis:/data redis:7-alpine \
  sh -c 'echo test > /data/probe.txt && cat /data/probe.txt'
```

```text
sh: can't create /data/probe.txt: Permission denied
```

```bash
ausearch -m avc -ts recent | tail -20             # SELinux 거부 감사 로그
ausearch -m avc -ts recent | grep -oE 'tcontext=[^ ]+' | tail -3
```

- 원인 : 컨테이너 프로세스는 `container_t` 도메인으로 동작하는데, 호스트 디렉터리 라벨은 `samba_share_t`(Part 09) 또는 `var_t`/`default_t` → **접근 정책 위반**
- 해결 : 마운트 옵션에 라벨 재지정 플래그를 붙임

| 옵션 | 동작 | 사용 시점 |
| --- | --- | --- |
| `:z` (소문자) | `container_file_t` **공유 라벨** 부여 — 여러 컨테이너가 함께 접근 | 공유 디렉터리 |
| `:Z` (대문자) | `container_file_t` + **MCS 카테고리**(`s0:c1,c2`) — 해당 컨테이너 전용 | 단독 사용 데이터 |
| `:ro` | 읽기 전용 | 원본 보호 |
| `:rw` | 읽기·쓰기 (기본) | — |

```bash
# 공유 라벨(:z) 로 재시도
docker run --rm --name t-bind -v /srv/share/redis:/data:z redis:7-alpine \
  sh -c 'echo test > /data/probe.txt && cat /data/probe.txt'
ls -Z /srv/share/redis/

# 전용 라벨(:Z) 로 재시도 — MCS 카테고리가 붙음
docker run --rm --name t-bind2 -v /srv/share/redis:/data:Z redis:7-alpine \
  sh -c 'echo test2 >> /data/probe.txt; ls -l /data'
ls -Zd /srv/share/redis
```

- ⚠️ `:z`/`:Z` 는 **대상 디렉터리 라벨을 실제로 변경**함 → `/srv/share` 처럼 Samba 가 함께 쓰는 경로에 걸면 Samba 접근이 깨질 수 있음. 재라벨은 `restorecon -Rv <경로>` 로 복구
- 라벨을 바꾸지 않고 허용하려면 불리언 `container_use_share_t` 계열이나 `semanage fcontext` 로 정책 추가 → [[10-security-firewall-selinux]]
- **`--privileged` 로 우회하지 말 것** — SELinux 제약을 통째로 해제하는 것이라 격리가 사라짐

**검증**

```bash
ls -Z /srv/share/redis/probe.txt
cat /srv/share/redis/probe.txt
ausearch -m avc -ts recent | wc -l                # 해결 후 새 거부가 안 늘어야 함
# 실습 정리 — 원래 라벨로 복구
restorecon -Rv /srv/share/redis
ls -Zd /srv/share/redis
rm -rf /srv/share/redis
```

```text
# (첫 시도) 
sh: can't create /data/probe.txt: Permission denied
# ausearch -m avc -ts recent | tail -2
type=AVC ... avc:  denied  { write } for  pid=... comm="sh" name="redis" dev="dm-.." 
  scontext=system_u:system_r:container_t:s0:c... tcontext=system_u:object_r:samba_share_t:s0 tclass=dir
# (:z 재시도)
test
# ls -Z /srv/share/redis/
system_u:object_r:container_file_t:s0 probe.txt
# (:Z 사용 시 디렉터리 라벨)
system_u:object_r:container_file_t:s0:c123,c456 /srv/share/redis
```

- **정리** : 이 실습의 `lab-redis` 는 바인드 마운트가 아니라 **이름 있는 볼륨**을 쓴다 → Docker 가 `/var/lib/docker/volumes` 를 이미 올바른 라벨로 관리하므로 SELinux 문제가 없음. 바인드 마운트를 택할 때만 `:z`/`:Z` 가 필요

> 📝 **시험 포인트**: SELinux 거부는 `getenforce` → `ausearch -m avc` → 라벨 확인(`ls -Z`) → 수정(`chcon`/`restorecon`/`semanage`) 순서로 진단(Part 10). 컨테이너 맥락에서는 `container_file_t` 와 `:z`/`:Z` 가 정답 키워드.

### 5-3. tmpfs 마운트와 --mount vs -v

> **상황**: 디스크에 남기면 안 되는 임시 파일은 메모리에 두는 편이 낫다. 마운트 문법 두 가지도 비교한다.

```bash
# tmpfs — 메모리 기반, 컨테이너 종료 시 소멸
docker run --rm --tmpfs /scratch:rw,size=32m,mode=1777 redis:7-alpine \
  sh -c 'df -h /scratch; echo secret > /scratch/x; ls -l /scratch'

# --mount 문법 (동일 동작을 명시적으로)
docker run --rm --mount type=tmpfs,destination=/scratch,tmpfs-size=33554432 redis:7-alpine \
  sh -c 'df -h /scratch'

# 볼륨을 --mount 로
docker run --rm --mount type=volume,source=redis-data,target=/data,readonly redis:7-alpine ls -l /data

# 바인드를 --mount 로
docker run --rm --mount type=bind,source=/etc/hostname,target=/host-hostname,readonly \
  redis:7-alpine cat /host-hostname
```

| 구분 | `-v` (`--volume`) | `--mount` |
| --- | --- | --- |
| 문법 | 콜론 구분 위치 인자 `src:dst:opts` | `key=value` 쉼표 나열 |
| 가독성 | 짧음 | 길지만 **명시적** |
| 존재하지 않는 호스트 경로 | **자동으로 디렉터리 생성** ⚠️ 오타 시 빈 디렉터리 마운트 | **오류로 실패** (안전) |
| tmpfs | `--tmpfs` 별도 옵션 | `type=tmpfs` 로 통합 |
| 드라이버 옵션 | 지정 불가 | `volume-opt=` 로 지정 가능 |
| 권장 | 간단한 경우 | 스크립트·운영 |

- `--mount` 키 : `type`(volume/bind/tmpfs), `source`/`src`, `destination`/`dst`/`target`, `readonly`/`ro`, `bind-propagation`, `volume-driver`, `tmpfs-size`, `tmpfs-mode`
- 읽기 전용 루트와 결합하면 견고해짐 : `--read-only --tmpfs /tmp`

**검증**

```bash
docker run --rm --tmpfs /scratch:size=32m redis:7-alpine sh -c 'df -hT /scratch | tail -1'
docker run --rm -v /nonexistent-path-test:/x redis:7-alpine ls /x; ls -d /nonexistent-path-test
docker run --rm --mount type=bind,source=/nonexistent-path-test2,target=/x redis:7-alpine ls /x 2>&1 | tail -1
rmdir /nonexistent-path-test 2>/dev/null
```

```text
# df -hT /scratch
tmpfs   tmpfs   32.0M   0  32.0M   0% /scratch
# (-v 로 없는 경로)  → 호스트에 빈 디렉터리가 생김
/nonexistent-path-test
# (--mount 로 없는 경로)
docker: Error response from daemon: bind source path does not exist: /nonexistent-path-test2
```

> 📝 **시험 포인트**: `-v` 는 **호스트 경로를 자동 생성**한다는 점이 실무 함정(오타 → 데이터 없는 빈 디렉터리). tmpfs 는 "메모리 기반, 재시작 시 소멸".

### 5-4. 네트워크 드라이버와 목록

> **상황**: 컨테이너 네트워크의 종류를 파악하고 기본 브리지 구조를 확인한다.

```bash
docker network ls
docker network ls --format 'table {{.Name}}\t{{.Driver}}\t{{.Scope}}'
docker network inspect bridge --format '{{.IPAM.Config}} {{.Options}}'
docker network inspect bridge --format '{{range $k,$v := .Containers}}{{$v.Name}} {{$v.IPv4Address}}{{end}}'
ip -br a show docker0
bridge link                                     # docker0 에 붙은 veth 인터페이스
ip route | grep docker0
```

| 드라이버 | 동작 | 용도 |
| --- | --- | --- |
| `bridge` | 호스트 안의 가상 브리지(`docker0`) + NAT. **기본값** | 단일 호스트 컨테이너 간 통신 |
| `host` | 호스트 네트워크 스택 **그대로 사용** (격리 없음) | 최고 성능, 포트 매핑 불필요 |
| `none` | `lo` 만 존재, 외부 통신 불가 | 격리 분석·오프라인 처리 |
| `overlay` | 여러 호스트에 걸친 가상 네트워크 (VXLAN) | Swarm·다중 호스트 클러스터 |
| `macvlan` | 컨테이너에 **물리망의 MAC/IP 직접 부여** | 레거시 장비 통합 |
| `ipvlan` | macvlan 유사, L2/L3 모드 | MAC 제한 환경 |

- 기본 `bridge` 네트워크의 특징 : **컨테이너 이름으로 서로를 찾지 못함**(내장 DNS 미적용) → 사용자 정의 브리지를 만들어야 이름 해석 가능(5-5)
- `veth` 쌍 : 컨테이너의 `eth0` 과 호스트의 `vethXXXX` 가 짝 → `bridge link` 로 확인

**검증**

```bash
docker network ls --format '{{.Name}}:{{.Driver}}' | sort
ip -br a show docker0
bridge link | grep -c veth
docker network inspect bridge --format '{{(index .IPAM.Config 0).Subnet}}'
```

```text
# docker network ls --format '{{.Name}}:{{.Driver}}'
bridge:bridge
host:host
none:null
# ip -br a show docker0
docker0   UP   172.17.0.1/16
# docker network inspect bridge --format '{{(index .IPAM.Config 0).Subnet}}'
172.17.0.0/16
# bridge link | grep -c veth
1
```

> 📝 **시험 포인트**: 기본 3종 `bridge` `host` `none` 이름과 성격은 필기 대비 필수. `overlay` = 다중 호스트, `macvlan` = 물리망 직접 노출.

### 5-5. 사용자 정의 브리지 — 이름 해석 검증

> **상황**: 웹 컨테이너가 `lab-redis` 라는 **이름**으로 캐시에 접속하게 하려면 사용자 정의 네트워크가 필요하다. 기본 브리지와의 차이를 실험으로 확인한다.

```bash
# ① 기본 브리지에서는 이름 해석 실패
docker run --rm redis:7-alpine redis-cli -h lab-redis ping 2>&1 | tail -1

# ② 사용자 정의 브리지 생성
docker network create lab-net
docker network create --driver bridge --subnet 172.28.0.0/16 --gateway 172.28.0.1 lab-net2
docker network ls --filter driver=bridge
docker network inspect lab-net --format '{{(index .IPAM.Config 0).Subnet}} {{.Driver}}'

# ③ lab-redis 를 lab-net 에 연결
docker network connect lab-net lab-redis
docker inspect lab-redis --format '{{range $k,$v := .NetworkSettings.Networks}}{{$k}}={{$v.IPAddress}} {{end}}'

# ④ 같은 네트워크의 다른 컨테이너에서 이름으로 접속
docker run --rm --network lab-net redis:7-alpine redis-cli -h lab-redis -a '<비밀번호>' ping
docker run --rm --network lab-net redis:7-alpine sh -c 'nslookup lab-redis 2>/dev/null || getent hosts lab-redis'
```

- `docker network create <이름>` : 사용자 정의 네트워크 생성
	- `--driver`/`-d` : 드라이버 (기본 `bridge`)
	- `--subnet` : 대역 지정 (미지정 시 자동 할당)
	- `--gateway` : 게이트웨이 주소
	- `--ip-range` : 실제 할당 범위를 서브넷 일부로 제한
	- `--internal` : 외부 통신 차단(내부 전용)
	- `--attachable` : overlay 네트워크에 개별 컨테이너 연결 허용
	- `--label`, `--opt`
- **사용자 정의 브리지 vs 기본 bridge**

| 항목 | 기본 `bridge` | 사용자 정의 브리지 |
| --- | --- | --- |
| 컨테이너 이름 DNS | ❌ (`--link` 레거시만) | ✅ **내장 DNS 로 이름 해석** |
| 격리 | 모든 컨테이너가 한 대역 | 네트워크 단위로 분리 |
| 연결 변경 | 재생성 필요 | `connect`/`disconnect` 로 실행 중 변경 |
| 별칭 | ❌ | `--network-alias` 지원 |

**검증**

```bash
docker run --rm --network lab-net redis:7-alpine redis-cli -h lab-redis -a '<비밀번호>' ping
docker inspect lab-redis --format '{{range $k,$v := .NetworkSettings.Networks}}{{$k}}={{$v.IPAddress}} {{end}}'
docker network inspect lab-net --format '{{range $k,$v := .Containers}}{{$v.Name}} {{$v.IPv4Address}}{{end}}'
```

```text
# ① 기본 브리지
Could not connect to Redis at lab-redis:6379: Name does not resolve
# ④ 사용자 정의 브리지
Warning: Using a password with '-a' ... may not be safe.
PONG
# docker inspect lab-redis --format '...'
bridge=172.17.0.2 lab-net=172.18.0.2
# docker network inspect lab-net --format '...'
lab-redis 172.18.0.2/16
```

- 컨테이너 하나가 **여러 네트워크에 동시에 소속**될 수 있음(주소가 2개)

> 📝 **시험 포인트**: "컨테이너 이름으로 통신하려면 사용자 정의 네트워크가 필요" 는 실무형 핵심. Compose 는 스택마다 네트워크를 자동 생성하므로 서비스 이름 해석이 기본 동작(7-5).

### 5-6. connect/disconnect · 포트 매핑 vs --network host

> **상황**: 네트워크 연결을 실행 중에 붙였다 떼고, `host` 네트워크와 포트 매핑의 차이를 눈으로 비교한다.

```bash
# 연결 해제
docker network disconnect lab-net lab-redis
docker inspect lab-redis --format '{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}'
# 다시 연결 (별칭 부여)
docker network connect --alias cache lab-net lab-redis
docker run --rm --network lab-net redis:7-alpine redis-cli -h cache -a '<비밀번호>' ping

# 네트워크 모드 3종 비교 — 보이는 인터페이스로 구분
docker run --rm redis:7-alpine ls /sys/class/net                       # bridge(기본)
docker run --rm --network host redis:7-alpine ls /sys/class/net        # host
docker run --rm --network none redis:7-alpine ls /sys/class/net        # none
docker run --rm --network host redis:7-alpine hostname                 # 호스트명도 호스트 것
```

- `docker network connect [--alias <별칭>] [--ip <주소>] <네트워크> <컨테이너>` : 실행 중 연결
- `docker network disconnect [-f] <네트워크> <컨테이너>` : 연결 해제
- `--network host` 특징
	- 포트 매핑(`-p`) **불필요하며 무시됨** — 컨테이너가 호스트 포트에 직접 바인딩
	- 네트워크 네임스페이스를 격리하지 않음 → **포트 충돌 위험**(Part 09 의 httpd 80 과 겹치면 기동 실패)
	- NAT·docker-proxy 를 거치지 않아 성능 최고, 그러나 **격리 최저**
- `--network none` : `lo` 만 존재 → 침해 데이터 분석 등 **외부 통신을 원천 차단**해야 할 때 → [[../../CONTAINER/docker]]

⚠️ `--network host` 로 웹 서버를 띄우면 Part 09 의 httpd(80) 와 충돌할 수 있음 — 아래는 조회 명령만 수행

**검증**

```bash
echo "== bridge";  docker run --rm redis:7-alpine ls /sys/class/net | tr '\n' ' '
echo; echo "== host"; docker run --rm --network host redis:7-alpine ls /sys/class/net | tr '\n' ' '
echo; echo "== none"; docker run --rm --network none redis:7-alpine ls /sys/class/net | tr '\n' ' '
echo; docker run --rm --network host redis:7-alpine hostname; hostname
```

```text
== bridge
eth0 lo
== host
docker0 enp0s1 lo ...            ← 호스트의 인터페이스가 그대로 보임
== none
lo
# docker run --rm --network host redis:7-alpine hostname
srv01.lab.local
# hostname
srv01.lab.local                  ← 동일 (UTS 네임스페이스도 공유)
```

> 📝 **시험 포인트**: `-p` 포트 매핑은 **bridge 모드에서만 의미**가 있음. `--network host` 는 매핑 없이 호스트 포트를 그대로 사용 — "포트 매핑 없이 서비스가 뜬 이유" 문항의 정답.

### 5-7. docker0 브리지 확인과 네트워크 정리

> **상황**: 실습으로 늘어난 네트워크를 정리하고, `lab-net` 만 남긴다(7절 Compose 에서 사용).

```bash
ip -br link show type bridge                     # 브리지 인터페이스 전체
bridge link                                       # 브리지에 연결된 포트(veth)
ip -d link show docker0 | head -3                 # 브리지 상세
iptables -t nat -S POSTROUTING | grep 172.17      # 컨테이너 대역 MASQUERADE 규칙

docker network rm lab-net2                        # 사용하지 않는 네트워크 삭제
docker network prune -f                           # ⚠️ 컨테이너가 안 쓰는 네트워크 일괄 삭제
docker network ls
```

- `docker network rm <이름>` : 연결된 컨테이너가 있으면 삭제 거부
- `docker network prune` : 사용 중이 아닌 **사용자 정의 네트워크**만 삭제 (`bridge`/`host`/`none` 기본 3종은 삭제 불가)
- `bridge link` : 브리지에 물린 인터페이스 목록 (`iproute2`) — 컨테이너 1개당 `vethXXXX` 1개

**검증**

```bash
docker network ls --format '{{.Name}}:{{.Driver}}' | sort
ip -br a show docker0
iptables -t nat -S POSTROUTING | grep -c 172.17
docker network inspect lab-net --format '{{range $k,$v := .Containers}}{{$v.Name}}{{end}}'
```

```text
# docker network ls --format '{{.Name}}:{{.Driver}}' | sort
bridge:bridge
host:host
lab-net:bridge
none:null
# iptables -t nat -S POSTROUTING | grep 172.17
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
# docker network inspect lab-net --format '...'
lab-redis
```

- `MASQUERADE` 규칙 = 컨테이너가 외부로 나갈 때 출발지를 호스트 IP 로 바꾸는 **동적 SNAT** → Part 10 의 iptables NAT 개념과 동일

> 📝 **시험 포인트**: 컨테이너 → 외부 통신은 **MASQUERADE(SNAT)**, 외부 → 컨테이너는 **DNAT(포트 게시)**. Part 10 의 `-t nat` 체인 해석 문항과 그대로 연결됨.

---

## 6. Ubuntu 컨테이너에서 apt · dpkg

> 이 절은 **기출 대비 핵심**이다. 시험은 RedHat 계열(`rpm`/`dnf`)과 Debian 계열(`dpkg`/`apt`)을 나란히 묻는데, Rocky 서버만으로는 Debian 명령을 실행해 볼 수 없다. Ubuntu 컨테이너가 그 실습 환경이 된다.

### 6-1. lab-ubuntu 기동과 배포판 확인

> **상황**: 팀 일부가 Ubuntu 를 쓴다. 같은 서버 안에 Ubuntu 24.04 컨테이너를 띄워 패키지 관리를 익힌다.

```bash
docker run -it --name lab-ubuntu --hostname ubuntu-lab ubuntu:24.04 bash
```

- `-it` : 대화형 셸이므로 표준입력 + TTY 필수 (`-d` 로 띄우면 `bash` 가 할 일이 없어 즉시 종료)
- `--hostname ubuntu-lab` : UTS 네임스페이스로 컨테이너 호스트명 지정
- ubuntu 이미지의 기본 `CMD` 는 `/bin/bash` 이므로 마지막 `bash` 는 생략 가능(명시가 명확)

**컨테이너 안에서 실행** (프롬프트 `root@ubuntu-lab:/#`)

```bash
cat /etc/os-release
cat /etc/debian_version
hostname
uname -a                       # ★ 커널은 호스트(Rocky)의 것
arch
ls /etc/apt/
which apt dpkg apt-get apt-cache
dpkg -l | wc -l                # 초기 설치 패키지 수
```

**검증** (컨테이너 안)

```bash
grep -E '^(NAME|VERSION_ID|ID|VERSION_CODENAME)=' /etc/os-release
uname -r
cat /proc/version | head -c 60; echo
```

```text
root@ubuntu-lab:/# grep -E '^(NAME|VERSION_ID|ID|VERSION_CODENAME)=' /etc/os-release
NAME="Ubuntu"
ID=ubuntu
VERSION_ID="24.04"
VERSION_CODENAME=noble
root@ubuntu-lab:/# cat /etc/debian_version
trixie/sid
root@ubuntu-lab:/# uname -r
5.14.0-....el9.aarch64          ← ★ Rocky 호스트의 커널
root@ubuntu-lab:/# hostname
ubuntu-lab
```

- **핵심 관찰** : `/etc/os-release` 는 Ubuntu 인데 `uname -r` 은 **Rocky 커널** → 1-3 표의 "컨테이너는 커널을 공유하고 사용자 공간만 다르다" 가 실증됨
- 그래서 Ubuntu 컨테이너에서 커널 모듈 적재(`modprobe`)나 커널 파라미터 변경(`sysctl -w`)은 원칙적으로 불가/영향 범위가 호스트

> 📝 **시험 포인트**: 배포판 계열 문항(필기 R01 #5, R04 #4, R07 #5, R10 #4) — **Ubuntu·Mint·Kali 는 Debian 계열**(`apt`), RHEL·CentOS·Rocky·Fedora·openSUSE 는 RPM 계열. `VERSION_CODENAME=noble` 이 24.04 의 코드명.

### 6-2. APT 저장소 설정 — deb822 vs 구형 sources.list

> **상황**: `apt update` 를 치기 전에 이 컨테이너가 어느 저장소를 보는지 확인한다. 24.04 부터 형식이 바뀌었으므로 두 형식을 모두 알아야 한다.

```bash
ls -l /etc/apt/sources.list /etc/apt/sources.list.d/ 2>&1
cat /etc/apt/sources.list.d/ubuntu.sources
ls /etc/apt/preferences.d /etc/apt/apt.conf.d | head
cat /etc/apt/apt.conf.d/docker-clean 2>/dev/null | head -5
```

**신형 — deb822 형식** (`/etc/apt/sources.list.d/ubuntu.sources`, 24.04 기본)

```text
Types: deb
URIs: http://ports.ubuntu.com/ubuntu-ports/
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

**구형 — 한 줄 형식** (`/etc/apt/sources.list`, 22.04 이하 · 지금도 유효)

```text
deb http://archive.ubuntu.com/ubuntu noble main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu noble-updates main restricted universe multiverse
deb-src http://archive.ubuntu.com/ubuntu noble main
```

| 항목 | 구형 한 줄 형식 | deb822 (`.sources`) |
| --- | --- | --- |
| 파일 | `/etc/apt/sources.list`, `sources.list.d/*.list` | `sources.list.d/*.sources` |
| 첫 필드 | `deb`(바이너리) / `deb-src`(소스) | `Types:` |
| URL | 두 번째 필드 | `URIs:` |
| 배포판 코드명 | 세 번째 필드 (`noble`, `focal`, `jammy`) | `Suites:` |
| 컴포넌트 | 나머지 필드 | `Components:` |
| 서명 키 | `[signed-by=...]` 옵션 | `Signed-By:` |
| 여러 항목 | 줄마다 반복 | 공백 줄로 블록 구분 |

- 컴포넌트 의미 : `main`(공식·자유) / `restricted`(공식·비자유 드라이버) / `universe`(커뮤니티·자유) / `multiverse`(커뮤니티·비자유)
- aarch64 컨테이너라 URI 가 `ports.ubuntu.com/ubuntu-ports` — amd64 는 `archive.ubuntu.com/ubuntu`
- RedHat 대응 : `/etc/yum.repos.d/*.repo` ↔ `/etc/apt/sources.list*` → [[../THEORY/package-software]] 2-3, 3-3

**검증**

```bash
grep -E '^(Types|URIs|Suites|Components)' /etc/apt/sources.list.d/ubuntu.sources
apt-cache policy | head -8              # 실제 사용 중인 저장소와 우선순위
```

```text
Types: deb
URIs: http://ports.ubuntu.com/ubuntu-ports/
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
# apt-cache policy | head
Package files:
 100 /var/lib/dpkg/status
     release a=now
 500 http://ports.ubuntu.com/ubuntu-ports noble/main arm64 Packages
     release v=24.04,o=Ubuntu,a=noble,c=main,b=arm64
```

> 📝 **시험 포인트**: 구형 형식의 필드 순서 **`deb <URL> <배포판코드명> <컴포넌트…>`** 는 필기에 그대로 나옴. `deb-src` 는 소스 패키지용.

### 6-3. 목록 갱신·조회 — apt update · list · search · show

> **상황**: 설치 전에 반드시 패키지 목록을 갱신해야 한다. RedHat 과 의미가 다른 지점이므로 주의한다.

```bash
apt update                                  # ★ 저장소 목록 갱신 (업그레이드 아님)
apt list --upgradable                       # 업그레이드 가능한 패키지
apt list --installed | head                 # 설치된 패키지
apt list 'htop*'                            # 패턴 조회
apt search htop                             # 이름·설명 검색
apt show htop                               # 상세 정보
apt show htop 2>/dev/null | grep -E '^(Package|Version|Depends|Description|Homepage|Size)'
apt policy htop                             # 설치본/후보 버전
```

- ⚠️ **가장 중요한 함정** : Debian 계열 `apt update` = **패키지 목록 갱신**, RedHat `dnf update` = **패키지 업그레이드**. Debian 에서 업그레이드는 `apt upgrade`
- `apt update` 가 갱신하는 것 : `/var/lib/apt/lists/` 의 인덱스 파일
- `apt list` 옵션 : `--installed`(설치됨), `--upgradable`(업그레이드 가능), `--all-versions`(모든 버전)
- `apt search <키워드>` : 패키지 이름 + 설명에서 검색 (내부적으로 `apt-cache search`)
- `apt show <패키지>` : 상세 정보 — `Depends`(의존), `Recommends`(권장), `Suggests`(제안), `Conflicts`(충돌)
- `apt` vs `apt-get`/`apt-cache`
	- `apt` : 사람이 쓰는 통합 명령 (진행바·색상, 출력 형식이 버전마다 변할 수 있음)
	- `apt-get`/`apt-cache` : **스크립트용** 안정 인터페이스 — 자동화에서는 이쪽 사용 권장

**검증**

```bash
ls /var/lib/apt/lists/ | head -5
apt policy htop
apt-cache policy htop | head -4
```

```text
root@ubuntu-lab:/# apt update
Get:1 http://ports.ubuntu.com/ubuntu-ports noble InRelease [256 kB]
...
Reading package lists... Done
Building dependency tree... Done
All packages are up to date.
root@ubuntu-lab:/# apt policy htop
htop:
  Installed: (none)
  Candidate: 3.3.0-4build1
  Version table:
     3.3.0-4build1 500
        500 http://ports.ubuntu.com/ubuntu-ports noble/universe arm64 Packages
```

> 📝 **시험 포인트**: `apt update`(목록 갱신) ↔ `apt upgrade`(업그레이드) 구분은 필기 최빈출 함정. RedHat 의 `dnf update` 와 의미가 반대라는 점을 반드시 각인.

### 6-4. 설치 — apt install

> **상황**: 개발팀이 요청한 도구 4종을 설치한다. 권장 패키지 처리 방식도 함께 확인한다.

```bash
apt install -y htop curl vim tree
htop --version; curl --version | head -1; vim --version | head -1; tree --version
```

- `-y`, `--yes` : 확인 프롬프트에 자동 승인 (**y**es)
- `--no-install-recommends` : `Recommends` 패키지를 설치하지 않음 → **컨테이너 이미지 슬림화의 정석**
- `--only-upgrade` : 이미 설치된 것만 업그레이드 (신규 설치 안 함) — RPM 의 `-Fvh`(freshen) 대응
- `-d`, `--download-only` : 내려받기만 하고 설치하지 않음
- `--reinstall` : 이미 설치된 패키지를 다시 설치
- `-s`, `--simulate`/`--dry-run` : 실제로 하지 않고 계획만 출력
- `-f`, `--fix-broken` : 깨진 의존성 복구 (6-9)
- `apt install ./local.deb` : 로컬 `.deb` 을 **의존성 해결과 함께** 설치 (경로에 `./` 필수)

```bash
# 권장 패키지 제외 비교
apt install -y --no-install-recommends net-tools
apt install -s tcpdump | head -12                 # 시뮬레이션 — 무엇이 함께 설치되는지
```

**검증**

```bash
dpkg -l htop curl vim tree | tail -5
command -v htop curl vim tree
apt list --installed 2>/dev/null | grep -cE '^(htop|curl|vim|tree)/'
```

```text
root@ubuntu-lab:/# dpkg -l htop curl vim tree | tail -5
ii  curl   8.5.0-2ubuntu10.x   arm64   command line tool for transferring data with URL syntax
ii  htop   3.3.0-4build1       arm64   interactive processes viewer
ii  tree   2.1.1-2ubuntu3      arm64   displays an indented directory tree
ii  vim    2:9.1.0016-1ubuntu7 arm64   Vi IMproved - enhanced vi editor
root@ubuntu-lab:/# command -v htop curl vim tree
/usr/bin/htop
/usr/bin/curl
/usr/bin/vim
/usr/bin/tree
```

- `dpkg -l` 첫 두 글자 `ii` = **정상 설치됨** (6-8 상태 코드표)

> 📝 **시험 포인트**: `apt install` ↔ `dnf install` 대응. `--no-install-recommends` 는 실무형. `apt install`(고수준, 의존성 자동) vs `dpkg -i`(저수준, 의존성 수동)의 차이가 기출 핵심.

### 6-5. apt-cache 계열 질의

> **상황**: 어떤 패키지가 무엇에 의존하고, 무엇이 이 패키지를 필요로 하는지 조사한다. 삭제 영향 범위 판단의 근거다.

```bash
apt-cache search editor | head -5             # 이름·설명 검색
apt-cache show htop | head -20                # 저장소 메타데이터 (미설치도 조회 가능)
apt-cache showpkg htop | head -15             # 버전·의존·역의존 원시 정보
apt-cache policy htop                          # 설치본 / 후보 / 버전 테이블
apt-cache depends htop                         # 이 패키지가 의존하는 것
apt-cache rdepends htop                        # 이 패키지에 의존하는 것 (reverse)
apt-cache rdepends --installed vim | head
apt-cache stats                                # 캐시 통계
apt-cache pkgnames | wc -l                     # 알려진 패키지 총 개수
```

- `apt-cache search` : 인덱스 검색 (`apt search` 의 실체)
- `apt-cache show` : **저장소의** 패키지 정보 (`dpkg -s` 는 **설치된** 패키지 정보 — 구분 필수)
- `apt-cache depends` : 의존 관계 트리. `--recurse` 로 재귀 조회
- `apt-cache rdepends` : 역의존 — 삭제 시 영향을 받는 패키지 파악
- `apt-cache policy` : 어떤 저장소의 어느 버전이 선택되는지(핀 우선순위 포함)
- RedHat 대응 : `dnf search` ↔ `apt-cache search`, `dnf info` ↔ `apt-cache show`, `dnf repoquery --requires` ↔ `apt-cache depends`

**검증**

```bash
apt-cache depends htop | head -6
apt-cache policy htop | head -3
apt-cache stats | head -3
```

```text
root@ubuntu-lab:/# apt-cache depends htop
htop
  Depends: libc6
  Depends: libncursesw6
  Depends: libtinfo6
  Recommends: lsof
  Recommends: strace
root@ubuntu-lab:/# apt-cache policy htop
htop:
  Installed: 3.3.0-4build1
  Candidate: 3.3.0-4build1
```

> 📝 **시험 포인트**: `apt-cache show`(저장소 정보) ↔ `dpkg -s`(설치 정보) 구분. 파일 → 패키지 검색은 Debian 에서 `apt-file search`(별도 설치)이고, 설치된 파일이면 `dpkg -S`.

### 6-6. remove vs purge vs autoremove

> **상황**: 삭제 명령 두 가지의 차이를 **설정 파일이 남는지 여부**로 직접 확인한다. 기출 단골 비교다.

```bash
# ① 설정 파일이 있는 패키지로 실험
apt install -y nano
ls -l /etc/nanorc                          # 설정 파일 존재 확인
dpkg -L nano | grep '^/etc' | head

# ② remove — 프로그램만 제거, 설정 유지
apt remove -y nano
command -v nano || echo "nano 실행 파일 없음"
ls -l /etc/nanorc                          # ★ 설정 파일은 남아 있음
dpkg -l nano | tail -1                     # 상태가 rc

# ③ purge — 설정까지 완전 제거
apt purge -y nano
ls -l /etc/nanorc 2>&1                     # ★ 사라짐
dpkg -l nano | tail -1                     # 목록에서 사라지거나 un

# ④ autoremove — 의존 때문에 자동 설치됐으나 이제 불필요한 패키지 제거
apt autoremove -y
apt autoremove --purge -y                  # 설정까지
```

| 명령 | 실행 파일 | 설정 파일(`/etc`) | `dpkg -l` 상태 |
| --- | --- | --- | --- |
| `apt remove <pkg>` | 삭제 | **유지** | `rc` |
| `apt purge <pkg>` (= `apt remove --purge`) | 삭제 | **삭제** | 목록에서 제거 / `un` |
| `apt autoremove` | 자동 설치된 고아 패키지 삭제 | 유지 | — |
| `apt autoremove --purge` | 고아 패키지 + 설정 삭제 | 삭제 | — |

- `rc` 상태 = **r**emoved 되었으나 **c**onfig 파일이 남음 → 재설치 시 이전 설정이 복원됨
- 남은 `rc` 패키지 일괄 정리 : `dpkg -l | awk '/^rc/ {print $2}' | xargs -r dpkg -P`
- RedHat 대응 : `dnf remove` 는 설정 파일을 `.rpmsave` 로 남기거나 삭제 — **`purge` 개념이 별도로 없음**

**검증**

```bash
dpkg -l | awk '/^rc/ {print $1, $2}'
ls /etc/nanorc 2>&1
apt list --installed 2>/dev/null | grep -c '^nano/'
```

```text
root@ubuntu-lab:/# (apt remove -y nano 후)
root@ubuntu-lab:/# ls -l /etc/nanorc
-rw-r--r-- 1 root root ... /etc/nanorc          ← 남아 있음
root@ubuntu-lab:/# dpkg -l nano | tail -1
rc  nano  8.0-1  arm64  small, friendly text editor inspired by Pico
root@ubuntu-lab:/# (apt purge -y nano 후)
root@ubuntu-lab:/# ls -l /etc/nanorc
ls: cannot access '/etc/nanorc': No such file or directory
```

> 📝 **시험 포인트**: **`remove` = 설정 유지, `purge` = 설정까지 삭제**. `dpkg` 저수준 대응은 `-r`(remove) / `-P`(Purge). 대문자 `-P` 라는 점도 함정.

### 6-7. 캐시 관리 — clean · autoclean

> **상황**: 컨테이너 이미지를 키우는 주범이 다운로드 캐시다. 위치와 정리 명령을 확인한다.

```bash
ls -l /var/cache/apt/archives/ | head
du -sh /var/cache/apt/archives/
apt install -d -y tcpdump                  # 내려받기만
ls -lh /var/cache/apt/archives/*.deb | head -3
du -sh /var/cache/apt

apt autoclean                              # 저장소에서 더 이상 받을 수 없는 오래된 .deb 만 삭제
apt clean                                  # ★ 캐시 .deb 전부 삭제
du -sh /var/cache/apt/archives/
ls /var/lib/apt/lists/ | wc -l             # 인덱스는 별도 (apt update 로 재생성)
rm -rf /var/lib/apt/lists/*                # 이미지 슬림화 시 함께 지우는 대상
```

| 경로 | 내용 | 정리 명령 |
| --- | --- | --- |
| `/var/cache/apt/archives/*.deb` | 내려받은 패키지 파일 | `apt clean` / `apt autoclean` |
| `/var/cache/apt/*.bin` | 패키지 캐시 DB | `apt clean` |
| `/var/lib/apt/lists/` | `apt update` 로 받은 저장소 인덱스 | `rm -rf` (다시 `apt update` 필요) |
| `/var/lib/dpkg/` | **설치 상태 DB — 절대 지우면 안 됨** | — |

- `apt clean` : `archives/` 의 모든 `.deb` 삭제 (**clean**)
- `apt autoclean` : 저장소에 더는 없는 버전의 `.deb` 만 삭제 (보수적)
- Dockerfile 의 정석 : `apt-get update && apt-get install -y --no-install-recommends <pkgs> && rm -rf /var/lib/apt/lists/*` 를 **한 RUN 레이어에** 묶어야 캐시가 이미지에 남지 않음(7-1)
- RedHat 대응 : `dnf clean all`, 캐시 경로 `/var/cache/dnf`

**검증**

```bash
du -sh /var/cache/apt/archives/
ls /var/cache/apt/archives/*.deb 2>&1 | tail -1
ls /var/lib/dpkg/status && wc -l < /var/lib/dpkg/status
```

```text
root@ubuntu-lab:/# du -sh /var/cache/apt/archives/
16K	/var/cache/apt/archives/
root@ubuntu-lab:/# ls /var/cache/apt/archives/*.deb
ls: cannot access '/var/cache/apt/archives/*.deb': No such file or directory
root@ubuntu-lab:/# wc -l < /var/lib/dpkg/status
2417
```

> 📝 **시험 포인트**: `apt clean`(전부) vs `apt autoclean`(오래된 것만). `/var/cache/apt/archives` 경로는 서술형 대비 암기.

### 6-8. dpkg 질의 — -l · -L · -S · -s · -I · -c

> **상황**: `rpm -q*` 계열에 대응하는 `dpkg` 질의 옵션을 전부 실행한다. 기출 대응표의 절반이 여기서 나온다.

```bash
dpkg -l                                    # 설치된 전체 패키지 목록
dpkg -l | head -6                          # 헤더의 상태 코드 범례
dpkg -l htop                               # 특정 패키지 상태
dpkg -l 'vim*'                             # 패턴
dpkg -l | grep -c '^ii'                    # 정상 설치 개수

dpkg -L htop                               # 이 패키지가 설치한 파일 목록
dpkg -L htop | grep -E 'bin|man' | head

dpkg -S /usr/bin/htop                      # 이 파일은 어느 패키지 소속인가
dpkg -S /etc/vim/vimrc
dpkg -S $(command -v curl)

dpkg -s htop                               # 설치된 패키지 상세 상태
dpkg -s htop | grep -E '^(Package|Status|Version|Depends|Installed-Size)'
```

**`dpkg -l` 상태 코드 (앞 2~3글자)** — 최빈출

| 표기 | 의미 |
| --- | --- |
| 1번째 글자 = **희망 상태**(desired) | `u` unknown / `i` install / `r` remove / `p` purge / `h` hold |
| 2번째 글자 = **현재 상태**(status) | `n` not-installed / `i` installed / `c` config-files / `U` unpacked / `F` half-configured / `H` half-installed / `W` trigger-await / `t` trigger-pending |
| 3번째 글자 = **오류 플래그** | (공백) 정상 / `R` reinstall-required |

| 조합 | 상태 해석 |
| --- | --- |
| `ii` | **정상 설치 완료** (가장 흔함) |
| `rc` | 프로그램은 지웠고 **설정 파일만 남음** (`apt remove` 결과) |
| `un` | 설치된 적 없음 / 알 수 없음 |
| `iU` | 설치 희망이나 **압축만 풀린 상태**(미설정) → `dpkg --configure -a` 필요 |
| `iF` | 설정 도중 실패 → `dpkg --configure -a` 또는 `apt -f install` |
| `hi` | **버전 고정(hold)** 된 설치 패키지 |
| `pn` | purge 희망 + 미설치 |

**`.deb` 파일 질의** (설치 전 파일에 대한 질의 — `rpm -qp` 대응)

```bash
cd /tmp && apt download htop               # .deb 파일만 내려받기
ls -lh htop_*.deb
dpkg -I htop_*.deb                         # 패키지 파일의 제어 정보 (대문자 i = info)
dpkg -I htop_*.deb | grep -E 'Package|Version|Depends|Architecture|Installed-Size'
dpkg -c htop_*.deb                         # 패키지 파일에 든 파일 목록 (contents)
dpkg -c htop_*.deb | head -5
dpkg-deb -f htop_*.deb Package Version     # 특정 필드만 추출
dpkg-deb -x htop_*.deb /tmp/htop-extract && ls /tmp/htop-extract
```

- `dpkg -I <파일>.deb` : **미설치 .deb 파일**의 제어 정보 (**I**nfo) ↔ `rpm -qip`
- `dpkg -c <파일>.deb` : **미설치 .deb 파일**의 내용 목록 (**c**ontents) ↔ `rpm -qlp`
- `dpkg -L <패키지>` : **설치된** 패키지의 파일 목록 ↔ `rpm -ql`
- `dpkg -S <경로>` : 파일 → 패키지 역추적 ↔ `rpm -qf` — **설치된 파일만** 대상
- `dpkg -s <패키지>` : 설치 상태·상세 ↔ `rpm -qi`
- `dpkg-deb` : `.deb` 파일 자체를 다루는 저수준 도구 (`-f` 필드 추출, `-x` 추출, `-b` 빌드)
- `apt download <패키지>` : `.deb` 만 내려받기 — root 로 실행 시 `_apt` 사용자 접근 경고가 나올 수 있으나 다운로드는 성공

**검증**

```bash
dpkg -l htop | tail -1
dpkg -L htop | grep -c '^/'
dpkg -S /usr/bin/htop
dpkg -s htop | grep -E '^(Status|Version)'
dpkg -I /tmp/htop_*.deb | grep -E ' (Package|Version):'
```

```text
root@ubuntu-lab:/# dpkg -l htop | tail -1
ii  htop  3.3.0-4build1  arm64  interactive processes viewer
root@ubuntu-lab:/# dpkg -S /usr/bin/htop
htop: /usr/bin/htop
root@ubuntu-lab:/# dpkg -s htop | grep -E '^(Status|Version)'
Status: install ok installed
Version: 3.3.0-4build1
root@ubuntu-lab:/# dpkg -c /tmp/htop_*.deb | head -3
drwxr-xr-x root/root  0 ... ./
drwxr-xr-x root/root  0 ... ./usr/
drwxr-xr-x root/root  0 ... ./usr/bin/
```

> 📝 **시험 포인트**: `dpkg -l`(목록) `-L`(파일 목록) `-S`(파일 소속) `-s`(상태) 네 개는 대소문자까지 정확히. `-l` 은 소문자 L, `-L` 은 대문자 — 바꿔 쓰면 오답.

### 6-9. .deb 직접 설치와 의존성 오류 복구

> **상황**: 인터넷이 안 되는 서버에 `.deb` 을 들고 가서 설치하는 상황을 재현한다. `dpkg` 가 의존성을 해결하지 못하는 것을 직접 본다.

```bash
cd /tmp
apt download htop                          # 또는 이미 받아 둔 파일 사용
apt purge -y htop                          # 일단 지우고
dpkg -i htop_*.deb                         # 저수준 직접 설치
dpkg -l htop | tail -1
```

- `dpkg -i <파일>.deb` : `.deb` 설치 (**i**nstall) ↔ `rpm -ivh`
	- 의존성이 없으면 성공, **의존 패키지가 빠져 있으면 `dependency problems` 오류**로 `iU`/`iF` 상태에 머무름
	- `--force-depends` : 의존성 무시 강제 (비권장) ↔ `rpm --nodeps`
	- `-R`/`--recursive` : 디렉터리 안의 `.deb` 재귀 설치
	- `-E` : 같은 버전이면 건너뜀, `-G` : 더 새 버전이 설치돼 있으면 건너뜀

**의존성 오류 재현과 복구**

```bash
# 의존 라이브러리를 일부러 지운 뒤 설치 시도 (의존성 문제 재현)
apt purge -y htop
dpkg -r libncursesw6 2>&1 | tail -3        # 다른 패키지가 의존 → 거부되는 것이 정상
dpkg -i htop_*.deb                          # 정상 환경이면 성공

# 실제로 의존성 오류가 났을 때의 표준 복구 절차
apt -f install                              # = apt --fix-broken install : 빠진 의존성 자동 설치
apt-get -f install                          # 스크립트용 동일 명령
dpkg --configure -a                         # 압축만 풀린(iU) 패키지 설정 마무리
dpkg --audit                                # 문제 있는 패키지 진단
```

- **의존성 해결의 정석 순서** : `dpkg -i` → 오류 → `apt -f install` → `dpkg -l` 로 `ii` 확인
- 처음부터 의존성까지 해결하며 로컬 파일을 설치하려면 : `apt install ./htop_3.3.0-4build1_arm64.deb` (경로에 `./` 필수 — 없으면 저장소 패키지 이름으로 해석)
- ⚠️ `--force-*` 계열은 시스템 일관성을 깨뜨림 — 시험 답안으로도 `apt -f install` 이 정답

**검증**

```bash
dpkg -l htop | tail -1
dpkg --audit; echo "audit rc=$?"
apt-get check                               # 의존성 무결성 검사
command -v htop && htop --version | head -1
```

```text
root@ubuntu-lab:/tmp# dpkg -i htop_*.deb
Selecting previously unselected package htop.
Unpacking htop (3.3.0-4build1) ...
Setting up htop (3.3.0-4build1) ...
root@ubuntu-lab:/tmp# dpkg -l htop | tail -1
ii  htop  3.3.0-4build1  arm64  interactive processes viewer
root@ubuntu-lab:/tmp# apt-get check
Reading package lists... Done
Building dependency tree... Done
# (의존성 오류가 났을 때의 전형적 메시지)
dpkg: dependency problems prevent configuration of htop:
 htop depends on libncursesw6 (>= 6); however:
  Package libncursesw6 is not installed.
```

> 📝 **시험 포인트**: "데비안 계열에서 내려받은 `.deb` 을 직접 설치" = **`dpkg -i pkg.deb`** (필기 R01 #44, R02 #41, R07 #35 — 3회 반복 출제). 오답 선지는 `apt-cache install`, `dpkg -r`, `apt-get update pkg.deb`.

### 6-10. dpkg 삭제·설정 — -r · -P · --configure · dpkg-reconfigure

> **상황**: 저수준 삭제 명령과 설정 재실행을 다룬다. 비대화형 환경(컨테이너·자동화)에서의 주의점도 확인한다.

```bash
apt install -y tree
dpkg -r tree                               # 삭제 (설정 유지)
dpkg -l tree | tail -1                     # rc
ls /usr/bin/tree 2>&1

dpkg -P tree                               # 완전 삭제 (설정 포함, 대문자 P)
dpkg -l tree 2>&1 | tail -1

dpkg --configure -a                        # 미설정(iU/iF) 패키지 일괄 설정
dpkg --audit                               # 문제 패키지 진단
dpkg --get-selections | head -5            # 패키지 선택 상태 덤프
dpkg --get-selections | grep -c install
```

- `dpkg -r <패키지>` : 제거, 설정 파일 유지 (**r**emove) ↔ `rpm -e`
- `dpkg -P <패키지>` : 완전 제거 (**P**urge) — 소문자 `-p` 는 다른 의미(`--print-avail`)이므로 **대문자 필수**
- `dpkg --configure -a` : 압축만 풀린 패키지 전부 설정 (**a**ll) — 설치 중단 후 복구의 표준 절차
- `dpkg --audit` : 반쯤 설치된 패키지 목록 출력
- `dpkg --get-selections` / `--set-selections` : 패키지 선택 상태 내보내기·복원 (서버 이관 시 사용)

**dpkg-reconfigure 와 비대화형 처리**

```bash
apt install -y tzdata                                   # 설치 중 지역 선택 대화가 뜰 수 있음
DEBIAN_FRONTEND=noninteractive apt install -y tzdata    # 대화 없이 기본값으로
dpkg-reconfigure --help | head -5
# dpkg-reconfigure tzdata                               # ※ 대화형 — 컨테이너에서는 프론트엔드 주의
DEBIAN_FRONTEND=noninteractive dpkg-reconfigure -f noninteractive tzdata
cat /etc/timezone
```

- `dpkg-reconfigure <패키지>` : 이미 설치된 패키지의 **설정 질문을 다시 실행** (debconf 기반) — RedHat 에는 대응 개념이 없음
- `DEBIAN_FRONTEND` : debconf 프론트엔드 선택 환경변수
	- `noninteractive` : **질문 없이 기본값 사용** — Dockerfile·자동화 필수
	- `dialog`(기본) / `readline` / `teletype`
	- ⚠️ Dockerfile 에서는 `ENV DEBIAN_FRONTEND=noninteractive` 를 **영구 설정하지 말고** `RUN` 앞에 일회성으로 붙이는 것이 권장(이미지 사용자에게 영향)
- debconf 설정값 조회 : `debconf-show tzdata`

**검증**

```bash
dpkg -l tree 2>&1 | tail -1
dpkg --audit; echo "rc=$?"
cat /etc/timezone
dpkg --get-selections | awk '$2=="install"' | wc -l
```

```text
root@ubuntu-lab:/# dpkg -r tree
Removing tree (2.1.1-2ubuntu3) ...
root@ubuntu-lab:/# dpkg -l tree | tail -1
rc  tree  2.1.1-2ubuntu3  arm64  displays an indented directory tree
root@ubuntu-lab:/# dpkg -P tree
Purging configuration files for tree (2.1.1-2ubuntu3) ...
root@ubuntu-lab:/# dpkg -l tree 2>&1 | tail -1
dpkg-query: no packages found matching tree
root@ubuntu-lab:/# cat /etc/timezone
Etc/UTC
```

> 📝 **시험 포인트**: `dpkg -r`(설정 유지) ↔ `dpkg -P`(설정 포함 완전 삭제)는 `apt remove`/`apt purge` 와 1:1 대응. `rpm -e` 에 대응하는 것은 `dpkg -r`.

### 6-11. 버전 고정·질의 포맷·대안 관리

> **상황**: 특정 패키지를 업그레이드에서 제외하고, 스크립트용 출력 포맷을 만들고, 기본 편집기를 바꾼다.

```bash
# ① 버전 고정 (hold)
apt-mark hold htop
apt-mark showhold
dpkg -l htop | tail -1                      # 상태가 hi 로 변경
apt install -y htop                          # 고정된 패키지는 갱신되지 않음
apt-mark unhold htop
apt-mark showhold
apt-mark showmanual | head -5                # 사용자가 직접 설치한 패키지
apt-mark showauto | wc -l                    # 의존성으로 자동 설치된 패키지
apt-mark auto tree 2>/dev/null; apt-mark manual curl

# ② 질의 포맷
dpkg-query -W -f='${Package} ${Version}\n' | head -5
dpkg-query -W -f='${binary:Package} ${Status} ${Installed-Size}\n' htop
dpkg-query -l 'lib*' | wc -l
dpkg-query -S /usr/bin/vim.basic
grep -A3 '^Package: htop' /var/lib/dpkg/status

# ③ 대안(alternatives) 관리
update-alternatives --display editor
update-alternatives --list editor
update-alternatives --query editor | head -8
# update-alternatives --config editor        # ※ 대화형 선택
update-alternatives --set editor /usr/bin/vim.basic
readlink -f /usr/bin/editor
```

- `apt-mark hold <패키지>` : 업그레이드 **보류** → `dpkg -l` 상태 `hi`. 해제는 `unhold`
	- `showhold` : 보류 목록, `showmanual` : 수동 설치 목록, `showauto` : 자동 설치 목록
	- `auto`/`manual` : 설치 사유 표시 변경 (`autoremove` 대상 여부를 좌우)
	- RedHat 대응 : `dnf versionlock`(플러그인) 또는 `.repo` 의 `exclude=`
- `dpkg-query -W -f='<포맷>'` : 원하는 필드만 출력 (**W** = show, **f** = format) — `rpm -qa --qf` 대응
	- 사용 가능 변수 : `${Package}` `${Version}` `${Architecture}` `${Status}` `${Installed-Size}` `${binary:Package}` `${db:Status-Abbrev}`
- `/var/lib/dpkg/status` : **설치 상태 DB 원본 텍스트 파일** (RPM 의 `/var/lib/rpm` 에 대응). 절대 수동 편집 금지
- `update-alternatives` : 같은 기능의 여러 프로그램 중 기본값 선택 (`editor`, `java`, `awk`, `pager`)
	- `--display <그룹>` : 현재 링크·후보 표시
	- `--config <그룹>` : **대화형 선택**
	- `--set <그룹> <경로>` : 비대화형 지정
	- `--install <링크> <그룹> <경로> <우선순위>` : 후보 등록
	- 실제 구조 : `/usr/bin/editor` → `/etc/alternatives/editor` → 실제 바이너리 (심볼릭 링크 2단)

**검증**

```bash
apt-mark showhold; echo "---"
dpkg-query -W -f='${Package} ${Version}\n' htop curl vim
ls -l /usr/bin/editor /etc/alternatives/editor
update-alternatives --display editor | head -3
```

```text
root@ubuntu-lab:/# apt-mark hold htop
htop set on hold.
root@ubuntu-lab:/# apt-mark showhold
htop
root@ubuntu-lab:/# dpkg -l htop | tail -1
hi  htop  3.3.0-4build1  arm64  interactive processes viewer
root@ubuntu-lab:/# dpkg-query -W -f='${Package} ${Version}\n' htop curl vim
htop 3.3.0-4build1
curl 8.5.0-2ubuntu10.x
vim 2:9.1.0016-1ubuntu7
root@ubuntu-lab:/# ls -l /usr/bin/editor
lrwxrwxrwx 1 root root 24 ... /usr/bin/editor -> /etc/alternatives/editor
root@ubuntu-lab:/# readlink -f /usr/bin/editor
/usr/bin/vim.basic
```

> 📝 **시험 포인트**: `dpkg -l` 의 `hi` 상태 = hold. `update-alternatives --config <이름>` 은 "기본 편집기/자바 버전 변경" 문항의 정답 명령.

### 6-12. 로그·저장소 추가·형식 변환 (참고)

> **상황**: 누가 언제 무엇을 설치했는지 추적하고, 외부 저장소를 추가하는 방법을 확인한다.

```bash
ls -l /var/log/apt/
cat /var/log/apt/history.log | tail -20             # 설치·삭제 이력 (사람이 읽는 형식)
grep -E '^(Start-Date|Commandline|Install|Remove|Purge|End-Date)' /var/log/apt/history.log | tail -12
tail -5 /var/log/apt/term.log                       # 실제 터미널 출력 원문
zcat /var/log/apt/history.log.*.gz 2>/dev/null | head -5   # 회전된 이전 이력
tail -5 /var/log/dpkg.log                           # dpkg 수준의 상세 이력
```

| 파일 | 내용 | RedHat 대응 |
| --- | --- | --- |
| `/var/log/apt/history.log` | apt 명령 단위 이력(시각·명령행·설치/삭제 목록) | `dnf history` (`/var/log/dnf.log`) |
| `/var/log/apt/term.log` | 설치 중 터미널 원문 출력 | `/var/log/dnf.rpm.log` |
| `/var/log/dpkg.log` | dpkg 단계별(unpack/configure) 로그 | `/var/log/dnf.librepo.log` 등 |

- ⚠️ Debian 계열에는 `dnf history undo` 같은 **롤백 명령이 없음** — 이력은 텍스트 조회만 가능

**저장소 추가 (참고)**

```bash
apt install -y software-properties-common          # add-apt-repository 제공
add-apt-repository --help | head -8
# add-apt-repository universe                       # 컴포넌트 활성
# add-apt-repository ppa:<사용자>/<PPA이름>          # ※ Ubuntu PPA — 이 실습에서는 미실행
# add-apt-repository -y 'deb http://example.com/repo noble main'
ls /etc/apt/sources.list.d/
```

- `add-apt-repository` : 저장소 항목과 서명 키를 자동 추가 (`software-properties-common` 패키지)
	- `-y` : 확인 생략, `-r` : 제거, `-n` : 추가 후 `apt update` 생략
- 서명 키는 `/etc/apt/keyrings/*.gpg` 에 두고 `Signed-By:` 로 지정하는 것이 현재 권장 방식 (`apt-key` 는 **폐지**)

**형식 변환 (참고)**

```bash
# apt install -y alien           # ※ 선택 — rpm ↔ deb 상호 변환 도구
# alien -d package.rpm           # rpm → deb (-d = to deb)
# alien -r package.deb           # deb → rpm (-r = to rpm)
```

- `alien` : 패키지 형식 변환. **의존성·스크립트가 완전히 옮겨지지 않으므로 임시방편** — 운영 사용 비권장

**검증**

```bash
grep -c 'Start-Date' /var/log/apt/history.log
grep -A2 'Commandline: apt install -y htop curl vim tree' /var/log/apt/history.log | head -4
ls /etc/apt/sources.list.d/
```

```text
root@ubuntu-lab:/# grep -E '^(Start-Date|Commandline|Install)' /var/log/apt/history.log | tail -6
Start-Date: 2026-09-03  14:22:10
Commandline: apt install -y htop curl vim tree
Install: htop:arm64 (3.3.0-4build1), tree:arm64 (2.1.1-2ubuntu3), ...
Start-Date: 2026-09-03  14:31:02
Commandline: apt purge -y nano
```

> 📝 **시험 포인트**: `dnf history` ↔ `/var/log/apt/history.log` 대응. Debian 에는 트랜잭션 롤백이 없다는 점이 비교 서술 포인트.

### 6-13. RPM ↔ DEB 명령 대응표 (기출 핵심)

> **상황**: 두 계열을 나란히 묻는 문항이 반복 출제된다. 지금까지 실행한 명령을 한 표로 압축한다.

| 기능 | RedHat 계열 (`rpm`/`dnf`) | Debian 계열 (`dpkg`/`apt`) |
| --- | --- | --- |
| 패키지 형식 | `.rpm` | `.deb` |
| 저수준 도구 | `rpm` | `dpkg` |
| 고수준 도구 | `dnf` / `yum` | `apt` / `apt-get` / `apt-cache` |
| 저장소 설정 | `/etc/yum.repos.d/*.repo` | `/etc/apt/sources.list`, `sources.list.d/*.{list,sources}` |
| 상태 DB | `/var/lib/rpm` | `/var/lib/dpkg/status` |
| 캐시 | `/var/cache/dnf` | `/var/cache/apt/archives` |
| **목록 갱신** | (자동, `dnf makecache`) | **`apt update`** ← 의미 주의 |
| **업그레이드** | `dnf update` / `dnf upgrade` | **`apt upgrade`** / `apt full-upgrade` |
| 저장소에서 설치 | `dnf install <pkg>` | `apt install <pkg>` |
| 삭제 | `dnf remove <pkg>` | `apt remove <pkg>` |
| 완전 삭제(설정 포함) | (대응 없음 — `.rpmsave` 잔존) | `apt purge <pkg>` |
| 고아 패키지 정리 | `dnf autoremove` | `apt autoremove` |
| 검색 | `dnf search <키워드>` | `apt search` / `apt-cache search` |
| 상세 정보(저장소) | `dnf info <pkg>` | `apt show` / `apt-cache show` |
| 파일 제공 패키지 검색 | `dnf provides <파일>` | `apt-file search <파일>` (별도 설치) |
| 설치 목록 | `dnf list installed` / `rpm -qa` | `apt list --installed` / `dpkg -l` |
| **로컬 파일 설치** | **`rpm -ivh <파일>.rpm`** | **`dpkg -i <파일>.deb`** |
| 업그레이드 설치 | `rpm -Uvh` | `dpkg -i` (동일 명령) |
| 갱신만 | `rpm -Fvh` (freshen) | `apt install --only-upgrade` |
| 삭제(저수준) | `rpm -e <pkg>` | `dpkg -r <pkg>` |
| 완전 삭제(저수준) | (없음) | `dpkg -P <pkg>` |
| 설치 목록 질의 | `rpm -qa` | `dpkg -l` |
| 패키지 파일 목록 | `rpm -ql <pkg>` | `dpkg -L <pkg>` |
| 파일 소속 패키지 | `rpm -qf <경로>` | `dpkg -S <경로>` |
| 패키지 상세 | `rpm -qi <pkg>` | `dpkg -s <pkg>` |
| 미설치 파일 정보 | `rpm -qip <파일>.rpm` | `dpkg -I <파일>.deb` |
| 미설치 파일 목록 | `rpm -qlp <파일>.rpm` | `dpkg -c <파일>.deb` |
| 무결성 검증 | `rpm -V <pkg>` | `debsums <pkg>` (별도 설치) |
| 의존성 무시 강제 | `rpm --nodeps` | `dpkg --force-depends` |
| 깨진 의존성 복구 | `dnf distro-sync` / `dnf reinstall` | **`apt -f install`** |
| 버전 고정 | `dnf versionlock` (플러그인) | `apt-mark hold` |
| 이력 | `dnf history` (롤백 가능) | `/var/log/apt/history.log` (조회만) |
| 캐시 정리 | `dnf clean all` | `apt clean` / `apt autoclean` |
| 그룹 설치 | `dnf group install "<그룹>"` | `apt install <메타패키지>` (`^task` 패턴, `tasksel`) |
| 형식 변환 | `alien -r <파일>.deb` | `alien -d <파일>.rpm` |

- 핵심 3쌍만 외워도 대부분 커버 : `rpm -ivh`↔`dpkg -i`, `rpm -qa`↔`dpkg -l`, `rpm -qf`↔`dpkg -S`
- ⚠️ **`apt update` ≠ 업그레이드** — RedHat 의 `dnf update` 와 의미가 정반대

**검증**

```bash
# 컨테이너 안 (Ubuntu / dpkg)
dpkg -S "$(command -v bash)"
dpkg -L bash | head -3
dpkg -l | grep -c '^ii'

# 호스트 (Rocky / rpm) — exit 후 또는 별도 터미널에서 같은 질의를 비교
# rpm -qf /usr/bin/bash
# rpm -ql bash | head -3
# rpm -qa | wc -l
```

```text
root@ubuntu-lab:/# dpkg -S "$(command -v bash)"
bash: /bin/bash
root@ubuntu-lab:/# dpkg -L bash | head -3
/.
/bin
/bin/bash
root@ubuntu-lab:/# dpkg -l | grep -c '^ii'
...
# (호스트) rpm -qf /usr/bin/bash
bash-5.1.8-....el9.aarch64
```

- 같은 질문("이 파일은 어느 패키지 소속인가")에 대해 `rpm -qf` ↔ `dpkg -S` 가 1:1 대응함을 두 화면으로 확인

> 📝 **시험 포인트**: 이 표 자체가 기출 범위. 특히 `.deb` 직접 설치(`dpkg -i`)는 10회분 중 3회 출제. `dnf`/`apt` 는 의존성 자동 해결, `rpm`/`dpkg` 는 수동.

### 6-14. 재진입과 이미지 커밋 — start -ai · commit

> **상황**: 셸에서 나가면 컨테이너가 멈춘다. 다시 들어가는 방법과, 지금까지 설치한 도구를 이미지로 굳히는 방법을 확인한다.

```bash
# 컨테이너 안에서
exit
```

**호스트에서**

```bash
docker ps -a --filter name=lab-ubuntu --format '{{.Names}} {{.Status}}'
docker start -ai lab-ubuntu               # 재진입 (attach + interactive)
# (다시 exit 로 나옴)

docker start lab-ubuntu                    # 백그라운드로만 시작 → bash 가 즉시 종료됨에 주의
docker ps -a --filter name=lab-ubuntu --format '{{.Names}} {{.Status}}'
docker start -ai lab-ubuntu                # 대화형 재진입이 정석
```

- `docker start -a -i <컨테이너>` : `-a`(**a**ttach 출력 연결) + `-i`(**i**nteractive 입력 연결) → 원래 `bash` 세션으로 복귀
- 메인 프로세스가 `bash` 인 컨테이너는 **셸이 끝나면 컨테이너도 종료** — 정상 동작
- 실행 중인 상태에서 추가 세션이 필요하면 `docker exec -it lab-ubuntu bash`

**이미지로 커밋**

```bash
docker commit -a 'lab' -m 'add htop curl vim tree' lab-ubuntu lab/ubuntu-tools:1.0
docker images | grep lab/ubuntu-tools
docker image history lab/ubuntu-tools:1.0 | head -3
docker image inspect lab/ubuntu-tools:1.0 --format '{{.Config.Cmd}} {{.Author}} {{.Comment}}'

# 새 컨테이너에서 도구가 그대로 있는지 검증
docker run --rm lab/ubuntu-tools:1.0 htop --version
docker run --rm lab/ubuntu-tools:1.0 sh -c 'command -v htop curl vim tree'
docker run --rm lab/ubuntu-tools:1.0 dpkg -l | grep -cE '^ii'
```

- `docker commit [옵션] <컨테이너> <이미지[:태그]>` : **컨테이너의 현재 상태를 새 이미지로 저장**
	- `-a` : 작성자 (**a**uthor)
	- `-m` : 커밋 메시지 (**m**essage)
	- `-c` : Dockerfile 지시자 적용 (예 `-c 'CMD ["bash"]'`, `-c 'ENV TZ=Asia/Seoul'`)
	- `-p` : 커밋 중 컨테이너 일시 정지 (기본 true, **p**ause)
- ⚠️ `commit` 은 **재현 불가능한 이미지**를 만듦 — 무엇이 들어갔는지 추적이 안 되므로 운영에서는 **Dockerfile 빌드가 정답**(7절). `commit` 은 긴급 스냅샷·장애 조사용
- 볼륨·바인드 마운트 영역은 커밋에 **포함되지 않음**

**검증**

```bash
docker images --format '{{.Repository}}:{{.Tag}} {{.Size}}' | grep ubuntu
docker run --rm lab/ubuntu-tools:1.0 sh -c 'htop --version; tree --version' | head -2
docker run --rm ubuntu:24.04 sh -c 'command -v htop || echo "원본 이미지에는 htop 없음"'
docker ps -a --format '{{.Names}} {{.Status}}' | grep lab-ubuntu
```

```text
# docker images --format ... | grep ubuntu
lab/ubuntu-tools:1.0 ...MB
ubuntu:24.04 97.1MB
# docker run --rm lab/ubuntu-tools:1.0 sh -c 'htop --version; tree --version'
htop 3.3.0
tree v2.1.1 © 1996 - 2024 by Steve Baker...
# docker run --rm ubuntu:24.04 sh -c 'command -v htop || echo "..."'
원본 이미지에는 htop 없음
# docker ps -a --format '{{.Names}} {{.Status}}' | grep lab-ubuntu
lab-ubuntu Exited (0) 1 minute ago
```

- `lab-ubuntu` 컨테이너는 **삭제하지 않고 남김** — [[12-backup-recovery-review]] 에서 컨테이너·이미지 백업 대상으로 사용

> 📝 **시험 포인트**: `docker commit` = 컨테이너 → 이미지. Dockerfile 빌드(`docker build`)와의 차이(재현성·추적성)를 묻는 서술형 가능. 이미지 생성 방법은 **① Dockerfile + build ② 컨테이너 + commit ③ import** 세 가지.

---

## 7. Dockerfile · Compose · 자원 제한

### 7-1. Dockerfile 작성

> **상황**: `docker commit` 으로 만든 이미지는 재현이 안 된다. 인트라넷 웹의 정적 페이지를 담은 이미지를 **Dockerfile 로** 만든다. 빌드 컨텍스트는 Part 04 의 프로젝트 디렉터리 아래에 둔다.

```bash
mkdir -p /srv/devteam/proj/docker
cd /srv/devteam/proj/docker

cat > index.html <<'EOF'
<!doctype html>
<html lang="ko"><head><meta charset="utf-8"><title>Lab Intranet</title></head>
<body><h1>srv01.lab.local — intranet (container)</h1><p>LAB Part 11</p></body></html>
EOF

cat > Dockerfile <<'EOF'
# syntax=docker/dockerfile:1
FROM httpd:2.4-alpine

ARG APP_VER=1.0
LABEL maintainer="lab@lab.local" \
      description="LAB intranet static site" \
      version="${APP_VER}"

ENV TZ=Asia/Seoul \
    APP_HOME=/usr/local/apache2

RUN apk add --no-cache curl tzdata \
 && cp /usr/share/zoneinfo/${TZ} /etc/localtime \
 && echo "${TZ}" > /etc/timezone

WORKDIR ${APP_HOME}
COPY index.html ${APP_HOME}/htdocs/

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -fsS http://127.0.0.1/ >/dev/null || exit 1

# USER daemon        # ← 80 번(1024 미만) 바인딩에는 root 필요 → 주석 처리. 상위 포트라면 활성 권장
CMD ["httpd-foreground"]
EOF

cat -n Dockerfile
```

**Dockerfile 지시자 총람**

| 지시자 | 역할 | 시점 |
| --- | --- | --- |
| `FROM <이미지>[:태그]` | 베이스 이미지 지정 — **반드시 첫 지시자**(`ARG` 만 앞설 수 있음) | 빌드 |
| `ARG <이름>[=기본값]` | **빌드 시점** 변수 (`--build-arg` 로 주입, 최종 이미지에 남지 않음) | 빌드 |
| `ENV <키>=<값>` | **실행 시점**까지 유지되는 환경변수 | 빌드+실행 |
| `LABEL <키>=<값>` | 메타데이터 (`docker inspect` 로 조회, `--filter label=` 검색) | 빌드 |
| `RUN <명령>` | 빌드 중 명령 실행 → **레이어 생성** | 빌드 |
| `COPY <src> <dst>` | 빌드 컨텍스트의 파일 복사 (권장) | 빌드 |
| `ADD <src> <dst>` | COPY + **URL 다운로드 + tar 자동 해제** (예측 불가 → 특별한 경우만) | 빌드 |
| `WORKDIR <경로>` | 이후 지시자·실행의 작업 디렉터리 (없으면 생성) | 빌드+실행 |
| `EXPOSE <포트>` | **문서화 목적** 포트 선언 — 실제 개방 아님 | 메타데이터 |
| `USER <사용자>` | 이후 명령·실행의 사용자 | 빌드+실행 |
| `VOLUME <경로>` | 익명 볼륨 마운트 지점 선언 | 실행 |
| `HEALTHCHECK CMD <명령>` | 상태 점검 — `docker ps` STATUS 에 `(healthy)` 표시 | 실행 |
| `ENTRYPOINT` | **고정 실행 파일** | 실행 |
| `CMD` | 기본 실행 명령/인자 (`docker run` 뒤 인자로 **덮어쓰기 가능**) | 실행 |
| `ONBUILD` | 이 이미지를 베이스로 쓸 때 실행될 지시자 | 파생 빌드 |
| `STOPSIGNAL` | 정지 시 보낼 시그널 (기본 SIGTERM) | 실행 |
| `SHELL` | `RUN`/`CMD` 의 기본 셸 변경 | 빌드 |

- ⚠️ **`EXPOSE 80` 만으로는 호스트 포트가 열리지 않음** — 실제 게시는 `docker run -p` 또는 `-P`
- `RUN` 한 줄 = 레이어 한 개 → `&&` 로 묶고 캐시 정리까지 한 레이어에서 끝내는 것이 이미지 크기 절감의 정석
- 이 이미지는 alpine 기반이라 패키지 관리자가 `apk` (Debian 이면 `apt-get`, RHEL 이면 `dnf`)

**검증**

```bash
ls -l /srv/devteam/proj/docker/
grep -cE '^(FROM|RUN|COPY|CMD|EXPOSE|LABEL|ENV|ARG|WORKDIR|HEALTHCHECK)' Dockerfile
```

```text
# ls -l /srv/devteam/proj/docker/
-rw-r--r-- 1 root root  ... Dockerfile
-rw-r--r-- 1 root root  ... index.html
```

> 📝 **시험 포인트**: `RUN` 은 **빌드 시점**에 실행되어 레이어를 만들고, `CMD` 는 **컨테이너 시작 시** 기본 실행 명령(필기 R10 #89 ①). `EXPOSE` 만으로 호스트 포트가 열린다는 선지는 오답(같은 문항 ③).

### 7-2. .dockerignore 와 빌드

> **상황**: 빌드 컨텍스트에 불필요한 파일이 섞이면 전송이 느려지고 비밀 파일이 이미지에 들어갈 수 있다. 제외 목록을 만들고 빌드한다.

```bash
cd /srv/devteam/proj/docker
cat > .dockerignore <<'EOF'
.git
*.log
*.tar
*.tar.gz
secrets/
.env
node_modules
EOF

docker build -t lab/intranet:1.0 .
```

- `docker build [옵션] <컨텍스트경로>` : 마지막 인자 `.` 은 **빌드 컨텍스트**(데몬에 전송할 디렉터리)
	- `-t <이름:태그>` : 태그 지정 (**t**ag). 여러 번 지정 가능
	- `-f <경로>` : Dockerfile 경로 지정 (**f**ile) — 컨텍스트 밖의 Dockerfile 사용 시
	- `--build-arg <키>=<값>` : `ARG` 값 주입
	- `--no-cache` : 레이어 캐시 무시하고 전부 재실행
	- `--pull` : 베이스 이미지를 항상 최신으로 다시 받음
	- `--target <스테이지>` : 멀티스테이지 빌드에서 특정 단계까지만
	- `--progress=plain` : 축약 없이 전체 빌드 로그
	- `-q` : 이미지 ID 만 출력
- **빌드 컨텍스트** : `docker build .` 의 `.` 디렉터리 전체가 데몬으로 전송됨 → 홈이나 `/` 를 컨텍스트로 지정하면 재앙. `.dockerignore` 로 제외
- `COPY` 는 **컨텍스트 안의 경로만** 참조 가능 (`COPY ../file` 불가)

```bash
docker build -f Dockerfile -t lab/intranet:1.1 --build-arg APP_VER=1.1 .
docker images | grep lab/intranet
docker image inspect lab/intranet:1.1 --format '{{index .Config.Labels "version"}}'
```

**검증**

```bash
docker images --format '{{.Repository}}:{{.Tag}} {{.Size}}' | grep intranet
docker image inspect lab/intranet:1.0 --format '{{.Config.Env}} {{.Config.WorkingDir}} {{.Config.ExposedPorts}}'
docker image history lab/intranet:1.0 | head -8
```

```text
# docker build -t lab/intranet:1.0 .
[+] Building 12.3s (10/10) FINISHED
 => [internal] load build definition from Dockerfile
 => [internal] load .dockerignore
 => [1/3] FROM docker.io/library/httpd:2.4-alpine
 => [2/3] RUN apk add --no-cache curl tzdata && ...
 => [3/3] COPY index.html /usr/local/apache2/htdocs/
 => exporting to image
 => => naming to docker.io/lab/intranet:1.0
# docker image inspect lab/intranet:1.0 --format '{{.Config.WorkingDir}} {{.Config.ExposedPorts}}'
/usr/local/apache2 map[80/tcp:{}]
```

> 📝 **시험 포인트**: 흐름 문제 — **Dockerfile 작성 → `docker build -t app:1.0 .` → `docker images` 확인 → `docker run`**(필기 R07 #89 ③). `-t` 는 이름:태그 지정.

### 7-3. 레이어 캐시 관찰과 실행 검증

> **상황**: 같은 Dockerfile 을 다시 빌드하면 왜 빠른지, 어디를 고치면 캐시가 깨지는지 확인한다.

```bash
cd /srv/devteam/proj/docker
docker build -t lab/intranet:1.0 . 2>&1 | grep -E 'CACHED|FINISHED'      # 전부 CACHED

echo '<!-- updated -->' >> index.html                                     # COPY 대상만 변경
docker build -t lab/intranet:1.0 . 2>&1 | grep -E 'CACHED|COPY|FINISHED' # RUN 은 CACHED, COPY 부터 재실행

docker build --no-cache -t lab/intranet:1.0 . 2>&1 | grep -cE 'CACHED'   # 0 = 캐시 미사용
```

- 캐시 규칙 : 지시자와 그 입력(파일 내용 해시 포함)이 **같으면 재사용**. 한 줄이라도 바뀌면 **그 줄부터 끝까지 전부 재실행**
- 그래서 Dockerfile 은 **변경이 적은 것을 위에, 자주 바뀌는 것을 아래에** 배치 (의존성 설치 → 애플리케이션 코드 복사 순)

**컨테이너 실행과 검증**

```bash
docker run -d --name lab-web -p 127.0.0.1:8081:80 lab/intranet:1.0
docker ps --filter name=lab-web --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
sleep 8                                                  # HEALTHCHECK start-period 대기
docker ps --filter name=lab-web --format '{{.Names}} {{.Status}}'
curl -s http://127.0.0.1:8081/ | head -3
docker inspect lab-web --format '{{.State.Health.Status}}'
docker inspect lab-web --format '{{range .State.Health.Log}}{{.ExitCode}} {{end}}'
docker image history lab/intranet:1.0
```

- ⚠️ 8081 은 Part 10 에서 임시로 썼다가 회수한 포트 — 여기서는 **루프백 바인딩**이라 방화벽 개방 불필요. 사내망에 열려면 `firewall-cmd --permanent --zone=internal --add-port=8081/tcp` 후 `--reload`

**검증**

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8081/
docker inspect lab-web --format '{{.State.Status}} health={{.State.Health.Status}}'
docker logs --tail 5 lab-web
docker image history lab/intranet:1.0 | wc -l
```

```text
# docker build (2회차)
 => CACHED [1/3] FROM docker.io/library/httpd:2.4-alpine
 => CACHED [2/3] RUN apk add --no-cache curl tzdata && ...
 => CACHED [3/3] COPY index.html /usr/local/apache2/htdocs/
# (index.html 변경 후)
 => CACHED [2/3] RUN apk add ...
 => [3/3] COPY index.html /usr/local/apache2/htdocs/       ← 여기부터 재실행
# curl -s http://127.0.0.1:8081/
<!doctype html>
<html lang="ko"><head><meta charset="utf-8"><title>Lab Intranet</title></head>
# docker ps --filter name=lab-web --format '{{.Names}} {{.Status}}'
lab-web Up 15 seconds (healthy)
# curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8081/
200
```

- `STATUS` 에 `(healthy)` 가 붙는 것이 `HEALTHCHECK` 의 결과 — 실패가 누적되면 `(unhealthy)`

> 📝 **시험 포인트**: 빌드 캐시는 "변경 지점 이후 전부 재실행". `docker ps` 의 `STATUS` 에 `unhealthy` 가 보이면 헬스체크 실패 고착 상태 → [[../../CONTAINER/docker]].

### 7-4. CMD vs ENTRYPOINT

> **상황**: 이미지의 기본 실행 명령을 정하는 두 지시자를 구분한다. 4-2 에서 `redis-server --appendonly yes` 로 CMD 를 덮어쓸 수 있었던 이유가 여기 있다.

| 구분 | `CMD` | `ENTRYPOINT` |
| --- | --- | --- |
| 성격 | **기본 명령/인자** | **고정 실행 파일** |
| `docker run <이미지> <명령>` | **완전히 대체됨** | 대체되지 않음 (인자로 **덧붙음**) |
| 덮어쓰기 옵션 | 인자로 직접 | `--entrypoint` 필요 |
| 둘 다 있을 때 | CMD 가 ENTRYPOINT 의 **기본 인자**가 됨 | — |
| 개수 | 마지막 1개만 유효 | 마지막 1개만 유효 |

**두 가지 표기 형식**

| 형식 | 표기 | 동작 |
| --- | --- | --- |
| exec 형식 (권장) | `CMD ["httpd-foreground"]` | **셸 없이 직접 실행** → PID 1 이 되어 시그널을 정상 수신 |
| shell 형식 | `CMD httpd-foreground` | `/bin/sh -c "..."` 로 실행 → PID 1 이 `sh` 가 되어 **SIGTERM 이 전달되지 않을 수 있음** |

```bash
# 실험용 Dockerfile 두 개로 차이 확인
mkdir -p /tmp/entrytest && cd /tmp/entrytest
printf 'FROM alpine:3.20\nCMD ["echo","default-cmd"]\n' > Dockerfile.cmd
printf 'FROM alpine:3.20\nENTRYPOINT ["echo","fixed"]\nCMD ["default"]\n' > Dockerfile.entry
docker pull alpine:3.20 -q
docker build -q -f Dockerfile.cmd   -t lab/t-cmd:1 .
docker build -q -f Dockerfile.entry -t lab/t-entry:1 .

docker run --rm lab/t-cmd:1                       # default-cmd
docker run --rm lab/t-cmd:1 echo overridden       # overridden  (CMD 완전 대체)
docker run --rm lab/t-entry:1                     # fixed default
docker run --rm lab/t-entry:1 extra               # fixed extra (CMD 만 교체)
docker run --rm --entrypoint echo lab/t-entry:1 bypass   # bypass (ENTRYPOINT 교체)

docker rmi lab/t-cmd:1 lab/t-entry:1 alpine:3.20; cd /; rm -rf /tmp/entrytest
```

- redis 이미지 구조 : `ENTRYPOINT ["docker-entrypoint.sh"]` + `CMD ["redis-server"]`
	- → `docker run redis:7-alpine redis-server --appendonly yes` 는 **CMD 만 교체**, 진입 스크립트는 그대로 실행

**검증**

```text
# docker run --rm lab/t-cmd:1
default-cmd
# docker run --rm lab/t-cmd:1 echo overridden
overridden
# docker run --rm lab/t-entry:1
fixed default
# docker run --rm lab/t-entry:1 extra
fixed extra
# docker run --rm --entrypoint echo lab/t-entry:1 bypass
bypass
```

> 📝 **시험 포인트**: `CMD` 는 `docker run` 뒤 인자로 대체 가능, `ENTRYPOINT` 는 `--entrypoint` 로만 대체. exec 형식(JSON 배열)이 시그널 처리에 유리하다는 점도 실무 서술 포인트.

### 7-5. Docker Compose

> **상황**: 웹 + 캐시를 한 파일로 묶어 한 번에 올리고 내린다. 서비스 이름으로 서로를 찾는 것까지 확인한다.

```bash
cd /srv/devteam/proj/docker
cat > compose.yaml <<'EOF'
name: lab-stack

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        APP_VER: "1.1"
    image: lab/intranet:1.1
    container_name: lab-compose-web
    ports:
      - "127.0.0.1:8082:80"
    volumes:
      - web-logs:/usr/local/apache2/logs
    environment:
      - TZ=Asia/Seoul
    networks:
      - lab-net
    depends_on:
      - redis
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    container_name: lab-compose-redis
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - compose-redis:/data
    networks:
      - lab-net
    restart: unless-stopped

volumes:
  web-logs:
  compose-redis:

networks:
  lab-net:
    external: true
EOF

docker compose config                       # 문법 검사 + 병합 결과 출력
docker compose up -d
```

| 키 | 의미 |
| --- | --- |
| `name` | 프로젝트 이름 (컨테이너·네트워크 접두사) |
| `services` | 컨테이너 정의 묶음 |
| `image` | 사용할 이미지 (`build` 와 함께 쓰면 빌드 결과의 태그) |
| `build` | 빌드 정의 — `context`, `dockerfile`, `args` |
| `container_name` | 컨테이너 이름 고정 (미지정 시 `<프로젝트>-<서비스>-<번호>`) |
| `ports` | 포트 게시 (`"호스트:컨테이너"`) |
| `volumes` | 볼륨·바인드 마운트 |
| `environment` / `env_file` | 환경변수 |
| `networks` | 연결할 네트워크 |
| `depends_on` | **기동 순서**만 보장 (준비 완료는 보장 안 함 → `condition: service_healthy` 필요) |
| `restart` | 재시작 정책 |
| `command` | 이미지 CMD 덮어쓰기 |
| `healthcheck` | 상태 점검 정의 |

```bash
docker compose ps
docker compose ps -a
docker compose logs --tail 10
docker compose logs -f web                       # 서비스별 실시간 로그 (Ctrl+C)
docker compose exec web sh -c 'curl -sf http://127.0.0.1/ | head -2'
docker compose exec web sh -c 'getent hosts redis'      # 서비스 이름으로 해석
docker compose top
docker compose images
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8082/
```

- `docker compose` (v2, 플러그인) ↔ `docker-compose` (v1, 별도 파이썬 스크립트 — **폐지**)
- 주요 하위 명령
	- `up -d` : 생성·기동 (`--build` 강제 재빌드, `--force-recreate` 재생성)
	- `down` : 정지 + 컨테이너·네트워크 삭제 (`-v` 볼륨까지, `--rmi all` 이미지까지)
	- `ps` / `logs` / `exec` / `top` / `images` / `port`
	- `config` : 병합·검증 결과 출력 (`--services` 서비스 이름만)
	- `start` / `stop` / `restart` / `pause` / `unpause`
	- `pull` / `build` / `run`(일회성)
- 파일 이름 우선순위 : `compose.yaml` > `compose.yml` > `docker-compose.yaml` > `docker-compose.yml`

**정리 (검증 후 제거)**

```bash
docker compose down -v
docker compose ps -a
docker volume ls | grep -E 'web-logs|compose-redis' || echo "compose 볼륨 삭제됨"
docker network ls | grep lab-net                 # external 네트워크는 유지됨
```

**검증**

```bash
curl -s http://127.0.0.1:8082/ | head -2
docker compose ps --format 'table {{.Name}}\t{{.Service}}\t{{.Status}}'
docker compose exec -T web getent hosts redis
docker compose down -v && docker ps -a --format '{{.Names}}' | grep -c lab-compose || echo "스택 제거 완료"
```

```text
# docker compose up -d
[+] Running 3/3
 ✔ Container lab-compose-redis  Started
 ✔ Container lab-compose-web    Started
# docker compose ps --format 'table {{.Name}}\t{{.Service}}\t{{.Status}}'
NAME                 SERVICE   STATUS
lab-compose-redis    redis     running
lab-compose-web      web       running (healthy)
# docker compose exec -T web getent hosts redis
172.18.0.3   redis lab-compose-redis
# curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8082/
200
# docker compose down -v
[+] Running 4/4
 ✔ Container lab-compose-web    Removed
 ✔ Container lab-compose-redis  Removed
 ✔ Volume lab-stack_web-logs    Removed
 ✔ Volume lab-stack_compose-redis Removed
스택 제거 완료
```

- **서비스 이름 `redis` 로 해석**되는 것이 Compose 가 자동으로 사용자 정의 네트워크를 쓰기 때문(5-5)
- `down -v` 로 볼륨까지 지웠지만 `external: true` 인 `lab-net` 과 `lab-redis` 는 그대로 남음

> 📝 **시험 포인트**: `depends_on` 은 **기동 순서만** 보장하고 "준비 완료" 는 보장하지 않는다는 점이 실무 함정. `docker compose down -v` 의 `-v` 가 볼륨 삭제라는 점도 주의.

### 7-6. 자원 제한과 cgroup 확인

> **상황**: 컨테이너 하나가 서버 자원을 다 먹는 것을 막는다. 제한이 실제 cgroup 파일에 반영되는지, 동작으로도 확인되는지 본다.

```bash
# ① CPU 제한 — 코어 0.5 개
docker run -d --name lab-cpu --cpus 0.5 redis:7-alpine sh -c 'while :; do :; done'
sleep 5
docker stats --no-stream --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}'
top -b -n 1 | head -12                                   # 호스트에서도 관찰 (Part 06)
ps -eo pid,pcpu,comm --sort=-pcpu | head -5

# ② cgroup 파일 직접 확인
CID=$(docker inspect lab-cpu --format '{{.Id}}')
CG=/sys/fs/cgroup/system.slice/docker-${CID}.scope
ls "$CG" | head
cat "$CG"/cpu.max                                        # 할당량 주기 (50000 100000 = 0.5 코어)
cat "$CG"/memory.max                                     # max = 무제한
cat "$CG"/pids.max
cat "$CG"/cpu.stat | head -3

docker rm -f lab-cpu
```

- cgroup v2 파일 의미
	- `cpu.max` : `<할당량> <주기>` (µs). `50000 100000` = 100ms 중 50ms → **0.5 코어**
	- `memory.max` : 메모리 상한 (바이트). `max` = 무제한
	- `memory.current` : 현재 사용량
	- `pids.max` : 프로세스 수 상한 (`--pids-limit`)
	- `io.max` : 블록 I/O 상한 (`--device-read-bps` 등)
- 경로 규칙 : systemd cgroup 드라이버 + cgroup v2 → `/sys/fs/cgroup/system.slice/docker-<64자ID>.scope/`

```bash
# ③ 메모리 제한 — 상한 반영 확인
docker run -d --name lab-mem --memory 256m --memory-swap 256m redis:7-alpine
docker stats --no-stream --format '{{.Name}} {{.MemUsage}} {{.MemPerc}}'
MID=$(docker inspect lab-mem --format '{{.Id}}')
cat /sys/fs/cgroup/system.slice/docker-${MID}.scope/memory.max      # 268435456
docker inspect lab-mem --format '{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}}'
docker rm -f lab-mem
```

```bash
# ④ 한도 초과 시 동작 — 6절에서 만든 이미지로 부하 발생 (stress-ng 설치 필요)
docker run -d --name lab-oom --memory 128m --memory-swap 128m lab/ubuntu-tools:1.0 \
  bash -c 'apt-get update -qq && apt-get install -y -qq stress-ng >/dev/null 2>&1 && \
           stress-ng --vm 1 --vm-bytes 400M --timeout 20s'
sleep 60
docker inspect lab-oom --format 'Status={{.State.Status}} OOMKilled={{.State.OOMKilled}} Exit={{.State.ExitCode}}'
docker logs --tail 5 lab-oom
docker rm -f lab-oom
```

- 한도를 넘기면 커널 OOM killer 가 개입 → `OOMKilled=true` 이거나 `stress-ng` 가 스스로 할당 실패로 중단. **둘 중 하나가 관측됨**(어느 쪽인지는 타이밍에 따라 다름)
- `dmesg -T | grep -i 'oom\|killed process' | tail -5` 로 호스트 커널 로그에서도 확인 가능
- Part 06 의 `top`·`vmstat` 진단 절차를 그대로 적용할 수 있음 — 컨테이너 프로세스도 호스트 프로세스이므로(4-11)

**검증**

```bash
docker run -d --name lab-cpu --cpus 0.5 redis:7-alpine sh -c 'while :; do :; done'
sleep 5; docker stats --no-stream --format '{{.Name}} {{.CPUPerc}}'
cat /sys/fs/cgroup/system.slice/docker-$(docker inspect -f '{{.Id}}' lab-cpu).scope/cpu.max
docker rm -f lab-cpu
```

```text
# docker stats --no-stream --format '{{.Name}} {{.CPUPerc}}'
lab-cpu 50.0%                       ← --cpus 0.5 가 실제로 제한
# cat .../cpu.max
50000 100000
# cat .../memory.max   (--memory 256m)
268435456
# docker inspect lab-oom --format '...'
Status=exited OOMKilled=true Exit=137
```

> 📝 **시험 포인트**: **cgroup = 자원 제한, namespace = 격리**(필기 R10 #88 ②). `--cpus 0.5` → `cpu.max 50000 100000`, `--memory 256m` → `memory.max 268435456` 환산 관계.

### 7-7. 로그 파일과 컨테이너 서비스화

> **상황**: 서버 재부팅 후에도 캐시가 자동으로 뜨는지 확인하고, `--restart` 대신 systemd 유닛으로 관리하는 방식도 비교한다.

```bash
# 로그 파일 실체
CID=$(docker inspect lab-redis --format '{{.Id}}')
ls -lh /var/lib/docker/containers/${CID}/${CID}-json.log
head -c 300 /var/lib/docker/containers/${CID}/${CID}-json.log; echo
docker inspect lab-redis --format '{{.HostConfig.LogConfig.Type}} {{.HostConfig.LogConfig.Config}}'
```

- 형식 : 한 줄에 하나의 JSON (`{"log":"...","stream":"stdout","time":"..."}`) → `docker logs` 가 이 파일을 읽어 출력
- 2-6 의 `max-size`/`max-file` 이 적용되면 `<ID>-json.log.1` 처럼 회전 파일 생성

**재부팅 후 자동 기동 검증 (`--restart`)**

```bash
docker inspect lab-redis --format '{{.HostConfig.RestartPolicy.Name}}'
systemctl is-enabled docker
# 실제 확인은 재부팅 후 — reboot; (재로그인) docker ps
```

**systemd 유닛 방식 (참고)**

```bash
cat > /etc/systemd/system/lab-redis-container.service <<'EOF'
[Unit]
Description=LAB Redis container (systemd managed)
Requires=docker.service
After=docker.service network-online.target

[Service]
Type=simple
Restart=always
RestartSec=5
ExecStart=/usr/bin/docker start -a lab-redis
ExecStop=/usr/bin/docker stop -t 10 lab-redis

[Install]
WantedBy=multi-user.target
EOF
systemd-analyze verify /etc/systemd/system/lab-redis-container.service
systemctl daemon-reload
# systemctl enable --now lab-redis-container     # ※ --restart 정책과 중복되므로 이 실습에서는 미활성
```

| 방식 | 장점 | 단점 |
| --- | --- | --- |
| `--restart unless-stopped` | 간단, Docker 가 직접 관리 | systemd 의존성 표현 불가, `systemctl status` 로 안 보임 |
| systemd 유닛 (`docker start -a`) | 다른 유닛과 순서·의존 정의 가능, 로그가 journal 로 | 유닛 관리 부담, **두 방식 병용 시 충돌** |
| `podman generate systemd` | Podman 이 유닛 파일 자동 생성 | Podman 전용 |

- ⚠️ `--restart always` 와 systemd 유닛을 **동시에** 쓰면 서로 재시작을 유발할 수 있음 → 하나만 선택
- 유닛 문법·의존 관계는 [[07-boot-systemd-log]] 3절 참조

**검증**

```bash
ls -lh /var/lib/docker/containers/$(docker inspect -f '{{.Id}}' lab-redis)/*-json.log
docker inspect lab-redis --format '{{.HostConfig.LogConfig.Type}} {{index .HostConfig.LogConfig.Config "max-size"}}'
systemd-analyze verify /etc/systemd/system/lab-redis-container.service && echo "유닛 문법 OK"
rm -f /etc/systemd/system/lab-redis-container.service && systemctl daemon-reload   # 참고용이므로 제거
```

```text
-rw-r----- 1 root root 12K ... <ID>-json.log
{"log":"1:C ... Redis version=7.x.x\n","stream":"stdout","time":"2026-09-03T..."}
# docker inspect lab-redis --format '...'
json-file 10m
유닛 문법 OK
```

> 📝 **시험 포인트**: `--restart` 정책 4종(`no`/`on-failure`/`always`/`unless-stopped`)과 의미. systemd 유닛의 `Restart=always`, `After=`, `WantedBy=` 는 Part 07 범위와 동일.

### 7-8. Podman 대응 (참고)

> **상황**: 실무에서 Rocky/RHEL 표준은 Podman 이다. 같은 작업을 Podman 으로 하면 어떻게 되는지 명령만 대응시켜 둔다. ※ 이 실습 VM 에는 Podman 미설치 → **미실행**

| Docker | Podman | 비고 |
| --- | --- | --- |
| `docker run -d --name x img` | `podman run -d --name x img` | 문법 동일 |
| `docker ps` / `images` / `logs` / `exec` | 동일 | `alias docker=podman` 가능 |
| `docker build -t x .` | `podman build -t x .` (또는 `buildah bud`) | — |
| `docker compose up` | `podman-compose up` / `podman play kube` | 별도 패키지 |
| `--restart always` | `podman generate systemd --new --name x > x.service` | **유닛 파일 생성이 표준 방식** |
| 데몬 | `dockerd` 필요 | **없음** (fork-exec) |
| rootless | 별도 구성 필요 | **기본 지원** (`podman info` 의 `rootless: true`) |
| 소켓 | `/var/run/docker.sock` | `podman.socket` (Docker API 호환 제공 가능) |
| 이미지 저장소 | `/var/lib/docker` | `/var/lib/containers`(root) / `~/.local/share/containers`(rootless) |

```bash
# ※ 미실행 — 참고용 명령
# dnf install -y podman
# podman run -d --name p-redis -p 127.0.0.1:6380:6379 redis:7-alpine
# podman generate systemd --new --name p-redis > /etc/systemd/system/p-redis.service
# systemctl daemon-reload && systemctl enable --now p-redis
# podman ps; podman images; podman logs p-redis
```

- OCI 표준(1-6) 덕분에 **Docker 로 만든 이미지를 Podman 이 그대로 실행**하고 그 반대도 성립
- rootless 의 대가 : 1024 미만 포트 바인딩 불가(기본), 일부 네트워크 기능 제약

**검증**

```bash
rpm -q podman || echo ">> podman 미설치 — 이 절은 참고용(※ 미실행)"
dnf info podman 2>/dev/null | grep -E '^(Name|Version|Repository)'
docker --version
```

```text
package podman is not installed
>> podman 미설치 — 이 절은 참고용(※ 미실행)
Name         : podman
Version      : 4.9.4
Repository   : appstream
Docker version 27.x.x, build ...
```

> 📝 **시험 포인트**: Podman 은 "데몬리스·rootless" 두 키워드. 1급 필기는 여전히 Docker 명령 중심이므로 Podman 은 개념 수준으로 충분.

---

## 8. libvirt · KVM — 조회 위주 (※ 중첩 가상화 미지원)

> 1-7 에서 확인했듯 이 VM 에는 `/dev/kvm` 이 없다. **가상머신을 실제로 부팅해 OS 를 설치하는 것은 불가능**하므로, 이 절은 ① 설치·데몬 구조 ② 조회 명령 전 범위 ③ 네트워크·스토리지 풀 정의 ④ 도메인 정의와 상태 전환까지 수행하고, 실제 게스트 설치는 `※ 미실행` 로 표기한다.

### 8-1. 설치와 데몬 구조

> **상황**: libvirt 도구 모음을 설치한다. RHEL 9 의 데몬 구조가 이전 버전과 달라졌으므로 함께 확인한다.

```bash
dnf list --available qemu-kvm libvirt virt-install libguestfs-tools virt-top 2>/dev/null | tail -8
dnf install -y qemu-kvm libvirt virt-install libguestfs-tools
rpm -q qemu-kvm libvirt libvirt-daemon-driver-qemu virt-install
```

| 패키지 | 내용 |
| --- | --- |
| `qemu-kvm` | QEMU 에뮬레이터 + KVM 가속 지원 (aarch64 는 `/usr/libexec/qemu-kvm`) |
| `libvirt` | 데몬·드라이버·`virsh` 등 관리 계층 메타패키지 |
| `virt-install` | CLI 로 게스트 설치를 시작하는 도구 (`libosinfo` 동반) |
| `libguestfs-tools` | 게스트 디스크 이미지를 호스트에서 조작 (`virt-df`, `guestfish`, `virt-customize`) |
| `virt-manager` | GUI 관리 도구 (X 필요 → ※ 미설치) |

```bash
# 데몬 기동 — 두 방식
systemctl enable --now libvirtd                 # 모놀리식(레거시 호환)
systemctl status libvirtd --no-pager | head -5

# RHEL 9 의 모듈식 데몬 (권장 방향)
systemctl list-unit-files | grep -E 'virt(qemu|network|storage|nodedev|interface|secret|proxy)d' | head
systemctl status virtqemud.socket --no-pager 2>/dev/null | head -3
```

- **RHEL 9 의 데몬 구조 변화** : 하나의 `libvirtd` 가 모든 드라이버를 담당하던 방식에서, 드라이버별 **모듈식 데몬**으로 분리
	- `virtqemud` : QEMU/KVM 하이퍼바이저 드라이버
	- `virtnetworkd` : 가상 네트워크
	- `virtstoraged` : 스토리지 풀
	- `virtnodedevd` / `virtsecretd` / `virtinterfaced` : 장치·비밀·인터페이스
	- `virtproxyd` : 원격 접속 프록시
	- 각각 `.socket` 유닛으로 **소켓 활성화**(요청이 오면 데몬 기동) → 평소 프로세스가 떠 있지 않을 수 있음
- `libvirtd` 는 호환성을 위해 남아 있으나 **폐지 예정** — 둘을 동시에 활성화하면 충돌
- 접속 URI : 시스템 전역 `qemu:///system`(root), 사용자 세션 `qemu:///session` — `virsh -c <URI>` 또는 `LIBVIRT_DEFAULT_URI`

**검증**

```bash
systemctl is-active libvirtd
virsh version
virsh uri
virsh -c qemu:///system list --all
```

```text
# systemctl is-active libvirtd
active
# virsh version
Compiled against library: libvirt ...
Using library: libvirt ...
Using API: QEMU ...
Running hypervisor: QEMU ...
# virsh uri
qemu:///system
# virsh list --all
 Id   Name   State
--------------------
```

> 📝 **시험 포인트**: `libvirtd` = 관리 데몬, `virsh` = CLI, `virt-manager` = GUI 세 짝은 필기 필수. libvirt 자체는 **하이퍼바이저가 아니라 관리 계층**.

### 8-2. virt-host-validate — KVM FAIL 기록

> **상황**: 이 서버가 가상화 호스트 자격을 갖췄는지 공식 진단 도구로 확인한다. **FAIL 결과를 그대로 남기는 것이 이 단계의 목적**이다.

```bash
virt-host-validate
virt-host-validate qemu
```

- `virt-host-validate [하이퍼바이저]` : 가상화 호스트 요구 사항 점검 (`libvirt-client` 제공)
	- 점검 항목 : CPU 가상화 확장, `/dev/kvm` 존재·권한, `/dev/vhost-net`, IOMMU, cgroup 컨트롤러, 보안 드라이버(SELinux)
	- 결과 `PASS` / `WARN`(성능·기능 제약) / `FAIL`(사용 불가)

**검증**

```bash
virt-host-validate 2>&1 | grep -E 'PASS|WARN|FAIL' | head -20
virt-host-validate 2>&1 | grep -c FAIL
ls /dev/kvm 2>&1
```

```text
# virt-host-validate
  QEMU: Checking if device /dev/kvm exists                       : FAIL (Check that CPU and firmware supports virtualization and kvm module is loaded)
  QEMU: Checking if device /dev/vhost-net exists                 : WARN (Load the 'vhost_net' module to improve performance of virtio networking)
  QEMU: Checking for cgroup 'cpu' controller support             : PASS
  QEMU: Checking for cgroup 'cpuacct' controller support         : PASS
  QEMU: Checking for cgroup 'cpuset' controller support          : PASS
  QEMU: Checking for cgroup 'memory' controller support          : PASS
  QEMU: Checking for cgroup 'devices' controller support         : ...
  QEMU: Checking for device assignment IOMMU support             : ...
  QEMU: Checking for secure guest support                        : ...
```

- **`/dev/kvm exists : FAIL` 이 중첩 가상화 미지원의 공식 증거** — 1-7 의 관찰과 일치
- 결과 : 이 호스트에서는 **KVM 가속 도메인(`<domain type='kvm'>`) 정의·기동 불가**. TCG 에뮬레이션(`type='qemu'`)만 가능하며 극도로 느림
- cgroup 항목이 PASS 인 것은 컨테이너 실습(7-6)과 같은 커널 기능을 쓰기 때문

> 📝 **시험 포인트**: KVM 사용 전제 3종 — ① CPU 가상화 확장(`/proc/cpuinfo` 의 `vmx`/`svm`) ② `kvm` 커널 모듈 적재 ③ `/dev/kvm` 존재. 필기 R07 #86 의 첫 단계가 이 확인.

### 8-3. virsh 기본 조회

> **상황**: `virsh` 의 조회 명령을 훑는다. 도메인이 아직 없으므로 호스트 정보 위주다.

```bash
virsh version
virsh nodeinfo                                   # CPU·메모리·NUMA 요약
virsh capabilities | head -40                     # 호스트가 지원하는 게스트 아키텍처·기능 (XML)
virsh capabilities | grep -E '<arch|<machine|<domain type' | head -10
virsh domcapabilities --virttype qemu 2>/dev/null | head -20
virsh list                                        # 실행 중 도메인
virsh list --all                                  # 정지 포함 전체
virsh list --inactive                             # 정지된 것만
virsh nodememstats
virsh nodecpustats --percent 2>/dev/null | head -5
virsh sysinfo 2>/dev/null | head -5
virsh --help | head -20                           # 명령 그룹 목록
virsh help domain | head -20                      # 그룹별 명령
```

- `virsh` : libvirt CLI. 인자 없이 실행하면 대화형 셸(`virsh #` 프롬프트), `quit` 로 종료
	- `-c <URI>` : 연결 대상 (**c**onnect)
	- `-q` : 부가 출력 억제 (**q**uiet)
	- `-r` : 읽기 전용 연결 (**r**eadonly)
- `virsh nodeinfo` 필드 : `CPU model`, `CPU(s)`, `CPU frequency`, `CPU socket(s)`, `Core(s) per socket`, `Thread(s) per core`, `NUMA cell(s)`, `Memory size`
- `virsh capabilities` : 호스트가 만들 수 있는 게스트의 아키텍처·머신 타입·도메인 타입 — **여기에 `<domain type='kvm'>` 이 없으면 KVM 불가**

**검증**

```bash
virsh nodeinfo
virsh list --all | tail -3
virsh capabilities | grep -oE "domain type='[a-z]+'" | sort -u
```

```text
# virsh nodeinfo
CPU model:           aarch64
CPU(s):              2
CPU frequency:       ... MHz
CPU socket(s):       1
Core(s) per socket:  2
Thread(s) per core:  1
NUMA cell(s):        1
Memory size:         3xxxxxx KiB
# virsh capabilities | grep -oE "domain type='[a-z]+'" | sort -u
domain type='qemu'          ← kvm 이 없음 = 가속 불가 (8-2 와 일치)
# virsh list --all
 Id   Name   State
--------------------
```

> 📝 **시험 포인트**: `virsh list` 는 **실행 중만**, `virsh list --all` 은 정지 포함 전체(필기 R03 #88 ①, R08 #85 ③). 출력의 `Id` 가 `-` 이면 정지 상태.

### 8-4. 가상 네트워크 — virbr0

> **상황**: libvirt 는 기본 NAT 네트워크 `default`(192.168.122.0/24)를 제공한다. Docker 의 `docker0` 와 구조가 같으므로 비교하며 본다.

```bash
virsh net-list --all
virsh net-info default
virsh net-dumpxml default
virsh net-uuid default
virsh net-start default 2>/dev/null || echo "이미 활성"
virsh net-autostart default
virsh net-list --all

ip -br a show virbr0
ip route | grep 192.168.122
iptables -t nat -L -n | grep 192.168.122 || nft list ruleset 2>/dev/null | grep -c 192.168.122
ps -ef | grep -c '[d]nsmasq'                       # libvirt 가 DHCP·DNS 용으로 띄움
virsh net-dhcp-leases default
```

- `virsh net-*` 명령
	- `net-list [--all|--inactive]` : 네트워크 목록
	- `net-info <이름>` : 상태·자동시작·브리지 이름
	- `net-dumpxml <이름>` : 정의 XML 출력
	- `net-define <파일>` / `net-undefine <이름>` : 영구 정의 추가·제거
	- `net-create <파일>` : 임시(재부팅 시 소멸) 생성
	- `net-start` / `net-destroy` : 활성·비활성 (destroy = 정지이지 정의 삭제 아님)
	- `net-autostart [--disable]` : 부팅 시 자동 시작
	- `net-edit` : XML 대화식 편집
	- `net-dhcp-leases` : 할당된 임대 목록
- `default` 네트워크 XML 핵심

```xml
<network>
  <name>default</name>
  <forward mode='nat'/>                     <!-- NAT 모드 -->
  <bridge name='virbr0' stp='on' delay='0'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp><range start='192.168.122.2' end='192.168.122.254'/></dhcp>
  </ip>
</network>
```

- `forward mode` 종류 : `nat`(기본, 외부로 SNAT) / `route`(NAT 없이 라우팅) / `bridge`(물리망 직결) / 생략(격리 네트워크)
- Docker 와의 대응 : `virbr0` ↔ `docker0`, libvirt 의 `dnsmasq` ↔ Docker 내장 DNS, 둘 다 **MASQUERADE 로 외부 통신**

**검증**

```bash
virsh net-list --all
ip -br a show virbr0
virsh net-info default | grep -E 'Active|Persistent|Autostart|Bridge'
iptables -t nat -S POSTROUTING | grep 192.168.122 | head -2
```

```text
# virsh net-list --all
 Name      State    Autostart   Persistent
--------------------------------------------
 default   active   yes         yes
# ip -br a show virbr0
virbr0    DOWN   192.168.122.1/24
# virsh net-info default
Name:           default
UUID:           ...
Active:         yes
Persistent:     yes
Autostart:      yes
Bridge:         virbr0
# iptables -t nat -S POSTROUTING | grep 192.168.122
-A POSTROUTING -s 192.168.122.0/24 -d 224.0.0.0/24 -j RETURN
-A POSTROUTING -s 192.168.122.0/24 ! -d 192.168.122.0/24 -p tcp -j MASQUERADE --to-ports 1024-65535
```

- 게스트가 없으므로 `virbr0` 상태는 `DOWN` 이 정상
- libvirt 버전에 따라 방화벽 백엔드가 iptables 대신 **nftables** 일 수 있음 → `iptables` 로 안 보이면 `nft list ruleset | grep 192.168.122` 로 확인

> 📝 **시험 포인트**: `virbr0` + `192.168.122.0/24` + NAT 조합은 KVM 기본 구성으로 출제 가능. `net-destroy` 는 "정의 삭제" 가 아니라 "정지" 라는 점이 함정(`undefine` 이 정의 삭제).

### 8-5. 스토리지 풀과 볼륨

> **상황**: 게스트 디스크 이미지를 담을 저장소를 정의한다. 디렉터리 기반 풀을 만들고 볼륨을 하나 생성한다.

```bash
virsh pool-list --all
virsh pool-info default 2>/dev/null

mkdir -p /var/lib/libvirt/images/lab
virsh pool-define-as lab-pool dir --target /var/lib/libvirt/images/lab
virsh pool-list --all
virsh pool-build lab-pool                       # 대상 디렉터리 준비(권한·SELinux 라벨)
virsh pool-start lab-pool
virsh pool-autostart lab-pool
virsh pool-info lab-pool
virsh pool-dumpxml lab-pool
```

- `virsh pool-define-as <이름> <유형> [옵션]` : 풀을 **영구 정의**
	- 유형 : `dir`(디렉터리) / `fs`(파일시스템 파티션) / `netfs`(NFS) / `logical`(LVM VG) / `disk`(디스크 전체) / `iscsi` / `gluster`
	- `--target <경로>` : 실제 저장 경로
	- `pool-create-as` 는 임시 생성(재부팅 시 소멸)
- 풀 수명주기 : `define` → `build` → `start` → `autostart` / 해제는 `destroy`(정지) → `undefine`(정의 삭제)
- `pool-refresh` : 외부에서 파일을 넣었을 때 목록 갱신

```bash
virsh vol-create-as lab-pool test.qcow2 1G --format qcow2
virsh vol-list lab-pool
virsh vol-list --details lab-pool
virsh vol-info test.qcow2 --pool lab-pool
virsh vol-path test.qcow2 --pool lab-pool
virsh vol-dumpxml test.qcow2 --pool lab-pool | head -12
ls -lh /var/lib/libvirt/images/lab/
```

- `virsh vol-create-as <풀> <이름> <용량> [--format <형식>]` : 볼륨 생성
	- `--format qcow2` : **씬 프로비저닝**(실제 사용분만 차지) + 스냅샷 지원
	- `--format raw` : 단순 블록 이미지, 성능 우위·기능 없음
	- `--allocation 0` : 초기 할당 0 (희소 파일)
- `vol-delete`, `vol-resize`, `vol-clone`, `vol-upload`/`vol-download` 도 제공

**검증**

```bash
virsh pool-list --all
virsh pool-info lab-pool | grep -E 'State|Autostart|Capacity|Available'
virsh vol-list lab-pool
ls -lh /var/lib/libvirt/images/lab/test.qcow2
```

```text
# virsh pool-list --all
 Name       State    Autostart
--------------------------------
 default    active   yes
 lab-pool   active   yes
# virsh pool-info lab-pool
Name:           lab-pool
State:          running
Persistent:     yes
Autostart:      yes
Capacity:       ...GiB
Available:      ...GiB
# virsh vol-list lab-pool
 Name          Path
--------------------------------------------------
 test.qcow2    /var/lib/libvirt/images/lab/test.qcow2
# ls -lh /var/lib/libvirt/images/lab/test.qcow2
-rw------- 1 qemu qemu 193K ... test.qcow2       ← 1G 지정이지만 실제 점유는 작음(qcow2 씬)
```

> 📝 **시험 포인트**: 기본 이미지 경로 `/var/lib/libvirt/images` 는 암기. `qcow2` = 씬 프로비저닝 + 스냅샷, `raw` = 단순·빠름.

### 8-6. qemu-img

> **상황**: 풀을 거치지 않고 직접 디스크 이미지를 다루는 명령을 익힌다. 도메인 정의에 쓸 이미지도 여기서 만든다.

```bash
cd /var/lib/libvirt/images/lab
qemu-img create -f qcow2 lab-vm.qcow2 2G
qemu-img info lab-vm.qcow2
qemu-img info --output=json lab-vm.qcow2 | head -8

qemu-img create -f raw plain.img 512M                 # raw 이미지
ls -lh plain.img; du -h plain.img                      # 희소 파일 → apparent size 와 실제 사용량 차이
qemu-img convert -f raw -O qcow2 plain.img plain.qcow2 # 형식 변환
qemu-img info plain.qcow2

qemu-img resize lab-vm.qcow2 +1G                       # 확장 (축소는 --shrink 필요, 위험)
qemu-img info lab-vm.qcow2 | grep -E 'virtual size|disk size'

qemu-img snapshot -c snap1 lab-vm.qcow2                # 내부 스냅샷 생성 (qcow2 전용)
qemu-img snapshot -l lab-vm.qcow2                      # 스냅샷 목록
qemu-img snapshot -d snap1 lab-vm.qcow2                # 삭제
qemu-img check lab-vm.qcow2                            # 무결성 검사
rm -f plain.img plain.qcow2
```

- `qemu-img create -f <형식> <파일> <크기>` : 이미지 생성 (**f**ormat)
- `qemu-img info <파일>` : 형식·가상 크기·실제 점유·백킹 파일·스냅샷
- `qemu-img convert -f <입력형식> -O <출력형식> <입력> <출력>` : 형식 변환 (**O** = output format, 대문자)
	- 지원 형식 : `qcow2` `raw` `vmdk`(VMware) `vdi`(VirtualBox) `vhdx`(Hyper-V) `vpc`
	- `-c` : 변환 시 압축, `-p` : 진행률 표시
- `qemu-img resize <파일> [+]<크기>` : 크기 조정 — **확장 후 게스트 안에서 파티션·FS 확장 필요**(Part 05 의 `growpart`/`xfs_growfs` 개념)
- `qemu-img snapshot -c|-l|-a|-d` : 생성(**c**reate) / 목록(**l**ist) / 적용(**a**pply) / 삭제(**d**elete)
- ⚠️ **실행 중인 게스트의 이미지에 `qemu-img` 로 직접 쓰기 금지** — 손상 위험. 실행 중 스냅샷은 `virsh snapshot-create-as` 사용

**검증**

```bash
qemu-img info /var/lib/libvirt/images/lab/lab-vm.qcow2 | grep -E 'file format|virtual size|disk size'
ls -lh /var/lib/libvirt/images/lab/
qemu-img check /var/lib/libvirt/images/lab/lab-vm.qcow2 | tail -2
```

```text
# qemu-img info lab-vm.qcow2
image: lab-vm.qcow2
file format: qcow2
virtual size: 3 GiB (3221225472 bytes)
disk size: 196 KiB
# qemu-img check lab-vm.qcow2
No errors were found on the image.
Image end offset: ...
```

> 📝 **시험 포인트**: `qemu-img create -f qcow2 disk.img 20G` 형태가 필기에 그대로 등장. KVM VM 생성 절차에서 **디스크 이미지 생성이 `virt-install` 보다 앞**(필기 R07 #86 ㄱ→ㄹ).

### 8-7. 도메인 정의와 조회

> **상황**: 최소 XML 로 도메인(가상머신)을 정의한다. KVM 가속이 없으므로 `type='qemu'`(TCG 소프트웨어 에뮬레이션)로 작성한다.

```bash
cat > /root/lab-vm.xml <<'EOF'
<domain type='qemu'>
  <name>lab-vm</name>
  <memory unit='MiB'>512</memory>
  <currentMemory unit='MiB'>512</currentMemory>
  <vcpu placement='static'>1</vcpu>
  <os firmware='efi'>
    <type arch='aarch64' machine='virt'>hvm</type>
    <boot dev='hd'/>
  </os>
  <features>
    <acpi/>
    <gic version='3'/>
  </features>
  <cpu mode='custom' match='exact'>
    <model fallback='allow'>cortex-a57</model>
  </cpu>
  <clock offset='utc'/>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <devices>
    <emulator>/usr/libexec/qemu-kvm</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/lab/lab-vm.qcow2'/>
      <target dev='vda' bus='virtio'/>
    </disk>
    <interface type='network'>
      <source network='default'/>
      <model type='virtio'/>
    </interface>
    <console type='pty'>
      <target type='serial' port='0'/>
    </console>
    <memballoon model='virtio'/>
  </devices>
</domain>
EOF

virsh define /root/lab-vm.xml
virsh list --all
```

- XML 주요 요소
	- `<domain type='...'>` : **`kvm`**(가속) / **`qemu`**(TCG 에뮬레이션) / `xen` / `lxc`
	- `<memory>` / `<currentMemory>` : 최대·현재 메모리
	- `<vcpu>` : 가상 CPU 수
	- `<os><type arch machine>` : 아키텍처·머신 타입. `firmware='efi'` 로 UEFI 펌웨어 자동 선택(aarch64 는 UEFI 필수)
	- `<emulator>` : QEMU 실행 파일 경로 (RHEL 계열은 `/usr/libexec/qemu-kvm`)
	- `<disk>` : `type`(file/block), `driver type`(qcow2/raw), `source`, `target dev/bus`(virtio/sata/scsi)
	- `<interface type='network'><source network='default'/>` : 8-4 의 가상 네트워크에 연결
	- `<console type='pty'>` : `virsh console` 로 붙을 시리얼 콘솔
- `virsh define <파일>` : **영구 정의**(재부팅 후에도 유지) — `/etc/libvirt/qemu/<이름>.xml` 로 저장
- `virsh create <파일>` : 정의 없이 **즉시 기동**(일회성, 종료하면 사라짐)
- 정의 실패 시 `virsh define` 이 원인 줄을 그대로 출력 → `arch`·`machine`·`emulator` 경로·펌웨어 항목을 순서대로 확인

```bash
virsh dominfo lab-vm
virsh domstate lab-vm
virsh domuuid lab-vm
virsh dumpxml lab-vm | head -20                # libvirt 가 보정·확장한 최종 XML
virsh domblklist lab-vm                        # 연결된 블록 장치
virsh domiflist lab-vm                         # 연결된 네트워크 인터페이스
virsh domstats lab-vm 2>/dev/null | head -8
virsh vcpucount lab-vm
virsh dommemstat lab-vm 2>/dev/null
ls -l /etc/libvirt/qemu/lab-vm.xml
# virsh edit lab-vm                             # ※ 대화식 XML 편집 ($EDITOR) — 검증 후 자동 반영
```

**검증**

```bash
virsh list --all
virsh dominfo lab-vm | grep -E 'Name|State|CPU\(s\)|Max memory|Persistent|Autostart'
virsh domblklist lab-vm
virsh domiflist lab-vm
ls /etc/libvirt/qemu/
```

```text
# virsh define /root/lab-vm.xml
Domain 'lab-vm' defined from /root/lab-vm.xml
# virsh list --all
 Id   Name     State
------------------------
 -    lab-vm   shut off
# virsh dominfo lab-vm
Name:           lab-vm
UUID:           ...
OS Type:        hvm
State:          shut off
CPU(s):         1
Max memory:     524288 KiB
Persistent:     yes
Autostart:      disable
# virsh domblklist lab-vm
 Target   Source
-------------------------------------------------
 vda      /var/lib/libvirt/images/lab/lab-vm.qcow2
# virsh domiflist lab-vm
 Interface   Type      Source    Model    MAC
-----------------------------------------------------------
 -           network   default   virtio   52:54:00:...
```

- `Id` 가 `-`, `State` 가 `shut off` = **정의는 되어 있으나 정지 상태** → `virsh start lab-vm` 으로 기동 가능한 상태

> 📝 **시험 포인트**: `virsh list --all` 출력에서 `Id` 가 `-` 이고 `shut off` 인 도메인은 **정의는 존재, 정지 상태**이며 `virsh start <이름>` 으로 기동(필기 R08 #85 ④). "정의 자체가 삭제된 상태" 라는 선지는 오답(그건 `undefine` 후).

### 8-8. 도메인 제어 명령 총람과 상태표

> **상황**: 도메인 상태 전환 명령을 정리하고, TCG 로라도 상태가 바뀌는지 한 번만 확인한다.

**도메인 상태표**

| 상태 | 의미 | 전환 명령 |
| --- | --- | --- |
| `shut off` | 정의되어 있으나 정지 | `start` |
| `running` | 실행 중 | `shutdown` `destroy` `suspend` `reboot` |
| `idle` | 실행 중이나 유휴(입출력 대기) | — |
| `paused` | 일시 중지 (메모리 유지, CPU 정지) | `resume` |
| `in shutdown` | 종료 진행 중 | — |
| `crashed` | 게스트 비정상 종료 | `destroy` 후 `start` |
| `pmsuspended` | 게스트 전원 관리에 의한 절전 | `dompmwakeup` |

**제어 명령**

| 명령 | 동작 | 비유 |
| --- | --- | --- |
| `virsh start <도메인>` | 기동 (`--console` 로 콘솔 동시 연결) | 전원 ON |
| `virsh shutdown <도메인>` | **게스트 OS 에 정상 종료 요청**(ACPI) — 게스트 협조 필요 | 종료 버튼 |
| `virsh destroy <도메인>` | **즉시 강제 전원 차단** (정의는 남음) | 전원 코드 뽑기 |
| `virsh reboot <도메인>` | 게스트에 재부팅 요청 | 재시작 |
| `virsh reset <도메인>` | 강제 리셋(하드 리셋) | 리셋 버튼 |
| `virsh suspend <도메인>` | 일시 중지 → `paused` | 일시 정지 |
| `virsh resume <도메인>` | 재개 | — |
| `virsh save <도메인> <파일>` / `restore <파일>` | 메모리 상태를 파일로 저장·복원 | 최대 절전 |
| `virsh managedsave <도메인>` | libvirt 가 관리하는 위치에 상태 저장 | — |
| `virsh autostart [--disable] <도메인>` | 호스트 부팅 시 자동 기동 | `systemctl enable` |
| `virsh undefine <도메인>` | **정의 삭제** (`--remove-all-storage` 로 디스크까지) | 등록 해제 |
| `virsh console <도메인>` | 시리얼 콘솔 접속 (빠져나오기 `Ctrl+]`) | 콘솔 케이블 |
| `virsh setmem <도메인> <크기>` | 실행 중 메모리 조정 (`--config` 영구) | — |
| `virsh setmaxmem <도메인> <크기> --config` | 최대 메모리 변경 (정지 상태에서) | — |
| `virsh setvcpus <도메인> <수>` | vCPU 수 조정 (`--config` 영구, `--live` 즉시) | — |
| `virsh attach-disk` / `detach-disk` | 디스크 착탈 | — |
| `virsh attach-interface` / `detach-interface` | NIC 착탈 | — |
| `virsh migrate` | 다른 호스트로 이주 (`--live` 무중단) | — |

⚠️ **TCG 에뮬레이션은 매우 느림**(가속 없이 CPU 명령을 소프트웨어로 번역). 아래는 **상태 전환만 확인**하고 즉시 `destroy` 한다. 디스크에 OS 가 없으므로 UEFI 셸에서 멈추는 것이 정상

```bash
virsh domstate lab-vm
virsh start lab-vm
virsh domstate lab-vm
virsh list
virsh suspend lab-vm; virsh domstate lab-vm
virsh resume lab-vm;  virsh domstate lab-vm
virsh destroy lab-vm                             # 강제 정지 (정의는 유지)
virsh domstate lab-vm
virsh list --all
```

```bash
# ※ 미실행 — 실제 게스트가 있을 때만 의미 있는 명령
# virsh console lab-vm            # 시리얼 콘솔 (Ctrl+] 로 탈출)
# virsh shutdown lab-vm           # 게스트 ACPI 종료 — OS 가 없으면 응답 없음
# virsh setmem lab-vm 256M --config
# virsh setvcpus lab-vm 2 --config --maximum
# virsh snapshot-create-as lab-vm snap1 --description 'before update'
# virsh snapshot-list lab-vm
# virsh snapshot-revert lab-vm snap1
# virsh snapshot-delete lab-vm snap1
```

- `shutdown` vs `destroy` : **정상 종료 요청** vs **강제 전원 차단**. 게스트에 ACPI 데몬(`acpid`)이 없으면 `shutdown` 이 무응답
- `destroy` vs `undefine` : `destroy` 는 **전원 차단**(정의 유지), `undefine` 은 **정의 삭제**(디스크는 `--remove-all-storage` 없으면 유지)
- 스냅샷은 `qcow2` + libvirt 조합에서 지원 (`--disk-only`, `--memspec` 옵션)

**검증**

```bash
virsh domstate lab-vm
virsh list --all
virsh dominfo lab-vm | grep -E 'State|Autostart'
ls -l /var/log/libvirt/qemu/lab-vm.log 2>/dev/null && tail -5 /var/log/libvirt/qemu/lab-vm.log
```

```text
# virsh start lab-vm
Domain 'lab-vm' started
# virsh domstate lab-vm
running
# virsh suspend lab-vm; virsh domstate lab-vm
Domain 'lab-vm' suspended
paused
# virsh resume lab-vm; virsh domstate lab-vm
Domain 'lab-vm' resumed
running
# virsh destroy lab-vm
Domain 'lab-vm' destroyed
# virsh domstate lab-vm
shut off
# tail -3 /var/log/libvirt/qemu/lab-vm.log
... /usr/libexec/qemu-kvm -name guest=lab-vm ... -accel tcg ...     ← 가속이 tcg
```

- 로그의 `-accel tcg` 가 **KVM 가속 미사용**의 최종 확인
- `start` 가 실패하면 `/var/log/libvirt/qemu/lab-vm.log` 의 마지막 줄에 원인이 그대로 기록됨

> 📝 **시험 포인트**: **`virsh shutdown` = 게스트 OS 에 정상 종료 신호**(필기 R02 #88 정답, R03 #88 ③ 오답 근거). `destroy` 를 "정의 삭제" 로 오해하게 만드는 선지 주의.

### 8-9. virt-install · GUI 도구 (※ 미실행)

> **상황**: 실제 게스트 설치 명령을 옵션 단위로 정리한다. `/dev/kvm` 이 없어 실행하지 않는다.

```bash
osinfo-query os | grep -i rocky | head -5           # 지원 OS 변형 목록 조회
osinfo-query os | grep -iE 'ubuntu24|centos-stream9' | head -3
virt-install --help | head -20
virt-install --osinfo list 2>/dev/null | head -5
```

```bash
# ※ 미실행 — KVM 불가 환경. 옵션 학습용
# virt-install \
#   --name rocky9-guest \
#   --memory 2048 \
#   --vcpus 2 \
#   --disk path=/var/lib/libvirt/images/lab/rocky9.qcow2,size=20,format=qcow2,bus=virtio \
#   --cdrom /var/lib/libvirt/images/Rocky-9-minimal.iso \
#   --os-variant rocky9 \
#   --network network=default,model=virtio \
#   --graphics none \
#   --console pty,target_type=serial \
#   --extra-args 'console=ttyS0,115200n8 inst.ks=http://.../ks.cfg'
```

| 옵션 | 의미 |
| --- | --- |
| `--name` | 도메인 이름 |
| `--memory <MiB>` | 메모리 크기 |
| `--vcpus <n>` | 가상 CPU 수 |
| `--disk path=,size=,format=,bus=` | 디스크 (없으면 생성). `size` 는 GiB |
| `--cdrom <ISO>` | 설치 미디어(ISO)로 부팅 |
| `--location <URL|경로>` | 네트워크·로컬 트리에서 커널·initrd 직접 추출해 설치(텍스트 설치에 필요) |
| `--os-variant` / `--osinfo` | 게스트 OS 종류 — 최적 장치·드라이버 선택 (`osinfo-query os` 로 조회) |
| `--network network=default,model=virtio` | 가상 네트워크 연결 (`bridge=br0` 도 가능) |
| `--graphics none` | **그래픽 콘솔 없이** 설치 (X 없는 서버용) — `vnc`/`spice` 도 가능 |
| `--console pty,target_type=serial` | 시리얼 콘솔 지정 |
| `--extra-args` | 커널 부팅 파라미터 전달 (킥스타트 지정 등) |
| `--import` | 이미 준비된 디스크 이미지를 설치 없이 등록 |
| `--noautoconsole` | 설치 후 콘솔 자동 연결 안 함 |
| `--print-xml` | 실제 생성 없이 XML 만 출력 (**정의 미리보기**) |

```bash
# 실행 없이 XML 만 확인하는 안전한 방법
virt-install --name preview-vm --memory 512 --vcpus 1 \
  --disk path=/var/lib/libvirt/images/lab/lab-vm.qcow2,format=qcow2 \
  --import --os-variant rocky9 --graphics none --print-xml 2>/dev/null | head -20
```

**GUI·부가 도구 (※ 미실행 — X 윈도 미설치)**

| 도구 | 용도 |
| --- | --- |
| `virt-manager` | GUI 가상머신 관리자 (생성·콘솔·자원 조정) |
| `virt-viewer` | 게스트 화면만 보는 뷰어 (SPICE/VNC) |
| `virt-clone --original <원본> --name <새이름> --auto-clone` | 도메인 복제 |
| `virt-sysprep -d <도메인>` | 복제 전 고유 정보(호스트명·SSH 키·머신 ID) 제거 |
| `virt-df -a <이미지>` | 게스트 디스크 사용량 조회 (게스트 기동 없이) |
| `virt-cat` / `virt-edit` / `guestfish` | 게스트 이미지 안의 파일 열람·편집 |
| `virt-top` | 도메인별 자원 사용량 (top 유사) |
| `virt-xml` | 도메인 XML 을 CLI 로 수정 |

```bash
# 게스트를 켜지 않고 이미지 내부를 보는 도구 (libguestfs-tools)
virt-df -a /var/lib/libvirt/images/lab/lab-vm.qcow2 2>&1 | head -3   # 빈 이미지라 파일시스템 없음
```

**검증**

```bash
osinfo-query os 2>/dev/null | grep -ci rocky
virt-install --version
command -v virt-clone virt-sysprep virt-df guestfish virt-xml 2>/dev/null
```

```text
# osinfo-query os | grep -i rocky | head -3
 rocky9              | Rocky Linux 9      | 9        | http://rockylinux.org/rocky/9
 rocky9.1            | Rocky Linux 9.1    | 9.1      | ...
# virt-install --version
4.x.x
# virt-df -a lab-vm.qcow2
virt-df: no filesystems were found in the disk image.      ← 빈 qcow2 이므로 정상
```

> 📝 **시험 포인트**: KVM VM 생성 절차 = **① 가상화 지원 확인 → ② `qemu-img create` 디스크 생성 → ③ `virt-install` 설치 → ④ `virsh` 로 관리**(필기 R07 #86, 정답 ㄴ→ㄱ→ㄹ→ㄷ). `virt-install` 은 **생성** 명령이지 삭제 명령이 아님(R03 #88 ② 오답).

### 8-10. libvirt 경로표와 정리

> **상황**: 설정·이미지·로그가 어디에 있는지 표로 고정하고, 실습에서 만든 도메인·풀을 정리한다.

| 경로 | 내용 |
| --- | --- |
| `/etc/libvirt/libvirtd.conf` | 데몬 설정(인증·리스닝) |
| `/etc/libvirt/qemu.conf` | QEMU 드라이버 설정(실행 사용자·SELinux 라벨) |
| `/etc/libvirt/qemu/<도메인>.xml` | **도메인 정의 XML** (직접 편집 금지 → `virsh edit` 사용) |
| `/etc/libvirt/qemu/networks/*.xml` | 가상 네트워크 정의 |
| `/etc/libvirt/storage/*.xml` | 스토리지 풀 정의 |
| `/var/lib/libvirt/images/` | **기본 디스크 이미지 저장소**(`default` 풀) |
| `/var/lib/libvirt/qemu/` | 실행 중 도메인의 런타임 상태(모니터 소켓, nvram) |
| `/var/log/libvirt/qemu/<도메인>.log` | **도메인별 QEMU 실행 로그** — 기동 실패 원인이 여기 |
| `/var/log/libvirt/libvirtd.log` | 데몬 로그(설정에 따라) |
| `/run/libvirt/` | 소켓·PID |

```bash
ls -l /etc/libvirt/qemu/ /etc/libvirt/qemu/networks/ /etc/libvirt/storage/
ls -l /var/log/libvirt/qemu/
```

**정리**

⚠️ `--remove-all-storage` 는 도메인의 디스크 이미지를 **함께 삭제**함. 대상 확인 후 실행

```bash
virsh domblklist lab-vm                      # 삭제될 디스크 확인
virsh destroy lab-vm 2>/dev/null || true
virsh undefine lab-vm --nvram                # 정의 삭제 (UEFI nvram 포함). 디스크는 남김
virsh list --all

virsh vol-delete test.qcow2 --pool lab-pool
virsh vol-list lab-pool
virsh pool-destroy lab-pool                  # 풀 비활성
virsh pool-undefine lab-pool                 # 풀 정의 삭제
virsh pool-list --all
rm -rf /var/lib/libvirt/images/lab /root/lab-vm.xml
```

- `virsh undefine` 옵션 : `--remove-all-storage`(디스크 삭제) / `--nvram`(UEFI 변수 파일 삭제) / `--managed-save`(저장 상태 삭제) / `--snapshots-metadata`
- 도메인 삭제 순서 : `destroy`(정지) → `undefine`(정의 삭제) → 이미지 수동 삭제
- 풀 삭제 순서 : `vol-delete` → `pool-destroy` → `pool-undefine` (`pool-delete` 는 대상 디렉터리 내용까지 삭제 ⚠️)

**검증**

```bash
virsh list --all
virsh pool-list --all
ls /etc/libvirt/qemu/*.xml 2>&1 | tail -1
ls /var/lib/libvirt/images/ 2>/dev/null
systemctl is-active libvirtd
```

```text
# virsh list --all
 Id   Name   State
--------------------
# virsh pool-list --all
 Name      State    Autostart
-------------------------------
 default   active   yes
# ls /etc/libvirt/qemu/*.xml
ls: cannot access '/etc/libvirt/qemu/*.xml': No such file or directory
```

> 📝 **시험 포인트**: 도메인 정의 XML 경로 `/etc/libvirt/qemu/`, 이미지 경로 `/var/lib/libvirt/images/`, 로그 `/var/log/libvirt/qemu/` 3종은 서술형 대비 암기.

### 8-11. 다른 가상화 도구의 CLI (참고)

> **상황**: 필기에서 제품 이름과 관리 명령을 짝짓는 문항이 나온다. 이름만 정리한다. ※ 이 환경에 미설치 — 미실행

| 제품 | 관리 CLI | 대표 명령 |
| --- | --- | --- |
| KVM/QEMU (libvirt) | `virsh` | `virsh list --all`, `virsh start <도메인>` |
| Xen | `xl` (구 `xm`) | `xl list`, `xl create <cfg>`, `xl destroy <도메인>`, `xl console` |
| VirtualBox | `VBoxManage` | `VBoxManage list vms`, `VBoxManage startvm <VM> --type headless`, `VBoxManage controlvm <VM> poweroff` |
| VMware ESXi | `esxcli`, `vim-cmd` | `vim-cmd vmsvc/getallvms` |
| Hyper-V | PowerShell | `Get-VM`, `Start-VM` |
| Vagrant (프로비저닝) | `vagrant` | `vagrant up`, `vagrant ssh`, `vagrant halt`, `vagrant destroy` — `Vagrantfile` 기반 |
| LXC/LXD | `lxc` | `lxc list`, `lxc launch ubuntu:24.04 c1` |
| Kubernetes | `kubectl` | `kubectl get pods`, `kubectl apply -f`, `kubectl logs` |

- **쿠버네티스 최소 개념**(필기 R03 #87)
	- **Pod** : 하나 이상의 컨테이너를 묶는 **최소 배포 단위**(네트워크·스토리지 공유)
	- **Service** : Pod 집합에 대한 **고정 접근 지점**(가상 IP + 로드밸런싱)
	- **Deployment** : Pod 복제본 수·롤링 업데이트 관리
	- **Node** : 클러스터를 구성하는 물리·가상 서버
	- **kubelet** : 각 노드에서 Pod 를 실행·감시하는 에이전트 (**레지스트리가 아님**)
- 클러스터 유형(필기 대비) : **HPC**(고계산, 베어울프) / **LVS**(부하분산, Director + Real Server) / **HA**(고가용, Heartbeat·Pacemaker) → [[../THEORY/network-service]] 7-2

**검증**

```bash
command -v virsh xl VBoxManage vagrant lxc kubectl 2>/dev/null
virsh --version
rpm -qf "$(command -v virsh)"
```

```text
/usr/bin/virsh                 ← 이 환경에 설치된 가상화 CLI 는 virsh 뿐
# virsh --version
10.x.x
# rpm -qf /usr/bin/virsh
libvirt-client-...el9.aarch64
```

- 나머지 CLI 는 해당 제품이 설치된 환경에서만 존재 → 이름·용도만 암기

> 📝 **시험 포인트**: `virsh` 는 **libvirt/KVM 관리 도구이지 Docker 관리 도구가 아님**(필기 R03 #88 ④ 오답). Xen 은 `xl`, VirtualBox 는 `VBoxManage`.

---

## 9. 정리와 검증

### 9-1. 남길 것과 지울 것

> **상황**: 실습으로 만든 자원 중 다음 파트에서 필요한 것과 그렇지 않은 것을 구분한다. **`lab-redis` 와 `lab-ubuntu` 는 반드시 남긴다.**

```bash
docker ps -a --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.Size}}'
docker volume ls
docker network ls
```

| 자원 | 처리 | 이유 |
| --- | --- | --- |
| 컨테이너 `lab-redis` | **유지** | Part 09 웹의 캐시. [[12-backup-recovery-review]] 백업 대상 |
| 컨테이너 `lab-ubuntu` | **유지** | apt/dpkg 복습용. Part 12 컨테이너 백업 대상 |
| 볼륨 `redis-data` | **유지** | Part 12 볼륨 백업 실습의 원본 |
| 이미지 `redis:7-alpine`, `ubuntu:24.04`, `lab/ubuntu-tools:1.0` | **유지** | 위 컨테이너의 기반 |
| 네트워크 `lab-net` | **유지** | 5-5 검증 구성 보존 |
| 컨테이너 `lab-web` | **제거** | Dockerfile 빌드 검증 완료 |
| 이미지 `lab/intranet:1.0`, `:1.1`, `httpd:2.4-alpine` | 제거(선택) | 재빌드 가능 |
| Compose 스택 `lab-stack` | **제거 완료**(7-5 `down -v`) | — |

```bash
# lab-web 정리
docker stop lab-web && docker rm lab-web
docker rmi lab/intranet:1.0 lab/intranet:1.1
docker ps -a --format '{{.Names}} {{.Status}}'

# compose 잔여물 확인
docker ps -a --filter name=lab-compose --format '{{.Names}}' || echo "compose 컨테이너 없음"
docker volume ls | grep -E 'lab-stack' || echo "compose 볼륨 없음"
```

**검증**

```bash
docker ps -a --format 'table {{.Names}}\t{{.Status}}'
docker volume ls --format '{{.Name}}'
docker network ls --format '{{.Name}}'
docker images --format '{{.Repository}}:{{.Tag}}' | sort
```

```text
# docker ps -a --format 'table {{.Names}}\t{{.Status}}'
NAMES        STATUS
lab-redis    Up 1 hour
lab-ubuntu   Exited (0) 30 minutes ago
# docker volume ls --format '{{.Name}}'
redis-data
# docker network ls --format '{{.Name}}'
bridge
host
lab-net
none
```

- `lab-ubuntu` 가 `Exited` 인 것은 정상 — 메인 프로세스가 `bash` 라 셸 종료 시 함께 멈춤(6-14). 필요 시 `docker start -ai lab-ubuntu`

> 📝 **시험 포인트**: 정지 컨테이너도 디스크·이름을 점유하므로 `docker ps -a` 로 주기 점검. 이름은 중복될 수 없어 같은 이름으로 재생성하려면 먼저 `docker rm` 필요.

### 9-2. 디스크 사용량과 일괄 정리 — system df · prune

> **상황**: Docker 가 차지한 용량을 확인하고 정리한다. `prune` 의 삭제 범위를 정확히 알고 실행한다.

```bash
docker system df
docker system df -v | head -30
du -sh /var/lib/docker
df -h /var/lib/docker
docker system info --format '{{.Containers}} containers, {{.Images}} images'
docker system events --since 30m --until 0m 2>/dev/null | head -5    # 최근 이벤트 이력
```

⚠️ **`docker system prune` 삭제 범위** — 실행 전 반드시 확인

| 명령 | 삭제 대상 |
| --- | --- |
| `docker system prune` | 정지된 컨테이너 + **미사용 네트워크** + dangling 이미지 + 빌드 캐시 |
| `docker system prune -a` | 위 + **어떤 컨테이너도 쓰지 않는 모든 이미지** ⚠️ 재다운로드 필요 |
| `docker system prune --volumes` | 위 + **미사용 볼륨** ⚠️⚠️ **데이터 소실 위험** |
| `docker container prune` | 정지된 컨테이너만 |
| `docker image prune` / `-a` | dangling 이미지 / 미사용 전체 이미지 |
| `docker volume prune` | 미사용 볼륨 ⚠️ |
| `docker network prune` | 미사용 사용자 정의 네트워크 |
| `docker builder prune` | 빌드 캐시 |

- **`lab-ubuntu` 가 정지 상태이므로 `docker system prune` 을 그냥 실행하면 삭제됨** → 이 실습에서는 범위를 좁혀 실행

```bash
# 안전한 정리 — 대상 확인 후 개별 실행
docker images --filter 'dangling=true' -q                 # 대상 확인
docker image prune -f                                      # dangling 이미지만
docker builder prune -f                                    # 빌드 캐시만
docker network prune -f                                    # lab-net 은 lab-redis 가 사용 중이라 유지됨
docker system df
```

```bash
# ※ 이 실습에서는 실행하지 말 것 — lab-ubuntu·redis-data 가 삭제될 수 있음
# docker system prune -a --volumes -f
```

**검증**

```bash
docker system df
docker ps -a --format '{{.Names}}' | sort
docker volume ls --format '{{.Name}}'
docker network ls --format '{{.Name}}' | sort
```

```text
# docker system df
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          3         2         ...MB     ...MB (..%)
Containers      2         1         ...kB     ...kB (..%)
Local Volumes   1         1         ...MB     0B (0%)
Build Cache     0         0         0B        0B
# docker ps -a --format '{{.Names}}' | sort
lab-redis
lab-ubuntu
# docker volume ls --format '{{.Name}}'
redis-data
```

- `Local Volumes ... RECLAIMABLE 0B` = 볼륨이 컨테이너에 물려 있어 회수 대상이 아님 → 실수 삭제 위험이 낮은 상태

> 📝 **시험 포인트**: `prune` 계열의 공통 의미는 "**미사용 자원 일괄 삭제**". `-a`(모든 미사용 이미지)와 `--volumes`(볼륨 포함)가 위험 옵션.

### 9-3. 검증 스크립트 check-part11.sh

> **상황**: 재부팅 후에도 이 파트의 결과물이 살아 있는지 한 번에 확인할 스크립트를 만든다. Part 05 의 `check-part05.sh` 와 같은 형식이다.

```bash
cat > /usr/local/bin/check-part11.sh <<'EOF'
#!/bin/bash
# LAB 11 검증 — 컨테이너·가상화. 전부 OK 여야 함
ok(){ printf '  [OK]  %s\n' "$1"; }  ng(){ printf '  [NG]  %s\n' "$1"; RC=1; }
RC=0

echo "== Docker 엔진"
systemctl is-active --quiet docker && ok "docker.service active" || ng "docker.service 비활성"
systemctl is-enabled --quiet docker && ok "docker.service enabled" || ng "docker 부팅 활성 아님"
docker info >/dev/null 2>&1 && ok "데몬 API 응답" || ng "데몬 응답 없음"
[ "$(docker info --format '{{.Driver}}')" = "overlay2" ] && ok "Storage Driver overlay2" \
  || ng "Storage Driver=$(docker info --format '{{.Driver}}')"
[ "$(docker info --format '{{.CgroupVersion}}')" = "2" ] && ok "Cgroup Version 2" \
  || ng "Cgroup Version=$(docker info --format '{{.CgroupVersion}}')"
[ "$(docker info --format '{{.LiveRestoreEnabled}}')" = "true" ] && ok "live-restore 적용" \
  || ng "daemon.json live-restore 미적용"
[ -f /etc/docker/daemon.json ] && python3 -m json.tool /etc/docker/daemon.json >/dev/null 2>&1 \
  && ok "daemon.json JSON 유효" || ng "daemon.json 없음/문법 오류"

echo "== 권한"
id -nG dev1 2>/dev/null | tr ' ' '\n' | grep -qx docker && ok "dev1 docker 그룹" || ng "dev1 docker 그룹 아님"

echo "== 컨테이너"
docker ps --format '{{.Names}}' | grep -qx lab-redis && ok "lab-redis 실행 중" || ng "lab-redis 미실행"
docker ps -a --format '{{.Names}}' | grep -qx lab-ubuntu && ok "lab-ubuntu 존재" || ng "lab-ubuntu 없음"
[ "$(docker inspect lab-redis --format '{{.HostConfig.RestartPolicy.Name}}' 2>/dev/null)" = "unless-stopped" ] \
  && ok "lab-redis restart=unless-stopped" || ng "lab-redis 재시작 정책 확인 필요"
docker port lab-redis 2>/dev/null | grep -q '127.0.0.1:6379' \
  && ok "6379 루프백 바인딩" || ng "6379 바인딩 확인 필요(외부 노출 위험)"
ss -tlnH 2>/dev/null | grep -q '127.0.0.1:6379' && ok "호스트 6379 LISTEN" || ng "6379 리슨 없음"
ss -tlnH 2>/dev/null | grep -qE '0\.0\.0\.0:6379|\[::\]:6379' && ng "⚠️ 6379 전 대역 노출" || ok "6379 전 대역 미노출"

echo "== 볼륨·네트워크"
docker volume ls --format '{{.Name}}' | grep -qx redis-data && ok "redis-data 볼륨" || ng "redis-data 볼륨 없음"
[ -d "$(docker volume inspect redis-data --format '{{.Mountpoint}}' 2>/dev/null)" ] \
  && ok "볼륨 실제 경로 존재" || ng "볼륨 경로 없음"
docker network ls --format '{{.Name}}' | grep -qx lab-net && ok "lab-net 네트워크" || ng "lab-net 없음"

echo "== 이미지"
for img in redis:7-alpine ubuntu:24.04 lab/ubuntu-tools:1.0; do
  docker images --format '{{.Repository}}:{{.Tag}}' | grep -qx "$img" && ok "이미지 $img" || ng "이미지 $img 없음"
done

echo "== 데이터 영속성"
docker exec lab-redis redis-cli -e -a "${REDIS_PW:-<비밀번호>}" get lab:part11 >/dev/null 2>&1 \
  && ok "redis 키 lab:part11 조회" || ng "redis 조회 실패(비밀번호는 REDIS_PW 로 전달)"

echo "== 가상화"
[ -e /dev/kvm ] && ok "/dev/kvm 존재(가속 가능)" || echo "  [--]  /dev/kvm 없음 = 중첩 가상화 미지원(정상, 8-2 참조)"
systemctl is-active --quiet libvirtd && ok "libvirtd active" || echo "  [--]  libvirtd 비활성(조회 실습만 했다면 무방)"
virsh -q list --all >/dev/null 2>&1 && ok "virsh 연결 가능" || echo "  [--]  virsh 미설치/미연결"

echo "== 용량"; docker system df
echo; [ $RC -eq 0 ] && echo "ALL OK" || echo "일부 실패 — 위 [NG] 확인"
exit $RC
EOF
chmod +x /usr/local/bin/check-part11.sh
REDIS_PW='<비밀번호>' /usr/local/bin/check-part11.sh
```

- 구성 : `systemctl is-active` → `docker info --format` 값 비교 → `docker ps/volume/network/images` 존재 확인 → `ss` 로 **노출 여부까지 점검** → `docker exec` 로 실제 데이터 조회
- `[--]` 표기 : 환경 제약으로 실패가 정상인 항목(KVM) — `[NG]` 로 세지 않음
- 비밀번호는 인자가 아니라 환경변수 `REDIS_PW` 로 전달해 `ps` 노출을 줄임

**검증**

```bash
REDIS_PW='<비밀번호>' /usr/local/bin/check-part11.sh | grep -E '^\s+\[(OK|NG|--)\]|ALL OK'
echo "rc=$?"
# 재부팅 후 재확인
# reboot → 재로그인 → REDIS_PW='<비밀번호>' /usr/local/bin/check-part11.sh
```

```text
  [OK]  docker.service active
  [OK]  docker.service enabled
  [OK]  데몬 API 응답
  [OK]  Storage Driver overlay2
  [OK]  Cgroup Version 2
  [OK]  live-restore 적용
  [OK]  daemon.json JSON 유효
  [OK]  dev1 docker 그룹
  [OK]  lab-redis 실행 중
  [OK]  lab-ubuntu 존재
  [OK]  lab-redis restart=unless-stopped
  [OK]  6379 루프백 바인딩
  [OK]  호스트 6379 LISTEN
  [OK]  6379 전 대역 미노출
  [OK]  redis-data 볼륨
  [OK]  볼륨 실제 경로 존재
  [OK]  lab-net 네트워크
  [OK]  이미지 redis:7-alpine
  [OK]  이미지 ubuntu:24.04
  [OK]  이미지 lab/ubuntu-tools:1.0
  [OK]  redis 키 lab:part11 조회
  [--]  /dev/kvm 없음 = 중첩 가상화 미지원(정상, 8-2 참조)
  [OK]  libvirtd active
  [OK]  virsh 연결 가능
ALL OK
```

- 재부팅 후에도 `lab-redis` 가 자동으로 뜨는 것이 `--restart unless-stopped` + `docker.service enabled` 의 효과

> 📝 **시험 포인트**: 실기 서술형은 "설정 후 어떻게 검증하는가" 를 함께 묻는 경우가 많음 — 서비스는 `systemctl is-active`, 포트는 `ss -tlnp`, 컨테이너는 `docker ps`/`docker exec` 가 표준 답안.

---

## 체크리스트

| 수행 항목 | 명령 | 확인 방법 | ☐ |
| --- | --- | --- | --- |
| 가상화 유형·하이퍼바이저 분류 정리 | 1-1·1-2 표 | 전가상화/반가상화/Type1/Type2 구분 서술 | ☐ |
| 컨테이너 vs VM 비교표 | 1-3 표 | 커널 공유·부팅 시간·격리 수준 3축 | ☐ |
| namespace·cgroup 실물 확인 | `ls -l /proc/1/ns/` `stat -fc %T /sys/fs/cgroup` | ns inode 7종, `cgroup2fs` | ☐ |
| 중첩 가상화 미지원 증거 확보 | `ls /dev/kvm` `systemd-detect-virt` `virt-what` | `No such file`, `apple`/`qemu` | ☐ |
| 충돌 패키지 확인·정리 | `rpm -q podman buildah runc` → `dnf remove` | `not installed` | ☐ |
| docker-ce 저장소 등록 | `dnf config-manager --add-repo …docker-ce.repo` | `dnf repolist \| grep docker` | ☐ |
| Docker 설치·기동 | `dnf install -y docker-ce …` `systemctl enable --now docker` | `docker run --rm hello-world` | ☐ |
| docker info 핵심값 확인 | `docker info --format '{{.Driver}} {{.CgroupVersion}}'` | `overlay2 2` | ☐ |
| dev1 docker 그룹 위임 + 위험 인지 | `usermod -aG docker dev1` | `su - dev1 -c 'docker ps'` 성공 | ☐ |
| daemon.json 로그 회전·live-restore | `/etc/docker/daemon.json` 작성 → `systemctl restart docker` | `docker info \| grep Live Restore` | ☐ |
| /var/lib/docker 구조 파악 | `du -sh /var/lib/docker/*` | overlay2·volumes·containers | ☐ |
| firewalld·iptables 관계 확인 | `iptables -t nat -L DOCKER -n` `firewall-cmd --get-active-zones` | `docker` 존, DNAT 규칙 | ☐ |
| 이미지 검색·pull | `docker search` `docker pull redis:7-alpine ubuntu:24.04` | `docker images` | ☐ |
| 아키텍처 확인 | `docker image inspect … --format '{{.Architecture}}'` | `arm64` | ☐ |
| history·tag·rmi·prune | `docker history` `tag` `rmi` `image prune` | `Untagged:`/`Deleted:` | ☐ |
| save/load 왕복 검증 | `docker save -o` → `docker rmi` → `docker load -i` | `Loaded image:` + ENTRYPOINT 보존 | ☐ |
| export/import 차이 확인 | `docker export`/`import` | `Cmd=[] Entry=[]` (메타 소실) | ☐ |
| lab-redis 기동 (루프백·인증·볼륨) | `docker run -d --name lab-redis -p 127.0.0.1:6379:6379 -v redis-data:/data …` | `docker ps` PORTS `127.0.0.1:6379->6379/tcp` | ☐ |
| docker-proxy 리슨 확인 | `ss -tlnp \| grep 6379` | `docker-proxy` | ☐ |
| redis 접속 검증 | `docker exec -it lab-redis redis-cli -a … ping` | `PONG` | ☐ |
| 볼륨 영속성 3단 검증 | `set` → `stop/start` → `rm -f` → 재생성 | `get lab:part11` 유지 | ☐ |
| logs 옵션 전 범위 | `docker logs -f --tail --since --timestamps` | `Ready to accept connections` | ☐ |
| inspect·stats·top·port·diff | 각 명령 | Pid·CPU%·PORTS·`A`/`C`/`D` | ☐ |
| docker cp 양방향 | `docker cp` 컨테이너↔호스트 | `docker diff` 에 `A /tmp/hello.sh` | ☐ |
| 생명주기 제어 전 범위 | `pause/unpause/restart/kill/rename/update/wait` | `docker ps` STATUS 전이 | ☐ |
| attach vs exec 차이 | `docker attach`(Ctrl+P,Q) / `docker exec -it` | attach 후에도 `Up` | ☐ |
| 호스트 PID·네임스페이스 대조 | `ps -ef \| grep redis-server` + `/proc/<PID>/ns/` | 호스트 PID ≠ 컨테이너 PID 1 | ☐ |
| nsenter 로 컨테이너 netns 조회 | `nsenter -t <PID> -n ss -tlnp` | `0.0.0.0:6379` | ☐ |
| 볼륨 관리 명령 | `docker volume create/ls/inspect/rm/prune` | `Mountpoint` 경로 | ☐ |
| 바인드 마운트 SELinux 재현·해결 | `-v /srv/share/redis:/data` → 실패 → `:z`/`:Z` | `ausearch -m avc`, `ls -Z` `container_file_t` | ☐ |
| tmpfs·`--mount` vs `-v` | `--tmpfs` `--mount type=bind` | 없는 경로: `-v` 자동생성 / `--mount` 오류 | ☐ |
| 사용자 정의 네트워크 이름 해석 | `docker network create lab-net` + `connect` | `redis-cli -h lab-redis ping` → PONG | ☐ |
| 네트워크 모드 3종 비교 | `--network bridge/host/none` | `ls /sys/class/net` 결과 차이 | ☐ |
| lab-ubuntu 기동·커널 공유 확인 | `docker run -it --name lab-ubuntu --hostname ubuntu-lab ubuntu:24.04 bash` | os-release=Ubuntu, `uname -r`=Rocky 커널 | ☐ |
| APT 저장소 형식 2종 | `/etc/apt/sources.list.d/ubuntu.sources` | `Types/URIs/Suites/Components` | ☐ |
| apt update/list/search/show | 각 명령 | `apt policy htop` Candidate | ☐ |
| apt install 4종 + 권장 제외 | `apt install -y htop curl vim tree` | `dpkg -l` 이 `ii` | ☐ |
| apt-cache depends/rdepends/policy | 각 명령 | Depends 목록 | ☐ |
| remove vs purge 실증 | `apt remove nano` → `ls /etc/nanorc` → `apt purge nano` | `rc` 상태 → 파일 소멸 | ☐ |
| 캐시 정리 | `apt clean` `autoclean` | `/var/cache/apt/archives` 비움 | ☐ |
| dpkg 질의 6종 | `-l -L -S -s -I -c` | `htop: /usr/bin/htop` 등 | ☐ |
| .deb 직접 설치·의존성 복구 | `apt download` → `dpkg -i` → `apt -f install` | `dpkg -l htop` = `ii` | ☐ |
| dpkg -r / -P / --configure -a | 각 명령 | `rc` → 목록에서 제거 | ☐ |
| apt-mark hold / dpkg-query / alternatives | `apt-mark hold htop` `dpkg-query -W -f=` `update-alternatives --display editor` | `hi` 상태, `/usr/bin/editor` 링크 | ☐ |
| apt 이력 확인 | `/var/log/apt/history.log` | `Commandline:` 줄 | ☐ |
| RPM↔DEB 대응표 암기 | 6-13 표 | `rpm -ivh`↔`dpkg -i` 등 | ☐ |
| lab-ubuntu 재진입·commit | `docker start -ai` → `docker commit … lab/ubuntu-tools:1.0` | 새 컨테이너에 htop 존재 | ☐ |
| Dockerfile 작성·빌드 | `/srv/devteam/proj/docker/Dockerfile` → `docker build -t lab/intranet:1.0 .` | `docker images` | ☐ |
| 레이어 캐시 관찰 | 재빌드 / `--no-cache` | `CACHED` 출력 유무 | ☐ |
| lab-web 실행·헬스체크 | `docker run -d -p 127.0.0.1:8081:80` | `curl` 200, STATUS `(healthy)` | ☐ |
| CMD vs ENTRYPOINT 실험 | 실험용 이미지 2종 | 인자 대체 결과 차이 | ☐ |
| Compose 스택 up/ps/logs/exec/down -v | `compose.yaml` 작성 → `docker compose …` | 서비스명 `redis` 해석, 8082 응답 200 | ☐ |
| 자원 제한·cgroup 확인 | `--cpus 0.5` `--memory 256m` | `docker stats` 50%, `cpu.max 50000 100000` | ☐ |
| 컨테이너 로그 파일 경로 | `/var/lib/docker/containers/<ID>/<ID>-json.log` | JSON 한 줄 형식 | ☐ |
| libvirt 설치·데몬 구조 | `dnf install -y qemu-kvm libvirt virt-install libguestfs-tools` | `virsh version`, `virtqemud` 유닛 | ☐ |
| virt-host-validate FAIL 기록 | `virt-host-validate` | `/dev/kvm exists : FAIL` | ☐ |
| virsh 호스트 조회 | `nodeinfo` `capabilities` `list --all` | `domain type='qemu'` 만 존재 | ☐ |
| 가상 네트워크 확인 | `virsh net-list/net-info/net-dumpxml default` | `virbr0` 192.168.122.1/24 NAT | ☐ |
| 스토리지 풀 정의~시작 | `pool-define-as` `pool-build` `pool-start` `pool-autostart` | `virsh pool-info lab-pool` running | ☐ |
| 볼륨 생성·조회 | `vol-create-as lab-pool test.qcow2 1G --format qcow2` | `virsh vol-list lab-pool` | ☐ |
| qemu-img 전 범위 | `create/info/convert/resize/snapshot -l/check` | `file format: qcow2` | ☐ |
| 도메인 정의·조회 | `virsh define lab-vm.xml` → `dominfo/dumpxml/domblklist/domiflist` | `shut off`, `Id` = `-` | ☐ |
| 도메인 상태 전환 | `start` → `suspend` → `resume` → `destroy` | `running`→`paused`→`shut off` | ☐ |
| TCG 확인 | `/var/log/libvirt/qemu/lab-vm.log` | `-accel tcg` | ☐ |
| virt-install 옵션 학습 (※ 미실행) | `--name --memory --vcpus --disk --cdrom --os-variant --network --graphics` | `osinfo-query os \| grep rocky` | ☐ |
| libvirt 정리 | `undefine` `vol-delete` `pool-destroy` `pool-undefine` | `virsh list --all` 비어 있음 | ☐ |
| lab-web·compose 제거, lab-redis·lab-ubuntu 유지 | 9-1 | `docker ps -a` 2개 | ☐ |
| 용량 확인·안전 정리 | `docker system df` `image prune -f` `builder prune -f` | RECLAIMABLE 감소 | ☐ |
| 검증 스크립트 | `/usr/local/bin/check-part11.sh` | `ALL OK` | ☐ |
| 재부팅 후 자동 복원 | `reboot` → `check-part11.sh` | `lab-redis` 자동 기동 | ☐ |

---

## 기출 연결

> EXAM-PRACTICAL(실기 6회분)에는 Docker·가상화·`dpkg`/`apt` 직접 출제 문항이 없음 — 이 주제는 **EXAM-WRITTEN-FULL(필기 10회분)에 집중**되어 있으며, 10회분 전체에서 컨테이너·가상화 대역(83~90번)이 매 회차 3~5문항씩 고정 출제됨.

| 기출 (문제집·회차·번호 또는 주제) | 이 파트의 단계 |
| --- | --- |
| 필기 R01 #5 — 데비안 계열 배포판 고르기 | 6-1, 6-13 |
| 필기 R01 #44 — `.deb` 직접 설치 = `dpkg -i package.deb` | 6-9, 6-13 |
| 필기 R01 #89 — 커널 모듈 하이퍼바이저 + VT-x/AMD-V + QEMU 결합 = **KVM** (오답 OpenVZ·Docker·chroot) | 1-2, 1-4, 8-1 |
| 필기 R01 #90 — Docker 명령 설명 (`docker ps` = 실행 중 컨테이너 목록) | 4-3, 3-1 |
| 필기 R02 #4 — RPM 계열 배포판 짝짓기 | 6-1, 6-13 |
| 필기 R02 #41 — `.deb` 직접 설치 `dpkg -i pkg.deb` (오답 `apt-cache install`·`dpkg -r`) | 6-9, 6-10 |
| 필기 R02 #87 — 호스트 OS 위 응용 프로그램형 = **Type 2 하이퍼바이저 = VirtualBox** | 1-2 |
| 필기 R02 #88 — 게스트에 정상 종료 신호 = **`virsh shutdown vm1`** (오답 destroy·undefine·suspend) | 8-8 |
| 필기 R02 #89 — 실행 중 컨테이너에서 대화식 셸 = **`docker exec -it web bash`** | 4-4, 4-10 |
| 필기 R03 #84 — 반가상화는 커널 수정 + 하이퍼콜 → 전가상화보다 오버헤드 적음 | 1-1 |
| 필기 R03 #85 — 이미지=읽기전용 레이어, 컨테이너=쓰기 레이어, **named volume 은 컨테이너 삭제로 지워지지 않음** | 1-5, 4-5, 5-1 |
| 필기 R03 #86 — `docker ps -a` = 중지 포함 전체 (오답 `run -d` = 삭제) | 4-1, 4-3 |
| 필기 R03 #87 — 쿠버네티스 Pod(최소 배포 단위)·Service(고정 접근 지점) | 8-11 |
| 필기 R03 #88 — `virsh list --all` 은 정지 도메인 포함 / `virsh` 는 Docker 관리 도구 아님 | 8-3, 8-7, 8-11 |
| 필기 R04 #4 — 데비안 계열 배포판 | 6-1 |
| 필기 R04 #87 — `docker run -d -p 8080:80 --name web nginx` 해석 (호스트 8080 → 컨테이너 80, 백그라운드) | 4-1, 4-2 |
| 필기 R04 #88 — **`docker rmi` 는 이미지 삭제**(컨테이너 삭제 아님) | 3-3, 4-12 |
| 필기 R05 #4 — 배포판 ↔ 기본 패키지 도구 연결 (openSUSE — apt 는 오답) | 6-13 |
| 필기 R06 #86 — 다수 마이크로서비스·빠른 기동·게스트 커널 불필요 = **컨테이너** | 1-3 |
| 필기 R06 #87 — `docker run -d -p 8080:80 -v /srv/html:/usr/share/nginx/html nginx` | 4-1, 5-2, 5-3 |
| 필기 R06 #88 — 기동 상태·포트 매핑 목록 확인 = **`docker ps`** | 4-3 |
| 필기 R06 #89 — 커널 모듈 형태 + VT-x/AMD-V 활용 하이퍼바이저 = **KVM** | 1-2, 8-2 |
| 필기 R07 #5 — 데비안 계열만 고르기 | 6-1 |
| 필기 R07 #35 — `dpkg -i pkg.deb` | 6-9 |
| 필기 R07 #86 — KVM VM 생성 절차 (가상화 확인 → `qemu-img create` → `virt-install` → `virsh`) | 1-7, 8-2, 8-6, 8-9, 8-8 |
| 필기 R07 #87 — 전가상화(수정 불필요)·반가상화(커널 수정)·컨테이너(커널 공유 저오버헤드) / KVM 은 커널 모듈 필요 | 1-1, 1-3, 1-4 |
| 필기 R07 #88 — 컨테이너 라이프사이클 `pull → create → start → stop → rm` | 4-1, 4-9, 4-12 |
| 필기 R07 #89 — Dockerfile 작성 → `docker build -t app:1.0 .` → `docker images` → `docker run` | 7-1, 7-2, 7-3 |
| 필기 R08 #2 — `os-release` 의 `ID_LIKE=rhel` → `dnf`/`rpm` 계열 판정 | 6-1, 6-13 |
| 필기 R08 #83 — `docker ps` 출력 해석 (`0.0.0.0:8080->80/tcp`, `Up 2 hours` = 실행 중) | 4-2, 4-3 |
| 필기 R08 #84 — `docker images` 에서 **IMAGE ID 가 같으면 태그만 다른 동일 이미지** | 3-2, 3-3, 3-5 |
| 필기 R08 #85 — `virsh list --all` 의 `- db-vm shut off` = 정의 존재·정지 → `virsh start` 로 기동 | 8-7, 8-8 |
| 필기 R08 #86 — 반가상화 = 커널 수정 + 하이퍼콜, 에뮬레이션 오버헤드 적음 | 1-1 |
| 필기 R09 #4 — 보안 특화 배포판(Kali, 데비안 파생) | 6-1 |
| 필기 R09 #89 — VM 은 게스트 커널까지 분리 / 컨테이너는 **호스트 커널 공유(namespace·cgroup)** → 커널 취약점 전파 | 1-3, 1-4, 4-11 |
| 필기 R10 #4 — 우분투는 데비안 기반 파생 배포판 | 6-1 |
| 필기 R10 #87 — 반가상화 + **virtio 는 이 개념을 활용한 I/O 성능 향상 드라이버** | 1-1 |
| 필기 R10 #88 — **namespace 로 격리, cgroup 으로 자원 제한** (뒤바꾼 선지가 오답) | 1-4, 7-6 |
| 필기 R10 #89 — Dockerfile 에서 `RUN` = 빌드 시점 레이어 생성, `CMD` = 시작 시 기본 명령 / `EXPOSE` 만으로 호스트 포트 미개방 | 7-1, 7-4 |
| 필기 R10 #90 — 인프라 자원을 서비스로 제공 = **IaaS** | 1-2 |
| 실기 전 회차 — `systemctl enable --now <서비스>` 서비스 상시 기동 | 2-3, 7-7 |
| 실기 전 회차 — `ss -tlnp` 로 리스닝 포트·프로세스 확인 | 4-2, 9-3 |
| 실기 R02 #15 등 — SELinux 컨텍스트 확인·복구(`ls -Z`, `restorecon`) | 5-2 |
| 주제 — 저수준(`rpm`/`dpkg`) vs 고수준(`dnf`/`apt`) 의존성 자동 해결 차이 | 2-1, 6-9, 6-13 |
| 주제 — `apt update`(목록 갱신) ≠ `dnf update`(업그레이드) | 6-3, 6-13 |
| 주제 — 클러스터 유형(HPC·LVS·HA) | 8-11 |

---

## 이전 / 다음

[[10-security-firewall-selinux]] ← · → [[12-backup-recovery-review]]

- 허브: [[README]]
- 이론: [[../THEORY/network-service]] 7절(가상화·클러스터) · [[../THEORY/package-software]] 3절(dpkg/apt)
- 명령 문서: [[../../CONTAINER/docker]] · [[../../DATABASE/redis-cli]]
