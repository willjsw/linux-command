---
command: usermod
category: USER-PERMISSION
aliases: [useradd, userdel]
tags:
  - linux/user
  - topic/group
  - topic/sudo
  - task/configure
  - privilege/root
  - danger/data-loss
related: ["[[sudo]]", "[[passwd]]", "[[id]]", "[[ssh]]"]
distro: 전체 (shadow-utils 패키지)
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# usermod

- 기존 사용자 계정 속성 변경 도구
- 어원: **user** **mod**ify
- 계정 생성은 `useradd`, 삭제는 `userdel`

---

## usermod -aG (그룹 추가)

```bash
usermod -aG <group> <user>

# Examples
usermod -aG wheel master           # sudo 권한 부여 (RHEL 계열)
usermod -aG docker master          # docker 그룹 추가
usermod -aG wheel,docker master    # 다중 그룹 동시
```

### 명령어 설명
- 사용 목적
	- 사용자에게 sudo 권한 부여 시 사용 (`wheel` 그룹 등록)
	- 서비스별 접근 그룹 추가 시 사용
- 특이사항
	- **`-a` 누락 시 기존 그룹 전체 삭제** → 반드시 `-aG` 형태로 사용
	- RHEL 계열은 `wheel` 그룹이 sudo 권한 보유 (`/etc/sudoers` 에 `%wheel ALL=(ALL) ALL` 기본 등록)
	- **sudoers 파일 직접 편집 불필요**
	- 그룹 변경은 **다음 로그인부터 적용** → 기존 세션은 재로그인 필요

### 옵션
- `-a` : 기존 그룹 유지하며 추가 (**a**ppend) — **필수**
- `-G` : 보조 그룹 지정 (**G**roups)
- `-g` : 주 그룹 변경 (소문자, **g**roup)
- `-L` / `-U` : 계정 잠금 / 해제 (**L**ock / **U**nlock)
- `-s <shell>` : 로그인 셸 변경 (**s**hell)

---

## useradd (계정 생성)

```bash
useradd -m -G <group> <user>

# Examples
useradd -m -G wheel master     # 홈 디렉터리 + wheel 그룹으로 생성
passwd master                  # 비밀번호 설정 (별도 필수)
```

### 명령어 설명
- 사용 목적
	- 신규 사용자 계정 생성 시 사용
	- OS 설치 중 계정 미생성 시 사후 생성에 사용
- 특이사항
	- **비밀번호는 미설정 상태** → [[passwd]] 별도 실행 필수
	- 비밀번호 미설정 계정은 [[ssh]] 로그인 불가

### 옵션
- `-m` : 홈 디렉터리 생성 (**m**ake home)
- `-G <group>` : 보조 그룹 지정 (**G**roups)
- `-s <shell>` : 로그인 셸 지정 (**s**hell)

---

## 검증 절차

```bash
usermod -aG wheel master
id master                # groups 에 wheel 포함 확인
su - master              # 계정 전환
sudo whoami              # 'root' 출력 시 성공
exit
```

---

## 연관 명령어
- [[id]] : 그룹 등록 결과 확인
- [[passwd]] : 비밀번호 설정·상태 확인
- [[sudo]] : 부여된 권한 실제 사용
- [[ssh]] : 원격 접속 계정 준비
