---
command: passwd
category: USER-PERMISSION
aliases: []
tags:
  - linux/user
  - topic/password
  - topic/security
  - task/configure
  - task/recovery
  - privilege/mixed
related: ["[[usermod]]", "[[id]]", "[[sudo]]", "[[ssh]]", "[[chroot]]", "[[rescue-mode]]", "[[getenforce]]"]
distro: 전체 (shadow-utils 패키지)
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# passwd

- 사용자 비밀번호 설정·변경 도구
- 어원: **passw**or**d**
- 자신의 비밀번호는 일반 사용자 가능, 타 계정 변경은 root 필요

---

## passwd (설정·변경)

```bash
passwd [user]

# Examples
passwd              # 현재 로그인 계정 비밀번호 변경
passwd root         # root 비밀번호 변경 (root 권한 필요)
passwd master       # 특정 계정 비밀번호 설정
```

### 명령어 설명
- 사용 목적
	- 계정 비밀번호 신규 설정·변경 시 사용
	- `useradd` 로 생성한 계정에 비밀번호 부여 시 사용
	- 레스큐 모드에서 분실한 root 비밀번호 재설정 시 사용
- 특이사항
	- **입력 시 화면에 아무것도 표시되지 않음** (`*` 미표시) → 정상 동작, 입력 불가로 오인 빈발
	- 동일 비밀번호 2회 입력 필요
	- `BAD PASSWORD` 경고 출력 후에도 재입력 시 적용됨 (root 실행 시)
	- **한국어(`kr`) 키보드 레이아웃 환경에서 특수문자 위치 상이** → 콘솔 로그인 실패 원인, 영문+숫자 조합 권장

### 옵션
- `-S <user>` : 비밀번호 상태 조회 (**S**tatus)
- `-l <user>` : 계정 잠금 (**l**ock)
- `-u <user>` : 잠금 해제 (**u**nlock)
- `-d <user>` : 비밀번호 삭제 (**d**elete)
- `-e <user>` : 다음 로그인 시 변경 강제 (**e**xpire)

---

## passwd -S (상태 확인)

```bash
passwd -S <user>

# Examples
passwd -S master
# master P 2026-07-29 0 99999 7 -1      ← P = 비밀번호 설정 완료
```

### 상태 코드
- `P` : 비밀번호 설정 완료 (**P**assword set) → 정상 로그인 가능
- `LK` : 계정 잠김 (**L**oc**K**ed)
- `NP` : 비밀번호 없음 (**N**o **P**assword) → [[ssh]] 로그인 불가

### 명령어 설명
- 사용 목적
	- 계정 비밀번호 설정 여부 확인 시 사용
	- [[ssh]] 접속 실패 원인 진단 시 사용

---

## root 비밀번호 재설정 (레스큐 절차)

콘솔 로그인 `Login incorrect` 발생 시 설치 매체 없이 재설정 가능.

```bash
# ① 재부팅 → GRUB 메뉴에서 'e' 키
# ② linux 로 시작하는 줄 맨 끝에 추가:
#    rd.break enforcing=0
# ③ Ctrl+X 로 부팅 → switch_root:/# 프롬프트

mount -o remount,rw /sysroot     # ④ 쓰기 가능 전환
chroot /sysroot                  # ⑤ 설치 시스템 진입
passwd root                      # ⑥ 비밀번호 재설정
passwd master                    # (선택) 일반 계정도 함께
touch /.autorelabel              # ⑦ SELinux 재레이블 예약 — 필수
exit
exit                             # ⑧ 자동 재부팅
```

### 주의사항
- **`enforcing=0` 및 `touch /.autorelabel` 누락 금지**
	- SELinux가 변경된 `/etc/shadow` 의 레이블 불일치를 차단으로 판정 → 로그인 재차 실패
- 재부팅 후 SELinux 재레이블 수분 소요, 완료 후 자동 재부팅 1회 추가 발생

---

## 연관 명령어
- [[usermod]] : 계정 그룹·속성 변경
- [[id]] : 계정 존재·그룹 확인
- [[chroot]] : 레스큐 모드에서 설치 시스템 진입
- [[mount]] : 읽기전용 루트 쓰기 전환
- [[ssh]] : 원격 접속 인증
- [[rescue-mode]] : 레스큐 진입 절차
- [[getenforce]] : autorelabel 필요성
