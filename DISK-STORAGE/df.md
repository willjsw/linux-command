---
command: df
category: DISK-STORAGE
aliases: []
tags:
  - linux/disk
  - task/inspect
  - topic/filesystem
  - topic/capacity
  - privilege/user
related: ["[[lsblk]]", "[[lvm]]", "[[du]]", "[[mount]]"]
distro: 전체 (coreutils 패키지)
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# df

- 마운트된 파일시스템 용량·사용률 출력 도구
- 어원: **d**isk **f**ree
- 일반 사용자 실행 가능

---

## df -h

```bash
df -h

# Examples
df -h                    # 사람이 읽기 쉬운 단위(G/M)로 전체 출력
df -h /data              # 특정 마운트 지점만
df -hT                   # 파일시스템 종류(xfs 등) 포함
df -i                    # inode 사용량
```

### 출력 예시
```
파일시스템              크기  사용  가용 사용% 마운트위치
/dev/mapper/rl-root     100G   ...   ...   ...  /
/dev/sda1              1014M   ...   ...   ...  /boot
/dev/mapper/rl-home     100G   ...   ...   ...  /home
/dev/mapper/rl-app      200G   ...   ...   ...  /app
/dev/mapper/rl-log      200G   ...   ...   ...  /log
/dev/mapper/rl-data     400G   ...   ...   ...  /data
```

### 명령어 설명
- 사용 목적
	- 파티션별 용량·사용률 확인 시 사용
	- OS 설치 직후 파티션 레이아웃 검증 시 사용
	- 디스크 부족 원인 파티션 식별 시 사용
- 특이사항
	- **마운트된 파일시스템만 표시** → 미마운트 파티션은 [[lsblk]] 로 확인
	- LVM 논리볼륨은 `/dev/mapper/<VG>-<LV>` 형식으로 표시
	- 볼륨그룹의 미할당 여유 공간은 표시되지 않음 → [[lvm]] `vgs` 의 `VFree` 확인
	- 디렉터리별 사용량 분석은 [[du]] 사용

### 옵션
- `-h` : 읽기 쉬운 단위 (**h**uman-readable)
- `-T` : 파일시스템 종류 표시 (**T**ype)
- `-i` : inode 정보 (**i**node)
- `-a` : 전체 파일시스템 포함 (**a**ll)

---

## 연관 명령어
- [[lsblk]] : 미마운트 포함 전체 블록 디바이스 구조
- [[lvm]] : 볼륨그룹 여유 공간 확인 및 논리볼륨 확장
- [[du]] : 디렉터리 단위 사용량 분석
- [[mount]] : 마운트 상태 확인
