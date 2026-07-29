---
command: id
category: USER-PERMISSION
aliases: [groups, whoami]
tags:
  - linux/user
  - topic/group
  - task/inspect
  - privilege/user
  - task/verify
  - topic/sudo
related: ["[[usermod]]", "[[sudo]]", "[[passwd]]"]
distro: 전체 (coreutils 패키지)
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# id

- 사용자 UID·GID·소속 그룹 출력 도구
- 어원: **id**entity
- 일반 사용자 실행 가능

---

## id

```bash
id [user]

# Examples
id                  # 현재 계정 정보
id master           # 특정 계정 정보
# uid=1000(master) gid=1000(master) groups=1000(master),10(wheel)
```

### 명령어 설명
- 사용 목적
	- 계정 존재 여부 확인 시 사용
	- **`wheel` 그룹 등록 결과 검증** 시 사용 → sudo 권한 확인
	- UID·GID 확인 시 사용
- 특이사항
	- 미존재 계정 조회 시 `no such user` 출력
	- `groups=` 항목에 `10(wheel)` 포함 시 sudo 권한 보유 (RHEL 계열)
	- 그룹 변경 직후에는 **기존 세션에 미반영** → 재로그인 후 확인 필요

### 옵션
- `-u` : UID만 출력 (**u**ser)
- `-g` : 주 GID만 출력 (**g**roup)
- `-G` : 전체 GID 출력 (**G**roups)
- `-n` : 숫자 대신 이름 출력 (**n**ame)

---

## whoami / groups

```bash
whoami         # 현재 계정명만 출력
groups         # 현재 계정 소속 그룹명만 출력
groups master  # 특정 계정 그룹
```

### 명령어 설명
- 사용 목적
	- `whoami` : 현재 실행 권한 주체 확인 시 사용 → [[sudo]] 동작 검증에 활용
	- `groups` : 소속 그룹만 간결히 확인 시 사용

---

## 연관 명령어
- [[usermod]] : 그룹 추가·계정 속성 변경
- [[sudo]] : 권한 상승 실행 및 검증
- [[passwd]] : 비밀번호 상태 확인
