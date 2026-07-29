---
command: ssh
category: NETWORK-MANAGEMENT
aliases: [openssh]
tags:
  - linux/network
  - topic/remote-access
  - topic/security
  - task/connect
  - task/diagnose
  - privilege/user
related: ["[[ss]]", "[[systemctl]]", "[[firewall-cmd]]", "[[usermod]]", "[[passwd]]", "[[sshpass]]", "[[curl]]", "[[timeout]]"]
distro: 전체 (openssh-client / openssh-server)
verified: Rocky Linux 9.6
updated: 2026-07-30
---

# ssh

- 암호화 원격 셸 접속 도구
- 어원: **S**ecure **SH**ell
- 서버측 `openssh-server` 는 **최소 설치(Minimal) 환경에도 기본 포함·자동 시작** → 별도 설치 불필요

---

## ssh (접속)

```bash
ssh <user>@<host>

# Examples
ssh master@10.1.18.85            # 일반 사용자 접속
ssh -v master@10.1.18.85         # 상세 로그 출력 (오류 원인 진단)
ssh -p 2222 master@10.1.18.85    # 비표준 포트 접속
```

### 명령어 설명
- 사용 목적
	- 원격 서버 콘솔 접속 시 사용
	- GUI 없는 최소 설치 서버의 상시 운영 경로로 사용
- 특이사항
	- **root 계정은 비밀번호 로그인 기본 차단**
		- Rocky 9 의 `PermitRootLogin` 기본값 `prohibit-password`
		- 일반 사용자 접속 후 [[sudo]] 사용이 정석, 보안상 유지 권장
	- 접속 대상 계정에 비밀번호 미설정 시 인증 실패 → [[passwd]] 확인 필요
	- 접속 PC가 서버와 동일 대역이 아니면 라우팅·방화벽 정책에 차단 가능

### 옵션
- `-v` : 상세 로그 (**v**erbose), 중복 지정(`-vvv`) 시 상세도 증가
- `-p <port>` : 접속 포트 지정 (**p**ort)
- `-i <keyfile>` : 개인키 파일 지정 (**i**dentity)
- `-L` / `-R` : 로컬 / 원격 포트 포워딩

---

## 서버측 준비 상태 확인

```bash
rpm -q openssh-server        # 설치 여부
systemctl is-enabled sshd    # enabled → 재부팅 후 자동 시작
systemctl is-active sshd     # active
ss -tlnp | grep :22          # 0.0.0.0:22 리스닝 확인
firewall-cmd --list-all | grep -i ssh    # 방화벽 허용 여부
id master                    # 계정 존재 + wheel 그룹
passwd -S master             # 'P' = 비밀번호 설정 완료
```

---

## 접속 실패 원인 구분

`ssh -v` 출력 기준 판정.

| 증상 | 원인 |
| --- | --- |
| `Connection timed out` | 방화벽 차단 또는 네트워크 경로 문제 |
| `Connection refused` | sshd 미실행 (서비스 중지) |
| `Permission denied (publickey,password)` | 계정·비밀번호 문제 (서버 정상) |
| `No route to host` | 접속 PC가 다른 대역에 위치 |

### 조치 연결
- `timed out` → [[firewall-cmd]] 허용 확인, [[ping]] 도달성 확인
- `refused` → [[systemctl]] `start sshd`
- `Permission denied` → [[passwd]] 비밀번호 설정, [[usermod]] 그룹 확인
- `No route to host` → [[ip]] `route` 확인

---

## 연관 명령어
- [[ss]] : 22번 포트 리스닝 확인
- [[systemctl]] : sshd 서비스 기동·자동시작 설정
- [[firewall-cmd]] : ssh 서비스 방화벽 허용
- [[usermod]] : 접속 계정 `wheel` 그룹 등록
- [[passwd]] : 접속 계정 비밀번호 설정·상태 확인
- [[sudo]] : 접속 후 관리자 권한 실행
- [[sshpass]] : 비대화형 비밀번호 전달 — **키 인증 설정 시 불필요**
- [[curl]] : HTTP 계층 원격 검증 — SSH 불필요 시 대체
- [[timeout]] : 접속 무한 대기 차단
