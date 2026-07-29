---
command: hostnamectl
category: NETWORK-MANAGEMENT
aliases: []
tags:
  - linux/network
  - linux/systemd
  - task/configure
  - topic/hostname
  - privilege/root
related: ["[[nmcli]]", "[[nmtui]]", "[[systemctl]]", "[[localectl]]"]
distro: systemd 사용 배포판
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# hostnamectl

- systemd 계열 호스트 이름 조회·설정 도구
- 어원: **hostname** **ctl**(control)
- `/etc/hostname` 직접 편집 대체

---

## hostnamectl (조회)

```bash
hostnamectl

# Examples
hostnamectl                  # 호스트명·OS·커널·아키텍처 정보
hostnamectl hostname         # 호스트명만 출력
```

### 명령어 설명
- 사용 목적
	- 현재 호스트 이름 및 시스템 기본 정보 확인 시 사용
- 특이사항
	- 설치 시 호스트명 미설정 상태에서는 `localhost` 표시 → 프롬프트가 `[root@localhost ~]#` 형태

---

## hostnamectl set-hostname

```bash
hostnamectl set-hostname <name>

# Examples
hostnamectl set-hostname demo-iccs-01     # 사내 네이밍 규칙 적용
```

### 명령어 설명
- 사용 목적
	- 서버 식별용 호스트 이름 영구 설정 시 사용
	- OS 설치 중 호스트명 미입력 시 사후 설정에 사용
- 특이사항
	- 즉시 적용, 재부팅 불필요
	- 셸 프롬프트 반영에는 재로그인 필요
	- `/etc/hostname` 파일에 기록
	- [[nmtui]] 의 `Set system hostname` 메뉴로도 동일 설정 가능

---

## 연관 명령어
- [[nmcli]] : 네트워크 IP·DNS 설정
- [[nmtui]] : 호스트명 포함 대화형 네트워크 설정
- [[systemctl]] : systemd 서비스 관리
- [[localectl]] : 로케일·시간대 설정 (동일 계열)
