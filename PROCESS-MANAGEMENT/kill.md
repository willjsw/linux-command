---
command: kill
category: PROCESS-MANAGEMENT
aliases: []
tags:
  - linux/process
  - task/restart
  - task/diagnose
  - topic/troubleshooting
  - privilege/mixed
  - danger/destructive
related: ["[[pkill]]", "[[pgrep]]", "[[ps]]", "[[lsof]]", "[[xargs]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# kill

- 지정 PID 에 시그널 전송 도구
- 어원: **kill** (프로세스 종료 — 실제로는 시그널 전송)
- 자기 소유 프로세스는 일반 사용자, 타 사용자 프로세스는 root 필요

---

## kill

```bash
kill [-<시그널>] <PID>

# Examples
kill 8056                                  # SIGTERM 정상 종료 요청
kill $BGPID 2>/dev/null                    # 스크립트 내 백그라운드 PID 종료
kill 37815 2>/dev/null                     # 존재 불확실 PID (오류 억제)
kill -9 $PID                               # SIGKILL 강제 종료
lsof -ti:8080 | xargs -r kill 2>/dev/null  # 포트 점유 프로세스 종료
```

### 명령어 설명
- 사용 목적
	- 특정 PID 프로세스 종료 시 사용 ([[ps]]·[[pgrep]] 로 취득)
	- 스크립트 내 백그라운드 작업 정리 시 사용 (`$!` 로 취득한 PID)
	- 포트 점유 프로세스 해제 시 사용 ([[lsof]] 연계)
- 특이사항
	- **기본 시그널은 `SIGTERM`(15)** → 프로세스가 정리 후 자체 종료
		- JVM 등은 셧다운 훅 실행 → 데이터 정합성 유지
	- **`-9`(`SIGKILL`) 는 정리 절차 미수행** ⚠ → 임시파일 잔존·데이터 손실 가능
		- `SIGTERM` 무응답 확인 후에만 사용
	- 종료 요청 후 즉시 소멸 아님 → [[pgrep]]·[[lsof]] 로 실제 종료 확인 필요
	- PID 재사용 위험 존재 → 취득과 종료 사이 시간차 최소화 권장
	- 잘못된 PID 는 무해하나 **`kill 1` 등 시스템 PID 지정 시 심각 영향** ⚠
	- 패턴 기준 종료는 [[pkill]] 사용 → PID 조회 단계 생략

### 옵션
- `-9` : `SIGKILL` 강제 종료 (시그널 번호 9) ⚠ 정리 절차 미수행
- `-15` : `SIGTERM` 정상 종료 요청 — 기본값
- `-l` : 시그널 목록 출력 (**l**ist) ※ 미검증
- `-HUP` : `SIGHUP` 설정 재적용 요청 (**H**ang**UP**) ※ 미검증
- `-0` : 시그널 미전송, 존재 여부만 확인 (**0** = null signal) ※ 미검증

---

## 연관 명령어
- [[pkill]] : 패턴 기준 종료 — PID 조회 불필요, 광범위 종료 위험 존재
- [[pgrep]] : 종료 대상 PID 취득 및 종료 후 확인
- [[ps]] : PID 및 상세 정보 확인
- [[lsof]] : `lsof -ti:포트 | xargs kill` 포트 점유 해제 패턴
- [[xargs]] : 파이프 전달 PID 를 `kill` 인자로 변환
