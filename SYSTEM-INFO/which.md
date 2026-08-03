---
command: which
category: SYSTEM-INFO
aliases: [type, command, whereis]
tags:
  - linux/system
  - task/inspect
  - task/verify
  - topic/troubleshooting
  - privilege/user
related: ["[[timeout]]", "[[dnf]]", "[[rpm]]", "[[file]]", "[[env]]", "[[ldd]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# which

- `PATH` 상에서 실행 파일 위치 조회 도구
- 어원: **which** (어느 것 — 실제 실행되는 바이너리 식별)
- 일반 사용자 실행 가능

---

## which

```bash
which <명령>

# Examples
which timeout gtimeout 2>/dev/null || echo "no timeout cmd"   # 대체 명령 존재 확인
which psql 2>/dev/null                                         # 클라이언트 설치 여부
which pdftotext mutool qpdf                                    # 다중 후보 일괄 확인
which mmdc 2>/dev/null                                         # Node 전역 도구 확인
which jira acli 2>/dev/null                                    # CLI 도구 확인
which glab                                                     # GitLab CLI 확인
which setsid                                                   # 유틸 존재 확인
```

### 명령어 설명
- 사용 목적
	- **명령 설치 여부 사전 확인 시 사용** → 스크립트 분기·대체 경로 선택
	- 동일 명령의 다중 설치 시 실제 실행 대상 확인 시 사용
	- macOS·Linux 간 명령 가용성 차이 대응 시 사용 (`timeout` ↔ `gtimeout`)
- 특이사항
	- **종료 코드가 판정 수단** → 0 = 존재, 1 = 부재
		- `which cmd >/dev/null || echo "미설치"` 형태 분기 관용
	- **`PATH` 미포함 경로의 실행 파일은 미탐지** → 존재하나 부재로 판정 가능
	- 셸 내장 명령(`cd`·`export` 등)·별칭은 미탐지 → `type` 사용 필요
	- 다중 인자 지정 시 존재하는 것만 출력 → 부재 항목은 침묵
	- **셸 함수·별칭 포함 정확 판정은 `type` 또는 `command -v` 권장** → POSIX 표준
	- 미설치 확인 후 설치는 [[dnf]] 또는 패키지 관리자 사용

### 옵션
- `-a` : `PATH` 내 모든 일치 경로 출력 (**a**ll) ※ 미검증

---

## type / command -v

```bash
type <명령>
command -v <명령>

# Examples
type cd                    # 셸 내장 여부 확인
command -v python3         # 이식성 높은 존재 확인 (POSIX)
```

### 명령어 설명
- 사용 목적
	- 셸 내장·별칭·함수·외부 명령 구분 시 사용
	- 이식성 필요한 스크립트에서 존재 확인 시 사용
- 특이사항
	- **`which` 는 외부 명령 전용 → 내장 명령은 `type` 만 판정 가능**
	- `command -v` 는 POSIX 표준 → 셸 이식성 최상
	- 어원: **type** (유형) / **command** (명령 조회)

---

## 연관 명령어
- [[timeout]] : macOS `gtimeout` 대체 확인 대상
- [[dnf]] : 미설치 확인 후 설치 수단
- [[rpm]] : 명령 제공 패키지 역추적 (`rpm -qf $(which cmd)`)
- [[file]] : 확인된 실행 파일의 유형·아키텍처 판정
- [[env]] : `PATH` 환경변수 확인 — `which` 탐색 범위 결정
- [[ldd]] : `ldd $(which <명령>)` 로 확보 경로의 의존성 조회
