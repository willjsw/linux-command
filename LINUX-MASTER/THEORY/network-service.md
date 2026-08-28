---
title: 네트워크 서비스
type: exam-theory
category: LINUX-MASTER
tags:
  - exam/linux-master
  - exam/theory
  - linux/network
  - linux/service
  - topic/dns
  - topic/web-server
  - topic/mail
  - task/configure
related: ["[[system-security]]", "[[network-security]]", "[[systemctl]]", "[[firewall-cmd]]", "[[docker]]", "[[ss]]"]
updated: 2026-08-28
---

# 네트워크 서비스

- 리눅스마스터 1급 필기 대비 이론 — 웹·인증·파일·메일·DNS·가상화 서비스
- **서비스명 ↔ 데몬명 ↔ 설정 파일 경로 ↔ 기본 포트** 4개 축 매칭이 최빈출

---

## 0. 서비스 총괄표 (최우선 암기)

| 서비스 | 데몬 | 주 설정 파일 | 기본 포트 |
| --- | --- | --- | --- |
| 웹 (Apache) | `httpd` | `/etc/httpd/conf/httpd.conf` | 80, 443 |
| 웹 (Nginx) | `nginx` | `/etc/nginx/nginx.conf` | 80, 443 |
| DNS (BIND) | `named` | `/etc/named.conf` | 53 (UDP/TCP) |
| DHCP | `dhcpd` | `/etc/dhcp/dhcpd.conf` | 67/68 (UDP) |
| 메일 전송 | `sendmail` / `postfix` | `/etc/mail/sendmail.cf` / `/etc/postfix/main.cf` | 25 |
| 메일 수신 | `dovecot` | `/etc/dovecot/dovecot.conf` | POP3 110, IMAP 143 |
| FTP | `vsftpd` | `/etc/vsftpd/vsftpd.conf` | 21(제어), 20(데이터) |
| 파일 공유 (Windows) | `smbd`, `nmbd` | `/etc/samba/smb.conf` | 139, 445 |
| 파일 공유 (Unix) | NFS (`nfsd`) | `/etc/exports` | 2049 |
| 인증 (NIS) | `ypserv` / `ypbind` | `/etc/sysconfig/network` 등 | RPC 기반 |
| 프록시 | `squid` | `/etc/squid/squid.conf` | 3128 |
| 원격 접속 | `sshd` | `/etc/ssh/sshd_config` | 22 |

- 서비스 기동·확인은 [[systemctl]] · [[ss]], 포트 개방은 [[firewall-cmd]]

---

## 1. 웹 서비스 — Apache (httpd)

### 1-1. httpd.conf 주요 지시자

| 지시자 | 의미 |
| --- | --- |
| `ServerRoot` | 서버 설정·로그 기준 디렉터리 (`/etc/httpd`) |
| `DocumentRoot` | 웹 문서 루트 (`/var/www/html`) |
| `Listen` | 수신 포트 (`Listen 80`) |
| `ServerName` | 서버 도메인명 |
| `DirectoryIndex` | 기본 문서 (`index.html index.php`) |
| `KeepAlive` | 연결 유지 여부 (`On`/`Off`) |
| `MaxKeepAliveRequests` | 연결당 최대 요청 수 |
| `UserDir` | 사용자 홈 웹 디렉터리 (`public_html`) |
| `ErrorLog` / `CustomLog` | 오류·접근 로그 경로 |
| `Include` | 추가 설정 포함 (`conf.d/*.conf`) |

### 1-2. 가상 호스트

```apache
<VirtualHost *:80>
    ServerName www.example.com
    DocumentRoot /var/www/example
    ErrorLog logs/example-error_log
</VirtualHost>
```

- 이름 기반(1 IP 다수 도메인) / IP 기반(도메인별 IP) 구분 출제
- 설정 문법 검사: `httpd -t` / `apachectl configtest`
- MPM 방식: `prefork` (프로세스, 안정) / `worker` (스레드 혼합) / `event` (비동기, 기본)

### 1-3. 접근 제어·인증

```apache
<Directory "/var/www/secure">
    AuthType Basic
    AuthName "Restricted"
    AuthUserFile /etc/httpd/.htpasswd
    Require valid-user
</Directory>
```

- 계정 파일 생성: `htpasswd -c /etc/httpd/.htpasswd user1` (`-c` 최초 1회만, **c**reate)
- `.htaccess` 허용: `AllowOverride All` (기본 `None`)
- `Require all granted` / `Require ip 192.168.0.0/24` — 2.4 문법 (구 2.2 는 `Order allow,deny`)

---

## 2. DNS — BIND (named)

### 2-1. /etc/named.conf 골격

```
options {
    listen-on port 53 { any; };
    directory "/var/named";        // zone 파일 위치
    allow-query { any; };
    recursion yes;
    forwarders { 8.8.8.8; };       // 재귀 위임
};
zone "example.com" IN {
    type master;                    // master | slave | hint
    file "example.com.zone";
};
zone "0.168.192.in-addr.arpa" IN {  // 역방향 존
    type master;
    file "192.168.0.rev";
};
```

- `type hint` : 루트 힌트 존 (`.`), `type slave` 는 `masters { IP; };` 필수

### 2-2. Zone 파일 레코드 (최빈출)

```
$TTL 86400
@   IN  SOA  ns1.example.com. admin.example.com. (
            2026082801 ; Serial (증가 필수 — 슬레이브 전송 트리거)
            3600       ; Refresh
            1800       ; Retry
            604800     ; Expire
            86400 )    ; Minimum TTL
    IN  NS   ns1.example.com.
    IN  MX   10 mail.example.com.
ns1 IN  A    192.168.0.10
www IN  A    192.168.0.20
ftp IN  CNAME www
```

| 레코드 | 의미 |
| --- | --- |
| `SOA` | 존 권한 시작 (Serial·Refresh·Retry·Expire·TTL) |
| `NS` | 네임서버 지정 |
| `A` / `AAAA` | 호스트 → IPv4 / IPv6 |
| `CNAME` | 별칭 |
| `MX` | 메일 서버 (숫자 = 우선순위, **낮을수록 우선**) |
| `PTR` | 역방향 (IP → 도메인) |

- FQDN 끝의 `.` 누락 시 존 이름 자동 접미 — 함정 단골
- 문법 검사: `named-checkconf`, `named-checkzone example.com <존파일>`
- 조회 도구: `dig`, `nslookup`, `host` / 클라이언트 설정: `/etc/resolv.conf` (`nameserver`), `/etc/hosts`, 순서 결정은 `/etc/nsswitch.conf` (`hosts: files dns`)

---

## 3. 메일 서비스

- 구성 요소: **MTA** (전송 — sendmail·postfix), **MDA** (배달 — procmail), **MUA** (사용자 — mutt·Outlook)
- 프로토콜: SMTP 25 (전송), POP3 110 (수신 후 삭제), IMAP 143 (서버 보관·동기화)

### 3-1. sendmail

- 설정: `/etc/mail/sendmail.cf` (m4 매크로 `sendmail.mc` 에서 생성)
- `/etc/mail/access` : 릴레이 허용·거부 (`RELAY`, `REJECT`, `DISCARD`) → `makemap hash access.db < access` 로 DB 생성
- `/etc/mail/local-host-names` : 수신 대상 도메인 목록
- `/etc/mail/virtusertable` : 가상 사용자 → 실제 계정 매핑
- `/etc/aliases` : 별칭 → 계정 매핑, 수정 후 `newaliases` 필수
- `~/.forward` : 개인 전달 설정

### 3-2. postfix / dovecot

- postfix 설정: `/etc/postfix/main.cf` — `myhostname`, `mydomain`, `mynetworks` (릴레이 허용 대역), `inet_interfaces`
- dovecot: POP3/IMAP 제공, `/etc/dovecot/dovecot.conf` 의 `protocols = imap pop3`

---

## 4. 파일 공유 서비스

### 4-1. NFS

- 서버 설정: `/etc/exports`

```
/share  192.168.0.0/24(rw,sync,no_root_squash)
```

| 옵션 | 의미 |
| --- | --- |
| `rw` / `ro` | 읽기쓰기 / 읽기전용 |
| `sync` / `async` | 동기 / 비동기 기록 |
| `root_squash` | 클라이언트 root 를 nobody 로 매핑 (기본) |
| `no_root_squash` | root 권한 유지 ⚠ 보안 위험 |
| `all_squash` | 전 사용자 nobody 매핑 |

- 반영: `exportfs -ra` (**r**e-export **a**ll), 조회: `exportfs -v`, 원격 확인: `showmount -e <서버>`
- 마운트: `mount -t nfs 192.168.0.10:/share /mnt` → [[mount]]

### 4-2. Samba

- 설정: `/etc/samba/smb.conf` — 데몬 `smbd` (파일·인쇄, 445), `nmbd` (이름 해석, 137~139)

```ini
[global]
   workgroup = WORKGROUP
   security = user
[share]
   path = /srv/samba
   writable = yes
   valid users = user1
```

- 계정 등록: `smbpasswd -a user1` (**a**dd), 설정 검사: `testparm`
- 클라이언트: `smbclient //서버/share -U user1`, 마운트: `mount -t cifs`

### 4-3. FTP — vsftpd

- `/etc/vsftpd/vsftpd.conf` 주요 항목
	- `anonymous_enable=NO` : 익명 접속 차단
	- `local_enable=YES` : 로컬 계정 허용
	- `write_enable=YES` : 업로드 허용
	- `chroot_local_user=YES` : 홈 디렉터리 상위 이동 차단
- `/etc/vsftpd/ftpusers` : **접속 거부** 계정 목록 (PAM 기반)
- 능동(Active) 모드: 서버 20 포트가 데이터 연결 개시 / 수동(Passive): 클라이언트가 임의 포트로 연결 — 방화벽 환경은 Passive

---

## 5. 인증 서비스 — NIS / LDAP

### 5-1. NIS (Network Information Service)

- 계정 정보를 서버에서 중앙 관리 — RPC 기반 (`rpcbind` 필수)
- 서버 데몬: `ypserv`, `yppasswdd` / 클라이언트: `ypbind`
- 도메인 설정: `nisdomainname <도메인>` 또는 `/etc/sysconfig/network` 의 `NISDOMAIN`
- 클라이언트 명령
	- `ypwhich` : 바인딩된 NIS 서버 확인
	- `ypcat passwd` : NIS 맵 내용 조회
	- `yppasswd` : NIS 비밀번호 변경
	- `yptest` : 동작 종합 점검

### 5-2. LDAP

- 디렉터리 서비스 프로토콜 — 포트 389 (LDAPS 636), 데몬 `slapd`
- 항목 구성: `dn` (고유 식별), `dc` (도메인 구성요소), `ou` (조직 단위), `cn` (일반 이름)
- 조작 도구: `ldapadd`, `ldapsearch`, `ldapmodify`

---

## 6. 슈퍼데몬·프록시·DHCP

### 6-1. xinetd

- 요청 시에만 서비스 기동하는 슈퍼데몬 — `/etc/xinetd.conf`, `/etc/xinetd.d/<서비스>`
- 주요 속성: `disable = no` (활성), `only_from` (허용 IP), `no_access` (거부), `access_times` (허용 시간대), `instances` (최대 동시 수)
- TCP Wrapper 연동 → [[system-security]] 2-6

### 6-2. Squid (프록시)

- `/etc/squid/squid.conf` — `http_port 3128`, `acl` + `http_access` 조합

```
acl localnet src 192.168.0.0/24
http_access allow localnet
http_access deny all
```

### 6-3. DHCP

- `/etc/dhcp/dhcpd.conf`

```
subnet 192.168.0.0 netmask 255.255.255.0 {
    range 192.168.0.100 192.168.0.200;
    option routers 192.168.0.1;
    option domain-name-servers 8.8.8.8;
    default-lease-time 600;
}
```

- 임대 이력: `/var/lib/dhcpd/dhcpd.leases`
- 클라이언트 확인: DORA 절차 (Discover → Offer → Request → Ack)

---

## 7. 가상화·클러스터

### 7-1. 가상화 유형

| 유형 | 특징 | 예 |
| --- | --- | --- |
| 전가상화 (Full) | 게스트 OS 수정 불필요, 하드웨어 완전 에뮬레이션 | KVM, VMware |
| 반가상화 (Para) | 게스트 OS 커널 수정 필요, 성능 우위 | Xen (PV) |
| 컨테이너 | OS 커널 공유, 프로세스 격리 | Docker, LXC → [[docker]] |

- KVM: 커널 모듈 (`kvm.ko`) 기반 — CPU 가상화 지원 필수 (Intel VT-x `vmx` / AMD-V `svm`, `/proc/cpuinfo` 로 확인)
- 관리 도구: `libvirtd` 데몬 + `virsh` (CLI) / `virt-manager` (GUI)
	- `virsh list --all` (전체 목록), `virsh start|shutdown|destroy <VM>`, `virsh console <VM>`
- 이미지 도구: `qemu-img create -f qcow2 disk.img 20G`

### 7-2. 클러스터 유형 (개념)

| 유형 | 목적 |
| --- | --- |
| 고계산 (HPC) | 연산 성능 집적 (베어울프 클러스터) |
| 부하분산 (LVS) | 요청 분산 — Director + Real Server |
| 고가용 (HA) | 장애 시 대기 노드 전환 (Heartbeat·Pacemaker) |

- RAID 와 결합 문제 출제: RAID 0 (스트라이핑·성능), 1 (미러링), 5 (패리티 분산, n-1), 6 (이중 패리티, n-2), 10 (1+0) — 가용 용량 계산 최빈출

---

## 연관 문서
- [[system-security]] — TCP Wrapper·PAM·로그
- [[network-security]] — 서비스별 공격·방어
- [[systemctl]] — 서비스 기동·활성화
- [[firewall-cmd]] — 서비스 포트 개방
- [[ss]] — 리스닝 포트 확인
- [[docker]] — 컨테이너 실사용 문서
- [[mount]] — NFS·CIFS 마운트
