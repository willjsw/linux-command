---
command: man
category: SYSTEM-INFO
aliases: [manual, whatis, apropos, mandb, whereis, help, manpath]
tags:
  - linux/system
  - task/inspect
  - task/search
  - task/verify
  - task/diagnose
  - topic/troubleshooting
  - privilege/mixed
related: ["[[which]]", "[[find]]", "[[fhs]]", "[[rpm]]", "[[grep]]", "[[env]]", "[[file]]", "[[dnf]]"]
distro: 전체 (RHEL 계열 구현체는 man-db)
verified: Rocky Linux 9.3 (aarch64) — man-db 2.9.3-9.el9 / man-pages 6.04-10.el9_8
updated: 2026-09-06
---

# man

- 매뉴얼 페이지 조회 도구 — 명령·설정파일·시스템콜의 **정본 설명서**
- 어원: **man**ual (설명서)
- RHEL 계열 구현체는 `man-db` 패키지 → `man` `whatis` `apropos` `mandb` `manpath` 전부 동일 패키지 제공
- 조회는 일반 사용자 가능, 검색 캐시 갱신(`mandb`)은 root 필요
- **최소 설치본·컨테이너는 매뉴얼 미포함** → `dnf install man-db man-pages` 선행 필요 ([[dnf]])
	- RPM 설치 시 `%_excludedocs 1` 매크로가 문서 파일 자체를 제외 → 패키지 재설치로 복구
- 미지 명령 조사 전체 흐름은 문서 하단 [[#미지 명령 조사 절차]] 참조

---

## man

```bash
man <명령>
man <섹션> <명령>

# Examples
man ls                     # 섹션 무관 첫 일치 페이지
man 5 passwd               # 5번 섹션(파일 형식) 한정 — /etc/passwd 형식
man -s 5 passwd            # 위와 동일 (-s 명시형)
man man                    # man 자신의 매뉴얼 — 섹션 번호표 수록
man 7 man-pages            # 매뉴얼 페이지 작성 규약 — 섹션 구성 근거
man -P cat whatis          # 페이저 없이 표준출력으로 전량 출력
MANPAGER=cat man whatis    # 환경변수로 페이저 대체 (동일 효과)
MANWIDTH=60 man -P cat ls  # 출력 폭 강제 (파이프 시 폭 고정 목적)
man -l ./mytool.1.gz       # 설치되지 않은 로컬 매뉴얼 파일 직접 표시
```

### 명령어 설명
- 사용 목적
	- 처음 쓰는 명령의 **정확한 문법·옵션 확인 시 사용** → 추측 실행 방지
	- 설정 파일의 필드 구성 확인 시 사용 (`/etc/passwd`·`/etc/fstab` 등 5번 섹션)
	- 옵션 동작이 배포판·버전마다 다를 때 **해당 시스템의 실제 동작 확인 시 사용**
- 특이사항
	- **동일 이름이 여러 섹션에 존재** → 섹션 미지정 시 낮은 번호 우선 (`passwd` 는 1번 명령 vs 5번 파일)
	- 검색 순서는 `man --usage` 의 `-S LIST` 기본값 기준 → `1 1p 8 2 3 3p 3pm 4 5 6 7 9 0p n l p o ...`
	- 페이지 부재 시 `No manual entry for <명령>` + **종료 코드 16** → 스크립트 분기 판정 가능
	- **셸 내장 명령은 매뉴얼 미제공** → `cd` `export` `help` 등은 `help <명령>` 사용
	- 페이저 기본값은 `less` → 종료는 `q`, 검색은 `/패턴`
	- 실행 파일이 있어도 매뉴얼이 없을 수 있음 → 존재 판정은 [[which]] 로 별도 수행

### 옵션
- `-s <섹션>` / `<섹션>` : 섹션 한정 (**s**ection) — `man 5 passwd` = `man -s 5 passwd`
- `-a` : 일치하는 **모든** 섹션을 순차 표시 (**a**ll) — 하나 종료 시 다음 페이지로 이동
- `-w` : 페이지를 출력하지 않고 **파일 경로만** 출력 (**w**here) — `--path` `--location` 동의
- `-f` : 한 줄 요약만 출력 (`--whatis` 동의) — `whatis` 와 동일
- `-k` : 키워드로 요약 검색 (`--apropos` 동의) — `apropos` 와 동일
- `-K` : 매뉴얼 **본문 전문** 검색 (대문자 **K**) — `-k` 로 못 찾을 때 최후 수단
- `-P <페이저>` : 페이저 지정 (**P**ager) — `-P cat` 으로 파이프·grep 조합
- `-l <파일>` : 로컬 매뉴얼 파일 직접 표시 (**l**ocal file) — 미설치 도구의 첨부 매뉴얼 확인
- `--path` : 매뉴얼 검색 경로 출력 (= `manpath`)
- `-M <경로>` : 검색 경로 강제 지정 (**M**anpath) ※ 미검증
- `-L <로케일>` : 로케일 한정 표시 (**L**ocale) ※ 미검증

---

## man 섹션 번호

- 동일 이름의 명령·파일·함수를 구분하는 **번호 체계** → 섹션을 모르면 엉뚱한 페이지 조회
- 출처: `man man` 실기 출력 (Rocky 9.3)

| 섹션 | 범위 | 대표 예시 |
| --- | --- | --- |
| **1** | 실행 프로그램·셸 명령 | `man 1 ls` `man 1 grep` |
| **2** | 시스템 콜 (커널 제공 함수) | `man 2 open` `man 2 mount` |
| **3** | 라이브러리 함수 (libc 등) | `man 3 printf` `man 3 malloc` |
| **4** | 특수 파일 (주로 `/dev`) | `man 4 null` `man 4 tty` |
| **5** | **파일 형식·규약** | `man 5 passwd` `man 5 fstab` `man 5 crontab` |
| **6** | 게임 | — |
| **7** | 기타 (규약·매크로·개념) | `man 7 man-pages` `man 7 regex` `man 7 signal` |
| **8** | **시스템 관리 명령** (주로 root) | `man 8 mount` `man 8 useradd` |
| **9** | 커널 루틴 (비표준) | — |

### 명령어 설명
- 사용 목적
	- 같은 이름의 명령/파일 형식 구분 시 사용 → `passwd` 는 1번(명령)과 5번(파일) 양쪽 존재
	- 설정 파일 문법 확인은 **거의 항상 5번 섹션**
	- 관리 명령 조회는 8번 섹션 → 일반 명령(1번)과 옵션 체계가 다른 경우 존재
- 특이사항
	- **`mount` 는 2번(시스템콜)과 8번(관리 명령) 양쪽 존재** → 셸에서 쓰는 것은 8번 ※ 미검증
	- `/usr/share/man` 하위 디렉터리명이 곧 섹션 (`man1` `man5` `man8`) → [[fhs]] 배치 규약
	- `man1p` `man3p` 등 `p` 접미는 POSIX 규격판 → GNU 확장 미포함 판본
	- 섹션 표기 관례는 `명령(섹션)` 형식 → 문서·에러 메시지의 `passwd(5)` 는 "5번 섹션 passwd" 의미

---

## whatis / man -f — 한 줄 요약

```bash
whatis <명령>
man -f <명령>

# Examples
whatis passwd              # passwd (5) - password file
whatis -w "pass*"          # 와일드카드 일치 전량
man -f printf              # printf (3) - formatted output conversion
```

### 명령어 설명
- 사용 목적
	- **명령의 정체를 한 줄로 확인 시 사용** → 전체 매뉴얼 열기 전 선별
	- 존재하는 섹션 목록 확인 시 사용 → 다중 섹션 여부 즉시 판별
- 특이사항
	- **정확 일치만 검색** → 부분 문자열 미탐지, 부분 검색은 `apropos` 사용
	- 캐시(`mandb`) 미생성 시 `<명령>: nothing appropriate.` 반환 → 페이지 부재와 구분 불가
	- 어원: **what is** (무엇인가)

### 옵션
- `-w` : 인자를 와일드카드로 해석 (**w**ildcard) — `"pass*"` 형태, 따옴표 필수
- `-s <섹션>` : 섹션 한정 (**s**ection)
- `-r` : 인자를 정규표현식으로 해석 (**r**egex) ※ 미검증

---

## apropos / man -k — 키워드 검색

```bash
apropos <키워드>
man -k <키워드>

# Examples
man -k passwd              # 요약·이름에 passwd 포함 전량
man -k "^passwd"           # 정규식 — 이름이 passwd 로 시작
apropos -s 5 passwd        # 5번 섹션으로 한정
man -k "copy files"        # 명령 이름을 모를 때 기능 문구로 역탐색
```

- 실기 출력 (Rocky 9.3)

```
$ man -k passwd
fgetpwent_r (3)      - get passwd file entry reentrantly
getpwent_r (3)       - get passwd file entry reentrantly
passwd (5)           - password file
passwd2des (3)       - RFS password encryption
```

### 명령어 설명
- 사용 목적
	- **명령 이름을 모를 때 기능 설명으로 역탐색 시 사용** → "무슨 명령을 써야 하는가" 단계의 진입점
	- 관련 명령군 일괄 파악 시 사용 (`man -k lvm` → LVM 도구 전량)
- 특이사항
	- **검색 대상은 이름과 한 줄 요약뿐** → 본문 내용은 미검색, 본문까지는 `man -K`
	- **인자는 정규표현식** → `^` `$` `.` 유효, 리터럴 점·별표는 이스케이프 필요
	- 캐시 미생성 시 `nothing appropriate.` → 원인은 검색 실패가 아닌 **DB 부재**, `mandb` 선행 필요
	- `-k` 와 `-w` 는 배타 옵션 → `man: -k -w : incompatible options` 발생
	- 어원: **apropos** (∼에 관하여, 프랑스어 à propos)

### 옵션
- `-s <섹션>` : 섹션 한정 (**s**ection) — `apropos -s 5 passwd`
- `-a` : 다중 키워드 **AND** 결합 (**a**nd) ※ 미검증
- `-e` : 정확 일치만 (**e**xact) ※ 미검증

---

## man -K — 본문 전문 검색

```bash
man -K <문자열>
man -K -w <문자열>

# Examples
man -K -w MANPATH          # 본문에 MANPATH 를 포함한 페이지 경로 열거
man -K SIGKILL             # 본문 언급 페이지를 순차 표시 (대화형)
```

### 명령어 설명
- 사용 목적
	- 옵션 이름·환경변수·에러 문구가 **어느 매뉴얼에 설명돼 있는지 역추적 시 사용**
	- `-k` 로 안 잡히는 세부 항목 탐색 시 사용 (요약에 없는 옵션명 등)
- 특이사항
	- **전 매뉴얼 압축 해제 후 검색** → 페이지 수에 비례해 느림, `-w` 병용 시 즉시 반환
	- 동일 파일이 중복 출력될 수 있음 → 페이지당 다중 일치 시 발생
	- 대화형 모드는 페이지마다 표시 여부를 묻는 방식 → 스크립트 부적합

---

## mandb — 검색 캐시 생성·갱신

```bash
sudo mandb
sudo mandb -q

# Examples
sudo mandb                 # 신규 설치 매뉴얼 색인 반영
sudo mandb -q              # 조용히 실행 (스크립트용)
```

- 실기 출력 (Rocky 9.3, man-pages 설치 직후)

```
$ mandb
46 man subdirectories contained newer manual pages.
2474 manual pages were added.
0 stray cats were added.
0 old database entries were purged.
```

### 명령어 설명
- 사용 목적
	- **`whatis`·`apropos` 가 `nothing appropriate.` 반환 시 최우선 조치**
	- 패키지 설치 직후 신규 매뉴얼을 검색 대상에 편입 시 사용
- 특이사항
	- **캐시 없이는 `-k` `-f` 전부 무결과** → `man <명령>` 직접 조회는 캐시와 무관하게 동작
	- 전역 DB 는 `/var/cache/man/` → 갱신에 root 필요, 일반 사용자는 자기 홈에만 생성
	- 정기 갱신은 `man-db-cache-update.service` 가 수행 (man-db 패키지 제공) → 수동 실행은 즉시 반영 목적
	- 어원: **man** + **d**ata**b**ase
	- 구형 명칭 `makewhatis` 는 man-db 전환으로 폐기 → RHEL 7 이후 `mandb`

### 옵션
- `-q` : 경고 억제 조용히 실행 (**q**uiet)
- `-c` : 기존 DB 삭제 후 재생성 (**c**reate) ※ 미검증

---

## man -w / manpath — 매뉴얼 파일 위치

```bash
man -w <명령>
manpath

# Examples
man -w 5 passwd            # /usr/share/man/man5/passwd.5.gz
man -aw passwd             # 일치하는 전 섹션 경로 열거
manpath                    # /usr/local/share/man:/usr/share/man
man --path                 # manpath 와 동일 출력
ls /usr/share/man/         # 섹션 디렉터리 구조 확인
find /usr/share/man -name "passwd*"   # 파일 기준 직접 탐색
```

### 명령어 설명
- 사용 목적
	- **매뉴얼 원본 파일 경로 확인 시 사용** → 복사·변환·소스 확인 목적
	- 페이지 존재 여부를 출력 없이 판정 시 사용 (종료 코드 0/16)
	- 자체 제작 매뉴얼 배치 위치 결정 시 사용 → `/usr/local/share/man/man1/`
- 특이사항
	- **매뉴얼은 `.gz` 압축 저장** → `cat` 불가, `zcat`·`man -l` 사용
	- 검색 경로는 `PATH` 기반으로 동적 결정 → `/usr/local/bin` 이 `PATH` 에 있으면 `/usr/local/share/man` 편입
	- 경로 추가는 `MANPATH` 환경변수 또는 `/etc/man_db.conf` ([[env]])
	- `man -w` 결과를 [[rpm]] 에 전달하면 **제공 패키지 역추적** 가능

---

## whereis — 바이너리·소스·매뉴얼 일괄 조회

```bash
whereis <명령>

# Examples
whereis passwd             # passwd: /usr/bin/passwd /etc/passwd /usr/share/man/man5/passwd.5.gz
whereis -b passwd          # 바이너리만
whereis -m passwd          # 매뉴얼만
```

### 명령어 설명
- 사용 목적
	- 실행 파일·매뉴얼·설정 파일 위치를 **한 번에 확인 시 사용**
	- [[which]] 로 실행 경로, `man -w` 로 매뉴얼 경로를 따로 조회할 필요 제거
- 특이사항
	- **`PATH` 가 아닌 고정 표준 경로 목록을 탐색** → `PATH` 밖 설치본도 탐지 가능, 반대로 비표준 경로는 미탐지
	- 실행 대상 판정에는 부적합 → 실제 실행 파일은 [[which]] `type` 기준
	- `/etc/passwd` 처럼 **동명의 무관한 파일이 섞여 출력** → 결과 해석 주의
	- 어원: **where is** (어디에 있는가)

### 옵션
- `-b` : 바이너리만 (**b**inary)
- `-m` : 매뉴얼만 (**m**anual)
- `-s` : 소스만 (**s**ource) ※ 미검증

---

## --help / help — 매뉴얼 없을 때

```bash
<명령> --help
help <셸 내장 명령>

# Examples
grep --help | head -20     # 요약 사용법 (매뉴얼보다 짧음)
ls --help | grep -- "-h"   # 특정 옵션만 발췌
help cd                    # 셸 내장 명령 — man 페이지 없음
help                       # 내장 명령 전체 목록
man --usage                # man 자신의 옵션 한 줄 요약
```

### 명령어 설명
- 사용 목적
	- **매뉴얼 미설치 환경(컨테이너·최소 설치)에서 대체 수단으로 사용**
	- 옵션 철자만 빠르게 확인 시 사용 → 매뉴얼 열기보다 빠름
	- 셸 내장 명령 확인 시 사용 → `cd` `export` `alias` 등은 `man` 불가
- 특이사항
	- **`-h` 는 표준 아님** → `-h` 가 `--human-readable` 인 명령 다수 (`df -h`·`ls -h`), 오동작 위험
	- 일부 명령은 `--help` 를 stderr 로 출력 → 파이프 시 `2>&1` 필요
	- 내장 명령 판정은 `type <명령>` → `is a shell builtin` 출력 시 `help` 대상 ([[which]])
	- 정보량은 `--help` < `man` → 옵션 상호작용·예외는 매뉴얼에만 기재

---

## 미지 명령 조사 절차

- 처음 보는 명령·옵션을 만났을 때 **판단 순서** → 위에서부터 순차 적용

| 순서 | 질문 | 명령 | 판정 |
| --- | --- | --- | --- |
| 1 | 명령이 존재하는가 | `type <명령>` / `which <명령>` | 부재 시 → 5번으로 |
| 2 | 내장인가 외부인가 | `type <명령>` | `shell builtin` → `help <명령>` |
| 3 | 한 줄 정체는 | `whatis <명령>` | 다중 섹션 여부 동시 확인 |
| 4 | 상세 문법은 | `man <섹션> <명령>` | 없으면 → `<명령> --help` |
| 5 | 어느 패키지 소속인가 | `rpm -qf $(which <명령>)` | 미설치면 `dnf provides */<명령>` ※ 미검증 |
| 6 | 이름을 모른다 | `man -k "<기능 문구>"` | 무결과면 → `sudo mandb` 후 재시도 |
| 7 | 옵션·변수명만 안다 | `man -K -w <문자열>` | 본문 전문 검색 |
| 8 | 파일이 어디 있는가 | `whereis` / `man -w` / `find` | 비표준 경로는 [[find]] |

```bash
# 전형적 조사 흐름 — 미지 명령 mandb 를 조사하는 경우
type mandb                          # /usr/bin/mandb → 외부 명령
whatis mandb                        # mandb (8) - create or update the manual page index caches
man 8 mandb                         # 8번 = 관리 명령, 상세 확인
rpm -qf $(which mandb)              # man-db-2.9.3-9.el9.aarch64 → 소속 패키지
man -w 8 mandb                      # /usr/share/man/man8/mandb.8.gz → 원본 위치

# 기능만 아는 경우 — "디스크 사용량" 명령 찾기
man -k "disk space"                 # 요약 기준 후보 열거
man -k "^df"                        # 이름 기준 좁히기

# 매뉴얼이 아예 없는 환경
grep --help 2>&1 | head -20         # 요약 사용법으로 대체
sudo dnf install -y man-db man-pages   # 근본 해결 (네트워크 필요)
sudo mandb                          # 설치 후 색인 생성 필수
```

### 특이사항
- **1단계를 건너뛰면 오판 발생** → 매뉴얼 부재를 명령 부재로 착각, 반대도 성립
- 매뉴얼 내용과 실제 동작이 다를 수 있음 → 배포판 패치 반영 지연, 최종 근거는 **실행 결과**
- 파괴적 명령(`dd`·`mkfs`·`parted`)은 매뉴얼 확인 후에도 **테스트 환경 선행 검증 필수**
- 옵션 의미가 명령마다 다름 → `-a` 는 `ls` 에서 all, `man` 에서 all-sections, `useradd` 에 부재

---

## 페이저 조작 (less)

- `man` 기본 페이저는 `less` → 매뉴얼 열람 효율은 페이저 조작에 좌우

| 키 | 동작 |
| --- | --- |
| `q` | 종료 |
| `Space` / `b` | 한 화면 아래 / 위 |
| `d` / `u` | 반 화면 아래 / 위 |
| `g` / `G` | 문서 처음 / 끝 |
| `/패턴` | 아래 방향 검색 |
| `?패턴` | 위 방향 검색 |
| `n` / `N` | 다음 / 이전 일치 |
| `h` | 페이저 도움말 |

- 옵션 항목으로 바로 이동 시 `/^ *-<옵션문자>` 형태 검색 관용
- 전량을 텍스트로 다뤄야 하면 `man -P cat <명령>` 후 [[grep]] 파이프

---

## 연관 명령어
- [[which]] : 명령 존재·실행 경로 판정 — 매뉴얼 조회 **선행 단계**, 내장 명령은 `type` 필요
- [[find]] : `man -w` 로 못 찾는 비표준 경로 매뉴얼·문서 탐색 (`find / -name "*.1.gz"`)
- [[grep]] : `man -P cat <명령> | grep` 으로 옵션 발췌 — 페이저 없이 필터
- [[rpm]] : `rpm -qf $(man -w <명령>)` 로 매뉴얼 제공 패키지 역추적, `rpm -qd <패키지>` 로 문서 목록
- [[dnf]] : `man-db` `man-pages` 설치 수단, `dnf provides */<명령>` 으로 미설치 명령의 패키지 역탐색
- [[fhs]] : `/usr/share/man` 배치 근거 — 섹션 디렉터리 구조의 표준
- [[env]] : `MANPATH` `MANPAGER` `MANWIDTH` 등 매뉴얼 동작 제어 환경변수
- [[file]] : `man -l` 대상 파일의 형식 판정 (gzip·troff 여부)
