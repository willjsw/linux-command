---
command: lsof
category: PROCESS-MANAGEMENT
aliases: []
tags:
  - linux/process
  - linux/network
  - task/inspect
  - task/diagnose
  - topic/port
  - topic/socket
  - topic/troubleshooting
  - privilege/mixed
related: ["[[ss]]", "[[kill]]", "[[ps]]", "[[xargs]]", "[[awk]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# lsof

- 열린 파일·소켓과 점유 프로세스 조회 도구
- 어원: **l**i**s**t **o**pen **f**iles
- 자기 소유 프로세스는 일반 사용자, 타 사용자 프로세스 포함 전체 조회는 root 필요

---

## lsof 포트 점유 조회

```bash
lsof -nP -iTCP:<포트> -sTCP:LISTEN

# Examples
lsof -nP -iTCP:8080 -sTCP:LISTEN 2>/dev/null | head -2 || echo "8080 리스너 없음"
lsof -nP -iTCP:5432 -sTCP:LISTEN 2>/dev/null | head            # DB 포트 확인
lsof -nP -iTCP:8081 -sTCP:LISTEN 2>/dev/null | head -2 || echo "8081 free"
lsof -nP -iTCP:5432 -sTCP:LISTEN | awk '{print $1, $2, $9}'    # 명령·PID·주소만
lsof -ti:8080 | xargs -r kill 2>/dev/null                       # 점유 프로세스 종료
lsof -ti:8080 >/dev/null 2>&1                                   # 점유 여부만 판정
```

### 명령어 설명
- 사용 목적
	- 포트 점유 프로세스 식별 시 사용 (`Address already in use` 진단)
	- 애플리케이션 리스닝 개시 여부 확인 시 사용
	- 재기동 전 포트 해제 여부 확인 시 사용
- 특이사항
	- **`-n` 미지정 시 DNS 역조회 발생 → 응답 지연** → 진단 시 `-nP` 관용 조합
		- `-n` : 호스트명 미해석 (IP 그대로)
		- `-P` : 포트명 미해석 (번호 그대로, `5432` → `postgresql` 방지)
	- **`-t` 는 PID 만 출력** → [[xargs]] `kill` 파이프 전용
		- 헤더·컬럼 없음 → 스크립트 파싱 불필요
	- 대상 미존재 시 종료 코드 1, 출력 없음 → `|| echo` 로 부재 판정 관용
	- root 미사용 시 타 사용자 프로세스 누락 → **점유 중이나 미표시 가능** ⚠
	- Linux 에서는 [[ss]] `-tlnp` 가 경량·기본 설치 → 서버 환경 우선 사용
	- macOS 는 `ss` 미제공 → `lsof` 가 표준 수단

### 옵션
- `-n` : 호스트명 역조회 생략 (**n**o DNS)
- `-P` : 포트 번호를 서비스명으로 변환 안 함 (**P**ort number)
- `-i` : 네트워크 소켓 대상 (**i**nternet) — `-iTCP:8080` 형태
- `-sTCP:LISTEN` : TCP 상태 필터 (**s**tate) — 리스닝만
- `-t` : PID 만 간결 출력 (**t**erse) — `xargs kill` 연계
- `-ti:<포트>` : 해당 포트 점유 PID 만 출력 — 최빈출 조합
- `-p <PID>` : 특정 프로세스가 연 파일 목록 (**p**id) ※ 미검증
- `-u <user>` : 특정 사용자 대상 (**u**ser) ※ 미검증

---

## 연관 명령어
- [[ss]] : Linux 표준 소켓 조회 — 서버 환경 우선, `lsof` 는 macOS 대안
- [[kill]] : `lsof -ti:포트 | xargs kill` 점유 해제 패턴
- [[ps]] : `lsof` 로 취득한 PID 상세 확인
- [[xargs]] : `-t` 출력 PID 를 명령 인자로 변환
- [[awk]] : `lsof` 출력에서 필요 컬럼만 추출
