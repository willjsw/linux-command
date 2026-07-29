---
command: mount
category: DISK-STORAGE
aliases: [umount, remount]
tags:
  - linux/disk
  - topic/filesystem
  - task/mount
  - task/recovery
  - privilege/root
related: ["[[lsblk]]", "[[df]]", "[[chroot]]", "[[grub2-install]]", "[[passwd]]", "[[rescue-mode]]"]
distro: 전체 (util-linux 패키지)
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# mount

- 파일시스템을 디렉터리 트리에 연결하는 도구
- 어원: 물리 매체를 시스템에 "장착"하는 개념
- 영구 마운트는 `/etc/fstab` 등록 필요

---

## mount (조회)

```bash
mount

# Examples
mount | grep boot        # 특정 마운트 확인
mount | column -t        # 정렬 출력
```

### 명령어 설명
- 사용 목적
	- 현재 마운트된 파일시스템 목록·옵션 확인 시 사용
	- 레스큐 모드에서 `/boot` 등 정상 마운트 여부 검증 시 사용
- 특이사항
	- 용량 중심 확인은 [[df]] 사용

---

## mount (마운트)

```bash
mount <device> <mountpoint>

# Examples
mount /dev/sdc1 /mnt/boot          # 장치를 디렉터리에 연결
mount -o remount,rw /sysroot       # 읽기전용 → 쓰기가능 재마운트
mount -o remount,rw /              # 루트 재마운트
```

### 명령어 설명
- 사용 목적
	- 파일시스템을 디렉터리에 연결 시 사용
	- **레스큐·단일사용자 모드에서 읽기전용 루트를 쓰기가능으로 전환** 시 사용
- 특이사항
	- `rd.break` 부팅 시 `/sysroot` 는 읽기전용 상태 → 비밀번호 변경 전 `remount,rw` 필수
	- 재부팅 시 해제 → 영구 적용은 `/etc/fstab` 등록

### 옵션
- `-o <옵션>` : 마운트 옵션 지정 (**o**ptions)
	- `remount` : 마운트 상태 변경
	- `rw` / `ro` : 읽기쓰기 / 읽기전용
	- `bind` : 디렉터리 바인드 마운트
- `-t <type>` : 파일시스템 종류 지정 (**t**ype)
- `-a` : `/etc/fstab` 전체 마운트 (**a**ll)

---

## umount

```bash
umount <mountpoint>

# Examples
umount /mnt/boot
umount -l /mnt/boot      # 지연 해제 (사용 중일 때)
```

### 명령어 설명
- 사용 목적
	- 마운트 해제 시 사용
- 특이사항
	- 명령명에 `n` 없음 (`unmount` 아님)
	- 해당 경로 사용 중이면 `target is busy` 오류 → `-l` 또는 사용 프로세스 종료 필요

---

## 레스큐 모드 활용 절차

```bash
mount -o remount,rw /sysroot     # ① 쓰기 가능 전환
chroot /sysroot                  # ② 설치 시스템 진입
passwd root                      # ③ 작업 수행
touch /.autorelabel              # ④ SELinux 재레이블 예약
exit                             # ⑤ 종료
```

---

## 연관 명령어
- [[lsblk]] : 마운트 대상 장치명·마운트 지점 확인
- [[df]] : 마운트된 파일시스템 용량 확인
- [[chroot]] : 마운트한 시스템으로 루트 전환
- [[grub2-install]] : 레스큐 환경에서 부트로더 복구
- [[passwd]] : 레스큐 모드 비밀번호 재설정
- [[rescue-mode]] : 레스큐 환경 진입 절차
