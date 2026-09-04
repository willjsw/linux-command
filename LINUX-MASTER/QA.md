---
title: QA — 리눅스마스터 학습 질의응답 기록
type: exam-qa
category: LINUX-MASTER
tags:
  - exam/linux-master
  - exam/qa
  - moc/index
  - task/inspect
related: ["[[README]]", "[[LAB/README]]", "[[THEORY/user-permission]]", "[[THEORY/disk-device]]"]
distro: Rocky Linux 9 (aarch64, UTM)
updated: 2026-09-04
---

# QA — 리눅스마스터 학습 질의응답 기록

- 학습·실습 중 나온 질문과 해소된 궁금증을 누적 기록
- 작성 원칙 — **약어는 원어를 풀어서**, **옵션은 의미와 사용 목적을 함께** 서술
- 실습 절차서 [[LAB/README]] 진행 중 발생한 문제와 그 해결도 포함

---

## 목차

### A. UTM 실습 환경
- [[#A-1. ISO 이미지는 직접 받아야 하는가]]
- [[#A-2. UTM 이 기본 생성한 디스크는 삭제해야 하는가]]
- [[#A-3. `Display output is not active.` 가 뜨는 이유]]
- [[#A-4. 설치 후 ISO 를 제거하는 이유와 방법]]
- [[#A-5. 실습은 SSH 로 하는가 UTM 콘솔로 하는가]]
- [[#A-6. `unknown terminal type` 오류]]

### B. 디스크·파일시스템
- [[#B-1. `lsblk -f` 출력 전체 해설]]
- [[#B-2. 마운트란 무엇인가]]

### C. 계정·권한
- [[#C-1. wheel 그룹이란 무엇이며 sudo 와 어떤 관계인가]]

### D. 네트워크
- [[#D-1. DHCP 란 무엇인가]]
- [[#D-2. `dhcpd_leases` 에 응답 없는 IP 가 보이는 이유]]
- [[#D-3. `ip addr` 출력 전체 해설]]
- [[#D-4. `ip route` 출력 전체 해설]]

### E. 하드웨어·장치
- [[#E-1. PCI 버스란 무엇인가]]

---

# A. UTM 실습 환경

## A-1. ISO 이미지는 직접 받아야 하는가

**Q.** 실습에 쓸 Rocky Linux ISO 는 내가 직접 다운로드해야 하는가.

**A.** 그렇다. 두 개 파일이 필요하다.

| 파일 | 용도 |
| --- | --- |
| `Rocky-9.x-aarch64-minimal.iso` | 설치 매체 (약 2 GB) |
| `CHECKSUM` | 무결성 검증용 해시 목록 |

- **ISO** = **I**nternational **O**rganization for **S**tandardization 의 9660 규격 이미지 → 광학 디스크 내용을 파일 하나로 담은 형식
- **aarch64** = ARM 64비트 아키텍처 이름 (ARM 사의 AArch64 실행 상태에서 유래). Apple Silicon 맥이므로 x86_64 판이 아닌 이쪽
- **minimal** = GUI·부가 패키지를 뺀 최소 구성. 서버 실습용으로 적합

### 이미지 종류 구분 (시험 출제)

| 종류 | 내용 | 쓰임 |
| --- | --- | --- |
| Minimal | 최소 패키지만 포함 | 서버 설치 |
| DVD | 전체 패키지 포함 | **네트워크 없이** 설치할 때 |
| Boot | 부팅에 필요한 것만, 나머지는 네트워크로 | 대역폭 여유 있을 때 |

### 무결성 검증

```bash
shasum -a 256 -c CHECKSUM --ignore-missing
```

- `shasum` : macOS 기본 해시 계산 도구. 리눅스의 `sha256sum` 에 대응
- `-a 256` : **a**lgorithm 지정 → SHA-256 (**S**ecure **H**ash **A**lgorithm, 출력 256비트)
- `-c` : **c**heck — 직접 계산하는 대신 파일에 적힌 해시와 대조하는 검증 모드
- `--ignore-missing` : 목록에는 있으나 로컬에 없는 파일은 건너뜀 → CHECKSUM 에 DVD·Boot 판 해시도 함께 있으므로 필요

> MD5·SHA-1 은 충돌이 발견되어 무결성 검증에 부적합. SHA-256 이상 사용이 정답으로 출제

절차서 위치: [[LAB/01-vm-setup-and-inspection]] 1-0, 1-1

---

## A-2. UTM 이 기본 생성한 디스크는 삭제해야 하는가

**Q.** 드라이브 목록에 여러 개가 보이는데, 기본 생성된 것을 지워야 하는가. 크기가 `196KB` 로 나온다.

**A.** 삭제할 것은 없다. 그리고 `196KB` 는 용량이 아니다.

### 정상 구성

| 항목 | 개수 | 게스트 장치명 |
| --- | --- | --- |
| USB 드라이브 (CD/DVD) | 1 | `/dev/sr0` — ISO 연결용 |
| VirtIO 드라이브 | 5 | `/dev/vda` ~ `/dev/vde` |

- **VirtIO** = 가상화 환경 전용 반가상화(paravirtualized) 장치 규격. 실제 하드웨어를 흉내 내지 않고 게스트가 가상화를 인지한 채 통신 → 에뮬레이션보다 빠름
- VirtIO 디스크는 게스트에서 **`/dev/vd*`** 로 보임 (`vd` = **v**irtio **d**isk)

### `196KB` 의 정체

- **qcow2** = **Q**EMU **C**opy **O**n **W**rite version **2**
- 미리 자리를 잡지 않고 **실제로 쓴 만큼만 파일이 커지는** 형식 (thin provisioning)
- 갓 만든 5 GB 디스크도 파일 크기는 200 KB 안팎 → UTM 이 표시하는 값은 **파일의 현재 실제 사용량**
- 실제 용량은 게스트에서 확인

```bash
lsblk -d -o NAME,SIZE,TYPE
```

- `-d` : **d**isk 만 표시, 파티션은 숨김
- `-o` : **o**utput — 표시할 열을 직접 지정

### 추가 순서가 중요한 이유

추가한 순서대로 `vdb` → `vdc` → `vdd` → `vde` 로 이름이 붙는다. 순서가 어긋나면 [[LAB/05-disk-lvm-raid-swap-quota]] 의 `mkfs`·`mdadm` 명령이 **의도한 것과 다른 디스크를 대상으로 삼아** 데이터를 지운다.

---

## A-3. `Display output is not active.` 가 뜨는 이유

**Q.** VM 을 켰는데 화면에 이 메시지만 나온다.

**A.** UTM 의 기본 디스플레이 카드 `virtio-gpu-pci` 가 원인. **`virtio-ramfb`** 로 바꾼다.

### 각 디스플레이 카드의 동작 차이

| 카드 | 동작 | 결과 |
| --- | --- | --- |
| `virtio-gpu-pci` (기본값) | 게스트 커널의 virtio-gpu 드라이버가 올라온 뒤에야 출력 | UEFI·GRUB·초기 부팅이 **전부 검은 화면** |
| `ramfb` (단독) | 펌웨어가 초기화해 주어야 동작 | UTM 의 UEFI 가 초기화하지 않아 `Guest has not initialized the display (yet).` 에서 멈춤 |
| **`virtio-ramfb`** | ramfb + virtio-gpu 를 함께 제공 | 펌웨어 단계부터 출력. **이것을 선택** |

- **ramfb** = **RAM** **f**rame**b**uffer → 메모리 영역을 그대로 화면으로 쓰는 가장 단순한 방식. 전용 드라이버가 필요 없어 부팅 초기에 유용
- **PCI** = **P**eripheral **C**omponent **I**nterconnect → 주변장치 연결 버스 규격. 뒤에 `-pci` 가 붙으면 PCI 버스에 붙는 장치라는 뜻

### 남는 문제와 대응

`virtio-ramfb` 로도 커널이 화면 제어를 넘겨받는 시점에 출력이 끊길 수 있다. 그래서 **직렬 포트**를 함께 붙이고 커널이 그쪽으로도 출력하도록 지정한다.

```bash
sudo grubby --update-kernel=ALL --args="console=tty0 console=ttyAMA0,115200"
```

- `grubby` : GRUB 설정 파일을 직접 고치지 않고 부팅 항목을 다루는 도구. RHEL 9 표준
- `--update-kernel=ALL` : 설치된 **모든** 커널 항목에 적용. `DEFAULT` 는 기본 항목만
- `--args=` : 커널 명령줄(kernel command line)에 파라미터 추가. 제거는 `--remove-args=`
- `console=tty0` : **t**ele**ty**pewriter 0 → 그래픽 콘솔
- `console=ttyAMA0,115200` : **AMA** = **A**RM **A**MBA 계열 PL011 직렬 포트. `115200` 은 통신 속도(bps)
- `console=` 를 여러 번 쓰면 모두 출력하되 **마지막 것이 `/dev/console`** → 로그인 프롬프트가 뜨는 곳

> **부팅 성공 여부는 화면 없이도 판정 가능** — macOS 에서 게스트 IP 의 22번 포트가 열렸는지 확인하면 된다 (A-5 참조)

절차서 위치: [[LAB/01-vm-setup-and-inspection]] 1-2 ③④, 2-3

---

## A-4. 설치 후 ISO 를 제거하는 이유와 방법

**Q.** ISO 를 왜 빼야 하고, 어떻게 빼는가.

**A.** 빼지 않으면 **UEFI 가 디스크보다 CD 를 먼저 잡아** 설치 프로그램이 다시 뜬다.

- **UEFI** = **U**nified **E**xtensible **F**irmware **I**nterface → BIOS 를 대체한 펌웨어 규격. 부팅 항목을 NVMe 가 아닌 **NVRAM**(비휘발성 메모리)에 목록으로 저장하고 순서대로 시도
- 설치 메뉴에서 `Install…` 을 고르면 **방금 설치한 디스크를 처음부터 덮어쓴다**
- Rocky 9 의 `Troubleshooting` 하위에는 `Boot from local drive` 항목이 **없다** (구 CentOS 계열에만 존재)

### 제거 방법

| 상태 | 경로 |
| --- | --- |
| VM 실행 중 | 창 상단 툴바의 **디스크(CD) 아이콘** → 해당 드라이브 → **꺼내기** |
| VM 정지 | `srv01` 우클릭 → **편집** → 좌측 **드라이브** 그룹의 **`USB 드라이브`** → 이미지 비우고 **저장** |

### 한 번만 디스크로 부팅해 보려면

재시작 직후 GRUB 이 뜨기 **전에** `Esc` 를 여러 번 → UEFI 설정 → **Boot Manager** → 디스크 항목 선택.

- **GRUB** = **GR**and **U**nified **B**ootloader → 커널을 골라 메모리에 올리는 부트로더

절차서 위치: [[LAB/01-vm-setup-and-inspection]] 1-3

---

## A-5. 실습은 SSH 로 하는가 UTM 콘솔로 하는가

**Q.** 매번 로컬에서 SSH 로 붙어야 하는가, 아니면 UTM 콘솔을 쓰는가.

**A.** **기본은 SSH**, 콘솔은 정해진 몇 순간에만.

- **SSH** = **S**ecure **SH**ell → 암호화된 원격 셸 접속 프로토콜. 기본 포트 22
- UTM 콘솔은 복사·붙여넣기가 불편해 긴 명령을 다루기 어려움

### 접속 별칭 등록

```bash
cat >> ~/.ssh/config <<'EOF'

Host srv01
    HostName 192.168.64.3
    User admin1
EOF
chmod 600 ~/.ssh/config
```

- `Host` : 별칭. 이후 `ssh srv01` 만으로 접속
- `HostName` : 실제 주소. [[LAB/08-network-config]] 에서 고정 IP `192.168.64.10` 으로 바꾸면 이 값만 수정
- `User` : 생략 시 macOS 로그인 계정으로 시도하므로 지정 필요
- `chmod 600` : 소유자만 읽기·쓰기. 권한이 느슨하면 SSH 가 설정 파일을 무시

### 콘솔이 필요한 순간

| 상황 | 파트 |
| --- | --- |
| 고정 IP 전환, SSH 포트 2222 변경 | 08 |
| `rd.break` 로 root 비밀번호 복구 | 07 |
| `/etc/fstab` 오타 → emergency 모드 복구 | 07 |
| `iptables -P INPUT DROP` 등 방화벽 잠김 | 10 |
| 재부팅 후 자동 복원 검증 | 05 · 12 |

전부 **SSH 가 끊기거나 아직 안 뜬 상황**이다. 절차서에 ⚠️ 로 표시해 두었다.

### 게스트 IP 를 호스트에서 찾기

게스트 화면이 안 나올 때 유용하다.

```bash
cat /var/db/dhcpd_leases
ping -c1 192.168.64.3
nc -z -G2 192.168.64.3 22 && echo "sshd 응답"
```

- `/var/db/dhcpd_leases` : macOS 내장 **DHCP**(**D**ynamic **H**ost **C**onfiguration **P**rotocol) 서버의 임대 기록. `ip_address` 와 `hw_address`(MAC) 쌍이 저장됨
- UTM 설정의 **네트워크 → MAC 주소**와 대조해 어느 VM 인지 식별
- `nc` = **n**et**c**at → 임의의 TCP/UDP 연결을 만드는 도구
- `-z` : **z**ero-I/O — 데이터를 보내지 않고 **포트 개방 여부만** 확인. 포트 스캔·헬스체크 용도
- `-G2` : 연결 시도 제한 2초 (macOS 판 `nc` 의 timeout 옵션)
- **22번이 열려 있으면 부팅이 끝나고 `sshd` 까지 올라온 것** → 화면이 검어도 서버는 정상

절차서 위치: [[LAB/01-vm-setup-and-inspection]] 2-1, 2-2

---

## A-6. `unknown terminal type` 오류

**Q.** SSH 접속 후 `clear` 를 치니 `'xterm-ghostty': unknown terminal type.` 이 나온다.

**A.** 클라이언트가 보낸 `TERM` 값에 해당하는 **terminfo 정의가 서버에 없다.**

- **TERM** : 터미널 종류를 알려주는 환경변수. 화면 제어 프로그램이 이 값으로 정의를 찾음
- **terminfo** = **term**inal **info**rmation → 터미널별 커서 이동·색상·화면 지우기 제어 문자열을 담은 데이터베이스. `/usr/share/terminfo/` 에 저장 (구형 유닉스는 `/etc/termcap`)
- Ghostty·Kitty·WezTerm 등 최신 터미널은 고유 `TERM` 값을 쓰는데, 서버에 그 정의가 없으면 실패
- `clear` 뿐 아니라 `vi`·`top`·`less`·`nmtui` 등 **화면을 그리는 모든 프로그램**이 같은 이유로 실패

### 해결 — terminfo 를 서버로 전송 (권장)

```bash
infocmp -x | ssh srv01 -- tic -x -
```

- `infocmp` = **info**rmation **comp**are → terminfo 정의를 사람이 읽을 수 있는 텍스트로 출력. 인자를 생략하면 현재 `$TERM` 대상
- `-x` : e**x**tended — 확장 기능(capability)까지 포함. 생략하면 최신 터미널의 기능 일부가 누락
- `tic` = **t**erminfo **c**ompiler → 텍스트 정의를 바이너리로 컴파일해 설치
- `tic -x -` 의 `-` : 파일 대신 **표준 입력**에서 읽으라는 관례적 표기
- 서버의 `~/.terminfo/` 에 저장 → **root 권한 불필요**, 해당 사용자에게만 적용
- 서버에 `tic` 이 없으면 `sudo dnf install -y ncurses` 로 설치

### 간단한 대안 — TERM 고정

```bash
echo 'export TERM=xterm-256color' >> ~/.bashrc
source ~/.bashrc
```

터미널 고유 기능은 포기하지만 즉시 해결된다.

### 검증

```bash
echo $TERM
clear && echo "clear 정상"
tput cols; tput lines
```

- `tput` = **t**ermina**l** **put**capabilities → terminfo 를 조회해 제어 문자열이나 값을 출력
- `cols`/`lines` : 터미널의 가로·세로 크기. 숫자가 나오면 정의를 제대로 읽고 있다는 뜻

절차서 위치: [[LAB/01-vm-setup-and-inspection]] 2-2

---

# B. 디스크·파일시스템

## B-1. `lsblk -f` 출력 전체 해설

**Q.** 아래 출력의 각 줄과 열이 무엇을 뜻하는가.

```text
NAME         FSTYPE      FSVER    LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
vda
├─vda1       vfat        FAT32          BCAB-0CFA                                 591M     1% /boot/efi
├─vda2       xfs                        8f1e0e0e-55ee-4677-b9af-58620a93c8dc    611.7M    36% /boot
└─vda3       LVM2_member LVM2 001       Z9mmyu-7t7J-BhHN-LG55-LAu5-usaW-bh0iXM
  ├─rlm-root xfs                        a8b0aa6a-54dd-4a91-abdb-5c517fd30b64     32.6G     5% /
  └─rlm-swap swap        1              5ad5f15b-4874-40e7-a105-c8b4eb329b34                  [SWAP]
```

**A.** 명령과 열, 그리고 각 줄을 차례로 본다.

### 명령

- `lsblk` = **l**i**s**t **bl**ock devices → 블록 장치(디스크·파티션·논리볼륨)를 트리로 나열
  - **블록 장치** : 데이터를 정해진 크기 덩어리(블록) 단위로 읽고 쓰는 장치. 문자 장치(키보드·터미널)와 구분
- `-f` : **f**ilesystem — 파일시스템 종류·라벨·UUID·사용량·마운트 지점을 함께 표시
  - 기본 `lsblk` 는 장치 이름과 크기만 → 포맷 여부와 마운트 상태를 보려면 `-f` 필요

### 열의 의미

| 열 | 원어 | 뜻 |
| --- | --- | --- |
| `NAME` | name | 장치 이름. 트리 기호로 포함 관계 표시 |
| `FSTYPE` | **f**ile**s**ystem **type** | 파일시스템 종류. 비어 있으면 미포맷 |
| `FSVER` | **f**ile**s**ystem **ver**sion | 파일시스템 버전 |
| `LABEL` | label | 사람이 붙인 이름표. 미지정 시 공란 |
| `UUID` | **U**niversally **U**nique **ID**entifier | 전 세계적으로 겹치지 않는 식별자 |
| `FSAVAIL` | **f**ile**s**ystem **avail**able | 남은 공간 |
| `FSUSE%` | **f**ile**s**ystem **use** percent | 사용률 |
| `MOUNTPOINTS` | mount points | 마운트된 경로. 공란이면 미마운트 |

### 트리 구조

```
vda                     ← 물리 디스크 40 GB (자체는 FSTYPE 없음)
├─vda1  vfat            ← 1번 파티션
├─vda2  xfs             ← 2번 파티션
└─vda3  LVM2_member     ← 3번 파티션 (LVM 에 편입)
  ├─rlm-root  xfs       ← vda3 위에 얹힌 논리 볼륨
  └─rlm-swap  swap
```

`vda` 자체에 `FSTYPE` 이 없는 이유는 디스크를 통째로 포맷한 것이 아니라 **파티션으로 나눈 뒤 각 파티션을 포맷**했기 때문. 파티션 테이블은 파일시스템이 아니다.

### 줄별 해설

**`vda1` — EFI 시스템 파티션 (ESP)**

- `vfat` = **v**irtual **F**ile **A**llocation **T**able → 리눅스 커널의 FAT 계열 파일시스템 드라이버 이름
- `FAT32` : 32비트 클러스터 주소를 쓰는 FAT 판본
- UEFI 펌웨어가 부트로더(`shimaa64.efi`, `grubaa64.efi`)를 읽어가는 곳
- **펌웨어가 읽을 수 있는 파일시스템이 FAT 뿐**이라 이 파티션만 예외적으로 `vfat`
- UUID 가 `BCAB-0CFA` 로 유독 짧은 이유 — FAT 은 진짜 UUID 가 아니라 32비트 **볼륨 일련번호**를 쓰기 때문
- 사용률 1% 로 거의 비어 있는 것이 정상

**`vda2` — 부트 파티션**

- `xfs` : 대용량·고성능에 강한 저널링 파일시스템. RHEL 계열 기본값
  - **저널링** : 변경 내용을 먼저 기록(journal)한 뒤 실제 반영 → 갑작스러운 정전에도 복구 가능
- 커널(`vmlinuz`)과 초기 램디스크(`initramfs`)가 위치
- **LVM 바깥에 따로 두는 이유** : 부팅 초기에는 아직 LVM 을 다룰 수 없음
- 36% 사용 중 — 커널 업데이트가 쌓이면 가득 차는 일이 잦음 ([[LAB/02-package-management]] 에서 정리 방법)

**`vda3` — LVM 물리 볼륨**

- `LVM2_member` : 이 파티션이 **LVM 의 물리 볼륨(PV)으로 편입된 상태**라는 표시
- **LVM** = **L**ogical **V**olume **M**anager → 여러 디스크를 묶어 유연하게 나눠 쓰는 계층
- `LVM2 001` : LVM 메타데이터 형식 버전
- 파일시스템이 아니므로 **마운트 대상이 아님** → `MOUNTPOINTS` 공란
- UUID 형태가 다른 이유 — 일반 파일시스템 UUID 가 아니라 **PV UUID** 이기 때문

**`rlm-root` — 루트 논리 볼륨**

- `rlm` 이 **볼륨 그룹(VG)** 이름, `root` 가 **논리 볼륨(LV)** 이름
  - VG 이름은 설치 매체에서 유래 — Minimal ISO 는 `rlm`, DVD ISO 는 `rl`
- `/` 에 마운트 → 시스템 전체가 여기에 위치
- 논리 볼륨이라 디스크를 추가해 **온라인 확장 가능** ([[LAB/05-disk-lvm-raid-swap-quota]])

**`rlm-swap` — 스왑 볼륨**

- `MOUNTPOINTS` 가 `[SWAP]` — 대괄호는 **디렉터리 마운트가 아니라 커널이 스왑 영역으로 활성화**했다는 표시
- 파일을 담는 공간이 아니므로 `FSAVAIL`·`FSUSE%` 공란
- 활성화 명령도 `mount` 가 아니라 `swapon`
- 사용량은 `free -h` 또는 `swapon --show` 로 확인

### 계층 구조 요약

```
물리 디스크(vda) → 파티션(vda3) → PV → VG(rlm) → LV(rlm-root) → 파일시스템(xfs) → 마운트(/)
```

### 시험 포인트

- `UUID` 는 [[LAB/05-disk-lvm-raid-swap-quota]] 의 `/etc/fstab` 등록에 사용. 장치 이름은 인식 순서에 따라 바뀌지만 UUID 는 고정이라 `UUID=` 표기가 표준
- `vda1` 이 `vfat` 인 이유(펌웨어가 FAT 만 읽음), `/boot` 가 LVM 밖에 있는 이유, `[SWAP]` 표기의 의미가 빈출
- `LVM2_member` 를 보고 "이 파티션은 LVM PV" 로 읽어내는 출력 해석 문제 출제

---

## B-2. 마운트란 무엇인가

**Q.** 마운트되어 있다는 것은 리눅스에서 파일시스템을 통해 그 디스크에 접근할 수 있다는 뜻인가.

**A.** 맞다. 정확히는 **파일 단위로** 접근할 수 있게 된다는 뜻이다.

### 왜 필요한가

- 윈도우는 디스크마다 `C:` `D:` 를 붙여 **별개의 트리**를 만듦
- 리눅스는 **디렉터리 트리가 `/` 하나뿐** → 모든 저장 장치는 그 트리 어딘가에 붙여야 경로로 접근 가능
- **mount** : 파일시스템을 디렉터리 트리의 특정 지점에 연결하는 작업

```
/                    ← rlm-root (xfs)
├── boot             ← vda2 (xfs)      여기부터 다른 장치
│   └── efi          ← vda1 (vfat)     여기부터 또 다른 장치
└── ...
```

`/boot/config-…` 를 열면 커널이 "`/boot` 부터는 `vda2` 담당" 임을 알고 그쪽 파일시스템에서 찾는다. 사용자는 어느 디스크인지 의식할 필요가 없다.

### 마운트 포인트는 평범한 디렉터리

- `/boot` 는 특별한 무언가가 아니라 그냥 빈 디렉터리
- 거기에 `vda2` 를 마운트하면 그 순간부터 `vda2` 의 내용이 보임
- **원래 있던 파일은 지워지는 것이 아니라 가려진다** → 마운트 해제 시 다시 나타남
- 파일이 있는 디렉터리에 마운트해 "파일이 사라졌다" 고 착각하는 사고가 흔함

### 마운트 안 된 장치는 접근이 불가능한가

아니다. 이 구분이 핵심이다.

| 접근 방식 | 마운트 | 예 |
| --- | --- | --- |
| 파일 단위 | **필요** | `cat`, `ls`, `cp` |
| 블록 단위 | 불필요 | `dd if=/dev/vda3 of=backup.img`, `fdisk /dev/vda` |

- `vda3` 는 마운트되지 않아 파일로는 못 읽지만, `/dev/vda3` 장치 파일을 통해 **바이트 덩어리로는** 읽고 쓸 수 있음
- [[LAB/12-backup-recovery-review]] 의 `dd` 백업이 이 방식 — 파일시스템을 거치지 않으므로 안에 무슨 파일이 있는지 모른 채 통째로 복사
- `vda3` 가 마운트되지 않은 진짜 이유는 별개 — 파일시스템이 아니라 LVM PV 라 마운트 대상 자체가 없음

### `[SWAP]` 은 마운트가 아니다

- 디렉터리에 붙는 것이 아니라 **커널이 메모리 부족 시 쓰는 예비 공간**으로 등록된 상태
- 경로로 접근 불가 → `FSAVAIL` 공란
- 활성화 명령이 `swapon`, 해제가 `swapoff`

### 확인 명령

```bash
findmnt                          # 마운트 관계를 트리로
findmnt /boot/efi                # 특정 경로가 어느 장치인지
stat -c '%m  %n' /boot/efi/EFI   # 이 파일이 속한 마운트 포인트
df -hT /boot /boot/efi           # 두 경로가 서로 다른 파일시스템임을 확인
cat /proc/mounts | head          # 커널이 인식 중인 마운트 목록 원본
```

- `findmnt` = **find** **m**ou**nt** → 마운트 정보를 트리로 조회. `mount` 명령의 나열보다 구조 파악에 유리
- `stat` : 파일의 메타데이터 조회
  - `-c` : **c**ustom format — 출력 형식 지정
  - `%m` : 그 파일이 속한 **m**ount point
  - `%n` : 파일 이름(**n**ame)
- `df` = **d**isk **f**ree → 파일시스템별 사용량
  - `-h` : **h**uman-readable — `1048576` 대신 `1M` 처럼 읽기 쉬운 단위
  - `-T` : 파일시스템 **T**ype 열 추가
- `/proc/mounts` : 커널이 관리하는 실제 마운트 목록. `/etc/mtab` 은 이 파일을 가리키는 심볼릭 링크

실습 위치: [[LAB/05-disk-lvm-raid-swap-quota]] 3~4절

---

# C. 계정·권한

## C-1. wheel 그룹이란 무엇이며 sudo 와 어떤 관계인가

**Q.** wheel 그룹이 무엇이고 sudo 와 어떤 관계인가.

**A.** wheel 은 **관리자 권한을 위임받을 사용자를 모아 두는 그룹**이고, sudo 는 **그 위임을 실제로 집행하는 명령**이다. 둘은 별개이며 `/etc/sudoers` 설정으로 이어져 있다.

### 이름의 유래

- **wheel** : "big wheel" 이라는 영어 속어에서 유래. **조직의 실세·거물**을 뜻함
- 1980년대 유닉스에서 `su` 를 아무나 실행하지 못하도록 제한하면서, 허용된 사용자만 담는 그룹으로 관습화
- 리눅스에서는 **GID 10** 으로 고정 (**GID** = **G**roup **ID**entifier)

```bash
getent group wheel
```

```text
wheel:x:10:admin1
```

- `getent` = **get** **ent**ries → NSS(**N**ame **S**ervice **S**witch)를 통해 사용자·그룹 정보를 조회
  - `/etc/group` 을 직접 `cat` 하는 것과 달리 **LDAP·NIS 등 외부 소스까지 포함**해 조회하므로 더 정확
- 출력 4필드 : `그룹명 : 비밀번호(x=gshadow 로 분리) : GID : 소속 사용자 목록`

### wheel 그룹 자체에는 아무 권한이 없다

여기가 핵심이다. wheel 에 속했다는 사실만으로 얻는 권한은 **하나도 없다**. 리눅스 커널은 wheel 을 특별 취급하지 않는다. 오직 **설정 파일이 그 그룹을 언급할 때만** 의미가 생긴다.

wheel 을 참조하는 곳은 두 군데다.

| 설정 파일 | 효과 |
| --- | --- |
| `/etc/sudoers` | wheel 구성원에게 `sudo` 사용 허가 |
| `/etc/pam.d/su` | (설정 시) wheel 구성원만 `su` 실행 허용 |

### `/etc/sudoers` 의 해당 줄

```bash
sudo grep -E '^%wheel' /etc/sudoers
```

```text
%wheel	ALL=(ALL)	ALL
```

각 항목의 의미는 이렇다.

| 위치 | 값 | 뜻 |
| --- | --- | --- |
| `%wheel` | 대상 | `%` 는 **그룹**을 의미. `%` 없으면 개별 사용자 |
| `ALL=` | 호스트 | 어느 호스트에서 접속했든 적용 (한 sudoers 를 여러 서버가 공유하던 시절의 흔적) |
| `(ALL)` | 실행 주체 | 어떤 사용자로든 실행 가능. `(root)` 로 좁힐 수 있음 |
| 마지막 `ALL` | 명령 | 모든 명령 허용. 여기에 명령 목록을 적어 제한 가능 |

즉 이 한 줄이 "wheel 그룹에 속한 사람은 어디서 접속하든 어떤 사용자 자격으로든 모든 명령을 실행할 수 있다" 는 위임 규칙이다. **이 줄을 지우면 wheel 은 아무 의미 없는 그룹이 된다.**

### 설치할 때 "관리자로 지정" 의 정체

Anaconda 설치 프로그램의 **Make this user administrator** 체크박스가 하는 일은 단 하나, **그 계정을 wheel 그룹에 넣는 것**이다.

```bash
id admin1
```

```text
uid=1000(admin1) gid=1000(admin1) groups=1000(admin1),10(wheel)
```

- `id` : 사용자의 UID·GID·소속 그룹 조회
- `uid=1000` : 일반 사용자 시작 번호. **UID** = **U**ser **ID**entifier
- `gid=1000(admin1)` : **주 그룹**(primary group). RHEL 계열은 사용자와 같은 이름의 그룹을 자동 생성 — **UPG**(**U**ser **P**rivate **G**roup) 방식
- `groups=…,10(wheel)` : **보조 그룹**(supplementary group) 목록에 wheel 포함

### su 와 sudo 의 차이

| 항목 | `su` | `sudo` |
| --- | --- | --- |
| 원어 | **s**witch **u**ser | **s**uper**u**ser **do** (현재는 substitute user do) |
| 입력할 비밀번호 | **대상 사용자**(보통 root)의 것 | **자기 자신**의 것 |
| root 비밀번호 공유 | 필요 → 여러 명이 알게 됨 | 불필요 |
| 권한 범위 | 전부 (셸을 통째로 얻음) | `/etc/sudoers` 에 적힌 만큼만 |
| 로그 | 전환 사실만 기록 | **실행한 명령까지** `/var/log/secure` 에 기록 |
| 지속 시간 | `exit` 할 때까지 | 명령 하나 단위 (기본 5분간 재입력 면제) |

**보안상 sudo 가 권장되는 이유**가 여기서 나온다. root 비밀번호를 나눠 갖지 않아도 되고, 사람마다 허용 명령을 다르게 줄 수 있으며, 누가 무엇을 했는지 추적된다.

```bash
sudo -l
```

- `-l` : **l**ist — 현재 사용자가 sudo 로 실행할 수 있는 명령 목록 조회
- 자기 권한을 확인할 때 가장 먼저 쓰는 옵션

### `/etc/pam.d/su` — wheel 로 su 를 제한하기

기본값은 **비활성**이라, wheel 이 아니어도 root 비밀번호만 알면 `su` 가 된다. 아래 줄의 주석을 풀면 wheel 구성원만 허용된다.

```text
#auth		required	pam_wheel.so use_uid
```

- **PAM** = **P**luggable **A**uthentication **M**odules → 인증 절차를 모듈 단위로 갈아 끼우는 체계
- `auth` : 인증 단계에서 동작 (그 외 `account`·`password`·`session`)
- `required` : 이 모듈이 실패하면 최종 실패. 단, **나머지 모듈도 끝까지 실행**한 뒤 결과 통보 (어떤 단계에서 막혔는지 숨기기 위함)
- `pam_wheel.so` : 호출자가 wheel 그룹인지 검사하는 모듈
- `use_uid` : 로그인 이름이 아니라 **현재 실효 UID**를 기준으로 판정 → `su` 로 여러 번 전환한 뒤에도 정확

실습 위치: [[LAB/03-user-group-permission]] 5-4, 7절

### 데비안 계열과의 차이

| 배포판 | 관리자 그룹 |
| --- | --- |
| RHEL 계열 (Rocky, CentOS, Fedora) | `wheel` (GID 10) |
| 데비안 계열 (Ubuntu, Debian) | `sudo` (그룹 이름이 sudo) |

시험은 RHEL 계열 기준이므로 **wheel** 이 정답. 데비안에서는 `usermod -aG sudo <user>` 로 같은 일을 한다.

### 관련 명령 정리

```bash
usermod -aG wheel dev1        # wheel 에 추가
gpasswd -a dev1 wheel         # 같은 작업, 그룹 관리 도구 쪽
gpasswd -d dev1 wheel         # 제거
groups dev1                   # 소속 그룹 확인
visudo                        # sudoers 편집 (문법 검사 후 저장)
visudo -c                     # 문법만 검사
```

- `usermod` : 사용자 계정 속성 수정
  - `-a` : **a**ppend — 기존 보조 그룹에 **추가**. **이 옵션을 빠뜨리면 기존 보조 그룹이 전부 사라진다** (사고 단골)
  - `-G` : 보조 **G**roup 목록 지정 (대문자). 소문자 `-g` 는 주 그룹이므로 혼동 주의
- `gpasswd` : 그룹 관리 전용 도구
  - `-a` : **a**dd, `-d` : **d**elete
- `visudo` : `/etc/sudoers` 전용 편집기
  - 저장 시 **문법을 검사**해 오류가 있으면 반려 → 잘못된 문법으로 sudo 가 통째로 막히는 사고 방지
  - `sudoers` 를 `vi` 로 직접 여는 것은 금기
  - `-c` : **c**heck — 편집 없이 문법만 검사

> 📝 **시험 포인트** : "일반 사용자에게 관리자 권한을 주는 방법" → `usermod -aG wheel <user>`. `-a` 누락 시 부작용, `%wheel ALL=(ALL) ALL` 의 4개 필드 해석, `su` 와 `sudo` 의 비밀번호 차이가 빈출

---

# D. 네트워크

## D-1. DHCP 란 무엇인가

**Q.** DHCP 란 무엇인가.

**A.** **D**ynamic **H**ost **C**onfiguration **P**rotocol — 네트워크에 접속한 장비에게 **IP 주소를 비롯한 네트워크 설정을 자동으로 나눠 주는 프로토콜**이다.

### 왜 필요한가

IP 통신을 하려면 최소 네 가지를 알아야 한다.

| 항목 | 없으면 생기는 일 |
| --- | --- |
| IP 주소 | 자기 주소가 없어 통신 자체 불가 |
| 서브넷 마스크 | 어디까지가 같은 네트워크인지 판단 불가 |
| 기본 게이트웨이 | 외부 네트워크로 나가지 못함 |
| DNS 서버 | 도메인 이름을 주소로 바꾸지 못함 |

이걸 장비마다 손으로 넣으면 대수가 늘수록 관리가 불가능해지고, 같은 주소를 두 대에 주는 **IP 충돌**이 발생한다. DHCP 는 이 배포와 회수를 서버가 중앙에서 관리한다.

- 전신은 **BOOTP**(**BOOT**strap **P**rotocol) — 고정 할당만 가능했던 것을 동적 할당까지 확장한 것이 DHCP

### 포트 (시험 빈출)

| 역할 | 포트 | 프로토콜 |
| --- | --- | --- |
| 서버 | **67** | UDP |
| 클라이언트 | **68** | UDP |

- **UDP** = **U**ser **D**atagram **P**rotocol → 연결 수립 없이 보내는 방식
- 아직 IP 가 없는 상태에서 통신을 시작해야 하므로, 연결 수립이 필요한 TCP 를 쓸 수 없음
- 첫 요청은 목적지를 모르므로 **브로드캐스트**(`255.255.255.255`)로 발신

### 할당 절차 — DORA (순서 암기)

| 순서 | 메시지 | 방향 | 내용 |
| --- | --- | --- | --- |
| **D** | DHCP**DISCOVER** | 클라이언트 → 브로드캐스트 | "DHCP 서버 있습니까" |
| **O** | DHCP**OFFER** | 서버 → 클라이언트 | "이 주소 쓰시겠습니까" |
| **R** | DHCP**REQUEST** | 클라이언트 → 브로드캐스트 | "그 주소 쓰겠습니다" |
| **A** | DHCP**ACK** | 서버 → 클라이언트 | "확정. 임대 기간은 N초" |

- REQUEST 를 다시 브로드캐스트하는 이유 — 서버가 여러 대일 때 **선택받지 못한 서버가 제안을 회수**하도록 알리기 위함
- **ACK** = **ACK**nowledgement(확인 응답). 거절은 **NAK**(**N**egative **ACK**)
- 반납은 DHCP**RELEASE**, 이미 쓰이는 주소면 DHCP**DECLINE**

### 임대(lease) 개념

DHCP 는 주소를 **영구히 주지 않고 기한을 정해 빌려준다**. 기기가 사라져도 주소가 회수되도록 하기 위함이다.

| 시점 | 동작 |
| --- | --- |
| 임대 시간의 50% (T1) | 클라이언트가 갱신(renew) 요청 |
| 임대 시간의 87.5% (T2) | 갱신 실패 시 다른 서버에도 요청 |
| 만료 | 주소 반납 후 DORA 재시작 |

- 갱신이 성공하면 **같은 주소를 계속 유지**하므로, DHCP 라도 주소가 자주 바뀌지는 않음
- DHCP 서버를 못 찾으면 **APIPA**(**A**utomatic **P**rivate **IP** **A**ddressing)로 `169.254.0.0/16` 대역의 주소를 스스로 붙임 → 이 주소가 보이면 **DHCP 실패 신호**

### 실습 환경에서의 DHCP

UTM 의 Shared Network 는 macOS 의 `vmnet` 공유 네트워크를 쓰며, **macOS 가 DHCP 서버 역할**을 한다.

```bash
cat /var/db/dhcpd_leases
```

```text
{
	ip_address=192.168.64.3
	hw_address=1,d6:7:47:f3:e7:35
	lease=0x6a9a5277
}
```

- `/var/db/dhcpd_leases` : macOS 내장 DHCP 서버의 임대 기록
- `hw_address` : **h**ard**w**are address = **MAC**(**M**edia **A**ccess **C**ontrol) 주소. 앞의 `1,` 은 하드웨어 유형(1 = 이더넷)
- `lease` : 만료 시각을 16진수 유닉스 시각으로 기록
- 게이트웨이 겸 DNS 는 `192.168.64.1`(호스트)

게스트 쪽에서 확인하는 명령이다.

```bash
nmcli connection show enp0s1 | grep -i dhcp
ip -4 addr show enp0s1
cat /etc/resolv.conf
```

- `nmcli` = **N**etwork**M**anager **c**ommand **l**ine **i**nterface
- RHEL 9 는 NetworkManager 가 **내장 DHCP 클라이언트**를 사용 (구형은 별도 `dhclient` 데몬)
- 받아온 값이 `/etc/resolv.conf` 에 자동 반영됨 → 파일 상단에 "수동 편집 금지" 주석이 붙는 이유

### 고정 IP 로 바꾸는 이유

서버는 주소가 바뀌면 곤란하다. 클라이언트가 찾아올 주소가 흔들리고, DNS·방화벽 규칙이 어긋난다. 그래서 [[LAB/08-network-config]] 에서 `192.168.64.10` 으로 고정한다.

```bash
nmcli con mod enp0s1 ipv4.method manual ipv4.addresses 192.168.64.10/24 \
  ipv4.gateway 192.168.64.1 ipv4.dns "192.168.64.1 8.8.8.8"
```

- `ipv4.method manual` : DHCP(`auto`) 대신 **수동 지정**으로 전환. 이 값을 바꾸지 않으면 나머지 설정이 무시됨
- `/24` : 서브넷 마스크를 **CIDR**(**C**lassless **I**nter-**D**omain **R**outing) 표기로. `255.255.255.0` 과 동일
- 고정 IP 를 쓸 때는 **DHCP 풀 범위 밖**의 주소를 골라야 충돌이 없음

### DHCP 서버 구축 (시험 범위)

RHEL 계열의 설정 파일과 주요 지시자다.

| 항목 | 값 |
| --- | --- |
| 패키지 | `dhcp-server` |
| 데몬 | `dhcpd` |
| 설정 파일 | `/etc/dhcp/dhcpd.conf` |
| 임대 기록 | `/var/lib/dhcpd/dhcpd.leases` |

```text
subnet 192.168.64.0 netmask 255.255.255.0 {
    range 192.168.64.100 192.168.64.200;
    option routers 192.168.64.1;
    option domain-name-servers 192.168.64.1;
    default-lease-time 3600;
    max-lease-time 7200;
}

host printer01 {
    hardware ethernet 00:11:22:33:44:55;
    fixed-address 192.168.64.50;
}
```

- `subnet … netmask …` : 이 서버가 관리할 네트워크 대역 선언
- `range` : 동적으로 나눠 줄 주소 범위. **이 범위 밖 주소는 고정 IP 용으로 남겨 둠**
- `option routers` : 클라이언트에 알려 줄 기본 게이트웨이
- `option domain-name-servers` : 알려 줄 DNS 서버
- `default-lease-time` : 클라이언트가 기간을 요청하지 않을 때 적용할 임대 시간(초)
- `max-lease-time` : 클라이언트가 요청하더라도 넘길 수 없는 상한(초)
- `host` 블록 : MAC 주소를 보고 **항상 같은 주소를 주는 고정 할당**(DHCP 예약). 프린터·서버처럼 주소가 고정돼야 하는 장비에 사용

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

- `-t` : **t**est — 서비스를 띄우지 않고 **설정 문법만 검사**. 잘못된 설정으로 서비스가 죽는 것을 막기 위해 항상 선행
- `-cf` : **c**onfig **f**ile 경로 지정

> ⚠️ 실습 VM 에서 `dhcpd` 를 **실제로 기동하면 UTM 의 NAT DHCP 와 충돌**해 네트워크가 끊긴다 → [[LAB/09-network-services]] 에서는 문법 검사까지만 수행

### 시험 포인트

- 포트 **67(서버) / 68(클라이언트) UDP** — 숫자와 방향을 바꿔 낸 선지가 오답
- **DORA 순서** — Discover → Offer → Request → Ack
- `range` 와 `fixed-address` 의 역할 구분
- `default-lease-time` 과 `max-lease-time` 의 차이
- `169.254.x.x` 주소가 보이면 DHCP 서버 응답 실패
- DHCP 는 **브로드캐스트**를 쓰므로 라우터를 넘지 못함 → 다른 서브넷에 서버가 있으면 **DHCP 릴레이 에이전트**(`dhcrelay`) 필요

관련 문서: [[THEORY/network-basics]] · [[THEORY/network-service]] · [[LAB/08-network-config]] · [[LAB/09-network-services]]

---

## D-2. `dhcpd_leases` 에 응답 없는 IP 가 보이는 이유

**Q.** `/var/db/dhcpd_leases` 에 `192.168.64.2` 와 `192.168.64.3` 두 개가 나오는데, ping 이 되는 것은 `.3` 뿐이다. `.2` 는 왜 목록에 있으며, 이 파일은 무엇을 위한 것인가.

**A.** 이 파일은 **현재 켜져 있는 장비 목록이 아니라, 과거에 나눠 준 주소의 장부**다. 응답이 없는 것은 그 장비가 꺼져 있기 때문이다.

### 실제 확인 결과

| IP | MAC | 소속 | 임대 만료 | 상태 |
| --- | --- | --- | --- | --- |
| `192.168.64.3` | `d6:07:47:f3:e7:35` | `srv01` (실습 VM) | 2026-09-04 14:17 | 실행 중 → 응답 |
| `192.168.64.2` | `aa:f8:9d:9a:26:c8` | `Red Star OS 2.0` (다른 VM) | 2026-07-09 02:04 | 정지 → 무응답 |

`.2` 는 **약 2개월 전에 한 번 켰던 다른 VM** 의 기록이다. 그 VM 은 지금 꺼져 있으니 ping 에 응답할 주체가 없다.

MAC 주소로 대조하면 어느 VM 인지 특정할 수 있다.

```bash
# macOS — UTM 이 각 VM 에 부여한 MAC 조회
plutil -extract Network json -o - ~/Library/Containers/com.utmapp.UTM/Data/Documents/srv01.utm/config.plist
```

- `plutil` = **p**roperty **l**ist **util**ity → macOS 의 `.plist`(설정 파일) 조회·변환 도구
- `-extract <키경로>` : 특정 키만 뽑아냄
- `-o -` : **o**utput 을 파일이 아닌 **표준 출력**으로 (`-` 는 표준 입출력을 뜻하는 유닉스 관례)

### 임대 만료 시각 읽기

`lease=0x6a9a5464` 의 16진수는 **유닉스 시각**(1970-01-01 00:00:00 UTC 부터의 초)이다.

```bash
python3 -c "import datetime; print(datetime.datetime.fromtimestamp(0x6a9a5464))"
# 또는 macOS 의 date
date -r $((0x6a9a5464))
```

- `date -r <초>` : macOS 판 `date` 의 옵션. 유닉스 시각을 사람이 읽는 형식으로 변환
- 리눅스(GNU date)는 같은 일을 `date -d @<초>` 로 수행 → **배포판별 옵션 차이로 출제 가능**

만료 시각이 **이미 지난** 항목은 죽은 기록이며, 서버가 그 주소를 다른 장비에 재할당할 수 있다.

### 이 파일의 목적

macOS 내장 DHCP 서버(`bootpd`)가 상태를 **디스크에 남겨 두는 이유**는 세 가지다.

| 목적 | 설명 |
| --- | --- |
| 주소 연속성 | 같은 MAC 이 다시 접속하면 **이전과 같은 IP** 를 돌려줌 → VM 을 껐다 켜도 주소가 유지 |
| 중복 방지 | 이미 빌려준 주소를 다른 장비에 주지 않도록 기록 |
| 재시작 후 복구 | 호스트나 DHCP 데몬이 재시작해도 임대 상태를 잃지 않음 |

- 메모리에만 두면 재시작 시 전부 잊어버려 IP 충돌이 발생하므로 **파일로 영속화**
- **DHCP 서버 쪽 장부**이므로, 클라이언트가 살아 있는지 여부는 기록하지 않음 → 생존 확인은 별도 수단 필요

### 살아 있는지 확인하는 방법

```bash
ping -c1 -W1000 192.168.64.3          # ICMP 응답
nc -z -G2 192.168.64.3 22             # 22번 포트 개방 여부
arp -n 192.168.64.3                   # ARP 캐시에 MAC 이 잡히는지
```

- `ping` : **ICMP**(**I**nternet **C**ontrol **M**essage **P**rotocol) echo 요청. 방화벽이 ICMP 를 막으면 살아 있어도 무응답이므로 **단독 판단은 위험**
  - `-c1` : **c**ount — 1회만 보내고 종료. 스크립트에서 무한 반복을 막는 용도
  - `-W1000` : 응답 **W**ait 제한 (macOS 는 밀리초, 리눅스는 초 단위 — 배포판 차이)
- `nc -z` : 포트 개방만 확인 → **서비스 수준의 생존 확인**이라 ping 보다 확실
- `arp -n` : 같은 네트워크 안이라면 MAC 이 잡히는지로 물리적 존재 확인
  - **ARP** = **A**ddress **R**esolution **P**rotocol → IP 주소로 MAC 주소를 알아내는 프로토콜
  - `-n` : **n**umeric — 이름 해석을 생략하고 숫자로 출력. DNS 조회 대기가 없어 빠름

### 리눅스 DHCP 서버의 대응 파일

| OS | 임대 기록 위치 |
| --- | --- |
| macOS (`bootpd`) | `/var/db/dhcpd_leases` |
| RHEL 계열 (`dhcpd`) | `/var/lib/dhcpd/dhcpd.leases` |
| 클라이언트 측 (NetworkManager) | `/var/lib/NetworkManager/` 하위 |

리눅스 `dhcpd.leases` 는 형식이 더 상세해 시작·종료 시각, 클라이언트 호스트명, 상태(`active`·`free`)까지 기록한다.

> 📝 **시험 포인트** : DHCP 서버의 임대 기록 파일 경로는 `/var/lib/dhcpd/dhcpd.leases`(RHEL). 임대 목록에 있다고 해서 **현재 접속 중이라는 뜻은 아님** — 만료 시각을 함께 봐야 함

관련 항목: [[#D-1. DHCP 란 무엇인가]] · [[#A-5. 실습은 SSH 로 하는가 UTM 콘솔로 하는가]]

---

## D-3. `ip addr` 출력 전체 해설

**Q.** 아래 출력의 각 항목이 무엇을 뜻하는가.

```text
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 192.168.64.3/24 brd 192.168.64.255 scope global dynamic noprefixroute enp0s1
       valid_lft 2785sec preferred_lft 2785sec
```

**A.** 세 줄이 각각 **장치 계층 → 주소 → 주소의 수명**을 나타낸다.

- `ip` : iproute2 패키지의 통합 네트워크 도구. 구형 `ifconfig`(net-tools)를 대체
- `ip addr` = `ip address show` 축약. 더 줄여 `ip a` 로도 사용

### 1행 — 장치(링크) 계층

| 항목 | 뜻 |
| --- | --- |
| `2:` | **인터페이스 인덱스**(ifindex). 커널이 부여하는 일련번호. `1` 은 항상 루프백 `lo` 이므로 첫 물리 인터페이스는 보통 `2` |
| `enp0s1` | 인터페이스 이름 |
| `<...>` | 플래그 목록 |
| `mtu 1500` | 한 번에 보낼 수 있는 최대 데이터 크기 |
| `qdisc fq_codel` | 송신 대기열 처리 방식 |
| `state UP` | 실제 동작 상태 |
| `group default` | 인터페이스 그룹 |
| `qlen 1000` | 송신 대기열 길이 |

#### 인터페이스 이름 `enp0s1` 의 규칙

RHEL 7 부터 **예측 가능한 네트워크 인터페이스 이름**(Predictable Network Interface Names) 규칙을 사용한다.

| 부분 | 뜻 |
| --- | --- |
| `en` | **en**thernet (유선). 무선은 `wl`(**w**ire**l**ess), WWAN 은 `ww` |
| `p0` | **p**CI 버스 번호 0 |
| `s1` | **s**lot 번호 1 |

- 구형 `eth0` 방식은 **인식 순서에 따라 번호가 바뀌어** NIC 을 추가하면 `eth0` 과 `eth1` 이 뒤바뀌는 문제가 있었음
- 새 방식은 물리적 위치에 기반하므로 **재부팅해도 이름이 고정**
- `eth0` 방식으로 되돌리려면 커널 파라미터에 `net.ifnames=0 biosdevname=0` 추가

#### 플래그 4종

| 플래그 | 뜻 |
| --- | --- |
| `BROADCAST` | 브로드캐스트 전송을 지원하는 장치 |
| `MULTICAST` | 멀티캐스트 전송을 지원하는 장치 |
| `UP` | **관리자가 켠 상태** (`ip link set enp0s1 up` 으로 설정) |
| `LOWER_UP` | **물리 계층 연결이 살아 있음** (캐리어 감지) |

- `UP` 과 `LOWER_UP` 의 구분이 핵심 — `UP` 은 "켜라고 지시했다", `LOWER_UP` 은 "실제로 선이 연결돼 있다"
- 랜선을 뽑으면 `UP` 은 남고 `LOWER_UP` 이 사라지며 `NO-CARRIER` 가 표시됨 → **장애 진단의 1차 지표**
- 루프백에는 `LOOPBACK` 플래그가 붙음

#### `mtu 1500`

- **MTU** = **M**aximum **T**ransmission **U**nit → 한 프레임에 실을 수 있는 **페이로드 최대 바이트**
- 1500 은 이더넷 표준값. 이보다 큰 데이터는 **단편화**(fragmentation)되어 나뉘어 전송
- VPN·터널을 쓰면 헤더가 추가돼 1500 을 넘길 수 없어 1400 대로 낮추기도 함
- 9000(점보 프레임)은 스토리지 전용망 등에서 사용
- 변경 : `ip link set enp0s1 mtu 1400` (임시) / `nmcli con mod enp0s1 802-3-ethernet.mtu 1400` (영구)

#### `qdisc fq_codel`

- **qdisc** = **q**ueueing **disc**ipline → 패킷을 어떤 순서·속도로 내보낼지 정하는 정책
- **fq_codel** = **F**air **Q**ueuing **Co**ntrolled **Del**ay → 흐름별로 공평하게 나누고 큐가 길어지면 미리 버려 지연을 억제
  - 한 대용량 다운로드가 대역폭을 독점해 다른 통신이 느려지는 **버퍼블로트**(bufferbloat) 완화가 목적
- 대안 : `pfifo_fast`(단순 선입선출), `noqueue`(루프백·브리지)

#### `state UP` 과 `qlen`

- `state` : 실제 운용 상태. `UP` · `DOWN` · `UNKNOWN`(루프백 등)
- `qlen 1000` = **q**ueue **len**gth → 커널이 드라이버로 넘기기 전 대기시킬 패킷 수. `txqueuelen` 과 같은 값
- `group default` : 인터페이스를 묶어 한 번에 제어하기 위한 그룹. 대부분 `default`

### 2행 — 주소(IP) 계층

| 항목 | 뜻 |
| --- | --- |
| `inet` | 주소 계열이 **IPv4**. IPv6 는 `inet6` |
| `192.168.64.3/24` | 주소와 **프리픽스 길이**(CIDR 표기) |
| `brd 192.168.64.255` | 브로드캐스트 주소 |
| `scope global` | 주소의 유효 범위 |
| `dynamic` | 자동 할당(DHCP)으로 받은 주소 |
| `noprefixroute` | 이 주소에 대한 경로를 커널이 자동 생성하지 않음 |
| 끝의 `enp0s1` | 주소 **라벨** |

#### `/24` 와 브로드캐스트

- **CIDR** = **C**lassless **I**nter-**D**omain **R**outing → 앞에서부터 몇 비트가 네트워크 부분인지 표기
- `/24` = 상위 24비트가 네트워크 → 서브넷 마스크 `255.255.255.0` 과 동일
- 네트워크 주소 `192.168.64.0`, 브로드캐스트 `192.168.64.255`, 호스트 사용 가능 범위 `.1` ~ `.254` (총 254개)
- `brd` = **br**oa**d**cast → 해당 네트워크 전체에 보내는 주소. 커널이 프리픽스로부터 자동 계산

#### `scope` 의 값

| 값 | 뜻 |
| --- | --- |
| `global` | 어디서나 유효. 외부와 통신 가능한 일반 주소 |
| `link` | 같은 네트워크 안에서만 유효 (예: `169.254.x.x`) |
| `host` | 자기 자신 안에서만 유효 (예: `127.0.0.1`) |

#### `dynamic` 과 `noprefixroute`

- `dynamic` : **수명이 정해진 주소**라는 커널 표시. DHCP 또는 IPv6 자동설정으로 받은 경우 붙음
  - 고정 IP 로 바꾸면 이 표시가 사라짐 → **DHCP 인지 고정인지 한눈에 구분하는 지표**
- `noprefixroute` : 보통 주소를 붙이면 커널이 `192.168.64.0/24` 경로를 자동으로 만드는데, 이를 **하지 말라**는 뜻
  - NetworkManager 가 경로를 직접 관리하기 위해 붙임. 경로 우선순위(metric)를 세밀히 제어하려는 목적
  - 경로는 `ip route` 로 별도 확인

### 3행 — 주소의 수명

```text
valid_lft 2785sec preferred_lft 2785sec
```

- `valid_lft` = **valid lifetime** → 이 주소가 **유효한** 남은 시간
- `preferred_lft` = **preferred lifetime** → 이 주소를 **새 연결에 우선 사용할** 남은 시간
- `preferred` 가 먼저 끝나면 주소는 **deprecated** 상태 — 기존 연결은 유지하되 새 연결에는 쓰지 않음 (주로 IPv6 주소 교체 시 사용)
- 고정 IP 는 수명이 없으므로 두 값이 `forever` 로 표시

#### DHCP 임대와의 관계

`2785sec` 은 약 46분이다. macOS 기본 임대가 1시간(3600초)이므로 **약 14분 전에 임대를 받았다**는 뜻이다. 이 값은 초 단위로 계속 줄어들며, 절반쯤 남았을 때 클라이언트가 자동 갱신을 시도해 다시 채워진다 ([[#D-1. DHCP 란 무엇인가]] 의 T1 시점).

```bash
watch -n5 'ip -4 addr show enp0s1 | grep valid_lft'
```

- `watch` : 명령을 주기적으로 반복 실행해 변화를 관찰
- `-n5` : **n**umber of seconds — 5초 간격
- 값이 줄다가 갑자기 늘어나는 순간이 **갱신 성공 시점**

### 관련 조회 명령

```bash
ip -br addr              # 한 줄 요약 (brief)
ip -4 addr               # IPv4 만
ip -s link show enp0s1   # 송수신 통계·에러·드롭
ip route                 # 라우팅 테이블
ifconfig enp0s1          # 구형 도구 (net-tools 설치 필요)
```

- `-br` : **br**ief — 인터페이스마다 한 줄로 요약. 다수 인터페이스를 훑을 때 유용
- `-4` / `-6` : 주소 계열 한정
- `-s` : **s**tatistics — 패킷 수, 오류(errors), 버려진 패킷(dropped) 표시. **NIC 장애 진단의 핵심**

### 구형 명령 대응

| 신형 (iproute2) | 구형 (net-tools) |
| --- | --- |
| `ip addr` | `ifconfig` |
| `ip link set eth0 up` | `ifconfig eth0 up` |
| `ip route` | `route -n` |
| `ip neigh` | `arp -n` |
| `ss` | `netstat` |

RHEL 9 는 `net-tools` 가 기본 미설치 → 구형 명령을 쓰려면 `dnf install net-tools` 필요. **시험은 양쪽 다 출제**되므로 대응 관계를 외워야 한다.

> 📝 **시험 포인트** : `UP` 과 `LOWER_UP` 의 차이(설정 vs 물리 연결), `NO-CARRIER` 의 의미, MTU 기본값 1500, `scope` 3종, `dynamic` 표시로 DHCP 여부 판별, `en p0 s1` 명명 규칙, `ip` ↔ `ifconfig` 대응

관련 항목: [[#D-1. DHCP 란 무엇인가]] · 절차서 [[LAB/08-network-config]] 1절

---

## D-4. `ip route` 출력 전체 해설

**Q.** `ip route` 출력의 각 항목이 무엇을 뜻하는가.

**A.** 각 줄이 **"어떤 목적지로 갈 때, 어디를 거쳐, 어느 장치로 내보낼지"** 를 정한 규칙 하나다.

DHCP 를 쓰는 지금 환경에서는 보통 두 줄이 나온다.

```text
default via 192.168.64.1 dev enp0s1 proto dhcp src 192.168.64.3 metric 100
192.168.64.0/24 dev enp0s1 proto kernel scope link src 192.168.64.3 metric 100
```

- `ip route` = `ip route show` 축약. `ip r` 로도 사용
- 구형 대응 명령은 `route -n` (net-tools)

### 각 필드

| 필드 | 뜻 |
| --- | --- |
| 맨 앞 (목적지) | 이 규칙이 적용될 **목적지 대역** |
| `via <주소>` | **다음 홉**(next hop) — 이 주소로 넘겨서 보냄 |
| `dev <장치>` | 패킷을 내보낼 **인터페이스** |
| `proto <값>` | 이 경로를 **누가 만들었는지** |
| `scope <값>` | 목적지까지의 **도달 범위** |
| `src <주소>` | 이 경로로 나갈 때 쓸 **출발지 주소** |
| `metric <숫자>` | **우선순위**. 값이 **작을수록 우선** |

### 1행 — 기본 경로

```text
default via 192.168.64.1 dev enp0s1 proto dhcp src 192.168.64.3 metric 100
```

- `default` : 목적지 `0.0.0.0/0` 의 별칭 → **다른 어떤 규칙에도 해당하지 않는 모든 목적지**
- `via 192.168.64.1` : 게이트웨이. 외부로 나가는 모든 패킷을 여기로 넘김
- `proto dhcp` : **DHCP 로 받은 게이트웨이 정보**를 바탕으로 만들어진 경로
- `src 192.168.64.3` : 이 경로를 탈 때 패킷의 출발지 주소로 쓸 값
- `metric 100` : 인터페이스가 여러 개일 때 우선순위. 유선·무선이 동시에 있으면 metric 이 작은 쪽이 선택됨

**이 줄이 없으면 같은 네트워크 안에서만 통신 가능하고 인터넷은 되지 않는다.**

### 2행 — 직접 연결 경로

```text
192.168.64.0/24 dev enp0s1 proto kernel scope link src 192.168.64.3 metric 100
```

- `192.168.64.0/24` : 자신이 속한 네트워크 대역
- **`via` 가 없다** → 게이트웨이를 거치지 않고 **직접 전달**한다는 뜻
- `proto kernel` : 인터페이스에 IP 를 붙일 때 **커널이 자동 생성**한 경로
  - 단, `noprefixroute` 가 붙어 있으면 커널 대신 NetworkManager 가 만들므로 `proto static` 또는 `proto dhcp` 로 표시될 수 있음 → **환경에 따라 다르므로 실제 출력 확인 필요**
- `scope link` : 목적지가 **같은 링크(네트워크) 안에** 있음. 라우터를 거치지 않음

### `proto` 값의 종류

| 값 | 만든 주체 |
| --- | --- |
| `kernel` | 커널이 IP 설정 시 자동 생성 |
| `dhcp` | DHCP 클라이언트가 받은 정보로 생성 |
| `static` | 관리자가 수동 지정 (`ip route add`, NetworkManager 설정) |
| `boot` | 부팅 과정에서 생성 |
| `ra` | IPv6 **R**outer **A**dvertisement 로 생성 |

> `proto` 는 동작에 영향을 주지 않는 **출처 표시용 꼬리표**. 어떤 경로를 지워도 되는지 판단할 때 유용

### `scope` 값의 종류

| 값 | 뜻 |
| --- | --- |
| `global` | 라우터를 거쳐야 하는 원격 목적지 (기본값, 생략 가능) |
| `link` | 같은 네트워크 안 — 직접 전달 |
| `host` | 자기 자신 (루프백) |

### 경로 선택 규칙 — 최장 프리픽스 일치

패킷을 보낼 때 커널은 **가장 구체적인(프리픽스가 긴) 경로**를 먼저 고른다.

| 경로 | 프리픽스 길이 | 우선순위 |
| --- | --- | --- |
| `192.168.64.0/24` | 24 | 높음 (더 구체적) |
| `default` (= `/0`) | 0 | 낮음 (최후의 수단) |

- `192.168.64.50` 으로 보낼 때 → 2행에 걸림 → **게이트웨이 없이 직접 전달**
- `8.8.8.8` 으로 보낼 때 → 2행에 안 걸림 → 1행 `default` 로 → **게이트웨이에 넘김**
- 프리픽스 길이가 같으면 `metric` 이 작은 쪽 선택

실제로 어느 경로가 쓰이는지 확인하는 명령이 있다.

```bash
ip route get 8.8.8.8
ip route get 192.168.64.50
```

- `ip route get <목적지>` : 커널에게 **"이 주소로 보낸다면 어느 경로를 쓸 것인가"** 를 물어봄
- 라우팅 문제 진단에서 가장 먼저 쓰는 명령 — 표를 눈으로 훑어 추측하는 것보다 정확

### 경로 추가·삭제

```bash
# 임시 (재부팅 시 소멸)
ip route add 10.10.0.0/16 via 192.168.64.1
ip route del 10.10.0.0/16
ip route add default via 192.168.64.1

# 영구 (NetworkManager 설정에 저장)
nmcli con mod enp0s1 +ipv4.routes "10.10.0.0/16 192.168.64.1"
nmcli con up enp0s1
```

- `add` / `del` : 경로 추가·삭제. **root 권한 필요**
- `+ipv4.routes` : `+` 는 기존 값에 **추가**. `-` 는 제거, 부호 없으면 **전체 교체**
- 임시 명령은 진단·테스트용, 영구 설정은 반드시 NetworkManager 를 통해야 재부팅 후에도 유지

### 라우팅 테이블

리눅스는 여러 개의 라우팅 테이블을 갖는다.

```bash
ip route show table main      # 기본 테이블 (그냥 ip route 와 동일)
ip route show table local     # 커널이 관리하는 로컬·브로드캐스트 주소
ip route show table all       # 전부
```

| 테이블 | 용도 |
| --- | --- |
| `local` (255) | 자기 주소·브로드캐스트. 커널 전용, 수정 금지 |
| `main` (254) | 일반 경로. `ip route` 가 보여주는 것 |
| `default` (253) | 거의 비어 있음 |

- `/etc/iproute2/rt_tables` 에 테이블 번호와 이름이 정의됨
- 출발지에 따라 다른 테이블을 쓰게 하려면 `ip rule` 로 정책 라우팅 구성 (고급)

### 구형 명령 대응

```bash
route -n          # ip route 에 대응 (net-tools 필요)
netstat -rn       # 동일
```

- `-n` : **n**umeric — 호스트명·서비스명으로 바꾸지 않고 숫자 그대로. DNS 조회 대기가 없어 빠르고, 이름 해석 실패로 멈추는 일도 없음
- `route -n` 출력의 `Flags` 열 — `U`(**U**p, 활성) · `G`(**G**ateway, 게이트웨이 경유) · `H`(**H**ost, 단일 호스트 대상)

> 📝 **시험 포인트** : `default` 경로가 없으면 외부 통신 불가. `via` 유무로 직접 전달 여부 판별. `metric` 은 작을수록 우선. **최장 프리픽스 일치** 원칙. `route -n` 의 `Flags` 열 `U`·`G`·`H` 해석. 영구 설정은 `nmcli`, 임시는 `ip route add`

관련 항목: [[#D-3. `ip addr` 출력 전체 해설]] · 절차서 [[LAB/08-network-config]] 1~3절

---

# E. 하드웨어·장치

## E-1. PCI 버스란 무엇인가

**Q.** 인터페이스 이름 `enp0s1` 의 `p0` 이 PCI 버스 0번이라고 했는데, PCI 버스가 무엇인가.

**A.** **P**eripheral **C**omponent **I**nterconnect — CPU·메모리와 주변장치를 잇는 **표준 연결 통로**다.

### 버스(bus)라는 개념

- **버스** : 여러 장치가 **공유하는 데이터 통로**. 여러 사람이 같은 노선의 버스를 타는 것에 빗댄 이름
- 컴퓨터 내부에서 CPU 가 그래픽카드·네트워크카드·저장장치와 데이터를 주고받는 길
- 각 장치마다 전용 배선을 까는 대신 **공통 규격의 통로**를 두어, 규격만 맞으면 어떤 장치든 꽂을 수 있게 함 → 확장성의 핵심

### 세대별 변천

| 규격 | 시기 | 특징 |
| --- | --- | --- |
| **ISA** (**I**ndustry **S**tandard **A**rchitecture) | 1980년대 | 초기 확장 버스. 느리고 수동 설정 필요 |
| **PCI** | 1990년대 | 병렬 공유 버스. **플러그 앤 플레이** 도입 |
| **PCI-X** | 2000년대 초 | PCI 의 속도 확장판, 서버용 |
| **PCIe** (**PCI E**xpress) | 2004~현재 | **점대점 직렬** 방식으로 전면 재설계 |

- 현재 물리 규격은 사실상 전부 **PCIe** 이지만, **소프트웨어가 보는 모습(주소 체계·설정 공간)은 PCI 그대로** 유지
- 그래서 리눅스는 여전히 `lspci`, `/sys/bus/pci/` 처럼 PCI 라는 이름을 쓴다
- PCIe 의 `x1` `x4` `x8` `x16` 은 **레인(lane) 수** — 레인이 많을수록 대역폭 증가 (그래픽카드가 x16 을 쓰는 이유)

### 주소 체계 — BDF

PCI 장치는 **도메인:버스:장치.기능** 형식으로 식별된다.

```text
0000:00:01.0
 │    │  │ └─ Function : 한 장치 안의 기능 번호 (다기능 카드는 .0 .1 …)
 │    │  └─── Device   : 버스에 붙은 장치(슬롯) 번호
 │    └────── Bus      : 버스 번호
 └─────────── Domain   : PCI 도메인 (대개 0000)
```

- **BDF** = **B**us / **D**evice / **F**unction
- 이 주소는 **물리적 위치에 대응**하므로 재부팅해도 바뀌지 않음 → 인터페이스 이름을 여기서 따오는 이유

### `enp0s1` 과의 연결

| 부분 | 근거 |
| --- | --- |
| `en` | 이더넷 장치 |
| `p0` | **P**CI **버스 0번** |
| `s1` | **s**lot(장치) **1번** |

즉 `enp0s1` 은 **"PCI 버스 0번, 슬롯 1번에 꽂힌 이더넷 카드"** 라는 뜻이다. 물리적 위치를 이름에 박아 넣었기 때문에, NIC 을 추가해도 기존 이름이 바뀌지 않는다. 구형 `eth0` 방식이 인식 순서에 따라 뒤바뀌던 문제를 이렇게 해결했다.

### 조회 명령

```bash
lspci                    # PCI 장치 목록
lspci -nn                # 벤더·장치 ID 를 숫자로 함께 표시
lspci -k                 # 각 장치가 쓰는 커널 드라이버
lspci -v                 # 상세 (IRQ·메모리 주소 등)
lspci -s 00:01.0         # 특정 주소만
ls /sys/bus/pci/devices/ # 커널이 인식한 PCI 장치의 sysfs 경로
```

- `lspci` = **l**i**s**t **PCI** → `pciutils` 패키지에 포함
- `-nn` : **n**umeric 을 두 번 → 이름과 **숫자 ID 를 함께** 표시. 드라이버를 검색할 때 `8086:100e` 같은 ID 가 필요
- `-k` : **k**ernel — 어떤 드라이버가 물려 있는지. **장치는 보이는데 동작하지 않을 때 드라이버 누락을 확인**하는 용도
- `-v` / `-vv` / `-vvv` : **v**erbose 단계별 상세
- `-s` : **s**elect — BDF 주소로 대상 한정

### 가상머신에서의 PCI

UTM/QEMU 는 게스트에게 **가상의 PCI 버스를 만들어 제공**한다. VirtIO 네트워크·디스크 장치도 이 가상 PCI 버스에 붙은 것으로 보이며, 그래서 인터페이스 이름이 `enp0s1` 이 되고 디스플레이 카드 이름에도 `virtio-gpu-pci` 처럼 `-pci` 가 붙는다 ([[#A-3. `Display output is not active.` 가 뜨는 이유]] 참조).

```bash
lspci -k | grep -A3 -i ethernet
```

```text
00:01.0 Ethernet controller: Red Hat, Inc. Virtio 1.0 network device ...
	Subsystem: Red Hat, Inc. Device ...
	Kernel driver in use: virtio-pci
```

- `Red Hat, Inc.` : VirtIO 규격을 만든 곳이 Red Hat 이라 벤더로 표시됨
- `Kernel driver in use: virtio-pci` : 가상화 전용 드라이버가 물려 있음을 확인

### 다른 버스와의 비교

| 버스 | 용도 |
| --- | --- |
| **PCI/PCIe** | 내장 확장 카드 (NIC·GPU·NVMe·RAID) |
| **USB** (**U**niversal **S**erial **B**us) | 외장 주변장치. 핫플러그 지원 |
| **SATA** (**S**erial **AT** **A**ttachment) | 저장장치 전용 |
| **SCSI** (**S**mall **C**omputer **S**ystem **I**nterface) | 서버용 저장장치·테이프 |

조회 명령도 버스별로 다르다 — `lspci` · `lsusb` · `lsblk` · `lsscsi`.

> 📝 **시험 포인트** : `lspci`(PCI) · `lsusb`(USB) · `lsblk`(블록 장치) · `dmidecode`(BIOS·메인보드 정보) 의 용도 구분. `lspci -k` 로 드라이버 확인. 인터페이스 명명 규칙에서 `p`=PCI 버스, `s`=슬롯

관련 항목: [[#D-3. `ip addr` 출력 전체 해설]] · 절차서 [[LAB/01-vm-setup-and-inspection]] 3-6

---

## 관련 문서

- [[README]] — LINUX-MASTER 허브
- [[LAB/README]] — 실습 절차서 허브
- [[THEORY/user-permission]] · [[THEORY/disk-device]] · [[THEORY/system-security]]
