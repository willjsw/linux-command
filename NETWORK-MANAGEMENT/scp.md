---
command: scp
category: NETWORK-MANAGEMENT
aliases: []
tags:
  - linux/network
  - task/connect
  - topic/remote-access
  - topic/security
  - privilege/user
related: ["[[ssh]]", "[[ssh-keygen]]", "[[sha256sum]]", "[[tar]]", "[[sudo]]"]
distro: 전체 (openssh-client)
verified: macOS (Darwin 25.5) ↔ Rocky Linux 9.6 (사고 대응 세션)
updated: 2026-08-01
---

# scp

- SSH 채널 기반 파일 복사 도구
- 어원: **s**ecure **c**o**p**y
- `/root` 등 원격 권한 부재 경로는 복사 실패 → `ssh + sudo cat` 스트림으로 우회

---

## scp

```bash
scp -i <키> <사용자>@<호스트>:<원격경로> <로컬경로>

# Examples
scp -i ~/.ssh/bandage-dev.pem rocky@3.37.243.226:/home/rocky/file .   # 원격 → 로컬
# /root 경로는 scp 실패 → ssh + sudo cat 우회
ssh -i <키> rocky@3.37.243.226 "sudo cat /root/ir-evidence.tar.gz" > ir-evidence.tar.gz
```

### 명령어 설명
- 사용 목적
	- 원격 서버와 파일 송수신 시 사용
	- 증거 아카이브 반출 시 사용
- 특이사항
	- **`scp`는 접속 사용자 권한으로 원격 파일 읽음** → `/root` 읽기 권한 부재 시 실패
		- 대응: `ssh -i <키> <사용자>@<호스트> "sudo cat <파일>" > <로컬파일>` 스트림 방식
	- **명령 실행 위치 확인 필수** → SSH 접속된 서버 창에서 실행 시 서버가 자기 자신에 접속 시도(개인키 부재로 실패). 반드시 로컬 셸에서 실행 → [[ssh]]
	- 스트림 우회 실패 시 0바이트·오류 메시지 파일 생성 → `file`로 `gzip compressed data` 여부, `head -c 200`으로 오류 포함 여부 확인
	- 전송 후 무결성 검증 필수 → [[sha256sum]]

### 옵션
- `-i` : 인증 개인키 파일 경로 (**i**dentity)
- `-r` : 디렉터리 재귀 복사 (**r**ecursive) ※ 미검증
- `-P` : 원격 SSH 포트 지정 (대문자, ssh는 소문자 `-p`) ※ 미검증

---

## 연관 명령어
- [[ssh]] : 원격 접속 및 `sudo cat` 스트림 우회 기반
- [[ssh-keygen]] : 인증 키 지문 확인·식별
- [[sha256sum]] : 전송 무결성 검증
- [[tar]] : 전송 대상 증거 아카이브 생성
- [[sudo]] : 원격 `/root` 파일 읽기 권한 확보
