---
command: parted
category: DISK-STORAGE
aliases: []
tags:
  - linux/disk
  - task/partition
  - topic/partition
  - topic/boot
  - privilege/root
  - danger/destructive
related: ["[[fdisk]]", "[[lsblk]]", "[[dd]]", "[[grub2-install]]", "[[lvm]]"]
distro: 전체 (parted 패키지)
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# parted

- 파티션 생성·삭제·플래그 설정 도구
- 어원: **part**ition **ed**itor
- [[fdisk]] 대비 스크립트 친화적, GPT 지원 우수

---

## parted print (조회)

```bash
parted <device> print

# Examples
parted /dev/sda print          # 파티션 목록·플래그
parted -l                      # 전체 디스크
```

### 명령어 설명
- 사용 목적
	- 파티션 구성 및 플래그 상태 확인 시 사용
- 특이사항
	- `Flags` 컬럼에 `boot`, `lvm` 등 표시

---

## parted set (플래그 설정)

```bash
parted <device> set <partition-number> <flag> on|off

# Examples
parted /dev/sda set 1 boot on      # 1번 파티션에 부트 플래그 설정
parted /dev/sdc set 1 boot on
parted /dev/sda set 1 lvm on       # LVM 플래그
```

### 명령어 설명
- 사용 목적
	- 부트 플래그 설정으로 부팅 실패 해소 시 사용
	- 구형 BIOS가 부트 플래그를 요구하는 환경에서 사용
- 특이사항
	- GRUB2 MBR 설치 후에는 부트 플래그 필수 아님 → [[grub2-install]] 이 MBR에 1단계 로더 기록
	- 부트 플래그 부재 상태에서 부팅 실패 지속 시 보조 조치로 사용
	- 플래그 확인은 [[fdisk]] `-l` 의 `Boot` 컬럼(`*`)으로도 가능

### 주요 플래그
- `boot` : 부트 파티션 표시
- `lvm` : LVM 물리볼륨
- `esp` : EFI 시스템 파티션 (UEFI)
- `swap` : 스왑 영역

---

## 연관 명령어
- [[fdisk]] : 파티션 테이블·부트 플래그 조회
- [[lsblk]] : 장치명 확인
- [[dd]] : 파티션 테이블 초기화
- [[grub2-install]] : 부트로더 설치
- [[lvm]] : LVM 볼륨 관리
