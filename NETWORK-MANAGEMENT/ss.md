---
command: ss
category: NETWORK-MANAGEMENT
aliases: []
tags:
  - linux/network
  - task/diagnose
  - topic/port
  - topic/socket
  - privilege/mixed
  - replaces/netstat
related: ["[[ssh]]", "[[systemctl]]", "[[firewall-cmd]]", "[[ping]]", "[[grep]]", "[[ip]]", "[[lsof]]", "[[curl]]", "[[ps]]"]
distro: 전체 (iproute2 패키지)
verified: Rocky Linux 9.6
updated: 2026-07-30
---

# ss

- 소켓 상태 조회 도구
- 어원: **s**ocket **s**tatistics
- `netstat` 대체 (iproute2 패키지), 대규모 연결에서 더 빠름

---

## ss -tlnp

```bash
ss -tlnp

# Examples
ss -tlnp                     # TCP 리스닝 포트 + 프로세스
ss -tlnp | grep :22          # SSH 포트 리스닝 확인
ss -tulnp                    # TCP + UDP 동시
ss -tan                      # 전체 TCP 연결 상태
```

### 명령어 설명
- 사용 목적
	- 서비스가 특정 포트에서 실제 리스닝 중인지 확인 시 사용
	- SSH·웹·DB 등 서비스 접근 불가 원인 진단 시 사용
	- 포트 점유 프로세스 식별 시 사용
- 특이사항
	- `0.0.0.0:22` → 전체 인터페이스에서 수신 (정상)
	- `127.0.0.1:22` → 로컬 전용 수신 → 외부 접속 불가
	- 프로세스명(`-p`) 표시에는 root 권한 필요
	- 리스닝 정상이나 외부 접속 실패 시 방화벽 확인 → [[firewall-cmd]]

### 옵션
- `-t` : TCP 소켓 (**t**cp)
- `-u` : UDP 소켓 (**u**dp)
- `-l` : 리스닝 상태만 (**l**istening)
- `-n` : 포트를 숫자로 표시, 이름 변환 생략 (**n**umeric)
- `-p` : 소켓 사용 프로세스 표시 (**p**rocess)
- `-a` : 전체 소켓 (**a**ll)

---

## 진단 순서 (서비스 접근 불가 시)

```bash
systemctl is-active sshd            # ① 서비스 기동 여부
ss -tlnp | grep :22                 # ② 포트 리스닝 여부
firewall-cmd --list-all | grep ssh  # ③ 방화벽 허용 여부
ping -c 3 <서버IP>                  # ④ 네트워크 도달 여부
```

---

## 연관 명령어
- [[systemctl]] : 서비스 기동 상태 확인·제어
- [[firewall-cmd]] : 포트·서비스 방화벽 허용
- [[ssh]] : 원격 접속 및 단계별 오류 진단
- [[ping]] : 네트워크 계층 도달성 확인
- [[grep]] : 포트 목록 필터링
- [[ip]] : 주소·라우팅·물리 링크 확인
- [[lsof]] : 포트 점유 프로세스 조회 — macOS 대안
- [[curl]] : HTTP 계층 응답 검증 — 리스닝 확인 후 수행
- [[ps]] : 리스닝 프로세스 상세 확인
