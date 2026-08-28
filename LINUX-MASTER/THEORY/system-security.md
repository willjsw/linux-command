---
title: 시스템 보안 및 관리
type: exam-theory
category: LINUX-MASTER
tags:
  - exam/linux-master
  - exam/theory
  - linux/security
  - linux/log
  - topic/security
  - topic/backup
  - task/inspect
related: ["[[network-security]]", "[[network-service]]", "[[last]]", "[[journalctl]]", "[[sudo]]", "[[tar]]", "[[dd]]"]
updated: 2026-08-28
---

# 시스템 보안 및 관리

- 리눅스마스터 1급 필기 대비 이론 — 시스템 분석·시스템 보안·백업 3개 영역
- 로그 파일의 종류·경로·조회 명령 조합이 최빈출 → 표 단위 암기 필요

---

## 1. 시스템 분석 — 로그 관리

### 1-1. 주요 로그 파일 (최빈출)

| 로그 파일 | 형식 | 내용 | 조회 명령 |
| --- | --- | --- | --- |
| `/var/log/wtmp` | 바이너리 | 로그인·로그아웃·재부팅 **성공** 이력 | `last` |
| `/var/log/btmp` | 바이너리 | 로그인 **실패** 이력 | `lastb` |
| `/var/run/utmp` | 바이너리 | **현재** 로그인 사용자 상태 | `w`, `who`, `users`, `finger` |
| `/var/log/lastlog` | 바이너리 | 계정별 **마지막** 로그인 정보 | `lastlog` |
| `/var/log/messages` | 텍스트 | 시스템 전반 메시지 (커널·데몬) | `cat`, `tail` |
| `/var/log/secure` | 텍스트 | 인증·보안 관련 (ssh, su, sudo, login) | `cat`, `tail` |
| `/var/log/dmesg` | 텍스트 | 부팅 시 커널 메시지 | `dmesg` |
| `/var/log/cron` | 텍스트 | cron 작업 실행 이력 | `cat`, `tail` |
| `/var/log/maillog` | 텍스트 | 메일 송수신 이력 | `cat`, `tail` |
| `/var/log/xferlog` | 텍스트 | FTP 파일 전송 이력 | `cat`, `tail` |

- 바이너리 로그는 전용 명령으로만 조회 가능 → `cat` 으로 열람 불가가 함정 포인트
- `utmp` 만 `/var/run` (= `/run`) 하위 — 나머지는 `/var/log` 하위
- systemd 저널 조회는 [[journalctl]] — `-p err` 우선순위 필터, `-u <유닛>` 단위 필터

### 1-2. syslog / rsyslog

- 설정 파일: `/etc/rsyslog.conf` (구형 syslog 는 `/etc/syslog.conf`)
- 규칙 형식: `facility.priority  action`

```
# 형식: <facility>.<priority>    <action>
authpriv.*                       /var/log/secure
mail.*                           -/var/log/maillog     # '-' = 비동기 기록
*.emerg                          :omusrmsg:*           # 전 사용자 터미널로 전송
cron.*                           /var/log/cron
*.info;mail.none;authpriv.none   /var/log/messages
```

- **facility**: `kern` `user` `mail` `daemon` `auth` `authpriv` `cron` `lpr` `news` `uucp` `local0~7`
- **priority (낮음→높음)**: `debug` < `info` < `notice` < `warning` < `err` < `crit` < `alert` < `emerg`
	- `mail.info` → info **이상** 전부 기록 (지정 레벨 이상 포함이 기본)
	- `mail.=info` → info **만** 기록
	- `mail.!err` → err 이상 **제외**
	- `*.none` → 해당 facility 제외
- 원격 로그 서버 전송: `*.* @192.168.0.10` (UDP 514), `@@` 는 TCP

### 1-3. logrotate

- 로그 순환(비대화 방지) 도구 — cron 에 의해 주기 실행
- 설정: `/etc/logrotate.conf` (전역), `/etc/logrotate.d/` (서비스별)
- 주요 지시자: `daily`/`weekly`/`monthly` (주기), `rotate 4` (보관 세대수), `compress` (압축), `create 0644 root root` (신규 파일 생성), `notifempty` (빈 파일 미순환), `postrotate` (순환 후 스크립트)

---

## 2. 시스템 보안

### 2-1. 특수 권한 (SetUID / SetGID / Sticky Bit)

| 구분 | 8진수 | 표기 | 효과 |
| --- | --- | --- | --- |
| SetUID | 4000 | `-rwsr-xr-x` (소유자 x 자리 `s`) | 실행 시 **소유자 권한**으로 동작 (예: `/usr/bin/passwd`) |
| SetGID | 2000 | `-rwxr-sr-x` (그룹 x 자리 `s`) | 실행 시 그룹 권한 / 디렉터리엔 그룹 상속 |
| Sticky Bit | 1000 | `drwxrwxrwt` (기타 x 자리 `t`) | 디렉터리 내 파일 삭제를 소유자만 허용 (예: `/tmp`) |

- 실행 권한 없이 특수 권한만 있으면 대문자 표기 (`S`, `T`)
- SetUID 파일 전수 검색: `find / -perm -4000 -type f` — 침해 점검 단골 → [[find]]

### 2-2. umask

- 새 파일·디렉터리의 **기본 권한 차감 마스크**
- 파일 기본 666, 디렉터리 기본 777 에서 umask 값 차감
	- umask `022` → 파일 `644`, 디렉터리 `755`
	- umask `077` → 파일 `600`, 디렉터리 `700`
- 파일은 umask 와 무관하게 실행 권한 미부여 (666 기준 계산)

### 2-3. 파일 속성 — chattr / lsattr

- `chattr +i <파일>` : 불변(immutable) — root 도 삭제·수정 불가 (**i**mmutable)
- `chattr +a <파일>` : 추가만 허용 (**a**ppend only) — 로그 변조 방지
- `lsattr` : 속성 조회
- 침해 대응 시 변조 방지·백도어 고정 양쪽에 쓰이는 양날 속성

### 2-4. PAM (Pluggable Authentication Modules)

- 인증을 애플리케이션에서 분리한 모듈형 인증 체계
- 설정: `/etc/pam.d/<서비스명>`, 모듈: `/usr/lib64/security/`
- 제어 구문 4종: `required` (실패해도 나머지 검사 후 최종 거부) / `requisite` (실패 시 즉시 거부) / `sufficient` (성공 시 즉시 허용) / `optional` (참고용)
- 대표 모듈
	- `pam_securetty.so` : `/etc/securetty` 에 등록된 터미널만 root 로그인 허용
	- `pam_wheel.so` : wheel 그룹만 `su` 허용 (`/etc/pam.d/su`)
	- `pam_tally2.so` / `pam_faillock.so` : 로그인 실패 횟수 제한
	- `pam_cracklib.so` / `pam_pwquality.so` : 비밀번호 복잡도 강제

### 2-5. su / sudo

- `su -` : 대상 사용자 **환경변수까지 전환** (`-` 유무가 출제 포인트) → [[sudo]]
- sudo 설정: `/etc/sudoers` — 반드시 `visudo` 로 편집 (문법 검증 제공)
	- 형식: `사용자 호스트=(대상사용자) 명령` → `user1 ALL=(ALL) NOPASSWD: /usr/bin/systemctl`
	- `%wheel ALL=(ALL) ALL` : wheel 그룹 전체 허용
- 실행 이력: `/var/log/secure` 에 기록

### 2-6. TCP Wrapper

- 라이브러리 `libwrap` 기반 호스트 접근 제어 (xinetd·sshd 등 연동)
- `/etc/hosts.allow` → `/etc/hosts.deny` 순서로 검사, **allow 우선**, 양쪽 모두 없으면 허용
- 형식: `데몬 : 클라이언트 [: 옵션]`

```
# /etc/hosts.allow
sshd : 192.168.0.          # 대역 허용 (마침표 종료 = 프리픽스)
in.telnetd : .example.com  # 도메인 접미 일치

# /etc/hosts.deny
ALL : ALL                  # 나머지 전부 거부
```

- `twist` (응답 후 거부), `spawn` (명령 실행) 옵션 출제 이력

### 2-7. 계정·비밀번호 정책

- `/etc/passwd` : `계정:x:UID:GID:설명:홈:셸` — 7개 필드
- `/etc/shadow` : `계정:암호화비밀번호:최종변경일:최소기간:최대기간:경고일:비활성일:만료일:예약`
	- 비밀번호 필드 `!` `*` = 잠금 상태
- `chage -l <계정>` : 만료 정책 조회, `chage -M 90 <계정>` : 최대 사용기간 (**M**axdays)
- 로그인 차단: `usermod -L` (잠금) / `usermod -s /sbin/nologin` (셸 차단) → [[usermod]]
- `/etc/login.defs` : UID 범위·비밀번호 기본 정책 전역 설정

### 2-8. 보안 점검 도구 (개념 암기)

| 도구 | 용도 |
| --- | --- |
| John the Ripper | 비밀번호 크래킹 (취약 비밀번호 점검) |
| Tripwire | 파일 무결성 검사 (변조 탐지) |
| Nmap | 포트 스캔·서비스 식별 |
| Nessus / OpenVAS | 종합 취약점 스캐너 |
| tcpdump / Wireshark | 패킷 캡처·분석 |
| fail2ban | 로그 기반 반복 실패 IP 자동 차단 |

- SELinux 모드·감사 로그는 [[getenforce]] · [[ausearch]] 참조

---

## 3. 백업

### 3-1. 백업 전략 (최빈출)

| 전략 | 대상 | 복원 | 특징 |
| --- | --- | --- | --- |
| 전체 백업 (Full) | 전체 데이터 | 1회 복원 | 시간·용량 최대, 복원 단순 |
| 증분 백업 (Incremental) | **직전 백업 이후** 변경분 | 전체 + 증분 전부 순서 복원 | 백업 최소·복원 복잡 |
| 차등 백업 (Differential) | **마지막 전체 백업 이후** 변경분 | 전체 + 최신 차등 1개 | 증분보다 용량 크고 복원 단순 |

### 3-2. 백업 도구

| 도구 | 특징 | 예시 |
| --- | --- | --- |
| `tar` | 아카이브 표준, 증분 지원 (`-g` 스냅샷) | `tar -g snap -cvzf full.tar.gz /home` → [[tar]] |
| `cpio` | 파일 목록을 표준입력으로 수신, 손상 아카이브 부분 복원에 강함 | `find /home | cpio -o > backup.cpio` / 복원 `cpio -id < backup.cpio` |
| `dump` / `restore` | **파일시스템(파티션) 단위** 백업, 레벨 0~9 (0=전체, 1~9=증분) | `dump -0u -f /dev/st0 /dev/sda1` / `restore -rf /dev/st0` |
| `dd` | 블록 단위 저수준 복제 (디스크 이미지) | `dd if=/dev/sda of=/dev/sdb bs=4M` → [[dd]] ⚠ 파괴적 |
| `rsync` | 원격 동기화, 변경 블록만 전송 | `rsync -avz /data user@host:/backup` |

- `dump` 레벨 운용: 일 0 → 월 1 → 화 2 … 방식의 증분 스케줄 계산 문제 출제
- `cpio` 옵션: `-o` (생성, **o**utput) / `-i` (추출, **i**nput) / `-d` (디렉터리 생성) / `-v` (상세)
- `tar` 증분: `-g` (리스트 파일 기반, **g** = listed-incremental)

---

## 연관 문서
- [[network-security]] — 네트워크 침해 유형·대응 (iptables·IDS)
- [[network-service]] — 서비스별 설정 파일·데몬
- [[last]] — wtmp 조회 실사용 문서
- [[journalctl]] — systemd 저널 조회
- [[sudo]] — 권한 상승 실사용 문서
- [[tar]] · [[dd]] — 백업 도구 실사용 문서
- [[find]] — SetUID 전수 검색
- [[getenforce]] · [[ausearch]] — SELinux
