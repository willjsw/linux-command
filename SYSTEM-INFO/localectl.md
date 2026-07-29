---
command: localectl
category: SYSTEM-INFO
aliases: [locale, timedatectl]
tags:
  - linux/systemd
  - topic/locale
  - topic/encoding
  - task/configure
  - privilege/root
related: ["[[dnf]]", "[[hostnamectl]]", "[[systemctl]]", "[[env]]", "[[iconv]]", "[[which]]"]
distro: systemd 사용 배포판
verified: Rocky Linux 9.6
updated: 2026-07-30
---

# localectl

- 시스템 로케일·키보드 레이아웃 조회·설정 도구
- 어원: **locale** **ctl**(control)
- 관련: `timedatectl`(시각·시간대), `hostnamectl`(호스트명)

---

## localectl

```bash
localectl

# Examples
localectl                                # 현재 로케일·키맵
localectl list-locales                   # 사용 가능 로케일 목록
localectl set-locale LANG=C.UTF-8        # 영문 로케일 고정
localectl set-locale LANG=ko_KR.UTF-8    # 한국어 로케일
localectl set-keymap us                  # 키보드 레이아웃
```

### 명령어 설명
- 사용 목적
	- 시스템 로케일·키보드 레이아웃 설정 시 사용
	- 최소 설치 환경의 로케일 경고 해소 시 사용
- 특이사항
	- **최소 설치 환경에서 `Failed to set locale, defaulting to C.UTF-8` 경고 발생**
		- 원인: 한국어 로케일 데이터(`glibc-langpack-ko`) 미포함
		- `dnf` 실행 시마다 출력되나 **동작에는 무영향**
		- 해소: `dnf install -y glibc-langpack-ko`
	- 서버 로그를 영문으로 유지하려면 `LANG=C.UTF-8` 고정이 검색·문서 대조에 유리
	- **한국어 키보드 레이아웃(`kr`) 환경에서 특수문자 위치 상이** → 비밀번호 입력 실패 원인

---

## timedatectl

```bash
timedatectl

# Examples
timedatectl                                  # 현재 시각·시간대·NTP 상태
timedatectl set-timezone Asia/Seoul          # 시간대 설정
timedatectl list-timezones | grep Seoul      # 시간대 검색
timedatectl set-ntp true                     # NTP 동기화 활성화
```

### 명령어 설명
- 사용 목적
	- 시스템 시간대 설정 시 사용
	- OS 설치 중 시간대 미설정 시 사후 설정에 사용
	- NTP 시각 동기화 상태 확인 시 사용

---

## 연관 명령어
- [[dnf]] : `glibc-langpack-ko` 설치로 로케일 경고 해소
- [[hostnamectl]] : 호스트 이름 설정 (동일 계열 도구)
- [[systemctl]] : systemd 서비스 관리
- [[env]] : `env LC_ALL=C` 명령 단위 로케일 재정의
- [[iconv]] : 파일 인코딩 변환 — 로케일과 별개 계층
- [[which]] : 로케일 관련 도구 설치 여부 확인
