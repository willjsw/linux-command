---
command: dd
category: DISK-STORAGE
aliases: []
tags:
  - linux/disk
  - task/wipe
  - task/clone
  - topic/mbr
  - topic/partition
  - privilege/root
  - danger/destructive
  - danger/irreversible
related: ["[[lsblk]]", "[[fdisk]]", "[[parted]]", "[[grub2-install]]"]
distro: 전체 (coreutils 패키지)
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# dd

- 블록 단위 저수준 데이터 복사·변환 도구
- 어원: **d**ata **d**efinition (별칭: "disk destroyer" — 오용 시 복구 불가)
- **파괴적·비가역 명령** → 실행 전 대상 확인 필수

---

## ⚠ 실행 전 필수 확인

```bash
lsblk
```

- **디스크 장치명은 고정값 아님** → 인식 순서에 따라 변동
- 설치 USB 연결 시 USB가 `sdb` 점유 → 두 번째 HDD가 `sdc` 로 이동
- `RM` 컬럼이 `1` 이고 용량이 작은 장치(예: 14.7G)가 USB
- **대상 오지정 시 데이터 전체 소실, 복구 불가**

---

## dd (파티션 헤더 초기화)

```bash
dd if=<input> of=<output> bs=<size> count=<n>

# Examples
dd if=/dev/zero of=/dev/sda bs=512 count=100    # MBR·파티션 테이블 영역 초기화
dd if=/dev/zero of=/dev/sdc bs=512 count=100
```

### 명령어 설명
- 사용 목적
	- 파티션 테이블·MBR 영역 초기화 시 사용
	- **Duplicate UUID 오류로 OS 설치 중단 시** 해소 목적으로 사용
	- 이전 OS 이미지 복제 잔재(동일 UUID) 제거 시 사용
- 특이사항
	- `bs=512 count=100` → 선두 51,200바이트만 초기화 (파티션 테이블 영역)
	- 디스크 전체 소거 아님 → 데이터 영역은 남으나 접근 불가 상태
	- 실행 후 파티션 재인식 필요 (설치 화면의 새로고침 또는 재부팅)
	- 진행률 미표시 → 대용량 작업 시 `status=progress` 추가

### 옵션
- `if=` : 입력 파일·장치 (**i**nput **f**ile), `/dev/zero` = 0으로 채움
- `of=` : 출력 파일·장치 (**o**utput **f**ile)
- `bs=` : 블록 크기 (**b**lock **s**ize)
- `count=` : 복사할 블록 개수
- `status=progress` : 진행 상황 표시

---

## Duplicate UUID 오류 해소 절차

OS 설치 화면 진입 전 아래 오류 발생 시 사용.

```
Duplicate UUID 'f480f2ac-01' found for devices: 'sdb1' and 'sda1'
```

```bash
# ① 설치 화면에서 Ctrl+Alt+F2 로 콘솔 진입
lsblk                                            # ② 대상 확인 (USB 제외)
dd if=/dev/zero of=/dev/sda bs=512 count=100     # ③ 초기화
dd if=/dev/zero of=/dev/sdc bs=512 count=100
# ④ Ctrl+Alt+F6 으로 그래픽 화면 복귀 → Retry
```

---

## 연관 명령어
- [[lsblk]] : 대상 장치명 확인 (실행 전 필수)
- [[fdisk]] : 초기화 결과 파티션 테이블 확인
- [[parted]] : 파티션 재생성
- [[grub2-install]] : MBR 초기화 후 부트로더 재설치
