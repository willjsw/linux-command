---
title: LAB 05 — 디스크·LVM·RAID·스왑·쿼터·커널 모듈
type: exam-lab
part: 05
tags:
  - exam/linux-master
  - exam/lab
  - linux/disk
  - linux/filesystem
  - linux/kernel
  - task/configure
  - task/verify
related: ["[[README]]", "[[04-file-text-shell]]", "[[06-process-scheduling-diagnosis]]", "[[../THEORY/disk-device]]"]
distro: Rocky Linux 9 (aarch64, UTM)
updated: 2026-09-03
---

# LAB 05 — 디스크·LVM·RAID·스왑·쿼터·커널 모듈

- UTM 에 붙인 추가 디스크 4개(`vdb` `vdc` `vdd` `vde`)를 **데이터 파티션(ext4+쿼터)·스왑(파티션+파일)·RAID 1 백업 영역·LVM 공유 영역**으로 구성
- 파티션(`fdisk`) → 파일시스템(`mkfs`) → 마운트(`mount`) → 영구 등록(`/etc/fstab`) 4단계를 자원마다 반복 — **만들었으면 `lsblk -f`·`findmnt`·`df -hT` 로 확인**
- LVM 온라인 확장·스냅샷 복구, RAID 장애 시뮬레이션, 쿼터 초과 검증, 커널 모듈 적재·제거까지 실제 동작으로 확인
- 마지막에 `reboot` 후 전부 살아있는지 검증 스크립트로 일괄 점검

> **이 파트의 시나리오**: Part 03 에서 계정(`dev1` `dev2` `ops1`, 그룹 `devteam`)이, Part 04 에서 셸 스크립트 작성 능력이 준비됐다. 이제 서버에 꽂힌 빈 디스크 4개를 용도별로 나눈다 — 개발팀 데이터는 `/data`(ext4, 사용자별 쿼터), 백업 대상지는 RAID 1 `/srv/raid`, Samba 공유(Part 09)는 LVM `/srv/share` 로 두어 나중에 무중단 확장할 수 있게 한다. 메모리 부족 대비 스왑도 파티션·파일 두 방식으로 추가한다. 전부 fstab 에 등록해 재부팅 후에도 유지되는지 검증한다.

---

## 0. 사전 준비

### 0-1. 스냅샷·패키지 설치

> **상황**: 디스크 작업은 되돌리기 어렵다. UTM(QEMU 백엔드)이면 스냅샷을 먼저 찍고, 이 파트에서 쓸 도구를 한 번에 설치한다.

```bash
dnf -y install mdadm quota parted lvm2 xfsprogs e2fsprogs \
               genisoimage pciutils usbutils lshw hdparm sg3_utils smartmontools
rpm -q mdadm quota parted lvm2 xfsprogs e2fsprogs      # 설치 확인
```

- `mdadm` : 소프트웨어 RAID(md) 관리
- `quota` : `quotacheck` `quotaon` `edquota` `repquota` 제공
- `parted` : GPT 파티션 도구 + `partprobe` 제공
- `lvm2` : `pvcreate` `vgcreate` `lvcreate` 계열
- `xfsprogs` / `e2fsprogs` : xfs / ext 계열 mkfs·repair·tune 도구
- `genisoimage` : ISO 이미지 생성(`mkisofs` 호환) — loop 마운트 실습용
- `pciutils` `usbutils` `lshw` : `lspci` `lsusb` `lshw`
- `hdparm` `sg3_utils` `smartmontools` : 디스크 파라미터·SCSI 질의·SMART (가상 디스크에서는 대부분 ※ 참고)

**검증**
```bash
which fdisk mkfs.ext4 mkfs.xfs mdadm pvcreate quotacheck partprobe
```

```text
/usr/sbin/fdisk
/usr/sbin/mkfs.ext4
/usr/sbin/mkfs.xfs
/usr/sbin/mdadm
/usr/sbin/pvcreate
/usr/sbin/quotacheck
/usr/sbin/partprobe
```

> 📝 **시험 포인트**: 필기에서 "소프트웨어 RAID 관리 도구 → `mdadm`", "쿼터 도구 패키지 → `quota`" 처럼 도구↔패키지 매칭 출제. `partprobe` 가 `parted` 패키지에 들어 있다는 점도 실무 함정.

---

## 1. 디스크 인식 확인

### 1-1. 블록 장치 트리 — lsblk

> **상황**: UTM 에 추가한 디스크 4개가 커널에 `vdb`~`vde` 로 잡혔는지, 아직 파티션·파일시스템이 없는 빈 디스크인지 확인한다.

```bash
lsblk                                                    # 트리 형태
lsblk -f                                                 # 파일시스템·LABEL·UUID·마운트
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,UUID           # 컬럼 지정
lsblk -p /dev/vdb                                        # 전체 경로 표기
```

- `-f` : 파일시스템 정보 표시 (**f**ilesystem — FSTYPE, LABEL, UUID, MOUNTPOINTS)
- `-o <컬럼>` : 출력 컬럼 지정 (**o**utput — `lsblk -h` 로 컬럼 목록)
- `-p` : `/dev/` 포함 전체 경로 표시 (**p**aths)

**검증**
```bash
lsblk -d -o NAME,SIZE,TYPE,MODEL | grep -E '^vd[b-e]'
```

```text
vdb   5G disk
vdc   5G disk
vdd   5G disk
vde   2G disk
```

- 기대 트리 (시스템 디스크 `vda` 는 설치 시 자동 파티션)

```text
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda         252:0    0   40G  0 disk
├─vda1      252:1    0  600M  0 part /boot/efi
├─vda2      252:2    0    1G  0 part /boot
└─vda3      252:3    0 38.4G  0 part
  ├─rl-root 253:0    0  ...   0 lvm  /
  └─rl-swap 253:1    0  ...   0 lvm  [SWAP]
vdb         252:16   0    5G  0 disk
vdc         252:32   0    5G  0 disk
vdd         252:48   0    5G  0 disk
vde         252:64   0    2G  0 disk
```

> 📝 **시험 포인트**: `lsblk` 출력 해석 문제(필기 R08-47) — `TYPE` 의 `disk`/`part`/`lvm`, `[SWAP]` 표기, `MAJ:MIN` 252=virtio 블록·253=device-mapper·8=SCSI(sd) 구분.

### 1-2. 파티션 테이블·장치 파일 — fdisk -l, /proc/partitions, /dev

> **상황**: `lsblk` 와 다른 관점(커널 파티션 표, 장치 파일, 영구 식별자 링크)으로 같은 사실을 교차 확인한다.

```bash
fdisk -l /dev/vdb                        # 파티션 테이블 (빈 디스크면 Disklabel 없음)
fdisk -l | grep -E '^Disk /dev/vd'       # 전체 디스크 요약
cat /proc/partitions                     # 커널이 인식한 블록 장치·파티션 표
ls -l /dev/vd*                           # 장치 파일 (b = 블록 장치, major 252)
ls -l /dev/disk/by-id/   | grep vd       # 하드웨어 ID 기반 링크 (virtio 는 serial 있을 때만)
ls -l /dev/disk/by-path/ | grep vd       # PCI 경로 기반 링크
ls -l /dev/disk/by-uuid/                 # 파일시스템 UUID 링크 (아직 vdb~vde 없음)
```

- `fdisk -l` : 파티션 테이블 목록 출력 (**l**ist)
- `/proc/partitions` : major, minor, #blocks(1 KiB 단위), name — 커널 인식 원본
- `/dev/disk/by-id|by-path|by-uuid|by-label` : udev 가 만드는 **영구 이름** 심볼릭 링크

**검증**
```bash
grep -E 'vd[b-e]' /proc/partitions
ls -l /dev/vdb
```

```text
 252       16    5242880 vdb
 252       32    5242880 vdc
 252       48    5242880 vdd
 252       64    2097152 vde
brw-rw----. 1 root disk 252, 16 ... /dev/vdb
```

> 📝 **시험 포인트**: `fdisk -l` 출력 해석(R08-46) — `Disklabel type: dos|gpt`, `Type` 컬럼 `Linux LVM`/`Linux swap`. `/proc/partitions` 의 `#blocks` 는 **1 KiB 블록** 단위 → 5242880 = 5 GiB.

### 1-3. 커널 메시지·UUID·udev 속성 — dmesg, blkid, udevadm

> **상황**: 디스크가 부팅 시 어떤 드라이버로 붙었는지(virtio_blk), 파일시스템 서명이 남아 있는지, udev 가 어떤 속성으로 인식하는지 본다.

```bash
dmesg | grep -E 'vd[a-e]'                       # 커널 인식 로그
blkid                                           # 파일시스템 서명 있는 장치만 출력 → vdb~vde 는 없음
blkid /dev/vda2                                 # 특정 장치 UUID·TYPE
udevadm info --query=all -n /dev/vdb | head -20 # udev 데이터베이스 속성
udevadm info --query=property -n /dev/vdb | grep -E 'DEVNAME|DEVTYPE|ID_PATH|MAJOR|MINOR'
```

- `dmesg` : 커널 링 버퍼 출력 (**d**iagnostic **mes**sa**g**e)
- `blkid` : 블록 장치 UUID·LABEL·TYPE 조회 (**bl**oc**k** **id**) — 서명 없는 빈 디스크는 표시되지 않음
- `udevadm info` : udev 장치 정보 (**udev** **adm**inistration)
  - `--query=all|property|path|name` : 조회 항목 (**query**)
  - `-n <장치>` : 장치 노드 지정 (**n**ame)

**검증**
```bash
dmesg | grep -c 'virtio_blk'
blkid | grep -c vdb          # 0 이어야 빈 디스크
```

```text
[    1.2...] virtio_blk virtio1: [vda] 83886080 512-byte logical blocks (42.9 GB/40.0 GiB)
[    1.2...] virtio_blk virtio2: [vdb] 10485760 512-byte logical blocks (5.37 GB/5.00 GiB)
...
P: /devices/pci0000:00/.../block/vdb
N: vdb
E: DEVNAME=/dev/vdb
E: DEVTYPE=disk
E: ID_PATH=pci-0000:00:0...
```

> 📝 **시험 포인트**: "블록 장치 UUID·파일시스템 유형 확인 명령 → `blkid`"(R04-55, R05-52, R07-55). udev 는 장치 **동적** 인식·`/dev` 생성, 규칙은 `/etc/udev/rules.d/`(R03-52, R06-52, R09-49).

### 1-4. 장치명 규칙표

| 인터페이스·유형 | 장치명 | 파티션 표기 | 비고 |
| --- | --- | --- | --- |
| SATA / SCSI / SAS / USB | `/dev/sda` `sdb` … | `sda1` `sda2` | 인식 순서대로 알파벳 — USB 꽂으면 순서 밀림 → UUID 권장 |
| virtio (KVM·UTM) | `/dev/vda` `vdb` … | `vdb1` | 이 실습 환경. major 252 |
| NVMe | `/dev/nvme0n1` | `nvme0n1p1` | `nvme<컨트롤러>n<네임스페이스>p<파티션>` |
| IDE(구형) | `/dev/hda` `hdb` | `hda1` | 현재 커널은 libata 로 `sd*` 통합 |
| SD/eMMC | `/dev/mmcblk0` | `mmcblk0p1` | 임베디드 |
| 소프트웨어 RAID | `/dev/md0` `md127` | `md0p1` | mdadm.conf 미등록 시 재부팅 후 `md127` |
| LVM | `/dev/<VG>/<LV>` = `/dev/mapper/<VG>-<LV>` = `/dev/dm-N` | — | device-mapper, major 253 |
| 루프 | `/dev/loop0` | `loop0p1` | 파일을 블록 장치로 |
| 광학 | `/dev/sr0` (`/dev/cdrom` 링크) | — | ISO9660 |

- MBR 파티션 번호: **1~4 = 주(primary)·확장(extended)**, **논리(logical) = 5번부터** (주 파티션이 2개뿐이어도 논리는 5부터)
- GPT: 1~128 순차, 주/확장/논리 구분 없음

> 📝 **시험 포인트**: "`/dev/sda5` 는 다섯 번째 주 파티션" → **틀림**(첫 번째 논리 파티션). `nvme0n1p1` 의 `p` 의미, `vda` = virtio(R02-46, R10-52).

---

## 2. 파티션 — fdisk (MBR) / parted (GPT)

### 2-1. fdisk 키 시퀀스표·MBR vs GPT

| fdisk 키 | 원어 | 동작 |
| --- | --- | --- |
| `m` | **m**enu | 도움말 |
| `p` | **p**rint | 파티션 테이블 출력 |
| `n` | **n**ew | 새 파티션 → `p`(**p**rimary) / `e`(**e**xtended) / 논리는 확장 안에서 `l`(**l**ogical) |
| `d` | **d**elete | 파티션 삭제 |
| `t` | **t**ype | 파티션 타입(hex) 변경 — `82` swap, `83` Linux, `8e` LVM, `fd` RAID autodetect |
| `l` (t 안에서 `L`) | **l**ist | 타입 코드 목록 |
| `F` | **F**ree | 미할당 공간 표시 |
| `a` | **a**ctive | 부트 플래그 토글 |
| `v` | **v**erify | 테이블 검증 |
| `g` | **g**pt | 새 **GPT** 테이블 생성 |
| `o` | d**o**s | 새 **MBR(DOS)** 테이블 생성 |
| `w` | **w**rite | 저장 후 종료 — 이때만 디스크 기록 |
| `q` | **q**uit | 저장 없이 종료 |

| 구분 | MBR (msdos) | GPT |
| --- | --- | --- |
| 최대 디스크 | **2 TiB** | 8 ZiB (사실상 무제한) |
| 파티션 수 | 주 최대 **4** (주 3 + 확장 1 → 논리 다수) | 기본 **128** |
| 테이블 위치·백업 | 선두 512 B 1개, 백업 없음 | 선두 + **말미 이중화**, CRC 검사 |
| 호환 | BIOS 부팅 | UEFI 부팅, 선두에 **보호 MBR**(protective MBR) 두어 구형 도구가 빈 디스크로 오인 안 함 |
| 도구 | `fdisk`(`o`), `parted mklabel msdos` | `fdisk`(`g`), `gdisk`, `parted mklabel gpt` |

> 📝 **시험 포인트**: `n → t → w` 순서(R04-31), 키 의미(R01-46). MBR 2TB·4개 한계와 GPT 128개·보호 MBR 은 매회 출제.

### 2-2. vdb 를 MBR 로 주 파티션 3개 생성

> **상황**: `/dev/vdb`(5 GB)를 데이터 2G / 스왑 1G / LVM 확장용 나머지 로 나눈다. MBR 방식으로 `fdisk` 대화식 진행 — 키 입력을 그대로 따라간다.

⚠️ 대상 디스크 재확인 — `vda` 는 시스템 디스크. 반드시 `vdb` 인지 `lsblk` 로 확인 후 진입
```bash
lsblk -d -o NAME,SIZE /dev/vdb            # 5G, 빈 디스크 확인
fdisk /dev/vdb
```

- 대화식 입력 (왼쪽 = 프롬프트, 오른쪽 = 입력값)

```text
Command (m for help): o                       ← 새 MBR(DOS) 테이블
Command (m for help): n                       ← 새 파티션
Partition type: p                             ← primary
Partition number (1-4, default 1): [Enter]
First sector (2048-..., default 2048): [Enter]
Last sector, +/-sectors or +/-size{K,M,G,T,P}: +2G

Command (m for help): n
Partition type: p
Partition number (2-4, default 2): [Enter]
First sector: [Enter]
Last sector: +1G

Command (m for help): n
Partition type: p
Partition number (3,4, default 3): [Enter]
First sector: [Enter]
Last sector: [Enter]                          ← 나머지 전체

Command (m for help): p                       ← 확인 (아직 미저장)
```

- `o` : 빈 DOS(MBR) 파티션 테이블 생성 — 빈 디스크면 자동 생성되므로 생략 가능
- `n` → `p` → 번호 → 시작 섹터(기본 2048 = 1 MiB 정렬) → 끝(`+2G` 크기 지정)
- 3번째 파티션 끝은 기본값(마지막 섹터) → 나머지 전부

**검증** (fdisk 안에서 `p`)
```text
Device     Boot   Start      End  Sectors Size Id Type
/dev/vdb1          2048  4196351  4194304   2G 83 Linux
/dev/vdb2       4196352  6293503  2097152   1G 83 Linux
/dev/vdb3       6293504 10485759  4192256   2G 83 Linux
```

> 📝 **시험 포인트**: 세 파티션 모두 기본 타입 `83 Linux` — 실기에서 "스왑 파티션 만들기" 는 `n` 뒤 **`t` 로 82 변경**이 채점 포인트.

### 2-3. 타입 변경(82·8e) → 저장 → 커널 반영

> **상황**: 아직 fdisk 안에 있다. vdb2 는 스왑(`82`), vdb3 은 LVM(`8e`) 으로 타입을 바꾸고 `w` 로 저장한다.

```text
Command (m for help): t
Partition number (1-3, default 3): 2
Hex code or alias (type L to list all): 82
Changed type of partition 'Linux' to 'Linux swap / Solaris'.

Command (m for help): t
Partition number (1-3, default 3): 3
Hex code or alias (type L to list all): 8e
Changed type of partition 'Linux' to 'Linux LVM'.

Command (m for help): p                       ← 최종 확인
Command (m for help): w                       ← 저장·종료
The partition table has been altered.
Syncing disks.
```

```bash
partprobe /dev/vdb            # 커널에 파티션 테이블 재읽기 요청 (w 뒤 "Re-reading failed" 시 필수)
udevadm settle                # udev 이벤트 처리 완료 대기
```

- `t` : 타입 변경 (**t**ype) — Rocky 9 fdisk 는 hex 대신 별칭 `swap` `lvm` `linux` `raid` 도 허용
- `w` : 기록 (**w**rite) — `q` 로 나가면 전부 취소
- `partprobe` : 파티션 테이블 변경을 커널에 통보 (**part**ition **probe**) — 사용 중 디스크(`vda`)는 재부팅 필요할 수 있음
- `udevadm settle` : 대기 중인 udev 이벤트가 끝날 때까지 블록 (**settle**)

**검증**
```bash
lsblk /dev/vdb
fdisk -l /dev/vdb | tail -4
ls /dev/vdb*
cat /proc/partitions | grep vdb
```

```text
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
vdb    252:16   0   5G  0 disk
├─vdb1 252:17   0   2G  0 part
├─vdb2 252:18   0   1G  0 part
└─vdb3 252:19   0   2G  0 part
Device     Boot   Start      End  Sectors Size Id Type
/dev/vdb1          2048  4196351  4194304   2G 83 Linux
/dev/vdb2       4196352  6293503  2097152   1G 82 Linux swap / Solaris
/dev/vdb3       6293504 10485759  4192256   2G 8e Linux LVM
/dev/vdb  /dev/vdb1  /dev/vdb2  /dev/vdb3
```

> 📝 **시험 포인트**: 타입 코드 **82=swap, 83=Linux, 8e=LVM, fd=RAID** 암기. `w` 없이 `q` 하면 아무 변화 없음 → "저장 후 종료 = w" (R01-46 ④ 함정).

### 2-4. GPT 도구 조회 — parted / gdisk / sfdisk 백업

> **상황**: `vdc` `vdd` 는 RAID 에 통째로, `vde` 는 LVM 에 통째로 쓰므로 파티션을 만들지 않는다. GPT 도구는 조회·문법만 확인한다.

```bash
parted -l                                   # 전체 디스크 파티션 테이블 (GPT/msdos 표기)
parted /dev/vdc print                       # 특정 디스크 — 빈 디스크는 "unrecognised disk label"
parted /dev/vdc unit GB print               # 단위 지정 출력
sfdisk -d /dev/vdb > /root/vdb-parttable.bak   # 파티션 테이블 덤프(텍스트) 백업
cat /root/vdb-parttable.bak
```

- `parted -l` : 모든 블록 장치 파티션 표 (**l**ist)
- `print` : 파티션 표 출력 / `unit GB|MiB|s` : 표시 단위 (**unit**)
- `sfdisk -d` : 파티션 테이블을 재생성 가능한 스크립트 형태로 덤프 (**d**ump) — 복원은 `sfdisk /dev/vdb < 파일`
- GPT 생성 문법 (※ 참고, 이 실습에서는 미실행 — vdc/vdd 는 RAID 에 통디스크 사용)
  ```bash
  parted /dev/vdX mklabel gpt                       # GPT 라벨 (MBR 은 mklabel msdos)
  parted /dev/vdX mkpart primary xfs 1MiB 100%      # 파티션 생성 (이름 primary 는 GPT 에선 단순 라벨)
  parted /dev/vdX set 1 lvm on                      # 플래그 (lvm, raid, boot, esp)
  gdisk /dev/vdX                                    # fdisk 와 같은 키 (n, t, w, p, o, ?) — GPT 전용
  ```

**검증**
```bash
parted -l 2>/dev/null | grep -E 'Disk /dev/vd|Partition Table'
head -3 /root/vdb-parttable.bak
```

```text
Disk /dev/vda: 42.9GB
Partition Table: gpt
Disk /dev/vdb: 5369MB
Partition Table: msdos
label: dos
label-id: 0x...
device: /dev/vdb
```

> 📝 **시험 포인트**: `parted` 는 MBR·GPT 모두 지원, 2TB 초과 가능, **`w` 없이 즉시 반영**(R01-47). GPT 생성 서브명령 = `mklabel gpt`(R02-47). `sfdisk -d` 는 MBR 백업 `dd bs=512 count=1` 과 함께 Part 12 백업에서 재등장.

---

## 3. 파일시스템 생성·마운트

### 3-1. mkfs — ext4 로 /dev/vdb1 포맷

> **상황**: 데이터 파티션 `vdb1` 을 ext4 로 만든다. 나중에 쿼터를 걸 것이므로 ext4(RHEL 에서 쿼터 실습 표준) 를 택한다.

⚠️ `mkfs` 는 기존 데이터 전부 삭제 — 대상이 `vdb1` 인지 재확인
```bash
lsblk -f /dev/vdb1                           # FSTYPE 비어 있음 확인
mkfs.ext4 -L DATA /dev/vdb1                  # ext4 생성 + 라벨 DATA
```

- `mkfs.ext4` : `mke2fs -t ext4` 의 심볼릭 링크 — `mkfs -t ext4 /dev/vdb1`, `mke2fs -t ext4 /dev/vdb1` 모두 동일
- `-L <라벨>` : 볼륨 라벨 (**L**abel) — `LABEL=DATA` 로 fstab 등록 가능
- 다른 형식 (참고, 여기서 실행하지 않음)
  - `mkfs -t xfs /dev/…` = `mkfs.xfs /dev/…` — `-t` : 유형 (**t**ype)
  - `mkfs.xfs -f /dev/…` — 기존 서명이 있어 거부될 때 강제 (**f**orce)

| 비교 | ext4 | xfs |
| --- | --- | --- |
| 저널 | 있음 (jbd2, 메타데이터 기본) | 있음 (메타데이터) |
| 확장 | `resize2fs` 온라인 가능 | `xfs_growfs` 온라인 가능 |
| **축소** | `resize2fs` **가능** (언마운트 후) | **불가** — 백업·재생성·복원 |
| 최대 파일시스템 | 1 EiB (RHEL 9 지원 50 TiB) | 8 EiB (RHEL 9 지원 1 PiB) |
| 최대 파일 | 16 TiB | 8 EiB |
| 점검 | `e2fsck`/`fsck.ext4` (부팅 시 pass 2) | `xfs_repair` (`fsck.xfs` 는 아무 일 안 함 → pass **0**) |
| 튜닝·라벨 | `tune2fs` `e2label` `dumpe2fs` | `xfs_admin` `xfs_info` `xfs_db` |
| RHEL 기본 | — | RHEL 7 이후 기본 루트 FS |

**검증**
```bash
lsblk -f /dev/vdb1
blkid /dev/vdb1
```

```text
NAME FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
vdb1 ext4   1.0   DATA  1f2e...-....-....-....-............
/dev/vdb1: LABEL="DATA" UUID="1f2e...." BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="....-01"
```

> 📝 **시험 포인트**: ext4 생성 명령 4형식 중 틀린 것 고르기(R04-32 — `fsck.ext4` 는 점검 도구). "xfs 축소 불가·ext4 는 resize2fs 축소 가능"(R10-30). 실기 R01-3 "sdb1 을 ext4 포맷 후 /data 마운트" = `mkfs.ext4 /dev/sdb1` → `mount /dev/sdb1 /data`.

### 3-2. ext4 메타데이터 조회·조정 — tune2fs, e2label, dumpe2fs

> **상황**: 방금 만든 ext4 의 슈퍼블록 정보(라벨·UUID·마운트 횟수·기능)를 읽고, 정기 점검 주기를 조정한다.

```bash
tune2fs -l /dev/vdb1 | grep -E 'volume name|UUID|Mount count|Maximum mount|Check interval|features'
tune2fs -c 20 -i 1m /dev/vdb1          # 20회 마운트 또는 1개월마다 fsck 강제
tune2fs -L DATA /dev/vdb1              # 라벨 변경 (mkfs -L 과 동일 결과)
e2label /dev/vdb1                      # 라벨 조회 (e2label /dev/vdb1 NEW 로 변경)
dumpe2fs -h /dev/vdb1 | head -30       # 슈퍼블록 헤더만
tune2fs -O has_journal /dev/vdb1       # 기능 플래그 (이미 켜져 있음 → 변화 없음)
```

- `tune2fs` : ext2/3/4 파라미터 조정 (**tune** **2**nd extended **f**ile **s**ystem)
  - `-l` : 슈퍼블록 정보 출력 (**l**ist)
  - `-c <N>` : 최대 마운트 횟수 — 초과 시 부팅 fsck 강제 (**c**ount), `-1` 또는 `0` 이면 비활성
  - `-i <기간>` : 점검 간격 `d`/`w`/`m` (**i**nterval)
  - `-L <라벨>` : 라벨 (**L**abel)
  - `-O [^]<기능>` : 기능 켜기/끄기 (**O**ption feature) — `^has_journal` 은 저널 제거 ⚠️ 언마운트 상태에서만, 실습 생략
  - `-j` : ext2 → ext3 저널 추가 (**j**ournal) — 기출 개념
- `e2label` : ext 라벨 조회·설정 (**e**xt**2** **label**)
- `dumpe2fs -h` : 슈퍼블록·그룹 정보 덤프, `-h` 헤더만 (**h**eader)
- xfs 대응 명령 `xfs_admin -L/-u`, `xfs_info` 는 6-4 (RAID xfs) 에서 실습

**검증**
```bash
tune2fs -l /dev/vdb1 | grep -E 'Maximum mount count|Check interval|volume name'
```

```text
Filesystem volume name:   DATA
Maximum mount count:      20
Check interval:           2592000 (1 month)
```

> 📝 **시험 포인트**: `tune2fs` 는 **ext 전용** — "xfs 라벨 조정에 사용" 은 틀림(R02-33). `-l` 슈퍼블록 출력, `-c` 마운트 횟수, `-j` ext3 변환.

### 3-3. 마운트 — mount /dev/vdb1 /data

> **상황**: 마운트 포인트 디렉터리를 만들고 수동 마운트한다. 마운트 확인은 4가지 도구로 교차한다.

```bash
mkdir -p /data
mount /dev/vdb1 /data                 # 유형 자동 감지 (blkid 기반)
mount | grep /data                    # 방식 1: mount 목록
findmnt /data                         # 방식 2: 트리형 (fstab·/proc/mounts 통합)
df -hT /data                          # 방식 3: 용량 + 유형
lsblk -f /dev/vdb1                    # 방식 4: MOUNTPOINTS 컬럼
ls -la /data                          # lost+found 존재 = ext 계열 표시
```

- `mount <장치> <디렉터리>` : 연결. 유형 생략 시 libblkid 로 자동 판별
- `findmnt [경로]` : 마운트 목록 트리 (**find** **m**ou**nt**) — `-T /data` 대상 지정, `--verify` 는 4-2
- `df -hT` : 사람 읽기 크기 + 파일시스템 유형 (**h**uman, **T**ype)
- `/data/lost+found` : `e2fsck` 가 복구한 고아 inode 를 넣는 ext 전용 디렉터리 (xfs 에는 없음)

**검증**
```bash
findmnt -no SOURCE,TARGET,FSTYPE,OPTIONS /data
touch /data/test.txt && ls /data
```

```text
/dev/vdb1 /data ext4 rw,relatime,seclabel
lost+found  test.txt
```

> 📝 **시험 포인트**: `mount` 출력 해석(R08-27) — `장치 on 마운트포인트 type 유형 (옵션)`. 마운트 포인트에 기존 파일이 있으면 마운트 동안 **가려짐**(삭제 아님).

### 3-4. 언마운트·사용 중(busy) 원인 추적 — umount, fuser, lsof

> **상황**: `/data` 안에 셸을 두고 언마운트를 시도해 "target is busy" 를 재현한 뒤, 누가 잡고 있는지 찾아 해결한다.

```bash
cd /data                                      # 현재 셸이 /data 점유
umount /data                                  # → umount: /data: target is busy.
fuser -vm /data                               # 점유 프로세스 (PID, 접근 종류)
lsof +D /data                                 # 열린 파일 목록 (디렉터리 이하 재귀)
cd /                                          # 점유 해제
umount /data
mount /dev/vdb1 /data                         # 다음 단계 위해 재마운트
```

- `umount <장치|디렉터리>` : 연결 해제 (**u**n**mount**) — 사용 중이면 실패
  - `umount -l /data` : 지연 해제 (**l**azy) — 이름은 즉시 떼고 사용 종료 시 실제 해제
  - `umount -f` : 강제 (**f**orce) — 응답 없는 NFS 용
- `fuser -vm <마운트포인트>` : 해당 파일시스템 사용 프로세스 (**v**erbose, **m**ount) — `-k` 로 kill 가능
- `lsof +D <디렉터리>` : 하위 전체 열린 파일 (**l**i**s**t **o**pen **f**iles, `+D` **D**irectory 재귀)

**검증**
```bash
fuser -vm /data 2>&1 | head -5
findmnt /data || echo "unmounted"
```

```text
                     USER        PID ACCESS COMMAND
/data:               root     kernel mount /data
                     root       1234 ..c.. bash            ← c = 현재 디렉터리
unmounted
```

> 📝 **시험 포인트**: "target is busy" 원인 = 셸 cwd·열린 파일·다른 마운트. 해결 순서: `fuser -vm` → 프로세스 종료·`cd /` → `umount`. `umount -l` 은 최후 수단.

### 3-5. 마운트 옵션 — remount,ro / noexec,nosuid,nodev / -t / -a

> **상황**: 운영 중 `/data` 를 읽기 전용으로 잠갔다가 풀고, 보안 옵션 `noexec` 을 걸어 스크립트 실행이 막히는지 실제로 검증한다.

```bash
mount -o remount,ro /data                     # 읽기 전용 전환 (언마운트 없이)
touch /data/ro-test                           # → Read-only file system
mount -o remount,rw /data                     # 복귀

mount -o remount,noexec,nosuid,nodev /data    # 보안 옵션 3종
printf '#!/bin/bash\necho RUN\n' > /data/run.sh && chmod +x /data/run.sh
/data/run.sh                                  # → Permission denied (noexec)
bash /data/run.sh                             # → RUN (인터프리터로 읽는 건 허용 — noexec 우회 지점)
mount -o remount,exec,suid,dev /data          # 해제
/data/run.sh                                  # → RUN

umount /data
mount -t ext4 -o noatime /dev/vdb1 /data      # 유형·옵션 명시 마운트
mount -a                                      # fstab 전체 (아직 미등록 → 변화 없음, 4-2 에서 재실행)
```

- `-o remount` : 재마운트 — 옵션만 바꿈 (**o**ptions)
- `ro` / `rw` : 읽기 전용 / 읽기·쓰기
- `noexec` : 실행 파일 직접 실행 금지 / `nosuid` : SetUID·SetGID 비트 무시 / `nodev` : 장치 파일 해석 금지
- `noatime` : 접근 시각 갱신 생략 (I/O 절감) / `relatime` : 기본값(수정 후 첫 접근만 갱신)
- `-t <유형>` : 파일시스템 유형 (**t**ype)
- `-a` : `/etc/fstab` 의 `noauto` 아닌 항목 전부 마운트 (**a**ll)

**검증**
```bash
findmnt -no OPTIONS /data
```

```text
rw,noatime,seclabel
```

> 📝 **시험 포인트**: `mount -o remount,ro /data` 의미(R04-29). `nosuid` = SetUID 무효(R02-31, R09-52), `noexec` = 실행 금지(R08-54). `defaults` 에 exec/suid/dev 가 **포함**되므로 보안 강화 시 명시적으로 `no*` 추가.

### 3-6. 루프 장치 마운트 — 이미지 파일·ISO

> **상황**: 파일을 블록 장치처럼 마운트하는 loop 방식을 두 가지로 실습한다 — `dd` 로 만든 빈 이미지에 ext4 를 얹는 방법, `genisoimage` 로 ISO 를 만들어 읽기 전용 마운트하는 방법.

```bash
# (1) 이미지 파일 → ext4 → loop 마운트
dd if=/dev/zero of=/root/loop.img bs=1M count=64 status=progress
mkfs.ext4 -q /root/loop.img                  # 파일에 직접 mkfs (⚠️ 파일 경로 재확인)
mkdir -p /mnt/loop
mount -o loop /root/loop.img /mnt/loop
losetup -a                                   # 자동 할당된 loop 장치 확인
echo hello > /mnt/loop/hi.txt

# (2) ISO 이미지 → iso9660 읽기 전용
mkdir -p /root/isosrc && cp /etc/hostname /etc/os-release /root/isosrc/
genisoimage -o /root/lab.iso -R -J -V LABISO /root/isosrc     # mkisofs 동일 문법
mkdir -p /mnt/iso
mount -o loop,ro -t iso9660 /root/lab.iso /mnt/iso
ls /mnt/iso

# (3) 수동 losetup
losetup -f                                   # 다음 빈 loop 장치명
losetup /dev/loop5 /root/loop.img 2>/dev/null || true   # 이미 loop 로 붙어 있으면 실패 — 정상
umount /mnt/iso /mnt/loop
losetup -D                                   # 모든 loop 해제 (개별은 losetup -d /dev/loopN)
```

- `dd if= of= bs= count=` : 입력·출력·블록 크기·개수 (**i**nput **f**ile, **o**utput **f**ile, **b**lock **s**ize) — `status=progress` 진행 표시
- `mkfs.ext4 -q` : 조용히 (**q**uiet)
- `-o loop` : 파일을 loop 장치로 자동 연결해 마운트
- `genisoimage -o <출력> -R -J -V <볼륨명> <디렉터리>` : ISO9660 생성 — `-R` **R**ock Ridge(유닉스 권한·긴 이름), `-J` **J**oliet(Windows 긴 이름), `-V` **V**olume ID. `mkisofs` 는 동일 도구의 옛 이름(심볼릭 링크)
- `-t iso9660` : CD/DVD 파일시스템 유형
- `losetup` : loop 장치 관리 (**lo**op **setup**) — `-a` 전체 목록(**a**ll), `-f` 빈 장치 찾기(**f**ind), `-d` 해제(**d**etach), `-D` 전부 해제

**검증**
```bash
mount -o loop /root/loop.img /mnt/loop && cat /mnt/loop/hi.txt && findmnt -no SOURCE,FSTYPE /mnt/loop
umount /mnt/loop; losetup -D
blkid /root/lab.iso
```

```text
hello
/dev/loop0 ext4
/root/lab.iso: UUID="2026-09-03-..." LABEL="LABISO" TYPE="iso9660" ...
```

> 📝 **시험 포인트**: "ISO 파일 마운트 → `mount -o loop -t iso9660 파일 디렉터리`". `mkisofs`/`genisoimage` = ISO 생성 도구. loop 장치는 `losetup -a` 로 확인.

---

## 4. /etc/fstab 영구 등록

### 4-1. 6개 필드·옵션 상세표

| 순서 | 필드 | 값 예 | 설명 |
| --- | --- | --- | --- |
| 1 | 장치 | `UUID=…` / `LABEL=DATA` / `/dev/vdb1` / `/dev/mapper/vg_lab-lv_share` / `srv:/nfs` | **UUID 권장** — 장치명은 인식 순서에 따라 변동 |
| 2 | 마운트포인트 | `/data`, `swap`(스왑은 `none` 또는 `swap`) | 디렉터리는 미리 존재해야 함 |
| 3 | 유형 | `ext4` `xfs` `swap` `iso9660` `nfs` `cifs` `tmpfs` `auto` | `auto` = blkid 자동 판별 |
| 4 | 옵션 | `defaults` `noatime,usrquota` `ro` `noauto` `nofail` `_netdev` | 쉼표 구분, 공백 없음 |
| 5 | dump | `0` / `1` | `dump` 백업 대상 여부 (1 = 백업) — Part 12 `dump` 와 연동 |
| 6 | pass | `0` / `1` / `2` | 부팅 시 `fsck` 순서 — **루트=1**, 다른 ext=2, **xfs·swap·nfs=0** |

- `defaults` 의 실제 구성 = **`rw,suid,dev,exec,auto,nouser,async`**

| 옵션 | 의미 |
| --- | --- |
| `auto` / `noauto` | `mount -a`·부팅 시 자동 마운트 / 제외 (수동 `mount /경로` 는 가능) |
| `user` / `nouser` | 일반 사용자 마운트 허용 (자동으로 `noexec,nosuid,nodev` 동반) / 금지 |
| `async` / `sync` | 비동기 쓰기 / 동기 쓰기 |
| `ro` / `rw` | 읽기 전용 / 읽기·쓰기 |
| `exec` `suid` `dev` | 실행·SetUID·장치 파일 허용 (`no*` 로 금지) |
| `noatime` `relatime` | 접근 시각 갱신 정책 |
| `usrquota` `grpquota` | ext4 쿼터 활성 준비 (9절) / xfs 는 `uquota` `gquota` `pquota` |
| `_netdev` | 네트워크 장치 — 네트워크 준비 후 마운트 (NFS·iSCSI·CIFS) |
| `nofail` | 장치가 없어도 부팅 계속 (외장·선택 디스크) |
| `x-systemd.automount` | 접근 시점에 systemd 가 마운트 |
| `pri=N` | 스왑 우선순위 (5-3) |

> 📝 **시험 포인트**: 6번째 필드 `2` 의미(R01-33, R04-30, R06-30, R09-53), 5번째 `1` 의미(R05-33), `noauto`(R05-34), 실기 R03-13 fstab 6필드 서술. `defaults` 구성 7개 암기.

### 4-2. UUID 로 /data 등록 → mount -a 로 문법 검증

> **상황**: 수동 마운트는 재부팅 시 사라진다. `blkid` 로 UUID 를 뽑아 fstab 에 넣고, 재부팅 없이 `mount -a`·`findmnt --verify` 로 문법을 검증한다.

```bash
cp -a /etc/fstab /etc/fstab.bak-part05                 # 원본 백업
umount /data                                            # 수동 마운트 해제 후 fstab 경로로 다시
UUID_DATA=$(blkid -s UUID -o value /dev/vdb1)
echo "UUID=$UUID_DATA  /data  ext4  defaults,noatime  0 2" >> /etc/fstab
tail -1 /etc/fstab
systemctl daemon-reload                                 # systemd 가 fstab → .mount 유닛 재생성
mount -a                                                # fstab 전체 적용 — 오류 없이 조용하면 문법 OK
findmnt --verify                                        # fstab 정적 검사
findmnt --verify --verbose | tail -5
```

- `blkid -s UUID -o value` : 특정 태그만 값으로 출력 (**s**how tag, **o**utput format)
- `systemctl daemon-reload` : RHEL 9 는 fstab 편집 후 `mount` 가 "systemd still uses the old version" 힌트를 출력 → 재로딩으로 `data.mount` 유닛 갱신
- `mount -a` : `noauto` 제외 전부 마운트 (**a**ll) — 이미 마운트된 것은 건너뜀
- `findmnt --verify` : fstab 각 줄의 장치 존재·유형·옵션 유효성 검사 (**verify**), `--verbose` 로 성공 항목도 표시

**검증**
```bash
findmnt /data
grep -c "$UUID_DATA" /etc/fstab
systemctl status data.mount --no-pager | head -3
```

```text
TARGET SOURCE    FSTYPE OPTIONS
/data  /dev/vdb1 ext4   rw,noatime,seclabel
1
● data.mount - /data
     Loaded: loaded (/etc/fstab; generated)
     Active: active (mounted) ...
```

> 📝 **시험 포인트**: 실기 R04-7 "fstab 전체 마운트 → `mount -a`". `UUID=` 가 장치명보다 안전한 이유 = 장치명 변동(R08-48). fstab 편집 후 **재부팅 전에 반드시 `mount -a` 로 오타 검증**.

### 4-3. /etc/mtab 과 /proc/mounts

> **상황**: 현재 마운트 목록을 커널이 직접 제공하는 파일과, 옛 방식 파일의 관계를 확인한다.

```bash
ls -l /etc/mtab                     # → /proc/self/mounts 심볼릭 링크
grep /data /proc/mounts             # 커널 뷰 (실시간, 정본)
grep /data /etc/mtab                # 동일 내용
grep /data /proc/self/mountinfo     # 마운트 ID·부모 ID 포함 확장 정보
diff <(cat /proc/mounts) <(cat /etc/mtab) && echo same
```

- `/proc/mounts` : 커널 마운트 테이블 — **읽기 전용, 항상 정확**
- `/etc/mtab` : 과거엔 `mount` 가 갱신하던 일반 파일 → 현재는 `/proc/self/mounts` **심볼릭 링크**
- `/proc/self/mountinfo` : `findmnt` 가 읽는 원본

**검증**
```bash
readlink /etc/mtab
```

```text
../proc/self/mounts
```

> 📝 **시험 포인트**: "현재 마운트 목록을 커널이 제공하는 파일 → `/proc/mounts`"(R05-35). `/etc/fstab` 은 **설정**(부팅 시 의도), `/proc/mounts` 는 **현재 상태** 구분.

### 4-4. fstab 오타 → emergency 모드 복구 절차

> **상황**: fstab 에 존재하지 않는 장치를 `nofail` 없이 적으면 부팅이 emergency 타겟에서 멈춘다. `nofail` 로 안전하게 재현해 보고, 실제 멈췄을 때의 복구 절차를 정리한다.

```bash
# 안전 재현: nofail 항목은 장치가 없어도 부팅·mount -a 가 계속됨
echo "/dev/vdz1  /mnt/none  ext4  defaults,nofail  0 0" >> /etc/fstab
mkdir -p /mnt/none
systemctl daemon-reload && mount -a; echo "exit=$?"      # 경고 없이 0 또는 장치 없음 메시지만
findmnt --verify 2>&1 | grep -i vdz                      # 검증 도구는 문제 지적
sed -i '\#/dev/vdz1#d' /etc/fstab                        # 실습 줄 제거
systemctl daemon-reload
```

⚠️ 아래는 **실제 부팅 실패를 유발**하는 선택 실습 — `nofail` 없이 잘못된 줄을 넣고 `reboot`. UTM 콘솔에서 진행하며, 복구 못 하면 스냅샷 복원
```text
[emergency 화면] Give root password for maintenance (or press Control-D to continue): <root 비밀번호>
# journalctl -xb | grep -iE 'mount|fstab' | tail        ← 실패 원인 확인
# mount -o remount,rw /                                  ← 루트가 ro 로 올라옴 → rw 전환
# vi /etc/fstab                                          ← 잘못된 줄 수정·삭제 (또는 nofail 추가)
# systemctl daemon-reload && mount -a                    ← 오류 없는지 재확인
# exit   (또는 reboot)                                   ← 정상 부팅 계속
```

- emergency 모드 = `emergency.target` — 루트만 **읽기 전용** 마운트, 최소 셸
- `mount -o remount,rw /` : fstab 편집 전 필수 — 루트가 ro 라 저장 불가
- 근본 예방: `mount -a` + `findmnt --verify` 를 fstab 편집 직후 실행, 선택 디스크는 `nofail`
- root 비밀번호를 모르는 경우의 `rd.break` 복구는 [[07-boot-systemd-log]] 에서 실습

**검증**
```bash
findmnt --verify && echo "fstab OK"
grep -c vdz /etc/fstab            # 0
```

```text
Success, no errors or warnings detected
fstab OK
0
```

> 📝 **시험 포인트**: "fstab 오타로 루트만 ro 마운트된 최소 셸 → `emergency.target`"(R06-6). `rescue.target` 은 fstab 정상·서비스만 안 올린 상태(모든 로컬 FS 마운트) 와 구분.

---

## 5. 스왑

### 5-1. 스왑 파티션 — mkswap, swapon

> **상황**: 2-3 에서 타입 82 로 만든 `vdb2`(1 GB)를 스왑으로 포맷해 즉시 활성화한다.

⚠️ `mkswap` 도 대상 데이터를 지움 — `vdb2` 확인
```bash
lsblk -f /dev/vdb2                        # FSTYPE 비어 있음
mkswap -L SWAP1 /dev/vdb2
swapon /dev/vdb2
swapon --show                             # = swapon -s (구식) 
cat /proc/swaps
free -h
```

- `mkswap -L <라벨>` : 스왑 서명 기록 (**m**a**k**e **swap**, **L**abel)
- `swapon <장치>` : 활성화 / `swapon -a` : fstab 의 swap 전부 (**a**ll) / `--show` : 활성 목록 (`-s`: summary, 구식)
- `/proc/swaps` : 커널 스왑 목록 — Filename, Type(partition/file), Size, Used, Priority
- `free -h` : 메모리·스왑 총량 (**h**uman)

**검증**
```bash
swapon --show=NAME,TYPE,SIZE,PRIO
blkid /dev/vdb2
```

```text
NAME         TYPE      SIZE PRIO
/dev/dm-1    partition   4G   -2        ← 설치 시 만든 rl-swap
/dev/vdb2    partition   1G   -3
/dev/vdb2: LABEL="SWAP1" UUID="...." TYPE="swap" PARTUUID="....-02"
```

> 📝 **시험 포인트**: 절차 `mkswap → swapon`(fdisk 타입 82 선행). 우선순위 미지정 시 **음수 자동 부여**(나중 것이 더 낮음). `swapon -s` 와 `/proc/swaps` 는 같은 정보(R08-51).

### 5-2. 스왑 fstab 등록·비활성/재활성

> **상황**: 스왑 파티션을 fstab 에 등록하고, `swapoff` 후 `swapon -a` 로 fstab 경로가 동작하는지 확인한다.

```bash
UUID_SWAP=$(blkid -s UUID -o value /dev/vdb2)
echo "UUID=$UUID_SWAP  swap  swap  defaults  0 0" >> /etc/fstab
systemctl daemon-reload
swapoff /dev/vdb2                         # 비활성 (사용 중 페이지를 RAM 으로 회수)
swapon --show | grep -c vdb2              # 0
swapon -a                                 # fstab 기준 전부 활성
```

- 2번 필드 `swap`(또는 `none`), 3번 `swap`, dump/pass **`0 0`**
- `swapoff <장치>` : 비활성 — 스왑 사용량이 가용 RAM 보다 크면 실패

**검증**
```bash
swapon --show | grep vdb2
grep swap /etc/fstab
```

```text
/dev/vdb2 partition 1G 0B   -2
/dev/mapper/rl-swap none swap defaults 0 0
UUID=....  swap  swap  defaults  0 0
```

> 📝 **시험 포인트**: 스왑 fstab 줄 `… swap swap defaults 0 0` 빈칸 채우기 단골. pass 는 fsck 대상이 아니므로 **0**.

### 5-3. 스왑 파일 /swapfile 512 MB·우선순위

> **상황**: 파티션 추가 없이 스왑을 늘리는 파일 방식. 루트가 xfs 라 `dd` 로 실제 블록을 채운다. 우선순위를 높여 파일 스왑이 먼저 쓰이도록 한다.

⚠️ `of=/swapfile` 경로 확인 — `of=/dev/…` 로 잘못 쓰면 디스크 파괴
```bash
dd if=/dev/zero of=/swapfile bs=1M count=512 status=progress
chmod 600 /swapfile                        # 다른 사용자 읽기 금지 — swapon 이 0644 면 경고
mkswap /swapfile
swapon -p 10 /swapfile                     # 우선순위 10
swapon --show
echo "/swapfile  swap  swap  defaults,pri=10  0 0" >> /etc/fstab
systemctl daemon-reload
swapoff /swapfile && swapon -a && swapon --show | grep swapfile   # fstab 경로 검증
```

- `bs=1M count=512` : 1 MiB × 512 = 512 MiB
- `chmod 600` : `swapon` 이 "insecure permissions 0644" 경고 → 필수
- `-p <N>` : 우선순위 (**p**riority) — **숫자 클수록 먼저 사용**, 같은 값이면 라운드로빈(스트라이핑 효과). fstab 은 `pri=N`
- `fallocate -l 512M /swapfile` 로 순간 생성 가능하나, **xfs 에서는 미기록(unwritten) extent 때문에 커널 버전에 따라 `swapon` 이 거부**할 수 있어 RHEL 문서·시험 모두 `dd` 사용

**검증**
```bash
swapon --show=NAME,TYPE,SIZE,PRIO
free -h | grep -i swap
ls -l /swapfile
```

```text
NAME       TYPE      SIZE PRIO
/dev/dm-1  partition   4G   -2
/dev/vdb2  partition   1G   -3
/swapfile  file      512M   10
Swap:          5.5Gi          0B       5.5Gi
-rw-------. 1 root root 536870912 ... /swapfile
```

> 📝 **시험 포인트**: 스왑 파일 순서 `dd → chmod 600 → mkswap → swapon`(R06-34, R07-41). 우선순위 `swapon -p`/`pri=`, **클수록 우선**(R03-51, R08-51 — PRIO 10 인 /swapfile 이 먼저 사용).

### 5-4. 스왑 성향 — vm.swappiness

> **상황**: 스왑을 얼마나 적극적으로 쓰는지 조절하는 커널 파라미터를 확인·즉시 변경·영구 등록한다.

```bash
sysctl vm.swappiness                            # 현재 값 (기본 60)
cat /proc/sys/vm/swappiness                     # 동일 값의 procfs 경로
sysctl -w vm.swappiness=10                      # 즉시 변경 (재부팅 시 소멸)
echo 'vm.swappiness = 10' > /etc/sysctl.d/99-lab.conf
sysctl -p /etc/sysctl.d/99-lab.conf             # 파일 적용
sysctl --system                                 # 모든 sysctl.d 재적용 (부팅 시와 동일)
```

- `sysctl <키>` : 커널 파라미터 조회 (**sys**tem **c**on**t**ro**l**) — `.` 은 `/proc/sys/` 하위 `/`
- `-w` : 즉시 쓰기 (**w**rite)
- `-p [파일]` : 파일 로드 (**p**reload, 기본 `/etc/sysctl.conf`)
- `--system` : `/etc/sysctl.d/*.conf` `/run/sysctl.d` `/usr/lib/sysctl.d` 전부 적용
- `vm.swappiness` 0~200: 낮을수록 파일 캐시 유지·스왑 회피, 100 이면 동등 취급

**검증**
```bash
sysctl vm.swappiness; cat /proc/sys/vm/swappiness
```

```text
vm.swappiness = 10
10
```

> 📝 **시험 포인트**: `sysctl -w` 는 일시적, 영구는 `/etc/sysctl.d/*.conf` + `sysctl -p`(R02-55 `net.ipv4.ip_forward` 와 동일 패턴). `vm.swappiness=0` 은 `ip_forward` 문제의 오답 선지(R06-93) — 스왑 성향과 라우팅 매개변수 구분.

---

## 6. RAID 1 — mdadm

### 6-1. RAID 레벨표

| 레벨 | 최소 디스크 | 가용 용량 (n개 × D) | 장애 허용 | 특징 |
| --- | --- | --- | --- | --- |
| RAID 0 | 2 | **n×D** | 0개 | 스트라이핑 — 성능↑, 1개 고장 = 전체 손실 |
| RAID 1 | 2 | **D** (1개분) | n−1개 | 미러링 — 효율 50%, 읽기 성능↑ |
| RAID 5 | 3 | **(n−1)×D** | 1개 | 분산 패리티 — 쓰기 시 패리티 계산 부하 |
| RAID 6 | 4 | **(n−2)×D** | 2개 | 이중 패리티 |
| RAID 10 (1+0) | 4 | **(n/2)×D** | 미러 쌍당 1개 | 미러 후 스트라이핑 — 성능·안정 절충 |
| Linear / JBOD | 2 | n×D | 0개 | 단순 연결 |

- 1TB×4 → RAID 5 = **3TB**, RAID 6 = 2TB, RAID 10 = 2TB, RAID 0 = 4TB
- 이 실습: `vdc` + `vdd` 5G×2 → RAID 1 → 가용 5G

> 📝 **시험 포인트**: 가용 용량 계산(R05-50), 최소 디스크(R07-49 — RAID 6 은 **4개**), 용도 선택(R06-33 "4장, 효율↑, 1장 장애 허용 → RAID 5"), 실기 R02-15 RAID 1 vs 5 서술.

### 6-2. 어레이 생성 — mdadm --create + 동기화 관찰

> **상황**: `vdc` `vdd` 통디스크로 RAID 1 `/dev/md0` 를 만들고, 초기 동기화(resync) 진행을 `/proc/mdstat` 로 지켜본다.

⚠️ 두 디스크의 기존 내용 전부 삭제 — `vdc` `vdd` 가 빈 5G 디스크인지 확인
```bash
lsblk -f /dev/vdc /dev/vdd                                      # FSTYPE 비어 있음
mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/vdc /dev/vdd
#   "Continue creating array? y" 프롬프트가 나오면 y
cat /proc/mdstat
watch -n 1 cat /proc/mdstat                                     # 진행률 관찰, Ctrl+C 로 종료
```

- `--create <md장치>` : 새 어레이 (**create**) — `-C`
- `--level=1` : RAID 레벨 (**level**) — `-l 1`
- `--raid-devices=2` : 구성 디스크 수 (**raid-devices**) — `-n 2`
- `--spare-devices=N` : 예비 디스크 수 (**spare**) — `-x N` (6-6 참고)
- `/proc/mdstat` : `[UU]` 정상, `[U_]` 1개 탈락, `[2/2]` 활성/전체, `resync = 45.2%` 진행률
- `watch -n 1` : 1초 간격 반복 실행 (**n** interval)

**검증**
```bash
cat /proc/mdstat
lsblk /dev/vdc /dev/vdd
```

```text
Personalities : [raid1]
md0 : active raid1 vdd[1] vdc[0]
      5237760 blocks super 1.2 [2/2] [UU]
      [=====>...............]  resync = 28.3% (1483776/5237760) finish=0.6min speed=98918K/sec

NAME  MAJ:MIN RM SIZE RO TYPE  MOUNTPOINTS
vdc   252:32   0   5G  0 disk
└─md0   9:0    0   5G  0 raid1
vdd   252:48   0   5G  0 disk
└─md0   9:0    0   5G  0 raid1
```

> 📝 **시험 포인트**: 명령 해석(R02-51, R10-51 `--level=5 --raid-devices=3 --spare-devices=1`), `/proc/mdstat` 해석(R03-50, R08-50 — `[U_]` = degraded), 리빌드 진행률 확인 = `cat /proc/mdstat`(R06-55). md 장치 major 는 **9**.

### 6-3. 상세 조회·설정 저장 — --detail, --examine, mdadm.conf

> **상황**: 어레이 상태와 각 멤버의 슈퍼블록을 확인하고, 재부팅 후 `md0` 이름이 유지되도록 설정 파일에 저장한다.

```bash
mdadm --detail /dev/md0                          # 어레이 관점
mdadm --examine /dev/vdc                         # 멤버 디스크 슈퍼블록 관점
mdadm --detail --scan                            # ARRAY 한 줄 요약
mdadm --detail --scan >> /etc/mdadm.conf         # 설정 저장 (Rocky 9 기본 파일 없음 → 생성)
cat /etc/mdadm.conf
dracut -f                                        # initramfs 에 mdadm.conf 포함 (부팅 시 이름 고정)
```

- `--detail` : 어레이 상세 (**detail**) — `-D`. State, Active/Working/Failed/Spare Devices, UUID
- `--examine` : 멤버 장치의 md 슈퍼블록 (**examine**) — `-E`. Array UUID, Device Role, Events
- `--scan` : 모든 어레이 자동 탐색 (**scan**) — `-s`. `--detail --scan` 출력이 곧 mdadm.conf 문법
- `/etc/mdadm.conf` : `ARRAY /dev/md0 metadata=1.2 name=srv01:0 UUID=…` — 없으면 재부팅 후 `/dev/md127` 로 조립되어 fstab 장치명 불일치 가능(UUID 로 등록하면 마운트는 됨)
- `dracut -f` : initramfs 재생성 (**f**orce)

**검증**
```bash
mdadm --detail /dev/md0 | grep -E 'State|Active Devices|Working Devices|Failed'
grep -c ARRAY /etc/mdadm.conf
```

```text
             State : clean
    Active Devices : 2
   Working Devices : 2
    Failed Devices : 0
1
```

> 📝 **시험 포인트**: RAID 1 구축 순서 = 파티션 준비 → `--create` → `--detail --scan >> /etc/mdadm.conf` → `mkfs` → 마운트(R07-48). "상세 상태는 `--create` 로 확인" 은 틀림(R03-50 ④) → `--detail`.

### 6-4. xfs 생성·라벨·마운트 — mkfs.xfs, xfs_admin, xfs_info

> **상황**: 동기화 완료를 기다릴 필요 없이 `md0` 에 xfs 를 만들어 `/srv/raid` 에 마운트하고 fstab 에 등록한다. 여기서 xfs 메타데이터 도구도 실습한다.

⚠️ `/dev/md0` 대상 확인
```bash
cat /proc/mdstat | grep md0
mkfs.xfs /dev/md0                              # 라벨 없이 생성
xfs_admin -L RAID /dev/md0                     # 라벨 부여 (언마운트 상태 필수)
xfs_admin -l /dev/md0                          # 라벨 조회
xfs_admin -u /dev/md0                          # UUID 조회
mkdir -p /srv/raid
mount /dev/md0 /srv/raid
xfs_info /srv/raid                             # 블록 크기·AG 수·inode 크기 (마운트 후 경로 지정)
UUID_RAID=$(blkid -s UUID -o value /dev/md0)
echo "UUID=$UUID_RAID  /srv/raid  xfs  defaults  0 0" >> /etc/fstab
systemctl daemon-reload && mount -a
```

- `mkfs.xfs [-f] [-L 라벨]` : xfs 생성 — 기존 서명 있으면 `-f` 필요
- `xfs_admin` : xfs 파라미터 (**admin**) — `-L` 라벨 설정(**L**abel), `-l` 라벨 조회, `-u` UUID 조회, `-U generate` UUID 재생성 (스냅샷 복제 시)
- `xfs_info <마운트포인트|장치>` : 지오메트리 정보 — `meta-data … bsize=4096 … agcount=4`, `data blocks=`
- xfs 는 부팅 fsck 를 쓰지 않으므로 pass **0**

**검증**
```bash
findmnt -no SOURCE,FSTYPE,OPTIONS /srv/raid
df -hT /srv/raid
lsblk -f /dev/md0
```

```text
/dev/md0 xfs rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/md0       xfs   5.0G   69M  5.0G   2% /srv/raid
NAME FSTYPE FSVER LABEL UUID  FSAVAIL FSUSE% MOUNTPOINTS
md0  xfs          RAID  ....     5G     1% /srv/raid
```

> 📝 **시험 포인트**: xfs 라벨은 `tune2fs` 아닌 **`xfs_admin -L`**. "xfs 생성 명령 → `mkfs.xfs`"(R01-32, R05-53). `/srv/raid` 는 Part 12 백업 대상지(`/srv/raid/backup/`).

### 6-5. 장애 시뮬레이션 — --fail, --remove, --add

> **상황**: 운영 중 `vdd` 가 고장난 상황을 가정한다. 강제 실패 → degraded 상태 확인 → 제거 → 같은 디스크를 새 디스크처럼 다시 추가 → 재동기화 검증. 마운트 상태에서 진행해 서비스 무중단을 확인한다.

```bash
echo "before-fail" > /srv/raid/marker.txt
mdadm /dev/md0 --fail /dev/vdd                   # 강제 장애 표시
cat /proc/mdstat                                 # [U_] + (F)
mdadm --detail /dev/md0 | grep -E 'State|Failed'
cat /srv/raid/marker.txt                         # 서비스 계속 동작 확인
mdadm /dev/md0 --remove /dev/vdd                 # 어레이에서 제거
cat /proc/mdstat                                 # [2/1] [U_]
mdadm --zero-superblock /dev/vdd                 # 새 디스크처럼 슈퍼블록 초기화 (재사용 시)
mdadm /dev/md0 --add /dev/vdd                    # 교체 디스크 추가 → 자동 recovery
watch -n 1 cat /proc/mdstat                      # recovery 진행 → [UU] 되면 Ctrl+C
```

- `--fail <장치>` : 장애 표시 (**fail**) — `-f`. 출력 `(F)`
- `--remove <장치>` : 어레이에서 분리 (**remove**) — `-r`. failed/spare 상태에서만 가능
- `--zero-superblock <장치>` : md 메타데이터 삭제 — 다른 어레이에 재사용하거나 완전 폐기 시
- `--add <장치>` : 추가 (**add**) — `-a`. 부족한 자리 있으면 즉시 recovery, 아니면 spare
- 진행 중 `/proc/mdstat`: `recovery = 12.5% … finish=0.8min`

**검증**
```bash
cat /proc/mdstat | grep -A1 md0
mdadm --detail /dev/md0 | grep -E 'State :|Active Devices|Failed Devices'
cat /srv/raid/marker.txt
```

```text
md0 : active raid1 vdd[2] vdc[0]
      5237760 blocks super 1.2 [2/2] [UU]
             State : clean
    Active Devices : 2
    Failed Devices : 0
before-fail
```

> 📝 **시험 포인트**: 교체 절차 `--fail → --remove → (물리 교체) → --add`, `[U_]` = degraded(R08-50 ①). `vdd[2]` 처럼 멤버 번호가 바뀌는 건 정상(역할 슬롯은 1).

### 6-6. 참고 — --stop, RAID 5+spare 문법

> **상황**: 어레이 해체 명령과 RAID 5 생성 문법은 실행하지 않고 문법만 확인한다 (md0 은 이후 파트에서 계속 사용).

```bash
# ※ 미실행 — 해체 절차
# umount /srv/raid
# mdadm --stop /dev/md0                             # 어레이 정지
# mdadm --zero-superblock /dev/vdc /dev/vdd         # 멤버 메타데이터 삭제
# sed -i '/md0/d' /etc/mdadm.conf; sed -i '\#/srv/raid#d' /etc/fstab

# ※ 미실행 — RAID 5 (3 활성 + 1 스페어, 디스크 4개 필요)
# mdadm --create /dev/md1 --level=5 --raid-devices=3 --spare-devices=1 /dev/sdb /dev/sdc /dev/sdd /dev/sde
mdadm --create --help | head -5                   # 문법 확인만
```

- `--stop` : 어레이 비활성 (**stop**) — `-S`. 마운트 해제 후
- `--assemble --scan` : mdadm.conf 기준 재조립 (**assemble**) — `-A -s`
- RAID 5 3+1 : 가용 = (3−1)×D, 1개 장애 시 스페어가 자동 투입되어 재구축

**검증**
```bash
mdadm --detail /dev/md0 | grep -E 'Raid Level|Spare Devices'
```

```text
        Raid Level : raid1
     Spare Devices : 0
```

> 📝 **시험 포인트**: `--spare-devices=1` 의미 = 핫스페어(R10-51). RAID 5 4장 중 2장 동시 장애 → 데이터 손실(R03-49).

---

## 7. LVM — /srv/share

### 7-1. LVM 계층 그림·PV 생성 — pvcreate

> **상황**: `vde`(2 GB)를 통디스크로 PV 로 초기화한다. 이후 `vdb3` 를 추가해 VG 를 확장하는 그림을 먼저 그려둔다.

```text
물리 장치           PV                 VG (PE 4 MiB 풀)          LV (LE)            FS·마운트
/dev/vde  (2G) ─→ PV /dev/vde  ─┐
                                 ├─→ vg_lab ──→ lv_share (1G → 1.5G → 1.7G) ─→ xfs ─→ /srv/share
/dev/vdb3 (2G) ─→ PV /dev/vdb3 ─┘  (7-7 vgextend)   └─ lv_share_snap (200M, CoW 스냅샷, 7-8)

PE (Physical Extent) : PV 를 나누는 고정 단위(기본 4 MiB)   LE (Logical Extent) : LV 쪽 단위, PE 와 1:1 매핑
```

⚠️ `pvcreate` 는 기존 파일시스템 서명을 지움 — `vde` 확인
```bash
lsblk -f /dev/vde
pvcreate /dev/vde
pvs                                    # 요약
pvdisplay /dev/vde                     # 상세 (PE Size 는 VG 소속 후 표시)
pvscan                                 # 전체 PV 스캔
```

- `pvcreate <장치>` : 물리 볼륨 초기화 (**p**hysical **v**olume **create**) — LVM 라벨·메타데이터 영역 기록
- `pvs` : PV 요약 (**s**ummary) — PV, VG, Fmt, Attr, PSize, PFree
- `pvdisplay` : 상세 — `Allocatable`, `PE Size`, `Total PE`, `Free PE`
- `pvscan` : 모든 장치에서 PV 스캔·목록

**검증**
```bash
pvs /dev/vde
blkid /dev/vde
```

```text
  PV         VG Fmt  Attr PSize PFree
  /dev/vde      lvm2 ---  2.00g 2.00g
/dev/vde: UUID="...." TYPE="LVM2_member"
```

> 📝 **시험 포인트**: LVM 순서 **`pvcreate → vgcreate → lvcreate → mkfs → mount`**(R01-48, R05-49, R07-46). PV/VG/LV 정의(R01-49, R05-48). 기존 `rl` VG(`vda3`) 는 건드리지 않음.

### 7-2. VG 생성 — vgcreate

> **상황**: PV 하나로 볼륨 그룹 `vg_lab` 을 만들고 PE 크기·여유 PE 를 확인한다.

```bash
vgcreate vg_lab /dev/vde
vgs
vgdisplay vg_lab                        # PE Size, Total PE, Alloc PE, Free PE
vgscan
```

- `vgcreate <VG명> <PV…>` : 볼륨 그룹 생성 (**v**olume **g**roup **create**) — `-s 8M` 으로 PE 크기 변경 가능(기본 4 MiB)
- `vgs` : VG 요약 — `#PV`, `#LV`, `#SN`(스냅샷), `VSize`, `VFree`
- `vgdisplay` : 상세 — `PE Size 4.00 MiB`, `Total PE 511`, `Free PE / Size 511 / <2.00 GiB`
- `vgscan` : VG 스캔

**검증**
```bash
vgs vg_lab
vgdisplay vg_lab | grep -E 'PE Size|Total PE|Free  PE'
```

```text
  VG     #PV #LV #SN Attr   VSize  VFree
  vg_lab   1   0   0 wz--n- <2.00g <2.00g
  PE Size               4.00 MiB
  Total PE              511
  Free  PE / Size       511 / <2.00 GiB
```

> 📝 **시험 포인트**: 실기 R06-3 "PV 준비된 `/dev/sdc` 로 `vg01` 생성 → `vgcreate vg01 /dev/sdc`". 2 GiB 디스크가 `<2.00g`(511 PE) 인 이유 = 메타데이터 영역 1 MiB 제외 후 PE 정렬.

### 7-3. LV 생성 — lvcreate

> **상황**: `vg_lab` 에서 1 GB 논리 볼륨 `lv_share` 를 잘라내고, 장치 경로 3종이 같은 장치를 가리키는지 확인한다.

```bash
lvcreate -L 1G -n lv_share vg_lab
lvs
lvdisplay /dev/vg_lab/lv_share
lvscan
ls -l /dev/vg_lab/lv_share /dev/mapper/vg_lab-lv_share      # 둘 다 /dev/dm-N 심볼릭 링크
```

- `lvcreate -L <크기> -n <LV명> <VG>` : 논리 볼륨 생성 (**l**ogical **v**olume **create**, **L**arge size, **n**ame)
  - `-l <PE수|N%FREE|N%VG>` : extent 단위 지정 (소문자 **l**) — `-l 100%FREE`
- `lvs` : LV 요약 — LV, VG, Attr, LSize, Origin, Snap%
- `lvdisplay` : 상세 — `LV Path`, `LV Size`, `Current LE`, `Block device 253:N`
- `/dev/<VG>/<LV>` 와 `/dev/mapper/<VG>-<LV>` : 둘 다 device-mapper 노드 `/dev/dm-N` 링크. VG·LV 이름에 `-` 가 있으면 mapper 이름은 `--` 로 이스케이프

**검증**
```bash
lvs vg_lab
readlink -f /dev/vg_lab/lv_share /dev/mapper/vg_lab-lv_share
lsblk /dev/vde
```

```text
  LV       VG     Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_share vg_lab -wi-a----- 1.00g
/dev/dm-2
/dev/dm-2
NAME               MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
vde                252:64   0   2G  0 disk
└─vg_lab-lv_share  253:2    0   1G  0 lvm
```

> 📝 **시험 포인트**: `lvcreate -L 100G -n lv01 vg01` 형식(R07-46), `/dev/mapper/vg-lv` = LVM 경로(R08-26 — `/dev/mapper` 가 RAID 전용이라는 선지는 틀림).

### 7-4. xfs 생성·/srv/share 마운트·fstab

> **상황**: Samba 공유(Part 09)가 될 `/srv/share` 를 xfs 로 만들어 마운트하고 fstab 에 등록한다.

⚠️ 대상 `/dev/vg_lab/lv_share`
```bash
mkfs.xfs -L SHARE /dev/vg_lab/lv_share
mkdir -p /srv/share
mount /dev/vg_lab/lv_share /srv/share
UUID_SHARE=$(blkid -s UUID -o value /dev/vg_lab/lv_share)
echo "UUID=$UUID_SHARE  /srv/share  xfs  defaults  0 0" >> /etc/fstab
systemctl daemon-reload && mount -a
df -h /srv/share                                 # 확장 전 기준값 기록
```

- LVM 장치는 `/dev/mapper/vg_lab-lv_share` 로 fstab 에 적어도 되지만 UUID 로 통일
- `df -h` 기준값: 확장 전후 비교용

**검증**
```bash
findmnt -no SOURCE,FSTYPE /srv/share
df -h /srv/share | tail -1
```

```text
/dev/mapper/vg_lab-lv_share xfs
/dev/mapper/vg_lab-lv_share  1014M   40M  975M   4% /srv/share
```

> 📝 **시험 포인트**: `findmnt`·`df` 는 LVM 을 항상 `/dev/mapper/…` 이름으로 표시. 1 GiB LV 가 `1014M` 으로 보이는 이유 = xfs 로그·메타데이터 예약.

### 7-5. 온라인 확장 (1) — lvextend -r

> **상황**: 공유 영역이 부족해졌다. 마운트를 유지한 채 500 MB 확장하고 파일시스템까지 한 번에 늘린다. 확장 중에도 파일 접근이 되는지 확인한다.

```bash
echo "keep-me" > /srv/share/keep.txt
df -h /srv/share | tail -1                        # 전: ~1014M
lvextend -L +500M -r /dev/vg_lab/lv_share         # LV + 파일시스템 동시 확장
df -h /srv/share | tail -1                        # 후: ~1.5G
lvs vg_lab
cat /srv/share/keep.txt                           # 데이터 보존
```

- `lvextend -L +500M` : 500 MiB **증분** (**L**arge size, `+` 없으면 절대 크기)
- `-r` : 파일시스템도 함께 조정 (**r**esizefs) — 내부적으로 `fsadm resize` 호출 → xfs 는 `xfs_growfs`, ext4 는 `resize2fs`
- `-l +100%FREE` : VG 여유 전부 (7-7 참고)

**검증**
```bash
lvs -o lv_name,lv_size vg_lab
df -h /srv/share | tail -1
```

```text
  LV       LSize
  lv_share 1.49g
/dev/mapper/vg_lab-lv_share  1.5G   41M  1.5G   3% /srv/share
```

> 📝 **시험 포인트**: 실기 R03-3 "5GB 증설 + FS 확장 한 줄 → `lvextend -L +5G -r /dev/vg0/lv0`". `-r` 없으면 LV 만 커지고 `df` 는 그대로(R03-55 ④).

### 7-6. 온라인 확장 (2) — lvextend 후 xfs_growfs 분리 실행

> **상황**: `-r` 없이 LV 만 늘렸을 때 `df` 가 변하지 않는 것을 직접 보고, `xfs_growfs` 로 파일시스템을 따라 늘린다.

```bash
lvextend -L +200M /dev/vg_lab/lv_share            # LV 만 확장
lvs -o lv_name,lv_size vg_lab                     # 1.68g
df -h /srv/share | tail -1                        # 아직 1.5G ← 파일시스템 미반영
xfs_growfs /srv/share                             # xfs 확장 — 마운트포인트 지정, 마운트 상태 필수
df -h /srv/share | tail -1                        # 1.7G
```

- `xfs_growfs <마운트포인트>` : xfs 를 장치 크기까지 확장 (**grow** **fs**) — `-D <블록수>` 로 부분 확장 가능, 축소 불가
- ext4 라면 `resize2fs /dev/vg_lab/lv_share` — 장치 경로 지정, 온라인 확장 가능(축소는 언마운트 후 `resize2fs 장치 크기`)
- 다른 FS 도구 `fsadm resize <장치>` : 유형 자동 판별

**검증**
```bash
xfs_info /srv/share | grep -E '^data'
df -h /srv/share | tail -1
```

```text
data     =                       bsize=4096   blocks=440320, imaxpct=25
/dev/mapper/vg_lab-lv_share  1.7G   41M  1.7G   3% /srv/share
```

> 📝 **시험 포인트**: xfs 확장 = `lvextend` → **마운트된 상태**에서 `xfs_growfs /마운트포인트`(R02-32, R06-31, R07-47). ext4 = `resize2fs 장치`(R07-47 ③). "umount 후에만 xfs_growfs" 는 틀림.

### 7-7. VG 확장 — vgextend 로 /dev/vdb3 편입

> **상황**: `vde` 만으로는 VG 여유가 거의 없다(2G 중 1.7G 사용). 2-3 에서 타입 8e 로 만든 `vdb3` 를 PV 로 만들어 `vg_lab` 에 편입시켜 여유를 늘린다.

```bash
vgs vg_lab                                        # 전: VFree ~300M
pvcreate /dev/vdb3
vgextend vg_lab /dev/vdb3
vgs vg_lab                                        # 후: #PV 2, VFree ~2.3G
pvs                                               # 두 PV 모두 vg_lab 소속
vgdisplay vg_lab | grep -E 'Total PE|Free  PE'
# ※ 참고 — 여유 전부 할당 (실행하지 않음: 스냅샷 공간 남겨야 함)
# lvextend -l +100%FREE -r /dev/vg_lab/lv_share
```

- `vgextend <VG> <PV…>` : VG 에 PV 추가 (**extend**)
- `-l +100%FREE` : 남은 extent 전부 — `-L` 절대 크기 계산 없이 최대 확장

**검증**
```bash
vgs -o vg_name,pv_count,vg_size,vg_free vg_lab
pvs -o pv_name,vg_name,pv_size,pv_free
```

```text
  VG     #PV VSize VFree
  vg_lab   2 3.99g 2.30g
  PV         VG     PSize  PFree
  /dev/vda3  rl     ...    ...
  /dev/vdb3  vg_lab <2.00g <2.00g
  /dev/vde   vg_lab <2.00g 308.00m
```

> 📝 **시험 포인트**: VG 여유 부족 시 `pvcreate → vgextend`(R06-32), 이어서 `lvextend → resize2fs/xfs_growfs`(R02-48 순서 ㉡→㉠→㉣→㉢). `#PV` 증가로 확인.

### 7-8. 스냅샷 — 생성 → 파일 삭제 → 스냅샷에서 복구 → 제거

> **상황**: 공유 폴더 백업을 위해 시점 스냅샷을 만든다. 원본에서 파일을 실수로 지운 뒤 스냅샷을 읽기 전용으로 마운트해 되살리고, 스냅샷을 정리한다.

```bash
echo "important" > /srv/share/report.txt
lvcreate -s -L 200M -n lv_share_snap /dev/vg_lab/lv_share      # CoW 스냅샷 (변경분 200M 까지 추적)
lvs vg_lab                                                      # Origin, Data% 컬럼
rm /srv/share/report.txt                                        # 실수 삭제
ls /srv/share
mkdir -p /mnt/snap
mount -o ro,nouuid /dev/vg_lab/lv_share_snap /mnt/snap          # xfs 는 nouuid 필수 (원본과 UUID 동일)
ls /mnt/snap                                                    # report.txt 존재
cp -a /mnt/snap/report.txt /srv/share/                          # 복구
umount /mnt/snap
lvremove -y /dev/vg_lab/lv_share_snap                           # 스냅샷 제거
```

- `lvcreate -s -L 200M -n <이름> <원본LV>` : 스냅샷 (**s**napshot) — 원본 **전체 복사가 아니라** 변경 전 블록만 CoW 로 저장. `-L` 은 변경분 수용 용량, 100% 차면 스냅샷 무효화
- `-o ro,nouuid` : xfs 는 같은 UUID 두 개를 동시에 마운트 거부 → `nouuid`. 로그가 dirty 라 마운트 실패하면 `-o ro,nouuid,norecovery`
- `lvremove -y` : 확인 없이 제거 (**y**es)
- 대안: `lvconvert --merge /dev/vg_lab/lv_share_snap` 로 원본을 스냅샷 시점으로 롤백(언마운트 후 적용)

**검증**
```bash
cat /srv/share/report.txt
lvs vg_lab                                       # 스냅샷 사라짐
```

```text
important
  LV       VG     Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_share vg_lab -wi-ao---- 1.68g
```

- 스냅샷 존재 중 `lvs` 기대 출력

```text
  LV            VG     Attr       LSize   Pool Origin   Data%
  lv_share      vg_lab owi-aos---   1.68g
  lv_share_snap vg_lab swi-a-s--- 200.00m      lv_share 0.01
```

> 📝 **시험 포인트**: "스냅샷은 생성 시점에 원본 전체 복사" → **틀림**(R03-48 ②), CoW 변경분만. `lvcreate -s` 로 무중단 일관 백업(R09-50). Attr `o`=origin, `s`=snapshot.

### 7-9. 참고 — lvreduce·vgreduce·pvmove·vgchange·이름 변경·export/import

> **상황**: 축소·제거·이동 계열은 xfs 라 실행할 수 없거나(축소) 실습 자원을 훼손하므로, 무해한 이름 변경·활성 토글만 실제 실행하고 나머지는 문법을 확인한다.

```bash
# 실행 — 이름 변경 후 원복 (마운트 중에도 가능, fstab 은 UUID 라 영향 없음)
lvrename vg_lab lv_share lv_share_tmp && lvs vg_lab && lvrename vg_lab lv_share_tmp lv_share
# 실행 — VG 비활성/활성 (LV 사용 중이면 비활성 거부됨 → 마운트 해제 후 시도, 확인만)
vgchange -a n vg_lab 2>&1 | head -2                # "Logical volume vg_lab/lv_share in use" → 정상 거부
vgchange -a y vg_lab

# ※ 미실행 — 문법
# lvreduce -L -500M -r /dev/vg/lv_ext4        # ext4 만 가능: -r 이 umount→e2fsck -f→resize2fs→lvreduce 자동 수행
#                                              # xfs 는 축소 불가 → 백업 → lvremove → lvcreate → mkfs.xfs → 복원
# pvmove /dev/vdb3                             # vdb3 의 PE 를 같은 VG 다른 PV 로 이동 (온라인)
# vgreduce vg_lab /dev/vdb3                    # 빈 PV 를 VG 에서 제거 → pvremove /dev/vdb3
# vgrename vg_lab vg_share                     # VG 이름 변경 (/dev/mapper 이름·fstab 장치명 사용 시 갱신 필요)
# vgexport vg_lab  →  (디스크 다른 서버로 이동)  →  vgimport vg_lab  →  vgchange -a y vg_lab
```

- `lvrename <VG> <구> <신>` : LV 이름 변경 (**rename**)
- `vgchange -a y|n <VG>` : 활성/비활성 (**a**ctivate) — 부팅 시 `-a y` 자동
- `lvreduce -L -<크기> -r` : 축소 (**reduce**) ⚠️ `-r` 없이 축소하면 파일시스템 잘려 손상
- `pvmove <PV>` : PE 이동 → PV 교체·제거 전 단계
- `vgreduce <VG> <PV>` : VG 에서 PV 제거 (PE 비어 있어야 함) / `pvremove` : LVM 라벨 삭제
- `vgexport` / `vgimport` : VG 를 다른 시스템으로 이전 시 메타데이터 잠금/해제

**검증**
```bash
lvs -o lv_name,lv_attr vg_lab
findmnt /srv/share >/dev/null && echo mounted
```

```text
  LV       Attr
  lv_share -wi-ao----
mounted
```

> 📝 **시험 포인트**: "`pvcreate -s` 로 PV 축소" 는 없는 옵션(R09-50 ②). `lvreduce` 순서(ext4): umount → e2fsck → resize2fs → lvreduce → mount. xfs 축소 불가(R10-30).

---

## 8. 점검·용량 관리

### 8-1. 사용량 — df, du

> **상황**: 만든 파일시스템의 용량·inode 를 한 번에 보고, 디렉터리별 사용량으로 어디가 큰지 확인한다.

```bash
df -hT                                   # 전체 + 유형
df -h /data /srv/raid /srv/share         # 대상 지정
df -i /data                              # inode 사용량
df -hT -x tmpfs -x devtmpfs              # 가상 FS 제외
du -sh /srv/*                            # 디렉터리 합계
du -sh --apparent-size /srv/share        # 실제 파일 크기 합 (블록 할당량 아님)
du -h --max-depth=1 /srv | sort -h       # 1단계 하위별 정렬
```

- `df` : 파일시스템 단위 (**d**isk **f**ree) — `-h` 사람 읽기, `-T` 유형(**T**ype), `-i` **i**node, `-x <유형>` 제외(e**x**clude)
- `du` : 파일·디렉터리 단위 (**d**isk **u**sage) — `-s` 합계(**s**ummarize), `--max-depth=N` 깊이, `--apparent-size` 표기 크기
- `df` 와 `du` 차이: 삭제됐지만 열린 파일·예약 블록(ext4 5%)·메타데이터 → `df` 가 더 크게 나옴

**검증**
```bash
df -hT | grep -E 'vdb1|md0|lv_share'
df -i /data | tail -1
```

```text
/dev/vdb1                   ext4  2.0G   24K  1.9G   1% /data
/dev/md0                    xfs   5.0G   69M  5.0G   2% /srv/raid
/dev/mapper/vg_lab-lv_share xfs   1.7G   41M  1.7G   3% /srv/share
/dev/vdb1  131072   12 131060  1% /data
```

> 📝 **시험 포인트**: `df -h` 출력 해석(R01-54, R06-28, R08-26), `df -i` inode 고갈 진단("용량 남았는데 파일 생성 불가"). Part 06 `sysreport.sh` 에 `df` 요약 포함.

### 8-2. ext4 점검 — fsck, e2fsck, tune2fs -c, lost+found, debugfs

> **상황**: ext4 는 마운트 해제 후 점검해야 한다. `/data` 를 내려 읽기 전용 점검 → 강제 전체 점검 → 다시 마운트한다.

⚠️ 마운트된 파일시스템에 `fsck` 쓰기 모드 실행 금지 — 손상 유발. 반드시 `umount` 후
```bash
umount /data
fsck -n /dev/vdb1                   # 읽기 전용 점검 (변경 없음) — fsck.ext4 로 위임
e2fsck -f /dev/vdb1                 # 강제 전체 점검 (clean 이어도 검사)
fsck -y /dev/vdb1                   # 모든 질문에 yes (자동 복구) — 여기선 오류 없음
tune2fs -l /dev/vdb1 | grep -E 'state|Last checked|Mount count'
debugfs -R 'ls -l /' /dev/vdb1      # 저수준 탐색 (읽기 전용) — lost+found inode 확인
mount /data                         # fstab 기준 재마운트
```

- `fsck [-t 유형] <장치>` : 유형별 `fsck.<유형>` 호출 (**f**ile **s**ystem **c**hec**k**)
  - `-n` : 변경 없이 점검 (**n**o) / `-y` : 전부 **y**es / `-a` : 자동 복구(**a**uto, 안전한 것만) / `-A` : fstab 전체(**A**ll) / `-C` : 진행률
- `e2fsck -f` : 강제 (**f**orce) — `-p` preen(자동), `-c` 배드블록 검사 병행
- `tune2fs -c 20` : 3-2 에서 설정 — 마운트 20회마다 부팅 시 자동 `fsck`
- `/lost+found` : `e2fsck` 가 디렉터리 연결을 잃은 inode 를 `#inode번호` 이름으로 넣는 곳 — 지우지 말 것
- `debugfs -R '<명령>' <장치>` : ext 디버거 일회 명령 (**R**equest) — `stat`, `ls`, `undel` 참고
- 부팅 시 fsck 는 fstab pass 필드 순서로 `systemd-fsck@` 가 수행

**검증**
```bash
tune2fs -l /dev/vdb1 | grep -E 'Filesystem state|Mount count'
findmnt /data >/dev/null && echo remounted
```

```text
Filesystem state:         clean
Mount count:              1
remounted
```

> 📝 **시험 포인트**: "언마운트한 ext4 무결성 점검·복구 → `e2fsck`/`fsck -y`"(R06-54), "fsck 는 마운트 상태에서 실행" 은 틀림(R10-31). `lost+found` 의미.

### 8-3. xfs 점검 — xfs_repair

> **상황**: xfs 는 `fsck` 가 아니라 `xfs_repair` 다. RAID 영역을 내려 읽기 전용 점검 → 실제 복구 모드 → 재마운트.

⚠️ 언마운트 필수 — 마운트 상태면 "xfs_repair: … contains a mounted filesystem" 으로 거부
```bash
umount /srv/raid
xfs_repair -n /dev/md0              # 점검만 (no modify)
xfs_repair /dev/md0                 # 복구 모드 — 로그 dirty 면 "mount the filesystem to replay the log" 안내
mount /srv/raid
fsck.xfs /dev/md0                   # 아무 일 안 하고 0 반환 (호환용 더미)
```

- `xfs_repair -n` : 변경 없이 점검 (**n**o modify)
- `xfs_repair` : 복구 — 로그 재생 필요 시 먼저 `mount`/`umount` 한 번 수행, 불가하면 `-L` 로 로그 폐기(⚠️ 최후 수단)
- `fsck.xfs` : xfs 는 마운트 시 자동 로그 재생 → 부팅 fsck 불필요, fstab pass 0

**검증**
```bash
xfs_repair -n /dev/md0 2>&1 | tail -2; findmnt /srv/raid >/dev/null && echo mounted
```

```text
xfs_repair: /dev/md0 contains a mounted filesystem
...
mounted
```
- (언마운트 상태에서 `-n` 정상 출력 예)
```text
Phase 1 - find and verify superblock...
...
Phase 7 - verify link counts...
No modify flag set, skipping filesystem flush and exiting.
```

> 📝 **시험 포인트**: xfs 점검·복구 = `xfs_repair`(R01-32 ③ 은 "생성" 이 아님, R02-32 ③), `fsck.xfs` 는 no-op. ext 계열 `e2fsck` 와 대응 관계로 출제.

### 8-4. 참고 도구 — badblocks, fstrim, sync, smartctl, hdparm

> **상황**: 물리 디스크 건강 점검 도구는 가상 디스크에서 대부분 의미가 없다. 문법과 결과를 확인하고 미지원 항목은 표기한다.

```bash
sync                                       # 버퍼 → 디스크 강제 기록 (umount·전원 차단 전)
fstrim -v /data                            # TRIM/discard (SSD·thin) — 가상 디스크는 미지원 메시지 가능
badblocks -sv -c 1024 /dev/vdb2 2>&1 | tail -2   # 스왑 파티션에 읽기 전용 검사 (swapoff 먼저 권장) ※ 시간 소요
smartctl -a /dev/vda                       # ※ 미실행 결과 — virtio 는 SMART 미지원 → "Unable to detect device type"
hdparm -I /dev/vda 2>&1 | head -3          # ※ 참고 — virtio 는 ATA 명령 미지원 → ioctl 오류
hdparm -tT /dev/vda                        # 순차 읽기 속도 측정은 동작 (캐시/버퍼 없는 읽기)
sg_inq /dev/vda 2>&1 | head -2             # ※ 참고 — SCSI INQUIRY, virtio-blk 은 미지원
```

- `sync` : 페이지 캐시 플러시 (**sync**hronize)
- `fstrim -v <마운트포인트>` : 미사용 블록 discard 통보 (**v**erbose) — `fstrim.timer` 가 주간 실행
- `badblocks -sv` : 배드블록 스캔 (**s**how progress, **v**erbose) — `-w` 쓰기 검사는 데이터 파괴 ⚠️. 실무는 `e2fsck -c` 로 결과를 FS 에 반영
- `smartctl -a` : SMART 전체 정보 (**a**ll) — 실 SATA/NVMe 에서 `-H` 건강 상태, `-t short` 자가 테스트
- `hdparm -I` : ATA 식별 정보 (**I**dentify) / `-tT` : 읽기 속도(**t**imed, **T** cache)
- `sg_inq` : SCSI INQUIRY (sg3_utils)

**검증**
```bash
fstrim -v /data 2>&1; echo "exit=$?"
```

```text
fstrim: /data: the discard operation is not supported        ← UTM virtio 기본 (또는 "/data: 1.9 GiB trimmed")
exit=1
```

> 📝 **시험 포인트**: "SMART 정보로 교체 판단 → `smartctl -a /dev/sda`"(R06-53, R09-54). `hdparm -tT` 는 속도 측정, `badblocks` 는 불량 블록 검사 — 셋 구분.

---

## 9. 디스크 쿼터

### 9-1. ext4 쿼터 준비 — fstab 옵션 → remount → quotacheck → quotaon

> **상황**: 개발자별 `/data` 사용량을 제한한다. fstab 에 `usrquota,grpquota` 를 넣고 재마운트 → 쿼터 DB 파일 생성 → 활성화 순으로 진행한다.

```bash
sed -i 's#\(/data\s\+ext4\s\+\)defaults,noatime#\1defaults,noatime,usrquota,grpquota#' /etc/fstab
grep /data /etc/fstab
systemctl daemon-reload
mount -o remount /data                     # fstab 의 새 옵션 반영
mount | grep /data                         # usrquota,grpquota 표시 확인
quotacheck -cugm /data                     # aquota.user / aquota.group 생성
ls -l /data/aquota.*
quotaon -v /data                           # 활성화
quotaon -p /data                           # 상태 출력 (print)
```

- `usrquota` / `grpquota` : 사용자·그룹 쿼터 마운트 옵션 (ext4). 별칭 `uquota`/`gquota`, `usrjquota=aquota.user,jqfmt=vfsv1` 은 저널 쿼터
- `quotacheck` : 사용량 스캔·쿼터 파일 생성/갱신
  - `-c` : 파일 새로 생성 (**c**reate) / `-u` : **u**ser / `-g` : **g**roup / `-m` : 재마운트 없이 진행 (no-re**m**ount) / `-a` : 모든 쿼터 FS / `-v` 상세
- `quotaon -v <마운트포인트>` : 활성 (**v**erbose) / `-p` : 상태만 출력 (**p**rint) / `-a` : 전체
- `/data/aquota.user`, `/data/aquota.group` : 바이너리 쿼터 DB (root 600)

**검증**
```bash
quotaon -p /data
findmnt -no OPTIONS /data | tr ',' '\n' | grep quota
```

```text
group quota on /data (/dev/vdb1) is on
user quota on /data (/dev/vdb1) is on
usrquota
grpquota
```

> 📝 **시험 포인트**: 절차 **fstab `usrquota` → remount → `quotacheck -cum` → `edquota` → `quotaon`**(R07-26 순서 ㄴ→ㄹ→ㄷ→ㄱ 형태). fstab 옵션명(R05-45), `quotacheck` 옵션 `-cugm`.

### 9-2. 한도 설정 — edquota, setquota

> **상황**: `dev1`(Part 03 생성) 에게 블록 soft 100 MB / hard 120 MB, inode 무제한을 준다. 대화식 `edquota` 필드를 이해하고, 스크립트 가능한 `setquota` 로 실제 설정한다.

```bash
mkdir -p /data/dev1 && chown dev1:devteam /data/dev1 && chmod 750 /data/dev1
edquota -u dev1                    # vi 로 열림 — 필드 확인만 하고 :q! 로 나가기 (아래 표)
setquota -u dev1 100M 120M 0 0 /data          # soft hard isoft ihard (단위 접미사 허용, 없으면 KiB)
# setquota -u dev1 102400 122880 0 0 /data    ← KiB 표기 동일
edquota -p dev1 dev2               # dev1 설정을 dev2 에 복제 (prototype)
```

- `edquota -u <사용자>` : 편집기로 한도 편집 (**ed**it **quota**, **u**ser) / `-g <그룹>` / `-p <원본> <대상…>` : 복제(**p**rototype) / `-t` : 유예 기간(9-4)
- `edquota` 화면 필드

| 필드 | 의미 | 단위 |
| --- | --- | --- |
| `Filesystem` | 대상 장치 | — |
| `blocks` | 현재 사용 블록 | **1 KiB** 블록 |
| `soft` | 블록 soft 한도 — 초과 시 경고, 유예 기간 내 허용 | KiB, 0 = 무제한 |
| `hard` | 블록 hard 한도 — **절대 초과 불가** | KiB |
| `inodes` | 현재 파일 수 | 개 |
| `soft` / `hard` | inode soft/hard 한도 | 개 |

- `setquota -u <사용자> <bsoft> <bhard> <isoft> <ihard> <마운트포인트>` : 비대화식 설정 — `K/M/G` 접미사 가능(quota 4.x)

**검증**
```bash
quota -vu dev1
repquota /data | grep dev1
```

```text
Disk quotas for user dev1 (uid 2001):
     Filesystem  blocks   quota   limit   grace   files   quota   limit   grace
      /dev/vdb1       4  102400  122880               1       0       0
dev1      --       4  102400  122880              1     0     0
```

> 📝 **시험 포인트**: "soft/hard 편집 명령 → `edquota -u`"(R01-34), `edquota` 는 편집·`repquota` 는 보고서(R03-39 ① 함정). soft 는 유예 있음, hard 는 절대 한도.

### 9-3. 초과 검증 — Disk quota exceeded

> **상황**: `dev1` 으로 로그인해 한도를 넘는 파일을 써 본다. hard 120 MB 에서 쓰기가 끊기는지, 상태 보고에 `+` 플래그가 붙는지 확인한다.

```bash
su - dev1 -c 'dd if=/dev/zero of=/data/dev1/fill.bin bs=1M count=110'      # soft 초과, hard 이내 → 성공 + 경고
su - dev1 -c 'quota'                                                        # 자기 쿼터 — grace 시작
su - dev1 -c 'dd if=/dev/zero of=/data/dev1/fill2.bin bs=1M count=30'      # hard 초과 → 중간에 실패
quota -u dev1
repquota -a                                                                  # 전체 보고 (root)
repquota -s /data                                                            # 사람 읽기 단위
rm -f /data/dev1/fill*.bin                                                   # 정리
```

- `quota [-u 사용자] [-v]` : 사용자 관점 사용량·한도 — `-v` 는 0 사용 FS 도 표시, `-s` 사람 읽기
- `repquota -a` : 쿼터 활성 FS 전체 보고 (**rep**ort, **a**ll) / `-s` : 단위 자동(**s**) / `-u`/`-g`
- 보고서 플래그 컬럼: `--` 정상, `+-` 블록 soft 초과(유예 중), `-+` inode soft 초과, `++` 둘 다
- `dd: error writing '/data/dev1/fill2.bin': Disk quota exceeded` — hard 도달 지점에서 중단

**검증**
```bash
repquota /data | grep -E 'dev1|Block grace'
```

```text
Block grace time: 7days; Inode grace time: 7days
dev1      +-  122880  102400  122880  6days       2     0     0
```

> 📝 **시험 포인트**: `repquota` 출력 해석(R02-34, R08-57, R10-32) — `+-` = 블록 soft 초과, `grace` 컬럼 = 남은 유예. hard 한도 = "Disk quota exceeded".

### 9-4. 유예 기간·비활성·경고 — edquota -t, quotaoff, warnquota

> **상황**: soft 초과 후 hard 처럼 막히기까지의 유예 기간을 조정하고, 쿼터 끄기/켜기와 경고 메일 도구를 확인한다.

```bash
edquota -t                         # 편집기: "Block grace period: 7days" → 3days 로 수정 후 저장
repquota /data | head -3           # grace 반영 확인
quotaoff -v /data                  # 비활성
quotaon -p /data                   # off 확인
quotaon -v /data                   # 재활성
warnquota -s                       # ※ 참고 — soft 초과 사용자에게 메일 (/etc/warnquota.conf, Postfix 는 Part 09)
```

- `edquota -t` : 유예 기간 편집 (**t**ime) — 단위 `days` `hours` `minutes` `seconds`
- `quotaoff -v <마운트포인트>` : 비활성 (**off**) — `-a` 전체
- `warnquota` : soft 초과자에 메일 발송 (cron 일일 실행용) — `-s` 사람 읽기 단위. 메일 서버 없으면 발송 실패 → Part 09 이후 재시도
- 유예 만료 후에는 soft 가 hard 처럼 동작

**검증**
```bash
quotaon -p /data
repquota /data | grep 'grace time'
```

```text
group quota on /data (/dev/vdb1) is on
user quota on /data (/dev/vdb1) is on
Block grace time: 3days; Inode grace time: 7days
```

> 📝 **시험 포인트**: 유예 기간 설정 = `edquota -t`. soft 초과 → 유예 기간 내 허용 → 만료 시 차단. 쿼터 명령 5종(`quotacheck` `quotaon/off` `edquota` `quota` `repquota`) 역할 구분.

### 9-5. xfs 쿼터 — /srv/share 에 uquota + xfs_quota

> **상황**: xfs 는 `quotacheck`·aquota 파일이 없다. 마운트 옵션 `uquota` 로 켜고 `xfs_quota` 한 도구로 한도·보고를 처리한다. xfs 쿼터는 remount 로 켜지지 않아 실제 umount/mount 가 필요하다.

```bash
sed -i 's#\(/srv/share\s\+xfs\s\+\)defaults#\1defaults,uquota#' /etc/fstab
grep /srv/share /etc/fstab
systemctl daemon-reload
umount /srv/share && mount /srv/share            # remount 로는 xfs 쿼터 활성 불가
mount | grep /srv/share                          # usrquota 표시
mkdir -p /srv/share/dev1 && chown dev1:devteam /srv/share/dev1
xfs_quota -x -c 'limit bsoft=100m bhard=120m dev1' /srv/share
xfs_quota -x -c 'report -h' /srv/share
xfs_quota -x -c 'state' /srv/share
su - dev1 -c 'dd if=/dev/zero of=/srv/share/dev1/fill.bin bs=1M count=130'   # → Disk quota exceeded
xfs_quota -x -c 'report -h' /srv/share
rm -f /srv/share/dev1/fill.bin
```

- `uquota` (= `usrquota`) / `gquota` / `pquota`(프로젝트) : xfs 쿼터 마운트 옵션 — `uqnoenforce` 는 집계만
- `xfs_quota -x -c '<명령>' <마운트포인트>` : 전문가 모드(**x**pert) 로 명령(**c**ommand) 실행
  - `limit bsoft= bhard= [isoft= ihard=] <사용자>` : 한도 (`-g` 그룹)
  - `report -h` : 보고서 (**h**uman) / `state` : 활성 상태 / `quota -u dev1` : 개별 조회 / `timer -b 3days` : 유예
- xfs 는 항상 저널 쿼터 → `quotacheck` 불필요, 재부팅마다 자동 활성

**검증**
```bash
xfs_quota -x -c 'report -h' /srv/share | grep -E 'dev1|User quota'
findmnt -no OPTIONS /srv/share | grep -o 'usrquota'
```

```text
User quota on /srv/share (/dev/mapper/vg_lab-lv_share)
dev1          0   100M   120M  00 [------]
usrquota
```

> 📝 **시험 포인트**: ext4 (`usrquota` + `quotacheck` + `edquota`) vs xfs (`uquota` + `xfs_quota -x -c limit`) 도구 대비. xfs 쿼터 옵션은 **재마운트 불가 → umount/mount** 실무 함정.

---

## 10. 커널 모듈·장치

### 10-1. 적재 모듈 조회 — lsmod, modinfo, /proc/modules, /sys/module

> **상황**: 이 파트에서 쓴 기능(xfs, ext4, raid1, device-mapper, virtio_blk)이 어떤 커널 모듈로 동작하는지 확인한다.

```bash
lsmod | head -5
lsmod | grep -E '^(xfs|ext4|raid1|dm_mod|dm_snapshot|virtio_blk|loop)'
modinfo xfs                              # 파일 경로·라이선스·의존성·매개변수
modinfo -F filename raid1                # 필드 하나만
modinfo -p loop                          # 모듈 매개변수 목록
cat /proc/modules | head -3              # lsmod 의 원본
ls /sys/module/ | head                   # 적재·내장 모듈 sysfs
cat /sys/module/xfs/refcnt               # 참조 수 (마운트된 xfs 수와 연관)
ls /lib/modules/$(uname -r)/             # modules.dep, modules.alias, kernel/ 디렉터리
ls /lib/modules/$(uname -r)/kernel/fs/xfs/
```

- `lsmod` : 적재 모듈 목록 (**l**i**s**t **mod**ules) — Module, Size, Used by(참조 수·의존 모듈)
- `modinfo <모듈>` : 모듈 정보 (**mod**ule **info**) — `-F <필드>` 필드만(**F**ield), `-p` 매개변수(**p**arameters), `-n` 파일명
- `/proc/modules` : 커널 뷰(lsmod 원본) / `/sys/module/<모듈>/` : parameters, refcnt, holders
- `/lib/modules/$(uname -r)/` : 실행 중 커널의 모듈 트리 — `kernel/` 아래 `.ko.xz` (RHEL 9 는 xz 압축), `modules.dep` 의존성 표

**검증**
```bash
lsmod | grep -cE '^(xfs|ext4|raid1|dm_mod)'
modinfo -F filename xfs
```

```text
4
/lib/modules/5.14.0-...aarch64/kernel/fs/xfs/xfs.ko.xz
```

> 📝 **시험 포인트**: 적재 목록 = `lsmod`(R04-46), `lsmod` 출력 해석 — `Used by` 컬럼(R08-52, ext4 가 mbcache·jbd2 를 사용). 모듈 정보 = `modinfo`. `lsmod` 에 `-r` 같은 옵션 없음.

### 10-2. 적재·제거 — modprobe, rmmod, insmod, depmod

> **상황**: 아직 안 쓰는 `raid0` 모듈로 적재·제거를 연습한다. `modprobe` 와 `insmod` 의 의존성 처리 차이를 실제로 본다.

```bash
lsmod | grep raid0 || echo "not loaded"
modprobe -v raid0                                       # 의존 모듈 포함 적재 (-v: 실행한 insmod 표시)
lsmod | grep raid0
modprobe -r -v raid0                                    # 제거 (사용 중이면 거부)
lsmod | grep raid0 || echo "removed"

insmod /lib/modules/$(uname -r)/kernel/drivers/md/raid0.ko.xz   # 절대 경로 필수, 의존성 미해결
lsmod | grep raid0
rmmod raid0                                             # 이름으로 제거
modprobe -n -v raid1                                    # dry-run — 실행할 insmod 만 출력
modprobe --show-depends xfs                             # 의존 사슬
depmod -a                                               # modules.dep 재생성 (새 .ko 설치 후)
grep 'raid1.ko' /lib/modules/$(uname -r)/modules.dep | head -1
```

- `modprobe <모듈>` : `modules.dep` 참조해 **의존 모듈까지** 적재 (**mod**ule **probe**) — 이름만 지정, `/etc/modprobe.d` 설정 반영
  - `-v` : 상세(**v**erbose) / `-r` : 제거(**r**emove, 의존 모듈도 미사용이면 함께) / `-n` : 시험 실행(dry-ru**n**) / `--show-depends` : 의존 목록
- `insmod <파일경로>` : 파일 하나만 적재 (**ins**ert **mod**ule) — 의존성 미해결 시 `Unknown symbol` 오류, 설정 파일 미참조
- `rmmod <모듈>` : 제거 (**r**e**m**ove **mod**ule) — 의존 모듈 자동 처리 없음
- `depmod -a` : `modules.dep`·`modules.alias` 갱신 (**dep**endency **mod**ule, **a**ll)

**검증**
```bash
modprobe -v raid0 && lsmod | grep '^raid0' && modprobe -r raid0 && echo "load/unload OK"
```

```text
insmod /lib/modules/5.14.0-...aarch64/kernel/drivers/md/raid0.ko.xz
raid0                  ...  0
load/unload OK
```

> 📝 **시험 포인트**: `modprobe` vs `insmod`(R04-47, R08-53 — modprobe 는 `modules.dep` 참조·의존 자동), 제거 = `modprobe -r`(R01-52, R04-48 — `lsmod -r`·`insmod -r`·`depmod -r` 는 없음), `depmod` 역할(R07-51), NIC 드라이버 적재 `modprobe e1000e`(R06-47).

### 10-3. 모듈 설정 — /etc/modprobe.d, /etc/modules-load.d

> **상황**: 필요 없는 모듈을 부팅 시 차단(blacklist)하고, 매개변수를 고정하고, 부팅 시 자동 적재할 모듈을 등록한다.

```bash
cat > /etc/modprobe.d/lab.conf <<'EOF'
# 사용 안 하는 모듈 차단 — 자동 적재(alias) 금지
blacklist floppy
# 모듈 매개변수 고정 (예: loop 장치 최대 개수)
options loop max_loop=16
# 완전 차단 — 직접 modprobe 해도 /bin/false 실행 (USB 저장장치 차단 패턴)
# install usb-storage /bin/false
EOF
cat > /etc/modules-load.d/lab.conf <<'EOF'
raid0
EOF
modprobe -c | grep -E 'blacklist floppy|options loop'      # 현재 설정 확인
systemctl restart systemd-modules-load.service
lsmod | grep '^raid0'
```

- `/etc/modprobe.d/*.conf` : `modprobe` 동작 설정
  - `blacklist <모듈>` : 하드웨어 alias 로 자동 적재 금지 (명시적 `modprobe` 는 가능)
  - `options <모듈> <파라미터=값>` : 적재 시 매개변수
  - `install <모듈> <명령>` : 적재 대신 명령 실행 → `/bin/false` 면 완전 차단
  - `alias <별칭> <모듈>`
- `/etc/modules-load.d/*.conf` : 부팅 시 `systemd-modules-load` 가 적재할 모듈 이름 나열
- `modprobe -c` : 병합된 설정 출력 (**c**onfig)
- `blacklist` 만으로는 initramfs 단계 적재를 못 막을 수 있음 → `dracut -f` 병행 또는 커널 매개변수 `modprobe.blacklist=`

**검증**
```bash
modprobe -c | grep -c floppy
systemctl is-active systemd-modules-load && lsmod | grep -c '^raid0'
```

```text
1
active
1
```

> 📝 **시험 포인트**: 부팅 시 특정 모듈 영구 차단 = `/etc/modprobe.d/` 에 `blacklist`(R05-46), USB 저장장치 차단 `install usb-storage /bin/false`(R09-48). `rmmod` 는 일시적.

### 10-4. 하드웨어 조회 — lspci, lsusb, lshw, /sys

> **상황**: 디스크 컨트롤러가 어떤 드라이버(virtio)에 붙었는지, USB·PCI 목록과 하드웨어 요약을 본다.

```bash
lspci                                  # PCI 장치 목록
lspci -k                               # 각 장치에 바인딩된 커널 드라이버·모듈
lspci -k | grep -A3 -i 'block\|scsi\|storage'
lspci -nn | head -5                    # 벤더:디바이스 ID
lsusb                                  # USB 장치 (UTM 은 xHCI 컨트롤러·태블릿 정도)
lsusb -t                               # 트리
lshw -short -class disk -class storage # 하드웨어 요약 (설치한 lshw)
lshw -class disk | head -20
cat /sys/block/vdb/size                # 512 B 섹터 수 → ×512 = 바이트
cat /sys/block/vdb/queue/rotational    # 0 = SSD/가상, 1 = HDD
cat /sys/block/vdb/device/vendor 2>/dev/null || echo "(virtio: vendor 파일 없음)"
```

- `lspci` : PCI 버스 장치 (**l**i**s**t **PCI**) — `-k` 커널 드라이버(**k**ernel), `-nn` 숫자 ID(**n**umeric), `-v` 상세
- `lsusb` : USB 장치 (**l**i**s**t **USB**) — `-t` 트리(**t**ree), `-v` 상세
- `lshw` : 하드웨어 전체 (**l**i**s**t **h**ard**w**are) — `-short` 요약, `-class <종류>` 필터(disk, storage, network, memory), `-html`/`-json`
- `/sys/block/<장치>/` : 크기(`size`), 큐 속성(`queue/`), 파티션(`vdb1/`)

**검증**
```bash
lspci -k | grep -c 'Kernel driver in use: virtio-pci'
echo $(( $(cat /sys/block/vdb/size) * 512 / 1024 / 1024 / 1024 ))G
```

```text
5
5G
```

> 📝 **시험 포인트**: PCI 장치 목록 = `lspci`(R02-53, R04-53), USB = `lsusb`, 하드웨어 종합 = `lshw`/`dmidecode`. `lspci -k` 로 "드라이버 미바인딩" 진단.

### 10-5. udev — 이벤트 관찰·속성 조회·규칙 예시

> **상황**: 디스크가 붙을 때 udev 가 어떤 이벤트를 받는지 `partprobe` 로 재현해 관찰하고, 특정 파티션에 고정 심볼릭 링크를 만드는 규칙을 작성해 검증한다.

```bash
udevadm monitor --udev --subsystem-match=block &      # 백그라운드 관찰
sleep 1; partprobe /dev/vdb                          # remove/add/change 이벤트 유발
sleep 2; kill %1                                     # 관찰 종료

udevadm info --query=property -n /dev/vdb1 | grep -E 'ID_FS_LABEL|ID_FS_UUID|ID_FS_TYPE|DEVPATH'
udevadm info -a -n /dev/vdb1 | grep -E 'KERNEL==|SUBSYSTEM==' | head -4     # 규칙에 쓸 속성

cat > /etc/udev/rules.d/99-lab-disk.rules <<'EOF'
# DATA 라벨 ext4 파티션에 고정 이름 링크 생성
SUBSYSTEM=="block", ENV{ID_FS_LABEL}=="DATA", SYMLINK+="lab/data"
# vdb 통디스크는 disk 그룹 + 0660 (기본값과 동일 — 문법 예시)
KERNEL=="vdb", SUBSYSTEM=="block", GROUP="disk", MODE="0660"
EOF
udevadm control --reload-rules
udevadm trigger --subsystem-match=block --action=change
udevadm settle
ls -l /dev/lab/data
```

- `udevadm monitor` : 커널(`--kernel`)·udev(`--udev`) 이벤트 실시간 출력 — `--subsystem-match=block` 필터
- `udevadm info -a -n <장치>` : 규칙 매칭용 속성 체인 (**a**ttribute-walk)
- 규칙 문법: 매칭 키 `==`(`KERNEL`, `SUBSYSTEM`, `ATTR{}`, `ENV{}`), 할당 키 `=`/`+=`(`NAME`, `SYMLINK`, `OWNER`, `GROUP`, `MODE`, `RUN`). 파일명 숫자 순 처리, `/etc/udev/rules.d/` 가 `/usr/lib/udev/rules.d/` 보다 우선
- `udevadm control --reload-rules` : 규칙 재로딩 / `udevadm trigger` : 이벤트 재생성 (재부팅 없이 적용) / `settle` : 완료 대기
- USB 핫플러그 실습(※ 참고): UTM → USB 장치 연결 시 `udevadm monitor` 에 `add` 이벤트, `dmesg` 에 `sd*` 인식, `lsusb` 추가

**검증**
```bash
readlink -f /dev/lab/data
udevadm test /sys/block/vdb/vdb1 2>&1 | grep -E 'lab/data|99-lab' | head -3
```

```text
/dev/vdb1
...: /etc/udev/rules.d/99-lab-disk.rules:2 ...
```

- `udevadm monitor` 관찰 예

```text
monitor will print the received events for:
UDEV - the event which udev sends out after rule processing
UDEV  [1234.567] change   /devices/pci0000:00/.../block/vdb (block)
UDEV  [1234.570] remove   /devices/pci0000:00/.../block/vdb/vdb1 (block)
UDEV  [1234.580] add      /devices/pci0000:00/.../block/vdb/vdb1 (block)
```

> 📝 **시험 포인트**: udev = **동적** 장치 관리, `/dev` 자동 생성, 규칙 `/etc/udev/rules.d/`, 속성 기반 고정 이름 링크(R03-52 ③, R06-52, R09-49). "부팅 시 정적 생성" 은 틀림.

---

## 11. 재부팅 검증

### 11-1. 재부팅 전 최종 점검 → reboot → 일괄 검증 스크립트

> **상황**: fstab·mdadm.conf·modules-load.d·sysctl.d 에 넣은 설정이 재부팅을 견디는지 확인한다. 재부팅 전에 `mount -a`·`findmnt --verify` 로 fstab 을 다시 확인하고, 재부팅 후 검증 스크립트를 돌린다.

```bash
findmnt --verify && mount -a && echo "fstab OK"          # 재부팅 전 마지막 문법 확인
cat /etc/fstab | grep -vE '^#|^$'
sync
reboot
```

- 재부팅 후 root 로 로그인 → 아래 스크립트 저장·실행

```bash
cat > /usr/local/bin/check-part05.sh <<'EOF'
#!/bin/bash
# LAB 05 재부팅 검증 — 전부 OK 여야 함
ok(){ printf '  [OK]  %s\n' "$1"; }  ng(){ printf '  [NG]  %s\n' "$1"; RC=1; }
RC=0
echo "== 블록 장치"; lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS | grep -E 'vd[b-e]|md0|vg_lab'
echo "== 마운트"
for mp in /data /srv/raid /srv/share; do findmnt -n "$mp" >/dev/null && ok "$mp 마운트" || ng "$mp 마운트 안 됨"; done
findmnt -no OPTIONS /data | grep -q usrquota && ok "/data usrquota 옵션" || ng "/data usrquota 없음"
findmnt -no OPTIONS /srv/share | grep -q usrquota && ok "/srv/share uquota 옵션" || ng "/srv/share uquota 없음"
echo "== 스왑"; swapon --show
swapon --show=NAME | grep -q vdb2 && ok "vdb2 스왑" || ng "vdb2 스왑 없음"
swapon --show=NAME | grep -q /swapfile && ok "/swapfile 스왑" || ng "/swapfile 없음"
[ "$(sysctl -n vm.swappiness)" = "10" ] && ok "vm.swappiness=10" || ng "swappiness=$(sysctl -n vm.swappiness)"
echo "== RAID"; cat /proc/mdstat | grep -A1 '^md'
grep -q '\[UU\]' /proc/mdstat && ok "md0 [UU]" || ng "md0 degraded 또는 없음"
[ -e /dev/md0 ] && ok "/dev/md0 이름 유지" || ng "md0 이름 아님 (md127?) → mdadm.conf·dracut 확인"
echo "== LVM"; pvs; vgs; lvs
lvs --noheadings -o lv_name vg_lab | grep -q lv_share && ok "lv_share" || ng "lv_share 없음"
echo "== 쿼터"; quotaon -p /data
quotaon -p /data | grep -q 'user quota.*is on' && ok "/data 사용자 쿼터 on" || ng "/data 쿼터 off"
xfs_quota -x -c state /srv/share | grep -q 'User quota state on: ON' && ok "/srv/share xfs 쿼터 on" || ng "/srv/share 쿼터 off"
echo "== 모듈"; lsmod | grep -q '^raid0' && ok "raid0 자동 적재(modules-load.d)" || ng "raid0 미적재"
[ -L /dev/lab/data ] && ok "udev 링크 /dev/lab/data" || ng "udev 링크 없음"
echo "== 용량"; df -hT | grep -E 'Filesystem|vdb1|md0|lv_share'
echo; [ $RC -eq 0 ] && echo "ALL OK" || echo "일부 실패 — 위 [NG] 확인"
exit $RC
EOF
chmod +x /usr/local/bin/check-part05.sh
/usr/local/bin/check-part05.sh
```

- 스크립트 구성: `findmnt -n` 존재 여부 → `swapon --show=NAME` → `/proc/mdstat` `[UU]` → `lvs --noheadings -o` → `quotaon -p` → `xfs_quota state` → `lsmod` → `df -hT`
- `md127` 로 올라왔으면: `cat /etc/mdadm.conf` 확인 → 없으면 `mdadm --detail --scan >> /etc/mdadm.conf && dracut -f && reboot`

**검증**
```bash
/usr/local/bin/check-part05.sh | grep -E '^\s+\[(OK|NG)\]|ALL OK'
```

```text
  [OK]  /data 마운트
  [OK]  /srv/raid 마운트
  [OK]  /srv/share 마운트
  [OK]  /data usrquota 옵션
  [OK]  /srv/share uquota 옵션
  [OK]  vdb2 스왑
  [OK]  /swapfile 스왑
  [OK]  vm.swappiness=10
  [OK]  md0 [UU]
  [OK]  /dev/md0 이름 유지
  [OK]  lv_share
  [OK]  /data 사용자 쿼터 on
  [OK]  /srv/share xfs 쿼터 on
  [OK]  raid0 자동 적재(modules-load.d)
  [OK]  udev 링크 /dev/lab/data
ALL OK
```

> 📝 **시험 포인트**: "재부팅 후에도 유지" = fstab(마운트·스왑) + mdadm.conf(RAID 이름) + LVM 메타데이터(자동) + `/etc/sysctl.d`·`/etc/modules-load.d`·`/etc/udev/rules.d`(커널 설정). 실기 서술형에서 "영구 적용 방법" 을 물으면 이 파일들을 답한다.

---

## 체크리스트

| 수행 항목 | 명령 | 확인 방법 | ☐ |
| --- | --- | --- | --- |
| 추가 디스크 4개 인식 | `lsblk` `fdisk -l` `cat /proc/partitions` | `vdb`~`vde` 5G/5G/5G/2G | ☐ |
| vdb MBR 주 파티션 3개 (2G/1G/나머지) | `fdisk /dev/vdb` `n p` ×3 | `lsblk /dev/vdb` 3 part | ☐ |
| vdb2 타입 82, vdb3 타입 8e | `t 2 82` `t 3 8e` `w` `partprobe` | `fdisk -l /dev/vdb` Id 컬럼 | ☐ |
| 파티션 테이블 백업 | `sfdisk -d /dev/vdb > 파일` | `cat /root/vdb-parttable.bak` | ☐ |
| vdb1 ext4 (라벨 DATA) | `mkfs.ext4 -L DATA /dev/vdb1` | `blkid /dev/vdb1` TYPE=ext4 | ☐ |
| tune2fs 마운트 횟수·간격 | `tune2fs -c 20 -i 1m` | `tune2fs -l \| grep Maximum` | ☐ |
| /data 마운트·busy 해결 | `mount` `fuser -vm` `umount` | `findmnt /data` | ☐ |
| remount,ro / noexec 검증 | `mount -o remount,…` | `touch` 실패, `./run.sh` 거부 | ☐ |
| loop·ISO 마운트 | `mount -o loop` `genisoimage` `losetup -a` | `findmnt /mnt/loop` | ☐ |
| /data fstab UUID 등록 | `blkid -s UUID -o value` → `/etc/fstab` | `mount -a` `findmnt --verify` | ☐ |
| /etc/mtab ↔ /proc/mounts | `readlink /etc/mtab` | `../proc/self/mounts` | ☐ |
| nofail 재현·emergency 절차 숙지 | `mount -o remount,rw /` | `findmnt --verify` Success | ☐ |
| vdb2 스왑 + fstab | `mkswap -L SWAP1` `swapon` | `swapon --show` `/proc/swaps` | ☐ |
| /swapfile 512M pri=10 | `dd` `chmod 600` `mkswap` `swapon -p 10` | `swapon --show` PRIO 10 | ☐ |
| vm.swappiness 영구 10 | `sysctl -w` `/etc/sysctl.d/99-lab.conf` | `sysctl vm.swappiness` | ☐ |
| RAID 1 md0 (vdc+vdd) | `mdadm --create … --level=1 --raid-devices=2` | `cat /proc/mdstat` `[UU]` | ☐ |
| mdadm.conf 저장·initramfs | `mdadm --detail --scan >> /etc/mdadm.conf` `dracut -f` | `grep ARRAY /etc/mdadm.conf` | ☐ |
| md0 xfs 라벨·/srv/raid fstab | `mkfs.xfs` `xfs_admin -L RAID` | `xfs_info /srv/raid` `df -hT` | ☐ |
| 장애 시뮬레이션·복구 | `--fail` `--remove` `--zero-superblock` `--add` | `/proc/mdstat` `[U_]`→`[UU]` | ☐ |
| PV/VG/LV (vde → vg_lab → lv_share 1G) | `pvcreate` `vgcreate` `lvcreate -L 1G -n` | `pvs` `vgs` `lvs` | ☐ |
| /srv/share xfs + fstab | `mkfs.xfs -L SHARE` | `findmnt /srv/share` | ☐ |
| 온라인 확장 -r (+500M) | `lvextend -L +500M -r` | `df -h` 1.5G | ☐ |
| 확장 분리 (+200M) | `lvextend` → `xfs_growfs /srv/share` | `df -h` 1.7G | ☐ |
| VG 확장 (vdb3) | `pvcreate /dev/vdb3` `vgextend vg_lab /dev/vdb3` | `vgs` #PV 2, VFree↑ | ☐ |
| 스냅샷 생성·복구·제거 | `lvcreate -s -L 200M -n` `mount -o ro,nouuid` `lvremove` | `report.txt` 복구 | ☐ |
| lvrename·vgchange | `lvrename` `vgchange -a n/y` | `lvs` | ☐ |
| ext4 점검 | `umount` `fsck -n` `e2fsck -f` `fsck -y` | `tune2fs -l \| grep state` clean | ☐ |
| xfs 점검 | `umount` `xfs_repair -n` `xfs_repair` | Phase 1~7 출력 | ☐ |
| fstrim·smartctl 미지원 확인 | `fstrim -v` `smartctl -a` | 메시지 확인 (※) | ☐ |
| /data 쿼터 활성 | fstab `usrquota,grpquota` `remount` `quotacheck -cugm` `quotaon -v` | `quotaon -p /data` on | ☐ |
| dev1 한도 100M/120M | `setquota -u dev1 100M 120M 0 0 /data` | `quota -vu dev1` | ☐ |
| 초과 검증 | `su - dev1 -c 'dd …'` | `Disk quota exceeded`, `repquota` `+-` | ☐ |
| 유예 기간 3days | `edquota -t` | `repquota \| grep grace` | ☐ |
| /srv/share xfs 쿼터 | fstab `uquota` `umount/mount` `xfs_quota -x -c limit` | `xfs_quota -x -c report` | ☐ |
| 모듈 조회 | `lsmod` `modinfo xfs` `/proc/modules` | 경로 `.ko.xz` | ☐ |
| raid0 적재·제거 3방식 | `modprobe -v` `modprobe -r` `insmod 경로` `rmmod` `depmod -a` | `lsmod \| grep raid0` | ☐ |
| modprobe.d / modules-load.d | `blacklist floppy` `options loop` `raid0` | `modprobe -c` 재부팅 후 `lsmod` | ☐ |
| 하드웨어 조회 | `lspci -k` `lsusb` `lshw -short` | virtio-pci 드라이버 | ☐ |
| udev 규칙 링크 | `/etc/udev/rules.d/99-lab-disk.rules` `udevadm control --reload-rules` `trigger` | `ls -l /dev/lab/data` | ☐ |
| 재부팅 검증 | `reboot` → `/usr/local/bin/check-part05.sh` | `ALL OK` | ☐ |

---

## 기출 연결

| 기출 (문제집·회차·번호 또는 주제) | 이 파트의 단계 |
| --- | --- |
| 실기 R01-3 `/dev/sdb1` ext4 포맷 → `/data` 마운트 명령 2개 | 3-1 `mkfs.ext4`, 3-3 `mount` |
| 실기 R02-15 RAID 1 vs RAID 5 원리·최소 디스크·가용 용량 서술 | 6-1 RAID 레벨표 |
| 실기 R03-3 `/dev/vg0/lv0` 5GB 증설 + FS 확장 한 줄(`-r`) | 7-5 `lvextend -L +5G -r` |
| 실기 R03-13 fstab 한 줄 6개 필드 의미 서술 | 4-1 6필드표 |
| 실기 R04-7 fstab 전체 마운트 `mount ______` | 4-2 `mount -a` |
| 실기 R06-3 PV 준비된 `/dev/sdc` 로 `vg01` 생성 | 7-2 `vgcreate` |
| 필기 R01-32, R05-53 xfs 생성 명령 | 3-1 비교표, 6-4 `mkfs.xfs` |
| 필기 R01-33, R04-30, R06-30, R09-53 fstab 6번째 필드 `2` | 4-1 pass 필드 |
| 필기 R05-33 fstab 5번째 필드 `1` (dump) | 4-1 dump 필드 |
| 필기 R05-34 부팅 시 자동 마운트 제외 옵션 `noauto` | 4-1 옵션표 |
| 필기 R05-35 커널이 제공하는 마운트 목록 `/proc/mounts` | 4-3 |
| 필기 R02-30, R03-47, R08-54, R10-29 fstab 항목 해석 | 4-1, 4-2 |
| 필기 R02-31, R09-52 `nosuid` 의미 / R08-54 `noexec` | 3-5 보안 옵션 검증 |
| 필기 R04-29 `mount -o remount,ro /data` | 3-5 |
| 필기 R08-27 `mount` 출력 해석 | 3-3 |
| 필기 R06-6 fstab 오타 → 루트 ro 최소 셸 = `emergency.target` | 4-4 |
| 필기 R01-46 fdisk 내부 명령 / R04-31 `n → t → w` 순서 | 2-1 키표, 2-2, 2-3 |
| 필기 R01-47 parted 특징 / R02-47 `mklabel gpt` | 2-4 |
| 필기 R08-46 `fdisk -l` 출력 해석(GPT, Linux LVM) | 1-2, 2-3 |
| 필기 R02-46, R10-52 장치명 규칙(`sda5`, `nvme0n1p1`, `vda`) | 1-4 장치명표 |
| 필기 R08-47 `lsblk` 출력 해석 | 1-1 |
| 필기 R04-55, R05-52, R07-55, R08-48 UUID·유형 확인 `blkid` | 1-3, 4-2 |
| 필기 R06-46 핫스왑 디스크 인식 확인 `fdisk -l`/`lsblk` | 1-1, 1-2 |
| 필기 R04-32 ext4 생성 명령 중 틀린 것(`fsck.ext4`) | 3-1 |
| 필기 R02-33 `tune2fs` 설명 중 틀린 것(xfs 에 사용) | 3-2 |
| 필기 R06-54 언마운트 ext4 점검·복구 `e2fsck`/`fsck -y` / R10-31 fsck 는 언마운트 후 | 8-2 |
| 필기 R02-32, R03-55, R06-31, R07-47 xfs 온라인 확장 `xfs_growfs` | 7-5, 7-6 |
| 필기 R10-30 ext4 vs xfs (축소 가능 여부) | 3-1 비교표, 7-9 |
| 필기 R01-48, R05-49, R07-46 LVM 명령 순서 pv→vg→lv→mkfs | 7-1 ~ 7-4 |
| 필기 R01-49, R05-48 PV/VG/LV 정의 | 7-1 계층 그림 |
| 필기 R02-48, R06-32 VG 여유 부족 → `pvcreate` → `vgextend` → `lvextend` → `resize2fs` | 7-7 |
| 필기 R10-48 LVM 구성 절차 빈칸 | 7-1 ~ 7-3 |
| 필기 R03-48, R09-50 LVM 스냅샷(CoW, `lvcreate -s`) | 7-8 |
| 필기 R08-26 `df -h` 출력의 `/dev/mapper` 해석 | 7-3, 8-1 |
| 필기 R01-51, R02-49, R07-49, R10-50 RAID 레벨 특징 | 6-1 |
| 필기 R05-50 1TB×4 RAID 5 가용 용량 / R06-33 4장·효율·1장 장애 → RAID 5 / R09-51 미러링 = RAID 1 | 6-1 |
| 필기 R03-49 RAID 5 디스크 2개 동시 장애 | 6-1, 6-6 |
| 필기 R02-51, R10-51 `mdadm --create … --spare-devices` 해석 | 6-2, 6-6 |
| 필기 R03-50, R08-50 `/proc/mdstat` 해석 `[UU]`/`[U_]` | 6-2, 6-5 |
| 필기 R06-55 리빌드 진행률 `cat /proc/mdstat` | 6-2, 6-5 |
| 필기 R07-48 RAID 1 구축 절차(`--detail --scan >> /etc/mdadm.conf`) | 6-3 |
| 필기 R06-34, R07-41 스왑 파일 절차 `dd → chmod 600 → mkswap → swapon` | 5-3 |
| 필기 R03-51, R08-51 스왑 우선순위 `swapon -p`/`pri=`, 클수록 우선 | 5-3 |
| 필기 R08-28 `free -h` 스왑 해석 | 5-1 |
| 필기 R02-55 `sysctl -w` 즉시 변경 / R06-93 `vm.swappiness` 선지 | 5-4 |
| 필기 R01-34 soft/hard 편집 `edquota -u` / R03-39 `edquota` vs `repquota` | 9-2 |
| 필기 R05-45 fstab 쿼터 옵션 `usrquota` | 9-1 |
| 필기 R07-26 쿼터 적용 절차 순서 | 9-1 ~ 9-2 |
| 필기 R02-34, R08-57, R10-32 `repquota` 출력 해석(`+-`, grace) | 9-3 |
| 필기 R04-46 적재 모듈 목록 `lsmod` / R08-52 `lsmod` 출력 해석 | 10-1 |
| 필기 R01-52, R04-48 의존성 고려 제거 `modprobe -r` | 10-2 |
| 필기 R04-47, R08-53, R05-47 `modprobe` vs `insmod` | 10-2 |
| 필기 R07-51 `depmod` (modules.dep 갱신) | 10-2 |
| 필기 R06-47 NIC 드라이버 적재 `modprobe e1000e` | 10-2 |
| 필기 R05-46 부팅 시 모듈 영구 차단 `blacklist` / R09-48 USB 저장장치 차단 | 10-3 |
| 필기 R02-53, R04-53 PCI 장치 목록 `lspci` | 10-4 |
| 필기 R06-53, R09-54 SMART 점검 `smartctl -a` | 8-4 (※ 가상 디스크 미지원) |
| 필기 R03-52, R06-52, R09-49 udev 특징·규칙 위치 | 1-3, 10-5 |
| 필기 R07-25, R06-29 새 디스크 사용 절차 fdisk → mkfs → mount → fstab | 2 → 3 → 4 전체 |
| 필기 R08-63 dump 백업과 fstab dump 필드 연동 | 4-1 (Part 12 연결) |

---

## 이전 / 다음

[[04-file-text-shell]] ← · → [[06-process-scheduling-diagnosis]]

- 허브: [[README]]
- 이론: [[../THEORY/disk-device]]
- 명령어 문서: [[../../DISK-STORAGE/fdisk|fdisk]] · [[../../DISK-STORAGE/parted|parted]] · [[../../DISK-STORAGE/lsblk|lsblk]] · [[../../DISK-STORAGE/mount|mount]] · [[../../DISK-STORAGE/lvm|lvm]] · [[../../DISK-STORAGE/df|df]] · [[../../DISK-STORAGE/du|du]] · [[../../DISK-STORAGE/dd|dd]]
- 이 파트 자원을 쓰는 뒤 파트: `/srv/share` → [[09-network-services]] Samba, `/srv/raid/backup` → [[12-backup-recovery-review]], emergency·`rd.break` → [[07-boot-systemd-log]]
