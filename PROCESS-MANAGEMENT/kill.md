---
command: kill
category: PROCESS-MANAGEMENT
aliases: []
tags:
  - linux/process
  - task/restart
  - task/diagnose
  - task/inspect
  - topic/troubleshooting
  - privilege/mixed
  - danger/destructive
related: ["[[pkill]]", "[[pgrep]]", "[[ps]]", "[[lsof]]", "[[xargs]]", "[[nohup]]", "[[systemctl]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6 / Rocky Linux 9.8 (aarch64, 시그널 표 실검증)
updated: 2026-09-06
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
- `-l` : 시그널 목록·번호↔이름 변환 (**l**ist) → 아래 [[#kill -l]] 참조
- `-HUP` : `SIGHUP` 전송 (**H**ang**UP**) — 데몬의 설정 재적재 관례
	- 전송 자체는 검증. **재적재 동작은 애플리케이션 구현에 따름** → 미지원 시 종료됨
- `-0` : 시그널 미전송, **프로세스 존재 여부만 확인** (**0** = null signal)
	- 생존 시 종료 코드 `0`, 부재 시 `1` + `No such process`

---

## kill -l

```bash
kill -l                 # 전체 시그널 목록
kill -l <번호>          # 번호 → 이름
kill -l <이름>          # 이름 → 번호

# Examples
$ kill -l 9             # KILL
$ kill -l TERM          # 15
$ kill -l | wc -w       # 124 = 62개 항목 × 2단어
```

### 명령어 설명
- 사용 목적
	- 시그널 번호·이름 대응 확인 시 사용
	- 스크립트에서 종료 코드(`128+N`)를 시그널 이름으로 역추적 시 사용
- 특이사항
	- **번호는 아키텍처마다 상이** → 아래 표는 Linux x86_64·aarch64 기준
	- `SIGKILL`(9)·`SIGSTOP`(19)·`SIGTERM`(15) 등 주요 번호는 전 플랫폼 동일
	- **32·33번 결번** → glibc 스레드 구현(NPTL)이 내부 예약, `kill -l 32` 는 빈 출력
	- 실측 항목 수 62개 = 표준 31개(1~31) + 실시간 31개(34~64)

### 전체 목록 (Rocky Linux 9.8 aarch64 실측)

```
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL       5) SIGTRAP
 6) SIGABRT      7) SIGBUS       8) SIGFPE       9) SIGKILL     10) SIGUSR1
11) SIGSEGV     12) SIGUSR2     13) SIGPIPE     14) SIGALRM     15) SIGTERM
16) SIGSTKFLT   17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU     25) SIGXFSZ
26) SIGVTALRM   27) SIGPROF     28) SIGWINCH    29) SIGIO       30) SIGPWR
31) SIGSYS      34) SIGRTMIN    ...             63) SIGRTMAX-1  64) SIGRTMAX
```

---

## 시그널 표 — 표준 시그널 (1~31)

- **기본 동작** 5종 — `Term`=종료 · `Core`=종료+코어덤프 · `Ign`=무시 · `Stop`=정지 · `Cont`=재개
- **포착**(catch) = `trap` 핸들러 등록 가능 여부, **무시**(ignore) = `trap ''` 로 차단 가능 여부

| 번호 | 이름 | 원어 | 기본 동작 | 포착·무시 | 발생 상황·용도 |
| --- | --- | --- | --- | --- | --- |
| **1** | **SIGHUP** | **H**ang **UP** | Term | 가능 | 터미널 연결 끊김. **데몬은 설정 재적재 관례로 사용** |
| **2** | **SIGINT** | **INT**errupt | Term | 가능 | **Ctrl+C** |
| **3** | SIGQUIT | **QUIT** | **Core** | 가능 | **Ctrl+\\** — 코어 덤프 동반 |
| 4 | SIGILL | **ILL**egal instruction | Core | 가능 | 잘못된 기계어 실행 |
| 5 | SIGTRAP | **TRAP** | Core | 가능 | 디버거 중단점(breakpoint) |
| 6 | SIGABRT | **ABR**or**T** | Core | 가능 | `abort()` 호출, assert 실패 |
| 7 | SIGBUS | **BUS** error | Core | 가능 | 정렬 오류·매핑 밖 메모리 접근 |
| 8 | SIGFPE | **F**loating **P**oint **E**xception | Core | 가능 | 0 나누기 등 산술 오류 |
| **9** | **SIGKILL** | **KILL** | **Term** | **불가능** ⚠ | `kill -9`. **커널이 즉시 종료 — 프로세스에 전달조차 안 됨** |
| 10 | SIGUSR1 | **US**e**R** defined 1 | Term | 가능 | 애플리케이션 정의 용도 |
| 11 | SIGSEGV | **SEG**mentation **V**iolation | Core | 가능 | 잘못된 메모리 접근 |
| 12 | SIGUSR2 | **US**e**R** defined 2 | Term | 가능 | 애플리케이션 정의 용도 |
| 13 | SIGPIPE | **PIPE** | Term | 가능 | 읽는 쪽이 닫힌 파이프에 쓰기 (`cmd \| head` 패턴) |
| 14 | SIGALRM | **AL**a**RM** | Term | 가능 | `alarm()` 타이머 만료 |
| **15** | **SIGTERM** | **TERM**inate | Term | 가능 | **`kill` 기본값.** 정상 종료 요청 |
| 16 | SIGSTKFLT | **ST**ac**K** **F**au**LT** | Term | 가능 | 보조프로세서 스택 오류. 현재 미사용 |
| **17** | **SIGCHLD** | **CH**i**LD** | **Ign** | 가능 | 자식 프로세스 종료·정지 알림 |
| **18** | **SIGCONT** | **CONT**inue | **Cont** | 가능 | 정지된 프로세스 재개 (`fg`·`bg`) |
| **19** | **SIGSTOP** | **STOP** | **Stop** | **불가능** ⚠ | `kill -STOP`. 강제 정지 |
| **20** | **SIGTSTP** | **T**erminal **ST**o**P** | Stop | 가능 | **Ctrl+Z** |
| 21 | SIGTTIN | **TT**y **IN**put | Stop | 가능 | 백그라운드 작업의 터미널 읽기 시도 |
| 22 | SIGTTOU | **TT**y **OU**t**put** | Stop | 가능 | 백그라운드 작업의 터미널 쓰기 시도 |
| 23 | SIGURG | **URG**ent | **Ign** | 가능 | 소켓 긴급 데이터 도착 |
| 24 | SIGXCPU | e**X**ceeded **CPU** | Core | 가능 | CPU 시간 한도 초과 (`ulimit -t`) |
| 25 | SIGXFSZ | e**X**ceeded **F**ile **S**i**Z**e | Core | 가능 | 파일 크기 한도 초과 (`ulimit -f`) |
| 26 | SIGVTALRM | **V**ir**T**ual **AL**a**RM** | Term | 가능 | 가상 타이머 만료 |
| 27 | SIGPROF | **PROF**iling | Term | 가능 | 프로파일링 타이머 만료 |
| **28** | **SIGWINCH** | **WIN**dow **CH**ange | **Ign** | 가능 | **터미널 창 크기 변경** |
| 29 | SIGIO | **I**nput/**O**utput | Term | 가능 | 비동기 입출력 가능 상태 (= SIGPOLL) |
| 30 | SIGPWR | **P**o**W**e**R** failure | Term | 가능 | 전원 이상 (UPS 연동) |
| 31 | SIGSYS | bad **SYS**tem call | Core | 가능 | 잘못된 시스템 콜 (seccomp 위반 포함) |

### 실시간 시그널 (34~64)

| 구분 | 범위 | 특징 |
| --- | --- | --- |
| `SIGRTMIN` ~ `SIGRTMIN+15` | 34~49 | 커널이 **큐잉** — 표준 시그널과 달리 중복 전달이 유실되지 않음 |
| `SIGRTMAX-14` ~ `SIGRTMAX` | 50~64 | 전달 순서 보장(낮은 번호 우선), 데이터 첨부 가능 |

- 기본 동작은 전부 `Term`, 전용 용도 없음 → **애플리케이션이 자유 정의**
- 실무 사례 — `systemctl` 이 PID 1 에 실시간 시그널로 전원 명령 전달

---

## 시그널 실검증 (Rocky Linux 9.8)

### 포착 불가 시그널 — SIGKILL·SIGSTOP

```bash
# SIGKILL: trap 등록은 오류 없이 통과하나 핸들러가 실행되지 않음
$ bash -c 'trap "echo CAUGHT-KILL" SIGKILL; kill -KILL $$; echo SURVIVED-KILL'
Killed                                    # CAUGHT-KILL·SURVIVED-KILL 미출력
$ echo $?
137                                       # 128+9

# 비교 — SIGTERM 은 정상 포착
$ bash -c 'trap "echo CAUGHT-TERM; exit 0" SIGTERM; kill -TERM $$; echo AFTER-TERM'
CAUGHT-TERM
$ echo $?
0

# SIGINT 무시는 성립
$ bash -c 'trap "" SIGINT; kill -INT $$; echo SURVIVED-INT'
SURVIVED-INT

# SIGSTOP 무시는 불성립 → 여전히 정지(T)
$ bash -c 'trap "" SIGSTOP; sleep 5' & P=$!; kill -STOP $P; ps -o stat= -p $P
T
```

- **`trap` 등록이 실패하지 않는다는 점이 함정** → 셸은 받아들이나 커널이 배달하지 않음
- `SIGKILL`·`SIGSTOP` 만 포착·무시 **모두 불가** → 관리자의 최후 통제 수단 보존 목적

### 상태 전이 — STOP / CONT

```bash
$ sleep 300 & P=$!
$ kill -STOP $P; ps -o stat= -p $P
T                                         # T = stopped
$ kill -CONT $P; ps -o stat= -p $P
S                                         # S = interruptible sleep
```

- 프로세스 상태 문자는 [[ps]] 참조

### 종료 코드 = 128 + 시그널 번호

```bash
$ for s in HUP TERM USR1 USR2 PIPE ALRM KILL; do ... ; done
HUP  → 129        # 128+1
TERM → 143        # 128+15
USR1 → 138        # 128+10
USR2 → 140        # 128+12
PIPE → 141        # 128+13
ALRM → 142        # 128+14
KILL → 137        # 128+9
```

- **스크립트 종료 코드 128 초과 시 시그널 종료로 판정** → `$(( 코드 - 128 ))` 로 원인 시그널 역산
- `137` = OOM Killer 또는 강제 종료, `143` = 정상 종료 요청 — 컨테이너 진단의 기본 지표

### 기본 동작 = 무시(Ign) 인 시그널

```bash
$ for s in CHLD WINCH URG CONT; do sleep 5 & kill -$s $!; ... ; done
CHLD  → 생존
WINCH → 생존
URG   → 생존
CONT  → 생존
```

---

## /proc/\<PID\>/status 시그널 마스크

프로세스가 각 시그널을 **어떻게 처리하도록 등록했는지** 조회. 64비트 16진수 비트마스크로, `N` 번째 비트(1부터) = 시그널 `N`.

```bash
$ grep -E '^Sig' /proc/1/status
SigQ:   1/14135                           # 대기 중 실시간 시그널 / 한도
SigPnd: 0000000000000000                  # 전달 대기(pending)
SigBlk: 7fe3c0fe28014a03                  # 차단(blocked)
SigIgn: 0000000000001000                  # 무시(ignored)
SigCgt: 00000001000004ec                  # 포착(caught) — 핸들러 등록
```

### PID 1(systemd) 마스크 해석

| 필드 | 시그널 | 의미 |
| --- | --- | --- |
| `SigIgn` | PIPE | `SIGPIPE` 무시 — 파이프 끊김으로 죽지 않도록 |
| `SigCgt` | QUIT, ILL, ABRT, BUS, FPE, SEGV | **치명적 오류만 핸들러 처리** (덤프·로깅) |
| `SigBlk` | HUP, INT, USR1, USR2, TERM, CHLD, WINCH, PWR, RT34~ | **차단 후 `signalfd` 로 수신** |

- **`SigBlk` 에 있다고 무응답이 아님** → 차단 후 이벤트 루프에서 읽는 `signalfd` 방식
	- systemd 가 `SigCgt` 에 `TERM` 이 없음에도 종료 요청을 처리하는 이유
- 진단 활용 — "시그널을 보냈는데 반응이 없음" 시 `SigIgn`·`SigBlk` 확인으로 원인 판별

```bash
# 비트마스크 해석 예 (SigIgn 0x...1000 → 13번 = SIGPIPE)
$ printf '%d\n' 0x1000        # 4096 = 2^12 → 13번째 비트
```

---

## 함정 — 셸 환경에 따른 시그널 동작 차이

### 비대화형 셸의 백그라운드 작업은 SIGINT·SIGQUIT 를 무시로 상속

```bash
$ bash -c 'sleep 30 & P=$!; grep -E "^SigIgn" /proc/$P/status'
SigIgn: 0000000000000006                  # 비트 2,3 = SIGINT(2), SIGQUIT(3)

$ grep -E '^SigIgn' /proc/self/status     # 포그라운드 프로세스
SigIgn: 0000000000000000                  # 무시 없음
```

- POSIX 규정 — 잡 제어 없는 셸이 `&` 로 띄운 작업은 `SIGINT`·`SIGQUIT` 가 `SIG_IGN` 으로 설정됨
- **결과: 스크립트·`ssh 명령` 환경에서 `kill -INT` 가 먹히지 않음** → `SIGTERM` 사용 필요

### 잡 제어 비활성 시 SIGTSTP·SIGTTIN·SIGTTOU 미적용

```bash
$ bash -c 'set -m; sleep 10 & P=$!; kill -TSTP $P; ps -o stat= -p $P'
T                                         # 잡 제어 활성 → 정지

$ bash -c 'sleep 10 & P=$!; kill -TSTP $P; ps -o stat= -p $P'
S                                         # 잡 제어 없음 → 정지되지 않음
```

- POSIX 규정 — **고아 프로세스 그룹에 전달된 정지 시그널은 폐기**
- **`SIGSTOP` 은 예외 없이 적용** → 확실한 정지가 필요하면 `-STOP` 사용

---

## 실무 전송 패턴

```bash
kill $PID                    # 1차: SIGTERM — 정리 절차 수행 기회 부여
sleep 5
kill -0 $PID 2>/dev/null && kill -9 $PID    # 2차: 무응답 시에만 SIGKILL

kill -HUP $(cat /run/nginx.pid)             # 데몬 설정 재적재
kill -USR1 $PID                             # 애플리케이션 정의 동작(로그 재개방 등)
kill -STOP $PID ; kill -CONT $PID           # 일시 정지·재개 (부하 조절)
kill -0 $PID                                # 존재 여부만 확인
```

- **`SIGTERM` → 유예 → `SIGKILL`** 순서가 원칙 → [[systemctl]] 의 서비스 정지도 동일 방식
	- 유예 시간은 유닛의 `TimeoutStopSec` 가 결정
- `SIGHUP` 회피 목적의 백그라운드 기동은 [[nohup]] 참조

---

## 연관 명령어
- [[pkill]] : 패턴 기준 종료 — PID 조회 불필요, 광범위 종료 위험 존재
- [[pgrep]] : 종료 대상 PID 취득 및 종료 후 확인
- [[ps]] : PID 및 상세 정보 확인
- [[lsof]] : `lsof -ti:포트 | xargs kill` 포트 점유 해제 패턴
- [[xargs]] : 파이프 전달 PID 를 `kill` 인자로 변환
- [[nohup]] : `SIGHUP` 회피 백그라운드 기동 — 시그널 1번의 대표 대응 수단
- [[systemctl]] : 서비스 정지도 `SIGTERM` → 유예 → `SIGKILL` 동일 순서 (`TimeoutStopSec`)
