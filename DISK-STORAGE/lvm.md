---
command: lvm
category: DISK-STORAGE
aliases: [vgs, lvs, pvs, lvextend, xfs_growfs]
tags:
  - linux/disk
  - topic/lvm
  - topic/capacity
  - task/inspect
  - task/expand
  - privilege/root
related: ["[[lsblk]]", "[[df]]", "[[parted]]", "[[fdisk]]", "[[du]]"]
distro: 전체 (lvm2 패키지)
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# LVM (Logical Volume Manager)

- 물리 디스크를 논리적으로 묶어 유연하게 분할·확장하는 볼륨 관리 체계
- 구조: **PV**(Physical Volume) → **VG**(Volume Group) → **LV**(Logical Volume)
- 디스크 2개 이상을 단일 볼륨그룹으로 통합 가능 → 물리 디스크 경계 초월 할당

---

## 조회 (pvs / vgs / lvs)

```bash
pvs        # 물리 볼륨 목록
vgs        # 볼륨 그룹 목록
lvs        # 논리 볼륨 목록

# Examples
vgs        # VFree 컬럼 = 미할당 여유 공간
lvs -o +devices    # 논리볼륨이 어느 물리디스크에 위치하는지
```

### 명령어 설명
- 사용 목적
	- 볼륨그룹 미할당 여유 공간 확인 시 사용
	- 논리볼륨 구성·크기 파악 시 사용
	- 확장 가능 용량 산정 시 사용
- 특이사항
	- 어원: **p**hysical/**v**olume **s**how, **v**olume**g**roup **s**how, **l**ogical**v**olume **s**how
	- [[df]] 로는 볼륨그룹 여유 공간 확인 불가 → `vgs` 의 `VFree` 필수
	- 단일 논리볼륨이 복수 물리디스크에 분산 배치 가능

---

## lvextend (논리볼륨 확장)

```bash
lvextend -L +<size> /dev/mapper/<VG>-<LV>
lvextend -l +100%FREE /dev/mapper/<VG>-<LV>

# Examples
lvextend -L +50G /dev/mapper/rl-log        # 50GiB 추가
lvextend -l +100%FREE /dev/mapper/rl-data  # 여유 공간 전부 할당
```

### 명령어 설명
- 사용 목적
	- 운영 중 부족해진 파티션 용량 확장 시 사용
	- 설치 시 의도적으로 남긴 여유 공간 활용 시 사용
- 특이사항
	- **논리볼륨 확장만 수행** → 파일시스템 확장은 `xfs_growfs` 별도 실행 필수
	- **서비스 중단 없이 온라인 확장 가능** (xfs 지원)
	- **축소 불가** → xfs 는 파일시스템 축소 미지원, 필요 시 백업→재생성→복원
	- 확장은 자유로우나 되돌릴 수 없음 → 소량씩 단계적 확장 권장

### 옵션
- `-L <size>` : 절대 크기 지정 (**L**arge, `+` 로 증분 지정)
- `-l <extents>` : 익스텐트 단위 지정 (소문자 **l**), `+100%FREE` 형식 지원

---

## xfs_growfs (파일시스템 확장)

```bash
xfs_growfs <mountpoint>

# Examples
xfs_growfs /log          # 마운트 지점 지정 (장치명 아님)
```

### 명령어 설명
- 사용 목적
	- `lvextend` 로 확장된 논리볼륨 크기를 파일시스템에 반영 시 사용
- 특이사항
	- **마운트 지점 지정** (장치 경로 아님)
	- xfs 전용 → ext4 는 `resize2fs` 사용
	- 마운트 상태에서 실행 (온라인 확장)

---

## 확장 전체 절차

```bash
vgs                                          # ① 여유 공간(VFree) 확인
lvextend -L +50G /dev/mapper/rl-log          # ② 논리볼륨 확장
xfs_growfs /log                              # ③ 파일시스템 확장
df -h /log                                   # ④ 결과 확인
```

---

## 연관 명령어
- [[lsblk]] : LVM 논리볼륨 트리 구조 확인
- [[df]] : 확장 결과 용량 검증
- [[parted]] : 파티션 레벨 조작
- [[fdisk]] : 파티션 테이블 확인
- [[du]] : 용량 부족 원인 디렉터리 추적
