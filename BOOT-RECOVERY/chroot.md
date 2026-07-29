---
command: chroot
category: BOOT-RECOVERY
aliases: []
tags:
  - linux/boot
  - task/recovery
  - topic/rescue
  - privilege/root
  - topic/troubleshooting
related: ["[[mount]]", "[[rescue-mode]]", "[[grub2-install]]", "[[passwd]]", "[[lsblk]]"]
distro: 전체 (coreutils 패키지)
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# chroot

- 루트 디렉터리를 지정 경로로 변경해 해당 시스템 내부처럼 명령 실행하는 도구
- 어원: **ch**ange **root**
- 레스큐·복구 작업의 핵심 명령

---

## chroot

```bash
chroot <newroot>

# Examples
chroot /mnt/sysroot        # 레스큐 모드 (설치 매체 부팅)
chroot /sysroot            # rd.break 단일사용자 모드
```

### 명령어 설명
- 사용 목적
	- 레스큐 환경에서 **디스크에 설치된 시스템 내부로 진입** 시 사용
	- 부팅 불가 시스템의 부트로더·비밀번호·패키지 복구 시 사용
- 특이사항
	- 진입 후 명령은 **설치된 시스템 기준으로 동작** → `/boot`, `/etc` 등이 실제 시스템 경로를 가리킴
	- 진입 전 대상이 **쓰기 가능 마운트 상태**여야 함 → [[mount]] `-o remount,rw` 선행 필요
	- `exit` 으로 복귀
	- 마운트 경로가 환경별로 다름
		- 레스큐 모드(설치 USB) : `/mnt/sysroot`
		- `rd.break` 모드 : `/sysroot`

---

## 사용 맥락별 절차

### 부트로더 복구 (레스큐 모드)
```bash
# 설치 USB → Troubleshooting → Rescue a Rocky Linux system → 1) Continue
lsblk                                     # 장치명 확인
chroot /mnt/sysroot
grub2-install /dev/sda
grub2-mkconfig -o /boot/grub2/grub.cfg
exit
reboot
```

### root 비밀번호 재설정 (rd.break 모드)
```bash
# GRUB 메뉴 'e' → linux 줄 끝에 'rd.break enforcing=0' 추가 → Ctrl+X
mount -o remount,rw /sysroot              # 쓰기 가능 전환 (필수)
chroot /sysroot
passwd root
touch /.autorelabel                       # SELinux 재레이블 예약 (필수)
exit
exit
```

---

## 연관 명령어
- [[mount]] : 진입 전 쓰기 가능 재마운트
- [[rescue-mode]] : 레스큐 환경 진입 방법
- [[grub2-install]] : chroot 내부에서 부트로더 복구
- [[passwd]] : chroot 내부에서 비밀번호 재설정
- [[lsblk]] : 마운트 상태·장치명 확인
