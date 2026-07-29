---
command: pkill
category: PROCESS-MANAGEMENT
aliases: [killall]
tags:
  - linux/process
  - task/restart
  - task/diagnose
  - topic/troubleshooting
  - privilege/mixed
  - danger/destructive
related: ["[[pgrep]]", "[[kill]]", "[[ps]]", "[[lsof]]", "[[nohup]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# pkill

- 패턴 일치 프로세스에 시그널 전송 도구
- 어원: **p**rocess **kill**
- 자기 소유 프로세스는 일반 사용자, 타 사용자·시스템 프로세스는 root 필요

---

## pkill

```bash
pkill -f <패턴>

# Examples
pkill -f "bootRun" 2>/dev/null                     # Gradle 기동 프로세스 종료
pkill -f "BandManagerApplication" 2>/dev/null       # 애플리케이션 종료
pkill -f "BandManagerApplicationKt" 2>/dev/null     # Kotlin 진입점 종료
pkill -f "gradle-wrapper.jar bootRun" 2>&1          # 정밀 패턴 지정
pkill -f "GradleWorkerMain" 2>/dev/null             # 워커 프로세스 정리
pkill -f "org.gradle.launcher" 2>/dev/null          # 런처 데몬 정리
```

### 명령어 설명
- 사용 목적
	- 백그라운드 기동 애플리케이션 종료 시 사용 ([[nohup]] 실행분 정리)
	- 포트 점유 해제 목적 프로세스 정리 시 사용
	- 재기동 전 잔존 프로세스 일괄 제거 시 사용
- 특이사항
	- **패턴이 광범위하면 의도 외 프로세스 동시 종료 발생** ⚠
		- 대응: **[[pgrep]] 로 동일 패턴 대상 사전 확인 필수**
		- 예: `pgrep -fl "bootRun"` 확인 → `pkill -f "bootRun"` 실행
	- 기본 매칭은 프로세스명만 → JVM·스크립트는 `-f` 필수 ([[pgrep]] 과 동일 제약)
	- 대상 미존재 시 종료 코드 1 → `2>/dev/null` 로 억제 관용
	- **기본 시그널은 `SIGTERM`(15)** → 정상 종료 유도
		- 무응답 시 `-9`(`SIGKILL`) 사용 → 데이터 정리 미수행, 최후 수단
	- 자기 자신은 대상에서 자동 제외
	- 종료 후 실제 소멸 여부는 [[pgrep]]·[[lsof]] 재확인 권장

### 옵션
- `-f` : 전체 명령행 기준 매칭 (**f**ull command line) — 실질 필수
- `-9` : `SIGKILL` 강제 종료 (시그널 번호) ⚠ 정리 절차 미수행
- `-15` : `SIGTERM` 정상 종료 요청 — 기본값
- `-u <user>` : 특정 사용자 프로세스만 (**u**ser) ※ 미검증
- `-signal <name>` : 시그널명 지정 (`-TERM`, `-HUP` 등) ※ 미검증

---

## 연관 명령어
- [[pgrep]] : **동일 패턴 사전 검증 필수** — 오종료 방지
- [[kill]] : 단일 PID 정밀 종료 — 대상 확정 시 안전
- [[ps]] : 종료 전후 프로세스 상태 확인
- [[lsof]] : 포트 점유 해제 여부 확인
- [[nohup]] : `pkill` 대상이 되는 백그라운드 프로세스 기동
