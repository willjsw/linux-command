---
title: LAB 01 — VM 준비와 시스템 점검
type: exam-lab
part: 01
tags:
  - exam/linux-master
  - exam/lab
  - linux/basics
  - linux/filesystem
  - linux/shell
  - task/configure
  - task/verify
related: ["[[README]]", "[[02-package-management]]", "[[../THEORY/linux-basics]]", "[[../THEORY/system-structure]]"]
distro: Rocky Linux 9 (aarch64, UTM)
updated: 2026-09-03
---

# LAB 01 — VM 준비와 시스템 점검

- Rocky Linux 9 aarch64 ISO 직접 내려받기 → 해시 검증 → UTM VM 생성(디스플레이·직렬 포트 포함) → 설치 → ISO 제거
- 게스트 IP 를 호스트·게스트 양쪽에서 확인 → SSH 별칭 등록 → 직렬 콘솔 활성화까지 작업 환경 확보
- 호스트명·시간대 설정 후 커널·CPU·메모리·디스크·부팅 과정·세션 정보를 조회 명령으로 전수 점검
- FHS 최상위 디렉터리, 가상 파일시스템, 장치 파일 유형, inode·링크, `/etc/fstab`·`/etc/passwd` 형식 선행 학습
- 셸 환경변수·alias·history·초기화 파일 로드 순서·man 섹션까지 실습 후 스냅샷 저장

> **이 파트의 시나리오**: 개발팀 인트라넷 서버 `srv01.lab.local` 을 새로 구축한다. 아직 아무 자원도 없는 상태이므로, 먼저 VM 을 만들고 설치 매체가 손상되지 않았는지 확인한 뒤, 관리자 `admin1` 로 접속해 하드웨어·OS·부팅 상태를 점검하고 작업 환경(호스트명·시간대·셸)을 갖춘다. 이후 모든 파트는 이 VM 위에서 진행된다.

---

## 1. VM 준비

### 1-0. 설치 매체 내려받기 (macOS, 사용자 직접 수행)

> **상황**: 실습에 필요한 파일 중 **ISO 만은 직접 내려받아야 한다**. Apple Silicon 맥이므로 x86_64 가 아닌 **aarch64** 판을, 그중 GUI 없이 가장 가벼운 **minimal** 판을 받는다. CHECKSUM 파일도 같이 받아야 1-1 의 검증이 가능하다.

- 공식 배포 페이지: `https://rockylinux.org/download`
  - 아키텍처 **aarch64** 행 → **Minimal** 선택 (DVD·Boot 판이 아님)
  - 같은 디렉터리의 `CHECKSUM` 파일도 함께 저장
- 브라우저 대신 터미널로 받는 경우 — 미러 디렉터리를 먼저 열어 **현재 올라와 있는 정확한 파일명**을 확인한 뒤 받는다 (9.x 의 x 는 시점마다 다름)

```bash
# macOS 터미널
cd ~/Downloads

# ① 미러 디렉터리 목록에서 실제 파일명 확인
curl -s https://download.rockylinux.org/pub/rocky/9/isos/aarch64/ | grep -o 'Rocky-9[^"]*minimal.iso' | sort -u

# ② 확인한 파일명으로 ISO 와 CHECKSUM 내려받기 (<파일명> 을 ①의 출력로 치환)
curl -L -O -C - https://download.rockylinux.org/pub/rocky/9/isos/aarch64/<파일명>
curl -L -O    https://download.rockylinux.org/pub/rocky/9/isos/aarch64/CHECKSUM
```

- `curl -L` : 리다이렉트 추종 (**L**ocation) — 미러로 넘어가므로 필수
- `-O` : URL 의 파일명 그대로 저장 (대문자 **O**. 소문자 `-o` 는 파일명 직접 지정)
- `-C -` : 중단된 지점부터 이어받기 (**C**ontinue, `-` = 자동 판단) — 대용량이라 권장
- `-s` : 진행률 표시 없이 조용히 (**s**ilent) — 파이프로 넘길 때 사용
- `wget` 을 쓴다면 `wget -c <URL>` 이 `curl -L -O -C -` 에 대응 (`-c` = **c**ontinue)

**검증**

```bash
ls -lh ~/Downloads/Rocky-9*-aarch64-minimal.iso ~/Downloads/CHECKSUM
file ~/Downloads/Rocky-9*-aarch64-minimal.iso
```

```text
-rw-r--r--  1 ...  ~2G ... Rocky-9.x-aarch64-minimal.iso
-rw-r--r--  1 ...  ... CHECKSUM
Rocky-9.x-aarch64-minimal.iso: ISO 9660 CD-ROM filesystem data ...
```

> 📝 **시험 포인트**: 배포판 이미지 종류 구분 — **Minimal**(최소 패키지, 서버용) · **DVD**(전체 패키지 포함, 오프라인 설치) · **Boot**(부팅만, 나머지는 네트워크에서 받음). 실기에서 "네트워크 없이 설치해야 할 때 받는 이미지" 는 DVD.

### 1-1. 설치 매체 해시 검증 (macOS)

> **상황**: 1-0 에서 받은 설치 매체가 다운로드 중 손상되었거나 변조되었다면 이후 모든 실습이 무의미하다. 같이 받아 둔 공식 CHECKSUM 파일과 대조해 SHA-256 해시를 검증한다.

```bash
# macOS 터미널 — 다운로드 폴더로 이동 (ISO 와 CHECKSUM 을 같은 폴더에 둠)
cd ~/Downloads
ls -l Rocky-9*-aarch64-minimal.iso CHECKSUM

# ① 해시 직접 계산 후 CHECKSUM 파일의 값과 눈으로 대조
shasum -a 256 Rocky-9*-aarch64-minimal.iso
grep minimal CHECKSUM

# ② CHECKSUM 파일 기준 자동 대조 (목록에 있는 파일만 검사)
shasum -a 256 -c CHECKSUM --ignore-missing
```

- `shasum -a 256` : macOS 기본 해시 도구, **a**lgorithm 256 = SHA-256 (리눅스의 `sha256sum` 에 대응)
- `-c` : **c**heck — 파일에 적힌 해시와 실제 파일 해시를 대조
- `--ignore-missing` : CHECKSUM 에 있지만 로컬에 없는 파일(DVD ISO 등)은 건너뜀
- 리눅스에서는 동일 작업이 `sha256sum -c CHECKSUM --ignore-missing` — Part 04 에서 VM 안에서 재실습

**검증**

```bash
shasum -a 256 -c CHECKSUM --ignore-missing; echo "exit=$?"
```

```text
Rocky-9.x-aarch64-minimal.iso: OK
exit=0
```

> 📝 **시험 포인트**: 실기에 "파일의 SHA-256 해시 계산 명령" 이 `sha256sum <파일>` 로 출제. MD5·SHA-1 은 충돌이 발견된 알고리즘이므로 무결성 검증에는 SHA-256 이상 사용 (필기 함정).

### 1-2. UTM 에서 VM 생성 (GUI)

> **상황**: README 1절 명세대로 2 vCPU / 4 GB / 시스템 디스크 40 GB + 추가 디스크 4개(5G·5G·5G·2G) 를 가진 VM 을 만든다. 추가 디스크는 Part 05 의 파티션·LVM·RAID·스왑 실습용이므로 지금 미리 붙여 둔다.

#### ① 생성 마법사

- UTM 실행 → **+ (Create a New Virtual Machine)** → **Virtualize** 선택 (Apple Silicon 에서 aarch64 게스트를 네이티브 가상화)
- 운영체제 → **Linux**
- Boot ISO Image → **Browse…** → 1-1 에서 검증한 `Rocky-9.x-aarch64-minimal.iso` 지정
  - "Use Apple Virtualization" 체크 시 스냅샷 기능 불가 → **체크 해제(QEMU 백엔드)** 권장
- Hardware → Memory **4096 MB**, CPU Cores **2**
- Storage → **40 GB** (시스템 디스크 → 게스트에서 `/dev/vda`)
- Shared Directory → 건너뜀 (Continue)
- Summary → Name **srv01** → Save

#### ② 추가 디스크 4개 (VM 우클릭 → **편집**)

- 좌측 **드라이브** 그룹 맨 아래 **`새로 만들기…`** 를 4회 반복
- 인터페이스 **VirtIO**, 크기 각각 **5 GB / 5 GB / 5 GB / 2 GB** → 게스트에서 `/dev/vdb` `/dev/vdc` `/dev/vdd` `/dev/vde`
- **추가 순서가 곧 장치 이름 순서** → 반드시 5·5·5·2 순으로 추가. 순서가 어긋나면 Part 05 의 `mkfs`·`mdadm` 이 다른 디스크를 대상으로 삼음
- 완료 후 드라이브 목록은 **`USB 드라이브`(CD/DVD, ISO용) 1개 + `VirtIO 드라이브` 5개** = 총 6개. 이 구성이 정상이며 삭제할 항목 없음
- 드라이브를 선택했을 때 표시되는 **크기 `196KB`** 는 용량이 아니라 **qcow2 파일의 현재 실제 사용량**. qcow2 는 쓴 만큼만 커지므로 갓 만든 5 GB 디스크도 이렇게 보임 → 실제 용량은 게스트에서 `lsblk` 로 확인

#### ③ 디스플레이 카드 (⚠️ 기본값이면 화면이 검게 나옴)

- 좌측 **`디스플레이`** → **에뮬레이트된 디스플레이 카드** → **`virtio-ramfb`** 선택
- UTM 기본값 `virtio-gpu-pci` 는 게스트의 virtio-gpu 드라이버가 올라온 뒤에야 출력 → **UEFI 화면·GRUB 메뉴·초기 부팅 로그가 통째로 검게 나오고** `Display output is not active.` 만 표시됨
- `ramfb`(단독) 는 UTM 의 UEFI 가 초기화하지 않아 `Guest has not initialized the display (yet).` 에서 멈춤 → 선택하지 말 것
- `virtio-ramfb` 로도 커널이 화면을 넘겨받는 시점에 출력이 끊길 수 있음 → 그래서 ④ 의 직렬 포트를 함께 붙임

#### ④ 직렬 포트 (콘솔 확보용, 필수)

- 좌측 **`사운드`** 아래의 **`새로 만들기…`** → **`직렬 포트`** 선택 (맨 아래 드라이브 그룹의 `새로 만들기…` 가 아님)
- 대상을 **`내장 터미널`** 로 두고 저장 → VM 실행 시 창에 **디스플레이 ↔ 직렬 포트 전환 탭**이 생김
- 게스트 커널이 직렬로 출력하도록 지정하는 작업은 설치 후 2-4 에서 수행
- Part 07 의 `rd.break` 복구·emergency 모드, Part 08 의 SSH 포트 변경 실패 시 **유일한 통로**가 되므로 반드시 추가

#### ⑤ 네트워크

- 좌측 **`네트워크`** → 네트워크 모드 **Shared Network** (NAT, 서브넷 `192.168.64.0/24`) → **저장**

**검증** (설치 완료 후 게스트에서 확인 — 3-5 에서 재확인)

```bash
lsblk -d -o NAME,SIZE,TYPE
nproc; free -g | head -2
```

```text
NAME SIZE TYPE
sr0  ...  rom
vda   40G disk
vdb    5G disk
vdc    5G disk
vdd    5G disk
vde    2G disk
2
               total        used        free ...
Mem:               3    ...
```

> 📝 **시험 포인트**: virtio 디스크는 `/dev/vd*`, SCSI/SATA 는 `/dev/sd*`, NVMe 는 `/dev/nvme0n1p1` — 장치 파일명 규칙 문제에서 `/dev/vda` = 가상머신 virtio 디스크로 출제.

### 1-3. Rocky Linux 9 설치 시 선택 항목

> **상황**: Anaconda 설치 프로그램에서 파티션은 자동(LVM VG `rlm`), root 비밀번호 설정, 관리자 계정 `admin1` 을 wheel 그룹 포함으로 생성한다. 이 값이 이후 파트의 전제다.

- VM 시작 → GRUB 메뉴 **Install Rocky Linux 9** → 언어 **English (United States)** (한글 로케일은 3-8 에서 참고 설정)
- **Installation Destination** → `vda` 40 GiB 만 선택 (vdb~vde 는 선택 해제) → Storage Configuration **Automatic** → Done
  - 자동 파티션 결과: `/boot/efi`(vfat) · `/boot`(xfs) · LVM VG `rlm` 의 `root`·`swap` LV
  - **VG 이름은 설치 매체에 따라 다름** — Minimal ISO 는 `rlm`, DVD ISO 는 `rl`. 이후 절차서의 `rlm-root`·`rlm-swap` 은 `lsblk` 로 확인한 실제 이름으로 치환
- **Software Selection** → **Minimal Install** (X 윈도 미설치 — 이후 GUI 항목은 `※ 미실행`)
- **Network & Host Name** → `enp0s1` 스위치 **ON** (DHCP) · Host Name 은 비워 둠 (3-1 에서 명령으로 설정)
- **Time & Date** → Asia/Seoul (설치 후 3-8 에서 `timedatectl` 로 재확인)
- **Root Password** → 설정, "Allow root SSH login with password" 는 **체크 해제** (SSH 강화는 Part 08)
- **User Creation** → Full name/User name **admin1** → **Make this user administrator** ☑ (wheel 그룹 자동 추가) → 비밀번호 설정
- **Begin Installation** → 완료 후 **Reboot System**

#### 설치 직후 — ISO 제거 (⚠️ 생략하면 설치 화면이 다시 뜸)

- 재부팅하면 UEFI 가 **디스크보다 CD 를 먼저** 잡아 `Install Rocky Linux Minimal 9.x` GRUB 메뉴가 다시 나타남
  - 이때 `Install…` 을 고르면 **방금 설치한 디스크를 처음부터 덮어씀** → 절대 선택 금지
  - Rocky 9 의 `Troubleshooting` 하위에는 `Boot from local drive` 항목이 **없음**(구 CentOS 계열에만 존재)
- 제거 방법 — VM 실행 중이면 창 상단 툴바의 **디스크(CD) 아이콘** → 해당 드라이브 → **꺼내기**
- VM 정지 상태면 `srv01` 우클릭 → **편집** → 좌측 **드라이브** 그룹의 **`USB 드라이브`**(이미지 종류 `CD/DVD`) → 이미지 비우고 **저장**
- ISO 를 뺀 뒤 재시작하면 디스크의 GRUB(`Rocky Linux 9.x`) 로 부팅
- 한 번만 디스크로 부팅해 보려면 재시작 직후 GRUB 이 뜨기 **전에** `Esc` 를 여러 번 눌러 UEFI 설정 → **Boot Manager** → 디스크 항목 선택

**검증** (첫 로그인 후)

```bash
id admin1
sudo -l | tail -2
lsblk -f /dev/vda
```

```text
uid=1000(admin1) gid=1000(admin1) groups=1000(admin1),10(wheel)
User admin1 may run the following commands on srv01:
    (ALL) ALL
NAME        FSTYPE      ... MOUNTPOINTS
vda
├─vda1      vfat        ... /boot/efi
├─vda2      xfs         ... /boot
└─vda3      LVM2_member
  ├─rlm-root xfs         ... /
  └─rlm-swap swap        ... [SWAP]
```

> 📝 **시험 포인트**: 설치 시 "관리자로 지정" = `wheel` 그룹(GID 10) 추가 → `/etc/sudoers` 의 `%wheel ALL=(ALL) ALL` 로 sudo 권한. UEFI 부팅은 `/boot/efi`(ESP, vfat) 파티션이 반드시 존재.

---

## 2. 첫 로그인과 원격 접속

### 2-1. 게스트 IP 확인 — 호스트 쪽과 게스트 쪽 두 경로

> **상황**: DHCP 로 받은 주소를 알아야 SSH 로 붙을 수 있다. 게스트 콘솔에서 확인하는 방법이 정석이지만, 화면이 안 나오거나 콘솔 입력이 번거로울 때를 대비해 **macOS 쪽에서 조회하는 방법**을 함께 익힌다. 이 방법은 게스트에 로그인하지 않고도 부팅 성공 여부까지 판정할 수 있어 진단용으로 유용하다.

**경로 A — macOS 호스트에서 조회 (권장)**

UTM 의 Shared Network 는 macOS 의 `vmnet` 공유 네트워크를 쓰므로, 호스트의 DHCP 서버가 임대 기록을 남긴다.

```bash
# macOS 터미널
cat /var/db/dhcpd_leases                 # 임대된 IP 와 MAC 목록
ifconfig | grep -B4 '192.168.64.1'       # 게스트가 붙은 브리지 인터페이스
```

- `/var/db/dhcpd_leases` : macOS 내장 DHCP 서버의 임대 기록 — `ip_address`·`hw_address` 쌍
- 여러 VM 이 있으면 MAC 으로 구분 → UTM 설정의 **네트워크 → MAC 주소**와 대조
- `192.168.64.1` 은 호스트(게이트웨이) 주소이며 게스트의 기본 게이트웨이·DNS

```bash
# 응답·SSH 개방 확인 (IP 는 위에서 찾은 값)
ping -c1 192.168.64.3
nc -z -G2 192.168.64.3 22 && echo "sshd 응답"
```

- `nc -z` : 데이터 전송 없이 포트 개방 여부만 확인 (**z**ero-I/O)
- `-G2` : 연결 시도 제한 2초 (macOS 판 `nc`)
- **22 번이 열려 있으면 부팅이 끝나고 `sshd` 까지 올라온 것** → 화면이 검어도 서버는 정상

**경로 B — 게스트 콘솔에서 확인 (정석)**

UTM 창(디스플레이 또는 직렬 포트 탭)에서 `admin1` 로 로그인한 뒤 조회한다.

```bash
ip a                        # 전체 인터페이스 주소
ip -4 addr show enp0s1      # IPv4 만, 인터페이스 지정
hostname -I                 # 커널이 알고 있는 IP 목록만 간단 출력
```

- `ip a` : `ip address show` 축약 — 인터페이스별 L2/L3 주소
- `-4` : IPv**4** 만 출력
- `hostname -I` : 모든 네트워크 주소(**I**P addresses) 출력, 루프백 제외

**검증**

```bash
ip route show default
ping -c 2 192.168.64.1
```

```text
default via 192.168.64.1 dev enp0s1 proto dhcp src 192.168.64.x metric 100
2 packets transmitted, 2 received, 0% packet loss
```

> 📝 **시험 포인트**: `ifconfig` 는 `net-tools` 설치 후에만 사용 가능(Part 08) — RHEL 9 기본 도구는 `ip`. `ip a` 출력의 `inet` = IPv4, `inet6` = IPv6, `link/ether` = MAC.

### 2-2. macOS 터미널에서 SSH 접속

> **상황**: UTM 콘솔은 복사·붙여넣기가 불편하므로 이후 실습은 SSH 세션에서 진행한다. 첫 접속 시 호스트 키 지문을 저장한다.

```bash
# macOS 터미널
ssh admin1@192.168.64.x            # x = 2-1 에서 확인한 값
# "Are you sure you want to continue connecting (yes/no/[fingerprint])?" → yes
```

- `ssh <사용자>@<호스트>` : 원격 로그인. 포트 생략 시 22 (Part 08 에서 2222 로 변경 → `-p 2222` 필요)
- 첫 접속 시 서버 호스트 키가 `~/.ssh/known_hosts` 에 저장 → 이후 키가 바뀌면 경고

**별칭 등록** — 이후 모든 파트에서 `ssh srv01` 한 줄로 접속

```bash
# macOS 터미널
cat >> ~/.ssh/config <<'EOF'

Host srv01
    HostName 192.168.64.3
    User admin1
EOF
chmod 600 ~/.ssh/config
ssh srv01
```

- `Host` : 별칭. `ssh srv01` 로 아래 설정이 적용됨
- `HostName` : 실제 주소 — Part 08 에서 고정 IP `192.168.64.10` 으로 바꾸면 이 값만 수정
- `User` : 생략 시 macOS 로그인 계정으로 접속 시도
- Part 08 에서 포트를 2222 로 바꾼 뒤에는 `Port 2222`, 키 인증 후에는 `IdentityFile ~/.ssh/id_ed25519` 를 추가
- `~/.ssh/config` 권한이 느슨하면 무시되므로 `600` 필요

**검증** (VM 쪽 세션에서)

```bash
echo $SSH_CONNECTION
who am i
```

```text
192.168.64.1 5xxxx 192.168.64.x 22
admin1   pts/0        2026-09-03 10:00 (192.168.64.1)
```

> 📝 **시험 포인트**: `who am i`(공백 포함) 는 현재 터미널의 로그인 정보 한 줄, `whoami` 는 실효 사용자명만. `pts/N` = 원격/터미널 에뮬레이터 세션, `tty1` = 콘솔.

**터미널 종류(TERM) 문제 해결** — Ghostty·Kitty·WezTerm 등 최신 터미널 사용 시 필수

접속 직후 `clear` 나 `top` 이 `'xterm-ghostty': unknown terminal type.` 으로 실패하면, 클라이언트가 보낸 `TERM` 값에 해당하는 terminfo 정의가 서버에 없는 것이다. `vi`(Part 04) · `top`(Part 06) · `less` · `nmtui`(Part 08) 등 화면을 그리는 프로그램이 모두 같은 이유로 실패하므로 먼저 해결한다.

```bash
# macOS 터미널 (SSH 세션이 아닌 로컬)
infocmp -x | ssh srv01 -- tic -x -
```

- `infocmp -x` : 로컬 터미널의 terminfo 정의를 텍스트로 출력 (**x**: 확장 기능 포함). 인자를 생략하면 현재 `$TERM` 대상
- `tic -x -` : 표준 입력으로 받은 정의를 컴파일해 설치 (**t**erminfo **c**ompiler)
- 서버의 `~/.terminfo/` 에 저장 → root 권한 불필요, 해당 사용자에게만 적용
- 서버에 `tic` 이 없으면 `sudo dnf install -y ncurses` 로 설치 후 재시도

간단한 대안 — 서버 쪽에서 표준 터미널 종류로 고정 (Ghostty 고유 기능은 포기)

```bash
echo 'export TERM=xterm-256color' >> ~/.bashrc
source ~/.bashrc
```

**검증**

```bash
echo $TERM
clear && echo "clear 정상"
tput cols; tput lines
infocmp | head -1
```

```text
xterm-ghostty
clear 정상
120
30
#	Reconstructed via infocmp from file: /home/admin1/.terminfo/x/xterm-ghostty
```

- `tput cols`/`lines` : terminfo 를 읽어 터미널 크기 조회 → 숫자가 나오면 정상 인식
- `infocmp` 첫 줄의 경로로 어느 정의를 쓰는지 확인

> 📝 **시험 포인트**: `TERM` 은 터미널 종류를 알려주는 환경변수이며 정의는 `/usr/share/terminfo/` 에 저장(구형은 `/etc/termcap`). 화면 제어 프로그램이 커서 이동·색상 제어 문자열을 여기서 조회 (Part 01 7절 환경변수 참조).

### 2-3. 직렬 콘솔 활성화 — 커널 출력 경로 지정

> **상황**: 1-2 ④ 에서 직렬 포트 장치를 붙였지만, 커널이 그쪽으로 출력하라는 지시를 받지 않으면 탭은 비어 있다. 커널 명령줄에 콘솔을 추가해 **화면과 직렬 양쪽으로 부팅 로그와 로그인 프롬프트가 나오도록** 만든다. Part 07 의 복구 실습과 Part 08 의 네트워크 변경 때 SSH 가 끊기므로, 지금 미리 확보해 둔다.

```bash
sudo grubby --update-kernel=ALL --args="console=tty0 console=ttyAMA0,115200"
sudo grubby --info=ALL | grep -E '^(title|args)'
```

- `grubby` : GRUB 설정을 직접 편집하지 않고 부팅 항목을 다루는 도구 (Part 07 에서 상세)
- `--update-kernel=ALL` : 설치된 **모든** 커널 항목에 적용 (`DEFAULT` 는 기본 항목만)
- `--args="…"` : 커널 명령줄에 파라미터 추가 (제거는 `--remove-args`)
- `console=tty0` : 그래픽 콘솔 (UTM 디스플레이 탭)
- `console=ttyAMA0,115200` : aarch64 `virt` 머신의 PL011 직렬 포트, 속도 115200 bps
- `console=` 를 여러 번 지정하면 모두 출력하되 **마지막에 적은 장치가 `/dev/console`**(로그인 프롬프트가 뜨는 곳)

**검증**

```bash
sudo reboot
# 재접속 후
cat /proc/cmdline
systemctl status serial-getty@ttyAMA0.service --no-pager | head -3
```

```text
BOOT_IMAGE=... console=tty0 console=ttyAMA0,115200
● serial-getty@ttyAMA0.service - Serial Getty on ttyAMA0
     Loaded: loaded (/usr/lib/systemd/system/serial-getty@.service; ...)
     Active: active (running) ...
```

- 재부팅 중 UTM 창의 **직렬 포트 탭**에 부팅 로그가 흐르고 로그인 프롬프트가 뜨면 성공
- 장치 이름이 다르면 `dmesg | grep -i tty` 로 실제 이름 확인 후 `--remove-args` 로 지우고 다시 지정
- `serial-getty@ttyAMA0` 은 `console=` 지정 시 systemd 가 자동 활성화

> 📝 **시험 포인트**: 커널 파라미터 추가·삭제는 `grubby --update-kernel=ALL --args=` / `--remove-args=` 가 RHEL 9 표준. `/etc/default/grub` 수정 후 `grub2-mkconfig` 는 신규 설치 항목에만 반영되는 경우가 있어 실기에서는 `grubby` 가 정답으로 출제 (Part 07 참조).

### 2-4. su / su - / sudo -i 환경 차이

> **상황**: 관리 작업은 root 로 해야 하지만 `su` 와 `su -` 는 환경변수 처리가 다르다. 이후 파트에서 "PATH 에 /usr/sbin 이 없어 명령이 안 보이는" 실수를 막기 위해 차이를 직접 확인한다.

```bash
# ① 현재 admin1 상태 기록
echo $USER; echo $HOME; pwd; echo $PATH

# ② su (하이픈 없음) — root 로 바뀌지만 환경은 admin1 것 유지
su
echo $USER; echo $HOME; pwd; echo $PATH
exit

# ③ su - (로그인 셸) — root 의 환경으로 완전히 전환
su -
echo $USER; echo $HOME; pwd; echo $PATH
exit

# ④ sudo -i — admin1 비밀번호로 root 로그인 셸
sudo -i
echo $USER; echo $HOME; pwd; echo $PATH
exit

# ⑤ 한 줄 명령만 root 로
sudo cat /etc/shadow | head -1
sudo -s                      # 비로그인 root 셸 (환경 유지, su 와 유사)
exit
```

- `su` : **s**witch **u**ser — 인자 없으면 root. 대상 사용자 비밀번호 필요
- `su -` (= `su -l`, `su --login`) : 로그인 셸로 전환 → `HOME`·`PATH`·작업 디렉터리가 대상 사용자 기준으로 재설정
- `sudo -i` : root 로그인 셸 시뮬레이션(**i**nitial login) — **자기** 비밀번호 사용, `/etc/sudoers` 권한 필요
- `sudo -s` : 현재 환경을 유지한 root **s**hell
- `sudo <명령>` : 단일 명령만 권한 상승, `/var/log/secure` 에 기록

**검증**

```bash
sudo -i
echo "$USER $HOME $(pwd)"; echo $PATH | tr ':' '\n' | grep sbin
logname; whoami; id -un
exit
```

```text
root /root /root
/usr/local/sbin
/usr/sbin
admin1
root
root
```

> 📝 **시험 포인트**: "`su` 는 대상 사용자 비밀번호, `sudo` 는 자신의 비밀번호" 와 "`su -` 만 환경변수 초기화" 가 반복 출제. `logname` 은 최초 로그인 이름(admin1), `whoami` 는 실효 UID 이름(root) — 결과가 다른 이유를 묻는 문항.

---

## 3. 시스템 식별

> 이 절부터 별도 표기가 없으면 **`sudo -i` 로 얻은 root 셸(`#`)** 에서 진행. 조회 명령은 일반 사용자도 대부분 가능.

### 3-1. 호스트명 설정

> **상황**: 시나리오의 서버 이름 `srv01.lab.local` 을 부여한다. 설치 시 비워 두었으므로 현재는 `localhost` 이다.

```bash
hostnamectl                                   # 현재 상태
hostnamectl set-hostname srv01.lab.local      # 정적 호스트명 설정 (/etc/hostname 갱신)
hostnamectl set-hostname "Lab Intranet Server" --pretty   # 표시용 이름 (선택)
```

- `hostnamectl` : systemd 호스트명 관리 도구. 인자 없으면 `status`
- `set-hostname <이름>` : 정적(static)·일시(transient)·표시(pretty) 이름을 한 번에 설정
- `--pretty` : 공백·대문자 허용되는 **표시용** 이름만 설정 (`/etc/machine-info`)
- `hostname <이름>` 은 재부팅 시 사라지는 일시 이름만 변경 → 영구 설정은 `hostnamectl` 또는 `/etc/hostname`

**검증**

```bash
cat /etc/hostname
hostname; hostname -s; hostname -d; hostname -f
hostnamectl --static
exec bash          # 프롬프트에 새 이름 반영 (또는 재로그인)
```

```text
srv01.lab.local
srv01.lab.local
srv01
lab.local
srv01.lab.local
srv01.lab.local
[root@srv01 ~]#
```

> 📝 **시험 포인트**: `-s`(**s**hort) 짧은 이름, `-d`(**d**omain) 도메인, `-f`(**f**qdn) 전체 이름. 정적 호스트명 파일은 `/etc/hostname` (RHEL 6 의 `/etc/sysconfig/network` 는 폐지).

### 3-2. /etc/hosts 등록

> **상황**: Part 09 에서 BIND 를 세우기 전까지는 이름 해석을 `/etc/hosts` 에 의존한다. 서버의 최종 고정 IP `192.168.64.10` 으로 등록해 두고, 현재 DHCP 주소와 다른 점은 주석으로 남긴다.

```bash
cp -a /etc/hosts /etc/hosts.orig
cat >> /etc/hosts <<'EOF'
# LAB: srv01 고정 IP (Part 08 에서 nmcli 로 실제 부여. 그 전까지 DHCP 주소와 다름)
192.168.64.10   srv01.lab.local srv01
EOF
cat /etc/hosts
```

- `cp -a` : **a**rchive — 권한·소유자·타임스탬프 보존 복사 (원본 백업 습관)
- `/etc/hosts` 형식 : `IP  정식이름(FQDN)  별명...` — 공백/탭 구분, `#` 주석
- 이름 해석 순서는 `/etc/nsswitch.conf` 의 `hosts: files dns myhostname` → hosts 파일이 DNS 보다 우선

**검증**

```bash
getent hosts srv01
getent hosts srv01.lab.local
ping -c 1 srv01      # 고정 IP 부여 전이므로 실패 또는 무응답이 정상
grep -n hosts /etc/nsswitch.conf
```

```text
192.168.64.10   srv01.lab.local srv01
192.168.64.10   srv01.lab.local srv01
...
hosts:      files dns myhostname
```

> 📝 **시험 포인트**: `/etc/hosts` 필드 순서(IP → FQDN → 별명), `/etc/nsswitch.conf` 의 `hosts:` 줄이 해석 순서 결정. `/etc/resolv.conf` 는 DNS 서버 지정 파일 (Part 08).

### 3-3. 커널·OS·아키텍처

> **상황**: 시험 기준(RHEL 계열)과 실습 환경(aarch64)의 차이를 명확히 하기 위해 커널 버전과 배포판 정보를 확인한다.

```bash
uname -a          # 전체
uname -r          # 커널 릴리스
uname -m          # 하드웨어 아키텍처
uname -s -n -v    # 커널 이름 · 호스트명 · 커널 빌드 버전
arch              # uname -m 과 동일
cat /etc/os-release
cat /etc/redhat-release
cat /etc/rocky-release
ls -l /etc/system-release      # → rocky-release 심볼릭 링크
```

- `uname -a` : **a**ll — 커널명·호스트명·릴리스·빌드 버전·머신·OS 를 한 줄에
- `-r` : 커널 **r**elease (`5.14.0-xxx.el9.aarch64` 형식 → `el9` 가 RHEL 9 계열 표시)
- `-m` : **m**achine 하드웨어 이름 (`aarch64` / `x86_64`)
- `-s` : 커널 이름(**s**ystem), `-n` : **n**odename, `-v` : 커널 빌드 **v**ersion(날짜)
- `/etc/os-release` : systemd 표준 배포판 식별 파일 (`ID=rocky`, `VERSION_ID`, `PLATFORM_ID`)
- `/etc/redhat-release` : RHEL 계열 호환 파일 — Rocky 는 `rocky-release` 를 가리키는 심볼릭 링크

**검증**

```bash
grep -E '^(NAME|VERSION_ID|ID|PLATFORM_ID)=' /etc/os-release
uname -r | grep -o 'el9'
```

```text
NAME="Rocky Linux"
VERSION_ID="9.x"
ID="rocky"
PLATFORM_ID="platform:el9"
el9
```

> 📝 **시험 포인트**: 커널 버전 `5.14.0-xxx` = 메이저.마이너.패치 — 3.0 이후 짝수/홀수 안정·개발 구분 규칙은 폐지. `uname -r` 과 `uname -m` 옵션 혼동 문제 빈출.

### 3-4. CPU 와 메모리

> **상황**: Part 06 에서 부하 진단을 하려면 코어 수와 메모리 총량을 알아야 한다. 요약 도구와 `/proc` 원본을 함께 본다.

```bash
lscpu
lscpu | grep -E '^(Architecture|CPU\(s\)|Model name|Vendor ID)'
nproc
cat /proc/cpuinfo | head -20
grep -c ^processor /proc/cpuinfo

free -h
free -m
cat /proc/meminfo | head -5
grep -E 'MemTotal|SwapTotal' /proc/meminfo
```

- `lscpu` : `/proc/cpuinfo` + sysfs 를 요약 — 아키텍처·코어·소켓·스레드·캐시
- `nproc` : 사용 가능한 프로세서 수만 출력
- `/proc/cpuinfo` : 커널이 제공하는 CPU 원본 정보 (aarch64 는 `Model name` 대신 `CPU implementer`·`CPU part` 위주)
- `free -h` : **h**uman-readable 단위(Gi/Mi), `-m` : **m**egabyte 고정 단위
- `/proc/meminfo` : `free` 의 원본 — `MemTotal`·`MemFree`·`MemAvailable`·`Buffers`·`Cached`·`SwapTotal`

**검증**

```bash
[ "$(nproc)" -eq 2 ] && echo "vCPU OK"
free -g | awk '/^Mem:/{print "Mem(GiB)=" $2}'
awk '/MemTotal/{printf "MemTotal(MiB)=%d\n", $2/1024}' /proc/meminfo
```

```text
vCPU OK
Mem(GiB)=3
MemTotal(MiB)=3xxx
```

> 📝 **시험 포인트**: `free` 출력의 `available` = 실제 새 프로세스가 쓸 수 있는 양(buff/cache 회수분 포함) — `free` 열만 보고 "메모리 부족" 으로 판단하는 것이 함정. `/proc/cpuinfo`·`/proc/meminfo` 는 디스크를 점유하지 않는 가상 파일.

### 3-5. 디스크와 파일시스템

> **상황**: 1-2 에서 붙인 추가 디스크 4개가 커널에 인식되었는지, 시스템 디스크의 파티션·LVM 구성이 어떤지 확인한다. Part 05 의 대상 장치를 여기서 확정한다.

```bash
lsblk                       # 블록 장치 트리
lsblk -f                    # 파일시스템 유형·UUID·마운트 포인트
lsblk -d -o NAME,SIZE,TYPE,TRAN   # 디스크만, 전송 방식 포함
df -hT                      # 마운트된 FS 의 사용량 + 유형
df -i                       # inode 사용량
cat /proc/partitions
```

- `lsblk` : 블록 장치를 트리로 출력 (`/sys/dev/block` 기반). 기본 열 NAME·MAJ:MIN·RM·SIZE·RO·TYPE·MOUNTPOINTS
- `-f` : **f**ilesystem 정보(FSTYPE·LABEL·UUID·FSAVAIL·FSUSE%·MOUNTPOINTS)
- `-d` : 파티션 없이 **d**isk 만, `-o` : 출력 열(**o**utput) 지정
- `df -h` : **h**uman-readable, `-T` : 파일시스템 **T**ype 열 추가, `-i` : **i**node 사용량
- `/proc/partitions` : 커널이 인식한 파티션 목록(major·minor·블록 수·이름)

**검증**

```bash
lsblk -dn -o NAME,SIZE | grep -E '^vd[b-e]'
df -hT / /boot /boot/efi | awk '{print $1, $2, $7}'
```

```text
vdb   5G
vdc   5G
vdd   5G
vde   2G
Filesystem Type Mounted
/dev/mapper/rlm-root xfs /
/dev/vda2 xfs /boot
/dev/vda1 vfat /boot/efi
```

> 📝 **시험 포인트**: `df -i` 는 inode 고갈 점검 — "용량은 남았는데 파일 생성 불가" 상황의 원인. `lsblk -f` 로 UUID 확인 후 `/etc/fstab` 에 사용(Part 05). `blkid` 도 UUID·TYPE 확인 명령.

### 3-6. PCI · USB · DMI 하드웨어 조회

> **상황**: 가상 환경이라 실제 장치는 적지만, 하드웨어 조회 명령 자체가 시험 범위다. 도구 패키지가 없으면 설치한다(상세는 Part 02).

```bash
rpm -q pciutils usbutils dmidecode || dnf install -y pciutils usbutils dmidecode

lspci                  # PCI 장치 목록
lspci -v | head -20    # 상세
lsusb                  # USB 장치 목록 (VM 은 비어 있거나 QEMU 태블릿 1개)
dmidecode -t system    # SMBIOS/DMI 시스템 정보
dmidecode -t memory | head -20
```

- `lspci` : PCI 버스의 장치 나열 (`pciutils`). `-v` : **v**erbose
- `lsusb` : USB 버스·장치 나열 (`usbutils`)
- `dmidecode` : 펌웨어의 SMBIOS(DMI) 테이블 해석. `-t <type>` : **t**ype 지정(`system`, `bios`, `memory`, `processor`)
- Apple Virtualization 백엔드에서는 DMI 테이블이 없어 `No SMBIOS nor DMI entry point found` 가 나올 수 있음 → 환경 특성이며 오류 아님

**검증**

```bash
lspci | grep -ci virtio
lspci -nn | grep -iE 'virtio|ethernet|scsi' | head
```

```text
...
00:01.0 Ethernet controller [0200]: Red Hat, Inc. Virtio network device [1af4:1041]
00:02.0 SCSI storage controller [0100]: Red Hat, Inc. Virtio block device [1af4:1042]
...
```

> 📝 **시험 포인트**: `lspci`/`lsusb`/`lsblk`/`lscpu` — "ls + 장치종류" 명령 매칭 문제. udev 가 장치 연결 시 `/dev` 에 장치 파일을 동적 생성하며 규칙은 `/etc/udev/rules.d/`.

### 3-7. 커널 메시지 · 가동 시간 · 날짜

> **상황**: 부팅 직후 커널이 남긴 메시지와 현재 가동 시간·시각을 확인한다. `dmesg` 는 4-4 에서 하드웨어 인식 확인에 다시 쓴다.

```bash
dmesg | head -20
dmesg -T | tail -5           # 사람이 읽는 타임스탬프
uptime
uptime -p                    # pretty
uptime -s                    # since — 부팅 시각
date
date '+%Y-%m-%d %H:%M:%S %Z'
date +%s                     # epoch 초
cat /proc/uptime
```

- `dmesg` : 커널 링 버퍼 출력 (RHEL 9 기본은 root 만 가능 — `kernel.dmesg_restrict=1`)
- `-T` : 사람이 읽을 수 있는 **T**imestamp 로 변환
- `uptime -p` : **p**retty (`up 1 hour, 5 minutes`), `-s` : 부팅 시각(**s**ince)
- `date +포맷` : `%Y`연 `%m`월 `%d`일 `%H`시 `%M`분 `%S`초 `%Z`시간대 `%s`epoch
- `/proc/uptime` : 가동 초 · 유휴 초 (uptime 의 원본)

**검증**

```bash
uptime | grep -o 'load average.*'
date -d "@$(date +%s)" | grep -q "$(date +%Y)" && echo "epoch OK"
```

```text
load average: 0.00, 0.01, 0.05
epoch OK
```

> 📝 **시험 포인트**: `uptime` 의 load average 3값 = 1·5·15분 평균 실행 대기 프로세스 수 (Part 06 `top` 첫 줄과 동일). `dmesg` 는 `/var/log/dmesg` 파일이 아닌 커널 링 버퍼를 읽음.

### 3-8. 시간대 · 로케일 · 달력

> **상황**: 로그 시각이 KST 로 기록되도록 시간대를 `Asia/Seoul` 로 고정한다. 로케일은 영어를 유지하되 변경 명령을 익힌다.

```bash
timedatectl                              # 현재 시각·시간대·NTP 상태
timedatectl list-timezones | grep Seoul
timedatectl set-timezone Asia/Seoul
timedatectl set-ntp true                 # chronyd 활성 (상세 Part 08)

localectl                                # 현재 로케일·키맵
localectl list-locales | head
localectl list-locales | grep -i ko      # 한글 로케일 없으면 glibc-langpack-ko 설치 필요
# localectl set-locale LANG=ko_KR.UTF-8  # 참고 — 이 실습은 en_US.UTF-8 유지
localectl list-keymaps | grep -E '^(us|kr)$'

cal
cal 2026
cal -3                                   # 전월·당월·익월
cal -j | head -3                         # 율리우스일(1~365)
```

- `timedatectl set-timezone <Area/City>` : `/etc/localtime` 심볼릭 링크를 `/usr/share/zoneinfo/...` 로 교체
- `set-ntp true` : `systemd-timedated` 가 NTP 클라이언트(chronyd) 활성화
- `localectl set-locale LANG=...` : `/etc/locale.conf` 갱신 — 재로그인 후 적용
- `list-locales` / `list-keymaps` : 사용 가능한 로케일·콘솔 키맵 목록
- `cal -3` : **3**개월, `-j` : **j**ulian 일수 표기, `cal <연도>` : 연간

**검증**

```bash
timedatectl | grep -E 'Time zone|NTP service'
ls -l /etc/localtime
date +%Z
cat /etc/locale.conf
```

```text
                Time zone: Asia/Seoul (KST, +0900)
              NTP service: active
lrwxrwxrwx. 1 root root 30 ... /etc/localtime -> ../usr/share/zoneinfo/Asia/Seoul
KST
LANG=en_US.UTF-8
```

> 📝 **시험 포인트**: 시간대 설정 파일은 `/etc/localtime`(심볼릭 링크), 로케일 파일은 `/etc/locale.conf`, 키맵은 `/etc/vconsole.conf`. `LANG` 환경변수가 메시지 언어·정렬·날짜 형식을 결정.

---

## 4. 부팅 과정 관찰

### 4-1. 부팅 소요 시간 분석

> **상황**: 방금 부팅한 시스템이 커널·사용자 공간 각각에 얼마나 걸렸는지, 어떤 유닛이 느렸는지 확인한다. 이후 서비스가 늘어날 때 비교 기준이 된다.

```bash
systemd-analyze                  # 전체 부팅 시간 (= systemd-analyze time)
systemd-analyze blame | head     # 유닛별 소요 시간 내림차순
systemd-analyze critical-chain   # 기본 타겟까지의 의존 사슬
```

- `systemd-analyze` (= `time`) : firmware → loader → kernel → initrd → userspace 각 단계 시간
- `blame` : 각 유닛의 활성화 소요 시간을 **긴 순서**로 나열
- `critical-chain` : 타겟 도달을 지연시킨 유닛 사슬 (`@`=활성 시각, `+`=소요 시간)

**검증**

```bash
systemd-analyze | grep -o 'Startup finished.*'
systemd-analyze blame | wc -l
```

```text
Startup finished in ...s (kernel) + ...s (initrd) + ...s (userspace) = ...s
...
```

> 📝 **시험 포인트**: "부팅을 지연시킨 유닛을 소요 시간 내림차순으로 확인" → `systemd-analyze blame`. `plot` 은 SVG 그래프 생성, `verify` 는 유닛 파일 문법 검사.

### 4-2. 커널 부팅 매개변수와 /boot 내용

> **상황**: GRUB2 가 커널에 어떤 인자를 넘겼는지, `/boot` 에 어떤 파일이 있고 설치된 커널 패키지가 몇 개인지 본다. GRUB 설정 변경은 Part 07 에서 다룬다.

```bash
cat /proc/cmdline
ls /boot
ls -l /boot/vmlinuz-* /boot/initramfs-*.img
ls /boot/efi/EFI/rocky/            # UEFI 부트로더 파일 (grubaa64.efi, grub.cfg)
ls /boot/loader/entries/           # BLS 방식 부팅 항목
rpm -q kernel
rpm -q kernel-core
grubby --default-kernel
```

- `/proc/cmdline` : 현재 커널이 받은 부팅 인자 — `BOOT_IMAGE`, `root=`, `ro`, `crashkernel=`, `rd.lvm.lv=`
- `/boot/vmlinuz-<버전>` : 압축 커널 이미지, `initramfs-<버전>.img` : 초기 램디스크, `System.map-*` : 커널 심볼, `config-*` : 커널 빌드 설정
- `/boot/efi/EFI/rocky/` : ESP 안의 GRUB2 EFI 바이너리(aarch64 는 `grubaa64.efi`, x86_64 는 `grubx64.efi`)
- `/boot/loader/entries/*.conf` : RHEL 8+ BLS(Boot Loader Specification) 항목 — `grub.cfg` 직접 편집 대신 여기서 커널 항목 관리
- `rpm -q kernel` : 설치된 커널 메타패키지 버전 (여러 개면 GRUB 메뉴에 그 수만큼 항목)
- `grubby --default-kernel` : 기본 부팅 커널 경로

**검증**

```bash
grep -o 'root=[^ ]*' /proc/cmdline
[ "$(grubby --default-kernel)" = "/boot/vmlinuz-$(uname -r)" ] && echo "running = default kernel"
```

```text
root=/dev/mapper/rlm-root
running = default kernel
```

> 📝 **시험 포인트**: 부팅 순서 "BIOS/UEFI(POST) → 부트로더(GRUB2) → 커널 + initramfs → systemd(PID 1) → default.target". GRUB2 설정은 `/etc/default/grub` + `/etc/grub.d/` 를 `grub2-mkconfig` 로 생성하며 `grub.cfg` 직접 편집 금지 (Part 07).

### 4-3. 런레벨과 systemd 타겟

> **상황**: 시스템이 어느 타겟(런레벨)으로 부팅되는지 확인한다. 미니멀 설치는 `multi-user.target`(런레벨 3) 이 기본이다.

```bash
systemctl get-default
runlevel
who -r
systemctl list-units --type=target --state=active
ls -l /usr/lib/systemd/system/runlevel*.target    # 런레벨 호환 심볼릭 링크
ls -l /etc/systemd/system/default.target
```

- `systemctl get-default` : 기본 타겟 (`default.target` 심볼릭 링크의 대상)
- `runlevel` : `이전 현재` 런레벨 출력 — systemd 호환 명령, 이전 값이 없으면 `N`
- `who -r` : utmp 의 **r**un-level 레코드 출력 (`run-level 3  2026-09-03 10:00`)
- `list-units --type=target --state=active` : 현재 활성 타겟 목록
- `/usr/lib/systemd/system/runlevelN.target` : SysV 런레벨 번호 → 타겟 심볼릭 링크

| 런레벨 | systemd 타겟 | 의미 | 전환 명령 |
| --- | --- | --- | --- |
| 0 | `poweroff.target` | 시스템 종료 | `systemctl poweroff` |
| 1 | `rescue.target` | 단일 사용자(복구) 모드, root 만 | `systemctl rescue` / `isolate rescue.target` |
| 2 | `multi-user.target` | (SysV: NFS 없는 다중 사용자) — systemd 는 3 과 동일 | `systemctl isolate multi-user.target` |
| 3 | `multi-user.target` | 다중 사용자 + 네트워크, 텍스트 콘솔 | 같음 |
| 4 | `multi-user.target` | 예약(미사용) — systemd 는 3 과 동일 | 같음 |
| 5 | `graphical.target` | 다중 사용자 + GUI 로그인 | `systemctl isolate graphical.target` |
| 6 | `reboot.target` | 재부팅 | `systemctl reboot` |

- `emergency.target` : rescue 보다 더 최소(루트 읽기 전용, 서비스 없음) — 런레벨 대응 없음
- 기본 타겟 변경은 `systemctl set-default <타겟>` — Part 07 에서 rescue 진입과 함께 실습

**검증**

```bash
systemctl get-default
runlevel
readlink -f /usr/lib/systemd/system/runlevel3.target
readlink -f /usr/lib/systemd/system/runlevel5.target
systemctl is-active multi-user.target graphical.target
```

```text
multi-user.target
N 3
/usr/lib/systemd/system/multi-user.target
/usr/lib/systemd/system/graphical.target
active
inactive
```

> 📝 **시험 포인트**: 런레벨↔타겟 표는 매 회차 출제 — 특히 1=rescue, 3=multi-user, 5=graphical, 6=reboot. "현재 런레벨 확인 명령" = `runlevel` 또는 `who -r`; "기본 타겟 확인" = `systemctl get-default`.

### 4-4. 하드웨어 인식과 부팅 오류 로그

> **상황**: 커널이 virtio 디스크·네트워크 장치를 제대로 잡았는지, 이번 부팅에서 오류가 있었는지 로그로 확인한다.

```bash
dmesg -T | grep -iE 'virtio|vd[a-e]'        # virtio 블록·네트워크 인식
dmesg -T | grep -iE 'memory|cpu' | head -5
dmesg -T | grep -iE 'error|fail' | head      # 커널 오류 메시지
journalctl -b -p err                          # 이번 부팅의 err 이상 로그
journalctl -b -p warning --no-pager | tail -5
journalctl -k | head -5                       # 커널 메시지만 (dmesg 와 유사)
journalctl --list-boots
```

- `dmesg -T | grep -i` : 대소문자 무시(**i**gnore-case) 검색으로 장치 인식 줄 추출
- `journalctl -b` : 현재 **b**oot 의 로그만 (`-b -1` = 직전 부팅)
- `-p err` : **p**riority 필터 — `emerg(0) alert(1) crit(2) err(3) warning(4) notice(5) info(6) debug(7)` 중 err 이상
- `-k` : **k**ernel 메시지만 (`dmesg` 와 동일 출처)
- `--no-pager` : less 없이 바로 출력, `--list-boots` : 저널에 남은 부팅 목록 (기본은 휘발성 → Part 07 에서 영구 저장 설정)

**검증**

```bash
dmesg | grep -c virtio
journalctl -b -p err --no-pager | grep -vc '^-- '   # 오류 건수 (0 이면 깨끗한 부팅)
```

```text
...
0
```

> 📝 **시험 포인트**: `journalctl -b`(현재 부팅), `-u <유닛>`, `-p <우선순위>`, `-f`(follow), `-k`(커널), `--since` 옵션 의미 문제. 우선순위 숫자·이름 대응(err=3) 암기.

---

## 5. 접속·세션 정보

### 5-1. 현재 접속자와 자기 신원

> **상황**: 서버에 누가 접속해 있고 내 세션은 어떤 터미널인지 확인한다. 이후 `wall`·`write` 대상 확인에도 사용한다.

```bash
who               # 접속 사용자 · 터미널 · 시각 · 원격지
who -H            # 헤더 포함
who -b            # 마지막 부팅 시각
who -q            # 사용자명과 인원수만
w                 # who + 각 세션의 현재 작업·부하
whoami
id
id -u; id -g; id -G; id -un
tty
logname
```

- `who` : `/var/run/utmp` 의 현재 로그인 세션. `-H` : 헤더(**H**eading), `-b` : **b**oot 시각, `-q` : **q**uick 카운트, `-r` : 런레벨
- `w` : 접속자 + 유휴 시간(IDLE)·JCPU·PCPU·현재 명령(WHAT). 첫 줄은 `uptime`
- `whoami` : 실효 UID 의 이름, `id` : UID·GID·보조 그룹 전체
- `id -u`(UID) `-g`(주 GID) `-G`(모든 GID) `-n`(숫자 대신 이름, `-un` = 사용자명)
- `tty` : 현재 터미널 장치 파일 (`/dev/pts/0`)
- `logname` : 로그인 당시 이름(`/var/run/utmp` 기준) — `su` 후에도 원래 이름

**검증** (UTM 콘솔에도 admin1 로 로그인해 두면 세션 2개가 보임)

```bash
who | wc -l
w -h | awk '{print $1, $2, $3}'
```

```text
2
admin1 tty1 -
admin1 pts/0 192.168.64.1
```

> 📝 **시험 포인트**: `who` 는 `utmp`(현재), `last` 는 `wtmp`(이력), `lastb` 는 `btmp`(실패), `lastlog` 는 `lastlog`(사용자별 최근) — 명령↔파일 연결 문제 매 회차 출제.

### 5-2. 로그인 이력 · 실패 · 최근 로그인

> **상황**: 설치 후 지금까지의 로그인 이력, 실패 시도(무차별 대입 흔적), 사용자별 마지막 로그인을 점검한다. 비밀번호를 한 번 틀려서 `btmp` 에 기록을 남겨 본다.

```bash
# 사전 준비: macOS 에서 ssh admin1@<ip> 로 접속 시 비밀번호를 1회 틀린 뒤 정상 로그인

last                     # 로그인·로그아웃·재부팅 이력 (wtmp)
last -n 5                # 최근 5건
last reboot              # 재부팅 기록만
last -x | head           # 종료·런레벨 변경 포함
lastb                    # 로그인 실패 기록 (btmp, root 전용)
lastb -n 3
lastlog                  # 전 사용자의 마지막 로그인
lastlog -u admin1        # 특정 사용자
lastlog -t 1             # 최근 1일 이내 로그인한 사용자
```

- `last` : `/var/log/wtmp` 를 읽어 이력 출력. `-n N` : N 건, `reboot` : 가상 사용자 `reboot` 기록만, `-x` : 시스템 종료·런레벨 변경 포함(e**x**tended)
- `lastb` : `/var/log/btmp`(**b**ad login) 의 실패 기록 — 파일 권한 600 이므로 root 만
- `lastlog` : `/var/log/lastlog` 의 사용자별 최근 로그인. `-u` : **u**ser 지정, `-t N` : 최근 N 일(**t**ime) 이내

**검증**

```bash
last -n 1 admin1 | head -1
lastb | grep -c admin1
lastlog -u admin1 | tail -1
```

```text
admin1   pts/0        192.168.64.1     ... still logged in
1
admin1           pts/0    192.168.64.1     ... 2026
```

> 📝 **시험 포인트**: "`/var/log/btmp` 의 실패 기록 확인 명령" → `lastb` (필기 거의 매 회차). `last` 출력의 `still logged in`, `down`, `crash` 의미. `lastlog` 의 `**Never logged in**` 표기.

### 5-3. wtmp · btmp · lastlog 파일의 성격

> **상황**: 위 세 파일은 바이너리라 `cat` 으로 읽을 수 없다. 파일 유형을 확인하고 `utmpdump` 로 텍스트 변환한다.

```bash
ls -l /var/log/wtmp /var/log/btmp /var/log/lastlog /var/run/utmp
file /var/log/wtmp /var/log/btmp /var/log/lastlog
utmpdump /var/log/wtmp | tail -3
utmpdump /var/run/utmp
utmpdump /var/log/btmp
```

- `/var/run/utmp` : **현재** 로그인 세션 (`who`, `w` 참조) — 재부팅 시 초기화
- `/var/log/wtmp` : 로그인·로그아웃·재부팅 **누적 이력** (`last` 참조)
- `/var/log/btmp` : **실패한** 로그인 (`lastb` 참조), 권한 `-rw-------` root 전용
- `/var/log/lastlog` : 사용자별 마지막 로그인 (`lastlog` 참조) — UID 를 인덱스로 하는 희소(sparse) 파일이라 `ls -l` 크기가 실제보다 크게 보임
- `utmpdump` : utmp 형식 바이너리를 텍스트로 덤프 (`util-linux`). `-r` 로 역변환 가능
- 세 파일 모두 `logrotate` 대상 (`/etc/logrotate.conf` 에 `wtmp`·`btmp` 항목, Part 07)

**검증**

```bash
file -b /var/log/wtmp
du -h /var/log/lastlog; ls -lh /var/log/lastlog | awk '{print $5}'    # 실제 점유 vs 표시 크기
stat -c '%a %U %n' /var/log/btmp
```

```text
data
... /var/log/lastlog
...
600 root /var/log/btmp
```

> 📝 **시험 포인트**: "`wtmp` 는 텍스트라 `cat` 으로 열람" 은 틀린 지문 — 바이너리이며 `last`/`utmpdump` 로 읽음. `lastlog` 파일이 커 보이는 이유 = 희소 파일.

### 5-4. 접속자 메시지: wall · write · mesg

> **상황**: 점검 작업 전 접속자 전원에게 공지하고, 특정 세션에만 메시지를 보내는 절차를 확인한다. UTM 콘솔(tty1)과 SSH(pts/0) 두 세션을 열어 둔다.

```bash
# SSH 세션(pts/0) 에서
mesg                                   # 내 터미널의 메시지 수신 상태
mesg y                                 # 수신 허용
wall "LAB01: 10분 후 점검을 시작합니다"     # 전 세션 브로드캐스트 (root 는 mesg n 도 무시)
echo "점검 공지" | wall                 # 표준입력으로도 가능
write admin1 tty1                      # tty1 세션에 1:1 메시지 → 입력 후 Ctrl+D 로 종료
```

- `wall` : **w**rite **all** — 모든 로그인 터미널에 메시지. 인자 없으면 표준입력에서 읽음
- `write <사용자> [터미널]` : 특정 사용자의 터미널에 대화식 메시지. 같은 사용자가 여러 세션이면 터미널 지정 필요. `Ctrl+D` 종료
- `mesg [y|n]` : 자기 터미널로의 `write`/`wall` 수신 허용(**y**es)/차단(**n**o). 인자 없으면 현재 상태(`is y` / `is n`)
- `write` 는 `util-linux-user` 패키지 → 없으면 `dnf install -y util-linux-user`

**검증** (콘솔 tty1 화면에 메시지가 나타남)

```bash
mesg           # SSH 세션
# tty1 세션에서: mesg n → 다시 SSH 에서 write admin1 tty1 시도
```

```text
is y
write: admin1 has messages disabled on tty1
```

> 📝 **시험 포인트**: `wall` = 전체, `write` = 개인, `mesg n` = 수신 거부. root 의 `wall` 은 `mesg n` 상태의 터미널에도 전달. 세션 중 `Ctrl+D` 는 EOF(입력 종료) 신호.

---

## 6. FHS 탐방

### 6-1. 최상위 디렉터리와 역할

> **상황**: 이후 모든 파트에서 설정 파일·로그·바이너리 위치를 찾아다녀야 한다. 루트 계층을 한 번 훑고 RHEL 9 의 `/bin → /usr/bin` 통합(UsrMerge)을 확인한다.

```bash
ls /
ls -l / | grep '^l'          # 심볼릭 링크인 최상위 항목
ls -ld /bin /sbin /lib /lib64
ls /usr
ls /var
ls /etc | head
du -sh /usr /var /etc /boot 2>/dev/null
```

- `ls -ld` : 디렉터리 자체(**d**irectory) 정보 — 링크 대상 확인
- `du -sh` : 디스크 사용량 **s**ummary, **h**uman-readable

| 디렉터리 | 역할 | 이 절차서에서 만나는 예 |
| --- | --- | --- |
| `/bin` → `/usr/bin` | 필수 사용자 명령 (`ls` `cp` `bash`) | 전 파트 |
| `/sbin` → `/usr/sbin` | 시스템 관리 명령 (`fdisk` `useradd` `sshd`) | 03·05·08 |
| `/lib`, `/lib64` → `/usr/lib*` | 공유 라이브러리·커널 모듈 (`/lib/modules/<커널>`) | 02·05 |
| `/usr` | 2차 계층 — 프로그램·라이브러리·문서 (`/usr/share/man`) | 02·07 |
| `/usr/local` | 관리자가 직접 설치한 소프트웨어 (`/usr/local/bin/backup.sh`) | 02·04·06 |
| `/etc` | 전역 텍스트 설정 (`passwd` `fstab` `ssh/sshd_config`) | 전 파트 |
| `/var` | 가변 데이터 — `log` `spool` `lib` `cache` `tmp` `ftp` `www` | 07·09 |
| `/home` | 일반 사용자 홈 (`/home/admin1`) | 03 |
| `/root` | root 홈 | 01 |
| `/tmp` | 임시 파일 (Sticky Bit `t`, 재부팅 시 정리 — `systemd-tmpfiles`) | 03·04 |
| `/boot` | 커널·initramfs·GRUB·ESP(`/boot/efi`) | 04·07 |
| `/dev` | 장치 파일 (devtmpfs, udev 관리) | 05 |
| `/proc` | 프로세스·커널 정보 가상 FS | 06 |
| `/sys` | 장치·드라이버 정보 가상 FS (sysfs) | 05 |
| `/run` | 부팅 후 런타임 데이터 (tmpfs, `/var/run` → `/run`) | 06·07 |
| `/opt` | 서드파티 패키지 (`/opt/<벤더>`) | 02·11 |
| `/mnt` | 임시 수동 마운트 (`/mnt/nfs` `/mnt/smb`) | 09 |
| `/media` | 이동식 매체 자동 마운트 | — |
| `/srv` | 서비스 제공 데이터 (`/srv/www` `/srv/share` `/srv/raid`) | 05·09·12 |

**검증**

```bash
ls -l /bin /sbin /lib /lib64 | awk '{print $NF, $(NF-2), $(NF-1)}' | grep -- '->' || ls -ld /bin
readlink /bin; readlink /var/run
ls -ld /tmp | cut -c1-10
```

```text
lrwxrwxrwx. 1 root root 7 ... /bin -> usr/bin
lrwxrwxrwx. 1 root root 8 ... /sbin -> usr/sbin
lrwxrwxrwx. 1 root root 7 ... /lib -> usr/lib
lrwxrwxrwx. 1 root root 9 ... /lib64 -> usr/lib64
usr/bin
../run
drwxrwxrwt.
```

> 📝 **시험 포인트**: FHS 용도 문제에서 "`/proc`·`/sys` 는 디스크에 저장" · "`/tmp` 는 재부팅 후에도 유지" · "`/usr` 은 사용자 홈" 이 틀린 지문으로 등장. `/tmp` 권한 `1777`(Sticky Bit).

### 6-2. 가상 파일시스템 /proc /sys /dev

> **상황**: `/proc`·`/sys`·`/dev` 가 디스크 파티션이 아니라 커널이 메모리에서 만들어 주는 파일시스템임을 마운트 정보로 확인한다.

```bash
mount | grep -E 'proc|sysfs|devtmpfs'
findmnt -t proc,sysfs,devtmpfs,tmpfs
cat /proc/mounts | head -5
cat /proc/filesystems | grep nodev | head
ls /proc | head -20              # 숫자 디렉터리 = PID
ls /proc/$$                      # 현재 셸 프로세스
cat /proc/$$/status | head -5
cat /proc/version
cat /proc/interrupts | head -5
cat /proc/ioports 2>/dev/null | head -3     # aarch64 는 비어 있을 수 있음
ls /sys/class/net/ /sys/block/
cat /sys/class/net/enp0s1/address
df -h /proc /sys /dev
```

- `mount` : 현재 마운트 목록 — `proc on /proc type proc`, `sysfs on /sys type sysfs`, `devtmpfs on /dev type devtmpfs`
- `findmnt -t <유형,…>` : 마운트를 **t**ype 필터로 트리 출력
- `/proc/mounts` : 커널이 직접 제공하는 마운트 목록 (`mount` 의 원본, `/etc/mtab` → `/proc/self/mounts` 심볼릭 링크)
- `/proc/filesystems` : 커널이 지원하는 FS. `nodev` = 블록 장치 불필요(가상)
- `$$` : 현재 셸의 PID 변수 → `/proc/<PID>/status`·`cmdline`·`fd/`
- `/proc/version` : 커널 버전 문자열, `/proc/interrupts` : IRQ 통계, `/proc/ioports` : I/O 포트(x86 중심)
- `/sys/class/net/<if>/address` : sysfs 로 읽는 MAC 주소

**검증**

```bash
findmnt -no FSTYPE /proc /sys /dev
df -h /proc | tail -1
ls -ld /proc/1 && cat /proc/1/comm
```

```text
proc
sysfs
devtmpfs
proc    0  0  0  - /proc
dr-xr-xr-x. 9 root root 0 ... /proc/1
systemd
```

> 📝 **시험 포인트**: "`/proc` 은 디스크 공간을 차지하지 않는 메모리 기반 가상 FS", "현재 마운트 목록을 커널이 직접 제공하는 파일 = `/proc/mounts`", "PID 1 = systemd" 가 출제 지문.

### 6-3. 장치 파일 유형 (블록 · 문자)

> **상황**: Part 05 에서 `/dev/vdb` 에 파티션을 만들기 전에, 장치 파일의 `b`/`c` 유형과 major·minor 번호의 의미를 확인한다.

```bash
ls -l /dev/vda /dev/vda1 /dev/vdb
ls -l /dev/null /dev/zero /dev/tty /dev/tty1 /dev/pts/0 /dev/random /dev/urandom
ls -l /dev/disk/by-uuid/ | head -5
ls -l /dev/mapper/
ls -l /dev/stdin /dev/stdout /dev/stderr
file /dev/vda /dev/null
stat -c '%F  major=%t minor=%T  %n' /dev/vda /dev/null /dev/zero /dev/tty
```

- `ls -l` 첫 문자 : `b` 블록 장치, `c` 문자 장치, `l` 심볼릭 링크, `-` 일반, `d` 디렉터리, `s` 소켓, `p` 파이프
- 장치 파일은 크기 자리에 `major, minor` 표시 — major = 드라이버(virtio-blk, mem, tty), minor = 개별 장치/파티션
- `/dev/null`(1,3) : 쓰면 버림·읽으면 EOF, `/dev/zero`(1,5) : 읽으면 0x00 무한, `/dev/tty`(5,0) : 현재 제어 터미널
- `/dev/disk/by-uuid/` : UUID → 장치 심볼릭 링크 (udev 생성), `/dev/mapper/` : device-mapper(LVM) 장치
- `stat -c` : 사용자 정의 출력(**c**ustom format). `%F` 파일 유형, `%t`/`%T` 16진 major/minor, `%n` 이름

**검증**

```bash
ls -l /dev/vda /dev/null | cut -c1-1
echo test > /dev/null; echo "exit=$?"
head -c 16 /dev/zero | od -An -tx1
ls -l /dev/vd? | awk '{print $1, $5, $6, $NF}'
```

```text
b
c
exit=0
 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
brw-rw----. 252, 0 /dev/vda
brw-rw----. 252, 16 /dev/vdb
brw-rw----. 252, 32 /dev/vdc
brw-rw----. 252, 48 /dev/vdd
brw-rw----. 252, 64 /dev/vde
```

> 📝 **시험 포인트**: `ls -l` 출력에서 첫 글자 `b`/`c` 판별, "블록 장치는 버퍼 통해 블록 단위, 문자 장치는 버퍼 없이 문자 단위". `/dev/null` 로 출력 버리기(`2>/dev/null`), `/dev/zero` 로 `dd` 더미 파일 생성(Part 05).

### 6-4. file · stat · inode

> **상황**: 파일 이름이 아닌 inode 에 메타데이터가 저장됨을 직접 본다. 실습 파일은 `/tmp/lab01` 에 만든다.

```bash
mkdir -p /tmp/lab01 && cd /tmp/lab01
echo "hello lab01" > note.txt
cp /usr/bin/ls ./ls.bin
mkdir subdir
file note.txt ls.bin subdir /etc/passwd /bin /dev/vda
file -b note.txt                 # brief
file -i note.txt                 # MIME
stat note.txt
stat -c '%i %h %U:%G %a %s %n' note.txt   # inode 링크수 소유 권한 크기 이름
ls -i note.txt
ls -li
ls -id /tmp/lab01                # 디렉터리 inode
df -i /tmp
```

- `file` : 매직 넘버(내용) 기반 유형 판별 — 확장자 무관. `-b` : 파일명 생략(**b**rief), `-i` : M**I**ME 형식
- `stat` : inode 메타데이터 전체 — Size·Blocks·IO Block·Device·Inode·Links·Access(권한)·Uid·Gid·Access/Modify/Change/Birth 시각
- `stat -c` 포맷 : `%i` inode 번호, `%h` 하드링크 수, `%U`/`%G` 소유자/그룹, `%a` 8진 권한, `%s` 크기, `%n` 이름
- `ls -i` : 각 항목의 **i**node 번호, `-li` : long + inode
- inode 저장 정보 = 유형·권한·소유자·크기·타임스탬프 3종·링크 수·데이터 블록 포인터. **파일 이름은 없음**(디렉터리 엔트리가 보유)

**검증**

```bash
stat -c '%i' note.txt; ls -i note.txt | awk '{print $1}'      # 두 값 동일
stat -c 'atime=%x' note.txt; cat note.txt >/dev/null; stat -c 'atime=%x' note.txt
touch note.txt; stat -c 'mtime=%y ctime=%z' note.txt
chmod 600 note.txt; stat -c 'mtime=%y ctime=%z' note.txt        # ctime 만 변경
```

```text
<inode>
<inode>
atime=...
atime=...            # relatime 정책상 즉시 변하지 않을 수 있음
mtime=... ctime=...
mtime=... ctime=...  # mtime 동일, ctime 갱신
```

> 📝 **시험 포인트**: "inode 에 저장되지 않는 것 = 파일 이름" 이 대표 문항. `atime`(접근) `mtime`(내용 수정) `ctime`(inode 변경 — 권한·소유자 변경 시) 구분. `df -i` 로 inode 사용률.

### 6-5. 하드링크 vs 심볼릭 링크

> **상황**: 같은 파일에 하드링크·심볼릭 링크를 만들고 inode·링크 수·원본 삭제 후 동작을 비교한다.

```bash
cd /tmp/lab01
ln note.txt note.hard              # 하드링크
ln -s note.txt note.soft           # 심볼릭 링크
ln -s /etc/hostname host.soft      # 절대 경로 대상 심볼릭 링크
ls -li note.txt note.hard note.soft host.soft
stat -c '%i %h %F %n' note.txt note.hard note.soft
readlink note.soft
readlink -f note.soft              # 최종 실제 경로
ln subdir subdir.hard 2>&1         # 디렉터리 하드링크 → 오류
ln -s subdir subdir.soft           # 디렉터리 심볼릭 링크 → 가능
ln note.txt /boot/note.hard 2>&1   # 다른 파일시스템 → 오류 (Invalid cross-device link)
ln -s /tmp/lab01/note.txt /boot/note.soft && rm -f /boot/note.soft   # 심볼릭 링크는 FS 경계 넘음

# 원본 삭제 후 비교
rm note.txt
cat note.hard                       # 데이터 유지
cat note.soft                       # No such file or directory (dangling)
ls -li note.hard note.soft
```

- `ln <원본> <링크>` : 하드링크 — 같은 inode 를 가리키는 또 하나의 이름. 링크 수(`%h`) +1
- `ln -s <원본> <링크>` : **s**ymbolic — 경로 문자열을 담은 새 inode(`l` 유형). 크기 = 경로 길이
- `readlink` : 심볼릭 링크가 가리키는 경로, `-f` : 심볼릭 링크를 끝까지 따라간 정규화 경로(**f**ollow)
- 하드링크 제약 : 디렉터리 불가, 파일시스템 경계 불가 / 심볼릭 링크 : 둘 다 가능, 원본 삭제 시 깨짐(dangling)

| 구분 | 하드링크 | 심볼릭 링크 |
| --- | --- | --- |
| inode | 원본과 동일 | 새 inode |
| `ls -l` 유형 | `-` (일반 파일) | `l`, `링크 -> 대상` 표시 |
| 링크 수 | 원본 +1 | 원본 불변 (자기 자신 1) |
| 원본 삭제 | 데이터 유지 | 깨짐 |
| 디렉터리 | 불가 | 가능 |
| FS 경계 | 불가 | 가능 |

**검증**

```bash
cd /tmp/lab01
echo again > note.txt; ln note.txt n2; ln -s note.txt s2
[ "$(stat -c %i note.txt)" = "$(stat -c %i n2)" ] && echo "hard: same inode"
[ "$(stat -c %i note.txt)" != "$(stat -c %i s2)" ] && echo "soft: different inode"
stat -c '%h' note.txt              # 2
ls -l s2 | cut -c1; find . -xtype l   # 깨진 링크 검색 → note.soft
```

```text
hard: same inode
soft: different inode
2
l
./note.soft
```

> 📝 **시험 포인트**: "심볼릭 링크는 원본과 동일 inode 공유" 는 틀린 지문(하드링크 특성). `ls -l` 링크 수 열이 2 이상 = 하드링크 존재. 디렉터리 링크 수 = 2 + 하위 디렉터리 수 (`.` 과 각 하위의 `..` 때문).

### 6-6. /etc/fstab 읽기 (6필드 예고)

> **상황**: Part 05 에서 새 파일시스템을 등록할 파일이다. 설치 프로그램이 만든 기본 항목으로 6개 필드 구조를 먼저 읽어 둔다.

```bash
cat /etc/fstab
grep -v '^#' /etc/fstab | column -t
findmnt --fstab                  # fstab 기준 트리
findmnt --verify                 # 문법·장치 존재 검사
blkid                            # UUID 와 fstab 대조
```

- 필드 1 : 장치 — `/dev/mapper/rlm-root`, `UUID=…`, `LABEL=…`
- 필드 2 : 마운트 포인트 (`/`, `/boot`, `/boot/efi`, 스왑은 `none`)
- 필드 3 : 파일시스템 유형 (`xfs`, `vfat`, `swap`, `ext4`, `nfs`)
- 필드 4 : 마운트 옵션 (`defaults` = rw,suid,dev,exec,auto,nouser,async / `noauto`, `ro`, `nosuid`, `usrquota` 등)
- 필드 5 : `dump` 백업 대상 여부 (0 = 제외, 1 = 포함)
- 필드 6 : `fsck` 순서 (0 = 검사 안 함, 1 = 루트, 2 = 그 외) — xfs 는 관례상 0
- `findmnt --verify` : fstab 항목 문법·UUID 실존 여부 검증, `column -t` : 표 정렬

**검증**

```bash
awk '!/^#/ && NF==6 {print $2, $3, $5, $6}' /etc/fstab
findmnt --verify 2>&1 | tail -1
```

```text
/ xfs 0 0
/boot xfs 0 0
/boot/efi vfat 0 2
none swap 0 0
Success, no errors or warnings detected
```

> 📝 **시험 포인트**: fstab 6필드 순서·의미는 필기·실기 모두 매 회차. 특히 5번(dump 0/1)·6번(fsck 0/1/2) 혼동, `noauto`(부팅 시 자동 마운트 제외), `nosuid`(SUID 무시) 옵션.

### 6-7. /etc/passwd 형식 미리보기

> **상황**: Part 03 에서 사용자를 만들기 전에 `passwd` 7필드 구조를 admin1 항목으로 읽고, 시험 빈출인 `awk -F:` 필드 추출을 한 번 해 본다.

```bash
head -3 /etc/passwd
grep admin1 /etc/passwd
getent passwd admin1
awk -F: '$3 >= 1000 {print $1}' /etc/passwd          # UID 1000 이상 사용자명
awk -F: '{print $1, $7}' /etc/passwd | sort -k2 | uniq -c -f1 | head
cut -d: -f1,3,7 /etc/passwd | head -5
ls -l /etc/passwd /etc/shadow /etc/group
```

- 7필드 : `이름:비밀번호(x):UID:GID:설명(GECOS):홈:로그인셸`
- `x` = 실제 해시는 `/etc/shadow`(권한 000, root 만) 에 분리 저장
- `awk -F:` : 필드 구분자(**F**ield separator) `:` 지정, `$3` 세 번째 필드 조건
- `cut -d: -f1,3,7` : **d**elimiter `:`, **f**ield 1·3·7 추출
- 시스템 계정 UID < 1000, 일반 사용자 UID ≥ 1000 (`/etc/login.defs` 의 `UID_MIN`)
- 로그인 차단 셸 `/sbin/nologin`, `/bin/false` — Part 03 에서 실습

**검증**

```bash
awk -F: '$1=="admin1"{print "UID="$3, "GID="$4, "HOME="$6, "SHELL="$7}' /etc/passwd
awk -F: '$3>=1000 && $3<65534' /etc/passwd | wc -l
```

```text
UID=1000 GID=1000 HOME=/home/admin1 SHELL=/bin/bash
1
```

> 📝 **시험 포인트**: "4번째 필드 1001 = GID", "7번째 필드 = 로그인 셸". 실기에 `awk -F: '$3>=1000{print $1}' /etc/passwd` 가 반복 출제.

---

## 7. 셸 환경

> 이 절은 **admin1 일반 사용자 셸(`$`)** 에서 진행 (root 셸이면 `exit`).

### 7-1. 현재 셸과 사용 가능한 셸

> **상황**: admin1 의 로그인 셸이 bash 인지, 시스템에 어떤 셸이 등록돼 있는지 확인한다. 셸 변경 명령 `chsh` 의 사용법도 확인한다.

```bash
echo $SHELL                  # 로그인 셸 (passwd 7필드)
echo $0                      # 현재 실행 중인 셸 이름
ps -p $$ -o comm=            # 현재 셸 프로세스명
cat /etc/shells
rpm -q util-linux-user || sudo dnf install -y util-linux-user   # chsh 제공 패키지
chsh -l                      # /etc/shells 목록
chsh -s /bin/bash            # 자기 로그인 셸 변경 (그대로 bash 지정 → 변화 없음)
grep admin1 /etc/passwd | cut -d: -f7
which bash sh
ls -l /bin/sh                # sh → bash 심볼릭 링크
```

- `$SHELL` : 로그인 셸 경로 환경변수 (현재 셸이 아니라 **passwd 에 등록된** 셸)
- `$0` : 현재 셸/스크립트 이름 — 로그인 셸이면 `-bash` 처럼 `-` 접두
- `chsh -l` : `/etc/shells` 나열(**l**ist), `-s <셸>` : 로그인 셸 변경(**s**hell). 일반 사용자는 `/etc/shells` 에 있는 셸로만, 자기 자신만 변경 가능
- `/etc/shells` : 로그인 셸로 허용된 셸 목록 — 여기 없는 셸은 `chsh` 거부, FTP 등도 참조

**검증**

```bash
getent passwd admin1 | cut -d: -f7
grep -c . /etc/shells
readlink -f /bin/sh
```

```text
/bin/bash
...
/usr/bin/bash
```

> 📝 **시험 포인트**: "로그인 셸 목록 파일 = `/etc/shells`", "`chsh -s`", "`chsh` 는 root 가 아니면 타인 셸 변경 불가·`/etc/shells` 외 셸 불가". 셸 종류(sh·bash·csh·tcsh·ksh·zsh) 계열 구분 — bash·ksh·zsh 는 Bourne 계열, csh·tcsh 는 C 계열.

### 7-2. 환경변수와 셸 변수

> **상황**: 셸 변수와 환경변수(자식 상속) 차이를 서브셸로 직접 확인하고, 주요 변수 값을 읽는다.

```bash
env | head                      # 환경변수 목록
printenv HOME                   # 특정 변수
printenv | wc -l; set | wc -l   # set 은 셸 변수·함수까지 포함 → 더 많음
echo $PATH
echo $HOME $USER $LOGNAME $PWD
echo "$PS1"
echo $HISTSIZE $HISTFILESIZE $HISTFILE
echo $LANG $LC_ALL
echo $TERM $SHLVL $MAIL $HOSTNAME

# 셸 변수 vs 환경변수
MYVAR=hello                     # 셸 변수 (현재 셸만)
bash -c 'echo "child sees: [$MYVAR]"'
export MYVAR                    # 환경변수로 승격
bash -c 'echo "child sees: [$MYVAR]"'
export MYVAR2=world             # 선언+export 한 번에
unset MYVAR MYVAR2
echo "[$MYVAR][$MYVAR2]"

# PATH 확장 (세션 한정)
export PATH=$PATH:$HOME/bin
echo $PATH | tr ':' '\n'
```

- `env` / `printenv` : 환경변수(export 된 것)만. `printenv <이름>` : 한 변수 값
- `set` : 셸 변수 + 환경변수 + 함수 전체 (인자 없을 때)
- `export VAR[=값]` : 자식 프로세스로 상속되는 환경변수로 등록, `unset VAR` : 제거
- `bash -c '<명령>'` : 자식 셸에서 명령 실행 — 상속 여부 확인용
- 주요 변수 : `PATH` 명령 탐색 경로(`:` 구분) · `HOME` 홈 · `PS1` 1차 프롬프트(기본 `[\u@\h \W]\$`) · `PS2` 연속 입력 프롬프트(`>`) · `HISTSIZE` 메모리 이력 수 · `HISTFILESIZE` 파일 이력 수 · `LANG` 로케일 · `SHLVL` 셸 중첩 깊이 · `TERM` 터미널 유형

**검증**

```bash
MYVAR=x; bash -c 'echo "[${MYVAR:-unset}]"'; export MYVAR; bash -c 'echo "[${MYVAR:-unset}]"'; unset MYVAR
echo $PATH | tr ':' '\n' | grep -c .
[ "$(printenv HISTSIZE)" = "1000" ] && echo "HISTSIZE default OK"
```

```text
[unset]
[x]
...
HISTSIZE default OK
```

> 📝 **시험 포인트**: "변수를 자식 프로세스에 전달 = `export`", "`env` 는 환경변수만·`set` 은 셸 변수 포함". `PATH` 맨 앞에 `.` 을 두면 현재 디렉터리의 악성 `ls` 가 먼저 실행되는 보안 위험.

### 7-3. alias

> **상황**: 자주 쓰는 명령을 별칭으로 등록·해제해 본다. 영구 등록은 7-6 에서 `~/.bashrc` 로.

```bash
alias                            # 현재 alias 목록 (Rocky 기본: ls, ll, grep 색상 등)
alias ll='ls -l --color=auto'
alias la='ls -la'
alias rm='rm -i'
alias hg='history | grep'
ll /etc/host*
hg hostnamectl
type ll                          # alias 인지 확인
\ls -l /etc/hostname             # 백슬래시로 alias 우회
command ls -l /etc/hostname      # command 로 우회
unalias hg
unalias -a                       # 전부 해제 (재로그인 시 기본 alias 복원)
```

- `alias 이름='명령'` : 별칭 등록 (현재 셸 한정). 인자 없으면 목록
- `unalias 이름` : 해제, `-a` : **a**ll 해제
- `\명령` 또는 `command 명령` : alias 를 무시하고 원본 실행
- 기본 alias 출처 : `/etc/profile.d/colorls.sh`(ls 색상), `~/.bashrc`(`rm -i` 등은 root 의 `.bashrc`)

**검증**

```bash
alias ll='ls -l'; alias ll; type -t ll
unalias ll; type -t ll 2>&1 || echo "ll removed"
```

```text
alias ll='ls -l'
alias
ll removed
```

> 📝 **시험 포인트**: "재로그인 후에도 alias 유지" → `~/.bashrc` 에 등록. `alias` 만 치면 목록, `unalias -a` 전체 해제. `type` 으로 alias/내장/외부 구분.

### 7-4. history 와 이력 확장

> **상황**: 명령 이력을 조회·재실행하고, 이력에 시각을 남기도록 `HISTTIMEFORMAT` 을 설정한다.

```bash
history | tail -5
history 10                       # 최근 10개
!!                               # 직전 명령 재실행
!his                             # 'his' 로 시작하는 가장 최근 명령
echo /etc/hostname
cat !$                           # 직전 명령의 마지막 인자 → cat /etc/hostname
ls -l /etc/passwd /etc/group
echo !^                          # 직전 명령의 첫 인자
echo !*                          # 직전 명령의 모든 인자
history | tail -3                # 번호 확인 후
!<번호>                          # 해당 번호 재실행 (예: !102)
^hostname^hosts^                 # 직전 명령에서 문자열 치환 후 실행

export HISTTIMEFORMAT='%F %T '
history 3
history -c                       # 메모리 이력 삭제 (파일은 유지)
history -w                       # 메모리 이력을 ~/.bash_history 에 즉시 기록
history -r                       # 파일 이력을 메모리로 다시 읽기
history -d 5                     # 5번 항목만 삭제
cat ~/.bash_history | tail -3
echo $HISTCONTROL                # ignoredups → 연속 중복 미기록
```

- `history [N]` : 이력 출력(N 개). 번호는 `HISTSIZE` 내에서 증가
- `!!` 직전 명령 · `!N` N 번 · `!-N` N 개 전 · `!문자열` 해당 문자열로 시작하는 최근 명령 · `!$` 직전 마지막 인자 · `!^` 첫 인자 · `!*` 전체 인자
- `^old^new^` : 직전 명령의 `old` 를 `new` 로 바꿔 실행
- `HISTTIMEFORMAT` : `history` 출력에 시각 표시 형식 (`%F`=YYYY-MM-DD, `%T`=HH:MM:SS). 영구화는 `~/.bashrc`
- `history -c` **c**lear, `-w` **w**rite, `-r` **r**ead, `-d N` **d**elete N. 이력 파일 `~/.bash_history` 는 로그아웃 시 기록
- `HISTCONTROL=ignoredups`(연속 중복 무시) / `ignorespace`(공백으로 시작한 명령 무시) / `ignoreboth`

**검증**

```bash
echo lab01-marker; history 2 | head -1 | grep -c lab01-marker
history -w; grep -c lab01-marker ~/.bash_history
echo $HISTTIMEFORMAT
```

```text
1
1
%F %T
```

> 📝 **시험 포인트**: "`!102` 는 이력 102번 명령 재실행", "`!!` 직전 명령", "`HISTSIZE` 메모리 이력 수 vs `HISTFILESIZE` 파일 이력 수". `history -c` 는 삭제.

### 7-5. 셸 초기화 파일과 로그인/비로그인 셸

> **상황**: SSH 로그인 셸과 그 안에서 실행한 `bash` 서브셸이 읽는 파일이 다르다. 파일에 추적 문구를 넣고 어느 순서로 읽히는지 눈으로 확인한 뒤 되돌린다.

```bash
# 읽기 순서 관련 파일 확인
ls -l /etc/profile /etc/bashrc /etc/profile.d/
ls -la ~ | grep -E '\.bash|\.profile'
grep -n 'profile.d\|bashrc' /etc/profile ~/.bash_profile ~/.bashrc | head

# 로그인 셸 여부 확인
echo $0                          # -bash 면 로그인 셸
shopt login_shell                # on / off
bash                             # 비로그인 대화형 서브셸
echo $0; shopt login_shell; echo $SHLVL
exit
bash -l                          # 로그인 셸로 서브셸
echo $0; shopt login_shell
exit

# 추적 문구 삽입 (일시적)
echo 'echo ">> ~/.bash_profile read"' >> ~/.bash_profile
echo 'echo ">> ~/.bashrc read"' >> ~/.bashrc
bash        # → ~/.bashrc 만
exit
bash -l     # → ~/.bash_profile → (그 안에서) ~/.bashrc
exit
bash --noprofile --norc         # 아무 초기화 파일도 읽지 않음
exit

# 원복
sed -i '/>> ~\/.bash_profile read/d' ~/.bash_profile
sed -i '/>> ~\/.bashrc read/d' ~/.bashrc
```

- `shopt login_shell` : 현재 셸이 로그인 셸이면 `on` (읽기 전용 옵션)
- `bash -l` (`--login`) : 로그인 셸로 시작 → profile 계열 읽음, `bash` : 비로그인 대화형 → `~/.bashrc` 만
- `--noprofile` / `--norc` : profile / rc 파일 읽기 생략 (디버깅용)
- `$SHLVL` : 서브셸 진입마다 +1

| 파일 | 범위 | 읽는 시점 |
| --- | --- | --- |
| `/etc/profile` | 전역 | 로그인 셸 — 가장 먼저. `PATH`·`umask`·`HISTSIZE` 기본값, `/etc/profile.d/*.sh` 호출 |
| `/etc/profile.d/*.sh` | 전역 | `/etc/profile` 이 순서대로 source (`colorls.sh`, `lang.sh` 등) |
| `~/.bash_profile` | 개인 | 로그인 셸 — 사용자별. Rocky 기본 내용이 `~/.bashrc` 를 source 하고 `PATH=$PATH:$HOME/.local/bin:$HOME/bin` |
| `~/.bash_login`, `~/.profile` | 개인 | `~/.bash_profile` 이 없을 때만 순서대로 대체 |
| `~/.bashrc` | 개인 | 비로그인 대화형 셸의 진입점. `/etc/bashrc` 를 source, alias·함수 정의 위치 |
| `/etc/bashrc` | 전역 | `~/.bashrc` 가 source — `PS1` 기본값, 비로그인 셸용 전역 설정 |
| `~/.bash_logout` | 개인 | 로그인 셸 종료 시 (`clear` 등) |

```text
로그인 셸  : /etc/profile → /etc/profile.d/*.sh → ~/.bash_profile → ~/.bashrc → /etc/bashrc
비로그인 셸: ~/.bashrc → /etc/bashrc
로그아웃   : ~/.bash_logout
```

**검증**

```bash
echo $0; shopt -q login_shell && echo "login shell"
bash -c 'shopt -q login_shell || echo "child: non-login"'
grep -q '^. ~/.bashrc\|\. ~/.bashrc' ~/.bash_profile && echo ".bash_profile sources .bashrc"
grep -q '/etc/bashrc' ~/.bashrc && echo ".bashrc sources /etc/bashrc"
```

```text
-bash
login shell
child: non-login
.bash_profile sources .bashrc
.bashrc sources /etc/bashrc
```

> 📝 **시험 포인트**: "bash 로그인 셸이 가장 먼저 읽는 파일 = `/etc/profile`", 읽기 순서 나열 문제, "`~/.bashrc` 는 비로그인 셸도 읽음", "`~/.bash_logout` 은 로그아웃 시". `.bash_profile` 부재 시 `.bash_login` → `.profile` 대체 순서.

### 7-6. ~/.bashrc 에 alias·변수 영구 등록

> **상황**: 7-3·7-4 에서 만든 alias 와 `HISTTIMEFORMAT` 을 재로그인 후에도 유지되도록 `~/.bashrc` 에 등록하고 `source` 로 즉시 반영한다.

```bash
cp ~/.bashrc ~/.bashrc.bak
cat >> ~/.bashrc <<'EOF'

# LAB01 사용자 설정
alias ll='ls -l --color=auto'
alias la='ls -la --color=auto'
alias hg='history | grep'
export HISTTIMEFORMAT='%F %T '
export HISTSIZE=5000
export HISTFILESIZE=10000
EOF
source ~/.bashrc          # 또는  . ~/.bashrc
tail -8 ~/.bashrc
```

- `source <파일>` (= `. <파일>`) : 현재 셸에서 파일의 명령을 실행 → 변수·alias 가 현재 셸에 반영 (`bash 파일` 로 실행하면 자식 셸에만 적용되고 사라짐)
- alias 는 `~/.bashrc` 에, 전 사용자 공통은 `/etc/profile.d/<이름>.sh` 에 (Part 03 에서 `umask` 정책과 함께)
- `HISTSIZE`/`HISTFILESIZE` 상향 : 실습 이력 보존용

**검증**

```bash
exit          # SSH 재접속 후
alias ll hg; echo $HISTSIZE $HISTTIMEFORMAT
grep -c 'LAB01' ~/.bashrc
```

```text
alias ll='ls -l --color=auto'
alias hg='history | grep'
5000 %F %T
1
```

> 📝 **시험 포인트**: "alias 영구 적용 파일 = `~/.bashrc`", "`source`/`.` 는 현재 셸에서 실행". `bash script.sh`(자식 셸)·`./script.sh`(실행 권한 필요)·`source script.sh`(현재 셸) 의 차이 (Part 04).

### 7-7. 명령 위치 확인: type · which · whereis

> **상황**: 어떤 명령이 alias·셸 내장·외부 파일 중 무엇인지, 실행 파일과 매뉴얼이 어디 있는지 찾는다.

```bash
type ls cd echo ll bash            # 종류 판별
type -a echo                       # 모든 후보 (내장 + /usr/bin/echo)
type -t cd; type -t ls; type -t bash
which bash ls
which -a python3 2>&1
whereis bash
whereis -b passwd                  # 바이너리만
whereis -m passwd                  # 매뉴얼만
whereis -s bash                    # 소스 (없음)
command -v ls
hash                               # 셸이 캐시한 명령 경로
```

- `type` : 셸이 이름을 어떻게 해석하는지 — `alias` / `builtin`(내장) / `file`(외부) / `function` / `keyword`. `-t` : 종류 단어만, `-a` : **a**ll 후보
- `which` : `PATH` 에서 실행 파일 위치 검색 (내장 명령·alias 는 못 찾음. Rocky 의 `which` 는 alias 도 표시하도록 래핑됨). `-a` : PATH 상의 모든 일치
- `whereis` : 바이너리(**b**)·매뉴얼(**m**)·소스(**s**) 의 표준 위치를 검색 (PATH 가 아닌 고정 디렉터리 목록)
- `command -v` : POSIX 표준 방식으로 명령 경로/이름 출력 (스크립트에서 존재 확인용)
- `hash` : 이번 세션에서 실행한 외부 명령의 경로 캐시 (`hash -r` 로 초기화)

**검증**

```bash
type -t cd echo ls
which ls | grep -o '/usr/bin/ls'
whereis -b ls | awk '{print $2}'
```

```text
builtin
builtin
alias
/usr/bin/ls
/usr/bin/ls
```

> 📝 **시험 포인트**: `cd`·`echo`·`export`·`alias` 는 셸 내장(builtin) → `which cd` 는 못 찾음. `whereis` 는 바이너리+매뉴얼+소스, `which` 는 PATH 실행 파일만.

### 7-8. 매뉴얼: man · info · --help · mandb

> **상황**: 이후 모든 명령의 옵션은 매뉴얼로 확인한다. 미니멀 설치 직후엔 `man -k` 색인이 비어 있으므로 `mandb` 로 생성한다.

```bash
sudo dnf install -y man-pages info 2>/dev/null   # 없을 때만 (man-db 는 기본 포함)
sudo mandb                       # 매뉴얼 색인 DB 생성/갱신
man passwd                       # 섹션 1 (명령) — q 로 종료
man 5 passwd                     # 섹션 5 (파일 형식)
man -f passwd                    # = whatis: 섹션별 한 줄 설명
whatis passwd
man -k hostname                  # = apropos: 키워드로 매뉴얼 검색
apropos -s 1 hostname            # 섹션 1 한정
man -a passwd                    # 모든 섹션 차례로
man -w ls                        # 매뉴얼 파일 경로
man man | head -30               # 섹션 번호 표
info coreutils 'ls invocation'   # info 문서 (q 종료)
info ls
ls --help | head -5
hostnamectl --help | head -3
help cd                          # 셸 내장 명령은 help
help | head -5
```

- `man [섹션] <이름>` : 섹션 생략 시 낮은 번호부터 첫 일치. 같은 이름이 여러 섹션에 있을 때 번호 지정 필수 (`passwd`(1) 명령 vs `passwd`(5) 파일)
- `man -f` = `whatis` : 이름 정확 일치의 한 줄 설명, `man -k` = `apropos` : 설명 문장 키워드 검색 (`mandb` 색인 필요)
- `man -a` : **a**ll 섹션 순회, `man -w` : **w**here(파일 경로), `-s N` (apropos) : 섹션 한정
- `mandb` : `/var/cache/man/` 의 whatis 색인 DB 생성 — `cron`/`systemd timer` 로 주기 갱신
- `info` : GNU 하이퍼텍스트 문서 (노드 이동 `n`/`p`/`u`, 링크 `Enter`)
- `<명령> --help` : 외부 명령의 간단 사용법, `help <내장>` : bash 내장 명령 설명

| 섹션 | 내용 | 예 |
| --- | --- | --- |
| 1 | 사용자 명령 | `man 1 ls`, `man 1 passwd` |
| 2 | 시스템 콜 | `man 2 open` |
| 3 | 라이브러리 함수 | `man 3 printf` |
| 4 | 장치 파일·드라이버 | `man 4 null`, `man 4 tty` |
| 5 | 파일 형식·설정 파일 | `man 5 passwd`, `man 5 fstab`, `man 5 crontab` |
| 6 | 게임 | — |
| 7 | 기타·규약·표준 | `man 7 hier`(FHS), `man 7 signal` |
| 8 | 시스템 관리 명령 | `man 8 useradd`, `man 8 mount` |
| 9 | 커널 루틴 | — |

**검증**

```bash
whatis passwd
man -w 5 passwd
apropos -s 5 fstab | head -2
man 7 hier | grep -m1 -A1 '/proc'
```

```text
passwd (1)           - update user's authentication tokens
passwd (5)           - password file
/usr/share/man/man5/passwd.5.gz
fstab (5)            - static information about the filesystem
...
```

> 📝 **시험 포인트**: "설정 파일 형식은 섹션 5, 관리 명령은 섹션 8, 시스템 콜은 2, 라이브러리는 3". `man -k` = `apropos`, `man -f` = `whatis`. `man 7 hier` 가 FHS 설명 페이지.

---

## 8. 스냅샷과 다음 파트

### 8-1. UTM 스냅샷 저장

> **상황**: 여기까지가 "깨끗한 설치 + 기본 점검 완료" 상태다. 이후 파트에서 시스템을 망가뜨려도 돌아올 수 있도록 스냅샷을 남긴다.

```bash
# VM 안에서 — 임시 파일 정리 후 정상 종료
rm -rf /tmp/lab01
sudo systemctl poweroff
```

- UTM (QEMU 백엔드) : VM 목록에서 **srv01** 선택 → 상단 툴바 또는 우클릭 → **Snapshots… → +** → 이름 `01-clean-inspected` → Save
- Apple Virtualization 백엔드는 스냅샷 미지원 → 대안: VM 종료 후 `~/Library/Containers/com.utmapp.UTM/Data/Documents/srv01.utm` 폴더를 Finder 에서 복제
- 이후 권장 스냅샷 지점: Part 05 진입 전(디스크 작업), Part 10 진입 전(방화벽·SELinux)

**검증**

- UTM → srv01 → Snapshots 목록에 `01-clean-inspected` 표시, 또는 복제 폴더 존재
- VM 재시작 후 `hostnamectl --static` = `srv01.lab.local`, `timedatectl | grep 'Time zone'` = `Asia/Seoul`, `alias ll` 유지 확인

```text
srv01.lab.local
                Time zone: Asia/Seoul (KST, +0900)
alias ll='ls -l --color=auto'
```

> 📝 **시험 포인트**: `systemctl poweroff` = 런레벨 0, `systemctl reboot` = 6. `shutdown -h now`, `shutdown -r +5 "msg"`, `halt`, `init 0` 은 Part 12 에서 정리.

### 8-2. 다음 파트 예고

- [[02-package-management]] : `dnf`/`rpm` 으로 이번 파트에서 임시 설치한 `pciutils` `usbutils` `dmidecode` `util-linux-user` `man-pages` `info` 를 정식으로 점검·관리, EPEL 등록, `net-tools`·`sysstat` 등 이후 파트 도구 일괄 설치, 소스 컴파일과 공유 라이브러리
- 이 파트에서 확인한 값 중 다음 파트가 사용하는 것: 커널 버전(`uname -r`, 커널 패키지 관리), 아키텍처 `aarch64`(패키지 선택), `/dev/vdb~vde`(Part 05), `192.168.64.10`(Part 08)

---

## 체크리스트

| 수행 항목 | 명령 | 확인 방법 | ☐ |
| --- | --- | --- | --- |
| ISO SHA-256 검증 | `shasum -a 256 -c CHECKSUM --ignore-missing` | `OK` 출력, exit 0 | ☐ |
| VM 생성 (2 vCPU/4 GB/40 GB + 5·5·5·2 GB) | UTM GUI | `lsblk -d` 에 vda~vde, `nproc`=2 | ☐ |
| 디스플레이 카드 `virtio-ramfb` 지정 | UTM 편집 → 디스플레이 | 부팅 시 UEFI·GRUB 화면 표시 | ☐ |
| 직렬 포트 추가 (내장 터미널) | UTM 편집 → 새로 만들기 → 직렬 포트 | VM 창에 직렬 탭 생성 | ☐ |
| Rocky 9 minimal 설치, admin1 wheel | Anaconda | `id admin1` 에 `10(wheel)` | ☐ |
| 설치 후 ISO 제거 | 툴바 CD 아이콘 → 꺼내기 | 재부팅 시 설치 메뉴 대신 Rocky GRUB | ☐ |
| 게스트 IP 확인 (호스트 경로) | `cat /var/db/dhcpd_leases`, `nc -z <ip> 22` | MAC 일치 항목의 IP, 22번 열림 | ☐ |
| SSH 접속·별칭 등록 | `ssh admin1@<ip>`, `~/.ssh/config` | `ssh srv01` 로 접속, `who am i` 에 pts/0 | ☐ |
| terminfo 설치 (TERM 오류 시) | `infocmp -x \| ssh srv01 -- tic -x -` | `clear`·`tput cols` 정상 동작 | ☐ |
| 직렬 콘솔 활성화 | `grubby --update-kernel=ALL --args="console=tty0 console=ttyAMA0,115200"` | `cat /proc/cmdline`, 직렬 탭에 로그인 프롬프트 | ☐ |
| su / su - / sudo -i 차이 | `echo $PATH; pwd` | `su -`·`sudo -i` 만 `/root`, PATH 에 sbin | ☐ |
| 호스트명 srv01.lab.local | `hostnamectl set-hostname` | `hostname -f`, `cat /etc/hostname` | ☐ |
| /etc/hosts 에 192.168.64.10 등록 | `cat >> /etc/hosts` | `getent hosts srv01` | ☐ |
| 커널·OS 식별 | `uname -a/-r/-m`, `cat /etc/os-release` | `el9`, `aarch64`, `ID="rocky"` | ☐ |
| CPU·메모리 | `lscpu`, `free -h`, `/proc/cpuinfo`, `/proc/meminfo` | 코어 2, MemTotal ≈ 3.7 GiB | ☐ |
| 디스크·FS | `lsblk -f`, `df -hT`, `df -i` | vda1 vfat·vda2 xfs·rlm-root xfs | ☐ |
| PCI/USB/DMI | `lspci`, `lsusb`, `dmidecode -t system` | virtio 장치 표시 | ☐ |
| 시간대 Asia/Seoul, NTP | `timedatectl set-timezone`, `set-ntp true` | `date +%Z` = KST, `/etc/localtime` 링크 | ☐ |
| 로케일·키맵 조회 | `localectl`, `list-locales` | `LANG=en_US.UTF-8` | ☐ |
| 부팅 시간 분석 | `systemd-analyze`, `blame`, `critical-chain` | Startup finished 줄 | ☐ |
| 커널 인자·/boot | `cat /proc/cmdline`, `ls /boot`, `rpm -q kernel`, `grubby --default-kernel` | `root=/dev/mapper/rlm-root`, 실행 커널 = 기본 커널 | ☐ |
| 런레벨·타겟 | `systemctl get-default`, `runlevel`, `who -r` | `multi-user.target`, `N 3` | ☐ |
| 하드웨어 인식·오류 로그 | `dmesg -T \| grep -i virtio`, `journalctl -b -p err` | virtio 줄 존재, err 0건 | ☐ |
| 세션 정보 | `who`, `w`, `id`, `tty`, `logname` | 세션 2개(tty1, pts/0) | ☐ |
| 로그인 이력·실패 | `last`, `lastb`, `lastlog` | btmp 에 실패 1건 | ☐ |
| 바이너리 로그 덤프 | `file /var/log/wtmp`, `utmpdump` | `data`, 텍스트 레코드 | ☐ |
| 메시지 | `wall`, `write admin1 tty1`, `mesg n` | tty1 에 표시 / 차단 메시지 | ☐ |
| FHS·UsrMerge | `ls -l /`, `readlink /bin` | `/bin -> usr/bin`, `/tmp` 권한 `t` | ☐ |
| 가상 FS | `mount \| grep -E 'proc\|sysfs\|devtmpfs'`, `findmnt -t proc` | proc/sysfs/devtmpfs, df 크기 0 | ☐ |
| 장치 파일 b/c | `ls -l /dev/vda /dev/null /dev/zero /dev/tty` | `b`, `c`, major/minor | ☐ |
| file·stat·inode | `file`, `stat -c '%i %h'`, `ls -i`, `df -i` | inode 번호 일치, ctime 변경 | ☐ |
| 하드·심볼릭 링크 | `ln`, `ln -s`, `ls -li`, `readlink -f`, `find -xtype l` | 동일 inode·링크수 2, dangling | ☐ |
| fstab 6필드 | `cat /etc/fstab`, `findmnt --verify` | `Success, no errors` | ☐ |
| passwd 7필드 | `awk -F: '$3>=1000{print $1}' /etc/passwd` | admin1 | ☐ |
| 셸·/etc/shells·chsh | `echo $SHELL`, `chsh -l` | `/bin/bash` | ☐ |
| 환경변수 vs 셸 변수 | `export`, `unset`, `env`, `set`, `bash -c` | 자식 상속 여부 [unset]/[x] | ☐ |
| alias·unalias | `alias ll=…`, `type ll`, `unalias -a` | `type -t ll` = alias | ☐ |
| history 확장 | `!!`, `!N`, `!$`, `^a^b^`, `HISTTIMEFORMAT` | 시각 포함 이력 | ☐ |
| 로그인/비로그인 셸 | `bash` vs `bash -l`, `shopt login_shell`, `echo $0` | on/off, `-bash`/`bash` | ☐ |
| ~/.bashrc 영구 등록 | `cat >> ~/.bashrc`, `source` | 재로그인 후 `alias ll` 유지 | ☐ |
| type·which·whereis | `type -a echo`, `which ls`, `whereis -m passwd` | builtin/file 구분 | ☐ |
| man 섹션·색인 | `mandb`, `man 5 passwd`, `man -k`, `whatis`, `info` | `passwd (5)` 설명 출력 | ☐ |
| 스냅샷 | UTM Snapshots | `01-clean-inspected` 존재 | ☐ |

---

## 기출 연결

| 기출 (문제집·회차·번호 또는 주제) | 이 파트의 단계 |
| --- | --- |
| EXAM-PRACTICAL r04-4 (SHA-256 해시 계산 명령), EXAM-WRITTEN-FULL r06-96 (충돌 발견 해시 대신 사용할 알고리즘) | 1-1 ISO 해시 검증 |
| EXAM-WRITTEN-FULL r02-46 (장치 파일명 — `/dev/vda` virtio) | 1-2 VM 생성, 6-3 장치 파일 |
| EXAM-WRITTEN-FULL r03-21, r07-60 (`su` 와 `sudo` 차이) | 2-4 su / su - / sudo -i |
| EXAM-WRITTEN-FULL r02-31, r07-33 (`grubby --update-kernel`·커널 파라미터 추가) | 2-3 직렬 콘솔 활성화 · Part 07 |
| EXAM-WRITTEN-FULL r03-1, r04-3 (커널 버전 표기) | 3-3 커널·OS |
| EXAM-WRITTEN-FULL r08-28 (`free -h` 해석), r05-1·r10-1 (커널 역할) | 3-4 CPU·메모리 |
| EXAM-WRITTEN-FULL r08-26 (`df -h` 해석), r08-47 (`lsblk` 해석), r04-34 (`df -i` inode), r04-55 (UUID·FS 유형 확인) | 3-5 디스크, 6-4 inode |
| EXAM-WRITTEN-FULL r06-52 (udev 장치 파일 자동 생성) | 3-6 하드웨어 조회, 6-3 |
| EXAM-WRITTEN-FULL r03-6 (`systemd-analyze blame`) | 4-1 부팅 시간 분석 |
| EXAM-WRITTEN-FULL r01-6, r02-5, r07-1 (부팅 순서), r01-8·r03-5·r04-5·r04-6·r05-5·r05-6·r07-16 (GRUB2 설정 파일·반영 명령) | 4-2 커널 인자·/boot (설정 변경은 Part 07) |
| EXAM-WRITTEN-FULL r01-7, r02-6, r03-18, r05-7, r07-15 (런레벨↔타겟), r04-8 (현재 런레벨 확인), r07-2 (타겟 도달 순서), r10-6·r10-7 (systemd vs SysV, inittab) | 4-3 런레벨과 타겟 |
| EXAM-WRITTEN-FULL r04-62, r08-45 (`journalctl` 옵션·해석), r07-38 (저널 영구 보존 — Part 07) | 4-4 부팅 오류 로그 |
| EXAM-WRITTEN-FULL r02-59 (로그인 기록 명령↔파일), r03-62 (로그인 기록 파일 성격), r08-56 (`last` 출력 해석) | 5-1 who, 5-2 last, 5-3 wtmp/btmp |
| EXAM-WRITTEN-FULL r04-56, r05-63, r06-60, r07-63, r09-60 (`/var/log/btmp` 확인 = `lastb`) | 5-2 lastb |
| EXAM-WRITTEN-FULL r01-9, r02-7, r05-9 (FHS 디렉터리 용도) | 6-1 최상위 디렉터리 |
| EXAM-WRITTEN-FULL r03-45, r05-8 (`/proc` 설명), r05-35 (`/proc/mounts`) | 6-2 가상 FS |
| EXAM-WRITTEN-FULL r04-9, r08-3 (`ls -l` 출력의 파일 유형 해석), r10-28 (디렉터리 링크 수) | 6-3 장치 파일, 6-5 링크 |
| EXAM-WRITTEN-FULL r01-10 (inode 저장 정보), r08-25 (`stat` 출력 해석) | 6-4 file·stat·inode |
| EXAM-WRITTEN-FULL r01-11, r03-7, r07-44 (하드링크 vs 심볼릭 링크) | 6-5 링크 비교 |
| EXAM-WRITTEN-FULL r01-33, r02-30, r03-47, r04-30, r05-33, r05-34, r06-30, r08-54, r09-52, r09-53, r10-29 · EXAM-PRACTICAL r03-13 (`/etc/fstab` 6필드·옵션) | 6-6 fstab 읽기 |
| EXAM-WRITTEN-FULL r01-22, r03-43, r05-21, r07-31 (`/etc/passwd` 필드) · EXAM-PRACTICAL r03-2, r06-5 (`awk -F:` UID≥1000 사용자명) | 6-7 passwd 형식 |
| EXAM-WRITTEN-FULL r01-12, r03-19, r07-17 (셸 종류·계열), r05-19·r08-18 (`/etc/shells`), r04-10 (`chsh`), r03-40 (로그인 셸 관리), r09-19 (제한 셸) | 7-1 셸 확인 |
| EXAM-WRITTEN-FULL r05-11, r07-36 (`export`), r08-6 (`env` vs `set`), r10-12 (bash 환경변수), r09-8 (PATH 의 `.` 위험), r01-13 (`HISTSIZE` 용도) | 7-2 환경변수, 7-4 history |
| EXAM-WRITTEN-FULL r06-10 (alias 영구 유지 — `~/.bashrc`) | 7-3 alias, 7-6 bashrc 등록 |
| EXAM-WRITTEN-FULL r08-7 (`!102` 이력 재실행), r01-13 (history 관련 명령) | 7-4 history 확장 |
| EXAM-WRITTEN-FULL r05-10 (로그인 셸이 가장 먼저 읽는 파일), r07-22 (초기화 파일 읽기 순서) | 7-5 초기화 파일·로그인 셸 |
| EXAM-WRITTEN-FULL r03-9, r07-6 (`exec`·fork-exec 실행 흐름 — 셸 내장/외부 구분) | 7-7 type·which·whereis |
| EXAM-WRITTEN-FULL r03-1 등 다수 문항의 `man 5` 참조 파일(`passwd`, `fstab`, `crontab`) | 7-8 man 섹션 |

---

## 이전 / 다음

[[README]] ← · → [[02-package-management]]

- [[README]] — LAB 허브(시나리오·자원 명세)
- 참조 이론: [[../THEORY/linux-basics]] (부팅·런레벨·라이선스), [[../THEORY/system-structure]] (FHS·inode·링크·셸·하드웨어)
