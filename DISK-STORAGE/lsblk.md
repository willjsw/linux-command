---
command: lsblk
category: DISK-STORAGE
aliases: []
tags:
  - linux/disk
  - task/inspect
  - topic/partition
  - topic/lvm
  - privilege/user
related: ["[[df]]", "[[fdisk]]", "[[lvm]]", "[[dd]]", "[[parted]]", "[[chroot]]", "[[mount]]", "[[du]]", "[[dmesg]]", "[[ls]]", "[[stat]]"]
distro: 전체 (util-linux 패키지)
verified: Rocky Linux 9.6
updated: 2026-07-30
---

# lsblk

- 블록 디바이스(디스크·파티션·LVM) 트리 구조 출력 도구
- 어원: **l**i**s**t **bl**oc**k** devices
- 일반 사용자 실행 가능

---

## lsblk

```bash
lsblk

# Examples
lsblk                        # 전체 블록 디바이스 트리
lsblk -f                     # 파일시스템·UUID·라벨 포함
lsblk -o NAME,SIZE,RM,TYPE,MOUNTPOINTS   # 출력 컬럼 지정
```

### 출력 예시
```
NAME        SIZE RM TYPE MOUNTPOINTS
sda       558.9G  0 disk
└─sda1    508.4G  0 part
  ├─rl-root  100G  0 lvm  /
  ├─rl-app   200G  0 lvm  /app
  └─rl-home  100G  0 lvm  /home
sdb        14.7G  1 disk                  ← RM=1 : 설치 USB
└─sdb1     14.7G  1 part /run/install/repo
sdc       558.9G  0 disk
├─sdc1        1G  0 part /boot
└─sdc2    507.4G  0 part
  ├─rl-data  400G  0 lvm  /data
  └─rl-swap 15.7G  0 lvm  [SWAP]
```

### 명령어 설명
- 사용 목적
	- 디스크·파티션·LVM 논리볼륨 구성 파악 시 사용
	- **디스크 장치명(`sda`/`sdb`/`sdc`) 실제 확인** 시 사용
	- 파괴적 명령 실행 전 대상 검증 시 사용
- 특이사항
	- **디스크 장치명은 고정값 아님** → 인식 순서에 따라 변동
		- 설치 USB 연결 시 USB가 `sdb` 점유 → 두 번째 HDD가 `sdc` 로 이동
		- 설치 화면에서 `sda`/`sdb` 로 보인 디스크가 레스큐 모드에서 `sda`/`sdc` 로 변경 가능
	- **`RM` 컬럼 = 제거 가능 미디어 여부** → `1` 이면 USB
	- [[dd]] · [[grub2-install]] 등 대상 지정 명령 실행 전 **매번 확인 필수**

### 옵션
- `-f` : 파일시스템 정보 표시 (**f**ilesystem)
- `-o <컬럼>` : 출력 컬럼 지정 (**o**utput)
- `-p` : 전체 장치 경로 표시 (`/dev/sda`)

### 주요 컬럼
- `NAME` : 장치명
- `SIZE` : 용량
- `RM` : 제거 가능 미디어 (1 = USB 등)
- `RO` : 읽기 전용
- `TYPE` : `disk` / `part` / `lvm` / `rom`
- `MOUNTPOINTS` : 마운트 위치

---

## 연관 명령어
- [[df]] : 마운트된 파일시스템 사용량 확인
- [[fdisk]] : 파티션 테이블·부트 플래그 상세 확인
- [[lvm]] : LVM 볼륨그룹·논리볼륨 조회 및 확장
- [[dd]] : 디스크 초기화 (대상 확인에 lsblk 필수)
- [[parted]] : 파티션 조작·부트 플래그 설정
- [[chroot]] : 레스큐 작업 시 마운트 상태 확인
- [[mount]] : 마운트·재마운트
- [[du]] : 디렉터리 단위 사용량 집계
- [[dmesg]] : 디스크 인식 커널 메시지 확인
- [[ls]] : `/dev` 하위 장치 노드 목록 확인
- [[stat]] : 장치 노드 속성 확인
