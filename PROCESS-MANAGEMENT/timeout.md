---
command: timeout
category: PROCESS-MANAGEMENT
aliases: [gtimeout]
tags:
  - linux/process
  - task/verify
  - task/diagnose
  - topic/troubleshooting
  - privilege/user
related: ["[[nohup]]", "[[kill]]", "[[ping]]", "[[tail]]", "[[which]]"]
distro: 전체 (macOS 는 coreutils 별도 설치 필요)
verified: macOS (Darwin 25.5, gtimeout) / Rocky Linux 9.6
updated: 2026-07-30
---

# timeout

- 지정 시간 초과 시 명령을 강제 종료하는 실행 래퍼
- 어원: **time** **out** (제한 시간 초과)
- 일반 사용자 실행 가능

---

## timeout

```bash
timeout <초> <명령>

# Examples
timeout 150 ./gradlew bootRun > /tmp/bootrun.log 2>&1     # 기동 시간 상한 지정
timeout 60 ./gradlew bootRun > /tmp/bootrun2.log 2>&1     # 짧은 검증용 기동
timeout 5 ping -c 2 10.1.17.62 2>&1 | tail -3             # 무응답 호스트 대기 차단
timeout 3 bash -c "echo > /dev/tcp/10.1.17.62/22" 2>/dev/null   # 포트 도달성 검사
timeout 480 gh pr checks 136 --watch --interval 20 2>&1 | tail -15   # CI 대기 상한
timeout 90 mvn -o spring-boot:run -Dspring-boot.run.profiles=local > /tmp/gw.log 2>&1
which timeout gtimeout 2>/dev/null || echo "no timeout cmd"      # 사전 존재 확인
```

### 명령어 설명
- 사용 목적
	- 자동화 환경에서 무한 대기 방지 시 사용 (기동·CI 감시·네트워크 검사)
	- 응답 없는 호스트·포트 검사 시 대기 시간 제한 시 사용
	- 기동 후 자동 종료되는 일회성 검증 시 사용
- 특이사항
	- **macOS 는 기본 미제공** → Homebrew coreutils 설치 시 `gtimeout` 으로 제공
		- 대응: `which timeout gtimeout` 사전 확인 후 분기
	- **종료 코드 124 = 시간 초과로 강제 종료** → 정상 종료(0)와 구분 가능
	- 기본 시그널은 `SIGTERM` → 무응답 프로세스는 잔존 가능
		- 대응: `-k <초>` 로 후속 `SIGKILL` 예약 ※ 미검증
	- 시간 단위 접미사 지정 가능 → `10s` `5m` `1h` (미지정 시 초)
	- **자기 완결 종료 명령에는 불필요** → 대기·감시 성격 명령에만 적용
	- 백그라운드 지속 실행이 목적이면 [[nohup]] 사용 → 목적 상반

### 옵션
- `-k <초>` : 지정 시간 후 `SIGKILL` 추가 전송 (**k**ill after) ※ 미검증
- `-s <시그널>` : 전송 시그널 지정 (**s**ignal) ※ 미검증
- `--preserve-status` : 대상 명령의 종료 코드 유지 ※ 미검증

---

## 연관 명령어
- [[nohup]] : 지속 백그라운드 실행 — `timeout` 과 목적 상반
- [[kill]] : `timeout` 이 내부적으로 전송하는 시그널 수단
- [[ping]] : `timeout 5 ping` 무응답 호스트 대기 차단
- [[tail]] : `timeout` 실행 로그 결과 확인
- [[which]] : macOS `gtimeout` 존재 여부 사전 확인
