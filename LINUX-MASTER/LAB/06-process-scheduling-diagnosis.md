---
title: LAB 06 — 프로세스·스케줄링·시스템 진단(top 심화)
type: exam-lab
part: 06
tags:
  - exam/linux-master
  - exam/lab
  - linux/process
  - linux/signal
  - linux/scheduling
  - task/configure
  - task/diagnose
  - task/verify
related: ["[[README]]", "[[05-disk-lvm-raid-swap-quota]]", "[[07-boot-systemd-log]]", "[[../THEORY/process-management]]", "[[../../PROCESS-MANAGEMENT/ps]]", "[[../../PROCESS-MANAGEMENT/kill]]", "[[../../PROCESS-MANAGEMENT/pgrep]]", "[[../../PROCESS-MANAGEMENT/pkill]]", "[[../../PROCESS-MANAGEMENT/nohup]]", "[[../../PROCESS-MANAGEMENT/lsof]]", "[[../../PROCESS-MANAGEMENT/timeout]]", "[[../../SERVICE-SYSTEMD/crontab]]"]
distro: Rocky Linux 9 (aarch64, UTM)
updated: 2026-09-03
---

# LAB 06 — 프로세스·스케줄링·시스템 진단(top 심화)

- 프로세스 조회(`ps`·`pstree`·`pgrep`·`/proc`)·잡 제어·시그널·우선순위(nice/renice)를 실제 부하 프로세스로 실습
- **`top` 을 옵션·헤더·열·대화식 키 전부 눌러보고**, CPU 폭주·메모리 압박·I/O 병목·좀비·FD 점유 5개 장애를 재현→진단→조치
- `vmstat`·`iostat`·`mpstat`·`pidstat`·`sar`(sysstat) 로 교차 확인, `at`·cron·systemd timer 로 정기 작업 등록
- 선행 자원: 사용자 `dev1`(Part 03), `/data` ext4 2 GB·스왑 1 GB + `/swapfile` 512 MB(Part 05), EPEL·`stress-ng`(Part 02)

> **이 파트의 시나리오**: 서버 `srv01` 운영 중 개발팀에서 "서버가 느려졌다"는 신고가 들어왔다. 관리자 `admin1` 은 원인을 모른 채 `top` 을 여는 대신, 이번 파트에서 CPU·메모리·I/O·좀비·파일 디스크립터 5가지 장애를 **의도적으로 만들어** 지표가 어떻게 변하는지 눈으로 익히고, 원인 프로세스를 특정해 우선순위 조정·시그널로 정리한다. 마지막으로 진단 스크립트 `sysreport.sh` 와 백업을 `at`·cron·systemd timer 에 등록해 정기 점검 체계를 만든다.

---

## 0. 준비

### 0-1. 도구 설치 확인

> **상황**: 부하 생성기 `stress-ng`(EPEL), 통계 도구 `sysstat`, 터미널 다중화 `tmux` 가 없으면 이 파트 진행이 불가하므로 먼저 확인한다.

```bash
rpm -q epel-release stress-ng sysstat tmux procps-ng psmisc lsof util-linux at cronie cronie-anacron
dnf install -y stress-ng sysstat tmux psmisc lsof at cronie cronie-anacron   # 없는 것만 설치됨
dnf install -y htop iotop                                                    # EPEL, 선택
```

- `rpm -q` : 패키지 설치 여부 조회 (**q**uery)
- `psmisc` : `pstree`·`killall`·`fuser` 제공 패키지
- `procps-ng` : `ps`·`top`·`vmstat`·`free`·`pgrep`·`pkill`·`watch` 제공 패키지 (기본 설치)
- `cronie` / `cronie-anacron` : `crond`·`crontab` / `anacron` 제공 패키지
- `iotop` : EPEL 제공. 패키지가 없으면 `dnf search iotop` 으로 `iotop-c` 확인

**검증**

```bash
stress-ng --version | head -1
which sar vmstat iostat mpstat pidstat pstree killall fuser tmux at crontab
nproc; free -h | head -2
```

기대 출력

```text
stress-ng, version 0.1x.xx ...
/usr/bin/sar
/usr/bin/vmstat
...
2
               total        used        free      shared  buff/cache   available
Mem:           3.7Gi       ...
```

> 📝 **시험 포인트**: `top`·`ps`·`vmstat`·`free` 는 `procps-ng`, `iostat`·`sar`·`mpstat`·`pidstat` 는 `sysstat`, `pstree`·`killall`·`fuser` 는 `psmisc` 패키지 — 명령과 패키지 매핑이 출제됨.

### 0-2. 작업용 터미널 2개 준비 (tmux)

> **상황**: 한 창에서 `top` 을 띄워 놓고 다른 창에서 부하를 만들어야 하므로 `tmux` 세션 `lab` 을 열고 창을 분할한다.

```bash
tmux new -s lab            # 세션 lab 생성·접속
# 세션 안에서:  Ctrl+b  "   → 상하 분할,   Ctrl+b  o  → 창 이동,   Ctrl+b  d  → 세션 분리(detach)
tmux ls                    # 세션 목록
tmux attach -t lab         # 재접속
```

- `new -s <이름>` : 새 세션 생성 (**s**ession name)
- `ls` : 세션 목록 (`list-sessions`)
- `attach -t <이름>` : 분리된 세션 재접속 (**t**arget)
- `Ctrl+b` : tmux 접두 키(prefix). `d` 분리, `"` 상하 분할, `%` 좌우 분할, `o` 패널 이동, `x` 패널 닫기
- `screen` 은 RHEL 9 기본 저장소에서 제외(EPEL) → `screen -S lab` / `Ctrl+a d` / `screen -r lab` 대응 관계만 참고

**검증**

```bash
tmux ls
ps -o pid,ppid,tty,cmd -C tmux
```

기대 출력

```text
lab: 1 windows (created ...) (attached)
    PID    PPID TT       CMD
   ...       1 ?        tmux new -s lab
```

> 📝 **시험 포인트**: `tmux`/`screen` 은 SSH 끊김에도 세션 유지 → `nohup` 과 함께 "로그아웃 후에도 작업 유지" 수단으로 비교 출제.

---

## 1. 프로세스 기초

### 1-1. `ps` — BSD 형식과 UNIX 형식

> **상황**: 신고 직후 현재 실행 중인 프로세스 전체 스냅샷을 두 가지 표기법으로 얻고 열 차이를 확인한다.

```bash
ps aux | head -5                 # BSD 형식 (하이픈 없음)
ps -ef | head -5                 # UNIX/System V 형식 (하이픈)
ps -e -l | head -5               # long 형식: F S UID PID PPID C PRI NI ... WCHAN
ps -u dev1                       # 특정 사용자
ps -p 1 -o pid,ppid,user,cmd     # 특정 PID
ps -C sshd -o pid,ppid,user,stat,cmd   # 명령 이름으로
ps -ef --forest | head -20       # 트리 형식
ps -ejH | head                   # 계층(H) + 세션(j)
ps -eo pid,ppid,ni,pri,stat,%cpu,%mem,rss,vsz,tty,time,cmd --sort=-%cpu | head -10
```

- `a` : 다른 사용자 프로세스 포함 (**a**ll) · `u` : 사용자 지향 형식(%CPU %MEM 열) (**u**ser) · `x` : 제어 터미널 없는 프로세스 포함
- `-e` : 전체 프로세스 (**e**very) · `-f` : 풀 포맷(UID PID PPID C STIME TTY TIME CMD) (**f**ull) · `-l` : long 포맷(PRI NI WCHAN 열) (**l**ong)
- `-u <user>` : 해당 사용자 소유 · `-p <PID>` : PID 지정 · `-C <cmd>` : 명령 이름 지정
- `--forest` : 부모-자식을 ASCII 트리로 · `-j` : 잡 형식(PGID SID) · `-H` : 계층 표시(**H**ierarchy)
- `-o <필드,…>` : 출력 열 직접 지정 (**o**utput format) · `--sort=-%cpu` : `%cpu` 내림차순(`-` 접두) 정렬
- 필드: `ni`(nice) `pri`(우선순위) `stat`(상태) `rss`(실메모리 KiB) `vsz`(가상메모리 KiB) `time`(누적 CPU 시간)

**검증**

```bash
ps aux | head -1        # BSD 헤더에 %CPU %MEM STAT 존재
ps -ef | head -1        # UNIX 헤더에 PPID C STIME 존재, %CPU 없음
ps -e | wc -l
```

기대 출력

```text
USER  PID %CPU %MEM    VSZ   RSS TTY  STAT START   TIME COMMAND
UID   PID  PPID  C STIME TTY          TIME CMD
1xx
```

> 📝 **시험 포인트**: `ps aux`(BSD, 하이픈 없음, %CPU·%MEM·STAT 출력) vs `ps -ef`(UNIX, PPID·STIME 출력, %CPU 없음) — 필기 R04 #35. `-aux` 처럼 하이픈을 붙이면 사용자 `x` 로 해석되어 경고.

### 1-2. STAT 코드 읽기

> **상황**: `ps` 의 STAT 열에서 R/S/D/T/Z 와 부가 기호를 실제 프로세스에서 찾아 대응시킨다.

```bash
ps -eo pid,stat,cmd | awk 'NR==1 || $2 ~ /^R/'      # 실행 중
ps -eo pid,stat,cmd | awk '$2 ~ /^D/'               # I/O 대기 (평시엔 거의 없음)
ps -eo pid,stat,cmd | awk '$2 ~ /^Z/'               # 좀비
ps -eo pid,stat,cmd | awk '$2 ~ /^I/' | head -3     # 유휴 커널 스레드
ps -o pid,stat,cmd $$                               # 현재 셸: Ss+ 계열
```

| 코드 | 의미 | 비고 |
| --- | --- | --- |
| `R` | Running/Runnable — 실행 중 또는 실행 대기 큐 | load average 계산 대상 |
| `S` | Interruptible Sleep — 시그널로 깨어남 | 대부분의 데몬 평시 상태 |
| `D` | Uninterruptible Sleep — 주로 디스크 I/O 대기 | `kill -9` 도 즉시 안 통함, load average 계산 대상 |
| `T` | Stopped — `Ctrl+Z`(SIGTSTP)·SIGSTOP 으로 정지 | `t` 는 디버거 트레이스 정지 |
| `Z` | Zombie — 종료됐으나 부모가 `wait()` 미회수 | 프로세스 테이블만 점유 |
| `I` | Idle 커널 스레드 (procps-ng 3.3.x 이후) | `kworker` 등 |
| `s` | 세션 리더 | `sshd`, 로그인 셸 |
| `l` | 멀티스레드(CLONE_THREAD) | `java`, `dockerd` |
| `<` | 높은 우선순위(NI 음수) | root 만 부여 가능 |
| `N` | 낮은 우선순위(NI 양수) | `nice -n 10` 적용 시 |
| `+` | 포그라운드 프로세스 그룹 | 터미널 소유 |
| `L` | 메모리 페이지 잠김(mlock) | 실시간 프로세스 등 |

**검증**

```bash
ps -o stat= -p $$          # 현재 셸의 STAT 만
sleep 100 &  ps -o pid,stat,cmd -p $!    # 백그라운드 sleep: S (+ 없음)
kill %1
```

기대 출력

```text
Ss
    PID STAT CMD
   ... S    sleep 100
```

> 📝 **시험 포인트**: `D` 는 인터럽트 불가 → 시그널로 종료 불가, `Z` 는 이미 종료된 상태 → `kill -9` 무의미. `S+` 와 `Ss` 부가 기호 의미 — 필기 R03 #38, R08 #32.

### 1-3. `pstree` · `pgrep` · `pidof`

> **상황**: 트리로 부모-자식 관계를 보고, 이름·사용자·패턴으로 PID 를 정확히 뽑아낸다(`ps | grep` 의 자기포함 문제 회피).

```bash
pstree -p | head -20                 # 전체 트리 + PID
pstree -p -u dev1 2>/dev/null        # 소유자 전환 지점에 (user) 표기
pstree -p $(pidof -s sshd)           # sshd 하위 트리
pgrep sshd                           # 이름(comm) 일치 PID
pgrep -l sshd                        # PID + 이름
pgrep -a sshd                        # PID + 전체 명령행
pgrep -u dev1                        # dev1 소유
pgrep -f 'sleep 1'                   # 전체 명령행 패턴 (-f 없으면 'sleep' 만 매칭)
pgrep -x sshd                        # 이름 정확 일치 (exact)
pgrep -c sshd                        # 개수
pidof sshd                           # 이름 → PID 목록 (공백 구분)
```

- `pstree -p` : PID 병기 (**p**id) · `-u` : UID 변경 지점 표시 (**u**id) · `-a` : 인자 표시 · `-s <PID>` : 해당 PID 의 조상만
- `pgrep -l` : 이름 병기 (**l**ist) · `-a` : 전체 명령행 (**a**) · `-u` : 사용자 · `-f` : 명령행 전체 매칭 (**f**ull) · `-x` : 정확 일치 (e**x**act) · `-c` : 개수 (**c**ount) · `-n`/`-o` : 최신/최고령 1개
- `pidof -s` : 1개만 반환 (**s**ingle) · `-x` : 스크립트 이름도 포함

**검증**

```bash
pgrep -x sshd | head -1;  pidof sshd | tr ' ' '\n' | head -1     # 같은 PID 인지
ps -p "$(pgrep -d, sshd)" -o pid,cmd                             # -d 구분자 , 로 ps 에 전달
```

기대 출력

```text
8xx
8xx
    PID CMD
    8xx sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
    ...
```

> 📝 **시험 포인트**: 실기 R01 #4 "`nginx` 포함 프로세스 PID 조회" → `pgrep nginx` 또는 `ps -ef | grep [n]ginx | awk '{print $2}'`. `pgrep -f` 는 인자까지 매칭, 없으면 15자 프로세스명만.

### 1-4. `/proc/PID/` 직접 열어보기

> **상황**: `ps`·`top` 이 보여주는 값은 모두 `/proc` 에서 나온다. 현재 셸의 디렉터리를 열어 원본 정보를 확인한다.

```bash
echo $$ $PPID                                 # 내 셸 PID, 부모 PID
ls /proc/$$/
cat /proc/$$/status | head -12                # Name State Tgid Pid PPid Uid Gid ... VmRSS Threads
tr '\0' ' ' < /proc/$$/cmdline; echo          # NUL 구분 명령행
ls -l /proc/$$/fd                             # 열린 파일 디스크립터 (0 1 2 = stdin/out/err)
tr '\0' '\n' < /proc/$$/environ | head -5     # 환경변수
cat /proc/$$/limits | head -8                 # ulimit 값 (open files, processes)
cat /proc/$$/oom_score /proc/$$/oom_score_adj # OOM killer 점수 (높을수록 먼저 종료)
readlink /proc/$$/exe /proc/$$/cwd            # 실행 파일, 작업 디렉터리
cat /proc/loadavg                             # 1분 5분 15분 실행중/전체 마지막PID
cat /proc/1/comm; ps -p 1 -o pid,comm,cmd     # PID 1 = systemd
nproc; grep -c ^processor /proc/cpuinfo       # CPU 개수 (load 해석 기준)
```

- `$$` : 현재 셸 PID · `$PPID` : 부모 PID · `$!` : 직전 백그라운드 PID · `$?` : 직전 종료 코드
- `status` : 상태·PPID·UID·VmRSS·Threads 요약 · `cmdline`/`environ` : NUL(`\0`) 구분 → `tr` 로 변환
- `fd/` : 열린 FD 심볼릭 링크 · `limits` : 프로세스별 리소스 한도 · `oom_score` : 0~1000, `oom_score_adj` -1000(제외)~1000
- `/proc/loadavg` 5필드 : 1·5·15분 load, `실행중/전체 스케줄 개체`, 마지막 할당 PID

**검증**

```bash
grep -E '^(Name|State|PPid|VmRSS|Threads)' /proc/$$/status
uptime                                  # /proc/loadavg 앞 3개와 동일
```

기대 출력

```text
Name:   bash
State:  S (sleeping)
PPid:   ...
VmRSS:      ... kB
Threads:        1
 ...  up ...,  2 users,  load average: 0.0x, 0.0x, 0.0x
```

> 📝 **시험 포인트**: PID 1 = `systemd`(SysV 는 `init`), PID 0 = 스케줄러. `$$`(현재 셸 PID) vs `$!`(백그라운드 PID) 혼동 — 필기 R03 #35. load average 는 R+D 프로세스 평균 개수, 코어 수 대비 해석 — R08 #30.

### 1-5. 부모-자식 · `exec` · 포그라운드/백그라운드/데몬

> **상황**: 셸이 명령을 실행하는 fork→exec 흐름과, `exec` 로 셸 자체를 다른 프로그램으로 교체하는 차이를 PID 로 증명한다.

```bash
bash                                    # 자식 셸 생성 (fork+exec)
echo "child=$$ parent=$PPID"            # 부모 = 원래 셸 PID
ps -o pid,ppid,cmd -p $$ -p $PPID
exit                                    # 자식 셸 종료 → 원래 셸 복귀

bash -c 'echo before=$$; exec sleep 30' &     # exec: 같은 PID 가 sleep 으로 교체됨
sleep 1; ps -o pid,ppid,cmd -p $!             # PID 동일, CMD 가 sleep 30
kill $!

ps -eo pid,ppid,tty,stat,cmd | awk '$3=="?" && $4 ~ /^Ss/' | head -5   # 데몬: TTY ? + 세션리더
```

- fork : 부모 복제 → PID 새로 생성 · exec : 현 프로세스 코드 교체 → **PID 유지**
- 포그라운드 : 터미널 입력을 점유(`+`) · 백그라운드 : `&` 로 터미널 반환 · 데몬 : 터미널 없음(`TTY ?`), 세션 리더(`s`), 부모가 보통 PID 1
- `bash -c '<문자열>'` : 문자열을 명령으로 실행 (**c**ommand)

**검증**

```bash
ps -o pid,ppid,tty,stat,cmd -C sshd | head -2      # sshd 리스너: PPID 1, TTY ?
```

기대 출력

```text
    PID    PPID TT       STAT CMD
    8xx       1 ?        Ss   sshd: /usr/sbin/sshd -D [listener] ...
```

> 📝 **시험 포인트**: "셸이 외부 명령을 실행하는 순서 = fork → exec → wait → exit" — 필기 R07 #6. `exec` 는 새 PID 없음이 함정. 데몬 = 백그라운드 상주, 터미널 없음.

---

## 2. 잡 제어

### 2-1. `&` · `jobs` · `fg` · `bg` · `Ctrl+Z`

> **상황**: 긴 작업을 백그라운드로 보내고, 잡 번호로 전환·재개·종료하는 흐름을 손에 익힌다.

```bash
sleep 300 &                  # [1] PID
sleep 400 &                  # [2] PID
jobs -l                      # 잡 번호 + PID + 상태
fg %1                        # 1번 잡 포그라운드 →  Ctrl+Z  로 정지(SIGTSTP)
jobs                         # [1]+ Stopped
bg %1                        # 정지된 잡을 백그라운드에서 재개 (SIGCONT)
fg %2                        # → Ctrl+C 로 종료(SIGINT)
kill %1                      # 잡 번호로 SIGTERM
wait                         # 모든 백그라운드 잡 종료 대기 (즉시 반환)
jobs
```

- `&` : 백그라운드 실행 · `jobs -l` : PID 병기 (**l**ong) · `-p` : PID 만 · `-r`/`-s` : 실행중/정지만
- `fg %N` : N번 잡 포그라운드 (**f**ore**g**round) · `bg %N` : 정지 잡 백그라운드 재개 · `%N` 생략 시 현재 잡(`+`)
- `Ctrl+Z` : SIGTSTP(20) 정지 · `Ctrl+C` : SIGINT(2) 종료 · `Ctrl+\` : SIGQUIT(3) 종료+코어 · `Ctrl+D` : EOF(시그널 아님, 입력 끝)
- `kill %N` : 잡 번호에 시그널 · `wait [PID]` : 자식 종료 대기, `$?` 로 종료 코드

**검증**

```bash
sleep 500 &  jobs -l;  ps -o pid,stat,cmd -p $!
kill -STOP %1; jobs;  ps -o stat= -p $!      # T
kill -CONT %1; jobs;  ps -o stat= -p $!      # S
kill %1; sleep 0.5; jobs
```

기대 출력

```text
[1]+ 12345 Running                 sleep 500 &
[1]+  Stopped                 sleep 500
T
[1]+  Running                 sleep 500 &
S
[1]+  Terminated              sleep 500
```

> 📝 **시험 포인트**: "`Ctrl+Z` 로 정지한 작업을 백그라운드에서 계속 실행" = `bg` — 필기 R01 #38. `jobs` 출력에서 `[1]-`·`[2]+` 중 특정 잡을 포그라운드로 = `fg %1` — R08 #34. 잡 번호(`%1`) ≠ PID.

### 2-2. `nohup` · `disown` · `setsid` — 로그아웃 후에도 실행

> **상황**: `dev1` 이 SSH 접속을 끊어도 계속 돌아야 하는 작업 세 가지 방식으로 띄우고, 실제로 세션을 끊은 뒤 살아있는지 확인한다.

```bash
su - dev1
cd ~
nohup sleep 600 &                     # nohup.out 생성 (출력 리다이렉트 없을 때)
nohup sleep 601 > ~/job.log 2>&1 &    # 로그 지정 (권장 형태)
sleep 602 & disown                    # 이미 띄운 잡을 셸 잡 테이블에서 제거
setsid sleep 603 > /dev/null 2>&1 &   # 새 세션(SID) 으로 완전 분리
jobs -l                               # disown·setsid 건은 목록에 없음
ls -l nohup.out job.log
exit                                  # dev1 로그아웃 (SIGHUP 발생)
```

- `nohup <cmd> &` : SIGHUP 무시 + stdout/stderr 를 `nohup.out` 으로 (**no** **h**ang**up**)
- `disown [%N]` : 잡 테이블에서 제거 → 셸 종료 시 SIGHUP 미전달. `-h` : 제거 대신 HUP 만 면제
- `setsid <cmd>` : 새 세션 리더로 실행 → 터미널과 무관 (**set** **s**ession **id**)
- `2>&1` : stderr 를 stdout 으로 합침 (순서: `> file 2>&1`)

**검증**

```bash
pgrep -u dev1 -a sleep                          # 4개 모두 생존
ps -o pid,ppid,sid,tty,stat,cmd -u dev1 | grep sleep
cat /home/dev1/nohup.out                        # "nohup: ignoring input" 등
pkill -u dev1 sleep                             # 정리
```

기대 출력

```text
1234 sleep 600
1235 sleep 601
1236 sleep 602
1237 sleep 603
    PID    PPID     SID TT       STAT CMD
   1234       1    ...  ?        S    sleep 600
   ...
   1237       1   1237  ?        Ss   sleep 603     ← setsid: 자기 자신이 세션 리더
```

> 📝 **시험 포인트**: `nohup cmd &` 출력은 `nohup.out`, 이미 리다이렉트했으면 생성 안 됨, 셸이 죽어도 프로세스 유지 — 필기 R04 #38. 로그아웃 후 PPID 는 1 로 재지정(고아 입양).

### 2-3. `timeout` · `watch`

> **상황**: 무한 대기 위험이 있는 명령에 시간 상한을 걸고, 반복 관찰이 필요한 명령은 `watch` 로 자동 갱신한다.

```bash
timeout 5 sleep 30; echo "exit=$?"            # 5초 뒤 SIGTERM, 종료 코드 124
timeout -s KILL 3 sleep 30; echo "exit=$?"    # SIGKILL 지정 → 137
timeout -k 2 5 sleep 30                       # 5초 뒤 TERM, 2초 더 버티면 KILL
watch -n 1 -d 'cat /proc/loadavg; free -m | head -2'   # 1초 간격, 변경 부분 강조 →  Ctrl+C
```

- `timeout <초> <cmd>` : 시간 초과 시 SIGTERM · `-s <SIG>` : 보낼 시그널 (**s**ignal) · `-k <초>` : TERM 후 추가 대기 뒤 KILL (**k**ill-after) · 접미사 `s m h d`
- 종료 코드 124 = 시간 초과, 137 = 128+9(KILL)
- `watch -n <초>` : 갱신 간격 (i**n**terval) · `-d` : 이전 출력과 차이 강조 (**d**ifferences) · `-t` : 헤더 숨김

**검증**

```bash
time timeout 2 sleep 10
```

기대 출력

```text
real    0m2.00xs
```

> 📝 **시험 포인트**: `timeout` 기본 시그널 TERM, 종료 코드 124. `watch -n 1 <명령>` 은 실기에서 "1초마다 반복 실행" 으로 출제.

---

## 3. 시그널

### 3-1. 시그널 목록과 번호표

> **상황**: 시그널 이름↔번호를 실제 시스템에서 출력해 표와 대조한다(aarch64 도 x86_64 와 동일).

```bash
kill -l                     # 전체 목록 (번호) 이름
kill -l 9; kill -l TERM     # 번호↔이름 변환
kill -l | tr '\t' '\n' | grep -E '\b(1|2|3|9|15|17|18|19|20)\)'
trap -l                     # bash 내장 동일 목록
```

| 번호 | 이름 | 기본 동작 | 발생·용도 | 무시/가로채기 |
| --- | --- | --- | --- | --- |
| 1 | SIGHUP | 종료 | 터미널 끊김. 데몬은 관용적으로 **설정 재읽기** | 가능 |
| 2 | SIGINT | 종료 | `Ctrl+C` | 가능 |
| 3 | SIGQUIT | 종료+코어덤프 | `Ctrl+\` | 가능 |
| 9 | SIGKILL | **강제 종료** | 최후 수단, 정리 코드 미실행 | **불가** |
| 15 | SIGTERM | 종료 | `kill` **기본값**, 정상 종료 요청 | 가능 |
| 17 | SIGCHLD | 무시 | 자식 종료 시 부모에게 통지 | 가능 |
| 18 | SIGCONT | 재개 | 정지 프로세스 재개 (`bg`, `fg`) | — |
| 19 | SIGSTOP | 정지 | 강제 정지 | **불가** |
| 20 | SIGTSTP | 정지 | `Ctrl+Z` (터미널 정지) | 가능 |
| 10/12 | SIGUSR1/2 | 종료 | 사용자 정의 (로그 회전 등) | 가능 |

- `kill -l` : 시그널 목록 (**l**ist) · `kill -l <번호|이름>` : 상호 변환
- `kill -9 PID` = `kill -KILL PID` = `kill -SIGKILL PID` = `kill -s KILL PID` 모두 동일

**검증**

```bash
kill -l 1 2 3 9 15 18 19 20
```

기대 출력

```text
HUP
INT
QUIT
KILL
TERM
CONT
STOP
TSTP
```

> 📝 **시험 포인트**: 9/15/1/2/19/20 매핑이 최빈출 — 필기 R03 #10(SIGINT 는 3 아닌 2), R01 #16·R07 #7(가로챌 수 없는 것 = 9 KILL, 19 STOP), R04 #37(기본 = 15 TERM), R10 #13(STOP/CONT).

### 3-2. `kill -15` vs `kill -9` · `kill -HUP` · `killall` · `pkill`

> **상황**: 폭주 프로세스 대응 절차는 "TERM 으로 정상 종료 요청 → 무응답 시 KILL". 이름·사용자·패턴 기반 도구까지 함께 쓴다.

```bash
stress-ng --cpu 1 --timeout 600 &              # 대상 생성
PID=$(pgrep -n stress-ng)                       # 가장 최근 1개
kill -15 $PID; sleep 1; pgrep -a stress-ng      # TERM: stress-ng 는 정리 후 종료
stress-ng --cpu 1 --timeout 600 &
kill -0 $! && echo alive                        # -0: 존재 여부만 확인
kill -9 $!; sleep 0.5; kill -0 $! 2>/dev/null || echo gone

# HUP = 데몬 설정 재읽기 (sshd 리스너 MainPID 에만)
kill -HUP $(systemctl show -p MainPID --value sshd)
journalctl -u sshd -n 3 --no-pager             # "Received SIGHUP; restarting."

# 이름·사용자·패턴 기반
stress-ng --cpu 1 --timeout 600 & stress-ng --cpu 1 --timeout 600 &
killall stress-ng                              # 이름 일치 전부 TERM
pgrep -a stress-ng || echo none
su - dev1 -c 'nohup sleep 700 >/dev/null 2>&1 &'
pkill -9 -f 'sleep 700'                        # 패턴(-f 전체 명령행) 로 KILL
```

⚠️ `killall -u dev1` 은 해당 사용자의 **로그인 셸·SSH 세션까지 전부** 종료 → 대상 재확인 후 실행

```bash
pgrep -u dev1 -a                               # 먼저 목록 확인
killall -u dev1                                # dev1 프로세스 전부 TERM (⚠️)
```

- `kill -15`/`-TERM` : 정상 종료 요청(기본) · `-9`/`-KILL` : 강제 종료, 파일·락 정리 안 됨 · `-0` : 전송 없이 존재·권한 확인
- `-HUP` : 데몬 설정 재로드 관용. `pidof sshd` 는 세션 자식 PID 도 포함하므로 **MainPID 만** 지정
- `systemctl show -p MainPID --value <svc>` : 유닛 주 PID 만 출력 (`systemctl reload sshd` 가 동일 효과)
- `killall <이름>` : 프로세스 **이름** 일치 전부 · `-u <user>` : 사용자 소유 전부 · `-i` : 확인 프롬프트 · `-w` : 종료까지 대기
- `pkill -f <패턴>` : 명령행 패턴 · `-u <user>` · `-9` · `-x` : 정확 일치. 실행 전 같은 인자로 `pgrep -a` 필수

**검증**

```bash
pgrep -a stress-ng; pgrep -u dev1 -a
systemctl is-active sshd                       # HUP 이후에도 active
```

기대 출력

```text
(출력 없음)
active
```

> 📝 **시험 포인트**: 필기 R06 #35 "top 에 97.8% 프로세스 → `kill -15` 후 무응답 시 `kill -9`" 가 정답 절차. `kill`=PID, `killall`=이름(R04 #36), `pkill`=패턴/속성. HUP 로 설정 재적재 = `kill -HUP 1234` 또는 `kill -1 1234`(R01 #37). 일반 사용자는 root 프로세스에 시그널 불가(R09 #20, 커널 UID 검사).

### 3-3. `kill -STOP` / `-CONT` 일시정지·재개

> **상황**: 종료하기엔 아까운 배치 작업을 잠시 멈추고 나중에 재개한다. CPU 사용률이 0 이 되는 것을 확인한다.

```bash
stress-ng --cpu 1 --timeout 600 &
P=$(pgrep -n stress-ng)
kill -STOP $P; sleep 2
ps -o pid,stat,%cpu,cmd -p $P              # STAT T
top -b -n 2 -d 1 -p $P | tail -2           # %CPU 0.0
kill -CONT $P; sleep 2
ps -o pid,stat,%cpu,cmd -p $P              # STAT R
kill $P
```

- `-STOP`(19) : 무시 불가 정지 · `-TSTP`(20) : 터미널 정지(무시 가능) · `-CONT`(18) : 재개
- `top -p` 첫 프레임은 부팅 이후 누적 기준이라 부정확 → `-n 2` 로 두 번째 프레임 사용

**검증**

```bash
pgrep -a stress-ng || echo none
```

기대 출력

```text
    PID STAT %CPU CMD
  12345 T    ...  stress-ng --cpu 1 ...
  12345 R    ...  stress-ng --cpu 1 ...
none
```

> 📝 **시험 포인트**: SIGSTOP(19) 은 가로챌 수 없는 정지, SIGCONT(18) 로만 재개 — R10 #13. `Ctrl+Z` 는 SIGTSTP(20).

### 3-4. `trap` 으로 시그널 처리 스크립트 검증

> **상황**: 데몬처럼 SIGTERM 을 받으면 정리 후 종료하고 SIGHUP 을 받으면 설정을 다시 읽는 스크립트를 만들고, `kill` 로 각각 보내 동작을 눈으로 확인한다.

```bash
cat > /usr/local/bin/trapdemo.sh <<'EOF'
#!/bin/bash
CONF=/tmp/trapdemo.conf
load() { echo "[$(date +%T)] 설정 읽기: $(cat $CONF 2>/dev/null || echo 기본값)"; }
trap 'echo "[$(date +%T)] SIGTERM 수신 → 정리 후 종료"; rm -f /tmp/trapdemo.pid; exit 0' TERM
trap 'echo "[$(date +%T)] SIGHUP 수신 → 재설정"; load' HUP
trap 'echo "[$(date +%T)] SIGINT 는 무시"' INT
echo $$ > /tmp/trapdemo.pid; load
while true; do sleep 1; done
EOF
chmod +x /usr/local/bin/trapdemo.sh
/usr/local/bin/trapdemo.sh > /tmp/trapdemo.log 2>&1 &
P=$(cat /tmp/trapdemo.pid)
echo "level=debug" > /tmp/trapdemo.conf
kill -HUP $P; sleep 1.5
kill -INT $P; sleep 1.5                 # 무시됨 → 계속 실행
kill -TERM $P; sleep 1.5                # 핸들러 실행 후 종료
cat /tmp/trapdemo.log
```

- `trap '<명령>' <SIG…>` : 시그널 수신 시 실행할 명령 등록 · `trap '' SIG` : 무시 · `trap - SIG` : 기본 동작 복원 · `trap -p` : 등록 목록
- `sleep 1` 루프 : 핸들러는 현재 포그라운드 명령(sleep)이 끝난 뒤 실행 → 최대 1초 지연
- `KILL`(9)·`STOP`(19) 은 `trap` 등록 자체가 무의미(커널이 직접 처리)

**검증**

```bash
trap '' KILL 2>&1 | head -1 ; true            # 등록 시도 → 오류 또는 무시
/usr/local/bin/trapdemo.sh > /tmp/trapdemo.log 2>&1 &  sleep 1
kill -9 $(cat /tmp/trapdemo.pid); sleep 0.5
pgrep -f trapdemo.sh || echo "KILL 은 핸들러 없이 즉시 종료";  tail -1 /tmp/trapdemo.log
```

기대 출력

```text
[..:..:..] 설정 읽기: 기본값
[..:..:..] SIGHUP 수신 → 재설정
[..:..:..] 설정 읽기: level=debug
[..:..:..] SIGINT 는 무시
[..:..:..] SIGTERM 수신 → 정리 후 종료
KILL 은 핸들러 없이 즉시 종료
```

> 📝 **시험 포인트**: SIGTERM 은 프로세스가 가로채 정리(임시파일·락 해제) 가능, SIGKILL 은 불가 → "정리 절차가 필요하면 15 먼저". `trap` 문법은 실기 스크립트 문항에 등장.

---

## 4. 우선순위 (nice / renice)

### 4-1. `nice` 로 시작 우선순위 지정

> **상황**: 야간 배치처럼 다른 서비스에 영향을 주지 않을 작업은 가장 낮은 우선순위(NI 19)로 시작한다. 음수(높은 우선순위)는 root 만 가능하다는 것도 검증한다.

```bash
nice                                            # 현재 셸 NI (0)
nice -n 10 sleep 800 &  ps -o pid,ni,pri,stat,cmd -p $!     # NI 10, PR 30, STAT SN
nice -n 19 sleep 801 &  ps -o pid,ni,pri,cmd -p $!          # NI 19 (최저)
nice -n -5 sleep 802 &  ps -o pid,ni,pri,cmd -p $!          # root: NI -5, PR 15, STAT S<
su - dev1 -c 'nice -n -5 sleep 1'               # 일반 사용자: Permission denied
su - dev1 -c 'nice -n 25 sleep 803 & sleep 0.2; ps -o ni= -C sleep | sort -u | tail -1'   # 19 로 상한 클램프
```

- `nice -n <값> <cmd>` : NI 가산값 지정 (**n**iceness). 생략 시 +10. 범위 **-20(최고) ~ 19(최저)**
- `PR`(ps `pri`, top `PR`) = 20 + NI (일반 스케줄링 정책 기준). NI 0 → PR 20, NI 19 → PR 39, NI -20 → PR 0
- 음수 NI 는 root(`CAP_SYS_NICE`) 만. 일반 사용자는 올리기(양보)만, `/etc/security/limits.conf` 의 `nice` 항목으로 허용 범위 조정 가능 → [[03-user-group-permission]]

**검증**

```bash
ps -o pid,user,ni,pri,stat,cmd -C sleep
top -b -n 1 -o -NI | grep sleep       # top 의 NI·PR 열과 대조
```

기대 출력

```text
    PID USER      NI PRI STAT CMD
   ...  root      10  30 SN   sleep 800
   ...  root      19  39 SN   sleep 801
   ...  root      -5  15 S<   sleep 802
   ...  dev1      19  39 SN   sleep 803
```

> 📝 **시험 포인트**: "가장 낮은 우선순위로 배치 실행" = `nice -n 19 ./batch.sh` — 필기 R06 #36. NI 값이 **클수록 우선순위 낮음**, 기본 0, 음수는 root 만 — R01 #35, R03 #28, R07 #40, R10 #33. `ps -l` 의 NI 5 = 기본보다 낮은 우선순위 — R02 #35, R08 #32.

### 4-2. `renice` 로 실행 중 프로세스 변경

> **상황**: 이미 돌고 있는 프로세스의 우선순위를 PID·사용자·그룹 단위로 바꾼다. 일반 사용자는 낮추기만 가능하다.

```bash
P=$(pgrep -f 'sleep 800')
renice -n 5 -p $P                      # NI 10 → 5 (root 는 올릴 수 있음)
renice -n 15 $P                        # -p 생략 가능 (기본 PID)
renice -n -10 -p $P                    # 음수도 root 가능
renice -n 19 -u dev1                   # dev1 소유 전체
renice -n 10 -g $(getent group devteam | cut -d: -f3)   # 프로세스 그룹 ID 기준 (PGID)
su - dev1 -c "renice -n 5 -p $(pgrep -f 'sleep 803')"   # 19 → 5 : 일반 사용자는 낮은 값으로 못 감 → Permission denied
```

- `renice -n <값> -p <PID>` : 절대 NI 값 설정 (nice 와 달리 **가산 아님**) · `-p` : PID (**p**rocess, 기본) · `-u` : 사용자 (**u**ser) · `-g` : 프로세스 그룹 ID (**g**roup)
- 일반 사용자는 자기 프로세스의 NI 를 **현재보다 큰 값으로만** 변경 가능(한 번 올리면 되돌리기 불가)
- 참고: `chrt -f 10 <cmd>` / `chrt -p PID` 실시간(SCHED_FIFO) 정책, `ionice -c 3 <cmd>` I/O 우선순위(idle 클래스) — 시험은 개념 수준

**검증**

```bash
ps -o pid,user,ni,pri,cmd -C sleep
pkill -f 'sleep 80'                    # 800~803 정리
```

기대 출력

```text
    PID USER      NI PRI CMD
   ...  root     -10  10 sleep 800
   ...  root      19  39 sleep 801
   ...  root      -5  15 sleep 802
   ...  dev1      19  39 sleep 803
```

> 📝 **시험 포인트**: "PID 1234 nice 값을 10 으로" = `renice -n 10 -p 1234`(또는 `renice 10 1234`) — R02 #36. "-5 로(root)" = `renice -n -5 -p 1234` — R08 #43. `nice` 는 새 명령 시작, `renice` 는 실행 중 변경. R06 #35 의 `nice -n 19 2874` 는 오답(nice 는 PID 를 받지 않음).

### 4-3. 검증 — 같은 CPU 에 NI 0 vs NI 19 부하 경쟁

> **상황**: 2 vCPU 에서는 부하 2개가 각자 코어를 차지해 차이가 안 보이므로, `taskset` 으로 **같은 코어 0** 에 묶어 NI 차이가 %CPU 배분에 미치는 영향을 `top` 에서 관찰한다.

```bash
taskset -c 0 nice -n 0  stress-ng --cpu 1 --timeout 120 &
taskset -c 0 nice -n 19 stress-ng --cpu 1 --timeout 120 &
sleep 5
top -b -n 2 -d 2 -p $(pgrep -d, -f 'stress-ng-cpu') | tail -3
# 대체: yes > /dev/null 을 taskset -c 0 nice -n 19 로 띄우고 비교
```

- `taskset -c <코어> <cmd>` : CPU 친화도 고정 (**c**pu-list). `-p PID` 로 실행 중 프로세스 변경
- `stress-ng --cpu N` : CPU 워커 N개 · `--timeout <초>` : 자동 종료. 워커 프로세스 이름은 `stress-ng-cpu`

**검증**

```bash
ps -o pid,ni,pri,psr,%cpu,cmd -C stress-ng-cpu      # psr = 실행 중인 코어 번호 (둘 다 0)
pkill stress-ng
```

기대 출력

```text
    PID  NI PRI PSR %CPU CMD
   ...    0  20   0 9x.x stress-ng-cpu ...
   ...   19  39   0  x.x stress-ng-cpu ...
```

> 📝 **시험 포인트**: NI 19 프로세스는 NI 0 과 경쟁 시 CPU 를 거의 양보(CFS 가중치 15 vs 1024). 코어가 남으면 NI 가 커도 100% 쓸 수 있음 — "nice 는 경쟁 상황에서만 의미".

---

## 5. top 심화

### 5-1. 실행 옵션 전부 써보기

> **상황**: 대화식으로 보는 것 외에, 스크립트·리포트용 배치 출력과 특정 PID·사용자·정렬 필드 지정 옵션을 하나씩 확인한다.

```bash
top                                  # 기본 (q 로 종료)
top -d 2                             # 2초 갱신
top -n 3 -d 1                        # 3회 갱신 후 종료
top -b -n 1 | head -20               # 배치 모드: 파일·스크립트용
top -b -n 2 -d 1 > /tmp/top.$(date +%F_%H%M).txt   # 저장 (2번째 프레임이 정확)
top -p $(pgrep -d, sshd)             # 특정 PID 들만 (최대 20개, 쉼표 구분)
top -u dev1                          # 유효 UID 기준 사용자 필터
top -U dev1                          # 실효·저장·파일시스템 UID 중 하나라도 일치
top -o %MEM                          # 메모리 사용률 정렬 (필드명은 헤더 표기와 동일)
top -o +%CPU -n 1 -b | head -12      # +: 내림차순(기본), -: 오름차순
top -H                               # 스레드 단위 표시
top -c                               # 전체 명령행(인자 포함)
top -i                               # idle(0% CPU) 프로세스 숨김
top -w 200 -b -n 1 | head -12        # 출력 폭 지정 (배치 시 COMMAND 잘림 방지)
top -1                               # 시작 시 CPU 코어별 표시
top -S                               # 누적 CPU 시간 모드 (TIME+ 에 자식 시간 포함)
top -E g                             # 요약 영역 메모리 단위 GiB (k m g t p e)
top -e m                             # 태스크 영역 메모리 단위 MiB (procps-ng 3.3.17)
top -s                               # 보안 모드: k(kill)·r(renice)·d(주기 변경) 금지
top -b -n 1 -H -p $(pidof dockerd 2>/dev/null || pgrep -n sshd) | head
```

- `-d <초>` : 갱신 주기 (**d**elay, 소수 가능) · `-n <횟수>` : 반복 후 종료 (**n**umber) · `-b` : 배치 모드, 키 입력 없음 (**b**atch)
- `-p <PID,…>` : 지정 PID 만 · `-u <user>` : 유효 UID (**u**ser) · `-U <user>` : 모든 UID 종류 대상
- `-o <필드>` : 정렬 필드 (**o**rder), 접두 `+` 내림·`-` 오름 · `-H` : 스레드 모드 (t**H**reads) · `-c` : 명령행 토글 (**c**ommand line) · `-i` : idle 숨김 토글 · `-w [폭]` : 출력 폭 (**w**idth)
- `-1` : 코어별 CPU 행 · `-S` : 누적 시간 (**S**um) · `-E`/`-e <단위>` : 요약/태스크 영역 메모리 단위(**E**xtend) · `-s` : 보안 모드 (**s**ecure)
- 배치 모드 첫 프레임의 %CPU 는 부팅 이후 평균에 가까움 → 진단 리포트는 `-n 2` 뒤 프레임 사용

**검증**

```bash
top -b -n 1 -o %MEM | sed -n '7,9p'          # 헤더 다음 2행이 RES 큰 순인지
top -b -n 1 -u dev1 | tail -n +8 | awk '{print $2}' | sort -u   # USER 열 전부 dev1
top -h 2>&1 | head -3                        # 사용 가능 옵션 요약
```

기대 출력

```text
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
    ... root      20   0  ...     ...    ... S   0.0   x.x   0:0x.xx  ...
dev1
  procps-ng 3.3.17
Usage:
  top -hv | -bcEeHiOSs1 -d secs -n max -u|U user -p pid(s) -o field -w [cols]
```

> 📝 **시험 포인트**: `top -b -n 1` 은 스크립트·리포트용(sysreport.sh), `-d` 주기, `-p` PID 지정, `-u` 사용자 — 옵션 의미 문항. `-n` 은 "반복 횟수" 이며 `nice` 와 무관.

### 5-2. 헤더 5줄 완전 해석

> **상황**: 신고를 받으면 프로세스 목록보다 먼저 헤더 5줄을 읽는다. 각 줄의 필드를 `uptime`·`free`·`nproc` 와 대응시켜 해석 기준을 세운다.

```bash
top -b -n 1 | head -5
uptime; nproc; cat /proc/loadavg
free -m                     # 4·5행과 대응
```

기대 출력 (평시)

```text
top - 14:20:01 up 3 days,  2:11,  2 users,  load average: 0.15, 0.30, 0.45
Tasks: 142 total,   1 running, 141 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.3 us,  0.2 sy,  0.0 ni, 99.5 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   3821.4 total,   2900.1 free,    420.3 used,    501.0 buff/cache
MiB Swap:   1535.9 total,   1535.9 free,      0.0 used.   3200.5 avail Mem
```

| 행 | 필드 | 의미 | 해석 기준 |
| --- | --- | --- | --- |
| 1 | `up` | 가동 시간 | `uptime` 과 동일 |
| 1 | `users` | 로그인 세션 수 | `who | wc -l` |
| 1 | `load average` | 1·5·15분 평균 실행(R)+대기(D) 프로세스 수 | **`nproc` 대비**: 2코어에서 2.0 = 포화, 1분 > 15분 = 상승 추세 |
| 2 | `Tasks` | total / running / sleeping / stopped / zombie | zombie > 0 → 5-7, stopped > 0 → `Ctrl+Z` 잔재 |
| 3 | `us` | 사용자 공간(NI 0 이상 제외한 일반) | 높음 → 응용 프로그램 CPU 폭주(5-5) |
| 3 | `sy` | 커널 공간 | 높음 → 시스템 콜·드라이버·컨텍스트 스위치 과다 |
| 3 | `ni` | NI 양수(낮은 우선순위) 사용자 프로세스 | 높음 → 배치 작업 진행 중(정상) |
| 3 | `id` | 유휴 | 낮을수록 CPU 포화 |
| 3 | `wa` | I/O 완료 대기 | 높음 → **디스크 병목**(5-6), 프로세스 D 상태 |
| 3 | `hi` / `si` | 하드웨어 / 소프트웨어 인터럽트 | 높음 → NIC·디스크 인터럽트 폭주, 네트워크 공격 |
| 3 | `st` | 하이퍼바이저에 빼앗긴 시간(steal) | VM 에서만 의미. 높음 → 호스트 과밀 |
| 4 | `total free used buff/cache` | 물리 메모리 | `free -m` 의 Mem 행 |
| 5 | `Swap total free used` | 스왑 | `used` 증가 + `vmstat` si/so > 0 → 메모리 압박(5-6) |
| 5 | `avail Mem` | 스왑 없이 새 프로세스에 줄 수 있는 추정치 | **free 아닌 avail 기준**으로 판단 (buff/cache 는 회수 가능) |

- `Tasks` 의 running 은 스냅샷 순간 R 상태 개수 → load average 와 다름
- 3행은 `t` 키로 4가지 표시 모드(숫자/그래프/숨김), `1` 키로 코어별 분리

**검증**

```bash
free -m | awk 'NR==2{print "avail="$7} NR==3{print "swap used="$3}'
top -b -n 1 | sed -n '4,5p' | grep -oE '[0-9.]+ avail Mem|[0-9.]+ used'
```

기대 출력

```text
avail=32xx
swap used=0
 0.0 used
3200.5 avail Mem
```

> 📝 **시험 포인트**: `%Cpu(s)` 행에서 `wa` 높음 = I/O 병목 의심(R08 #31), `us` 사용자/`sy` 커널/`id` 유휴 구분. 4코어에서 load 4.05 = 포화 근접·상승 추세(R08 #30). `free` 의 available 이 실제 가용(R08 #28).

### 5-3. 프로세스 열 (PID … COMMAND)

> **상황**: 목록에서 "메모리를 많이 쓴다"를 VIRT 로 오판하지 않도록 세 메모리 열의 차이를 실제 프로세스로 확인한다.

```bash
top -b -n 1 | sed -n '7p'                                   # 열 헤더
top -b -n 1 -o %MEM | sed -n '8,10p'
ps -o pid,vsz,rss,%mem,cmd -p $(pgrep -n sshd)              # VIRT≈VSZ, RES≈RSS (KiB)
grep -E 'VmSize|VmRSS|RssFile|RssShmem' /proc/$(pgrep -n sshd)/status
```

| 열 | 의미 | 비고 |
| --- | --- | --- |
| `PID` | 프로세스 ID | `-H` 모드에선 스레드 ID |
| `USER` | 유효 사용자 | `U` 키로 실제 UID 전환 가능 |
| `PR` | 커널 스케줄 우선순위 | 20+NI. 실시간은 `rt` 또는 음수 |
| `NI` | nice 값 | -20 ~ 19 |
| `VIRT` | 가상 메모리 총량 (코드+데이터+공유 라이브러리+매핑+스왑) | 실제 사용량 아님. `ps vsz` |
| `RES` | 물리 메모리 상주량 (스왑 제외) | **실사용 판단 기준**. `ps rss` |
| `SHR` | 공유 가능 메모리 (공유 라이브러리·shm) | RES 에 포함됨 |
| `S` | 상태 (R S D T Z I) | 1-2 표와 동일 |
| `%CPU` | 갱신 주기 동안 CPU 점유율 | 멀티코어에선 100% 초과 가능(Irix 모드, `I` 키로 Solaris 모드 전환) |
| `%MEM` | RES / 물리 메모리 | |
| `TIME+` | 누적 CPU 시간(1/100초) | `S` 옵션·키로 자식 포함 |
| `COMMAND` | 프로그램 이름 / `c` 키로 전체 명령행 | |

**검증**

```bash
top -b -n 1 -o %MEM | awk 'NR==8{print "VIRT="$5, "RES="$6, "SHR="$7, "CMD="$12}'
```

기대 출력

```text
VIRT=... RES=... SHR=... CMD=...     ← VIRT > RES ≥ SHR
```

> 📝 **시험 포인트**: RES(실사용) vs VIRT(가상) 혼동 문항. `PR` 과 `NI` 관계(PR = 20 + NI). `%CPU` 가 100% 를 넘는 이유(멀티스레드·다코어).

### 5-4. 대화식 키 전부 눌러보기

> **상황**: `top` 을 띄운 뒤 아래 표의 키를 위에서부터 순서대로 실제로 눌러 화면 변화를 확인한다. 부하가 있어야 정렬 차이가 보이므로 먼저 부하 2개를 만든다.

```bash
stress-ng --cpu 1 --timeout 900 &  stress-ng --vm 1 --vm-bytes 512M --vm-keep --timeout 900 &
sleep 800 &
top
```

| 키 | 동작 | 확인할 것 |
| --- | --- | --- |
| `h` / `?` | 도움말 화면 | 현재 유효한 키 목록. 아무 키로 복귀 |
| `Space` / `Enter` | 즉시 갱신 | |
| `d` / `s` | 갱신 주기 변경 (초, 소수 가능) | 프롬프트에 `0.5` 입력 |
| `k` | kill — PID 입력 → 시그널 입력(기본 15) | `stress-ng-cpu` PID → `15`. Tasks 감소 |
| `r` | renice — PID 입력 → NI 값 입력 | `stress-ng-vm` PID → `19`. NI·PR 열 변화 |
| `u` / `U` | 유효/모든 UID 사용자 필터 | `dev1` 입력, 빈 입력으로 해제. `!dev1` 제외 |
| `P` | %CPU 정렬 | 기본. `stress-ng-cpu` 최상단 |
| `M` | %MEM 정렬 | `stress-ng-vm` 최상단 |
| `T` | TIME+ 정렬 | 장기 실행 데몬 상위 |
| `N` | PID 정렬 | |
| `<` / `>` | 정렬 열을 왼쪽/오른�로 이동 | `x` 와 함께 쓰면 어느 열인지 보임 |
| `R` | 정렬 역순 토글 | |
| `x` | 정렬 열 강조 | |
| `y` | 실행 중(R) 행 강조 | |
| `b` | 강조 방식 굵게 ↔ 배경색 | `x`/`y` 활성 상태에서 |
| `z` | 컬러 ↔ 흑백 | |
| `Z` | 색상 편집 (S/M/H/T 영역, 0-7 색) | `q` 로 나옴, `W` 로 저장해야 유지 |
| `1` | CPU 코어별 표시 토글 | `%Cpu0`, `%Cpu1` 두 줄 |
| `t` | CPU 행 표시 모드 순환 (숫자 → 막대 그래프 → 블록 그래프 → 숨김) | 4번 눌러 원복 |
| `m` | 메모리 행 표시 모드 순환 | 동일 |
| `l` | load average 행 토글 | |
| `H` | 스레드 모드 토글 | Tasks → Threads, 항목 수 증가 |
| `c` | COMMAND 이름 ↔ 전체 명령행 | `stress-ng --cpu 1 …` 인자 표시 |
| `V` | 트리(forest) 뷰 토글 | 부모-자식 들여쓰기, `v` 로 하위 접기/펼치기 |
| `f` / `F` | 필드 관리 — 표시 열 선택(`d`/Space)·정렬 열 지정(`s`)·순서 이동(→ 후 ↑↓) | `PPID`, `SWAP`, `nTH`, `P`(코어) 추가. `q` 로 복귀 |
| `i` | idle(0% CPU) 프로세스 숨김 토글 | 부하 프로세스만 남음 |
| `e` / `E` | 태스크/요약 영역 메모리 단위 순환 (k m g t p) | RES 가 MiB 로 |
| `S` | 누적 시간 모드 토글 | TIME+ 에 종료된 자식 시간 포함 |
| `I` | Irix ↔ Solaris 모드 | Solaris 모드는 %CPU 를 코어 수로 나눔 |
| `0` | 0 값 공백 처리 토글 | |
| `L` | 문자열 검색 (Locate) | `stress` 입력 → 해당 행 강조 |
| `&` | 다음 검색 결과 | |
| `o` / `O` | 필터 추가 (대소문자 무시 / 구분). `FLD?VAL` 형식, `!` 부정 | `%CPU>20`, `COMMAND=stress`, `!USER=root` |
| `=` | 현재 창 필터·PID 제한 해제 | `+` 는 모든 창 |
| `n` / `#` | 표시 행 수 제한 | `5` 입력 → 상위 5개 |
| `C` | 좌표(스크롤 위치) 표시 토글 | |
| `j` / `J` | 숫자/문자 열 정렬 방향 토글 | |
| `W` | 현재 설정 저장 → `~/.config/procps/toprc`(디렉터리 존재 시) 또는 `~/.toprc` | 다음 실행 시 자동 적용 |
| `A` | 대체 화면 모드 — 4개 창(Def·Job·Mem·Usr) 동시 표시 | 각 창 정렬 기준 다름 |
| `g` | 창 그룹 선택 (1~4) | `A` 모드에서 현재 창 지정 |
| `a` / `w` | 다음/이전 창으로 이동 | `A` 모드 |
| `-` / `_` | 현재/전체 창 표시 토글 | `A` 모드 |
| `G` | 현재 창 이름 변경 | |
| `q` | 종료 | |

- 필터 `o` 문법: `필드연산자값`, 연산자 `=`(부분 일치) `<` `>` , `!` 접두로 부정. 예: `%CPU>20`, `COMMAND=stress`, `!USER=root`. 여러 개 누적 가능, `=` 로 해제
- `k`·`r` 프롬프트에서 Enter 만 치면 기본값(최상단 PID, 시그널 15) 적용 → 오조작 주의
- `W` 로 저장한 파일은 `cat ~/.toprc` 또는 `cat ~/.config/procps/toprc` 로 확인, 삭제하면 기본값 복귀

**검증**

```bash
# top 안에서: M → 최상단이 stress-ng-vm 인지,  k → 그 PID → 15,  W 로 저장 후 q
ls -l ~/.toprc ~/.config/procps/toprc 2>/dev/null
grep -c . ~/.toprc 2>/dev/null || grep -c . ~/.config/procps/toprc
pgrep -a stress-ng                          # k 로 죽인 것 사라짐
pkill stress-ng; pkill -f 'sleep 800'
```

기대 출력

```text
-rw-r--r--. 1 root root ... /root/.toprc     (또는 .config/procps/toprc)
1x
... stress-ng --cpu 1 --timeout 900          ← vm 만 종료되었으면 cpu 만 남음
```

> 📝 **시험 포인트**: `k`=kill(PID·시그널 입력), `r`=renice, `M`=메모리 정렬, `P`=CPU 정렬, `1`=코어별, `q`=종료 — 필기 R01 #36 에서 각 키 의미 교차 배치 함정(`M` 이 CPU 정렬 ✗).

### 5-5. 진단 시나리오 (a) — CPU 폭주

> **상황**: "서버가 느리다" 신고. `stress-ng` 로 2개 CPU 워커를 띄워 폭주를 재현하고, `top` 만으로 원인 PID 를 특정 → 우선순위 하향 → 정상 종료까지 진행한다.

```bash
# 1) 부하 생성 (대체: yes > /dev/null & yes > /dev/null &)
stress-ng --cpu 2 --timeout 300 &
sleep 10
# 2) top 관찰 (대화식):  1  → 코어별,  P  → CPU 정렬,  c  → 명령행,  x  → 정렬열 강조
top
# 배치 캡처
top -b -n 2 -d 2 | tail -n +$(( $(top -b -n 1 | wc -l) + 1 )) | head -12
```

기대 출력 (부하 중)

```text
top - ... load average: 1.9x, 0.8x, 0.3x          ← 1분 값이 nproc(2) 에 근접, 상승 추세
Tasks: 146 total,   3 running, 143 sleeping, ...
%Cpu0  : 99.x us,  0.x sy, ...  0.0 id ...
%Cpu1  : 99.x us,  0.x sy, ...  0.0 id ...
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   2311 root      20   0   ...     ...    ... R  99.7   0.1   0:12.34 stress-ng-cpu
   2312 root      20   0   ...     ...    ... R  99.3   0.1   0:12.30 stress-ng-cpu
```

```bash
# 3) 보조 도구 교차 확인
ps -eo pid,ppid,ni,pri,stat,%cpu,time,cmd --sort=-%cpu | head -4
mpstat -P ALL 1 2 | tail -4                   # 코어별 %usr ≈ 100, %idle ≈ 0
pidstat -u 1 2 -p $(pgrep -d, stress-ng-cpu)  # 프로세스별 %usr %system
uptime
# 4) 조치 1 — 우선순위 하향:  top 에서  r  → PID → 19   (또는 아래)
renice -n 19 -p $(pgrep -d' ' stress-ng-cpu)
top -b -n 2 -d 2 -p $(pgrep -d, stress-ng-cpu) | tail -2      # NI 19, PR 39, %Cpu(s) 의 us → ni 로 이동
# 5) 조치 2 — 정상 종료:  top 에서  k  → PID → 15   (또는 아래)
kill -15 $(pgrep -d' ' stress-ng-cpu); sleep 2
pgrep -a stress-ng || echo "종료 확인"
# 6) 조치 후 재확인
top -b -n 2 -d 2 | sed -n '1,3p'; uptime      # load 는 1분 값이 서서히 하강 (지수 이동 평균)
```

- `--sort=-%cpu` : CPU 내림차순 · `mpstat -P ALL` : 코어별 통계 (**P**rocessor) · `pidstat -u` : 프로세스별 CPU (**u**tilization)
- `renice` 후 `%Cpu(s)` 의 `us` 가 `ni` 로 옮겨감 — NI 양수 프로세스 CPU 는 `ni` 로 집계
- load average 는 1분 지수 이동 평균이라 종료 직후에도 바로 0 이 되지 않음 → 5·15분 값과 비교해 추세 판단

**검증**

```bash
pgrep -c stress-ng; cat /proc/loadavg
top -b -n 1 | sed -n '3p'
```

기대 출력

```text
0
0.7x 0.6x 0.3x 1/1xx ...
%Cpu(s):  0.x us,  0.x sy,  0.0 ni, 9x.x id, ...
```

> 📝 **시험 포인트**: 대응 절차 = 원인 PID 특정(`top` P 정렬·`ps --sort`) → 우선순위 조정(`renice`) 또는 `kill -15` → 무응답 시 `kill -9`. 즉시 재부팅·즉시 `kill -9` 는 오답(R06 #35).

### 5-6. 진단 시나리오 (b) — 메모리 압박·스왑·OOM

> **상황**: 4 GB VM 에서 3 GB 를 잡고 놓지 않는 프로세스를 띄워 `avail Mem` 감소 → 스왑 사용 → (경우에 따라) OOM killer 까지 관찰한다. Part 05 에서 만든 스왑 1.5 GB 가 실제로 쓰인다.

⚠️ 3 GB 할당은 OOM killer 가 SSH 세션·`tmux` 를 죽일 수 있음. UTM 콘솔을 열어 두고, 불안하면 `--vm-bytes 2G` 로 시작

```bash
swapon --show; free -h                          # 시작 전 기준값 기록
stress-ng --vm 1 --vm-bytes 3G --vm-keep --timeout 300 &
sleep 15
# top 관찰 (대화식):  M  → 메모리 정렬,  E  → 요약 단위 g,  e  → 태스크 단위 m
top -b -n 2 -d 2 -o %MEM | sed -n '4,5p;7,9p'
```

기대 출력 (부하 중)

```text
GiB Mem :      3.7 total,      0.1 free,      3.4 used,      0.2 buff/cache
GiB Swap:      1.5 total,      0.9 free,      0.6 used.      0.1 avail Mem        ← avail 급감, Swap used 증가
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   2401 root      20   0  3.1g    2.9g    ... R  xx.x  78.x   0:xx.xx stress-ng-vm
```

```bash
# 보조 도구 교차 확인
free -h
vmstat 1 5                                     # si/so 열 > 0 = 스왑 인/아웃 진행, free 급감
cat /proc/$(pgrep -n stress-ng-vm)/oom_score   # 가장 높은 프로세스가 OOM 1순위
for p in $(pgrep stress-ng-vm) 1 $(pgrep -n sshd); do printf "%s %s\n" $p $(cat /proc/$p/oom_score); done
pidstat -r 1 2 -p $(pgrep -n stress-ng-vm)     # RSS, %MEM, 페이지 폴트(minflt/majflt)
sar -r 1 3                                     # kbmemfree kbavail %memused
dmesg -T | grep -iE 'out of memory|killed process' | tail -3     # OOM 발생 시에만 출력
journalctl -k -g -i 'oom|killed process' --no-pager | tail -3
# 조치
kill -15 $(pgrep -n stress-ng)                 # 무응답 시 kill -9
sleep 3; free -h; swapon --show                # avail 회복. Swap used 는 즉시 0 이 되지 않음 (필요 시 swapoff -a && swapon -a)
```

- `--vm N` : 메모리 워커 · `--vm-bytes` : 워커당 할당량 · `--vm-keep` : 해제 없이 계속 점유
- `vmstat` `si`/`so` : 초당 스왑 인/아웃 KiB — 지속적으로 0 이 아니면 메모리 부족 진행 중
- `oom_score` : 커널이 OOM 시 종료 우선순위 산출값(0~1000). `oom_score_adj` 를 `-1000` 으로 쓰면 제외
- `pidstat -r` : 메모리 통계 (**r**esident) · `sar -r` : 메모리 사용률 (**r**am) · `dmesg -T` : 사람이 읽는 시각 (**T**ime) · `journalctl -k -g` : 커널 메시지 grep

**검증**

```bash
pgrep -a stress-ng || echo none
free -m | awk 'NR==2{print "avail MiB="$7}'
vmstat 1 2 | tail -1 | awk '{print "si="$7, "so="$8}'
```

기대 출력

```text
none
avail MiB=3xxx
si=0 so=0
```

> 📝 **시험 포인트**: 스왑 사용 증가 + `si/so` 지속 > 0 = 메모리 부족 신호(R08 #29 에서 si/so 는 스왑, bi/bo 는 블록 I/O). OOM killer 는 `oom_score` 높은 순으로 종료. `free` 의 buff/cache 는 회수 가능 → `available` 로 판단.

### 5-7. 진단 시나리오 (c) — I/O 병목 (iowait · D 상태)

> **상황**: `/data`(ext4, 2 GB) 에 직접 I/O 로 대용량 쓰기를 걸어 `wa` 급등과 D 상태를 관찰하고, `iostat` 으로 장치 포화(%util·await)를 확인한다.

⚠️ `/data` 는 2 GB 이므로 `count=1500`(1.5 GB) 사용. `of=` 경로가 장치가 아닌 **파일**인지 재확인

```bash
findmnt /data; df -h /data
dd if=/dev/zero of=/data/io.test bs=1M count=1500 oflag=direct &
# 대체: stress-ng --hdd 1 --hdd-bytes 1G --temp-path /data --timeout 120 &
sleep 3
top -b -n 2 -d 2 | sed -n '3p'                     # wa 상승
ps -eo pid,stat,wchan:20,cmd | awk '$2 ~ /^D/'    # D 상태 + 커널 대기 함수
```

기대 출력 (부하 중)

```text
%Cpu(s):  1.x us,  8.x sy,  0.0 ni, 4x.x id, 4x.x wa,  0.x hi,  0.x si,  0.0 st     ← wa 급등
    PID STAT WCHAN                CMD
   2501 D    ...                  dd if=/dev/zero of=/data/io.test ...
```

```bash
# 보조 도구 교차 확인
iostat -xz 1 3                       # vdb: w/s wkB/s wareq-sz aqu-sz w_await %util → %util 90~100
vmstat 1 5                           # b(블록 대기 프로세스) ≥ 1, bo(블록 아웃) 수만 KiB/s, wa 상승
pidstat -d 1 3                       # 프로세스별 kB_wr/s → dd
sar -b 1 3                           # tps wtps bwrtn/s 전체 블록 I/O
iotop -o -b -n 2 2>/dev/null | head  # EPEL 설치 시. -o: I/O 있는 것만, -b 배치
lsblk -o NAME,MOUNTPOINTS /dev/vdb   # 병목 장치 → 마운트 대응
# 조치: 완료 대기 (D 상태는 시그널 즉시 반영 안 됨) 또는 ionice 로 재실행
wait %1 2>/dev/null; jobs
ionice -c 3 dd if=/dev/zero of=/data/io.test bs=1M count=300 oflag=direct    # idle 클래스로 재실행 비교
rm -f /data/io.test
```

- `oflag=direct` : 페이지 캐시 우회 → 장치에 직접 쓰기 (실제 디스크 병목 재현)
- `iostat -x` : 확장 통계 (e**x**tended) · `-z` : 활동 없는 장치 생략 (**z**ero 생략) · `-c` : CPU 만 · `-d` : 장치만 · `-h` : 사람이 읽는 단위 · 열: `r/s w/s` IOPS, `rkB/s wkB/s`, `r_await w_await` 평균 응답 ms, `aqu-sz` 큐 길이, `%util` 장치 사용률
- `vmstat` `b` : 인터럽트 불가 대기(D) 프로세스 수 · `bi`/`bo` : 블록 장치 입/출력 (blocks/s) · `wa` : I/O 대기 CPU%
- `pidstat -d` : 프로세스별 디스크 I/O (**d**isk) · `sar -b` : 블록 I/O 요약 (**b**lock) · `iotop -o` : 활성 I/O 만 (**o**nly)
- `ionice -c 3` : I/O 스케줄링 클래스 idle (1 realtime, 2 best-effort, 3 idle) · `-n 0~7` : best-effort 내 우선순위

**검증**

```bash
top -b -n 2 -d 1 | sed -n '3p' | grep -oE '[0-9.]+ wa'
iostat -dz 1 2 | tail -3
ls /data/io.test 2>/dev/null || echo "삭제됨"
```

기대 출력

```text
0.0 wa
Device   tps  kB_read/s  kB_wrtn/s ...
vda      0.xx ...
삭제됨
```

> 📝 **시험 포인트**: `wa` 높음 = 디스크 I/O 병목(R08 #31), D 상태 = `kill -9` 도 즉시 안 통함, `vmstat` `b` 열 = 블록 대기 프로세스(좀비 아님, R08 #29 ②는 오답). `iostat -x` 의 `%util`·`await` 가 장치 포화 지표.

### 5-8. 진단 시나리오 (d) — 좀비·고아 프로세스

> **상황**: 종료됐는데도 `ps` 에 남는 좀비를 만들어 `kill -9` 가 통하지 않음을 증명하고, 부모를 종료해 해소한다. 이어서 부모가 먼저 죽은 고아가 PID 1 에 입양되는 것을 확인한다.

```bash
# 좀비 생성: 자식(sleep 1)은 1초 뒤 종료, 부모는 exec 로 sleep 300 이 되어 wait() 를 절대 호출하지 않음
(sleep 1 & exec sleep 300) &
sleep 2
top -b -n 1 | sed -n '2p'                                        # zombie 1
ps -eo pid,ppid,stat,cmd | awk 'NR==1 || $3 ~ /^Z/'              # Z, CMD 에 <defunct>
Z=$(ps -eo pid,stat | awk '$2 ~ /^Z/{print $1}')
ZP=$(ps -o ppid= -p $Z)
kill -9 $Z; sleep 1
ps -o pid,ppid,stat,cmd -p $Z                                    # 여전히 Z → KILL 무효
# 해소: 부모 종료 → 좀비가 PID 1 로 입양되고 systemd 가 즉시 wait() 로 회수
kill -15 $ZP; sleep 1
ps -p $Z || echo "좀비 소멸"
top -b -n 1 | sed -n '2p'                                        # zombie 0

# 고아 생성: 부모(서브셸)가 자식보다 먼저 종료
bash -c 'sleep 500 & echo "부모=$$ 자식=$!"' 
sleep 1
ps -o pid,ppid,stat,cmd -C sleep | grep 'sleep 500'              # PPID 1 (환경에 따라 systemd --user 인스턴스 PID)
pstree -p -s $(pgrep -f 'sleep 500')                             # systemd(1)───sleep(...)
kill $(pgrep -f 'sleep 500')
```

- `(… & exec …) &` : 서브셸 안에서 자식을 백그라운드로 띄운 뒤 서브셸 자체를 `exec` 로 교체 → 부모 코드가 사라져 `wait()` 불가
- `<defunct>` : `ps` 가 좀비 CMD 뒤에 붙이는 표기 · `pstree -s` : 지정 PID 의 조상 경로 표시 (**s**how parents)
- 좀비는 프로세스 테이블 항목(PID)만 점유 → 메모리·CPU 소비 없음. 대량 발생 시 `pid_max` 고갈이 문제
- 고아는 정상 프로세스 — PID 1(또는 서브리퍼)이 입양해 종료 시 회수

**검증**

```bash
ps -eo stat | grep -c '^Z'
top -b -n 1 | sed -n '2p' | grep -oE '[0-9]+ zombie'
pgrep -f 'sleep 500' || echo none
```

기대 출력

```text
0
0 zombie
none
```

> 📝 **시험 포인트**: 좀비 = 자식이 먼저 종료 + 부모가 `wait()` 미호출, `kill -9` 불가, 부모 종료로 해소 — R07 #18. 고아 = 부모가 먼저 종료 → PPID 1 로 재지정, 정상 동작. `pstree -p` 트리에서 부모-자식 방향 해석 — R08 #33.

### 5-9. 진단 시나리오 (e) — 삭제된 파일 점유·포트 점유 (lsof · fuser)

> **상황**: `df` 는 가득 찼다는데 `du` 는 파일이 없다는 전형적 불일치. 프로세스가 삭제된 큰 파일을 열고 있는 경우로, `lsof +L1` 로 찾아 FD 를 비워 공간을 회수한다. 포트·사용자·PID 기준 `lsof` 조회도 함께 익힌다.

```bash
# 재현
dd if=/dev/zero of=/data/big.log bs=1M count=500 status=none
tail -f /data/big.log > /dev/null &
rm -f /data/big.log
df -h /data                       # Used 에 500M 포함
du -sh /data                      # 파일 없음 → 불일치
# 진단
lsof +L1                          # 링크 수 < 1 (삭제됐지만 열려 있는 파일)
lsof +L1 | awk 'NR>1{print $2, $4, $7, $9}'      # PID FD SIZE NAME
P=$(lsof -t +L1 | head -1); FD=$(lsof +L1 -p $P | awk 'NR>1{print $4}' | tr -dc '0-9' | head -1)
ls -l /proc/$P/fd                 # → '/data/big.log (deleted)'
lsof -p $P                        # 해당 프로세스가 연 파일 전체
# 조치 1: 파일 내용만 비움 (프로세스는 계속 실행, 공간 즉시 회수)
: > /proc/$P/fd/$FD
df -h /data
# 조치 2: 프로세스 종료
kill $P
# 포트·사용자·PID·경로 기준 조회
lsof -i :22                       # 22 포트를 쓰는 프로세스 (sshd + 세션)
lsof -nP -iTCP -sTCP:LISTEN       # 리스닝 소켓 전체 (ss -tlnp 와 대응)
lsof -u dev1 | head               # dev1 이 연 파일
lsof /data | head                 # /data 아래 파일을 연 프로세스 (umount 실패 원인 추적)
fuser -vm /data                   # 마운트 지점 사용 프로세스 (verbose, mount)
fuser -v 22/tcp                   # 포트 사용 프로세스
# fuser -k /data                  # ⚠️ /data 사용 프로세스 전부 KILL — 사용 예만 참고
```

- `lsof +L1` : 링크 수(NLINK) 1 미만 파일 → 삭제됐으나 열린 파일 (**L**ink count) · `-t` : PID 만 (**t**erse) · `-p <PID>` : 프로세스 기준 · `-u <user>` · `-i [:포트]` : 네트워크 소켓 (**i**nternet) · `-nP` : 호스트·포트 이름 해석 생략 · `-sTCP:LISTEN` : 상태 필터
- `lsof <경로>` : 그 파일/디렉터리를 연 프로세스 → `umount: target is busy` 원인 추적
- `: > /proc/PID/fd/N` : FD 가 가리키는(삭제된) 파일을 0 바이트로 truncate → `df` 즉시 회복. 파일 자체는 프로세스 종료 시 완전 해제
- `fuser -v` : 상세 (**v**erbose) · `-m` : 마운트 지점/장치 기준 (**m**ount) · `-k` : 시그널 전송 (**k**ill, 기본 KILL) · `-n tcp` 또는 `포트/tcp` : 네트워크 포트

**검증**

```bash
lsof +L1 | wc -l                   # 0 (헤더 없음)
df -h /data | tail -1; du -sh /data
ss -tlnp | grep ':22 '             # lsof -i :22 와 같은 PID
```

기대 출력

```text
0
/dev/vdb1  2.0G  xxM  1.9G   x% /data
xxM  /data
LISTEN 0 128 0.0.0.0:22 ... users:(("sshd",pid=8xx,fd=3))
```

> 📝 **시험 포인트**: "`df` 와 `du` 불일치 → 삭제된 파일을 잡고 있는 프로세스 → `lsof +L1`" 실무형 문항. 포트 점유 프로세스 확인은 `ss -tlnp`(실기 R01 #5, R03 #8)·`lsof -i :포트`·`fuser -n tcp 포트` 세 가지 모두 답.

---

## 6. 시스템 통계 도구 (sysstat 등)

### 6-1. `vmstat` 열 해석

> **상황**: 5-5~5-7 에서 본 지표를 한 줄로 요약해 주는 `vmstat` 의 모든 열을 평시 값으로 기록해 두고, 요약·디스크 모드도 확인한다.

```bash
vmstat 1 5                 # 1초 간격 5회 (첫 행은 부팅 이후 평균)
vmstat -w 1 3              # 넓은 형식
vmstat -S M 1 2            # 단위 MiB
vmstat -s | head -12       # 이벤트 카운터 요약
vmstat -d                  # 디스크별 읽기/쓰기 통계
vmstat -a 1 2              # active/inactive 메모리
```

| 그룹 | 열 | 의미 | 이상 징후 |
| --- | --- | --- | --- |
| procs | `r` | 실행 대기(R) 프로세스 수 | 지속적으로 > nproc → CPU 부족 |
| procs | `b` | 인터럽트 불가 대기(D) 프로세스 수 | > 0 지속 → I/O 병목 |
| memory | `swpd` `free` `buff` `cache` | 스왑 사용량 / 여유 / 버퍼 / 페이지 캐시 (KiB) | `free` 만 보지 말고 cache 포함 |
| swap | `si` `so` | 스왑 인 / 아웃 (KiB/s) | 지속 > 0 → 메모리 부족 |
| io | `bi` `bo` | 블록 장치 읽기 / 쓰기 (blocks/s) | 급등 → 디스크 부하 |
| system | `in` `cs` | 초당 인터럽트 / 컨텍스트 스위치 | `cs` 폭증 → 스레드 경쟁·락 |
| cpu | `us` `sy` `id` `wa` `st` | 사용자 / 커널 / 유휴 / I/O 대기 / steal (%) | top 3행과 동일 |

- `vmstat <간격> <횟수>` · `-w` : 넓게 (**w**ide) · `-S <k|K|m|M>` : 단위 (**S**ize unit) · `-s` : 통계 요약 (**s**tats) · `-d` : 디스크 (**d**isk) · `-a` : active/inactive (**a**ctive)

**검증**

```bash
vmstat 1 2 | tail -1 | awk '{print "r="$1,"b="$2,"si="$7,"so="$8,"wa="$16}'
```

기대 출력

```text
r=0 b=0 si=0 so=0 wa=0
```

> 📝 **시험 포인트**: R08 #29 — `r` 은 실행/대기 프로세스 수(정답), `b` 는 좀비 아님, `si/so` 는 스왑, `wa` 는 I/O 대기. `vmstat 1 1` 첫 행은 부팅 이후 평균이라 실시간 값이 아님.

### 6-2. `iostat` · `mpstat` · `pidstat`

> **상황**: 장치·코어·프로세스 단위로 쪼개 보는 sysstat 세 도구의 옵션을 평시에 한 번씩 실행해 출력 형식을 익힌다.

```bash
iostat                          # 부팅 이후 평균 (CPU + 장치)
iostat -c 1 3                   # CPU 만
iostat -d 1 3                   # 장치만
iostat -x -h 2 3                # 확장 + 사람이 읽는 단위
iostat -xz -p vdb 1 2           # 파티션 포함 (vdb1 vdb2 vdb3)
mpstat                          # 전체 CPU 평균
mpstat -P ALL 1 3               # 코어별 (%usr %nice %sys %iowait %irq %soft %steal %idle)
mpstat -P 0 1 2                 # 0번 코어만
pidstat 1 3                     # 전체 프로세스 CPU (활동 있는 것만)
pidstat -u -r -d 1 3 -p $(pgrep -n sshd)     # 특정 PID 의 CPU·메모리·디스크
pidstat -t -p $(pgrep -n sshd) 1 1           # 스레드 단위
pidstat -w 1 2                               # 컨텍스트 스위치 (cswch/s nvcswch/s)
```

- `iostat -c` : CPU (**c**pu) · `-d` : 장치 (**d**evice) · `-x` : 확장 · `-h` : 단위 · `-z` : 유휴 장치 생략 · `-p <dev|ALL>` : 파티션 (**p**artition)
- `mpstat -P <n|ALL>` : 코어 지정 (**P**rocessor) · 열 `%iowait` `%steal` 은 top 의 `wa` `st`
- `pidstat -u` : CPU · `-r` : 메모리(minflt/s majflt/s VSZ RSS %MEM) · `-d` : 디스크(kB_rd/s kB_wr/s) · `-t` : 스레드 (**t**hreads) · `-w` : 컨텍스트 스위치 (s**w**itch) · `-p <PID|ALL>`

**검증**

```bash
mpstat -P ALL 1 1 | awk '/^Average/ && $2 ~ /^[0-9]/{print "cpu"$2, "idle="$NF}'
iostat -d -z 1 1 | grep -E '^vd'
```

기대 출력

```text
cpu0 idle=9x.xx
cpu1 idle=9x.xx
vda   ...
```

> 📝 **시험 포인트**: `iostat` 장치 I/O, `mpstat` 코어별 CPU, `pidstat` 프로세스별 — 도구·대상 매핑. 모두 `sysstat` 패키지.

### 6-3. `sar` — 수집 활성화와 과거 데이터 조회

> **상황**: "어젯밤 몇 시에 느렸나" 를 답하려면 과거 통계가 있어야 한다. `sysstat` 수집 타이머를 켜고 오늘 파일을 만들어 시간 범위 조회까지 해 본다.

```bash
systemctl enable --now sysstat                   # sysstat.service + sysstat-collect.timer(10분) + sysstat-summary.timer(일)
systemctl list-timers 'sysstat*' --no-pager
cat /etc/sysconfig/sysstat                        # HISTORY=28 (보존 일수), COMPRESSAFTER, SADC_OPTIONS
grep -v '^#' /etc/cron.d/sysstat 2>/dev/null      # RHEL 9 는 timer 방식 → 파일 없음
/usr/lib64/sa/sa1 1 1                             # 즉시 1회 수집 → 오늘 파일 생성
ls -l /var/log/sa/                                # saDD (바이너리), sarDD (텍스트, sa2 가 생성)
sar -u 1 3                                        # CPU (%user %nice %system %iowait %steal %idle)
sar -r 1 3                                        # 메모리 (kbmemfree kbavail kbmemused %memused kbbuffers kbcached)
sar -S 1 2                                        # 스왑 (kbswpfree kbswpused %swpused)
sar -b 1 3                                        # 블록 I/O (tps rtps wtps bread/s bwrtn/s)
sar -d -p 1 2                                     # 장치별 (-p: 장치명 vd* 로 표시)
sar -n DEV 1 3                                    # 네트워크 인터페이스 (rxpck/s txpck/s rxkB/s txkB/s)
sar -q 1 3                                        # 실행 큐·load average (runq-sz plist-sz ldavg-1 ldavg-5 ldavg-15 blocked)
sar -w 1 2                                        # 컨텍스트 스위치·프로세스 생성
sar -f /var/log/sa/sa$(date +%d)                  # 오늘 파일 (기본 CPU)
sar -f /var/log/sa/sa$(date +%d) -s 09:00:00 -e 12:00:00 -u   # 시간 범위
sar -A -f /var/log/sa/sa$(date +%d) | head -30    # 전체 항목
sadf -d /var/log/sa/sa$(date +%d) -- -u | head -5 # CSV 형식 (; 구분)
sadf -j /var/log/sa/sa$(date +%d) -- -r | head -c 300; echo   # JSON
```

- `sar -u` : CPU (**u**tilization) · `-r` : 메모리 · `-S` : 스왑 (**S**wap) · `-b` : 블록 I/O · `-d` : 장치별 · `-p` : 장치명 표시 (**p**retty) · `-n DEV|SOCK|TCP` : 네트워크 · `-q` : 큐·load (**q**ueue) · `-w` : 스위치 · `-A` : 전체 (**A**ll)
- `-f <파일>` : 저장 파일 읽기 (**f**ile) · `-s HH:MM:SS` / `-e HH:MM:SS` : 시작/끝 시각 (**s**tart / **e**nd) · `-o <파일>` : 저장
- `sa1` : `sadc` 호출 스크립트(수집) · `sa2` : 일일 텍스트 리포트 `sarDD` 생성 · `sadf -d` : CSV (**d**atabase) · `-j` : JSON · `--` 뒤는 sar 옵션
- `/etc/sysconfig/sysstat` `HISTORY` : 보존 일수 (28 초과 시 월별 디렉터리)

**검증**

```bash
systemctl is-active sysstat-collect.timer
ls /var/log/sa/sa$(date +%d) && sar -q -f /var/log/sa/sa$(date +%d) | tail -3
```

기대 출력

```text
active
/var/log/sa/sa03
Average:  runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15   blocked
Average:        0       1xx      0.xx      0.xx      0.xx         0
```

> 📝 **시험 포인트**: `sar` 는 **과거** 통계 조회가 핵심(`-f /var/log/sa/saDD`, `-s/-e`), 수집은 `sa1`/`sadc`, 데이터 위치 `/var/log/sa/`. `sar -u` CPU, `-r` 메모리, `-b` I/O, `-n DEV` 네트워크, `-q` load.

### 6-4. `free` · `uptime` · `/proc/stat` · `/proc/meminfo` · `ulimit`

> **상황**: 도구가 없을 때도 `/proc` 원본으로 같은 값을 읽을 수 있음을 확인하고, 프로세스 한도(`ulimit`)를 점검한다.

```bash
free -h -s 2 -c 3              # 2초 간격 3회
free -w -m                     # buffers 와 cache 분리 (wide)
uptime; uptime -p; uptime -s   # pretty / since
head -1 /proc/stat             # cpu user nice system idle iowait irq softirq steal (jiffies)
grep -E 'ctxt|procs_running|procs_blocked' /proc/stat
grep -E 'MemTotal|MemFree|MemAvailable|Buffers|^Cached|SwapTotal|SwapFree' /proc/meminfo
ulimit -a                      # 현재 셸 한도 (open files, max user processes, ...)
ulimit -u; ulimit -n           # 프로세스 수 / 파일 열기 한도
su - dev1 -c 'ulimit -u'       # /etc/security/limits.conf nproc 반영값 → [[03-user-group-permission]]
```

- `free -s <초>` : 반복 (**s**econds) · `-c <횟수>` : 횟수 (**c**ount) · `-w` : buffers/cache 분리 (**w**ide) · `-h/-m/-g` : 단위
- `uptime -p` : "up 3 days, 2 hours" 형식 (**p**retty) · `-s` : 부팅 시각 (**s**ince)
- `/proc/stat` `cpu` 행 : 부팅 이후 jiffies 누적 → `top`·`vmstat` 이 두 시점 차로 % 계산 · `procs_blocked` = vmstat `b`
- `/proc/meminfo` `MemAvailable` : `free` 의 available · `ulimit -u` : 사용자당 프로세스 수 (**u**ser processes) · `-n` : 열 수 있는 파일 수 (**n**umber of files) · `-a` : 전체

**검증**

```bash
awk '/MemAvailable/{printf "%d MiB\n", $2/1024}' /proc/meminfo; free -m | awk 'NR==2{print $7" MiB"}'
```

기대 출력

```text
3xxx MiB
3xxx MiB      ← 동일
```

> 📝 **시험 포인트**: "특정 계정 fork 남용 예방 → 프로세스 수 제한 파일" = `/etc/security/limits.conf` (`nproc`) — R06 #65. `ulimit -u` 로 확인. `/proc/meminfo`·`/proc/stat` 이 `free`·`top` 의 원천.

### 6-5. `htop` 조작 · 기타 도구 참고

> **상황**: 대화형 진단에 익숙해지도록 `htop` 의 기능키를 실제로 눌러 보고, 시험 범위의 나머지 도구는 용도만 확인한다.

```bash
htop                          # EPEL
# F1 도움  F2 설정(열·미터 편집)  F3 검색(/)  F4 필터(\)  F5 트리(t)  F6 정렬(>)  F7 nice-  F8 nice+  F9 kill(k)  F10 종료(q)
# u 사용자 선택  H 사용자 스레드 토글  K 커널 스레드 토글  Space 태그  U 태그 해제  c 태그(자식 포함)  l lsof  s strace  e 환경변수
htop -u dev1 -d 20 -s PERCENT_MEM   # 사용자·주기(1/10초)·정렬 열
htop -t                        # 트리 모드로 시작
dstat 1 3                      # Rocky 9: pcp-system-tools 의 dstat (dnf install pcp-system-tools) ※ 선택
strace -p $(pgrep -n sshd) -c -f -e trace=network 2>&1 | head   # 시스템 콜 추적 (strace 패키지) ※ 참고
# ltrace <cmd>                 # 라이브러리 호출 추적 (ltrace 패키지) ※ 참고
# perf top / perf record -g    # 커널·응용 프로파일링 (perf 패키지) ※ 참고
```

- `htop -u` : 사용자 · `-d` : 갱신 주기(1/10초 단위) (**d**elay) · `-s <열>` : 정렬 열 (**s**ort, `htop --sort-key help`) · `-t` : 트리 (**t**ree)
- `F7`/`F8` : 선택 프로세스 NI -1/+1 (renice) · `F9` : 시그널 목록에서 선택 후 전송 · `l` : 선택 프로세스에 `lsof` · `s` : `strace` 연결
- `dstat` : vmstat+iostat+ifstat 통합, RHEL 9 는 PCP 구현 · `strace -c` : 시스템 콜 통계 요약 (**c**ount) · `-f` : 자식 포함 (**f**ollow) · `-e trace=` : 종류 필터

**검증**

```bash
htop --version | head -1
rpm -q strace pcp-system-tools 2>&1 | head -2
```

기대 출력

```text
htop 3.x.x
strace-...  (또는 not installed)
```

> 📝 **시험 포인트**: `htop` 은 `top` 의 컬러·마우스·기능키 대체판(F9 kill, F7/F8 nice). `strace` 는 시스템 콜, `ltrace` 는 라이브러리 호출 추적 — 이름 대응만 기억.

---

## 7. 스케줄링 — at · cron · systemd timer

### 7-1. `at` — 1회성 예약

> **상황**: 방금 진단한 결과를 5분 뒤 자동으로 리포트 파일로 남기고, 내일 새벽 1회 실행도 예약한다. `sysreport.sh` 는 Part 04 에서 작성됨(없으면 아래 최소 버전으로 생성).

```bash
systemctl enable --now atd; systemctl is-active atd
test -x /usr/local/bin/sysreport.sh || cat > /usr/local/bin/sysreport.sh <<'EOF'
#!/bin/bash
# 시스템 진단 요약 (LAB 04 정본. 미존재 시 최소 버전)
OUT=/var/log/sysreport-$(date +%F_%H%M).log
{ echo "== $(date) =="; uptime; top -b -n 1 | head -12; echo; vmstat 1 2 | tail -1; echo; df -hT -x tmpfs -x devtmpfs; } > "$OUT"
EOF
chmod +x /usr/local/bin/sysreport.sh

at now + 5 minutes <<'EOF'
/usr/local/bin/sysreport.sh
EOF
at 03:30 tomorrow <<'EOF'
/usr/local/bin/sysreport.sh
EOF
echo "/usr/local/bin/sysreport.sh" | at 23:00           # 파이프 입력
at -f /usr/local/bin/sysreport.sh now + 1 hour          # 파일에서 읽기
atq                                                     # = at -l
at -c $(atq | awk 'NR==1{print $1}') | tail -5          # 예약 작업 내용(환경변수 + 명령)
atrm $(atq | awk '$5 ~ /23:00/{print $1}')              # 23:00 건 삭제 (= at -d)
echo "/usr/local/bin/sysreport.sh" | batch              # 부하가 임계치 아래일 때 실행
ls -l /var/spool/at/                                    # 작업 파일 a000…, 시퀀스 .SEQ
```

- `at <시각>` : 표준입력의 명령을 지정 시각 1회 실행. 시각 표기: `now + 5 minutes|hours|days|weeks`, `HH:MM [tomorrow|today]`, `noon`, `midnight`, `teatime`(16:00), `MM/DD/YY`
- `-l` : 목록 (**l**ist, = `atq`) · `-c <번호>` : 작업 내용 출력 (**c**at) · `-d`/`-r` : 삭제 (**d**elete, = `atrm`) · `-f <파일>` : 명령을 파일에서 (**f**ile) · `-m` : 출력 없어도 메일 (**m**ail) · `-q <큐>` : 큐 문자(a~z, 기본 a; batch 는 b)
- `batch` : 시스템 부하가 `atd -l` 임계치(man atd 참조) 아래로 내려갈 때 실행
- `/var/spool/at/` : 작업 스크립트 저장, 실행 시 환경변수·umask·작업 디렉터리를 재현

**검증**

```bash
atq
systemctl is-active atd; journalctl -u atd -n 2 --no-pager
# 5분 후:
ls -lt /var/log/sysreport-* | head -2
```

기대 출력

```text
3       Fri Sep  4 03:30:00 2026 a root
1       Thu Sep  3 ...:..:00 2026 a root
4       Thu Sep  3 ...:..:00 2026 a root
5       Thu Sep  3 ...:..:00 2026 b root       ← batch 큐 b
active
-rw-r--r--. 1 root root ... /var/log/sysreport-2026-09-03_....log
```

> 📝 **시험 포인트**: `at` = 1회, cron = 반복. 등록은 `at 시각` 후 명령 입력 `Ctrl+D`, 조회 `atq`/`at -l`, 삭제 `atrm 번호`/`at -d`, 데몬 `atd`.

### 7-2. `/etc/at.allow` · `/etc/at.deny` — dev1 거부 실습

> **상황**: 보안 정책상 개발자 `dev1` 은 `at` 예약을 못 하게 막는다. allow/deny 우선 규칙을 실제로 검증한다.

```bash
ls -l /etc/at.allow /etc/at.deny          # 기본: at.deny 만 존재(빈 파일)
su - dev1 -c 'echo id | at now + 1 minute'      # 허용됨 (deny 에 없음)
echo dev1 >> /etc/at.deny
su - dev1 -c 'echo id | at now + 1 minute'      # 거부
sed -i '/^dev1$/d' /etc/at.deny                 # 원복
echo admin1 > /etc/at.allow                     # allow 가 존재하면 allow 만 유효
su - dev1 -c 'atq'                              # 거부
rm -f /etc/at.allow                             # 원복 (deny 만 남김)
```

| `at.allow` | `at.deny` | 결과 |
| --- | --- | --- |
| 존재 | 무관 | **allow 에 있는 사용자만** 허용 (deny 무시) |
| 없음 | 존재 | deny 에 없는 모두 허용 |
| 없음 | 없음 | root 만 허용 |

- root 는 항상 허용. cron 의 `/etc/cron.allow`·`/etc/cron.deny` 도 동일 규칙(7-5)

**검증**

```bash
su - dev1 -c 'echo id | at now + 1 minute' && echo "dev1 허용"
atq | grep dev1 | awk '{print $1}' | xargs -r atrm
```

기대 출력

```text
warning: commands will be executed using /bin/sh
job 6 at Thu Sep  3 ...
dev1 허용
```

> 📝 **시험 포인트**: "`allow` 파일이 있으면 `allow` 만 본다, 둘 다 없으면 root 만" — R06 #25(cron.allow 에 user1 만 등록)와 같은 논리. 거부 메시지 "You do not have permission to use at."

### 7-3. 사용자 crontab — `dev1` 등록·조회·삭제

> **상황**: `dev1` 이 10분마다 자기 홈 사용량을 기록하는 작업을 등록한다. 관리자는 `-u` 로 타 사용자 crontab 을 확인·관리한다.

```bash
su - dev1
EDITOR=vi crontab -e                     # 편집기: 아래 한 줄 입력 후 :wq
```

```text
*/10 * * * * du -sh $HOME >> $HOME/du.log 2>&1
```

```bash
crontab -l                               # 내 crontab
exit
crontab -u dev1 -l                       # root 가 dev1 crontab 조회
cat /var/spool/cron/dev1                 # 실제 저장 파일 (root 만 접근)
ls -l /var/spool/cron/
crontab -u dev1 -e                       # root 가 dev1 것 편집 (환경변수 PATH 추가 예)
```

⚠️ `crontab -r` 은 **확인 없이 전체 삭제** → `-i` 병용 또는 `crontab -l > backup` 선행

```bash
crontab -u dev1 -l > /root/dev1-crontab.bak
crontab -u dev1 -i -r                    # 'y' 입력 시 삭제
crontab -u dev1 -l                       # "no crontab for dev1"
crontab -u dev1 /root/dev1-crontab.bak   # 파일로 일괄 등록 (복원)
```

- `crontab -e` : 편집 (**e**dit, `$EDITOR`·`$VISUAL` 사용) · `-l` : 조회 (**l**ist) · `-r` : 삭제 (**r**emove) · `-i` : 삭제 전 확인 (**i**nteractive) · `-u <user>` : 대상 사용자 (root 전용) · `crontab <파일>` : 파일 내용으로 교체
- 저장 위치 `/var/spool/cron/<사용자>` — 직접 편집 금지(crond 가 변경 감지 못 할 수 있음), 반드시 `crontab` 명령 사용
- `%` 는 crontab 에서 개행 의미 → 명령 안에서는 `\%` (`date +\%F`)
- cron 의 기본 `PATH=/usr/bin:/bin`, `SHELL=/bin/sh` → 절대경로 사용 또는 crontab 상단에 `PATH=` 선언

**검증**

```bash
crontab -u dev1 -l
ls -l /var/spool/cron/dev1
grep -i 'dev1' /var/log/cron | tail -2     # REPLACE / LIST 기록
```

기대 출력

```text
*/10 * * * * du -sh $HOME >> $HOME/du.log 2>&1
-rw-------. 1 dev1 dev1 ... /var/spool/cron/dev1
Sep  3 ... crontab[....]: (dev1) REPLACE (dev1)
Sep  3 ... crontab[....]: (root) LIST (dev1)
```

> 📝 **시험 포인트**: `crontab -e/-l/-r/-u` 의미 — R04 #40. `-r` 은 확인 없이 삭제(오답 유도). 사용자 crontab 은 **5 필드 + 명령**, 저장 위치 `/var/spool/cron/`.

### 7-4. cron 시간 필드 해석 · `/etc/crontab` · `/etc/cron.d/lab-backup`

> **상황**: 기출 시간 필드를 전부 해석한 뒤, 시스템 백업 작업을 `/etc/cron.d/` 조각 파일로 등록한다(7 필드: 사용자 포함). `backup.sh` 는 [[12-backup-recovery-review]] 에서 작성 — 여기서는 등록만.

```bash
cat /etc/crontab            # SHELL PATH MAILTO 와 7필드 형식 주석
cat > /etc/cron.d/lab-backup <<'EOF'
# 분 시 일 월 요일 사용자 명령
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
MAILTO=root
30 3 * * * root /usr/local/bin/backup.sh >> /var/log/lab-backup.log 2>&1
EOF
chmod 644 /etc/cron.d/lab-backup
ls -l /etc/cron.d/
```

| 항목 | 의미 |
| --- | --- |
| `30 3 * * *` | 매일 03:30 (실기 R01 #10) |
| `0 0 * * 0` | 매주 일요일 00:00 (0·7 = 일요일, 실기 R02 #10) |
| `0 0 1 * *` | 매월 1일 00:00 (실기 R04 #10) |
| `*/10 * * * *` | 10분마다 (실기 R03 #9) |
| `0 9-18 * * 1-5` | 평일 09~18시 정각마다 |
| `0 2 1,15 * *` | 매월 1일·15일 02:00 |
| `30 2 * * 0` | 매주 일요일 02:30 (필기 R07 #30) |
| `30 4 * * 1` | 매주 월요일 04:30 (필기 R08 #35) |
| `*/20 1-3 * * 0` | 일요일 01·02·03시에 20분마다 → 3×3 = 9회/주 (필기 R10 #34) |
| `0 2 * * 1 root /usr/local/bin/backup.sh` | `/etc/crontab` 형식: 매주 월요일 02:00 **root 로** 실행 (실기 R01 #11) |
| `@reboot` `@hourly` `@daily`(=`@midnight`) `@weekly` `@monthly` `@yearly` | 니모닉. `@reboot` 는 crond 시작 시 1회 |

- 필드 순서: **분(0-59) 시(0-23) 일(1-31) 월(1-12) 요일(0-7)** [사용자(시스템 파일만)] 명령
- 특수문자: `*` 전체 · `,` 나열 · `-` 범위 · `/` 간격 · 일과 요일 둘 다 지정 시 **OR** 로 동작
- `/etc/crontab`·`/etc/cron.d/*` : 사용자 필드 포함 7 필드, root 소유 644, 확장자 없는 파일명만 인식(`.rpmsave` 등 무시)
- `MAILTO=` : 출력을 메일로. 빈 값(`MAILTO=""`) 이면 메일 없음. MTA(Postfix, Part 09) 미설치 시 로그에 "No MTA installed, discarding output"

**검증**

```bash
cat /etc/cron.d/lab-backup | grep -v '^#'
systemctl is-active crond
journalctl -u crond -n 3 --no-pager           # "(CRON) INFO (RANDOM_DELAY…" 또는 재로드 메시지
```

기대 출력

```text
SHELL=/bin/bash
PATH=...
MAILTO=root
30 3 * * * root /usr/local/bin/backup.sh >> /var/log/lab-backup.log 2>&1
active
```

> 📝 **시험 포인트**: `/etc/crontab` 은 사용자 필드가 하나 더 있는 7 필드 — R05 #39, 실기 R01·R03·R04 #11. 요일 0 과 7 모두 일요일. `*/20 1-3 * * 0` 횟수 계산 — R10 #34.

### 7-5. `cron.{hourly,daily,weekly,monthly}` · anacron · allow/deny · 로그

> **상황**: 패키지가 놓는 주기 스크립트가 어떤 경로로 실행되는지(`run-parts` → anacron)를 추적하고, 사용자 제한과 실행 로그 확인 방법을 익힌다.

```bash
cat /etc/cron.d/0hourly                  # 01 * * * * root run-parts /etc/cron.hourly
ls -l /etc/cron.hourly/                  # 0anacron
cat /etc/cron.hourly/0anacron            # → anacron -s 실행 (daily/weekly/monthly 는 anacron 이 담당)
cat /etc/anacrontab                      # 주기(일) 지연(분) 작업ID 명령  ← cron.daily 1 5, weekly 7 25, monthly @monthly 45
ls /etc/cron.daily /etc/cron.weekly /etc/cron.monthly
cat /var/spool/anacron/cron.daily        # 마지막 실행 날짜
run-parts --test /etc/cron.daily         # 실행 대상 스크립트 목록만 (실행 안 함)
anacron -T                               # anacrontab 문법 검사
# 사용자 제한
ls -l /etc/cron.allow /etc/cron.deny     # 기본: cron.deny 만 (빈 파일)
echo dev1 > /etc/cron.deny
su - dev1 -c 'crontab -l'                # "You (dev1) are not allowed to use this program (crontab)"
: > /etc/cron.deny                       # 원복
# 로그
tail -5 /var/log/cron                    # rsyslog cron.* → /var/log/cron
journalctl -u crond --since '10 min ago' --no-pager
```

- `run-parts <디렉터리>` : 디렉터리 안 실행 파일을 이름순으로 모두 실행 · `--test` : 목록만
- `anacron` : 꺼져 있던 동안 놓친 daily/weekly/monthly 를 부팅 후 보정 실행. `/etc/anacrontab` 필드 = 주기(일) 지연(분) 식별자 명령, `RANDOM_DELAY`·`START_HOURS_RANGE` 변수
- `/var/spool/anacron/<작업ID>` : 마지막 실행 날짜 기록 → 하루 1회 보장
- `/etc/cron.allow` 우선 → 없으면 `/etc/cron.deny` → 둘 다 없으면(RHEL 계열) 모두 허용은 배포판 정책 따라 다름 → 시험은 "root 만" 으로 출제
- `/var/log/cron` : `crond[PID]: (user) CMD (명령)` 실행 기록, `crontab[PID]: (user) REPLACE/LIST/DELETE`

**검증**

```bash
grep -E 'cron.daily|cron.weekly|cron.monthly' /etc/anacrontab
grep -c CMD /var/log/cron
su - dev1 -c 'crontab -l' 2>&1 | head -1        # 원복 후 정상 (또는 "no crontab for dev1")
```

기대 출력

```text
1	5	cron.daily		nice run-parts /etc/cron.daily
7	25	cron.weekly		nice run-parts /etc/cron.weekly
@monthly 45	cron.monthly		nice run-parts /etc/cron.monthly
1x
no crontab for dev1
```

> 📝 **시험 포인트**: anacron 은 "미가동 시간에 놓친 작업 보정", 설정 `/etc/anacrontab`(주기 지연 ID 명령). `cron.allow` 에 user1 만 → 나머지 금지 — R06 #25. cron 로그 = `/var/log/cron` — R03 #58.

### 7-6. 즉시 검증 — 1분 뒤 실행 임시 항목 등록 → 로그 → 삭제

> **상황**: 등록한 cron 이 실제로 도는지 하루를 기다릴 수 없으므로, 다음 분에 실행되는 임시 항목으로 crond 동작·환경변수·`%` 이스케이프·메일 처리를 한 번에 검증한다.

```bash
M=$(date -d '+1 min' +%M); H=$(date -d '+1 min' +%H)
cat > /etc/cron.d/lab-test <<EOF
MAILTO=root
$M $H * * * root echo "cron ok \$(date +\\%F_\\%T) PATH=\$PATH" >> /tmp/cron.test; env | grep -c . >> /tmp/cron.test
$M $H * * * root /usr/local/bin/sysreport.sh
EOF
cat /etc/cron.d/lab-test
sleep 70
cat /tmp/cron.test                                 # cron ok 2026-09-03_HH:MM:01 PATH=/usr/bin:/bin  + 환경변수 개수(적음)
grep 'lab-test\|sysreport\|cron ok' /var/log/cron | tail -3
grep -i 'MTA\|mail' /var/log/cron | tail -2        # Postfix 미설치 시 "No MTA installed, discarding output"
ls /var/spool/mail/root 2>/dev/null && mail -H     # Postfix(Part 09) 이후: cron 출력 메일 확인 (s-nail)
rm -f /etc/cron.d/lab-test /tmp/cron.test           # 임시 항목 삭제
```

- `date -d '+1 min'` : 상대 시각 계산 (**d**ate string)
- 히어독 안 `\$` : 셸이 아닌 cron 실행 시 확장, `\\%` → 파일에 `\%` 로 기록 → cron 이 `%` 를 개행으로 해석하는 것 방지
- cron 환경은 로그인 셸과 다름(PATH 짧음, 환경변수 거의 없음) → 스크립트는 절대경로·필요 변수 자체 선언
- `mail -H` : 헤더 목록 (**H**eaders, s-nail) · `/var/spool/mail/root` : root 로컬 메일함

**검증**

```bash
ls /etc/cron.d/lab-test 2>/dev/null || echo "임시 항목 삭제됨"
ls -lt /var/log/sysreport-* | head -1
```

기대 출력

```text
임시 항목 삭제됨
-rw-r--r--. 1 root root ... /var/log/sysreport-2026-09-03_HHMM.log
```

> 📝 **시험 포인트**: cron 실행 기록 확인 = `/var/log/cron` 또는 `journalctl -u crond`. crontab 안 `%` 는 개행 → `\%` 이스케이프. 출력은 MAILTO 사용자에게 메일.

### 7-7. systemd timer — `lab-sysreport.timer`

> **상황**: cron 대신 systemd timer 로 매일 04:00 진단 리포트를 예약한다. `Persistent=true` 로 꺼져 있던 시간의 실행도 보정(anacron 역할)하고, 즉시 실행·저널 확인까지 한다.

```bash
cat > /etc/systemd/system/lab-sysreport.service <<'EOF'
[Unit]
Description=LAB daily system report (sysreport.sh)

[Service]
Type=oneshot
ExecStart=/usr/local/bin/sysreport.sh
Nice=10
IOSchedulingClass=idle
EOF

cat > /etc/systemd/system/lab-sysreport.timer <<'EOF'
[Unit]
Description=Run lab-sysreport daily at 04:00

[Timer]
OnCalendar=*-*-* 04:00:00
Persistent=true
RandomizedDelaySec=300
Unit=lab-sysreport.service

[Install]
WantedBy=timers.target
EOF

systemd-analyze calendar '*-*-* 04:00:00'          # 다음 실행 시각 검증
systemd-analyze verify /etc/systemd/system/lab-sysreport.{service,timer}
systemctl daemon-reload
systemctl enable --now lab-sysreport.timer
systemctl list-timers --all --no-pager | grep -E 'NEXT|lab-sysreport'
systemctl start lab-sysreport.service              # 즉시 1회 실행
journalctl -u lab-sysreport.service -n 5 --no-pager
ls -lt /var/log/sysreport-* | head -1
```

| 항목 | cron | systemd timer |
| --- | --- | --- |
| 정의 | 한 줄 (crontab) | `.timer` + `.service` 2개 유닛 |
| 놓친 실행 보정 | anacron 별도 | `Persistent=true` |
| 로그 | `/var/log/cron` + 메일 | `journalctl -u <svc>` 에 표준출력·오류 자동 기록 |
| 의존성·순서 | 없음 | `After=`, `Requires=` 로 네트워크·마운트 대기 가능 |
| 자원 제어 | `nice` 직접 | `Nice=`, `IOSchedulingClass=`, `CPUQuota=`, `MemoryMax=` |
| 시각 표현 | 5 필드 | `OnCalendar=`(달력) / `OnBootSec=`·`OnUnitActiveSec=`(단조) |
| 즉시 테스트 | 시각 조작 필요 | `systemctl start <svc>` |
| 무작위 지연 | 없음 | `RandomizedDelaySec=` |

- `Type=oneshot` : 실행 후 종료되는 작업 · `Nice=`/`IOSchedulingClass=` : 4절 nice·ionice 의 유닛 버전
- `OnCalendar=` : `요일 년-월-일 시:분:초`, `*` 와일드카드, `daily`(=`*-*-* 00:00:00`) `weekly` `hourly` 별칭 · `Persistent=true` : 마지막 실행 시각을 `/var/lib/systemd/timers/` 에 기록해 놓친 실행 보정 · `RandomizedDelaySec=` : 동시 실행 분산 · `Unit=` : 생략 시 같은 이름 `.service`
- `systemd-analyze calendar` : 달력식 검증·다음 실행 시각 · `verify` : 유닛 문법 검사 · `list-timers --all` : 비활성 포함

**검증**

```bash
systemctl is-enabled lab-sysreport.timer; systemctl is-active lab-sysreport.timer
systemctl list-timers lab-sysreport.timer --no-pager
systemctl show lab-sysreport.service -p Result -p ExecMainStatus
```

기대 출력

```text
enabled
active
NEXT                        LEFT     LAST                        PASSED  UNIT                 ACTIVATES
Fri 2026-09-04 04:0x:xx KST 1xh left ...                                 lab-sysreport.timer  lab-sysreport.service
Result=success
ExecMainStatus=0
```

> 📝 **시험 포인트**: timer 는 `.timer`(언제) + `.service`(무엇) 쌍, 활성화는 **`.timer` 를 enable**. `OnCalendar` 달력식 vs `OnBootSec` 단조식, `Persistent=true` = anacron 대체. 조회 `systemctl list-timers`. 유닛 작성 문법은 [[07-boot-systemd-log]].

---

## 8. 정리

### 8-1. 부하 프로세스·임시 파일 정리, 평시 값 기록

> **상황**: 다음 파트(부팅·systemd·로그)로 넘어가기 전 이 파트에서 만든 부하·임시 파일·예약을 모두 걷어내고, 평시 기준값을 기록해 둔다.

```bash
pgrep -a stress-ng; pkill stress-ng; pkill -f 'sleep [0-9]{3}'; pkill -f trapdemo.sh
pgrep -a 'stress|sleep|dd|tail' || echo "부하 없음"
rm -f /data/io.test /data/big.log /tmp/trapdemo.* /tmp/top.*.txt
atq; atq | awk '{print $1}' | xargs -r atrm; atq            # 남은 at 작업 정리 (03:30 건 포함)
crontab -u dev1 -l                                          # dev1 10분 작업은 유지
ls /etc/cron.d/                                             # lab-backup 유지, lab-test 없음
systemctl list-timers --no-pager | grep -E 'lab-|sysstat'
top -b -n 2 -d 1 | tail -n +$(( $(top -b -n 1 | wc -l) + 1 )) | head -5 | tee /root/top-baseline-$(date +%F).txt
free -h; uptime; swapon --show; df -h /data
```

- `pkill -f 'sleep [0-9]{3}'` : 이 파트에서 띄운 3자리 초 sleep 만 (ERE 지원)
- `tee` : 화면 출력과 파일 저장 동시 → 다음 장애 시 비교 기준(baseline)

**검증**

```bash
top -b -n 1 | sed -n '2,3p'
lsof +L1 | wc -l; ps -eo stat | grep -c '^Z'
```

기대 출력

```text
Tasks: 1xx total,   1 running, 1xx sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.x us,  0.x sy,  0.0 ni, 9x.x id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
0
0
```

> 📝 **시험 포인트**: 장애 대응 마무리 = "원인 프로세스 종료 확인(`pgrep`) → 자원 회복 확인(`top`/`free`/`df`) → 재발 방지(스케줄·모니터링 등록)". 평시 기준값이 있어야 "느려졌다" 를 수치로 판단 가능.

---

## 체크리스트

| 수행 항목 | 명령 | 확인 방법 | ☐ |
| --- | --- | --- | --- |
| 도구 설치 (stress-ng, sysstat, psmisc, lsof, tmux, at, cronie) | `dnf install` | `rpm -q …`, `which sar pstree` | ☐ |
| tmux 세션 생성·분리·재접속 | `tmux new -s lab` / `Ctrl+b d` / `tmux attach` | `tmux ls` | ☐ |
| `ps aux` vs `ps -ef` 열 차이 확인 | `ps aux`, `ps -ef`, `ps -eo … --sort=-%cpu` | 헤더 비교 | ☐ |
| STAT 코드 R/S/D/T/Z/I + s l < N + 대응 | `ps -eo pid,stat,cmd` | `kill -STOP` 후 T 확인 | ☐ |
| `pstree -p -u`, `pgrep -f -l -a -u -x`, `pidof` | 각 명령 | PID 일치 | ☐ |
| `/proc/PID/` status·cmdline·fd·environ·limits·oom_score 열람 | `cat /proc/$$/…` | `ps` 값과 대조 | ☐ |
| `$$ $PPID`, `exec` 로 PID 유지 확인 | `bash -c 'exec sleep 30' &` | `ps -p $!` CMD 변경·PID 동일 | ☐ |
| 잡 제어 `& jobs -l fg bg Ctrl+Z kill %1 wait` | 각 명령 | `jobs` 상태 변화 | ☐ |
| `nohup`·`disown`·`setsid` 로그아웃 생존 | `nohup sleep 600 &` 등 | 로그아웃 후 `pgrep -u dev1 sleep` | ☐ |
| `timeout` 종료 코드 124, `watch -n 1` | `timeout 5 sleep 30; echo $?` | 124 | ☐ |
| 시그널 번호표 (1 2 3 9 15 18 19 20) | `kill -l` | `kill -l 9` → KILL | ☐ |
| `kill -15` → `-9`, `kill -0`, `kill -HUP` MainPID | `kill …` | `pgrep`, `journalctl -u sshd` | ☐ |
| `killall`, `killall -u dev1`, `pkill -9 -f` | 각 명령 | `pgrep -a` 없음 | ☐ |
| `kill -STOP/-CONT` 일시정지·재개 | `kill -STOP PID` | STAT T → R, %CPU 0 | ☐ |
| `trap` 스크립트로 TERM/HUP 처리, KILL 불가 검증 | `trapdemo.sh` | 로그 메시지 | ☐ |
| `nice -n 10/19/-5`, 일반 사용자 음수 거부·19 클램프 | `nice -n …` | `ps -o ni,pri` (PR=20+NI) | ☐ |
| `renice -n -p/-u/-g`, 일반 사용자 하향 불가 | `renice …` | `ps -o ni` | ☐ |
| 동일 코어 NI 0 vs 19 %CPU 차 관찰 | `taskset -c 0 nice -n 19 stress-ng …` | `top -p` %CPU | ☐ |
| top 옵션 `-d -n -b -p -u -U -o -H -c -i -w -1 -S -E -e -s` | `top …` | 각 출력 | ☐ |
| top 헤더 5줄 필드 해석, `free`·`uptime`·`nproc` 대응 | `top -b -n 1 | head -5` | 값 일치 | ☐ |
| top 열 VIRT/RES/SHR 구분 | `top -o %MEM` | `ps -o vsz,rss` | ☐ |
| top 대화식 키 전부 (h d s k r u P M T N < > R x y b z Z 1 t m l H c V f i e E S I 0 L & o = n C W A g q) | `top` | 화면 변화, `~/.toprc` 생성 | ☐ |
| (a) CPU 폭주 → `1` `P` → renice → kill -15 | `stress-ng --cpu 2` | load↓, %Cpu us↓ | ☐ |
| (b) 메모리 압박 → `M`, avail↓, swap↑, `vmstat` si/so, `oom_score`, `dmesg` | `stress-ng --vm 1 --vm-bytes 3G --vm-keep` | `free -h` 회복 | ☐ |
| (c) I/O 병목 → wa↑, D 상태, `iostat -xz`, `vmstat` b/bo, `pidstat -d`, `iotop` | `dd … oflag=direct` | wa 0 복귀 | ☐ |
| (d) 좀비 생성 → `kill -9` 무효 → 부모 종료로 해소; 고아 PPID 1 | `(sleep 1 & exec sleep 300) &` | `Tasks … 0 zombie` | ☐ |
| (e) 삭제 파일 점유 `lsof +L1` → `: > /proc/PID/fd/N`; `lsof -i :22 -u -p`, `fuser -vm` | `lsof …` | `df` 회복 | ☐ |
| `vmstat 1 5 -s -d -a`, 열 의미 | `vmstat` | r b si so bi bo wa | ☐ |
| `iostat -c -d -x -h -z -p`, `mpstat -P ALL`, `pidstat -u -r -d -t -w -p` | 각 명령 | 출력 형식 | ☐ |
| `sar` 수집 활성화, `-u -r -S -b -d -n DEV -q -w -f -s -e`, `sadf -d -j` | `systemctl enable --now sysstat` | `/var/log/sa/saDD` 존재 | ☐ |
| `free -s -c -w`, `uptime -p -s`, `/proc/stat`, `/proc/meminfo`, `ulimit -a -u -n` | 각 명령 | 값 대응 | ☐ |
| `htop` F1~F10·t·/·u·k 조작 | `htop` | — | ☐ |
| `atd` 활성화, `at now + 5 minutes`(heredoc), `at 03:30 tomorrow`, `atq`, `at -c`, `atrm`, `batch` | `at …` | `atq`, 리포트 파일 | ☐ |
| `at.allow`/`at.deny` 로 dev1 거부·원복 | `echo dev1 >> /etc/at.deny` | 거부 메시지 | ☐ |
| dev1 crontab `*/10` 등록, `-l -u -e -i -r`, 파일로 복원 | `crontab …` | `/var/spool/cron/dev1` | ☐ |
| `/etc/cron.d/lab-backup` 7필드 등록 (03:30 root backup.sh) | `cat > /etc/cron.d/lab-backup` | `journalctl -u crond` | ☐ |
| 시간 필드 기출 10종 해석 | — | 표 | ☐ |
| `cron.hourly` → `run-parts` → `0anacron` → `/etc/anacrontab` 추적 | `cat …` | `run-parts --test` | ☐ |
| `cron.allow/deny` 로 dev1 거부·원복, `/var/log/cron`, `journalctl -u crond` | 각 명령 | 거부 메시지, CMD 로그 | ☐ |
| 1분 뒤 임시 cron 등록 → 로그·`%` 이스케이프·PATH 확인 → 삭제 | `/etc/cron.d/lab-test` | `/tmp/cron.test` | ☐ |
| `lab-sysreport.timer/.service` 작성, enable, `list-timers`, `start` 즉시 실행, `journalctl -u` | `systemctl …` | `Result=success` | ☐ |
| 정리: 부하 0, `/data/io.test` 삭제, at 잔여 0, 평시 top 기록 | `pgrep`, `atq`, `top -b -n 2` | `0 zombie`, `0.0 wa` | ☐ |

---

## 기출 연결

| 기출 (문제집·회차·번호 또는 주제) | 이 파트의 단계 |
| --- | --- |
| 실기 R01 #4 — `nginx` 포함 프로세스 PID 조회 (`pgrep`/`ps | grep`) | 1-3 |
| 실기 R01 #5, R03 #8 — 포트 LISTEN 프로세스 (`ss -tlnp`, `lsof -i`) | 5-9 |
| 실기 R01 #10 — 매일 03:30 (`30 3 * * *`) | 7-4 |
| 실기 R02 #10 — 매주 일요일 00:00 (`0 0 * * 0`) | 7-4 |
| 실기 R03 #9 — 10분마다 (`*/10`) | 7-3, 7-4 |
| 실기 R04 #10 — 매월 1일 00:00 (`0 0 1 * *`) | 7-4 |
| 실기 R01·R03·R04 #11 — `/etc/crontab` 7 필드 해석 (`0 2 * * 1 root …`) | 7-4 |
| 필기 R01 #16, R07 #7 — 가로챌 수 없는 시그널 (KILL 9, STOP 19), 기본 TERM, Ctrl+C = INT | 3-1, 3-4 |
| 필기 R01 #35, R03 #28, R07 #40, R10 #33 — nice/renice 범위·root 음수·NI 클수록 낮음 | 4-1, 4-2 |
| 필기 R01 #36 — top 키 (`k` kill, `M` 메모리, `P` CPU, `q` 종료) | 5-4 |
| 필기 R01 #37 — `kill -HUP 1234` 설정 재적재 | 3-2 |
| 필기 R01 #38 — Ctrl+Z 후 백그라운드 계속 = `bg` | 2-1 |
| 필기 R01 #40, R02 #37, R03 #36, R04 #39, R05 #40, R06 #26, R08 #35 — crontab 항목 실행 시점 | 7-4 |
| 필기 R02 #35, R08 #32 — `ps -l` 출력 NI/PRI/S 해석 | 1-1, 1-2, 4-1 |
| 필기 R02 #36 — `renice -n 10 -p 1234` | 4-2 |
| 필기 R03 #10 — 시그널 이름·번호 짝 (INT=2, STOP=19) | 3-1 |
| 필기 R03 #35 — `$$` 현재 셸 PID vs `$!` | 1-4 |
| 필기 R03 #38 — `ps` 출력 STAT 해석 | 1-2 |
| 필기 R03 #58 — `/var/log/cron` 크론 실행 기록 | 7-5, 7-6 |
| 필기 R04 #35 — `ps aux`(BSD) vs `ps -ef`(UNIX) 열 차이 | 1-1 |
| 필기 R04 #36 — 이름으로 시그널 = `killall` | 3-2 |
| 필기 R04 #37 — `kill` 기본 시그널 SIGTERM(15) | 3-2 |
| 필기 R04 #38 — `nohup ./batch.sh &` 동작·`nohup.out` | 2-2 |
| 필기 R04 #40 — `crontab -e -l -r -u` 옵션 | 7-3 |
| 필기 R05 #39 — `/etc/crontab` 과 사용자 crontab 형식 차이 (사용자 필드) | 7-4 |
| 필기 R06 #25 — `cron.allow` 에 user1 만 등록 → 나머지 금지 | 7-2, 7-5 |
| 필기 R06 #35 — top 97.8% 프로세스 대응 절차 (`kill -15` → `-9`) | 3-2, 5-5 |
| 필기 R06 #36 — 최저 우선순위 배치 = `nice -n 19 ./batch.sh` | 4-1 |
| 필기 R06 #65 — 프로세스 수 제한 `/etc/security/limits.conf` (`nproc`) | 6-4 |
| 필기 R07 #6 — 셸 명령 실행 fork → exec → wait → exit | 1-5 |
| 필기 R07 #18 — 좀비 프로세스 정의·`kill -9` 불가·부모 종료 | 5-8 |
| 필기 R07 #30 — `30 2 * * 0` 매주 일요일 02:30 | 7-4 |
| 필기 R08 #28 — `free` 의 available·buff/cache 해석 | 5-2, 6-4 |
| 필기 R08 #29 — `vmstat` r/b/si/so/wa 열 의미 | 5-6, 5-7, 6-1 |
| 필기 R08 #30 — 4코어 `uptime` load average 4.05 해석 | 1-4, 5-2 |
| 필기 R08 #31 — top `%Cpu(s)` wa 높음 = I/O 병목 | 5-2, 5-7 |
| 필기 R08 #33 — `pstree -p` 부모-자식 해석 | 1-3, 5-8 |
| 필기 R08 #34 — `jobs` 출력에서 `fg %1` | 2-1 |
| 필기 R08 #43 — `renice -n -5 -p 1234` (root) | 4-2 |
| 필기 R09 #20 — 일반 사용자가 root 프로세스에 kill 불가 이유 | 3-2 |
| 필기 R10 #13 — SIGSTOP/SIGCONT | 2-1, 3-3 |
| 필기 R10 #14 — 데몬 실행 방식 (standalone, 터미널 없음) | 1-5 |
| 필기 R10 #34 — `*/20 1-3 * * 0` 주간 실행 횟수 (9회) | 7-4 |

---

## 이전 / 다음

[[05-disk-lvm-raid-swap-quota]] ← · → [[07-boot-systemd-log]]

[[README]]
