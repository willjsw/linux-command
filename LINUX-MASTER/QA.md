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

### F. 셸·프로세스 환경
- [[#F-1. 환경변수·셸·tty 와 `su -` 가 환경을 초기화하는 이유]]
- [[#F-2. 로그인 셸과 비로그인 셸의 차이]]
- [[#F-3. `su`·`sudo -i` 를 반복하면 셸이 스택처럼 쌓이는가]]
- [[#F-4. `pstree` 출력 — 최소 설치 Rocky 9 의 프로세스 전수 해설]]

### G. 명령어·데몬 구조
- [[#G-1. `~d` 데몬과 `~ctl` 명령의 관계]]

### H. 로그·커널
- [[#H-1. 커널 링 버퍼란 무엇인가]]

### I. 로케일·국제화
- [[#I-1. 로케일이란 무엇인가]]

### J. 부팅·systemd
- [[#J-1. `systemd-analyze critical-chain` 출력 해석]]
- [[#J-2. 타겟(target)과 런레벨(runlevel)이란 정확히 무엇인가]]

### K. 소켓·파일 디스크립터
- [[#K-1. 소켓이란 무엇인가 — 파일 디스크립터와 함께]]

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

# F. 셸·프로세스 환경

## F-1. 환경변수·셸·tty 와 `su -` 가 환경을 초기화하는 이유

**Q.** `su -` 만 환경변수가 초기화되는 이유는 무엇인가. 환경변수란 무엇이고 어디에 저장되며 어떤 단위로 유지되는가. 셸과 tty 는 이와 어떤 관계인가.

**A.** 하나로 이어지는 이야기다. **환경변수는 프로세스마다 붙어 있는 메모리 속 목록**이고, **셸은 그 목록을 관리하며 자식에게 물려주는 프로그램**이며, **tty 는 그 셸이 붙어 있는 입출력 장치**다. `su -` 는 "새로 로그인한 것처럼" 시작하라는 지시이므로 목록을 비우고 다시 채운다.

---

### 1. 환경변수란 무엇인가

- **환경변수**(environment variable) : 프로세스가 지니고 다니는 **`이름=값` 쌍의 목록**
- 프로그램이 "내가 어떤 환경에서 실행되고 있는지" 를 알아내는 표준 수단
- 명령행 인자가 "이번 실행에 대한 지시" 라면, 환경변수는 **"주변 상황에 대한 정보"**

| 변수 | 담긴 정보 | 이 값이 없거나 틀리면 |
| --- | --- | --- |
| `PATH` | 명령을 찾을 디렉터리 목록 (`:` 구분) | 명령을 쳐도 `command not found` |
| `HOME` | 홈 디렉터리 | `cd` 가 갈 곳을 모름, `~` 확장 실패 |
| `USER` · `LOGNAME` | 사용자 이름 | 일부 프로그램이 사용자 식별 실패 |
| `SHELL` | 로그인 셸 경로 | `vi` 의 `:!` 등이 잘못된 셸 호출 |
| `PWD` | 현재 작업 디렉터리 | — |
| `LANG` · `LC_*` | 언어·문자셋 | 한글 깨짐, 정렬 순서 변화 |
| `TERM` | 터미널 종류 | `clear`·`vi`·`top` 실패 ([[#A-6. `unknown terminal type` 오류]]) |
| `PS1` | 셸 프롬프트 모양 | — |

---

### 2. 어디에 저장되는가 — 파일이 아니라 프로세스 메모리

**핵심** : 환경변수는 어떤 파일에 저장돼 있는 값이 아니다. **각 프로세스의 메모리 안에 있다.**

```bash
# 현재 셸의 환경변수
env
printenv PATH

# 특정 프로세스의 환경변수를 직접 열람
cat /proc/$$/environ | tr '\0' '\n' | head
sudo cat /proc/1/environ | tr '\0' '\n'
```

- `env` : 현재 프로세스의 **환경변수만** 출력 (셸 변수는 제외)
- `printenv <이름>` : 특정 환경변수 값만 출력
- `$$` : 현재 셸의 **PID**(**P**rocess **ID**entifier)
- `/proc/<PID>/environ` : 그 프로세스의 환경변수 원본. 값이 **NUL 문자(`\0`)로 구분**돼 있어 그대로 보면 한 줄로 붙어 나옴
- `tr '\0' '\n'` : NUL 을 줄바꿈으로 치환해 읽기 좋게 만듦
  - `tr` = **tr**anslate → 문자 단위 치환·삭제 도구
- `/proc/1/environ` 은 systemd 의 환경 → 다른 프로세스와 내용이 전혀 다름을 확인할 수 있음

> `/proc/<PID>/environ` 은 **프로세스가 시작될 때 받은 값**을 보여준다. 실행 중 `export` 로 바꾼 값은 반영되지 않음

**그럼 `/etc/profile` 같은 파일은 무엇인가.** 그것은 저장소가 아니라 **"셸이 시작할 때 실행할 스크립트"** 다. 그 안의 `export` 문이 실행되어 **그 순간 프로세스 메모리에 값이 만들어진다**. 파일을 고쳐도 이미 떠 있는 셸은 바뀌지 않고, 새로 시작하거나 `source` 해야 반영되는 이유가 이것이다.

---

### 3. 어떤 단위로 유지되는가 — 프로세스 단위, 그리고 단방향 상속

환경변수는 **프로세스 하나하나가 각자 사본을 갖는다.** 그리고 **자식이 생길 때 부모의 것을 복사**받는다.

```
systemd (PID 1)
 └─ sshd
     └─ bash          ← 로그인 셸. 여기서 export FOO=1
         ├─ vim       ← FOO=1 을 물려받음
         └─ bash      ← 자식 셸. FOO=1 물려받음
              └─ 여기서 FOO=2 로 바꿔도 부모의 FOO 는 1 그대로
```

- 상속은 **부모 → 자식 한 방향**. 자식이 무엇을 바꾸든 **부모에게는 전달되지 않는다**
- 이것이 **스크립트를 실행해도 현재 셸의 변수가 안 바뀌는 이유**

```bash
# 검증
echo 'export FOO=child' > /tmp/t.sh; chmod +x /tmp/t.sh

export FOO=parent
./tmp/t.sh   ;  echo $FOO      # → parent  (자식 프로세스에서 실행 → 영향 없음)
source /tmp/t.sh ; echo $FOO   # → child   (현재 셸에서 실행 → 반영)
```

- `./script` : **새 프로세스**를 띄워 실행 → 변경 사항이 종료와 함께 사라짐
- `source script` (= `. script`) : **현재 셸 안에서** 한 줄씩 실행 → 변경 사항이 남음
- `~/.bashrc` 를 고친 뒤 `source ~/.bashrc` 하는 이유가 바로 이것

#### 셸 변수 vs 환경변수

```bash
MYVAR=hello          # 셸 변수 — 이 셸 안에서만
export MYVAR         # 환경변수로 승격 — 자식에게 상속

set | grep MYVAR     # 셸 변수 + 환경변수 전부
env | grep MYVAR     # 환경변수만
```

| 명령 | 대상 |
| --- | --- |
| `set` | 셸 변수 + 환경변수 + 함수 전부 |
| `env` / `printenv` | **환경변수만** |
| `export` | 셸 변수를 환경변수로 승격 (또는 선언과 동시에) |
| `unset` | 변수 삭제 |
| `export -n` | 환경변수를 셸 변수로 강등 (삭제는 아님) |

- `export` 없이 만든 변수는 **자식 프로세스가 볼 수 없다** → 스크립트가 값을 못 읽는 흔한 원인

---

### 4. 셸과의 관계

- **셸**(shell) : 사용자의 명령을 받아 해석하고 프로그램을 실행해 주는 프로그램. 커널을 감싸는 껍데기라는 뜻
- 환경변수를 **보관·수정하고 자식 프로세스에 물려주는 주체**가 셸

#### 로그인 셸 vs 비로그인 셸 — 초기화 파일이 다르다

| 구분 | 언제 | 읽는 파일 (순서대로) |
| --- | --- | --- |
| **로그인 셸** | 콘솔·SSH 로 로그인, `su -`, `bash -l` | `/etc/profile` → `/etc/profile.d/*.sh` → `~/.bash_profile` → (그 안에서) `~/.bashrc` → `/etc/bashrc` |
| **비로그인 대화형 셸** | 터미널에서 `bash` 실행, `su` | `~/.bashrc` → `/etc/bashrc` |
| **비대화형** | 스크립트 실행 | `$BASH_ENV` 가 지정돼 있으면 그 파일만 |

```bash
# 지금 셸이 로그인 셸인지 확인
shopt -q login_shell && echo "로그인 셸" || echo "비로그인 셸"
echo $0        # 로그인 셸이면 앞에 하이픈이 붙어 -bash
```

- `shopt` = **sh**ell **opt**ions → bash 동작 옵션 조회·설정
- `-q` : **q**uiet — 출력 없이 **종료 코드로만** 결과 전달. `if` 문이나 `&&` 와 조합할 때 사용
- `$0` : 현재 실행 중인 셸(또는 스크립트)의 이름. 로그인 셸은 관례적으로 **`-bash`** 처럼 하이픈이 앞에 붙음
- **`/etc/profile` 은 로그인 셸만 읽는다** → 시스템 전역 `PATH` 설정이 여기 있으므로, 로그인 셸이 아니면 그 설정이 적용되지 않음

---

### 5. 그래서 `su` 와 `su -` 가 다르다

| | `su` | `su -` (= `su -l`, `su --login`) |
| --- | --- | --- |
| 셸 종류 | 비로그인 셸 | **로그인 셸** |
| 환경변수 | 대부분 **호출자 것 유지** | **초기화 후 대상 사용자 기준으로 재구성** |
| `PATH` | 호출자 것 그대로 | root 의 `PATH` (`/sbin`·`/usr/sbin` 포함) |
| 작업 디렉터리 | **바뀌지 않음** | 대상 사용자의 홈으로 이동 |
| 읽는 파일 | `~/.bashrc` | `/etc/profile` → `~/.bash_profile` → … |

#### 왜 초기화하는가

`su -` 의 `-` 는 **"그 사용자로 새로 로그인한 것과 똑같이 만들어라"** 는 지시다. 방금 로그인한 사람의 환경에 이전 사용자의 흔적이 남아 있으면 안 되므로, `su` 는 환경 목록을 비우고 최소한의 값(`HOME`·`SHELL`·`USER`·`LOGNAME`·`PATH`)만 대상 사용자 기준으로 채운 뒤 로그인 셸을 띄운다. 그 로그인 셸이 `/etc/profile` 부터 다시 읽으면서 나머지가 채워진다.

#### 실무에서 문제가 되는 지점

```bash
su
fdisk -l          # → command not found 또는 권한 오류
echo $PATH        # → /sbin, /usr/sbin 이 없음
exit

su -
which fdisk       # → /usr/sbin/fdisk  정상
```

- 관리 명령은 대부분 `/sbin`·`/usr/sbin` 에 있는데, 일반 사용자의 `PATH` 에는 이 경로가 없다
- `su` 는 그 `PATH` 를 그대로 물려받으므로 **root 인데도 관리 명령이 안 보이는** 상황이 생김
- **관리 작업에는 항상 `su -` 또는 `sudo -i`** 를 쓰는 이유

> util-linux 판 `su` 는 하위 호환을 위해 `-` 없이도 `HOME`·`SHELL` 은 대상 사용자 것으로 바꾼다(대상이 root 가 아니면 `USER`·`LOGNAME` 도). 그러나 **`PATH` 는 그대로**여서 위 문제가 남는다. 실제 값은 [[LAB/01-vm-setup-and-inspection]] 2-4 에서 직접 비교

| 명령 | 로그인 셸 | 비밀번호 | 환경 |
| --- | --- | --- | --- |
| `su` | ✗ | 대상 사용자 | 대체로 유지 |
| `su -` | ✓ | 대상 사용자 | 재구성 |
| `sudo -i` | ✓ | **자기 자신** | 재구성 |
| `sudo -s` | ✗ | **자기 자신** | 대체로 유지 |
| `sudo <명령>` | — | **자기 자신** | `secure_path` 적용 |

- `sudo` 는 `/etc/sudoers` 의 `Defaults secure_path` 로 **`PATH` 를 강제 지정**한다 → 사용자가 `PATH` 를 조작해 가짜 명령을 실행시키는 공격을 막기 위함
- `env_reset` 옵션이 기본 활성 → 위험한 환경변수(`LD_PRELOAD` 등)를 제거하고 실행

---

### 6. tty 란 무엇인가

- **tty** = **T**ele**TY**pewriter (전신 타자기)
- 1960~70년대 유닉스는 **물리적 전신 타자기**를 단말기로 썼고, 그 장치 이름이 그대로 굳어짐
- 현재는 **"셸이 붙어 있는 입출력 장치"** 를 통칭

```bash
tty                    # 현재 터미널의 장치 파일 경로
who                    # 접속자와 각자의 tty
ls -l /dev/pts/
ps -eo pid,tty,cmd | head
```

- `tty` : 표준 입력이 연결된 터미널 장치 경로 출력. 파이프로 넘기면 `not a tty` → **스크립트에서 대화형 여부 판별**에 사용
- `ps -o tty` : 각 프로세스의 **제어 터미널**. `?` 는 터미널이 없는 데몬

#### 종류

| 장치 | 정체 |
| --- | --- |
| `/dev/tty1` ~ `tty6` | **가상 콘솔**. 물리 화면·키보드에 직결 (`Ctrl+Alt+F1~F6`) |
| `/dev/pts/0`, `/dev/pts/1` … | **의사 터미널**(**p**seudo-**t**erminal **s**lave). SSH·터미널 에뮬레이터가 만드는 소프트웨어 터미널 |
| `/dev/ttyS0` | 직렬 포트 (x86) |
| `/dev/ttyAMA0` | 직렬 포트 (ARM PL011) — 실습 VM 의 직렬 콘솔 ([[#A-3. `Display output is not active.` 가 뜨는 이유]]) |
| `/dev/console` | 커널이 메시지를 보내는 시스템 콘솔 |

- SSH 로 접속하면 `who` 의 tty 열이 `pts/0` 으로 나옴 → **원격 세션**
- UTM 화면에서 직접 로그인하면 `tty1` → **로컬 콘솔**
- 이 구분으로 **누가 어디서 들어왔는지** 판별 가능 (보안 점검 항목)

#### 셸·프로세스와의 관계

- 셸은 **어떤 tty 에 붙어서** 실행된다. 그 tty 가 그 셸의 **제어 터미널**(controlling terminal)
- `Ctrl+C` 는 tty 가 **SIGINT** 를, `Ctrl+Z` 는 **SIGTSTP** 를 그 터미널의 포그라운드 프로세스 그룹에 보내는 것
- **터미널을 닫으면** 커널이 그 tty 에 붙은 프로세스들에게 **SIGHUP**(**H**ang **UP**, 전화 끊기)을 보냄 → 실행 중이던 작업이 함께 종료
- 이를 피하려고 쓰는 것이 `nohup`(**no** **hup**, SIGHUP 무시)·`setsid`·`tmux`·`screen` ([[LAB/06-process-scheduling-diagnosis]] 2절)

```bash
# tty 와 환경의 관계 확인
tty                              # /dev/pts/0
echo $TERM                       # 이 tty 를 어떤 터미널로 다룰지
ps -eo pid,ppid,tty,stat,cmd | grep bash
```

---

### 7. 전체를 한 줄로 잇기

```
tty (입출력 장치)
 └─ 셸 프로세스 (환경변수 목록을 메모리에 보유)
     ├─ 시작 시 초기화 파일(/etc/profile, ~/.bashrc)을 읽어 목록을 채움
     ├─ export 로 목록에 등재된 것만 자식에게 복사
     └─ 자식 프로세스 (부모의 목록 사본을 받음, 되돌려주지는 못함)
```

`su -` 는 이 그림에서 **셸 프로세스를 새로 만들면서 목록을 비우고 대상 사용자 기준으로 다시 채우는** 동작이다. `su` 는 목록을 대체로 물려받은 채 새 셸만 띄운다. tty 는 어느 쪽이든 그대로 유지된다 — 같은 터미널에서 실행했으므로.

> 📝 **시험 포인트** : 로그인 셸 초기화 파일 **순서**(`/etc/profile` → `~/.bash_profile` → `~/.bashrc` → `/etc/bashrc`)가 최빈출. `set`·`env`·`export`·`unset` 의 대상 차이. `source`/`.` 와 직접 실행의 차이. `su` 와 `su -` 의 `PATH` 문제. `tty1`(콘솔) vs `pts/N`(원격) 구분. `nohup` 이 막는 신호는 **SIGHUP(1번)**

관련 항목: [[#A-6. `unknown terminal type` 오류]] · [[#C-1. wheel 그룹이란 무엇이며 sudo 와 어떤 관계인가]] · 절차서 [[LAB/01-vm-setup-and-inspection]] 2-4·7절, [[LAB/04-file-text-shell]] 7절

---

## F-2. 로그인 셸과 비로그인 셸의 차이

**Q.** 로그인 셸과 일반(비로그인) 셸은 무엇이 다른가.

**A.** **어떤 초기화 파일을 읽는가**가 다르다. 그 결과로 `PATH` 같은 환경이 갖춰지는지 여부가 갈린다.

### 두 개의 축 — 여기서 헷갈린다

흔히 "로그인 셸 / 일반 셸" 로 나누지만, 실제로는 **독립된 두 축**이 있다.

| | **대화형** (interactive) | **비대화형** (non-interactive) |
| --- | --- | --- |
| **로그인** | 콘솔 로그인, SSH 접속, `su -`, `bash -l` | `bash -l -c '명령'` (드묾) |
| **비로그인** | 터미널에서 `bash` 실행, `su`, `screen`/`tmux` 새 창 | **스크립트 실행**, `ssh 호스트 '명령'`, **cron 작업** |

- **로그인 여부** : 사용자 인증을 거쳐 세션을 시작하는 셸인가
- **대화형 여부** : 사람이 프롬프트를 보고 명령을 입력하는가

이 둘의 조합에 따라 읽는 파일이 정해진다.

### 읽는 파일

#### ① 로그인 셸

```
/etc/profile
  └─ /etc/profile.d/*.sh          (profile 안에서 순회 실행)
~/.bash_profile                    (없으면 ~/.bash_login, 그것도 없으면 ~/.profile)
  └─ ~/.bashrc                     (RHEL 계열은 bash_profile 이 여기를 호출)
       └─ /etc/bashrc
```

- 세 후보 중 **먼저 발견된 하나만** 읽는다 → `~/.bash_profile` 이 있으면 `~/.profile` 은 무시
- 종료할 때 `~/.bash_logout` 실행

RHEL 계열 `~/.bash_profile` 의 실제 내용이다.

```bash
cat ~/.bash_profile
```

```text
# .bash_profile
if [ -f ~/.bashrc ]; then
	. ~/.bashrc
fi
PATH=$PATH:$HOME/.local/bin:$HOME/bin
export PATH
```

- `. ~/.bashrc` : `source` 의 축약형. **로그인 셸도 `~/.bashrc` 를 읽게 하려고** 명시적으로 호출
- 이 호출이 없으면 로그인 시 alias 가 하나도 적용되지 않음 → 배포판이 관례적으로 넣어 둠

#### ② 비로그인 대화형 셸

```
~/.bashrc
  └─ /etc/bashrc
```

- `/etc/profile` 을 **읽지 않는다** → 시스템 전역 환경 설정이 적용되지 않음
- 그럼에도 문제가 없는 이유는, 이미 로그인 셸이 설정해 둔 환경을 **부모로부터 상속**받기 때문 ([[#F-1. 환경변수·셸·tty 와 `su -` 가 환경을 초기화하는 이유]] 의 상속 구조)

#### ③ 비대화형 셸

- `$BASH_ENV` 환경변수에 파일 경로가 지정돼 있으면 그 파일만 읽음
- 지정돼 있지 않으면 **아무 초기화 파일도 읽지 않는다**
- `~/.bashrc` 상단에 아래 코드가 있는 이유가 이것 — 비대화형이면 즉시 빠져나감

```bash
# .bashrc 상단 (배포판 기본)
[ -z "$PS1" ] && return          # 또는
case $- in *i*) ;; *) return;; esac
```

- `$PS1` : 프롬프트 문자열. **비대화형 셸에는 설정되지 않음** → 비어 있으면 대화형이 아님
- `$-` : 현재 셸에 켜진 옵션 문자 모음. **`i` 가 있으면 대화형**
- `-z` : 문자열 길이가 **z**ero 인지 검사

### 현재 셸 판별

```bash
shopt -q login_shell && echo "로그인 셸" || echo "비로그인 셸"
[[ $- == *i* ]] && echo "대화형" || echo "비대화형"
echo $0
```

- `shopt` = **sh**ell **opt**ions → bash 동작 옵션 조회·설정
  - `-q` : **q**uiet — 출력 없이 종료 코드로만 결과 전달. 조건문과 조합할 때 사용
- `$0` : 로그인 셸이면 관례적으로 **`-bash`** 처럼 앞에 하이픈이 붙음. 비로그인은 `bash`
- `$-` 예시 값 `himBHs` — `i` 가 포함되면 대화형

### 상황별 정리 (실제로 겪는 경우)

| 상황 | 로그인 | 대화형 | 읽는 파일 |
| --- | --- | --- | --- |
| SSH 로 접속 (`ssh srv01`) | ✓ | ✓ | profile 계열 전부 |
| UTM 콘솔·직렬 포트 로그인 | ✓ | ✓ | profile 계열 전부 |
| `su -` / `sudo -i` | ✓ | ✓ | profile 계열 전부 |
| 접속 후 `bash` 실행 | ✗ | ✓ | `~/.bashrc` 만 |
| `su` / `sudo -s` | ✗ | ✓ | `~/.bashrc` 만 |
| `tmux`·`screen` 새 창 | ✗ | ✓ | `~/.bashrc` 만 |
| `./script.sh` 실행 | ✗ | ✗ | 없음 (`$BASH_ENV` 없으면) |
| `ssh srv01 'hostname'` | ✗ | ✗ | 없음 |
| **cron 작업** | ✗ | ✗ | 없음 |

> macOS 의 Terminal·iTerm 은 새 창을 **로그인 셸로** 여는 반면, 리눅스 데스크톱의 터미널은 **비로그인 셸로** 연다. 같은 설정을 넣었는데 한쪽만 동작하는 원인이 대개 이것

### 실무에서 갈리는 지점

#### cron 의 PATH 문제 (최빈출 사고)

cron 작업은 **비로그인·비대화형**이라 초기화 파일을 하나도 읽지 않는다. 그래서 `PATH` 가 극도로 짧다.

```text
PATH=/usr/bin:/bin
```

- 터미널에서 잘 되던 스크립트가 cron 에서만 `command not found` 로 실패하는 전형적 원인
- 해결책 세 가지

```bash
# ① 스크립트 안에서 PATH 를 직접 선언
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# ② 명령을 절대경로로 작성
/usr/sbin/xfs_growfs /srv/share

# ③ crontab 파일 상단에 PATH 지정
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
30 3 * * * root /usr/local/bin/backup.sh
```

절차서 [[LAB/06-process-scheduling-diagnosis]] 7절에서 실습한다.

#### 무엇을 어디에 넣을 것인가

| 넣을 내용 | 위치 | 이유 |
| --- | --- | --- |
| 환경변수·`PATH` (개인) | `~/.bash_profile` | 로그인 시 한 번만 설정하면 **자식 프로세스가 상속** |
| alias·함수·프롬프트 | `~/.bashrc` | **alias 는 상속되지 않아** 모든 대화형 셸에서 매번 정의해야 함 |
| 환경변수 (전 사용자) | `/etc/profile.d/이름.sh` | `/etc/profile` 직접 수정은 패키지 업데이트 시 덮어쓰기 위험 |
| alias (전 사용자) | `/etc/bashrc` | 비로그인 대화형 셸까지 적용 |
| 신규 계정에 기본 제공 | `/etc/skel/.bashrc` | 계정 생성 시 홈으로 복사됨 ([[LAB/03-user-group-permission]] 1-3) |

**alias 가 상속되지 않는 이유** : alias 는 환경변수가 아니라 **셸 내부 기능**이다. `export` 대상이 아니므로 자식 프로세스에 전달되지 않는다. 그래서 대화형 셸이 뜰 때마다 `~/.bashrc` 가 다시 정의해 주어야 한다.

#### 설정을 고친 뒤

```bash
source ~/.bashrc        # 현재 셸에 즉시 반영
exec bash -l            # 로그인 셸로 다시 시작
```

- `source` (= `.`) : **현재 셸 안에서** 실행 → 변경 사항이 남음
- `exec bash -l` : 현재 셸 **프로세스를 대체**하며 로그인 셸로 재시작. 새 창을 열지 않고 전체 초기화를 다시 거치고 싶을 때
  - `-l` = `--login` : 로그인 셸로 시작하라는 옵션

### 검증 실습

```bash
# 비로그인 대화형
bash
shopt -q login_shell && echo LOGIN || echo NONLOGIN     # → NONLOGIN
echo $0                                                  # → bash
exit

# 로그인
bash -l
shopt -q login_shell && echo LOGIN || echo NONLOGIN     # → LOGIN
echo $0                                                  # → -bash
exit
```

각 초기화 파일 끝에 표시 문구를 넣어 **실제 읽히는 순서를 눈으로 확인**하는 실습이 [[LAB/01-vm-setup-and-inspection]] 7절에 있다.

> 📝 **시험 포인트** : 로그인 셸 초기화 **순서**(`/etc/profile` → `/etc/profile.d/*` → `~/.bash_profile` → `~/.bashrc` → `/etc/bashrc`)가 최빈출. `~/.bash_profile` · `~/.bash_login` · `~/.profile` 중 **먼저 발견된 하나만** 읽음. 종료 시 `~/.bash_logout`. 비로그인 셸은 `/etc/profile` 을 읽지 않음. cron 은 초기화 파일을 읽지 않아 `PATH` 를 직접 지정해야 함

관련 항목: [[#F-1. 환경변수·셸·tty 와 `su -` 가 환경을 초기화하는 이유]] · 절차서 [[LAB/01-vm-setup-and-inspection]] 7절, [[LAB/04-file-text-shell]] 7~8절

---

## F-3. `su`·`sudo -i` 를 반복하면 셸이 스택처럼 쌓이는가

**Q.** `sudo -i` 로 계속 셸을 바꾸고 `exit` 하면 스택처럼 이전 셸로 되돌아오는가.

**A.** 그렇다. 다만 스택이라는 자료구조가 따로 있는 것이 아니라, **프로세스 부모-자식 관계가 그렇게 동작하는 것**이다.

### 무슨 일이 벌어지는가

새 셸을 띄우면 기존 셸이 사라지는 것이 아니라, **자식 프로세스로 새 셸이 생기고 부모는 그대로 기다린다.**

```
bash (admin1)          ← ①  처음 접속한 로그인 셸
 └─ sudo               ← ②  권한 검사·세션 관리
     └─ bash (root)    ← ③  sudo -i 가 띄운 root 로그인 셸
         └─ su - dev1
             └─ bash (dev1)   ← ④  현재 여기
```

- 부모 셸은 자식이 끝나기를 **`wait` 상태로 대기**한다 → 살아 있지만 입력을 받지 않음
- 자식이 `exit` 하면 부모의 `wait` 가 풀리고 **부모가 다시 프롬프트를 표시**
- 그래서 나가는 순서가 들어온 순서의 역순 → **LIFO**(**L**ast **I**n **F**irst **O**ut), 곧 스택처럼 보임

### 눈으로 확인하기

```bash
pstree -p $$
ps -f --forest
echo "PID=$$  부모PID=$PPID"
whoami; id -un
```

- `pstree` : 프로세스를 **트리 형태**로 표시
  - `-p` : **p**ID 를 함께 표시. 어느 프로세스가 어느 프로세스의 자식인지 정확히 확인
  - `$$` : 현재 셸의 PID → 이 지점부터의 가지만 봄
- `ps -f --forest` : `ps` 로도 같은 계층을 표시
  - `-f` : **f**ull format — UID·PID·PPID·시작시각·명령 전체
  - `--forest` : 부모-자식 관계를 들여쓰기로 표현
- `$PPID` : **P**arent **P**rocess **ID** — 부모의 PID
- `whoami` : 현재 **실효 사용자**. 몇 겹 들어왔는지 헷갈릴 때 먼저 확인

### 깊이를 알려주는 변수 — `SHLVL`

bash 는 자기가 몇 번째로 중첩되었는지를 `SHLVL` 에 기록한다.

```bash
echo $SHLVL      # 1
bash
echo $SHLVL      # 2
bash
echo $SHLVL      # 3
exit; exit
echo $SHLVL      # 1
```

- **SHLVL** = **SH**ell **L**e**V**e**L** → 부모에게서 물려받은 값에 1 을 더해 설정
- 프롬프트에 표시해 두면 실수를 줄일 수 있다

```bash
PS1='[\u@\h \W](lv$SHLVL)\$ '
```

- `\u` : 사용자명(**u**ser), `\h` : 호스트명(**h**ost), `\W` : 현재 디렉터리 이름만, `\$` : root 면 `#`, 아니면 `$`

> ⚠️ `sudo -i` 와 `su -` 는 **환경을 초기화**하므로([[#F-1. 환경변수·셸·tty 와 `su -` 가 환경을 초기화하는 이유]]) `SHLVL` 이 이어지지 않고 다시 1 부터 시작할 수 있다. **중첩 깊이의 확실한 확인 수단은 `pstree -p`** 이며, `SHLVL` 은 같은 사용자로 `bash` 를 겹칠 때 정확하다. 실제 동작은 직접 확인할 것

### `exit` · `logout` · `Ctrl+D`

| 방법 | 동작 |
| --- | --- |
| `exit` | 현재 셸 종료. 어디서든 사용 가능 |
| `exit 3` | 종료 코드 3 으로 종료 → 부모에서 `echo $?` 로 확인 |
| `logout` | **로그인 셸에서만** 동작. 비로그인 셸에서는 `not login shell` 오류 |
| `Ctrl+D` | **EOF**(**E**nd **O**f **F**ile) 입력 → `exit` 와 같은 효과 |

- 가장 바깥 로그인 셸에서 `exit` 하면 더 돌아갈 부모가 없으므로 **세션 종료**(SSH 연결 끊김)
- `Ctrl+D` 로 실수로 로그아웃되는 것을 막으려면 `set -o ignoreeof` 또는 `export IGNOREEOF=3`

### 중첩되지 않게 하려면 — `exec`

```bash
exec su - dev1
```

- `exec` : 새 프로세스를 만들지 않고 **현재 프로세스를 지정한 명령으로 대체**
- 부모가 남지 않으므로 `exit` 해도 **돌아갈 곳이 없다** → 그대로 세션 종료
- 셸 깊이를 늘리지 않는 대신 되돌아올 수 없으므로, 되돌아올 필요가 없을 때만 사용
- `exec bash -l` 로 **현재 셸을 로그인 셸로 갈아끼우는** 용도로도 쓰임 ([[#F-2. 로그인 셸과 비로그인 셸의 차이]])

### 주의할 점

**① 환경 변경은 되돌아오지 않는다**

중첩된 셸에서 `export` 한 값은 그 셸이 끝나면 사라진다. 환경변수 상속이 **부모 → 자식 단방향**이기 때문이다.

```bash
export FOO=bar
exit
echo $FOO        # → 빈 값
```

**② 어느 사용자인지 헷갈리기 쉽다**

`sudo -i` 로 root 가 된 상태를 잊고 위험한 명령을 실행하는 사고가 잦다. 작업 전 습관적으로 확인한다.

```bash
whoami; pwd; echo $SHLVL
```

**③ 프로세스가 계속 쌓인다**

중첩 셸 하나하나가 실제 프로세스다. 수십 겹 쌓을 일은 없지만, 스크립트가 재귀적으로 셸을 띄우면 자원을 잠식한다. `pstree` 로 확인하고 정리한다.

**④ 로그에는 전환 기록이 남는다**

`su`·`sudo` 사용은 `/var/log/secure` 에 기록되므로, 중첩 이력은 사후 추적이 가능하다 ([[LAB/10-security-firewall-selinux]] 6절).

```bash
sudo grep -E 'sudo|su\[' /var/log/secure | tail
```

> 📝 **시험 포인트** : `exit` 는 현재 셸 종료, `logout` 은 **로그인 셸 전용**, `Ctrl+D` 는 EOF 로 `exit` 과 동일. `exec` 는 현재 프로세스를 **대체**하므로 복귀 불가. 자식 셸의 환경 변경은 부모에 반영되지 않음. `$$`·`$PPID`·`SHLVL` 의 의미

관련 항목: [[#F-1. 환경변수·셸·tty 와 `su -` 가 환경을 초기화하는 이유]] · [[#C-1. wheel 그룹이란 무엇이며 sudo 와 어떤 관계인가]] · 절차서 [[LAB/01-vm-setup-and-inspection]] 2-4, [[LAB/06-process-scheduling-diagnosis]] 1절

---

## F-4. `pstree` 출력 — 최소 설치 Rocky 9 의 프로세스 전수 해설

**Q.** `pstree` 출력에 나온 각 프로세스는 무엇인가.

**A.** 최소 설치 상태의 Rocky 9 에서 도는 것들이다. **systemd 가 뿌리이고 나머지는 전부 그 자손**이다.

### 먼저 읽는 법

| 표기 | 뜻 |
| --- | --- |
| `이름(숫자)` | 프로세스와 그 **PID** |
| `─┬─` `└─` | 부모-자식 관계 |
| `{이름}(숫자)` | **프로세스가 아니라 스레드**(thread). 중괄호가 표시 |
| `pstree(23+` | 화면 폭에 잘림. `-l` 옵션으로 전체 표시 |

- **스레드** : 한 프로세스 안에서 메모리를 공유하며 동시에 도는 실행 흐름. `NetworkManager(744)` 아래 `{NetworkManager}(746)(747)` 은 **별개 프로그램이 아니라 같은 프로그램의 작업 스레드**
- **PID 가 작을수록 먼저 시작** → `systemd-journal(570)` · `systemd-udevd(583)` 이 가장 이르고, `sshd-session(2303)` 처럼 큰 번호는 나중에 접속하며 생긴 것

### 1. 뿌리 — `systemd(1)`

- **PID 1**. 커널이 부팅 마지막에 직접 실행하는 **최초의 사용자 공간 프로세스**
- 나머지 모든 프로세스의 **조상**. 서비스를 순서·의존성에 맞춰 띄우고 감시
- 부모를 잃은 고아 프로세스를 **입양**해 정리하는 역할도 함
- 과거 SysV `init` 을 대체 ([[LAB/07-boot-systemd-log]])

### 2. 시스템 기반 서비스

| 프로세스 | 정체 | 역할 |
| --- | --- | --- |
| `systemd-journal(570)` | **journald** | 모든 로그를 바이너리로 수집·보관. `journalctl` 이 읽는 대상 |
| `systemd-udevd(583)` | **udev** **d**aemon | 장치 감지·장치 파일(`/dev/*`) 생성·이름 규칙 적용. `enp0s1` 이름이 붙는 것도 이 단계 |
| `systemd-logind(702)` | **login** **d**aemon | 로그인 세션·좌석(seat) 관리, 전원 키 처리 |
| `dbus-broker-lau(692)` → `dbus-broker(693)` | **D-Bus** 메시지 버스 | 프로세스 간 통신 창구. `lau` 는 launcher(실행기)가 이름 길이로 잘린 것 |
| `auditd(666)` | **audit** **d**aemon | 커널 감사 기록을 `/var/log/audit/audit.log` 로 저장. SELinux 거부 추적에 사용 ([[LAB/10-security-firewall-selinux]] 5절) |
| `rsyslogd(796)` | **r**ocket-fast **sys**tem **log** | 전통 텍스트 로그(`/var/log/messages`·`secure`) 작성. journald 와 병행 |
| `irqbalance(697)` | **IRQ** balance | **I**nterrupt **R**e**Q**uest(하드웨어 인터럽트)를 여러 CPU 코어에 분산해 한 코어 쏠림 방지 |

- **journald 와 rsyslog 가 둘 다 도는 이유** : journald 는 구조화된 바이너리 로그, rsyslog 는 사람이 읽고 원격 전송하기 쉬운 텍스트 로그. 시험은 양쪽 다 출제

### 3. 네트워크·시간·방화벽

| 프로세스 | 정체 | 역할 |
| --- | --- | --- |
| `NetworkManager(744)` | 네트워크 관리자 | 인터페이스·IP·DNS·경로 설정. `nmcli` 가 이 데몬에 지시 ([[#D-3. `ip addr` 출력 전체 해설]]) |
| `chronyd(700)` | **chrony** **d**aemon | NTP 시간 동기화. 과거 `ntpd` 를 대체 |
| `firewalld(696)` | 방화벽 관리자 | nftables 규칙을 존(zone) 단위로 관리. `firewall-cmd` 가 지시 |

### 4. 작업 예약

| 프로세스 | 역할 |
| --- | --- |
| `crond(779)` | **cron** **d**aemon — 정해진 시각에 작업 실행. `/etc/crontab`·`/etc/cron.d/`·사용자 crontab 감시 ([[LAB/06-process-scheduling-diagnosis]] 7절) |

- `atd` 는 목록에 없음 → **아직 설치·활성화하지 않은 상태**. Part 06 에서 켠다

### 5. 로그인 경로 두 갈래 — 여기가 핵심

이 출력에는 **콘솔 로그인**과 **원격 로그인**이 동시에 잡혀 있다.

#### ① 콘솔 (직렬 포트 / 디스플레이)

```text
├─agetty(780)
├─login(1865)───bash(1888)
```

- `agetty` = **a**lternative **getty**. **getty** 는 **get** **tty** → "터미널을 잡아 로그인 프롬프트를 띄우는 프로그램"
- 동작 순서 : `agetty` 가 tty 를 열고 대기 → 사용자가 이름 입력 → **`login` 으로 자기 자신을 대체**(`exec`) → 인증 성공 시 셸을 자식으로 실행
- 그래서 목록에 **둘이 따로 보인다**
  - `agetty(780)` : 아직 아무도 로그인하지 않은 tty 에서 **대기 중**
  - `login(1865)→bash(1888)` : 다른 tty 에서 **로그인이 끝난 세션**
- 직렬 포트를 붙이고 `console=ttyAMA0` 을 지정했으므로([[#A-3. `Display output is not active.` 가 뜨는 이유]]) `serial-getty@ttyAMA0` 와 디스플레이 쪽 getty 가 함께 뜬 상태
- `agetty` 가 `exec` 로 자신을 대체하는 것이 [[#F-3. `su`·`sudo -i` 를 반복하면 셸이 스택처럼 쌓이는가]] 에서 설명한 `exec` 의 실제 사용 예

#### ② 원격 (SSH) — 지금 이 세션

```text
└─sshd(774)───sshd-session(2303)───sshd-session(2308)───bash(2309)───su(2355)───bash(2359)───pstree
```

| 단계 | 프로세스 | 역할 |
| --- | --- | --- |
| 1 | `sshd(774)` | 22번 포트에서 **접속을 기다리는 부모 데몬**. 부팅 시 시작 |
| 2 | `sshd-session(2303)` | 접속 하나마다 생기는 **권한 분리용 자식**. 인증 전이라 낮은 권한 |
| 3 | `sshd-session(2308)` | 인증 성공 후 **그 사용자 권한으로** 전환된 세션 |
| 4 | `bash(2309)` | 로그인 셸 |
| 5 | `su(2355)` | `su` 실행 |
| 6 | `bash(2359)` | `su` 가 띄운 새 셸 — **현재 여기** |
| 7 | `pstree` | 방금 실행한 명령 |

- **권한 분리**(privilege separation) : 인증 전 코드를 낮은 권한 프로세스에 가둬, 취약점이 있어도 root 권한을 얻지 못하게 하는 설계. OpenSSH 의 오랜 보안 관행 (프로세스 이름은 버전에 따라 `sshd` 로만 표시되기도 함)
- **4→5→6 이 `su` 중첩의 실물** — `exit` 하면 6 이 죽고 4 로 돌아간다

### 6. 사용자별 systemd 인스턴스

```text
├─systemd(1881)───(sd-pam)(1883)
```

- 사용자가 로그인하면 **그 사용자 전용 systemd 인스턴스**가 뜬다 (`systemd --user`)
- 사용자 단위 서비스·타이머를 관리하는 용도
- `(sd-pam)` : **s**ystem**d**-**PAM** 연동 보조 프로세스. 괄호는 **커널이 인식하는 이름과 실행 파일 이름이 다름**을 뜻함
- PID 1881 이 `login(1865)` 과 가까운 번호 → **콘솔 로그인 시점에 함께 생성**된 것

### 7. 지금 없는 것들

최소 설치라 아직 없다. 각 파트에서 설치하며 이 목록이 늘어난다.

| 없는 프로세스 | 추가되는 시점 |
| --- | --- |
| `httpd` · `named` · `smbd` · `nmbd` · `vsftpd` · `master`(postfix) | [[LAB/09-network-services]] |
| `dockerd` · `containerd` · `libvirtd` | [[LAB/11-container-virtualization]] |
| `atd` · `sysstat` 계열 | [[LAB/06-process-scheduling-diagnosis]] |

### 유용한 옵션

```bash
pstree -p            # PID 표시
pstree -u            # 사용자가 바뀌는 지점에 사용자명 표시
pstree -a            # 명령행 인자까지
pstree -l            # 긴 줄을 자르지 않음
pstree -s <PID>      # 그 프로세스의 조상 계보만
pstree -T            # 스레드 숨김 (중괄호 항목 제거)
pstree admin1        # 특정 사용자의 프로세스만
```

- `-p` : **p**ID. 어느 것이 부모인지 정확히 볼 때 필수
- `-u` : **u**ser — 권한이 바뀌는 경계를 확인. `su`·`sudo` 추적에 유용
- `-a` : **a**rguments — 같은 이름의 프로세스가 여럿일 때 무엇을 실행 중인지 구분
- `-s` : **s**how parents — "이 프로세스가 왜 떠 있는가" 를 거슬러 올라가 확인
- `-T` : 스레드를 빼고 프로세스만 → 목록이 훨씬 간결해짐

### 대응 명령

```bash
ps -ef --forest              # 같은 계층을 ps 로
ps -eo pid,ppid,user,tty,stat,cmd --forest
systemctl list-units --type=service --state=running
systemd-cgls                 # cgroup(서비스 단위) 기준 트리
```

- `systemd-cgls` : systemd 의 **c**ontrol **g**roup **l**i**s**t → 서비스 단위로 묶어 보여주므로 "어느 유닛이 어느 프로세스를 갖고 있는가" 파악에 유리

> 📝 **시험 포인트** : **PID 1 은 systemd**(과거 `init`). `getty`/`agetty` 는 터미널에 로그인 프롬프트를 띄우는 프로그램. `journald`(바이너리) 와 `rsyslogd`(텍스트)의 역할 구분. 고아 프로세스는 **PID 1 이 입양**. `pstree` 의 중괄호는 스레드

관련 항목: [[#F-1. 환경변수·셸·tty 와 `su -` 가 환경을 초기화하는 이유]] · [[#F-3. `su`·`sudo -i` 를 반복하면 셸이 스택처럼 쌓이는가]] · 절차서 [[LAB/06-process-scheduling-diagnosis]] 1절, [[LAB/07-boot-systemd-log]] 3절

---

# G. 명령어·데몬 구조

## G-1. `~d` 데몬과 `~ctl` 명령의 관계

**Q.** `~ctl` 이 붙은 명령어와 `~d` 가 붙은 프로세스는 무엇인가. 명령어가 곧 프로세스인가. 둘은 어떤 관계인가.

**A.** 먼저 전제를 하나 정정해야 한다. **명령어와 프로세스는 같은 것이 아니다.** 그리고 `~d` 와 `~ctl` 은 **서버와 클라이언트** 관계다.

---

### 1. 명령어 ≠ 프로세스

| 구분 | 정체 | 상태 |
| --- | --- | --- |
| **명령어**(프로그램) | 디스크에 저장된 **실행 파일** | 정적. 실행되기 전에는 그냥 파일 |
| **프로세스** | 그 파일이 **메모리에 올라가 실행 중인 상태** | 동적. PID·메모리·환경변수를 가짐 |

```bash
file /usr/bin/sshd 2>/dev/null || file /usr/sbin/sshd    # 파일로서의 실체
which sshd; ls -l $(which sshd)                          # 디스크 위치
pgrep -a sshd                                            # 실행 중인 프로세스
```

- `file` : 파일의 종류 판별 → `ELF 64-bit LSB executable` 처럼 실행 파일임을 확인
  - **ELF** = **E**xecutable and **L**inkable **F**ormat, 리눅스 실행 파일 형식
- `pgrep -a` : 이름으로 프로세스를 찾아 **a**rguments(명령행) 까지 표시

**하나의 프로그램에서 여러 프로세스가 생길 수 있다.** `bash` 실행 파일은 하나지만 [[#F-4. `pstree` 출력 — 최소 설치 Rocky 9 의 프로세스 전수 해설]] 의 트리에는 `bash` 프로세스가 여러 개 떠 있었다. 붕어빵 틀(프로그램) 하나로 붕어빵(프로세스) 여러 개를 굽는 것과 같다.

---

### 2. `~d` — 데몬(daemon)

- **daemon** : 배경에서 **계속 상주하며** 요청을 기다리는 서비스 프로세스
- 어원은 그리스 신화의 **δαίμων**(다이몬) — 보이지 않는 곳에서 일하는 **수호신**. 악마(demon)가 아님
- 관례적으로 이름 끝에 **`d`** 를 붙인다 : `sshd` = SSH **d**aemon

#### 데몬의 특징

| 특징 | 확인 방법 |
| --- | --- |
| 제어 터미널이 없음 | `ps -eo tty` 에서 **`?`** 로 표시 |
| 부모가 `systemd(1)` | `ps -eo ppid` 가 대부분 1 |
| 부팅 시 자동 시작 | `systemctl is-enabled <서비스>` |
| 로그를 파일·journald 로 보냄 | 터미널이 없으니 화면에 못 찍음 |

```bash
ps -eo pid,ppid,tty,user,cmd | awk '$3=="?"' | head
```

- 터미널이 없으므로 [[#F-1. 환경변수·셸·tty 와 `su -` 가 환경을 초기화하는 이유]] 에서 다룬 **SIGHUP 으로 죽지 않는다** → 로그아웃해도 계속 동작
- 오히려 데몬에게 `SIGHUP` 은 관례적으로 **"설정 파일을 다시 읽어라"** 라는 신호로 재정의됨

#### 주의 — 모든 데몬이 `d` 로 끝나지는 않는다

| 서비스 | 데몬 프로세스 이름 |
| --- | --- |
| Postfix | `master` (하위에 `qmgr`·`pickup`) |
| Nginx | `nginx` |
| Dovecot | `dovecot` |
| Squid | `squid` |
| Docker | `dockerd` · `containerd` |
| Samba | `smbd` · `nmbd` · `winbindd` |

---

### 3. `~ctl` — 컨트롤(control) 명령

- **ctl** = **c**on**t**ro**l** 의 축약
- 데몬에게 **명령을 전달하고 결과를 받아 출력한 뒤 즉시 종료**하는 **클라이언트 도구**
- 상주하지 않는다. `systemctl status` 를 실행하면 그 순간만 프로세스가 생겼다 사라진다

```bash
systemctl status sshd     # 실행 → 출력 → 종료
pgrep systemctl           # 아무것도 안 나옴 (이미 끝났으므로)
```

---

### 4. 둘의 관계 — 클라이언트 · 서버

```
사용자 ──▶ systemctl (클라이언트, 잠깐 실행)
              │  요청 전달 (D-Bus / 소켓 / 시그널)
              ▼
           systemd (데몬, 상주)
              │  실제 작업 수행
              ▼
           sshd 시작·중지·상태 보고
```

- **데몬이 실제 일을 하고, ctl 은 그 데몬에게 부탁만 한다**
- 그래서 **데몬이 죽어 있으면 ctl 명령도 실패**한다

```bash
systemctl stop firewalld
firewall-cmd --list-all      # → "FirewallD is not running" 오류
```

#### 통신 수단

| 방식 | 사용하는 도구 |
| --- | --- |
| **D-Bus** (프로세스 간 메시지 버스) | `systemctl` · `hostnamectl` · `nmcli` · `firewall-cmd` |
| **유닉스 도메인 소켓** (파일 형태의 통신구) | `docker`(`/var/run/docker.sock`) · `chronyc` |
| **시그널** | `kill -HUP <PID>` → 설정 재읽기 |
| **네트워크 소켓** | `rndc`(BIND, 953번 포트) |

#### 왜 분리하는가

- **권한 분리** : 데몬은 root 권한으로 상주하고, 사용자는 일반 권한의 ctl 로 **요청만** 보낸다. 사용자가 직접 시스템을 건드리지 않음
- **인증** : 요청이 들어오면 데몬이 **polkit** 등으로 권한을 확인. 일반 사용자가 `systemctl restart sshd` 를 실행하면 비밀번호를 묻는 이유
- **일관성** : 여러 사용자가 동시에 요청해도 데몬 한 곳에서 순서를 정리

---

### 5. `~ctl` 명령 전체 목록

#### systemd 계열 (대부분 `~ctl`)

| 명령 | 짝이 되는 데몬 | 역할 |
| --- | --- | --- |
| `systemctl` | `systemd`(PID 1) | 서비스·타겟 시작·중지·활성화·상태 조회 |
| `journalctl` | `systemd-journald` | 통합 로그 조회 |
| `hostnamectl` | `systemd-hostnamed` | 호스트명·머신 정보 조회·설정 |
| `timedatectl` | `systemd-timedated` | 시간·시간대·NTP 설정 |
| `localectl` | `systemd-localed` | 로케일·키맵 설정 |
| `loginctl` | `systemd-logind` | 로그인 세션·사용자 관리 |
| `coredumpctl` | `systemd-coredump` | 코어 덤프 조회 |
| `machinectl` | `systemd-machined` | 컨테이너·VM 관리 |
| `busctl` | `dbus-broker` | D-Bus 메시지 조회 |
| `resolvectl` | `systemd-resolved` | DNS 조회·캐시 (**RHEL 9 는 기본 비활성**) |
| `networkctl` | `systemd-networkd` | 네트워크 (**RHEL 9 는 NetworkManager 사용**) |

#### systemd 계열이 아닌 `~ctl`

| 명령 | 대상 | 특이점 |
| --- | --- | --- |
| `apachectl` | `httpd` | 데몬에 요청하는 것이 아니라 **직접 실행·제어하는 셸 스크립트**. `apachectl configtest` 는 설정 검사 |
| `sysctl` | 커널 | **데몬이 아니라 커널 파라미터**(`/proc/sys/`)를 다룸. 이름만 비슷 |

> ⚠️ `sysctl` 은 `~ctl` 이지만 데몬과 무관하다. **커널 설정값**을 읽고 쓰는 도구다 ([[LAB/10-security-firewall-selinux]] 7절)

#### `~ctl` 이 아닌 제어 도구 — 접미사가 다양하다

| 접미사 | 뜻 | 예시 | 대상 데몬 |
| --- | --- | --- | --- |
| `cli` | **c**ommand **l**ine **i**nterface | `nmcli` | `NetworkManager` |
| `cmd` | **c**o**mm**an**d** | `firewall-cmd` | `firewalld` |
| `c` | **c**lient / **c**ontrol | `chronyc` | `chronyd` |
| `sh` | **sh**ell | `virsh` | `libvirtd` |
| `adm` | **adm**in | `lpadmin` | `cupsd` |
| (없음) | — | `docker` | `dockerd` |
| `rndc` | **r**emote **n**ame **d**aemon **c**ontrol | `rndc` | `named` |

---

### 6. 시험 최빈출 — 4축 매칭표

**서비스 ↔ 데몬 ↔ 제어 명령 ↔ 설정 파일** 의 대응이 가장 많이 출제된다.

| 서비스 | 데몬 | 제어 명령 | 주 설정 파일 | 포트 |
| --- | --- | --- | --- | --- |
| SSH | `sshd` | `systemctl`, `sshd -t` | `/etc/ssh/sshd_config` | 22 |
| 웹 (Apache) | `httpd` | `apachectl`, `httpd -t` | `/etc/httpd/conf/httpd.conf` | 80·443 |
| DNS | `named` | `rndc`, `named-checkconf` | `/etc/named.conf` | 53 |
| 방화벽 | `firewalld` | `firewall-cmd` | `/etc/firewalld/` | — |
| 네트워크 | `NetworkManager` | `nmcli`, `nmtui` | `/etc/NetworkManager/` | — |
| 시간 | `chronyd` | `chronyc`, `timedatectl` | `/etc/chrony.conf` | 123 |
| 로그 | `rsyslogd` | `logger`, `rsyslogd -N1` | `/etc/rsyslog.conf` | 514 |
| 로그(journal) | `systemd-journald` | `journalctl` | `/etc/systemd/journald.conf` | — |
| 예약 실행 | `crond` | `crontab` | `/etc/crontab` | — |
| 파일 공유 (SMB) | `smbd`·`nmbd` | `smbcontrol`, `testparm` | `/etc/samba/smb.conf` | 139·445 |
| 파일 공유 (NFS) | `nfsd` | `exportfs`, `showmount` | `/etc/exports` | 2049 |
| FTP | `vsftpd` | `systemctl` | `/etc/vsftpd/vsftpd.conf` | 21 |
| 메일 | `master`(postfix) | `postfix`, `postconf` | `/etc/postfix/main.cf` | 25 |
| 인쇄 | `cupsd` | `lpadmin`, `lpstat` | `/etc/cups/cupsd.conf` | 631 |
| 감사 | `auditd` | `auditctl`, `ausearch` | `/etc/audit/auditd.conf` | — |
| 컨테이너 | `dockerd` | `docker` | `/etc/docker/daemon.json` | — |
| 가상화 | `libvirtd` | `virsh` | `/etc/libvirt/` | — |

- 이 표가 [[LAB/09-network-services]] 전체와 [[THEORY/network-service]] 의 핵심

---

### 7. 실습으로 확인

```bash
# 데몬은 터미널이 없다 (tty 열이 ?)
ps -eo pid,ppid,tty,cmd | grep -E 'sshd|crond|chronyd' | grep -v grep

# ctl 은 실행 후 즉시 사라진다
systemctl is-active sshd; pgrep systemctl || echo "systemctl 프로세스 없음"

# 데몬을 멈추면 ctl 이 실패한다
sudo systemctl stop chronyd
chronyc tracking            # → 연결 실패
sudo systemctl start chronyd
chronyc tracking            # → 정상

# 설정 재읽기 — 재시작 없이
sudo systemctl reload sshd  # 내부적으로 SIGHUP 전달
```

- `is-active` : 실행 중인지만 확인. 스크립트에서 조건 판별에 사용
- `reload` 와 `restart` 의 차이 — `reload` 는 **프로세스를 죽이지 않고** 설정만 다시 읽음 → **연결이 끊기지 않는다**. `restart` 는 완전 재시작이라 기존 세션이 끊길 수 있음

> 📝 **시험 포인트** : `d` 는 **daemon**, `ctl` 은 **control**. 데몬은 상주·터미널 없음(`tty=?`)·PID 1 의 자식. ctl 은 클라이언트라 실행 후 종료. **데몬이 멈추면 ctl 도 실패**. `reload`(설정만) 와 `restart`(완전 재시작) 구분. `sysctl` 은 데몬이 아니라 **커널 파라미터** 도구. 서비스↔데몬↔설정파일↔포트 **4축 매칭**이 최빈출

관련 항목: [[#F-4. `pstree` 출력 — 최소 설치 Rocky 9 의 프로세스 전수 해설]] · 절차서 [[LAB/07-boot-systemd-log]] 3절, [[LAB/09-network-services]] 전체

---

# H. 로그·커널

## H-1. 커널 링 버퍼란 무엇인가

**Q.** 커널 링 버퍼란 무엇인가.

**A.** **커널이 자기 메시지를 적어 두는, 메모리 안의 고정 크기 기록장**이다. `dmesg` 로 읽는 그 내용이다.

---

### 1. "링 버퍼" 라는 자료구조

- **ring buffer**(원형 버퍼, circular buffer) : 끝과 처음이 이어진 **고정 크기** 저장 공간
- 가득 차면 **가장 오래된 기록부터 덮어쓴다** → 크기가 무한정 늘지 않음
- 회전 초밥 벨트와 같다. 자리가 정해져 있고, 새 접시를 올리려면 가장 오래된 접시가 내려간다

```
   ┌───────────────────────┐
   │ 새 메시지 ──▶ [ ][ ][ ] │
   │              ▲       │
   │              └─ 오래된 것부터 덮어씀
   └───────────────────────┘
```

이 구조를 쓰는 이유는 **커널이 로그 때문에 메모리를 무한정 소모하면 안 되기 때문**이다. 대신 오래된 메시지는 사라진다 — **오래 켜 둔 서버에서 부팅 메시지가 안 보이는 이유**가 이것이다.

---

### 2. 왜 파일이 아니라 메모리인가

커널은 **부팅 아주 초기부터** 메시지를 남겨야 한다. 그런데 그 시점에는

- 아직 **파일시스템이 마운트되지 않았고** ([[#B-2. 마운트란 무엇인가]])
- 로그 데몬(`rsyslogd`·`systemd-journald`)도 **아직 실행되지 않았다**

디스크에 쓸 방법이 없으므로 **메모리에 쌓아 두고**, 나중에 사용자 공간이 준비되면 로그 데몬이 이를 읽어 파일로 옮긴다.

- 커널 내부에서 메시지를 남기는 함수가 **`printk()`** (커널판 `printf`)
- 크기는 커널 빌드 옵션 `CONFIG_LOG_BUF_SHIFT` 로 정해지며, 부팅 파라미터 `log_buf_len=1M` 으로 조정 가능

---

### 3. 읽는 방법

```bash
dmesg
dmesg -T
dmesg -l err,warn
dmesg -w
journalctl -k
journalctl -k -b -1
```

- `dmesg` = **d**iagnostic **mes**sa**g**es → 커널 링 버퍼의 내용을 출력
- `-T` : 타임스탬프를 **사람이 읽는 시각**으로 변환 (기본값은 부팅 후 경과 초)
  - **가장 먼저 붙이게 되는 옵션.** `[12.345678]` 만 봐서는 언제인지 알 수 없음
- `-l` : **l**evel — 우선순위로 필터 (`emerg,alert,crit,err,warn,notice,info,debug`)
- `-w` : **w**ait/follow — 새 메시지를 실시간으로 이어서 출력. `tail -f` 와 같은 감각
- `-H` : **H**uman — 색상·페이저 적용해 보기 좋게
- `-k` : **k**ernel 메시지만 (`/dev/kmsg` 에 사용자 공간이 쓴 것 제외)
- `-x` : 각 줄에 **시설(facility)·수준(level)** 을 문자로 표시
- `-C` : 버퍼를 **C**lear (⚠️ 기록이 사라짐), `-c` : 출력 후 지움
- `-n <레벨>` : 콘솔에 출력할 최소 수준 지정

`journalctl -k` 는 **journald 가 수집한 커널 메시지**를 보여준다. `-b -1` 처럼 **이전 부팅의 것도 조회**할 수 있다는 점이 `dmesg` 와의 결정적 차이다 (링 버퍼는 재부팅하면 비워짐).

#### 접근 권한

```bash
sysctl kernel.dmesg_restrict
```

- `1` 이면 일반 사용자는 `dmesg` 를 읽을 수 없다 → `sudo dmesg` 필요
- 커널 메시지에 메모리 주소 등 공격에 쓰일 정보가 노출될 수 있어 제한하는 것
- 배포판·버전마다 기본값이 다르므로 실제 값 확인 필요

---

### 4. 로그 수준 8단계

`printk` 로 남기는 모든 메시지에는 우선순위가 붙는다.

| 번호 | 이름 | 뜻 |
| --- | --- | --- |
| 0 | `emerg` | 시스템 사용 불가 |
| 1 | `alert` | 즉시 조치 필요 |
| 2 | `crit` | 심각한 오류 |
| 3 | `err` | 오류 |
| 4 | `warn` | 경고 |
| 5 | `notice` | 정상이지만 주목할 일 |
| 6 | `info` | 정보 |
| 7 | `debug` | 디버깅용 |

- **숫자가 작을수록 심각**하다
- `rsyslog` 의 우선순위 체계와 동일 ([[LAB/07-boot-systemd-log]] 6절)
- `dmesg -l err,crit` 처럼 심각한 것만 골라 보는 것이 실무 습관

---

### 5. 무엇이 기록되는가 — 실전 활용

| 상황 | 확인 명령 | 실습 파트 |
| --- | --- | --- |
| 디스크를 추가했는데 안 보임 | `dmesg -T \| grep -i vd` | [[LAB/05-disk-lvm-raid-swap-quota]] 1절 |
| 메모리 부족으로 프로세스가 강제 종료됨 | `dmesg -T \| grep -i 'out of memory'` | [[LAB/06-process-scheduling-diagnosis]] 5절 |
| 파일시스템·I/O 오류 | `dmesg -l err,crit` | [[LAB/05-disk-lvm-raid-swap-quota]] 8절 |
| 부팅 시 하드웨어 인식 확인 | `dmesg -T \| head -40` | [[LAB/01-vm-setup-and-inspection]] 3-7 |
| 커널 모듈 적재·오류 | `dmesg -T \| tail` (modprobe 직후) | [[LAB/05-disk-lvm-raid-swap-quota]] 10절 |
| 방화벽이 버린 패킷 로그 | `dmesg -w` (LOG 타깃) | [[LAB/10-security-firewall-selinux]] 3절 |

**OOM Killer** 가 대표적이다. 메모리가 바닥나면 커널이 프로세스를 골라 강제 종료하는데, 그 판단 근거와 대상이 링 버퍼에만 남는다. 애플리케이션 로그에는 "왜 죽었는지" 가 없어서, `dmesg` 를 봐야 원인을 알 수 있다.

- **OOM** = **O**ut **O**f **M**emory

---

### 6. 링 버퍼와 로그 파일의 관계

```
커널 (printk)
   │
   ▼
커널 링 버퍼 (메모리, 고정 크기, 재부팅 시 소멸)
   │
   ├──▶ dmesg              직접 읽기
   ├──▶ systemd-journald ──▶ /var/log/journal/  (journalctl -k)
   └──▶ rsyslogd ─────────▶ /var/log/messages   (kern.* 규칙)
                          └▶ /var/log/dmesg     (부팅 시점 스냅샷)
```

| 경로 | 특징 |
| --- | --- |
| `dmesg` | **현재 버퍼**만. 재부팅하면 사라지고, 오래되면 덮어써짐 |
| `journalctl -k` | 디스크에 저장 → **이전 부팅 기록도 조회 가능** (`-b -1`) |
| `/var/log/messages` | rsyslog 가 텍스트로 기록. `grep` 하기 쉬움 |
| `/var/log/dmesg` | **부팅 완료 시점의 스냅샷** 파일 (서비스가 활성인 경우) |

- journald 를 영구 저장으로 설정해 두면(`Storage=persistent`) 이전 부팅 로그가 남는다 ([[LAB/07-boot-systemd-log]] 5절)
- **장애 분석에서 `journalctl -k -b -1` 이 결정적**인 이유 — 서버가 죽어서 재부팅됐을 때, 죽기 직전의 커널 메시지를 볼 수 있는 유일한 수단

---

### 7. 관련 인터페이스

```bash
ls -l /dev/kmsg
sudo cat /proc/kmsg | head        # (읽으면 소비됨 — 주의)
echo "테스트 메시지" | sudo tee /dev/kmsg
dmesg | tail -1
```

- `/dev/kmsg` : 링 버퍼에 **읽고 쓸 수 있는** 장치 파일. 사용자 공간에서 메시지를 넣을 수도 있음
- `/proc/kmsg` : 로그 데몬이 사용하는 전용 통로. **한 번 읽으면 소비**되므로 직접 읽지 말 것
- `tee` : 표준 입력을 파일과 화면에 동시에 출력. `sudo` 와 조합해 **권한이 필요한 파일에 쓸 때** 사용 (`sudo echo > 파일` 은 리다이렉션이 sudo 밖에서 일어나 실패)

> 📝 **시험 포인트** : `dmesg` 는 **커널 링 버퍼**를 출력. 부팅 메시지 확인 명령으로 출제. 링 버퍼는 **고정 크기·순환**이라 오래된 것이 덮어써지고 **재부팅 시 소멸**. 영구 기록은 `/var/log/dmesg`·`/var/log/messages`·journald. `journalctl -k` 는 `dmesg` 와 동등하되 **이전 부팅 조회 가능**. 로그 수준 8단계는 숫자가 **작을수록 심각**

관련 항목: [[#F-4. `pstree` 출력 — 최소 설치 Rocky 9 의 프로세스 전수 해설]] · [[#G-1. `~d` 데몬과 `~ctl` 명령의 관계]] · 절차서 [[LAB/01-vm-setup-and-inspection]] 3-7, [[LAB/07-boot-systemd-log]] 5~6절

---

# I. 로케일·국제화

## I-1. 로케일이란 무엇인가

**Q.** 로케일이란 무엇인가.

**A.** **언어와 지역에 따라 달라지는 표기 관습을 모아 둔 설정 묶음**이다. 같은 프로그램이라도 로케일에 따라 날짜·숫자·정렬·메시지가 다르게 나온다.

- **locale** : 영어로 "장소·현장" 을 뜻하는 단어
- 프로그램을 여러 언어권에서 쓸 수 있게 만드는 **국제화**(**i18n** = **i**nternationalizatio**n**, 사이 글자 18개)의 핵심 장치

---

### 1. 로케일이 결정하는 것

| 항목 | 예 (`C` 로케일) | 예 (`ko_KR.UTF-8`) |
| --- | --- | --- |
| 날짜 표기 | `Thu Sep  4 13:00:00 2026` | `2026년 09월 04일 목요일 13시 00분` |
| 숫자 소수점 | `1234.56` | `1234.56` (유럽 일부는 `1234,56`) |
| 통화 | `$` | `₩` |
| 정렬 순서 | 바이트 값 순 | 한글 가나다순 |
| 오류 메시지 | `No such file or directory` | `그런 파일이나 디렉터리가 없습니다` |
| 문자 인코딩 | ASCII | UTF-8 (한글 표현 가능) |

---

### 2. 로케일 이름의 구조

```
ko_KR.UTF-8
│  │   └── 문자 인코딩
│  └────── 국가·지역 코드 (ISO 3166-1, 대문자)
└───────── 언어 코드 (ISO 639-1, 소문자)
```

- **`ko`** : 한국어. 영어는 `en`, 일본어 `ja`
- **`KR`** : 대한민국. 미국 `US`, 영국 `GB`
- **`UTF-8`** = **U**nicode **T**ransformation **F**ormat, 8비트 단위 → 전 세계 문자를 담는 표준 인코딩
- 언어가 같아도 지역이 다르면 관습이 다르다 → `en_US`(월/일/년) vs `en_GB`(일/월/년)

#### 특별한 로케일 — `C` 와 `POSIX`

- **`C`**(= `POSIX`) : 어떤 지역에도 속하지 않는 **기본 로케일**. ASCII 문자, 바이트 순 정렬, 영어 메시지
- 번역·변환을 거치지 않아 **가장 빠르고 결과가 예측 가능**
- **스크립트에서 일부러 지정**하는 경우가 많다 (뒤의 5절)
- `C.UTF-8` : `C` 의 규칙에 UTF-8 인코딩만 더한 것

---

### 3. 환경변수 체계 — 우선순위가 있다

로케일은 [[#F-1. 환경변수·셸·tty 와 `su -` 가 환경을 초기화하는 이유]] 에서 다룬 **환경변수로 전달**된다.

```bash
locale
```

```text
LANG=ko_KR.UTF-8
LC_CTYPE="ko_KR.UTF-8"
LC_NUMERIC="ko_KR.UTF-8"
LC_TIME="ko_KR.UTF-8"
LC_COLLATE="ko_KR.UTF-8"
LC_MONETARY="ko_KR.UTF-8"
LC_MESSAGES="ko_KR.UTF-8"
...
LC_ALL=
```

#### 우선순위 (시험 출제)

```
LC_ALL  >  개별 LC_*  >  LANG
```

| 변수 | 역할 |
| --- | --- |
| **`LC_ALL`** | **모든 항목을 강제로 덮어씀.** 최우선. 평소에는 비워 둠 |
| **`LC_*`** (개별) | 특정 항목만 따로 지정 |
| **`LANG`** | 지정되지 않은 나머지 전부의 **기본값** |

즉 `LANG` 으로 전체를 정해 두고, 바꾸고 싶은 항목만 `LC_TIME` 처럼 개별 지정하는 방식이다. `LC_ALL` 은 **모든 설정을 무시하고 하나로 통일**하므로 일시적 강제 지정에만 쓴다.

#### 주요 `LC_*` 항목

| 변수 | 영향 |
| --- | --- |
| `LC_CTYPE` | 문자 분류·대소문자 변환·**인코딩**. 한글 입출력의 핵심 |
| `LC_COLLATE` | **정렬 순서**. `sort`·`ls` 결과가 달라짐 |
| `LC_TIME` | `date` 출력 형식, 요일·월 이름 |
| `LC_NUMERIC` | 소수점·천 단위 구분 기호 |
| `LC_MONETARY` | 통화 기호·자릿수 |
| `LC_MESSAGES` | **프로그램 메시지 언어** |
| `LC_PAPER` | 기본 용지 크기 (A4 / Letter) |

---

### 4. 조회·설정 명령

```bash
locale                     # 현재 로케일 전체
locale -a                  # 사용 가능한 로케일 목록
locale -a | grep ko        # 한국어 로케일이 설치돼 있는지
localectl status           # 시스템 전역 설정
localectl list-locales     # 설치된 로케일 목록
```

- `locale` : **현재 셸의** 로케일 관련 환경변수 출력
- `-a` : **a**ll — 시스템에 **설치된** 로케일 전부 나열
- `localectl` : systemd 의 로케일 제어 도구 ([[#G-1. `~d` 데몬과 `~ctl` 명령의 관계]] 의 `~ctl` 계열)

#### 일시 변경 — 해당 명령에만 적용

```bash
LANG=C date
LC_TIME=ko_KR.UTF-8 date
LC_ALL=C ls -l
```

- 명령 **앞에** 변수를 붙이면 **그 명령의 환경에만** 적용되고 셸에는 남지 않음
- 상속 구조상 자식 프로세스에만 전달되기 때문 ([[#F-1. 환경변수·셸·tty 와 `su -` 가 환경을 초기화하는 이유]])

#### 현재 셸에만 적용

```bash
export LANG=ko_KR.UTF-8
```

#### 시스템 전역 영구 설정

```bash
sudo localectl set-locale LANG=ko_KR.UTF-8
cat /etc/locale.conf
```

- `/etc/locale.conf` : RHEL 7 이후의 시스템 전역 로케일 설정 파일
- 구형(RHEL 6 이하)은 `/etc/sysconfig/i18n` → **시험에서 경로를 바꿔 낸 선지 주의**
- 로그인 시 `/etc/profile` 계열이 이 파일을 읽어 환경변수로 설정 ([[#F-2. 로그인 셸과 비로그인 셸의 차이]])

#### 한국어 로케일이 없을 때

RHEL 8 이후로는 로케일이 **패키지로 분리**되어 최소 설치에는 영어만 들어 있다.

```bash
locale -a | grep -i ko_KR || sudo dnf install -y glibc-langpack-ko
localectl set-locale LANG=ko_KR.UTF-8
```

- `glibc-langpack-<언어코드>` : 해당 언어의 로케일 데이터 패키지
- `glibc-all-langpacks` : 전체 언어 (용량 큼)
- `localedef` 로 직접 생성도 가능하나 패키지 설치가 표준

---

### 5. 실무에서 문제가 되는 지점

#### ① 정렬 결과가 달라진다 (`LC_COLLATE`)

```bash
printf 'b\nA\na\nB\n' | LC_ALL=C sort          # A B a b  (대문자 먼저 — 바이트 순)
printf 'b\nA\na\nB\n' | LC_ALL=en_US.UTF-8 sort # a A b B  (사전식)
```

- 스크립트가 정렬 결과에 의존한다면 **로케일에 따라 동작이 달라진다**
- `sort`·`uniq`·`comm`·`join` 은 정렬 순서를 전제로 하므로 특히 위험 ([[LAB/04-file-text-shell]] 4절)

#### ② 메시지 언어 때문에 파싱이 깨진다 (`LC_MESSAGES`)

```bash
LC_ALL=C df -h            # 영어 헤더 — grep 패턴이 안정적
```

- 스크립트에서 명령 출력을 `grep` 으로 걸러낼 때, 로케일이 바뀌면 **문자열이 번역되어 매칭 실패**
- 그래서 스크립트 상단에 `export LC_ALL=C` 를 넣는 것이 관례

#### ③ 소수점 기호 (`LC_NUMERIC`)

일부 유럽 로케일은 소수점이 쉼표(`1234,56`)다. `awk` 로 실수를 계산하는 스크립트가 그 환경에서 오작동한다.

#### ④ SSH 접속 시 클라이언트 로케일이 넘어온다

```text
-bash: warning: setlocale: LC_CTYPE: cannot change locale (ko_KR.UTF-8): No such file or directory
```

- SSH 클라이언트의 `LANG`·`LC_*` 가 **서버로 전달**되는데, 서버에 그 로케일이 없으면 이 경고가 뜬다
- `TERM` 이 서버에 없어서 났던 문제와 **같은 구조**다 ([[#A-6. `unknown terminal type` 오류]])
- 해결 방법 세 가지

| 방법 | 내용 |
| --- | --- |
| 서버에 설치 | `sudo dnf install glibc-langpack-ko` |
| 서버가 안 받도록 | `/etc/ssh/sshd_config` 의 `AcceptEnv LANG LC_*` 를 주석 처리 |
| 클라이언트가 안 보내도록 | `~/.ssh/config` 에서 해당 Host 의 `SendEnv` 제거 |

---

### 6. 문자 인코딩과의 관계

로케일의 인코딩 부분(`.UTF-8`)은 **문자를 바이트로 표현하는 방식**을 정한다.

| 인코딩 | 특징 |
| --- | --- |
| **ASCII** | 영문·숫자·기호 128자. 1바이트 |
| **EUC-KR** | 과거 한국 표준. 한글 2바이트 |
| **CP949** | EUC-KR 확장 (윈도 계열) |
| **UTF-8** | 전 세계 문자. ASCII 와 호환, 한글은 3바이트 |

- 한글이 깨져 보이면 **파일의 인코딩과 로케일의 인코딩이 다른 것**
- 변환 도구는 `iconv`

```bash
file -i doc.txt                              # 인코딩 추정
iconv -f EUC-KR -t UTF-8 doc.txt > doc_utf8.txt
```

- `file -i` : MIME 형식과 **charset** 표시
- `iconv` : **i**nternationalization **conv**ersion
  - `-f` : **f**rom — 원본 인코딩
  - `-t` : **t**o — 변환할 인코딩
  - `-c` : 변환 불가 문자를 버리고 진행

---

### 7. 실습 환경 권장 설정

절차서는 **영어 로케일 유지**를 전제로 한다.

- 오류 메시지가 영어여야 검색이 쉽고, 기출 문제의 출력 예시와도 일치
- 시간대만 한국으로 맞추면 로그 시각 해석에 충분

```bash
localectl status
sudo timedatectl set-timezone Asia/Seoul
date
```

- **로케일과 시간대는 별개 설정**이다. `localectl` 은 언어·문자, `timedatectl` 은 시간대를 다룬다 → 혼동 주의

> 📝 **시험 포인트** : 우선순위 **`LC_ALL` > `LC_*` > `LANG`**. 시스템 전역 파일은 **`/etc/locale.conf`**(구형 `/etc/sysconfig/i18n`). `locale -a` 로 설치 목록 확인, `localectl set-locale` 로 영구 설정. 로케일 이름은 **언어_지역.인코딩**. `LC_COLLATE` 가 `sort` 결과를 바꿈. 로케일(`localectl`)과 시간대(`timedatectl`)는 **다른 설정**

관련 항목: [[#F-1. 환경변수·셸·tty 와 `su -` 가 환경을 초기화하는 이유]] · [[#A-6. `unknown terminal type` 오류]] · 절차서 [[LAB/01-vm-setup-and-inspection]] 3-8

---

# J. 부팅·systemd

## J-1. `systemd-analyze critical-chain` 출력 해석

**Q.** 아래 출력은 무엇을 뜻하는가.

```text
multi-user.target @1.378s
└─rsyslog.service @1.368s +9ms
  └─network-online.target @1.364s
    └─NetworkManager-wait-online.service @1.329s +33ms
      └─NetworkManager.service @1.278s +38ms
        └─network-pre.target @1.238s
          └─firewalld.service @780ms +455ms
            └─basic.target @775ms
              ...
                └─system.slice
                  └─-.slice
```

**A.** **부팅에서 가장 오래 걸린 의존성 사슬(임계 경로)** 을 보여준다. "무엇이 무엇을 기다리느라 부팅이 이만큼 걸렸는가" 를 추적한 것이다.

---

### 1. 읽는 방향

```
multi-user.target          ← 최종 목표 (맨 위)
└─ rsyslog.service         ← 이것이 끝나야 위가 시작
   └─ network-online.target
      └─ ...
         └─ -.slice        ← 가장 먼저 (맨 아래)
```

- **트리는 위에서 아래로 "무엇에 의존하는가"** 를 표시
- **시간 순서는 아래에서 위로** — 맨 아래가 가장 먼저 완료되고, 맨 위가 마지막
- 즉 `multi-user.target` 이 `rsyslog` 를 기다렸고, `rsyslog` 는 `network-online` 을 기다렸다는 뜻

### 2. `@` 와 `+` — 가장 헷갈리는 부분

| 기호 | 뜻 |
| --- | --- |
| **`@1.278s`** | 부팅 시작으로부터 **이 유닛이 활성화된 시각** |
| **`+38ms`** | 이 유닛이 **초기화에 걸린 시간** |

```text
NetworkManager.service @1.278s +38ms
```

→ 부팅 후 **1.278초 시점에 시작**해서 **38밀리초 동안** 초기화했다.

- `.target` 에는 `+` 가 없다 → **타겟은 실제로 무언가를 실행하는 유닛이 아니라 "여기까지 왔다" 를 표시하는 이정표**이기 때문
- 시각이 아래에서 위로 항상 단조 증가하지는 않는다. 일부 보조 유닛(credentials 마운트 등)은 시간이 아니라 **의존 관계 기준**으로 배치되기 때문

### 3. 이 출력에서 읽어야 할 결론

| 항목 | 값 |
| --- | --- |
| `multi-user.target` 도달 | **1.378초** — 매우 빠름 |
| 가장 오래 걸린 유닛 | **`firewalld.service` +455ms** |
| 그 유닛이 전체에서 차지하는 비중 | 약 **33%** |

**`firewalld` 하나가 부팅 시간의 3분의 1을 쓰고 있다.** 방화벽 규칙을 nftables 에 적재하는 작업이라 원래 무겁다. 그리고 `network-pre.target` 앞에 있다는 점이 중요하다 — **네트워크가 올라오기 전에 방화벽이 먼저 준비되어야 한다**는 보안 설계다. 잠깐이라도 무방비 상태가 생기지 않도록 순서를 강제한 것.

---

### 4. 사슬에 등장한 유닛 해설

아래에서 위로, 즉 실행 순서대로 본다.

#### 최하단 — cgroup 계층

| 유닛 | 정체 |
| --- | --- |
| `-.slice` | **루트 슬라이스**. `-` 는 `/` 를 뜻하는 systemd 표기. 모든 cgroup 의 최상위 |
| `system.slice` | 시스템 서비스 전체를 묶는 자원 그룹 |

- **slice** : **cgroup**(**c**ontrol **group**)으로 프로세스를 묶어 CPU·메모리를 배분하는 단위
- 실제로 실행되는 것이 아니라 **자원 관리용 그릇**

#### 초기화 단계

| 유닛 | 역할 |
| --- | --- |
| `systemd-journald.socket` | 로그 수집 소켓. **가장 먼저 준비**해야 이후 모든 메시지를 놓치지 않음 |
| `kmod-static-nodes.service` `+54ms` | 아직 적재되지 않은 커널 모듈용 **정적 장치 노드**를 `/dev` 에 미리 생성 |
| `systemd-tmpfiles-setup-dev.service` `+13ms` | `/dev` 아래 장치 파일의 권한·심볼릭 링크 설정 |
| `local-fs-pre.target` | **로컬 파일시스템 마운트 직전** 이정표 |
| `run-credentials-…mount` | systemd 자격 증명 전달용 임시 마운트. 이름의 `\x2d` 는 **`-` 를 16진수로 이스케이프**한 것 |
| `local-fs.target` | `/etc/fstab` 의 로컬 파일시스템 **마운트 완료** ([[#B-2. 마운트란 무엇인가]]) |
| `systemd-tmpfiles-setup.service` `+82ms` | `/tmp`·`/run` 등의 임시 파일·디렉터리 생성·정리 |
| `auditd.service` `+14ms` | 감사 데몬 시작 |
| `systemd-update-utmp.service` `+3ms` | 부팅 사실을 `/var/log/wtmp` 에 기록 → `last reboot` 로 조회되는 근거 |
| `sysinit.target` | **시스템 초기화 완료** 이정표 |

#### 기본 시스템 단계

| 유닛 | 역할 |
| --- | --- |
| `dbus.socket` | D-Bus 통신 소켓 |
| `dbus-broker.service` `+5ms` | 프로세스 간 통신 버스 ([[#F-4. `pstree` 출력 — 최소 설치 Rocky 9 의 프로세스 전수 해설]]) |
| `basic.target` | **기본 시스템 준비 완료**. 이후부터 일반 서비스가 뜰 수 있음 |

#### 네트워크 단계

| 유닛 | 역할 |
| --- | --- |
| `firewalld.service` `+455ms` | 방화벽 규칙 적재. **최대 병목** |
| `network-pre.target` | 네트워크 설정 **직전** 이정표. 방화벽처럼 먼저 준비돼야 할 것들의 기준선 |
| `NetworkManager.service` `+38ms` | 인터페이스·IP·경로 설정 ([[#D-3. `ip addr` 출력 전체 해설]]) |
| `NetworkManager-wait-online.service` `+33ms` | **네트워크가 실제로 연결될 때까지 대기** |
| `network-online.target` | 네트워크 사용 가능 이정표 |
| `rsyslog.service` `+9ms` | 로그 데몬. 원격 전송 가능성 때문에 네트워크 이후에 시작 |
| `multi-user.target` | **부팅 완료** (CLI 다중 사용자 모드) |

> `NetworkManager-wait-online.service` 는 실제 서버에서 **부팅 지연의 단골 원인**이다. DHCP 응답이 늦으면 기본 30초까지 기다린다. 네트워크를 기다릴 필요가 없는 서버라면 `systemctl disable NetworkManager-wait-online.service` 로 비활성화하기도 한다. 실습 VM 은 33ms 로 문제없음

---

### 5. 함께 쓰는 명령

```bash
systemd-analyze
systemd-analyze blame
systemd-analyze critical-chain
systemd-analyze critical-chain sshd.service
systemd-analyze plot > /tmp/boot.svg
```

| 명령 | 보여주는 것 |
| --- | --- |
| `systemd-analyze` | 커널·initrd·userspace 단계별 총 소요 시간 |
| `blame` | **모든 유닛을 소요 시간 순으로** 정렬 |
| `critical-chain` | **병목이 되는 의존성 사슬만** |
| `critical-chain <유닛>` | 특정 유닛까지의 사슬 |
| `plot` | 전체 타임라인을 **SVG 그림**으로 |

#### `blame` 과 `critical-chain` 의 차이 — 중요

- `blame` 은 **오래 걸린 순서**로 나열한다. 그러나 **오래 걸렸다고 부팅을 늦춘 것은 아니다** — 다른 서비스와 **병렬로** 실행됐다면 전체 시간에 영향이 없다
- `critical-chain` 은 **실제로 다음 단계를 막고 있던 경로**만 보여준다
- **부팅을 실제로 단축하려면 `critical-chain` 을 봐야 한다**

```bash
systemd-analyze blame | head -5
```

---

### 6. 타겟(`.target`)이란

- 여러 유닛을 묶은 **동기화 지점**. SysV 의 **런레벨**에 대응
- 실행 파일이 없으므로 `+` 소요 시간이 표시되지 않음

| 타겟 | 런레벨 | 뜻 |
| --- | --- | --- |
| `poweroff.target` | 0 | 종료 |
| `rescue.target` | 1 | 단일 사용자 |
| `multi-user.target` | 3 | **CLI 다중 사용자** (서버 기본) |
| `graphical.target` | 5 | GUI |
| `reboot.target` | 6 | 재부팅 |

```bash
systemctl get-default
systemctl list-dependencies multi-user.target
```

- `get-default` : 부팅 시 도달할 기본 타겟 확인 → 지금은 `multi-user.target`
- `list-dependencies` : 그 타겟이 무엇을 필요로 하는지 트리로 표시

> 📝 **시험 포인트** : `@` 는 **시작 시각**, `+` 는 **소요 시간**. `.target` 은 실행 단위가 아닌 **동기화 지점**이라 소요 시간이 없음. `blame`(전체 나열) 과 `critical-chain`(병목 경로) 의 차이. `multi-user.target` = 런레벨 3. 방화벽이 `network-pre.target` 앞에 오는 것은 **보안 설계**

관련 항목: [[#F-4. `pstree` 출력 — 최소 설치 Rocky 9 의 프로세스 전수 해설]] · [[#G-1. `~d` 데몬과 `~ctl` 명령의 관계]] · 절차서 [[LAB/01-vm-setup-and-inspection]] 4절, [[LAB/07-boot-systemd-log]] 1·3절

---

# K. 소켓·파일 디스크립터

## K-1. 소켓이란 무엇인가 — 파일 디스크립터와 함께

**Q.** 리눅스에서 소켓이란 무엇인가. 파일 디스크립터 개념과 함께 설명해 달라.

**A.** 출발점은 유닉스의 설계 철학이다. **"모든 것은 파일이다"**(everything is a file). 소켓도 그 철학 위에서 **"통신을 파일처럼 다루게 만든 것"** 이고, 파일 디스크립터는 **그 파일을 가리키는 번호표**다.

---

### 1. 먼저 — "모든 것은 파일이다"

리눅스는 성격이 전혀 다른 대상들을 **같은 방식(열기·읽기·쓰기·닫기)으로** 다룬다.

```bash
ls -l /dev/vda /dev/null /etc/passwd /run/dbus/system_bus_socket
ls -l /proc/$$/fd
```

`ls -l` 출력 **첫 글자**가 그 대상의 종류다.

| 문자 | 종류 | 예 |
| --- | --- | --- |
| `-` | 일반 파일 | `/etc/passwd` |
| `d` | 디렉터리 | `/etc` |
| `l` | 심볼릭 링크 | `/bin` → `usr/bin` |
| `b` | **블록** 장치 | `/dev/vda` — 블록 단위 입출력 |
| `c` | **문자** 장치 | `/dev/null`, `/dev/tty` — 바이트 단위 |
| `p` | 파이프(FIFO) | `mkfifo` 로 생성 |
| **`s`** | **소켓** | `/run/dbus/system_bus_socket` |

프로그램 입장에서는 이 모두가 **똑같이 `read()`·`write()` 로 다룰 수 있는 대상**이다. 덕분에 `cat`·`grep` 같은 도구가 파일이든 장치든 파이프든 가리지 않고 동작한다.

---

### 2. 파일 디스크립터(FD)

- **file descriptor** : 프로세스가 열어 둔 대상을 가리키는 **작은 정수 번호**
- 커널이 실제 객체(파일·소켓·파이프)를 관리하고, 프로세스는 **번호로만 접근**한다
- 프로세스마다 **자기만의 FD 테이블**을 갖는다 → 같은 번호라도 프로세스가 다르면 다른 대상

```
프로세스 A                     커널
┌──────────────┐         ┌─────────────────┐
│ FD 0 ────────┼────────▶│ /dev/pts/0      │
│ FD 1 ────────┼────────▶│ /dev/pts/0      │
│ FD 2 ────────┼────────▶│ /dev/pts/0      │
│ FD 3 ────────┼────────▶│ TCP 소켓        │
└──────────────┘         └─────────────────┘
```

#### 예약된 세 개

| FD | 이름 | 원어 | 기본 연결 |
| --- | --- | --- | --- |
| **0** | 표준 입력 | **std**ard **in**put | 키보드(터미널) |
| **1** | 표준 출력 | **std**ard **out**put | 화면(터미널) |
| **2** | 표준 오류 | **std**ard **err**or | 화면(터미널) |

**리다이렉션의 정체가 바로 이 번호 바꿔치기다.**

```bash
command > out.txt          # FD 1 을 파일로 연결
command 2> err.txt         # FD 2 를 파일로 연결
command > out.txt 2>&1     # FD 1 을 파일로, 그다음 FD 2 를 FD 1 과 같은 곳으로
command 2>&1 > out.txt     # 순서가 다르면 결과도 다름 (⚠️ 함정)
```

- `2>&1` : FD 2 가 **FD 1 이 현재 가리키는 곳**을 함께 가리키게 복제 (`&` 는 "번호" 를 뜻함)
- 순서가 중요한 이유 — `2>&1 > out.txt` 는 FD 2 를 **아직 터미널인** FD 1 에 붙인 뒤 FD 1 만 파일로 옮기므로, 오류는 화면에 남는다 ([[LAB/04-file-text-shell]] 6절)

#### 직접 확인하기

```bash
ls -l /proc/$$/fd
exec 3< /etc/passwd        # FD 3 에 파일 열기
ls -l /proc/$$/fd
head -1 <&3                # FD 3 에서 읽기
exec 3<&-                  # FD 3 닫기
```

- `/proc/<PID>/fd/` : 그 프로세스가 연 모든 FD 를 **심볼릭 링크로** 보여줌
- `exec 3< 파일` : 셸 자신에게 FD 3 을 열어 둠. `exec` 는 명령 없이 쓰면 **현재 셸의 FD 만 조작** ([[#F-3. `su`·`sudo -i` 를 반복하면 셸이 스택처럼 쌓이는가]] 의 프로세스 대체와는 다른 용법)
- `<&3` : FD 3 에서 입력받기, `3<&-` : FD 3 닫기

#### 개수 제한

```bash
ulimit -n
ulimit -Hn
lsof -p $$ | wc -l
```

- `ulimit -n` : 한 프로세스가 열 수 있는 FD 최대 개수 (**n**umber of open files). 기본 1024 인 경우가 많음
- `-H` : **H**ard limit — 일반 사용자가 넘을 수 없는 상한. `-S` 는 soft
- 웹 서버·DB 처럼 연결이 많은 서비스는 이 값이 부족해 **`Too many open files`** 오류가 난다 → `/etc/security/limits.conf` 또는 유닛 파일의 `LimitNOFILE` 로 상향 ([[LAB/03-user-group-permission]] 5-5)

---

### 3. 소켓이란

- **socket** : **통신의 끝점**(endpoint). 두 프로세스가 데이터를 주고받기 위해 만드는 창구
- 원래 뜻은 전구를 끼우는 **소켓**, 즉 "꽂는 자리"
- 만들면 **FD 로 돌려받는다** → 이후 파일처럼 `read()`·`write()` 로 사용

**파이프와의 차이** : 파이프는 부모-자식처럼 **혈연 관계인 프로세스끼리, 한 방향**만 가능하다. 소켓은 **관계없는 프로세스끼리, 양방향**으로, 심지어 **다른 컴퓨터와도** 통신한다.

---

### 4. 소켓의 종류

#### ① 주소 계열 — 어디까지 통신하는가

| 계열 | 상수 | 범위 | 확인 |
| --- | --- | --- | --- |
| **유닉스 도메인 소켓** | `AF_UNIX` | **같은 컴퓨터 안**. 파일 경로로 식별 | `ss -x` |
| **IPv4 소켓** | `AF_INET` | 네트워크. IP + 포트로 식별 | `ss -t4` |
| **IPv6 소켓** | `AF_INET6` | 네트워크 | `ss -t6` |
| **패킷 소켓** | `AF_PACKET` | 링크 계층 직접 접근 | `tcpdump` 가 사용 |

- **AF** = **A**ddress **F**amily

#### ② 타입 — 어떻게 전달하는가

| 타입 | 대응 프로토콜 | 성격 |
| --- | --- | --- |
| `SOCK_STREAM` | **TCP** | 연결 지향. 순서·도달 보장 |
| `SOCK_DGRAM` | **UDP** | 비연결. 빠르지만 보장 없음 ([[#D-1. DHCP 란 무엇인가]] 가 이것을 쓰는 이유) |
| `SOCK_RAW` | IP 직접 | 헤더를 직접 다룸. `ping` 의 ICMP 가 사용 |

---

### 5. 유닉스 도메인 소켓 — 파일시스템에 보이는 소켓

```bash
ls -l /run/dbus/system_bus_socket /run/systemd/journal/socket
ss -xl | head
```

파일처럼 경로가 있지만 **디스크에 내용이 저장되지는 않는다.** 이름표 역할만 하고, 실제 데이터는 커널 메모리를 통해 오간다.

#### 네트워크 대신 이걸 쓰는 이유

| 이유 | 설명 |
| --- | --- |
| **빠르다** | TCP/IP 스택을 거치지 않음. 체크섬·라우팅 불필요 |
| **파일 권한으로 접근 제어** | `chmod`·`chown` 이 그대로 적용 |
| **외부 노출 없음** | 네트워크에 뜨지 않으므로 원격 공격 불가 |

**`/var/run/docker.sock` 이 대표적이다.** 이 소켓 파일의 그룹이 `docker` 라서, `docker` 그룹에 속하면 데몬에 명령할 수 있다. 그리고 도커 데몬은 root 권한으로 돌기 때문에 — **`docker` 그룹은 사실상 root 권한과 같다.** [[LAB/11-container-virtualization]] 2절에서 경고하는 근거가 이것이다.

```bash
ls -l /var/run/docker.sock
```

```text
srw-rw----. 1 root docker 0 ... /var/run/docker.sock
```

- 첫 글자 **`s`** → 소켓
- `root docker` → 소유자 root, 그룹 docker

---

### 6. 네트워크 소켓의 일생

```
서버                                  클라이언트
socket()   소켓 생성 → FD 획득
bind()     IP·포트에 결속
listen()   연결 대기 상태 (LISTEN)
                                      socket()
accept()   ◀──────────────────────────connect()
  └ 연결마다 새 FD 생성                (ESTABLISHED)
read()/write()  ◀────────────────────▶ read()/write()
close()                                close()
```

- **`accept()` 가 연결마다 새 FD 를 만든다** → 접속자가 많으면 FD 를 많이 쓰는 이유
- [[#F-4. `pstree` 출력 — 최소 설치 Rocky 9 의 프로세스 전수 해설]] 에서 본 `sshd(774)` 가 `listen` 중인 부모, `sshd-session(2303)` 이 `accept` 후 생긴 연결별 프로세스

---

### 7. 확인 명령

```bash
ss -tulnp                  # 듣고 있는 TCP/UDP 소켓과 프로세스
ss -tan state established  # 연결된 TCP
ss -xl                     # 유닉스 도메인 소켓 (듣는 것만)
ss -s                      # 소켓 종류별 요약 통계
lsof -i :22                # 22번 포트를 쓰는 프로세스
lsof -U                    # 유닉스 소켓
lsof -p <PID>              # 그 프로세스가 연 모든 FD
ls -l /proc/<PID>/fd
```

- `ss` = **s**ocket **s**tatistics → `netstat` 의 현대적 대체. `/proc/net/` 대신 커널 인터페이스를 직접 써서 빠름
  - `-t` **t**cp · `-u` **u**dp · `-x` 유닉스 · `-l` **l**istening · `-n` **n**umeric(이름 해석 생략) · `-p` **p**rocess · `-a` **a**ll
- `lsof` = **l**i**s**t **o**pen **f**iles → **"모든 것은 파일"** 철학 그대로, 열린 파일·소켓·장치를 전부 나열
  - `-i` : **i**nternet 소켓만. `-i :22` 처럼 포트 지정 가능
  - `-U` : **U**nix 도메인 소켓만
  - `-p` : 특정 **p**ID
  - `+L1` : 링크 수가 1 미만인 파일 → **삭제됐는데 프로세스가 붙잡고 있는 파일** 탐지. `df` 와 `du` 가 안 맞을 때 사용 ([[LAB/06-process-scheduling-diagnosis]] 5절)

---

### 8. systemd 소켓 활성화

[[#J-1. `systemd-analyze critical-chain` 출력 해석]] 의 부팅 사슬에 `dbus.socket`·`systemd-journald.socket` 이 있었다. 이것이 **소켓 활성화**(socket activation)다.

- **`.socket` 유닛이 먼저 소켓만 열어 두고**, 실제 요청이 들어오면 그때 서비스를 시작
- 장점 : 부팅이 빨라지고(서비스는 필요할 때 시작), 서비스가 재시작되는 동안에도 **연결 요청이 소켓 큐에 쌓여 유실되지 않음**
- 구형 `xinetd`(슈퍼 데몬)의 역할을 systemd 가 흡수한 것 ([[LAB/09-network-services]] 9절)

```bash
systemctl list-sockets
```

---

### 9. 전체를 잇는 그림

```
"모든 것은 파일이다"
   │
   ├─ 일반 파일 ─┐
   ├─ 장치      ─┤
   ├─ 파이프    ─┼──▶ 파일 디스크립터(정수 번호)로 접근
   └─ 소켓      ─┘        │
                          ├─ 0/1/2 : 표준 입출력 → 리다이렉션의 원리
                          ├─ 3~    : open()·socket() 이 반환
                          └─ ulimit -n 으로 개수 제한
```

> 📝 **시험 포인트** : FD **0=stdin, 1=stdout, 2=stderr**. `2>&1` 의 의미와 **순서에 따른 차이**. `ls -l` 첫 글자 **`s`=소켓, `p`=파이프, `b`=블록, `c`=문자**. `ss` 는 `netstat` 대체이며 옵션 `-tulnp` 조합이 최빈출. `lsof` 는 열린 파일·소켓 조회. 유닉스 도메인 소켓은 **파일 권한으로 접근 제어**. TCP=`SOCK_STREAM`, UDP=`SOCK_DGRAM`

관련 항목: [[#D-3. `ip addr` 출력 전체 해설]] · [[#F-4. `pstree` 출력 — 최소 설치 Rocky 9 의 프로세스 전수 해설]] · [[#J-1. `systemd-analyze critical-chain` 출력 해석]] · 절차서 [[LAB/04-file-text-shell]] 6·10절, [[LAB/06-process-scheduling-diagnosis]] 5절, [[LAB/08-network-config]] 5절

---

## J-2. 타겟(target)과 런레벨(runlevel)이란 정확히 무엇인가

**Q.** 타겟이란 정확히 무엇인가. 런레벨은 무엇인가.

**A.** 둘 다 **"시스템을 어떤 상태로 만들 것인가"** 를 나타낸다. **런레벨은 옛 방식(SysV init), 타겟은 현재 방식(systemd)** 이며, 타겟이 런레벨을 대체하면서 **훨씬 넓은 개념**이 되었다.

---

### 1. 런레벨 — 옛 방식

- **runlevel** : SysV init 시절, 시스템의 동작 모드를 **0~6 숫자**로 정의한 것
- 각 번호마다 "어떤 서비스를 켜고 끌지" 가 미리 정해져 있음

#### 동작 방식

```
/etc/inittab 에 기본 런레벨 지정      id:3:initdefault:
        │
        ▼
/etc/rc.d/rc3.d/ 디렉터리를 순서대로 실행
        ├─ K01xxx  → 중지(Kill)할 서비스
        └─ S85httpd → 시작(Start)할 서비스
```

| 특징 | 내용 |
| --- | --- |
| 식별자 | **숫자** 0~6 |
| 배타성 | 한 번에 **하나만** 활성 |
| 실행 방식 | 심볼릭 링크 이름의 **번호 순서대로 하나씩** → 느림 |
| 설정 파일 | `/etc/inittab` |
| 전환 명령 | `init 3` · `telinit 3` |

- 심볼릭 링크 이름의 `S85` 에서 `85` 는 **실행 순서**. 앞의 것이 끝나야 다음이 시작 → 병렬 처리 불가
- RHEL 7 부터 systemd 로 대체. **`/etc/inittab` 은 남아 있지만 안내 문구만 들어 있다**

```bash
cat /etc/inittab
```

```text
# inittab is no longer used.
#
# ADDING CONFIGURATION HERE WILL HAVE NO EFFECT ON YOUR SYSTEM.
...
```

> 📝 시험에서 `/etc/inittab` 의 형식(`id:3:initdefault:`)을 묻는 문제가 여전히 출제된다. **동작하지 않지만 개념은 알아야 한다**

---

### 2. 타겟 — 현재 방식

- **target** : systemd의 **유닛(unit) 종류 중 하나**. 확장자가 `.target`
- **여러 유닛을 묶는 그룹이자, "여기까지 도달했다" 를 나타내는 동기화 지점**
- **실행 파일이 없다.** 타겟 자체는 아무것도 실행하지 않는다 → [[#J-1. `systemd-analyze critical-chain` 출력 해석]] 에서 `.target` 에 `+`(소요 시간)가 없던 이유

#### systemd 유닛의 종류

| 확장자 | 대상 |
| --- | --- |
| `.service` | 데몬·프로그램 |
| **`.target`** | **유닛 묶음·동기화 지점** |
| `.socket` | 소켓 ([[#K-1. 소켓이란 무엇인가 — 파일 디스크립터와 함께]]) |
| `.mount` · `.automount` | 마운트 지점 |
| `.timer` | 예약 실행 (cron 대체) |
| `.path` | 파일·디렉터리 변화 감시 |
| `.slice` | cgroup 자원 그룹 |
| `.device` | 장치 |
| `.swap` | 스왑 영역 |

#### 타겟 파일 들여다보기

```bash
systemctl cat multi-user.target
```

```text
[Unit]
Description=Multi-User System
Documentation=man:systemd.special(7)
Requires=basic.target
Conflicts=rescue.service rescue.target
After=basic.target rescue.service rescue.target
AllowIsolate=yes
```

- `Requires=basic.target` : **basic.target 이 반드시 성공해야** 이 타겟이 성립
- `After=` : 순서 지정 (의존성과 별개로 "먼저 끝나야 함")
- `Conflicts=` : 함께 활성화될 수 없는 유닛 → rescue 와 multi-user 는 동시에 못 감
- `AllowIsolate=yes` : `systemctl isolate` 로 **전환 대상이 될 수 있음**
- **실행할 프로그램(`ExecStart=`)이 없다** → 타겟의 본질이 "상태 표시" 임을 보여줌

---

### 3. 그럼 타겟은 어떻게 서비스를 끌어오는가

**`.wants` 디렉터리의 심볼릭 링크**로 한다.

```bash
ls -l /etc/systemd/system/multi-user.target.wants/
```

```text
sshd.service -> /usr/lib/systemd/system/sshd.service
crond.service -> /usr/lib/systemd/system/crond.service
firewalld.service -> /usr/lib/systemd/system/firewalld.service
...
```

**`systemctl enable` 이 하는 일이 정확히 이 링크를 만드는 것이다.**

```bash
sudo systemctl enable httpd
# → /etc/systemd/system/multi-user.target.wants/httpd.service 링크 생성
```

- 어느 타겟에 걸릴지는 서비스 유닛 파일의 `[Install]` 섹션 `WantedBy=` 가 정한다
- `disable` 은 그 링크를 지운다 → **서비스 파일 자체는 그대로**

#### SysV 와 비교하면 구조가 같다

| SysV | systemd |
| --- | --- |
| `/etc/rc.d/rc3.d/S85httpd` → 스크립트 | `multi-user.target.wants/httpd.service` → 유닛 |
| `chkconfig httpd on` | `systemctl enable httpd` |
| 숫자로 **순서 강제** | 의존성으로 **필요한 것만 먼저**, 나머지는 병렬 |

---

### 4. 타겟이 런레벨보다 나은 점

| 구분 | 런레벨 | 타겟 |
| --- | --- | --- |
| 식별 | 숫자 (의미 불명확) | **이름** (`graphical`·`network-online`) |
| 동시 활성 | 하나만 | **여러 개 동시에** |
| 실행 | 순차 | **의존성 기반 병렬** |
| 용도 | 부팅 모드만 | 부팅 모드 + **중간 동기화 지점** |
| 확장 | 0~6 고정 | 새 타겟을 **자유롭게 정의** |

**가장 큰 차이는 마지막 두 줄이다.** 런레벨은 "부팅 모드" 하나뿐이었지만, 타겟은 **부팅 과정의 이정표**로도 쓰인다.

[[#J-1. `systemd-analyze critical-chain` 출력 해석]] 의 사슬에 나왔던 것들이 그런 타겟이다.

| 타겟 | 뜻 | 런레벨 대응 |
| --- | --- | --- |
| `sysinit.target` | 시스템 초기화 완료 | 없음 |
| `basic.target` | 기본 시스템 준비 | 없음 |
| `network-pre.target` | 네트워크 설정 **직전** | 없음 |
| `network-online.target` | 네트워크 사용 가능 | 없음 |
| `local-fs.target` | 로컬 파일시스템 마운트 완료 | 없음 |
| `multi-user.target` | 다중 사용자 CLI | **3** |
| `graphical.target` | GUI | **5** |

- 이런 중간 타겟 덕분에 "방화벽은 네트워크보다 먼저" 같은 순서를 **선언적으로** 표현할 수 있다
- 런레벨 방식에서는 `S` 번호를 손으로 조정해야 했다

---

### 5. 런레벨 호환성

systemd 는 옛 명령과 개념을 **호환용으로** 유지한다.

```bash
ls -l /usr/lib/systemd/system/runlevel*.target
runlevel
who -r
init 3
```

```text
runlevel0.target -> poweroff.target
runlevel1.target -> rescue.target
runlevel2.target -> multi-user.target
runlevel3.target -> multi-user.target
runlevel4.target -> multi-user.target
runlevel5.target -> graphical.target
runlevel6.target -> reboot.target
```

- **2·3·4 가 모두 `multi-user.target` 을 가리킨다** → systemd 에서는 셋이 사실상 같음
- `runlevel` 명령은 `이전 현재` 를 출력. 이전 값이 없으면 `N`(**N**one)
- `who -r` : utmp 의 런레벨 레코드 ([[#F-4. `pstree` 출력 — 최소 설치 Rocky 9 의 프로세스 전수 해설]] 의 `systemd-update-utmp` 가 기록한 것)
- `init 3` · `telinit 3` 도 동작하지만 내부적으로 `systemctl isolate` 로 변환됨

---

### 6. 조회·전환 명령

```bash
systemctl get-default                        # 기본 타겟
systemctl set-default multi-user.target      # 기본 타겟 변경
systemctl list-units --type=target           # 활성 타겟
systemctl list-dependencies multi-user.target
systemctl isolate rescue.target              # ⚠️ 전환
systemctl rescue                             # 위와 동일한 축약
```

#### `start` 와 `isolate` 의 차이 — 중요

| 명령 | 동작 |
| --- | --- |
| `systemctl start X.target` | X 를 **추가로** 활성화. 기존 것은 그대로 |
| `systemctl isolate X.target` | X 와 그 의존성만 남기고 **나머지는 모두 중지** |

**`isolate` 가 런레벨 전환(`init N`)에 해당한다.** 이름 그대로 "고립시킨다" 는 뜻이다.

⚠️ `systemctl isolate rescue.target` 을 SSH 세션에서 실행하면 **네트워크 서비스가 중지되어 접속이 끊긴다.** 반드시 콘솔에서 수행 ([[LAB/07-boot-systemd-log]] 3절).

#### `default.target` 의 실체

```bash
ls -l /etc/systemd/system/default.target
```

```text
default.target -> /usr/lib/systemd/system/multi-user.target
```

- `default.target` 은 **심볼릭 링크**다. `set-default` 는 이 링크를 바꾸는 것
- GUI 설치 시스템에서 텍스트 모드로 부팅하고 싶다면 이 링크를 `multi-user.target` 으로 바꾼다

---

### 7. 복구용 타겟 3종 비교

| 타겟 | 마운트 | 네트워크 | 서비스 | 용도 |
| --- | --- | --- | --- | --- |
| `multi-user.target` | 전부 | ✓ | 전부 | 정상 운영 |
| `rescue.target` | 로컬 파일시스템 | ✗ | 최소 | 시스템 복구 |
| `emergency.target` | **루트만, 읽기 전용** | ✗ | 없음 | 최후 수단 |

- `rescue` 는 로컬 디스크가 마운트되므로 파일 수정이 가능
- `emergency` 는 `/etc/fstab` 오류처럼 **마운트 자체가 실패했을 때** 진입 → `mount -o remount,rw /` 부터 해야 함 ([[LAB/07-boot-systemd-log]] 9절의 fstab 복구 실습)

부팅 시 진입하려면 GRUB 편집으로 커널 파라미터를 준다.

```text
systemd.unit=rescue.target
systemd.unit=emergency.target
```

> 📝 **시험 포인트** : 런레벨↔타겟 표(**1=rescue, 3=multi-user, 5=graphical, 6=reboot**)가 매 회차 출제. 현재 런레벨 확인은 `runlevel`·`who -r`, 기본 타겟 확인은 `systemctl get-default`. `start` 와 `isolate` 의 차이. **타겟은 실행 파일이 없는 동기화 지점**. `systemctl enable` 은 `.wants` 디렉터리에 **심볼릭 링크를 만드는 것**. `/etc/inittab` 은 RHEL 7 이후 미사용

관련 항목: [[#J-1. `systemd-analyze critical-chain` 출력 해석]] · [[#G-1. `~d` 데몬과 `~ctl` 명령의 관계]] · 절차서 [[LAB/01-vm-setup-and-inspection]] 4-3, [[LAB/07-boot-systemd-log]] 3절

---

## 관련 문서

- [[README]] — LINUX-MASTER 허브
- [[LAB/README]] — 실습 절차서 허브
- [[THEORY/user-permission]] · [[THEORY/disk-device]] · [[THEORY/system-security]]
