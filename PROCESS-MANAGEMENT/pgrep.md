---
command: pgrep
category: PROCESS-MANAGEMENT
aliases: [pidof]
tags:
  - linux/process
  - task/inspect
  - task/diagnose
  - topic/troubleshooting
  - privilege/mixed
related: ["[[pkill]]", "[[ps]]", "[[kill]]", "[[lsof]]", "[[grep]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# pgrep

- 패턴 일치 프로세스의 PID 조회 도구
- 어원: **p**rocess **grep**
- 조회는 일반 사용자 가능

---

## pgrep

```bash
pgrep [옵션] <패턴>

# Examples
pgrep -f "bootRun" 2>&1 || echo "STOPPED"           # 기동 여부 판정 (종료코드 활용)
pgrep -fl "bootRun\|BandManagerApplication"          # PID + 명령행 동시 출력
pgrep -fl "GradleDaemon\|bootRun" | head -5          # 다중 패턴 + 출력 제한
pgrep -f "bootRun" | head -1                         # 첫 PID만 취득
pgrep -af stat-scheduler || echo "none running"      # 전체 명령행 표시 (GNU)
pgrep -af "[s]tat-scheduler-0.0.1" || echo NONE      # 대괄호 트릭 병용
```

### 명령어 설명
- 사용 목적
	- 애플리케이션 기동 여부 판정 시 사용 (종료 코드 기반 분기)
	- 종료 대상 PID 취득 시 사용 ([[kill]] 전달)
	- `ps aux | grep` 의 자기제외 처리 회피 시 사용
- 특이사항
	- **기본 매칭 대상은 프로세스명(comm)만 → 15자 제한 및 인자 미포함**
		- JVM·스크립트 등은 프로세스명이 `java`·`bash` → 패턴 불일치
		- 대응: `-f` 로 전체 명령행 매칭 필수
	- **종료 코드가 판정 수단** → 0 = 일치 존재, 1 = 미일치
		- `pgrep -f pat || echo STOPPED` 형태로 기동 여부 분기
	- `-a`(전체 명령행 표시)는 GNU procps 전용 → **macOS 미지원**
		- macOS 대응: `-l` 사용 (PID + 프로세스명, 명령행 인자 제외)
	- 자기 자신은 결과에서 자동 제외 → `grep -v grep` 불필요
	- 다중 패턴은 `\|`(BRE) 사용, `-f` 와 병용

### 옵션
- `-f` : 전체 명령행 기준 매칭 (**f**ull command line) — JVM·스크립트 필수
- `-l` : PID 와 함께 이름 출력 (**l**ist name)
- `-a` : PID 와 전체 명령행 출력 (**a**ll, GNU 전용) — macOS 미지원
- `-n` : 최근 생성 프로세스만 (**n**ewest) ※ 미검증
- `-u <user>` : 특정 사용자 프로세스만 (**u**ser) ※ 미검증

---

## 연관 명령어
- [[pkill]] : 동일 패턴 문법으로 종료 — `pgrep` 로 대상 확인 후 실행
- [[ps]] : 상세 정보(CPU·MEM) 필요 시 사용
- [[kill]] : `pgrep` 취득 PID 전달
- [[lsof]] : 포트 점유 기준 역추적 — 프로세스명 불명 시 사용
- [[grep]] : `ps` 조합 방식 — `pgrep` 이 전용 대체
