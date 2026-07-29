---
command: ps
category: PROCESS-MANAGEMENT
aliases: []
tags:
  - linux/process
  - task/inspect
  - task/diagnose
  - topic/troubleshooting
  - topic/socket
  - privilege/mixed
related: ["[[pgrep]]", "[[pkill]]", "[[kill]]", "[[lsof]]", "[[grep]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# ps

- 실행 중 프로세스 스냅샷 조회 도구
- 어원: **p**rocess **s**tatus
- 조회는 일반 사용자 가능, 타 사용자 프로세스 상세는 권한 종속

---

## ps

```bash
ps aux | grep <패턴> | grep -v grep

# Examples
ps aux | grep -i "bootRun\|BandmanagerApplication\|gradle" | grep -v grep | head -3
ps aux | grep -i "gradle\|bootRun\|java" | grep -v grep       # JVM 프로세스 확인
ps aux | grep "jira issue create" | grep -v grep              # 특정 명령 실행 여부
ps aux | grep "[s]tat-scheduler-0.0.1-SNAPSHOT.jar"           # 대괄호 트릭 (grep 자기제외)
ps aux | grep "[s]tat-scheduler.jar" || echo "프로세스 종료됨"  # 종료 여부 판정
lsof -nP -iTCP:5432 -sTCP:LISTEN | awk '{print $1, $2, $9}'   # 포트 기준 역추적
```

### 명령어 설명
- 사용 목적
	- 애플리케이션 기동 여부 확인 시 사용 (Gradle·JVM 프로세스)
	- 종료 대상 프로세스의 PID 식별 시 사용
	- 좀비·중복 프로세스 검출 시 사용
- 특이사항
	- **`ps aux | grep 패턴` 은 grep 자기 자신도 결과에 포함** → 오탐 발생
		- 대응 1: `| grep -v grep` 추가 (가독성 높음)
		- 대응 2: `grep "[b]ootRun"` 대괄호 트릭 — 패턴 문자열과 실제 프로세스명 불일치로 자기제외
		- 대응 3: [[pgrep]] 사용 (전용 도구, 자기제외 내장)
	- **`aux` 는 BSD 스타일 옵션 → 하이픈 없음** (`-aux` 와 동작 상이)
	- 명령행이 길면 잘려 표시 → 전체 확인은 `ps -ww` ※ 미검증
	- macOS·Linux 모두 `aux` 지원 → 출력 컬럼 순서 동일
	- 스냅샷 조회 → 순간 상태만 반영, 지속 관찰은 `top`·`watch` 필요

### 옵션
- `a` : 타 사용자 프로세스 포함 (**a**ll users)
- `u` : 사용자 지향 상세 형식 (CPU·MEM 포함) (**u**ser format)
- `x` : 제어 터미널 없는 프로세스 포함 (**x** = 터미널 무관, 데몬 포함)
- `aux` : 전체 프로세스 상세 조회 — 최빈출 조합
- `-ef` : System V 스타일 전체 조회 (**e**very + **f**ull) ※ 미검증
- `-p <PID>` : 특정 PID만 조회 (**p**id) ※ 미검증

---

## 연관 명령어
- [[pgrep]] : 패턴 기준 PID 조회 — `ps | grep` 의 전용 대체
- [[pkill]] : 패턴 기준 프로세스 종료 — `ps` 로 대상 확인 후 실행
- [[kill]] : PID 기준 종료 — `ps` 로 취득한 PID 전달
- [[lsof]] : 포트 점유 기준 프로세스 역추적
- [[grep]] : `ps` 출력 필터링 — 자기제외 처리 필요
