---
command: rescue-mode
category: BOOT-RECOVERY
aliases: [rd.break, single-user-mode, grub-edit]
tags:
  - linux/boot
  - topic/rescue
  - topic/grub
  - topic/kernel-parameter
  - task/recovery
  - privilege/root
  - distro/rhel
  - distro/rocky
related: ["[[chroot]]", "[[mount]]", "[[grub2-install]]", "[[passwd]]", "[[getenforce]]"]
distro: RHEL 계열 (Rocky, CentOS)
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# 레스큐 모드 / 단일 사용자 모드

- 부팅 불가·로그인 불가 시스템 복구용 환경 진입 방법
- 명령어가 아닌 **절차** 문서 (커널 파라미터·부팅 메뉴 조작)
- 두 방식 존재: **설치 매체 레스큐** / **GRUB `rd.break`**

---

## 방식 1. 설치 매체 레스큐 (부트로더 복구용)

설치 USB 필요. 부팅 자체가 불가한 경우 사용.

```
① 설치 USB로 부팅 (F11 → USB 선택)
② 부팅 메뉴 → Troubleshooting
③ Rescue a Rocky Linux system
④ 1) Continue 선택
```

### 진입 성공 메시지
```
Your system has been mounted under /mnt/sysroot.
```

- 위 메시지 출력 시 기존 설치를 정상 인식한 상태 → 설치 자체는 온전, 부트로더만 문제
- `Enter` 로 셸 진입 → `sh-5.1#` 프롬프트

### 선택지 의미
- `1) Continue` : 읽기·쓰기로 마운트 (일반 복구)
- `2) Read-only mount` : 읽기 전용 마운트
- `3) Skip to shell` : 마운트 없이 셸 진입
- `4) Quit (Reboot)` : 재부팅

### 주의사항
- **레스큐 환경은 USB에 계속 의존** → 작업 중 USB 제거 금지
- USB 읽기 불량 시 `reboot` 실행 후 무한 오류 발생
	- `SQUASHFS error: Failed to read block ... -5`
	- 대응: 전원 5초 이상 강제 종료 → USB 제거 → 재부팅
- 사용 후 첫 부팅에서 **SELinux autorelabel** 수분 소요, 완료 후 자동 재부팅 1회 추가

---

## 방식 2. GRUB rd.break (비밀번호 재설정용)

설치 매체 불필요. 부팅은 되나 로그인 불가한 경우 사용.

```
① 재부팅 → GRUB 메뉴에서 대상 커널에 커서 → 'e' 키
② 'linux' 로 시작하는 줄 맨 끝으로 이동 (End 키)
③ 아래 파라미터 추가 (앞에 공백 한 칸)
     rd.break enforcing=0
④ Ctrl+X (또는 F10) 으로 부팅
⑤ switch_root:/# 프롬프트 진입
```

### 편집 화면 구조
```
linux ($root)/vmlinuz-5.14.0-570.17.1.el9_6.x86_64 root=/dev/mapper/rl-root ro
  ... rd.lvm.lv=rl/swap rd.break enforcing=0        ← 여기에 추가
initramfs ($root)/initramfs-5.14.0-570.17.1.el9_6.x86_64.img
```

- 줄이 길어 화면에서 여러 줄로 감기지만 **실제로는 한 줄**
- `initramfs` 줄보다 **앞**에 위치해야 함
- 마우스 미지원 → 방향키·`End` 키 사용
- GRUB 메뉴가 빠르게 지나갈 경우 부팅 직후 `Esc` 또는 방향키 연타

### 커널 파라미터 의미
- `rd.break` : initramfs 단계에서 부팅 중단 (**r**am**d**isk break)
- `enforcing=0` : SELinux 일시 비활성화 → 파일 레이블 차단 방지

### 후속 절차
```bash
mount -o remount,rw /sysroot     # 쓰기 가능 전환 (필수)
chroot /sysroot                  # 설치 시스템 진입
passwd root                      # 비밀번호 재설정
touch /.autorelabel              # SELinux 재레이블 예약 (필수)
exit
exit                             # 자동 재부팅
```

### 주의사항
- **`enforcing=0` 및 `touch /.autorelabel` 누락 금지**
	- SELinux가 변경된 `/etc/shadow` 레이블 불일치를 차단으로 판정 → 로그인 재실패

---

## 방식 선택 기준

| 상황 | 방식 |
| --- | --- |
| 부팅 자체 불가 (GRUB 미표시, `Booting from Hard drive C:` 정지) | 방식 1 (설치 매체 레스큐) |
| 부팅 정상, 로그인 실패 (`Login incorrect`) | 방식 2 (GRUB `rd.break`) |
| 파일시스템 손상 검사 | 방식 1 (`3) Skip to shell` 후 `fsck`) |

---

## 연관 명령어
- [[chroot]] : 설치 시스템으로 루트 전환
- [[mount]] : 읽기전용 → 쓰기가능 재마운트
- [[grub2-install]] : 부트로더 재설치
- [[passwd]] : 비밀번호 재설정
- [[getenforce]] : SELinux 상태 확인
