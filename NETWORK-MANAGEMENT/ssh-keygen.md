---
command: ssh-keygen
category: NETWORK-MANAGEMENT
aliases: []
tags:
  - linux/network
  - linux/security
  - task/verify
  - task/inspect
  - topic/remote-access
  - topic/security
  - privilege/user
related: ["[[ssh]]", "[[scp]]", "[[grep]]", "[[stat]]"]
distro: 전체 (openssh)
verified: macOS (Darwin 25.5) (사고 대응 세션)
updated: 2026-08-01
---

# ssh-keygen

- SSH 키 생성·지문 조회·known_hosts 관리 도구
- 어원: **ssh** + **key** **gen**erator
- 다수 키 파일 중 특정 키 식별 시 지문(`-lf`) 대조에 사용

---

## ssh-keygen -lf

```bash
ssh-keygen -lf <키파일>

# Examples
ssh-keygen -lf ./bandage-dev.pem      # 키 지문(SHA256) 출력
grep -l "PRIVATE KEY" * 2>/dev/null   # 디렉터리 내 개인키 파일 후보 식별
chmod 400 <키파일>                    # 개인키 권한 교정 (too open 오류 해소)
```

### 명령어 설명
- 사용 목적
	- 키 파일의 지문 출력으로 정상 키 식별 시 사용
	- `/var/log/secure`의 `Accepted publickey ... SHA256:...` 기록과 대조 시 사용
- 특이사항
	- `-lf` 결과 지문이 로그·기준값과 일치하는 파일이 해당 키
	- **개인키 본문은 입력·전달 대상 아님** → `ssh -i`는 파일 경로만 지정 → [[ssh]]
	- 개인키 권한이 `0755` 등 과다 개방 시 `bad permissions`로 무시 → `chmod 400` 필요
		- 디렉터리를 `-i`로 지정한 오류와 혼동 주의 (디렉터리 기본 권한이 0755)
	- 개인키는 `~/.ssh/`(0700)에 0400으로 보관 권장 → `~/Desktop` 등 0755 경로 지양

### 옵션
- `-l` : 키 지문 출력 (**l**ist fingerprint)
- `-f` : 대상 키 파일 지정 (**f**ile)

---

## 연관 명령어
- [[ssh]] : 지문 확인 후 `-i`로 키 지정 접속
- [[scp]] : 동일 키로 파일 전송
- [[grep]] : 키 파일 후보 식별 (`grep -l "PRIVATE KEY"`)
- [[stat]] : 키 파일 권한(0400) 확인
