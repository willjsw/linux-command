---
command: grub2-install
category: BOOT-RECOVERY
aliases: [grub2-mkconfig, grub-install]
tags:
  - linux/boot
  - topic/bootloader
  - topic/grub
  - topic/mbr
  - topic/bls
  - task/recovery
  - privilege/root
  - distro/rhel
  - distro/rocky
related: ["[[chroot]]", "[[lsblk]]", "[[fdisk]]", "[[parted]]", "[[rescue-mode]]", "[[dnf]]", "[[mount]]", "[[dd]]"]
distro: RHEL 계열 (Rocky, CentOS)
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# grub2-install / grub2-mkconfig

- GRUB2 부트로더 설치 및 설정 생성 도구
- 어원: **GR**and **U**nified **B**ootloader
- 부팅 실패(부트로더 손상·미설치) 복구 시 사용

---

## ⚠ 실행 전 확인

```bash
lsblk                        # ① 디스크 장치명 확인 (USB 제외 필수)
ls /sys/firmware/efi         # ② 부팅 모드 판별
```

- `/sys/firmware/efi` **존재 → UEFI**, `No such file` **→ BIOS(레거시)**
- 모드에 따라 명령이 다름 → 오적용 시 복구 실패
- **설치 USB(`RM=1`)에 설치 금지** → USB 손상

---

## grub2-install (BIOS 모드)

```bash
grub2-install <device>

# Examples
grub2-install /dev/sda       # 첫 번째 HDD MBR에 설치
grub2-install /dev/sdc       # 두 번째 HDD에도 설치
```

### 명령어 설명
- 사용 목적
	- MBR에 부트로더 1단계 기록 시 사용
	- `Booting from Hard drive C:` 에서 멈추는 부팅 실패 복구 시 사용
- 특이사항
	- **디스크 2개 이상 구성 시 양쪽 모두 설치 권장**
		- RAID 컨트롤러(PERC 등)가 BIOS에 어느 디스크를 먼저 제시하는지 예측 불가
		- 양쪽 설치 시 순서와 무관하게 부팅 성공
	- **MBR 선두 512바이트만 사용** → 파티션 데이터 영향 없음
	- 정상 출력: `Installing for i386-pc platform.` / `Installation finished. No error reported.`
	- `i386-pc` = BIOS 모드 플랫폼 식별자 (UEFI는 `x86_64-efi`)
	- 부트 플래그와 무관하게 동작 → 플래그 설정은 [[parted]] 로 보조 조치 가능

---

## grub2-mkconfig (설정 생성)

```bash
grub2-mkconfig -o <output>

# Examples
grub2-mkconfig -o /boot/grub2/grub.cfg              # BIOS 모드
grub2-mkconfig -o /boot/efi/EFI/rocky/grub.cfg      # UEFI 모드
```

### 명령어 설명
- 사용 목적
	- 부팅 메뉴 설정 파일 재생성 시 사용
- 특이사항
	- **Rocky 9 에서 `Found linux image:` 미출력이 정상**
		- BLS(BootLoaderSpec) 방식 사용 → 부팅 항목을 `grub.cfg` 가 아닌 `/boot/loader/entries/*.conf` 로 관리
		- `/etc/default/grub` 의 `GRUB_ENABLE_BLSCFG=true` 가 해당 방식 지정
		- 정상 출력: `Generating grub configuration file ...` / `Adding boot menu entry for UEFI Firmware Settings ...` / `done`
	- 구버전(GRUB 레거시)에서는 `Found linux image:` 출력이 정상 → 버전 혼동 주의

### 옵션
- `-o <file>` : 출력 파일 경로 (**o**utput)

---

## UEFI 모드 복구

```bash
dnf reinstall -y grub2-efi-x64 shim-x64
grub2-mkconfig -o /boot/efi/EFI/rocky/grub.cfg
```

---

## 부팅 실패 복구 전체 절차

```bash
# ① 설치 USB로 부팅 → Troubleshooting → Rescue a Rocky Linux system → 1) Continue
# ② "Your system has been mounted under /mnt/sysroot." 확인 후 Enter

lsblk                                        # ③ 장치명 확인 (USB는 RM=1)
ls /sys/firmware/efi                         # ④ 부팅 모드 판별

chroot /mnt/sysroot                          # ⑤ 설치 시스템 진입

grub2-install /dev/sda                       # ⑥ 양쪽 HDD에 설치
grub2-install /dev/sdc
grub2-mkconfig -o /boot/grub2/grub.cfg       # ⑦ 설정 생성

exit
reboot                                       # ⑧ USB 제거 후 재부팅
```

### 상태 점검 명령
```bash
ls -l /boot                       # vmlinuz-*, initramfs-* 존재 확인
ls -l /boot/loader/entries/       # BLS .conf 파일 존재 확인
cat /etc/default/grub | grep -i bls    # GRUB_ENABLE_BLSCFG 확인
fdisk -l /dev/sda | tail -5       # 부트 플래그(*) 위치
```

---

## 연관 명령어
- [[rescue-mode]] : 레스큐 환경 진입 절차
- [[chroot]] : 설치 시스템으로 루트 전환
- [[lsblk]] : 대상 디스크 확인 (필수)
- [[fdisk]] : 부트 플래그 확인
- [[parted]] : 부트 플래그 설정
- [[dnf]] : UEFI 부트로더 패키지 재설치
- [[mount]] : 레스큐 환경 재마운트
- [[dd]] : MBR 초기화 후 재설치
