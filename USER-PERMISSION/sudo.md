---
command: sudo
category: USER-PERMISSION
aliases: [su]
tags:
  - linux/user
  - topic/sudo
  - topic/security
  - task/privilege-escalation
  - privilege/user
related: ["[[usermod]]", "[[id]]", "[[passwd]]", "[[ssh]]", "[[file-ops]]", "[[env]]"]
distro: 전체 (sudo 패키지)
verified: Rocky Linux 9.6
updated: 2026-07-30
---

# sudo

- 다른 사용자(기본 root) 권한으로 명령 실행 도구
- 어원: **s**uper**u**ser **do**
- RHEL 계열은 `wheel` 그룹 소속자에게 권한 부여

---

## sudo

```bash
sudo <command>

# Examples
sudo whoami                  # 권한 검증 ('root' 출력 시 정상)
sudo dnf install -y vim      # 관리자 권한 필요 명령 실행
sudo -i                      # root 로그인 셸 진입
sudo -u postgres psql        # 특정 사용자로 실행
```

### 명령어 설명
- 사용 목적
	- 일반 사용자 계정에서 관리자 권한 명령 실행 시 사용
	- root 직접 로그인 없이 서버 운영 시 사용 (보안 권장 방식)
- 특이사항
	- **본인 비밀번호 입력** (root 비밀번호 아님)
	- 인증 후 일정 시간(기본 5분) 재입력 생략
	- `wheel` 그룹 미소속 시 `user is not in the sudoers file` 오류 → [[usermod]] `-aG wheel` 필요
	- `/etc/sudoers` 직접 편집 불필요 → `%wheel ALL=(ALL) ALL` 기본 등록
	- 편집 필요 시 `visudo` 사용 (문법 검증 포함)

### 옵션
- `-i` : root 로그인 셸 진입 (**i**nitial login)
- `-u <user>` : 지정 사용자 권한으로 실행 (**u**ser)
- `-l` : 현재 계정의 허용 명령 목록 (**l**ist)
- `-k` : 인증 캐시 무효화 (**k**ill)

---

## su (사용자 전환)

```bash
su - <user>

# Examples
su - master        # master 계정으로 전환 (환경변수 포함)
su master          # 전환하되 기존 환경변수 유지
su -               # root 로 전환
```

### 명령어 설명
- 사용 목적
	- 다른 사용자 계정으로 셸 전환 시 사용
	- sudo 권한 부여 결과 검증 시 사용
- 특이사항
	- 어원: **s**witch **u**ser
	- **`-` 옵션 시 대상 계정의 환경변수·홈디렉터리 적용** (로그인 셸)
	- 전환 대상 계정의 비밀번호 입력 필요
	- `exit` 로 원래 계정 복귀

---

## sudo 권한 부여 및 검증 절차

```bash
usermod -aG wheel master     # ① wheel 그룹 등록
id master                    # ② groups 에 wheel 포함 확인
su - master                  # ③ 계정 전환
sudo whoami                  # ④ 'root' 출력 시 성공
exit
```

> 그룹 변경은 다음 로그인부터 적용 → 기존 세션은 재로그인 필요

---

## 연관 명령어
- [[usermod]] : `wheel` 그룹 등록으로 권한 부여
- [[id]] : 그룹 소속 확인
- [[passwd]] : 계정 비밀번호 설정
- [[ssh]] : 원격 접속 후 권한 상승
- [[file-ops]] : 시스템 경로 파일 조작 시 권한 상승 필요
- [[env]] : `sudo` 는 환경변수 일부 초기화 → 값 전달 주의
