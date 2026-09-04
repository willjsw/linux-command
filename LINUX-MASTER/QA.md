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

## 관련 문서

- [[README]] — LINUX-MASTER 허브
- [[LAB/README]] — 실습 절차서 허브
- [[THEORY/user-permission]] · [[THEORY/disk-device]] · [[THEORY/system-security]]
