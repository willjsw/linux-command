---
command: nmtui
category: NETWORK-MANAGEMENT
aliases: []
tags:
  - linux/network
  - topic/networkmanager
  - topic/static-ip
  - task/configure
  - interface/tui
  - privilege/root
  - distro/rhel
  - distro/rocky
related: ["[[nmcli]]", "[[ip]]", "[[hostnamectl]]"]
distro: NetworkManager 사용 배포판
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# nmtui

- NetworkManager 텍스트 기반 대화형 설정 도구
- 어원: **N**etwork**M**anager **T**ext **U**ser **I**nterface
- [[nmcli]] 와 동일 기능, 화살표·엔터 조작 → 오타 위험 없음
- 최소 설치(Minimal) 환경에도 기본 포함

---

## nmtui

```bash
nmtui

# 메뉴 구성
# - Edit a connection      : 연결 프로파일 편집 (IP·게이트웨이·DNS)
# - Activate a connection  : 연결 활성/비활성
# - Set system hostname    : 호스트 이름 설정
```

### 명령어 설명
- 사용 목적
	- CLI 옵션 암기 없이 고정 IP·DNS 설정 시 사용
	- 콘솔 환경(GUI 없음)에서 네트워크 초기 설정 시 사용
	- 명령어 입력 미숙 상태에서 설정 오류 방지 목적으로 사용
- 특이사항
	- OS 설치 화면(Anaconda)의 [연결 편집] 창과 동일 레이아웃
	- 설정 후 `Activate a connection` 에서 재활성화 필요
	- 변경 결과는 [[nmcli]] `connection show` 로 확인 가능

### 고정 IP 설정 절차
```
nmtui
 → Edit a connection
 → 대상 인터페이스 선택 (예: eno1)
 → IPv4 설정: <자동> → <수동> 변경
 → 주소     <추가...>  10.1.18.85/24
 → 게이트웨이           10.1.18.254
 → DNS 서버 <추가...>  154.10.6.11
            <추가...>  8.8.8.8
 → [*] 자동으로 연결   ← 체크 필수 (화면 하단, 스크롤 필요)
 → <확인>
 → 뒤로 → Activate a connection → 대상 선택 후 활성화
```

### 주의사항
- **`[*] 자동으로 연결` 체크 필수**
	- [[nmcli]] 의 `connection.autoconnect yes` 와 동일 설정
	- 화면 하단에 위치해 누락 빈발
	- 누락 시 재부팅 후 네트워크 미기동 → 원격 접속 차단

---

## 연관 명령어
- [[nmcli]] : 동일 작업의 CLI 버전, 스크립트 자동화에 적합
- [[ip]] : 적용 결과(주소·라우팅) 확인
- [[hostnamectl]] : 호스트 이름 단독 설정
