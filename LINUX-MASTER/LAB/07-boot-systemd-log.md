---
title: LAB 07 — 부팅·systemd·로그 관리·복구
type: exam-lab
part: 07
tags:
  - exam/linux-master
  - exam/lab
  - linux/boot
  - linux/systemd
  - linux/log
  - task/configure
  - task/verify
  - task/recovery
related: ["[[README]]", "[[06-process-scheduling-diagnosis]]", "[[08-network-config]]", "[[../THEORY/systemd-service]]", "[[../THEORY/linux-basics]]", "[[../THEORY/system-security]]", "[[../../BOOT-RECOVERY/rescue-mode]]", "[[../../BOOT-RECOVERY/chroot]]", "[[../../BOOT-RECOVERY/grub2-install]]", "[[../../SERVICE-SYSTEMD/systemctl]]", "[[../../SERVICE-SYSTEMD/journalctl]]"]
distro: Rocky Linux 9 (aarch64, UTM)
updated: 2026-09-03
---

# LAB 07 — 부팅·systemd·로그 관리·복구

- 부팅 흐름(UEFI → GRUB2 → 커널·initramfs → systemd) 을 실제 파일·명령으로 추적하고 GRUB2 설정 변경·커널 파라미터 조작
- systemd 유닛·타겟 제어 전 범위(`systemctl` 하위 명령, 런레벨 대응, isolate/rescue) + 커스텀 서비스 `lab-monitor.service` 작성·재시작 정책 검증
- 로그 3계층 정비: journald 영구화 → rsyslog 규칙 추가(`local5`) → logrotate 순환 정책, `last`/`lastb`/`lastlog` 바이너리 로그 조회
- 장애 2종 실제 유발·복구: root 비밀번호 분실(`rd.break`), `/etc/fstab` 오타 → emergency 모드. GRUB 손상 복구는 절차 참고(※)

> **이 파트의 시나리오**: Part 04 에서 만든 진단 스크립트 `/usr/local/bin/sysreport.sh` 를 매분 실행하는 상시 감시 서비스로 등록하고, Part 09 서비스 구축 전에 서버의 로그 체계(journald·rsyslog·logrotate)를 정비한다. 운영 인수인계 대비로 "root 비밀번호를 잊었을 때" 와 "fstab 오타로 부팅이 멈췄을 때" 두 장애를 스스로 일으켜 복구해 본다.

- 선행 자원: `dev1`(Part 03), `/usr/local/bin/sysreport.sh`(Part 04), `/data`·`/srv/raid` 마운트(Part 05)
- ⚠️ 8절·9절은 재부팅과 UTM **콘솔 조작**이 필수 — SSH 세션만으로는 진행 불가. 시작 전 QEMU 백엔드면 스냅샷 권장

---

## 1. 부팅 과정 추적

### 1-1. 부팅 단계 정리

> **상황**: 복구 작업 전에 "어느 단계에서 멈췄는가" 를 판단할 기준표가 필요하다. 각 단계를 실제 파일·프로세스와 짝지어 둔다.

| 순서 | 단계 | 담당 | 실체(파일·명령) | 실패 시 증상 |
| --- | --- | --- | --- | --- |
| 1 | 전원 ON → 펌웨어 | UEFI(BIOS) — POST, 부팅 장치 선택 | `efibootmgr -v`, `/sys/firmware/efi` | 펌웨어 화면에서 정지, "No bootable device" |
| 2 | 부트로더 | GRUB2 (ESP 의 `shimaa64.efi` → `grubaa64.efi`) | `/boot/efi/EFI/rocky/`, `/boot/grub2/grub.cfg`, `/boot/loader/entries/*.conf` | `grub>` 프롬프트, 메뉴 미표시 |
| 3 | 커널 + initramfs | `vmlinuz` 적재, `initramfs` 를 임시 루트로 전개, 실제 루트(LVM `rl-root`) 마운트 | `/boot/vmlinuz-*`, `/boot/initramfs-*.img`, `/proc/cmdline`, `dmesg` | Kernel panic, `dracut:/#` 셸 |
| 4 | systemd(PID 1) | `/sbin/init` → `systemd`, 유닛 병렬 기동 | `ls -l /sbin/init`, `journalctl -b`, `systemd-analyze` | 특정 유닛 failed, emergency/rescue 진입 |
| 5 | default.target | `multi-user.target`(콘솔) 또는 `graphical.target` → 로그인 프롬프트 | `systemctl get-default` | 로그인 불가(비번 분실) |

- aarch64 UEFI(UTM) 특이점
  - BIOS/MBR 개념 없음 → `grub2-install` 사용 안 함, ESP(`/boot/efi`, vfat) 안의 `EFI/rocky/` 가 부트로더
  - 파일명이 `*aa64.efi` (x86_64 는 `*x64.efi`), 디바이스 트리 `/boot/dtb-<커널버전>/` 디렉터리 존재
  - 콘솔은 UTM 디스플레이 → GRUB 메뉴 편집·rd.break 셸은 모두 이 화면에서 조작

```bash
ls /sys/firmware/efi                 # 디렉터리 존재 = UEFI 부팅
ls /boot/efi/EFI/rocky/              # ESP 안의 부트로더 파일
efibootmgr -v                        # 펌웨어 부팅 항목·순서
ls /boot                             # 커널·initramfs·BLS 디렉터리
```

- `efibootmgr -v` : EFI 부팅 항목 상세 출력 (**v**erbose) — Apple Virtualization 백엔드에서 `EFI variables are not supported` 가 나오면 QEMU 백엔드 한정 참고

**검증**

```bash
lsblk -f /dev/vda | grep -E 'vfat|/boot'   # ESP(vfat) 와 /boot 파티션 확인
findmnt /boot/efi
```

```text
# ls /boot/efi/EFI/rocky/
BOOTAA64.CSV  fonts  grub.cfg  grubaa64.efi  grubenv  mmaa64.efi  shimaa64.efi ...
# efibootmgr -v
BootCurrent: 0001
BootOrder: 0001,...
Boot0001* Rocky Linux   HD(1,GPT,...)/File(\EFI\rocky\shimaa64.efi)
# ls /boot
config-5.14.0-...aarch64  dtb-5.14.0-...aarch64  efi  grub2  initramfs-5.14.0-...aarch64.img  loader  symvers-...  System.map-...  vmlinuz-5.14.0-...aarch64 ...
# findmnt /boot/efi
TARGET    SOURCE    FSTYPE OPTIONS
/boot/efi /dev/vda1 vfat   rw,relatime,...
```

> 📝 **시험 포인트**: 부팅 순서 나열(BIOS/UEFI → GRUB → 커널+initramfs → systemd → default.target) 은 거의 매 회차 출제. `initramfs` = "루트 마운트 전 필요한 드라이버를 담은 임시 루트" 정의 암기.

### 1-2. 커널·initramfs 내용 확인

> **상황**: 지금 부팅된 커널이 어떤 파라미터로 올라왔고 initramfs 에 무엇이 들어 있는지 확인한다. 뒤의 rd.break 실습에서 편집할 줄이 바로 이 값이다.

```bash
uname -r                                  # 실행 중 커널 버전
cat /proc/cmdline                         # 커널에 전달된 부팅 파라미터
lsinitrd | head -20                       # initramfs 내용(dracut 모듈·파일) 요약
lsinitrd | grep -c '^-'                   # 포함 파일 개수(참고)
```

- `lsinitrd` : initramfs 이미지 내용 나열 (**l**i**s**t **initr**am**d**isk), 인자 없으면 현재 커널용 이미지 (`dracut` 패키지)

⚠️ 참고 — `dracut -f` 는 initramfs 를 **덮어쓴다**. 손상 시 부팅 불가하므로 반드시 백업 후 실행. 이 실습에서는 필수 아님.

```bash
cp /boot/initramfs-$(uname -r).img /root/initramfs-$(uname -r).img.bak   # 백업 선행
dracut -f                                 # 현재 커널용 initramfs 재생성 (참고)
```

- `-f` : 기존 이미지 강제 덮어쓰기 (**f**orce)

**검증**

```bash
ls -l /boot/initramfs-$(uname -r).img /root/initramfs-*.bak
lsinitrd /boot/initramfs-$(uname -r).img | grep -E '^Version|dracut modules' 
```

```text
# cat /proc/cmdline
BOOT_IMAGE=(hd0,gpt2)/vmlinuz-5.14.0-...aarch64 root=/dev/mapper/rl-root ro crashkernel=... rd.lvm.lv=rl/root rd.lvm.lv=rl/swap rhgb quiet
# lsinitrd | head
Image: /boot/initramfs-5.14.0-...aarch64.img: ...M
========================================================================
Version: dracut-057-...
dracut modules:
systemd
...
```

> 📝 **시험 포인트**: `dracut --force` 는 initramfs 재생성, `grub2-mkconfig` 는 GRUB 설정 반영 — 필기 r05-6 처럼 "GRUB 변경 반영 명령" 오답 선지로 등장. `root=/dev/mapper/rl-root ro` — 루트를 **읽기 전용**으로 먼저 마운트한 뒤 systemd 가 rw 재마운트.

### 1-3. 부팅 로그 — dmesg · journalctl -b

> **상황**: 방금 부팅에서 커널이 디스크 5개(`vda~vde`)와 NIC 를 제대로 인식했는지, systemd 단계에서 실패한 유닛이 있는지 확인한다.

```bash
dmesg | grep -iE 'vd[a-e]|virtio_net|enp0s1' | head    # 하드웨어 인식
dmesg -T | tail -20                                     # 실제 시각으로 최근 메시지
dmesg -l err,warn                                       # 오류·경고만
journalctl -b                                           # 이번 부팅 전체 저널
journalctl -b -p err                                    # 이번 부팅 err 이상만
journalctl --list-boots                                 # 저장된 부팅 세션 목록
journalctl -b -1 -n 20                                  # 직전 부팅 마지막 20줄(영구 저널일 때)
```

- `dmesg -T` : 부팅 후 경과 초 대신 사람이 읽는 시각 (**T**ime)
- `dmesg -l` : 우선순위 레벨 필터 (**l**evel)
- `-b` : 이번 부팅 (**b**oot), `-b -1` 은 직전 부팅
- `-p` : 우선순위 필터 (**p**riority)
- `--list-boots` : 부팅 ID·시각 목록 — 휘발성 저널이면 현재 1건만 표시 (5절에서 영구화)

**검증**

```bash
journalctl -k -b | grep -c .        # 커널 메시지 건수 (dmesg 와 동일 소스)
dmesg | wc -l
```

```text
# journalctl --list-boots
IDX BOOT ID                          FIRST ENTRY                 LAST ENTRY
  0 3f2a...                          Wed 2026-09-03 09:00:01 KST Wed 2026-09-03 09:31:12 KST
# dmesg -l err,warn
[    2.31...] ...
```

> 📝 **시험 포인트**: `dmesg` = 커널 링 버퍼(`/var/log/dmesg` 는 부팅 시 스냅샷), `journalctl -k` 가 같은 커널 메시지. 필기 r06-43 "이번 부팅 이후 sshd 로그" = `journalctl -u sshd -b`.

### 1-4. 부팅 시간 분석 — systemd-analyze

> **상황**: 서버 재부팅이 오래 걸린다는 불만이 있을 때 어느 유닛이 시간을 잡아먹는지 찾는다.

```bash
systemd-analyze time                        # 커널/initrd/userspace 소요 시간 (인자 생략과 동일)
systemd-analyze blame | head -10            # 유닛별 소요 시간 내림차순
systemd-analyze critical-chain              # 임계 경로(의존 사슬)
systemd-analyze critical-chain sshd.service # 특정 유닛까지의 사슬
systemd-analyze plot > /root/boot.svg       # 타임라인 SVG (macOS 로 scp 해서 열람)
```

- `time` : 단계별 총 시간 — 기본 동작
- `blame` : 기동 시간 순 정렬 (**blame** = 지연 원인 지목)
- `critical-chain` : `@` 는 활성화 시점, `+` 는 소요 시간
- `plot` : SVG 간트 차트 표준 출력

**검증**

```bash
ls -lh /root/boot.svg && head -c 200 /root/boot.svg
```

```text
# systemd-analyze time
Startup finished in 3.2s (kernel) + 2.1s (initrd) + 8.7s (userspace) = 14.1s
multi-user.target reached after 8.5s in userspace
# systemd-analyze blame | head -3
5.012s NetworkManager-wait-online.service
1.203s dracut-initqueue.service
 ...
```

> 📝 **시험 포인트**: 필기 r03-6 — `systemd-analyze` 결과 뒤 "지연 유닛을 내림차순으로" → `blame`. `plot` 은 SVG, `critical-chain` 은 사슬.

---

## 2. GRUB2 부트로더

### 2-1. `/etc/default/grub` 와 BLS 구조

> **상황**: GRUB 메뉴가 5초 만에 지나가 편집 기회를 놓치기 쉽다. 설정 파일 구조를 먼저 읽고, 다음 단계에서 타임아웃을 늘린다.

```bash
cat /etc/default/grub                       # 사용자 설정 (편집 대상)
ls /etc/grub.d/                             # grub.cfg 생성 스크립트
ls /boot/loader/entries/                    # BLS 부팅 항목 (커널마다 1개)
cat /boot/loader/entries/*.conf | head -12
head -5 /boot/efi/EFI/rocky/grub.cfg        # UEFI 스텁: /boot/grub2/grub.cfg 로 넘김
```

| 항목 | 기본값 | 의미 |
| --- | --- | --- |
| `GRUB_TIMEOUT` | `5` | 메뉴 대기 초. `0` 이면 즉시 부팅, `-1` 이면 무한 대기 |
| `GRUB_DEFAULT` | `saved` | 기본 항목. `saved` = `grubenv` 의 `saved_entry` 사용 (`grub2-set-default`/`grubby --set-default` 로 변경) |
| `GRUB_DISTRIBUTOR` | `"$(sed 's, release .*$,,g' /etc/system-release)"` | 메뉴 제목 접두 |
| `GRUB_CMDLINE_LINUX` | `"crashkernel=... rd.lvm.lv=rl/root rd.lvm.lv=rl/swap rhgb quiet"` | 모든 항목에 붙는 커널 파라미터 |
| `GRUB_DISABLE_RECOVERY` | `"true"` | recovery 항목 생성 억제 |
| `GRUB_ENABLE_BLSCFG` | `true` | **BLS**(BootLoaderSpec) 사용 — 항목을 `grub.cfg` 대신 `/boot/loader/entries/*.conf` 로 관리 |
| `GRUB_TERMINAL_OUTPUT` | `"console"` | 출력 터미널 |

- BLS 항목 파일 `<machine-id>-<커널버전>.conf` : `title` `version` `linux` `initrd` `options` `grub_users` `grub_arg` `grub_class` 키
- RHEL 9 통합 경로: 실제 메뉴는 `/boot/grub2/grub.cfg`, UEFI 의 `/boot/efi/EFI/rocky/grub.cfg` 는 `configfile` 로 넘기는 몇 줄짜리 스텁. x86 BIOS 도 `/boot/grub2/grub.cfg` 동일. (RHEL 8 이전 UEFI 는 `/boot/efi/EFI/rocky/grub.cfg` 가 본체)

**검증**

```bash
grep -E '^GRUB_' /etc/default/grub
grep -l "$(uname -r)" /boot/loader/entries/*.conf
```

```text
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=... rd.lvm.lv=rl/root rd.lvm.lv=rl/swap rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
GRUB_ENABLE_BLSCFG=true
# head -5 /boot/efi/EFI/rocky/grub.cfg
search --no-floppy --fs-uuid --set=dev ...
set prefix=($dev)/grub2
export $prefix
configfile $prefix/grub.cfg
```

> 📝 **시험 포인트**: 필기 r04-6 "grub2-mkconfig 가 참조하는 설정 파일" = `/etc/default/grub`(+`/etc/grub.d/`). `grub.cfg` 직접 편집은 오답(r01-8, r03-5). GRUB Legacy 는 `/boot/grub/menu.lst`.

### 2-2. GRUB_TIMEOUT 변경 → grub2-mkconfig → 재부팅 확인

> **상황**: 뒤의 복구 실습에서 메뉴를 여유 있게 편집하도록 대기 시간을 10초로 늘린다.

```bash
cp /etc/default/grub /root/grub.default.bak                 # 백업
sed -i 's/^GRUB_TIMEOUT=.*/GRUB_TIMEOUT=10/' /etc/default/grub
grub2-mkconfig -o /boot/grub2/grub.cfg                      # 설정 재생성 (UEFI·BIOS 공통 경로)
```

- `sed -i` : 파일 내 치환 (**i**n-place)
- `grub2-mkconfig -o` : 스크립트(`/etc/grub.d/*`)와 `/etc/default/grub` 를 합쳐 출력 파일 생성 (**o**utput)
- Rocky 9 는 `Found linux image:` 줄이 나오지 않는 것이 정상 (BLS) → [[../../BOOT-RECOVERY/grub2-install]]

**검증**

```bash
grep -E '^set timeout' /boot/grub2/grub.cfg
grep -c 'menuentry' /boot/grub2/grub.cfg       # BLS 이므로 정적 항목은 적음(펌웨어 설정 항목 정도)
systemctl reboot                               # 재부팅 → UTM 콘솔에서 메뉴가 10초 머무는지 관찰
```

```text
# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Adding boot menu entry for UEFI Firmware Settings ...
done
# grep -E '^set timeout' /boot/grub2/grub.cfg
set timeout=10
```

> 📝 **시험 포인트**: 필기 r05-5 `GRUB_TIMEOUT=5` = "메뉴 대기 5초 후 기본 항목 부팅". 절차 순서 r07-16: `/etc/default/grub` 수정 → `grub2-mkconfig -o /boot/grub2/grub.cfg` → 재부팅.

### 2-3. grubby 로 커널 항목·파라미터 관리

> **상황**: 부팅 메시지를 눈으로 보고 싶어 `rhgb quiet` 를 제거해 본다. RHEL 9 에서 커널 파라미터 영구 변경의 표준 도구는 `grubby` (BLS 항목을 직접 갱신).

```bash
grubby --info=ALL                        # 모든 부팅 항목 상세
grubby --default-kernel                  # 기본 커널 경로
grubby --default-index                   # 기본 항목 인덱스
grubby --update-kernel=ALL --remove-args="rhgb quiet"     # 모든 항목에서 제거
grubby --info=DEFAULT | grep ^args
```

- `--info=ALL|DEFAULT|<커널경로>` : 항목 정보(index, kernel, args, root, initrd, title, id)
- `--default-kernel` / `--default-index` : 기본 항목 조회
- `--set-default=<커널경로>` : 기본 커널 지정 (`grubenv` 의 `saved_entry` 갱신)
- `--update-kernel=ALL|<경로>` : 대상 항목
- `--args="..."` : 파라미터 추가, `--remove-args="..."` : 제거

**검증**

```bash
grep ^options /boot/loader/entries/*.conf       # rhgb quiet 사라짐
systemctl reboot                                # 재부팅 후
cat /proc/cmdline                               # 실제 적용 확인 — 콘솔에 부팅 메시지 흐름
```

```text
# grubby --default-kernel
/boot/vmlinuz-5.14.0-...aarch64
# grubby --info=DEFAULT
index=0
kernel="/boot/vmlinuz-5.14.0-...aarch64"
args="ro crashkernel=... rd.lvm.lv=rl/root rd.lvm.lv=rl/swap"
root="/dev/mapper/rl-root"
initrd="/boot/initramfs-5.14.0-...aarch64.img"
title="Rocky Linux (5.14.0-...aarch64) 9.x (Blue Onyx)"
id="...-5.14.0-...aarch64"
```

- 실습 후 원복(선택): `grubby --update-kernel=ALL --args="rhgb quiet"`. 이 파트에서는 메시지 관찰을 위해 제거 상태 유지 가능
- `/etc/default/grub` 의 `GRUB_CMDLINE_LINUX` 를 고치고 `grub2-mkconfig` 를 실행하는 방법도 신규 설치본에서는 BLS 항목까지 갱신하나, 항목별 확실한 반영은 `grubby` 사용

> 📝 **시험 포인트**: `quiet`(커널 메시지 억제)·`rhgb`(Red Hat Graphical Boot) 는 기본 파라미터. `grubby --update-kernel=ALL --args=` 가 RHEL 계열 권장 절차.

### 2-4. grubenv · 기본 항목 · GRUB 비밀번호

> **상황**: 커널이 여러 개 설치된 뒤 어느 것이 기본으로 부팅되는지, 그리고 시험에 나오는 `grub2-setpassword` 의 위치를 확인한다.

```bash
grub2-editenv list                          # grubenv 변수 (saved_entry, boot_success ...)
grub2-editenv - list                        # 동일 ('-' = 기본 grubenv 경로)
grub2-set-default 0                         # 인덱스 0 항목을 기본으로 (grubby --set-default 와 같은 효과)
grub2-editenv list | grep saved_entry
grubby --set-default /boot/vmlinuz-$(uname -r)   # 커널 경로로 되돌리기
```

- `grub2-editenv list` : `/boot/grub2/grubenv` 환경 블록 출력 (`set`/`unset` 으로 변수 편집)
- `grub2-set-default <인덱스|id>` : `saved_entry` 설정 — `GRUB_DEFAULT=saved` 일 때만 의미

⚠️ 참고 (※ 미실행 권장) — `grub2-setpassword` 는 메뉴 편집(`e`)·명령줄(`c`)에 root 인증을 요구하게 함. 비밀번호를 잊으면 8절 rd.break 절차 자체가 막히므로 실습 VM 에서는 실행하지 않음. 해제는 `/boot/grub2/user.cfg` 삭제 후 `grub2-mkconfig`.

```bash
grub2-setpassword                            # 대화식으로 GRUB 사용자 root 비밀번호 설정 → /boot/grub2/user.cfg 생성 (※ 미실행)
```

- `grub2-install <장치>` 는 **BIOS(MBR) 전용** — UEFI aarch64 에서는 사용하지 않음. x86 BIOS 절차는 [[../../BOOT-RECOVERY/grub2-install]] 참조

**검증**

```bash
grub2-editenv list
grubby --default-kernel
```

```text
# grub2-editenv list
saved_entry=...-5.14.0-...aarch64
boot_success=1
boot_indeterminate=0
```

> 📝 **시험 포인트**: 필기 r09-5 "GRUB 메뉴 무단 편집 방지" = `grub2-setpassword`. r03-5 ④ "GRUB2 는 암호 기능 없음" 은 틀린 선지. r06-4 멀티부팅 복구는 BIOS 기준 `grub2-install /dev/sda` + `grub2-mkconfig`.

### 2-5. 커널 파라미터 참조표

> **상황**: GRUB 메뉴에서 `e` 로 임시 편집해 넣는 파라미터를 정리한다. 8·9·10절에서 실제 사용.

| 파라미터 | 효과 | 용도 |
| --- | --- | --- |
| `quiet` | 커널 메시지 억제 | 기본값 |
| `rhgb` | 그래픽 부팅 화면 | 기본값 (서버 콘솔에선 무의미) |
| `systemd.unit=rescue.target` | 지정 타겟으로 부팅 (root 비번 필요) | 단일 사용자 모드 진입 |
| `systemd.unit=emergency.target` | 최소 환경, `/` 읽기 전용 | fstab·디스크 문제 진단 |
| `rd.break` | initramfs 단계에서 중단 → `switch_root:/#` 셸 (비번 불필요) | root 비번 재설정 |
| `init=/bin/bash` | PID 1 을 systemd 대신 bash 로 | rd.break 대안 |
| `selinux=0` | SELinux 완전 비활성 (정책 미로드) | 라벨 문제로 부팅 불가 시 |
| `enforcing=0` | Permissive 로 부팅 | 라벨 차단 회피, rd.break 와 병용 |
| `single` / `s` / `1` | rescue.target (SysV 런레벨 호환) | 구형 표기 (r10-5) |
| `3` / `5` | multi-user / graphical 타겟 | 숫자 런레벨 호환 |
| `console=ttyAMA0` 등 | 시리얼 콘솔 지정 | aarch64 VM 에서 이미 있으면 유지 |

> 📝 **시험 포인트**: 필기 r06-5 "단일 사용자 유사 모드" = `systemd.unit=rescue.target`; r10-5 GRUB `e` → linux 줄 끝 `single` → `Ctrl+x`. `rd.break` 는 비밀번호 없이 셸을 주므로 물리 접근 보안이 중요.

---

## 3. systemd 유닛·타겟

### 3-1. 유닛 종류·위치 우선순위

> **상황**: 커스텀 유닛을 어디에 두어야 패키지 업데이트에 덮어쓰이지 않는지, 유닛 종류별 확장자를 확인한다.

| 유닛 | 확장자 | 용도 | 예 |
| --- | --- | --- | --- |
| 서비스 | `.service` | 데몬·프로세스 | `sshd.service` |
| 소켓 | `.socket` | 소켓 활성화 | `cockpit.socket` |
| 타겟 | `.target` | 유닛 그룹 = 런레벨 | `multi-user.target` |
| 타이머 | `.timer` | 시간 기반 실행(cron 대체) | `logrotate.timer` |
| 마운트 | `.mount` | 마운트 지점 (fstab 자동 변환) | `data.mount` |
| 경로 | `.path` | 파일 변화 감시 | `systemd-ask-password-console.path` |
| 슬라이스 | `.slice` | cgroup 자원 그룹 | `user.slice` |

```bash
ls /usr/lib/systemd/system | head            # 패키지 제공 (수정 금지)
ls /etc/systemd/system                       # 관리자 커스텀·오버라이드 (최우선)
ls /run/systemd/system                       # 런타임 생성 (휘발)
systemctl list-unit-files --type=service | head -5
systemctl list-units --type=mount | grep -E 'data|srv'      # fstab → .mount 유닛 자동 생성
```

- 우선순위: `/etc/systemd/system` > `/run/systemd/system` > `/usr/lib/systemd/system` (같은 이름이면 앞쪽이 이김)
- `list-unit-files` : 설치된 유닛 파일과 enable 상태, `list-units` : 로드된(메모리 상) 유닛과 active 상태

**검증**

```bash
systemctl show -p FragmentPath sshd            # 실제 로드된 유닛 파일 경로
systemctl show -p FragmentPath data.mount
```

```text
FragmentPath=/usr/lib/systemd/system/sshd.service
FragmentPath=/run/systemd/generator/data.mount
```

> 📝 **시험 포인트**: 확장자↔용도 매칭 최빈출. `.timer` = cron 대체, `.target` = 런레벨, `.mount` = 마운트. `/etc/systemd/system` 이 우선.

### 3-2. 유닛 목록·상태 조회

> **상황**: 서비스 구축 전 현재 서버에서 실행 중·실패한 유닛을 파악한다.

```bash
systemctl list-units                                  # 로드된 활성 유닛
systemctl list-units --type=service --state=running    # 실행 중 서비스만
systemctl list-units --type=service --state=failed
systemctl --failed                                     # 실패 유닛 (= list-units --state=failed)
systemctl list-unit-files --state=enabled | head
systemctl status sshd
systemctl is-active sshd; systemctl is-enabled sshd; systemctl is-failed sshd
systemctl list-dependencies multi-user.target | head -20
systemctl cat sshd                                     # 유닛 파일 원문(드롭인 포함)
systemctl show -p MainPID -p ActiveState -p NRestarts sshd
```

- `--type=` : 유닛 종류 필터, `--state=` : `running`/`failed`/`inactive`/`enabled` 등
- `--failed` : 실패 유닛 단축 옵션
- `is-active`/`is-enabled`/`is-failed` : 한 단어 응답 + 종료 코드 (스크립트용)
- `list-dependencies` : 트리 형태 의존성 (`●` 활성, `○` 비활성)
- `cat` : 로드된 유닛 파일 내용, `show -p` : 속성값 조회 (**p**roperty)

**검증**

```bash
systemctl is-active sshd && echo "sshd OK"
systemctl --failed --no-legend | wc -l          # 0 이 목표
```

```text
# systemctl is-active sshd
active
# systemctl is-enabled sshd
enabled
# systemctl --failed
  UNIT LOAD ACTIVE SUB DESCRIPTION
0 loaded units listed.
```

> 📝 **시험 포인트**: `is-active`(지금 실행 여부) vs `is-enabled`(부팅 자동시작 여부) 는 별개. `list-units` 는 로드된 것, `list-unit-files` 는 설치된 것.

### 3-3. 서비스 제어 — start/stop/restart/reload, enable/disable, mask

> **상황**: Part 09 에서 반복할 제어 명령을 `chronyd`(이미 설치·실행 중) 로 연습하고, `mask` 가 수동 start 까지 막는지 검증한다.

```bash
systemctl stop chronyd
systemctl is-active chronyd                    # inactive
systemctl start chronyd
systemctl restart chronyd                      # 중지 후 시작
systemctl reload chronyd                       # 설정만 재적용 (ExecReload 있는 유닛만)
systemctl try-restart chronyd                  # 실행 중일 때만 재시작
systemctl reload-or-restart chronyd            # reload 지원 시 reload, 아니면 restart
systemctl disable chronyd                      # 자동시작 해제 (지금 실행 중인 건 유지)
systemctl enable --now chronyd                 # 등록 + 즉시 시작

systemctl mask chronyd                         # /dev/null 링크 → start 불가
systemctl start chronyd                        # 실패 확인
systemctl unmask chronyd
systemctl enable --now chronyd
```

- `reload` : 데몬 재기동 없이 설정 재읽기 (SIGHUP 등), `try-restart` : 비활성이면 아무 것도 안 함
- `reload-or-restart` : reload 미지원 유닛 대응
- `enable` : `/etc/systemd/system/multi-user.target.wants/<유닛>` 심볼릭 링크 생성, `--now` 병행 시 start
- `mask` : `/etc/systemd/system/<유닛>` → `/dev/null` 링크 — 의존성으로도 기동 불가

**검증**

```bash
ls -l /etc/systemd/system/multi-user.target.wants/chronyd.service    # enable 링크
systemctl mask chronyd && ls -l /etc/systemd/system/chronyd.service   # → /dev/null
systemctl start chronyd; echo "exit=$?"
systemctl unmask chronyd && systemctl enable --now chronyd && systemctl is-active chronyd
```

```text
lrwxrwxrwx. 1 root root 40 ... /etc/systemd/system/multi-user.target.wants/chronyd.service -> /usr/lib/systemd/system/chronyd.service
lrwxrwxrwx. 1 root root 9 ... /etc/systemd/system/chronyd.service -> /dev/null
Failed to start chronyd.service: Unit chronyd.service is masked.
exit=1
active
```

> 📝 **시험 포인트**: 필기 r03-30, r06-38, r10-36 — `disable` 은 수동 start·의존성 기동 가능, `mask` 는 모두 차단. `enable --now` (r06-37, r08-44) 한 번에 등록+시작. 실기 r01-8 `systemctl enable httpd`.

### 3-4. 타겟 ↔ 런레벨, 기본 타겟

> **상황**: 서버는 GUI 없이 운영하므로 기본 타겟이 `multi-user.target` 인지 확인하고 SysV 호환 심볼릭 링크를 눈으로 본다.

| 런레벨 | 타겟 | 상태 |
| --- | --- | --- |
| 0 | `poweroff.target` | 종료 |
| 1 / S | `rescue.target` | 단일 사용자 (root 비번 필요, 로컬 FS 마운트, 네트워크 없음) |
| 2·3·4 | `multi-user.target` | 다중 사용자 텍스트 (3 이 표준) |
| 5 | `graphical.target` | 다중 사용자 + GUI |
| 6 | `reboot.target` | 재부팅 |
| — | `emergency.target` | 최소 환경, `/` 읽기 전용, sysinit 미실행 |

```bash
ls -l /usr/lib/systemd/system/runlevel?.target       # SysV 호환 링크
systemctl get-default
systemctl set-default multi-user.target
ls -l /etc/systemd/system/default.target
runlevel                                             # 이전 현재 런레벨 (N 3)
who -r                                               # 동일 정보
ls -l /sbin/init /sbin/telinit                       # systemd 로의 링크
systemctl list-dependencies graphical.target --no-pager | head -5     # multi-user 를 포함
```

- `get-default`/`set-default` : `/etc/systemd/system/default.target` 심볼릭 링크 조회·변경
- `runlevel` : `N` 은 이전 런레벨 없음(부팅 직후). `who -r` 동일
- `telinit <n>` / `init <n>` : systemd 가 `isolate` 로 변환해 처리 (호환 명령)

**검증**

```bash
systemctl get-default
readlink -f /etc/systemd/system/default.target
```

```text
# ls -l /usr/lib/systemd/system/runlevel?.target
... runlevel0.target -> poweroff.target
... runlevel1.target -> rescue.target
... runlevel2.target -> multi-user.target
... runlevel3.target -> multi-user.target
... runlevel4.target -> multi-user.target
... runlevel5.target -> graphical.target
... runlevel6.target -> reboot.target
# runlevel
N 3
# ls -l /sbin/init
lrwxrwxrwx. 1 root root 22 ... /sbin/init -> ../lib/systemd/systemd
# readlink -f /etc/systemd/system/default.target
/usr/lib/systemd/system/multi-user.target
```

> 📝 **시험 포인트**: 런레벨↔타겟 매칭은 r01-7, r02-6, r03-18, r05-7, r07-15 등 거의 매 회차. r08-9 `runlevel` 출력 `3 5` = 이전 3 → 현재 5. r07-2 타겟 도달 순서 `sysinit → basic → multi-user → graphical`.

### 3-5. isolate rescue.target → 복귀 (콘솔)

> **상황**: 운영 중 디스크 점검을 위해 단일 사용자 모드로 내려가는 절차를 연습한다. 네트워크가 끊기므로 UTM 콘솔에서 진행한다.

⚠️ `isolate rescue.target` 은 sshd·네트워크를 포함한 다른 유닛을 모두 정지 — **UTM 콘솔에서만** 실행. SSH 세션은 끊김.

```bash
# (UTM 콘솔, root)
systemctl isolate rescue.target        # = init 1 / telinit 1
# → "Give root password for maintenance" → root 비번 입력 → sh 프롬프트
systemctl list-units --type=service --state=running   # 최소 서비스만
findmnt /data                          # 로컬 FS 는 마운트됨
ip -br a                               # 주소 없음(또는 lo 만)
systemctl isolate multi-user.target    # 복귀 (= init 3)
```

- `isolate <타겟>` : 해당 타겟과 의존 유닛만 남기고 나머지 정지 — 재부팅 없이 런레벨 전환
- rescue 진입 시 root 비밀번호 필요 (rd.break 와의 차이)

| 구분 | `rescue.target` | `emergency.target` |
| --- | --- | --- |
| 진입 조건 | root 비번 | root 비번 |
| 루트 FS | rw 마운트 | **ro** 마운트 (`mount -o remount,rw /` 필요) |
| fstab 의 다른 FS | 마운트 시도 | 마운트 안 함 |
| sysinit.target | 실행 | 미실행 (최소) |
| 자동 진입 사례 | 수동 요청 | fstab 오류·로컬 FS 실패 (9절) |

**검증**

```bash
systemctl is-active multi-user.target sshd chronyd   # 복귀 후 모두 active
systemctl is-active rescue.target                    # inactive
```

```text
active
active
active
inactive
```

> 📝 **시험 포인트**: 필기 r06-6 "fstab 오타로 `/` 만 읽기 전용 최소 셸" = `emergency.target`. `isolate` = SysV `init N`. `set-default` 는 다음 부팅부터, `isolate` 는 즉시.

### 3-6. 종료·재부팅 명령군

> **상황**: 운영자에게 5분 뒤 재부팅을 예고하고 취소하는 흐름을 연습한다. 실제 재부팅은 2절·8절에서 수행.

```bash
shutdown -r +5 "5분 후 점검 재부팅 — 저장 후 로그아웃"     # 예약 + wall 방송
shutdown --show                                           # 예약 확인 (systemd 253+; 없으면 생략)
shutdown -c                                               # 취소
wall "테스트 방송입니다"                                   # 전체 터미널 메시지
# 실제 실행 시 아래 중 하나 (여기서는 미실행)
# systemctl reboot   == reboot   == shutdown -r now
# systemctl poweroff == poweroff == shutdown -h now (전원 차단)
# systemctl halt     == halt     (CPU 정지, 전원 유지 가능)
# systemctl suspend               (VM 에서는 미지원 가능)
```

- `-r` : 재부팅 (**r**eboot), `-h` : 정지/전원 종료 (**h**alt), `-c` : 예약 취소 (**c**ancel)
- `+5` : 5분 후, `now` = `+0`, `hh:mm` 절대 시각. 시간 생략 시 `+1`
- 예약 중에는 `/run/nologin` 생성 → 일반 사용자 신규 로그인 차단
- `wall` : 로그인한 모든 터미널에 메시지 (**w**rite **all**)

**검증**

```bash
shutdown -r +5 "test" ; ls -l /run/nologin ; shutdown -c ; ls /run/nologin
```

```text
Shutdown scheduled for Wed 2026-09-03 10:05:00 KST, use 'shutdown -c' to cancel.
-rw-r--r--. 1 root root ... /run/nologin
ls: cannot access '/run/nologin': No such file or directory
```

> 📝 **시험 포인트**: `shutdown -h now` = `poweroff`, `shutdown -r now` = `reboot`, `init 0` = `poweroff.target`, `init 6` = `reboot.target`. `-c` 로 취소.

### 3-7. SysV 호환 명령 확인

> **상황**: 구형 절차서에 나오는 `service`·`chkconfig` 가 Rocky 9 에서 어떻게 처리되는지 확인한다.

```bash
rpm -q chkconfig initscripts || dnf install -y chkconfig initscripts   # 없을 때만
chkconfig --list                        # systemd 안내문 + SysV 잔존 서비스만
service chronyd status                  # → systemctl status chronyd 로 리다이렉트
```

- `chkconfig --list` : SysV 서비스 런레벨별 on/off 표 — systemd 유닛은 표시 안 되고 안내문 출력
- `service <name> <action>` : `/etc/init.d/<name>` 없으면 `systemctl <action> <name>.service` 로 전달

**검증**

```bash
service chronyd status | head -3
```

```text
# chkconfig --list
Note: This output shows SysV services only and does not include native
      systemd services. SysV configuration data might be overridden by native
      systemd configuration.
      If you want to list systemd services use 'systemctl list-unit-files'.
      To see services enabled on particular target use
      'systemctl list-dependencies [target]'.
# service chronyd status
Redirecting to /bin/systemctl status chronyd.service
● chronyd.service - NTP client/server
     Loaded: loaded (/usr/lib/systemd/system/chronyd.service; enabled; ...)
```

> 📝 **시험 포인트**: `chkconfig --list` ↔ `systemctl list-unit-files`, `chkconfig on` ↔ `enable`, `service start` ↔ `systemctl start`. `/etc/inittab` 은 RHEL 7+ 에서 안내문만 남은 빈 파일.

---

## 4. 커스텀 서비스 `lab-monitor.service`

### 4-1. 감시 스크립트 작성

> **상황**: Part 04 의 `sysreport.sh` 를 60초마다 실행해 저널에 남기는 루프 스크립트를 만든다. 신호 처리를 넣어 `systemctl stop`(SIGTERM)·`reload`(SIGHUP) 가 정상 동작하게 한다.

```bash
test -x /usr/local/bin/sysreport.sh || echo "Part 04 미수행 — sysreport.sh 없음"
# (없을 때만) 임시 최소본
# cat > /usr/local/bin/sysreport.sh <<'EOF'
# #!/bin/bash
# echo "== $(date '+%F %T') load: $(cut -d' ' -f1-3 /proc/loadavg) mem: $(free -m | awk '/Mem/{print $3"/"$2"MB"}') disk: $(df -h / | awk 'NR==2{print $5}')"
# EOF
# chmod 755 /usr/local/bin/sysreport.sh

cat > /usr/local/bin/lab-monitor.sh <<'EOF'
#!/bin/bash
# lab-monitor.sh — 60초 주기로 sysreport.sh 실행, 출력은 journald 로
INTERVAL=${INTERVAL:-60}
trap 'echo "lab-monitor: SIGTERM 수신, 종료"; exit 0' TERM
trap 'echo "lab-monitor: SIGHUP 수신, 설정 재적용"' HUP
echo "lab-monitor 시작 (PID $$, interval ${INTERVAL}s)"
while true; do
    /usr/local/bin/sysreport.sh
    sleep "$INTERVAL" &
    wait $!
done
EOF
chmod 755 /usr/local/bin/lab-monitor.sh
```

- `trap '...' TERM|HUP` : 시그널 수신 시 실행할 명령 — `Type=simple` 서비스는 stop 시 SIGTERM 을 받음
- `sleep & wait $!` : 백그라운드 sleep 을 `wait` 로 기다려야 sleep 도중에도 trap 이 즉시 실행됨
- `${INTERVAL:-60}` : 환경변수 미설정 시 60 (드롭인의 `Environment=` 로 변경 가능)

**검증**

```bash
ls -l /usr/local/bin/lab-monitor.sh /usr/local/bin/sysreport.sh
bash -n /usr/local/bin/lab-monitor.sh && echo "syntax OK"
timeout 3 /usr/local/bin/lab-monitor.sh | head -3      # 3초만 돌려보기
```

```text
-rwxr-xr-x. 1 root root ... /usr/local/bin/lab-monitor.sh
-rwxr-xr-x. 1 root root ... /usr/local/bin/sysreport.sh
syntax OK
lab-monitor 시작 (PID ..., interval 60s)
== 2026-09-03 ... load: ...
```

> 📝 **시험 포인트**: 서비스용 스크립트는 **포그라운드 무한 루프**(`Type=simple`) 또는 자기 데몬화(`Type=forking`) 중 하나로 맞춰야 systemd 가 상태를 올바로 추적.

### 4-2. 유닛 파일 작성 → daemon-reload → enable --now

> **상황**: 스크립트를 systemd 서비스로 등록한다. 비정상 종료 시 5초 후 자동 재시작되도록 한다.

```bash
cat > /etc/systemd/system/lab-monitor.service <<'EOF'
[Unit]
Description=LAB system monitor (sysreport every 60s)
Documentation=file:/usr/local/bin/lab-monitor.sh
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/lab-monitor.sh
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload                 # 새 유닛 파일 인식
systemctl enable --now lab-monitor      # 등록 + 시작
systemctl status lab-monitor --no-pager
```

| Type | 동작 | 용도 |
| --- | --- | --- |
| `simple` | ExecStart 프로세스 = 메인 프로세스, 즉시 started 판정 | 포그라운드 데몬 (기본) |
| `forking` | 부모가 fork 후 종료 → 자식이 메인 (`PIDFile=` 권장) | 전통적 데몬화 프로그램 |
| `oneshot` | 프로세스 종료 후 started, `RemainAfterExit=yes` 병용 | 1회성 설정 스크립트 |
| `notify` | 데몬이 `sd_notify()` 로 준비 완료 통지 | systemd 인식 데몬 |
| `exec` | 실제 exec 성공 시점에 started | simple 보다 엄격 |

- `[Unit] After=` : 순서만 (network.target 을 함께 시작시키진 않음 — 그건 `Wants=`/`Requires=`)
- `Documentation=` : `systemctl status` 에 표시
- `ExecReload`/`ExecStop` : `$MAINPID` 는 systemd 가 채우는 메인 PID 변수
- `Restart=on-failure` : 비정상 종료(0 이외 코드·시그널)만 재시작. `always` 는 정상 종료도 재시작
- `RestartSec=5` : 재시작 대기 초 (기본 100ms)
- `User=root` : 실행 계정 (명시)
- `[Install] WantedBy=multi-user.target` : enable 시 `multi-user.target.wants/` 에 링크

**검증**

```bash
systemctl is-active lab-monitor; systemctl is-enabled lab-monitor
ls -l /etc/systemd/system/multi-user.target.wants/lab-monitor.service
systemctl show -p MainPID -p Restart -p RestartUSec -p Type lab-monitor
journalctl -u lab-monitor -n 5 --no-pager
```

```text
active
enabled
lrwxrwxrwx. 1 root root 40 ... lab-monitor.service -> /etc/systemd/system/lab-monitor.service
MainPID=...
Type=simple
Restart=on-failure
RestartUSec=5s
Sep 03 10:20:01 srv01 systemd[1]: Started LAB system monitor (sysreport every 60s).
Sep 03 10:20:01 srv01 lab-monitor.sh[...]: lab-monitor 시작 (PID ..., interval 60s)
Sep 03 10:20:01 srv01 lab-monitor.sh[...]: == 2026-09-03 ... load: ...
```

> 📝 **시험 포인트**: 필기 r03-29·r05-42 — `WantedBy=multi-user.target` 은 enable 시 wants 링크 생성; `After=` 는 순서만; `Restart=on-failure` 는 정상 종료 시 재시작 안 함; `ExecStart` 만으로 자동 실행 보장 안 됨(enable 필요). 유닛 파일 수정 후 `daemon-reload` 필수.

### 4-3. 실시간 로그 추적과 reload/stop 동작

> **상황**: 서비스가 1분마다 보고를 남기는지 `journalctl -f` 로 지켜보고, reload·stop 시 trap 메시지가 찍히는지 확인한다.

```bash
journalctl -u lab-monitor -f          # 다른 터미널에서 실행, Ctrl+C 로 종료
systemctl reload lab-monitor          # SIGHUP → "설정 재적용" 로그
systemctl stop lab-monitor            # SIGTERM → "종료" 로그, 재시작 안 됨(정상 종료)
systemctl start lab-monitor
```

- `-f` : 신규 로그 실시간 출력 (**f**ollow)

**검증**

```bash
journalctl -u lab-monitor --since "2 min ago" --no-pager | grep -E 'SIGHUP|SIGTERM|Started|Stopped'
systemctl show -p NRestarts --value lab-monitor       # 정상 stop/start 는 0 유지
```

```text
Sep 03 10:22:10 srv01 lab-monitor.sh[...]: lab-monitor: SIGHUP 수신, 설정 재적용
Sep 03 10:22:30 srv01 lab-monitor.sh[...]: lab-monitor: SIGTERM 수신, 종료
Sep 03 10:22:30 srv01 systemd[1]: Stopped LAB system monitor (sysreport every 60s).
Sep 03 10:22:45 srv01 systemd[1]: Started LAB system monitor (sysreport every 60s).
0
```

> 📝 **시험 포인트**: `reload` 는 `ExecReload=` 가 정의된 유닛에서만 동작 — 없으면 "Job type reload is not applicable". `--value` 는 `show -p` 의 값만 출력.

### 4-4. kill -9 로 장애 유발 → Restart 검증

> **상황**: 감시 프로세스가 OOM 등으로 강제 종료돼도 5초 뒤 살아나는지 확인한다.

```bash
PID=$(systemctl show -p MainPID --value lab-monitor); echo $PID
kill -9 $PID                          # SIGKILL — trap 불가, 비정상 종료
sleep 7
systemctl show -p MainPID --value lab-monitor       # 새 PID
systemctl show -p NRestarts --value lab-monitor     # 1
systemctl status lab-monitor --no-pager | head -6
```

- `kill -9` : SIGKILL, 프로세스가 가로챌 수 없음 → 종료 코드 `killed (signal=KILL)` → `on-failure` 조건 충족

**검증**

```bash
journalctl -u lab-monitor -n 6 --no-pager
```

```text
Sep 03 10:25:00 srv01 systemd[1]: lab-monitor.service: Main process exited, code=killed, status=9/KILL
Sep 03 10:25:00 srv01 systemd[1]: lab-monitor.service: Failed with result 'signal'.
Sep 03 10:25:05 srv01 systemd[1]: lab-monitor.service: Scheduled restart job, restart counter is at 1.
Sep 03 10:25:05 srv01 systemd[1]: Started LAB system monitor (sysreport every 60s).
```

> 📝 **시험 포인트**: `NRestarts` 는 자동 재시작 횟수. `StartLimitBurst`(기본 5회/10초) 초과 시 `start-limit-hit` 로 더 이상 재시작 안 함 → `systemctl reset-failed` 필요.

### 4-5. 드롭인 오버라이드 (`systemctl edit`)

> **상황**: 재시작 대기를 10초로 늘리고 주기를 30초로 바꾸되, 원본 유닛 파일은 건드리지 않는다.

```bash
export SYSTEMD_EDITOR=vi
systemctl edit lab-monitor              # 드롭인 편집기 열림 → 아래 3줄 입력 후 :wq
# [Service]
# RestartSec=10
# Environment=INTERVAL=30

# 편집기 없이 동일 작업
mkdir -p /etc/systemd/system/lab-monitor.service.d
cat > /etc/systemd/system/lab-monitor.service.d/override.conf <<'EOF'
[Service]
RestartSec=10
Environment=INTERVAL=30
EOF
systemctl daemon-reload
systemctl restart lab-monitor
systemctl cat lab-monitor               # 원본 + 드롭인 함께 표시
# systemctl edit --full lab-monitor     # 원본 전체를 /etc 에 복사해 편집 (참고)
```

- `edit` : `<유닛>.d/override.conf` 드롭인 생성 — 지정한 키만 덮어씀
- `edit --full` : 유닛 파일 전체 복사본 편집 (`/usr/lib` 유닛을 `/etc` 로 가져올 때)
- `Environment=KEY=VAL` : 서비스 환경변수
- `SYSTEMD_EDITOR` : `systemctl edit` 가 쓰는 편집기 (`EDITOR` 보다 우선)

**검증**

```bash
systemctl show -p RestartUSec -p Environment lab-monitor
journalctl -u lab-monitor -n 2 --no-pager | grep interval
systemctl cat lab-monitor | grep -E '^# /'
```

```text
RestartUSec=10s
Environment=INTERVAL=30
... lab-monitor 시작 (PID ..., interval 30s)
# /etc/systemd/system/lab-monitor.service
# /etc/systemd/system/lab-monitor.service.d/override.conf
```

> 📝 **시험 포인트**: 드롭인 경로 `/etc/systemd/system/<유닛>.d/*.conf`. 패키지 유닛을 직접 수정하면 업데이트 때 덮어쓰이므로 드롭인이 정석.

### 4-6. 일시 유닛 `systemd-run` (참고)

> **상황**: 유닛 파일 없이 명령 하나를 잠깐 서비스로 돌려 cgroup·로그 관리 혜택을 받는 방법.

```bash
systemd-run --unit=labtest sleep 100        # 일시(transient) 서비스 생성
systemctl status labtest --no-pager | head -4
systemctl stop labtest                      # 중지하면 유닛 자동 소멸
systemd-run --on-active=1m --unit=labtimer /usr/local/bin/sysreport.sh   # 1분 뒤 1회 실행 (타이머 유닛)
systemctl list-timers labtimer* --no-pager
```

- `--unit=` : 유닛 이름 (생략 시 `run-<임의>.service`)
- `--on-active=` : 지정 시간 뒤 실행하는 일시 `.timer` 생성

**검증**

```bash
systemctl list-units 'lab*' --all --no-pager
```

```text
Running as unit: labtest.service
  UNIT                 LOAD   ACTIVE   SUB     DESCRIPTION
  lab-monitor.service  loaded active   running LAB system monitor ...
  labtimer.timer       loaded active   waiting /usr/local/bin/sysreport.sh
```

> 📝 **시험 포인트**: 일시 유닛은 `/run/systemd/transient/` 에 생성되어 재부팅 시 소멸.

---

## 5. journald

### 5-1. 조회 옵션 종합

> **상황**: 서비스 장애 조사에 쓰는 필터를 `sshd`·`lab-monitor` 로그로 한 번씩 훑는다.

```bash
journalctl -e                                  # 끝으로 이동 (end)
journalctl -n 50                               # 최근 50줄
journalctl -r | head                           # 역순 (최신 먼저)
journalctl -u sshd -n 20                       # 유닛
journalctl -u sshd -u lab-monitor --since today
journalctl -p err                              # err 이상 (emerg~err)
journalctl -p warning..err                     # 범위: warning, err
journalctl --since "1 hour ago" --until "10 min ago"
journalctl --since "2026-09-03 09:00" --until "2026-09-03 10:00"
journalctl _COMM=sshd -n 5                     # 프로세스 이름 필드
journalctl _PID=1 -n 5                         # PID 1 (systemd)
journalctl _UID=$(id -u dev1) -n 5             # dev1 이 남긴 로그
journalctl -k -n 10                            # 커널 (dmesg)
journalctl -u lab-monitor -o short-precise -n 3     # 마이크로초 시각
journalctl -u lab-monitor -o verbose -n 1           # 모든 필드
journalctl -u lab-monitor -o json -n 1 | head -c 300
journalctl -u lab-monitor -o json-pretty -n 1 | grep -E '"(_PID|_COMM|MESSAGE)"'
```

- `-e` : 페이저를 끝으로 (**e**nd), `-n` : 줄 수 (**n**umber), `-r` : 역순 (**r**everse), `-u` : 유닛 (**u**nit)
- `-p <lv>` / `-p a..b` : 우선순위 `emerg(0) alert crit err warning notice info debug(7)` — 단일 값은 "그 이상(심각)"
- `--since`/`--until` : `"YYYY-MM-DD HH:MM"`, `today`, `yesterday`, `"1 hour ago"` 허용
- `_COMM=`/`_PID=`/`_UID=` : 저널 메타 필드 필터 (`journalctl -F _COMM` 로 값 목록)
- `-k` : 커널 메시지 (**k**ernel), `-o` : 출력 형식 (**o**utput) `short`(기본) `short-precise` `verbose` `json` `json-pretty` `cat`

**검증**

```bash
journalctl -p err -b --no-pager | tail -3          # 이번 부팅 에러 (없으면 "-- No entries --")
journalctl -F _COMM | grep -c .                    # 필드 값 종류 수
```

```text
# journalctl -u lab-monitor -o short-precise -n 1
Sep 03 10:30:00.123456 srv01 lab-monitor.sh[...]: == 2026-09-03 10:30:00 load: ...
# journalctl -o json-pretty -n 1 -u lab-monitor | grep -E '"(_PID|_COMM)"'
        "_COMM" : "lab-monitor.sh",
        "_PID" : "...",
```

> 📝 **시험 포인트**: r04-62 `-r` 은 **역순 출력**(삭제 아님). r01-57·r02-58 `-u sshd -f` 실시간. r08-45 `--since today`. r10-37 `-b -p err --since "09:00"` = 이번 부팅·err 이상·09시 이후.

### 5-2. 디스크 사용량·정리·검증

> **상황**: 저널이 디스크를 얼마나 쓰는지 보고 정리 명령을 익힌다.

```bash
journalctl --disk-usage
journalctl --rotate                    # 현재 파일을 아카이브로 전환(새 파일 시작)
journalctl --vacuum-size=200M          # 아카이브 총량 200M 이하로 삭제
journalctl --vacuum-time=2weeks        # 2주 이전 아카이브 삭제
journalctl --verify                    # 파일 무결성 검사
```

- `--disk-usage` : 전체 저널 파일 크기
- `--rotate` : 활성 파일 아카이브 (vacuum 은 아카이브만 삭제하므로 선행 권장)
- `--vacuum-size=` / `--vacuum-time=` / `--vacuum-files=` : 아카이브 정리 기준
- `--verify` : `PASS`/`FAIL` 출력

**검증**

```bash
journalctl --disk-usage
ls /run/log/journal/*/ | head -3          # 휘발성 위치 (영구화 전)
```

```text
Archived and active journals take up 24.0M in the file system.
Vacuuming done, freed 0B of archived journals from /run/log/journal/...
PASS: /run/log/journal/.../system.journal
```

> 📝 **시험 포인트**: `--vacuum-*` 은 **아카이브**만 대상. 활성 파일까지 줄이려면 `--rotate` 후 실행.

### 5-3. 영구 저장 (Storage=persistent)

> **상황**: 재부팅하면 저널이 사라져 부팅 실패 원인을 못 본다. `/var/log/journal` 을 만들어 영구화하고 `--list-boots` 에 과거 부팅이 남는지 확인한다.

```bash
grep -E '^\[|Storage|SystemMaxUse' /etc/systemd/journald.conf        # 기본은 주석(#Storage=auto)
ls -d /var/log/journal 2>/dev/null || echo "없음 → 휘발성"
mkdir -p /var/log/journal
systemd-tmpfiles --create --prefix /var/log/journal                    # 소유권·ACL 정비
sed -i 's/^#\?Storage=.*/Storage=persistent/; s/^#\?SystemMaxUse=.*/SystemMaxUse=200M/' /etc/systemd/journald.conf
systemctl restart systemd-journald
```

| journald.conf 키 | 값 | 의미 |
| --- | --- | --- |
| `Storage=` | `auto`(기본) / `persistent` / `volatile` / `none` | `auto` = `/var/log/journal` 존재 시만 영구, `persistent` = 디렉터리 자동 생성 |
| `SystemMaxUse=` | `200M` | `/var/log/journal` 총 상한 (기본 FS 의 10%, 최대 4G) |
| `RuntimeMaxUse=` | | `/run/log/journal` 상한 |
| `MaxRetentionSec=` | `1month` | 보존 기간 |
| `ForwardToSyslog=` | `no`(기본, rsyslog 가 직접 읽음) | syslog 소켓 전달 |

- `systemd-tmpfiles --create --prefix` : `tmpfiles.d` 규칙대로 디렉터리 권한(`root:systemd-journal 2755`) 적용

**검증**

```bash
ls -ld /var/log/journal /var/log/journal/*/
journalctl --disk-usage                                  # 경로가 /var/log/journal 로
systemctl reboot                                         # 재부팅 후
journalctl --list-boots                                  # 2건 이상
journalctl -b -1 -n 3 --no-pager                         # 직전 부팅 로그 조회 가능
```

```text
drwxr-sr-x+ 3 root systemd-journal ... /var/log/journal
drwxr-sr-x+ 2 root systemd-journal ... /var/log/journal/<machine-id>/
Archived and active journals take up 32.0M in the file system.
# journalctl --list-boots
IDX BOOT ID   FIRST ENTRY                 LAST ENTRY
 -1 3f2a...   Wed 2026-09-03 09:00:01 KST Wed 2026-09-03 10:40:00 KST
  0 9c1d...   Wed 2026-09-03 10:41:10 KST Wed 2026-09-03 10:45:33 KST
```

> 📝 **시험 포인트**: 필기 r07-38 — 저널 영구 보존 = `/var/log/journal` 생성 또는 `Storage=persistent`. 기본 휘발 위치 `/run/log/journal`.

### 5-4. 일반 사용자 조회 권한 (systemd-journal 그룹)

> **상황**: 개발자 `dev1` 이 서버 로그를 보게 해 달라는 요청 — `systemd-journal` 그룹 보조 가입으로 해결한다.

```bash
su - dev1 -c 'journalctl -n 3 --no-pager'            # 자신의 로그만 + 안내문
usermod -aG systemd-journal dev1
su - dev1 -c 'journalctl -u sshd -n 3 --no-pager'    # 새 세션이므로 즉시 반영
```

- `usermod -aG` : 보조 그룹 추가 (**a**ppend **G**roups) — `-a` 없으면 기존 보조 그룹 대체

**검증**

```bash
id dev1 | grep -o 'systemd-journal'
getfacl /var/log/journal | grep group
```

```text
# su - dev1 -c 'journalctl -n 1'     (가입 전)
Hint: You are currently not seeing messages from other users and the system.
      Users in groups 'adm', 'systemd-journal', 'wheel' can see all messages.
...
systemd-journal
group:systemd-journal:r-x
```

> 📝 **시험 포인트**: 일반 사용자는 자기 UID 로그만. 전체 조회는 `adm`/`systemd-journal`/`wheel` 그룹. `/var/log/journal` 에 ACL(`+`) 이 붙어 있음.

---

## 6. rsyslog

### 6-1. 설정 구조와 facility·priority

> **상황**: 텍스트 로그(`/var/log/messages` 등)는 rsyslog 가 journald 에서 받아 기록한다. 규칙 문법을 기본 규칙으로 해석한다.

```bash
systemctl is-active rsyslog
grep -vE '^\s*(#|$)' /etc/rsyslog.conf          # 주석 제외 전체
ls /etc/rsyslog.d/
```

- 구조 3부
  - **MODULES** : `module(load="imuxsock")`(로컬 소켓), `module(load="imjournal" ...)`(journald 읽기), 주석된 `imudp`/`imtcp`(원격 수신)
  - **GLOBAL DIRECTIVES** : `global(workDirectory="/var/lib/rsyslog")`, `module(load="builtin:omfile" Template="RSYSLOG_TraditionalFileFormat")`, `include(file="/etc/rsyslog.d/*.conf" mode="optional")`
  - **RULES** : `facility.priority    action`

| facility | 발생원 | | priority(낮→높) | 숫자 |
| --- | --- | --- | --- | --- |
| `auth`/`authpriv` | 인증 (authpriv 는 비공개 파일) | | `debug` | 7 |
| `cron` | crond·at | | `info` | 6 |
| `daemon` | 기타 데몬 | | `notice` | 5 |
| `kern` | 커널 | | `warning`(`warn`) | 4 |
| `mail` | 메일 시스템 | | `err`(`error`) | 3 |
| `user` | 사용자 프로세스 (logger 기본) | | `crit` | 2 |
| `lpr`, `news`, `uucp`, `syslog` | 인쇄·뉴스·UUCP·rsyslog 자체 | | `alert` | 1 |
| `local0`~`local7` | 사용자 정의 (`local7` = boot.log) | | `emerg`(`panic`) | 0 |

| 선택자 문법 | 의미 |
| --- | --- |
| `mail.info` | info **이상**(info~emerg) |
| `mail.=info` | info **만** |
| `mail.!err` | err 이상 **제외** |
| `mail.none` | 해당 facility 전체 제외 |
| `*.info` | 모든 facility 의 info 이상 |
| `uucp,news.crit` | 여러 facility 를 `,` 로 |
| `*.info;mail.none` | 여러 선택자를 `;` 로 (뒤가 앞을 보정) |

| action | 의미 |
| --- | --- |
| `/var/log/messages` | 파일에 동기 기록 |
| `-/var/log/maillog` | `-` 비동기(버퍼) 기록 — 대량 로그용 |
| `@192.168.64.20:514` | 원격 **UDP** 전송 |
| `@@192.168.64.20:514` | 원격 **TCP** 전송 |
| `:omusrmsg:*` | 로그인한 모든 사용자 터미널에 출력 (구형 `*`) |
| `\|/path/fifo` | 명명 파이프 |
| `/dev/console` | 콘솔 장치 |

**검증**

```bash
grep -E '^\S+\.\S+\s+' /etc/rsyslog.conf          # 규칙 줄만
```

```text
*.info;mail.none;authpriv.none;cron.none                /var/log/messages
authpriv.*                                              /var/log/secure
mail.*                                                  -/var/log/maillog
cron.*                                                  /var/log/cron
*.emerg                                                 :omusrmsg:*
uucp,news.crit                                          /var/log/spooler
local7.*                                                /var/log/boot.log
```

- 기본 규칙 해석
  1. 모든 facility 의 info 이상, 단 mail·authpriv·cron 은 제외 → `messages`
  2. 인증 관련 전부 → `secure`
  3. 메일 전부 → `maillog` (비동기)
  4. cron 전부 → `cron`
  5. 긴급(emerg) 은 모든 로그인 사용자 화면에
  6. uucp·news 의 crit 이상 → `spooler`
  7. `local7` 전부 → `boot.log` (부팅 스크립트 출력)

> 📝 **시험 포인트**: 실기 r02-14 `mail.info`(이상) vs `mail.=info`(만). 실기 r05-12·필기 r01-56·r05-36·r05-37·r10-56 기본 규칙 해석. r10-42 priority 심각순 `emerg > alert > crit > err > warning > notice > info > debug`. r09-64 `@@` = TCP.

### 6-2. 커스텀 규칙 `local5` → `/var/log/lab.log`

> **상황**: Part 04 백업 스크립트와 감시 스크립트가 `logger -p local5.*` 로 남기는 메시지를 전용 파일에 모은다.

```bash
cat > /etc/rsyslog.d/lab.conf <<'EOF'
# LAB: local5 facility → 전용 파일, messages 에는 중복 기록 안 함
local5.*        /var/log/lab.log
& stop
EOF
rsyslogd -N1                          # 문법 검사 (레벨 1)
systemctl restart rsyslog
logger -p local5.info -t backup "test message from LAB 07"
logger -p local5.err  -t lab-monitor "simulated error"
logger -p authpriv.warning -t labtest "authpriv test → secure"
logger "plain logger → user.notice → messages"
```

- `& stop` : 앞 규칙에 매칭된 메시지의 후속 처리 중단 (`/var/log/messages` 중복 방지). 없으면 `*.info` 규칙에도 걸려 두 곳에 기록
- `rsyslogd -N1` : 설정 검사만 (**N** = config check level)
- `logger -p <facility.priority>` : 우선순위 지정 (**p**riority), `-t <tag>` : 태그 (**t**ag), 기본 `user.notice`

**검증**

```bash
tail -3 /var/log/lab.log
grep 'labtest' /var/log/secure | tail -1
grep 'plain logger' /var/log/messages | tail -1
grep -c 'LAB 07' /var/log/messages          # & stop 덕분에 0
```

```text
# rsyslogd -N1
rsyslogd: version 8.2xxx, config validation run (level 1), master config /etc/rsyslog.conf
rsyslogd: End of config validation run. Bye.
# tail -3 /var/log/lab.log
Sep  3 10:50:01 srv01 backup[...]: test message from LAB 07
Sep  3 10:50:02 srv01 lab-monitor[...]: simulated error
# grep labtest /var/log/secure
Sep  3 10:50:03 srv01 labtest[...]: authpriv test → secure
0
```

> 📝 **시험 포인트**: `/etc/rsyslog.d/*.conf` 는 `include` 로 합쳐짐 — 규칙 순서는 파일명 정렬. `logger` 로 rsyslog 규칙 테스트하는 방법이 실기 서술형 소재.

### 6-3. 원격 로그 전송 (※ 미실행)

> **상황**: 중앙 로그 서버 `192.168.64.20` 으로 모든 로그를 TCP 전송하는 설정 형식만 확인한다. 이 VM 은 단독이므로 실행하지 않음.

```bash
# 송신 측 (클라이언트) — /etc/rsyslog.d/remote.conf   ※ 미실행
# *.*  @@192.168.64.20:514            # TCP. UDP 는 @ 한 개
# authpriv.*  @@192.168.64.20:514      # 인증 로그만 전송하는 예 (r09-64)

# 수신 측 (서버) — /etc/rsyslog.conf 의 주석 해제   ※ 미실행
# module(load="imtcp")
# input(type="imtcp" port="514")
# + firewall-cmd --permanent --add-port=514/tcp && firewall-cmd --reload   (Part 10)
```

- `@` : UDP 514 (유실 가능, 경량), `@@` : TCP 514 (신뢰성)
- 수신 측은 `imudp`/`imtcp` 모듈 로드 + `input()` + 방화벽 포트 개방 필요

**검증** (설정 파일 존재 시 문법만)

```bash
rsyslogd -N1
ss -ulnp | grep 514 ; ss -tlnp | grep 514        # 수신 측이면 리스닝 확인 (여기서는 없음)
```

```text
(리스닝 없음 — 미실행)
```

> 📝 **시험 포인트**: `*.* @host` 한 줄이 원격 전송의 전부. `@`=UDP, `@@`=TCP 구분이 단골.

### 6-4. 주요 로그 파일과 바이너리 로그 조회

> **상황**: 침해 점검(Part 10) 전에 어떤 로그가 텍스트고 어떤 게 바이너리인지, 각각 어떤 명령으로 보는지 정리한다.

| 파일 | 형식 | 내용 | 조회 |
| --- | --- | --- | --- |
| `/var/log/messages` | 텍스트 | 일반 시스템 메시지 | `tail`, `grep` |
| `/var/log/secure` | 텍스트 | 인증 (ssh, su, sudo, login) | `grep 'Failed password'` |
| `/var/log/maillog` | 텍스트 | 메일 | `tail` |
| `/var/log/cron` | 텍스트 | cron/at 실행 | `tail` |
| `/var/log/boot.log` | 텍스트 | 부팅 스크립트 출력 (local7) | `cat` |
| `/var/log/dmesg` | 텍스트 | 부팅 시 커널 링 버퍼 스냅샷 (없을 수 있음 → `dmesg`) | `cat` |
| `/var/log/audit/audit.log` | 텍스트 | auditd·SELinux AVC | `ausearch`(Part 10) |
| `/var/log/httpd/`, `/var/log/samba/` | 텍스트 | 서비스별 (Part 09) | `tail` |
| `/var/log/wtmp` | **바이너리** | 로그인·로그아웃·재부팅 성공 이력 | `last` |
| `/var/log/btmp` | **바이너리** | 로그인 **실패** | `lastb` |
| `/var/log/lastlog` | **바이너리** | 계정별 마지막 로그인 | `lastlog` |
| `/run/utmp` | **바이너리** | 현재 로그인 세션 | `who`, `w` |
| `/var/log/journal/` | 바이너리 | systemd 저널 | `journalctl` |

```bash
last -n 5                             # wtmp 최근 5건
last -f /var/log/wtmp reboot          # 재부팅 이력만
last -x | head                        # 런레벨 변경·shutdown 포함
lastb -n 5                            # btmp (root 만 읽기 가능)
lastlog -u dev1                       # 특정 계정
lastlog -b 7                          # 7일 이전에 마지막 로그인한 계정
utmpdump /var/log/wtmp | tail -3      # 바이너리를 텍스트로 덤프
file /var/log/wtmp /var/log/messages  # 형식 확인
grep -c 'Failed password' /var/log/secure
```

- `last -n` : 건수, `-f <파일>` : 다른 wtmp 파일 (로테이트된 `wtmp-YYYYMMDD` 조회), `-x` : 시스템 종료·런레벨 항목 표시
- `lastb` : `last` 와 동일 인터페이스로 `btmp` 조회
- `lastlog -u` : 사용자 지정 (**u**ser), `-b <일>` : 지정 일수 **b**efore, `-t <일>` : 이내
- `utmpdump` : utmp/wtmp/btmp 레코드를 텍스트로 (`-r` 로 역변환)

**검증**

```bash
ssh dev1@localhost -o PreferredAuthentications=password true   # 틀린 비번 1회 입력 → btmp 기록
lastb -n 1
last -n 1 dev1
```

```text
# last -n 2
admin1   pts/0        192.168.64.1     Wed Sep  3 09:05   still logged in
reboot   system boot  5.14.0-...       Wed Sep  3 09:00   still running
# lastb -n 1
dev1     ssh:notty    localhost        Wed Sep  3 10:55 - 10:55  (00:00)
# lastlog -u dev1
Username         Port     From             Latest
dev1             pts/1    192.168.64.1     Wed Sep  3 10:52:11 +0900 2026
# file /var/log/wtmp
/var/log/wtmp: data
```

> 📝 **시험 포인트**: `wtmp`↔`last`, `btmp`↔`lastb`, `lastlog`↔`lastlog`, `utmp`↔`who` 매칭이 r01-58·r02-59·r03-62·r04-56·r05-63·r06-60·r07-63·r09-60 등 최다 출제. r03-62 ② "wtmp 는 텍스트" 가 틀린 선지. r07-37 바이너리 로그 = wtmp·btmp·lastlog. 실기 r03-1·r05-1 `grep -c 'Failed password' /var/log/secure`.

---

## 7. logrotate

### 7-1. 전역 설정과 실행 주체

> **상황**: `/var/log/lab.log` 가 무한히 커지지 않도록 순환 정책을 만들기 전에 전역 기본값과 실행 주체(RHEL 9 는 systemd 타이머)를 확인한다.

```bash
grep -vE '^\s*(#|$)' /etc/logrotate.conf
ls /etc/logrotate.d/
cat /etc/logrotate.d/rsyslog
systemctl list-timers logrotate.timer --no-pager     # RHEL 9: 타이머로 매일 실행
ls /etc/cron.daily/                                  # RHEL 8 이전엔 여기 logrotate 스크립트가 있었음
cat /var/lib/logrotate/logrotate.status | head -5    # 마지막 순환 시각 기록
```

| 지시자 | 의미 |
| --- | --- |
| `daily` / `weekly` / `monthly` / `yearly` | 순환 주기 |
| `rotate 4` | 이전 파일 4세대 보관 (5번째 순환 시 가장 오래된 것 삭제) |
| `create [모드 소유자 그룹]` | 순환 후 빈 새 파일 생성 |
| `dateext` | 접미 `-YYYYMMDD` (없으면 `.1`, `.2` 숫자) |
| `compress` / `nocompress` | gzip 압축 |
| `delaycompress` | 한 세대 늦게 압축 (데몬이 아직 쓰는 파일 보호) |
| `missingok` | 파일 없어도 오류 없이 진행 |
| `notifempty` | 빈 파일은 순환 안 함 |
| `size 10M` / `maxsize` / `minsize` | 크기 조건 |
| `copytruncate` | 복사 후 원본 비우기 (재오픈 못 하는 앱용) |
| `sharedscripts` | 여러 파일 매칭 시 postrotate 1회만 |
| `postrotate … endscript` | 순환 후 실행 (보통 데몬에 HUP) |
| `include /etc/logrotate.d` | 서비스별 설정 포함 |

**검증**

```bash
grep -E '^(weekly|rotate|create|dateext|compress|include)' /etc/logrotate.conf
```

```text
weekly
rotate 4
create
dateext
include /etc/logrotate.d
# systemctl list-timers logrotate.timer
NEXT                        LEFT     LAST                        PASSED  UNIT            ACTIVATES
Thu 2026-09-04 00:00:00 KST 13h left Wed 2026-09-03 00:00:07 KST 10h ago logrotate.timer logrotate.service
```

- Rocky 9 기본 `/etc/logrotate.conf` 는 `compress` 가 주석 처리 → 서비스별 설정에서 지정

> 📝 **시험 포인트**: r05-38 `rotate 4` = 4세대 보관 (주 4회·4MB 아님). r02-57 `missingok` = 없어도 오류 없음. r07-39 서비스별 설정 = `/etc/logrotate.d/`. 시험은 cron.daily 연동으로 묻지만 RHEL 9 실체는 `logrotate.timer`.

### 7-2. `/etc/logrotate.d/lab` 작성 → dry-run → 강제 실행

> **상황**: `lab.log` 를 매일 순환, 7일 보관, 압축(한 세대 지연) 하고, 순환 후 rsyslog 가 새 파일을 열도록 HUP 을 보낸다.

```bash
cat > /etc/logrotate.d/lab <<'EOF'
/var/log/lab.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root root
    postrotate
        /usr/bin/systemctl kill -s HUP rsyslog.service
    endscript
}
EOF
logrotate -d /etc/logrotate.d/lab             # dry-run: 무엇을 할지만 출력
logrotate -fv /etc/logrotate.d/lab            # 강제 1회 순환
logger -p local5.info -t backup "after rotate 1"
logrotate -fv /etc/logrotate.d/lab            # 2회째 → 1세대가 압축됨
```

- `-d` : 디버그·dry-run (**d**ebug) — 실제 변경 없음, `-v` 포함
- `-f` : 주기 무시하고 강제 순환 (**f**orce), `-v` : 상세 (**v**erbose)
- 설정 파일 하나만 인자로 주면 전역 `dateext` 가 적용되지 않아 `.1`, `.2.gz` 숫자 접미가 붙음 — `/etc/logrotate.conf` 를 인자로 주면 전체 로그가 순환되므로 주의
- `systemctl kill -s HUP` : 유닛의 프로세스에 시그널 (**s**ignal) — rsyslog 는 HUP 에 파일 재오픈

**검증**

```bash
ls -l /var/log/lab.log*
cat /var/log/lab.log                                 # 새 파일에 "after rotate 1" 이후 것만
grep lab.log /var/lib/logrotate/logrotate.status
logger -p local5.info -t backup "rsyslog reopened OK" && tail -1 /var/log/lab.log
```

```text
# logrotate -d /etc/logrotate.d/lab   (발췌)
reading config file /etc/logrotate.d/lab
Handling 1 logs
rotating pattern: /var/log/lab.log  forced from command line (7 rotations)
...
# ls -l /var/log/lab.log*
-rw-r-----. 1 root root   ... /var/log/lab.log
-rw-r-----. 1 root root   ... /var/log/lab.log.1
-rw-r-----. 1 root root   ... /var/log/lab.log.2.gz
"/var/log/lab.log" 2026-9-3-10:58:0
Sep  3 10:59:10 srv01 backup[...]: rsyslog reopened OK
```

> 📝 **시험 포인트**: r06-41·r09-45·r10-43 logrotate 블록 해석 — `delaycompress` 는 최신 1세대 미압축, `create 0640 root root` 권한, `postrotate` 로 데몬 재오픈. `sharedscripts` 는 와일드카드(`*.log`) 매칭 시 스크립트 1회.

---

## 8. 장애 복구 A — root 비밀번호 분실 (rd.break)

> **상황**: 인수인계 중 root 비밀번호를 아무도 모르는 상황을 가정한다. admin1 의 sudo 도 없다고 치고, 콘솔에서 GRUB 편집으로 재설정한다. 볼트 실검증 절차 [[../../BOOT-RECOVERY/rescue-mode]] 기준.

⚠️ UTM **콘솔 필수**. 2-2 에서 GRUB_TIMEOUT=10 으로 늘렸으므로 메뉴에서 여유 있게 `e` 입력 가능. 메뉴가 안 보이면 부팅 직후 `Esc` 또는 방향키 연타.

### 8-1. GRUB 편집 → rd.break 부팅

```text
① systemctl reboot
② GRUB 메뉴에서 첫 항목 선택 상태로 'e'
③ 'linux ($root)/vmlinuz-...' 로 시작하는 줄로 커서 이동 → End 키로 줄 끝
④ 공백 한 칸 두고 추가:  rd.break enforcing=0
   (기존 console=... 가 있으면 그대로 유지, initrd 줄보다 앞에 있어야 함)
⑤ Ctrl+x (또는 F10) 로 부팅
⑥ 몇 초 후  switch_root:/#  프롬프트 (비밀번호 없이 진입)
```

- `rd.break` : initramfs 가 실제 루트로 `switch_root` 하기 직전에 중단 (**r**am**d**isk break)
- `enforcing=0` : SELinux Permissive — 라벨 불일치로 인한 차단 방지 (볼트 검증 절차와 동일하게 병기)
- 줄이 화면에서 여러 줄로 감겨 보여도 실제로는 한 줄

### 8-2. sysroot 재마운트 → chroot → passwd → autorelabel

```bash
# switch_root:/#  프롬프트
mount | grep sysroot                # ro 로 마운트되어 있음
mount -o remount,rw /sysroot        # 쓰기 가능 전환 (필수)
chroot /sysroot                     # 설치 시스템으로 루트 전환 → sh-5.1# 
passwd root                         # 새 비밀번호 2회 입력
touch /.autorelabel                 # 다음 부팅에서 SELinux 전체 재레이블 예약 (필수)
exit                                # chroot 탈출
exit                                # initramfs 계속 → 부팅 진행
```

- `mount -o remount,rw` : 마운트 유지한 채 옵션만 변경 (**r**e**mount**, **r**ead-**w**rite)
- `chroot /sysroot` : 이후 명령이 실제 시스템의 `/etc/shadow` 를 수정 → [[../../BOOT-RECOVERY/chroot]]
- `/.autorelabel` : 변경된 `/etc/shadow` 의 SELinux 컨텍스트가 initramfs 에서는 정책 없이 기록되므로 재레이블 없으면 Enforcing 복귀 후 로그인 실패
- 재레이블은 파일 수에 따라 수 분 소요, 완료 후 자동 재부팅 1회 추가

**검증** (재부팅 완료 후 콘솔 또는 SSH)

```bash
# 콘솔 로그인: root / <새 비밀번호>
getenforce                              # Enforcing (재레이블 후 정상 복귀)
ls -Z /etc/shadow                       # system_u:object_r:shadow_t:s0
ls /.autorelabel                        # 없음 (소비됨)
journalctl -b -1 --no-pager | grep -iE 'relabel|switch_root' | head -3
passwd -S root
```

```text
# getenforce
Enforcing
# ls -Z /etc/shadow
system_u:object_r:shadow_t:s0 /etc/shadow
ls: cannot access '/.autorelabel': No such file or directory
# passwd -S root
root PS 2026-09-03 0 99999 7 -1 (Password set, SHA512 crypt.)
```

- `passwd -S` : 비밀번호 상태 한 줄 (**S**tatus) — 변경일이 오늘

### 8-3. 대안 — `init=/bin/bash` 와 차이

```text
GRUB 'e' → linux 줄 끝에  init=/bin/bash  (rd.break 대신) → Ctrl+x
bash-5.1# mount -o remount,rw /        # 이미 실제 루트 (chroot 불필요)
bash-5.1# passwd root
bash-5.1# touch /.autorelabel
bash-5.1# exec /sbin/init              # 또는 /sbin/reboot -f  (일반 reboot 은 PID1 이 systemd 가 아니라 동작 안 함)
```

| 구분 | `rd.break` | `init=/bin/bash` |
| --- | --- | --- |
| 중단 시점 | initramfs (루트 전환 전) | 커널이 실제 루트 마운트 후 PID 1 을 bash 로 |
| 루트 경로 | `/sysroot` → `chroot` 필요 | `/` 그대로 |
| SELinux 정책 | 미로드 → `enforcing=0`+autorelabel | 미로드 → autorelabel 필요 |
| 종료 | `exit` ×2 로 정상 부팅 이어감 | `exec /sbin/init` 또는 강제 재부팅 |
| 실패 사례 | LUKS 암호화 루트면 rd.break 후 추가 unlock | initramfs 가 루트를 못 찾으면 진입 불가 |

> 📝 **시험 포인트**: 절차 순서 "GRUB `e` → `rd.break` → `Ctrl+x` → `mount -o remount,rw /sysroot` → `chroot /sysroot` → `passwd` → `touch /.autorelabel` → `exit` ×2" 를 그대로 서술형으로 출제. r10-5 는 구형 `single`, r06-5 는 `systemd.unit=rescue.target`(비번 필요) — rd.break 만이 비번 없이 진입.

---

## 9. 장애 복구 B — /etc/fstab 오타 → emergency 모드

> **상황**: 운영자가 새 디스크를 fstab 에 등록하며 UUID 를 잘못 적어 재부팅 후 부팅이 멈추는 사고를 재현하고 복구한다.

### 9-1. 사전 검증 습관 → 고의로 오타 삽입

```bash
cp /etc/fstab /root/fstab.bak                # 백업
findmnt --verify                             # 현재 fstab 검증 (오류 0 확인)
mount -a                                     # 전 항목 마운트 시도 (조용하면 정상)
```

⚠️ 아래는 **의도적으로 부팅을 실패**시키는 항목. `nofail` 을 넣지 않아야 장애가 재현됨. 백업 확인 후 진행.

```bash
ls -l /root/fstab.bak
mkdir -p /mnt/bad
echo 'UUID=deadbeef-0000-4000-8000-000000000000 /mnt/bad xfs defaults 0 0' >> /etc/fstab
findmnt --verify                             # 경고로 미리 잡힘 → 실무에선 여기서 멈춰야 함
systemctl daemon-reload                      # fstab → .mount 유닛 재생성
systemctl reboot
```

- `findmnt --verify` : fstab 문법·장치 존재·FS 타입 검사 (`--verbose` 로 상세)
- `mount -a` : fstab 의 `noauto` 아닌 모든 항목 마운트 (**a**ll) — 오타를 재부팅 전에 잡는 습관
- `defaults 0 0` : 마운트 옵션 기본, dump 0, fsck 순서 0 — `nofail` 없으면 부팅 시 필수 마운트로 취급

```text
# findmnt --verify
/etc/fstab:12 [W] cannot detect on-disk filesystem type (source not found: UUID=deadbeef-...)
   0 parse errors, 0 errors, 1 warning
```

### 9-2. emergency 진입 관찰 → 복구

> **상황**: 재부팅 후 콘솔이 90초쯤 `A start job is running for /dev/disk/by-uuid/deadbeef…` 에서 대기한 뒤 emergency 셸을 준다.

```text
[콘솔 출력 요지]
[ TIME ] Timed out waiting for device /dev/disk/by-uuid/deadbeef-....
[DEPEND] Dependency failed for /mnt/bad.
[DEPEND] Dependency failed for Local File Systems.
You are in emergency mode. After logging in, type "journalctl -xb" to view
system logs, "systemctl reboot" to reboot, "systemctl default" or "exit"
to boot into default mode.
Give root password for maintenance
(or press Control-D to continue):
```

```bash
# root 비밀번호 입력 (8절에서 바꾼 값) → 프롬프트
journalctl -xb | grep -iE 'mnt-bad|Dependency failed' | head     # 원인 확인
systemctl --failed                                                # mnt-bad.mount, local-fs.target
mount | grep ' / '                                                # ro 이면 아래 실행
mount -o remount,rw /
vi /etc/fstab                     # 해당 줄 삭제, 또는 defaults → defaults,nofail (장치 없어도 부팅 진행)
findmnt --verify                  # 0 warning 확인
systemctl daemon-reload
mount -a
systemctl default                 # default.target 으로 계속 부팅 (= exit). 또는 systemctl reboot
```

- `journalctl -xb` : 이번 부팅 로그 + 설명문 (**x** = explain) — emergency 안내문이 지시하는 명령
- `nofail` : 장치 없어도 부팅 계속 (선택 마운트). 외장·네트워크 디스크는 항상 부여 권장
- `systemctl default` : `isolate default.target` — 재부팅 없이 정상 부팅 이어감

**검증**

```bash
systemctl is-system-running        # running (degraded 면 --failed 확인)
systemctl --failed
findmnt /data /srv/raid /srv/share  # Part 05 마운트 모두 정상
diff /etc/fstab /root/fstab.bak     # 오타 줄 제거되었는지
systemctl reboot && :               # 최종: 재부팅해서 emergency 없이 로그인 프롬프트까지 오는지
```

```text
# systemctl is-system-running
running
# systemctl --failed
0 loaded units listed.
# diff /etc/fstab /root/fstab.bak
(출력 없음 = 원복)   또는   < UUID=deadbeef-... /mnt/bad xfs defaults,nofail 0 0
```

> 📝 **시험 포인트**: r06-6 "fstab 오타 → `/` 읽기 전용 최소 셸" = `emergency.target`. 복구 3단계 `mount -o remount,rw /` → fstab 수정 → `systemctl default`. 예방은 `mount -a`/`findmnt --verify` 와 `nofail`.

---

## 10. 장애 복구 C — GRUB 손상 시 설치 매체 레스큐 (※ 미실행 가능)

> **상황**: 부트로더 자체가 손상돼 GRUB 메뉴가 뜨지 않는 경우. Rocky ISO 로 부팅해 설치 시스템에 `chroot` 한 뒤 부트로더를 다시 쓴다. 실습 VM 에서는 손상을 유발하지 않고 절차만 확인 (원하면 UTM 에 ISO 를 연결해 ①~④ 진입까지만 수행).

```text
① UTM → VM 설정 → Drives → Rocky ISO 를 CD/DVD 로 연결, 부팅 순서 앞으로
② 부팅 메뉴 → Troubleshooting → Rescue a Rocky Linux system
③ 1) Continue  → "Your system has been mounted under /mnt/sysroot." → Enter
④ sh-5.1# 프롬프트
```

```bash
lsblk                                        # 장치 확인 (ISO 는 RM=1 / sr0)
ls /sys/firmware/efi                         # 존재 → UEFI
chroot /mnt/sysroot                          # 설치 시스템 진입

# --- UEFI (이 VM, aarch64) ---
dnf reinstall -y grub2-efi-aa64 shim-aa64    # ESP 의 부트로더 파일 재배치
grub2-mkconfig -o /boot/grub2/grub.cfg
efibootmgr -v                                # 항목 없으면:
# efibootmgr -c -d /dev/vda -p 1 -L "Rocky Linux" -l '\EFI\rocky\shimaa64.efi'

# --- x86_64 BIOS 라면 ---
# grub2-install /dev/vda
# grub2-mkconfig -o /boot/grub2/grub.cfg

exit
reboot                                       # ISO 분리 후
```

- `dnf reinstall grub2-efi-aa64 shim-aa64` : x86_64 는 `grub2-efi-x64 shim-x64`. UEFI 에서는 `grub2-install` 을 쓰지 않음 (BIOS 전용) → [[../../BOOT-RECOVERY/grub2-install]]
- `efibootmgr -c -d <디스크> -p <ESP 파티션번호> -L <라벨> -l <로더 경로>` : 부팅 항목 생성 (**c**reate, **d**isk, **p**artition, **L**abel, **l**oader)
- 레스큐 환경은 ISO 에 계속 의존 → 작업 중 분리 금지. 첫 부팅에서 autorelabel 가능 → [[../../BOOT-RECOVERY/rescue-mode]]

- 부트로더는 살아 있고 시스템만 점검하려면 GRUB `e` → `systemd.unit=rescue.target` (root 비번 필요) 또는 `systemd.unit=emergency.target` 이 더 빠름

**검증** (재부팅 후)

```bash
ls /boot/efi/EFI/rocky/grubaa64.efi /boot/efi/EFI/rocky/shimaa64.efi
efibootmgr | head -3
grubby --default-kernel
```

```text
/boot/efi/EFI/rocky/grubaa64.efi  /boot/efi/EFI/rocky/shimaa64.efi
BootCurrent: 0001
BootOrder: 0001,...
```

> 📝 **시험 포인트**: r06-4 멀티부팅 GRUB 복구 = 라이브 미디어 → `chroot` → `grub2-install /dev/sda` + `grub2-mkconfig`. 레스큐 마운트 경로 `/mnt/sysroot`(설치 매체) vs `/sysroot`(rd.break) 구분.

---

## 11. 마무리 점검

> **상황**: 실습 중 만든 장애가 모두 해소되었는지 확인하고, 상시 감시 서비스는 자원 절약을 위해 멈추되 유닛은 남겨 Part 12 에서 재사용한다.

```bash
systemctl --failed                              # 0 이어야 함
systemctl is-system-running
journalctl -p err -b --no-pager | tail -20      # 이번 부팅 에러 검토
systemctl disable --now lab-monitor             # 중지 + 자동시작 해제 (유닛 파일은 유지)
systemctl is-enabled lab-monitor                # disabled
ls /etc/systemd/system/lab-monitor.service /etc/systemd/system/lab-monitor.service.d/override.conf
systemctl is-active chronyd sshd rsyslog systemd-journald
grep -c . /var/log/lab.log; ls /var/log/lab.log*
findmnt --verify | tail -1
```

- `disable --now` : `enable --now` 의 반대 — stop + wants 링크 제거. 유닛 파일·드롭인은 그대로

**검증**

```bash
systemctl status lab-monitor --no-pager | head -3
```

```text
○ lab-monitor.service - LAB system monitor (sysreport every 60s)
     Loaded: loaded (/etc/systemd/system/lab-monitor.service; disabled; preset: disabled)
    Drop-In: /etc/systemd/system/lab-monitor.service.d
     Active: inactive (dead) since ...
# systemctl --failed
0 loaded units listed.
# findmnt --verify | tail -1
   0 parse errors, 0 errors, 0 warnings
```

> 📝 **시험 포인트**: `disable` 후에도 `systemctl start` 는 가능(mask 와의 차이). `is-system-running` 의 `degraded` = failed 유닛 존재.

---

## 체크리스트

| 수행 항목 | 명령 | 확인 방법 | ☐ |
| --- | --- | --- | --- |
| UEFI·ESP·BLS 구조 확인 | `ls /boot/efi/EFI/rocky/`, `efibootmgr -v`, `ls /boot/loader/entries/` | 파일 목록 출력 | ☐ |
| 커널 파라미터·initramfs 확인 | `cat /proc/cmdline`, `lsinitrd \| head` | `root=/dev/mapper/rl-root` 확인 | ☐ |
| 부팅 로그·분석 | `journalctl -b`, `--list-boots`, `systemd-analyze blame/critical-chain/plot` | `/root/boot.svg` 생성 | ☐ |
| GRUB_TIMEOUT=10 반영 | `/etc/default/grub` → `grub2-mkconfig -o /boot/grub2/grub.cfg` | `grep 'set timeout' /boot/grub2/grub.cfg`, 재부팅 메뉴 10초 | ☐ |
| grubby 로 파라미터 제거 | `grubby --update-kernel=ALL --remove-args="rhgb quiet"` | `grubby --info=DEFAULT`, 재부팅 후 `/proc/cmdline` | ☐ |
| grubenv·기본 항목 | `grub2-editenv list`, `grubby --set-default` | `saved_entry` 값 | ☐ |
| 유닛 목록·상태 조회 | `list-units --type=service --state=running`, `--failed`, `list-unit-files` | 출력 확인 | ☐ |
| start/stop/restart/reload/mask | `systemctl mask chronyd` → `start` 실패 → `unmask` | `ls -l /etc/systemd/system/chronyd.service` → `/dev/null` | ☐ |
| 런레벨↔타겟 링크 | `ls -l /usr/lib/systemd/system/runlevel?.target` | 7개 링크 | ☐ |
| 기본 타겟 | `systemctl get-default` / `set-default multi-user.target` | `readlink -f /etc/systemd/system/default.target` | ☐ |
| rescue 진입·복귀 (콘솔) | `systemctl isolate rescue.target` → `isolate multi-user.target` | `is-active sshd` active | ☐ |
| shutdown 예약·취소 | `shutdown -r +5 "msg"` → `shutdown -c` | `/run/nologin` 생성·삭제 | ☐ |
| SysV 호환 | `chkconfig --list`, `service chronyd status` | `Redirecting to /bin/systemctl` | ☐ |
| lab-monitor.sh 작성 | `/usr/local/bin/lab-monitor.sh` | `bash -n`, `timeout 3` 실행 | ☐ |
| lab-monitor.service 등록 | `daemon-reload` → `enable --now` | `is-active`, `is-enabled`, wants 링크 | ☐ |
| Restart 검증 | `kill -9 $(systemctl show -p MainPID --value lab-monitor)` | `show -p NRestarts` = 1 | ☐ |
| 드롭인 오버라이드 | `lab-monitor.service.d/override.conf` | `systemctl cat`, `show -p RestartUSec` = 10s | ☐ |
| journald 조회 옵션 | `-u -b -p -k --since _COMM= -o json` | 각 출력 | ☐ |
| journald 영구화 | `mkdir /var/log/journal` + `Storage=persistent` + restart | 재부팅 후 `--list-boots` 2건 | ☐ |
| journald 정리·검증 | `--disk-usage`, `--rotate`, `--vacuum-size=200M`, `--verify` | `PASS` | ☐ |
| dev1 저널 권한 | `usermod -aG systemd-journal dev1` | `su - dev1 -c 'journalctl -u sshd -n 3'` | ☐ |
| rsyslog 규칙 추가 | `/etc/rsyslog.d/lab.conf` `local5.* /var/log/lab.log` → `rsyslogd -N1` → restart | `logger -p local5.info -t backup` → `tail /var/log/lab.log` | ☐ |
| authpriv → secure | `logger -p authpriv.warning -t labtest` | `grep labtest /var/log/secure` | ☐ |
| 바이너리 로그 조회 | `last -f /var/log/wtmp`, `lastb`, `lastlog -u dev1`, `utmpdump` | 출력 확인 | ☐ |
| logrotate 정책 | `/etc/logrotate.d/lab` → `logrotate -d` → `logrotate -fv` ×2 | `lab.log`, `lab.log.1`, `lab.log.2.gz` | ☐ |
| logrotate 실행 주체 | `systemctl list-timers logrotate.timer`, `/var/lib/logrotate/logrotate.status` | 타이머 NEXT 표시 | ☐ |
| 복구 A: rd.break | GRUB `e` → `rd.break enforcing=0` → remount,rw → chroot → passwd → `/.autorelabel` | 새 비번 로그인, `getenforce` Enforcing | ☐ |
| 복구 B: fstab 오타 | 잘못된 UUID 추가 → emergency → remount,rw → fstab 수정 → `systemctl default` | `systemctl --failed` 0, 재부팅 정상 | ☐ |
| 복구 C: 레스큐 절차 (※) | ISO → Rescue → `chroot /mnt/sysroot` → `dnf reinstall grub2-efi-aa64 shim-aa64` | 절차 숙지 (미실행 가능) | ☐ |
| 마무리 | `systemctl disable --now lab-monitor`, `journalctl -p err -b` | `is-enabled` disabled, 유닛 파일 잔존 | ☐ |

---

## 기출 연결

| 기출 (문제집·회차·번호 또는 주제) | 이 파트의 단계 |
| --- | --- |
| 필기 FULL r01-6, r02-5, r07-1 — 부팅 순서 나열 | 1-1 부팅 단계 정리표 |
| 필기 FULL r07-2 — 타겟 도달 순서 sysinit→basic→multi-user→graphical | 3-4 `list-dependencies graphical.target` |
| 필기 FULL r03-6 — `systemd-analyze` 후 지연 유닛 내림차순 → `blame` | 1-4 |
| 필기 FULL r01-8, r03-5, r04-6, r05-5, r05-6, r07-16 — `/etc/default/grub`, `grub2-mkconfig`, GRUB_TIMEOUT, 반영 절차 | 2-1, 2-2 |
| 필기 FULL r04-5 — GRUB vs LILO | 2-1 (GRUB2 설정·BLS 구조) |
| 필기 FULL r09-5 — `grub2-setpassword` | 2-4 |
| 필기 FULL r06-4 — 멀티부팅 GRUB 복구 (`chroot` + `grub2-install` + `grub2-mkconfig`) | 10 |
| 필기 FULL r06-5 — `systemd.unit=rescue.target`; r10-5 — GRUB `e` → `single` → `Ctrl+x` | 2-5 파라미터표, 8-1, 10 |
| 필기 FULL r01-7, r02-6, r03-18, r04-7, r05-7, r07-15 — 런레벨↔타겟 | 3-4 |
| 필기 FULL r10-6 — systemd vs SysV init 비교; r10-7 — `/etc/inittab` `id:3:initdefault:` 의미 | 3-4 (`set-default` 대응), 3-7 SysV 호환 명령 |
| 필기 FULL r04-8, r08-9 — `runlevel` 출력 해석 | 3-4 `runlevel`, `who -r` |
| 필기 FULL r06-6 — fstab 오타 → `emergency.target` | 3-5 비교표, 9 |
| 필기 FULL r03-29, r05-42 — 유닛 파일 `After`/`WantedBy`/`Restart=on-failure` | 4-2 |
| 필기 FULL r03-30, r06-38, r10-36 — `mask` vs `disable` | 3-3 |
| 필기 FULL r06-37, r08-44 — `enable --now` | 3-3, 4-2 |
| 필기 FULL r01-57, r02-58 — `journalctl -u sshd -f` | 4-3, 5-1 |
| 필기 FULL r06-43 — `journalctl -u sshd -b`; r08-45 — `--since today`; r10-37 — `-b -p err --since` | 5-1 |
| 필기 FULL r04-62 — `journalctl -r` 는 역순(삭제 아님) | 5-1 |
| 필기 FULL r07-38 — 저널 영구 보존 `/var/log/journal`, `Storage=persistent` | 5-3 |
| 필기 FULL r01-56, r02-56, r05-36, r05-37, r10-56 — rsyslog 기본 규칙 해석 (`authpriv.*`, `mail.none`, `mail.warn`) | 6-1 |
| 필기 FULL r10-42 — priority 심각순 나열 | 6-1 priority 표 |
| 필기 FULL r09-64 — `authpriv.* @@host:514` TCP 원격 전송 | 6-3 |
| 필기 FULL r02-57, r05-38, r06-41, r07-39, r09-45, r10-43 — logrotate 지시자 해석 (`rotate 4`, `missingok`, `notifempty`, `/etc/logrotate.d/`) | 7-1, 7-2 |
| 필기 FULL r01-58, r02-59, r03-62, r03-63, r04-56, r05-63, r06-60, r07-37, r07-63, r08-56, r09-60 — wtmp/btmp/lastlog ↔ last/lastb/lastlog, 바이너리 로그 | 6-4 |
| 실기 r01-8 — `systemctl enable httpd` | 3-3 |
| 실기 r02-14 — `mail.info` vs `mail.=info` | 6-1 선택자 문법표 |
| 실기 r05-12 — `authpriv.* /var/log/secure`, `*.info;mail.none;authpriv.none /var/log/messages` 해석 | 6-1 기본 규칙 해석 |
| 실기 r03-1, r05-1 — `grep -c 'Failed password' /var/log/secure` | 6-4 |
| 실기 r04-8 — `/var/log/app.log` `chattr +a` (로그 변조 방지) | 6-4 로그 파일 표 (Part 03 chattr 참조) |
| 주제 — root 비밀번호 복구 절차 서술 (rd.break → remount → chroot → passwd → autorelabel) | 8 |

---

## 이전 / 다음

[[06-process-scheduling-diagnosis]] ← · → [[08-network-config]]

[[README]]
