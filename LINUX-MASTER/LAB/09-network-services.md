---
title: LAB 09 — 네트워크 서비스 구축(Apache·BIND·NFS·Samba·FTP·메일·CUPS)
type: exam-lab
part: 09
tags:
  - exam/linux-master
  - exam/lab
  - linux/network
  - linux/service
  - topic/web-server
  - topic/dns
  - topic/mail
  - task/configure
  - task/verify
related: ["[[README]]", "[[08-network-config]]", "[[10-security-firewall-selinux]]", "[[../THEORY/network-service]]", "[[../THEORY/system-security]]", "[[../../SERVICE-SYSTEMD/systemctl]]", "[[../../NETWORK-MANAGEMENT/ss]]"]
distro: Rocky Linux 9 (aarch64, UTM)
updated: 2026-09-03
---

# LAB 09 — 네트워크 서비스 구축(Apache·BIND·NFS·Samba·FTP·메일·CUPS)

- 사내 서버 1대에 **7개 서비스**를 순서대로 올림 — 웹(Apache) · DNS(BIND) · 파일공유(NFS·Samba) · 파일전송(vsftpd) · 메일(Postfix+Dovecot) · 인쇄(CUPS)
- 모든 서비스를 **동일 골격 8단계**로 구축: 설치 → 설정 파일 편집 → 문법 검사 → 기동·enable → 방화벽 개방 → SELinux 조정 → **클라이언트 실접속 검증** → 로그 확인
- 서버·클라이언트를 한 VM 안에서 겸함 — 루프백 `127.0.0.1` 과 자기 IP `192.168.64.10` 두 경로로 각각 검증해 "listen 주소" 개념을 체감
- 필기 범위지만 UTM 에서 돌리면 충돌·무의미한 서비스(DHCP·Squid·xinetd·NIS·LDAP)는 **설정 파일 작성 + 문법 검사**까지만 하고 `※ 미실행` 표기

> **이 파트의 시나리오**: Part 08 에서 고정 IP `192.168.64.10` 과 호스트명 `srv01.lab.local`, SSH 키 인증까지 끝났다. 이제 이 서버를 실제 "사내 인트라넷 서버" 로 만든다 — 개발팀이 볼 웹 사이트, 사내 이름 해석을 담당할 DNS, 리눅스끼리 쓸 NFS 와 윈도·macOS 에서 쓸 Samba, 외부 반입용 FTP, 팀 내 알림용 로컬 메일, 그리고 프린터 큐를 차례로 올린다. 방화벽·SELinux 는 여기서는 "열기만" 하고, 정책 설계·강화는 [[10-security-firewall-selinux]] 에서 다룬다.

- 선행 자원
  - 계정 `dev1` `dev2` `ops1`, 그룹 `devteam` `opsteam` — **Part 03** 에서 생성됨
  - `/srv/share` (LVM `vg_lab/lv_share`, xfs) — **Part 05** 에서 마운트됨. Samba 공유로 사용
  - 고정 IP `192.168.64.10/24`, 호스트명 `srv01.lab.local`, `/etc/hosts` 항목 — **Part 08**
  - `policycoreutils-python-utils` (semanage) — **Part 02** 에서 설치됨. 없으면 `dnf install -y policycoreutils-python-utils`
- ⚠️ 이 파트는 서비스 7개를 동시에 띄운다. 각 절 끝의 검증을 통과하지 못한 채 다음 절로 넘어가면 원인 분리가 어려움 — **한 서비스씩 끝내고 넘어갈 것**

---

## 1. 공통 준비

### 1-1. 이번 파트에서 만들 것 — 서비스 총괄표

> **상황**: 작업 전에 "무엇을 · 어디에 · 어느 포트로" 를 한 장에 고정한다. 필기에서 **서비스명 ↔ 데몬명 ↔ 설정 파일 ↔ 포트** 4축 매칭이 최빈출이므로 이 표 자체가 암기 대상.

| 서비스 | 패키지 | 데몬(유닛) | 주 설정 파일 | 이 파트의 실습 경로 | 포트 |
| --- | --- | --- | --- | --- | --- |
| 웹 | `httpd` `httpd-tools` `mod_ssl` | `httpd` | `/etc/httpd/conf/httpd.conf` | `/var/www/html`, `/srv/www/intranet` | 80 · 8080 · 443/TCP |
| DNS | `bind` `bind-utils` | `named` | `/etc/named.conf` | `/var/named/lab.local.zone` | 53/UDP·TCP |
| NFS | `nfs-utils` | `nfs-server` | `/etc/exports` | `/srv/nfs/data` → `/mnt/nfs` | 2049/TCP, 111 |
| Samba | `samba` `samba-client` `samba-common-tools` `cifs-utils` | `smb`, `nmb` | `/etc/samba/smb.conf` | `/srv/share` → `/mnt/smb` | 139·445/TCP, 137·138/UDP |
| FTP | `vsftpd` `lftp` | `vsftpd` | `/etc/vsftpd/vsftpd.conf` | `/var/ftp/pub`, 사용자 홈 | 21/TCP(제어), 20(데이터) |
| 메일(발송) | `postfix` `s-nail` | `postfix` | `/etc/postfix/main.cf` | `/var/spool/mail/*` | 25/TCP |
| 메일(수신) | `dovecot` | `dovecot` | `/etc/dovecot/dovecot.conf` | POP3·IMAP | 110 · 143/TCP |
| 인쇄 | `cups` `cups-client` | `cups` | `/etc/cups/cupsd.conf` | 더미 프린터 `labprn` | 631/TCP |

- 데몬명과 systemd 유닛명이 다른 것에 주의 — Samba 는 데몬 `smbd`/`nmbd`, 유닛은 `smb`/`nmb`. NFS 는 데몬 `nfsd`, 유닛은 `nfs-server`
- DNS 는 **UDP 53 이 기본**, 응답이 512 바이트를 넘거나 존 전송(AXFR)일 때 TCP 53 사용 → 방화벽은 둘 다 열어야 함

> 📝 **시험 포인트**: 필기 FULL r01-75, r02-90, r04-74, r05-77, r07-76, r10-20/77 처럼 "프로토콜 — 포트" 짝을 틀린 것 고르기가 반복 출제. SMTP 25 / POP3 110 / IMAP 143 / SMTPS 465 / 서브미션 587 / IMAPS 993 / POP3S 995 는 통째로 암기.

### 1-2. 선행 자원 확인

> **상황**: 서비스 설정 파일에 `192.168.64.10`, `srv01.lab.local`, `/srv/share` 를 그대로 적어 넣을 것이므로, 먼저 Part 03·05·08 결과물이 살아 있는지 확인한다.

```bash
hostnamectl | grep -i 'static hostname'      # srv01.lab.local
ip -4 addr show enp0s1 | grep inet           # 192.168.64.10/24
getent hosts srv01.lab.local intranet.lab.local   # /etc/hosts 항목 (Part 08)
findmnt /srv/share                           # LVM 공유 볼륨 (Part 05)
id dev1; id dev2; id ops1                    # 계정·그룹 (Part 03)
getenforce                                   # SELinux 모드
```

- `getent hosts <이름>` : `/etc/nsswitch.conf` 의 `hosts:` 순서(files → dns)를 그대로 따라 이름 해석 (**get ent**ries)
- `findmnt <경로>` : 마운트 트리에서 해당 경로만 조회 — `mount | grep` 보다 정확

**검증**

```bash
grep -E 'lab\.local' /etc/hosts
```

```text
# hostnamectl | grep -i 'static hostname'
   Static hostname: srv01.lab.local
# ip -4 addr show enp0s1 | grep inet
    inet 192.168.64.10/24 brd 192.168.64.255 scope global noprefixroute enp0s1
# grep -E 'lab\.local' /etc/hosts
192.168.64.10   srv01.lab.local srv01
# findmnt /srv/share
TARGET     SOURCE                        FSTYPE OPTIONS
/srv/share /dev/mapper/vg_lab-lv_share   xfs    rw,relatime,...
# getenforce
Enforcing
```

- `intranet.lab.local` 이 아직 없으면 이 파트 2-9 에서 추가 (또는 3절 DNS 구축 후 DNS 로 해석)

> 📝 **시험 포인트**: 이름 해석 순서는 `/etc/nsswitch.conf` 의 `hosts: files dns` — **`/etc/hosts` 가 DNS 보다 먼저**. 필기 FULL r05-43, r07-11 에서 "hosts 파일에 있으면 DNS 질의를 하지 않는다" 로 출제.

### 1-3. 서비스 구축 8단계 골격

> **상황**: 7개 서비스를 같은 리듬으로 처리하기 위한 체크 순서. 실기 서술형에서 "서비스 구축 절차를 순서대로" 로 그대로 나오는 흐름이다.

| 단계 | 할 일 | 대표 명령 |
| --- | --- | --- |
| 1 | 패키지 설치 | `dnf install -y <패키지>` |
| 2 | 설정 파일 백업 → 편집 | `cp <conf> <conf>.orig` → `vi <conf>` |
| 3 | **문법 검사** | `apachectl configtest` · `named-checkconf` · `testparm` · `postfix check` · `dhcpd -t` |
| 4 | 기동 + 부팅 등록 | `systemctl enable --now <유닛>` |
| 5 | 리스닝 확인 | `ss -tulnp \| grep <포트>` |
| 6 | 방화벽 개방 | `firewall-cmd --permanent --add-service=<이름>` → `--reload` |
| 7 | SELinux 조정 | `semanage fcontext` + `restorecon` / `setsebool -P` |
| 8 | **클라이언트 접속 검증** + 로그 확인 | `curl` · `dig` · `showmount` · `smbclient` · `lftp` · `mail` · `lpstat` |

- 순서 함정: **설정 편집 → 문법 검사 → 재시작** 이지 "재시작 → 확인" 이 아님. 문법 오류 상태로 restart 하면 서비스가 죽은 채 멈춤
- 3단계 문법 검사는 서비스별로 도구가 다름 — 아래 표는 통째로 암기 대상

| 서비스 | 문법 검사 명령 | 정상 출력 |
| --- | --- | --- |
| Apache | `httpd -t` / `apachectl configtest` | `Syntax OK` |
| BIND | `named-checkconf` / `named-checkzone <존> <파일>` | 무출력 / `OK` |
| Samba | `testparm` | `Loaded services file OK.` |
| Postfix | `postfix check` | 무출력 |
| Dovecot | `doveconf -n` (설정 덤프, 오류 시 메시지) | 설정 요약 |
| NFS | `exportfs -ra` (오류 시 메시지) | 무출력 |
| DHCP | `dhcpd -t -cf /etc/dhcp/dhcpd.conf` | `Configuration file errors encountered -- exiting` 가 안 나오면 정상 |

> 📝 **시험 포인트**: 필기 FULL r02-70 — "Apache 설정 문법 검사" 의 정답은 `apachectl configtest`(= `httpd -t`), 오답 선지 `apachectl graceful`(무중단 재시작) · `httpd -l`(정적 모듈 목록) · `apachectl fullstatus`(상태 페이지) 구분.

---

## 2. Apache httpd — 사내 웹

### 2-1. 설치와 파일 구조 파악

> **상황**: 사내 인트라넷 웹을 올린다. 기본 사이트(`/var/www/html`)와 개발팀용 가상호스트(`/srv/www/intranet`) 두 개를 운영할 예정이므로, 먼저 패키지가 깔아 놓는 디렉터리 구조를 파악한다.

```bash
dnf install -y httpd httpd-tools mod_ssl
rpm -qi httpd | head -5                      # 버전·릴리스 확인
rpm -ql httpd | grep -E '^/etc/httpd' | head -20   # 설정 경로만 추림
ls -l /etc/httpd/                            # ServerRoot 내부
ls /etc/httpd/conf.d/                        # 드롭인 설정 (mod_ssl 이 ssl.conf 추가)
ls -ld /var/www/html /var/log/httpd
```

- `httpd` : Apache HTTP Server 본체
- `httpd-tools` : `htpasswd`(인증 계정 파일), `ab`(부하 테스트), `htdbm`, `logresolve` 등 부속 도구
- `mod_ssl` : HTTPS(TLS) 모듈 + `/etc/httpd/conf.d/ssl.conf` 기본 설정 제공
- `rpm -qi` : 패키지 정보 (**i**nfo) / `rpm -ql` : 설치 파일 목록 (**l**ist)

| 경로 | 성격 | 비고 |
| --- | --- | --- |
| `/etc/httpd` | **ServerRoot** | 아래 `logs`·`modules`·`run` 은 심볼릭 링크 |
| `/etc/httpd/conf/httpd.conf` | 주 설정 파일 | 필기 최빈출 |
| `/etc/httpd/conf.d/*.conf` | 드롭인 설정 | `IncludeOptional` 로 **알파벳 순** 포함 |
| `/etc/httpd/conf.modules.d/*.conf` | 모듈 적재(`LoadModule`) | MPM 선택도 여기(`00-mpm.conf`) |
| `/var/www/html` | 기본 **DocumentRoot** | |
| `/var/log/httpd/{access_log,error_log}` | 로그 | `/etc/httpd/logs` → 여기로 링크 |
| `/usr/sbin/httpd`, `/usr/sbin/apachectl` | 실행 파일 / 제어 스크립트 | |

**검증**

```bash
rpm -qf /etc/httpd/conf/httpd.conf     # 이 파일의 소속 패키지 역추적
ls -l /etc/httpd/logs /etc/httpd/modules
```

```text
# rpm -qf /etc/httpd/conf/httpd.conf
httpd-2.4.57-...el9.aarch64
# ls -l /etc/httpd/
lrwxrwxrwx. 1 root root   19  logs -> ../../var/log/httpd
lrwxrwxrwx. 1 root root   29  modules -> ../../usr/lib64/httpd/modules
drwxr-xr-x. 2 root root 4096  conf
drwxr-xr-x. 2 root root 4096  conf.d
drwxr-xr-x. 2 root root 4096  conf.modules.d
# ls /etc/httpd/conf.d/
autoindex.conf  README  ssl.conf  userdir.conf  welcome.conf
```

> 📝 **시험 포인트**: 필기 FULL r06-39, r07-33 — `rpm -qf <파일>` 은 "파일 → 패키지" 역추적, `rpm -ql <패키지>` 는 "패키지 → 파일 목록". 방향을 바꾼 오답 선지가 매번 등장.

### 2-2. httpd.conf 핵심 지시자

> **상황**: 편집 전에 기본값을 읽어 두고, 어떤 지시자가 무엇을 결정하는지 정리한다. 필기 66~70번대는 거의 이 표 안에서 나온다.

```bash
cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd.conf.orig     # 원본 백업 필수
grep -vE '^\s*#|^\s*$' /etc/httpd/conf/httpd.conf | head -40      # 주석 제거 후 실제 설정만
```

- `grep -v` : 일치하지 않는 줄 출력 (in**v**ert), `-E` : 확장 정규식 (**E**xtended)

| 지시자 | 기본값(Rocky 9) | 의미 |
| --- | --- | --- |
| `ServerRoot` | `/etc/httpd` | 설정·로그·모듈 경로의 **기준 디렉터리**. DocumentRoot 와 혼동 금지 |
| `Listen` | `80` | 수신 포트(주소 지정 가능: `Listen 192.168.64.10:80`). 여러 줄 가능 |
| `User` / `Group` | `apache` / `apache` | 자식 프로세스 실행 계정 — root 로 기동 후 권한 하향 |
| `ServerAdmin` | `root@localhost` | 오류 페이지에 노출되는 관리자 메일 주소 |
| `ServerName` | (주석) | 서버가 자신을 식별하는 이름·포트. 미설정 시 기동 경고 |
| `DocumentRoot` | `/var/www/html` | **웹 문서 최상위 디렉터리** |
| `<Directory 경로>` | — | 디렉터리 단위 접근·동작 규칙 블록 |
| `Options` | `Indexes FollowSymLinks` | `Indexes`(목록 자동 생성) `FollowSymLinks`(심볼릭 링크 추적) `ExecCGI`(CGI 실행) `MultiViews` `None` |
| `AllowOverride` | `None` | `.htaccess` 로 재정의 허용 범위 — `None`(무시) `All`(전부) `AuthConfig`(인증만) `Limit` `FileInfo` `Indexes` |
| `Require` | `all granted` | 2.4 접근 제어 — `all granted`/`all denied`/`ip 192.168.64.0/24`/`valid-user`/`user dev1`/`group devteam` |
| `DirectoryIndex` | `index.html` | 디렉터리 요청 시 찾을 **기본 문서** 목록(왼쪽 우선) |
| `ErrorLog` | `logs/error_log` | 오류 로그. 상대경로는 ServerRoot 기준 |
| `LogLevel` | `warn` | `debug info notice warn error crit alert emerg` |
| `LogFormat` / `CustomLog` | `combined` | 접근 로그 형식·경로 |
| `IncludeOptional` | `conf.d/*.conf` | 드롭인 포함(파일 없어도 오류 아님). `Include` 는 없으면 오류 |
| `KeepAlive` | `Off`(Rocky 기본) | 연결 재사용 여부 |
| `MaxKeepAliveRequests` | `100` | 한 연결에서 처리할 최대 요청 수 |
| `KeepAliveTimeout` | `5` | 다음 요청 대기 시간(초) |
| `Timeout` | `60` | 요청·응답 무응답 허용 시간(초) |
| `EnableSendfile` | `on` | 커널 sendfile 사용 |
| `AddDefaultCharset` | `UTF-8` | 기본 문자셋 |

- **2.2 → 2.4 접근 제어 문법 변화** (오답 단골)

| Apache 2.2 | Apache 2.4 |
| --- | --- |
| `Order allow,deny` + `Allow from all` | `Require all granted` |
| `Order deny,allow` + `Deny from all` | `Require all denied` |
| `Allow from 192.168.0.0/24` | `Require ip 192.168.0.0/24` |

**검증**

```bash
grep -nE '^(ServerRoot|Listen|User|Group|ServerAdmin|DocumentRoot|DirectoryIndex|ErrorLog|LogLevel|CustomLog|IncludeOptional)' /etc/httpd/conf/httpd.conf
```

```text
31:ServerRoot "/etc/httpd"
45:Listen 80
66:User apache
67:Group apache
86:ServerAdmin root@localhost
124:DocumentRoot "/var/www/html"
...
353:IncludeOptional conf.d/*.conf
```

> 📝 **시험 포인트**: 필기 FULL r01-66/67, r02-66, r04-67, r05-66/69, r07-69, r10-67 — `ServerRoot`(설정 기준 경로) vs `DocumentRoot`(문서 루트) vs `ServerName`(서버 이름) vs `DirectoryIndex`(기본 문서) 를 바꿔 놓은 선지가 반복. `ServerName` 을 "관리자 이메일" 로 설명하면 오답(그건 `ServerAdmin`).

### 2-3. MPM 확인 — httpd -V · httpd -M

> **상황**: 성능·튜닝 문제에 대비해 현재 서버가 어떤 MPM(다중 처리 모듈)으로 도는지, ssl 모듈이 적재됐는지 확인한다.

```bash
httpd -V                          # 컴파일·실행 정보 (Server MPM 포함)
httpd -M | head -20               # 적재된 모듈 목록
httpd -M | grep -E 'mpm|ssl|rewrite|auth_basic|authz_core'
httpd -l                          # 바이너리에 정적 링크된 모듈만
cat /etc/httpd/conf.modules.d/00-mpm.conf | grep -v '^#'
```

- `-V` : 버전 + 컴파일 설정 상세 (**V**ersion, 대문자)
- `-M` : 적재된 **M**odule 목록 (동적 포함) — `shared` 표시
- `-l` : 정적으로 **l**inked 된 모듈만. 동적 모듈은 안 나옴 → "모듈 확인" 문제의 오답 선지
- `-t` : 문법 검사 (**t**est)

| MPM | 방식 | 특징 |
| --- | --- | --- |
| `prefork` | 요청당 **프로세스** | 스레드 비안전 모듈(구 mod_php) 호환, 메모리 많이 씀. `StartServers`·`MinSpareServers`·`MaxSpareServers`·`MaxRequestWorkers` |
| `worker` | 프로세스 + **스레드** 혼합 | 메모리 절약, `ThreadsPerChild`·`ServerLimit` |
| `event` | worker + **비동기 연결 유지 처리** | **Rocky 9 기본**. KeepAlive 연결을 전용 스레드로 분리 |

**검증**

```bash
httpd -V | grep -i 'server mpm'
ls -l /etc/httpd/modules/mod_ssl.so
```

```text
# httpd -V
Server version: Apache/2.4.57 (Rocky Linux)
Server MPM:     event
 -D HTTPD_ROOT="/etc/httpd"
 -D SERVER_CONFIG_FILE="conf/httpd.conf"
# httpd -M | grep -E 'mpm|ssl'
 mpm_event_module (shared)
 ssl_module (shared)
```

> 📝 **시험 포인트**: 필기 FULL r02-67(prefork 의 `StartServers`), r03-66, r08-68, r10-66 — MPM 3종 특징 구분. r08-68 은 `httpd -M | grep ssl` 결과가 `(shared)` 이면 **동적 모듈**이라는 해석 문제.

### 2-4. ServerName·Listen 8080 추가 → 문법 검사

> **상황**: 기동 시 "could not reliably determine the server's FQDN" 경고를 없애고, 관리용 보조 포트 8080 을 추가로 연다.

```bash
vi /etc/httpd/conf/httpd.conf
```

```apache
# 45행 부근 — 기존 Listen 80 아래에 추가
Listen 80
Listen 8080

# 96행 부근 — 주석 처리된 ServerName 을 실제 값으로 교체
ServerName srv01.lab.local:80
```

- `sed` 로 처리해도 동일 (실기 r06-1 유형)

```bash
sed -i 's/^Listen 80$/Listen 80\nListen 8080/' /etc/httpd/conf/httpd.conf
sed -i 's/^#ServerName www.example.com:80/ServerName srv01.lab.local:80/' /etc/httpd/conf/httpd.conf
```

- `sed -i` : 파일을 직접(**i**n-place) 수정 — 백업하려면 `-i.bak`
- `s/찾을것/바꿀것/` : 치환. `\n` 으로 줄 추가

**검증**

```bash
grep -n '^Listen\|^ServerName' /etc/httpd/conf/httpd.conf
apachectl configtest          # = httpd -t
```

```text
# grep -n '^Listen\|^ServerName' /etc/httpd/conf/httpd.conf
45:Listen 80
46:Listen 8080
96:ServerName srv01.lab.local:80
# apachectl configtest
Syntax OK
```

> 📝 **시험 포인트**: 실기 r06-1 은 `sed -i 's/Listen 80/Listen 8080/' /etc/httpd/conf/httpd.conf` 형태로 "in-place 치환" 을 묻는다. `-i` 없으면 화면에만 출력되고 파일은 그대로 — 최빈출 함정.

### 2-5. 기동·부팅 등록·방화벽 개방

> **상황**: 설정이 통과했으니 서비스를 올리고, 재부팅 후에도 뜨도록 등록한 뒤 방화벽을 연다.

```bash
systemctl enable --now httpd
systemctl status httpd --no-pager
ss -tlnp | grep -E ':80|:8080|:443'
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload
```

- `enable --now` : 부팅 등록(**enable**) + 즉시 기동(**now**) 을 한 번에
- `--permanent` : 영구 규칙 파일에 기록 — **런타임에는 즉시 반영 안 됨**, `--reload` 필요
- `--add-service=http` : `/usr/lib/firewalld/services/http.xml` 이 정의한 80/TCP 개방
- `--add-port=8080/tcp` : firewalld 에 서비스 정의가 없는 포트는 포트로 직접 개방
- firewalld 존·리치룰 등 상세는 → [[10-security-firewall-selinux]]

⚠️ `Listen 8080` 은 SELinux 포트 라벨과 충돌할 수 있음. 기동이 `Permission denied (13): AH00072: make_sock: could not bind to address [::]:8080` 로 실패하면 아래를 확인.

```bash
semanage port -l | grep -E 'http_port_t|http_cache_port_t'
# 8080 이 http_port_t 에 없으면 라벨 추가
semanage port -a -t http_port_t -p tcp 8080
```

- `semanage port -a -t <타입> -p tcp <포트>` : 포트에 SELinux 타입 라벨 부여 (**a**dd)
- 이미 존재하는 라벨을 바꿀 때는 `-m`(**m**odify)

**검증**

```bash
systemctl is-enabled httpd; systemctl is-active httpd
ss -tlnp | grep httpd
firewall-cmd --list-services; firewall-cmd --list-ports
curl -I http://127.0.0.1
curl -I http://127.0.0.1:8080
```

```text
# systemctl is-enabled httpd
enabled
# ss -tlnp | grep httpd
LISTEN 0  511  *:80    *:*  users:(("httpd",pid=...,fd=4))
LISTEN 0  511  *:8080  *:*  users:(("httpd",pid=...,fd=6))
# firewall-cmd --list-services
cockpit dhcpv6-client http https ssh
# firewall-cmd --list-ports
8080/tcp
# curl -I http://127.0.0.1
HTTP/1.1 403 Forbidden
Server: Apache/2.4.57 (Rocky Linux)
Content-Type: text/html; charset=UTF-8
```

- 문서가 없으면 `welcome.conf` 의 테스트 페이지가 **403** 으로 응답 — 정상 동작. 2-6 에서 200 으로 바꿈
- `curl -I` : 응답 **헤더만** 요청(HEAD) (**I** = head)

> 📝 **시험 포인트**: 필기 FULL r06-37, r08-44 — "부팅 등록 + 즉시 기동" 은 `systemctl enable --now httpd` 한 줄. r10-36 은 "`enable` 만 하면 지금 시작되지는 않는다" 가 정답 선지. 실기 r02-9/r05-4 는 `firewall-cmd --permanent --add-service=http` 뒤 `--reload` 를 빠뜨리면 감점.

### 2-6. 기본 문서 작성 → 200 검증

> **상황**: 403 을 없애고 실제 페이지를 띄운다. 기본 사이트는 "이 서버가 살아 있다" 를 알리는 최소 페이지로 둔다.

```bash
cat > /var/www/html/index.html <<'EOF'
<!doctype html>
<html><head><meta charset="utf-8"><title>srv01</title></head>
<body><h1>srv01.lab.local default site</h1><p>LAB 09 Apache OK</p></body></html>
EOF
ls -lZ /var/www/html/index.html          # 소유·권한 + SELinux 컨텍스트
```

- `cat > 파일 <<'EOF' … EOF` : 히어독으로 파일 생성. 구분자를 따옴표로 감싸면 `$`·백틱을 **치환하지 않고** 그대로 기록
- `ls -Z` : SELinux 컨텍스트 표시 (**Z**)

**검증**

```bash
curl -I http://127.0.0.1
curl -s http://127.0.0.1 | head -3
curl -s http://192.168.64.10:8080 | grep -o 'default site'
```

```text
# curl -I http://127.0.0.1
HTTP/1.1 200 OK
Server: Apache/2.4.57 (Rocky Linux)
Content-Length: 143
Content-Type: text/html; charset=UTF-8
# ls -lZ /var/www/html/index.html
-rw-r--r--. 1 root root unconfined_u:object_r:httpd_sys_content_t:s0 143 ... index.html
# curl -s http://192.168.64.10:8080 | grep -o 'default site'
default site
```

- `/var/www/html` 아래 파일은 **자동으로 `httpd_sys_content_t`** 컨텍스트를 받음 — 2-8 의 `/srv/www` 와 대비되는 지점
- `curl -s` : 진행 표시 숨김 (**s**ilent)

> 📝 **시험 포인트**: HTTP 상태 코드 — 200(정상) · 301/302(리다이렉트) · 401(인증 필요) · 403(권한 거부·목록 차단) · 404(문서 없음) · 500(서버 오류). 필기 FULL r08-67 access_log 해석 문제에서 상태 코드 필드 위치와 함께 출제.

### 2-7. 이름 기반 가상호스트 — intranet.lab.local

> **상황**: IP 하나로 사이트 두 개를 서비스한다. 개발팀 인트라넷은 `/srv/www/intranet` 에 두고 `intranet.lab.local` 로 접속하게 만든다.

⚠️ **주의 — 첫 `<VirtualHost>` 를 정의하는 순간 동작이 바뀐다.** Apache 2.4 는 어떤 포트에 이름 기반 가상호스트가 하나라도 정의되면, 그 포트의 요청은 **전부 가상호스트 매칭으로 처리**되고 주 설정의 `DocumentRoot` 는 더 이상 쓰이지 않는다. 매칭되지 않는 요청은 **가장 먼저 읽힌 vhost**(= `conf.d` 알파벳 순 첫 번째)가 받는다. 그래서 기본 사이트도 vhost 로 명시해야 한다.

```bash
mkdir -p /srv/www/intranet
cat > /srv/www/intranet/index.html <<'EOF'
<!doctype html>
<html><head><meta charset="utf-8"><title>intranet</title></head>
<body><h1>DEV TEAM INTRANET</h1><p>vhost intranet.lab.local</p></body></html>
EOF

# ① 기본(fallback) 가상호스트 — 파일명 00- 으로 시작해 가장 먼저 읽히게 함
cat > /etc/httpd/conf.d/00-default.conf <<'EOF'
<VirtualHost *:80>
    ServerName srv01.lab.local
    DocumentRoot /var/www/html
    ErrorLog logs/default-error_log
    CustomLog logs/default-access_log combined
</VirtualHost>
EOF

# ② 인트라넷 가상호스트
cat > /etc/httpd/conf.d/intranet.conf <<'EOF'
<VirtualHost *:80>
    ServerName  intranet.lab.local
    ServerAlias intra.lab.local www.lab.local
    DocumentRoot /srv/www/intranet
    DirectoryIndex index.html

    <Directory "/srv/www/intranet">
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog  logs/intranet-error_log
    CustomLog logs/intranet-access_log combined
</VirtualHost>
EOF
```

| 지시자 | 의미 |
| --- | --- |
| `<VirtualHost *:80>` | 모든 로컬 주소의 80 포트로 들어온 요청을 이 블록에서 처리 |
| `ServerName` | 이 vhost 를 식별하는 **주 이름** — 요청의 `Host:` 헤더와 대조 |
| `ServerAlias` | 같은 vhost 를 가리키는 **추가 이름**(공백 구분, 와일드카드 `*.lab.local` 가능) |
| `DocumentRoot` | 이 vhost 의 문서 루트 |
| `ErrorLog` / `CustomLog` | vhost 전용 로그 — 상대경로는 ServerRoot(`/etc/httpd`) 기준 → 실제 `/var/log/httpd/` |
| `combined` | 로그 형식 이름 — `common` + Referer + User-Agent |
| `Options -Indexes` | 목록 자동 생성 **끄기**(`-` 는 제거, `+` 는 추가) |

**검증**

```bash
apachectl configtest
httpd -S            # 가상호스트 구성 덤프 — 어떤 vhost 가 default 인지 표시
```

```text
# httpd -S
VirtualHost configuration:
*:80                   is a NameVirtualHost
         default server srv01.lab.local (/etc/httpd/conf.d/00-default.conf:1)
         port 80 namevhost srv01.lab.local (/etc/httpd/conf.d/00-default.conf:1)
         port 80 namevhost intranet.lab.local (/etc/httpd/conf.d/intranet.conf:1)
                 alias intra.lab.local
                 alias www.lab.local
ServerRoot: "/etc/httpd"
...
```

- `httpd -S` : vhost **S**ettings 덤프. "default server" 줄이 매칭 실패 시 응답할 vhost

> 📝 **시험 포인트**: 필기 FULL r01-68, r02-69, r06-66, r10-68 — "IP 하나에 도메인 여러 개" = **이름 기반 가상호스트**, 구분 기준은 요청의 `Host` 헤더(= `ServerName`). "DocumentRoot 는 반드시 같아야 한다" 같은 선지는 오답.

### 2-8. SELinux — /srv/www 컨텍스트 (403 재현 → 해결)

> **상황**: 문서 루트를 `/var/www` 밖(`/srv/www`)으로 옮기면 권한이 정상인데도 403 이 난다. 이 파트에서 **가장 자주 걸리는 함정**이므로 일부러 재현하고 해결한다.

```bash
systemctl reload httpd                                    # 새 vhost 반영
curl -s -o /dev/null -w '%{http_code}\n' -H 'Host: intranet.lab.local' http://127.0.0.1/
ls -ldZ /srv /srv/www /srv/www/intranet                    # 컨텍스트 확인 → var_t
ls -l /srv/www/intranet/index.html                         # 파일 권한은 정상
```

- `-o /dev/null` : 본문 버림, `-w '%{http_code}'` : 상태 코드만 출력 (**w**rite-out)
- `-H 'Host: …'` : 요청 헤더 지정 — DNS 없이 이름 기반 vhost 를 시험하는 표준 방법

```text
# curl ... 
403
# ls -ldZ /srv/www/intranet
drwxr-xr-x. 2 root root unconfined_u:object_r:var_t:s0 ... /srv/www/intranet
```

- `/srv` 는 `var_t` 라벨 → httpd(`httpd_t`)는 `var_t` 를 읽을 권한이 없어 **403**. 파일 권한(`755`)은 멀쩡한데 막히는 게 특징

**원인 확인 — AVC 감사 로그**

```bash
ausearch -m avc -ts recent | tail -20
# setroubleshoot-server 설치 시 사람이 읽는 요약
journalctl -t setroubleshoot --since '-5 min' | tail
```

- `ausearch -m avc` : 감사 로그에서 **AVC**(Access Vector Cache) 거부 기록만 추출 (**m**essage type)
- `-ts recent` : 최근 10분 (**t**ime **s**tart)

```text
type=AVC msg=audit(...): avc:  denied  { getattr } for  pid=... comm="httpd"
  path="/srv/www/intranet/index.html" dev="dm-2" ino=...
  scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:var_t:s0 tclass=file permissive=0
```

**해결 — 파일 컨텍스트 규칙 추가 후 재라벨링**

```bash
semanage fcontext -a -t httpd_sys_content_t "/srv/www(/.*)?"
restorecon -Rv /srv/www
ls -ldZ /srv/www /srv/www/intranet /srv/www/intranet/index.html
```

- `semanage fcontext -a -t <타입> "<정규식>"` : 경로 패턴에 **영구** 컨텍스트 규칙 추가 (**a**dd). `/etc/selinux/targeted/contexts/files/file_contexts.local` 에 기록 → 재부팅·`autorelabel` 후에도 유지
- `"(/.*)?"` : 디렉터리 자신 + 하위 전체를 뜻하는 관용 표현
- `restorecon -Rv` : 규칙대로 실제 라벨을 **재적용** (**R**ecursive, **v**erbose). `chcon` 은 임시(재라벨링 시 소실) → 시험에서 둘의 차이 출제
- SELinux 상세는 → [[10-security-firewall-selinux]]

**검증**

```bash
ls -ldZ /srv/www/intranet
curl -s -o /dev/null -w '%{http_code}\n' -H 'Host: intranet.lab.local' http://127.0.0.1/
curl -s -H 'Host: intranet.lab.local' http://127.0.0.1/ | grep -o 'DEV TEAM INTRANET'
semanage fcontext -l -C            # 로컬(사용자 추가) 규칙만 확인
```

```text
# ls -ldZ /srv/www/intranet
drwxr-xr-x. 2 root root unconfined_u:object_r:httpd_sys_content_t:s0 ... /srv/www/intranet
# curl ...
200
DEV TEAM INTRANET
# semanage fcontext -l -C
SELinux fcontext          type       Context
/srv/www(/.*)?            all files  system_u:object_r:httpd_sys_content_t:s0
```

> 📝 **시험 포인트**: 필기 FULL r06-58 이 정확히 이 상황 — "문서 루트를 `/srv/www` 로 옮겼더니 403, 권한은 정상" 의 정답은 `semanage fcontext -a -t httpd_sys_content_t "/srv/www(/.*)?"` + `restorecon -Rv /srv/www`. r03-57·r09-59 는 `setsebool -P httpd_can_network_connect on` 의 `-P`(**P**ersistent, 재부팅 후 유지) 를 묻는다.

### 2-9. 가상호스트 접속 검증 — Host 헤더·이름·브라우저

> **상황**: `Host` 헤더로는 붙었으니, 이제 진짜 이름 `intranet.lab.local` 로도 붙는지 확인한다. 이름 해석은 `/etc/hosts`(Part 08) 또는 3절에서 만들 DNS 가 담당한다.

```bash
grep intranet /etc/hosts || \
  sed -i 's/^192.168.64.10.*/& intranet.lab.local intra.lab.local www.lab.local/' /etc/hosts
getent hosts intranet.lab.local
curl -s http://intranet.lab.local/ | grep -o 'DEV TEAM INTRANET'
curl -s http://www.lab.local/       | grep -o 'DEV TEAM INTRANET'   # ServerAlias 확인
curl -s http://srv01.lab.local/     | grep -o 'default site'        # 기본 vhost 확인
curl -s -H 'Host: nosuch.lab.local' http://192.168.64.10/ | grep -o 'default site'  # 미매칭 → default
```

- `sed 's/…/& …/'` : `&` 는 매칭된 원문 전체 → 줄 끝에 별칭 덧붙이기
- 마지막 줄: 등록되지 않은 이름은 **첫 번째 vhost(=00-default)** 가 응답 — 2-7 의 주의 사항을 실제로 확인

**macOS(호스트 PC) 브라우저에서 보기**

```bash
# macOS 터미널에서 실행 (게스트가 아님)
sudo sh -c 'echo "192.168.64.10  srv01.lab.local intranet.lab.local www.lab.local" >> /etc/hosts'
dscacheutil -flushcache; sudo killall -HUP mDNSResponder      # DNS 캐시 비우기
open http://intranet.lab.local/
```

- UTM Shared Network(NAT)는 macOS ↔ 게스트 통신이 가능하므로 브라우저로 바로 확인 가능
- 브라우저 캐시 때문에 옛 페이지가 보이면 강제 새로고침(⌘+Shift+R)

**검증**

```bash
curl -sI http://intranet.lab.local/ | head -1
tail -2 /var/log/httpd/intranet-access_log
```

```text
HTTP/1.1 200 OK
192.168.64.10 - - [.../Sep/2026:...] "GET / HTTP/1.1" 200 118 "-" "curl/7.76.1"
```

> 📝 **시험 포인트**: 실기 r06-11 — 가상호스트 블록을 주고 "`ServerName` 과 `DocumentRoot` 의 역할을 포함해 동작을 서술" 하는 문제. `ServerName` = 요청 `Host` 헤더와 대조할 이름, `DocumentRoot` = 그 사이트의 문서 최상위 경로.

### 2-10. 디렉터리 인증 — htpasswd · .htaccess · AllowOverride

> **상황**: 인트라넷 안에 `/secret` 영역을 만들어 개발자 `dev1` 만 보게 한다. `.htaccess` 를 쓰려면 상위에서 `AllowOverride` 로 권한을 내려 줘야 한다는 것을 실증한다.

```bash
mkdir -p /srv/www/intranet/secret
echo '<h1>SECRET AREA</h1>' > /srv/www/intranet/secret/index.html
restorecon -Rv /srv/www

# ① 인증 계정 파일 생성 (최초 1회만 -c)
htpasswd -c /etc/httpd/.htpasswd dev1        # 대화식으로 비밀번호 2회 입력
htpasswd    /etc/httpd/.htpasswd ops1        # 두 번째부터는 -c 없이 (있으면 파일이 새로 덮어써짐 ⚠️)
chown root:apache /etc/httpd/.htpasswd
chmod 640 /etc/httpd/.htpasswd
```

- `htpasswd -c <파일> <사용자>` : 계정 파일 **c**reate — ⚠️ 이미 있는 파일에 `-c` 를 다시 쓰면 **기존 내용 전체 삭제**
- `-b` : 비밀번호를 인자로 직접 전달(**b**atch) — 히스토리에 남으므로 실무 비권장
- `-D` : 사용자 삭제 (**D**elete), `-v` : 비밀번호 검증 (**v**erify)
- 문서 루트 **밖**(`/etc/httpd/`)에 두는 것이 원칙 — 문서 루트 안에 두면 웹으로 내려받힐 수 있음

```bash
# ② .htaccess 로 인증 지정
cat > /srv/www/intranet/secret/.htaccess <<'EOF'
AuthType Basic
AuthName "Dev Team Only"
AuthUserFile /etc/httpd/.htpasswd
Require valid-user
EOF

# ③ 상위 <Directory> 에서 AllowOverride 허용 — 이게 없으면 .htaccess 는 무시됨
sed -i 's|AllowOverride None|AllowOverride AuthConfig|' /etc/httpd/conf.d/intranet.conf
apachectl configtest && systemctl reload httpd
```

| 키 | 의미 |
| --- | --- |
| `AuthType Basic` | HTTP 기본 인증(평문 Base64 — HTTPS 병행 권장). 대안 `Digest` |
| `AuthName "문자열"` | 브라우저 인증 팝업에 표시될 영역(realm) 이름 |
| `AuthUserFile` | `htpasswd` 로 만든 계정 파일 경로 |
| `Require valid-user` | 파일에 등록된 **아무 사용자나** 인증 성공 시 허용 |
| `Require user dev1` | 특정 사용자만 |
| `Require group devteam` + `AuthGroupFile` | 그룹 단위 |

- `AllowOverride` 값: `None`(.htaccess 완전 무시) · `AuthConfig`(인증 지시자만 허용) · `Indexes` · `FileInfo` · `Limit` · `All`(전부)

**검증 — 미인증 401 → 인증 200**

```bash
curl -s -o /dev/null -w 'no-auth=%{http_code}\n'   http://intranet.lab.local/secret/
curl -s -o /dev/null -w 'wrong  =%{http_code}\n' -u dev1:틀린값 http://intranet.lab.local/secret/
curl -s -w '\nauth   =%{http_code}\n' -u dev1 http://intranet.lab.local/secret/   # 비밀번호 대화식 입력
curl -sI http://intranet.lab.local/secret/ | grep -i 'WWW-Authenticate'
```

```text
no-auth=401
wrong  =401
<h1>SECRET AREA</h1>
auth   =200
WWW-Authenticate: Basic realm="Dev Team Only"
```

**AllowOverride None 으로 되돌려 무시되는지 확인**

```bash
sed -i 's|AllowOverride AuthConfig|AllowOverride None|' /etc/httpd/conf.d/intranet.conf
systemctl reload httpd
curl -s -o /dev/null -w 'override-none=%{http_code}\n' http://intranet.lab.local/secret/   # 401 아닌 200 → .htaccess 무시됨
sed -i 's|AllowOverride None|AllowOverride AuthConfig|' /etc/httpd/conf.d/intranet.conf
systemctl reload httpd
```

```text
override-none=200
```

> 📝 **시험 포인트**: 필기 FULL r01-69, r02-68, r05-67/68, r06-68, r08-69 — `.htaccess` 를 쓰려면 `AllowOverride` 가 `None` 이 아니어야 하고, **데몬 재시작 없이 즉시 반영**된다(요청마다 읽음). "수정 후 반드시 재시작해야 한다" 는 오답. `htpasswd -c` 의 `-c` 는 최초 1회.

### 2-11. 디렉터리 목록 차단 — Options -Indexes

> **상황**: `index.html` 이 없는 디렉터리에 접근하면 파일 목록이 그대로 노출된다. 보안 점검 지적 사항이므로 차단한다.

```bash
mkdir -p /srv/www/intranet/files
echo 'note-1' > /srv/www/intranet/files/note1.txt
echo 'note-2' > /srv/www/intranet/files/note2.txt
restorecon -Rv /srv/www

# 먼저 목록이 노출되는지 확인 — 현재 vhost 는 -Indexes 이므로 잠시 켜서 재현
sed -i 's/Options -Indexes/Options +Indexes/' /etc/httpd/conf.d/intranet.conf
systemctl reload httpd
curl -s http://intranet.lab.local/files/ | grep -oE 'note[12]\.txt' | sort -u
```

```text
note1.txt
note2.txt
```

```bash
# 차단
sed -i 's/Options +Indexes/Options -Indexes/' /etc/httpd/conf.d/intranet.conf
apachectl configtest && systemctl reload httpd
```

- `Options -Indexes` : 자동 목록 생성 비활성 → 기본 문서 없으면 **403 Forbidden**
- `Options None` : 모든 옵션 끄기(더 강함). `Options Indexes FollowSymLinks` 처럼 `+/-` 없이 쓰면 **덮어쓰기**, `+/-` 를 붙이면 상위 설정에 **가감**

**검증**

```bash
curl -s -o /dev/null -w 'indexes-off=%{http_code}\n' http://intranet.lab.local/files/
curl -s http://intranet.lab.local/files/note1.txt      # 파일 직접 접근은 여전히 가능
grep -n 'Options' /etc/httpd/conf.d/intranet.conf
```

```text
indexes-off=403
note-1
7:        Options -Indexes +FollowSymLinks
```

> 📝 **시험 포인트**: 필기 FULL r06-69, r09-67 — "index.html 없는 디렉터리에서 파일 목록이 노출된다 → 차단 설정" 의 정답은 `Options -Indexes`. `DirectoryIndex` 를 지우는 것은 오히려 목록 노출을 유발.

### 2-12. 로그 해석 — access_log · error_log

> **상황**: 지금까지의 요청이 어떤 형식으로 기록됐는지 확인하고, combined 형식의 필드 순서를 몸에 익힌다.

```bash
grep -n '^LogFormat\|^CustomLog\|^ErrorLog\|^LogLevel' /etc/httpd/conf/httpd.conf
tail -3 /var/log/httpd/intranet-access_log
tail -5 /var/log/httpd/error_log
ls -l /var/log/httpd/
```

- combined 형식 문자열: `%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"`

| 필드 | 지시자 | 예 | 의미 |
| --- | --- | --- | --- |
| 1 | `%h` | `192.168.64.10` | 클라이언트 **h**ost/IP |
| 2 | `%l` | `-` | identd 사용자 (거의 항상 `-`) |
| 3 | `%u` | `dev1` | HTTP 인증 사용자 (미인증이면 `-`) |
| 4 | `%t` | `[03/Sep/2026:10:22:31 +0900]` | 요청 **t**ime |
| 5 | `"%r"` | `"GET /secret/ HTTP/1.1"` | 요청 라인(메서드·URI·프로토콜) |
| 6 | `%>s` | `200` | 최종 상태 코드 |
| 7 | `%b` | `118` | 응답 **b**ody 바이트 수(0 이면 `-`) |
| 8 | `"%{Referer}i"` | `"-"` | 참조 페이지 |
| 9 | `"%{User-Agent}i"` | `"curl/7.76.1"` | 클라이언트 프로그램 |

- `common` 형식 = 1~7 번까지. **combined = common + Referer + User-Agent**
- `error_log` 형식: `[시각] [모듈:심각도] [pid …] [client IP:포트] 메시지`

**검증**

```bash
curl -s -o /dev/null http://intranet.lab.local/nosuchpage
tail -1 /var/log/httpd/intranet-access_log
tail -1 /var/log/httpd/intranet-error_log
awk '{print $9}' /var/log/httpd/intranet-access_log | sort | uniq -c   # 상태 코드 집계
```

```text
# tail -1 .../intranet-access_log
192.168.64.10 - - [03/Sep/2026:10:25:02 +0900] "GET /nosuchpage HTTP/1.1" 404 196 "-" "curl/7.76.1"
# tail -1 .../intranet-error_log
[Thu Sep 03 ... 2026] [core:info] [pid ...] [client 192.168.64.10:...] AH00128: File does not exist: /srv/www/intranet/nosuchpage
# awk '{print $9}' ... | sort | uniq -c
      3 200
      2 401
      1 403
      1 404
```

> 📝 **시험 포인트**: 필기 FULL r03-68, r08-67 — access_log 한 줄을 주고 "상태 코드/전송 바이트/요청 메서드" 위치를 묻는다. `awk '{print $9}'` 로 상태 코드를 뽑는 것은 실기 단골(9번째 필드).

### 2-13. HTTPS — 자체 서명 인증서와 mod_ssl

> **상황**: 인트라넷을 HTTPS 로도 열어야 한다. 사설 CA 가 없으므로 자체 서명(self-signed) 인증서를 만들어 붙인다.

⚠️ 자체 서명 인증서는 브라우저에서 경고가 뜬다. 실습·내부용 한정이며 공개 서비스에는 사용 금지.

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/pki/tls/private/lab.key \
  -out    /etc/pki/tls/certs/lab.crt \
  -subj "/C=KR/ST=Seoul/O=LAB/CN=intranet.lab.local"
chmod 600 /etc/pki/tls/private/lab.key
ls -lZ /etc/pki/tls/certs/lab.crt /etc/pki/tls/private/lab.key
```

- `req` : 인증서 서명 요청(CSR) 관련 하위 명령
- `-x509` : CSR 대신 **자체 서명 인증서**를 바로 생성
- `-nodes` : 개인키를 암호화하지 않음 (**no DES**) — 없으면 httpd 기동 때마다 암호 입력 필요
- `-days 365` : 유효 기간
- `-newkey rsa:2048` : 새 RSA 2048비트 키를 함께 생성
- `-keyout` / `-out` : 개인키 / 인증서 출력 경로
- `-subj` : 대화식 질문을 건너뛰고 주체(Subject) 지정. **CN 은 접속할 도메인과 일치**해야 이름 검증 통과

```bash
# mod_ssl 기본 설정에서 인증서 경로 두 줄만 교체
cp /etc/httpd/conf.d/ssl.conf /etc/httpd/conf.d/ssl.conf.orig
sed -i \
 -e 's|^SSLCertificateFile .*|SSLCertificateFile /etc/pki/tls/certs/lab.crt|' \
 -e 's|^SSLCertificateKeyFile .*|SSLCertificateKeyFile /etc/pki/tls/private/lab.key|' \
 /etc/httpd/conf.d/ssl.conf
grep -nE '^(Listen|SSLEngine|SSLCertificate|SSLProtocol|DocumentRoot|ServerName)' /etc/httpd/conf.d/ssl.conf
apachectl configtest && systemctl restart httpd
```

| 지시자 | 의미 |
| --- | --- |
| `Listen 443 https` | TLS 수신 포트 |
| `SSLEngine on` | 이 vhost 에서 TLS 활성화 |
| `SSLCertificateFile` | **공개 인증서**(`.crt`) 경로 |
| `SSLCertificateKeyFile` | **개인키**(`.key`) 경로 — 600 권한 필수 |
| `SSLCertificateChainFile` | 중간 CA 체인(자체 서명은 불필요) |
| `SSLProtocol` / `SSLCipherSuite` | 허용 프로토콜 버전·암호 스위트 |

**검증**

```bash
ss -tlnp | grep :443
curl -k -s https://127.0.0.1/ | head -2                 # -k: 인증서 검증 생략
curl -sI https://intranet.lab.local/ --resolve intranet.lab.local:443:192.168.64.10 -k | head -1
openssl s_client -connect 127.0.0.1:443 -servername intranet.lab.local </dev/null 2>/dev/null | \
  openssl x509 -noout -subject -issuer -dates
```

- `-k` : 인증서 검증 생략 (**k** = insecure)
- `openssl s_client -connect` : TLS 핸드셰이크를 직접 수행해 서버 인증서 확인
- `-servername` : SNI(Server Name Indication) 전송 — 이름 기반 HTTPS vhost 구분에 필요
- `openssl x509 -noout -subject -issuer -dates` : 인증서에서 주체·발급자·유효기간만 추출

```text
LISTEN 0 511 *:443 *:* users:(("httpd",pid=...,fd=8))
subject=C = KR, ST = Seoul, O = LAB, CN = intranet.lab.local
issuer=C = KR, ST = Seoul, O = LAB, CN = intranet.lab.local
notBefore=Sep  3 ... 2026 GMT
notAfter=Sep  3 ... 2027 GMT
```

- `subject` 와 `issuer` 가 **같으면 자체 서명** — 시험에서 자주 묻는 판별법

**HTTP → HTTPS 강제 전환(참고)**

```apache
# /etc/httpd/conf.d/intranet.conf 의 *:80 블록 안에 추가하면 상시 리다이렉트
Redirect permanent / https://intranet.lab.local/
```

> 📝 **시험 포인트**: 필기 FULL r03-67, r06-70, r09-70/71 — HTTPS 구성은 "mod_ssl 설치 + 443 vhost + `SSLEngine on` + `SSLCertificateFile`/`SSLCertificateKeyFile`". `SSLCertificateKeyFile` = **개인키**(공개 인증서 아님). 80→443 전환은 `Redirect permanent`.

### 2-14. 가상호스트 3방식 비교

> **상황**: 이름 기반만 실습했으므로 나머지 두 방식을 개념으로 정리한다. 필기에서 셋을 섞어 출제한다.

| 방식 | 구분 기준 | 설정 예 | 특징 |
| --- | --- | --- | --- |
| **이름 기반**(name-based) | 요청의 `Host:` 헤더 = `ServerName` | `<VirtualHost *:80>` + `ServerName a.example.com` | IP 1개로 도메인 다수. **가장 일반적**. HTTPS 는 SNI 필요 |
| **IP 기반**(IP-based) | 접속한 **서버 IP** | `<VirtualHost 192.168.64.10:80>` / `<VirtualHost 192.168.64.11:80>` | 서버에 IP 여러 개 필요. SNI 미지원 구형 클라이언트 대응 |
| **포트 기반**(port-based) | 접속한 **포트** | `Listen 8080` + `<VirtualHost *:8080>` | IP 1개·이름 없이 분리. URL 에 포트 표기 필요 |

```apache
# IP 기반 예 (※ 미실행 — 이 VM 은 IP 1개)
<VirtualHost 192.168.64.10:80>
    ServerName site-a.lab.local
    DocumentRoot /srv/www/site-a
</VirtualHost>

# 포트 기반 예 — 2-4 에서 연 8080 을 쓰면 실제 동작
<VirtualHost *:8080>
    ServerName srv01.lab.local
    DocumentRoot /srv/www/intranet
</VirtualHost>
```

**검증** (포트 기반은 실제로 확인 가능)

```bash
cat > /etc/httpd/conf.d/port8080.conf <<'EOF'
<VirtualHost *:8080>
    ServerName srv01.lab.local
    DocumentRoot /srv/www/intranet
    <Directory "/srv/www/intranet">
        Require all granted
    </Directory>
</VirtualHost>
EOF
apachectl configtest && systemctl reload httpd
curl -s http://127.0.0.1:8080/ | grep -o 'DEV TEAM INTRANET'
httpd -S | grep -A2 '\*:8080'
```

```text
DEV TEAM INTRANET
*:8080                 is a NameVirtualHost
         default server srv01.lab.local (/etc/httpd/conf.d/port8080.conf:1)
```

> 📝 **시험 포인트**: "IP 하나에 여러 도메인" = 이름 기반, "도메인마다 IP 별도" = IP 기반. 이름 기반 HTTPS 는 **SNI** 없이는 불가능하다는 점도 출제 포인트.

### 2-15. 부하 테스트 — ab

> **상황**: 웹 성능 점검 도구를 익힌다. `httpd-tools` 에 포함된 `ab`(Apache Bench)로 간단한 부하를 준다.

```bash
ab -n 100 -c 10 http://127.0.0.1/
ab -n 200 -c 20 -k http://intranet.lab.local/       # KeepAlive 사용
```

- `-n <횟수>` : 총 요청 수 (**n**umber of requests)
- `-c <동시>` : 동시 연결 수 (**c**oncurrency)
- `-k` : HTTP **K**eepAlive 사용
- `-t <초>` : 지정 시간 동안 반복 (**t**imelimit)
- `-p <파일>` `-T <타입>` : POST 본문·Content-Type

**검증**

```bash
ab -n 100 -c 10 http://127.0.0.1/ 2>/dev/null | grep -E 'Requests per second|Failed requests|Time per request|Complete requests'
tail -1 /var/log/httpd/default-access_log
```

```text
Complete requests:      100
Failed requests:        0
Requests per second:    ...  [#/sec] (mean)
Time per request:       ...  [ms] (mean)
```

> 📝 **시험 포인트**: `ab` 는 Apache 부속 벤치마크 도구. `-n`(총 요청) 과 `-c`(동시) 를 뒤바꾼 선지가 오답으로 출제. 시스템 부하 재현은 Part 06 의 `stress-ng`, 웹 계층 부하는 `ab` 로 역할 구분.

### 2-16. Nginx 대응표 (※ 미실행)

> **상황**: 필기에서 Apache ↔ Nginx 비교가 나온다. 이 VM 은 80 포트를 httpd 가 쓰고 있어 설치·기동하지 않고 대응 관계만 정리한다.

| 항목 | Apache httpd | Nginx |
| --- | --- | --- |
| 주 설정 파일 | `/etc/httpd/conf/httpd.conf` | `/etc/nginx/nginx.conf` |
| 드롭인 | `/etc/httpd/conf.d/*.conf` | `/etc/nginx/conf.d/*.conf` |
| 가상호스트 블록 | `<VirtualHost *:80>` | `server { … }` |
| 수신 포트 | `Listen 80` | `listen 80;` |
| 문서 루트 | `DocumentRoot /var/www/html` | `root /usr/share/nginx/html;` |
| 서버 이름 | `ServerName a.example.com` | `server_name a.example.com;` |
| 기본 문서 | `DirectoryIndex index.html` | `index index.html;` |
| 경로별 규칙 | `<Directory>` / `<Location>` | `location /path { … }` |
| 문법 검사 | `apachectl configtest` | `nginx -t` |
| 무중단 반영 | `apachectl graceful` | `nginx -s reload` |
| 처리 모델 | MPM(prefork/worker/**event**) | **이벤트 기반 비동기** 단일 구조 |
| `.htaccess` | 지원 | **미지원**(설계상 없음) |

- Nginx 는 정적 파일·리버스 프록시에 강하고, Apache 는 모듈 생태계·`.htaccess` 유연성이 강점
- Apache 2.4 의 `event` MPM 은 Nginx 의 비동기 모델을 상당 부분 흡수

> 📝 **시험 포인트**: 필기 FULL r03-69 — "Nginx 는 `.htaccess` 를 지원한다"(오답), "Apache 에는 이벤트 기반 처리가 없다"(오답). 정답은 "Nginx 는 이벤트 기반 비동기 구조" 계열.

### 2-17. apachectl 서브명령 정리

> **상황**: 설정을 바꿀 때 `restart` 와 `graceful` 중 무엇을 쓸지 판단 기준을 세운다.

```bash
apachectl configtest         # 문법 검사 (= httpd -t)
apachectl graceful           # 접속 유지한 채 설정 재적용 (기존 연결은 처리 완료 후 교체)
apachectl restart            # 즉시 재시작 (진행 중 연결 끊김)
apachectl -S                 # 가상호스트 덤프
apachectl status             # mod_status 기반 상태 (텍스트 브라우저 필요)
systemctl reload httpd       # 내부적으로 graceful 수행
systemctl restart httpd
```

| 명령 | 동작 | 진행 중 연결 |
| --- | --- | --- |
| `apachectl configtest` / `httpd -t` | 문법만 검사, 서비스 무영향 | 유지 |
| `apachectl graceful` / `systemctl reload` | 자식 프로세스를 순차 교체하며 설정 반영 | **유지** |
| `apachectl restart` / `systemctl restart` | 부모까지 재기동 | **끊김** |
| `apachectl graceful-stop` | 처리 완료 후 정지 | 유지 후 종료 |
| `apachectl -k start\|stop\|restart` | 하위 호환 표기 | — |

**검증**

```bash
systemctl show -p MainPID --value httpd    # graceful 전
apachectl graceful
systemctl show -p MainPID --value httpd    # 값이 같으면 무중단 교체 성공
journalctl -u httpd -n 5 --no-pager
```

```text
1234
1234
```

> 📝 **시험 포인트**: 필기 FULL r04-66 — "접속 중인 클라이언트를 끊지 않고 설정 반영" = `apachectl graceful`. `restart` 는 끊김, `configtest` 는 검사만.

---

## 3. BIND — 내부 DNS

### 3-1. 설치와 파일 구조

> **상황**: `/etc/hosts` 를 모든 PC 에 복사하는 대신 사내 DNS 를 세운다. `lab.local` 정방향 존과 `192.168.64.0/24` 역방향 존을 이 서버가 권한(master) 서버로 관리한다.

```bash
dnf install -y bind bind-utils
rpm -ql bind | grep -E '^/etc|^/var/named' | head -15
ls -l /etc/named.conf /etc/rndc.key
ls -l /var/named/
```

- `bind` : 네임서버 데몬 `named` + 설정 파일
- `bind-utils` : 조회 도구 `dig` `host` `nslookup` (클라이언트 전용 — 서버 없이도 설치 가능)

| 경로 | 역할 |
| --- | --- |
| `/etc/named.conf` | 주 설정 — options + zone 선언 |
| `/etc/named.rfc1912.zones` | localhost·역방향 기본 존 선언(별도 파일, `include` 로 포함) |
| `/etc/rndc.key` | `rndc` 제어 채널 공유 키(설치 시 자동 생성) |
| `/var/named/` | **존 파일 디렉터리**(`directory` 지시자 값) |
| `/var/named/named.ca` | 루트 힌트(`type hint`) |
| `/var/named/slaves/` | 슬레이브가 받아 온 존 저장 위치(쓰기 가능) |
| `/var/log/messages`, `journalctl -u named` | 로그 |

**검증**

```bash
rpm -qf /etc/named.conf
named -v
ls -ld /var/named /var/named/slaves
```

```text
# rpm -qf /etc/named.conf
bind-9.16...el9.aarch64
# named -v
BIND 9.16.23-RH (Extended Support Version) <id:...>
# ls -ld /var/named
drwxrwx---. 5 root named 4096 ... /var/named
```

- `/var/named` 는 `root:named 0770` — 존 파일은 named 가 **읽을 수** 있어야 함

> 📝 **시험 포인트**: 데몬은 `named`, 패키지는 `bind`, 조회 도구는 `bind-utils`. 서비스 유닛명도 `named` — `systemctl start bind` 는 존재하지 않는 오답.

### 3-2. /etc/named.conf — options 블록

> **상황**: 기본 설정은 localhost 만 응답하도록 잠겨 있다. 사내 대역이 질의할 수 있게 열고, 외부 도메인은 게이트웨이로 넘기도록 포워더를 지정한다.

```bash
cp /etc/named.conf /etc/named.conf.orig
vi /etc/named.conf
```

```
options {
        listen-on port 53 { 127.0.0.1; 192.168.64.10; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";

        allow-query     { localhost; 192.168.64.0/24; };
        allow-transfer  { none; };
        recursion yes;
        forwarders      { 192.168.64.1; };
        forward first;

        dnssec-validation no;

        managed-keys-directory "/var/named/dynamic";
        geoip-directory "/usr/share/GeoIP";
        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
        include "/etc/crypto-policies/back-ends/bind.config";
};
```

| 옵션 | 의미 |
| --- | --- |
| `listen-on port 53 { … }` | **질의를 받을 자신의 주소** 목록. `any` 는 전체 인터페이스 |
| `directory "/var/named"` | 존 파일 상대 경로의 기준 디렉터리 |
| `allow-query { … }` | **질의를 허용할 클라이언트** 대역. `any`/`none`/`localhost`/네트워크 |
| `allow-transfer { … }` | **존 전송(AXFR)을 허용할 대상** — 기본 `none` 으로 잠가야 존 정보 유출 방지 |
| `recursion yes\|no` | 재귀 질의 수행 여부. 권한 서버만이면 `no` 권장(오픈 리졸버 방지) |
| `forwarders { IP; }` | 자신이 모르는 질의를 넘길 상위 DNS |
| `forward first\|only` | `first`(포워더 실패 시 직접 조회) / `only`(포워더만 사용) |
| `dnssec-validation yes\|no` | DNSSEC 검증. 내부 사설 존·비검증 포워더 환경에서는 `no` |
| `allow-update { none; }` | 동적 갱신 허용 대상 (존 블록에도 지정 가능) |

- 값 목록은 중괄호 `{ }` 로 감싸고 **각 항목마다 세미콜론**, 블록 끝에도 세미콜론 — 문법 오류 1순위
- 주석은 `//`, `#`, `/* */` 세 가지 모두 사용 가능

**검증**

```bash
named-checkconf                       # 무출력이면 정상
named-checkconf -p | head -25         # 파싱 결과를 정규화해서 출력
grep -nE 'listen-on|allow-query|forwarders|recursion|dnssec-validation|allow-transfer' /etc/named.conf
```

```text
# named-checkconf
(무출력)
# named-checkconf -p | head
options {
	directory "/var/named";
	dnssec-validation no;
	forwarders {
		192.168.64.1;
	};
	listen-on port 53 {
		127.0.0.1/32;
		192.168.64.10/32;
	};
...
```

- `named-checkconf -p` : 파싱된 설정을 **p**rint — 어떤 값이 실제로 적용됐는지 확인

> 📝 **시험 포인트**: 필기 FULL r02-73, r05-70/71 — `directory` 는 존 파일 위치, `allow-query` 는 질의 허용 대역, `forwarders` 는 상위 위임 대상. "`forwarders` 가 존 전송 허용 대상" 은 오답(그건 `allow-transfer`).

### 3-3. zone 선언 — 정방향·역방향

> **상황**: 이 서버가 `lab.local` 과 `64.168.192.in-addr.arpa` 두 존의 마스터임을 선언한다.

```bash
cat >> /etc/named.conf <<'EOF'

// ===== LAB zones =====
zone "lab.local" IN {
        type master;
        file "lab.local.zone";
        allow-update { none; };
        allow-transfer { none; };
};

zone "64.168.192.in-addr.arpa" IN {
        type master;
        file "192.168.64.rev";
        allow-update { none; };
        allow-transfer { none; };
};
EOF
```

| 키 | 의미 |
| --- | --- |
| `zone "<존 이름>" IN` | 관리할 존 이름. `IN` 은 인터넷 클래스 |
| `type master` | 이 서버가 **권한 원본**. 다른 값: `slave`(복제, `masters { IP; };` 필수) · `hint`(루트 힌트) · `forward` |
| `file "…"` | 존 파일명 — `directory` 기준 **상대 경로** |
| `allow-update` | 동적 DNS 갱신 허용 대상. 정적 존은 `none` |
| `allow-transfer` | AXFR/IXFR 허용 대상 |
| `also-notify { IP; }` | 존 변경 시 NOTIFY 를 추가로 보낼 서버 |

- **역방향 존 이름 만드는 법**: 네트워크 주소의 옥텟을 **거꾸로** 쓰고 `.in-addr.arpa` 를 붙임
  - `192.168.64.0/24` → `64.168.192.in-addr.arpa`
  - `10.0.0.0/8` → `10.in-addr.arpa`
  - IPv6 는 `.ip6.arpa`

**검증**

```bash
named-checkconf
named-checkconf -z 2>&1 | head        # 선언된 모든 존 파일까지 함께 검사
grep -n 'zone "' /etc/named.conf
```

```text
# grep -n 'zone "' /etc/named.conf
...:zone "." IN {
...:zone "lab.local" IN {
...:zone "64.168.192.in-addr.arpa" IN {
# named-checkconf -z
zone lab.local/IN: loading from master file lab.local.zone failed: file not found
```

- 아직 존 파일이 없어 실패하는 것이 정상 — 3-4 에서 생성

> 📝 **시험 포인트**: 필기 FULL r10-74 — `192.168.10.0/24` 의 역방향 존 이름은 `10.168.192.in-addr.arpa`. 옥텟 순서를 안 뒤집은 선지가 항상 함께 나온다. r01-72·r04-72·r05-70 은 `type master`/`slave` 구분.

### 3-4. 정방향 존 파일 — /var/named/lab.local.zone

> **상황**: 사내에서 쓸 이름을 실제 주소로 매핑한다. 웹(2절)·메일(7절)·FTP(6절)에서 쓸 이름을 미리 다 넣는다.

```bash
cat > /var/named/lab.local.zone <<'EOF'
$TTL 86400
@       IN      SOA     ns1.lab.local. admin.lab.local. (
                        2026090301      ; Serial
                        3600            ; Refresh
                        1800            ; Retry
                        604800          ; Expire
                        86400 )         ; Minimum TTL

; --- 네임서버 ---
@               IN      NS      ns1.lab.local.

; --- 호스트 (A) ---
ns1             IN      A       192.168.64.10
srv01           IN      A       192.168.64.10
www             IN      A       192.168.64.10
intranet        IN      A       192.168.64.10
mail            IN      A       192.168.64.10

; --- 별칭 (CNAME) ---
ftp             IN      CNAME   srv01
intra           IN      CNAME   intranet

; --- 메일 교환기 (MX) ---
@               IN      MX      10      mail.lab.local.

; --- 텍스트 (TXT) ---
@               IN      TXT     "v=spf1 ip4:192.168.64.10 -all"
lab.local.      IN      TXT     "LAB 09 internal zone"
EOF
```

| 항목 | 값 | 의미 |
| --- | --- | --- |
| `$TTL 86400` | 초 | 개별 TTL 이 없는 레코드의 **기본 캐시 시간**(24시간) |
| `@` | — | 존 이름 자기 자신(`lab.local.`) 을 뜻하는 축약 |
| `SOA` 1번째 | `ns1.lab.local.` | 이 존의 **주 네임서버** |
| `SOA` 2번째 | `admin.lab.local.` | 관리자 메일 주소 — `@` 를 `.` 로 바꿔 표기(`admin@lab.local`) |
| `Serial` | `2026090301` | **일련번호**. 관례 `YYYYMMDDnn`. 존 수정 시 **반드시 증가** → 슬레이브 전송 트리거 |
| `Refresh` | `3600`(1시간) | 슬레이브가 마스터 Serial 을 확인하러 오는 주기 |
| `Retry` | `1800`(30분) | Refresh 실패 시 재시도 간격 |
| `Expire` | `604800`(7일) | 마스터와 통신 불가 시 슬레이브가 존을 **폐기**하기까지의 시간 |
| `Minimum TTL` | `86400` | 부정 응답(NXDOMAIN) 캐시 시간(RFC 2308) |

| 레코드 | 역할 |
| --- | --- |
| `NS` | 이 존의 네임서버 지정 |
| `A` | 호스트 이름 → **IPv4** 주소 |
| `AAAA` | 호스트 이름 → IPv6 주소 |
| `CNAME` | 별칭 → 정식 이름. **다른 레코드와 공존 불가**, 존 정점(`@`)에 사용 불가 |
| `MX` | 메일 서버. 앞 숫자는 우선순위 — **작을수록 우선** |
| `PTR` | IP → 이름 (역방향 존 전용) |
| `TXT` | 임의 문자열 (SPF·소유권 검증 등) |

- ⚠️ **끝의 점(`.`) 함정**: `mail.lab.local.` 처럼 점으로 끝나면 절대 이름(FQDN). 점이 없으면 뒤에 존 이름이 **자동으로 붙는다** → `mail.lab.local` 이라 쓰면 `mail.lab.local.lab.local.` 이 됨. 반대로 `www` 처럼 짧은 이름은 점을 붙이면 안 됨

**검증**

```bash
named-checkzone lab.local /var/named/lab.local.zone
grep -c 'IN' /var/named/lab.local.zone
```

```text
# named-checkzone lab.local /var/named/lab.local.zone
zone lab.local/IN: loaded serial 2026090301
OK
```

> 📝 **시험 포인트**: 실기 r01-14(MX vs A 차이), r06-12(SOA 의 Serial·Refresh·Expire 의미 + Serial 증가 이유) 가 그대로 이 표. 필기 FULL r02-71, r04-71, r10-76 — MX 우선순위는 **숫자가 작은 쪽이 먼저**(`MX 10` > `MX 20`). r10-73 — CNAME 은 apex 에 못 쓰고 다른 레코드와 공존 불가.

### 3-5. 역방향 존 파일 — /var/named/192.168.64.rev

> **상황**: 메일 서버·로그 분석에서 IP → 이름 조회가 필요하다. 역방향 존을 만든다.

```bash
cat > /var/named/192.168.64.rev <<'EOF'
$TTL 86400
@       IN      SOA     ns1.lab.local. admin.lab.local. (
                        2026090301      ; Serial
                        3600            ; Refresh
                        1800            ; Retry
                        604800          ; Expire
                        86400 )         ; Minimum TTL

@       IN      NS      ns1.lab.local.

1       IN      PTR     gw.lab.local.
10      IN      PTR     srv01.lab.local.
EOF
```

- 왼쪽 숫자 `10` = 호스트 옥텟 → 존 이름이 붙어 `10.64.168.192.in-addr.arpa` 가 됨
- 오른쪽은 **반드시 FQDN + 끝점** (`srv01.lab.local.`)
- 한 IP 에 PTR 은 보통 **1개만** 둠 (여러 이름이 있어도 대표 하나)

**검증**

```bash
named-checkzone 64.168.192.in-addr.arpa /var/named/192.168.64.rev
named-checkconf -z 2>&1 | tail -5
```

```text
zone 64.168.192.in-addr.arpa/IN: loaded serial 2026090301
OK
```

> 📝 **시험 포인트**: 역방향 존에서 왼쪽에 **호스트 옥텟만** 쓰고 오른쪽에 **FQDN + 점**을 쓴다. 좌우를 바꾸거나 끝점을 뺀 선지가 오답으로 나온다.

### 3-6. 존 파일 소유·권한과 SELinux

> **상황**: `named` 프로세스가 존 파일을 읽지 못하면 존 로딩에 실패한다. 소유·권한·SELinux 컨텍스트를 한꺼번에 맞춘다.

```bash
chgrp named /var/named/lab.local.zone /var/named/192.168.64.rev
chmod 640   /var/named/lab.local.zone /var/named/192.168.64.rev
restorecon -Rv /var/named
ls -lZ /var/named/lab.local.zone /var/named/192.168.64.rev
```

- `chgrp named` : 그룹 소유자를 `named` 로 — 소유자는 root 로 두고 그룹 읽기만 허용
- `chmod 640` : 소유자 rw, 그룹 r, 기타 없음
- `restorecon` : 정책 기본값인 `named_zone_t` 로 라벨 복원

**검증**

```bash
ls -lZ /var/named/*.zone /var/named/*.rev
sudo -u named test -r /var/named/lab.local.zone && echo 'named can read'
```

```text
-rw-r-----. 1 root named unconfined_u:object_r:named_zone_t:s0 ... lab.local.zone
-rw-r-----. 1 root named unconfined_u:object_r:named_zone_t:s0 ... 192.168.64.rev
named can read
```

> 📝 **시험 포인트**: 존 파일이 로드되지 않는 3대 원인 — ① Serial 미증가(슬레이브 미반영) ② 권한/소유(`named` 그룹 읽기 불가) ③ 문법 오류(끝점 누락). `journalctl -u named` 에 원인이 그대로 찍힌다.

### 3-7. 기동·부팅 등록·방화벽

> **상황**: 존 준비가 끝났으니 데몬을 올리고 53 포트를 연다.

```bash
systemctl enable --now named
systemctl status named --no-pager | head -8
ss -tulnp | grep :53
firewall-cmd --permanent --add-service=dns
firewall-cmd --reload
```

- `dns` 서비스 정의는 **53/TCP + 53/UDP 를 모두** 개방 (`/usr/lib/firewalld/services/dns.xml`)

**검증**

```bash
systemctl is-active named; systemctl is-enabled named
ss -tulnp | grep named
journalctl -u named -n 15 --no-pager | grep -Ei 'zone|loaded|running'
firewall-cmd --list-services | tr ' ' '\n' | grep dns
```

```text
# ss -tulnp | grep named
udp UNCONN 0 0 192.168.64.10:53 0.0.0.0:* users:(("named",pid=...,fd=...))
udp UNCONN 0 0     127.0.0.1:53 0.0.0.0:* users:(("named",pid=...,fd=...))
tcp LISTEN 0 10  192.168.64.10:53 0.0.0.0:* users:(("named",pid=...,fd=...))
tcp LISTEN 0 10     127.0.0.1:53 0.0.0.0:* users:(("named",pid=...,fd=...))
# journalctl -u named ...
zone lab.local/IN: loaded serial 2026090301
zone 64.168.192.in-addr.arpa/IN: loaded serial 2026090301
all zones loaded
running
```

> 📝 **시험 포인트**: 실기 r03-8 — "53번 포트를 LISTEN 중인 **UDP** 소켓을 프로세스와 함께" 는 `ss -ulnp | grep :53` (`-u` UDP, `-l` listening, `-n` 숫자, `-p` 프로세스). DNS 가 TCP·UDP 를 모두 쓰는 이유(대용량 응답·존 전송)도 함께 출제.

### 3-8. dig 로 레코드별 조회

> **상황**: 만든 존이 실제로 응답하는지 레코드 종류별로 확인한다.

```bash
dig @127.0.0.1 www.lab.local
dig @127.0.0.1 www.lab.local +short
dig @192.168.64.10 intranet.lab.local A +short
dig @127.0.0.1 lab.local MX
dig @127.0.0.1 lab.local NS +short
dig @127.0.0.1 lab.local SOA
dig @127.0.0.1 lab.local TXT +short
dig @127.0.0.1 ftp.lab.local           # CNAME → A 두 줄로 응답
dig @127.0.0.1 nosuch.lab.local        # NXDOMAIN 확인
```

- `@<서버>` : 질의를 보낼 네임서버 지정 (미지정 시 `/etc/resolv.conf`)
- `+short` : 답변 값만 간결 출력
- `+noall +answer` : ANSWER 섹션만
- `+trace` : 루트부터 단계별 위임 추적
- `-t <타입>` 또는 마지막 인자로 타입 지정 (`A` `AAAA` `MX` `NS` `SOA` `TXT` `PTR` `CNAME` `ANY`)

**검증**

```bash
dig @127.0.0.1 www.lab.local +noall +answer
dig @127.0.0.1 lab.local MX +noall +answer
dig @127.0.0.1 ftp.lab.local +noall +answer
dig @127.0.0.1 nosuch.lab.local | grep -E 'status:'
```

```text
# dig @127.0.0.1 www.lab.local +noall +answer
www.lab.local.		86400	IN	A	192.168.64.10
# dig @127.0.0.1 lab.local MX +noall +answer
lab.local.		86400	IN	MX	10 mail.lab.local.
# dig @127.0.0.1 ftp.lab.local +noall +answer
ftp.lab.local.		86400	IN	CNAME	srv01.lab.local.
srv01.lab.local.	86400	IN	A	192.168.64.10
# dig @127.0.0.1 nosuch.lab.local | grep status:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: ...
```

- 응답 헤더 `status:` — `NOERROR`(정상) · `NXDOMAIN`(이름 없음) · `SERVFAIL`(서버 오류·존 미로드) · `REFUSED`(allow-query 거부)

> 📝 **시험 포인트**: 필기 FULL r04-90 — "8.8.8.8 에게 example.com 의 MX 조회" 는 `dig @8.8.8.8 example.com mx`. `@` 위치를 도메인 뒤로 보낸 선지가 오답. r02-74 — `+short` 는 간결 출력(SOA 전용 아님), `-x` 는 역방향.

### 3-9. 역방향 조회·존 전송 거부·다른 조회 도구

> **상황**: PTR 이 동작하는지, 그리고 `allow-transfer { none; }` 이 실제로 존 통째 유출을 막는지 확인한다.

```bash
dig @192.168.64.10 -x 192.168.64.10                 # 역방향(PTR)
dig @127.0.0.1 -x 192.168.64.1 +short
dig @127.0.0.1 lab.local AXFR                        # 존 전송 시도 → 거부되어야 정상
nslookup www.lab.local 127.0.0.1
nslookup -type=mx lab.local 127.0.0.1
host www.lab.local 127.0.0.1
host -t ns lab.local 127.0.0.1
host 192.168.64.10 127.0.0.1                         # host 는 IP 주면 자동 역방향
```

- `dig -x <IP>` : IP 를 `in-addr.arpa` 형식으로 자동 변환해 **역방향 조회**
- `AXFR` : 존 전체 전송(**A**ll zone **X**FR). 허용된 대상만 가능
- `nslookup -type=<타입>` : 레코드 종류 지정 (대화식 모드에서는 `set type=mx`)
- `host -t <타입>` : 레코드 종류 지정 (**t**ype), `host -a` : 전체

**검증**

```bash
dig @127.0.0.1 -x 192.168.64.10 +short
dig @127.0.0.1 lab.local AXFR | tail -3
host www.lab.local 127.0.0.1
```

```text
# dig @127.0.0.1 -x 192.168.64.10 +short
srv01.lab.local.
# dig @127.0.0.1 lab.local AXFR | tail -3
; Transfer failed.
# host www.lab.local 127.0.0.1
Using domain server:
Name: 127.0.0.1
www.lab.local has address 192.168.64.10
```

- `Transfer failed.` = `allow-transfer { none; }` 가 정상 동작. 슬레이브를 둘 때만 특정 IP 를 허용

> 📝 **시험 포인트**: 필기 FULL r08-74, r09-74 — 존 전송 제한은 `allow-transfer`(오답 선지: `allow-query`·`forwarders`·`allow-recursion`). 존 전송을 열어 두면 내부 호스트 목록이 통째로 유출되는 **정보 수집 공격** 경로.

### 3-10. 서버 자신을 DNS 클라이언트로 전환

> **상황**: 이제 이 서버가 자기 DNS 를 먼저 보게 만든다. `/etc/resolv.conf` 를 직접 고치면 NetworkManager 가 덮어쓰므로 `nmcli` 로 연결 프로파일을 수정한다.

```bash
nmcli con show                                        # 연결 이름 확인
nmcli con mod enp0s1 ipv4.dns "127.0.0.1 192.168.64.1"
nmcli con mod enp0s1 ipv4.dns-search "lab.local"
nmcli con mod enp0s1 ipv4.ignore-auto-dns yes         # DHCP 가 준 DNS 무시
nmcli con up enp0s1
cat /etc/resolv.conf
```

- `nmcli con mod <연결> ipv4.dns "…"` : 연결 프로파일의 DNS 목록 설정(공백 구분, 기존 값 대체)
- `ipv4.dns-search` : `search` 도메인 — 짧은 이름에 자동으로 붙일 접미사
- `ipv4.ignore-auto-dns yes` : 자동(DHCP) DNS 를 무시하고 수동 값만 사용
- `nmcli con up <연결>` : 프로파일 재적용 (**재부팅 없이** 반영)
- 네트워크 설정 상세는 → [[08-network-config]]

**검증**

```bash
cat /etc/resolv.conf
dig www.lab.local +short              # @ 없이 → resolv.conf 의 서버 사용
ping -c 2 intranet.lab.local
getent hosts mail.lab.local
resolvectl status 2>/dev/null | head -12 || nmcli dev show enp0s1 | grep -i dns
```

```text
# cat /etc/resolv.conf
# Generated by NetworkManager
search lab.local
nameserver 127.0.0.1
nameserver 192.168.64.1
# dig www.lab.local +short
192.168.64.10
# ping -c 2 intranet.lab.local
PING intranet.lab.local (192.168.64.10) 56(84) bytes of data.
64 bytes from srv01.lab.local (192.168.64.10): icmp_seq=1 ttl=64 time=0.0.. ms
```

- ⚠️ `/etc/hosts` 항목(2-9)이 먼저 매칭되므로, DNS 만으로 해석되는지 보려면 `/etc/hosts` 의 별칭 줄을 잠시 지우고 `dig` 로 확인

> 📝 **시험 포인트**: 필기 FULL r01-71, r06-16/17 — `/etc/resolv.conf` 의 `nameserver` 는 최대 3개, 위에서부터 순서대로 시도. RHEL 9 에서는 NetworkManager 가 생성하므로 직접 편집은 재부팅·재연결 시 소실.

### 3-11. 존 수정 → Serial 증가 → rndc reload

> **상황**: 개발용 호스트 `dev.lab.local` 을 추가한다. **Serial 을 올리지 않으면** 슬레이브가 변경을 인식하지 못한다는 것을 함께 확인한다.

```bash
# ① Serial 만 그대로 두고 레코드 추가 — 잘못된 절차 재현
echo 'dev             IN      A       192.168.64.20' >> /var/named/lab.local.zone
named-checkzone lab.local /var/named/lab.local.zone
rndc reload lab.local
dig @127.0.0.1 dev.lab.local +short          # 마스터 자신은 반영됨
dig @127.0.0.1 lab.local SOA +short          # Serial 이 그대로 → 슬레이브는 갱신 안 함
```

```text
192.168.64.20
ns1.lab.local. admin.lab.local. 2026090301 3600 1800 604800 86400
```

- 마스터는 파일을 다시 읽으므로 **자기 응답은 바뀐다**. 그러나 Serial 이 같으면 슬레이브는 Refresh 때 "변경 없음" 으로 판단해 **옛 데이터를 유지** → 마스터·슬레이브 불일치

```bash
# ② 올바른 절차 — Serial 증가 후 reload
sed -i 's/2026090301      ; Serial/2026090302      ; Serial/' /var/named/lab.local.zone
named-checkzone lab.local /var/named/lab.local.zone
rndc reload lab.local
```

- `rndc reload [존]` : 존을 지정하면 해당 존만, 생략하면 설정+전체 존 재적재
- `rndc reconfig` : **새로 추가된 zone 선언**을 읽어 들임(기존 존은 그대로) — `named.conf` 에 존을 추가했을 때
- `systemctl reload named` 로도 동일 효과

**검증**

```bash
dig @127.0.0.1 lab.local SOA +short
dig @127.0.0.1 dev.lab.local +short
journalctl -u named -n 5 --no-pager | grep -i 'loaded serial'
```

```text
# dig @127.0.0.1 lab.local SOA +short
ns1.lab.local. admin.lab.local. 2026090302 3600 1800 604800 86400
# dig @127.0.0.1 dev.lab.local +short
192.168.64.20
# journalctl ...
zone lab.local/IN: loaded serial 2026090302
```

> 📝 **시험 포인트**: 실기 r06-7 — "슬레이브 존 전송을 트리거하려면 반드시 증가시켜야 하는 SOA 항목" = **Serial**. 필기 FULL r03-71, r05-72, r07-73 — 슬레이브는 Refresh 주기마다 마스터의 SOA Serial 을 비교해 크면 AXFR(전체)/IXFR(증분) 로 갱신.

### 3-12. rndc 제어 명령

> **상황**: 데몬을 재시작하지 않고 상태를 보거나 캐시를 비우는 운영 명령을 익힌다.

```bash
rndc status                       # 서버 상태 요약
rndc reload                       # 설정 + 전체 존 재적재
rndc reload lab.local             # 특정 존만
rndc reconfig                     # named.conf 재읽기 (새 zone 인식)
rndc flush                        # 캐시 전체 비우기
rndc flushname www.example.com    # 특정 이름만 캐시 삭제
rndc querylog on                  # 질의 로깅 켜기 (다시 off)
rndc dumpdb -cache                # 캐시를 /var/named/data/cache_dump.db 로 덤프
rndc stats                        # 통계를 named_stats.txt 로 기록
```

| 서브명령 | 동작 |
| --- | --- |
| `status` | 버전·존 개수·질의 수·recursion 상태 |
| `reload [존]` | 존 파일 재적재 |
| `reconfig` | `named.conf` 만 재읽기 — **존 추가 시** |
| `flush` / `flushname` | 캐시 삭제 |
| `querylog on\|off` | 질의 로그 토글 |
| `freeze` / `thaw` | 동적 갱신 존 편집 전후 잠금·해제 |
| `dumpdb` / `stats` | 캐시·통계 파일 출력 |
| `halt` / `stop` | 즉시 종료 / 저장 후 종료 |

- ⚠️ `rndc restart` 는 **존재하지 않는다** — 재시작은 `systemctl restart named`
- 제어 채널 인증은 `/etc/rndc.key` 의 공유 키(HMAC) 사용

**검증**

```bash
rndc status
rndc querylog on
dig @127.0.0.1 www.lab.local +short >/dev/null
journalctl -u named -n 3 --no-pager | grep -i query
rndc querylog off
```

```text
# rndc status
version: BIND 9.16...
number of zones: ...
debug level: 0
recursive clients: 0/900/1000
server is up and running
# journalctl ...
client @0x... 127.0.0.1#...  (www.lab.local): query: www.lab.local IN A +E(0)K (127.0.0.1)
```

> 📝 **시험 포인트**: 필기 FULL r04-69 — `rndc` 서브명령 중 **틀린 설명 고르기**에서 "`restart` 는 named 를 재시작한다" 가 오답(그런 서브명령 없음). `reload`(존 재적재) vs `reconfig`(설정 재읽기) 구분도 출제.

### 3-13. 존 파일 오타 재현 → named-checkzone

> **상황**: 실제 운영에서 가장 흔한 실패인 "끝점 누락" 을 일부러 만들어 검사 도구가 어떻게 잡아 주는지 본다.

⚠️ 검사만 하고 원복한다. 오타 상태로 `reload` 하면 존 전체가 로드되지 않아 **SERVFAIL** 이 난다.

```bash
cp /var/named/lab.local.zone /var/named/lab.local.zone.bak
# MX 대상의 끝점을 제거 → mail.lab.local.lab.local. 로 해석됨
sed -i 's|MX      10      mail.lab.local.|MX      10      mail.lab.local|' /var/named/lab.local.zone
named-checkzone lab.local /var/named/lab.local.zone
```

```text
zone lab.local/IN: NS 'ns1.lab.local' has no address records (A or AAAA)
zone lab.local/IN: loaded serial 2026090302
OK
```

- 끝점 누락은 문법 오류가 아니라 **의미 오류** — `checkzone` 이 경고만 내고 통과할 수 있음. 그래서 `dig` 로 실제 값 확인이 필수

```bash
# 진짜 문법 오류도 재현 — SOA 괄호를 닫지 않음
printf 'bad     IN      A\n' >> /var/named/lab.local.zone
named-checkzone lab.local /var/named/lab.local.zone
```

```text
/var/named/lab.local.zone:29: unexpected end of line
zone lab.local/IN: loading from master file /var/named/lab.local.zone failed: unexpected end of input
zone lab.local/IN: not loaded due to errors.
```

- 출력의 `파일:행번호` 로 오류 위치를 바로 알 수 있음

```bash
# 원복 후 재검사
cp /var/named/lab.local.zone.bak /var/named/lab.local.zone
chgrp named /var/named/lab.local.zone; chmod 640 /var/named/lab.local.zone
restorecon -v /var/named/lab.local.zone
named-checkzone lab.local /var/named/lab.local.zone
rndc reload lab.local
```

**검증**

```bash
named-checkzone lab.local /var/named/lab.local.zone
dig @127.0.0.1 lab.local MX +short
dig @127.0.0.1 www.lab.local +short
```

```text
zone lab.local/IN: loaded serial 2026090302
OK
10 mail.lab.local.
192.168.64.10
```

> 📝 **시험 포인트**: 존 파일 검사는 `named-checkzone <존이름> <파일>` — **인자 2개, 순서 고정**. 설정 파일은 `named-checkconf`(인자 없음). 끝점(`.`) 누락은 오류 없이 잘못된 이름으로 이어지는 최악의 함정.

### 3-14. 슬레이브·캐시 전용 서버 설정 (※ 미실행)

> **상황**: VM 이 1대라 실제 슬레이브를 세울 수 없다. 설정 형태와 동작 원리만 정리한다.

**마스터 쪽 — 전송 허용 + 알림**

```
zone "lab.local" IN {
        type master;
        file "lab.local.zone";
        allow-transfer { 192.168.64.11; };     // 슬레이브 IP 만 허용
        also-notify    { 192.168.64.11; };     // 변경 시 즉시 NOTIFY
        notify yes;
};
```

**슬레이브 쪽 (※ 미실행)**

```
zone "lab.local" IN {
        type slave;
        masters { 192.168.64.10; };            // 마스터 IP (필수)
        file "slaves/lab.local.zone";          // 쓰기 가능한 경로
};
```

**캐시 전용(caching-only) 서버 (※ 미실행)**

```
options {
        recursion yes;                          // 재귀 허용
        allow-query { 192.168.64.0/24; };
        forwarders { 192.168.64.1; };
};
zone "." IN { type hint; file "named.ca"; };    // 루트 힌트만 보유
// 자체 관리 존 없음 → 질의 결과를 캐시해 응답 속도만 개선
```

| 서버 유형 | `type` | 존 데이터 보유 | 특징 |
| --- | --- | --- | --- |
| 마스터(주) | `master` | **원본** | 존 파일을 직접 편집 |
| 슬레이브(보조) | `slave` | 복제본 | `masters` 필수, AXFR/IXFR 로 수신 |
| 캐시 전용 | (자체 존 없음) | 없음 | `hint` + 재귀. 응답 캐싱만 |
| 포워딩 전용 | `forward` | 없음 | 지정 서버로 전달만 |

- 존 전송 흐름: 마스터 존 수정 → **Serial 증가** → 마스터가 NOTIFY 전송 → 슬레이브가 SOA 조회로 Serial 비교 → 크면 **AXFR**(전체) 또는 **IXFR**(증분) 요청 → 수신 후 `file` 경로에 저장

> 📝 **시험 포인트**: 필기 FULL r04-72, r06-72 — 슬레이브 존 선언에 반드시 필요한 것은 `masters { 마스터IP; };` 와 쓰기 가능한 `file` 경로. r07-73 — NOTIFY → SOA 비교 → AXFR/IXFR 순서.

### 3-15. named 로그 확인

> **상황**: 존 로딩·질의 오류를 어디서 보는지 확인한다.

```bash
journalctl -u named --no-pager | tail -20
journalctl -u named -p err --no-pager
grep -i named /var/log/messages | tail -10
ls -l /var/named/data/
```

- BIND 는 기본적으로 syslog `daemon` 퍼실리티로 기록 → journald + `/var/log/messages` 양쪽에 남음
- 별도 파일로 분리하려면 `named.conf` 에 `logging { channel … ; category … ; };` 블록 추가

**검증**

```bash
systemctl reload named
journalctl -u named -n 6 --no-pager
```

```text
named[...]: loading configuration from '/etc/named.conf'
named[...]: zone lab.local/IN: loaded serial 2026090302
named[...]: zone 64.168.192.in-addr.arpa/IN: loaded serial 2026090301
named[...]: all zones loaded
named[...]: running
```

> 📝 **시험 포인트**: 서비스 로그 조회는 `journalctl -u <유닛>` — Part 07 참조. `-p err` 로 심각도 필터, `-f` 로 실시간 추적.

---

## 4. NFS — 리눅스 간 파일 공유

### 4-1. 설치와 공유 디렉터리 설계

> **상황**: 개발팀이 빌드 산출물을 리눅스끼리 주고받을 공간이 필요하다. `/srv/nfs/data` 를 만들어 사내 대역에 내보낸다.

```bash
dnf install -y nfs-utils
mkdir -p /srv/nfs/data
chown root:devteam /srv/nfs/data
chmod 2775 /srv/nfs/data
echo 'nfs shared file' > /srv/nfs/data/README.txt
ls -ldZ /srv/nfs /srv/nfs/data
```

- `nfs-utils` : 서버 데몬(`rpc.nfsd`·`rpc.mountd`)과 클라이언트 도구(`showmount`·`nfsstat`·`mount.nfs`)를 모두 포함 — 서버·클라이언트 한 패키지
- `chmod 2775` : SetGID(2) — 이 디렉터리에서 만들어지는 파일이 **부모 그룹(devteam)** 을 상속. 팀 공유 디렉터리의 표준 권한
- 소유·권한은 **서버 쪽 파일시스템 기준**으로 적용됨 — NFS 는 UID/GID 숫자를 그대로 전달하므로 서버·클라이언트의 UID 가 일치해야 의도대로 동작

**검증**

```bash
ls -ld /srv/nfs/data
stat -c '%A %U %G %n' /srv/nfs/data
rpm -q nfs-utils
```

```text
drwxrwsr-x. 2 root devteam ... /srv/nfs/data
drwxrwsr-x root devteam /srv/nfs/data
nfs-utils-2.5.4-...el9.aarch64
```

- 그룹 실행 자리의 `s` = SetGID 설정됨

> 📝 **시험 포인트**: NFS 는 **UID/GID 숫자 기반** 인증이므로 서버·클라이언트의 계정 UID 가 다르면 엉뚱한 소유자로 보인다. 그래서 실무에서는 NIS·LDAP 로 계정을 중앙 관리(5절·9-4 참조).

### 4-2. /etc/exports 문법과 옵션

> **상황**: 어떤 디렉터리를 누구에게 어떤 조건으로 열지 선언한다. **공백 규칙**이 가장 흔한 실수 지점이다.

```bash
cat > /etc/exports <<'EOF'
/srv/nfs/data   192.168.64.0/24(rw,sync,no_subtree_check,root_squash)
/srv/nfs/data   127.0.0.1(rw,sync,no_subtree_check,no_root_squash)
EOF
cat /etc/exports
```

- 한 줄 형식: `<공유 디렉터리><공백><클라이언트1>(<옵션>) <클라이언트2>(<옵션>)`
- ⚠️ **클라이언트와 여는 괄호 사이에 공백을 넣으면 안 됨**
  - `192.168.64.0/24(rw)` → 해당 대역에 rw
  - `192.168.64.0/24 (rw)` → 해당 대역은 **기본값(ro)**, 그리고 **모든 호스트(`*`)에 rw** — 의도치 않은 전체 개방
- 클라이언트 표기: 단일 IP · CIDR(`192.168.64.0/24`) · 호스트명 · 와일드카드(`*.lab.local`) · `*`(전체)

| 옵션 | 의미 |
| --- | --- |
| `rw` | 읽기·쓰기 허용 |
| `ro` | **읽기 전용**(옵션 미지정 시 기본) |
| `sync` | 쓰기를 디스크에 반영한 뒤 응답 — 안전, 기본 권장 |
| `async` | 메모리에 받고 바로 응답 — 빠르지만 정전 시 유실 위험 |
| `root_squash` | 클라이언트 **root(UID 0) → 익명 사용자(nobody)** 로 강등. **기본값** |
| `no_root_squash` | 클라이언트 root 를 서버에서도 root 로 인정 ⚠️ 보안 위험 |
| `all_squash` | **모든 사용자**를 익명으로 매핑 (공개 읽기 공유용) |
| `no_all_squash` | 일반 사용자 UID 유지 (기본) |
| `anonuid=<n>` / `anongid=<n>` | 익명 매핑 대상 UID/GID 지정 (기본 65534 = nobody) |
| `subtree_check` | 요청 경로가 공유 하위인지 매번 검사 — 느림 |
| `no_subtree_check` | 하위 검사 생략 (**현재 기본 권장**) |
| `secure` | 클라이언트가 **1024 미만 특권 포트**에서 접속해야 허용 (기본) |
| `insecure` | 1024 이상 포트도 허용 |
| `wdelay` / `no_wdelay` | 쓰기 지연 묶음 처리 / 즉시 반영 |
| `hide` / `nohide` | 중첩 마운트 노출 여부 |

**검증**

```bash
cat -A /etc/exports | head -3        # 보이지 않는 공백·탭 확인
exportfs -ra 2>&1                    # 문법 오류가 있으면 여기서 메시지 출력
```

```text
# cat -A /etc/exports | head -2
/srv/nfs/data^I192.168.64.0/24(rw,sync,no_subtree_check,root_squash)$
/srv/nfs/data^I127.0.0.1(rw,sync,no_subtree_check,no_root_squash)$
```

- `cat -A` : 탭을 `^I`, 줄 끝을 `$` 로 표시 — 공백 함정 진단에 유용

> 📝 **시험 포인트**: 실기 r04-14, r05-13 — `rw`·`sync`·`no_root_squash` 의 의미 서술과 `no_root_squash` 의 위험(클라이언트 root 가 서버 파일을 마음대로 변경 가능) 이 그대로 출제. 필기 FULL r01-79, r02-79, r03-77, r04-78, r05-78/79, r09-83, r10-79 에서 반복.

### 4-3. exportfs 로 반영·조회

> **상황**: `/etc/exports` 를 고쳤으니 서비스 재시작 없이 반영한다.

```bash
exportfs -ra            # 전체 재적용 (re-export all)
exportfs -v             # 현재 내보낸 목록 + 적용된 전체 옵션
exportfs -s             # /etc/exports 형식 그대로 요약 출력
exportfs -u 192.168.64.0/24:/srv/nfs/data     # 특정 공유 해제 (unexport)
exportfs -a             # /etc/exports 의 모든 항목 export
exportfs -ra            # 다시 전체 적용
```

- `-r` : 재적용 (**r**e-export) — `/etc/exports` 와 현재 상태를 동기화
- `-a` : 전체 (**a**ll)
- `-u` : 해제 (**u**nexport)
- `-v` : 상세 (**v**erbose) — 명시하지 않은 **기본 옵션까지 전부** 표시
- `-s` : 요약 (**s**how)

**검증**

```bash
exportfs -v
cat /var/lib/nfs/etab | tr ',' '\n' | head -20
```

```text
# exportfs -v
/srv/nfs/data 	192.168.64.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
/srv/nfs/data 	127.0.0.1(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
```

- `/var/lib/nfs/etab` : 커널에 실제 적용된 **유효 export 테이블** — `/etc/exports` 에 안 쓴 기본값까지 다 들어 있음

> 📝 **시험 포인트**: 필기 FULL r02-80, r04-76, r05-80 — "`/etc/exports` 수정 후 재시작 없이 반영" 은 `exportfs -r`(또는 `-ra`). 실기 r05-8 도 `exportfs -ra` 를 빈칸으로 묻는다. `showmount` 는 **클라이언트에서 조회**하는 명령이라 여기서는 오답.

### 4-4. 기동·부팅 등록·방화벽

> **상황**: NFS 서버 데몬을 올린다. RPC 기반이므로 `rpcbind` 가 함께 떠야 한다.

```bash
systemctl enable --now nfs-server
systemctl status nfs-server --no-pager | head -6
systemctl is-active rpcbind          # nfs-server 가 의존해 자동 기동
firewall-cmd --permanent --add-service=nfs
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --permanent --add-service=mountd
firewall-cmd --reload
```

- `nfs-server` : NFS 서버 유닛명 (데몬 이름 `nfsd` 와 다름)
- `rpcbind` : RPC 프로그램 번호 ↔ 포트 매핑(포트 111). **NFSv3 에 필수**, v4 는 2049 단독으로도 동작
- `nfs` / `rpc-bind` / `mountd` : firewalld 서비스 정의 3종 — v3 호환까지 고려하면 셋 다 개방

**검증**

```bash
systemctl is-active nfs-server; systemctl is-enabled nfs-server
ss -tlnp | grep -E ':2049|:111'
rpcinfo -p | head -12
cat /proc/fs/nfsd/versions
firewall-cmd --list-services
```

```text
# ss -tlnp | grep 2049
LISTEN 0 64 0.0.0.0:2049 0.0.0.0:*
# rpcinfo -p | head
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   udp    111  portmapper
    100003    3   tcp   2049  nfs
    100003    4   tcp   2049  nfs
    100005    3   tcp   ...   mountd
# cat /proc/fs/nfsd/versions
-2 +3 +4 +4.1 +4.2
```

- `rpcinfo -p` : 로컬 rpcbind 에 등록된 **p**rogram 목록 — NFS·mountd 포트 확인
- `/proc/fs/nfsd/versions` : `+` 는 활성, `-` 는 비활성 버전. `/etc/nfs.conf` 의 `[nfsd]` 섹션으로 조정

```bash
grep -nA6 '^\[nfsd\]' /etc/nfs.conf
```

> 📝 **시험 포인트**: 필기 FULL r03-76 — NFSv4 는 **TCP 2049 단일 포트**로 동작해 rpcbind·별도 mountd 포트 의존이 줄었다. "NFSv3 는 rpcbind 없이 동작한다" 는 오답.

### 4-5. 공유 목록 확인 — showmount

> **상황**: 클라이언트 입장에서 이 서버가 무엇을 내보내는지 조회한다.

```bash
showmount -e 127.0.0.1              # export 목록
showmount -e 192.168.64.10
showmount -a 127.0.0.1              # 현재 마운트 중인 클라이언트:경로
showmount -d 127.0.0.1              # 마운트된 디렉터리 목록
```

- `-e` : **e**xport 목록 (가장 많이 쓰임)
- `-a` : **a**ll — `호스트:디렉터리` 쌍
- `-d` : **d**irectories only
- `--no-headers` : 헤더 줄 생략

**검증**

```bash
showmount -e 192.168.64.10
```

```text
Export list for 192.168.64.10:
/srv/nfs/data 192.168.64.0/24,127.0.0.1
```

> 📝 **시험 포인트**: 필기 FULL r01-80, r02-80, r04-77, r05-80, r06-77, r08-77 — "클라이언트에서 서버의 공유 목록 확인" 은 `showmount -e <서버>`. 오답 단골: `exportfs -r`(서버에서 반영), `nfsstat`(통계), `rpcinfo -p`(RPC 포트).

### 4-6. 클라이언트 마운트

> **상황**: 같은 VM 을 클라이언트로 삼아 `/mnt/nfs` 에 붙인다.

```bash
mkdir -p /mnt/nfs
mount -t nfs 192.168.64.10:/srv/nfs/data /mnt/nfs
df -hT /mnt/nfs
mount | grep nfs
findmnt /mnt/nfs
```

- `-t nfs` : 파일시스템 **t**ype. `nfs4` 로 명시할 수도 있음
- `<서버>:<서버측 경로> <로컬 마운트 지점>` 순서
- 주요 마운트 옵션
  - `vers=3|4|4.2` : 프로토콜 버전 고정
  - `soft` / `hard` : 서버 무응답 시 오류 반환 / 무한 재시도(기본 `hard`)
  - `timeo=<0.1초>` `retrans=<n>` : 타임아웃·재전송 횟수
  - `rsize=` `wsize=` : 읽기·쓰기 블록 크기
  - `noexec` `nosuid` `nodev` : 보안 강화
  - `_netdev` : 네트워크 준비 후 마운트 (fstab 전용)

**검증**

```bash
df -hT /mnt/nfs
findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS /mnt/nfs
cat /mnt/nfs/README.txt
nfsstat -m                      # 마운트별 실제 협상 옵션
nfsstat -c | head -8            # 클라이언트 통계
nfsstat -s | head -8            # 서버 통계
```

```text
# df -hT /mnt/nfs
Filesystem                    Type  Size  Used Avail Use% Mounted on
192.168.64.10:/srv/nfs/data   nfs4   36G  ...   ...   ..% /mnt/nfs
# findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS /mnt/nfs
TARGET   SOURCE                      FSTYPE OPTIONS
/mnt/nfs 192.168.64.10:/srv/nfs/data nfs4   rw,relatime,vers=4.2,rsize=...,wsize=...,hard,proto=tcp,...
# cat /mnt/nfs/README.txt
nfs shared file
```

- `FSTYPE` 이 `nfs4`, 옵션에 `vers=4.2` → **버전 협상 결과**가 v4.2

> 📝 **시험 포인트**: 필기 FULL r07-78 — NFS 사용 절차 순서: ① `/etc/exports` 작성 → ② `nfs-server` 기동 → ③ `exportfs -ra` 반영 → ④ 클라이언트 `mount -t nfs 서버:/공유 /mnt`. 순서를 섞은 선지가 오답.

### 4-7. root_squash 검증 — 소유자가 nobody 로 바뀌는가

> **상황**: 이 파트에서 가장 자주 출제되는 옵션을 눈으로 확인한다. 같은 디렉터리를 두 경로(대역 vs 루프백)로 내보냈으므로 차이를 바로 비교할 수 있다.

```bash
# ① 192.168.64.10 경유 = root_squash 적용
touch /mnt/nfs/rootfile-squashed
ls -l /mnt/nfs/rootfile-squashed
ls -l /srv/nfs/data/rootfile-squashed      # 서버 쪽 실제 소유자
id nobody
```

```text
# ls -l /mnt/nfs/rootfile-squashed
-rw-r--r--. 1 nobody nobody 0 ... rootfile-squashed
# id nobody
uid=65534(nobody) gid=65534(nobody) groups=65534(nobody)
```

- root 로 만들었는데 소유자가 **nobody(65534)** → `root_squash` 가 UID 0 을 익명으로 강등한 것

```bash
# ② 127.0.0.1 경유 = no_root_squash 적용
mkdir -p /mnt/nfs-local
mount -t nfs 127.0.0.1:/srv/nfs/data /mnt/nfs-local
touch /mnt/nfs-local/rootfile-noSquash
ls -l /srv/nfs/data/rootfile-noSquash
```

```text
-rw-r--r--. 1 root root 0 ... rootfile-noSquash
```

- 같은 서버·같은 디렉터리인데 **접속 경로(클라이언트 주소)에 따라 다른 옵션**이 적용됨 — `/etc/exports` 가 클라이언트별로 규칙을 갖는다는 증거

**검증**

```bash
ls -l /srv/nfs/data/
exportfs -v | grep -E 'root_squash'
umount /mnt/nfs-local
```

```text
-rw-r--r--. 1 nobody nobody 0 ... rootfile-squashed
-rw-r--r--. 1 root   root   0 ... rootfile-noSquash
-rw-r--r--. 1 root   root  16 ... README.txt
```

> 📝 **시험 포인트**: 실기 r04-14 서술형 — "`root_squash` = 클라이언트 root 를 nobody 로 매핑(기본), `no_root_squash` = root 권한 그대로 인정. 후자는 클라이언트를 장악한 공격자가 서버 파일을 root 권한으로 조작할 수 있어 위험". `all_squash` 는 **모든** 사용자를 익명화하는 별개 옵션.

### 4-8. fstab 등록 — _netdev 가 필요한 이유

> **상황**: 재부팅 후에도 자동 마운트되게 한다. 네트워크 파일시스템은 일반 디스크와 다른 옵션이 필요하다.

⚠️ fstab 오타는 부팅 실패로 이어짐(Part 07 9절 참조). 등록 후 반드시 `mount -a` 로 검증.

```bash
cp /etc/fstab /etc/fstab.bak-part09
echo '192.168.64.10:/srv/nfs/data  /mnt/nfs  nfs  defaults,_netdev  0 0' >> /etc/fstab
umount /mnt/nfs
mount -a
findmnt /mnt/nfs
```

- `_netdev` : **네트워크 장치**임을 표시 → systemd 가 `network-online.target` 이후에 마운트. 없으면 네트워크가 뜨기 전에 시도해 부팅이 지연·실패
- 5·6번째 필드 `0 0` : dump 대상 아님 / **fsck 순서 0**(원격 파일시스템은 fsck 하지 않음)
- 대안: `noauto,x-systemd.automount` — 실제 접근 시점에 마운트

**검증**

```bash
mount -a && echo 'fstab OK'
findmnt --verify --verbose 2>&1 | tail -5
df -hT | grep nfs
systemctl list-units --type=mount | grep -i mnt
```

```text
fstab OK
192.168.64.10:/srv/nfs/data  nfs4   36G ... /mnt/nfs
mnt-nfs.mount  loaded active mounted /mnt/nfs
```

> 📝 **시험 포인트**: 실기 r03-13 — fstab 6개 필드(장치 / 마운트지점 / 타입 / 옵션 / dump / fsck) 의미 서술. 네트워크 파일시스템은 **`_netdev` + fsck 0** 이 정답 포인트.

### 4-9. autofs — 요청 시 자동 마운트

> **상황**: 상시 마운트를 유지하면 서버 장애 시 클라이언트가 멈춘다. 접근할 때만 붙고 일정 시간 후 자동으로 떨어지는 autofs 로 바꿔 본다.

```bash
dnf install -y autofs
grep -vE '^#|^$' /etc/auto.master

# 마스터 맵에 항목 추가 — /mnt/auto 아래를 auto.nfs 가 관리
echo '/mnt/auto   /etc/auto.nfs   --timeout=60' >> /etc/auto.master

# 직접 맵 파일 작성: <키> <옵션> <서버:경로>
cat > /etc/auto.nfs <<'EOF'
data   -rw,sync,vers=4   192.168.64.10:/srv/nfs/data
EOF

systemctl enable --now autofs
systemctl status autofs --no-pager | head -5
```

- `/etc/auto.master` 형식: `<마운트 기준 디렉터리> <맵 파일> [옵션]`
  - 기본 포함 항목: `/misc /etc/auto.misc`, `+dir:/etc/auto.master.d`, `+auto.master`
  - `--timeout=<초>` : 유휴 시간 초과 시 자동 언마운트
- 맵 파일 형식: `<키> -<마운트옵션> <서버>:<경로>` → 실제 경로는 `<기준 디렉터리>/<키>`
- ⚠️ 기준 디렉터리(`/mnt/auto`)는 **직접 만들지 않는다** — autofs 가 관리

**검증**

```bash
ls /mnt/auto                      # 아직 비어 있음 (마운트 전)
mount | grep -c 192.168.64.10:/srv/nfs/data
ls /mnt/auto/data                 # 접근하는 순간 마운트됨
findmnt /mnt/auto/data
cat /mnt/auto/data/README.txt
# 60초 이상 방치 후 (다른 작업 하고 돌아와서)
findmnt /mnt/auto/data || echo 'auto-unmounted'
```

```text
# ls /mnt/auto
(비어 있음)
# ls /mnt/auto/data
README.txt  rootfile-noSquash  rootfile-squashed
# findmnt /mnt/auto/data
TARGET         SOURCE                      FSTYPE OPTIONS
/mnt/auto/data 192.168.64.10:/srv/nfs/data nfs4   rw,relatime,vers=4,...
# (60초 후)
auto-unmounted
```

- `/etc/auto.misc` 를 그대로 쓰면 `/misc/cd` 같은 예시 항목이 동작 — 마스터 맵의 `/misc /etc/auto.misc` 줄이 이미 존재

> 📝 **시험 포인트**: 필기 FULL r10-80 — autofs 는 "접근 시 자동 마운트, 유휴 시 자동 해제" 하는 데몬. "NFS 전용이라 로컬 디스크에는 못 쓴다" 는 오답(로컬 장치·CD 도 가능). 설정은 **마스터 맵(`/etc/auto.master`) + 직접 맵** 2단 구조.

### 4-10. NFSv3 vs v4 비교와 정리

> **상황**: 버전 차이가 필기에 나온다. 실제로 v3 로도 붙여 보고 표로 정리한다.

```bash
umount /mnt/nfs 2>/dev/null
mount -t nfs -o vers=3 192.168.64.10:/srv/nfs/data /mnt/nfs
findmnt -o TARGET,FSTYPE,OPTIONS /mnt/nfs
umount /mnt/nfs
mount -a                        # fstab 기준(v4)으로 원복
findmnt -o TARGET,FSTYPE,OPTIONS /mnt/nfs
```

| 항목 | NFSv3 | NFSv4 |
| --- | --- | --- |
| 포트 | 2049 + rpcbind 111 + mountd·statd·lockd **동적 포트** | **2049 단일** |
| `rpcbind` 의존 | **필수** | 불필요 |
| 전송 | UDP/TCP | **TCP** |
| 상태 | Stateless(별도 잠금 프로토콜 NLM) | **Stateful**(잠금·위임 내장) |
| 마운트 프로토콜 | 별도 MOUNT 프로토콜 | NFS 프로토콜에 통합 |
| 사용자 매핑 | UID/GID 숫자 | `user@domain` 문자열 (`idmapd`, `/etc/idmapd.conf`) |
| 방화벽 | 포트 고정 어려움 → 설정 복잡 | 2049 만 열면 됨 |
| ACL·보안 | 제한적 | ACL·Kerberos(`sec=krb5`) 지원 |

**검증**

```bash
findmnt -o TARGET,FSTYPE,OPTIONS /mnt/nfs
grep -E 'vers' /proc/mounts | grep nfs
nfsstat -m | head -6
```

```text
TARGET   FSTYPE OPTIONS
/mnt/nfs nfs4   rw,relatime,vers=4.2,...
```

> 📝 **시험 포인트**: 필기 FULL r03-76 — 정답은 "NFSv4 는 단일 TCP 2049 로 동작해 rpcbind·mountd 포트 의존이 줄었다". `nfsstat -c`(클라이언트) / `-s`(서버) / `-m`(마운트) 구분도 출제.

---

## 5. Samba — 윈도·macOS 파일 공유

### 5-1. 설치와 데몬 역할

> **상황**: Part 05 에서 만든 LVM 볼륨 `/srv/share` 를 개발팀 공유로 내보낸다. 윈도·macOS 에서 바로 붙을 수 있어야 하므로 SMB/CIFS 를 쓴다.

```bash
dnf install -y samba samba-client samba-common-tools cifs-utils
rpm -ql samba | grep -E 'sbin|/etc/samba' | head
ls -l /etc/samba/
```

- `samba` : 서버 데몬 `smbd`·`nmbd`
- `samba-client` : `smbclient`·`smbtar` 등 클라이언트 도구
- `samba-common-tools` : `testparm`·`smbpasswd`·`nmblookup`·`smbstatus`
- `cifs-utils` : `mount.cifs` — 리눅스에서 SMB 공유를 **마운트**할 때 필요

| 데몬(유닛) | 역할 | 포트 |
| --- | --- | --- |
| `smbd` (`smb.service`) | **파일·프린터 공유, 사용자 인증** — SMB 프로토콜 본체 | 445/TCP(직접 호스팅), 139/TCP(NetBIOS 경유) |
| `nmbd` (`nmb.service`) | **NetBIOS 이름 해석·브라우징**(네트워크 환경 목록) | 137/UDP(이름), 138/UDP(데이터그램) |
| `winbindd` (`winbind.service`) | AD/도메인 사용자·그룹 정보 연동 | — |

**검증**

```bash
rpm -q samba samba-client samba-common-tools cifs-utils
ls /etc/samba/smb.conf
systemctl list-unit-files | grep -E '^(smb|nmb|winbind)\.service'
```

```text
samba-4.x...  samba-client-4.x...  samba-common-tools-4.x...  cifs-utils-...
smb.service      disabled  disabled
nmb.service      disabled  disabled
winbind.service  disabled  disabled
```

> 📝 **시험 포인트**: 필기 FULL r01-82, r07-81, r10-81 — **`smbd` = 파일·프린터 공유와 인증, `nmbd` = NetBIOS 이름 해석·브라우징**. 둘을 바꾼 선지가 최빈출. "`smbclient` 가 비밀번호를 등록한다" 도 오답(그건 `smbpasswd`).

### 5-2. smb.conf 구조 — [global] 섹션

> **상황**: 설정 파일의 섹션 구조를 파악하고 전역 설정을 사내용으로 조정한다.

```bash
cp /etc/samba/smb.conf /etc/samba/smb.conf.orig
grep -vE '^\s*[#;]|^\s*$' /etc/samba/smb.conf     # 실제 유효 설정만
vi /etc/samba/smb.conf
```

```ini
[global]
        workgroup = LABGROUP
        server string = LAB09 Samba Server (%h)
        netbios name = SRV01
        security = user
        passdb backend = tdbsam
        map to guest = Bad User
        hosts allow = 192.168.64. 127.
        log file = /var/log/samba/log.%m
        max log size = 1000
        logging = file
        server role = standalone server
        printing = cups
        printcap name = cups
        load printers = yes
        cups options = raw
```

| 섹션 | 성격 |
| --- | --- |
| `[global]` | 서버 전역 설정. 유일 |
| `[homes]` | **특수 섹션** — 접속한 사용자의 홈 디렉터리를 자동 공유(공유명 = 사용자명) |
| `[printers]` | **특수 섹션** — 등록된 프린터를 자동 공유 |
| `[<임의이름>]` | 일반 공유 정의 |

| `[global]` 키 | 의미 |
| --- | --- |
| `workgroup` | 윈도 작업 그룹(또는 도메인) 이름 |
| `server string` | 네트워크 탐색기에 보이는 설명. `%h`(호스트명) `%v`(버전) 치환 |
| `netbios name` | NetBIOS 상 서버 이름(최대 15자, 대문자 관례) |
| `security = user` | **사용자 단위 인증** — 접속 시 계정/비밀번호 요구. 기본값이자 표준 |
| `security = share` | 공유 단위 인증 — **SMB1 시절 방식, 현재 제거됨**(개념 문제로만 출제) |
| `passdb backend` | 암호 DB 방식 — `tdbsam`(로컬 TDB, 기본) · `ldapsam`(LDAP) · `smbpasswd`(구식) |
| `map to guest` | 인증 실패 처리 — `Never`(거부) · `Bad User`(없는 계정은 게스트로) · `Bad Password` |
| `hosts allow` / `hosts deny` | 접속 허용·거부 대역. `192.168.64.` 처럼 **끝에 점**을 찍으면 그 대역 전체 |
| `log file` / `max log size` | 로그 경로(`%m` = 클라이언트명) · KB 단위 최대 크기 |
| `interfaces` / `bind interfaces only` | 수신 인터페이스 제한 |

**검증**

```bash
testparm -s 2>/dev/null | head -20
grep -nE 'workgroup|security|netbios|hosts allow' /etc/samba/smb.conf
```

```text
[global]
	netbios name = SRV01
	server string = LAB09 Samba Server (%h)
	workgroup = LABGROUP
	hosts allow = 192.168.64., 127.
	log file = /var/log/samba/log.%m
	max log size = 1000
	security = USER
```

> 📝 **시험 포인트**: 필기 FULL r05-81 — `security = user` 는 "공유 접근 시 사용자 계정·비밀번호로 인증". `[homes]`·`[printers]` 는 예약된 특수 섹션이라는 점도 출제(r10-82 의 "`[shared]` 는 프린터 전용 예약 섹션" 은 오답).

### 5-3. [share] 공유 섹션 작성

> **상황**: `/srv/share` 를 개발팀(`devteam`)만 쓰는 쓰기 가능 공유로 만든다.

```bash
cat >> /etc/samba/smb.conf <<'EOF'

[share]
        comment = Dev Team Shared Folder
        path = /srv/share
        browseable = yes
        writable = yes
        valid users = @devteam
        write list = @devteam
        create mask = 0664
        directory mask = 2775
        force group = devteam
        guest ok = no
EOF
```

| 키 | 의미 |
| --- | --- |
| `comment` | 탐색기에 보이는 설명 |
| `path` | **실제 서버 디렉터리**. 공유 이름(`[share]`)과 별개 |
| `browseable = yes` | 공유 목록에 노출. `no` 면 이름을 직접 입력해야 접근 |
| `writable = yes` | 쓰기 허용. **`read only = no` 와 완전히 동일**(반대 표기) |
| `read only = yes` | 읽기 전용 (= `writable = no`) |
| `valid users` | 접근 **허용** 사용자·그룹 목록. `@그룹` 또는 `+그룹` 은 그룹 지정 |
| `invalid users` | 접근 거부 목록 |
| `write list` | `read only = yes` 상태에서도 이 목록은 쓰기 가능 |
| `create mask` | 새 **파일** 권한 상한(8진수). `0664` → rw-rw-r-- |
| `directory mask` | 새 **디렉터리** 권한 상한. `2775` → SetGID 포함 |
| `force group` | 생성 파일의 그룹을 강제 지정 |
| `force user` | 생성 파일의 소유자를 강제 지정 |
| `guest ok = no` | 게스트(비인증) 접근 차단 (= `public = no`) |
| `veto files` / `hide files` | 특정 패턴 파일 숨김·차단 |

- ⚠️ `writable` 과 `read only` 는 **의미가 반대인 동의 키** — 둘을 같이 쓰면 뒤에 나온 값이 이김. 시험에서 "`writable = yes` = `read only = no`" 를 묻는다

**검증**

```bash
testparm -s 2>/dev/null | sed -n '/\[share\]/,$p'
```

```text
[share]
	comment = Dev Team Shared Folder
	path = /srv/share
	valid users = @devteam
	write list = @devteam
	force group = devteam
	read only = No
	create mask = 0664
	directory mask = 02775
```

- `testparm` 출력에서 `writable = yes` 가 **`read only = No`** 로 정규화되어 나오는 것을 확인

> 📝 **시험 포인트**: 필기 FULL r01-81, r02-81, r03-79, r04-81, r05-82, r08-80, r09-84, r10-82 — `valid users` 는 **허용** 목록(차단 목록 아님), `@devteam` 은 그룹. "공유 이름은 `path` 값이다" 는 오답 — 공유 이름은 대괄호 안의 섹션명.

### 5-4. testparm 으로 문법 검사

> **상황**: 데몬을 올리기 전에 설정 오류를 잡는다.

```bash
testparm                       # 대화식: Enter 치면 전체 설정 덤프
testparm -s                    # 덤프까지 한 번에 (suppress prompt)
testparm -s -v | head -30      # 기본값 포함 전체 파라미터 (verbose)
testparm --parameter-name='valid users' --section-name=share
```

- `-s` : 프롬프트 없이 즉시 출력 (**s**uppress)
- `-v` : 명시하지 않은 **기본값까지 전부** 표시 (**v**erbose) — "이 값의 기본은?" 확인용
- `--section-name` / `--parameter-name` : 특정 값만 조회

**검증**

```bash
testparm 2>&1 | grep -E 'Loaded services file|Server role|ERROR|WARNING'
```

```text
Loaded services file OK.
Server role: ROLE_STANDALONE
```

- `Loaded services file OK.` 가 나와야 정상. 오류가 있으면 줄 번호와 함께 표시

> 📝 **시험 포인트**: 필기 FULL r02-82, r04-80, r05-83, r10-82 — `smb.conf` 문법 검사는 **`testparm`**. 오답 단골: `smbpasswd`(계정 등록), `smbclient`(접속), `smbstatus`(접속 현황), `nmblookup`(이름 조회).

### 5-5. Samba 계정 등록 — smbpasswd · pdbedit

> **상황**: Samba 는 리눅스 비밀번호와 **별도의 암호 DB** 를 쓴다. `dev1`·`dev2` 를 등록한다.

```bash
smbpasswd -a dev1               # 대화식으로 새 SMB 비밀번호 2회 입력
smbpasswd -a dev2
pdbedit -L                      # 등록된 Samba 사용자 목록
pdbedit -L -v | head -20        # 상세 (계정 플래그·마지막 변경 시각 등)
```

- `-a` : 사용자 **a**dd — ⚠️ **리눅스 계정이 먼저 존재해야 함**(Part 03 에서 생성됨)
- `-x` : 삭제 (**x** = delete)
- `-d` : 비활성화 (**d**isable), `-e` : 활성화 (**e**nable)
- `-n` : 비밀번호 없음으로 설정
- 인자 없이 실행하면 **자기 자신의 SMB 비밀번호 변경**
- 저장 위치: `passdb backend = tdbsam` → `/var/lib/samba/private/passdb.tdb`

```bash
# 비활성화·재활성화 실습
smbpasswd -d dev2
pdbedit -L -v | grep -A1 'Unix username:.*dev2' | grep 'Account Flags'
smbpasswd -e dev2
```

**검증**

```bash
pdbedit -L
pdbedit -L -v | grep -E 'Unix username|Account Flags'
ls -l /var/lib/samba/private/passdb.tdb
# 리눅스 비밀번호와 별개임을 확인 — SMB 비번을 바꿔도 로그인 비번은 그대로
grep '^dev1:' /etc/shadow | cut -c1-25
```

```text
# pdbedit -L
dev1:2001:
dev2:2002:
# pdbedit -L -v | grep -E 'Unix username|Account Flags'
Unix username:        dev1
Account Flags:        [U          ]
Unix username:        dev2
Account Flags:        [U          ]
```

- `[U]` = 일반 사용자 계정 활성. `[UD]` = 비활성(**D**isabled)

> 📝 **시험 포인트**: 필기 FULL r03-78, r06-78, r07-80 — Samba 사용자 등록은 `smbpasswd -a <계정>`, 목록 확인은 `pdbedit -L`. "`smbpasswd` 가 리눅스 로그인 비밀번호를 바꾼다" 는 오답 — **SMB 전용 암호 DB** 를 다룬다.

### 5-6. 공유 디렉터리 권한과 SELinux

> **상황**: 파일시스템 권한과 SELinux 를 맞추지 않으면 인증에 성공해도 "접근 거부" 가 난다. 두 가지 SELinux 해법의 차이를 함께 확인한다.

```bash
chgrp devteam /srv/share
chmod 2775 /srv/share
ls -ldZ /srv/share
```

**방법 A — 불리언으로 통째 허용**

```bash
getsebool -a | grep samba
setsebool -P samba_export_all_rw on
getsebool samba_export_all_rw
```

- `samba_export_all_rw` : Samba 가 **모든 경로**를 읽기·쓰기로 공유하도록 허용하는 불리언
- `-P` : **P**ersistent — 재부팅 후에도 유지. 없으면 임시
- 장점: 즉시 효과. 단점: **범위가 넓어 보안상 느슨함**

**방법 B — 경로에 전용 라벨 부여 (권장)**

```bash
setsebool -P samba_export_all_rw off        # A 를 되돌리고 B 로
semanage fcontext -a -t samba_share_t "/srv/share(/.*)?"
restorecon -Rv /srv/share
ls -ldZ /srv/share
```

- `samba_share_t` : Samba 가 읽고 쓸 수 있는 **전용 타입**
- 장점: 해당 경로만 정확히 허용. 단점: 경로마다 규칙 추가 필요
- 상세는 → [[10-security-firewall-selinux]]

| 구분 | 방법 A `setsebool -P samba_export_all_rw on` | 방법 B `semanage fcontext … samba_share_t` |
| --- | --- | --- |
| 적용 범위 | 서버의 **모든 디렉터리** | **지정 경로만** |
| 저장 위치 | 불리언 정책 상태 | `file_contexts.local` + 파일 라벨 |
| 되돌리기 | `setsebool -P … off` | `semanage fcontext -d` + `restorecon` |
| 권장 | 임시·테스트 | **운영** |

**검증**

```bash
ls -ldZ /srv/share
getsebool samba_export_all_rw samba_enable_home_dirs
semanage fcontext -l -C | grep srv
```

```text
drwxrwsr-x. 4 root devteam unconfined_u:object_r:samba_share_t:s0 ... /srv/share
samba_export_all_rw --> off
samba_enable_home_dirs --> off
/srv/share(/.*)?    all files   system_u:object_r:samba_share_t:s0
/srv/www(/.*)?      all files   system_u:object_r:httpd_sys_content_t:s0
```

- `[homes]` 로 홈 디렉터리를 공유하려면 별도로 `setsebool -P samba_enable_home_dirs on` 필요

> 📝 **시험 포인트**: SELinux 불리언은 `getsebool -a | grep <서비스>` 로 찾고 `setsebool -P` 로 영구 적용. `-P` 빠뜨리면 재부팅 후 원복 — 필기 FULL r09-59 의 정답 포인트.

### 5-7. 기동·부팅 등록·방화벽

> **상황**: smbd·nmbd 를 함께 올리고 SMB 포트를 연다.

```bash
systemctl enable --now smb nmb
systemctl is-active smb nmb
ss -tulnp | grep -E ':445|:139|:137|:138'
firewall-cmd --permanent --add-service=samba
firewall-cmd --reload
firewall-cmd --info-service=samba
```

- `samba` firewalld 서비스 = 137·138/UDP + 139·445/TCP 를 한 번에 개방
- `samba-client` 서비스 정의는 클라이언트 측 브로드캐스트 응답용(여기서는 불필요)

**검증**

```bash
systemctl is-enabled smb nmb
ss -tlnp | grep smbd
ss -ulnp | grep nmbd
firewall-cmd --info-service=samba
```

```text
# ss -tlnp | grep smbd
LISTEN 0 50 *:445 *:* users:(("smbd",pid=...,fd=...))
LISTEN 0 50 *:139 *:* users:(("smbd",pid=...,fd=...))
# ss -ulnp | grep nmbd
UNCONN 0 0 192.168.64.255:137 0.0.0.0:* users:(("nmbd",...))
UNCONN 0 0      0.0.0.0:138 0.0.0.0:* users:(("nmbd",...))
# firewall-cmd --info-service=samba
samba
  ports: 137/udp 138/udp 139/tcp 445/tcp
```

> 📝 **시험 포인트**: 포트 4종 암기 — **137/UDP(이름 서비스) · 138/UDP(데이터그램) · 139/TCP(NetBIOS 세션) · 445/TCP(직접 호스팅 SMB)**. 유닛명은 `smb`·`nmb` 이고 데몬명은 `smbd`·`nmbd`.

### 5-8. smbclient 로 접속 검증

> **상황**: 마운트 전에 프로토콜 수준에서 붙는지 확인한다. FTP 와 비슷한 대화식 셸을 제공한다.

```bash
smbclient -L //127.0.0.1 -U dev1            # 공유 목록 (비밀번호 대화식)
smbclient -L //192.168.64.10 -U dev1%'<비밀번호>'   # 비대화식(히스토리 주의 ⚠️)
smbclient -L //127.0.0.1 -N                 # 익명(No password)
smbclient //127.0.0.1/share -U dev1         # 공유에 접속
```

- `-L <서버>` : 공유 목록 조회 (**L**ist)
- `-U <사용자>` : 접속 계정 (**U**ser). `사용자%비밀번호` 로 붙여 쓸 수 있으나 히스토리에 남음
- `-N` : 비밀번호 없이 (**N**o pass)
- `-c '<명령>'` : 대화식 대신 명령 1회 실행
- `-m SMB3` : 프로토콜 버전 지정

**대화식 명령**

```text
smb: \> ls                       # 목록
smb: \> pwd                      # 현재 위치
smb: \> mkdir testdir            # 디렉터리 생성
smb: \> lcd /tmp                 # 로컬 디렉터리 변경
smb: \> put hosts.txt            # 업로드
smb: \> get README.txt           # 다운로드
smb: \> rm hosts.txt             # 삭제
smb: \> cd testdir
smb: \> rmdir testdir
smb: \> help
smb: \> exit                     # 종료 (quit 도 동일)
```

**검증**

```bash
echo 'smb upload test' > /tmp/smbtest.txt
smbclient //127.0.0.1/share -U dev1 -c 'lcd /tmp; put smbtest.txt; ls'
ls -l /srv/share/smbtest.txt
smbclient -L //127.0.0.1 -U dev1 2>/dev/null | head -12
```

```text
# smbclient ... -c 'put smbtest.txt; ls'
putting file smbtest.txt as \smbtest.txt (... kb/s)
  .                                   D        0  ...
  ..                                  D        0  ...
  smbtest.txt                         A       16  ...
# ls -l /srv/share/smbtest.txt
-rw-rw-r--. 1 dev1 devteam 16 ... /srv/share/smbtest.txt
# smbclient -L //127.0.0.1 -U dev1
	Sharename       Type      Comment
	---------       ----      -------
	share           Disk      Dev Team Shared Folder
	IPC$            IPC       IPC Service (LAB09 Samba Server (srv01))
	dev1            Disk      Home Directories
```

- 업로드한 파일의 그룹이 **devteam**, 권한이 **664** → `create mask`·`force group` 이 그대로 적용됨

> 📝 **시험 포인트**: 필기 FULL r04-79, r07-80 — 공유 목록 확인은 `smbclient -L //<서버>`, 접속은 `smbclient //<서버>/<공유> -U <계정>`. 구축 절차 순서: smb.conf 작성 → `testparm` → `smbpasswd -a` + 데몬 기동 → 클라이언트 `smbclient`.

### 5-9. cifs 마운트와 SetGID 상속 검증

> **상황**: 리눅스 클라이언트에서 실제 파일시스템처럼 쓰도록 마운트한다.

```bash
mkdir -p /mnt/smb
mount -t cifs //192.168.64.10/share /mnt/smb \
  -o username=dev1,uid=dev1,gid=devteam
df -hT /mnt/smb
mount | grep cifs
```

- `-t cifs` : `cifs-utils` 의 `mount.cifs` 사용
- `username=` : SMB 인증 계정 (비밀번호는 대화식 입력)
- `uid=` / `gid=` : **클라이언트 쪽에서 보이는** 소유자·그룹 (SMB 는 POSIX UID 를 전달하지 않음)
- `vers=3.0` : 프로토콜 버전 고정
- `credentials=<파일>` : 계정 정보를 파일에서 읽기 (5-10)
- `ro` / `rw` / `noperm` / `dir_mode=` / `file_mode=` : 접근 제어

**검증 — 서버 쪽 소유·SetGID 상속 확인**

```bash
touch /mnt/smb/from-client.txt
mkdir /mnt/smb/from-client-dir
ls -l /mnt/smb/from-client.txt              # 클라이언트 시각 (uid/gid 옵션대로)
ls -l /srv/share/from-client.txt            # 서버 실제 소유자
ls -ld /srv/share/from-client-dir           # SetGID 상속 확인
stat -c '%A %U %G %n' /srv/share/from-client.txt /srv/share/from-client-dir
```

```text
# ls -l /srv/share/from-client.txt
-rw-rw-r--. 1 dev1 devteam 0 ... from-client.txt
# ls -ld /srv/share/from-client-dir
drwxrwsr-x. 2 dev1 devteam ... from-client-dir
# stat -c '%A %U %G %n' ...
-rw-rw-r-- dev1 devteam /srv/share/from-client.txt
drwxrwsr-x dev1 devteam /srv/share/from-client-dir
```

- 디렉터리에 `s`(SetGID)가 붙어 있음 → `directory mask = 2775` 가 적용되어 하위 생성물도 계속 `devteam` 그룹을 유지

> 📝 **시험 포인트**: SMB 마운트는 `mount -t cifs`, NFS 는 `mount -t nfs`. `uid=`/`gid=` 는 **클라이언트 표시용**일 뿐 서버 권한을 바꾸지 않는다 — 서버 쪽 실제 소유자는 인증 계정(`dev1`)과 `force group` 이 결정.

### 5-10. credentials 파일과 fstab 자동 마운트

> **상황**: 부팅 시 자동 마운트해야 하는데 fstab 에 비밀번호를 평문으로 쓸 수는 없다. 별도 자격 파일로 분리한다.

⚠️ 자격 파일은 **600 권한 + root 소유** 필수. 그렇지 않으면 다른 사용자가 비밀번호를 읽을 수 있음.

```bash
cat > /root/.smbcred <<'EOF'
username=dev1
password=<비밀번호>
EOF
chmod 600 /root/.smbcred
chown root:root /root/.smbcred
ls -l /root/.smbcred

umount /mnt/smb
echo '//192.168.64.10/share  /mnt/smb  cifs  credentials=/root/.smbcred,uid=dev1,gid=devteam,_netdev  0 0' >> /etc/fstab
mount -a
```

- 자격 파일 형식: `username=` / `password=` / `domain=` 각 한 줄
- `_netdev` : 네트워크 준비 후 마운트 (NFS 와 동일한 이유)
- `<비밀번호>` 는 자리표시자 — 실제 값으로 교체

**검증**

```bash
mount -a && echo 'fstab cifs OK'
findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS /mnt/smb
ls -l /mnt/smb | head -5
stat -c '%a %U' /root/.smbcred
grep smb /etc/fstab
```

```text
fstab cifs OK
TARGET   SOURCE                   FSTYPE OPTIONS
/mnt/smb //192.168.64.10/share    cifs   rw,relatime,vers=3.1.1,cache=strict,username=dev1,uid=...,gid=...
600 root
```

> 📝 **시험 포인트**: fstab 에 평문 비밀번호를 쓰지 않고 `credentials=<파일>` 로 분리하는 것이 표준. 파일 권한 600 은 반드시 함께 언급되는 채점 포인트.

### 5-11. hosts allow 접근 제어 검증

> **상황**: `[global]` 의 `hosts allow` 가 실제로 동작하는지 확인한다. RHEL 9 에서 TCP Wrapper 는 없어졌지만 Samba 는 **자체 구현**으로 같은 형식을 지원한다.

```bash
testparm -s 2>/dev/null | grep 'hosts allow'
# 일부러 루프백만 차단해 거부되는지 확인
sed -i 's/^        hosts allow = .*/        hosts allow = 192.168.64./' /etc/samba/smb.conf
testparm -s >/dev/null && systemctl reload smb
smbclient -L //127.0.0.1 -U dev1 2>&1 | tail -2
smbclient -L //192.168.64.10 -U dev1 2>/dev/null | grep -c share
```

```text
# smbclient -L //127.0.0.1 -U dev1
session setup failed: NT_STATUS_CONNECTION_REFUSED
# smbclient -L //192.168.64.10 ...
1
```

```bash
# 원복
sed -i 's/^        hosts allow = .*/        hosts allow = 192.168.64. 127./' /etc/samba/smb.conf
testparm -s >/dev/null && systemctl reload smb
```

- `hosts allow = 192.168.64. 127.` : 끝에 점을 찍으면 **접두 대역** 전체. `192.168.64.0/24` 표기도 가능
- `hosts deny` 와 함께 쓰면 **allow 가 우선**
- 공유 섹션에 따로 쓰면 그 공유에만 적용
- TCP Wrapper(`/etc/hosts.allow`·`hosts.deny`) 는 RHEL 9 에서 **제거됨** → [[10-security-firewall-selinux]] 참조. Samba 의 `hosts allow` 는 이름만 같고 별개 기능

**검증**

```bash
testparm -s 2>/dev/null | grep 'hosts allow'
smbclient -L //127.0.0.1 -U dev1 2>/dev/null | grep -c share
```

```text
	hosts allow = 192.168.64., 127.
1
```

> 📝 **시험 포인트**: 실기 r03-14·r06-14 는 TCP Wrapper 의 `hosts.allow`/`hosts.deny` 검사 순서(allow 먼저 → 매칭되면 허용, 아니면 deny 검사, 둘 다 없으면 허용)를 묻는다. **RHEL 9 에서는 라이브러리가 빠져 동작하지 않으며**, Samba·일부 응용의 자체 구현만 남았다는 점을 함께 알아 둘 것.

### 5-12. 상태 조회·로그·macOS 접속

> **상황**: 접속 현황과 로그 위치를 확인하고, macOS Finder 로도 붙어 본다.

```bash
smbstatus                      # 전체 (버전·세션·공유·잠금)
smbstatus -b                   # 세션(브리프)만
smbstatus -S                   # 공유 연결만
smbstatus -L                   # 잠긴 파일
nmblookup -A 192.168.64.10     # IP → NetBIOS 이름 (역방향)
nmblookup SRV01                # 이름 → IP
ls -l /var/log/samba/
tail -20 /var/log/samba/log.smbd
```

- `smbstatus -b` : **b**rief, `-S` : **S**hares, `-L` : **L**ocks
- `nmblookup -A <IP>` : 해당 IP 의 NetBIOS 이름 테이블 조회 (**A** = node status)
- 로그: `/var/log/samba/log.smbd`, `log.nmbd`, 클라이언트별 `log.<클라이언트명>`

**macOS Finder 에서 접속**

```text
Finder → 이동(Go) → 서버에 연결(⌘K) → smb://192.168.64.10/share
→ "등록된 사용자" 선택 → 이름 dev1 / 암호 <SMB 비밀번호>
```

- macOS 는 SMB2/3 을 사용 → `[global]` 에 별도 설정 없이 접속 가능
- 실패 시 확인 순서: ① `firewall-cmd --list-services` 에 `samba` ② `hosts allow` 대역 ③ `pdbedit -L` 계정 ④ SELinux `ls -ldZ /srv/share`

**검증**

```bash
smbstatus -b
smbstatus -S
nmblookup -A 192.168.64.10
ls /var/log/samba/
```

```text
# smbstatus -b
Samba version 4.x
PID  Username  Group    Machine            Protocol Version  Encryption  Signing
---------------------------------------------------------------------------------
...  dev1      devteam  192.168.64.10 (...)  SMB3_11         -           partial(...)
# smbstatus -S
Service    pid   Machine        Connected at        Encryption  Signing
---------------------------------------------------------------------
share      ...   192.168.64.10  ...                 -           -
# nmblookup -A 192.168.64.10
Looking up status of 192.168.64.10
	SRV01           <00> -         B <ACTIVE>
	LABGROUP        <00> - <GROUP> B <ACTIVE>
```

> 📝 **시험 포인트**: `smbstatus`(접속 현황) · `testparm`(문법) · `smbpasswd`(계정) · `smbclient`(접속) · `nmblookup`(이름 조회) 5개의 역할 구분이 그대로 선지로 나온다.

---

## 6. vsftpd — FTP

### 6-1. 설치와 설정 키 총정리

> **상황**: 외주 인력이 파일을 반입할 FTP 를 연다. 익명 접속은 기본 차단하고 로컬 계정만 쓰되, 홈 디렉터리 밖으로 못 나가게 chroot 를 건다.

```bash
dnf install -y vsftpd lftp
dnf install -y ftp                    # 고전 ftp 클라이언트 (기본 저장소에 없으면 EPEL 필요)
rpm -ql vsftpd | grep -E '^/etc'
cp /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.orig
```

- `vsftpd` : **v**ery **s**ecure **FTP** **d**aemon — RHEL 계열 기본 FTP 서버
- `lftp` : 스크립팅에 강한 고급 FTP/HTTP 클라이언트
- `ftp` : 고전 대화식 클라이언트. RHEL 9 기본 저장소에 없을 수 있음 → 없으면 `lftp`·`curl` 로 대체(6-4 에 둘 다 제시). EPEL 등록은 Part 02 참조

| 파일 | 역할 |
| --- | --- |
| `/etc/vsftpd/vsftpd.conf` | 주 설정 |
| `/etc/vsftpd/ftpusers` | **접속 무조건 거부** 계정 목록 (PAM 이 검사) |
| `/etc/vsftpd/user_list` | `userlist_enable`·`userlist_deny` 값에 따라 **거부 또는 허용** 목록 |
| `/etc/vsftpd/chroot_list` | chroot 예외(또는 대상) 목록 |
| `/var/ftp/pub` | 익명 FTP 기본 디렉터리 |
| `/var/log/xferlog` | 전송 로그(표준 형식) |
| `/var/log/vsftpd.log` | vsftpd 자체 로그 |

| 설정 키 | 의미 |
| --- | --- |
| `anonymous_enable=NO` | **익명(anonymous·ftp) 접속 차단** |
| `local_enable=YES` | 로컬 시스템 계정 로그인 허용 |
| `write_enable=YES` | 쓰기 명령(STOR/DELE/MKD 등) 허용 |
| `local_umask=022` | 로컬 사용자 업로드 파일의 umask |
| `anon_upload_enable=NO` | 익명 업로드 허용 여부 |
| `anon_mkdir_write_enable=NO` | 익명 디렉터리 생성 허용 여부 |
| `anon_umask=077` | 익명 업로드 umask |
| `dirmessage_enable=YES` | 디렉터리 진입 시 `.message` 파일 내용 표시 |
| `xferlog_enable=YES` | 전송 로그 기록 |
| `xferlog_file=/var/log/xferlog` | 전송 로그 경로 |
| `xferlog_std_format=YES` | **표준 xferlog 형식** 사용 |
| `connect_from_port_20=YES` | 능동 모드 데이터 연결을 **20번 포트**에서 개시 |
| `chroot_local_user=YES` | 로컬 사용자를 **홈 디렉터리에 가둠** |
| `allow_writeable_chroot=YES` | chroot 최상위가 쓰기 가능해도 허용(없으면 로그인 오류) |
| `chroot_list_enable=YES` | 예외 목록 기능 활성 |
| `chroot_list_file=/etc/vsftpd/chroot_list` | 예외 목록 경로 |
| `userlist_enable=YES` | `user_list` 파일 사용 |
| `userlist_deny=YES` | `user_list` 를 **거부** 목록으로(기본). `NO` 면 **허용 전용** 목록 |
| `userlist_file=/etc/vsftpd/user_list` | 목록 파일 경로 |
| `listen=YES` / `listen_ipv6=YES` | 독립 실행 모드 수신(IPv4 / IPv6). **둘 다 YES 로 두면 기동 실패** |
| `pasv_enable=YES` | 수동 모드 허용 |
| `pasv_min_port` / `pasv_max_port` | 수동 모드 데이터 포트 범위 |
| `max_clients` / `max_per_ip` | 최대 동시 접속 수 / IP 당 최대 |
| `idle_session_timeout` / `data_connection_timeout` | 유휴·데이터 타임아웃(초) |
| `ftpd_banner="문자열"` | 접속 시 배너 |
| `local_root=<경로>` | 로컬 사용자 로그인 시작 디렉터리 |
| `user_config_dir=<경로>` | 사용자별 개별 설정 디렉터리 |
| `ascii_upload_enable` / `ascii_download_enable` | ASCII 모드 전송 허용 |

**검증**

```bash
grep -vE '^#|^$' /etc/vsftpd/vsftpd.conf
ls -l /etc/vsftpd/
```

```text
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
listen=NO
listen_ipv6=YES
pam_service_name=vsftpd
userlist_enable=YES
```

> 📝 **시험 포인트**: 실기 r06-10 — "익명 접속 차단" 은 `anonymous_enable=NO`. 필기 FULL r01-83, r02-83, r04-82, r05-84, r07-83, r08-82, r09-82 에서 `anonymous_enable`·`local_enable`·`write_enable`·`chroot_local_user` 조합 해석이 반복 출제.

### 6-2. 설정 편집 — chroot·수동 포트·배너

> **상황**: 사내 정책에 맞춰 chroot 를 켜고, 방화벽에서 열 수 있도록 수동 모드 포트 범위를 고정한다.

```bash
cat >> /etc/vsftpd/vsftpd.conf <<'EOF'

# ===== LAB 09 =====
chroot_local_user=YES
allow_writeable_chroot=YES
ftpd_banner=Welcome to LAB09 FTP (srv01.lab.local)
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100
max_clients=20
max_per_ip=5
idle_session_timeout=300
use_localtime=YES
EOF
grep -nE 'chroot|pasv|banner|max_' /etc/vsftpd/vsftpd.conf
```

- `chroot_local_user=YES` 단독으로는 홈이 쓰기 가능할 때 `500 OOPS: vsftpd: refusing to run with writable root inside chroot()` 로 로그인 실패 → `allow_writeable_chroot=YES` 를 함께 지정
- `pasv_min_port`~`pasv_max_port` : 이 범위만 방화벽에서 열면 되므로 운영이 단순해짐

**검증**

```bash
grep -c '^' /etc/vsftpd/vsftpd.conf
grep -E '^(chroot_local_user|allow_writeable_chroot|pasv_min_port|pasv_max_port)' /etc/vsftpd/vsftpd.conf
```

```text
chroot_local_user=YES
allow_writeable_chroot=YES
pasv_min_port=40000
pasv_max_port=40100
```

> 📝 **시험 포인트**: `chroot_local_user=YES` = 로컬 사용자를 홈에 가둬 **상위 디렉터리 이동 차단**. 필기에서 "보안 목적으로 홈 상위 접근을 막는 설정" 으로 그대로 출제.

### 6-3. 기동·방화벽·SELinux

> **상황**: 데몬을 올리고 제어 포트 21 + 수동 데이터 포트 범위를 연다.

```bash
systemctl enable --now vsftpd
systemctl is-active vsftpd
ss -tlnp | grep :21
firewall-cmd --permanent --add-service=ftp
firewall-cmd --permanent --add-port=40000-40100/tcp
firewall-cmd --reload

getsebool -a | grep ftp
setsebool -P ftpd_full_access on
```

- `firewall-cmd --add-service=ftp` : 21/TCP 개방 + `nf_conntrack_ftp` 헬퍼 연동(능동 모드 대응)
- `--add-port=40000-40100/tcp` : **수동 모드 데이터 포트 범위** — 이걸 빠뜨리면 로그인은 되는데 `ls` 에서 멈춤
- `ftpd_full_access` : vsftpd 가 홈·공유 디렉터리를 읽고 **쓸 수** 있게 허용하는 불리언
- `ftp_home_dir` : 홈 디렉터리 읽기·쓰기만 허용하는 좁은 불리언(구버전 이름)
- SELinux 상세 → [[10-security-firewall-selinux]]

**검증**

```bash
systemctl is-enabled vsftpd
ss -tlnp | grep vsftpd
getsebool ftpd_full_access
firewall-cmd --list-services; firewall-cmd --list-ports
```

```text
enabled
LISTEN 0 32 *:21 *:* users:(("vsftpd",pid=...,fd=3))
ftpd_full_access --> on
... ftp http https samba dns nfs ...
8080/tcp 40000-40100/tcp
```

> 📝 **시험 포인트**: FTP 는 제어(21)와 데이터(20 또는 수동 임의 포트) **연결 2개**를 쓴다. 방화벽 환경에서 "로그인은 되는데 목록이 안 나온다" = 데이터 연결 차단 → 수동 포트 범위 개방이 정답.

### 6-4. ftp 대화식 접속 검증

> **상황**: 실제로 로그인해서 올리고 내려받는다. 시험에서 대화식 명령 이름을 그대로 묻는다.

```bash
cd /tmp && echo 'ftp upload test' > /tmp/ftptest.txt
ftp 127.0.0.1
```

```text
Connected to 127.0.0.1 (127.0.0.1).
220 Welcome to LAB09 FTP (srv01.lab.local)
Name (127.0.0.1:root): dev1
331 Please specify the password.
Password:
230 Login successful.
ftp> pwd                # 원격 현재 디렉터리 → chroot 때문에 "/" 로 보임
ftp> ls                 # 목록
ftp> lcd /tmp           # 로컬 디렉터리 변경
ftp> bin                # 바이너리 전송 모드 (= binary / type I)
ftp> asc                # ASCII 전송 모드 (= ascii)
ftp> passive            # 능동 ↔ 수동 모드 토글
ftp> put ftptest.txt    # 업로드
ftp> get ftptest.txt down.txt   # 다운로드(이름 변경)
ftp> mput *.txt         # 여러 개 업로드
ftp> mget *.txt         # 여러 개 다운로드
ftp> delete ftptest.txt # 원격 파일 삭제
ftp> mkdir subdir
ftp> cd subdir
ftp> cdup               # 상위 디렉터리
ftp> hash               # 진행 표시(#) 토글
ftp> status             # 현재 모드 확인
ftp> help               # 명령 목록
ftp> bye                # 종료 (quit·exit 동일)
```

| 명령 | 의미 |
| --- | --- |
| `user` | 다른 계정으로 재인증 |
| `ls` / `dir` | 원격 목록 |
| `pwd` / `lcd` | 원격 현재 위치 / **로컬** 디렉터리 변경 |
| `get` / `mget` | 다운로드 / 다중 다운로드 |
| `put` / `mput` | 업로드 / 다중 업로드 |
| `bin` / `asc` | **바이너리** / **ASCII** 전송 모드 — 실행 파일·이미지는 반드시 bin |
| `passive` | 수동 모드 토글 |
| `delete` / `mdelete` | 원격 삭제 |
| `mkdir` / `rmdir` | 원격 디렉터리 생성·삭제 |
| `bye` / `quit` | 종료 |

- `ftp` 명령이 없으면 아래 `lftp`·`curl` 로 동일 검증 가능

**검증**

```bash
# ftp 없이도 되는 비대화식 검증
curl -s -u dev1 ftp://127.0.0.1/                       # 목록 (비밀번호 대화식)
curl -s -T /tmp/ftptest.txt -u dev1 ftp://127.0.0.1/   # 업로드
ls -l ~dev1/ftptest.txt
```

```text
-rw-r--r--. 1 dev1 devteam 16 ... /home/dev1/ftptest.txt
```

- `curl -T <파일> ftp://…` : 업로드 (**T**ransfer/upload)
- `curl -u <계정>` : 인증 정보 (**u**ser)
- `curl -O ftp://…/파일` : 원격 이름 그대로 다운로드

> 📝 **시험 포인트**: 전송 모드 `bin`(바이너리) / `asc`(ASCII) 구분은 고전 단골. 텍스트가 아닌 파일을 ASCII 로 받으면 깨진다. `lcd` 는 **로컬** 디렉터리 변경(원격은 `cd`).

### 6-5. lftp · curl 로 비대화식 검증

> **상황**: 스크립트에서 쓸 수 있는 방식으로 다시 확인한다.

```bash
lftp -u dev1 192.168.64.10 -e 'ls; bye'
lftp -u dev1 192.168.64.10 <<'EOF'
ls
put /tmp/ftptest.txt -o lftp-upload.txt
ls
bye
EOF
curl -s -u dev1 ftp://192.168.64.10/ | head
```

- `lftp -u <계정>[,<비번>] <서버>` : 접속 (**u**ser)
- `-e '<명령들>'` : 접속 직후 실행할 명령 (**e**xecute), `;` 로 구분
- `-o <이름>` : 업로드 시 원격 파일명 지정 (**o**utput)
- `mirror` / `mirror -R` : 디렉터리 동기화(다운로드/업로드)

**검증**

```bash
ls -l ~dev1/lftp-upload.txt
lftp -u dev1 192.168.64.10 -e 'ls; bye' 2>/dev/null
```

```text
-rw-r--r--. 1 dev1 devteam 16 ... /home/dev1/lftp-upload.txt
-rw-r--r--    1 2001     2000           16 Sep 03 10:40 ftptest.txt
-rw-r--r--    1 2001     2000           16 Sep 03 10:41 lftp-upload.txt
```

- 목록에 소유자가 **숫자 UID/GID** 로 보이는 것은 chroot 안에 `/etc/passwd` 가 없기 때문 — 정상

> 📝 **시험 포인트**: `lftp` 는 큐·미러링·재개를 지원하는 확장 클라이언트. 시험에서는 `ftp` 기본 명령이 주로 나오지만, 스크립트 자동화 문항에서 `lftp -e` 가 등장.

### 6-6. ftpusers vs user_list — 접근 제어 2종

> **상황**: `dev2` 의 FTP 로그인을 막아야 한다. 비슷해 보이는 두 파일의 차이를 실제로 확인한다.

```bash
cat /etc/vsftpd/ftpusers
cat /etc/vsftpd/user_list
grep -E 'userlist_enable|userlist_deny|userlist_file' /etc/vsftpd/vsftpd.conf
```

| 파일 | 동작 | 제어 스위치 |
| --- | --- | --- |
| `/etc/vsftpd/ftpusers` | 여기 적힌 계정은 **무조건 거부** (PAM `pam_listfile` 이 검사) | 없음 — 항상 적용 |
| `/etc/vsftpd/user_list` | `userlist_deny=YES`(기본) → **거부 목록**<br>`userlist_deny=NO` → **이 목록만 허용** | `userlist_enable=YES` 필요 |

- 기본 상태에서 두 파일 모두 `root`·`bin`·`daemon` 등 시스템 계정이 들어 있음 → root 로 FTP 로그인 불가

```bash
# ① dev2 를 ftpusers 에 추가 → 무조건 거부
echo 'dev2' >> /etc/vsftpd/ftpusers
systemctl restart vsftpd
curl -s -u dev2:'<비밀번호>' ftp://127.0.0.1/ ; echo "exit=$?"
```

```text
exit=67
```

- 종료 코드 67 = `CURLE_LOGIN_DENIED` (로그인 거부)

```bash
# ② ftpusers 에서 빼고 user_list 를 허용 전용 목록으로 전환
sed -i '/^dev2$/d' /etc/vsftpd/ftpusers
cat > /etc/vsftpd/user_list <<'EOF'
dev1
ops1
EOF
sed -i 's/^userlist_enable=YES/userlist_enable=YES\nuserlist_deny=NO/' /etc/vsftpd/vsftpd.conf
systemctl restart vsftpd
curl -s -u dev1 ftp://127.0.0.1/ >/dev/null; echo "dev1 exit=$?"
curl -s -u dev2:'<비밀번호>' ftp://127.0.0.1/ >/dev/null; echo "dev2 exit=$?"
```

```text
dev1 exit=0
dev2 exit=67
```

- `userlist_deny=NO` 로 바꾸는 순간 `user_list` 는 **화이트리스트**가 되어 목록에 없는 계정이 전부 차단됨

```bash
# 원복 — 기본(거부 목록) 동작으로
sed -i '/^userlist_deny=NO$/d' /etc/vsftpd/vsftpd.conf
systemctl restart vsftpd
```

**검증**

```bash
grep -E 'userlist' /etc/vsftpd/vsftpd.conf
grep -c '' /etc/vsftpd/ftpusers
tail -5 /var/log/secure | grep -i vsftpd
```

> 📝 **시험 포인트**: 필기 FULL r06-81 — "특정 계정의 FTP 로그인을 금지할 목록 파일" 은 `/etc/vsftpd/ftpusers`. `user_list` 는 `userlist_deny` 값에 따라 **거부/허용이 뒤집히는** 것이 핵심 함정. `chroot_list` 는 chroot 예외 목록으로 또 다른 파일.

### 6-7. chroot 동작 확인

> **상황**: 로그인한 사용자가 정말 홈 밖으로 못 나가는지 확인한다.

```bash
curl -s -u dev1 ftp://127.0.0.1/ | head -3        # 홈 = 루트로 보임
lftp -u dev1 127.0.0.1 -e 'pwd; cd ..; pwd; ls; bye'
```

```text
lftp dev1@127.0.0.1:/> pwd
ftp://dev1@127.0.0.1/
lftp dev1@127.0.0.1:/> cd ..
lftp dev1@127.0.0.1:/> pwd
ftp://dev1@127.0.0.1/
```

- `cd ..` 를 해도 여전히 `/` — chroot 최상위 밖으로 못 나감. 실제 경로는 `/home/dev1`

```bash
# 예외 목록으로 특정 사용자만 chroot 면제 (참고)
echo 'ops1' > /etc/vsftpd/chroot_list
cat >> /etc/vsftpd/vsftpd.conf <<'EOF'
chroot_list_enable=YES
chroot_list_file=/etc/vsftpd/chroot_list
EOF
systemctl restart vsftpd
```

- `chroot_local_user=YES` + `chroot_list_enable=YES` → **목록에 적힌 계정만 chroot 면제**
- `chroot_local_user=NO` + `chroot_list_enable=YES` → **목록에 적힌 계정만 chroot 적용** (의미가 뒤집힘 ⚠️)

**검증**

```bash
lftp -u dev1 127.0.0.1 -e 'cd ..; pwd; bye' 2>/dev/null
lftp -u ops1 127.0.0.1 -e 'cd ..; pwd; bye' 2>/dev/null    # 면제 계정은 상위로 이동 가능
```

```text
ftp://dev1@127.0.0.1/            ← 갇혀 있음
ftp://ops1@127.0.0.1/home        ← 상위로 나감
```

> 📝 **시험 포인트**: `chroot_local_user` 와 `chroot_list_enable` 조합에 따라 목록의 의미가 **면제/적용으로 뒤집힌다**. 표로 외워 둘 것.

### 6-8. 익명 FTP 열기 → 검증 → 다시 차단

> **상황**: 공개 배포용 익명 다운로드를 잠시 열어 동작을 확인하고, 정책대로 다시 닫는다.

⚠️ 익명 FTP 는 인증 없이 누구나 접근한다. 실습 후 반드시 `anonymous_enable=NO` 로 원복.

```bash
echo 'public download file' > /var/ftp/pub/public.txt
chmod 644 /var/ftp/pub/public.txt
ls -ldZ /var/ftp /var/ftp/pub

sed -i 's/^anonymous_enable=NO/anonymous_enable=YES/' /etc/vsftpd/vsftpd.conf
systemctl restart vsftpd
```

```bash
# 익명 접속 — 계정 anonymous(또는 ftp), 비밀번호는 아무 이메일
curl -s ftp://127.0.0.1/pub/ 
curl -s ftp://127.0.0.1/pub/public.txt
lftp -e 'ls pub; bye' 127.0.0.1
```

```text
-rw-r--r--    1 0        0              21 Sep 03 10:50 public.txt
public download file
```

- `/var/ftp` 는 익명 사용자의 chroot 최상위 → `root:root 0755`, **쓰기 금지**여야 함
- 익명 업로드까지 허용하려면 `anon_upload_enable=YES` + 쓰기 가능한 하위 디렉터리(`/var/ftp/upload`) 별도 지정 — 보안상 비권장

**검증 후 원복**

```bash
sed -i 's/^anonymous_enable=YES/anonymous_enable=NO/' /etc/vsftpd/vsftpd.conf
systemctl restart vsftpd
curl -s ftp://127.0.0.1/pub/ ; echo "anon exit=$?"
grep '^anonymous_enable' /etc/vsftpd/vsftpd.conf
```

```text
anon exit=67
anonymous_enable=NO
```

> 📝 **시험 포인트**: 필기 FULL r06-80, r08-82 — 익명 **다운로드는 유지하고 업로드만 금지** = `anon_upload_enable=NO`(`anonymous_enable` 은 YES 유지). 둘을 혼동한 선지가 오답.

### 6-9. 전송 로그 xferlog 해석

> **상황**: 누가 무엇을 언제 올렸는지 추적한다. xferlog 필드 순서는 필기·실기 모두에 나온다.

```bash
ls -l /var/log/xferlog /var/log/vsftpd.log 2>/dev/null
curl -s -T /tmp/ftptest.txt -u dev1 ftp://127.0.0.1/logtest.txt
tail -3 /var/log/xferlog
```

```text
Thu Sep  3 10:55:12 2026 1 127.0.0.1 16 /home/dev1/logtest.txt b _ i r dev1 ftp 0 * c
```

| 위치 | 필드 | 예 | 의미 |
| --- | --- | --- | --- |
| 1 | current-time | `Thu Sep 3 10:55:12 2026` | 전송 완료 시각 |
| 2 | transfer-time | `1` | 전송 소요 시간(초) |
| 3 | remote-host | `127.0.0.1` | 클라이언트 주소 |
| 4 | file-size | `16` | 전송 바이트 |
| 5 | filename | `/home/dev1/logtest.txt` | 파일 경로 |
| 6 | transfer-type | `b` | `b`inary / `a`scii |
| 7 | special-action | `_` | 압축 등 (`_` 없음, `C` 압축, `T` tar) |
| 8 | direction | `i` | **i**ncoming(업로드) / **o**utgoing(다운로드) / `d`elete |
| 9 | access-mode | `r` | **r**eal(로컬 계정) / **a**nonymous / **g**uest |
| 10 | username | `dev1` | 사용자 |
| 11 | service-name | `ftp` | 서비스 |
| 12 | auth-method | `0` | 0 = 없음, 1 = RFC931 |
| 13 | auth-user-id | `*` | 인증 사용자 ID |
| 14 | completion-status | `c` | **c**omplete / **i**ncomplete |

**검증**

```bash
awk '{print $8, $9, $10, $NF, $5}' /var/log/xferlog | tail -5
grep -c ' i ' /var/log/xferlog        # 업로드 건수
grep -c ' o ' /var/log/xferlog        # 다운로드 건수
```

```text
i r dev1 c /home/dev1/logtest.txt
```

> 📝 **시험 포인트**: 8번째 필드 `i`/`o` 가 업로드/다운로드, 9번째 `r`/`a` 가 실계정/익명, 마지막 `c`/`i` 가 완료/미완료. `xferlog_std_format=YES` 여야 이 형식으로 기록된다.

### 6-10. 능동 모드 vs 수동 모드

> **상황**: 방화벽 문제의 근원인 두 모드 차이를 정리한다.

**능동(Active) 모드**

```text
클라이언트                                서버
  (임의 포트 N) ──── 제어 연결 ───────▶ 21
  (포트 N+1 대기)  ◀── 데이터 연결 ───── 20   ★ 서버가 클라이언트로 접속
```

- 클라이언트가 `PORT` 명령으로 자기 데이터 포트를 알려 주면, **서버가 20번 포트에서 클라이언트로 연결을 건다**
- 클라이언트 앞에 방화벽·NAT 가 있으면 인바운드가 막혀 **실패** → 현대 환경에서 잘 안 쓰임
- 관련 설정: `connect_from_port_20=YES`

**수동(Passive) 모드**

```text
클라이언트                                서버
  (임의 포트 N) ──── 제어 연결 ───────▶ 21
                    ◀── PASV 응답(포트 P) ──
  (임의 포트 M) ──── 데이터 연결 ──────▶ P   ★ 클라이언트가 서버로 접속
```

- 서버가 `PASV` 응답으로 자기 데이터 포트 P 를 알려 주면 **클라이언트가 접속**
- 두 연결 모두 클라이언트 → 서버 방향이라 **클라이언트 방화벽 문제 없음**
- 대신 **서버 방화벽에서 P 범위를 열어야 함** → 6-2 의 `pasv_min_port`~`pasv_max_port` + 6-3 의 포트 개방

| 항목 | 능동(Active) | 수동(Passive) |
| --- | --- | --- |
| 제어 연결 | 클라이언트 → 서버 21 | 동일 |
| **데이터 연결 개시자** | **서버**(20 → 클라이언트) | **클라이언트**(→ 서버 임의 포트) |
| 서버 데이터 포트 | 20 고정 | `pasv_min_port`~`pasv_max_port` |
| 클라이언트 방화벽 | 인바운드 허용 필요 → 자주 실패 | 문제 없음 |
| 서버 방화벽 | 20 개방 | **포트 범위 개방 필요** |
| 명령 | `PORT` | `PASV` |

**검증**

```bash
lftp -u dev1 127.0.0.1 -e 'set ftp:passive-mode true;  ls; bye' 2>&1 | tail -3
lftp -u dev1 127.0.0.1 -e 'set ftp:passive-mode false; ls; bye' 2>&1 | tail -3
ss -tnp | grep -E ':21|:400[0-9][0-9]' | head
```

> 📝 **시험 포인트**: 필기 FULL r03-80, r04-83, r05-85, r07-82, r10-83 — "능동 모드에서 데이터 연결을 개시하는 쪽" 은 **서버**(포트 20). "수동 모드에서 서버가 50000~50100 범위를 열고 클라이언트가 접속" 이 정답 유형.

### 6-11. FTPS · SFTP 와의 구분

> **상황**: 보안 감사 대비로 세 가지를 구분해 둔다.

| 항목 | FTP | FTPS | SFTP |
| --- | --- | --- | --- |
| 기반 | 자체 프로토콜 | FTP + **TLS** | **SSH** 서브시스템 |
| 포트 | 21(제어) / 20·수동 | 21(명시적 STARTTLS) 또는 990(암묵적) | **22** |
| 암호화 | **없음(평문)** | TLS | SSH |
| 데몬 | `vsftpd` | `vsftpd`(`ssl_enable=YES`) | `sshd`(`Subsystem sftp`) |
| 연결 수 | 제어 + 데이터 2개 | 2개 | **1개** |
| 방화벽 | 복잡(포트 범위) | 복잡 | 간단(22 만) |

```bash
# SFTP 는 sshd 가 이미 제공 (Part 08) — 별도 설치 불필요
grep -i 'subsystem' /etc/ssh/sshd_config
sftp -P 2222 dev1@127.0.0.1        # Part 08 에서 포트를 2222 로 바꿨다면
```

```bash
# vsftpd 에 TLS 를 얹는 설정 (참고 — 2-13 의 인증서 재사용)
# rsa_cert_file=/etc/pki/tls/certs/lab.crt
# rsa_private_key_file=/etc/pki/tls/private/lab.key
# ssl_enable=YES
# force_local_data_ssl=YES
# force_local_logins_ssl=YES
```

**검증**

```bash
grep -i '^Subsystem' /etc/ssh/sshd_config
ss -tlnp | grep -E ':21|:2222|:22'
```

```text
Subsystem	sftp	/usr/libexec/openssh/sftp-server
```

> 📝 **시험 포인트**: 필기 FULL r09-81 — **SFTP 는 SSH(22) 서브시스템, FTPS 는 FTP + TLS**. 둘을 바꾼 선지가 오답. r06-99 — telnet·FTP 평문 노출 대책은 SSH·SFTP 대체.

---

## 7. 메일 — Postfix · s-nail · Dovecot

### 7-1. 메일 흐름과 구성 요소

> **상황**: 팀 내 알림용 로컬 메일을 구성하기 전에, 메일이 어떤 경로로 흐르는지 고정한다. 필기에서 MTA/MDA/MUA 구분과 흐름 순서가 반복 출제된다.

```text
[발신자 MUA] ──SMTP(25/587)──▶ [발신 MTA] ──SMTP(25)──▶ [수신 MTA] ──▶ [MDA] ──▶ 메일함
                                    │                                              │
                             MX 레코드 조회(DNS)                    POP3(110)/IMAP(143)
                                                                                   ▼
                                                                            [수신자 MUA]
```

| 구성 요소 | 원어 | 역할 | 예 |
| --- | --- | --- | --- |
| **MUA** | Mail **U**ser **A**gent | 사용자가 메일을 읽고 씀 | `mail`(s-nail), `mutt`, Thunderbird, Outlook |
| **MTA** | Mail **T**ransfer **A**gent | 서버 간 **전송·중계** | **Postfix**, Sendmail, Exim, qmail |
| **MDA** | Mail **D**elivery **A**gent | 수신 메일을 사용자 메일함에 **최종 배달** | procmail, maildrop, Dovecot LDA/LMTP |
| **MRA/MSA** | Retrieval / Submission | 메일함 조회 / 발신 접수 | Dovecot(POP3·IMAP), Postfix submission(587) |

| 프로토콜 | 포트 | 역할 |
| --- | --- | --- |
| SMTP | **25** | MTA 간 메일 **전송** |
| SMTP Submission | **587** | MUA → MTA 발신 접수 (STARTTLS + SMTP AUTH) |
| SMTPS | **465** | 연결 즉시 TLS(암묵적) |
| POP3 | **110** | 메일 **수신** — 내려받고 서버에서 삭제(기본) |
| POP3S | **995** | POP3 over TLS |
| IMAP | **143** | 메일 **수신** — 서버에 보관·폴더 동기화 |
| IMAPS | **993** | IMAP over TLS |

| 항목 | POP3 | IMAP |
| --- | --- | --- |
| 저장 위치 | **로컬(클라이언트)** | **서버** |
| 다중 기기 | 동기화 어려움 | **동기화 가능** |
| 폴더 관리 | 클라이언트에서만 | **서버 폴더 지원** |
| 오프라인 | 강함 | 캐시 필요 |
| 서버 용량 | 적게 씀 | 많이 씀 |
| 포트 | 110 / 995 | 143 / 993 |

> 📝 **시험 포인트**: 필기 FULL r02-75, r03-89, r04-75, r07-75, r10-78 — "MTA 가 받은 메일을 사용자 메일함에 최종 배달하는 프로그램(procmail)" = **MDA**. 흐름 순서는 MUA →(SMTP) MTA →(MX 조회 → SMTP) MTA → MDA → 메일함 →(POP3/IMAP) MUA.

### 7-2. Postfix 설치와 main.cf 키

> **상황**: Rocky 9 는 Postfix 가 기본 MTA 다. 로컬 도메인 `lab.local` 메일을 이 서버가 최종 수신하도록 설정한다.

```bash
dnf install -y postfix s-nail
rpm -q postfix s-nail
cp /etc/postfix/main.cf /etc/postfix/main.cf.orig
ls -l /etc/postfix/ | head -15
```

- `postfix` : MTA 본체
- `s-nail` : `mail`/`mailx` 명령을 제공하는 MUA — **RHEL 9 에서 `mailx` 패키지가 `s-nail` 로 대체됨**

| `main.cf` 키 | 의미 |
| --- | --- |
| `myhostname` | 이 서버의 **FQDN** (`srv01.lab.local`) |
| `mydomain` | 도메인 (`lab.local`). 미지정 시 myhostname 에서 첫 레이블 제거 |
| `myorigin` | 발신 주소에 붙일 도메인 — `$myhostname` 또는 `$mydomain` |
| `inet_interfaces` | 수신할 인터페이스 — `localhost`(기본) · `all` · 특정 IP |
| `inet_protocols` | `ipv4` · `ipv6` · `all` |
| `mydestination` | **이 서버가 최종 수신할 도메인 목록** — 여기 없는 도메인은 중계 대상 |
| `mynetworks` | **중계(릴레이)를 허용할 클라이언트 대역** — 오픈 릴레이 방지의 핵심 |
| `relayhost` | 모든 외부 메일을 넘길 상위 SMTP 서버(`[smtp.isp.com]:587`) |
| `home_mailbox` | 사용자 홈에 저장할 때의 경로 (`Maildir/`) — 미지정 시 `/var/spool/mail/<사용자>` |
| `mail_spool_directory` | mbox 저장 디렉터리 (`/var/spool/mail`) |
| `mailbox_size_limit` | 메일함 최대 크기(바이트, 0 = 무제한) |
| `message_size_limit` | 메일 1통 최대 크기 |
| `smtpd_banner` | 접속 배너 — 버전 노출 최소화 대상 |
| `alias_maps` / `alias_database` | 별칭 조회 맵 / 생성 대상 |
| `smtpd_client_restrictions` 등 | 접속·발신·수신 제한 규칙 |

```bash
vi /etc/postfix/main.cf
```

```
myhostname = srv01.lab.local
mydomain = lab.local
myorigin = $mydomain
inet_interfaces = all
inet_protocols = ipv4
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
mynetworks = 127.0.0.0/8, 192.168.64.0/24
home_mailbox = 
message_size_limit = 10485760
smtpd_banner = $myhostname ESMTP
```

- `$변수` : `main.cf` 안에서 다른 파라미터 값을 참조하는 표기

**검증**

```bash
postconf -n | head -20            # 기본값과 다른 것만 (n = non-default)
postconf myhostname mydomain mydestination mynetworks inet_interfaces
postfix check                     # 무출력이면 정상
```

```text
# postconf -n
alias_database = hash:/etc/aliases
alias_maps = hash:/etc/aliases
inet_interfaces = all
inet_protocols = ipv4
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
mydomain = lab.local
myhostname = srv01.lab.local
mynetworks = 127.0.0.0/8, 192.168.64.0/24
myorigin = $mydomain
...
```

> 📝 **시험 포인트**: 필기 FULL r01-77, r07-77 — Postfix 주 설정 파일은 **`/etc/postfix/main.cf`**(`master.cf` 는 서비스 프로세스 정의). r05-74, r08-76 — `mydestination` = **이 서버가 최종 수신할 도메인**(중계 대상 목록 아님). r09-77 — 오픈 릴레이 차단은 `mynetworks` 제한 + `smtpd_recipient_restrictions`.

### 7-3. postconf 로 값 변경·조회

> **상황**: 설정 파일을 직접 열지 않고 명령으로 값을 바꾸는 방법을 익힌다. 스크립트·시험 답안 모두에서 쓰인다.

```bash
postconf -e 'inet_interfaces = all'         # 값 설정(edit) — main.cf 에 기록
postconf -e 'inet_protocols = ipv4'
postconf -d myhostname                      # 기본값(default) 확인
postconf -n                                 # 기본과 다른 값만
postconf -M                                 # master.cf 내용 (서비스 정의)
postconf -h myhostname                      # 값만 (헤더 없이)
postfix check
systemctl restart postfix
```

- `-e` : 파라미터 설정 (**e**dit) — 기존 줄이 있으면 교체, 없으면 추가
- `-d` : **d**efault 값 표시
- `-n` : **n**on-default 만
- `-M` : **M**aster.cf 표시
- `-h` : 이름 없이 값만 (**h**ide name)
- ⚠️ `inet_interfaces` 를 바꾼 뒤에는 **`reload` 가 아니라 `restart`** 필요

**검증**

```bash
postconf -h inet_interfaces inet_protocols
grep -nE '^inet_' /etc/postfix/main.cf
postfix check && echo 'postfix config OK'
```

```text
all
ipv4
postfix config OK
```

> 📝 **시험 포인트**: `postconf -n` 은 "기본값에서 바뀐 설정만" 보여 주므로 장애 진단의 첫 단계. `postconf -e` 는 파일을 직접 수정하는 명령이라는 점을 기억.

### 7-4. 기동·방화벽·MTA 확인

> **상황**: Postfix 를 올리고 25번 포트를 연다. 시스템의 기본 MTA 가 무엇인지도 확인한다.

```bash
systemctl enable --now postfix
systemctl is-active postfix
ss -tlnp | grep :25
firewall-cmd --permanent --add-service=smtp
firewall-cmd --reload
alternatives --display mta
ls -l /usr/sbin/sendmail /etc/alternatives/mta
```

- `alternatives --display mta` : 여러 MTA 중 현재 선택된 것을 표시 — Postfix 를 설치해도 `sendmail` 명령 이름은 **호환용으로 유지**됨
- `alternatives --config mta` : 대화식으로 기본 MTA 전환

**검증**

```bash
systemctl is-enabled postfix
ss -tlnp | grep master           # Postfix 의 부모 프로세스 이름은 master
alternatives --display mta | head -4
readlink -f /usr/sbin/sendmail
postconf mail_version
```

```text
# ss -tlnp | grep master
LISTEN 0 100 0.0.0.0:25 0.0.0.0:* users:(("master",pid=...,fd=13))
# alternatives --display mta
mta - status is auto.
 link currently points to /usr/sbin/sendmail.postfix
# readlink -f /usr/sbin/sendmail
/usr/sbin/sendmail.postfix
# postconf mail_version
mail_version = 3.5.x
```

> 📝 **시험 포인트**: Postfix 의 마스터 프로세스 이름은 `master` — `ps`·`ss` 출력에서 `postfix` 를 찾으면 안 나온다. `sendmail` 명령이 남아 있어도 실제 MTA 는 Postfix 일 수 있다는 점이 함정.

### 7-5. /etc/aliases 와 newaliases

> **상황**: `ops` 로 보낸 메일이 `ops1` 에게 가고, `devs` 로 보내면 `dev1`·`dev2` 양쪽에 가도록 별칭을 만든다.

```bash
cp /etc/aliases /etc/aliases.orig
ls -l /etc/aliases /etc/aliases.db

cat >> /etc/aliases <<'EOF'

# ===== LAB 09 =====
ops:            ops1
webmaster:      ops1
devs:           dev1,dev2
root:           ops1
EOF

newaliases                       # = postalias /etc/aliases
ls -l /etc/aliases.db
```

- `/etc/aliases` 형식: `<별칭>: <대상1>,<대상2>` — 대상은 로컬 계정·메일 주소·`|명령`·`/파일경로` 가능
- `newaliases` : 텍스트 `/etc/aliases` → 바이너리 `/etc/aliases.db` 로 **변환**. **이걸 안 하면 변경이 반영되지 않음**
- `postalias /etc/aliases` : Postfix 의 동일 기능 명령
- `postmap <파일>` : 일반 조회 테이블(`access`·`virtual` 등)을 `.db` 로 변환 — **aliases 는 `postalias`/`newaliases`** 를 쓴다

**검증**

```bash
ls -l --time-style=+%H:%M:%S /etc/aliases /etc/aliases.db     # db 타임스탬프가 더 최신
postalias -q ops /etc/aliases          # 별칭 조회
postalias -q devs /etc/aliases
grep -E '^(ops|devs|webmaster|root):' /etc/aliases
```

```text
# ls -l --time-style=+%H:%M:%S /etc/aliases /etc/aliases.db
-rw-r--r--. 1 root root  1600 11:02:10 /etc/aliases
-rw-r--r--. 1 root root 12288 11:02:15 /etc/aliases.db
# postalias -q ops /etc/aliases
ops1
# postalias -q devs /etc/aliases
dev1,dev2
```

> 📝 **시험 포인트**: 필기 FULL r02-76, r03-90, r04-73, r05-75, r06-73 — `/etc/aliases` 수정 후 반드시 **`newaliases`**. 오답 선지 `postmap /etc/aliases`(다른 맵용) · `mailq`(큐 조회) · `postfix reload`(반영 아님) 구분.

### 7-6. 발송·수신 검증 — mail 명령

> **상황**: 별칭이 실제로 동작하는지 메일을 보내 확인한다.

```bash
echo "LAB09 alias body" | mail -s "alias test" devs
echo "to ops1 direct" | mail -s "LAB test" ops1
echo "with attachment" | mail -s "attach" -a /etc/hostname ops1
mailq                                      # 큐가 비어 있으면 즉시 배달됨
ls -l /var/spool/mail/
```

- `mail -s "<제목>" <수신자>` : 제목 지정 (**s**ubject). 본문은 표준입력
- `-a <파일>` : 첨부 (**a**ttach)
- `-c` / `-b` : 참조 / 숨은 참조
- `-r <주소>` : 발신 주소 지정
- 본문을 대화식으로 입력할 때는 `.` 만 있는 줄 또는 `Ctrl+D` 로 종료

**수신 확인**

```bash
su - ops1 -c mail
```

```text
s-nail version v14.x.  Type `?' for help
"/var/spool/mail/ops1": 3 messages 3 new
>N  1 root    Thu Sep  3 11:05  18/650   "alias test"
 N  2 root    Thu Sep  3 11:05  18/640   "LAB test"
 N  3 root    Thu Sep  3 11:06  25/900   "attach"
& h            # 헤더(목록) 다시 보기
& 1            # 1번 메일 읽기
& d 1          # 1번 삭제 표시
& q            # 종료 — 삭제 표시 반영, 읽은 메일은 ~/mbox 로 이동
& x            # 종료 — 아무 변경도 저장하지 않음 (삭제 취소)
```

| 대화식 명령 | 의미 |
| --- | --- |
| `h` | 헤더 목록 표시 |
| `<번호>` / Enter | 해당 메일 읽기 |
| `d <번호>` | 삭제 표시 (`d 1-3` 범위 가능) |
| `u <번호>` | 삭제 표시 취소 (**u**ndelete) |
| `r` / `R` | 전체 회신 / 발신자에게만 회신 |
| `s <파일>` | 메일을 파일로 저장 |
| `q` | **변경 반영 후 종료** — 읽은 메일은 `~/mbox` 로 이동, 삭제 표시된 것은 제거 |
| `x` | **변경 없이 종료** — 메일함 원상 유지 |
| `?` | 도움말 |

**검증**

```bash
ls -l /var/spool/mail/dev1 /var/spool/mail/dev2 /var/spool/mail/ops1
grep -c '^From ' /var/spool/mail/dev1 /var/spool/mail/dev2
grep -E '^(To|Subject):' /var/spool/mail/dev2 | tail -4
su - dev1 -c 'mail -H' 2>/dev/null
```

```text
# ls -l /var/spool/mail/
-rw-rw----. 1 dev1 mail  650 ... dev1
-rw-rw----. 1 dev2 mail  650 ... dev2
-rw-rw----. 1 ops1 mail 2190 ... ops1
# grep -c '^From ' /var/spool/mail/dev1 /var/spool/mail/dev2
/var/spool/mail/dev1:1
/var/spool/mail/dev2:1
# grep -E '^(To|Subject):' /var/spool/mail/dev2
To: devs@lab.local
Subject: alias test
```

- `devs` 하나로 보냈는데 **dev1·dev2 양쪽 메일함에 각각 저장**됨 → 별칭 확장 성공
- `mail -H` : 헤더 목록만 출력하고 즉시 종료(비대화식 확인용)

> 📝 **시험 포인트**: `q` 와 `x` 의 차이 — `q` 는 삭제·이동을 **반영**, `x` 는 **취소**. 메일함 기본 위치는 mbox 형식의 `/var/spool/mail/<계정>`(= `/var/mail/<계정>`).

### 7-7. ~/.forward 로 개인 전달 설정

> **상황**: `dev2` 가 휴가 중이라 자기 메일을 `ops1` 에게 넘기려 한다. 관리자가 `/etc/aliases` 를 고치지 않고 **사용자 스스로** 처리하는 방법이다.

```bash
su - dev2 -c "printf 'ops1\n' > ~/.forward"
su - dev2 -c 'ls -l ~/.forward; cat ~/.forward'
chmod 644 /home/dev2/.forward

echo "forward test body" | mail -s "forward test" dev2
sleep 2
```

- `~/.forward` : 한 줄에 하나씩 전달 대상. 계정명·메일 주소·`\사용자`(자기 메일함에도 사본 보관)·`|명령` 가능
- `\dev2` 를 함께 적으면 **원본도 보관 + 전달** 동시 수행
- 파일은 **사용자 소유 + 다른 사용자 쓰기 불가** 여야 Postfix 가 인정

**검증**

```bash
grep -E '^(To|Subject):' /var/spool/mail/ops1 | tail -4
grep -c 'forward test' /var/spool/mail/ops1
grep -c 'forward test' /var/spool/mail/dev2 || echo 'dev2 mailbox: not stored'
grep 'forward' /var/log/maillog | tail -3
```

```text
To: dev2@lab.local
Subject: forward test
1
dev2 mailbox: not stored
... postfix/local[...]: ...: to=<ops1@lab.local>, orig_to=<dev2@lab.local>, relay=local, ..., status=sent (delivered to mailbox)
```

- 로그의 `orig_to=<dev2@lab.local>` → `to=<ops1@lab.local>` 이 전달 동작의 증거

```bash
# 원복
su - dev2 -c 'rm -f ~/.forward'
```

> 📝 **시험 포인트**: 필기 FULL r06-74 — "사용자가 **직접** 자기 메일을 다른 주소로 전달" = 홈 디렉터리의 **`~/.forward`**. `/etc/aliases` 수정은 **관리자** 권한 작업이라 오답.

### 7-8. 메일 큐 관리

> **상황**: 배달되지 못한 메일이 쌓였을 때 확인·재시도·삭제하는 방법을 익힌다.

```bash
# 존재하지 않는 외부 도메인으로 보내 큐에 남기기
echo 'queued mail' | mail -s "queue test" user@nosuch-domain-lab.invalid
sleep 3
mailq                                # = postqueue -p
postqueue -p
postqueue -f                         # 큐 즉시 재처리(flush)
postsuper -h ALL                     # 전체 보류(hold)
postsuper -H ALL                     # 보류 해제
mailq | tail -5
```

| 명령 | 의미 |
| --- | --- |
| `mailq` / `postqueue -p` | 큐 **p**rint — 대기 중인 메일 목록 |
| `postqueue -f` | **f**lush — 큐 전체 즉시 재전송 시도 |
| `postqueue -i <큐ID>` | 특정 메일만 재전송 |
| `postsuper -d <큐ID>` | 특정 메일 **d**elete |
| `postsuper -h/-H ALL` | 전체 보류 / 해제 |
| `postsuper -r ALL` | 전체 재큐잉 (**r**equeue) |
| `postcat -q <큐ID>` | 큐에 있는 메일 내용 확인 |

⚠️ 아래는 **큐의 모든 메일을 되돌릴 수 없이 삭제**한다. 운영 서버에서는 반드시 `mailq` 로 대상을 확인한 뒤 실행.

```bash
mailq | head -20                     # 대상 재확인 선행
postsuper -d ALL
```

**검증**

```bash
mailq
postqueue -p
ls /var/spool/postfix/deferred/ 2>/dev/null | head
grep -E 'status=(deferred|bounced|sent)' /var/log/maillog | tail -5
```

```text
# mailq  (삭제 전)
-Queue ID-  --Size-- ----Arrival Time---- -Sender/Recipient-------
A1B2C3D4E5*     456 Thu Sep  3 11:20:11  root@lab.local
                (Host or domain name not found. Name service error for name=nosuch-domain-lab.invalid ...)
                                         user@nosuch-domain-lab.invalid
-- 0 Kbytes in 1 Request.
# mailq  (postsuper -d ALL 후)
Mail queue is empty
```

> 📝 **시험 포인트**: 필기 FULL r08-75 — `mailq` 출력 해석: 큐 ID 옆의 `*` 는 활성 큐, 괄호 안 문장이 **지연 사유**. `postsuper -d ALL` 은 큐 전체 삭제라 위험 명령으로 출제.

### 7-9. /var/log/maillog 추적

> **상황**: 메일이 실제로 배달됐는지 로그로 확인하는 방법을 굳힌다.

```bash
tail -20 /var/log/maillog
grep 'status=sent' /var/log/maillog | tail -5
grep -E 'from=<|to=<' /var/log/maillog | tail -5
journalctl -u postfix -n 10 --no-pager
```

- `/var/log/maillog` : rsyslog 의 `mail.*` 규칙으로 기록됨 (Part 07 6-1 참조)
- 한 통의 메일은 **같은 큐 ID** 로 여러 줄에 걸쳐 기록됨 → 큐 ID 로 grep 하면 전체 여정 추적 가능

| 로그 필드 | 의미 |
| --- | --- |
| `from=<…>` / `to=<…>` | 발신·수신 주소 |
| `orig_to=<…>` | 별칭·forward 이전의 원래 수신자 |
| `relay=local` / `relay=<서버>[IP]:25` | 배달 경로 |
| `status=sent` | **배달 성공** |
| `status=deferred` | 일시 실패 → 재시도 대기 |
| `status=bounced` | 영구 실패 → 발신자에게 반송 |
| `dsn=2.0.0` | 전달 상태 코드(2=성공, 4=일시, 5=영구) |

**검증**

```bash
echo 'trace me' | mail -s "trace" ops1
sleep 2
QID=$(grep 'trace' /var/log/maillog | tail -1 | awk '{print $6}' | tr -d ':')
grep "$QID" /var/log/maillog
```

```text
postfix/pickup[...]: A1B2C3: uid=0 from=<root>
postfix/cleanup[...]: A1B2C3: message-id=<...@srv01.lab.local>
postfix/qmgr[...]: A1B2C3: from=<root@lab.local>, size=..., nrcpt=1 (queue active)
postfix/local[...]: A1B2C3: to=<ops1@lab.local>, relay=local, delay=0.02, dsn=2.0.0, status=sent (delivered to mailbox)
postfix/qmgr[...]: A1B2C3: removed
```

- `pickup → cleanup → qmgr → local → removed` 가 로컬 배달의 표준 흐름

> 📝 **시험 포인트**: 실기 r02-14, r05-12 — rsyslog 규칙 `mail.info /var/log/maillog` 해석. 필기 FULL r03-63 — `/var/log/maillog` = 메일 송수신 기록. `status=sent` 가 배달 성공의 확정 증거.

### 7-10. telnet 으로 SMTP 수작업 대화

> **상황**: 프로토콜 수준에서 메일을 직접 보내 본다. SMTP 명령 순서는 필기·실기 모두에 나온다.

```bash
dnf install -y telnet          # 클라이언트만 설치 (telnet-server 아님)
telnet 127.0.0.1 25
```

```text
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
220 srv01.lab.local ESMTP
EHLO srv01.lab.local                    ← 확장 SMTP 인사 (HELO 는 구형)
250-srv01.lab.local
250-PIPELINING
250-SIZE 10485760
250-VRFY
250-ETRN
250-ENHANCEDSTATUSCODES
250-8BITMIME
250-DSN
250 SMTPUTF8
MAIL FROM:<dev1@lab.local>              ← 발신자 (봉투 발신)
250 2.1.0 Ok
RCPT TO:<ops1@lab.local>                ← 수신자 (봉투 수신)
250 2.1.5 Ok
DATA                                    ← 본문 시작 선언
354 End data with <CR><LF>.<CR><LF>
Subject: manual smtp test
From: dev1@lab.local
To: ops1@lab.local

This mail was typed by hand over SMTP.
.                                       ← 점 하나만 있는 줄 = 본문 끝
250 2.0.0 Ok: queued as A1B2C3D4
QUIT
221 2.0.0 Bye
Connection closed by foreign host.
```

| SMTP 명령 | 역할 |
| --- | --- |
| `HELO <이름>` | 기본 인사 (구형 SMTP) |
| `EHLO <이름>` | 확장 인사 — 서버가 지원 기능 목록 응답 |
| `MAIL FROM:<주소>` | 봉투 발신자 지정 |
| `RCPT TO:<주소>` | 봉투 수신자 지정(여러 번 반복 가능) |
| `DATA` | 헤더+본문 입력 시작. **`.` 만 있는 줄로 종료** |
| `RSET` | 진행 중인 트랜잭션 초기화 |
| `VRFY <계정>` | 계정 존재 확인(정보 노출 위험 → 보통 비활성) |
| `QUIT` | 연결 종료 |

| 응답 코드 | 의미 |
| --- | --- |
| `220` | 서비스 준비 완료 |
| `250` | 요청 성공 |
| `354` | 본문 입력 대기 |
| `421` | 서비스 사용 불가(종료) |
| `450`/`451` | 일시 실패 |
| `550` | 사용자 없음·거부 (영구 실패) |

- `telnet` 이 없으면 `nc 127.0.0.1 25` 로 동일하게 진행 가능
- 탈출: `Ctrl+]` → `quit`

**검증**

```bash
grep -A2 'manual smtp test' /var/spool/mail/ops1 | head -5
grep 'manual smtp' /var/log/maillog | tail -2
su - ops1 -c 'mail -H' | tail -3
```

```text
Subject: manual smtp test
From: dev1@lab.local
To: ops1@lab.local

This mail was typed by hand over SMTP.
```

> 📝 **시험 포인트**: SMTP 대화 순서 **EHLO/HELO → MAIL FROM → RCPT TO → DATA → `.` → QUIT** 는 서술형 단골. `DATA` 종료가 "점(`.`) 하나뿐인 줄" 이라는 점이 정답 포인트.

### 7-11. 접근 제어 — access + postmap (참고)

> **상황**: 특정 IP·도메인의 메일을 거부하는 방법을 익힌다. 실제 차단은 걸어 두지 않고 형식만 확인한다.

```bash
cat > /etc/postfix/access <<'EOF'
192.168.64.99       REJECT   Blocked by LAB policy
spam-domain.invalid REJECT
192.168.64          OK
EOF
postmap /etc/postfix/access
ls -l /etc/postfix/access /etc/postfix/access.db
postmap -q 192.168.64.99 /etc/postfix/access
```

- `postmap <파일>` : 텍스트 조회 테이블을 `<파일>.db`(hash) 로 변환 — **aliases 를 제외한 모든 맵**에 사용
- `postmap -q <키> <파일>` : 조회 테스트 (**q**uery)
- 동작 값: `OK`(허용) · `REJECT`(거부) · `DISCARD`(조용히 버림) · `HOLD` · `FILTER`

```bash
# 적용하려면 (참고 — 이 실습에서는 적용하지 않음)
# postconf -e 'smtpd_client_restrictions = check_client_access hash:/etc/postfix/access'
# postfix reload
```

| 제한 파라미터 | 검사 시점 |
| --- | --- |
| `smtpd_client_restrictions` | 클라이언트 접속 시 |
| `smtpd_helo_restrictions` | HELO/EHLO 시 |
| `smtpd_sender_restrictions` | MAIL FROM 시 |
| `smtpd_recipient_restrictions` | RCPT TO 시 — **오픈 릴레이 차단의 핵심** |
| `smtpd_relay_restrictions` | 중계 허용 판단 (`permit_mynetworks, reject_unauth_destination`) |

**검증**

```bash
postmap -q 192.168.64.99 /etc/postfix/access
postmap -q 192.168.64    /etc/postfix/access
ls -l /etc/postfix/access.db
```

```text
REJECT   Blocked by LAB policy
OK
-rw-r--r--. 1 root root 12288 ... /etc/postfix/access.db
```

> 📝 **시험 포인트**: `postmap`(일반 맵) vs `newaliases`/`postalias`(aliases 전용) 구분. 오픈 릴레이 차단은 `mynetworks` 축소 + `smtpd_relay_restrictions` 의 `reject_unauth_destination`.

### 7-12. master.cf 구조 (참고)

> **상황**: `main.cf` 와 짝을 이루는 파일이 무엇인지 확인해 둔다.

```bash
grep -vE '^#|^$' /etc/postfix/master.cf | head -15
postconf -M | head -10
```

```text
smtp      inet  n       -       n       -       -       smtpd
pickup    unix  n       -       n       60      1       pickup
cleanup   unix  n       -       n       -       0       cleanup
qmgr      unix  n       -       n       300     1       qmgr
local     unix  -       n       n       -       -       local
smtp      unix  -       -       n       -       -       smtp
```

| 열 | 의미 |
| --- | --- |
| service | 서비스 이름(포트명 또는 내부 이름) |
| type | `inet`(TCP 소켓) · `unix`(유닉스 소켓) · `fifo` |
| private | 내부 전용 여부(`n` = 외부 접근 가능) |
| unpriv | 비특권 실행 여부 |
| chroot | chroot 실행 여부 |
| wakeup | 깨우기 주기(초) |
| maxproc | 최대 프로세스 수 |
| command | 실행할 데몬 프로그램 |

- **`main.cf` = 동작 파라미터**, **`master.cf` = 어떤 프로세스를 어떻게 띄울지** — 역할이 완전히 다름
- 587 서브미션 포트를 열려면 `master.cf` 의 `submission` 줄 주석을 해제

**검증**

```bash
postconf -M smtp/inet
grep -n '^#submission' /etc/postfix/master.cf | head -2
```

```text
smtp       inet  n  -  n  -  -  smtpd
```

> 📝 **시험 포인트**: 필기 FULL r01-77 의 오답 선지에 `/etc/postfix/master.cf` 가 등장 — 주 설정은 `main.cf`. `master.cf` 는 서비스(프로세스) 정의라는 구분을 알아 둘 것.

### 7-13. Sendmail 대응표 (※ 미실행)

> **상황**: 시험 범위에는 Sendmail 이 남아 있다. Rocky 9 에서는 Postfix 가 기본이므로 설치하지 않고 파일·명령 대응만 정리한다.

| 항목 | Sendmail | Postfix |
| --- | --- | --- |
| 주 설정 | `/etc/mail/sendmail.cf` (**직접 편집 안 함**) | `/etc/postfix/main.cf` |
| 설정 원본 | `/etc/mail/sendmail.mc` (m4 매크로) | — |
| 생성 명령 | `m4 sendmail.mc > sendmail.cf` (또는 `make -C /etc/mail`) | — |
| 서비스 정의 | — | `/etc/postfix/master.cf` |
| 릴레이 접근 제어 | `/etc/mail/access` → **`makemap hash access.db < access`** | `/etc/postfix/access` → **`postmap`** |
| 수신 도메인 | `/etc/mail/local-host-names` | `mydestination` |
| 가상 사용자 매핑 | `/etc/mail/virtusertable` (+ `makemap`) | `virtual_alias_maps` |
| 별칭 | `/etc/aliases` → **`newaliases`** | 동일 |
| 개인 전달 | `~/.forward` | 동일 |
| 큐 조회 | `mailq` | `mailq` / `postqueue -p` |
| 데몬 | `sendmail` | `master`(+ smtpd·qmgr·local) |

- `access` 파일 동작 값은 양쪽 공통: `RELAY`(중계 허용) · `REJECT`(거부) · `DISCARD`(폐기) · `OK`
- Sendmail 은 `.cf` 를 사람이 직접 고치지 않고 **`.mc` → m4 → `.cf`** 순서로 생성하는 것이 핵심

> 📝 **시험 포인트**: 필기 FULL r01-76 — "sendmail 에서 호스트·도메인별 릴레이 정책 파일" = **`/etc/mail/access`**(오답: `/etc/aliases`·`local-host-names`·`virtusertable`). `makemap hash`(sendmail) ↔ `postmap`(postfix) 짝도 함께 출제.

### 7-14. Dovecot 설치와 설정

> **상황**: 지금까지는 서버에 로그인해야만 메일을 읽을 수 있었다. POP3·IMAP 을 열어 메일 클라이언트로 받아 볼 수 있게 한다.

```bash
dnf install -y dovecot
cp /etc/dovecot/dovecot.conf /etc/dovecot/dovecot.conf.orig
ls /etc/dovecot/conf.d/ | head
```

```bash
# ① 프로토콜·수신 주소
sed -i 's/^#protocols = imap pop3 lmtp/protocols = imap pop3 lmtp/' /etc/dovecot/dovecot.conf
sed -i 's/^#listen = \*, ::/listen = *, ::/'                        /etc/dovecot/dovecot.conf

# ② 메일 저장 형식 — Postfix 가 쓰는 mbox 위치와 맞춤
sed -i 's|^#mail_location =.*|mail_location = mbox:~/mail:INBOX=/var/mail/%u|' /etc/dovecot/conf.d/10-mail.conf

# ③ 평문 인증 허용 (실습 한정 ⚠️ 운영에서는 TLS 필수)
sed -i 's/^#disable_plaintext_auth = yes/disable_plaintext_auth = no/'   /etc/dovecot/conf.d/10-auth.conf
sed -i 's/^auth_mechanisms = plain$/auth_mechanisms = plain login/'      /etc/dovecot/conf.d/10-auth.conf
```

| 파일 | 설정 |
| --- | --- |
| `/etc/dovecot/dovecot.conf` | `protocols = imap pop3 lmtp`, `listen = *, ::` |
| `conf.d/10-mail.conf` | `mail_location` — 메일 저장 형식·경로 |
| `conf.d/10-auth.conf` | `disable_plaintext_auth`, `auth_mechanisms` |
| `conf.d/10-master.conf` | 서비스별 리스너·포트 |
| `conf.d/10-ssl.conf` | TLS 인증서 |

- `mail_location` 형식
  - `mbox:~/mail:INBOX=/var/mail/%u` — **mbox**(한 파일에 여러 메일), INBOX 는 `/var/mail/<계정>`
  - `maildir:~/Maildir` — **Maildir**(메일 1통 = 파일 1개, 잠금 불필요·성능 우위)
- `%u` = 사용자명, `%d` = 도메인, `%h` = 홈 디렉터리
- ⚠️ Postfix 의 `home_mailbox` 를 `Maildir/` 로 바꿨다면 Dovecot 도 `maildir:~/Maildir` 로 맞춰야 함 — **불일치가 "메일이 안 보인다" 의 1순위 원인**

**검증**

```bash
doveconf -n | head -25
doveconf protocols mail_location disable_plaintext_auth auth_mechanisms
```

```text
# doveconf -n
# 2.3.x ...
auth_mechanisms = plain login
disable_plaintext_auth = no
listen = *, ::
mail_location = mbox:~/mail:INBOX=/var/mail/%u
protocols = imap pop3 lmtp
...
```

- `doveconf -n` : 기본값과 다른 설정만 출력 (Postfix 의 `postconf -n` 과 같은 역할)

> 📝 **시험 포인트**: 필기 FULL r01-78, r02-78, r05-76 — Dovecot 은 **POP3·IMAP 을 제공하는 수신 서버**(MTA 아님). 활성화 설정은 `protocols = imap pop3`.

### 7-15. Dovecot 기동·방화벽

> **상황**: 데몬을 올리고 110·143 을 연다.

```bash
systemctl enable --now dovecot
systemctl is-active dovecot
ss -tlnp | grep -E ':110|:143|:993|:995'
firewall-cmd --permanent --add-service=imap
firewall-cmd --permanent --add-service=pop3
firewall-cmd --reload
```

**검증**

```bash
systemctl is-enabled dovecot
ss -tlnp | grep dovecot
firewall-cmd --list-services | tr ' ' '\n' | grep -E 'imap|pop3'
journalctl -u dovecot -n 5 --no-pager
```

```text
# ss -tlnp | grep dovecot
LISTEN 0 100 0.0.0.0:110 0.0.0.0:* users:(("dovecot",pid=...,fd=...))
LISTEN 0 100 0.0.0.0:143 0.0.0.0:* users:(("dovecot",pid=...,fd=...))
LISTEN 0 100 0.0.0.0:993 0.0.0.0:* users:(("dovecot",pid=...,fd=...))
LISTEN 0 100 0.0.0.0:995 0.0.0.0:* users:(("dovecot",pid=...,fd=...))
```

- 993·995 는 기본 제공되는 TLS 리스너 — 자체 서명 인증서가 자동 생성되어 있음

> 📝 **시험 포인트**: 포트 4종을 한 화면에서 확인 — 110(POP3) · 143(IMAP) · 993(IMAPS) · 995(POP3S). 필기 FULL r03-75, r05-96 에서 "IMAPS — 995"(오답, 993 이 정답) 같은 뒤바꾼 선지가 나온다.

### 7-16. POP3 수작업 검증 — telnet 110

> **상황**: 프로토콜 명령으로 메일함을 직접 조회한다.

```bash
telnet 127.0.0.1 110
```

```text
+OK Dovecot ready.
USER ops1                     ← 계정
+OK
PASS <비밀번호>               ← 비밀번호 (평문 ⚠️)
+OK Logged in.
STAT                          ← 메일 수 / 총 바이트
+OK 4 2840
LIST                          ← 메일별 번호·크기
+OK 4 messages:
1 650
2 640
3 900
4 650
.
RETR 1                        ← 1번 메일 전체 내용 가져오기
+OK 650 octets
Return-Path: <root@lab.local>
Subject: alias test
...
.
TOP 1 5                       ← 헤더 + 본문 5줄만
DELE 4                        ← 4번 삭제 표시
+OK Marked to be deleted.
RSET                          ← 삭제 표시 전부 취소
+OK
NOOP                          ← 연결 유지
QUIT                          ← 종료 (이때 DELE 가 실제 반영)
+OK Logging out.
```

| POP3 명령 | 의미 |
| --- | --- |
| `USER` / `PASS` | 계정 / 비밀번호 |
| `STAT` | 메일 개수와 총 크기 |
| `LIST [번호]` | 메일 목록·크기 |
| `RETR <번호>` | 메일 본문 전체 가져오기 (**RETR**ieve) |
| `TOP <번호> <줄수>` | 헤더 + 본문 앞부분 |
| `DELE <번호>` | 삭제 **표시** |
| `RSET` | 삭제 표시 취소 |
| `NOOP` | 무동작(연결 유지) |
| `QUIT` | 종료 — **이 시점에 DELE 가 실제 반영** |

- 응답은 `+OK` / `-ERR` 두 가지뿐 (SMTP 의 숫자 코드와 대비)

**검증**

```bash
printf 'USER ops1\r\n' | nc -w2 127.0.0.1 110 | head -2
ss -tnp | grep :110 | head -2
grep -i pop3 /var/log/maillog | tail -3
```

```text
+OK Dovecot ready.
+OK
```

> 📝 **시험 포인트**: POP3 는 `USER`/`PASS` → `STAT`/`LIST` → `RETR` → `DELE` → `QUIT` 순서. 삭제가 **QUIT 시점**에 확정된다는 것이 함정 포인트. 응답은 `+OK`/`-ERR`.

### 7-17. IMAP 수작업 검증 — telnet 143

> **상황**: IMAP 은 태그(tag) 기반이라 명령 앞에 임의 문자열을 붙인다.

```bash
telnet 127.0.0.1 143
```

```text
* OK [CAPABILITY IMAP4rev1 ...] Dovecot ready.
a login ops1 <비밀번호>              ← 태그 a + 명령
a OK [CAPABILITY ...] Logged in
a list "" "*"                        ← 폴더 목록 (참조="" 패턴="*")
* LIST (\HasNoChildren) "/" "INBOX"
a OK List completed (0.001 + 0.000 secs).
a select inbox                       ← INBOX 선택
* FLAGS (\Answered \Flagged \Deleted \Seen \Draft)
* 4 EXISTS
* 0 RECENT
a OK [READ-WRITE] Select completed
a fetch 1 body[header]               ← 1번 메일 헤더
a search unseen                      ← 읽지 않은 메일 검색
a store 1 +flags \Seen               ← 읽음 표시
a logout
* BYE Logging out
a OK Logout completed.
```

| IMAP 명령 | 의미 |
| --- | --- |
| `<태그> LOGIN <계정> <비번>` | 인증 |
| `LIST "" "*"` | 폴더(메일박스) 목록 |
| `SELECT <폴더>` | 폴더 선택 — 이후 메시지 명령 대상 |
| `EXAMINE <폴더>` | 읽기 전용 선택 |
| `FETCH <번호> <항목>` | 메일 가져오기 (`body[]`·`body[header]`·`flags`) |
| `SEARCH <조건>` | 검색 (`UNSEEN`·`FROM`·`SINCE`) |
| `STORE <번호> +FLAGS \Seen` | 플래그 변경 |
| `CREATE` / `DELETE` / `RENAME` | 폴더 관리 — **서버 측 폴더**를 다루는 것이 POP3 와의 결정적 차이 |
| `EXPUNGE` | `\Deleted` 표시된 메일 실제 제거 |
| `LOGOUT` | 종료 |

- 응답 형태: `* <정보>` (서버 정보) / `<태그> OK|NO|BAD` (명령 결과)

**검증**

```bash
printf 'a login ops1 <비밀번호>\r\na list "" "*"\r\na logout\r\n' | nc -w3 127.0.0.1 143 | head -6
doveconf -n | grep protocols
```

```text
* OK [CAPABILITY IMAP4rev1 ...] Dovecot ready.
a OK [CAPABILITY ...] Logged in
* LIST (\HasNoChildren) "/" "INBOX"
a OK List completed
```

> 📝 **시험 포인트**: IMAP 은 **서버에 메일을 보관**하고 폴더를 서버 측에서 관리 → 여러 기기 동기화에 유리. POP3 는 내려받고 서버에서 삭제(기본). 이 차이가 필기 비교 문제의 정답 축.

---

## 8. CUPS — 인쇄 시스템

### 8-1. 설치와 기동

> **상황**: 실물 프린터는 없지만 인쇄 큐 관리 명령은 시험 필수 범위다. CUPS 를 올리고 더미 프린터를 만들어 큐 동작을 확인한다.

```bash
dnf install -y cups cups-client
systemctl enable --now cups
systemctl is-active cups
ss -tlnp | grep 631
ls /etc/cups/
```

- `cups` : **C**ommon **U**nix **P**rinting **S**ystem 서버(데몬 `cupsd`)
- `cups-client` : `lp`·`lpstat`·`lpadmin`·`cancel`·`lpr`·`lpq`·`lprm` 명령
- CUPS 는 **IPP**(Internet Printing Protocol) 기반, 관리 웹 인터페이스는 **631/TCP**

| 파일 | 역할 |
| --- | --- |
| `/etc/cups/cupsd.conf` | 데몬 설정(리스닝·접근 제어·로그) |
| `/etc/cups/printers.conf` | **등록된 프린터 정의** — `lpadmin` 이 자동 생성 |
| `/etc/cups/classes.conf` | 프린터 클래스(그룹) |
| `/etc/cups/ppd/*.ppd` | 프린터별 PPD 드라이버 |
| `/var/spool/cups/` | 인쇄 작업 스풀 |
| `/var/log/cups/{access_log,error_log,page_log}` | 로그 |
| `/etc/printcap` | **레거시 BSD lpd 호환 파일** — CUPS 가 자동 생성만 함 |

**검증**

```bash
systemctl is-enabled cups
ss -tlnp | grep cupsd
lpstat -r                     # 스케줄러 동작 여부
ls -l /etc/printcap
```

```text
# ss -tlnp | grep cupsd
LISTEN 0 128 127.0.0.1:631 0.0.0.0:* users:(("cupsd",pid=...,fd=...))
# lpstat -r
scheduler is running
```

- 기본은 **루프백만 리스닝** — 원격 관리는 8-8 에서 SSH 포워딩으로 해결

> 📝 **시험 포인트**: 필기 FULL r01-53, r04-49, r05-54, r06-48, r07-52, r10-53 — CUPS 는 **IPP 기반, 웹 관리 631 포트**. "설정 파일은 `/etc/printcap` 하나뿐" 은 오답(CUPS 는 `cupsd.conf`·`printers.conf`·PPD 를 쓰고 `printcap` 은 호환용).

### 8-2. 더미 프린터 등록 — lpadmin

> **상황**: 실물이 없으므로 존재하지 않는 소켓 주소를 가리키는 프린터 `labprn` 을 만든다. 인쇄 작업은 큐에 쌓인 채 대기하므로 큐 관리 명령을 모두 실습할 수 있다.

```bash
lpadmin -p labprn -E -v socket://127.0.0.1:9100 -m raw
lpadmin -d labprn                       # 기본 프린터로 지정
lpstat -p labprn
lpstat -d
cat /etc/cups/printers.conf
```

- `-p <이름>` : 프린터 이름 지정·생성 (**p**rinter)
- `-E` : **E**nable — 프린터 활성 + 작업 수락(`cupsenable` + `cupsaccept` 효과)
- `-v <장치 URI>` : 장치 주소 (**v** = device URI). `socket://IP:9100`(JetDirect) · `ipp://` · `usb://` · `lpd://` · `file:///`
- `-m <모델>` : PPD 모델 (**m**odel). `raw` = 드라이버 없이 그대로 전달
- `-d <이름>` : **d**efault 프린터 지정 (= `lpoptions -d`)
- `-x <이름>` : 프린터 삭제
- `-c <클래스>` : 클래스에 추가
- `-u allow:<사용자>` / `deny:` : 사용자별 접근 제어
- ⚠️ `-m raw` 가 거부되면 (`lpadmin: Unsupported ... raw`) 대신 `-m everywhere` 또는 `-m drv:///sample.drv/generic.ppd` 사용. 또는 장치를 `-v file:///dev/null` 로 지정

**검증**

```bash
lpstat -p -d
lpstat -v
grep -E '^<|DeviceURI|State|Accepting' /etc/cups/printers.conf
lpstat -a
```

```text
# lpstat -p -d
printer labprn is idle.  enabled since ...
system default destination: labprn
# lpstat -v
device for labprn: socket://127.0.0.1:9100
# lpstat -a
labprn accepting requests since ...
# grep ... /etc/cups/printers.conf
<DefaultPrinter labprn>
DeviceURI socket://127.0.0.1:9100
State Idle
Accepting Yes
```

> 📝 **시험 포인트**: `lpadmin` 은 **System V 계열 관리 명령** — 프린터 등록·삭제·기본 지정. `-E`(활성) 와 `-d`(기본 지정) 구분이 출제 포인트.

### 8-3. 상태 조회 — lpstat

> **상황**: 프린터·큐 상태를 보는 옵션을 한 번에 정리한다.

```bash
lpstat -p            # 프린터 상태 (idle / printing / disabled)
lpstat -d            # 기본 프린터
lpstat -a            # 작업 수락(accepting) 여부
lpstat -o            # 대기 중인 작업 목록
lpstat -t            # 전부 (total) — 스케줄러·장치·프린터·작업
lpstat -r            # 스케줄러 실행 여부
lpstat -v            # 프린터별 장치 URI
lpstat -s            # 요약 (기본 프린터 + 클래스 + 장치)
```

| 옵션 | 의미 |
| --- | --- |
| `-p` | **p**rinter 상태 |
| `-d` | **d**efault 프린터 |
| `-a` | **a**ccepting 여부 |
| `-o` | 대기 작업 (**o**utput/jobs) |
| `-t` | **t**otal — 모든 정보 |
| `-r` | 스케줄러 실행 여부 (**r**unning) |
| `-v` | 장치 URI (**v**erbose device) |
| `-u <사용자>` | 특정 사용자의 작업 |

**검증**

```bash
lpstat -t
```

```text
scheduler is running
system default destination: labprn
device for labprn: socket://127.0.0.1:9100
labprn accepting requests since ...
printer labprn is idle.  enabled since ...
```

> 📝 **시험 포인트**: 필기 FULL r01-53, r02-54 — "프린터와 인쇄 큐 상태 확인" = `lpstat`. `lpstat -p`(프린터 상태) 와 `lpq`(큐 목록) 의 계열 차이(System V vs BSD)를 함께 물어본다.

### 8-4. 인쇄 작업 제출 — lp · lpr

> **상황**: System V 계열(`lp`)과 BSD 계열(`lpr`)로 각각 작업을 넣어 본다. 실제 출력은 되지 않고 큐에 쌓인다.

```bash
lp -d labprn -n 2 /etc/hosts            # System V: -d 프린터, -n 매수
lp -d labprn -t "hosts-copy" /etc/hosts # -t 작업 제목
lpr -P labprn /etc/hostname             # BSD: -P 프린터
lpr -P labprn -# 3 /etc/hostname        # BSD: -# 매수
echo 'stdin print test' | lp -d labprn  # 표준입력도 가능
```

| 계열 | 프린터 지정 | 매수 지정 |
| --- | --- | --- |
| System V (`lp`) | **`-d <프린터>`** | **`-n <매수>`** |
| BSD (`lpr`) | **`-P <프린터>`** | **`-# <매수>`** |

- ⚠️ 옵션 문자가 계열마다 다르다 — `lpr -d`(❌), `lp -P`(❌) 형태의 오답 선지가 최빈출
- `lp -o <옵션>` : 인쇄 옵션 (`sides=two-sided-long-edge`, `media=A4`, `fit-to-page`)

**검증**

```bash
lpstat -o
lpq -P labprn
ls -l /var/spool/cups/ | head
```

```text
# lpstat -o
labprn-1    root    1024   Thu 03 Sep 2026 11:40:02 AM KST
labprn-2    root    1024   Thu 03 Sep 2026 11:40:05 AM KST
labprn-3    root      12   Thu 03 Sep 2026 11:40:08 AM KST
# lpq -P labprn
labprn is ready and printing
Rank    Owner   Job     File(s)                         Total Size
active  root    1       hosts                           1024 bytes
1st     root    2       hosts-copy                      1024 bytes
2nd     root    3       hostname                        1024 bytes
```

- 장치가 실제로 없으므로 작업이 `active` 상태로 머물거나 처리 실패 후 대기 — 큐 관리 실습에는 문제 없음

> 📝 **시험 포인트**: 필기 FULL r04-50, r05-55, r06-49 — "3부 인쇄" 는 `lp -n 3 <파일>`(System V) 또는 `lpr -# 3 <파일>`(BSD). "프린터 지정" 은 `lp -d` / `lpr -P`. r05-55 는 BSD 계열에서 `-P` 가 정답.

### 8-5. 큐 확인·취소 — lpq · lprm · cancel

> **상황**: 쌓인 작업을 확인하고 지운다.

```bash
lpq                          # 기본 프린터 큐
lpq -P labprn                # 특정 프린터 큐
lpq -a                       # 모든 프린터 큐
lpstat -o                    # System V 방식 작업 목록

lprm -P labprn 3             # BSD: 작업 번호 3 삭제
lprm -P labprn -             # BSD: 자기 작업 전부 삭제
cancel labprn-2              # System V: 작업 ID 지정 삭제
cancel -a labprn             # System V: 해당 프린터 전체 취소
cancel -a                    # 모든 프린터 전체 취소
```

| 계열 | 큐 조회 | 작업 취소 |
| --- | --- | --- |
| BSD | `lpq [-P 프린터]` | `lprm [-P 프린터] <번호>` / `lprm -` |
| System V | `lpstat -o` | `cancel <작업ID>` / `cancel -a <프린터>` |

**검증**

```bash
lpq -P labprn
lpstat -o
lpstat -o | wc -l
```

```text
# lpq -P labprn  (전체 취소 후)
labprn is ready
no entries
# lpstat -o
(출력 없음)
```

> 📝 **시험 포인트**: 필기 FULL r02-54, r04-51 — "인쇄 대기열의 작업을 삭제" 는 `lprm`(BSD) 또는 `cancel`(System V). "`lprm` 이 cupsd 를 재시작한다"·"`lpq` 가 작업을 등록한다" 는 오답.

### 8-6. 프린터 활성·수락 제어

> **상황**: 프린터 점검 중일 때 "작업은 받되 인쇄는 멈춤" 과 "작업 접수 자체를 차단" 을 구분해 적용한다.

```bash
cupsdisable labprn                     # 인쇄 중지 (큐에는 계속 쌓임)
lpstat -p labprn
lp -d labprn /etc/hostname             # 접수는 됨
lpstat -o

cupsreject labprn                      # 새 작업 접수 거부
lpstat -a
lp -d labprn /etc/hostname             # 거부됨

cupsaccept labprn                      # 접수 재개
cupsenable labprn                      # 인쇄 재개
lpstat -p -a
```

| 명령 | 대상 | 효과 |
| --- | --- | --- |
| `cupsdisable <프린터>` | **인쇄 동작** | 출력 중지, 큐 접수는 계속 |
| `cupsenable <프린터>` | 인쇄 재개 | |
| `cupsreject <프린터>` | **작업 접수** | 새 작업 거부 (기존 큐는 유지) |
| `cupsaccept <프린터>` | 접수 재개 | |
| `-r "<사유>"` | 두 disable/reject 에 사유 첨부 | `lpstat` 에 표시 |

- 구 명령 `enable`/`disable`(셸 내장과 충돌), `accept`/`reject` 는 `cups*` 접두 형태로 대체됨

**검증**

```bash
lpstat -p labprn
lpstat -a
cupsdisable -r "maintenance" labprn; lpstat -p labprn
cupsenable labprn; lpstat -p labprn
```

```text
printer labprn disabled since ... -
	maintenance
printer labprn is idle.  enabled since ...
```

> 📝 **시험 포인트**: `cupsdisable`(출력 중지) vs `cupsreject`(접수 거부) 의 차이가 핵심. "큐에는 쌓이지만 인쇄는 안 됨" = `cupsdisable`.

### 8-7. cupsd.conf 와 접근 제어

> **상황**: 원격에서 웹 관리에 접속하려면 리스닝 주소와 접근 제어를 알아야 한다. 여기서는 구조만 확인하고 실제 개방은 SSH 터널로 대체한다.

```bash
cp /etc/cups/cupsd.conf /etc/cups/cupsd.conf.orig
grep -vE '^\s*#|^\s*$' /etc/cups/cupsd.conf | head -40
```

```
LogLevel warn
MaxLogSize 0
Listen localhost:631
Listen /run/cups/cups.sock
Browsing Off
BrowseLocalProtocols dnssd
DefaultAuthType Basic
WebInterface Yes

<Location />
  Order allow,deny
</Location>

<Location /admin>
  Order allow,deny
</Location>

<Location /admin/conf>
  AuthType Default
  Require user @SYSTEM
  Order allow,deny
</Location>
```

| 지시자 | 의미 |
| --- | --- |
| `Listen <주소>:631` | 수신 주소·포트. `Port 631` 은 모든 인터페이스 |
| `Browsing On\|Off` | 네트워크 프린터 브로드캐스트 공유 |
| `BrowseLocalProtocols` | 공유 프로토콜(`dnssd` = mDNS/Bonjour) |
| `WebInterface Yes\|No` | 웹 관리 UI 사용 여부 |
| `DefaultAuthType Basic` | 기본 인증 방식 |
| `<Location <경로>>` | 경로별 접근 제어 블록 (Apache 문법과 유사) |
| `Order allow,deny` / `deny,allow` | 검사 순서 |
| `Allow @LOCAL` / `Allow 192.168.64.0/24` | 허용 대상 |
| `Require user @SYSTEM` | 관리 그룹(`SystemGroup`) 사용자만 |

- 원격 개방 예 (**이 실습에서는 적용하지 않음** — SSH 터널 사용)

```
# Listen 192.168.64.10:631
# <Location />
#   Order allow,deny
#   Allow 192.168.64.0/24
# </Location>
```

**검증**

```bash
grep -nE '^Listen|^Browsing|^WebInterface' /etc/cups/cupsd.conf
grep -nA3 '<Location />' /etc/cups/cupsd.conf
ss -tlnp | grep 631
```

```text
Listen localhost:631
Listen /run/cups/cups.sock
Browsing Off
WebInterface Yes
```

> 📝 **시험 포인트**: CUPS 접근 제어는 Apache 2.2 스타일 `Order`/`Allow` 문법을 그대로 쓴다. `<Location /admin>` 은 관리 페이지 전용 블록.

### 8-8. 웹 관리 631 — SSH 로컬 포워딩으로 접속

> **상황**: `cupsd` 는 루프백만 듣고 있다. 설정을 열지 않고 Part 08 에서 익힌 SSH 로컬 포워딩으로 macOS 브라우저에서 안전하게 접속한다.

```bash
# macOS 터미널에서 실행 (게스트 아님). Part 08 에서 SSH 포트를 2222 로 바꿨다면 -p 2222
ssh -p 2222 -N -L 6310:127.0.0.1:631 admin1@192.168.64.10
```

- `-L <로컬포트>:<목적지호스트>:<목적지포트>` : **로컬 포워딩** — macOS 의 6310 으로 온 연결을 SSH 터널을 통해 서버의 `127.0.0.1:631` 로 전달
- `-N` : 원격 명령 실행 없이 터널만 유지 (**N**o command)
- `-f` : 백그라운드 실행
- 상세 → [[08-network-config]]

```text
macOS 브라우저 → http://127.0.0.1:6310/
  → CUPS 관리 화면 (Printers 탭에서 labprn 확인)
```

- 관리 작업(프린터 추가·삭제)에는 서버의 `root` 또는 `SystemGroup`(보통 `wheel`) 계정 인증이 필요
- 방화벽에 631 을 여는 것보다 **SSH 터널이 안전** — 인증·암호화가 SSH 계층에서 처리됨

**검증**

```bash
# 게스트에서 터널 없이도 확인 가능한 부분
curl -s http://127.0.0.1:631/ | grep -o '<title>[^<]*</title>'
curl -s http://127.0.0.1:631/printers/ | grep -o 'labprn' | head -2
# macOS 에서 (터널 연결 상태)
# curl -s http://127.0.0.1:6310/ | head -3
```

```text
<title>Home - CUPS 2.x</title>
labprn
```

> 📝 **시험 포인트**: 필기 FULL r04-49, r06-48 — CUPS 웹 관리 포트는 **631**. SSH 로컬 포워딩 `-L` 은 "로컬 포트 → 원격 서비스", 원격 포워딩 `-R` 은 반대 방향.

### 8-9. BSD vs System V 인쇄 명령 대응표

> **상황**: 이 파트에서 가장 자주 출제되는 표다. 두 계열을 나란히 정리한다.

| 기능 | **BSD 계열** | **System V 계열** |
| --- | --- | --- |
| 인쇄 요청 | `lpr` | `lp` |
| 큐(작업) 확인 | `lpq` | `lpstat -o` |
| 작업 취소 | `lprm` | `cancel` |
| 프린터 상태 | `lpc status` | `lpstat -p` |
| 프린터 관리(등록·삭제) | `lpc` (제한적) | **`lpadmin`** |
| 프린터 지정 옵션 | **`-P <프린터>`** | **`-d <프린터>`** |
| 매수 지정 옵션 | **`-# <매수>`** | **`-n <매수>`** |
| 설정 파일(레거시) | `/etc/printcap` | `/etc/cups/printers.conf` |

```bash
# 같은 작업을 두 계열로
lpr -P labprn -# 2 /etc/hostname          # BSD
lp  -d labprn -n 2 /etc/hostname          # System V
lpq -P labprn                              # BSD 큐
lpstat -o                                  # System V 큐
lprm -P labprn -                           # BSD 전체 취소
cancel -a labprn                           # System V 전체 취소
```

- `lpc` : BSD 프린터 제어 셸 (`lpc status`·`lpc stop`·`lpc start`) — CUPS 에서는 기능이 축소됨
- CUPS 는 **양쪽 명령을 모두 제공**하는 호환 계층이라 어느 쪽을 써도 동작

**검증**

```bash
lpq -P labprn; echo '---'; lpstat -o; echo '---'
lpc status labprn 2>/dev/null || echo 'lpc: limited in CUPS'
which lp lpr lpq lprm lpstat cancel lpadmin
```

```text
labprn is ready
no entries
---
---
/usr/bin/lp
/usr/bin/lpr
/usr/bin/lpq
/usr/bin/lprm
/usr/bin/lpstat
/usr/bin/cancel
/usr/sbin/lpadmin
```

> 📝 **시험 포인트**: 필기 FULL r10-55 가 정확히 이 표 — "BSD 계열은 `lpr`·`lpq`·`lprm`, System V 계열은 `lp`·`lpstat`·`cancel`". r05-55·r06-49 는 `-P`(BSD) vs `-d`(System V) 를 뒤바꾼 선지로 출제.

### 8-10. 프린터 삭제와 정리 · 스캐너 참고

> **상황**: 실습이 끝났으니 더미 프린터를 정리한다.

```bash
cancel -a labprn
lpadmin -x labprn                # 프린터 삭제
lpstat -p 2>&1 | head -2
cat /etc/cups/printers.conf 2>/dev/null | head -3
```

- `-x <이름>` : 프린터 제거 — `printers.conf` 에서도 사라짐
- 실습을 계속 유지하려면 이 단계를 건너뛰어도 됨(10절 최종 점검에 `cups` 포함)

**스캐너 — SANE (참고, ※ 미실행)**

| 항목 | 내용 |
| --- | --- |
| 패키지 | `sane-backends`(드라이버), `sane-frontends`, `xsane`(GUI) |
| 명령 | `scanimage -L`(장치 목록), `scanimage > out.pnm`(스캔) |
| 설정 | `/etc/sane.d/*.conf` |
| 구조 | **백엔드**(장치 드라이버) + **프론트엔드**(사용자 도구) |

- UTM 가상 머신에는 스캐너 장치가 없어 실행 불가 — 개념·명령만 정리

**검증**

```bash
lpstat -p; lpstat -d
lpstat -t | head -3
```

```text
lpstat: No destinations added.
no system default destination
scheduler is running
```

> 📝 **시험 포인트**: SANE 은 **S**canner **A**ccess **N**ow **E**asy — 백엔드/프론트엔드 구조. 인쇄는 CUPS, 스캔은 SANE 으로 짝지어 출제.

---

## 9. 기타 필기 범위 서비스

### 9-1. DHCP 서버 — 설정 작성 + 문법 검사만

> **상황**: 필기 범위이지만 이 VM 은 UTM 의 NAT DHCP(192.168.64.1)를 쓰고 있어 실제로 띄우면 **IP 충돌로 네트워크가 끊긴다**. 설정 파일 작성과 문법 검사까지만 한다.

⚠️ **`systemctl start dhcpd` 를 실행하지 말 것** — UTM Shared Network 의 DHCP 와 충돌해 SSH 세션이 끊길 수 있다.

```bash
dnf install -y dhcp-server
rpm -ql dhcp-server | grep -E '^/etc|^/var/lib'
cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.orig 2>/dev/null

cat > /etc/dhcp/dhcpd.conf <<'EOF'
# 전역 설정
option domain-name              "lab.local";
option domain-name-servers      192.168.64.10, 192.168.64.1;
default-lease-time              600;
max-lease-time                  7200;
authoritative;
log-facility                    local7;

# 서브넷 선언
subnet 192.168.64.0 netmask 255.255.255.0 {
        range                       192.168.64.100 192.168.64.200;
        option routers              192.168.64.1;
        option subnet-mask          255.255.255.0;
        option broadcast-address    192.168.64.255;
        option domain-name-servers  192.168.64.10;
        default-lease-time          3600;
        max-lease-time              86400;
}

# 고정 할당 (MAC 기반 예약)
host labprinter {
        hardware ethernet   52:54:00:12:34:56;
        fixed-address       192.168.64.50;
}
EOF
```

| 지시자 | 의미 |
| --- | --- |
| `subnet <네트워크> netmask <마스크>` | 이 대역에 대한 설정 블록 |
| `range <시작> <끝>` | **동적 할당 IP 범위** |
| `option routers` | 클라이언트에 전달할 **기본 게이트웨이** |
| `option subnet-mask` | 서브넷 마스크 |
| `option broadcast-address` | 브로드캐스트 주소 |
| `option domain-name-servers` | **DNS 서버 주소**(쉼표로 다수) |
| `option domain-name` | 검색 도메인 |
| `default-lease-time` | 클라이언트가 요청하지 않았을 때의 **기본 임대 시간(초)** |
| `max-lease-time` | 허용 최대 임대 시간 |
| `authoritative` | 이 서버가 해당 대역의 공식 DHCP 임을 선언(잘못된 요청에 NAK 응답) |
| `host <이름> { hardware ethernet <MAC>; fixed-address <IP>; }` | **MAC 기반 고정 IP 예약** |

**검증 — 문법 검사만**

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
ls -l /var/lib/dhcpd/dhcpd.leases
systemctl is-active dhcpd            # inactive 여야 정상 (기동하지 않음)
```

```text
# dhcpd -t -cf /etc/dhcp/dhcpd.conf
Internet Systems Consortium DHCP Server 4.4.x
Copyright 2004-... Internet Systems Consortium.
...
Config file: /etc/dhcp/dhcpd.conf
Database file: /var/lib/dhcpd/dhcpd.leases
PID file: /run/dhcpd.pid
# systemctl is-active dhcpd
inactive
```

- `-t` : 설정 문법 검사 (**t**est), `-cf` : **c**onfig **f**ile 지정
- 오류가 있으면 `Configuration file errors encountered -- exiting` 과 함께 줄 번호 표시
- `/var/lib/dhcpd/dhcpd.leases` : **임대 이력** — 어떤 MAC 이 언제부터 언제까지 어떤 IP 를 쓰는지 기록

**DORA 절차**

| 단계 | 방향 | 내용 |
| --- | --- | --- |
| **D**iscover | 클라이언트 → **브로드캐스트** | DHCP 서버를 찾음 |
| **O**ffer | 서버 → 클라이언트 | 사용 가능한 IP 제안 |
| **R**equest | 클라이언트 → 브로드캐스트 | 특정 제안을 수락 요청 |
| **A**ck | 서버 → 클라이언트 | 임대 확정 |

- 포트: 서버 **67/UDP**, 클라이언트 **68/UDP**
- **DHCP 릴레이 에이전트**: 다른 서브넷의 브로드캐스트 요청을 받아 원격 DHCP 서버로 **유니캐스트 중계** (`dhcrelay`)

> 📝 **시험 포인트**: 필기 FULL r01-84, r02-84, r05-86/87, r06-82, r08-87 — `option routers` = **기본 게이트웨이**(DHCP 서버 자신의 IP 아님), `range` = 임대 범위, 고정 할당은 `host { hardware ethernet; fixed-address; }`. r03-81·r07-84 는 DORA 순서, r10-84 는 릴레이 에이전트.

### 9-2. Squid 프록시 (※ 미실행)

> **상황**: 필기 범위이나 이 VM 에는 내부 클라이언트가 없어 의미가 없다. 설정 형태만 정리한다.

```bash
# dnf install -y squid          # ※ 미실행
cat <<'EOF'
# /etc/squid/squid.conf 핵심
http_port 3128                              # 프록시 수신 포트 (기본 3128)
visible_hostname srv01.lab.local            # 오류 페이지에 표시될 이름

acl localnet src 192.168.64.0/24            # 접근 제어 목록 정의
acl SSL_ports port 443
acl Safe_ports port 80 21 443 70 210 1025-65535
acl CONNECT method CONNECT

http_access deny !Safe_ports                # 위험 포트 차단
http_access deny CONNECT !SSL_ports
http_access allow localhost manager
http_access deny manager
http_access allow localnet                  # 사내망 허용
http_access allow localhost
http_access deny all                        # 나머지 전부 거부 (마지막 줄)

cache_dir ufs /var/spool/squid 100 16 256   # 캐시 디렉터리 100MB, 1단 16, 2단 256
cache_mem 256 MB
access_log /var/log/squid/access.log
EOF
```

| 지시자 | 의미 |
| --- | --- |
| `http_port` | 프록시 수신 포트 — **기본 3128** |
| `acl <이름> <타입> <값>` | 접근 제어 목록 정의 (`src`·`dst`·`port`·`method`·`dstdomain`·`time`) |
| `http_access allow\|deny <acl>` | **위에서부터 순서대로** 평가, 첫 매칭이 결정 |
| `cache_dir <타입> <경로> <MB> <L1> <L2>` | 디스크 캐시 |
| `cache_mem` | 메모리 캐시 크기 |
| `visible_hostname` | 오류 페이지·헤더에 표시될 호스트명 |

- **순서가 중요** — `http_access deny all` 은 반드시 **맨 마지막**. 위에 두면 그 아래 규칙이 전부 무시됨
- 정방향 프록시(클라이언트 대행) / 리버스 프록시(서버 앞단 캐시) 개념 구분

> 📝 **시험 포인트**: 필기 FULL r01-88, r02-85, r03-82, r05-90, r06-85, r07-90, r10-86 — Squid 기본 포트는 **3128**(1080 은 SOCKS). `acl` 정의 + `http_access` 로 허용·차단, 규칙은 **위에서부터 첫 매칭**.

### 9-3. xinetd 부재와 systemd 소켓 활성화

> **상황**: 필기에는 슈퍼데몬 `xinetd` 가 나오지만 **RHEL 9/Rocky 9 기본 저장소에는 없다**. systemd 의 소켓 활성화가 그 자리를 대신한다.

```bash
dnf list xinetd 2>&1 | tail -2        # 기본 저장소에서 조회되지 않음
systemctl list-sockets --all | head -15
systemctl cat sshd.socket 2>/dev/null | head -20
ls /usr/lib/systemd/system/*.socket | head
```

```text
# dnf list xinetd
오류: 일치하는 패키지가 없습니다.        ← 기본 저장소에 없음 (※ 미실행)
# systemctl list-sockets --all | head
LISTEN                        UNIT                         ACTIVATES
/dev/log                      systemd-journald.socket      systemd-journald.service
/run/dbus/system_bus_socket   dbus.socket                  dbus-broker.service
/run/systemd/journal/socket   systemd-journald.socket      systemd-journald.service
...
```

**xinetd 개념 (필기용)**

| 항목 | 내용 |
| --- | --- |
| 설정 | `/etc/xinetd.conf`, 서비스별 `/etc/xinetd.d/<서비스>` |
| `disable = no` | 서비스 **활성화**(기본은 `yes` = 비활성) |
| `only_from` | 접속 허용 IP·대역 |
| `no_access` | 접속 거부 대상 |
| `access_times` | 허용 시간대 (`08:00-18:00`) |
| `instances` | 최대 동시 서버 수 |
| `per_source` | 출발지 IP 당 최대 연결 |
| `server` / `server_args` | 실행할 프로그램·인자 |
| `socket_type` / `wait` / `user` | 소켓 형식·대기 방식·실행 계정 |

**standalone vs 슈퍼데몬 방식**

| 구분 | standalone | inetd/xinetd |
| --- | --- | --- |
| 데몬 상주 | **항상 메모리에 상주** | 요청 시 슈퍼데몬이 기동 |
| 응답 속도 | **빠름** | 기동 지연 있음 |
| 메모리 | 많이 씀 | 적게 씀 |
| 적합 | httpd·sshd 등 빈번한 서비스 | 드물게 쓰는 서비스 |

**systemd 소켓 유닛 구조 (대체 수단)**

```ini
# /usr/lib/systemd/system/foo.socket
[Unit]
Description=Foo Socket

[Socket]
ListenStream=12345          # TCP 포트 (또는 유닉스 소켓 경로)
Accept=yes                  # 연결마다 인스턴스 생성 (foo@.service)

[Install]
WantedBy=sockets.target
```

- `Accept=yes` → 연결 1개당 `foo@<n>.service` 인스턴스 생성 (xinetd 와 동일한 동작)
- `Accept=no` → 소켓을 그대로 서비스에 넘김 (대부분의 현대 서비스)

**검증**

```bash
systemctl list-sockets --all | wc -l
systemctl list-unit-files --type=socket | head -10
ls /etc/xinetd.d/ 2>/dev/null || echo 'xinetd not installed (RHEL 9)'
```

```text
xinetd not installed (RHEL 9)
```

> 📝 **시험 포인트**: 필기 FULL r01-53, r09-44, r10-14 — xinetd 의 `disable = no` 는 **활성화**, `only_from` 은 허용 IP. standalone(상주·빠름) vs 슈퍼데몬(요청 시 기동·메모리 절약) 비교. **RHEL 9 에서는 xinetd 가 제거되고 systemd 소켓 활성화로 대체**되었다는 점도 함께 알아 둘 것.

### 9-4. NIS · LDAP · Kerberos 개념

> **상황**: 인증 서비스는 시험 범위지만 서버·클라이언트 2대 이상이 필요하다. 개념과 명령만 정리한다. (※ 미실행)

**NIS (Network Information Service)**

| 항목 | 내용 |
| --- | --- |
| 목적 | `passwd`·`group`·`hosts` 등 계정 정보를 **중앙 관리** |
| 기반 | **RPC** — `rpcbind` 필수 |
| 서버 데몬 | `ypserv`, `yppasswdd`, `ypxfrd` |
| 클라이언트 데몬 | `ypbind` |
| 도메인 설정 | `nisdomainname <도메인>` 또는 `/etc/sysconfig/network` 의 `NISDOMAIN` |
| 클라이언트 설정 | `/etc/yp.conf` (`domain <도메인> server <서버IP>`) |
| 이름 해석 연동 | `/etc/nsswitch.conf` 에 `nis` 추가 (`passwd: files nis`) |
| 맵 생성 | 서버에서 `make -C /var/yp` |

| 명령 | 의미 |
| --- | --- |
| `ypwhich` | 현재 바인딩된 NIS 서버 확인 |
| `ypcat passwd` | NIS 맵 내용 조회 |
| `ypmatch <키> <맵>` | 특정 항목 조회 |
| `yppasswd` | NIS 비밀번호 변경 |
| `ypbind` / `ypserv` | 클라이언트 / 서버 데몬 |
| `yptest` | 동작 종합 점검 |

**LDAP (Lightweight Directory Access Protocol)**

| 항목 | 내용 |
| --- | --- |
| 포트 | **389**(평문/STARTTLS), **636**(LDAPS) |
| 서버 데몬 | `slapd` (OpenLDAP) |
| 구조 | 트리(DIT) — `dn`(고유 식별명) 을 경로로 사용 |
| 속성 | `dc`(domain component) · `ou`(organizational unit) · `cn`(common name) · `uid` · `o`(organization) |
| DN 예 | `cn=dev1,ou=People,dc=lab,dc=local` |
| 도구 | `ldapsearch`, `ldapadd`, `ldapmodify`, `ldapdelete`, `ldappasswd` |
| 데이터 형식 | **LDIF** (LDAP Data Interchange Format) |

```bash
# 조회 예 (※ 미실행 — LDAP 서버 없음)
# ldapsearch -x -H ldap://192.168.64.10 -b "dc=lab,dc=local" "(uid=dev1)"
```

- `-x` : 단순 인증 (SASL 대신), `-H` : 서버 URI, `-b` : 검색 기준 DN (**b**ase)

**Kerberos**

| 항목 | 내용 |
| --- | --- |
| 포트 | **88**(KDC), 464(비밀번호 변경), 749(관리) |
| 구성 | KDC = **AS**(인증 서버) + **TGS**(티켓 부여 서버), 클라이언트, 서비스 서버 |
| 흐름 | 클라이언트 → AS 인증 → **TGT** 발급 → TGT 로 TGS 에 요청 → **서비스 티켓** 발급 → 서비스 접속 |
| 설정 | `/etc/krb5.conf` |
| 명령 | `kinit`(티켓 획득), `klist`(티켓 확인), `kdestroy`(티켓 삭제) |
| 특징 | **대칭키 기반 상호 인증**, 비밀번호가 네트워크를 흐르지 않음, 시각 동기(NTP) 필수 |

> 📝 **시험 포인트**: 필기 FULL r01-87, r03-83, r07-85, r08-90, r09-87 — LDAP 은 **389/636**, `ou` = 조직 단위, `dc` = 도메인 구성요소. r07-94 — Kerberos 흐름은 AS 인증 → TGT → TGS → 서비스 티켓 순서. NIS 는 `ypcat`·`ypwhich` 명령과 RPC 의존.

### 9-5. telnet-server 위험성과 대체

> **상황**: 필기에 telnet 이 나오므로 위험성과 대체 수단을 확인한다. 서버는 설치만 확인하고 **활성화하지 않는다**.

⚠️ `telnet-server` 는 **인증 정보와 모든 통신이 평문**으로 흐른다. 실습 목적이라도 상시 활성화 금지.

```bash
dnf info telnet-server 2>/dev/null | head -8
systemctl list-unit-files | grep -i telnet
# systemctl enable --now telnet.socket        # ※ 실행하지 않음
```

| 항목 | telnet | ssh |
| --- | --- | --- |
| 포트 | **23/TCP** | **22/TCP** |
| 암호화 | **없음(평문)** | 있음 |
| 인증 | 평문 ID/비밀번호 | 비밀번호 + **공개키 인증** |
| 무결성 | 없음 | MAC 검증 |
| 부가 기능 | 없음 | 포트 포워딩·SFTP·SCP·에이전트 |
| RHEL 9 기동 방식 | `telnet.socket`(소켓 활성화) | `sshd.service`(상주) |

- 평문 프로토콜 대체 짝: **telnet → ssh**, **FTP → SFTP/SCP**, **HTTP → HTTPS**, **POP3/IMAP → POP3S/IMAPS**, **SMTP → SMTPS/STARTTLS**
- `telnet` **클라이언트**는 7-10·7-16·7-17 처럼 프로토콜 디버깅 도구로 유용 — 서버(`telnet-server`)와 구분

**검증**

```bash
systemctl is-enabled telnet.socket 2>&1 | head -1
ss -tlnp | grep -c ':23 ' || echo 'port 23 closed'
which telnet
```

```text
Failed to get unit file state for telnet.socket: No such file or directory
port 23 closed
/usr/bin/telnet
```

> 📝 **시험 포인트**: 필기 FULL r06-99, r10-85 — telnet 은 23/TCP 평문이라 도청에 취약, 대체는 SSH. "telnet 은 기본적으로 암호화한다" 는 오답. RHEL 9 에서 telnet 서버는 `telnet.socket`(소켓 활성화) 로 제공된다는 점도 함께.

### 9-6. 웹·서비스 성능 점검 도구 요약

> **상황**: 서비스별 부하·성능 점검 도구를 한 표로 정리한다.

| 계층 | 도구 | 용도 |
| --- | --- | --- |
| 웹 | `ab -n <횟수> -c <동시> <URL>` | Apache Bench — 요청 처리량·응답 시간 (2-15) |
| 웹 | `curl -w '%{time_total}\n' -o /dev/null -s <URL>` | 단일 요청 소요 시간 |
| 네트워크 | `nc -zv <호스트> <포트>` | 포트 개방 확인 (Part 08) |
| 네트워크 | `nmap -sT <호스트>` | 포트 스캔 ([[10-security-firewall-selinux]]) |
| NFS | `nfsstat -c` / `-s` | 클라이언트·서버 RPC 통계 |
| Samba | `smbstatus` | 세션·공유·잠금 현황 |
| 시스템 | `vmstat` `iostat` `sar` `top` | CPU·I/O·메모리 (Part 06) |

```bash
ab -n 50 -c 5 http://127.0.0.1/ 2>/dev/null | grep -E 'Requests per second|Failed'
curl -o /dev/null -s -w 'total=%{time_total}s  code=%{http_code}\n' http://intranet.lab.local/
nfsstat -c | head -5
```

**검증**

```bash
curl -o /dev/null -s -w 'connect=%{time_connect}s ttfb=%{time_starttransfer}s total=%{time_total}s\n' http://127.0.0.1/
```

```text
connect=0.000...s ttfb=0.00...s total=0.00...s
```

> 📝 **시험 포인트**: `ab` 는 웹 계층 부하, `stress-ng` 는 시스템 부하(Part 06), `nmap`·`nc` 는 포트 점검(Part 08·10). 계층별 도구 매칭으로 출제.

---

## 10. 마무리 — 전 서비스 종합 점검

### 10-1. 서비스 상태 일괄 확인

> **상황**: 7개 서비스를 모두 올렸다. 한 줄로 전부 살아 있는지 확인한다.

```bash
systemctl is-active httpd named nfs-server smb nmb vsftpd postfix dovecot cups
systemctl is-enabled httpd named nfs-server smb nmb vsftpd postfix dovecot cups
systemctl --failed --no-pager
systemctl list-units --type=service --state=running | grep -E 'httpd|named|nfs|smb|nmb|vsftpd|postfix|dovecot|cups'
```

- `is-active` : 실행 중이면 `active`, 아니면 `inactive`/`failed` — **인자 여러 개를 순서대로 한 줄씩** 출력
- `is-enabled` : 부팅 등록 여부
- `--failed` : 실패한 유닛만 — 아무것도 안 나와야 정상

**검증**

```bash
for s in httpd named nfs-server smb nmb vsftpd postfix dovecot cups; do
  printf '%-12s %-10s %s\n' "$s" "$(systemctl is-active $s)" "$(systemctl is-enabled $s)"
done
```

```text
httpd        active     enabled
named        active     enabled
nfs-server   active     enabled
smb          active     enabled
nmb          active     enabled
vsftpd       active     enabled
postfix      active     enabled
dovecot      active     enabled
cups         active     enabled
```

> 📝 **시험 포인트**: 실기 r01-8 — "`httpd` 를 부팅 시 자동 시작 등록" = `systemctl enable httpd`. `is-active`(현재 실행) 와 `is-enabled`(부팅 등록) 는 **별개 상태**라는 점이 핵심.

### 10-2. 포트 ↔ 데몬 대응 확인

> **상황**: 지금 이 서버가 열어 둔 포트가 어떤 데몬 것인지 한눈에 대조한다. 필기의 "서비스-포트 매칭" 을 실물로 확인하는 단계다.

```bash
ss -tulnp | sort -k5
ss -tlnp | awk 'NR>1 {print $5, $NF}' | sort -t: -k2 -n
ss -tulnp | grep -oE 'users:\(\("[^"]+' | sort -u
```

- `-t` TCP · `-u` UDP · `-l` LISTEN · `-n` 숫자 표기 · `-p` 프로세스
- `sort -k5` : 5번째 필드(로컬 주소:포트) 기준 정렬

| 포트 | 프로토콜 | 데몬 | 서비스 |
| --- | --- | --- | --- |
| 21 | TCP | `vsftpd` | FTP 제어 |
| 22 (또는 2222) | TCP | `sshd` | SSH (Part 08) |
| 25 | TCP | `master` | Postfix SMTP |
| 53 | TCP·UDP | `named` | DNS |
| 80 · 443 · 8080 | TCP | `httpd` | 웹 |
| 110 · 143 · 993 · 995 | TCP | `dovecot` | POP3·IMAP(+S) |
| 111 | TCP·UDP | `rpcbind` | RPC 포트 매퍼 |
| 137 · 138 | UDP | `nmbd` | NetBIOS |
| 139 · 445 | TCP | `smbd` | SMB |
| 631 | TCP | `cupsd` | CUPS(IPP) |
| 2049 | TCP | 커널 `nfsd` | NFS |
| 40000–40100 | TCP | `vsftpd` | FTP 수동 데이터 |

**검증**

```bash
for p in 21 25 53 80 110 139 143 445 631 2049 8080; do
  printf '%-6s %s\n' "$p" "$(ss -tlnH "sport = :$p" | head -1 | awk '{print $4}')"
done
ss -tulnp | grep -cE 'httpd|named|smbd|nmbd|vsftpd|master|dovecot|cupsd'
```

```text
21     *:21
25     0.0.0.0:25
53     127.0.0.1:53
80     *:80
110    0.0.0.0:110
139    *:139
143    0.0.0.0:143
445    *:445
631    127.0.0.1:631
2049   0.0.0.0:2049
8080   *:8080
```

> 📝 **시험 포인트**: 실기 r01-5, r03-8 — `ss -tlnp | grep :8080`, `ss -ulnp | grep :53`. `nfsd` 는 **커널 스레드**라 `-p` 로도 프로세스명이 안 나올 수 있다는 점이 함정.

### 10-3. 방화벽 최종 상태

> **상황**: 이 파트에서 연 규칙을 전부 확인한다. 정책 설계·강화는 [[10-security-firewall-selinux]] 에서 이어 간다.

```bash
firewall-cmd --state
firewall-cmd --get-default-zone
firewall-cmd --list-all
firewall-cmd --permanent --list-all
firewall-cmd --list-services | tr ' ' '\n' | sort
```

**검증**

```bash
firewall-cmd --list-all
```

```text
public (active)
  target: default
  interfaces: enp0s1
  sources:
  services: cockpit dhcpv6-client dns ftp http https imap nfs pop3 rpc-bind mountd samba smtp ssh
  ports: 8080/tcp 40000-40100/tcp
  protocols:
  forward: yes
  masquerade: no
  ...
```

- `--list-all`(런타임) 과 `--permanent --list-all`(영구) 이 **일치해야** 재부팅 후에도 유지됨
- 불일치하면 `firewall-cmd --runtime-to-permanent` 로 저장

> 📝 **시험 포인트**: 실기 r02-9, r05-4 — `firewall-cmd --permanent --add-service=http` 뒤 반드시 `firewall-cmd --reload`. `--permanent` 없이 추가한 규칙은 재부팅 시 사라진다.

### 10-4. 서비스별 로그 위치 요약

> **상황**: 장애 시 어디를 볼지 한 표로 정리한다.

| 서비스 | 로그 위치 | 조회 명령 |
| --- | --- | --- |
| Apache | `/var/log/httpd/access_log`, `error_log`, vhost 별 `*-access_log` | `tail -f`, `journalctl -u httpd` |
| BIND | journald + `/var/log/messages` | `journalctl -u named` |
| NFS | journald | `journalctl -u nfs-server`, `nfsstat` |
| Samba | `/var/log/samba/log.smbd`, `log.nmbd`, `log.<클라이언트>` | `tail`, `smbstatus` |
| vsftpd | `/var/log/xferlog`(전송), `/var/log/vsftpd.log`, `/var/log/secure`(인증) | `tail`, `journalctl -u vsftpd` |
| Postfix | **`/var/log/maillog`** | `grep status=sent`, `mailq` |
| Dovecot | `/var/log/maillog` + journald | `journalctl -u dovecot` |
| CUPS | `/var/log/cups/{access_log,error_log,page_log}` | `tail` |
| 방화벽 | journald (`--log-denied` 설정 시) | `journalctl -u firewalld` |
| SELinux 거부 | `/var/log/audit/audit.log` | **`ausearch -m avc -ts recent`** |

```bash
ls -l /var/log/httpd/ /var/log/samba/ /var/log/cups/ 2>/dev/null | head -20
ls -l /var/log/maillog /var/log/xferlog /var/log/messages /var/log/secure
```

**검증**

```bash
for f in /var/log/maillog /var/log/xferlog /var/log/messages /var/log/secure \
         /var/log/httpd/error_log /var/log/samba/log.smbd /var/log/cups/error_log; do
  printf '%-35s %s\n' "$f" "$( [ -f "$f" ] && echo OK || echo MISSING )"
done
```

```text
/var/log/maillog                    OK
/var/log/xferlog                    OK
/var/log/messages                   OK
/var/log/secure                     OK
/var/log/httpd/error_log            OK
/var/log/samba/log.smbd             OK
/var/log/cups/error_log             OK
```

> 📝 **시험 포인트**: 필기 FULL r03-63 — `/var/log/maillog`(메일), `/var/log/cron`(cron), `/var/log/secure`(인증), `/var/log/messages`(일반). 로그 파일 ↔ 서비스 매칭은 매 회차 출제.

### 10-5. 검증 스크립트 — check-part09.sh

> **상황**: 다음에 이 VM 을 다시 켰을 때 7개 서비스가 정상인지 한 번에 확인할 스크립트를 남긴다.

```bash
cat > /usr/local/bin/check-part09.sh <<'EOF'
#!/bin/bash
# LAB 09 — 네트워크 서비스 종합 점검
# 사용법: check-part09.sh          (root 권장)

PASS=0; FAIL=0
ok()  { printf '[OK] %s\n' "$1"; PASS=$((PASS+1)); }
ng()  { printf '[NG] %s\n' "$1"; FAIL=$((FAIL+1)); }
chk() { if eval "$2" >/dev/null 2>&1; then ok "$1"; else ng "$1"; fi; }

echo "===== 1. 서비스 상태 ====="
for s in httpd named nfs-server smb nmb vsftpd postfix dovecot cups; do
    chk "service $s active"  "systemctl is-active --quiet $s"
    chk "service $s enabled" "systemctl is-enabled --quiet $s"
done

echo "===== 2. 리스닝 포트 ====="
for p in 21 25 53 80 110 139 143 445 631 2049 8080; do
    chk "port $p listening" "ss -tulnH 'sport = :$p' | grep -q ."
done

echo "===== 3. Apache ====="
chk "httpd syntax"            "httpd -t"
chk "default site 200"        "[ \$(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1/) = 200 ]"
chk "vhost intranet 200"      "[ \$(curl -s -o /dev/null -w '%{http_code}' -H 'Host: intranet.lab.local' http://127.0.0.1/) = 200 ]"
chk "secret area 401"         "[ \$(curl -s -o /dev/null -w '%{http_code}' -H 'Host: intranet.lab.local' http://127.0.0.1/secret/) = 401 ]"
chk "dir listing blocked 403" "[ \$(curl -s -o /dev/null -w '%{http_code}' -H 'Host: intranet.lab.local' http://127.0.0.1/files/) = 403 ]"
chk "https reachable"         "curl -k -s -o /dev/null https://127.0.0.1/"
chk "selinux /srv/www label"  "ls -ldZ /srv/www | grep -q httpd_sys_content_t"

echo "===== 4. BIND ====="
chk "named-checkconf"         "named-checkconf"
chk "zone forward OK"         "named-checkzone lab.local /var/named/lab.local.zone"
chk "zone reverse OK"         "named-checkzone 64.168.192.in-addr.arpa /var/named/192.168.64.rev"
chk "A  www.lab.local"        "[ \$(dig @127.0.0.1 www.lab.local +short) = 192.168.64.10 ]"
chk "PTR 192.168.64.10"       "dig @127.0.0.1 -x 192.168.64.10 +short | grep -q srv01"
chk "MX lab.local"            "dig @127.0.0.1 lab.local MX +short | grep -q mail.lab.local"
chk "AXFR denied"             "! dig @127.0.0.1 lab.local AXFR | grep -q 'XFR size'"

echo "===== 5. NFS ====="
chk "exportfs has share"      "exportfs -v | grep -q /srv/nfs/data"
chk "showmount -e"            "showmount -e 127.0.0.1 | grep -q /srv/nfs/data"
chk "/mnt/nfs mounted"        "findmnt /mnt/nfs"

echo "===== 6. Samba ====="
chk "testparm OK"             "testparm -s"
chk "share section exists"    "testparm -s 2>/dev/null | grep -q '\[share\]'"
chk "smb user dev1"           "pdbedit -L | grep -q '^dev1:'"
chk "selinux /srv/share"      "ls -ldZ /srv/share | grep -q samba_share_t"

echo "===== 7. vsftpd ====="
chk "anonymous disabled"      "grep -q '^anonymous_enable=NO' /etc/vsftpd/vsftpd.conf"
chk "chroot enabled"          "grep -q '^chroot_local_user=YES' /etc/vsftpd/vsftpd.conf"
chk "xferlog exists"          "[ -f /var/log/xferlog ]"

echo "===== 8. Mail ====="
chk "postfix check"           "postfix check"
chk "aliases.db newer"        "[ /etc/aliases.db -nt /etc/aliases ]"
chk "alias devs resolves"     "postalias -q devs /etc/aliases | grep -q dev1"
chk "smtp banner"             "printf 'QUIT\r\n' | timeout 3 nc 127.0.0.1 25 | grep -q '^220'"
chk "pop3 banner"             "printf 'QUIT\r\n' | timeout 3 nc 127.0.0.1 110 | grep -q '+OK'"
chk "imap banner"             "printf 'a logout\r\n' | timeout 3 nc 127.0.0.1 143 | grep -q '\* OK'"
chk "maillog has sent"        "grep -q 'status=sent' /var/log/maillog"

echo "===== 9. CUPS ====="
chk "scheduler running"       "lpstat -r | grep -q running"
chk "cups web port"           "curl -s -o /dev/null http://127.0.0.1:631/"

echo "===== 10. 방화벽 ====="
for svc in http https dns nfs samba ftp smtp imap pop3; do
    chk "firewall service $svc" "firewall-cmd --query-service=$svc"
done
chk "firewall port 8080"      "firewall-cmd --query-port=8080/tcp"

echo
printf '결과: PASS=%d  FAIL=%d\n' "$PASS" "$FAIL"
[ "$FAIL" -eq 0 ] && exit 0 || exit 1
EOF

chmod 755 /usr/local/bin/check-part09.sh
bash -n /usr/local/bin/check-part09.sh && echo 'syntax OK'
```

- `bash -n <파일>` : 실행하지 않고 문법만 검사 (**n**oexec) — Part 04 참조
- `ss -tulnH 'sport = :<포트>'` : 헤더 없이(**H**) 소스 포트로 필터
- `[ A -nt B ]` : A 가 B 보다 **최신**인지 (**n**ewer **t**han) — `newaliases` 실행 여부 판정에 사용

**검증**

```bash
/usr/local/bin/check-part09.sh | tail -20
/usr/local/bin/check-part09.sh | grep -c '^\[OK\]'
/usr/local/bin/check-part09.sh | grep '^\[NG\]'
echo "exit=$?"
```

```text
===== 10. 방화벽 =====
[OK] firewall service http
[OK] firewall service https
[OK] firewall service dns
[OK] firewall service nfs
[OK] firewall service samba
[OK] firewall service ftp
[OK] firewall service smtp
[OK] firewall service imap
[OK] firewall service pop3
[OK] firewall port 8080

결과: PASS=..  FAIL=0
```

- `[NG]` 가 나온 항목은 해당 절로 돌아가 재확인 — 대부분 ① 방화벽 `--reload` 누락 ② SELinux 컨텍스트 ③ 설정 반영(`reload`) 누락 셋 중 하나

> 📝 **시험 포인트**: 실기 서술형에서 "구축 후 정상 동작을 어떻게 확인할 것인가" 를 묻는 경우, **서비스 상태(`systemctl is-active`) → 포트(`ss`) → 방화벽(`firewall-cmd --list-all`) → 클라이언트 접속(`curl`·`dig`·`showmount`·`smbclient`) → 로그** 순서로 답하면 된다.

---

## 체크리스트

| 수행 항목 | 명령 | 확인 방법 | ☐ |
| --- | --- | --- | --- |
| 선행 자원 확인 | `hostnamectl`, `ip -4 a`, `findmnt /srv/share`, `id dev1` | IP 192.168.64.10, `/srv/share` 마운트 | ☐ |
| httpd·mod_ssl 설치 | `dnf install -y httpd httpd-tools mod_ssl` | `rpm -qf /etc/httpd/conf/httpd.conf` | ☐ |
| MPM·모듈 확인 | `httpd -V`, `httpd -M \| grep -E 'mpm\|ssl'` | `Server MPM: event` | ☐ |
| ServerName·Listen 8080 | `/etc/httpd/conf/httpd.conf` 편집 | `apachectl configtest` → `Syntax OK` | ☐ |
| httpd 기동·방화벽 | `enable --now httpd`, `--add-service={http,https}`, `--add-port=8080/tcp` | `ss -tlnp \| grep httpd`, `curl -I http://127.0.0.1` | ☐ |
| 기본 문서 200 | `/var/www/html/index.html` 작성 | `curl -I` → 200 | ☐ |
| 이름 기반 vhost | `conf.d/00-default.conf`, `conf.d/intranet.conf` | `httpd -S`, `curl -H 'Host: intranet.lab.local'` | ☐ |
| SELinux 403 재현·해결 | `semanage fcontext -a -t httpd_sys_content_t "/srv/www(/.*)?"` + `restorecon -Rv` | `ls -ldZ` → `httpd_sys_content_t`, 403 → 200 | ☐ |
| AVC 로그 확인 | `ausearch -m avc -ts recent` | `denied { getattr }` + `tcontext=…var_t` | ☐ |
| 이름 접속·macOS 확인 | `/etc/hosts` 별칭, macOS `/etc/hosts` | `curl http://intranet.lab.local/`, 브라우저 | ☐ |
| 디렉터리 인증 | `htpasswd -c /etc/httpd/.htpasswd dev1` + `.htaccess` + `AllowOverride AuthConfig` | 미인증 401 / 인증 200 | ☐ |
| AllowOverride None 검증 | 값을 `None` 으로 되돌리고 재시도 | `.htaccess` 무시 → 200 | ☐ |
| 목록 차단 | `Options -Indexes` | `/files/` → 403 | ☐ |
| 로그 필드 해석 | `tail /var/log/httpd/intranet-access_log` | combined 9개 필드, `awk '{print $9}'` | ☐ |
| HTTPS 자체 서명 | `openssl req -x509 -nodes -days 365 -newkey rsa:2048 …` + `ssl.conf` | `curl -k https://`, `openssl s_client` subject=issuer | ☐ |
| 포트 기반 vhost | `conf.d/port8080.conf` | `curl http://127.0.0.1:8080/` | ☐ |
| ab 부하 테스트 | `ab -n 100 -c 10 http://127.0.0.1/` | `Failed requests: 0` | ☐ |
| apachectl graceful | `apachectl graceful` | MainPID 유지 | ☐ |
| bind 설치·options | `dnf install -y bind bind-utils`, `/etc/named.conf` | `named-checkconf` 무출력 | ☐ |
| zone 선언 (정·역방향) | `zone "lab.local"`, `zone "64.168.192.in-addr.arpa"` | `named-checkconf -z` | ☐ |
| 정방향 존 작성 | `/var/named/lab.local.zone` (SOA·NS·A·CNAME·MX·TXT) | `named-checkzone lab.local …` → `OK` | ☐ |
| 역방향 존 작성 | `/var/named/192.168.64.rev` (PTR) | `named-checkzone 64.168.192.in-addr.arpa …` | ☐ |
| 존 파일 권한·라벨 | `chgrp named`, `chmod 640`, `restorecon -Rv /var/named` | `ls -lZ` → `named_zone_t` | ☐ |
| named 기동·방화벽 | `enable --now named`, `--add-service=dns` | `ss -tulnp \| grep :53` | ☐ |
| dig 레코드별 조회 | `dig @127.0.0.1 www.lab.local / MX / NS / SOA / TXT` | 각 ANSWER 확인 | ☐ |
| 역방향·AXFR 거부 | `dig -x 192.168.64.10`, `dig lab.local AXFR` | `srv01.lab.local.` / `Transfer failed.` | ☐ |
| 자기 DNS 지정 | `nmcli con mod enp0s1 ipv4.dns 127.0.0.1` → `con up` | `/etc/resolv.conf`, `ping intranet.lab.local` | ☐ |
| Serial 증가·rndc reload | Serial `+1` → `rndc reload lab.local` | `dig lab.local SOA +short` 값 변경 | ☐ |
| Serial 미증가 문제 이해 | 레코드만 추가 후 reload | 마스터는 반영 / 슬레이브 미갱신 개념 | ☐ |
| rndc 서브명령 | `status`, `flush`, `reconfig`, `querylog on/off` | `server is up and running` | ☐ |
| 존 오타 재현·복구 | 끝점 제거·구문 오류 → `named-checkzone` → 원복 | `not loaded due to errors` 확인 후 `OK` | ☐ |
| nfs-utils 설치·공유 | `mkdir -p /srv/nfs/data`, `chmod 2775` | `ls -ld` → `drwxrwsr-x` | ☐ |
| /etc/exports 작성 | 대역(`root_squash`) + 루프백(`no_root_squash`) | `cat -A /etc/exports` 공백 확인 | ☐ |
| exportfs 반영 | `exportfs -ra` → `-v` | `/var/lib/nfs/etab` | ☐ |
| nfs-server 기동·방화벽 | `enable --now nfs-server`, `--add-service={nfs,rpc-bind,mountd}` | `rpcinfo -p`, `/proc/fs/nfsd/versions` | ☐ |
| showmount 확인 | `showmount -e 192.168.64.10` | export list 출력 | ☐ |
| 클라이언트 마운트 | `mount -t nfs 192.168.64.10:/srv/nfs/data /mnt/nfs` | `findmnt` → `vers=4.2` | ☐ |
| root_squash 검증 | `touch /mnt/nfs/rootfile` | 소유자 `nobody`, 루프백 경유는 `root` | ☐ |
| fstab `_netdev` | `/etc/fstab` 추가 → `mount -a` | `findmnt /mnt/nfs` | ☐ |
| autofs 자동 마운트 | `/etc/auto.master` + `/etc/auto.nfs` → `enable --now autofs` | 접근 시 마운트, 타임아웃 후 해제 | ☐ |
| samba 설치·smb.conf | `[global]` + `[share]` 작성 | `testparm` → `Loaded services file OK.` | ☐ |
| Samba 계정 | `smbpasswd -a dev1`, `-a dev2` | `pdbedit -L` | ☐ |
| SELinux samba_share_t | `semanage fcontext -a -t samba_share_t "/srv/share(/.*)?"` + `restorecon` | `ls -ldZ /srv/share` | ☐ |
| smb·nmb 기동·방화벽 | `enable --now smb nmb`, `--add-service=samba` | `ss -tlnp \| grep smbd` (139·445) | ☐ |
| smbclient 검증 | `smbclient -L //127.0.0.1 -U dev1`, `-c 'put …'` | 공유 목록·업로드 성공 | ☐ |
| cifs 마운트·SetGID | `mount -t cifs … -o username=dev1,uid=dev1,gid=devteam` | `/srv/share` 파일 그룹 `devteam`, 디렉터리 `drwxrwsr-x` | ☐ |
| credentials + fstab | `/root/.smbcred` 600 + fstab `credentials=` | `mount -a`, `findmnt /mnt/smb` | ☐ |
| hosts allow 검증 | 루프백 제외 → 재시도 → 원복 | `NT_STATUS_CONNECTION_REFUSED` | ☐ |
| smbstatus·nmblookup | `smbstatus -b`, `nmblookup -A 192.168.64.10` | 세션·NetBIOS 이름 | ☐ |
| vsftpd 설치·설정 | `chroot_local_user`, `pasv_min/max_port` 추가 | `grep -E` 확인 | ☐ |
| vsftpd 기동·방화벽·SELinux | `--add-service=ftp`, `--add-port=40000-40100/tcp`, `setsebool -P ftpd_full_access on` | `ss -tlnp \| grep :21` | ☐ |
| FTP 접속·전송 | `ftp 127.0.0.1` 또는 `curl -T`/`lftp` | 홈에 업로드 파일 생성 | ☐ |
| ftpusers vs user_list | dev2 를 `ftpusers` 추가 → 거부, `userlist_deny=NO` 실습 | curl 종료 코드 67 | ☐ |
| chroot 동작 | `lftp -e 'cd ..; pwd'` | 여전히 `/` | ☐ |
| 익명 FTP 켬→검증→차단 | `anonymous_enable=YES` → `curl ftp://…/pub/` → `NO` 복귀 | 다운로드 성공 후 exit=67 | ☐ |
| xferlog 해석 | `tail /var/log/xferlog` | 8번째 `i/o`, 9번째 `r/a`, 끝 `c` | ☐ |
| postfix 설치·main.cf | `myhostname`·`mydestination`·`mynetworks` | `postconf -n`, `postfix check` | ☐ |
| postfix 기동·방화벽 | `enable --now postfix`, `--add-service=smtp` | `ss -tlnp \| grep master` | ☐ |
| aliases + newaliases | `ops`/`webmaster`/`devs` 추가 → `newaliases` | `/etc/aliases.db` 타임스탬프, `postalias -q devs` | ☐ |
| 별칭 배달 검증 | `mail -s "alias test" devs` | dev1·dev2 메일함 각각 도착 | ☐ |
| mail 대화식 수신 | `su - ops1 -c mail` (`h`·번호·`d`·`q`·`x`) | `/var/spool/mail/ops1` | ☐ |
| ~/.forward | dev2 홈에 `ops1` 기록 → 발송 | maillog `orig_to=<dev2…> to=<ops1…>` | ☐ |
| 메일 큐 관리 | `mailq`, `postqueue -f`, `postsuper -d ALL` ⚠️ | `Mail queue is empty` | ☐ |
| maillog 추적 | 큐 ID 로 grep | `pickup→cleanup→qmgr→local→removed`, `status=sent` | ☐ |
| telnet SMTP 수작업 | `telnet 127.0.0.1 25` (EHLO→MAIL FROM→RCPT TO→DATA→`.`→QUIT) | ops1 메일함 도착 | ☐ |
| access + postmap (참고) | `/etc/postfix/access` → `postmap` | `postmap -q` 조회 | ☐ |
| dovecot 설치·설정 | `protocols`, `mail_location`, `disable_plaintext_auth` | `doveconf -n` | ☐ |
| dovecot 기동·방화벽 | `enable --now dovecot`, `--add-service={imap,pop3}` | `ss -tlnp` 110·143·993·995 | ☐ |
| POP3 수작업 | `telnet 127.0.0.1 110` (USER/PASS/STAT/LIST/RETR/DELE/QUIT) | `+OK` 응답 | ☐ |
| IMAP 수작업 | `telnet 127.0.0.1 143` (`a login`/`a list`/`a select`/`a logout`) | `* OK`, `* LIST … INBOX` | ☐ |
| cups 설치·기동 | `dnf install -y cups cups-client`, `enable --now cups` | `lpstat -r` → running | ☐ |
| 더미 프린터 등록 | `lpadmin -p labprn -E -v socket://127.0.0.1:9100 -m raw`, `lpadmin -d labprn` | `lpstat -p -d -v` | ☐ |
| 인쇄 작업 제출 | `lp -d labprn -n 2 /etc/hosts`, `lpr -P labprn /etc/hostname` | `lpq -P labprn` 대기 목록 | ☐ |
| 큐 취소 | `lprm -P labprn -`, `cancel -a labprn` | `no entries` | ☐ |
| 활성·수락 제어 | `cupsdisable`/`cupsenable`, `cupsreject`/`cupsaccept` | `lpstat -p -a` 상태 변화 | ☐ |
| CUPS 웹 631 (SSH 터널) | macOS `ssh -N -L 6310:127.0.0.1:631 …` | 브라우저 `http://127.0.0.1:6310/` | ☐ |
| BSD↔System V 대응 | `lpr/lpq/lprm` ↔ `lp/lpstat/cancel`, `-P` ↔ `-d`, `-#` ↔ `-n` | 표 암기 | ☐ |
| DHCP 설정·문법만 (⚠️ 미기동) | `/etc/dhcp/dhcpd.conf` 작성 → `dhcpd -t -cf` | `systemctl is-active dhcpd` = inactive | ☐ |
| Squid 설정 형태 (※ 미실행) | `http_port 3128`, `acl`+`http_access` | 표·순서 이해 | ☐ |
| xinetd 부재 확인 | `dnf list xinetd`, `systemctl list-sockets` | 기본 저장소 없음 → 소켓 활성화 | ☐ |
| NIS·LDAP·Kerberos 개념 | `ypcat`/`ldapsearch`/`kinit` 표 | 포트 389·636·88 | ☐ |
| telnet 위험성 | `systemctl list-unit-files \| grep telnet` | 23 포트 미개방 | ☐ |
| 전 서비스 상태 일괄 | `systemctl is-active httpd named nfs-server smb nmb vsftpd postfix dovecot cups` | 전부 `active` | ☐ |
| 포트↔데몬 대조 | `ss -tulnp \| sort -k5` | 표와 일치 | ☐ |
| 방화벽 최종 확인 | `firewall-cmd --list-all` | services·ports 일치 | ☐ |
| 검증 스크립트 | `/usr/local/bin/check-part09.sh` | `FAIL=0` | ☐ |

---

## 기출 연결

| 기출 (문제집·회차·번호 또는 주제) | 이 파트의 단계 |
| --- | --- |
| 실기 r06-1 — `sed -i` 로 `Listen 80` → `Listen 8080` in-place 치환 | 2-4 |
| 실기 r06-6 — Apache 문법 검사 명령 2가지(`apachectl configtest` / `httpd -t`) | 1-3 표, 2-4, 2-17 |
| 실기 r06-11 — 가상호스트 블록 서술(`ServerName`·`DocumentRoot` 역할) | 2-7, 2-9 |
| 실기 r01-8 — `systemctl enable httpd` | 2-5, 10-1 |
| 실기 r06-7 — 존 수정 후 반드시 증가시킬 SOA 항목(Serial) | 3-4 표, 3-11 |
| 실기 r06-12 — SOA 의 Serial·Refresh·Expire 의미 + Serial 증가 이유 서술 | 3-4 SOA 필드표, 3-11 |
| 실기 r01-14 — Zone 파일의 MX 와 A 레코드 차이 서술 | 3-4 레코드표, 3-8 |
| 실기 r05-8 — `exportfs ______` (빈칸: `-ra`) | 4-3 |
| 실기 r04-14 — `root_squash` vs `no_root_squash` 차이와 보안 위험 서술 | 4-2 옵션표, 4-7 |
| 실기 r05-13 — `/etc/exports` 한 줄의 `rw`·`sync`·`no_root_squash` 의미 서술 | 4-2, 4-7 |
| 실기 r06-10 — vsftpd 익명 차단 `anonymous_enable=______` (NO) | 6-1, 6-8 |
| 실기 r06-8 — IMAP 기본 포트(143) | 1-1 표, 7-1 표, 7-17 |
| 실기 r02-14 · r05-12 — rsyslog `mail.info /var/log/maillog` 규칙 해석 | 7-9, 10-4 |
| 실기 r03-8 — 53번 UDP LISTEN 소켓을 프로세스와 함께 조회(`ss -ulnp`) | 3-7, 10-2 |
| 실기 r01-5 — 8080 LISTEN 프로세스 확인(`ss -tlnp`) | 2-5, 10-2 |
| 실기 r02-9 · r05-4 — `firewall-cmd --permanent --add-service=http` + `--reload` | 2-5, 10-3 |
| 실기 r03-14 · r06-14 — `hosts.allow`/`hosts.deny` 검사 순서 (RHEL 9 미지원 주의) | 5-11 |
| 실기 r03-13 — `/etc/fstab` 6개 필드 의미 (NFS·CIFS 의 `_netdev`·fsck 0) | 4-8, 5-10 |
| 필기 FULL r01-66/67, r05-66/69, r07-69, r10-67 — `DocumentRoot`·`ServerRoot`·`ServerName`·`DirectoryIndex` 구분 | 2-2 지시자표 |
| 필기 FULL r01-68, r02-69, r06-66, r10-68 — 이름 기반 가상호스트(1 IP 다수 도메인) | 2-7, 2-14 |
| 필기 FULL r01-69, r02-68, r05-67/68, r08-69 — `.htaccess` 와 `AllowOverride` | 2-10 |
| 필기 FULL r06-68 — `htpasswd -c` + `AuthType Basic`·`AuthUserFile`·`Require valid-user` | 2-10 |
| 필기 FULL r06-69, r09-67 — 디렉터리 목록 노출 차단 `Options -Indexes` | 2-11 |
| 필기 FULL r02-70, r04-66 — `apachectl configtest` vs `graceful` vs `fullstatus` | 1-3, 2-17 |
| 필기 FULL r02-67, r03-66, r08-68, r10-66 — MPM prefork/worker/event, `httpd -M` 의 `(shared)` | 2-3 |
| 필기 FULL r03-67, r06-70, r09-70/71 — mod_ssl, `SSLEngine on`, `SSLCertificateFile`/`KeyFile`, 80→443 리다이렉트 | 2-13 |
| 필기 FULL r03-68, r08-67 — access_log combined 필드 해석 | 2-12 |
| 필기 FULL r06-58 — `/srv/www` 이전 후 403 → `semanage fcontext` + `restorecon` | 2-8 |
| 필기 FULL r03-57, r09-59 — `setsebool -P httpd_can_network_connect on` 의 `-P` | 2-8, 5-6, 6-3 |
| 필기 FULL r09-57 — SELinux 컨텍스트 3번째 필드(타입) | 2-8 |
| 필기 FULL r03-69 — Nginx vs Apache (`.htaccess` 미지원, 이벤트 기반) | 2-16 |
| 필기 FULL r09-66 — Apache 서버 정보 노출 최소화(`ServerTokens`·`ServerSignature`) | 2-2 표, 2-13 |
| 필기 FULL r06-67, r09-68 — `Require ip 192.168.1.0/24` 등 2.4 접근 제어 문법 | 2-2 2.2↔2.4 대응표 |
| 필기 FULL r06-39, r07-33 — `rpm -qf` vs `rpm -ql` | 2-1 |
| 필기 FULL r01-72, r02-73, r04-72, r05-70/71 — `named.conf` 의 `directory`·`allow-query`·`forwarders`·`type master` | 3-2, 3-3 |
| 필기 FULL r02-71, r04-71, r10-76 — MX 우선순위(숫자 작을수록 우선) | 3-4 |
| 필기 FULL r01-74, r02-72, r04-70, r05-73, r06-71 — A·CNAME·MX·PTR 레코드 구분 | 3-4 레코드표 |
| 필기 FULL r10-73 — CNAME 은 apex 불가·타 레코드와 공존 불가 | 3-4 |
| 필기 FULL r10-74 — 역방향 존 이름 `10.168.192.in-addr.arpa` | 3-3, 3-5 |
| 필기 FULL r05-72, r08-73 — SOA 의 serial 용도 | 3-4, 3-11 |
| 필기 FULL r03-71, r07-73 — NOTIFY → SOA serial 비교 → AXFR/IXFR | 3-11, 3-14 |
| 필기 FULL r08-74, r09-74 — `allow-transfer` 로 존 전송 제한 | 3-3, 3-9, 3-14 |
| 필기 FULL r04-69 — `rndc` 서브명령 중 틀린 것(`restart` 없음) | 3-12 |
| 필기 FULL r02-74, r04-90, r10-75 — `dig @서버 도메인 mx`, `-x` 역방향, `+short`, `+trace` | 3-8, 3-9 |
| 필기 FULL r08-71 — `nslookup` 출력 해석 | 3-9 |
| 필기 FULL r01-71, r06-16/17, r07-11 — `/etc/resolv.conf`·이름 해석 순서 | 1-2, 3-10 |
| 필기 FULL r05-43 — `/etc/nsswitch.conf` 의 `hosts: files dns` | 1-2 |
| 필기 FULL r04-72, r06-72 — 슬레이브 존의 `masters { IP; }` + `slaves/` 경로 | 3-14 |
| 필기 FULL r01-79, r02-79, r03-77, r04-78, r05-78/79, r09-83, r10-79 — `/etc/exports` 옵션 해석 | 4-2, 4-7 |
| 필기 FULL r01-80, r02-80, r04-77, r05-80, r06-77, r08-77 — `showmount -e <서버>` | 4-5 |
| 필기 FULL r02-80, r04-76 — `exportfs -r` 로 재반영 | 4-3 |
| 필기 FULL r08-78 — `exportfs -v` 출력 해석(`ro`·`root_squash`·`no_subtree_check`) | 4-3 |
| 필기 FULL r03-76 — NFSv4 단일 TCP 2049, rpcbind 의존 감소 | 4-4, 4-10 |
| 필기 FULL r06-76 — `/data 192.168.20.0/24(rw,sync,root_squash)` 형태 선택 | 4-2 |
| 필기 FULL r07-78 — NFS 구축·사용 절차 순서 | 4-2 ~ 4-6 |
| 필기 FULL r07-79 — `root_squash` 정의 | 4-7 |
| 필기 FULL r10-80 — autofs (요청 시 마운트, 로컬 장치도 가능) | 4-9 |
| 필기 FULL r01-81/82, r02-81, r03-79, r04-81, r05-82, r08-80, r09-84, r10-82 — `smb.conf` 공유 설정·`valid users` 해석 | 5-3 |
| 필기 FULL r02-82, r04-80, r05-83 — `smb.conf` 문법 검사 `testparm` | 5-4 |
| 필기 FULL r03-78, r06-78, r07-80 — `smbpasswd -a`, `pdbedit -L`, 구축 절차 순서 | 5-5, 5-8 |
| 필기 FULL r01-82, r07-81, r10-81 — `smbd`(파일·인쇄·인증) vs `nmbd`(NetBIOS 이름·브라우징) | 5-1, 5-7 |
| 필기 FULL r04-79 — `smbclient -L //192.168.1.20` 로 공유 목록 | 5-8 |
| 필기 FULL r05-81 — `security = user` 의미 | 5-2 |
| 필기 FULL r06-79 — 그룹 쓰기 허용(`writable = yes` / `write list = @dev`) | 5-3 |
| 필기 FULL r01-83, r02-83, r04-82, r05-84, r07-83, r09-82 — `vsftpd.conf` 조합 해석(`anonymous_enable`·`local_enable`·`chroot_local_user`) | 6-1, 6-2, 6-7 |
| 필기 FULL r06-80, r08-82 — 익명 다운로드 유지·업로드만 금지(`anon_upload_enable=NO`) | 6-8 |
| 필기 FULL r06-81 — 로그인 금지 계정 목록 파일 `/etc/vsftpd/ftpusers` | 6-6 |
| 필기 FULL r03-80, r04-83, r05-85, r07-82, r10-83 — 능동/수동 모드, 20번 포트, `pasv_min/max_port` | 6-3, 6-10 |
| 필기 FULL r09-81 — SFTP(SSH 22) vs FTPS(FTP+TLS) | 6-11 |
| 필기 FULL r06-99, r10-85 — telnet·FTP 평문 → SSH·SFTP 대체 | 6-11, 9-5 |
| 필기 FULL r01-75, r02-90, r04-74, r05-77, r07-76, r10-20/77 — SMTP 25·POP3 110·IMAP 143 포트 매칭 | 1-1, 7-1, 7-17 |
| 필기 FULL r03-75, r05-96, r09-78/79 — SMTPS 465·IMAPS 993·POP3S 995, STARTTLS(587) | 7-1 포트표 |
| 필기 FULL r02-75, r03-89, r04-75, r07-75, r10-78 — MUA·MTA·MDA 역할과 메일 흐름 순서 | 7-1 |
| 필기 FULL r01-77, r07-77 — Postfix 주 설정 `/etc/postfix/main.cf` | 7-2, 7-12 |
| 필기 FULL r05-74, r08-76 — `mydestination` 의미 | 7-2 |
| 필기 FULL r02-77, r09-77 — `mynetworks` 와 오픈 릴레이 차단 | 7-2, 7-11 |
| 필기 FULL r02-76, r03-90, r04-73, r05-75, r06-73 — `/etc/aliases` 수정 후 `newaliases` | 7-5, 7-6 |
| 필기 FULL r06-74 — 사용자 본인이 전달 설정 → `~/.forward` | 7-7 |
| 필기 FULL r08-75 — `mailq` 출력 해석 | 7-8 |
| 필기 FULL r01-76 — sendmail 릴레이 정책 파일 `/etc/mail/access` | 7-13 |
| 필기 FULL r01-78, r02-78, r05-76 — dovecot 은 POP3·IMAP 제공, `protocols = imap pop3` | 7-14, 7-15 |
| 필기 FULL r03-63 — `/var/log/maillog` 등 로그 파일 매칭 | 7-9, 10-4 |
| 필기 FULL r09-80 — SMTP AUTH | 7-1, 7-10 |
| 필기 FULL r01-53, r04-49, r05-54, r06-48, r07-52, r10-53 — CUPS IPP·631 포트·설정 파일 | 8-1, 8-7, 8-8 |
| 필기 FULL r02-54, r04-51 — `lpr`(인쇄) `lpq`(큐) `lprm`(삭제) `lpstat -p`(상태) 역할 | 8-4, 8-5 |
| 필기 FULL r04-50, r05-55, r06-49 — `lp -n 3` / `lpr -# 3`, `lp -d` / `lpr -P` | 8-4, 8-9 |
| 필기 FULL r10-55 — BSD(`lpr lpq lprm`) vs System V(`lp lpstat cancel`) 계열 구분 | 8-9 |
| 필기 FULL r01-84, r02-84, r05-86/87, r06-82, r08-87 — `dhcpd.conf` 의 `range`·`option routers`·`host{hardware ethernet; fixed-address;}`, 67/68 포트 | 9-1 |
| 필기 FULL r03-81, r07-84, r10-84 — DORA 절차, DHCP 릴레이 에이전트 | 9-1 |
| 필기 FULL r04-16 — `/var/lib/dhcpd/dhcpd.leases` 임대 기록 | 9-1 |
| 필기 FULL r01-88, r02-85, r03-82, r05-90, r06-85, r07-90, r10-86 — Squid 3128, `acl`+`http_access` | 9-2 |
| 필기 FULL r01-53, r09-44, r10-14 — xinetd `disable = no`·`only_from`, standalone vs 슈퍼데몬 | 9-3 |
| 필기 FULL r10-6 — systemd 소켓 활성화 | 9-3 |
| 필기 FULL r01-87, r03-83, r07-85, r08-90, r09-87 — LDAP 389/636, `dn`·`dc`·`ou`·`cn`, `ldapsearch` | 9-4 |
| 필기 FULL r07-94 — Kerberos 인증 흐름(AS → TGT → TGS → 서비스 티켓) | 9-4 |
| 필기 FULL r03-17 — LDAP TCP 389 포트 매칭 | 9-4 |
| 필기 FULL r06-37, r08-44, r10-36 — `systemctl enable --now` vs `enable` 만 | 2-5, 10-1 |
| 필기 FULL r09-16 — httpd 가 root 로 기동 후 `apache` 계정으로 권한 하향 | 2-2 (`User`/`Group`) |
| 주제 — 서비스 구축 8단계(설치→설정→문법검사→기동→방화벽→SELinux→접속검증→로그) 서술 | 1-3, 10-5 |

---

## 이전 / 다음

[[08-network-config]] ← · → [[10-security-firewall-selinux]]

[[README]]
