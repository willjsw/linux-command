---
command: nohup
category: PROCESS-MANAGEMENT
aliases: [disown, setsid]
tags:
  - linux/process
  - task/restart
  - task/configure
  - topic/troubleshooting
  - privilege/user
related: ["[[pkill]]", "[[pgrep]]", "[[tail]]", "[[timeout]]", "[[ps]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# nohup

- 터미널 종료 시그널(`SIGHUP`)을 무시하는 백그라운드 실행 도구
- 어원: **no** **hup** (`SIGHUP` 무시)
- 일반 사용자 실행 가능

---

## nohup

```bash
nohup <명령> > <로그파일> 2>&1 &

# Examples
nohup ./gradlew bootRun > /tmp/bootrun.log 2>&1 &                    # 기본 백그라운드 기동
nohup ./gradlew bootRun --console=plain > /tmp/bootrun5.log 2>&1 & disown   # 셸 작업목록에서도 분리
nohup ./gradlew bootRun > /tmp/bootrun_bd205.log 2>&1 &              # 로그 분리 기동
tail -8 /tmp/bootrun.log                                             # 기동 결과 확인
pgrep -f "bootRun" || echo "STOPPED"                                 # 기동 여부 판정
pkill -f "bootRun" 2>/dev/null                                       # 종료
```

### 명령어 설명
- 사용 목적
	- 애플리케이션을 세션 독립적으로 기동 시 사용 (터미널 종료에도 잔존)
	- 장시간 실행 작업을 백그라운드 전환 시 사용
	- 대화형 셸 점유 없이 서버 기동 검증 시 사용
- 특이사항
	- **`&` 단독 사용만으로는 터미널 종료 시 함께 종료** → `nohup` 필수
	- **`2>&1` 미지정 시 stderr 가 터미널로 유출** → 로그 누락·화면 오염 발생
		- 표준 조합: `> logfile 2>&1 &` (리다이렉트 순서 중요)
	- 리다이렉트 미지정 시 `nohup.out` 파일 자동 생성 → 현재 디렉터리 오염
	- `disown` 추가 시 셸 작업 목록에서도 제거 → 셸 종료 경고 방지
	- **기동 성공 확인은 로그·프로세스 재확인 필수** → `nohup` 자체는 즉시 반환
		- 순서: `nohup ... &` → `sleep` → [[tail]] 로그 확인 → [[pgrep]] 생존 확인
	- 종료 시각 제한이 필요하면 [[timeout]] 병용 검토
	- 종료는 [[pkill]] `-f` 로 패턴 지정 → PID 미보관 시에도 대응 가능

### 옵션
- 자체 옵션 없음 → 리다이렉트·`&`·`disown` 셸 기능과 조합 사용

---

## 연관 명령어
- [[pkill]] : `nohup` 기동 프로세스 종료 — `-f` 패턴 지정
- [[pgrep]] : 기동 생존 여부 판정
- [[tail]] : 기동 로그 확인 — 성공·실패 판정
- [[timeout]] : 실행 시간 상한 필요 시 병용 — 자동화 무한대기 방지
- [[ps]] : 기동 프로세스 상세 확인
