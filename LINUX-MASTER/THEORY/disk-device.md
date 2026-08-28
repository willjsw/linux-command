---
title: 디스크·장치 관리
type: exam-theory
category: LINUX-MASTER
tags:
  - exam/linux-master
  - exam/theory
  - linux/disk
  - linux/filesystem
  - topic/storage
  - topic/raid
  - task/manage
related: ["[[fdisk]]", "[[parted]]", "[[lsblk]]", "[[mount]]", "[[df]]", "[[du]]", "[[lvm]]", "[[dd]]", "[[system-structure]]"]
updated: 2026-08-28
---

# 디스크·장치 관리

- 리눅스마스터 1급 필기 대비 이론 — 디스크 장치·파티션·파일시스템·마운트·스왑·RAID·LVM·주변장치 영역
- MBR/GPT 한계, `/etc/fstab` 필드 순서, RAID 가용용량 공식이 최빈출 → 수치·순서 정확 암기 필요

---

## 1. 디스크 장치명

### 1-1. 인터페이스별 장치명

| 인터페이스 | 장치명 | 비고 |
| --- | --- | --- |
| SATA / SCSI / USB | `/dev/sda`, `/dev/sdb` … | 알파벳순 (a=첫 디스크) |
| NVMe SSD | `/dev/nvme0n1`, `/dev/nvme1n1` | `nvme<컨트롤러>n<네임스페이스>` |
| 가상(virtio) | `/dev/vda`, `/dev/vdb` … | KVM 등 가상화 환경 |
| IDE(구형) | `/dev/hda`, `/dev/hdb` … | 현재는 거의 미사용 |

### 1-2. 파티션 번호 규칙

- SATA/SCSI: `/dev/sda1`, `/dev/sda2` … — 장치명 **뒤에 숫자** 직접 부착
- NVMe: `/dev/nvme0n1p1`, `/dev/nvme0n1p2` … — 네임스페이스 뒤 **`p<번호>`**
- MBR에서 1~4 는 주/확장 파티션 예약, 논리 파티션은 **5번부터** 시작
- 블록 장치·파티션 트리 확인 → [[lsblk]]

---

## 2. 파티션

### 2-1. MBR vs GPT (최빈출 비교표)

| 구분 | MBR | GPT |
| --- | --- | --- |
| 정식 명칭 | Master Boot Record | GUID Partition Table |
| 최대 디스크 용량 | **2TB** | 사실상 무제한 (ZB급) |
| 주 파티션 개수 | **최대 4개** (주+확장 합) | **최대 128개** |
| 2TB 초과 | 인식 불가 | 지원 |
| 파티션 정보 백업 | 없음 (선두 1개) | 헤더·테이블 이중화 (선두+말미) |
| 펌웨어 | 주로 BIOS | 주로 UEFI |

### 2-2. 파티션 종류 (MBR)

| 종류 | 설명 |
| --- | --- |
| 주 파티션 (primary) | 부팅 가능, 디스크당 최대 4개 |
| 확장 파티션 (extended) | 주 파티션 1개를 대체, 논리 파티션의 컨테이너 (데이터 저장 불가) |
| 논리 파티션 (logical) | 확장 파티션 내부에 생성, 5번부터 번호 부여 |

- MBR 4개 한계 극복: 주 3개 + 확장 1개(내부에 논리 다수) 구성

### 2-3. 파티션 도구

| 도구 | 특징 |
| --- | --- |
| `fdisk` | 대화형, MBR 중심 (2TB 이하) → [[fdisk]] |
| `gdisk` | `fdisk` 의 GPT 대응판 |
| `parted` | MBR·GPT 모두 지원, 2TB 초과 처리, 스크립트 가능 → [[parted]] |

---

## 3. 파일시스템

### 3-1. 리눅스 파일시스템 특징

| 파일시스템 | 저널링 | 특징 |
| --- | --- | --- |
| ext2 | 없음 | 초기 표준, 저널 미지원 → 복구 느림 |
| ext3 | 있음 | ext2 + 저널링 추가 |
| ext4 | 있음 | 대용량·대형 파일 지원, extent 기반 (현재 널리 사용) |
| xfs | 있음 | 대용량·고성능, RHEL 기본, 축소 불가(확장만) |
| btrfs | 있음 | 스냅샷·풀링·체크섬 등 차세대 기능 |

- **저널링(journaling)**: 변경 사항을 저널에 먼저 기록 → 비정상 종료 시 빠른 복구·무결성 보장

### 3-2. 파일시스템 생성 — mkfs

- `mkfs -t <유형> <장치>` 또는 `mkfs.ext4 <장치>` / `mkfs.xfs <장치>` 형태
- 예: `mkfs.ext4 /dev/sdb1`, `mkfs -t xfs /dev/sdb1`

### 3-3. 마운트용 파일시스템 종류

| 유형 | 용도 |
| --- | --- |
| `vfat` / `fat32` | Windows 호환, USB 메모리 |
| `ntfs` | Windows NTFS |
| `iso9660` | CD/DVD, ISO 이미지 |
| `nfs` | 네트워크 파일시스템 (원격 공유) |
| `tmpfs` | 메모리 기반 임시 파일시스템 (재부팅 시 소멸) |
| `swap` | 스왑 영역 |

---

## 4. 마운트

### 4-1. mount / umount

- `mount <장치> <마운트포인트>` : 파일시스템을 디렉터리에 연결
- `mount -t <유형> -o <옵션> <장치> <디렉터리>` : 유형·옵션 명시
- `umount <장치 또는 마운트포인트>` : 연결 해제 (사용 중이면 실패)
- 인자 없는 `mount` : 현재 마운트 목록 조회 → [[mount]]

### 4-2. /etc/fstab 6개 필드 (최빈출)

- 부팅 시 자동 마운트 정보 정의 — 필드 순서가 시험 단골

```
# 장치            마운트포인트   유형    옵션            dump  pass
UUID=abcd-1234    /            ext4   defaults        1     1
/dev/sdb1         /data        xfs    defaults        0     2
/dev/sdc1         swap         swap   defaults        0     0
tmpfs             /tmp         tmpfs  defaults         0     0
```

| 순서 | 필드 | 설명 |
| --- | --- | --- |
| 1 | 장치 | 장치명 또는 `UUID=` / `LABEL=` |
| 2 | 마운트 포인트 | 연결할 디렉터리 (swap 은 `swap`) |
| 3 | 파일시스템 유형 | `ext4`, `xfs`, `swap`, `nfs` 등 |
| 4 | 마운트 옵션 | `defaults`, `ro`, `rw`, `noauto`, `usrquota` 등 |
| 5 | dump | 백업 대상 여부 (0=제외, 1=대상) |
| 6 | pass (fsck 순서) | 0=검사 안 함, **1=루트(`/`)**, 2=나머지 |

### 4-3. UUID / blkid

- **UUID**: 장치 고유 식별자 — 장치명(`/dev/sdX`) 변동에도 안정적 → fstab 권장
- `blkid` : 장치별 UUID·LABEL·파일시스템 유형 조회
- 자동 마운트: fstab 등록 후 `mount -a` 로 전체 재적용

---

## 5. 스왑(swap)

### 5-1. 스왑 구성 절차

```bash
mkswap /dev/sdb2        # ① 스왑 영역 생성(포맷)
swapon /dev/sdb2        # ② 스왑 활성화
swapoff /dev/sdb2       # 스왑 비활성화
```

- 스왑 **파티션** 외에 스왑 **파일**도 사용 가능
  - `dd if=/dev/zero of=/swapfile bs=1M count=1024` → `mkswap /swapfile` → `swapon /swapfile` → [[dd]]
- 부팅 시 자동 활성화: `/etc/fstab` 에 `swap swap defaults 0 0` 등록

### 5-2. 스왑 상태 확인

- `/proc/swaps` : 활성 스왑 목록·크기·사용량
- `swapon -s` / `free -h` : 스왑 사용 현황 조회

---

## 6. 파일시스템 점검·용량

| 도구 | 기능 |
| --- | --- |
| `fsck` | 파일시스템 검사·복구 (**언마운트 상태**에서 수행) |
| `df` | 파일시스템별 디스크 사용량 (`-h` 사람 읽기, `-i` inode) → [[df]] |
| `du` | 디렉터리·파일 단위 사용량 (`-sh` 요약) → [[du]] |
| `quota` | 사용자·그룹별 디스크 사용 한도 관리 |

- `fsck` 의 pass 순서는 fstab 6번 필드로 제어 (루트=1)
- `quota` 활성화: fstab 옵션에 `usrquota`/`grpquota` 추가 → `quotacheck` → `quotaon`

---

## 7. RAID

### 7-1. 레벨별 특징·가용용량 (최빈출)

- 전제: 디스크 `n` 개, 각 용량 동일

| 레벨 | 최소 디스크 | 가용 용량 | 특징 |
| --- | --- | --- | --- |
| RAID 0 | 2 | **전체 합 (n×용량)** | 스트라이핑, 성능↑, 중복성 없음 (1개 고장 = 전체 손실) |
| RAID 1 | 2 | **디스크 1개 용량** | 미러링, 완전 이중화, 용량 효율 50% |
| RAID 5 | 3 | **(n−1)×용량** | 분산 패리티, 디스크 1개 고장 허용 |
| RAID 6 | 4 | **(n−2)×용량** | 이중 패리티, 디스크 2개 고장 허용 |
| RAID 10 | 4 | **(n/2)×용량** | 1+0 결합 (미러 후 스트라이핑) |

### 7-2. 가용용량 계산 예 (1TB 디스크 기준)

| 구성 | 계산 | 가용 용량 |
| --- | --- | --- |
| RAID 0 × 4개 | 1TB × 4 | 4TB |
| RAID 1 × 2개 | 1TB × 1 | 1TB |
| RAID 5 × 4개 | (4−1) × 1TB | **3TB** |
| RAID 6 × 4개 | (4−2) × 1TB | 2TB |
| RAID 10 × 4개 | (4/2) × 1TB | 2TB |

- RAID 5 는 **최소 3개**, RAID 6·10 은 **최소 4개** 필요
- 패리티 = 고장 복구용 정보 (RAID5 디스크 1개분, RAID6 디스크 2개분 소모)

---

## 8. LVM (Logical Volume Manager)

### 8-1. 구조 계층

```
물리 장치 → PV → VG → LV → 파일시스템·마운트
/dev/sdb1   PV ┐
/dev/sdc1   PV ┼→ VG (vg_data) → LV (lv_home) → mkfs → /home
/dev/sdd1   PV ┘
```

| 계층 | 명칭 | 설명 |
| --- | --- | --- |
| PV | Physical Volume | 물리 파티션·디스크를 LVM용으로 초기화 (`pvcreate`) |
| VG | Volume Group | 여러 PV를 묶은 저장소 풀 (`vgcreate`) |
| LV | Logical Volume | VG에서 잘라낸 논리 볼륨 = 실제 사용 단위 (`lvcreate`) |

- 통합 관리 명령 → [[lvm]]

### 8-2. 온라인 확장

- LV 크기 확장 후 **파일시스템도 확장**해야 실제 공간 반영
  - `lvextend -L +10G /dev/vg_data/lv_home` (LV 확장)
  - ext4: `resize2fs /dev/vg_data/lv_home`
  - xfs: `xfs_growfs /마운트포인트` (xfs 는 확장만·축소 불가)
- 대부분 마운트한 상태에서 온라인 확장 가능 → 무중단 증설 장점

### 8-3. 스냅샷

- 특정 시점의 LV 상태를 보존하는 읽기·복원용 볼륨
- 원본 변경분(CoW)만 저장 → 백업·롤백에 활용

---

## 9. 주변장치·커널 모듈

### 9-1. 주변장치

| 장치 | 관리 체계 | 비고 |
| --- | --- | --- |
| 프린터 | **CUPS** (Common Unix Printing System) | 웹 관리 631포트, `lp`/`lpr` 출력, `lpstat` 상태 |
| USB 저장장치 | udev 자동 인식 → `/dev/sdX` | `lsusb` 로 목록 확인 |
| 사운드 | **ALSA** (Advanced Linux Sound Architecture) | `alsamixer` 볼륨, 구형 OSS 대체 |

### 9-2. 커널 모듈

| 명령 | 기능 |
| --- | --- |
| `lsmod` | 적재된 모듈 목록 (`/proc/modules` 기반) |
| `modprobe <모듈>` | 의존성 포함 모듈 적재 (제거는 `-r`) |
| `insmod <파일>.ko` | 모듈 파일 직접 적재 (의존성 자동 해결 안 함) |
| `rmmod <모듈>` | 모듈 제거 |
| `modinfo <모듈>` | 모듈 정보 조회 |

- `modprobe` 는 의존성 자동 처리, `insmod` 는 수동 → 대비 출제
- 모듈 설정: `/etc/modprobe.d/*.conf`

---

## 연관 문서
- [[fdisk]] · [[parted]] — 파티션 관리 실사용 문서
- [[lsblk]] — 블록 장치·파티션 트리 조회
- [[mount]] — 마운트 실사용 문서
- [[df]] · [[du]] — 디스크·디렉터리 용량 조회
- [[lvm]] — LVM 통합 관리 실사용 문서
- [[dd]] — 블록 단위 복제·스왑 파일 생성
- [[system-structure]] — 파일시스템 계층·디렉터리 구조
