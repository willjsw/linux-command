---
command: fdisk
category: DISK-STORAGE
aliases: []
tags:
  - linux/disk
  - task/inspect
  - task/partition
  - topic/partition
  - topic/mbr
  - topic/boot
  - privilege/root
  - danger/destructive
related: ["[[lsblk]]", "[[parted]]", "[[dd]]", "[[grub2-install]]"]
distro: 전체 (util-linux 패키지)
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# fdisk

- 파티션 테이블 조회·편집 도구
- 어원: **f**ormat/**f**ixed **disk**
- 조회(`-l`)는 안전, 대화형 편집 모드는 파괴적 → 주의 필요

---

## fdisk -l (조회)

```bash
fdisk -l <device>

# Examples
fdisk -l                      # 전체 디스크 파티션 테이블
fdisk -l /dev/sda             # 특정 디스크
fdisk -l /dev/sda | tail -5   # 파티션 목록만
```

### 출력 예시
```
Disk /dev/sda: 558.91 GiB, 600127266816 bytes, 1172123568 sectors
Disk model: ST600MM0088
Disklabel type: dos
Disk identifier: 0x7cf731c6

Device     Boot Start        End    Sectors   Size Id Type
/dev/sda1       2048 1066092543 1066090496 508.4G 8e Linux LVM

Disk /dev/sdc: 558.91 GiB, ...
Device     Boot Start       End    Sectors   Size Id Type
/dev/sdc1  *    2048   2099199    2097152     1G 83 Linux      ← 부트 플래그
/dev/sdc2     2099200 1066092543 1063993344 507.4G 8e Linux LVM
```

### 명령어 설명
- 사용 목적
	- 파티션 테이블 구조·크기·타입 확인 시 사용
	- **부트 플래그(`*`) 위치 확인** 시 사용 → 부팅 실패 진단
	- 디스크 라벨 타입(`dos`=MBR / `gpt`) 확인 시 사용
- 특이사항
	- `Boot` 컬럼의 `*` = 부트 플래그
	- `Disklabel type: dos` → MBR 방식 (레거시 BIOS 부팅)
	- `Id` 값: `83`=Linux, `8e`=Linux LVM, `82`=swap
	- 부트 플래그 없는 디스크를 BIOS가 먼저 읽으면 부팅 중단 가능 → [[grub2-install]] 참고
	- 부트 플래그 설정은 [[parted]] `set 1 boot on` 사용

### 옵션
- `-l` : 파티션 테이블 출력 후 종료 (**l**ist)

---

## fdisk (대화형 편집)

```bash
fdisk <device>

# 주요 서브명령
# p : 파티션 테이블 출력 (print)
# n : 새 파티션 생성 (new)
# d : 파티션 삭제 (delete)
# t : 파티션 타입 변경 (type)
# a : 부트 플래그 토글
# w : 변경 저장 후 종료 (write)
# q : 변경 취소 후 종료 (quit)
```

### 주의사항
- **`w` 실행 전까지는 디스크 미변경** → 실수 시 `q` 로 취소
- 마운트 중 디스크의 파티션 변경 시 재부팅 필요
- 운영 중 서버에서는 [[parted]] 또는 [[lvm]] 사용 권장

---

## 연관 명령어
- [[lsblk]] : 장치명 확인 (fdisk 대상 지정 전 필수)
- [[parted]] : 부트 플래그 설정, 스크립트 친화적 파티션 조작
- [[dd]] : 파티션 테이블·MBR 초기화
- [[grub2-install]] : 부트로더 설치 (부트 플래그와 연관)
