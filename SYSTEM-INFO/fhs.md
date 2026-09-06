---
command: fhs
category: SYSTEM-INFO
aliases: [filesystem-hierarchy-standard, root-directory, 디렉터리구조, hier]
tags:
  - linux/system
  - linux/disk
  - topic/filesystem
  - topic/troubleshooting
  - task/inspect
  - privilege/mixed
  - distro/rhel
  - distro/rocky
related: ["[[ls]]", "[[du]]", "[[df]]", "[[mount]]", "[[stat]]", "[[which]]", "[[lsblk]]", "[[rpm]]"]
distro: RHEL 계열 (Rocky, CentOS) — 공통 표준은 배포판 무관
verified: Rocky Linux 9.8 (Blue Onyx) aarch64, 커널 5.14.0-687.10.1.el9_8
updated: 2026-09-06
---

# FHS — 루트 디렉터리 구조

- 리눅스 루트(`/`) 하위 디렉터리의 용도를 규정한 표준
- 어원: **F**ilesystem **H**ierarchy **S**tandard
- 명령어가 아닌 **참조** 문서 (디렉터리 배치 규약)
- 배포판이 달라도 배치가 동일 → 스크립트·패키지 이식성의 근거
- 본 문서 전 출력은 Rocky Linux 9.8 실기 확인분

---

## 전체 목록

```bash
$ ls /
afs  boot  etc   lib    media  opt   root  sbin  sys  usr
bin  dev   home  lib64  mnt    proc  run   srv   tmp  var
```

```bash
$ ls -la /
dr-xr-xr-x.   2 root root    6 afs
lrwxrwxrwx.   1 root root    7 bin -> usr/bin          # 심볼릭 링크
dr-xr-xr-x.   6 root root 4096 boot
drwxr-xr-x.  20 root root 3380 dev
drwxr-xr-x.  81 root root 8192 etc
drwxr-xr-x.   3 root root   20 home
lrwxrwxrwx.   1 root root    7 lib -> usr/lib          # 심볼릭 링크
lrwxrwxrwx.   1 root root    9 lib64 -> usr/lib64      # 심볼릭 링크
drwxr-xr-x.   2 root root    6 media
drwxr-xr-x.   2 root root    6 mnt
drwxr-xr-x.   2 root root    6 opt
dr-xr-xr-x. 175 root root    0 proc                    # 크기 0 = 가상 파일시스템
dr-xr-x---.   3 root root  163 root                    # 550, 소유자만 접근
drwxr-xr-x.  28 root root  800 run
lrwxrwxrwx.   1 root root    8 sbin -> usr/sbin        # 심볼릭 링크
drwxr-xr-x.   2 root root    6 srv
dr-xr-xr-x.  12 root root    0 sys                     # 크기 0 = 가상 파일시스템
drwxrwxrwt.  11 root root 4096 tmp                     # 1777, 스티키 비트
drwxr-xr-x.  12 root root  144 usr
drwxr-xr-x.  19 root root 4096 var
```

### 성격별 분류

| 성격 | 디렉터리 | 판별 근거 |
| --- | --- | --- |
| **심볼릭 링크** | `bin` `sbin` `lib` `lib64` | `ls -l` 에서 `l` 로 시작 → 전부 `usr/` 하위를 가리킴 |
| **가상 파일시스템** | `proc` `sys` `dev` `run` | 크기 0 또는 tmpfs·devtmpfs. 디스크 미점유 |
| **별도 파일시스템** | `boot` `boot/efi` | `df` 출력에서 독립 항목 |
| **빈 디렉터리 (규약상 예약)** | `afs` `media` `mnt` `opt` `srv` | 최소 설치 상태에서 내용 없음 |
| **일반 디렉터리** | `etc` `home` `root` `tmp` `usr` `var` | 루트 파일시스템에 실제 존재 |

### 용량 실측

```bash
$ sudo du -sh /boot /etc /home /opt /root /srv /tmp /usr /var
317M  /boot
 23M  /etc
 20K  /home
   0  /opt
 32K  /root
   0  /srv
4.0K  /tmp
1.5G  /usr        # 시스템 용량의 대부분
140M  /var
```

- **`/usr` 가 전체의 압도적 비중** → 설치된 소프트웨어 본체가 모두 여기 위치
- `/boot` 317M — 커널·initramfs 다중 세대 보관 결과
- `/opt` `/srv` 0 — 최소 설치 상태

### 마운트 구조

```bash
$ df -hT
Filesystem           Type      Size  Used Avail Use% Mounted on
devtmpfs             devtmpfs  1.8G     0  1.8G   0% /dev
tmpfs                tmpfs     1.8G     0  1.8G   0% /dev/shm
tmpfs                tmpfs     716M  9.7M  707M   2% /run
efivarfs             efivarfs  256K   27K  230K  11% /sys/firmware/efi/efivars
/dev/mapper/rlm-root xfs        35G  2.0G   33G   6% /
/dev/vda2            xfs       960M  349M  612M  37% /boot
/dev/vda1            vfat      599M  7.8M  592M   2% /boot/efi
tmpfs                tmpfs     358M     0  358M   0% /run/user/1000
```

- 루트는 **LVM 논리 볼륨**(`/dev/mapper/rlm-root`) 위의 XFS
- `/boot` 는 LVM 밖의 일반 파티션 → 부트로더가 LVM 을 해석하지 못하는 경우 대비
- `/boot/efi` 는 **vfat** → UEFI 펌웨어가 읽을 수 있는 유일한 형식
- 상세 조회는 [[df]] · [[mount]] 참조

---

## /bin · /sbin · /lib · /lib64 — usr-merge 심볼릭 링크

```bash
$ for d in bin sbin lib lib64; do echo -n "/$d -> "; readlink /$d; done
/bin -> usr/bin
/sbin -> usr/sbin
/lib -> usr/lib
/lib64 -> usr/lib64
```

### 설명
- 사용 목적
	- 전통 FHS 상 `/bin`=일반 사용자 명령, `/sbin`=관리자 명령, `/lib`=공유 라이브러리
	- RHEL 7 이후 **usr-merge** 적용 → 실체는 전부 `/usr` 하위, 루트의 4개는 호환용 링크
- 특이사항
	- `#!/bin/bash` 셰뱅이 계속 동작하는 이유 → 링크가 유지되기 때문
	- 실제 경로 확인은 [[which]] 로 수행

```bash
$ command -v ls bash ip fdisk useradd systemctl
/usr/bin/ls          # 일반 사용자 명령
/usr/bin/bash
/usr/sbin/ip         # 관리자 명령
/usr/sbin/fdisk
/usr/sbin/useradd
/usr/bin/systemctl   # systemd 는 bin 에 위치
```

- **`/usr/bin` 747개 · `/usr/sbin` 390개** (실측) → 관리자 명령이 별도 분리됨
- `ip` `fdisk` `useradd` 가 `sbin` 인 것은 **root 전용 성격** 반영

---

## /boot — 커널과 부트로더

```bash
$ ls /boot
config-5.14.0-687.10.1.el9_8.0.1.aarch64        # 커널 빌드 설정
dtb  dtb-5.14.0-...                             # 디바이스 트리 (ARM 전용)
efi                                             # ESP 마운트 지점 (vfat)
grub2                                           # GRUB2 설정·환경
initramfs-5.14.0-....img                        # 초기 램디스크
initramfs-0-rescue-....img                      # 레스큐용
initramfs-5.14.0-...kdump.img                   # 크래시 덤프용
loader                                          # BLS 부팅 항목
symvers-5.14.0-....gz  System.map-5.14.0-...    # 커널 심볼 정보
vmlinuz-5.14.0-687.10.1.el9_8.0.1.aarch64       # 커널 이미지 본체
vmlinuz-0-rescue-...
```

### 설명
- 사용 목적
	- 커널 이미지(`vmlinuz`)·초기 램디스크(`initramfs`)·부트로더 파일 보관
	- 루트 파일시스템 마운트 **이전에** 읽히는 파일만 배치
- 특이사항
	- 커널 갱신마다 세대가 누적 → **용량 부족의 흔한 원인** (실측 960M 중 349M 사용)
	- `dtb` 는 aarch64 전용 → x86_64 에는 없음
	- `/boot/grub2` 권한 `700` → 일반 사용자 조회 불가, `sudo` 필요

```bash
$ sudo ls -la /boot/grub2
drwx------. 2 root root   25 fonts
-rw-------. 1 root root 6423 grub.cfg     # 자동 생성물. 직접 편집 금지
-rw-------. 1 root root 1024 grubenv      # 다음 부팅 기본값 등 변수

$ sudo ls /boot/loader/entries            # BLS(BootLoaderSpec) 항목
af7dc60b...-0-rescue.conf
af7dc60b...-5.14.0-687.10.1.el9_8.0.1.aarch64.conf
```

- 커널에 전달된 파라미터는 `/proc/cmdline` 로 확인

```bash
$ cat /proc/cmdline
BOOT_IMAGE=(hd1,gpt2)/vmlinuz-5.14.0-... root=/dev/mapper/rlm-root ro
crashkernel=1G-4G:256M,... rd.lvm.lv=rlm/root rd.lvm.lv=rlm/swap
console=tty0 console=ttyAMA0,115200
```

- 복구 절차는 [[rescue-mode]] · [[grub2-install]] 참조

---

## /etc — 시스템 전역 설정

```bash
$ ls /etc | wc -l
167
```

### 설명
- 사용 목적
	- 호스트 고유의 **텍스트 설정 파일** 보관
	- 실행 파일·라이브러리 미배치 (설정만)
- 특이사항
	- **백업 1순위 대상** → 이 디렉터리만으로 시스템 성격이 재현됨
	- 실측 23M — 용량 대비 중요도가 가장 높은 구간
	- 일부 항목은 `/usr` 로의 심볼릭 링크 (패키지 기본값 참조)

```bash
$ ls -l /etc/os-release /etc/localtime /etc/mtab
/etc/os-release -> ../usr/lib/os-release              # 패키지 제공 기본값
/etc/localtime  -> ../usr/share/zoneinfo/Asia/Seoul   # 시간대 선택 결과
/etc/mtab       -> ../proc/self/mounts                # 현재 마운트 상태(가상)
```

### 주요 항목 (실기 확인분)

| 항목 | 용도 |
| --- | --- |
| `passwd` `shadow` `group` `gshadow` | 계정·그룹 정보 |
| `login.defs` `default/useradd` `skel/` | 계정 생성 정책·기본값·홈 원본 파일 |
| `sudoers` `sudoers.d/` | 권한 상승 규칙 ([[sudo]]) |
| `fstab` `crypttab` | 부팅 시 마운트 대상 ([[mount]]) |
| `hosts` `resolv.conf` `nsswitch.conf` | 이름 해석 |
| `firewalld/` `selinux/` `audit/` | 보안 설정 ([[firewall-cmd]] · [[getenforce]]) |
| `systemd/system/` | 관리자 정의 유닛 ([[systemctl]]) |
| `crontab` `cron.d/` `cron.{hourly,daily,weekly,monthly}/` `anacrontab` | 스케줄 ([[crontab]]) |
| `dnf/` `yum.repos.d/` | 패키지 저장소 ([[dnf]]) |
| `profile` `bashrc` `profile.d/` | 셸 전역 환경 ([[env]]) |
| `exports` | NFS 공유 정의 |
| `chrony.conf` | 시각 동기화 |

- `/etc/skel` 은 신규 계정 홈의 **복사 원본**

```bash
$ ls -la /etc/skel
-rw-r--r--. .bash_logout
-rw-r--r--. .bash_profile
-rw-r--r--. .bashrc
```

- 신규 계정 홈에 동일 파일이 복사됨을 실기 확인

```bash
$ ls -la /home/admin1
-rw-r--r--. 1 admin1 admin1  18 .bash_logout      # /etc/skel 에서 복사
-rw-r--r--. 1 admin1 admin1 141 .bash_profile
-rw-r--r--. 1 admin1 admin1 492 .bashrc
```

---

## /home — 일반 사용자 홈

```bash
$ ls -la /home
drwxr-xr-x.  3 root   root    20 .
drwx------.  4 admin1 admin1 115 admin1     # 700 → 본인만 접근
```

### 설명
- 사용 목적
	- 일반 사용자 계정의 개인 파일·개인 설정 보관
- 특이사항
	- **`root` 의 홈은 여기가 아님** → `/root` 로 분리
	- 계정 디렉터리 권한 기본 `700` → 사용자 간 상호 열람 차단
	- 운영 서버에서는 별도 파티션 분리 권장 (사용자 데이터가 루트 용량을 잠식하지 않도록)

---

## /root — root 전용 홈

```bash
$ stat -c '%n %A %a %U:%G' /root
/root dr-xr-x--- 550 root:root

$ sudo ls -la /root
-rw-------. anaconda-ks.cfg     # 설치 시 사용된 킥스타트 설정 (설치 이력)
-rw-------. .bash_history
-rw-r--r--. .bashrc .bash_profile .bash_logout .cshrc .tcshrc
drwx------. .ssh
```

### 설명
- 사용 목적
	- root 계정의 홈 디렉터리
- 특이사항
	- **`/home` 이 마운트되지 않아도 root 로그인이 가능해야 하므로 루트 파일시스템에 배치**
	- 권한 `550` → 일반 사용자 조회 불가 (`sudo` 필요)
	- `anaconda-ks.cfg` = 설치 당시 선택 사항 기록 → 동일 구성 재현에 활용 가능

---

## /dev — 장치 파일

```bash
$ ls /dev | wc -l
167

$ ls -l /dev/null /dev/zero /dev/urandom /dev/tty /dev/vda /dev/vda1
crw-rw-rw-. 1 root root   1, 3 /dev/null      # c = 문자 장치
crw-rw-rw-. 1 root root   1, 5 /dev/zero
crw-rw-rw-. 1 root root   1, 9 /dev/urandom
crw-rw-rw-. 1 root tty    5, 0 /dev/tty
brw-rw----. 1 root disk 252, 0 /dev/vda       # b = 블록 장치
brw-rw----. 1 root disk 252, 1 /dev/vda1
```

### 설명
- 사용 목적
	- 하드웨어·의사(pseudo) 장치를 **파일 형태로 노출** → 모든 장치를 파일 입출력으로 다룸
- 특이사항
	- **devtmpfs 로, 부팅 시 커널이 생성** → 디스크에 존재하지 않음 (실측 1.8G tmpfs)
	- 파일 유형 첫 글자: `b`=블록 장치(디스크), `c`=문자 장치(터미널·의사장치)
	- 크기 자리에 **주·부 장치 번호**(`252, 1`) 표시 → 일반 파일과 구분되는 특징
	- `/dev/vda` = virtio 디스크. 물리 환경은 `/dev/sda`, NVMe 는 `/dev/nvme0n1`

| 장치 | 용도 |
| --- | --- |
| `/dev/null` | 출력 폐기 (`2>/dev/null`) |
| `/dev/zero` | 0 바이트 무한 공급 (스왑 파일 생성 등) |
| `/dev/urandom` | 난수 공급 |
| `/dev/tty` | 현재 제어 터미널 |
| `/dev/vda`, `/dev/vda1` | 디스크 전체 / 파티션 |
| `/dev/mapper/*` | LVM·암호화 매핑 장치 |
| `/dev/shm` | 공유 메모리 (tmpfs, `1777`) |

```bash
$ ls -l /dev/mapper
crw-------. control
lrwxrwxrwx. rlm-root -> ../dm-0      # LVM 논리 볼륨 → dm 장치 링크
lrwxrwxrwx. rlm-swap -> ../dm-1
```

- 장치명 확인은 [[lsblk]] 선행 필수 — 특히 [[dd]] 등 파괴적 명령 전

---

## /proc — 프로세스·커널 정보 (가상)

```bash
$ ls /proc | head
1  10  11  12582  12586  ...        # 숫자 = PID 별 디렉터리
$ cat /proc/1/comm
systemd                              # PID 1 확인
```

### 설명
- 사용 목적
	- 실행 중인 **프로세스 정보**와 **커널 상태**를 파일 인터페이스로 제공
- 특이사항
	- **procfs 가상 파일시스템** → 디스크 미점유, 크기 0 표시
	- 숫자 디렉터리 = PID. `/proc/self` 는 현재 프로세스를 가리킴
	- 재부팅 시 전량 소멸 → 백업 대상 아님

| 경로 | 내용 |
| --- | --- |
| `/proc/cpuinfo` | CPU 모델·코어·기능 플래그 |
| `/proc/meminfo` | 메모리 상세 |
| `/proc/cmdline` | 커널 부팅 파라미터 |
| `/proc/mounts` | 현재 마운트 목록 (`/etc/mtab` 의 실체) |
| `/proc/modules` | 적재된 커널 모듈 |
| `/proc/uptime` | 가동 시간 (초) |
| `/proc/<PID>/` | 그 프로세스의 상태·환경·열린 파일 |
| `/proc/self/` | 현재 프로세스 자신 |

```bash
$ ls /proc/self | head -12
attr  auxv  cgroup  cmdline  comm  coredump_filter  cpuset  cwd  environ  exe
```

- 프로세스 조회는 [[ps]] · [[pgrep]] 참조

---

## /sys — 커널·장치 계층 (가상)

```bash
$ ls /sys
block  bus  class  dev  devices  firmware  fs  kernel  module  power
```

### 설명
- 사용 목적
	- 커널이 인식한 **장치·드라이버의 계층 구조** 노출 (sysfs)
- 특이사항
	- `/proc` 이 프로세스 중심이라면 `/sys` 는 **장치 중심**
	- 값 조회뿐 아니라 **쓰기로 커널 파라미터 변경 가능** → 오조작 주의
	- `/sys/firmware/efi/efivars` 는 별도 마운트(efivarfs) → UEFI 변수 영역

```bash
$ ls /sys/class/net
enp0s1  lo
$ cat /sys/class/net/enp0s1/address
<MAC 주소>                              # 인터페이스 MAC

$ ls /sys/block
dm-0  dm-1  sr0  vda  vdb  vdc  vdd  vde
$ cat /sys/block/vda/size
83886080                                # 512바이트 섹터 수 = 40GiB
```

---

## /run — 런타임 상태 (가상)

```bash
$ ls /run | head
agetty.reload  auditd.pid  blkid  chrony  console  credentials
crond.pid  cron.reboot  cryptsetup  dbus  faillock  firewalld
fsck  initctl  initramfs  irqbalance  lock  log  lvm  motd  mount
```

### 설명
- 사용 목적
	- 부팅 이후 발생한 **PID 파일·소켓·잠금 파일** 등 휘발성 런타임 데이터
- 특이사항
	- **tmpfs(메모리 기반)** → 재부팅 시 전량 초기화 (실측 716M)
	- 구형 경로 `/var/run` `/var/lock` 은 여기로의 심볼릭 링크

```bash
$ readlink -f /var/lock
/run/lock
$ ls -l /var/run /var/lock
/var/run  -> ../run
/var/lock -> ../run/lock
```

- 로그인 세션별 디렉터리 존재 → `/run/user/<UID>`

```bash
$ ls -ld /run/user/1000
drwx------. 3 admin1 admin1 80 /run/user/1000      # 700, 세션 소유자 전용
```

---

## /usr — 설치된 소프트웨어 본체

```bash
$ ls -la /usr
dr-xr-xr-x.  2 root root 20480 bin        # 일반 사용자 명령 747개
drwxr-xr-x.  2 root root     6 games
drwxr-xr-x.  3 root root    23 include    # C 헤더
dr-xr-xr-x. 29 root root  4096 lib        # 라이브러리·커널 모듈·systemd 유닛
dr-xr-xr-x. 42 root root 20480 lib64      # 64비트 공유 라이브러리
drwxr-xr-x. 25 root root  4096 libexec    # 사용자가 직접 실행하지 않는 보조 실행 파일
drwxr-xr-x. 12 root root   131 local      # 관리자 직접 설치 영역
dr-xr-xr-x.  2 root root 12288 sbin       # 관리자 명령 390개
drwxr-xr-x. 82 root root  4096 share      # 아키텍처 독립 데이터
drwxr-xr-x.  4 root root    34 src        # 커널·디버그 소스
lrwxrwxrwx.  1 root root    10 tmp -> ../var/tmp
```

### 설명
- 사용 목적
	- **패키지가 설치하는 파일 전량**의 보관처 (실행 파일·라이브러리·문서·데이터)
- 특이사항
	- 실측 **1.5G — 시스템 용량의 대부분**
	- 원칙상 **읽기 전용으로 마운트 가능** → 런타임 변경 데이터는 `/var` 로 분리되기 때문
	- `usr-merge` 로 `/bin` `/sbin` `/lib` `/lib64` 를 흡수

### 하위 용량 실측

```bash
$ sudo du -sh /usr/*
 75M  /usr/bin
986M  /usr/lib        # 커널 모듈·firmware 포함으로 최대
215M  /usr/lib64
 11M  /usr/libexec
4.0K  /usr/local      # 최소 설치 → 비어 있음
 38M  /usr/sbin
199M  /usr/share
   0  /usr/src
```

### /usr/local — 관리자 직접 설치 영역

```bash
$ ls /usr/local
bin  etc  games  include  lib  lib64  libexec  sbin  share  src
```

- `/usr` 와 동일한 구조를 그대로 반복 → **소스 컴파일 설치(`--prefix=/usr/local`)의 기본 목적지**
- 패키지 관리자가 건드리지 않는 영역 → **패키지 파일과 수동 설치 파일의 충돌 방지**
- 실측 4.0K(빈 상태) → 최소 설치에서는 미사용

### /usr/share — 아키텍처 독립 데이터

```bash
$ ls -d /usr/share/{man,doc,info,locale,zoneinfo}
/usr/share/doc  /usr/share/info  /usr/share/locale  /usr/share/man  /usr/share/zoneinfo
```

| 경로 | 내용 |
| --- | --- |
| `man/` | 매뉴얼 페이지 |
| `doc/` | 패키지 문서 |
| `locale/` | 번역 메시지 ([[localectl]]) |
| `zoneinfo/` | 시간대 데이터 (`/etc/localtime` 의 링크 대상) |

### 파일과 패키지의 소속 확인

```bash
$ rpm -qf /usr/bin/ls /usr/sbin/ip /usr/lib64/libc.so.6 /etc/fstab
coreutils-8.32-40.el9.aarch64
iproute-6.17.0-2.el9.aarch64
glibc-2.34-266.el9_8.aarch64
setup-2.13.7-10.el9.noarch

$ rpm -qf /var/log/messages
file /var/log/messages is not owned by any package     # 런타임 생성물
```

- **`/usr` 하위는 패키지 소속, `/var` 하위 로그는 소속 없음** → 두 디렉터리의 성격 차이가 그대로 드러남
- 조회 방법은 [[rpm]] 참조

---

## /var — 변동 데이터

```bash
$ ls -la /var
drwxr-xr-x. adm  cache  crash  db  empty  ftp  games  kerberos  lib  local
drwxr-xr-x. log  nis  opt  preserve  spool  tmp  yp
lrwxrwxrwx. lock -> ../run/lock
lrwxrwxrwx. mail -> spool/mail
lrwxrwxrwx. run  -> ../run
```

### 설명
- 사용 목적
	- 운영 중 **크기가 변하는 데이터** 보관 — 로그·캐시·스풀·DB·상태 파일
- 특이사항
	- **용량 폭증의 주범** → 로그·캐시 누적. 운영 서버는 별도 파티션 분리 권장
	- `/usr` 를 읽기 전용으로 유지하기 위한 **쓰기 영역 분리**가 존재 이유

### 하위 용량 실측

```bash
$ sudo du -sh /var/{cache,lib,log,spool,tmp}
 90M  /var/cache      # dnf 메타데이터·패키지 캐시가 대부분
 46M  /var/lib
5.1M  /var/log
 12K  /var/spool
4.0K  /var/tmp
```

- `/var/cache` 는 **삭제해도 재생성 가능** → 용량 확보 1순위 (`dnf clean all`)

### /var/log — 로그

```bash
$ ls /var/log
anaconda  audit  btmp  chrony  cron  dnf.librepo.log  dnf.log  dnf.rpm.log
firewalld  hawkey.log  lastlog  maillog  messages  private  README
secure  spooler  sssd  wtmp
```

| 파일 | 내용 | 조회 |
| --- | --- | --- |
| `messages` | 일반 시스템 로그 | `tail` · `grep` |
| `secure` | 인증·보안 (SSH 로그인 등) | `grep` |
| `cron` | cron 실행 이력 | — |
| `maillog` | 메일 | — |
| `audit/` | 감사 로그 (SELinux AVC 포함) | [[ausearch]] |
| `wtmp` | 로그인 이력 (바이너리) | [[last]] |
| `btmp` | 로그인 **실패** 이력 (바이너리) | `lastb` |
| `lastlog` | 계정별 마지막 로그인 (바이너리) | `lastlog` |
| `anaconda/` | 설치 과정 기록 | — |
| `dnf.log` `dnf.rpm.log` | 패키지 작업 이력 | [[dnf]] |

- **`wtmp` `btmp` `lastlog` 는 바이너리** → `cat` 불가, 전용 명령 필요
- systemd 저널은 별도 위치(`/var/log/journal` 또는 `/run/log/journal`) → [[journalctl]]

### /var/lib — 서비스 영속 상태

```bash
$ ls /var/lib | head
alternatives  authselect  bluetooth  chrony  dnf  fwupd  games  initramfs
kdump  logrotate  misc  NetworkManager  os-prober  private  rpm  rpm-state
rsyslog  selinux  sss  systemd  tpm2-tss
```

- `rpm/` = **RPM 데이터베이스 본체** → 손상 시 패키지 관리 불능

```bash
$ ls /var/lib/rpm
rpmdb.sqlite  rpmdb.sqlite-shm  rpmdb.sqlite-wal      # RHEL 9 는 SQLite 백엔드
```

### /var/spool — 처리 대기 큐

```bash
$ ls -la /var/spool
drwxr-xr-x. anacron
drwx------. cron          # 사용자별 crontab 실체
drwxr-xr-x. lpd           # 인쇄 큐
drwxrwxr-x. mail          # 사용자 메일함 (/var/mail 의 링크 대상)
```

- `crontab -e` 로 작성한 내용이 `/var/spool/cron/<사용자명>` 에 저장 → [[crontab]]

---

## /tmp · /var/tmp — 임시 파일

```bash
$ stat -c '%n %A %a %U:%G' /tmp /var/tmp
/tmp     drwxrwxrwt 1777 root:root
/var/tmp drwxrwxrwt 1777 root:root
```

### 설명
- 사용 목적
	- 프로그램·사용자의 임시 파일 저장
- 특이사항
	- **권한 `1777` — 맨 앞 `1` 이 스티키 비트**, `ls` 표시상 마지막 `t`
	- 스티키 비트 효과: 누구나 파일 생성 가능하되 **삭제는 파일 소유자·디렉터리 소유자·root 만** 가능
	- 스티키 비트가 없으면 디렉터리 `w` 권한만으로 **타인 파일 삭제 가능** → `/tmp` 에 필수
	- `/tmp` = 재부팅 시 정리 대상, `/var/tmp` = **재부팅 후에도 유지**되는 임시 파일

```bash
$ ls /tmp
systemd-private-...-chronyd.service-b0mpMT        # 서비스별 격리된 /tmp
systemd-private-...-dbus-broker.service-F0cXdu
systemd-private-...-kdump.service-uZZuMA
```

- systemd 의 `PrivateTmp=yes` 설정 서비스는 **자체 /tmp 를 별도로 부여받음** → 서비스 간 임시 파일 격리
- 권한 조회는 [[stat]] · [[ls]] 참조

---

## /opt · /srv · /mnt · /media · /afs — 예약 디렉터리

```bash
$ for d in /opt /srv /mnt /media /afs; do ls -A $d; done
                       # 5개 모두 비어 있음 (최소 설치 상태)

$ rpm -qf /opt /srv /mnt /media /afs
filesystem-3.16-5.el9.aarch64    # 5개 모두 filesystem 패키지가 생성
```

### 설명
- 사용 목적

| 디렉터리 | 용도 |
| --- | --- |
| **`/opt`** | 서드파티 소프트웨어를 **단일 디렉터리 통째로** 설치 (`/opt/<제품명>/`) |
| **`/srv`** | 서비스가 **외부에 제공하는 데이터** (`/srv/www`, `/srv/ftp`) |
| **`/mnt`** | 관리자가 **임시로 수동 마운트**하는 지점 |
| **`/media`** | **이동식 매체 자동 마운트** 지점 (USB·CD) |
| `/afs` | AFS(Andrew File System) 마운트 지점. 현재 사실상 미사용 |

- 특이사항
	- **비어 있어도 삭제 금지** → `filesystem` 패키지가 소유하는 규약상 디렉터리
	- `/opt` vs `/usr/local` — `/opt` 는 제품 단위 통째 배치, `/usr/local` 은 `bin`·`lib` 구조로 분산 배치
	- `/mnt` vs `/media` — 수동 임시 vs 자동 이동식

---

## 활용 관점 정리

### 백업 우선순위

| 순위 | 대상 | 근거 |
| --- | --- | --- |
| 1 | `/etc` | 시스템 성격 전체가 여기서 재현. 23M 로 비용 최소 |
| 2 | `/home` `/root` | 사용자 데이터·개인 설정 |
| 3 | `/var/lib` `/var/spool` | 서비스 영속 상태·큐 |
| 4 | `/srv` `/opt` | 서비스 제공 데이터·서드파티 설치분 |
| 제외 | `/proc` `/sys` `/dev` `/run` | 가상 파일시스템 — 재부팅 시 재생성 |
| 제외 | `/tmp` `/var/cache` | 재생성 가능 |
| 제외 | `/usr` | 패키지 재설치로 복원 가능 |

### 용량 부족 시 점검 순서

```bash
df -hT                                    # 어느 파일시스템인지 특정
sudo du -sh /* 2>/dev/null | sort -rh     # 루트 하위 큰 순서
sudo du -sh /var/* | sort -rh             # /var 이면 하위 세분화
```

| 증상 위치 | 흔한 원인 | 대응 |
| --- | --- | --- |
| `/boot` | 커널 세대 누적 | 구버전 커널 제거 |
| `/var/cache` | dnf 캐시 | `dnf clean all` |
| `/var/log` | 로그 미순환 | logrotate 정책 점검 |
| `/tmp` | 임시 파일 잔존 | 재부팅 또는 수동 정리 |
| `/home` | 사용자 데이터 | 별도 파티션 분리 검토 |

- 조회는 [[df]] → [[du]] 순서. 상세는 [[du]] 참조

### 특이사항 종합
- **가상 파일시스템 4종(`proc` `sys` `dev` `run`)은 `du` 집계 대상에서 제외** → 포함 시 무의미한 수치·무한 순회 발생
- `/etc/mtab` 이 `/proc/self/mounts` 링크 → 마운트 상태는 **커널이 정본**, 파일이 아님
- `man hier` 로 표준 문서 조회 가능하나 **최소 설치에는 `man-pages` 미포함** → 실기 확인 결과 `No manual entry for hier`
	- 필요 시 `dnf install man-pages` 후 조회

---

## 연관 명령어
- [[ls]] : 디렉터리 목록·권한·심볼릭 링크 판별 (`-la`, `-l` 첫 글자로 파일 유형 확인)
- [[df]] : 디렉터리가 별도 파일시스템인지 판별 (`-hT`)
- [[du]] : 디렉터리별 실사용량 집계 (용량 부족 원인 추적)
- [[mount]] : 마운트 지점 연결·해제, 가상 파일시스템 확인
- [[stat]] : 권한·스티키 비트·소유자 정밀 조회 (`-c '%A %a'`)
- [[which]] : 명령의 실제 경로 확인 (usr-merge 링크 검증)
- [[lsblk]] : `/dev` 블록 장치 트리 확인 (파괴적 명령 전 선행)
- [[rpm]] : 파일의 소속 패키지 역조회 (`-qf`)
