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
related: ["[[ss]]", "[[systemctl]]", "[[firewall-cmd]]", "[[usermod]]", "[[passwd]]", "[[sshpass]]", "[[curl]]", "[[timeout]]", "[[scp]]", "[[ssh-keygen]]"]
distro: 전체 (openssh-client / openssh-server)
verified: Rocky Linux 9.6
updated: 2026-08-01
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

## 원격 파일 반출 (sudo cat 스트림)

`/root` 등 접속 사용자 권한 밖 경로는 `scp` 실패 → `ssh + sudo cat` 스트림 우회.

```bash
# Examples
ssh -i <키> rocky@3.37.243.226 "sudo cat /root/ir-evidence.tar.gz" > ir-evidence.tar.gz
ssh-keygen -lf <키>              # 접속 실패 시 키 지문 확인 → 정상 키 식별
```

### 특이사항
- **명령 실행 위치 확인 필수** → SSH 접속된 서버 창에서 `scp`/`ssh` 실행 시 서버가 자기 자신에 접속 시도(개인키 부재로 `Permission denied (publickey)`). 반드시 로컬 셸에서 실행
- 개인키 권한 과다 개방 시 `bad permissions`로 무시 → `chmod 400` (디렉터리를 `-i`로 지정한 오류와 구분) → [[ssh-keygen]]
- 최초 접속 시 호스트 키 지문(`ED25519 key fingerprint`) 확인 프롬프트 → **정상 기준값으로 기록**. 한글 IME 상태에서 `yes` 입력 실패(`ㅛyes`) 빈발, 영문 전환 후 입력
- `~/.ssh/config`에 `Host` 별칭 등록 시 `ssh <별칭>`으로 단축

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
- [[scp]] : SSH 채널 파일 복사 — `/root`는 `sudo cat` 스트림으로 우회
- [[ssh-keygen]] : 키 지문 확인·식별, 권한 교정
