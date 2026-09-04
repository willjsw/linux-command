---
title: LAB 10 — 시스템·네트워크 보안(firewalld·iptables·SELinux·침해 점검·암호화)
type: exam-lab
part: 10
tags:
  - exam/linux-master
  - exam/lab
  - linux/security
  - linux/selinux
  - linux/network
  - task/configure
  - task/verify
related: ["[[README]]", "[[09-network-services]]", "[[11-container-virtualization]]", "[[../THEORY/network-security]]", "[[../THEORY/selinux-security]]", "[[../THEORY/system-security]]", "[[../../SECURITY-SELINUX/firewall-cmd]]", "[[../../SECURITY-SELINUX/iptables]]", "[[../../SECURITY-SELINUX/getenforce]]", "[[../../SECURITY-SELINUX/ausearch]]"]
distro: Rocky Linux 9 (aarch64, UTM)
updated: 2026-09-03
---

# LAB 10 — 시스템·네트워크 보안(firewalld·iptables·SELinux·침해 점검·암호화)

- 노출 파악(`ss`·`nmap`) → firewalld 존 설계(internal/public) → 같은 정책을 `iptables` 로 직접 작성·비교 → NAT → firewalld 복귀
- SELinux 위반을 일부러 만들어 `ausearch`/`sealert` 로 추적·`restorecon` 으로 수정, 포트 라벨·불린·`auditd` 감사 규칙
- sshd·PAM(faillock)·배너·미사용 계정 점검, SUID/무결성 기준선(`find`·`rpm -Va`·`sha256sum`·AIDE), sysctl 강화, 평문 프로토콜 스니핑 체감
- 해시·대칭·비대칭·서명·인증서(`gpg`·`openssl`) 와 사내 CA 신뢰 추가, 마지막에 `/usr/local/bin/seccheck.sh` 를 만들어 주간 cron 등록

> **이 파트의 시나리오**: Part 09 까지 `srv01.lab.local` 에 웹·DNS·NFS·Samba·FTP·메일·인쇄 서비스가 모두 올라갔고 firewalld 에는 서비스별 허용 규칙이 느슨하게 추가돼 있다. 운영팀(ops1)이 "외부 노출 전 보안 점검" 을 요구했다. 사내망(192.168.64.0/24)만 관리·파일공유 서비스에 접근하고 그 외에는 웹·DNS 만 열리도록 방화벽을 다시 설계하고, SELinux 를 Enforcing 으로 유지한 채 위반을 진단하는 절차를 익힌 뒤, 계정·파일 무결성·암호화 도구까지 한 번에 점검하는 스크립트를 남긴다.

- 전 과정 root(`#`). 일반 사용자 단계(`dev1`·`ops1` 의 gpg)는 `$` 표기
- **⚠️ 잠금 위험 파트** — 방화벽·sshd·PAM 변경 단계는 반드시 **UTM 콘솔** 에서 수행하고, macOS 터미널 `ssh -p 2222 admin1@192.168.64.10` 은 검증용으로만 사용. Part 10 진입 전 스냅샷 권장([[README]] 1절)
- 선행 자원: 계정·그룹(Part 03), `/srv/share`·`/srv/raid`(Part 05), cron(Part 06), 고정 IP·sshd 2222(Part 08), 서비스 전부(Part 09). `nmap`·`tcpdump`·`net-tools`·`policycoreutils-python-utils`·`setroubleshoot-server`·`epel-release` 는 Part 02 에서 설치됨 — 없으면 `dnf install -y <패키지>`

---

## 1. 현재 노출 파악

### 1-1. 리스닝 포트 전수 확인

> **상황**: 잠그기 전에 무엇이 열려 있는지부터 안다. Part 09 서비스 데몬이 어떤 주소·포트에서 대기 중인지 프로세스 이름과 함께 뽑는다.

```bash
ss -tulnp                                  # TCP/UDP 리스닝 소켓 + 프로세스
ss -tlnp | awk 'NR>1{print $4}' | sort -u  # 로컬 주소:포트만
```

- `-t` : TCP 소켓 (**t**cp)
- `-u` : UDP 소켓 (**u**dp)
- `-l` : 리스닝 상태만 (**l**istening)
- `-n` : 포트·주소를 숫자로 (**n**umeric)
- `-p` : 소켓을 소유한 프로세스 표시 (**p**rocess, root 필요)

**검증**

```bash
ss -tulnp | grep -cE 'LISTEN|UNCONN'
ss -tlnp | grep -E ':(2222|80|443|8080|53|2049|445|139|21|25|631|6379)\b'
```

```text
...
LISTEN 0 128 0.0.0.0:2222   0.0.0.0:* users:(("sshd",pid=...,fd=3))
LISTEN 0 511 *:80           *:*       users:(("httpd",pid=...,fd=4))
LISTEN 0 511 *:443          *:*       users:(("httpd",pid=...,fd=6))
LISTEN 0 10  192.168.64.10:53 0.0.0.0:* users:(("named",pid=...,fd=...))
LISTEN 0 64  0.0.0.0:2049   0.0.0.0:*
LISTEN 0 50  0.0.0.0:445    0.0.0.0:* users:(("smbd",pid=...,fd=...))
LISTEN 0 32  0.0.0.0:21     0.0.0.0:* users:(("vsftpd",pid=...,fd=3))
LISTEN 0 100 127.0.0.1:25   0.0.0.0:* users:(("master",pid=...,fd=13))
LISTEN 0 4096 127.0.0.1:631 0.0.0.0:* users:(("cupsd",pid=...,fd=...))
...
```

> 📝 **시험 포인트**: `ss -tulnp` 옵션 조합(`netstat -tulnp` 와 동일 의미)은 실기 단골. 리스닝 존재 ≠ 외부 접근 가능 — 방화벽은 별개 확인(1-3).

### 1-2. 자기 자신 포트 스캔 (nmap)

> **상황**: 공격자 시점의 정보를 확인한다. 전체 65535 포트 TCP+UDP 스캔은 UDP 때문에 수십 분이 걸리므로 범위를 한정한다.

```bash
nmap -sT -p 1-1024,2222,6379 192.168.64.10          # TCP connect 스캔 (비특권도 가능)
nmap -sS -p 1-1024,2222 192.168.64.10               # SYN(하프 오픈) 스캔 — root
nmap -sU -p 53,111,123,137,138,2049 192.168.64.10   # UDP 는 포트를 콕 찍어서
nmap -sV -p 2222,80,21 192.168.64.10                # 서비스 버전 배너
```

- `-sT` : TCP connect 스캔 — 3-way 완성 (**s**can **T**CP connect), 로그에 남음
- `-sS` : SYN 스캔 — SYN 만 보내고 SYN/ACK 확인 후 RST (**S**YN, 하프 오픈·스텔스). root 필요
- `-sU` : UDP 스캔 (**U**DP) — 응답 없음=open|filtered 로 판정이 느림
- `-sV` : 서비스·버전 탐지 (**V**ersion)
- `-p` : 포트 목록·범위 (**p**ort). `-p-` = 전체 1-65535
- 참고: `-sF`(FIN) · `-sN`(NULL) · `-sX`(XMAS) 스텔스 스캔, `-O` OS 추정, `-Pn` ping 생략

**검증**

```bash
nmap -sT -p 1-1024,2222,6379 192.168.64.10 | grep -E '^[0-9]+/(tcp|udp)'
```

```text
21/tcp   open  ftp
53/tcp   open  domain
80/tcp   open  http
111/tcp  open  rpcbind
139/tcp  open  netbios-ssn
443/tcp  open  https
445/tcp  open  microsoft-ds
2222/tcp open  EtherNetIP-1
...
```

> 📝 **시험 포인트**: `-sS` = 하프 오픈(스텔스) 스캔 — R01-99·R04-94·R07-98·R08-99 반복 출제. `-sT` 는 연결 완성으로 서버 로그에 기록. 자기 IP 스캔 결과는 **방화벽 통과 뒤** 의 모습(루프백이 아니라 실제 NIC 주소를 찍어야 존 규칙이 적용됨).

### 1-3. firewalld 상태·존 구조 조회

> **상황**: Part 09 에서 서비스마다 `--add-service` 만 반복했다. 어떤 존이 활성이고 무엇이 허용돼 있는지 전체 그림을 본다.

```bash
firewall-cmd --state                 # running / not running
firewall-cmd --get-default-zone      # 기본 존 (인터페이스 미지정 시 적용)
firewall-cmd --get-active-zones      # 인터페이스·소스가 붙은 존만
firewall-cmd --get-zones             # 정의된 존 9개
firewall-cmd --list-all              # 기본 존 전체 설정
firewall-cmd --list-all-zones | less # 모든 존 설정 (길다)
```

- `--state` : firewalld 데몬 동작 여부
- `--get-default-zone` / `--set-default-zone=<존>` : 기본 존 조회/변경
- `--get-active-zones` : 인터페이스 또는 소스가 바인딩된 존과 그 목록
- `--get-zones` : 존 이름 전체
- `--list-all` : 대상 존의 target·interfaces·sources·services·ports·protocols·masquerade·forward-ports·icmp-blocks·rich rules 일괄 출력. `--zone=<존>` 미지정 시 기본 존
- `--list-all-zones` : 존별 `--list-all` 을 연속 출력

**검증**

```bash
firewall-cmd --list-all | sed -n '1,8p'
```

```text
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s1
  sources:
  services: cockpit dhcpv6-client dns ftp http https mountd nfs rpc-bind samba ssh ...
  ports: 2222/tcp 8080/tcp
  ...
```

- 존 9개 성격 (신뢰도 낮음 → 높음)

| 존 | 기본 동작 | 용도 |
| --- | --- | --- |
| `drop` | 들어오는 패킷 전부 **무응답 폐기**, 나가는 것만 허용 | 완전 차단 |
| `block` | 들어오는 패킷 **거부 응답**(icmp-host-prohibited) | 차단 + 거부 통지 |
| `public` | 허용 목록만 통과 (기본값 `ssh dhcpv6-client cockpit`) | 공공망·**기본 존** |
| `external` | public + **masquerade 기본 on** | 라우터 외부 인터페이스 |
| `dmz` | 제한 공개 (`ssh` 만 기본) | 공개 서버 격리망 |
| `work` | `ssh dhcpv6-client cockpit` | 업무망 |
| `home` | work + `mdns samba-client` | 가정망 |
| `internal` | home 과 동일 시작점, **사내망 소스 지정용** | 내부망 — 이 파트 주 대상 |
| `trusted` | **모두 허용**(target ACCEPT) | 완전 신뢰 |

> 📝 **시험 포인트**: `--list-all` 출력의 `(active)` 표시·`interfaces`·`services`·`ports` 해석(R08-94). `drop`(무응답) vs `block`(거부 응답) 차이는 iptables `DROP` vs `REJECT` 와 같은 구도. 기본 존은 `public`.

### 1-4. 서비스 정의 파일 — 이름 뒤에 숨은 포트

> **상황**: `samba` 서비스 하나가 실제로 몇 개 포트를 여는지 알아야 iptables 로 옮겨 쓸 수 있다. 사전 정의 서비스는 xml 파일이고, 수정본은 `/etc/firewalld/` 에 놓인다.

```bash
firewall-cmd --get-services | tr ' ' '\n' | wc -l       # 사전 정의 서비스 수
firewall-cmd --info-service=samba
firewall-cmd --info-service=nfs
firewall-cmd --info-service=ssh                          # 22/tcp 고정 → 2222 는 별도 처리 필요
ls /usr/lib/firewalld/services/ | head; ls /usr/lib/firewalld/services | wc -l
cat /usr/lib/firewalld/services/samba.xml
ls /etc/firewalld/services/ /etc/firewalld/zones/        # 사용자 수정본 위치
```

- `--get-services` : 사용 가능한 서비스 이름 전체
- `--info-service=<이름>` : 서비스에 포함된 ports·protocols·modules(conntrack helper)·destination
- `/usr/lib/firewalld/{services,zones,icmptypes}/` : 패키지 제공 기본 정의 (수정 금지)
- `/etc/firewalld/{services,zones}/` : 관리자 변경분 — 같은 이름이 있으면 `/etc` 가 우선. `--permanent` 변경은 여기 xml 로 저장

**검증**

```bash
firewall-cmd --info-service=samba | grep ports
diff <(ls /usr/lib/firewalld/zones) <(ls /etc/firewalld/zones)
```

```text
  ports: 137/udp 138/udp 139/tcp 445/tcp
...
> public.xml
```

> 📝 **시험 포인트**: `ssh` 서비스 = **22/tcp 고정** — 포트를 2222 로 바꿨으면 `--add-port=2222/tcp` 또는 커스텀 서비스(2-8) 필요. `/etc/firewalld/zones/public.xml` 이 존재 = public 존이 한 번이라도 `--permanent` 변경됨.

### 1-5. firewalld 전역 설정 파일

> **상황**: 기본 존·백엔드·로그 정책이 어디에 저장되는지 확인한다. RHEL 9 는 nftables 백엔드다.

```bash
grep -vE '^\s*(#|$)' /etc/firewalld/firewalld.conf
firewall-cmd --get-log-denied
```

- `DefaultZone=public` : `--set-default-zone` 결과가 기록되는 키
- `FirewallBackend=nftables` : 규칙을 nft 로 생성 (RHEL 8 부터). `iptables` 값은 레거시
- `LogDenied=off` : 거부 패킷 커널 로그 (`--set-log-denied` 로 변경, 2-9)
- `CleanupOnExit=yes` : 데몬 종료 시 규칙 제거
- `AllowZoneDrifting=no` : 존 간 소스/인터페이스 중복 허용 안 함 (보안 기본값)
- `IndividualCalls=no`, `IPv6_rpfilter=yes`, `RFC3964_IPv4=yes` : 참고

**검증**

```bash
grep -E '^(DefaultZone|FirewallBackend|LogDenied)' /etc/firewalld/firewalld.conf
```

```text
DefaultZone=public
LogDenied=off
FirewallBackend=nftables
```

> 📝 **시험 포인트**: firewalld → nftables(RHEL 8+) / iptables(RHEL 7) 백엔드 관계, "firewalld 는 iptables/nftables 의 상위 관리 도구" (R10-96).

---

## 2. firewalld 존 설계

### 2-1. 런타임 vs 영구 — 세 가지 반영 방식

> **상황**: Part 09 에서 `--permanent` 를 붙였는데 바로 안 열려 당황했던 경험을 정리한다. 임시 포트 8081 로 세 방식을 모두 실습한다.

```bash
# ① 런타임만 (즉시 적용, 재부팅·reload 시 소멸)
firewall-cmd --add-port=8081/tcp
firewall-cmd --list-ports
firewall-cmd --permanent --list-ports          # 영구엔 없음

# ② 런타임을 영구로 복사
firewall-cmd --runtime-to-permanent
firewall-cmd --permanent --list-ports          # 이제 있음

# ③ 영구만 (파일에만 저장 → reload 필요)
firewall-cmd --permanent --remove-port=8081/tcp
firewall-cmd --list-ports                      # 런타임엔 여전히 있음
firewall-cmd --reload                          # 영구 → 런타임 재적용
firewall-cmd --list-ports                      # 사라짐
```

- `--add-port=<포트>/<프로토콜>` / `--remove-port=…` : 포트 허용 추가/제거
- `--permanent` : `/etc/firewalld/zones/<존>.xml` 에 저장만. 런타임 미반영
- `--reload` : 영구 설정을 다시 읽어 런타임 교체. **런타임 전용 변경은 이때 사라짐**, 기존 연결 상태(conntrack)는 유지
- `--runtime-to-permanent` : 현재 런타임 전체를 영구로 저장
- `--complete-reload` : 커널 넷필터 모듈까지 재적용 — 기존 연결 끊김 (⚠️ SSH 세션 단절 가능, 2-9)
- `--list-ports` / `--list-services` : 포트/서비스만 출력

**검증**

```bash
firewall-cmd --query-port=8081/tcp; firewall-cmd --permanent --query-port=8081/tcp
grep 8081 /etc/firewalld/zones/public.xml || echo "no 8081 in xml"
```

```text
no
no
no 8081 in xml
```

> 📝 **시험 포인트**: 실기 R02-9·R05-4 "http 영구 허용 후 즉시 반영" = `firewall-cmd --permanent --add-service=http` → `firewall-cmd --reload` 2개 명령. `--permanent` 만 치면 "반영 안 됨" 이 정답 함정. `--query-*` 는 yes/no 와 종료코드 0/1.

### 2-2. internal 존 — 사내망 소스 바인딩과 허용 서비스

> **상황**: 사내망 192.168.64.0/24(macOS 호스트 192.168.64.1 포함) 에서 오는 패킷은 `internal` 존으로 분류하고, 여기에만 관리(ssh 2222)·파일공유(samba·nfs)·웹·DNS 를 허용한다. 소스 기반 존은 인터페이스 존보다 우선 평가된다.

⚠️ 이 단계부터 2-3 까지 UTM 콘솔에서 수행 (macOS ssh 세션은 유지되지만 실수 시 복구 경로 확보)

```bash
firewall-cmd --permanent --zone=internal --add-source=192.168.64.0/24
firewall-cmd --permanent --zone=internal --add-port=2222/tcp
firewall-cmd --permanent --zone=internal --add-service={http,https,dns,samba,nfs,rpc-bind,mountd,ftp}
firewall-cmd --permanent --zone=internal --remove-service={ssh,mdns,samba-client,dhcpv6-client,cockpit}
firewall-cmd --reload
```

- `--zone=<존>` : 대상 존 (미지정 시 기본 존)
- `--add-source=<CIDR>` : 출발지 주소 대역을 이 존에 바인딩 — 인터페이스 바인딩보다 **우선 매칭**. `--remove-source`, `--change-source`, `--list-sources`
- `--add-service={a,b,c}` : 셸 중괄호 확장으로 다중 서비스. `nfs` 는 2049 만 → v3 호환용 `rpc-bind`(111)·`mountd`(20048) 함께
- `--remove-service=ssh` : 22 는 더 이상 안 쓰므로 제거(sshd 는 2222 만 리슨). `mdns`·`samba-client` 는 클라이언트용 기본값 제거

**검증**

```bash
firewall-cmd --get-active-zones
firewall-cmd --list-all --zone=internal
# macOS 터미널
# ssh -p 2222 admin1@192.168.64.10 'echo ok'
```

```text
internal
  sources: 192.168.64.0/24
public
  interfaces: enp0s1
internal (active)
  target: default
  icmp-block-inversion: no
  interfaces:
  sources: 192.168.64.0/24
  services: dns ftp http https mountd nfs rpc-bind samba
  ports: 2222/tcp
  protocols:
  forward: yes
  masquerade: no
  ...
ok
```

> 📝 **시험 포인트**: 존 결정 우선순위 **소스 > 인터페이스 > 기본 존**. `--add-source` 와 `--change-interface` 의 구분. `(active)` 는 소스·인터페이스 중 하나라도 바인딩된 존.

### 2-3. public 존 축소 — 웹·DNS 만

> **상황**: 사내망 밖(인터페이스 enp0s1 로 들어오지만 소스가 192.168.64.0/24 가 아닌 트래픽)은 public 으로 떨어진다. 여기서는 웹·DNS 만 남기고 Part 09 가 열어 둔 나머지를 모두 걷어낸다.

⚠️ `ssh` 와 `2222/tcp` 를 public 에서 제거 — macOS 는 internal 존(2-2) 으로 들어오므로 안전하지만, `--get-active-zones` 로 internal 이 active 인지 먼저 확인

```bash
firewall-cmd --get-active-zones | grep -A1 internal
firewall-cmd --permanent --zone=public --remove-service={ssh,cockpit,dhcpv6-client,ftp,samba,nfs,rpc-bind,mountd,smtp,ipp}
firewall-cmd --permanent --zone=public --remove-port={2222/tcp,8080/tcp}
firewall-cmd --permanent --zone=public --add-service={http,https,dns}
firewall-cmd --reload
```

- `--remove-service={…}` : Part 09 잔여 허용 제거. 없는 항목은 `Warning: NOT_ENABLED` 만 출력하고 계속
- `--remove-port={…}` : 2222·8080 제거 (8080 은 사내망에서만 쓰기로 함 → internal 에 필요하면 2-2 방식으로 추가)

**검증**

```bash
firewall-cmd --list-all --zone=public
firewall-cmd --query-service=ssh --zone=public; echo "exit=$?"
firewall-cmd --query-service=http --zone=public
```

```text
public (active)
  target: default
  interfaces: enp0s1
  sources:
  services: dns http https
  ports:
  ...
no
exit=1
yes
```

> 📝 **시험 포인트**: `--query-service` 종료 코드(0=yes, 1=no) 는 스크립트 조건문에 사용. 기본 존 변경은 `--set-default-zone=internal` 처럼 가능하지만 이 설계에서는 public 유지 — "인터페이스를 내부 존에 붙이는" `--change-interface=enp0s1 --zone=internal` 을 쓰면 소스 구분이 무의미해짐(설명만).

### 2-4. Rich Rule — 조건 결합·로그·거부

> **상황**: 서비스 단위로는 표현 못 하는 "특정 대역만 ssh", "FTP 접속을 로그로 남기며 분당 3회만" 같은 규칙을 rich rule 로 쓴다. 2222 를 rich rule 로 허용하고 앞의 단순 `--add-port` 는 비교용으로 남긴다.

```bash
firewall-cmd --permanent --zone=internal --add-rich-rule='rule family=ipv4 source address=192.168.64.0/24 port port=2222 protocol=tcp accept'
firewall-cmd --permanent --zone=public   --add-rich-rule='rule family=ipv4 port port=2222 protocol=tcp reject'
firewall-cmd --permanent --zone=internal --add-rich-rule='rule family=ipv4 source address=192.168.64.0/24 service name=ftp log prefix="FTP " level=info limit value=3/m accept'
firewall-cmd --reload
firewall-cmd --list-rich-rules --zone=internal
```

- `rule family=ipv4|ipv6` : 주소 계열 (source/destination 쓰면 필수)
- `source address=<CIDR>` / `destination address=` : 출발지/목적지. `NOT` 부정 가능 (`source NOT address=…`)
- `service name=<서비스>` / `port port=<n> protocol=tcp|udp` : 매칭 대상
- `log prefix="<접두>" level=<emerg…debug> limit value=<n>/<s|m|h|d>` : 커널 로그 + 속도 제한
- `audit` : auditd 로 기록 (참고)
- `accept` / `reject [type=…]` / `drop` / `mark set=` : 동작. `reject` 기본은 icmp-port-unreachable
- `--list-rich-rules`, `--remove-rich-rule='…'` (문자열 완전 일치 필요), `--query-rich-rule`

**검증**

```bash
# macOS 에서 FTP 접속 4회 유도 → 3회만 로그
# for i in 1 2 3 4; do nc -zv -w1 192.168.64.10 21; done
journalctl -k -g 'FTP ' -n 5 --no-pager
firewall-cmd --permanent --zone=internal --remove-port=2222/tcp; firewall-cmd --reload   # 단순 포트 제거 → rich rule 만으로 ssh 유지 확인
# macOS: ssh -p 2222 admin1@192.168.64.10 'echo still-ok'
```

```text
... kernel: FTP IN=enp0s1 OUT= MAC=... SRC=192.168.64.1 DST=192.168.64.10 ... PROTO=TCP SPT=... DPT=21 ...
... kernel: FTP IN=enp0s1 ...
... kernel: FTP IN=enp0s1 ...
still-ok
```

> 📝 **시험 포인트**: rich rule 문법 `rule family=… source address=… service name=… accept` 빈칸 채우기. rich rule 은 존 안에서 **일반 서비스/포트 규칙보다 먼저** 평가. 로그는 커널(`journalctl -k` = `dmesg`)에 남음.

### 2-5. 포트 포워딩과 마스커레이드

> **상황**: 외부에서 80 으로 들어온 요청을 8080 vhost 로 넘겨 보고, 서버를 라우터로 쓸 때 필요한 masquerade 도 켜 본다. 포트 포워딩은 PREROUTING(nat) 에서 처리되므로 **서버 자신의 curl 로는 검증 불가** — macOS 에서 확인한다.

```bash
firewall-cmd --permanent --zone=internal --add-forward-port=port=80:proto=tcp:toport=8080
firewall-cmd --permanent --zone=internal --add-masquerade
firewall-cmd --reload
firewall-cmd --zone=internal --list-forward-ports
firewall-cmd --zone=internal --query-masquerade
```

- `--add-forward-port=port=<수신>:proto=<tcp|udp>:toport=<전달포트>[:toaddr=<IP>]` : 목적지 포트(주소) 변환. `toaddr` 다른 호스트로 보낼 땐 `masquerade` 필요
- `--add-masquerade` / `--remove-masquerade` / `--query-masquerade` : 이 존을 통해 **나가는** 패킷의 출발지를 서버 IP 로 바꿈(동적 SNAT). `ip_forward` 도 자동 활성
- `--list-forward-ports` : 포워딩 규칙 목록

**검증**

```bash
sysctl net.ipv4.ip_forward                 # masquerade 로 1 로 바뀜
curl -s http://127.0.0.1:8080/ | head -3   # 서버 로컬: 8080 vhost 내용
# macOS: curl -s http://192.168.64.10/ | head -3     → 위와 같은(8080) 내용이어야 포워딩 성공
nft list chain inet firewalld nat_PRE_internal_allow 2>/dev/null | grep -E 'dport 80|redirect|dnat'
# 확인 후 원복
firewall-cmd --permanent --zone=internal --remove-forward-port=port=80:proto=tcp:toport=8080
firewall-cmd --permanent --zone=internal --remove-masquerade
firewall-cmd --reload; sysctl -w net.ipv4.ip_forward=0
```

```text
net.ipv4.ip_forward = 1
<!DOCTYPE html>
...
        tcp dport 80 redirect to :8080
net.ipv4.ip_forward = 0
```

> 📝 **시험 포인트**: 포트 포워딩 = DNAT/REDIRECT(PREROUTING), masquerade = 동적 SNAT(POSTROUTING) — iptables 절(3-9) 과 대응. `external` 존은 masquerade 가 기본 on.

### 2-6. ICMP 차단 — ping 막고 풀기

> **상황**: 보안 정책상 외부 ping 응답을 막으라는 요구가 흔하다. echo-request 를 차단하고 macOS 에서 실패를 확인한 뒤, 사내망 진단이 불편하므로 다시 푼다.

```bash
firewall-cmd --get-icmptypes | tr ' ' '\n' | grep echo
firewall-cmd --zone=internal --add-icmp-block=echo-request          # 런타임만
firewall-cmd --zone=internal --list-icmp-blocks
```

- `--get-icmptypes` : 차단 가능한 ICMP 타입 이름 목록
- `--add-icmp-block=<타입>` / `--remove-icmp-block=` / `--list-icmp-blocks` / `--query-icmp-block=`
- `--add-icmp-block-inversion` : 목록에 **없는** ICMP 만 차단으로 반전 (참고)

**검증**

```bash
# macOS: ping -c 2 -W 1 192.168.64.10      → 100% packet loss
firewall-cmd --zone=internal --remove-icmp-block=echo-request
# macOS: ping -c 2 192.168.64.10           → 정상 응답
firewall-cmd --zone=internal --list-icmp-blocks | wc -w
```

```text
0
```

> 📝 **시험 포인트**: iptables 대응 명령 `iptables -A INPUT -p icmp --icmp-type echo-request -j DROP`(R02-96). Smurf 대응은 브로드캐스트 ICMP 무시(`icmp_echo_ignore_broadcasts`, 6-8) 와 구분.

### 2-7. 커스텀 서비스 정의 — lab-redis (6379)

> **상황**: Part 11 의 Redis 컨테이너 포트 6379 를 매번 `--add-port` 로 쓰기보다 이름 있는 서비스로 정의해 두면 `--list-all` 에서 의미가 드러난다. 커스텀 존 생성도 같은 요령이다.

```bash
firewall-cmd --permanent --new-service=lab-redis
firewall-cmd --permanent --service=lab-redis --set-short="Lab Redis"
firewall-cmd --permanent --service=lab-redis --set-description="Redis cache for lab (Part 11 container)"
firewall-cmd --permanent --service=lab-redis --add-port=6379/tcp
firewall-cmd --reload
firewall-cmd --info-service=lab-redis
cat /etc/firewalld/services/lab-redis.xml
firewall-cmd --permanent --new-zone=labdmz && firewall-cmd --reload && firewall-cmd --get-zones | tr ' ' '\n' | grep labdmz
firewall-cmd --permanent --delete-zone=labdmz && firewall-cmd --reload
```

- `--new-service=<이름>` : `/etc/firewalld/services/<이름>.xml` 생성 (`--permanent` 필수)
- `--service=<이름> --set-short=` / `--set-description=` / `--add-port=` / `--add-protocol=` / `--add-module=` : 서비스 정의 편집
- `--new-service-from-file=<xml>`, `--delete-service=` : 파일에서 생성/삭제
- `--new-zone=<이름>` / `--delete-zone=` : 커스텀 존 (역시 `--permanent` 필수, reload 후 사용 가능)
- `--check-config` : `/etc/firewalld` 아래 xml 문법·참조 검사

**검증**

```bash
firewall-cmd --check-config && echo "config ok"
firewall-cmd --permanent --zone=internal --add-service=lab-redis && firewall-cmd --reload
firewall-cmd --list-services --zone=internal
```

```text
config ok
success
success
dns ftp http https lab-redis mountd nfs rpc-bind samba
```

- `/etc/firewalld/services/lab-redis.xml` 형식

```xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>Lab Redis</short>
  <description>Redis cache for lab (Part 11 container)</description>
  <port port="6379" protocol="tcp"/>
</service>
```

> 📝 **시험 포인트**: 서비스 xml 의 `<port port="…" protocol="…"/>` 요소. 커스텀 정의는 `/etc/firewalld/`, 배포판 정의는 `/usr/lib/firewalld/` — 위치 문제 출제.

### 2-8. 거부 로그·패닉 모드·direct 인터페이스

> **상황**: 무엇이 차단되는지 보려면 거부 로그를 켠다. 로그가 폭증하므로 관찰 후 반드시 끈다. 패닉 모드와 direct 는 동작만 이해한다.

```bash
firewall-cmd --get-log-denied
firewall-cmd --set-log-denied=all           # 런타임+영구 동시 (자동 reload)
# macOS: nc -zv -w1 192.168.64.10 6379  (컨테이너 미기동 → REJECT 로그)   /  nc -zv -w1 192.168.64.10 9999
journalctl -k -g 'FINAL_REJECT|REJECT' -n 5 --no-pager
firewall-cmd --set-log-denied=off
```

- `--get-log-denied` / `--set-log-denied=<all|unicast|broadcast|multicast|off>` : 존 target 에 걸려 거부·폐기되는 패킷을 커널 로그로. `firewalld.conf` 의 `LogDenied=` 갱신
- `--panic-on` / `--panic-off` / `--query-panic` : ⚠️ **모든 패킷 즉시 차단**(수신·송신 전부, 기존 세션 포함) — 원격에서 실행하면 그 즉시 잠김. 침해 격리 시 **콘솔에서만**. 설명만, 미실행
- `--direct --add-rule ipv4 filter INPUT 0 …` : iptables 문법 규칙을 firewalld 체인에 직접 삽입 — nftables 백엔드에서는 비권장(deprecated), `--permanent --direct --get-all-rules` 로 조회 (참고)

**검증**

```bash
firewall-cmd --get-log-denied
grep LogDenied /etc/firewalld/firewalld.conf
```

```text
... kernel: FINAL_REJECT: IN=enp0s1 OUT= MAC=... SRC=192.168.64.1 DST=192.168.64.10 ... PROTO=TCP SPT=... DPT=9999 ...
off
LogDenied=off
```

> 📝 **시험 포인트**: firewalld 거부 로그의 접두어 `FINAL_REJECT`, iptables 의 `LOG` 타겟 접두어(`--log-prefix`, 3-6) 모두 **커널 링버퍼**(`dmesg`/`journalctl -k`). `--panic-on` 은 "긴급 전체 차단".

### 2-9. 접속 재검증과 nft 규칙 엿보기

> **상황**: 설계가 끝났으니 Part 09 클라이언트 명령으로 모든 서비스가 사내망에서 여전히 동작하는지 확인하고, firewalld 가 실제로 생성한 nftables 규칙을 본다.

```bash
curl -sI http://192.168.64.10/ | head -1
curl -skI https://192.168.64.10/ | head -1
dig @192.168.64.10 srv01.lab.local +short
smbclient -L //192.168.64.10 -U dev1
showmount -e 192.168.64.10
nc -zv -w1 192.168.64.10 21
firewall-cmd --list-all --zone=internal
nft list ruleset | head -40
nft list tables
```

- `nft list ruleset` : 커널 nftables 규칙 전체 (firewalld 는 `inet firewalld` 테이블 사용)
- `nft list tables` : 테이블 목록 (`inet firewalld`, `ip firewalld`, `ip6 firewalld`)
- 참고 nft 직접 명령 (firewalld 가 관리하지 않는 별도 테이블에만 — 혼용 금지)

| nft 명령 | 의미 |
| --- | --- |
| `nft list ruleset` | 전체 규칙 출력 |
| `nft add table inet filter` | 테이블 생성 |
| `nft add chain inet filter input '{ type filter hook input priority 0; policy drop; }'` | 기본 체인 생성(훅·정책) |
| `nft add rule inet filter input tcp dport 22 accept` | 규칙 추가 |
| `nft -a list chain inet filter input` | 핸들 번호 포함 출력 |
| `nft delete rule inet filter input handle <n>` | 핸들로 삭제 |
| `nft flush ruleset` | 전체 삭제 (⚠️) |
| `nft list ruleset > /etc/sysconfig/nftables.conf` | 저장 (`nftables.service`) |

**검증**

```bash
nft list ruleset | grep -cE 'tcp dport (2222|80|443|445|2049) '
nft list chain inet firewalld filter_IN_internal_allow | grep -E 'dport (2222|445)'
```

```text
...
        tcp dport 2222 ct state { new, untracked } accept
        tcp dport 445 ct state { new, untracked } accept
HTTP/1.1 200 OK
HTTP/1.1 200 OK
192.168.64.10
```

> 📝 **시험 포인트**: RHEL 9 방화벽 계층 = `firewall-cmd`(관리) → `firewalld`(데몬) → `nftables`(커널 인터페이스) → netfilter(커널). `iptables` 명령도 `iptables-nft` 래퍼로 nft 규칙을 만든다(3-1).

---

## 3. iptables 직접 실습 — 같은 정책을 손으로

### 3-1. 준비: firewalld 중단·마스킹, iptables-services 활성

> **상황**: firewalld 와 iptables 서비스는 같은 넷필터를 두고 충돌한다. 실습 동안만 firewalld 를 마스킹하고, 규칙을 파일로 저장·복원해 주는 `iptables-services` 를 쓴다. Rocky 9 의 `iptables` 는 nft 백엔드 래퍼다.

⚠️ 이 절(3-1 ~ 3-12) 전체를 **UTM 콘솔** 에서 수행. 정책을 DROP 으로 바꾸는 순간 규칙이 없으면 SSH 가 끊긴다

```bash
dnf install -y iptables-services
iptables -V                                  # (nf_tables) 표기 확인
systemctl disable --now firewalld
systemctl mask firewalld
systemctl enable --now iptables
iptables -F; iptables -X; iptables -Z        # 기본 규칙 비우고 시작
iptables -t nat -F; iptables -t nat -X
iptables -L -n -v --line-numbers
```

- `iptables-services` : `iptables.service`·`ip6tables.service` 유닛 + `/etc/sysconfig/iptables` 로드/저장 스크립트. 규칙 도구 자체(`iptables-nft`)는 기본 설치
- `-V` : 버전과 백엔드 (**V**ersion). `(nf_tables)` = nftables 커널 API 사용, `(legacy)` 는 구식
- `systemctl mask` : `/dev/null` 심볼릭 링크로 유닛 시작 자체를 봉쇄 (의존 관계로 딸려 올라오는 것 방지)
- `-F [체인]` : 규칙 전부 삭제 (**F**lush). 정책은 유지
- `-X [체인]` : 비어 있는 사용자 정의 체인 삭제 (e**X**pire/delete chain)
- `-Z [체인]` : 패킷·바이트 카운터 0 으로 (**Z**ero)
- `-t <테이블>` : 대상 테이블 (**t**able, 기본 `filter`)
- `-L` : 규칙 목록 (**L**ist) · `-n` 숫자 표기 · `-v` 카운터·인터페이스 (**v**erbose) · `--line-numbers` 규칙 번호

**검증**

```bash
systemctl is-active firewalld iptables
systemctl is-enabled firewalld
iptables -S
```

```text
inactive
active
masked
-P INPUT ACCEPT
-P FORWARD ACCEPT
-P OUTPUT ACCEPT
```

> 📝 **시험 포인트**: `-F`(규칙 비움) ≠ `-P`(정책 설정) ≠ `-X`(체인 삭제) — 옵션 설명 "틀린 것" 유형(R01-94·R04-91). 정책이 ACCEPT 인 빈 상태 = 무방비.

### 3-2. 테이블·체인 구조와 패킷 흐름

> **상황**: 규칙을 쓰기 전에 어느 테이블의 어느 체인을 패킷이 지나가는지 그림으로 정리한다. FORWARD 문제(R07-91) 와 NAT 체인 위치(R03-92·R10-95) 가 여기서 나온다.

```bash
iptables -t filter -L -n | grep '^Chain'
iptables -t nat    -L -n | grep '^Chain'
iptables -t mangle -L -n | grep '^Chain'
iptables -t raw    -L -n | grep '^Chain'
```

- 테이블 ↔ 기본 체인

| 테이블 | 용도 | 포함 체인 |
| --- | --- | --- |
| `filter` (기본) | 허용/차단 | `INPUT` `FORWARD` `OUTPUT` |
| `nat` | 주소·포트 변환 (새 연결 첫 패킷만) | `PREROUTING` `INPUT` `OUTPUT` `POSTROUTING` |
| `mangle` | 패킷 헤더 변조 (TTL·TOS·MARK) | 5개 체인 전부 |
| `raw` | conntrack 제외(`NOTRACK`) | `PREROUTING` `OUTPUT` |
| `security` | SELinux 패킷 라벨(SECMARK) | `INPUT` `FORWARD` `OUTPUT` (참고) |

- 패킷 흐름

```text
        수신 NIC
          │
   [raw/mangle/nat] PREROUTING   ← DNAT · REDIRECT 여기
          │
     라우팅 결정 ── 목적지가 나? ──┐
          │ no                    │ yes
   [mangle/filter] FORWARD   [mangle/filter] INPUT
          │                       │
          │                   로컬 프로세스
          │                       │
          │                 [raw/mangle/nat/filter] OUTPUT
          │                       │
          └──────────┬────────────┘
   [mangle/nat] POSTROUTING      ← SNAT · MASQUERADE 여기
          │
        송신 NIC
```

**검증**

```bash
iptables -t nat -L -n | grep -c '^Chain'
```

```text
Chain INPUT (policy ACCEPT)
Chain FORWARD (policy ACCEPT)
Chain OUTPUT (policy ACCEPT)
Chain PREROUTING (policy ACCEPT)
Chain INPUT (policy ACCEPT)
Chain OUTPUT (policy ACCEPT)
Chain POSTROUTING (policy ACCEPT)
...
4
```

> 📝 **시험 포인트**: 포워딩 패킷 경로 **PREROUTING → FORWARD → POSTROUTING**(R07-91); 나에게 오는 패킷 PREROUTING → INPUT; 내가 보내는 패킷 OUTPUT → POSTROUTING. DNAT 는 PREROUTING, SNAT/MASQUERADE 는 POSTROUTING.

### 3-3. 기본 정책과 필수 허용 규칙 — internal 존 재현

> **상황**: 2-2 의 internal 존 정책을 iptables 로 다시 쓴다. 순서가 중요하다 — 허용 규칙을 먼저 넣고 **마지막에** 정책을 DROP 으로 바꾼다.

⚠️ `-P INPUT DROP` 은 콘솔에서. 그 전에 `-i lo`·`ESTABLISHED,RELATED`·2222 허용 규칙 3개가 들어갔는지 `iptables -S INPUT` 으로 재확인

```bash
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
iptables -A INPUT -p tcp --dport 2222 -s 192.168.64.0/24 -j ACCEPT
iptables -A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT
iptables -A INPUT -p udp --dport 53 -j ACCEPT
iptables -A INPUT -p tcp --dport 53 -j ACCEPT
iptables -A INPUT -p tcp -m multiport --dports 139,445 -s 192.168.64.0/24 -j ACCEPT
iptables -A INPUT -p udp -m multiport --dports 137,138 -s 192.168.64.0/24 -j ACCEPT
iptables -A INPUT -p tcp -m multiport --dports 111,2049,20048 -s 192.168.64.0/24 -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT
iptables -S INPUT                                  # 규칙 확인 후
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```

- `-A <체인>` : 체인 끝에 추가 (**A**ppend)
- `-P <체인> <ACCEPT|DROP>` : 기본 정책 (**P**olicy) — 어느 규칙에도 안 걸린 패킷의 운명. REJECT 는 정책으로 불가(규칙으로만)
- `-i <NIC>` / `-o <NIC>` : 입력/출력 인터페이스 (**i**n / **o**ut). `lo` 허용은 로컬 서비스 간 통신 필수
- `-m conntrack --ctstate ESTABLISHED,RELATED` : 연결 추적 매치 (**m**atch module). `-m state --state …` 와 동일 의미(구형 이름, 여전히 동작). `ESTABLISHED`=이미 맺어진 연결의 패킷, `RELATED`=연관 연결(FTP 데이터, ICMP 오류), `NEW`=첫 패킷, `INVALID`=추적 불가
- `-p tcp|udp|icmp|all` : 프로토콜 (**p**rotocol)
- `--dport <포트>` / `--sport <포트>` : 목적지/출발지 포트 (`-p` 필수). 범위 `--dport 6000:6010`
- `-s <IP|CIDR>` / `-d <IP|CIDR>` : 출발지/목적지 주소 (**s**ource / **d**estination)
- `-m multiport --dports a,b,c` : 포트 여러 개(최대 15) 한 규칙에
- `--icmp-type echo-request` : ICMP 타입 (`echo-reply`, `destination-unreachable` …)
- `-m limit --limit 1/s [--limit-burst n]` : 토큰 버킷 속도 제한 — ping flood 완화
- `-j <타겟>` : 매칭 시 동작 (**j**ump) — `ACCEPT` `DROP` `REJECT` `LOG` `RETURN` 또는 사용자 체인

**검증**

```bash
iptables -L INPUT -n -v --line-numbers
# macOS: ssh -p 2222 admin1@192.168.64.10 'echo ok'; curl -sI http://192.168.64.10/ | head -1; nc -zv -w1 192.168.64.10 21
ping -c 3 -i 0.2 192.168.64.10                  # 1/s 제한 → 일부 손실
```

```text
Chain INPUT (policy DROP 0 packets, 0 bytes)
num   pkts bytes target  prot opt in  out source            destination
1       ..    .. ACCEPT  0    --  lo  *   0.0.0.0/0         0.0.0.0/0
2       ..    .. ACCEPT  0    --  *   *   0.0.0.0/0         0.0.0.0/0    ctstate RELATED,ESTABLISHED
3        0     0 DROP    0    --  *   *   0.0.0.0/0         0.0.0.0/0    ctstate INVALID
4       ..    .. ACCEPT  6    --  *   *   192.168.64.0/24   0.0.0.0/0    tcp dpt:2222
5       ..    .. ACCEPT  6    --  *   *   0.0.0.0/0         0.0.0.0/0    multiport dports 80,443
...
11      ..    .. ACCEPT  1    --  *   *   0.0.0.0/0         0.0.0.0/0    icmptype 8 limit: avg 1/sec burst 5
ok
HTTP/1.1 200 OK
nc: connectx to 192.168.64.10 port 21 (tcp) failed: Operation timed out      ← 21 은 DROP(무응답)
```

> 📝 **시험 포인트**: 실기 최다 출제 세트 — `-P INPUT DROP` + `-m state --state ESTABLISHED,RELATED -j ACCEPT` + `-p tcp --dport 22 -j ACCEPT` 의 최종 동작(R04-12 실기, R06-91·R10-94 필기): "기본 차단, 기존 연결 응답과 새 SSH 만 허용". `-m state` 빈칸(R06-9 실기). 카운터 열 `pkts` 로 매칭 여부 판독(R08-93).

### 3-4. DROP 과 REJECT 체감

> **상황**: 21 번을 무응답으로 버리는 것과 즉시 거부하는 것의 차이를 클라이언트 입장에서 느낀다. 응답 없는 DROP 은 타임아웃까지 기다리고, REJECT 는 바로 실패한다.

```bash
iptables -A INPUT -p tcp --dport 21 -j REJECT --reject-with tcp-reset
iptables -A INPUT -p tcp --dport 6379 -j DROP
iptables -A INPUT -p udp --dport 9999 -j REJECT          # 기본 icmp-port-unreachable
```

- `REJECT` : 거부 + 오류 응답. `--reject-with` 로 종류 지정 — `icmp-port-unreachable`(기본) `icmp-host-prohibited` `tcp-reset`(TCP 전용, "연결 거부" 로 보임)
- `DROP` : 응답 없이 폐기 — 스캐너에게 포트가 `filtered` 로 보이고 정찰이 느려짐. 정당 사용자는 원인 파악 어려움

**검증**

```bash
# macOS
# time nc -zv -w3 192.168.64.10 21        → 즉시 "Connection refused" (RST)
# time nc -zv -w3 192.168.64.10 6379      → 3초 후 timeout
# nmap -p 21,6379 192.168.64.10
iptables -L INPUT -n -v | grep -E 'dpt:(21|6379)'
```

```text
21/tcp   closed   ftp          ← REJECT tcp-reset = closed 로 보임
6379/tcp filtered redis        ← DROP = filtered
   ..   .. REJECT  6  -- * * 0.0.0.0/0 0.0.0.0/0 tcp dpt:21 reject-with tcp-reset
   ..   .. DROP    6  -- * * 0.0.0.0/0 0.0.0.0/0 tcp dpt:6379
```

> 📝 **시험 포인트**: "차단하면서 발신지에 거부 응답을 보내는 타겟" = **REJECT**(R04-93). firewalld `block` 존 = REJECT, `drop` 존 = DROP.

### 3-5. 삽입·삭제·교체 — 규칙 번호 다루기

> **상황**: 규칙은 위에서 아래로 첫 매칭이 적용되므로 위치가 결과를 바꾼다. 번호로 삽입·삭제·교체를 연습한다.

```bash
iptables -L INPUT -n --line-numbers | head -6
iptables -I INPUT 1 -s 192.168.64.99 -j DROP             # 1번 위치에 삽입 (특정 호스트 차단은 맨 위)
iptables -I INPUT -p tcp --dport 8080 -s 192.168.64.0/24 -j ACCEPT   # 번호 생략 = 1번
iptables -L INPUT -n --line-numbers | head -4
iptables -D INPUT 1                                      # 번호로 삭제 (8080 규칙)
iptables -D INPUT -s 192.168.64.99 -j DROP               # 규칙 그대로 지정해 삭제
iptables -R INPUT 3 -m conntrack --ctstate INVALID -j LOG --log-prefix "INVALID: "   # 3번 교체
iptables -R INPUT 3 -m conntrack --ctstate INVALID -j DROP                            # 원복
iptables -S INPUT | head -5
```

- `-I <체인> [번호]` : 지정 위치에 삽입 (**I**nsert). 번호 생략 시 1 (맨 위)
- `-D <체인> <번호>` 또는 `-D <체인> <규칙 명세>` : 삭제 (**D**elete). 명세 삭제는 옵션까지 완전 일치해야 함
- `-R <체인> <번호> <규칙>` : 해당 번호 규칙 교체 (**R**eplace)
- `-S [체인]` : 규칙을 **명령 형식**으로 출력 (**S**pecification) — `iptables-save` 와 유사, 복붙용
- `-C <체인> <규칙>` : 규칙 존재 검사 (**C**heck) — 종료코드로 판단 (참고)

**검증**

```bash
iptables -C INPUT -s 192.168.64.99 -j DROP; echo "exit=$?"       # 1 = 없음
iptables -L INPUT -n --line-numbers | sed -n '4,6p'
```

```text
iptables: Bad rule (does a matching rule exist in that chain?).
exit=1
3   DROP    0  -- 0.0.0.0/0        0.0.0.0/0   ctstate INVALID
4   ACCEPT  6  -- 192.168.64.0/24  0.0.0.0/0   tcp dpt:2222
5   ACCEPT  6  -- 0.0.0.0/0        0.0.0.0/0   multiport dports 80,443
```

> 📝 **시험 포인트**: 실기 R04-8 `iptables ______ INPUT 3` = `-D`(3번 규칙 삭제). `-I` 는 앞, `-A` 는 뒤 — 차단 규칙을 `-A` 로 붙이면 앞의 ACCEPT 에 밀려 무효(3-7).

### 3-6. 사용자 정의 체인 + LOG 타겟

> **상황**: 차단할 때마다 로그를 남기고 싶다. `LOG` 는 종결 타겟이 아니어서 다음 규칙으로 계속 흐르므로, "LOG 후 DROP" 두 줄을 사용자 체인에 묶어 재사용한다.

```bash
iptables -N LAB_LOG
iptables -A LAB_LOG -m limit --limit 5/m --limit-burst 10 -j LOG --log-prefix "LAB-DROP: " --log-level 4
iptables -A LAB_LOG -j DROP
iptables -A INPUT -p tcp --dport 23 -j LAB_LOG              # telnet 시도 → 로그+차단
iptables -A INPUT -p tcp --dport 6379 -j LAB_LOG            # 3-4 의 6379 DROP 뒤라 도달 안 함 → 3-7 에서 확인
iptables -L LAB_LOG -n -v
```

- `-N <이름>` : 사용자 정의 체인 생성 (**N**ew chain). 정책 없음 — 끝까지 매칭 안 되면 호출한 체인으로 **RETURN**
- `-j LOG --log-prefix "<접두> " --log-level <0-7|이름>` : 커널 로그 기록 후 **계속 진행**(비종결). level 4 = warning
- `-j RETURN` : 호출 체인으로 복귀 (사용자 체인에서만 의미)
- `-E <구이름> <신이름>` : 체인 이름 변경 (r**E**name, 참고)
- `-j <사용자체인>` : 해당 체인으로 점프

**검증**

```bash
# macOS: nc -zv -w2 192.168.64.10 23
journalctl -k -g 'LAB-DROP' -n 3 --no-pager
iptables -L LAB_LOG -n -v | tail -2
```

```text
... kernel: LAB-DROP: IN=enp0s1 OUT= MAC=... SRC=192.168.64.1 DST=192.168.64.10 LEN=64 ... PROTO=TCP SPT=... DPT=23 ... SYN ...
    1    64 LOG   0  --  *  *  0.0.0.0/0  0.0.0.0/0  limit: avg 5/min burst 10 LOG flags 0 level 4 prefix "LAB-DROP: "
    1    64 DROP  0  --  *  *  0.0.0.0/0  0.0.0.0/0
```

> 📝 **시험 포인트**: `LOG` 는 **비종결 타겟**(다음 규칙 계속), `ACCEPT/DROP/REJECT` 는 종결. 로그 위치 = 커널 메시지(`/var/log/messages`·`dmesg`·`journalctl -k`), 형식 `SRC= DST= PROTO= SPT= DPT=`.

### 3-7. 매칭 순서 실습 — 첫 매칭이 이긴다

> **상황**: 6379 는 이미 3-4 에서 DROP 규칙이 있는데 3-6 에서 LAB_LOG 를 뒤에 붙였다. 뒤 규칙은 절대 도달하지 않음을 카운터로 확인하고, 반대로 ACCEPT 뒤에 DROP 을 붙여도 통과함을 본다.

```bash
iptables -Z INPUT; iptables -Z LAB_LOG
# macOS: nc -zv -w2 192.168.64.10 6379   (2회)
iptables -L INPUT -n -v --line-numbers | grep 6379
iptables -A INPUT -p tcp --dport 80 -j DROP                  # 80 ACCEPT(5번) 뒤에 DROP 추가
# macOS: curl -sI http://192.168.64.10/ | head -1            → 여전히 200
iptables -L INPUT -n -v | grep 'dpt:80'
iptables -D INPUT -p tcp --dport 80 -j DROP
iptables -D INPUT -p tcp --dport 6379 -j LAB_LOG
```

- 평가 규칙: 체인 안에서 **위 → 아래 순차**, 첫 매칭 규칙의 종결 타겟이 결과. 매칭 없으면 정책
- `-Z` 뒤 카운터 증가분으로 "어느 규칙이 잡았는지" 판독

**검증**

```bash
iptables -L INPUT -n -v --line-numbers | grep -E 'dpt:(6379|80)'
```

```text
13   2   120 DROP    6  -- * * 0.0.0.0/0 0.0.0.0/0  tcp dpt:6379           ← 여기서 끝
14   0     0 LAB_LOG 6  -- * * 0.0.0.0/0 0.0.0.0/0  tcp dpt:6379           ← 도달 못 함 (0)
 5   3   180 ACCEPT  6  -- * * 0.0.0.0/0 0.0.0.0/0  multiport dports 80,443
15   0     0 DROP    6  -- * * 0.0.0.0/0 0.0.0.0/0  tcp dpt:80             ← 0 = 무효
```

> 📝 **시험 포인트**: "ACCEPT 규칙 뒤에 같은 조건 DROP 을 `-A` 로 추가하면?" → 앞 규칙이 먼저 매칭돼 **허용 유지**. 차단을 우선시키려면 `-I`. 카운터 0 인 규칙은 죽은 규칙.

### 3-8. 고급 매치 — 부정·SYN·recent·MAC·string

> **상황**: 무차별 대입과 SYN flood 를 iptables 만으로 완화하는 전형 예를 넣고, 기출에 나오는 매치 모듈을 한 번씩 만져 본다. recent 규칙은 macOS 접속까지 막을 수 있으니 확인 후 바로 제거한다.

```bash
# ① SSH 무차별 대입 완화: 60초 내 4번째 NEW 연결부터 차단
iptables -I INPUT 4 -p tcp --dport 2222 -m conntrack --ctstate NEW -m recent --name SSH --set
iptables -I INPUT 5 -p tcp --dport 2222 -m conntrack --ctstate NEW -m recent --name SSH --update --seconds 60 --hitcount 4 -j LAB_LOG
# ② SYN flood 완화: SYN 을 초당 10개(버스트 20)만, 초과는 DROP — 사용자 체인 + RETURN
iptables -N SYN_FLOOD
iptables -A SYN_FLOOD -m limit --limit 10/s --limit-burst 20 -j RETURN
iptables -A SYN_FLOOD -j DROP
iptables -I INPUT 2 -p tcp --syn -j SYN_FLOOD
# ③ 부정·MAC·string (참고용, 카운터만 확인)
iptables -A INPUT ! -s 192.168.64.0/24 -p tcp --dport 2222 -j LAB_LOG              # 사내망 아닌 곳의 2222
iptables -A INPUT -m mac --mac-source 00:11:22:33:44:55 -j DROP                    # 특정 MAC 차단 (L2 인접망만 의미)
iptables -A INPUT -p tcp --dport 80 -m string --string "/etc/passwd" --algo bm -j LAB_LOG   # 페이로드 문자열 (IDS 흉내, 성능 주의)
```

- `!` : 매치 부정 — `! -s`, `! -p`, `! --dport` 등 (옵션 **앞**에 위치)
- `--syn` : `--tcp-flags SYN,RST,ACK,FIN SYN` 약어 = 새 연결 요청 패킷만
- `--tcp-flags <검사 목록> <설정되어야 할 목록>` : 플래그 조합 매치 (XMAS 스캔 차단 `--tcp-flags ALL ALL`, NULL 스캔 `--tcp-flags ALL NONE`, 참고)
- `-m recent --name <표> --set` : 출발지 IP 를 표에 기록. `--update --seconds n --hitcount k` : n초 내 k회 이상이면 매치(+갱신). `--rcheck` 는 갱신 없이 검사, `--remove` 삭제. 표는 `/proc/net/xt_recent/<이름>`
- `-m limit --limit <n/s|m|h|d> --limit-burst <n>` : 평균 속도·초기 허용량
- `-m mac --mac-source <MAC>` : 출발지 MAC (INPUT/FORWARD/PREROUTING 만)
- `-m string --string "<문자열>" --algo bm|kmp` : 페이로드 검색 (**b**oyer-**m**oore). `--hex-string` 도 가능
- 기타 참고: `-m owner --uid-owner`(OUTPUT 만), `-m time`, `-m iprange --src-range`, `-m connlimit --connlimit-above 20`

**검증**

```bash
# macOS: for i in 1 2 3 4 5; do nc -zv -w2 192.168.64.10 2222; done   → 4·5번째 timeout
cat /proc/net/xt_recent/SSH
journalctl -k -g 'LAB-DROP' -n 2 --no-pager | grep -c 'DPT=2222'
iptables -L SYN_FLOOD -n -v
# 확인 후 recent 규칙 제거 (정상 접속 방해)
iptables -D INPUT -p tcp --dport 2222 -m conntrack --ctstate NEW -m recent --name SSH --update --seconds 60 --hitcount 4 -j LAB_LOG
iptables -D INPUT -p tcp --dport 2222 -m conntrack --ctstate NEW -m recent --name SSH --set
```

```text
src=192.168.64.1 ttl: 64 last_seen: ... oldest_pkt: 5 ...
2
Chain SYN_FLOOD (1 references)
 pkts bytes target  prot opt in out source     destination
   ..    .. RETURN  0   --  *  *   0.0.0.0/0  0.0.0.0/0  limit: avg 10/sec burst 20
    0     0 DROP    0   --  *  *   0.0.0.0/0  0.0.0.0/0
```

> 📝 **시험 포인트**: `-m limit --limit 3/min` 을 SSH 에 건 규칙 해석(R09-97) = "신규 연결 속도 제한으로 무차별 대입 완화". SYN Flooding 방어 = SYN 쿠키(6-8) + 속도 제한 + 백로그 확대(실기 R05-14).

### 3-9. NAT — MASQUERADE·REDIRECT·DNAT/SNAT

> **상황**: 서버를 사내망 라우터로 쓸 때의 규칙을 넣어 본다. NIC 가 하나라 실제 포워딩 트래픽은 없으므로 MASQUERADE 는 카운터·`ip_forward` 로만, REDIRECT 는 로컬 생성 패킷용 `nat OUTPUT` 체인을 써서 `curl` 로 검증한다.

```bash
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A POSTROUTING -o enp0s1 -s 192.168.64.0/24 -j MASQUERADE
iptables -A FORWARD -i enp0s1 -o enp0s1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -s 192.168.64.0/24 -j ACCEPT
# 포트 리다이렉트: 8082(리스너 없음) → 80. 외부용은 PREROUTING, 자기 자신 테스트는 OUTPUT
iptables -t nat -A PREROUTING -p tcp --dport 8082 -j REDIRECT --to-port 80
iptables -t nat -A OUTPUT -o lo -p tcp --dport 8082 -j REDIRECT --to-port 80
# 참고 (다른 호스트 대상 — 여기선 실행 안 함)
# iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.64.20:8080
# iptables -t nat -A POSTROUTING -s 192.168.64.0/24 -o enp0s1 -j SNAT --to-source 203.0.113.10
iptables -t nat -L -n -v --line-numbers
```

- `net.ipv4.ip_forward=1` : 커널 라우팅(포워딩) 활성 — NAT 라우터의 전제. `-w` 즉시 적용(**w**rite), 영구는 `/etc/sysctl.d/`
- `-t nat -A POSTROUTING -o <외부NIC> -j MASQUERADE` : 나가는 패킷 출발지를 **그 NIC 의 현재 IP** 로 — 동적 IP 환경용 SNAT
- `-j SNAT --to-source <고정IP>` : 출발지를 지정 IP 로 (고정 공인 IP)
- `-t nat -A PREROUTING … -j DNAT --to-destination <IP>[:포트]` : 목적지 변환 = 포트포워딩
- `-j REDIRECT --to-port <포트>` : 목적지를 **자기 자신** 의 다른 포트로 (투명 프록시). PREROUTING(외부 유입)·OUTPUT(로컬 생성)
- `FORWARD` 허용 규칙 : 정책이 DROP 이므로 라우팅될 패킷을 명시 허용

**검증**

```bash
curl -sI http://127.0.0.1:8082/ | head -1              # REDIRECT 로 80 응답
ss -tln | grep -c ':8082'                               # 리스너 없음 = 0
iptables -t nat -L OUTPUT -n -v | grep 8082             # pkts 증가
iptables -t nat -L POSTROUTING -n -v                    # MASQUERADE 카운터(포워딩 트래픽 없어 0)
sysctl net.ipv4.ip_forward
```

```text
HTTP/1.1 200 OK
0
    1    60 REDIRECT  6  -- * lo 0.0.0.0/0 0.0.0.0/0  tcp dpt:8082 redir ports 80
Chain POSTROUTING (policy ACCEPT ...)
    0     0 MASQUERADE 0 -- * enp0s1 192.168.64.0/24 0.0.0.0/0
net.ipv4.ip_forward = 1
```

> 📝 **시험 포인트**: 실기 R03-12·R06-13 `-t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to 192.168.0.10:8080` = "nat 테이블 PREROUTING, 80 유입을 내부 192.168.0.10:8080 으로 목적지 변환(포트포워딩)". MASQUERADE 해석(R03-92·R10-95) = "내부 사설 대역이 외부로 나갈 때 출발지를 NIC IP 로 바꾸는 동적 SNAT". PREROUTING 은 라우팅 **전**, POSTROUTING 은 **후**.

### 3-10. 저장·복원과 파일 포맷

> **상황**: 지금까지의 규칙은 메모리에만 있다. `iptables-services` 가 부팅 시 읽는 파일로 저장하고, 포맷을 읽고, 지웠다가 복원한다.

```bash
iptables-save > /etc/sysconfig/iptables            # 방법 1
service iptables save                                # 방법 2 (같은 파일에 기록)
cat /etc/sysconfig/iptables
grep -E '^(IPTABLES_SAVE_ON_STOP|IPTABLES_SAVE_ON_RESTART)' /etc/sysconfig/iptables-config
iptables -F INPUT; iptables -P INPUT ACCEPT          # 일부러 비움 (콘솔)
iptables -S INPUT | head -2
iptables-restore < /etc/sysconfig/iptables           # 복원
iptables -S INPUT | head -6
ip6tables -S | head -3                                # IPv6 는 별도 도구·별도 파일 /etc/sysconfig/ip6tables
```

- `iptables-save` : 모든 테이블 규칙을 표준출력으로 (카운터 포함 `-c`). 리다이렉트로 저장
- `service iptables save` : `iptables-services` 의 init 스크립트 호출 → `/etc/sysconfig/iptables` 갱신 (systemd 환경에서도 이 하위 명령은 동작)
- `iptables-restore [-n]` : 파일에서 복원. 기본은 **기존 규칙 flush 후** 적용, `-n` 은 flush 없이 추가 (**n**oflush)
- `/etc/sysconfig/iptables-config` : `IPTABLES_SAVE_ON_STOP=no`·`IPTABLES_SAVE_ON_RESTART=no`(기본) — yes 면 서비스 중지·재시작 시 자동 저장
- `ip6tables` / `ip6tables-save` / `/etc/sysconfig/ip6tables` : IPv6 대응 (참고)
- 파일 포맷

```text
# Generated by iptables-save v1.8.x (nf_tables) on ...
*nat                              ← 테이블 시작
:PREROUTING ACCEPT [0:0]          ← :체인 정책 [패킷:바이트]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A PREROUTING -p tcp -m tcp --dport 8082 -j REDIRECT --to-ports 80
-A POSTROUTING -s 192.168.64.0/24 -o enp0s1 -j MASQUERADE
COMMIT                            ← 테이블 끝
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
:LAB_LOG - [0:0]                  ← 사용자 체인은 정책 '-'
:SYN_FLOOD - [0:0]
-A INPUT -i lo -j ACCEPT
-A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
...
COMMIT
```

**검증**

```bash
grep -c '^-A' /etc/sysconfig/iptables
iptables -S | grep -c '^-A'                 # 같은 수
systemctl restart iptables && iptables -L INPUT -n | grep -c ACCEPT
```

```text
2x
2x
1x
```

> 📝 **시험 포인트**: "현재 메모리의 iptables 규칙을 파일로 저장" = `iptables-save > 파일` 또는 `service iptables save`(R05-93). 저장 파일 `/etc/sysconfig/iptables`, 사용자 체인 표기 `:이름 - [0:0]`.

### 3-11. 규칙 해석 문제 5개 (기출 스타일)

> **상황**: 필기·실기에서 규칙 한 줄을 주고 의미를 묻는다. 위에서 직접 친 규칙들을 문제로 바꿔 스스로 답한다.

```bash
iptables -S INPUT | sed -n '2,3p;5p'
iptables -t nat -S | grep -E 'MASQUERADE|REDIRECT'
```

- **문제 1** `iptables -A INPUT -p tcp --dport 22 -s 192.168.0.0/24 -j ACCEPT`
  → INPUT 체인 **끝**에 추가. 출발지 192.168.0.0/24 에서 목적지 22/tcp(SSH) 로 오는 패킷 **허용**. 다른 대역의 22 는 이 규칙에 안 걸리고 뒤 규칙·정책으로
- **문제 2** `iptables -P INPUT DROP` / `-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT` / `-A INPUT -p tcp --dport 22 -j ACCEPT`
  → 기본 정책 전부 폐기. 단, 이미 성립된 연결의 응답·연관 패킷은 허용, 새 연결은 22/tcp 만 허용. 이 서버에서 시작한 통신(웹 접속·dnf) 의 응답은 ESTABLISHED 로 돌아옴
- **문제 3** `iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to 192.168.0.10:8080`
  → nat 테이블, 라우팅 전 단계. 80/tcp 로 들어온 패킷의 **목적지**를 192.168.0.10:8080 으로 변환 = 포트포워딩. 실제 전달엔 `ip_forward=1` + FORWARD 허용 필요
- **문제 4** `iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -o eth0 -j MASQUERADE`
  → 라우팅 후, eth0 로 나가는 192.168.0.0/24 출발 패킷의 **출발지**를 eth0 의 현재 IP 로 = 동적 SNAT(인터넷 공유). 고정 IP 면 `SNAT --to-source`
- **문제 5** `-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT` / `-A INPUT -p tcp --dport 22 -m limit --limit 3/min -j ACCEPT` / `-A INPUT -p tcp --dport 22 -j DROP`
  → 기존 연결 응답 허용, **새** SSH 패킷은 분당 3개까지만 허용하고 초과분은 다음 줄에서 DROP → 무차별 대입 속도 제한. (엄밀히는 NEW 상태 한정 `--ctstate NEW` 를 붙이는 게 정확)

**검증**

```bash
iptables -L INPUT -n -v | awk 'NR>2 && $1>0' | wc -l      # 카운터가 0 보다 큰(실제로 쓰인) 규칙 수
```

```text
...
```

> 📝 **시험 포인트**: 해석 문제의 채점 키워드 — **체인 이름·추가 위치(-A 끝/-I 앞)·프로토콜/포트/출발지·타겟·테이블(nat 면 변환 대상이 출발지인지 목적지인지)**. 서술형은 이 5 요소를 빠뜨리지 않게.

### 3-12. iptables 종료 → firewalld 복귀

> **상황**: 실습이 끝났으니 iptables 서비스를 내리고 firewalld 를 원래대로 되살린다. 규칙 파일은 참고용으로 보관한다.

⚠️ 콘솔에서 수행. iptables 를 내리면 정책 ACCEPT 빈 상태가 잠깐 생기므로 곧바로 firewalld 기동

```bash
cp /etc/sysconfig/iptables /root/iptables.lab.rules
systemctl disable --now iptables
iptables -F; iptables -X; iptables -t nat -F; iptables -t nat -X
iptables -P INPUT ACCEPT; iptables -P FORWARD ACCEPT
sysctl -w net.ipv4.ip_forward=0
systemctl unmask firewalld
systemctl enable --now firewalld
firewall-cmd --state
firewall-cmd --get-active-zones
```

- `systemctl unmask` : 마스킹 해제 (`/etc/systemd/system/firewalld.service → /dev/null` 링크 제거)
- `systemctl enable --now` : 부팅 활성 + 즉시 시작

**검증**

```bash
systemctl is-active firewalld iptables
firewall-cmd --list-all --zone=internal | grep -E 'sources|services|rich'
nft list ruleset | grep -c 'table inet firewalld'
# macOS: ssh -p 2222 admin1@192.168.64.10 'echo back'
```

```text
active
inactive
  sources: 192.168.64.0/24
  services: dns ftp http https lab-redis mountd nfs rpc-bind samba
  rich rules:
1
back
```

> 📝 **시험 포인트**: firewalld 와 iptables.service 는 **동시 사용 금지**(둘 다 넷필터 규칙을 소유하려 함). RHEL 7 이후 기본은 firewalld; 레거시 iptables 규칙을 쓰려면 firewalld 를 `mask` 하고 `iptables-services` 활성.

---

## 4. SELinux — 모드·컨텍스트·포트 라벨·불린

### 4-1. 현재 모드 확인과 일시 전환

> **상황**: 방화벽은 "어디서 오는 패킷인가" 를 보지만 SELinux 는 "어떤 프로세스가 어떤 파일에 접근하는가" 를 본다. Part 09 서비스들이 Enforcing 상태에서 정상 동작 중인지부터 확인한다.

```bash
getenforce                 # Enforcing / Permissive / Disabled 한 단어
sestatus                   # 모드·정책·마운트 요약
sestatus -v                # + 프로세스·파일 컨텍스트 상세 (/etc/sestatus.conf 목록 기준)
cat /sys/fs/selinux/enforce   # 1 = Enforcing, 0 = Permissive
```

- `getenforce` : 현재 **적용 중인** 모드만 출력 — 스크립트 조건문에 그대로 사용
- `sestatus` : SELinux status·mount·root directory·정책 이름·**Current mode** 와 **Mode from config file** 을 함께 표시 → 둘이 다르면 `setenforce` 로 일시 변경된 상태
- `-v` : 상세 (**v**erbose) — `/etc/sestatus.conf` 에 나열된 파일·프로세스의 컨텍스트를 함께 출력
- `/sys/fs/selinux/` : selinuxfs 가상 파일시스템 — `enforce`·`policyvers`·`booleans/` 등 커널 상태 노출

**검증**

```bash
getenforce
sestatus | grep -E 'status|Current mode|Mode from config|Loaded policy'
```

```text
Enforcing
SELinux status:                 enabled
Current mode:                   enforcing
Mode from config file:          enforcing
Loaded policy name:             targeted
```

> 📝 **시험 포인트**: 모드 값 **Enforcing=1 / Permissive=0** (필기 FULL r05-57·r09-56·r10-58). `getenforce` 는 **조회 전용** — `getenforce 0` 같은 인자는 없음(오답 선택지 단골).

### 4-2. 일시 전환 vs 영구 설정

> **상황**: 문제 진단을 위해 잠깐 Permissive 로 내렸다가 되돌리는 흐름을 몸에 익힌다. 영구 설정은 파일이 따로 있다는 것을 `sestatus` 두 줄 차이로 눈으로 확인한다.

⚠️ Permissive 상태에서는 정책 위반이 **차단되지 않음** — 진단 목적으로만 잠깐, 반드시 다시 `setenforce 1`

```bash
setenforce 0               # → Permissive (일시)
getenforce
sestatus | grep -E 'Current mode|Mode from config'   # 두 줄이 달라짐
setenforce 1               # → Enforcing 복귀
setenforce Permissive      # 숫자 대신 이름도 가능
setenforce Enforcing
```

- `setenforce 0|Permissive` : Permissive 로 일시 전환
- `setenforce 1|Enforcing` : Enforcing 으로 일시 전환
- **`setenforce` 로 Disabled 전환 불가** — Disabled 는 `/etc/selinux/config` + 재부팅만 가능
- 재부팅하면 `setenforce` 값은 사라지고 `/etc/selinux/config` 의 `SELINUX=` 값으로 복귀

**검증**

```bash
setenforce 0; getenforce; sestatus | grep 'Current mode'
setenforce 1; getenforce
grep -v '^#' /etc/selinux/config | grep -v '^$'
```

```text
Permissive
Current mode:                   permissive
Enforcing
SELINUX=enforcing
SELINUXTYPE=targeted
```

> 📝 **시험 포인트**: "재부팅 없이 일시적으로 permissive" = `setenforce 0` (실기 r01-7 빈칸, 필기 FULL r05-57·r09-56). "영구" = `/etc/selinux/config` 수정 (필기 FULL r07-58).

### 4-3. `/etc/selinux/config` 와 재라벨링

> **상황**: 영구 설정 파일의 두 키를 정확히 익힌다. 실제로 `disabled` 로 내리지는 않지만, disabled ↔ enforcing 왕복 시 **전체 재라벨링**이 왜 필요한지와 그 트리거 방법을 확인한다.

```bash
cp -a /etc/selinux/config /root/selinux-config.bak     # 원본 보존
cat /etc/selinux/config
ls -l /etc/sysconfig/selinux                            # config 로 향하는 심볼릭 링크
```

| 키 | 값 | 의미 |
| --- | --- | --- |
| `SELINUX=` | `enforcing` | 정책 위반 **차단 + 로그** (운영 권장) |
| | `permissive` | 위반을 **차단하지 않고 로그만** (정책 튜닝) |
| | `disabled` | SELinux 완전 비활성 — **라벨링도 중단** |
| `SELINUXTYPE=` | `targeted` | 기본 — 지정 데몬만 제한, 나머지는 unconfined |
| | `mls` | 다단계 보안(MLS) — 군·정부용 |
| | `minimum` | targeted 축소판 |

- Enforcing ↔ Permissive : **재부팅 불필요** (`setenforce`)
- Disabled ↔ 활성(Enforcing/Permissive) : **재부팅 필요**
- Disabled 로 운영하던 동안 새로 만들어진 파일에는 라벨이 없거나 잘못 붙음 → 활성 복귀 시 **전체 재라벨링** 필수

```bash
# ※ 미실행 — 재라벨링 트리거 두 가지 (실행하면 다음 부팅이 수 분~수십 분 소요)
# touch /.autorelabel && reboot            # 부팅 시 전체 파일시스템 재라벨 후 자동 재부팅
# fixfiles -F onboot                       # 같은 효과 (/.autorelabel 을 -F 플래그로 생성)
# fixfiles -R httpd restore                # 특정 패키지가 소유한 파일만 재라벨
```

- `/.autorelabel` : 루트에 이 빈 파일이 있으면 부팅 초기에 `fixfiles restore` 수행 후 파일 삭제·재부팅
- `fixfiles -F onboot` : `-F` = **F**orce (컨텍스트가 맞아 보여도 강제 재설정), `onboot` = 다음 부팅에 예약
- `fixfiles check|restore|verify [경로]` : 즉시 점검·복원

**검증**

```bash
grep -E '^SELINUX=|^SELINUXTYPE=' /etc/selinux/config
ls -l /.autorelabel 2>/dev/null || echo "no autorelabel pending"
ls /etc/selinux/targeted/policy/
```

```text
SELINUX=enforcing
SELINUXTYPE=targeted
no autorelabel pending
policy.31
```

> 📝 **시험 포인트**: 필기 FULL r05-56 은 `SELINUX=enforcing` 의 의미(위반을 **차단하고 로그 기록**)를 묻고, r07-58 은 "disabled 로의 전환은 재부팅 필요" 를 정답으로 둔다. Part 07 의 `rd.break` 복구에서 `/.autorelabel` 을 만든 이유도 같은 원리(레스큐 셸에서 만든 `/etc/shadow` 라벨 복구). ※ RHEL 9 문서는 완전 비활성화가 필요할 때 config 대신 커널 파라미터 `selinux=0` 방식을 안내 — 이 실습에서는 비활성화하지 않음.

### 4-4. 보안 컨텍스트 4필드 — `-Z` 로 보기

> **상황**: SELinux 판단의 기준인 라벨을 파일·프로세스·사용자 세 관점에서 모두 꺼내 본다. 네 필드가 각각 무슨 뜻인지 눈으로 대조한다.

```bash
ls -Z /etc/passwd /etc/shadow
ls -Zd /var/www/html /srv/www /srv/share /var/ftp/pub
ps -eZ | grep httpd | head -3
ps -eZ | grep -E 'sshd|named|smbd|vsftpd' | head
id -Z                       # 현재 로그인 사용자의 컨텍스트
ls -Z /root/ | head -3
```

- `-Z` : SELinux 컨텍스트 표시 — `ls`·`ps`·`id`·`cp`·`mv`·`mkdir`·`ss` 등에 공통 (암기 포인트)
- `ls -Zd <디렉터리>` : 디렉터리 **자신의** 컨텍스트 (`-d` 없으면 내용물)
- `ps -eZ` : 모든 프로세스(`-e`)의 도메인. `ps -eZ` = `ps -e -o label,pid,cmd` 와 같은 정보

| 위치 | 필드 | 예 | 의미 |
| --- | --- | --- | --- |
| 1 | **user** (SELinux 사용자) | `system_u`, `unconfined_u`, `staff_u` | 리눅스 계정과 별개의 SELinux 사용자. `semanage login` 으로 매핑 |
| 2 | **role** (역할) | `object_r`, `system_r`, `unconfined_r` | 진입 가능한 도메인 집합. **파일은 항상 `object_r`** |
| 3 | **type** (타입/도메인) | `httpd_sys_content_t`, `httpd_t` | **접근 통제의 핵심** — 파일은 type, 프로세스는 **도메인** |
| 4 | **level** (레벨) | `s0`, `s0-s0:c0.c1023` | MLS/MCS 민감도·범주. targeted 에서는 대개 `s0` |

**검증**

```bash
ls -Zd /var/www/html
ps -eZ | grep httpd | head -2
id -Z
```

```text
system_u:object_r:httpd_sys_content_t:s0 /var/www/html
system_u:system_r:httpd_t:s0     ... /usr/sbin/httpd -DFOREGROUND
system_u:system_r:httpd_t:s0     ... /usr/sbin/httpd -DFOREGROUND
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

> 📝 **시험 포인트**: 컨텍스트 순서 **user:role:type:level** — 필기 FULL r09-57 은 `system_u:object_r:httpd_sys_content_t:s0` 의 **세 번째 필드(type)** 가 접근 통제의 핵심임을 묻는다. 파일 목록과 컨텍스트를 함께 = `ls -Z` (필기 FULL r01-65).

### 4-5. TE(Type Enforcement)·도메인 전이와 주요 타입

> **상황**: 왜 `httpd_t` 가 `admin_home_t` 를 못 읽는지 규칙 차원에서 이해한다. targeted 정책의 실제 규칙을 조회해 본다.

```bash
dnf install -y setools-console          # seinfo·sesearch 제공
seinfo --stats | head -12               # 정책 통계 (타입·도메인·불린 개수)
sesearch -A -s httpd_t -t httpd_sys_content_t -c file -p read | head
sesearch -A -s httpd_t -t admin_home_t  -c file -p read | head   # 결과 없음 = 허용 규칙 부재
seinfo -t | grep -c '_t$'               # 정의된 타입 개수
```

- `seinfo` : 로드된 정책 정보 조회 — `--stats`, `-t`(타입), `-r`(역할), `-u`(사용자), `-b`(불린), `--portcon`(포트 컨텍스트)
- `sesearch` : 정책 규칙 검색 — `-A` allow 규칙, `-s` 소스(도메인), `-t` 타겟(타입), `-c` 객체 클래스, `-p` 권한
- **TE(Type Enforcement)** : "도메인 X 는 타입 Y 에 대해 Z 동작을 허용" 형태의 규칙 집합. targeted 정책의 뼈대
- **도메인 전이(domain transition)** : `systemd`(`init_t`) 가 `/usr/sbin/httpd`(`httpd_exec_t`) 를 실행하면 새 프로세스가 자동으로 `httpd_t` 도메인으로 전이 → 실행 파일 라벨이 도메인을 결정

| 타입 | 성격 | 쓰임 |
| --- | --- | --- |
| `httpd_t` | **도메인** | Apache 프로세스 자신 |
| `httpd_exec_t` | 실행 파일 | `/usr/sbin/httpd` — 실행 시 `httpd_t` 로 전이 |
| `httpd_sys_content_t` | 파일 | 웹 문서 **읽기 전용** (`/var/www/html`) |
| `httpd_sys_rw_content_t` | 파일 | 웹에서 **읽기·쓰기** (업로드 디렉터리) |
| `httpd_sys_script_exec_t` | 파일 | CGI 스크립트 |
| `httpd_log_t` | 파일 | `/var/log/httpd` |
| `samba_share_t` | 파일 | Samba 공유 대상 |
| `public_content_t` | 파일 | FTP·Samba·NFS **공용 읽기** |
| `public_content_rw_t` | 파일 | 공용 읽기·쓰기 (+ 해당 서비스 불린 필요) |
| `user_home_t` / `user_home_dir_t` | 파일 | 일반 사용자 홈 내용 / 홈 디렉터리 |
| `admin_home_t` | 파일 | **`/root` 및 그 하위** — 서비스 도메인이 못 읽음 |
| `var_log_t` | 파일 | `/var/log` 일반 로그 |
| `ssh_port_t` / `http_port_t` | 포트 | 22·2222 / 80·443·8080 등 |
| `unconfined_t` | 도메인 | 정책 제한을 받지 않는 사용자 세션 |

**검증**

```bash
ls -Zd /root
ls -Z /usr/sbin/httpd
sesearch -A -s httpd_t -t httpd_sys_content_t -c file -p read | wc -l
```

```text
system_u:object_r:admin_home_t:s0 /root
system_u:object_r:httpd_exec_t:s0 /usr/sbin/httpd
1
```

> 📝 **시험 포인트**: "프로세스의 type 을 특히 **도메인**이라 부른다" 는 서술형 단골. targeted 정책 = 지정 데몬만 제한, 로그인 셸은 `unconfined_t` 라서 SELinux 를 켜도 평소 작업은 영향 없음.

### 4-6. 위반 재현 — `mv` 로 옮긴 파일이 403

> **상황**: 실제 운영에서 가장 흔한 SELinux 사고를 그대로 만든다. `/root` 에서 만든 HTML 을 `mv` 로 웹 루트에 옮기면 **DAC 권한은 멀쩡한데** 웹은 403 을 낸다.

```bash
echo '<h1>SELinux lab</h1>' > /root/report.html
chmod 644 /root/report.html
ls -Z /root/report.html                       # admin_home_t
mv /root/report.html /var/www/html/
ls -lZ /var/www/html/report.html              # 권한 644 root:root, 컨텍스트는 그대로 admin_home_t
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1/report.html
```

- `mv` : 같은 파일시스템 안에서는 **이름만 바꾸는 연산** → inode 에 붙은 SELinux 라벨이 그대로 따라옴
- `curl -o /dev/null -w '%{http_code}'` : 본문 버리고 HTTP 상태 코드만 출력 (**w**rite-out 포맷)

**검증**

```bash
ls -lZ /var/www/html/report.html
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1/report.html
tail -3 /var/log/httpd/error_log
```

```text
-rw-r--r--. 1 root root system_u:object_r:admin_home_t:s0 21 ... /var/www/html/report.html
403
... [core:error] ... (13)Permission denied: [client 127.0.0.1:...] AH00132: file permissions deny server access: /var/www/html/report.html
```

> 📝 **시험 포인트**: **DAC 통과 + MAC 거부 = 403**. `ls -l` 만 보고 "권한 644 인데 왜 안 되지" 로 막히는 게 출제 의도 — 진단 순서는 `ls -lZ` → `ausearch -m avc` → `restorecon`. ※ Part 09 의 `intranet` vhost 가 첫 번째로 정의돼 있어 `127.0.0.1` 요청이 그쪽으로 가면, 같은 절차를 `/srv/www/intranet/` 에서 수행하고 `curl http://intranet.lab.local/report.html` 로 확인.

### 4-7. AVC 추적 — `ausearch` · `audit2allow -w` · `sealert`

> **상황**: 403 의 원인을 추측하지 않고 감사 로그에서 확인한다. AVC(Access Vector Cache) denied 메시지가 `/var/log/audit/audit.log` 에 남아 있다.

```bash
ausearch -m avc -ts recent                       # 최근 10분 AVC 메시지
ausearch -m avc -ts recent -i | tail -20         # 숫자를 사람이 읽는 이름으로 해석
ausearch -m avc -c httpd -ts today | tail -20    # 명령 이름으로 필터
ausearch -c httpd --raw | audit2allow -w         # 왜 거부됐는지 사람이 읽는 설명
```

- `-m <타입>` : 메시지 타입 (**m**essage) — `avc`, `USER_LOGIN`, `SYSCALL`, `USER_AUTH`, `ADD_USER`
- `-ts <시각>` : 시작 시각 (**t**ime **s**tart) — `recent`(10분), `today`, `yesterday`, `boot`, `now`, `MM/DD/YYYY HH:MM:SS`
- `-te` : 종료 시각 (**t**ime **e**nd)
- `-c <명령>` : 실행 파일 이름 (**c**omm)
- `-i` : 숫자 UID·시스템콜 번호·시각을 해석해 출력 (**i**nterpret)
- `--raw` : 원본 레코드 그대로 → `audit2allow` 입력용
- `audit2allow -w` : 거부 이유를 **설명문**으로 출력 (**w**hy) — 관련 불린이 있으면 함께 안내
- `audit2allow -a` : 전체 audit 로그에서 정책 규칙 생성(출력만)

```bash
# setroubleshoot-server 설치 시 사람이 읽는 권고안 제공
dnf install -y setroubleshoot-server
sealert -a /var/log/audit/audit.log | head -40
journalctl -t setroubleshoot --no-pager | tail -5
```

- `sealert -a <로그>` : 로그 전체를 분석(**a**nalyze)해 원인·권고 명령·신뢰도를 출력
- `journalctl -t setroubleshoot` : setroubleshootd 가 저널에 남긴 요약 (`-t` = **t**ag = syslog identifier)

**검증**

```bash
ausearch -m avc -ts recent | grep -c 'denied'
ausearch -c httpd --raw | audit2allow -w | head -12
```

```text
2
Was caused by:
	Missing type enforcement (TE) allow rule.
	You can use audit2allow to generate a loadable module to allow this access.
...
```

```text
# sealert 요약 (발췌)
SELinux is preventing /usr/sbin/httpd from getattr access on the file /var/www/html/report.html.
*****  Plugin restorecon (99.5 confidence) suggests   ***********************
	/sbin/restorecon -v '/var/www/html/report.html'
```

> 📝 **시험 포인트**: AVC denied 로그 위치 `/var/log/audit/audit.log`, 검색 도구 `ausearch -m avc`, 해설 도구 `sealert`·`audit2allow`. auditd 가 죽어 있으면 AVC 가 커널 링버퍼로 가므로 `dmesg | grep -i avc` 로 확인(필기 대비).

### 4-8. 해결 — `restorecon` 으로 기본값 복원 → 200 검증

> **상황**: sealert 권고대로 정책이 정의한 **기본 컨텍스트**로 되돌린다. 파일을 옮긴 게 원인이므로 새 라벨을 만드는 게 아니라 복원이 정답이다.

```bash
restorecon -v /var/www/html/report.html          # 단일 파일 복원 (변경 내역 출력)
ls -Z /var/www/html/report.html
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1/report.html
restorecon -Rv /var/www/html                     # 디렉터리 전체 재귀 복원
```

- `restorecon` : `/etc/selinux/targeted/contexts/files/` 의 규칙과 대조해 **정책 기본 컨텍스트로 복원**
- `-R` : 재귀 (**R**ecursive)
- `-v` : 변경된 항목만 상세 출력 (**v**erbose)
- `-n` : 실제로 바꾸지 않고 무엇이 바뀔지만 표시 (**n**o change = dry-run)
- `-F` : 컨텍스트 **전체 4필드** 강제 재설정 (기본은 type 만) (**F**orce)

**검증**

```bash
restorecon -Rv -n /var/www/html        # 이제 바꿀 게 없어야 함 → 출력 없음
ls -Z /var/www/html/report.html
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1/report.html
ausearch -m avc -ts recent | grep -c denied || echo "no new denial"
```

```text
Relabeled /var/www/html/report.html from system_u:object_r:admin_home_t:s0 to system_u:object_r:httpd_sys_content_t:s0
system_u:object_r:httpd_sys_content_t:s0 /var/www/html/report.html
200
no new denial
```

> 📝 **시험 포인트**: "파일의 SELinux 컨텍스트를 정책 기본값으로 복원" = `restorecon -Rv <경로>` (필기 FULL r09-58, r10-59). `restorecon -Rv -n` 은 **적용 전 점검**용 — 실기 서술형에서 "적용 전에 무엇이 바뀌는지 확인하는 옵션" 으로 물을 수 있음.

### 4-9. `cp` vs `mv` vs `cp -a` — 컨텍스트 상속 차이

> **상황**: 같은 파일을 세 가지 방법으로 웹 루트에 넣어 라벨이 어떻게 달라지는지 한 화면에서 비교한다. 실무 사고의 90% 가 이 표에 들어 있다.

```bash
echo test > /root/a.html; echo test > /root/b.html; echo test > /root/c.html
cp    /root/a.html /var/www/html/     # 대상 디렉터리 기본 타입 상속
cp -a /root/b.html /var/www/html/     # 원본 컨텍스트까지 보존
mv    /root/c.html /var/www/html/     # 원본 컨텍스트 유지
ls -Z /var/www/html/{a,b,c}.html
for f in a b c; do
  printf '%s -> ' $f
  curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1/$f.html
done
```

- `cp` (옵션 없음) : 새 inode 생성 → **대상 디렉터리의 기본 타입**을 상속 (정상 동작)
- `cp -a` : `-dR --preserve=all` — 소유자·시각·ACL·**SELinux 컨텍스트까지 보존** (**a**rchive)
- `cp --preserve=context` : 컨텍스트만 보존
- `cp -Z` / `cp --context=<컨텍스트>` : 대상 컨텍스트를 명시 지정
- `mv` : 같은 파일시스템이면 inode 유지 → **컨텍스트 그대로**. 다른 파일시스템이면 복사+삭제라 기본 타입 상속

| 명령 | 결과 컨텍스트 | HTTP | 정리 |
| --- | --- | --- | --- |
| `cp /root/a.html /var/www/html/` | `httpd_sys_content_t` (대상 디렉터리 상속) | 200 | **안전** |
| `cp -a /root/b.html /var/www/html/` | `admin_home_t` (원본 보존) | 403 | 백업 복원 시 사고 |
| `mv /root/c.html /var/www/html/` | `admin_home_t` (원본 유지) | 403 | 가장 흔한 사고 |
| 위 실패분 + `restorecon -Rv /var/www/html` | `httpd_sys_content_t` | 200 | 표준 해결 |
| `tar` 로 풀 때 `--selinux` 없이 | 대상 디렉터리 기본 타입 | 200 | `tar --selinux` 쓰면 보존 |

**검증**

```bash
ls -Z /var/www/html/{a,b,c}.html | awk '{print $1, $2}'
restorecon -Rv /var/www/html
for f in a b c; do curl -s -o /dev/null -w "$f=%{http_code} " http://127.0.0.1/$f.html; done; echo
rm -f /var/www/html/{a,b,c}.html /var/www/html/report.html
```

```text
system_u:object_r:httpd_sys_content_t:s0 /var/www/html/a.html
unconfined_u:object_r:admin_home_t:s0 /var/www/html/b.html
unconfined_u:object_r:admin_home_t:s0 /var/www/html/c.html
Relabeled /var/www/html/b.html from unconfined_u:object_r:admin_home_t:s0 to system_u:object_r:httpd_sys_content_t:s0
Relabeled /var/www/html/c.html from unconfined_u:object_r:admin_home_t:s0 to system_u:object_r:httpd_sys_content_t:s0
a=200 b=200 c=200
```

> 📝 **시험 포인트**: "복사는 상속, 이동은 유지" 를 한 문장으로 답할 수 있어야 함. 백업 복원(`tar -xpf`, `rsync -a`, `cp -a`) 후 서비스가 403 을 내면 1순위 의심은 컨텍스트 → `restorecon -R`.

### 4-10. `chcon`(일시) vs `semanage fcontext`(영구)

> **상황**: 새 웹 경로 `/srv/www/report` 를 만든다. 먼저 `chcon` 으로 급히 붙였다가 `restorecon` 한 번에 날아가는 것을 재현하고, 그 다음 `semanage fcontext` 로 정석대로 등록한다.

```bash
mkdir -p /srv/www/report
echo '<h1>report</h1>' > /srv/www/report/index.html
ls -Zd /srv/www/report                              # 상속받은 기본 타입 확인

# (1) 일시 — chcon
chcon -R -t httpd_sys_content_t /srv/www/report
ls -Zd /srv/www/report
restorecon -Rv /srv/www/report                      # ← 여기서 원복됨
ls -Zd /srv/www/report
```

- `chcon` : 컨텍스트 **직접 변경** (**ch**ange **con**text) — 정책 DB 에는 기록되지 않음
- `-t <타입>` : type 필드만 변경 (**t**ype). `-u` user, `-r` role, `-l` level
- `-R` : 재귀 (**R**ecursive)
- `--reference=<파일>` : 다른 파일의 컨텍스트를 그대로 복사

```bash
# (2) 영구 — semanage fcontext + restorecon
dnf install -y policycoreutils-python-utils         # semanage 제공
semanage fcontext -a -t httpd_sys_content_t '/srv/www/report(/.*)?'
semanage fcontext -l | grep '/srv/www'
restorecon -Rv /srv/www/report                      # ← 이번엔 정책값이 곧 새 타입
ls -Zd /srv/www/report
```

- `semanage fcontext` : 파일 컨텍스트 규칙을 **정책 DB 에 영구 등록**
- `-a` : 규칙 추가 (**a**dd)
- `-t <타입>` : 부여할 타입
- `'<정규식>'` : 경로 정규식. `(/.*)?` = "이 디렉터리 자신과 그 아래 전부" 관용구 — 셸 확장을 막기 위해 **반드시 따옴표**
- `-l` : 등록된 규칙 목록 (**l**ist). `-C` 는 로컬 커스터마이즈만
- `-m` : 기존 규칙 수정 (**m**odify), `-d` : 삭제 (**d**elete)
- `-e <기존경로> <새경로>` : **동등 경로**(equivalence) — 새 경로가 기존 경로의 라벨 규칙을 통째로 따름
- **등록만으로는 기존 파일에 적용되지 않음** → 반드시 `restorecon -Rv` 로 실제 파일에 반영

**검증**

```bash
semanage fcontext -l -C                       # 로컬에 추가한 규칙만
matchpathcon /srv/www/report/index.html       # 정책상 "이 경로에 붙어야 할" 컨텍스트
ls -Z /srv/www/report/index.html
grep -R 'srv/www' /etc/selinux/targeted/contexts/files/file_contexts.local
```

```text
SELinux fcontext                        type       Context
/srv/www/report(/.*)?                   all files  system_u:object_r:httpd_sys_content_t:s0
/srv/www/report/index.html	system_u:object_r:httpd_sys_content_t:s0
system_u:object_r:httpd_sys_content_t:s0 /srv/www/report/index.html
/srv/www/report(/.*)?    system_u:object_r:httpd_sys_content_t:s0
```

> 📝 **시험 포인트**: 필기 FULL r03-56·r06-58·r10-59 의 정답 축은 항상 같음 — ① `chcon` 은 **재라벨링 시 원복**되는 일시 변경 ② 영구 반영은 `semanage fcontext -a -t <type> "<경로>(/.*)?"` **후 `restorecon -Rv`** ③ `semanage fcontext -a` 만으로는 기존 파일에 즉시 적용되지 않음.

### 4-11. 규칙 관리 — 목록·삭제·동등 경로·`matchpathcon`

> **상황**: 등록한 규칙을 조회·삭제하고, 경로가 여러 개일 때 규칙을 복제하지 않는 방법(동등 경로)을 익힌다.

```bash
semanage fcontext -l | grep -E '/var/www|/srv/www' | head
semanage fcontext -l -C                                  # 로컬 추가분만 (-C = Customized)
matchpathcon -V /srv/www/report/index.html               # 실제 라벨과 정책 기대값 비교(-V = verify)

# 동등 경로: /srv/www 는 /var/www 와 같은 규칙을 따르게
semanage fcontext -a -e /var/www /srv/www
semanage fcontext -l | grep -A1 'Equivalence'
restorecon -Rv -n /srv/www | head                        # dry-run 으로 영향 범위 확인
restorecon -Rv /srv/www

# 정리: 실습용 개별 규칙 삭제
semanage fcontext -d '/srv/www/report(/.*)?'
semanage fcontext -l -C
```

- `-e <src> <dst>` : dst 를 src 와 **동등**하게 등록 — 이후 `/srv/www/...` 는 `/var/www/...` 규칙을 그대로 적용받음. 경로별 규칙을 일일이 복제할 필요가 없어짐
- `matchpathcon <경로>` : 정책이 그 경로에 부여할 컨텍스트를 계산해 출력. `-V` 는 현재 라벨과 비교해 `verified`/`should be` 표시
- `restorecon -Rv -n` : **dry-run** — 대량 경로에 적용 전 필수
- 로컬 규칙 저장 위치 : `/etc/selinux/targeted/contexts/files/file_contexts.local`
- 배포판 기본 규칙 : `/etc/selinux/targeted/contexts/files/file_contexts` (직접 편집 금지 — `semanage` 로만)

**검증**

```bash
semanage fcontext -l -C
matchpathcon -V /srv/www/intranet/index.html
ls -Zd /srv/www /srv/www/intranet
```

```text
SELinux Local fcontext Equivalence

/srv/www = /var/www
/srv/www/intranet/index.html verified.
system_u:object_r:httpd_sys_content_t:s0 /srv/www
system_u:object_r:httpd_sys_content_t:s0 /srv/www/intranet
```

> 📝 **시험 포인트**: `file_contexts`(정책 제공) vs `file_contexts.local`(관리자 추가) 구분. `matchpathcon` 은 `libselinux-utils` 제공이며 신규 도구는 `selabel_lookup` — 시험은 여전히 `matchpathcon` 기준으로 출제.

### 4-12. 포트 라벨 — httpd 를 8888 로 띄우기

> **상황**: 비표준 포트를 쓰면 "설정도 맞고 방화벽도 열었는데 데몬이 뜨지 않는" 증상이 난다. `Listen 8888` 을 넣어 실패를 재현하고 포트 라벨로 해결한다.

```bash
semanage port -l | grep -E '^http_port_t|^ssh_port_t'
semanage port -l | grep -w 8888 || echo "8888 unlabeled"
cp -a /etc/httpd/conf/httpd.conf /root/httpd.conf.bak
sed -i '/^Listen 80$/a Listen 8888' /etc/httpd/conf/httpd.conf
apachectl configtest
systemctl restart httpd            # ← 실패
systemctl status httpd --no-pager -l | tail -12
ausearch -m avc -ts recent | grep -E 'name_bind|tcp_socket' | tail -3
```

- `semanage port -l` : 포트 ↔ 타입 매핑 목록. `-C` 로 로컬 추가분만
- 기본적으로 `http_port_t` 는 80·443·488·8008·8009·8443·9000 등을 포함(배포판별 상이) — 8888 은 미포함
- 실패 메시지 : `(13)Permission denied: AH00072: make_sock: could not bind to address [::]:8888` + AVC `name_bind`

```bash
semanage port -a -t http_port_t -p tcp 8888     # 라벨 추가
semanage port -l -C
systemctl restart httpd                          # ← 이제 성공
ss -tlnp | grep 8888
firewall-cmd --permanent --zone=internal --add-port=8888/tcp && firewall-cmd --reload
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8888/
```

- `semanage port -a -t <타입> -p tcp|udp <포트>` : 추가 (**a**dd)
- `-m` : 이미 **다른 타입에 정의된** 포트를 옮길 때 (수정, **m**odify) — `-a` 로 시도하면 `already defined` 오류
- `-d` : 삭제 (**d**elete) — 로컬 추가분만 삭제 가능
- `-p` : 프로토콜 (**p**rotocol)

**검증**

```bash
semanage port -l | grep -w 8888
ss -tlnp | grep -E ':8888'
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8888/
# 정리
sed -i '/^Listen 8888$/d' /etc/httpd/conf/httpd.conf
semanage port -d -t http_port_t -p tcp 8888
firewall-cmd --permanent --zone=internal --remove-port=8888/tcp; firewall-cmd --reload
systemctl restart httpd; ss -tlnp | grep -c 8888 || echo "8888 closed"
```

```text
http_port_t                    tcp      8888, 80, 81, 443, 488, 8008, 8009, 8443, 9000
LISTEN 0 511 *:8888 *:* users:(("httpd",pid=...,fd=...))
200
8888 closed
```

> 📝 **시험 포인트**: "서비스는 설정했는데 바인딩이 거부된다" → **포트 라벨 누락**. `semanage port -a -t http_port_t -p tcp <포트>` 는 실기 빈칸 유형. SSH 포트를 2222 로 바꿀 때도 같은 원리(`ssh_port_t`) — Part 08 에서 2222 를 이미 등록했으므로 `semanage port -l | grep ssh_port_t` 로 확인.

### 4-13. 불린(boolean) — 정책을 재컴파일 없이 토글

> **상황**: 타입은 맞는데도 막히는 경우가 있다. "httpd 가 외부로 접속", "사용자 홈을 웹으로 노출" 같은 **선택적 동작**은 불린으로 켜고 끈다.

```bash
getsebool -a | wc -l                              # 전체 불린 개수
getsebool -a | grep httpd | head -20
getsebool httpd_can_network_connect httpd_enable_homedirs
semanage boolean -l | grep -E 'httpd_can_network_connect|samba_enable_home_dirs'
semanage boolean -l -C                            # 기본값과 다른 것만
```

- `getsebool -a` : 전체 불린과 on/off (**a**ll)
- `semanage boolean -l` : 불린 + **현재값(기본값)** + 설명 3열 → `getsebool` 보다 정보량 많음
- `-C` : 기본값에서 변경된 것만 (**C**ustomized) — 인수인계·점검 시 1순위 확인 대상

```bash
setsebool httpd_can_network_connect on            # 일시 (재부팅 시 원복)
getsebool httpd_can_network_connect
setsebool -P httpd_can_network_connect on         # 영구 (정책 DB 기록, 수 초 소요)
semanage boolean -l -C
```

- `setsebool <불린> on|off|1|0` : 토글
- `-P` : **영구** (**P**ersistent) — 이게 빠지면 재부팅 시 원복. **최빈출 함정**

| 불린 | 켜면 허용되는 것 | 쓰는 상황 |
| --- | --- | --- |
| `httpd_can_network_connect` | httpd 가 임의 포트로 외부 접속 | 리버스 프록시, 외부 API 호출 |
| `httpd_can_network_connect_db` | httpd → DB 포트 접속 | 웹앱 + 원격 DB |
| `httpd_enable_homedirs` | httpd 가 `~user/public_html` 읽기 | UserDir 사용 |
| `httpd_use_nfs` | httpd 가 NFS 마운트 경로 읽기 | 웹 문서를 NFS 에 둘 때 |
| `httpd_unified` | httpd 관련 타입 통합 접근 | (레거시) |
| `samba_enable_home_dirs` | smbd 가 사용자 홈 공유 | `[homes]` 섹션 |
| `samba_export_all_rw` | smbd 가 임의 경로 읽기·쓰기 | 라벨을 못 붙이는 공유 |
| `ftpd_full_access` | vsftpd 가 전체 파일시스템 읽기·쓰기 | ⚠️ 매우 광범위 — 상시 사용 금지 |
| `ftpd_anon_write` | FTP 익명 업로드 | `public_content_rw_t` 와 함께 |
| `nfs_export_all_rw` | NFS 로 임의 경로 rw 내보내기 | `/etc/exports` 경로 라벨 미정 시 |
| `use_nfs_home_dirs` | 홈이 NFS 일 때 로그인 허용 | NIS/LDAP + NFS 홈 |

**검증**

```bash
getsebool httpd_can_network_connect
semanage boolean -l -C
# 웹에서 외부 연결이 필요한지 실제로 테스트할 게 없으면 원복
setsebool -P httpd_can_network_connect off
semanage boolean -l -C | wc -l
```

```text
httpd_can_network_connect --> on
SELinux boolean                State  Default Description
httpd_can_network_connect      (on   ,  off)  Allow httpd to can network connect
0
```

> 📝 **시험 포인트**: 필기 FULL r03-57·r09-59 는 "불린을 **재부팅 후에도 유지**되게" → `setsebool -P <불린> on` 이 정답. `getsebool` 은 조회, `semanage boolean -l` 은 목록+설명. `-P` 없는 `setsebool` 을 정답으로 고르면 오답.

### 4-14. 정책 모듈 — `audit2allow -M` 과 그 위험

> **상황**: 컨텍스트·포트·불린 어디에도 해당하지 않는 진짜 예외에만 커스텀 모듈을 만든다. 만드는 법과 **왜 함부로 쓰면 안 되는지**를 함께 남긴다.

⚠️ `audit2allow -M` 은 "거부된 것을 전부 허용" 하는 규칙을 생성한다. 공격으로 발생한 거부까지 영구 허용할 수 있으므로 **원인 파악(4-7) 을 건너뛰고 쓰면 안 됨**

```bash
semodule -l | head                                  # 로드된 정책 모듈 목록
semodule -l | wc -l
# 생성 절차 (참고 — 이번 실습에서는 restorecon 으로 이미 해결됨)
mkdir -p /root/selinux-mod && cd /root/selinux-mod
ausearch -c httpd --raw | audit2allow -w            # ① 반드시 먼저 "왜" 를 읽는다
ausearch -c httpd --raw | audit2allow -M myhttpd    # ② .te(소스) + .pp(바이너리) 생성
cat myhttpd.te
# semodule -i myhttpd.pp                            # ③ 설치 (이번 실습에서는 미실행)
```

- `semodule -l` : 설치된 모듈 목록 (**l**ist)
- `semodule -i <모듈.pp>` : 설치 (**i**nstall)
- `semodule -r <모듈명>` : 제거 (**r**emove) — 확장자 없이 이름만
- `semodule -d <모듈명>` / `-e` : 비활성화(**d**isable) / 활성화(**e**nable)
- `semodule -B` : 정책 재빌드 (**B**uild)
- `audit2allow -M <이름>` : `<이름>.te`(사람이 읽는 규칙 소스) 와 `<이름>.pp`(로드용 패키지) 생성
- 판단 순서: **① 컨텍스트 오류인가(`restorecon`) → ② 포트 라벨인가(`semanage port`) → ③ 불린으로 되는가(`setsebool -P`) → ④ 그래도 안 되면 모듈**

**검증**

```bash
semodule -l | wc -l
ls /root/selinux-mod/
cat /root/selinux-mod/myhttpd.te 2>/dev/null | head
semodule -l | grep -c myhttpd || echo "module not installed (intended)"
```

```text
...
myhttpd.pp  myhttpd.te
module myhttpd 1.0;
require {
	type admin_home_t;
	type httpd_t;
	class file { getattr open read };
}
#============= httpd_t ==============
allow httpd_t admin_home_t:file { getattr open read };
module not installed (intended)
```

> 📝 **시험 포인트**: `audit2allow` 는 "거부 로그로부터 정책 모듈을 만드는 도구" 로 개념 출제. 실무 판단은 **컨텍스트 → 포트 → 불린 → 모듈** 순서. `.te` 안의 `allow <도메인> <타입>:<클래스> { 권한 };` 문법을 읽을 수 있으면 충분.

### 4-15. MAC vs DAC 와 사용자 매핑 — 개념 정리

> **상황**: 필기 서술·비교 문항 대비. 지금까지 실습한 것을 DAC 과 나란히 놓고 정리하고, SELinux 사용자 매핑 조회 명령까지 확인한다.

```bash
semanage login -l          # 리눅스 계정 ↔ SELinux 사용자 매핑
semanage user -l           # SELinux 사용자 ↔ 역할·레벨
id -Z; su - dev1 -c 'id -Z'
runcon -t httpd_t -- id -Z 2>/dev/null || echo "runcon: 전이 권한 필요 (참고)"
```

- `semanage login -l` : `__default__` → `unconfined_u` 처럼 로그인 계정을 SELinux 사용자에 매핑. `-a -s staff_u dev1` 로 특정 계정을 제한 사용자로 묶을 수 있음
- `semanage user -l` : SELinux 사용자별 허용 역할·MLS 범위
- `runcon -t <타입> -- <명령>` : 지정 도메인으로 명령 실행 (**run** with **con**text) — 정책이 전이를 허용해야 성공

| 구분 | DAC (임의 접근 제어) | MAC (강제 접근 제어) |
| --- | --- | --- |
| 판단 기준 | 소유자·그룹·기타 `rwx`, ACL | 보안 컨텍스트(type) + 정책 규칙 |
| 권한 부여 주체 | **파일 소유자**가 임의로 | **시스템 정책** (관리자·정책 작성자) |
| root 취급 | root 는 전권 | root 도 정책에 종속(`unconfined` 예외 제외) |
| 우회 가능성 | 소유자가 `chmod 777` 하면 개방 | 소유자가 바꿀 수 없음 |
| 검사 순서 | **먼저** 검사 | DAC 통과 후 **추가로** 검사 (이중 검사) |
| 대표 구현 | 전통 유닉스 퍼미션 | **SELinux**, AppArmor |
| 확인 명령 | `ls -l`, `getfacl` | `ls -Z`, `ps -eZ`, `id -Z` |

**검증**

```bash
semanage login -l
getenforce
sestatus | grep -E 'Policy from config|Current mode'
```

```text
Login Name           SELinux User         MLS/MCS Range        Service
__default__          unconfined_u         s0-s0:c0.c1023       *
root                 unconfined_u         s0-s0:c0.c1023       *
Enforcing
Current mode:                   enforcing
```

> 📝 **시험 포인트**: "DAC 을 통과해도 MAC 에서 거부되면 최종 차단" 이 이중 검사 서술의 핵심. SELinux 는 **NSA 가 개발한 커널 수준 MAC 구현**, 목표는 **최소 권한(least privilege)**.

---

## 5. auditd — 감사 데몬과 감사 규칙

### 5-1. auditd 상태 확인과 `systemctl stop` 이 막히는 이유

> **상황**: 4절의 AVC 로그를 남긴 주체가 auditd 다. 이 데몬은 systemd 로 **중지할 수 없게** 만들어져 있다 — 감사 로그가 임의로 끊기면 안 되기 때문이다. 시험에서 자주 묻는 특이점이다.

```bash
systemctl status auditd --no-pager | head -8
systemctl is-active auditd
systemctl stop auditd            # ← 거부됨
grep -E 'RefuseManual' /usr/lib/systemd/system/auditd.service
ls /usr/libexec/initscripts/legacy-actions/auditd/ 2>/dev/null
```

- auditd.service 에 `RefuseManualStop=yes` 가 설정돼 있어 `systemctl stop|restart auditd` 는 거부
- 실제 중지·재시작은 레거시 액션 경유: **`service auditd stop|restart|reload`**
- `systemctl status`·`is-active`·`enable`·`disable` 은 정상 동작

**검증**

```bash
systemctl stop auditd; echo "exit=$?"
systemctl is-active auditd
service auditd reload; echo "exit=$?"
systemctl is-active auditd
```

```text
Failed to stop auditd.service: Operation refused, unit auditd.service may be requested by dependency only (it is configured to refuse manual start/stop).
exit=1
active
Redirecting to /bin/systemctl reload auditd.service
exit=0
active
```

> 📝 **시험 포인트**: "auditd 는 `systemctl stop` 이 거부되고 `service auditd stop` 을 써야 한다" 는 RHEL 계열 고유 특이점. `service` 명령이 `systemctl` 로 리다이렉트되는 다른 서비스(Part 07 3-7) 와 구분.

### 5-2. `/etc/audit/auditd.conf` — 로그 크기·보관·디스크 정책

> **상황**: 감사 로그가 디스크를 채우면 시스템이 멈추도록 설정될 수도 있다. 기본값을 읽고 실습 VM 에 맞는 값인지 확인한다.

```bash
grep -vE '^\s*#|^\s*$' /etc/audit/auditd.conf
ls -lh /var/log/audit/
du -sh /var/log/audit/
```

| 키 | 기본값(예) | 의미 |
| --- | --- | --- |
| `log_file` | `/var/log/audit/audit.log` | 감사 로그 경로 |
| `log_format` | `ENRICHED` | `RAW`(원본) / `ENRICHED`(UID·이름 해석 정보 동봉) |
| `max_log_file` | `8` | 파일 하나의 최대 크기 (**MB**) |
| `num_logs` | `5` | 보관할 순환 파일 개수 (`audit.log.1` … `.4`) |
| `max_log_file_action` | `ROTATE` | 최대 크기 도달 시 동작 — `IGNORE`/`SYSLOG`/`SUSPEND`/`ROTATE`/`KEEP_LOGS` |
| `space_left` | `75` | 남은 디스크 여유가 이 값(MB) 이하일 때 |
| `space_left_action` | `SYSLOG` | 그때 취할 동작 — `EMAIL`/`EXEC`/`SUSPEND`/`SINGLE`/`HALT` |
| `admin_space_left` / `_action` | `50` / `SUSPEND` | 더 위험한 임계치와 동작 |
| `disk_full_action` | `SUSPEND` | 디스크가 꽉 찼을 때 |
| `flush` / `freq` | `INCREMENTAL_ASYNC` / `50` | 디스크 동기화 방식·주기 |

- `SUSPEND` : 감사 기록만 중단(시스템은 계속), `SINGLE` : 단일 사용자 모드 진입, `HALT` : 시스템 정지 — 운영에서 `HALT` 오설정이 사고로 이어짐

**검증**

```bash
grep -E '^(log_file|max_log_file|num_logs|max_log_file_action|space_left|space_left_action)' /etc/audit/auditd.conf
ls /var/log/audit/
```

```text
log_file = /var/log/audit/audit.log
max_log_file = 8
max_log_file_action = ROTATE
num_logs = 5
space_left = 75
space_left_action = SYSLOG
audit.log
```

> 📝 **시험 포인트**: `max_log_file` 단위는 **MB**, `num_logs` 는 **개수**. 감사 로그는 rsyslog·logrotate 가 아니라 **auditd 자신**이 순환시킨다(Part 07 logrotate 와 구분).

### 5-3. 감시 규칙 추가 → `useradd` 로 재현 → `ausearch -k`

> **상황**: "누가 `/etc/passwd` 를 건드렸는가" 를 추적할 수 있게 감시 규칙(watch) 을 걸고, 실제로 계정을 하나 만들어 로그에 잡히는지 확인한 뒤 정리한다.

```bash
auditctl -l                                        # 현재 규칙 (기본은 "No rules")
auditctl -s                                        # 상태 (enabled, pid, backlog, lost)
auditctl -w /etc/passwd -p wa -k passwd_change
auditctl -w /etc/shadow -p wa -k shadow_watch
auditctl -w /etc/sudoers -p wa -k sudoers_watch
auditctl -l
```

- `-w <경로>` : 파일·디렉터리 **감시(watch)** 추가 (**w**atch)
- `-p <권한>` : 감시할 접근 종류 — `r`(읽기) `w`(쓰기) `x`(실행) `a`(속성 변경, **a**ttribute) 조합. `wa` 가 변조 탐지 표준
- `-k <키>` : 규칙에 태그(**k**ey) 부여 → `ausearch -k` 로 한 번에 검색
- `-W <경로>` : 감시 제거, `-D` : **전체 규칙 삭제**, `-l` : 목록, `-s` : 상태
- `-e 0|1|2` : 감사 활성(0=끔, 1=켬, 2=**잠금** — 재부팅 전까지 규칙 변경 불가)
- `auditctl` 로 추가한 규칙은 **재부팅 시 사라짐** (영구화는 5-4)

```bash
useradd -M -s /sbin/nologin tmpx          # 감사 대상 이벤트 발생
ausearch -k passwd_change -i | tail -25
ausearch -k passwd_change -i | grep -E 'type=SYSCALL|type=PATH' | tail -4
userdel -r tmpx 2>/dev/null; userdel tmpx 2>/dev/null   # 정리
ausearch -k passwd_change -i | grep -c 'type=SYSCALL'
```

- `ausearch -k <키>` : 키로 검색 (**k**ey)
- `-i` : 숫자를 이름으로 해석 (**i**nterpret) — `uid=0` → `root`, `syscall=257` → `openat`

**검증**

```bash
auditctl -l
ausearch -k passwd_change -i -ts recent | grep -E 'comm=|exe=|auid=' | tail -3
getent passwd tmpx || echo "tmpx removed"
```

```text
-w /etc/passwd -p wa -k passwd_change
-w /etc/shadow -p wa -k shadow_watch
-w /etc/sudoers -p wa -k sudoers_watch
... comm="useradd" exe="/usr/sbin/useradd" ... auid=root ...
... comm="userdel" exe="/usr/sbin/userdel" ...
tmpx removed
```

> 📝 **시험 포인트**: 필기 FULL r03-64·r09-61 — `auditctl -w /etc/shadow -p wa -k shadow_watch` = "쓰기·속성 변경을 감사 기록으로 남기고 `ausearch -k shadow_watch` 로 조회". 같은 문항의 오답 축은 "**auditctl 로 추가한 규칙이 재부팅 후에도 유지된다**"(틀림) 와 "`ausearch` 가 규칙을 추가한다"(틀림).

### 5-4. 시스템콜 규칙과 영구 규칙 파일 — `augenrules --load`

> **상황**: 파일 감시 외에 "모든 실행(execve) 을 기록" 같은 시스템콜 규칙도 있다. 규칙을 파일로 옮겨 재부팅 후에도 살아남게 만든다.

```bash
# 시스템콜 규칙 (참고 — 로그량이 매우 많음)
auditctl -a always,exit -F arch=b64 -S execve -k exec_trace
auditctl -l | grep exec_trace
auditctl -d always,exit -F arch=b64 -S execve -k exec_trace   # 곧바로 제거
```

- `-a <목록>,<동작>` : 규칙 추가 (**a**ppend) — 목록 `task`/`exit`/`user`/`exclude`, 동작 `always`(기록)/`never`(무시)
- `-A` : 목록 **맨 앞**에 추가, `-d` : 같은 조건의 규칙 삭제 (**d**elete)
- `-F <필드>=<값>` : 필터 (**F**ield) — `arch=b64`, `auid>=1000`, `auid!=-1`(=unset), `euid=0`, `path=`, `dir=`, `exit=-EACCES`
- `-S <시스템콜>` : 감시할 시스템콜 (**S**yscall) — `execve`, `openat`, `unlink`, `chmod`, 여러 개는 `-S a -S b`
- `arch=b64` : 64비트 ABI. 32/64 혼용 시스템은 `b32` 규칙도 함께 넣어야 우회 불가

```bash
# 영구화: rules.d 에 파일로 작성 → augenrules 로 컴파일·적용
cat > /etc/audit/rules.d/50-lab.rules <<'EOF'
# 계정·권한 파일 변조 감시
-w /etc/passwd  -p wa -k passwd_change
-w /etc/shadow  -p wa -k shadow_watch
-w /etc/group   -p wa -k group_change
-w /etc/sudoers -p wa -k sudoers_watch
-w /etc/sudoers.d/ -p wa -k sudoers_watch
# sshd 설정 변조 감시
-w /etc/ssh/sshd_config -p wa -k sshd_config
# 권한 상승 실행 추적
-w /usr/bin/sudo -p x -k priv_exec
EOF
chmod 600 /etc/audit/rules.d/50-lab.rules
augenrules --check
augenrules --load
auditctl -l
```

- `/etc/audit/rules.d/*.rules` : 조각 파일 (숫자 접두사 순서대로 병합)
- `augenrules --load` : 조각들을 `/etc/audit/audit.rules` 로 **합쳐서 즉시 로드** (**a**udit **gen**erate **rules**)
- `augenrules --check` : 병합 결과가 현재 규칙과 다른지 확인
- 부팅 시에는 `auditd` 가 `/etc/audit/audit.rules` 를 읽어 자동 적용

**검증**

```bash
ls /etc/audit/rules.d/
auditctl -l | wc -l
grep -c '^-w' /etc/audit/audit.rules
auditctl -s | grep -E 'enabled|backlog_limit|lost'
```

```text
50-lab.rules  audit.rules
7
7
enabled 1
backlog_limit 8192
lost 0
```

> 📝 **시험 포인트**: 일시 = `auditctl`, 영구 = `/etc/audit/rules.d/*.rules` + `augenrules --load`. `auditctl -e 2` 는 규칙을 **잠가** 재부팅 전까지 변경 불가로 만드는 감사 무결성 옵션.

### 5-5. 로그 조회·보고서 — `ausearch` · `aureport` · 필드 해석

> **상황**: 사고 조사 시나리오. 오늘 실패한 로그인, 특정 사용자(auid) 의 행위, 파일 접근 요약을 각각 뽑는다.

```bash
ausearch -m USER_LOGIN --success no -ts today -i | tail -20
ausearch -m USER_AUTH,USER_LOGIN -ts today -i | tail -10
ausearch -ua 2001 -ts today -i | tail -10        # auid 2001 = dev1
ausearch -ui 0 -ts boot -i | wc -l               # uid 0 이벤트 수
ausearch -f /etc/passwd -i | tail -5             # 특정 파일 관련
```

- `-m <타입[,타입]>` : 메시지 타입
- `--success no|yes` : 성공/실패 필터
- `-ua <auid>` : **audit UID**(로그인 원 사용자) — `su`·`sudo` 로 바뀌어도 추적되는 값
- `-ui <uid>` / `-ue <euid>` : 실제/유효 UID
- `-f <파일>` : 파일 경로
- `-p <PID>`, `-x <실행파일>`, `-sv no`(시스템콜 실패만)

```bash
aureport --summary                # 전체 요약
aureport -f --summary             # 파일 접근 요약 (-f = file)
aureport -au --summary            # 인증 시도 요약 (-au = authentication)
aureport -l --summary             # 로그인 요약 (-l = login)
aureport -m                       # 계정 변경(modification) 이벤트
aureport -k                        # 키별 집계
aureport --failed --summary       # 실패 이벤트만
```

- `aureport` : audit.log 를 사람이 읽는 보고서로 — `--summary` 는 건수 집계, 없으면 개별 목록
- 옵션: `-f` 파일 / `-au` 인증 / `-l` 로그인 / `-u` 사용자 / `-p` 프로세스 / `-e` 이벤트 / `-k` 키 / `-x` 실행파일 / `--failed` `--success`
- 기간 지정은 `-ts`/`-te` 를 그대로 사용

**audit.log 레코드 필드 해석**

```text
type=SYSCALL msg=audit(1756900000.123:456): arch=c000003e syscall=257 success=no exit=-13
  a0=ffffff9c a1=7ffd... a2=0 a3=0 items=1 ppid=1234 pid=1250 auid=1000 uid=48 gid=48
  euid=48 suid=48 fsuid=48 egid=48 sgid=48 fsgid=48 tty=(none) ses=3 comm="httpd"
  exe="/usr/sbin/httpd" subj=system_u:system_r:httpd_t:s0 key="passwd_change"
type=AVC msg=audit(1756900000.123:456): avc:  denied  { read } for  pid=1250 comm="httpd"
  name="report.html" dev="dm-0" ino=... scontext=system_u:system_r:httpd_t:s0
  tcontext=unconfined_u:object_r:admin_home_t:s0 tclass=file permissive=0
```

| 필드 | 의미 |
| --- | --- |
| `type=` | 레코드 종류 — `SYSCALL` `AVC` `PATH` `CWD` `USER_LOGIN` `USER_AUTH` `ADD_USER` `CRED_ACQ` |
| `msg=audit(<epoch>.<ms>:<serial>)` | 발생 시각(에폭) 과 **이벤트 일련번호** — 같은 번호끼리 한 사건 |
| `arch=` / `syscall=` | ABI(c000003e=x86_64) 와 시스템콜 번호 (`-i` 로 이름 해석) |
| `success=` / `exit=` | 성공 여부와 반환값 (`-13` = EACCES 권한 거부) |
| `auid=` | **로그인 원 사용자** — `su`/`sudo` 후에도 불변 → 책임 추적의 핵심 |
| `uid=` / `euid=` | 실행 시점 실제/유효 UID |
| `ses=` | 로그인 세션 ID |
| `comm=` / `exe=` | 프로세스 이름 / 실행 파일 경로 |
| `subj=` | 주체(프로세스) 의 SELinux 컨텍스트 |
| `key=` | `-k` 로 붙인 태그 |
| AVC `scontext` / `tcontext` / `tclass` | 소스 컨텍스트 / 대상 컨텍스트 / 객체 클래스 |
| AVC `permissive=` | `0`=Enforcing 에서 실제 차단, `1`=Permissive 라 기록만 |

**검증**

```bash
aureport --summary | head -12
aureport -k | head -8
ausearch -m USER_LOGIN --success no -ts today -i | grep -c 'type=USER_LOGIN' || echo "no failed login yet"
```

```text
Summary Report
======================
Range of time in logs: ...
Number of changes in configuration: ...
Number of authentications: ...
Number of failed authentications: ...
Number of logins: ...
Number of failed logins: ...
Number of AVC's: 2
Key Report
===============================================
# date time key success exe auid event
...
no failed login yet
```

> 📝 **시험 포인트**: `auid` 는 "누가 로그인해서 시작한 일인가" — `sudo su -` 로 root 가 돼도 원 사용자를 추적할 수 있는 필드. AVC 레코드의 `scontext`(주체) / `tcontext`(객체) / `tclass`(객체 종류) 3종 세트는 SELinux 문제 해석의 기본.

---

## 6. 접근 통제·인증 강화

### 6-1. TCP Wrapper — RHEL 9 미지원 확인 (※ 미실행)

> **상황**: 필기에서 매 회차 나오지만 **Rocky/RHEL 9 에는 동작하지 않는다**. 실제로 지원이 빠졌는지 확인한 뒤, 문법과 검사 순서는 필기 대비로 정확히 정리한다.

```bash
ldd /usr/sbin/sshd | grep -c wrap          # 0 = libwrap 링크 없음
ldd /usr/sbin/sshd | grep -ci libwrap
rpm -q tcp_wrappers tcp_wrappers-libs 2>&1 | head -2
ls -l /etc/hosts.allow /etc/hosts.deny 2>/dev/null || echo "파일 없음"
sshd -T 2>/dev/null | grep -ci wrap || echo "sshd 옵션에도 없음"
```

- `ldd <바이너리>` : 동적 링크 라이브러리 목록 (**l**ist **d**ynamic **d**ependencies) — Part 02 참조
- RHEL 8 부터 `tcp_wrappers` 패키지가 제거되고 sshd·vsftpd 등이 `libwrap` 링크 없이 빌드됨
- 따라서 `/etc/hosts.allow`·`/etc/hosts.deny` 를 작성해도 **아무 효과 없음** → 대체 수단은 6-2

**검증**

```bash
ldd /usr/sbin/sshd | grep -c wrap
rpm -qa | grep -c tcp_wrappers
```

```text
0
0
```

**필기 대비 — 문법과 검사 순서**

```text
# 형식:  데몬 목록 : 클라이언트 목록 [ : 옵션 ]

# /etc/hosts.allow
sshd : 192.168.0.                 # 마침표로 끝 = 192.168.0.0/24 프리픽스 일치
sshd : 192.168.64.0/255.255.255.0 # 넷마스크 표기도 가능
in.telnetd : .example.com         # 마침표로 시작 = 도메인 접미 일치
vsftpd : 192.168.0.10 192.168.0.11
ALL : LOCAL                       # 도메인 없는(같은 네트워크) 호스트 전체
sshd : ALL EXCEPT 192.168.0.100   # 한 대만 빼고 전부
sshd : 192.168.0. : spawn (/bin/echo "%c -> %s" >> /var/log/tcpw.log) &

# /etc/hosts.deny
ALL : ALL                         # 나머지 전부 거부
in.telnetd : ALL : twist /bin/echo "Access denied"
```

| 와일드카드 | 의미 |
| --- | --- |
| `ALL` | 모든 데몬 / 모든 클라이언트 |
| `LOCAL` | 이름에 마침표가 없는 호스트 (= 같은 도메인·로컬) |
| `KNOWN` / `UNKNOWN` | 호스트명·주소가 정방향·역방향 해석되는 / 안 되는 호스트 |
| `PARANOID` | 정방향·역방향 조회 결과가 **불일치**하는 호스트 |
| `EXCEPT` | 앞 목록에서 뒤 목록을 제외 (`A EXCEPT B`) |

| 옵션 | 동작 |
| --- | --- |
| `spawn <명령> &` | 접속 시 **별도 프로세스로 명령 실행** 후 원래 판정대로 진행 (로그 기록용) |
| `twist <명령>` | 서비스 대신 **그 명령의 출력을 클라이언트에 돌려주고** 연결 종료 |
| `%c %s %h %a %d %u` | 확장 문자 — 클라이언트 정보 / 서버 정보 / 호스트명 / 주소 / 데몬명 / 사용자명 |

| 순서 | 검사 대상 | 매칭되면 |
| --- | --- | --- |
| 1 | `/etc/hosts.allow` | **즉시 허용** (hosts.deny 는 보지 않음) |
| 2 | `/etc/hosts.deny` | **거부** |
| 3 | 둘 다 매칭 없음 | **허용** (기본 정책이 허용) |

**기출 해석 예 3개**

```text
[예 1] 실기 r06-14 · 필기 FULL r09-43
  /etc/hosts.allow : sshd : 192.168.10.
  /etc/hosts.deny  : ALL : ALL
  → 192.168.10.0/24 에서 오는 sshd 접속만 허용. 그 외 모든 호스트의 모든 서비스는 거부.
    (allow 를 먼저 보므로 "deny 의 ALL:ALL 때문에 전부 막힌다" 는 오답)

[예 2] 필기 FULL r05-91
  /etc/hosts.allow : sshd : 192.168.1.
  → 마침표로 끝나는 표기는 호스트명이 아니라 **프리픽스**. 192.168.1.0/24 대역의 sshd 접근 허용.

[예 3] 필기 FULL r03-59 · r07-62 · r08-61 · r10-60
  질문: 검사 순서와 기본 정책
  → hosts.allow 매칭 시 허용 → 아니면 hosts.deny 매칭 시 거부 → **어느 쪽에도 없으면 허용**.
    "deny 를 먼저 검사" / "둘 다 없으면 거부" 는 전부 오답.
```

> 📝 **시험 포인트**: 순서 **allow → deny → (둘 다 없으면) 허용** 은 매 회차 출제. 실무에서는 RHEL 8/9 에서 **동작하지 않으므로** 시험 답안과 실제 운영을 분리해서 기억할 것.

### 6-2. 대체 수단 — sshd `Match` · `AllowUsers` · firewalld rich rule

> **상황**: TCP Wrapper 가 하던 "호스트 기반 접근 제어" 를 RHEL 9 에서 실제로 구현한다. sshd 자체 기능과 방화벽 rich rule 두 갈래가 있다.

⚠️ sshd 설정 변경 — UTM 콘솔에서 수행. 반영 전 반드시 `sshd -t` 통과 확인, 반영 후 **기존 세션을 끊지 말고** 새 터미널로 접속 테스트

```bash
cp -a /etc/ssh/sshd_config /root/sshd_config.bak.$(date +%F)
cat >> /etc/ssh/sshd_config <<'EOF'

# --- Part 10: 호스트 기반 접근 제어 (TCP Wrapper 대체) ---
AllowGroups wheel devteam
DenyUsers guest1
Match Address 192.168.64.0/24
    PasswordAuthentication yes
Match Address *,!192.168.64.0/24
    PasswordAuthentication no
EOF
sshd -t && echo "syntax OK"
systemctl reload sshd
```

- `AllowUsers <user>[@<호스트패턴>]` : 지정 사용자만 허용 — `AllowUsers admin1@192.168.64.*` 처럼 **사용자@출발지** 조합 가능
- `AllowGroups <그룹…>` : 지정 그룹 소속만 허용 (본 실습은 `wheel`·`devteam`)
- `DenyUsers` / `DenyGroups` : 거부 목록. 평가 순서는 **DenyUsers → AllowUsers → DenyGroups → AllowGroups**
- `Match <조건>` : 이하 지시자를 조건부 적용 — 조건 키워드 `Address`, `User`, `Group`, `Host`, `LocalPort`. **Match 블록은 다음 Match 또는 파일 끝까지** 유효하므로 파일 **맨 끝**에 둘 것
- `*,!<대역>` : 부정 패턴 (`!` = except)
- `sshd -t` : 설정 문법 검사 (**t**est). `-T` 는 최종 적용값 전체 덤프

```bash
# 같은 통제를 방화벽 계층에서 (2-4 rich rule 문법 참조)
firewall-cmd --permanent --zone=internal \
  --add-rich-rule='rule family=ipv4 source address=192.168.64.0/24 port port=2222 protocol=tcp accept'
firewall-cmd --permanent --zone=public \
  --add-rich-rule='rule family=ipv4 port port=2222 protocol=tcp log prefix="SSH-DENY " level=notice limit value=5/m reject'
firewall-cmd --reload
firewall-cmd --list-rich-rules --zone=public
```

**검증**

```bash
sshd -T | grep -iE '^allowgroups|^denyusers'
sshd -t; echo "config exit=$?"
# macOS: ssh -p 2222 admin1@192.168.64.10 'id -nG'      # wheel 소속 → 성공
# UTM 콘솔: su - guest1 후 ssh -p 2222 guest1@192.168.64.10 → 거부 확인
grep -c 'Match Address' /etc/ssh/sshd_config
```

```text
allowgroups wheel devteam
denyusers guest1
config exit=0
wheel admin1
2
```

> 📝 **시험 포인트**: `AllowUsers`/`AllowGroups` 는 **화이트리스트** — 하나라도 쓰면 목록에 없는 계정은 전부 거부. `Match` 블록의 유효 범위(다음 Match 까지) 가 서술형 함정.

### 6-3. sshd 강화 항목 재점검

> **상황**: Part 08 에서 포트 2222·키 인증·`PermitRootLogin no` 까지는 끝냈다. 그 위에 무차별 대입·세션 방치 대응 항목을 얹는다.

⚠️ 콘솔에서 수행. 아래 값 중 `PasswordAuthentication no` 는 **키 인증이 확실히 동작하는지 먼저 확인**한 뒤에만 적용

```bash
cat >> /etc/ssh/sshd_config <<'EOF'

# --- Part 10: 인증·세션 강화 ---
MaxAuthTries 3
MaxSessions 5
LoginGraceTime 30
PermitEmptyPasswords no
X11Forwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
Banner /etc/issue.net
EOF
sshd -t && systemctl reload sshd
sshd -T | grep -iE 'maxauthtries|logingracetime|permitemptypasswords|x11forwarding|clientalive|banner|permitrootlogin|passwordauthentication|^port'
```

| 지시자 | 값 | 효과 |
| --- | --- | --- |
| `Port 2222` | (Part 08) | 기본 22 회피 — 자동화 스캔 감소. `semanage port` 로 `ssh_port_t` 등록 필요 |
| `PermitRootLogin no` | (Part 08) | root 직접 로그인 금지 — 일반 계정 로그인 후 `su`/`sudo` 유도. `prohibit-password` 는 키만 허용 |
| `PubkeyAuthentication yes` | (Part 08) | 공개키 인증 활성 |
| `MaxAuthTries 3` | 신규 | 한 연결에서 인증 시도 **3회** 초과 시 연결 종료 |
| `MaxSessions 5` | 신규 | 한 연결의 동시 세션(다중화) 상한 |
| `LoginGraceTime 30` | 신규 | 로그인 완료까지 **30초** — 미완 연결 점유 공격 완화 |
| `PermitEmptyPasswords no` | 신규 | 빈 비밀번호 계정 로그인 금지 |
| `X11Forwarding no` | 신규 | X11 터널 차단 (X 미설치 환경에서는 불필요 기능 제거) |
| `ClientAliveInterval 300` | 신규 | 300초마다 클라이언트 생존 확인 |
| `ClientAliveCountMax 2` | 신규 | 무응답 2회 → 세션 종료 (**300×2 = 10분** 후 끊김) |
| `Banner /etc/issue.net` | 신규 | 인증 **전** 경고 배너 표시 (6-7) |

**검증**

```bash
sshd -T | grep -iE 'maxauthtries|logingracetime|clientaliveinterval|clientalivecountmax|banner|x11forwarding|permitemptypasswords'
systemctl is-active sshd
# macOS: ssh -p 2222 admin1@192.168.64.10 'echo still-ok'
```

```text
maxauthtries 3
logingracetime 30
clientaliveinterval 300
clientalivecountmax 2
banner /etc/issue.net
x11forwarding no
permitemptypasswords no
active
still-ok
```

> 📝 **시험 포인트**: 필기 FULL r02-65·r05-88·r06-83·r09-85·r10-64 는 `sshd_config` 블록을 주고 종합 효과를 묻는다. 핵심 조합 — **`PermitRootLogin no` + `PasswordAuthentication no` + `PubkeyAuthentication yes`** = "root 직접 로그인 불가, 비밀번호 인증 불가, 키 인증만 가능". `ClientAliveInterval × ClientAliveCountMax` 가 실제 타임아웃.

### 6-4. 실패 로그인 유도와 로그 집계

> **상황**: 무차별 대입이 로그에 어떻게 남는지 직접 만들어 본다. 일부러 3~5회 틀린 비밀번호로 접속을 시도한 뒤 세 가지 경로로 집계한다.

```bash
# UTM 콘솔 또는 macOS 에서 일부러 실패시키기 (존재하지 않는 계정 + 틀린 비번)
# $ ssh -p 2222 nosuchuser@192.168.64.10        ← 3회 반복
# $ ssh -p 2222 dev1@192.168.64.10              ← 틀린 비번 3회

lastb | head -10                                    # /var/log/btmp (바이너리)
lastb -n 5 -a
journalctl -u sshd --since today | grep -iE 'failed|invalid' | tail -10
grep 'Failed password' /var/log/secure | wc -l
grep -c 'Invalid user' /var/log/secure
```

- `lastb` : `/var/log/btmp` 의 **로그인 실패** 이력 (root 전용). `last` 는 `/var/log/wtmp` 의 성공 이력
- `-n <개수>` : 최근 N건, `-a` : 호스트명을 마지막 열에, `-f <파일>` : 다른 파일 지정
- `journalctl -u sshd` : sshd 유닛 저널 (Part 07 5-1)
- `/var/log/secure` : rsyslog `authpriv.*` 규칙으로 기록되는 인증 로그

```bash
# 공격 출발지 IP 집계 — 실기 서술형 단골
grep 'Failed password' /var/log/secure | grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' \
  | sort | uniq -c | sort -rn | head
# 대상 계정 집계
grep 'Failed password' /var/log/secure | awk '{for(i=1;i<=NF;i++) if($i=="for"){print $(i+1)}}' \
  | sort | uniq -c | sort -rn | head
# 시간대별 분포
grep 'Failed password' /var/log/secure | awk '{print $1, $2, substr($3,1,2)"시"}' | uniq -c
```

- `grep -oE` : 매칭된 부분만 출력 (**o**nly-matching) + 확장 정규식 (**E**)
- `sort | uniq -c | sort -rn` : 빈도 집계 관용구 — `-c` 중복 개수, `-r` 역순, `-n` 숫자 정렬

**검증**

```bash
lastb | wc -l
grep -c 'Failed password' /var/log/secure
grep 'Failed password' /var/log/secure | grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' | sort -u
aureport -au --summary
```

```text
7
6
192.168.64.1
Authentication Attempt Summary Report
=====================================
total  acct
=====================================
6  dev1
```

> 📝 **시험 포인트**: 실기 r03-1·r05-1 이 그대로 `grep -c 'Failed password' /var/log/secure`. 필기 FULL r04-56·r05-63·r06-60·r07-63·r09-60 은 전부 "`/var/log/btmp` 조회 = **`lastb`**" — `last`(wtmp)·`lastlog`(lastlog) 와 혼동 금지.

### 6-5. pam_faillock — 실패 횟수로 계정 잠그기

> **상황**: sshd 의 `MaxAuthTries` 는 **한 연결 안**의 시도만 제한한다. 연결을 새로 맺으며 반복하는 공격은 PAM 계층에서 막아야 한다. RHEL 9 에서는 `pam_tally2` 가 제거되고 `pam_faillock` 만 남았다.

```bash
authselect current                                  # 현재 프로파일과 활성 기능
authselect list-features sssd 2>/dev/null | head
rpm -q pam | head -1
ls /usr/lib64/security/pam_tally2.so 2>/dev/null || echo "pam_tally2: RHEL 9 에서 제거됨"
```

- `authselect` : `/etc/pam.d/*` 를 **직접 편집하지 않고** 프로파일 단위로 관리하는 RHEL 8+ 도구
- `authselect current` : 현재 프로파일(`sssd`/`minimal`/`winbind`) 과 켜진 기능 목록
- `/etc/pam.d/*` 를 손으로 고치면 authselect 가 덮어쓰므로 **기능 토글로 처리**

```bash
cp -a /etc/security/faillock.conf /root/faillock.conf.bak
cat > /etc/security/faillock.conf <<'EOF'
# 실패 3회 → 600초 잠금
deny = 3
unlock_time = 600
fail_interval = 900
# root 도 잠금 대상 (콘솔 복구 경로 확보 후에만 켤 것)
even_deny_root
root_unlock_time = 300
# 감사 로그·기록 디렉터리
audit
dir = /var/run/faillock
EOF
authselect enable-feature with-faillock
authselect current
grep -c faillock /etc/pam.d/system-auth /etc/pam.d/password-auth
```

| 키 | 값 | 의미 |
| --- | --- | --- |
| `deny` | `3` | 연속 실패 **3회** 초과 시 잠금 |
| `unlock_time` | `600` | 잠금 유지 **600초**(10분). `0` 이면 관리자 해제 전까지 영구 |
| `fail_interval` | `900` | 실패 카운트를 누적하는 시간 창(초) |
| `even_deny_root` | (플래그) | **root 도 잠금** — 콘솔 복구 수단이 확실할 때만 |
| `root_unlock_time` | `300` | root 전용 잠금 시간 |
| `audit` | (플래그) | 존재하지 않는 계정 시도도 감사 로그에 기록 |
| `silent` | (플래그) | 잠금 사유를 사용자에게 알리지 않음 |
| `dir` | `/var/run/faillock` | 계정별 실패 기록 파일 위치 |

⚠️ `even_deny_root` 활성 상태에서 root 를 잠그면 SSH 로 못 들어온다 — **UTM 콘솔 접근이 가능한지 먼저 확인**

```bash
# 잠금 재현: dev1 로 3회 실패
# UTM 콘솔에서
#   $ su - dev1        ← 틀린 비번 3회
#   또는 ssh -p 2222 dev1@192.168.64.10 로 틀린 비번 3회
faillock --user dev1
tail -5 /var/log/secure | grep -i faillock
```

```bash
# 해제
faillock --user dev1 --reset
faillock --user dev1
# 전체 사용자 상태
faillock
```

- `faillock --user <계정>` : 해당 계정의 실패 기록(시각·유형·출발지) 조회
- `faillock --user <계정> --reset` : 카운터 초기화 = **잠금 해제**
- `faillock` (인수 없음) : 기록이 있는 모든 계정 표시

**검증**

```bash
faillock --user dev1
# 3회 실패 후
#   dev1:
#   When                Type  Source                     Valid
#   2026-09-04 10:12:01 TTY   pts/1                          V
#   ... (3행)
faillock --user dev1 --reset && faillock --user dev1
# UTM 콘솔: su - dev1 → 정상 로그인 확인
authselect current | head -4
```

```text
dev1:
When                Type  Source                                           Valid
Profile ID: sssd
Enabled features:
- with-faillock
```

> 📝 **시험 포인트**: 필기 FULL r06-57 "로그인 5회 실패 시 10분 잠금" → **`pam_faillock`**(`pam_limits`·`pam_env`·`pam_rootok` 아님). r09-39 는 `auth required pam_faillock.so preauth deny=5 unlock_time=600` 해석. RHEL 7 까지의 `pam_tally2` 는 **RHEL 9 에서 제거** — 시험에서 보기로 나오면 "구버전 모듈" 로 판단.

### 6-6. fail2ban — 로그 감시 기반 자동 차단 (선택)

> **상황**: faillock 은 계정을 잠그고, fail2ban 은 **출발지 IP** 를 방화벽에서 막는다. EPEL 저장소(Part 02) 를 이미 붙였으므로 설치해 동작만 확인한다.

```bash
dnf install -y fail2ban fail2ban-firewalld
cat > /etc/fail2ban/jail.local <<'EOF'
[DEFAULT]
bantime  = 600
findtime = 600
maxretry = 3
backend  = systemd
banaction = firewallcmd-rich-rules
ignoreip = 127.0.0.1/8 192.168.64.1

[sshd]
enabled = true
port    = 2222
logpath = %(sshd_log)s
EOF
systemctl enable --now fail2ban
fail2ban-client status
fail2ban-client status sshd
```

| 키 | 의미 |
| --- | --- |
| `bantime` | 차단 유지 시간(초). `-1` 이면 영구 |
| `findtime` | 실패를 세는 시간 창(초) |
| `maxretry` | `findtime` 안에 이 횟수를 넘으면 차단 |
| `enabled` | 해당 jail 활성 여부 |
| `port` | 차단 규칙을 적용할 포트 (**2222 로 바꿨으므로 반드시 명시**) |
| `logpath` / `backend` | 감시 대상 로그 / 수집 방식(`systemd` = 저널 직접 읽기) |
| `banaction` | 차단 수단 — firewalld rich rule / iptables 등 |
| `ignoreip` | 절대 차단하지 않을 대역 — **관리자 IP 를 반드시 넣을 것** |

```bash
# 차단 확인·해제
fail2ban-client status sshd
fail2ban-client set sshd unbanip 192.168.64.1
fail2ban-client get sshd bantime
journalctl -u fail2ban -n 10 --no-pager
```

**검증**

```bash
systemctl is-active fail2ban
fail2ban-client status
fail2ban-client status sshd | grep -E 'Currently failed|Total failed|Currently banned|Banned IP'
firewall-cmd --list-rich-rules | grep -c 'f2b\|drop' || echo "no ban active"
```

```text
active
Status
|- Number of jail:	1
`- Jail list:	sshd
   |- Currently failed:	0
   |- Total failed:	6
   |- Currently banned:	0
   `- Banned IP list:
no ban active
```

> 📝 **시험 포인트**: 필기 FULL r05-95·r08-95 는 `jail.local` 값 해석(`findtime` 안에 `maxretry` 초과 → `bantime` 동안 차단) 과 `fail2ban-client status sshd` 출력 해석. **`jail.conf` 는 패키지 소유라 직접 수정 금지 → `jail.local` 에 덮어쓰기**가 정석.

### 6-7. 배너 — `/etc/issue` · `/etc/issue.net` · `/etc/motd`

> **상황**: 법적 고지·경고 문구를 접속 시점에 표시한다. 세 파일이 각각 언제 나오는지 실제로 확인한다.

```bash
cat > /etc/issue <<'EOF'
********************************************************
  srv01.lab.local  -  AUTHORIZED ACCESS ONLY
  All activity is monitored and logged.
********************************************************
EOF
cp /etc/issue /etc/issue.net
cat > /etc/motd <<'EOF'
[lab] 이 서버는 실습용입니다. 변경 전 스냅샷을 확인하세요.
EOF
chmod 644 /etc/issue /etc/issue.net /etc/motd
grep -i '^Banner' /etc/ssh/sshd_config
systemctl reload sshd
```

| 파일 | 표시 시점 | 표시 대상 |
| --- | --- | --- |
| `/etc/issue` | **로그인 프롬프트 직전** | 로컬 콘솔·getty(가상 터미널) |
| `/etc/issue.net` | **인증 전** | 원격 — sshd 는 `Banner /etc/issue.net` 지시자로 사용 |
| `/etc/motd` | **로그인 성공 후** | 모든 로그인 (Message Of The Day) |
| `/etc/motd.d/*` | 로그인 성공 후 | 조각 파일 (RHEL 8+, 패키지가 추가) |

- 이스케이프 문자(`\n` 노드명, `\r` 커널 릴리스, `\m` 아키텍처, `\l` 터미널) 는 **`/etc/issue` 에서만** 해석 — `issue.net` 은 sshd 가 그대로 출력하므로 시스템 정보 노출을 피하려면 넣지 않는 게 안전

**검증**

```bash
cat /etc/issue.net
sshd -T | grep -i banner
# macOS: ssh -p 2222 admin1@192.168.64.10       ← 비밀번호 물어보기 전에 배너 표시
# 로그인 후 첫 화면에 /etc/motd 내용 표시
# UTM 콘솔: 로그아웃 후 login: 프롬프트 위에 /etc/issue 표시
```

```text
********************************************************
  srv01.lab.local  -  AUTHORIZED ACCESS ONLY
  All activity is monitored and logged.
********************************************************
banner /etc/issue.net
```

> 📝 **시험 포인트**: **issue = 로컬 로그인 전 / issue.net = 원격 인증 전 / motd = 로그인 후** 3단 구분이 그대로 출제. sshd 는 `Banner` 지시자를 지정해야만 배너를 보여 준다(기본값 `none`).

### 6-8. 미사용 계정·서비스 점검, `/etc/securetty` 부재

> **상황**: 공격 표면을 줄이는 가장 값싼 방법은 "안 쓰는 것을 끄는 것" 이다. 로그인 가능한 계정과 실행 중인 서비스를 전수 확인한다.

```bash
# ① 로그인 가능한 셸을 가진 계정
awk -F: '$7 !~ /nologin|false/ {print $1, $3, $7}' /etc/passwd
awk -F: '$7 !~ /nologin|false/' /etc/passwd | wc -l
# ② 잠기지 않은(비밀번호가 설정된) 계정
awk -F: '$2 !~ /^[!*]/ {print $1}' /etc/shadow
# ③ 계정 상태 일괄 조회
for u in dev1 dev2 ops1 guest1 admin1; do printf '%-8s ' "$u"; passwd -S "$u" 2>/dev/null; done
# ④ 만료·비활성 정책
chage -l dev1 | head -6
```

- `awk -F:` : 구분자를 `:` 로 (**F**ield separator) — `/etc/passwd` 7필드, `/etc/shadow` 9필드
- `$7` = 로그인 셸, `$3` = UID. `/sbin/nologin`·`/bin/false` 는 로그인 불가
- `passwd -S <계정>` : 상태 요약 (**S**tatus) — `PS`(설정됨) / `LK`(잠김) / `NP`(비밀번호 없음)

```bash
# 실행 중 서비스 전수 → 불필요한 것 중지
systemctl list-units --type=service --state=running --no-pager
systemctl list-unit-files --type=service --state=enabled --no-pager | wc -l
# 예: 이 실습에서 안 쓰는 서비스 정리 (필요에 맞게 취사)
systemctl disable --now cups.service cups.socket 2>/dev/null
systemctl disable --now avahi-daemon.service avahi-daemon.socket 2>/dev/null
systemctl list-units --type=service --state=running --no-pager | wc -l
ss -tulnp | wc -l
```

- `--state=running` / `--state=enabled` : 현재 실행 중 / 부팅 활성
- `disable --now` : 부팅 활성 해제 + 즉시 중지 (Part 07 3-3)
- 소켓 활성화 서비스(`cups.socket`) 는 **서비스만 끄면 소켓이 다시 깨움** → 소켓도 함께 disable

```bash
# /etc/securetty — RHEL 9 에서 제거됨
ls -l /etc/securetty 2>/dev/null || echo "/etc/securetty: 없음 (RHEL 9 에서 제거)"
ls /usr/lib64/security/pam_securetty.so 2>/dev/null || echo "pam_securetty.so: 없음"
grep -r securetty /etc/pam.d/ 2>/dev/null | head || echo "pam.d 에 참조 없음"
```

- `/etc/securetty` + `pam_securetty.so` : **root 가 로그인할 수 있는 터미널 목록**을 제한하던 전통 방식 — RHEL 9 에서 파일·모듈 모두 제거
- 같은 목적의 현행 수단: sshd `PermitRootLogin no`(원격), `pam_wheel.so`(`su` 제한), 물리 콘솔 접근 통제

**검증**

```bash
awk -F: '$7 !~ /nologin|false/ {print $1}' /etc/passwd
systemctl list-units --type=service --state=running --no-pager | tail -3
ss -tulnp | grep -cE 'LISTEN|UNCONN'
ls /etc/securetty 2>&1 | tail -1
```

```text
root
admin1
dev1
dev2
ops1
...
No such file or directory
```

> 📝 **시험 포인트**: 필기 FULL r09-30 은 `/etc/securetty` 의 역할(=root 로그인 허용 터미널 목록) 을 묻는다 — **개념은 출제되지만 RHEL 9 에는 없음**. r09-27 의 "wheel 그룹만 `su` 허용" 은 `/etc/pam.d/su` 의 `auth required pam_wheel.so use_uid` 활성 (Part 03 참조).

### 6-9. 세션 타임아웃(`TMOUT`) 과 `sudo` 감사

> **상황**: 자리를 비운 터미널이 열려 있는 것도 취약점이다. 전역 자동 로그아웃을 걸고, 사용자가 바꾸지 못하게 잠근다. 마지막으로 권한 상승 이력을 확인한다.

```bash
cat > /etc/profile.d/tmout.sh <<'EOF'
# 10분(600초) 무입력 시 셸 자동 종료 — 사용자가 해제하지 못하도록 readonly
TMOUT=600
readonly TMOUT
export TMOUT
EOF
chmod 644 /etc/profile.d/tmout.sh
ls -l /etc/profile.d/tmout.sh
```

- `TMOUT` : bash 내장 변수 — 이 초 동안 입력이 없으면 셸 종료
- `readonly TMOUT` : 이후 `TMOUT=0` 으로 해제 시도 시 오류 → **우회 차단**이 핵심
- `/etc/profile.d/*.sh` : 로그인 셸 시작 시 `/etc/profile` 이 자동으로 읽는 조각 (Part 01·04 참조)
- 적용 시점은 **다음 로그인부터** — 현재 세션은 `source /etc/profile.d/tmout.sh` 로 즉시 반영 가능

```bash
# sudo 감사
journalctl _COMM=sudo --since today --no-pager | tail -10
grep 'sudo' /var/log/secure | tail -5
ausearch -m USER_CMD -ts today -i 2>/dev/null | tail -5
# sudoers 설정 확인 (Part 03 에서 구성)
grep -vE '^\s*#|^\s*$' /etc/sudoers | head
ls /etc/sudoers.d/
```

- `journalctl _COMM=sudo` : 실행 파일 이름이 `sudo` 인 레코드만 (`_COMM` = systemd 저널 필드)
- `/var/log/secure` 에도 `sudo: <user> : TTY=... ; PWD=... ; USER=root ; COMMAND=...` 형식으로 기록
- `ausearch -m USER_CMD` : auditd 가 남기는 sudo 실행 레코드

**검증**

```bash
# 새 로그인 세션에서
# $ echo $TMOUT            → 600
# $ TMOUT=0                → bash: TMOUT: readonly variable
su - dev1 -c 'echo TMOUT=$TMOUT'
journalctl _COMM=sudo --since today --no-pager | wc -l
grep -c 'COMMAND=' /var/log/secure
```

```text
TMOUT=600
5
5
```

> 📝 **시험 포인트**: `TMOUT` 은 **초 단위**, 적용 위치는 `/etc/profile` 또는 `/etc/profile.d/*.sh`(전역) · `~/.bash_profile`(개인). `readonly` 를 붙이는 이유(사용자 우회 방지) 가 서술형 포인트. sudo 이력은 `/var/log/secure` (Part 07 rsyslog `authpriv.*` 규칙).

---

## 7. 침해 점검·무결성

### 7-1. 기준선(baseline) 생성 — SetUID/SetGID 전수 수집

> **상황**: 무결성 점검의 전제는 "정상 상태의 사진" 이다. 침해가 의심될 때 비교할 기준선을 지금(깨끗하다고 믿는 시점) 만들어 둔다.

```bash
mkdir -p /root/baseline && chmod 700 /root/baseline
find / -xdev \( -perm -4000 -o -perm -2000 \) -type f -exec ls -l {} \; \
  > /root/baseline/suid.base 2>/dev/null
wc -l /root/baseline/suid.base
head -5 /root/baseline/suid.base
find / -xdev -perm -4000 -type f 2>/dev/null | wc -l      # SetUID 만
find / -xdev -perm -2000 -type f 2>/dev/null | wc -l      # SetGID 만
```

- `-xdev` : 다른 파일시스템으로 내려가지 않음 (**x** = don't cross **dev**ice) — `/proc`·`/sys`·NFS 마운트 제외 효과
- `-perm -4000` : 앞의 `-` = "이 비트가 **포함**되면 매칭" (정확히 4000 이 아님). `4000`=SetUID, `2000`=SetGID, `1000`=Sticky
- `\( … -o … \)` : OR 그룹 — 셸이 괄호를 해석하지 못하게 `\` 로 이스케이프
- `-type f` : 일반 파일만 (**f**ile) — 디렉터리 SetGID 제외
- `-exec ls -l {} \;` : 매칭마다 `ls -l` 실행. `{}` 는 파일 경로, `\;` 는 종료 표시. 빠른 대안은 `-exec ls -l {} +` 또는 `-ls`
- `2>/dev/null` : 권한 없는 디렉터리의 오류 메시지 버리기

**검증**

```bash
ls -l /root/baseline/
wc -l /root/baseline/suid.base
grep -c 'rws' /root/baseline/suid.base
grep -E '/usr/bin/(passwd|su|sudo|chage|mount)' /root/baseline/suid.base
```

```text
-rw-r--r--. 1 root root ... suid.base
20 /root/baseline/suid.base
...
-rwsr-xr-x. 1 root root ... /usr/bin/passwd
-rwsr-xr-x. 1 root root ... /usr/bin/su
-rwsr-xr-x. 1 root root ... /usr/bin/sudo
```

> 📝 **시험 포인트**: 실기 r02-1·r05-3 과 필기 FULL r04-65·r05-65·r06-62·r09-31 이 모두 `find / -perm -4000 -type f`. 오답 축은 "권한이 **정확히** 4000 인 파일"(→ `-perm 4000`) 과 "4000분 이내 수정"(→ `-mmin`). 실기 답안은 `-type f` 를 빠뜨리지 말 것.

### 7-2. 변조 → `diff` 탐지 → 원복

> **상황**: 기준선이 실제로 작동하는지 확인한다. 공격자가 `find` 에 SetUID 를 심었다고 가정하고 탐지 절차를 그대로 돌린다.

⚠️ `chmod u+s /usr/bin/find` 는 **실제 권한 상승 취약점**을 만든다. 아래 원복 명령까지 한 번에 수행할 것

```bash
ls -l /usr/bin/find
chmod u+s /usr/bin/find                       # ← 변조 (실습)
ls -l /usr/bin/find                           # rws 확인

# 재수집 → 비교
find / -xdev \( -perm -4000 -o -perm -2000 \) -type f -exec ls -l {} \; \
  > /root/baseline/suid.now 2>/dev/null
diff /root/baseline/suid.base /root/baseline/suid.now
diff <(awk '{print $NF}' /root/baseline/suid.base | sort) \
     <(awk '{print $NF}' /root/baseline/suid.now  | sort)
```

- `diff <(cmd1) <(cmd2)` : 프로세스 치환 — 명령 출력을 임시 파일처럼 비교 (Part 04 참조)
- `<` 로 시작하는 행 = 기준선에만 있음(사라진 파일), `>` = 현재에만 있음(**새로 생긴 SetUID = 침해 의심**)
- `awk '{print $NF}'` : 마지막 필드(경로) 만 — 타임스탬프 변화로 인한 오탐 제거

```bash
# 원복 + 재확인
chmod u-s /usr/bin/find
ls -l /usr/bin/find
find / -xdev \( -perm -4000 -o -perm -2000 \) -type f -exec ls -l {} \; \
  > /root/baseline/suid.now 2>/dev/null
diff <(awk '{print $NF}' /root/baseline/suid.base | sort) \
     <(awk '{print $NF}' /root/baseline/suid.now  | sort) && echo "일치 — 변조 없음"
```

**검증**

```bash
ls -l /usr/bin/find | cut -c1-11
diff <(awk '{print $NF}' /root/baseline/suid.base | sort) \
     <(awk '{print $NF}' /root/baseline/suid.now  | sort); echo "diff exit=$?"
```

```text
> -rwsr-xr-x. 1 root root ... /usr/bin/find      ← 변조 시점 diff 출력
-rwxr-xr-x                                        ← 원복 후
diff exit=0
일치 — 변조 없음
```

> 📝 **시험 포인트**: "SetUID 가 붙은 **표준 도구가 아닌** 실행 파일"·"`/tmp`·홈 디렉터리의 SetUID 셸" 은 침해 1순위 징후. `chmod u+s` = `chmod 4755`, 해제는 `chmod u-s` = `chmod 755`.

### 7-3. 위험 파일·계정 전수 점검

> **상황**: SetUID 외에 점검 항목을 한 번에 훑는다. 각각이 왜 위험한지와 함께 정리한다.

```bash
# ① 누구나 쓸 수 있는 파일 (world-writable)
find / -xdev -type f -perm -0002 ! -type l 2>/dev/null | head -20
find / -xdev -type f -perm -0002 2>/dev/null | wc -l
# ② 누구나 쓸 수 있는 디렉터리 중 Sticky 없는 것 (더 위험)
find / -xdev -type d -perm -0002 ! -perm -1000 2>/dev/null
# ③ 소유자·그룹이 없는 파일 (삭제된 계정의 잔재 = 재사용 UID 로 탈취 가능)
find / -xdev \( -nouser -o -nogroup \) 2>/dev/null | head
# ④ /tmp 의 숨김 파일 (백도어 은닉 단골)
ls -la /tmp /var/tmp /dev/shm
find /tmp /var/tmp /dev/shm -name '.*' ! -name '.' ! -name '..' 2>/dev/null
# ⑤ UID 0 계정 (root 외에 있으면 백도어 계정)
awk -F: '$3==0 {print $1, $3, $7}' /etc/passwd
# ⑥ 빈 비밀번호 계정
awk -F: '($2==""){print $1 " : EMPTY PASSWORD"}' /etc/shadow
awk -F: '($2=="" || $2=="!!"){print $1, ($2==""?"EMPTY":"NEVER-SET")}' /etc/shadow
```

- `-perm -0002` : "기타(other) 쓰기" 비트가 켜진 것 — 임의 사용자가 내용을 바꿀 수 있음
- `! -type l` : 심볼릭 링크 제외 (링크의 퍼미션은 무의미)
- `! -perm -1000` : Sticky 비트가 **없는** 것 — `/tmp` 처럼 1777 이면 남의 파일 삭제가 막히지만, Sticky 없이 777 인 디렉터리는 위험
- `-nouser` / `-nogroup` : `/etc/passwd`·`/etc/group` 에 없는 UID/GID 소유
- `/dev/shm` : tmpfs 공유 메모리 — 디스크에 안 남아 은닉 장소로 쓰임

| 점검 항목 | 명령 | 왜 위험한가 |
| --- | --- | --- |
| SetUID/SetGID | `find / -xdev -perm -4000 -type f` | 실행 즉시 소유자 권한 획득 → 권한 상승 |
| world-writable 파일 | `find / -xdev -type f -perm -0002` | 스크립트·설정 변조로 코드 실행 |
| Sticky 없는 777 디렉터리 | `find / -xdev -type d -perm -0002 ! -perm -1000` | 남의 파일 삭제·교체 |
| 소유자 없는 파일 | `find / -xdev \( -nouser -o -nogroup \)` | 같은 UID 로 계정 생성 시 자동 소유 |
| UID 0 계정 | `awk -F: '$3==0' /etc/passwd` | root 와 동일 권한의 은닉 계정 |
| 빈 비밀번호 | `awk -F: '$2==""' /etc/shadow` | 비밀번호 없이 로그인 |
| 숨김 파일 in `/tmp` | `find /tmp -name '.*'` | 백도어·수집 데이터 은닉 |

**검증**

```bash
awk -F: '$3==0 {print $1}' /etc/passwd
awk -F: '($2==""){print $1}' /etc/shadow | wc -l
find / -xdev -type d -perm -0002 ! -perm -1000 2>/dev/null | wc -l
ls -ld /tmp | cut -c1-11
```

```text
root
0
0
drwxrwxrwt
```

> 📝 **시험 포인트**: `/tmp` 의 정상 권한은 **1777 (`drwxrwxrwt`)** — Sticky 비트 `t` 가 있어야 소유자만 삭제 가능. UID 0 이 root 하나뿐인지는 침해 점검 첫 항목. `awk -F: '$3==0 {print $1}' /etc/passwd` 는 실기 서술형에 그대로 나올 수 있는 관용구.

### 7-4. 패키지 무결성 — `rpm -Va` 와 `sha256sum` 기준선

> **상황**: 시스템 바이너리가 교체됐는지 두 가지 방법으로 본다. RPM DB 대조(설치 시점 대비) 와 직접 만든 해시 기준선(현재 시점 대비) 이다.

```bash
rpm -Va | head -20                                  # 전체 패키지 검증 (수 분 소요)
rpm -Va | grep -E '^..5|^missing' | head            # 해시 변경 또는 파일 소실만
rpm -Va --nomtime --nomode | grep -v '^\.\{8\}' | head
rpm -V httpd openssh-server sudo coreutils          # 특정 패키지만
rpm -qf /usr/bin/find                                # 이 파일이 속한 패키지
```

- `rpm -V <패키지>` : 설치 시 기록한 메타데이터와 현재 파일 비교 (**V**erify)
- `-a` : 전체 패키지 (**a**ll) → `rpm -Va`
- 결과 8자리 코드 — 이상 없으면 `.`, 다르면 해당 문자

| 위치 | 문자 | 의미 |
| --- | --- | --- |
| 1 | `S` | **S**ize — 파일 크기 변경 |
| 2 | `M` | **M**ode — 권한·파일 종류 변경 |
| 3 | `5` | MD5(현재는 SHA) **체크섬 변경** = 내용 변조 |
| 4 | `D` | **D**evice — 장치 번호 |
| 5 | `L` | **L**ink — 심볼릭 링크 대상 |
| 6 | `U` | **U**ser — 소유자 |
| 7 | `G` | **G**roup — 그룹 |
| 8 | `T` | m**T**ime — 수정 시각 |
| 9 | `P` | ca**P**abilities |
| 앞 | `missing` | 파일이 없음 |
| 뒤 | `c` `d` `g` `l` `r` | 파일 유형 — **c**onfig / **d**oc / **g**host / **l**icense / **r**eadme |

- `c`(설정 파일) 가 붙은 `S.5....T.` 는 **관리자가 편집한 정상 상태**일 수 있음 → `/etc` 설정은 오탐으로 걸러야 함
- 반대로 `/usr/bin`·`/usr/sbin` 의 **바이너리에 `5`** 가 뜨면 심각

```bash
# 직접 만드는 해시 기준선
sha256sum /usr/bin/* /usr/sbin/* > /root/baseline/bin.sha256 2>/dev/null
wc -l /root/baseline/bin.sha256
chmod 600 /root/baseline/bin.sha256
# 나중에 비교
sha256sum -c --quiet /root/baseline/bin.sha256 2>/dev/null | head
echo "check exit=$?"
```

- `sha256sum -c <목록파일>` : 목록의 해시와 현재 파일을 대조 (**c**heck)
- `--quiet` : **실패한 것만** 출력 — OK 는 침묵
- `--status` : 출력 없이 종료 코드로만 판정 (스크립트용)
- 기준선 파일 자체가 변조되면 무의미 → 오프라인 매체·다른 호스트 보관 또는 `chattr +i` (Part 03)

**검증**

```bash
rpm -Va | grep -cE '^..5' || echo "체크섬 변경 없음"
rpm -V coreutils; echo "coreutils exit=$?"
sha256sum -c --status /root/baseline/bin.sha256 2>/dev/null; echo "baseline exit=$?"
```

```text
체크섬 변경 없음
coreutils exit=0
baseline exit=0
```

> 📝 **시험 포인트**: 필기 FULL r10-38 은 `rpm -V httpd` 의 `S.5....T.  c /etc/httpd/conf/httpd.conf` 해석 — **설정 파일(c) 의 크기(S)·해시(5)·수정 시각(T) 이 설치 시점과 달라짐**. 8자리 코드에서 `5` 의 위치(3번째) 와 의미를 외울 것.

### 7-5. AIDE — 파일 무결성 검사 도구

> **상황**: `rpm -Va` 는 RPM 이 설치한 파일만 본다. 설정·데이터까지 포함한 전면 무결성 감시는 AIDE(HIDS) 로 한다. DB 생성 → 변조 → 탐지 → 갱신 흐름을 한 바퀴 돈다.

```bash
dnf install -y aide
grep -vE '^\s*#|^\s*$' /etc/aide.conf | head -30
grep -E '^(database|database_out|gzip_dbout)' /etc/aide.conf
```

| `/etc/aide.conf` 항목 | 의미 |
| --- | --- |
| `database=file:/var/lib/aide/aide.db.gz` | 비교 기준 DB 경로 |
| `database_out=file:/var/lib/aide/aide.db.new.gz` | `--init`·`--update` 가 만드는 새 DB |
| `gzip_dbout=yes` | DB 압축 저장 |
| `NORMAL = FIPSR+sha512` 등 **규칙 그룹** | 검사 항목 묶음 이름 |
| `p` `i` `n` `u` `g` `s` `m` `c` `md5` `sha256` `sha512` `acl` `selinux` `xattrs` | 권한/inode/링크수/소유자/그룹/크기/mtime/ctime/해시/ACL/컨텍스트/확장속성 |
| `/etc/ NORMAL` | 이 경로에 이 규칙 적용 |
| `!/var/log/.*` | `!` 로 시작 = **검사 제외** (자주 변하는 로그) |

```bash
# ① 기준 DB 생성 (수 분 소요)
aide --init
ls -lh /var/lib/aide/
mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz     # 새 DB 를 기준 DB 로 승격

# ② 변조 발생 (실습)
echo "# lab tamper $(date)" >> /etc/hosts
touch /etc/lab-suspicious.conf

# ③ 탐지
aide --check | head -40
aide --check | grep -E '^(Added|Removed|Changed) entries|^Total number'

# ④ 원복 후 DB 갱신
sed -i '/# lab tamper/d' /etc/hosts
rm -f /etc/lab-suspicious.conf
aide --update && mv -f /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
aide --check | grep -E 'found differences|AIDE found NO differences'
```

- `aide --init` : 기준 DB 최초 생성 → `aide.db.new.gz` 로 나오므로 **직접 `aide.db.gz` 로 옮겨야** 비교가 시작됨
- `aide --check` : 현재 상태와 DB 비교. 차이가 있으면 종료 코드 ≠ 0
- `aide --update` : 검사 + 새 DB 생성(정상 변경을 기준선에 반영)
- `aide --compare` : 두 DB 직접 비교
- `--config=<파일>` : 다른 설정 파일 사용
- 기준 DB 도 변조 대상 → 별도 매체 보관 또는 `chattr +i /var/lib/aide/aide.db.gz`

**검증**

```bash
ls -lh /var/lib/aide/aide.db.gz
aide --check > /root/baseline/aide-check.txt 2>&1; echo "aide exit=$?"
grep -E 'AIDE found|Total number of entries|Changed entries' /root/baseline/aide-check.txt
```

```text
-rw-------. 1 root root ... /var/lib/aide/aide.db.gz
AIDE found differences between database and filesystem!!      ← 변조 시점
Changed entries:  1
Added entries:    1
...
AIDE found NO differences between database and filesystem. Looks okay!!   ← 원복·갱신 후
aide exit=0
```

> 📝 **시험 포인트**: 필기 FULL r10-63 — "설치 직후 생성한 **기준 데이터베이스**와 현재 파일의 해시·속성을 비교해 변조를 탐지" 가 Tripwire·AIDE 의 정의. 분류상 **HIDS**(호스트 기반) — NIDS(Snort·Suricata) 와 구분. `--init` 후 **DB 이름을 바꿔 주는 단계**를 빼먹으면 `--check` 가 실패한다.

### 7-6. 불변 속성·로그인 이력 확인

> **상황**: Part 03 에서 `chattr +i` 를 걸어 둔 파일이 아직 잠겨 있는지 확인하고, 누가 언제 들어왔는지 4종 로그를 모두 훑는다.

```bash
lsattr /etc/passwd /etc/shadow /etc/group /etc/gshadow 2>/dev/null
lsattr -d /etc /root
# 잠겨 있으면 계정 작업 전 해제 → 작업 → 재설정
# chattr -i /etc/passwd ; useradd … ; chattr +i /etc/passwd
```

- `lsattr` : 확장 속성 조회. `----i---------` 의 `i` = immutable, `a` = append-only
- `chattr +i` 상태에서는 root 도 수정·삭제·이름변경 불가 → `useradd` 가 실패하는 원인이 됨

```bash
last -x | head -15              # wtmp: 로그인·로그아웃 + 재부팅·런레벨
last -n 10 -a -F                # 호스트를 마지막 열에(-a), 전체 시각(-F)
last reboot | head -5           # 재부팅 이력만
lastb -n 10                     # btmp: 실패
lastlog                         # 계정별 마지막 로그인
lastlog -b 90                   # 90일 이상 로그인 없는 계정 (-b = before)
lastlog -u dev1
who -a                          # 현재 접속자 (utmp)
w                               # 현재 접속자 + 실행 중 명령 + load average
```

| 명령 | 읽는 파일 | 내용 |
| --- | --- | --- |
| `last` | `/var/log/wtmp` | 로그인·로그아웃 **성공** 이력, `reboot`·`shutdown` 포함 |
| `lastb` | `/var/log/btmp` | 로그인 **실패** 이력 (root 전용) |
| `lastlog` | `/var/log/lastlog` | 계정별 **마지막** 로그인 시각 |
| `who` `w` `users` | `/var/run/utmp` | **현재** 접속 중인 세션 |

- `last -x` : 시스템 이벤트(런레벨 변경·shutdown) 포함 (**x**)
- `lastlog -b <일>` : 지정 일수 **이전**이 마지막인 계정 = **미사용 계정 후보**
- `lastlog -t <일>` : 지정 일수 **이내**에 로그인한 계정만

**검증**

```bash
last -x | head -5
lastlog -b 90 | head
who -a | head -3
lsattr /etc/passwd 2>/dev/null
```

```text
admin1   pts/0        192.168.64.1     Fri Sep  4 10:02   still logged in
reboot   system boot  5.14.0-...       Fri Sep  4 09:58   still running
Username         Port     From             Latest
guest1                                     **Never logged in**
...
--------------e------- /etc/passwd
```

> 📝 **시험 포인트**: 로그 파일 ↔ 조회 명령 매칭은 필기 최빈출(FULL r01-58·r02-59·r03-62·r04-56·r05-63·r06-60·r07-63·r08-56·r09-60). **바이너리 로그는 `cat` 으로 못 읽는다**, `utmp` 만 `/var/run` 하위, 나머지는 `/var/log`.

### 7-7. 의심 포트·프로세스·cron 백도어 점검

> **상황**: 침해가 의심될 때 "지금 무엇이 통신하고 있고, 누가 그것을 띄웠는가" 를 추적하는 순서를 익힌다.

```bash
# ① 연결·리스닝 전수
ss -tanp | head -20                     # TCP 전체 (연결 포함)
ss -tulnp                               # 리스닝만 (1-1 참조)
ss -tanp state established               # 성립된 연결만
ss -tp '( dport = :443 or sport = :443 )' # 특정 포트
# ② 열린 파일·소켓
lsof -i -P -n | head -20
lsof -i :2222
lsof -p <PID>                            # 특정 프로세스가 연 파일
lsof +L1                                 # 링크 수 0 = 삭제됐는데 실행 중 (은닉 단골)
# ③ 프로세스 계보
ps -ef --forest | head -30
ps auxf | grep -v '\[' | head -20
ps -eo pid,ppid,user,etime,cmd --sort=start_time | tail -10   # 최근 시작 순
```

- `ss -tanp` : TCP(**t**) 전체(**a**) 숫자(**n**) 프로세스(**p**) — 연결 상태까지 포함
- `lsof -i -P -n` : 네트워크 소켓(**i**), 포트를 숫자로(**P**), 주소를 숫자로(**n**)
- `lsof +L1` : 링크 수가 1 미만인 열린 파일 = **삭제된 실행 파일이 아직 돌고 있음**
- `ps -ef --forest` : 부모-자식 트리 — 웹 서버 아래 셸(`httpd → sh → nc`) 같은 이상 계보 탐지
- `[대괄호]` 로 감싸인 것은 커널 스레드

```bash
# ④ cron 백도어 점검 — 지속성 확보의 1순위 수단
cat /etc/crontab
ls -la /etc/cron.d/ && cat /etc/cron.d/* 2>/dev/null | grep -v '^#'
ls -la /etc/cron.hourly/ /etc/cron.daily/ /etc/cron.weekly/ /etc/cron.monthly/
ls -la /var/spool/cron/
for f in /var/spool/cron/*; do echo "== $f"; cat "$f"; done 2>/dev/null
crontab -l -u dev1; crontab -l -u root
cat /etc/cron.allow /etc/cron.deny 2>/dev/null
# systemd 타이머도 지속성 수단
systemctl list-timers --all --no-pager
# at 잡
atq; ls -la /var/spool/at/ 2>/dev/null
# ⑤ 로그인 시 실행되는 파일
ls -la /etc/profile.d/ ~/.bashrc ~/.bash_profile /etc/bashrc
grep -rE 'curl|wget|nc |base64|/dev/tcp' /etc/profile.d/ 2>/dev/null
```

- `/var/spool/cron/<사용자>` : 사용자 crontab 실체 — 직접 편집 대신 `crontab -e` (Part 06)
- `/etc/cron.d/*` : 조각 파일(사용자 필드 포함 7필드) — 공격자가 파일 하나만 떨궈도 지속성 확보
- `systemctl list-timers` : cron 대신 systemd 타이머로 숨는 경우
- `/dev/tcp/<host>/<port>` : bash 내장 리버스 셸 관용구 — 프로필 스크립트에 있으면 즉시 의심

**검증**

```bash
ss -tulnp | grep -vE ':(22|2222|80|443|53|21|25|111|139|445|631|2049|8080|6379|20048)\b' | grep LISTEN || echo "예상 외 리스닝 없음"
lsof +L1 2>/dev/null | head -3 || echo "삭제된 실행 파일 없음"
ls /var/spool/cron/ 2>/dev/null; ls /etc/cron.d/
systemctl list-timers --no-pager | tail -3
```

```text
예상 외 리스닝 없음
삭제된 실행 파일 없음
dev1
0hourly  lab-backup  raid-check
...logrotate.timer ...
```

> 📝 **시험 포인트**: "이상 프로세스·연결 확인" 은 `ps -ef`·`ss -antp`(=`netstat -antp`)·`lsof -i` 3종. cron 점검 경로 4곳(`/etc/crontab`, `/etc/cron.d/`, `/etc/cron.*/`, `/var/spool/cron/`) 을 열거할 수 있어야 함.

### 7-8. `rkhunter` · `chkrootkit` (선택)

> **상황**: 알려진 루트킷 시그니처와 시스템 명령 변조를 자동 점검한다. EPEL 저장소(Part 02) 필요.

```bash
dnf install -y rkhunter
rkhunter --update                       # 시그니처 갱신 (네트워크 필요)
rkhunter --propupd                      # 현재 파일 속성을 "정상" 으로 기록 (기준선)
rkhunter --check --sk --rwo             # 점검 실행
less /var/log/rkhunter/rkhunter.log
```

- `--propupd` : 파일 속성 DB 갱신 (**prop**erties **upd**ate) — **깨끗한 시점에** 한 번, 패키지 업데이트 후마다
- `--check` : 전체 점검
- `--sk` / `--skip-keypress` : 섹션마다 Enter 대기 생략 (스크립트·자동화용)
- `--rwo` / `--report-warnings-only` : 경고만 출력
- `--enable <테스트>` / `--disable <테스트>` : 개별 테스트 선택
- 설정: `/etc/rkhunter.conf`, 로그: `/var/log/rkhunter/rkhunter.log`

```bash
# chkrootkit (EPEL)
dnf install -y chkrootkit 2>/dev/null && chkrootkit | grep -vE 'not found|not infected' | head
```

- ⚠️ 두 도구 모두 **오탐이 흔함** — 특히 `--propupd` 를 하지 않은 상태의 "file properties have changed" 경고. 결과는 반드시 `rpm -V`·AIDE 와 교차 확인

**검증**

```bash
rkhunter --check --sk --rwo | tail -20
grep -cE 'Warning' /var/log/rkhunter/rkhunter.log
tail -5 /var/log/rkhunter/rkhunter.log
```

```text
Warning: The command '/usr/bin/...' has been replaced by a script: ...
System checks summary
=====================
File properties checks...
    Files checked: ...
    Suspect files: 0
Rootkit checks...
    Rootkits checked : ...
    Possible rootkits: 0
```

> 📝 **시험 포인트**: 침해 점검 도구 분류 — **rkhunter·chkrootkit = 루트킷 탐지**, **Tripwire·AIDE = 파일 무결성**, **John the Ripper = 취약 비밀번호**, **Nmap = 포트 스캔**, **Nessus·OpenVAS = 종합 취약점**.

### 7-9. 공격 유형·대응표와 `sysctl` 커널 하드닝

> **상황**: 필기 8~10문항이 걸린 공격 유형을 정리하고, 그중 커널 파라미터로 막을 수 있는 것을 실제 설정 파일로 만든다.

| 분류 | 공격 | 원리 | 대응 |
| --- | --- | --- | --- |
| **DoS/DDoS** | **SYN Flooding** | 3-way 중 SYN 만 대량 전송 → 백로그 큐 고갈 | `tcp_syncookies=1`, `tcp_max_syn_backlog` 증대, `tcp_synack_retries` 축소, 방화벽 rate limit |
| | **Smurf** | 브로드캐스트로 ICMP 증폭, 출발지를 피해자로 위조 | `icmp_echo_ignore_broadcasts=1`, 라우터의 directed broadcast 차단 |
| | **Ping of Death** | 규격 초과(>65535) ICMP 로 재조합 오류 | 커널 패치(현대 커널은 면역), ICMP 크기 제한 |
| | **Teardrop** | 조작된 프래그먼트 오프셋으로 재조합 오류 | 커널 패치, 방화벽의 프래그먼트 검사 |
| | **Land** | 출발지=목적지 동일 위조 패킷으로 자기 참조 루프 | `rp_filter=1`, 스푸핑 필터링 |
| | **DDoS** | 다수 좀비(핸들러–에이전트) 분산 공격 | 상위 ISP 차단, CDN·스크러빙, 이상 트래픽 탐지 |
| **스푸핑·MITM** | **스니핑** | 패킷 도청(수동 공격) | **암호화 프로토콜**(SSH·TLS), 스위치, 포트 보안 |
| | **ARP 스푸핑** | ARP 캐시 위조 → 스위치 환경 MITM | **정적 ARP**, DAI, arpwatch, 동일 MAC 중복 감시 |
| | **IP 스푸핑** | 출발지 IP 위조로 신뢰 관계 악용 | `rp_filter=1`, `accept_source_route=0`, 경계 라우터 ingress 필터 |
| | **DNS 스푸핑/파밍** | 위조 응답으로 악성 사이트 유도 | DNSSEC, 신뢰 리졸버 고정, `hosts` 무결성 |
| | **세션 하이재킹** | 인증된 세션 탈취(시퀀스 예측·TCP 재설정) | TLS, 세션 토큰 재발급, `accept_redirects=0` |
| **악성코드** | **백도어** | 인증 우회 은닉 통로 | 포트·프로세스·cron 점검(7-7), 무결성 검사 |
| | **트로이목마** | 정상 프로그램으로 위장 | 서명 검증(`rpm -K`), 출처 통제 |
| | **루트킷** | 커널·명령 교체로 존재 은닉 | rkhunter·chkrootkit, 외부 매체 부팅 점검 |
| | **웜** | 자가 전파 | 패치, 불필요 서비스 차단 |
| | **랜섬웨어** | 파일 암호화 후 금전 요구 | **오프라인 백업**(Part 12), 최소 권한, 매크로 차단 |
| **코드 실행** | **버퍼 오버플로** | 경계 검사 미흡 → 스택·힙 덮어쓰기 | ASLR·NX·스택 카나리, 안전 함수, 컴파일러 보호 옵션 |
| | **포맷 스트링** | `printf` 서식 문자열 취약점 | 서식 문자열 고정 |
| | **레이스 컨디션(TOCTOU)** | 검사·사용 시점 차이 악용 | 원자적 연산, SetUID 임시파일 회피 |
| **정찰** | **포트 스캔** | 열린 포트·서비스 탐지 | 방화벽, 최소 노출, IDS 로 스캔 패턴 탐지 |
| | 스캔 유형 | `-sS` SYN(하프 오픈·스텔스) / `-sT` connect / `-sF` FIN / `-sN` NULL / `-sX` XMAS / `-sU` UDP | FIN·NULL·XMAS 는 플래그 조작으로 로그 회피 시도 |
| | **풋프린팅** | whois·DNS·검색으로 사전 정보 수집 | 등록 정보 최소화, 존 전송 제한(Part 09) |
| **사람** | **사회공학** | 사람을 속여 정보 획득 | 교육, 절차 검증 |
| | **피싱/파밍** | 가짜 사이트 유도 / DNS 조작 | 도메인 확인, 인증서 검증, DNSSEC |

```bash
# 커널 파라미터 하드닝
cat > /etc/sysctl.d/90-hardening.conf <<'EOF'
# --- SYN Flooding 대응 ---
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 2

# --- Smurf / ICMP 대응 ---
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
# net.ipv4.icmp_echo_ignore_all = 1        # ping 전면 차단 (진단이 어려워지므로 기본 0 유지)

# --- IP 스푸핑 / 소스 라우팅 / 리다이렉트 ---
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# --- 비정상 패킷 로깅 ---
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# --- 라우터가 아니므로 포워딩 차단 (3-9 실습 후 원복) ---
net.ipv4.ip_forward = 0
EOF
sysctl -p /etc/sysctl.d/90-hardening.conf
```

| 파라미터 | 값 | 막는 것 |
| --- | --- | --- |
| `net.ipv4.tcp_syncookies` | `1` | **SYN Flooding** — 백로그가 넘칠 때 상태 저장 없이 쿠키로 검증 |
| `net.ipv4.tcp_max_syn_backlog` | `2048` | 반개방 연결 큐 확대 |
| `net.ipv4.tcp_synack_retries` | `2` | 응답 없는 반개방 연결을 빨리 정리 |
| `net.ipv4.icmp_echo_ignore_broadcasts` | `1` | **Smurf** — 브로드캐스트 ping 무응답 |
| `net.ipv4.icmp_echo_ignore_all` | `1`(선택) | 모든 ping 무응답 — 진단 곤란해짐 |
| `net.ipv4.conf.all.rp_filter` | `1` | **IP 스푸핑·Land** — 역경로 검증(Reverse Path Filter) |
| `net.ipv4.conf.all.accept_source_route` | `0` | 소스 라우팅으로 경로 우회 |
| `net.ipv4.conf.all.accept_redirects` | `0` | ICMP 리다이렉트로 라우팅 테이블 오염 (MITM) |
| `net.ipv4.conf.all.send_redirects` | `0` | 라우터가 아니면 보낼 이유 없음 |
| `net.ipv4.conf.all.log_martians` | `1` | 출처가 말이 안 되는 패킷을 커널 로그로 |
| `net.ipv4.ip_forward` | `0` | 의도치 않은 라우팅 |

- `/etc/sysctl.d/*.conf` : 조각 파일 (파일명 숫자 순으로 적용). `/etc/sysctl.conf` 는 레거시
- `sysctl -p <파일>` : 지정 파일 즉시 적용 (**p**reload). 파일 생략 시 `/etc/sysctl.conf`
- `sysctl --system` : `/usr/lib/sysctl.d/` → `/run/sysctl.d/` → `/etc/sysctl.d/` → `/etc/sysctl.conf` 순 전체 재적용
- `sysctl -w <키>=<값>` : 일시 적용 (**w**rite, 재부팅 시 원복)
- `sysctl -a` : 전체 조회

**검증**

```bash
sysctl -a 2>/dev/null | grep -E 'tcp_syncookies|tcp_max_syn_backlog|icmp_echo_ignore_broadcasts|all.rp_filter|all.accept_redirects|all.log_martians|ip_forward$'
cat /proc/sys/net/ipv4/tcp_syncookies
sysctl --system >/dev/null && echo "reloaded"
# macOS: ping -c 2 192.168.64.10        ← 유니캐스트 ping 은 여전히 응답
```

```text
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.ip_forward = 0
1
reloaded
```

> 📝 **시험 포인트**: **SYN Flooding ↔ SYN 쿠키·백로그 증대**, **Smurf ↔ 브로드캐스트 ICMP 차단**, **IP 스푸핑 ↔ rp_filter** 매칭이 필기 정답 축(FULL r01-91·r02-91·r02-93·r04-96·r05-100·r06-98·r07-95, 실기 r05-14). `sysctl -w` = 일시 / 파일 등록 + `sysctl -p` = 영구 구분.

### 7-10. ARP 스푸핑 대응 — 정적 ARP·`arping`·`tcpdump`

> **상황**: 게이트웨이(192.168.64.1) 의 MAC 을 고정해 ARP 캐시 오염을 막는다. 캡처로 ARP 트래픽이 실제로 어떻게 오가는지 확인한다.

```bash
ip neigh show                                  # 현재 ARP(이웃) 캐시
ip neigh show dev enp0s1
arp -an 2>/dev/null || echo "arp: net-tools 필요"
GW=$(ip route | awk '/^default/{print $3}'); echo "GW=$GW"
GWMAC=$(ip neigh show "$GW" | awk '{print $5}'); echo "GWMAC=$GWMAC"
```

- `ip neigh show` : 이웃(ARP/NDP) 캐시. 상태 — `REACHABLE`(확인됨) / `STALE`(오래됨) / `PERMANENT`(정적) / `FAILED`
- `arp -an` : 레거시 표기 (net-tools). `ip neigh` 가 현행

```bash
# 정적(PERMANENT) 항목 등록 — 스푸핑된 ARP 응답을 무시하게 됨
ip neigh replace "$GW" lladdr "$GWMAC" nud permanent dev enp0s1
ip neigh show "$GW"
ping -c 2 "$GW"

# 원복
ip neigh del "$GW" dev enp0s1
ip neigh show "$GW"
```

- `ip neigh add|replace <IP> lladdr <MAC> nud permanent dev <인터페이스>` : 정적 매핑
- `nud` : Neighbour Unreachability Detection 상태 지정 — `permanent` 는 만료되지 않고 갱신 요청도 무시
- `replace` 는 있으면 덮어쓰기, `add` 는 이미 있으면 오류
- `ip neigh flush dev <인터페이스>` : 캐시 비우기
- 영구화하려면 부팅 스크립트나 NetworkManager dispatcher 필요 (파일 하나로는 안 됨)

```bash
# ARP 트래픽 관찰
dnf install -y iputils tcpdump
tcpdump -nn -i enp0s1 arp -c 10 &            # 백그라운드 캡처
arping -c 3 -I enp0s1 "$GW"                   # ARP 요청 직접 발생
wait
# 스푸핑 징후: 서로 다른 IP 가 같은 MAC 을 갖는 항목
ip neigh show | awk '{print $5}' | sort | uniq -c | sort -rn | head
```

- `tcpdump -nn -i <if> arp` : 이름 해석 없이(**nn**) 지정 인터페이스의 ARP 만
- `-c <n>` : n 패킷 캡처 후 종료 (**c**ount)
- `arping -c 3 -I <if> <IP>` : ARP 요청을 직접 보내 응답 MAC 확인 — **여러 MAC 이 응답하면 스푸핑 의심**
- 탐지 도구: `arpwatch`(MAC 변경 감시), 스위치의 DAI(Dynamic ARP Inspection)

**검증**

```bash
ip neigh replace "$GW" lladdr "$GWMAC" nud permanent dev enp0s1
ip neigh show "$GW"
arping -c 2 -I enp0s1 "$GW" | tail -3
ip neigh del "$GW" dev enp0s1; ip neigh show "$GW"
```

```text
192.168.64.1 dev enp0s1 lladdr xx:xx:xx:xx:xx:xx PERMANENT
Unicast reply from 192.168.64.1 [xx:xx:xx:xx:xx:xx]  0.7ms
Sent 2 probes (1 broadcast(s))
Received 2 response(s)
192.168.64.1 dev enp0s1 lladdr xx:xx:xx:xx:xx:xx REACHABLE
```

> 📝 **시험 포인트**: 실기 r06-15 는 ARP 스푸핑 원리(ARP 캐시 위조 → MITM·스니핑) 와 대응 1가지(정적 ARP·DAI·arpwatch) 서술. 필기 FULL r03-96·r04-97 의 정답 축은 "**서로 다른 IP 가 동일한 MAC 을 갖는 항목이 다수 발견되면 스푸핑 의심**". ARP 는 암호화 대상이 아니므로 "ARP 요청을 암호화하면 방어된다" 는 오답.

### 7-11. 평문 프로토콜 위험 시연 — FTP 비밀번호 캡처

> **상황**: "왜 SSH·TLS 를 써야 하는가" 를 한 번 눈으로 보면 잊지 않는다. 루프백에서 FTP 로그인을 캡처해 비밀번호가 그대로 보이는 것을 확인한다.

⚠️ **실습 계정(dev1) 한정**. 실제 비밀번호가 화면·캡처 파일에 평문으로 남는다. 실습 후 `passwd dev1` 로 변경하고 캡처 파일을 삭제할 것. 타인의 트래픽을 동의 없이 캡처하는 것은 불법

```bash
# 터미널 1 — 루프백 21번 포트 캡처
tcpdump -A -nn -i lo port 21 -c 40 -w /root/ftp-plain.pcap &
CAP=$!
sleep 1
# 터미널 2 (또는 같은 셸) — FTP 로그인. RHEL 9 base 에 ftp 클라이언트 없음 → curl 사용
curl -s -u dev1:'<비밀번호>' ftp://127.0.0.1/ -o /dev/null || true
sleep 1
kill $CAP 2>/dev/null; wait 2>/dev/null
# 캡처 파일에서 평문 확인
tcpdump -A -r /root/ftp-plain.pcap 2>/dev/null | grep -aE 'USER |PASS |^220|^230' | head
```

- `tcpdump -A` : 패킷 페이로드를 **ASCII 로 출력** (**A**SCII) — 평문 프로토콜이면 그대로 읽힘
- `-w <파일>` : pcap 파일로 저장, `-r <파일>` : 저장 파일 읽기
- `-i lo` : 루프백 인터페이스 (자기 자신과의 통신)
- `grep -a` : 바이너리 파일도 텍스트로 취급 (**a** = text)
- FTP 는 제어 채널 21번에서 `USER`·`PASS` 명령을 **평문**으로 주고받음

```bash
# 대조군 — SSH(2222) 는 같은 방식으로 캡처해도 암호문
tcpdump -A -nn -i lo port 2222 -c 20 2>/dev/null | grep -a 'PASS' || echo "SSH: 평문 비밀번호 없음"
# 정리
shred -u /root/ftp-plain.pcap 2>/dev/null || rm -f /root/ftp-plain.pcap
passwd dev1        # 노출된 실습 비밀번호 변경
```

| 평문 프로토콜 | 포트 | 암호화 대체 | 포트 |
| --- | --- | --- | --- |
| Telnet | 23 | **SSH** | 22 |
| FTP | 21 | **SFTP**(SSH) / **FTPS**(TLS) | 22 / 990·21+TLS |
| HTTP | 80 | **HTTPS**(TLS) | 443 |
| POP3 / IMAP | 110 / 143 | POP3S / IMAPS | 995 / 993 |
| SMTP | 25 | SMTPS / STARTTLS | 465 / 587 |
| LDAP | 389 | LDAPS | 636 |
| rsh / rlogin / rcp | 514 등 | ssh / scp / rsync over ssh | 22 |
| SNMP v1/v2c | 161 | SNMP v3 | 161 |

**검증**

```bash
ls /root/ftp-plain.pcap 2>/dev/null || echo "캡처 파일 삭제됨"
ss -tlnp | grep ':21 '
sshd -T | grep -i '^port'
```

```text
# 캡처 결과 (발췌) — 평문이 그대로 보임
220 (vsFTPd 3.0.5)
USER dev1
331 Please specify the password.
PASS <비밀번호가 평문으로>
230 Login successful.

SSH: 평문 비밀번호 없음
캡처 파일 삭제됨
```

> 📝 **시험 포인트**: 필기 FULL r01-85 "통신 내용이 평문으로 전송되어 스니핑에 취약" 은 **Telnet/FTP** 서술. 스니핑은 **수동(passive) 공격** — 트래픽을 바꾸지 않아 탐지가 어렵고, 근본 대책은 차단이 아니라 **암호화**. 스위치 환경에서 스니핑하려면 ARP 스푸핑이 선행돼야 한다는 연결 고리도 자주 출제(r03-95·r07-96).

### 7-12. IDS/IPS·방화벽 유형·보안 도구 분류

> **상황**: 실습으로 만질 수 없는 개념 항목을 표로 확정한다. 필기 9·10과목 마지막 블록이 여기서 나온다.

| 구분 | 정의 | 특징 | 대표 |
| --- | --- | --- | --- |
| **IDS** | 침입 **탐지**·경보 | 수동(passive) — 탐지 후 알림. 오탐(False Positive)·미탐(False Negative) | Snort(탐지 모드), Suricata |
| **IPS** | 침입 탐지 + **차단** | 능동(inline) — 경로상에 놓여 즉시 차단 | Snort(inline), Suricata IPS |
| **HIDS** | **호스트** 기반 | 파일 무결성·로그·시스템콜 감시 | **Tripwire, AIDE**, OSSEC |
| **NIDS** | **네트워크** 기반 | 패킷 시그니처·트래픽 이상 감시 | **Snort, Suricata**, Zeek |

| 탐지 방식 | 원리 | 장점 | 단점 |
| --- | --- | --- | --- |
| **오용 탐지**(시그니처/지식 기반) | 알려진 공격 패턴과 대조 | 오탐 적음, 빠름 | **미지(0-day) 공격 탐지 불가** |
| **이상 탐지**(행위/통계 기반) | 정상 프로파일에서 벗어난 행위 탐지 | 미지 공격 탐지 가능 | **오탐 많음**, 학습 기간 필요 |

```text
# Snort 룰 형식 (필기 FULL r02-97·r10-97)
alert tcp any any -> 192.168.0.0/24 80 (msg:"WEB attack"; content:"/etc/passwd"; sid:1000001; rev:1;)
 └동작  └프로토콜 └출발지  └방향  └목적지          └포트  └옵션(메시지·탐지문자열·규칙ID·개정)
```

- 동작: `alert`(경보+로그) / `log` / `pass` / `drop`(inline 차단) / `reject`
- `->` 단방향, `<>` 양방향. `any` = 모든 주소·포트
- `sid` 는 규칙 고유 ID (로컬 규칙은 1000000 이상), `rev` 는 개정 번호

| 방화벽 유형 | 동작 계층 | 특징 |
| --- | --- | --- |
| **패킷 필터링** | 3~4 (네트워크·전송) | IP·포트·플래그만 검사. 빠르지만 상태 모름 (`iptables -j ACCEPT` 단순 규칙) |
| **상태 추적(Stateful Inspection)** | 3~4 + 연결 테이블 | 연결 상태 기억 → 응답 패킷 자동 허용 (`-m state --state ESTABLISHED,RELATED`, nftables/conntrack) |
| **애플리케이션 게이트웨이(프록시)** | 7 (응용) | 세션을 대신 맺어 내용까지 검사. 느리지만 정밀 (Squid) |
| **회선 게이트웨이(Circuit-level)** | 5 (세션) | TCP 핸드셰이크 수준 중계 (SOCKS) |
| **NGFW** | 3~7 통합 | 애플리케이션 식별·IPS·사용자 인증·TLS 검사 통합 |

| 개념 | 정의 |
| --- | --- |
| **DMZ** | 외부 공개 서버(웹·메일·DNS) 를 두는 완충 구역 — 내부망과 분리해 침해 확산 차단 |
| **허니팟** | 일부러 취약하게 만든 미끼 시스템 — 공격 기법 수집·유인 |
| **VPN** | 공중망 위에 암호화 터널 — IPSec(AH 무결성 / ESP 기밀성), SSL/TLS VPN, L2TP/IPSec, WireGuard |
| **NAC** | 접속 단말의 보안 상태를 검사해 네트워크 접근 허용·격리 |
| **SIEM** | 여러 장비 로그를 모아 상관 분석·경보 |

| 도구 | 분류 | 용도 |
| --- | --- | --- |
| `nmap` | 스캐너 | 포트·서비스·OS 탐지 |
| `tcpdump` / `wireshark` / `tshark` | 패킷 분석 | 캡처·프로토콜 해석 (`tshark` = wireshark CLI) |
| Nessus / OpenVAS | 취약점 스캐너 | 알려진 취약점 종합 점검 |
| Tripwire / **AIDE** | 무결성(HIDS) | 기준선 DB 대비 변조 탐지 |
| Snort / Suricata | NIDS/IPS | 시그니처 기반 패킷 탐지 |
| John the Ripper / hashcat | 크래킹 | 취약 비밀번호 점검 |
| `rkhunter` / `chkrootkit` | 루트킷 탐지 | 명령 교체·은닉 탐지 |
| `fail2ban` | 자동 차단 | 로그 감시 → 방화벽 차단 |
| `gnupg`(gpg) | 암호화·서명 | 파일·메일 암호화, 무결성 서명 |
| `openssl` | 암호 라이브러리·CLI | 키·인증서·해시·TLS 진단 |
| `sudo` / **PAM** | 접근 통제 | 권한 상승 통제 / 모듈형 인증 |
| **SELinux** | MAC | 강제 접근 제어 |
| `auditd` | 감사 | 시스템콜·파일 접근 기록 |

**검증**

```bash
rpm -q nmap tcpdump aide fail2ban gnupg2 openssl audit 2>&1 | head
which nmap tcpdump aide gpg openssl auditctl 2>/dev/null
```

```text
nmap-...
tcpdump-...
aide-...
/usr/bin/nmap
/usr/sbin/tcpdump
/usr/sbin/aide
/usr/bin/gpg
/usr/bin/openssl
/usr/sbin/auditctl
```

> 📝 **시험 포인트**: **IDS=탐지(수동) / IPS=탐지+차단(능동)**, **HIDS=Tripwire·AIDE / NIDS=Snort·Suricata** 매칭이 필기 FULL r03-94·r07-97·r08-98·r09-98 의 정답 축. 오용 탐지 = 알려진 공격에 강함·0-day 취약, 이상 탐지 = 그 반대 — 두 방식의 장단점이 서로 뒤바뀐 선택지가 오답으로 나온다.

---

## 8. 암호화 실습 — 해시·대칭·비대칭·서명·인증서

### 8-1. 해시 — 무결성 검증

> **상황**: 암호화의 기본 벽돌부터. 같은 파일에서 네 가지 해시를 뽑고, 1바이트만 바꿔도 값이 완전히 달라지는 것(눈사태 효과) 을 확인한다.

```bash
mkdir -p /root/crypto && cd /root/crypto
echo "linux master lab 2026" > plain.txt
md5sum    plain.txt
sha1sum   plain.txt
sha256sum plain.txt
sha512sum plain.txt
openssl dgst -sha256 plain.txt
openssl dgst -md5 -sha1 plain.txt 2>/dev/null | head -2
```

- `md5sum`(128bit) · `sha1sum`(160bit) · `sha256sum`(256bit) · `sha512sum`(512bit) : 단방향 해시
- `openssl dgst -<알고리즘> <파일>` : 같은 계산 (**d**i**g**e**st**) — `-sha256`, `-sha512`, `-md5`
- 해시는 **단방향**(복호화 불가), **고정 길이**, **충돌 저항성**이 성질. MD5·SHA-1 은 충돌이 발견돼 **무결성 검증 용도로 비권장**

```bash
# 눈사태 효과 + 검증 파일 사용법
cp plain.txt plain2.txt; echo "x" >> plain2.txt
sha256sum plain.txt plain2.txt
sha256sum plain.txt > plain.sha256
sha256sum -c plain.sha256
echo "tampered" >> plain.txt
sha256sum -c plain.sha256; echo "exit=$?"
echo "linux master lab 2026" > plain.txt      # 원복
sha256sum -c plain.sha256
```

- `sha256sum -c <목록>` : 목록의 해시와 대조 — 일치 `OK`, 불일치 `FAILED`
- `-c --quiet` : 실패만 출력, `-c --status` : 출력 없이 종료 코드로만

**검증**

```bash
sha256sum plain.txt | cut -c1-16
sha256sum -c plain.sha256
openssl dgst -sha256 plain.txt | awk '{print $NF}' | cut -c1-16
```

```text
...
plain.txt: OK
...
```

> 📝 **시험 포인트**: 실기 r04-4 가 그대로 `sha256sum image.iso`. 필기 FULL r06-96 은 "충돌이 발견된 해시(MD5) 대신" → **SHA-256/512 로 비교** 가 정답. 해시는 암호화가 아니라 **무결성** 수단 — 기밀성은 대칭·비대칭 암호의 몫.

### 8-2. 비밀번호 해시 — `openssl passwd` 와 `/etc/shadow` 의 `$6$`

> **상황**: `/etc/shadow` 두 번째 필드의 구조를 직접 만들어 대조한다. `$` 로 구분된 세 토막이 각각 무엇인지 확인한다.

```bash
openssl passwd -6 -salt LabSalt01 'ExamplePass!'      # SHA-512
openssl passwd -5 -salt LabSalt01 'ExamplePass!'      # SHA-256
openssl passwd -1 -salt LabSalt01 'ExamplePass!'      # MD5
openssl passwd -6                                      # 대화식 입력 + 랜덤 솔트
getent shadow dev1 | cut -d: -f2 | cut -c1-30
awk -F: '{print $1, substr($2,1,3)}' /etc/shadow | grep -vE '\*|!!' | head
```

- `openssl passwd` : `crypt(3)` 형식 비밀번호 해시 생성
- `-6` SHA-512 / `-5` SHA-256 / `-1` MD5 / `-apr1` Apache MD5
- `-salt <문자열>` : 솔트 고정 (생략 시 랜덤) — 같은 비밀번호도 솔트가 다르면 결과가 달라짐
- ⚠️ 명령줄에 실제 비밀번호를 넣으면 `history` 에 남음 → 운영에서는 인수 없이 대화식으로

```text
# /etc/shadow 두 번째 필드 구조
$6$<솔트>$<해시>
 │    │      └ 해시 결과 (Base64 변형)
 │    └ 솔트 — 같은 비밀번호의 해시를 다르게 만들어 레인보우 테이블 무력화
 └ 알고리즘 식별자
```

| 식별자 | 알고리즘 | 비고 |
| --- | --- | --- |
| `$1$` | MD5 | 구형, 취약 |
| `$2a$`/`$2y$` | Blowfish (bcrypt) | 일부 배포판 |
| `$5$` | SHA-256 | |
| `$6$` | **SHA-512** | **RHEL/Rocky 9 기본** (`/etc/login.defs` 의 `ENCRYPT_METHOD SHA512`) |
| `$y$` | yescrypt | 일부 배포판 기본 |
| `!` / `!!` / `*` 로 시작 | — | 잠금 / 미설정 / 로그인 불가 계정 |

```bash
grep -E '^ENCRYPT_METHOD|^SHA_CRYPT' /etc/login.defs
authselect current >/dev/null 2>&1; grep -rE 'sha512' /etc/pam.d/system-auth | head -2
```

**검증**

```bash
openssl passwd -6 -salt LabSalt01 'ExamplePass!' | cut -c1-12
grep -E '^ENCRYPT_METHOD' /etc/login.defs
getent shadow dev1 | cut -d: -f2 | cut -c1-3
```

```text
$6$LabSalt01
ENCRYPT_METHOD SHA512
$6$
```

> 📝 **시험 포인트**: 필기 FULL r04-59·r09-25 — `/etc/shadow` 의 `$6$` = **SHA-512**. `$1$`=MD5, `$5$`=SHA-256 과 세트로 암기. `/etc/shadow` 9필드(계정:암호:최종변경일:최소:최대:경고:비활성:만료:예약) 는 Part 03 참조.

### 8-3. 대칭키 암호 — `gpg -c` 와 `openssl enc`

> **상황**: 같은 키(암호) 로 잠그고 여는 방식. 다른 계정에서 암호 없이는 열 수 없다는 것까지 확인한다.

```bash
cd /root/crypto
echo "salary 2026 confidential" > secret.txt
gpg -c --cipher-algo AES256 secret.txt          # → secret.txt.gpg (대화식 암호 입력)
ls -l secret.txt secret.txt.gpg
file secret.txt.gpg
rm -f secret.txt
gpg -d secret.txt.gpg                            # 복호 → 표준출력
gpg -o secret.out -d secret.txt.gpg              # 파일로
cat secret.out
```

- `gpg -c` / `--symmetric` : **대칭키(암호 문구)** 로 암호화 — 키 쌍 불필요
- `--cipher-algo AES256` : 알고리즘 지정 (`gpg --version` 으로 지원 목록 확인)
- `-d` / `--decrypt` : 복호화, `-o <파일>` : 출력 파일
- `-a` / `--armor` : 바이너리 대신 ASCII(Base64) 출력 → `.asc`
- 산출물 `.gpg` 는 바이너리, `.asc` 는 텍스트

```bash
# 다른 계정에서 암호 없이 열기 시도 → 실패
cp secret.txt.gpg /tmp/ && chmod 644 /tmp/secret.txt.gpg
su - dev1 -c 'gpg --batch -d /tmp/secret.txt.gpg' 2>&1 | tail -3
```

```bash
# openssl 대칭 암호화
openssl enc -aes-256-cbc -pbkdf2 -salt -in secret.out -out secret.enc
ls -l secret.enc
openssl enc -aes-256-cbc -pbkdf2 -d -in secret.enc -out secret.dec
diff secret.out secret.dec && echo "복호 일치"
openssl enc -aes-256-cbc -pbkdf2 -salt -a -in secret.out -out secret.b64   # Base64 출력
head -c 60 secret.b64; echo
```

- `openssl enc -<암호>` : 대칭 암호화 (**enc**rypt) — `-aes-256-cbc`, `-aes-128-cbc`, `-des3`(레거시)
- `-pbkdf2` : 암호 문구에서 키를 유도할 때 PBKDF2 사용 — **최신 openssl 에서 사실상 필수**
- `-salt` : 솔트 사용(기본값). `-nosalt` 는 금지
- `-d` : 복호화 (**d**ecrypt)
- `-a` : Base64 인코딩 출력
- `-in` / `-out` : 입력·출력 파일
- `-k <암호>` : 명령줄로 암호 전달 — **history 노출**되므로 실습 외 금지

**검증**

```bash
ls -l secret.txt.gpg secret.enc
file secret.txt.gpg | cut -d, -f1
gpg -o - -d secret.txt.gpg 2>/dev/null | head -1
openssl enc -aes-256-cbc -pbkdf2 -d -in secret.enc | head -1
su - dev1 -c 'gpg --batch -d /tmp/secret.txt.gpg' 2>&1 | grep -ci 'passphrase\|failed'
rm -f /tmp/secret.txt.gpg
```

```text
-rw-r--r--. 1 root root ... secret.txt.gpg
secret.txt.gpg: PGP symmetric key encrypted data
salary 2026 confidential
salary 2026 confidential
1
```

> 📝 **시험 포인트**: 대칭키의 특징 — **동일 키로 암·복호**, **빠름**, **키 분배 문제**. 대표 알고리즘 **DES·3DES·AES·SEED·ARIA**(국내 표준 SEED·ARIA 도 보기로 등장). 필기 FULL r01-98·r03-98·r04-99·r09-100 이 알고리즘 분류를 직접 묻는다.

### 8-4. 비대칭키 — GPG 키 쌍 생성과 공개키 배포

> **상황**: dev1 이 키 쌍을 만들고 공개키를 ops1 에게 전달한다. 여기서부터는 일반 사용자 계정(`$`) 으로 작업한다.

```bash
# dev1 계정에서
su - dev1
$ gpg --full-gen-key
#   1) RSA and RSA  → 2) 키 길이 3072 → 3) 유효기간 1y → 이름 dev1 → 메일 dev1@lab.local
#   → 개인키 보호 암호(passphrase) 입력
$ gpg --list-keys
$ gpg --list-secret-keys
$ gpg --fingerprint dev1@lab.local
$ ls -l ~/.gnupg/
```

- `gpg --full-gen-key` : 대화식으로 알고리즘·길이·만료·UID 를 모두 지정
- `gpg --quick-gen-key '<UID>' [알고리즘] [용도] [만료]` : 한 줄 생성 — 예 `gpg --quick-gen-key 'dev1 <dev1@lab.local>' rsa3072 default 1y`
- `--list-keys` (`-k`) : 공개키고리(`pubring.kbx`), `--list-secret-keys` (`-K`) : 개인키
- `--fingerprint` : 지문 — **공개키가 진짜인지 대면·전화로 대조하는 값**
- 키링 위치 `~/.gnupg/` (권한 700 필수)

```bash
$ gpg --export -a dev1@lab.local > /tmp/dev1-pub.asc     # 공개키만 ASCII 로 내보내기
$ head -2 /tmp/dev1-pub.asc
$ chmod 644 /tmp/dev1-pub.asc
$ exit
```

- `--export -a <UID>` : **공개키** 내보내기 (**a**rmor = ASCII)
- `--export-secret-keys` : 개인키 내보내기 — 백업 외에는 절대 유출 금지
- ⚠️ **공개키만 배포**. 개인키는 어떤 경우에도 전달하지 않음

```bash
# ops1 계정에서 공개키 가져오기
su - ops1
$ gpg --import /tmp/dev1-pub.asc
$ gpg --list-keys
$ gpg --fingerprint dev1@lab.local        # dev1 이 알려준 지문과 대조
```

**검증**

```bash
su - dev1 -c 'gpg --list-keys | head -8'
su - ops1 -c 'gpg --list-keys | grep -c dev1@lab.local'
su - ops1 -c 'gpg --list-secret-keys' 2>&1 | tail -1
ls -ld /home/dev1/.gnupg
```

```text
pub   rsa3072 2026-09-04 [SC] [expires: 2027-09-04]
      ABCD1234...
uid           [ultimate] dev1 <dev1@lab.local>
sub   rsa3072 2026-09-04 [E] [expires: 2027-09-04]
1
gpg: checking the trustdb
drwx------. 3 dev1 devteam ... /home/dev1/.gnupg
```

> 📝 **시험 포인트**: `[SC]` = **S**ign + **C**ertify(주키), `[E]` = **E**ncrypt(부키). 공개키 배포 = `--export -a`, 수신 측 = `--import`. 지문(fingerprint) 대조가 중간자 공격 방지 절차라는 점이 서술형 포인트.

### 8-5. 공개키 암호화·복호화 — 기밀성 검증

> **상황**: ops1 이 dev1 의 **공개키**로 파일을 암호화하고, dev1 만 자기 **개인키**로 연다. 제3자(root) 는 못 여는 것까지 확인한다.

```bash
# ops1 계정에서 암호화
su - ops1
$ echo "ops1 -> dev1: 배포 승인 요청" > /home/ops1/msg.txt
$ gpg -e -r dev1@lab.local /home/ops1/msg.txt     # → msg.txt.gpg
$ gpg -e -a -r dev1@lab.local -o /tmp/msg.asc /home/ops1/msg.txt
$ chmod 644 /tmp/msg.asc
$ gpg -d /tmp/msg.asc 2>&1 | tail -2               # ops1 은 개인키가 없어 복호 불가
$ exit
```

- `-e` / `--encrypt` : 공개키 암호화
- `-r <수신자>` / `--recipient` : **수신자의 공개키** 지정. 여러 명이면 `-r` 반복
- `-a` : ASCII 출력, `-o` : 출력 파일
- 암호화한 본인도 (자기 키를 `-r` 에 넣지 않았다면) 복호할 수 없음 — 공개키 암호의 성질

```bash
# dev1 계정에서 복호화
su - dev1
$ gpg -d /tmp/msg.asc                              # 개인키 암호 입력 → 평문
$ gpg -o /home/dev1/msg.plain -d /tmp/msg.asc
$ cat /home/dev1/msg.plain
$ exit

# root(제3자) 는 복호 불가
gpg --batch -d /tmp/msg.asc 2>&1 | tail -2
```

| 목적 | 암호화에 쓰는 키 | 복호/검증에 쓰는 키 |
| --- | --- | --- |
| **기밀성**(남이 못 읽게) | **수신자의 공개키** | 수신자의 개인키 |
| **전자서명**(내가 썼음을 증명) | **송신자의 개인키** | 송신자의 공개키 |
| 기밀성 + 서명 | 수신자 공개키 + 송신자 개인키 | 수신자 개인키 + 송신자 공개키 |

**검증**

```bash
su - dev1 -c 'gpg -o - -d /tmp/msg.asc 2>/dev/null' | head -1
gpg --batch -d /tmp/msg.asc 2>&1 | grep -ci 'secret key\|decryption failed'
ls -l /tmp/msg.asc
```

```text
ops1 -> dev1: 배포 승인 요청
1
-rw-r--r--. 1 ops1 opsteam ... /tmp/msg.asc
```

> 📝 **시험 포인트**: 필기 FULL r02-99 — "수신자의 공개키로 파일 암호화" = **`gpg -e -r <수신자> file`**. r04-100 — 전자서명 **생성**에 쓰는 키 = **송신자의 개인키**. 이 두 문장을 뒤집은 선택지가 항상 오답으로 들어간다.

### 8-6. 전자서명 — `--clearsign` · `--detach-sign` · 변조 검증 실패

> **상황**: 내용이 바뀌지 않았음과 작성자가 누구인지를 증명한다. 서명 후 한 글자만 바꿔 검증이 실패하는 것을 확인한다.

```bash
su - dev1
$ cd ~
$ echo "release-2026.09 승인" > notice.txt

# (1) 클리어 서명 — 원문 + 서명이 한 파일에
$ gpg --clearsign notice.txt          # → notice.txt.asc
$ head -6 notice.txt.asc
$ gpg --verify notice.txt.asc

# (2) 분리 서명 — 원문은 그대로, 서명만 별도 파일
$ gpg --detach-sign -a notice.txt     # → notice.txt.asc (덮어씀 주의)
$ gpg --detach-sign -a -o notice.sig notice.txt
$ gpg --verify notice.sig notice.txt

# (3) 변조 후 검증 실패 재현
$ cp notice.txt notice.bad
$ sed -i 's/승인/반려/' notice.bad
$ gpg --verify notice.sig notice.bad ; echo "exit=$?"
$ exit
```

- `--clearsign` : 평문을 그대로 두고 아래에 서명 블록 첨부 (메일 본문용)
- `--detach-sign` : **서명만** 별도 파일로 — 배포 파일(ISO·tar) 검증에 표준
- `-s` / `--sign` : 원문을 압축·포장한 서명 파일 생성 (`--verify` 로 검증, `-d` 로 원문 추출)
- `--verify [서명파일] [원문]` : 검증. 분리 서명은 **서명 → 원문** 순서
- 검증 성공 = `Good signature`, 변조 = `BAD signature`
- `[unknown]` 신뢰 경고는 "서명은 유효하나 이 키를 신뢰한다고 표시하지 않았다" 는 뜻 → `--lsign-key` 로 로컬 신뢰 부여

```bash
# ops1 이 dev1 서명을 검증 (공개키를 이미 import 했으므로 가능)
su - ops1 -c 'cp /home/dev1/notice.txt /home/dev1/notice.sig /tmp/ 2>/dev/null; gpg --verify /tmp/notice.sig /tmp/notice.txt'
```

```bash
# 키 정리
su - dev1 -c 'gpg --list-keys'
# su - dev1 -c 'gpg --delete-secret-keys dev1@lab.local'   # 개인키 먼저
# su - dev1 -c 'gpg --delete-keys dev1@lab.local'          # 그 다음 공개키
su - ops1 -c 'gpg --delete-keys dev1@lab.local' 2>/dev/null; echo "ops1 키링 정리"
```

- `--delete-keys <UID>` : 공개키 삭제. 개인키가 있으면 먼저 `--delete-secret-keys` 를 해야 함
- `--edit-key <UID>` : 대화식 편집 (`trust`, `expire`, `adduid`, `revkey`)
- `--gen-revoke` : 폐기 인증서 생성 — 키 분실 대비로 미리 만들어 보관

**검증**

```bash
su - dev1 -c 'gpg --verify ~/notice.sig ~/notice.txt' 2>&1 | grep -E 'Good|BAD'
su - dev1 -c 'gpg --verify ~/notice.sig ~/notice.bad' 2>&1 | grep -E 'Good|BAD'; echo "exit=$?"
su - dev1 -c 'head -3 ~/notice.txt.asc'
```

```text
gpg: Good signature from "dev1 <dev1@lab.local>" [ultimate]
gpg: BAD signature from "dev1 <dev1@lab.local>" [ultimate]
exit=0
-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA512
```

> 📝 **시험 포인트**: 서명은 **무결성 + 부인방지(인증)** 를 제공하고 **기밀성은 제공하지 않는다**(원문이 그대로 보임). 배포 파일 검증 관용 절차 — 해시(`sha256sum -c`) + 서명(`gpg --verify`) 두 단계. RPM 도 같은 원리(`rpm --import` → `rpm -K`, 필기 FULL r03-31).

### 8-7. OpenSSL RSA 키·CSR·자체 서명 인증서

> **상황**: TLS 인증서를 손으로 만들어 각 파일이 무엇인지 익힌다. 개인키 → CSR → 인증서 순서를 확인한다.

```bash
cd /root/crypto
openssl genrsa -out key.pem 2048                       # RSA 개인키 (공개키 포함)
openssl rsa -in key.pem -pubout -out pub.pem           # 공개키만 추출
openssl rsa -in key.pem -noout -text | head -5         # 키 내부 구조
openssl rsa -in key.pem -noout -modulus | md5sum       # 키 지문 대조용
ls -l key.pem pub.pem; chmod 600 key.pem
```

- `openssl genrsa -out <파일> <비트수>` : RSA 개인키 생성. 최신 대안 `openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048`
- `openssl rsa -in <키> -pubout` : 개인키에서 **공개키 추출** (개인키는 공개키를 포함)
- `-noout` : 키 본문(PEM) 을 출력하지 않음
- `-text` : 사람이 읽는 형태로 내부 값 표시
- `-modulus` : 모듈러스 — 키·CSR·인증서가 **같은 쌍인지** 대조할 때 사용

```bash
# CSR (인증서 서명 요청)
openssl req -new -key key.pem -out req.csr \
  -subj "/C=KR/ST=Seoul/L=Seoul/O=LabOrg/OU=IT/CN=intranet.lab.local"
openssl req -in req.csr -noout -text | head -8
openssl req -in req.csr -noout -subject

# 자체 서명 인증서 (CA 없이 자기 키로 서명)
openssl x509 -req -days 365 -in req.csr -signkey key.pem -out cert.crt
openssl x509 -in cert.crt -noout -subject -issuer -dates -fingerprint -sha256
openssl x509 -in cert.crt -noout -text | head -12
```

- `openssl req -new` : CSR 생성 (**req**uest)
- `-subj "/C=/ST=/L=/O=/OU=/CN="` : 대화식 입력을 생략 — **C**ountry / **S**ta**T**e / **L**ocality / **O**rganization / **O**rganizational**U**nit / **C**ommon**N**ame
- **CN 은 서비스 도메인과 일치**해야 함 (현대 클라이언트는 SAN 을 더 우선)
- `-x509` 를 붙이면 CSR 대신 **자체 서명 인증서**를 한 번에 생성
- `openssl x509 -req -signkey <키>` : 자기 개인키로 서명 = 자체 서명
- `openssl x509` 조회 옵션 — `-subject`(주체) `-issuer`(발급자) `-dates`(유효기간) `-fingerprint`(지문) `-serial` `-purpose`

```bash
# 세 파일이 같은 쌍인지 확인 (modulus 해시가 모두 같아야 함)
openssl rsa  -in key.pem  -noout -modulus | openssl md5
openssl req  -in req.csr  -noout -modulus | openssl md5
openssl x509 -in cert.crt -noout -modulus | openssl md5
openssl rand -hex 16                       # 난수 생성 (세션키·솔트용)
openssl rand -base64 24
# openssl speed rsa2048                     # 벤치마크 (참고 — 수 분 소요)
```

**검증**

```bash
openssl x509 -in cert.crt -noout -subject -issuer -dates
openssl x509 -in cert.crt -noout -fingerprint -sha256 | cut -c1-40
ls -l key.pem pub.pem req.csr cert.crt
```

```text
subject=C=KR, ST=Seoul, L=Seoul, O=LabOrg, OU=IT, CN=intranet.lab.local
issuer=C=KR, ST=Seoul, L=Seoul, O=LabOrg, OU=IT, CN=intranet.lab.local
notBefore=Sep  4 ... 2026 GMT
notAfter=Sep  4 ... 2027 GMT
SHA256 Fingerprint=...
-rw-------. 1 root root ... key.pem
```

> 📝 **시험 포인트**: **자체 서명 = subject 와 issuer 가 동일**. 필기 FULL r07-71 의 공인 CA 절차 순서 — **① 개인키 생성 → ② CSR 생성 → ③ CA 에 제출·서명받기 → ④ 서버에 인증서·키 배치 → ⑤ 웹서버 설정·재시작**. r03-67·r05-97 은 `SSLCertificateFile`(인증서) / `SSLCertificateKeyFile`(개인키) 지시자 구분.

### 8-8. 자체 CA 구축 — CA 로 서명하고 신뢰 저장소에 등록

> **상황**: Part 09 의 `mod_ssl` 은 자체 서명이라 `curl -k` 가 필요했다. 사내 CA 를 만들어 서버 인증서를 서명하고, CA 인증서를 시스템 신뢰 저장소에 넣어 `-k` 없이 통과시킨다.

```bash
mkdir -p /root/ca && cd /root/ca && chmod 700 /root/ca

# ① CA 키·자체 서명 루트 인증서
openssl genrsa -out ca.key 4096
chmod 600 ca.key
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt \
  -subj "/C=KR/O=Lab Internal CA/CN=Lab Root CA"
openssl x509 -in ca.crt -noout -subject -issuer -dates
```

- `-x509` : CSR 대신 **인증서** 직접 생성 (루트 CA 는 자기 자신을 서명)
- `-nodes` : 개인키를 암호 없이 저장 (**no DES**) — 자동화용. 운영 CA 는 암호 보호 권장
- `-days 3650` : 루트 CA 는 길게 (10년)

```bash
# ② 서버 키·CSR
openssl genrsa -out intranet.key 2048
chmod 600 intranet.key
openssl req -new -key intranet.key -out intranet.csr \
  -subj "/C=KR/O=LabOrg/CN=intranet.lab.local"

# ③ SAN 확장 파일 (현대 클라이언트는 CN 이 아니라 SAN 을 본다)
cat > intranet.ext <<'EOF'
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = DNS:intranet.lab.local, DNS:srv01.lab.local, IP:192.168.64.10
EOF

# ④ CA 로 서명
openssl x509 -req -in intranet.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out intranet.crt -days 365 -sha256 -extfile intranet.ext
openssl x509 -in intranet.crt -noout -subject -issuer -dates
openssl x509 -in intranet.crt -noout -text | grep -A2 'Subject Alternative Name'
openssl verify -CAfile ca.crt intranet.crt
```

- `-CA <파일> -CAkey <키>` : 서명 주체 지정
- `-CAcreateserial` : 일련번호 파일(`ca.srl`) 생성 — 이미 있으면 `-CAserial ca.srl`
- `-extfile` / `-extensions` : 확장 필드(SAN 등) 적용
- `basicConstraints=CA:FALSE` : 이 인증서로는 다른 인증서를 서명할 수 없음
- `openssl verify -CAfile <CA> <인증서>` : 체인 검증 → `OK`

```bash
# ⑤ 웹 서버에 적치 (Part 09 mod_ssl 설정 경로)
install -o root -g root -m 600 intranet.key /etc/pki/tls/private/intranet.key
install -o root -g root -m 644 intranet.crt /etc/pki/tls/certs/intranet.crt
restorecon -v /etc/pki/tls/private/intranet.key /etc/pki/tls/certs/intranet.crt
grep -nE 'SSLCertificateFile|SSLCertificateKeyFile' /etc/httpd/conf.d/ssl.conf
sed -i 's#^SSLCertificateFile .*#SSLCertificateFile /etc/pki/tls/certs/intranet.crt#' /etc/httpd/conf.d/ssl.conf
sed -i 's#^SSLCertificateKeyFile .*#SSLCertificateKeyFile /etc/pki/tls/private/intranet.key#' /etc/httpd/conf.d/ssl.conf
apachectl configtest && systemctl restart httpd

# ⑥ CA 인증서를 시스템 신뢰 저장소에 등록
cp ca.crt /etc/pki/ca-trust/source/anchors/lab-ca.crt
update-ca-trust extract
trust list | grep -i 'Lab Root CA' -A2 | head
```

- `/etc/pki/ca-trust/source/anchors/` : 관리자가 추가하는 **신뢰 루트 CA** 위치 (PEM 또는 DER)
- `update-ca-trust extract` : 번들(`/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem`) 재생성 — **이 명령을 빼먹으면 반영 안 됨**
- `trust list` / `trust anchor` : p11-kit 기반 신뢰 저장소 조회·추가

**검증**

```bash
openssl verify -CAfile /root/ca/ca.crt /root/ca/intranet.crt
grep -c 'Lab Root CA' /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem
getent hosts intranet.lab.local          # Part 09 BIND 존 또는 /etc/hosts
curl -sS -o /dev/null -w '%{http_code} %{ssl_verify_result}\n' https://intranet.lab.local/
echo | openssl s_client -connect 127.0.0.1:443 -servername intranet.lab.local 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

```text
/root/ca/intranet.crt: OK
1
192.168.64.10   intranet.lab.local
200 0
subject=C=KR, O=LabOrg, CN=intranet.lab.local
issuer=C=KR, O=Lab Internal CA, CN=Lab Root CA
notBefore=... notAfter=...
```

- `%{ssl_verify_result}` 가 **0** = 검증 성공 → `-k` 없이 통과. 이전(자체 서명) 에는 21(`unable to verify the first certificate`) 등이 나옴
- `openssl s_client -connect <호스트:포트> -servername <SNI>` : TLS 핸드셰이크 진단 — 제시된 인증서 체인·프로토콜·암호 스위트 확인

> 📝 **시험 포인트**: 필기 FULL r03-99 — "**CA 는 자신의 개인키로 서버 인증서에 서명하며, 클라이언트는 신뢰 저장소의 루트 인증서(CA 공개키) 로 서명을 검증**". 오답 축은 "서버 인증서에 서버의 개인키가 포함되어 배포된다"(절대 아님) 와 "CRL 은 새로 발급된 인증서 목록"(폐기 목록임).

### 8-9. 암호 방식 비교와 PKI·TLS 개념 정리

> **상황**: 실습한 것을 필기 답안 형태로 압축한다. 여기 표가 8·10과목 암호 문항의 정답 근거다.

| 구분 | **대칭키(비밀키)** | **비대칭키(공개키)** | **해시** |
| --- | --- | --- | --- |
| 키 | 하나 (송·수신 공유) | 쌍 (공개키 + 개인키) | 키 없음 |
| n명 통신 시 키 개수 | **n(n-1)/2** | **2n** | — |
| 속도 | **빠름** | 느림 (대칭의 수백~수천 분의 1) | 매우 빠름 |
| 키 분배 | **문제 있음** (안전한 채널 필요) | 해결 (공개키는 공개해도 됨) | — |
| 제공 보안성 | 기밀성 | 기밀성, 인증, **부인방지** | **무결성** |
| 복호 | 가능 | 가능 | **불가(단방향)** |
| 대표 알고리즘 | **DES, 3DES, AES, SEED, ARIA**, Blowfish, RC4 | **RSA, DSA, ECC, ElGamal, DH** | **MD5, SHA-1, SHA-2(256/512), HMAC** |
| 실습 명령 | `gpg -c`, `openssl enc -aes-256-cbc` | `gpg -e -r`, `openssl genrsa` | `sha256sum`, `openssl dgst` |

- **하이브리드 방식**: 실제 TLS·GPG 는 둘을 섞어 씀 — **세션키(대칭)** 를 만들어 데이터를 빠르게 암호화하고, 그 **세션키만 상대 공개키(비대칭)로** 안전하게 전달
- **DH(Diffie-Hellman)** 는 암호화가 아니라 **키 교환** 알고리즘. ECDHE 는 타원곡선 기반 + 순방향 비밀성(PFS)
- **HMAC** 은 해시 + 비밀키 → 무결성 **+ 인증**

| 용어 | 정의 |
| --- | --- |
| **전자서명** | 원문 해시를 **송신자 개인키로 암호화**한 값. 수신자는 송신자 공개키로 풀어 해시와 대조 → 무결성·인증·부인방지 |
| **PKI** | 공개키 기반 구조 — 인증서로 "이 공개키가 이 주체의 것" 을 보증하는 체계 |
| **CA** | 인증기관 — 자신의 개인키로 인증서에 서명. 루트 CA → 중간 CA → 서버 인증서의 **체인** |
| **RA** | 등록기관 — 신원 확인을 대행 |
| **인증서(X.509)** | 주체·공개키·발급자·유효기간·SAN·CA 서명을 담은 구조. 확장자 `.crt`/`.pem`(텍스트) `.der`(바이너리) `.p12`/`.pfx`(키+인증서 묶음) |
| **CRL** | 인증서 **폐기 목록** — 유효기간 전에 무효화된 인증서 목록을 주기적으로 배포 |
| **OCSP** | 폐기 여부 **실시간 조회** 프로토콜. OCSP Stapling 은 서버가 응답을 대신 첨부 |
| **CSR** | 인증서 서명 요청 — 공개키 + 주체 정보를 담아 CA 에 제출 |
| **자체 서명** | subject = issuer. 브라우저가 자동 신뢰하지 않음 → 신뢰 저장소에 수동 등록(8-8) |

```text
# TLS 핸드셰이크 (요약)
① ClientHello    : 지원 TLS 버전·암호 스위트 목록·랜덤값 전송
② ServerHello    : 선택된 버전·암호 스위트 + 서버 랜덤값 + **서버 인증서** 전송
③ 인증서 검증     : 클라이언트가 신뢰 저장소의 CA 공개키로 서명·유효기간·도메인(SAN) 확인
④ 키 교환        : (EC)DHE 등으로 **세션키(대칭키)** 합의  ※ RSA 키 교환은 레거시
⑤ Finished       : 양측이 세션키로 검증 메시지 교환 → 핸드셰이크 완료
⑥ 응용 데이터     : 이후 모든 데이터는 **대칭키**로 암호화
```

| 프로토콜 | 계층 | 용도 |
| --- | --- | --- |
| SSL/TLS | 전송~세션 | HTTPS(443), FTPS, SMTPS. TLS 1.2/1.3 사용, SSL 2/3·TLS 1.0/1.1 은 폐기 |
| SSH | 응용 | 원격 접속 암호화. 공개키 인증 흐름은 Part 08 |
| **IPSec** | **네트워크(3계층)** | VPN — **AH**(인증·무결성) / **ESP**(기밀성+무결성). 전송 모드 / 터널 모드 |
| PGP/GPG | 응용 | 메일·파일 암호화·서명 |
| **Kerberos** | 응용 | **티켓 기반 SSO** — KDC(AS + TGS), TGT 발급 → 서비스 티켓. 시계 동기화 필수(`chronyd`, Part 08) |
| SAML / OAuth2 / OIDC | 응용 | 웹 SSO·위임 인가 |

**검증**

```bash
openssl ciphers -v 'HIGH:!aNULL' | head -5
echo | openssl s_client -connect 127.0.0.1:443 -servername intranet.lab.local 2>/dev/null \
  | grep -E 'Protocol|Cipher|Verify return code'
openssl x509 -in /root/ca/intranet.crt -noout -text | grep -E 'Signature Algorithm|Public-Key' | head -3
```

```text
TLS_AES_256_GCM_SHA384  TLSv1.3 Kx=any      Au=any   Enc=AESGCM(256) Mac=AEAD
Protocol  : TLSv1.3
Cipher    : TLS_AES_256_GCM_SHA384
Verify return code: 0 (ok)
Signature Algorithm: sha256WithRSAEncryption
Public-Key: (2048 bit)
```

> 📝 **시험 포인트**: 필기 FULL r01-97·r02-98·r07-70 의 TLS 핸드셰이크 정답 축 — "**비대칭키로 세션키를 교환하고, 이후 데이터는 대칭키로 암호화**". r05-99·r07-99·r09-99·r10-98 은 IPSec 이 **네트워크 계층**이며 **AH=인증/무결성, ESP=기밀성** 이라는 점. 대칭키 n명 통신 시 키 개수 **n(n-1)/2** 는 계산 문항으로 출제.

---

## 9. 종합 점검 스크립트 `seccheck.sh`

### 9-1. 스크립트 작성

> **상황**: 1~8절에서 손으로 친 점검 항목을 한 번에 도는 스크립트로 남긴다. 결과는 `[OK]`/`[WARN]` 으로 표시하고 날짜별 로그 파일에 저장한다.

```bash
cat > /usr/local/bin/seccheck.sh <<'EOF'
#!/bin/bash
# seccheck.sh — LAB Part 10 종합 보안 점검
# 사용법: seccheck.sh [--quiet]
# 출력  : 표준출력 + /var/log/seccheck-YYYY-MM-DD.log

set -u
LOG="/var/log/seccheck-$(date +%F).log"
BASE="/root/baseline"
WARN=0

log()  { printf '%s\n' "$*" | tee -a "$LOG"; }
ok()   { log "[OK]   $*"; }
warn() { log "[WARN] $*"; WARN=$((WARN+1)); }
sec()  { log ""; log "=== $* ==="; }

: > "$LOG"; chmod 600 "$LOG"
log "seccheck  host=$(hostname -f)  date=$(date '+%F %T')"

# 1) UID 0 계정
sec "UID 0 계정"
uid0=$(awk -F: '$3==0 {print $1}' /etc/passwd | tr '\n' ' ')
[ "$(echo "$uid0" | wc -w)" -eq 1 ] && ok "UID 0 = $uid0" || warn "UID 0 다중: $uid0"

# 2) 빈 비밀번호
sec "빈 비밀번호 계정"
empty=$(awk -F: '($2==""){print $1}' /etc/shadow | tr '\n' ' ')
[ -z "$empty" ] && ok "빈 비밀번호 없음" || warn "빈 비밀번호: $empty"

# 3) SetUID/SetGID 기준선 비교
sec "SetUID/SetGID 기준선"
if [ -f "$BASE/suid.base" ]; then
  find / -xdev \( -perm -4000 -o -perm -2000 \) -type f 2>/dev/null | sort > /tmp/suid.now.$$
  awk '{print $NF}' "$BASE/suid.base" | sort > /tmp/suid.base.$$
  d=$(comm -13 /tmp/suid.base.$$ /tmp/suid.now.$$)
  [ -z "$d" ] && ok "신규 SetUID/SetGID 없음" || warn "신규 SetUID/SetGID:"$'\n'"$d"
  rm -f /tmp/suid.now.$$ /tmp/suid.base.$$
else
  warn "기준선 없음 — find / -xdev \\( -perm -4000 -o -perm -2000 \\) -type f -exec ls -l {} \; > $BASE/suid.base"
fi

# 4) world-writable 파일
sec "world-writable 파일"
ww=$(find / -xdev -type f -perm -0002 ! -type l 2>/dev/null | head -20)
[ -z "$ww" ] && ok "world-writable 일반 파일 없음" || warn "world-writable:"$'\n'"$ww"

# 5) 로그인 실패
sec "로그인 실패"
fail=$(grep -c 'Failed password' /var/log/secure 2>/dev/null || echo 0)
[ "$fail" -lt 10 ] && ok "Failed password $fail 건" || warn "Failed password $fail 건 (무차별 대입 의심)"
lastb -n 5 2>/dev/null | head -5 | tee -a "$LOG" >/dev/null

# 6) 열린 포트
sec "열린 포트"
ports=$(ss -tulnH 2>/dev/null | awk '{print $5}' | sed 's/.*://' | sort -un | tr '\n' ' ')
ok "리스닝 포트: $ports"
allow=" 21 22 25 53 111 139 445 443 631 2049 2222 6379 8080 20048 "
for p in $ports; do
  case "$allow" in *" $p "*) : ;; *) warn "예상 외 포트: $p" ;; esac
done

# 7) SELinux
sec "SELinux"
m=$(getenforce)
[ "$m" = "Enforcing" ] && ok "getenforce = $m" || warn "getenforce = $m (Enforcing 아님)"
avc=$(ausearch -m avc -ts today 2>/dev/null | grep -c denied || true)
[ "${avc:-0}" -eq 0 ] && ok "오늘 AVC 거부 없음" || warn "오늘 AVC 거부 $avc 건 (ausearch -m avc -ts today)"

# 8) 방화벽
sec "방화벽"
st=$(firewall-cmd --state 2>&1)
[ "$st" = "running" ] && ok "firewalld = $st" || warn "firewalld = $st"
firewall-cmd --list-all --zone=internal 2>/dev/null | grep -E 'sources:|services:|ports:' | tee -a "$LOG" >/dev/null

# 9) rpm -Va 요약
sec "패키지 무결성 (rpm -Va)"
rv=$(rpm -Va 2>/dev/null | grep -E '^..5|^missing' | grep -v ' c /etc/' | head -10)
[ -z "$rv" ] && ok "변조된 바이너리 없음" || warn "rpm -Va 이상:"$'\n'"$rv"

# 10) AIDE 요약
sec "AIDE"
if [ -f /var/lib/aide/aide.db.gz ]; then
  if aide --check >/tmp/aide.$$ 2>&1; then
    ok "AIDE 차이 없음"
  else
    warn "AIDE 차이 발견: $(grep -E 'Added|Removed|Changed' /tmp/aide.$$ | tr '\n' ' ')"
  fi
  rm -f /tmp/aide.$$
else
  warn "AIDE DB 없음 — aide --init 후 aide.db.new.gz → aide.db.gz"
fi

sec "요약"
log "WARN 건수: $WARN"
log "로그: $LOG"
[ "$WARN" -eq 0 ] && exit 0 || exit 1
EOF
chmod 750 /usr/local/bin/seccheck.sh
bash -n /usr/local/bin/seccheck.sh && echo "문법 OK"
```

- `set -u` : 정의되지 않은 변수 사용 시 오류 — 오타로 인한 오작동 방지 (**u**nset)
- `tee -a <파일>` : 표준출력과 파일에 **동시** 기록 (**a**ppend)
- `comm -13 <기준> <현재>` : 두 정렬 파일 비교 — `-1` 첫 파일 전용 열 숨김, `-3` 공통 열 숨김 → **현재에만 있는 항목(신규 SetUID)** 만 남음
- `$$` : 현재 셸 PID — 임시 파일 이름 충돌 방지
- `ss -tulnH` : 헤더 없이 (**H**) → awk 처리 편의
- `chmod 750` : root 만 실행. 로그는 `chmod 600`
- `bash -n <스크립트>` : 실행하지 않고 문법만 검사 (**n**o-exec) — Part 04 참조

**검증**

```bash
ls -l /usr/local/bin/seccheck.sh
bash -n /usr/local/bin/seccheck.sh; echo "syntax exit=$?"
head -5 /usr/local/bin/seccheck.sh
```

```text
-rwxr-x---. 1 root root ... /usr/local/bin/seccheck.sh
syntax exit=0
#!/bin/bash
# seccheck.sh — LAB Part 10 종합 보안 점검
```

> 📝 **시험 포인트**: 스크립트 첫 줄 셔뱅 `#!/bin/bash` (실기 r01-9 빈칸), `bash -n` 문법 검사, `tee -a` 동시 출력, 종료 코드로 성공·실패 전달(`exit 0`/`exit 1`) 은 실기 스크립트 문항의 공통 채점 포인트.

### 9-2. 실행·검증

> **상황**: 실제로 돌려 결과를 확인한다. 일부러 경고를 만들어 `[WARN]` 이 잡히는지도 본다.

```bash
/usr/local/bin/seccheck.sh
echo "exit=$?"
ls -l /var/log/seccheck-*.log
grep -c '\[WARN\]' /var/log/seccheck-$(date +%F).log
grep '\[WARN\]' /var/log/seccheck-$(date +%F).log
```

```bash
# 경고 유도 → 탐지 확인 → 원복
chmod u+s /usr/bin/find                       # ⚠️ 실습 변조
/usr/local/bin/seccheck.sh | grep -A3 'SetUID'
chmod u-s /usr/bin/find                       # 원복
/usr/local/bin/seccheck.sh | grep -A2 'SetUID'
```

**검증**

```bash
/usr/local/bin/seccheck.sh > /dev/null; echo "exit=$?"
tail -6 /var/log/seccheck-$(date +%F).log
ls -l /var/log/seccheck-$(date +%F).log
```

```text
=== SetUID/SetGID 기준선 ===
[WARN] 신규 SetUID/SetGID:
/usr/bin/find
...
[OK]   신규 SetUID/SetGID 없음

=== 요약 ===
WARN 건수: 0
로그: /var/log/seccheck-2026-09-04.log
exit=0
-rw-------. 1 root root ... /var/log/seccheck-2026-09-04.log
```

> 📝 **시험 포인트**: 점검 스크립트의 핵심은 **기준선과의 비교** — 절대 기준(“SetUID 파일이 20개다”)이 아니라 상대 변화(“어제 없던 게 생겼다”)를 본다. 스크립트가 종료 코드를 반환하면 cron·모니터링이 실패를 자동 감지할 수 있다.

### 9-3. 주간 cron 등록

> **상황**: Part 06 에서 쓴 `/etc/cron.d/` 조각 파일 방식으로 매주 월요일 04:00 자동 실행을 건다.

```bash
cat > /etc/cron.d/lab-seccheck <<'EOF'
# 분 시 일 월 요일 사용자 명령
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
MAILTO=root
0 4 * * 1 root /usr/local/bin/seccheck.sh >> /var/log/lab-seccheck-cron.log 2>&1
EOF
chmod 644 /etc/cron.d/lab-seccheck
ls -l /etc/cron.d/
systemctl is-active crond
```

| 필드 | 값 | 의미 |
| --- | --- | --- |
| 분 | `0` | 정각 |
| 시 | `4` | 04시 |
| 일 | `*` | 매일 |
| 월 | `*` | 매월 |
| 요일 | `1` | **월요일** (0·7=일, 1=월 … 6=토) |
| 사용자 | `root` | `/etc/cron.d`·`/etc/crontab` 은 **7필드** — 사용자 필드가 있음 |
| 명령 | `/usr/local/bin/seccheck.sh …` | 절대 경로 필수 (cron 의 PATH 는 매우 짧음) |

- `>> <파일> 2>&1` : 표준출력·표준오류를 함께 누적 기록
- `MAILTO=root` : 출력이 있으면 root 에게 메일 (Part 09 Postfix 로컬 배송)
- 사용자 crontab(`crontab -e`) 은 **6필드**(사용자 필드 없음) — 필드 개수 차이가 출제 포인트

```bash
# run-parts 를 거치지 않고 직접 실행해 동작 확인
/usr/local/bin/seccheck.sh > /dev/null 2>&1; echo "manual exit=$?"
# cron 이 파일을 읽었는지 확인
journalctl -u crond --since '5 min ago' --no-pager | tail -5
grep -R 'seccheck' /var/log/cron 2>/dev/null | tail -3
```

**검증**

```bash
cat /etc/cron.d/lab-seccheck | grep -v '^#'
ls -l /etc/cron.d/lab-seccheck
systemctl is-active crond
ls -l /var/log/seccheck-*.log
```

```text
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
MAILTO=root
0 4 * * 1 root /usr/local/bin/seccheck.sh >> /var/log/lab-seccheck-cron.log 2>&1
-rw-r--r--. 1 root root ... /etc/cron.d/lab-seccheck
active
-rw-------. 1 root root ... /var/log/seccheck-2026-09-04.log
```

> 📝 **시험 포인트**: `/etc/cron.d/*` 와 `/etc/crontab` 은 **사용자 필드 포함 7필드**, `crontab -e` 로 만든 사용자 crontab 은 **6필드**. 요일 `1` = 월요일, `0`·`7` = 일요일. `/etc/cron.d` 파일은 **실행 권한이 아니라 읽기 권한(644)** 이면 되고, 소유자는 root.

---

## 10. 마무리 — 최종 상태 확인과 위험 설정 원복

### 10-1. 최종 상태 점검

> **상황**: Part 10 에서 바꾼 모든 축(방화벽·SELinux·sshd·리스닝) 이 의도한 상태인지 한 번에 확인한다.

```bash
# 방화벽
firewall-cmd --state
firewall-cmd --get-active-zones
firewall-cmd --list-all --zone=internal
firewall-cmd --list-all --zone=public
firewall-cmd --list-rich-rules --zone=internal
# SELinux
getenforce
sestatus | grep -E 'status|Current mode|Mode from config|Loaded policy'
semanage boolean -l -C
semanage port -l -C
semanage fcontext -l -C
# 리스닝
ss -tulnp
# sshd 최종 적용값
sshd -T | grep -E '^permitrootlogin|^passwordauthentication|^pubkeyauthentication|^port|^maxauthtries|^logingracetime|^banner|^allowgroups'
# 감사·PAM
auditctl -l
authselect current
# 커널 파라미터
sysctl -a 2>/dev/null | grep -E 'tcp_syncookies|all.rp_filter|ip_forward$'
```

**검증**

```bash
getenforce
firewall-cmd --state
firewall-cmd --list-services --zone=public
sshd -T | grep -E '^permitrootlogin|^port'
ss -tlnH | awk '{print $4}' | sed 's/.*://' | sort -un | tr '\n' ' '
```

```text
Enforcing
running
dns http https
permitrootlogin no
port 2222
21 25 53 80 111 139 443 445 631 2049 2222 6379 8080 ...
```

> 📝 **시험 포인트**: 상태 확인 3종 세트 — `getenforce`(SELinux) · `firewall-cmd --state`/`--list-all`(방화벽) · `ss -tulnp`(리스닝). 실기 서술형에서 "현재 보안 설정을 확인하는 명령" 을 물으면 이 조합을 답하면 된다.

### 10-2. 외부 시점 재스캔 (macOS 에서)

> **상황**: 1-2 에서 서버 자신이 찍은 스캔은 방화벽 통과 후 모습이었다. 이번엔 macOS 호스트(192.168.64.1) 에서 다시 스캔해 **internal 존에서 열린 포트만** 보이는지 확인한다.

```bash
# macOS 터미널 (호스트는 192.168.64.0/24 = internal 존)
# $ nmap -p- 192.168.64.10                 # 전체 포트 (수 분 소요)
# $ nmap -sT -p 1-1024,2222,6379,8080 192.168.64.10
# $ nmap -sV -p 2222,80,443 192.168.64.10
# $ nmap -sU -p 53,111,137,138,2049 192.168.64.10
```

```bash
# 서버 쪽에서 스캔 흔적 확인
journalctl -k --since '10 min ago' | grep -iE 'martian|DROP|REJECT|FTP ' | tail -5
ss -tan state syn-recv | head
aureport --summary | head -6
```

| 항목 | 기대 결과 | 어긋나면 |
| --- | --- | --- |
| 22/tcp | **closed/filtered** (sshd 는 2222 만 리슨) | sshd 설정 확인 |
| 2222/tcp | open (internal 존 rich rule) | `firewall-cmd --list-rich-rules --zone=internal` |
| 80·443/tcp | open (public·internal 모두 허용) | — |
| 53/tcp·udp | open (내부 DNS) | 외부 노출 시 재귀 질의 제한 확인(Part 09) |
| 445·139·2049·111·21 | open **(internal 존에서만)** | public 존에 남아 있으면 2-3 재수행 |
| 25/tcp | 127.0.0.1 만 → **외부에서 안 보임** | `postconf inet_interfaces` |
| 631/tcp | 127.0.0.1 만 (cups 중지했다면 없음) | 6-8 |
| 6379/tcp | Part 11 컨테이너 기동 후에만 | — |
| 그 외 | **없어야 함** | `ss -tulnp` 로 주인 확인 |

- macOS 호스트는 `192.168.64.1` 이므로 **internal 존**으로 분류됨 → 여기서 보이는 것이 "사내망 시점"
- 진짜 "외부 시점" 은 다른 대역에서 와야 하지만 UTM Shared Network 단일 서브넷 환경에서는 재현 불가 → `firewall-cmd --list-all --zone=public` 의 서비스 목록이 곧 외부 노출 범위 (**※ 미실행** 대체 확인)

**검증**

```bash
firewall-cmd --list-services --zone=public
firewall-cmd --list-ports --zone=public
firewall-cmd --list-services --zone=internal
ss -tlnp | grep -E '127.0.0.1:(25|631)'
```

```text
dns http https
(포트 없음)
dns ftp http https lab-redis mountd nfs rpc-bind samba
LISTEN 0 100 127.0.0.1:25 ... users:(("master",...))
```

> 📝 **시험 포인트**: 필기 FULL r08-91 은 nmap 결과 해석 — `open`(응답 있음) / `closed`(RST 응답, 호스트는 살아 있음) / `filtered`(무응답 = 방화벽 DROP). **DROP 은 filtered, REJECT 는 closed 로 보인다**는 차이가 3-4 실습과 연결된다.

### 10-3. 실습으로 켠 위험 설정 원복

> **상황**: 학습을 위해 일부러 켠 설정을 남겨 두면 그대로 취약점이 된다. 체크리스트로 되돌린다.

⚠️ 아래 항목 중 **하나라도 남아 있으면 시스템이 실습 전보다 취약함** — 하나씩 확인하며 원복

```bash
# ① SELinux 불린 — 실습으로 켠 것 전부 off
semanage boolean -l -C
setsebool -P ftpd_full_access off        2>/dev/null
setsebool -P ftpd_anon_write off         2>/dev/null
setsebool -P nfs_export_all_rw off       2>/dev/null
setsebool -P samba_export_all_rw off     2>/dev/null
setsebool -P httpd_can_network_connect off 2>/dev/null
semanage boolean -l -C

# ② SELinux 포트·컨텍스트 실습 잔재
semanage port -l -C
semanage port -d -t http_port_t -p tcp 8888 2>/dev/null
semanage fcontext -l -C

# ③ SELinux 모드 — 반드시 Enforcing
setenforce 1
grep '^SELINUX=' /etc/selinux/config
getenforce

# ④ 방화벽 — 실습용 포트·규칙 제거
firewall-cmd --permanent --zone=internal --remove-port=8888/tcp 2>/dev/null
firewall-cmd --permanent --zone=internal --remove-forward-port=port=80:proto=tcp:toport=8080 2>/dev/null
firewall-cmd --permanent --zone=internal --remove-masquerade 2>/dev/null
firewall-cmd --reload
firewall-cmd --list-all --zone=internal | grep -E 'masquerade|forward-ports|ports:'
firewall-cmd --panic-off 2>/dev/null; firewall-cmd --query-panic

# ⑤ iptables 잔재 (3-12 에서 이미 정리 — 재확인)
systemctl is-enabled iptables 2>/dev/null || echo "iptables.service disabled"
iptables -L -n | head -4
sysctl net.ipv4.ip_forward

# ⑥ SetUID 변조 원복 (7-2, 9-2)
ls -l /usr/bin/find | cut -c1-11
find / -xdev -perm -4000 -type f 2>/dev/null | sort > /tmp/suid.chk
diff <(awk '{print $NF}' /root/baseline/suid.base | sort) <(sort /tmp/suid.chk) | head
rm -f /tmp/suid.chk

# ⑦ 평문 캡처 파일·노출된 비밀번호 (7-11)
ls /root/ftp-plain.pcap 2>/dev/null && shred -u /root/ftp-plain.pcap
# passwd dev1     ← 캡처로 노출된 실습 비밀번호 변경

# ⑧ 익명 FTP·NFS no_root_squash 등 Part 09 실습 설정
grep -E '^anonymous_enable|^anon_upload_enable|^anon_mkdir_write_enable' /etc/vsftpd/vsftpd.conf
grep -n 'no_root_squash' /etc/exports 2>/dev/null || echo "no_root_squash 없음"
# 필요 시 Part 09 로 돌아가 root_squash 로 복구 후 exportfs -ra

# ⑨ pam_faillock — even_deny_root 유지 여부 결정
grep -E '^even_deny_root|^deny|^unlock_time' /etc/security/faillock.conf
faillock            # 잠긴 계정 없는지
# faillock --user <계정> --reset

# ⑩ 임시 계정·실습 파일
getent passwd tmpx || echo "tmpx 없음"
ls /root/crypto /root/ca /root/selinux-mod /root/baseline 2>/dev/null
```

| 원복 대상 | 확인 명령 | 정상 상태 |
| --- | --- | --- |
| SELinux 모드 | `getenforce` | `Enforcing` |
| 위험 불린 | `semanage boolean -l -C` | 출력 없음(기본값과 동일) |
| 실습 포트 라벨 | `semanage port -l -C` | 8888 없음 (2222 는 Part 08 것이므로 유지) |
| firewalld 마스커레이드·포워딩 | `firewall-cmd --list-all --zone=internal` | `masquerade: no`, `forward-ports:` 비어 있음 |
| 패닉 모드 | `firewall-cmd --query-panic` | `no` |
| iptables.service | `systemctl is-enabled iptables` | `disabled` |
| `ip_forward` | `sysctl net.ipv4.ip_forward` | `0` |
| `/usr/bin/find` SetUID | `ls -l /usr/bin/find` | `-rwxr-xr-x` |
| FTP 평문 캡처 | `ls /root/ftp-plain.pcap` | 없음 |
| 익명 FTP 쓰기 | `grep anon_upload_enable /etc/vsftpd/vsftpd.conf` | `NO` 또는 주석 |
| NFS `no_root_squash` | `grep no_root_squash /etc/exports` | 없음 |
| 실습 계정 `tmpx` | `getent passwd tmpx` | 없음 |
| sshd 백업 | `ls /root/sshd_config.bak.*` | 존재(롤백 대비) |

**검증**

```bash
getenforce
semanage boolean -l -C | wc -l
semanage port -l -C
firewall-cmd --query-panic; firewall-cmd --list-all --zone=internal | grep -E 'masquerade|forward-ports'
sysctl -n net.ipv4.ip_forward
ls -l /usr/bin/find | cut -c1-11
/usr/local/bin/seccheck.sh | tail -4
```

```text
Enforcing
0
(출력 없음)
no
  masquerade: no
  forward-ports:
0
-rwxr-xr-x
=== 요약 ===
WARN 건수: 0
로그: /var/log/seccheck-2026-09-04.log
```

> 📝 **시험 포인트**: "실습·디버깅을 위해 완화한 설정을 되돌린다" 는 실무 절차 그 자체. 특히 **`setenforce 0` 상태 방치**, **`ftpd_full_access`·`no_root_squash` 잔존**, **`iptables -P INPUT ACCEPT` 상태로 방치**는 감점·사고의 3대 원인.

---

## 체크리스트

| 수행 항목 | 명령 | 확인 방법 | ☐ |
| --- | --- | --- | --- |
| 리스닝 포트 전수 파악 | `ss -tulnp` | 서비스별 포트·프로세스 매칭 | ☐ |
| 자기 IP 포트 스캔 | `nmap -sT -p 1-1024,2222 192.168.64.10`, `-sS`, `-sU`, `-sV` | open 목록이 예상과 일치 | ☐ |
| firewalld 존 구조 조회 | `--state`, `--get-default-zone`, `--get-active-zones`, `--list-all` | 존·서비스·포트 확인 | ☐ |
| 서비스 정의 파일 확인 | `/usr/lib/firewalld/services/*.xml` | 서비스명 ↔ 포트 대응 | ☐ |
| 런타임 vs 영구 반영 | `--add-service` / `--permanent` + `--reload` / `--runtime-to-permanent` | `--list-all` 과 `--permanent --list-all` 비교 | ☐ |
| internal 존 소스 바인딩 | `--permanent --zone=internal --add-source=192.168.64.0/24` | `--get-active-zones` 에 sources 표시 | ☐ |
| public 존 축소 | `--zone=public --remove-service={…}` + `--add-service={http,https,dns}` | `--list-all --zone=public` = dns http https | ☐ |
| Rich Rule 작성 | `--add-rich-rule='rule family=ipv4 source address=… accept'` | `--list-rich-rules`, 접속 테스트 | ☐ |
| 포트 포워딩·마스커레이드 | `--add-forward-port=port=80:proto=tcp:toport=8080`, `--add-masquerade` | macOS 에서 접속 확인 | ☐ |
| 커스텀 서비스 정의 | `--new-service=lab-redis` + `--add-port` | `--info-service=lab-redis` | ☐ |
| iptables 전환 | `systemctl mask firewalld` + `enable --now iptables` | `iptables -L -n -v` | ☐ |
| 기본 정책 DROP + 필수 허용 | `-P INPUT DROP`, `-A INPUT -i lo -j ACCEPT`, `-m state --state ESTABLISHED,RELATED` | 접속 유지 확인 | ☐ |
| DROP vs REJECT 체감 | `-j DROP` / `-j REJECT` | `nc -zv` 타임아웃 vs 즉시 거부 | ☐ |
| 규칙 삽입·삭제·교체 | `-I`, `-D <체인> <번호>`, `-R`, `--line-numbers` | `iptables -L --line-numbers` | ☐ |
| 사용자 정의 체인 + LOG | `-N LOGDROP`, `-j LOG --log-prefix` | `journalctl -k -g` | ☐ |
| NAT 규칙 | `-t nat -A POSTROUTING -j MASQUERADE`, `PREROUTING -j DNAT/REDIRECT` | `iptables -t nat -L -n` | ☐ |
| 규칙 저장·복원 | `iptables-save > /etc/sysconfig/iptables`, `iptables-restore` | 파일 내용 확인 | ☐ |
| firewalld 복귀 | `unmask` + `enable --now firewalld` | `firewall-cmd --state` = running | ☐ |
| SELinux 모드 확인 | `getenforce`, `sestatus -v` | `Enforcing` / `targeted` | ☐ |
| 일시 전환·복귀 | `setenforce 0` → `setenforce 1` | `sestatus` 의 Current vs config 두 줄 | ☐ |
| 영구 설정 확인 | `/etc/selinux/config` `SELINUX=`·`SELINUXTYPE=` | `grep -E '^SELINUX'` | ☐ |
| 재라벨링 트리거 이해 | `/.autorelabel`, `fixfiles -F onboot` (※ 미실행) | 절차 숙지 | ☐ |
| 컨텍스트 4필드 조회 | `ls -Z`, `ls -Zd`, `ps -eZ`, `id -Z` | user:role:type:level 확인 | ☐ |
| TE·도메인 규칙 조회 | `seinfo --stats`, `sesearch -A -s httpd_t -t httpd_sys_content_t` | 허용 규칙 유무 | ☐ |
| 위반 재현 (403) | `/root/report.html` → `mv /var/www/html/` → `curl` | 403 + `ls -Z` 가 `admin_home_t` | ☐ |
| AVC 추적 | `ausearch -m avc -ts recent`, `audit2allow -w`, `sealert -a` | denied 레코드 확인 | ☐ |
| restorecon 으로 해결 | `restorecon -v <파일>` → `curl` | 컨텍스트 변경 + 200 | ☐ |
| cp / cp -a / mv 상속 차이 | 세 방법으로 복사 후 `ls -Z` + `curl` | 200 / 403 / 403 → restorecon 후 전부 200 | ☐ |
| chcon(일시) vs semanage(영구) | `chcon -t` → `restorecon` 로 원복 → `semanage fcontext -a -t` + `restorecon` | `semanage fcontext -l -C` | ☐ |
| 규칙 삭제·동등 경로 | `semanage fcontext -d`, `-a -e /var/www /srv/www`, `matchpathcon` | `-l -C` Equivalence 섹션 | ☐ |
| dry-run 확인 | `restorecon -Rv -n <경로>` | 변경 예정 목록 | ☐ |
| 포트 라벨 실습 | `Listen 8888` → restart 실패 → `semanage port -a -t http_port_t -p tcp 8888` → 성공 | `ss -tlnp \| grep 8888`, curl 200 | ☐ |
| 불린 조회·토글 | `getsebool -a \| grep httpd`, `semanage boolean -l`, `setsebool -P` | `semanage boolean -l -C` | ☐ |
| 정책 모듈 개념 | `semodule -l`, `audit2allow -M`(⚠️ 미설치) | `.te` 내용 확인 | ☐ |
| MAC vs DAC·사용자 매핑 | `semanage login -l`, `semanage user -l` | `__default__` → `unconfined_u` | ☐ |
| auditd 상태·중지 특이점 | `systemctl stop auditd`(거부) → `service auditd reload` | `is-active` active | ☐ |
| auditd.conf 확인 | `grep -v '^#' /etc/audit/auditd.conf` | `max_log_file`·`num_logs`·`space_left_action` | ☐ |
| 감사 규칙 추가·재현 | `auditctl -w /etc/passwd -p wa -k passwd_change` → `useradd tmpx` → `ausearch -k` | SYSCALL 레코드 확인 후 `userdel` | ☐ |
| 영구 규칙 등록 | `/etc/audit/rules.d/50-lab.rules` + `augenrules --load` | `auditctl -l` 목록 | ☐ |
| 감사 보고서 | `aureport --summary`, `-f`, `-au`, `-l`, `ausearch -m USER_LOGIN --success no` | 요약 출력 | ☐ |
| TCP Wrapper 미지원 확인 | `ldd /usr/sbin/sshd \| grep -c wrap` | `0` (※ 미실행 — 필기 문법·순서만 정리) | ☐ |
| hosts.allow/deny 순서 암기 | — | allow → deny → 둘 다 없으면 **허용** | ☐ |
| sshd 접근 제어 대체 | `AllowGroups`, `DenyUsers`, `Match Address` + firewalld rich rule | `sshd -T \| grep allowgroups` | ☐ |
| sshd 강화 항목 | `MaxAuthTries 3`, `LoginGraceTime 30`, `ClientAlive*`, `Banner`, `X11Forwarding no` | `sshd -t` 통과 + `sshd -T` 확인 | ☐ |
| 실패 로그인 집계 | `lastb`, `grep -c 'Failed password' /var/log/secure`, IP 집계 | 건수·출발지 IP | ☐ |
| pam_faillock 잠금·해제 | `authselect enable-feature with-faillock`, `/etc/security/faillock.conf` | 3회 실패 → `faillock --user dev1` → `--reset` | ☐ |
| fail2ban (선택) | `/etc/fail2ban/jail.local` + `fail2ban-client status sshd` | jail 활성 확인 | ☐ |
| 배너 3종 | `/etc/issue`, `/etc/issue.net`, `/etc/motd` | 콘솔·SSH 인증 전·로그인 후 표시 | ☐ |
| 미사용 계정·서비스 점검 | `awk -F: '$7 !~ /nologin\|false/' /etc/passwd`, `systemctl list-units --state=running` | 불필요 서비스 `disable --now` | ☐ |
| 세션 타임아웃 | `/etc/profile.d/tmout.sh` `TMOUT=600` + `readonly` | 새 로그인에서 `echo $TMOUT` | ☐ |
| sudo 감사 | `journalctl _COMM=sudo`, `grep sudo /var/log/secure` | COMMAND= 레코드 | ☐ |
| SUID 기준선 생성 | `find / -xdev \( -perm -4000 -o -perm -2000 \) -type f -exec ls -l {} \;` | `/root/baseline/suid.base` | ☐ |
| 변조 → diff 탐지 → 원복 | `chmod u+s /usr/bin/find` → 재수집 → `diff` → `chmod u-s` | diff 출력 → 원복 후 일치 | ☐ |
| 위험 파일·계정 점검 | world-writable, `-nouser`, UID 0, 빈 비밀번호, `/tmp` 숨김 | 각 항목 0건 | ☐ |
| 패키지 무결성 | `rpm -Va \| grep -E '^..5\|^missing'` | 바이너리 이상 없음 | ☐ |
| 해시 기준선 | `sha256sum /usr/bin/* > bin.sha256` → `sha256sum -c --quiet` | 실패 0건 | ☐ |
| AIDE 한 바퀴 | `aide --init` → DB 승격 → 변조 → `--check` → `--update` | "found differences" → "NO differences" | ☐ |
| 불변 속성·로그인 이력 | `lsattr /etc/passwd`, `last -x`, `lastb`, `lastlog -b 90`, `who`, `w` | 각 출력 확인 | ☐ |
| 의심 포트·프로세스·cron | `ss -tanp`, `lsof -i -P -n`, `lsof +L1`, `ps -ef --forest`, cron 4곳 | 예상 외 항목 없음 | ☐ |
| rkhunter (선택) | `--propupd` → `--check --sk --rwo` | Suspect files 0 | ☐ |
| sysctl 하드닝 | `/etc/sysctl.d/90-hardening.conf` + `sysctl -p` | `sysctl -a \| grep` 로 값 반영 | ☐ |
| 정적 ARP·arping·tcpdump | `ip neigh replace … nud permanent`, `arping -c 3`, `tcpdump -nn arp` | `PERMANENT` 표시 후 `ip neigh del` | ☐ |
| 평문 프로토콜 시연 | `tcpdump -A -i lo port 21` + `curl ftp://` | USER/PASS 평문 확인 → 캡처 파일 삭제 | ☐ |
| 해시 4종 + 검증 파일 | `md5sum`/`sha1sum`/`sha256sum`/`sha512sum`, `-c` | `OK` / `FAILED` | ☐ |
| shadow 해시 형식 | `openssl passwd -6 -salt` ↔ `/etc/shadow` `$6$` | 식별자 일치 | ☐ |
| 대칭키 암호화 | `gpg -c --cipher-algo AES256`, `openssl enc -aes-256-cbc -pbkdf2` | 복호 일치, 타 계정 실패 | ☐ |
| 비대칭 키 생성·배포 | `gpg --full-gen-key`, `--export -a`, `--import`, `--fingerprint` | ops1 키링에 dev1 공개키 | ☐ |
| 공개키 암호화·복호 | `gpg -e -r dev1@lab.local` → dev1 `gpg -d` | 평문 복원, root 는 실패 | ☐ |
| 전자서명·변조 검증 | `--clearsign`, `--detach-sign`, `--verify` | Good → BAD signature | ☐ |
| RSA 키·CSR·자체 서명 | `openssl genrsa`, `req -new`, `x509 -req -signkey` | subject = issuer | ☐ |
| 자체 CA + 신뢰 등록 | CA 생성 → 서버 CSR 서명 → `/etc/pki/ca-trust/source/anchors/` → `update-ca-trust extract` | `curl https://intranet.lab.local` 이 `-k` 없이 200 | ☐ |
| TLS 진단 | `openssl s_client -connect 127.0.0.1:443 -servername …` | `Verify return code: 0 (ok)` | ☐ |
| seccheck.sh 작성·실행 | `/usr/local/bin/seccheck.sh` | `bash -n` 통과, `[OK]`/`[WARN]` 출력 | ☐ |
| 주간 cron 등록 | `/etc/cron.d/lab-seccheck` `0 4 * * 1 root …` | 파일 644, `crond` active | ☐ |
| 최종 상태 확인 | `getenforce`, `firewall-cmd --list-all`, `ss -tulnp`, `sshd -T` | Enforcing·running·의도한 포트만 | ☐ |
| 외부 시점 재스캔 | macOS `nmap -p- 192.168.64.10` | internal 존 허용 포트만 open | ☐ |
| 위험 설정 원복 | 불린 off, 포트 라벨 삭제, masquerade 해제, SetUID 원복, 캡처 삭제 | `seccheck.sh` WARN 0 | ☐ |

---

## 기출 연결

| 기출 (문제집·회차·번호 또는 주제) | 이 파트의 단계 |
| --- | --- |
| 실기 r01-12, r02-12, r03-12 — iptables 규칙 한 줄 해석 서술 | 3-11 문제 1~5 |
| 실기 r04-12 — 두 iptables 규칙의 **매칭 순서** 근거 서술 | 3-7, 3-11 문제 5 |
| 실기 r06-2 — `iptables -P INPUT DROP` 작성 | 3-3 |
| 실기 r06-9 — 연결 상태 추적 매치 모듈 = `state`(`conntrack`) | 3-3, 3-11 문제 2 |
| 실기 r06-13 — NAT 규칙 해석(테이블·체인·최종 결과) | 3-9, 3-11 문제 3·4 |
| 실기 r02-9, r05-4 — firewalld `--permanent` + `--reload` | 2-1 |
| 실기 r03-4 — firewalld 실행 상태 확인 (`systemctl is-active firewalld`) | 1-3, 10-1 |
| 실기 r02-4 — ESTABLISHED 연결 조회 `ss -tan state established` | 7-7 |
| 필기 FULL r01-93, r02-95, r03-92, r04-92, r05-94, r09-97, r10-94, r10-95 — iptables 규칙 해석 | 3-11 |
| 필기 FULL r01-94, r04-91 — iptables 옵션(`-A`/`-I`/`-D`/`-F`/`-P`/`-j`) 의미 | 3-2, 3-5 |
| 필기 FULL r04-93 — 발신지에 거부 응답을 보내는 타깃 = `REJECT` | 3-4 |
| 필기 FULL r02-96 — ICMP 에코 요청 차단 iptables 규칙 | 2-6, 3-8 |
| 필기 FULL r05-93 — 메모리의 iptables 규칙을 파일로 저장 = `iptables-save` / `service iptables save` | 3-10 |
| 필기 FULL r07-91 — 포워딩 패킷의 체인 통과 순서 (PREROUTING→FORWARD→POSTROUTING) | 3-2 |
| 필기 FULL r07-92 — 22번 SSH 허용 규칙 | 3-3 |
| 필기 FULL r08-93 — `iptables -L -n -v` 출력 해석 | 3-3, 3-7 |
| 필기 FULL r03-93, r08-94 — firewalld 존·`firewall-cmd` 출력 해석 | 1-3, 2-2, 2-3 |
| 실기 r01-7 — SELinux 일시 Permissive 전환 = `setenforce 0` | 4-2 |
| 실기 r04-6 — 현재 SELinux 적용 모드 확인 = `getenforce` | 4-1 |
| 필기 FULL r05-57, r09-56, r10-58 — 재부팅 없이 permissive 전환 / `setenforce 0` 직후 `getenforce` 출력 | 4-2 |
| 필기 FULL r05-56 — `/etc/selinux/config` `SELINUX=enforcing` 의 의미 | 4-3 |
| 필기 FULL r07-58 — 영구 변경은 config, **disabled 전환은 재부팅 필요** | 4-2, 4-3 |
| 필기 FULL r06-58 — `semanage fcontext -a -t httpd_sys_content_t "/srv/www(/.*)?"` 후 `restorecon -Rv` | 4-10 |
| 필기 FULL r08-58 — `sestatus` 출력 해석 (enabled / current mode) | 4-1 |
| 필기 FULL r01-65 — 파일 목록 + SELinux 컨텍스트 = `ls -Z` | 4-4 |
| 필기 FULL r09-57 — 컨텍스트 `system_u:object_r:httpd_sys_content_t:s0` 의 **세 번째 필드(type)** | 4-4, 4-5 |
| 필기 FULL r09-58 — 컨텍스트를 정책 기본값으로 복원 = `restorecon -Rv /var/www/html` | 4-8 |
| 필기 FULL r03-56, r10-59 — `chcon` 은 relabel 시 원복 / `semanage fcontext` 는 영구 / `-a` 만으론 기존 파일 미적용 | 4-9, 4-10 |
| 필기 FULL r03-57, r09-59 — 불린 영구 변경 = `setsebool -P` | 4-13 |
| 필기 FULL r09-18 — 강제 접근 제어(MAC) 구현 = SELinux·AppArmor | 4-15 |
| 필기 FULL r06-58 — SELinux 차단 시 올바른 대응(비활성화·패키지 제거가 아님) | 4-7, 4-8, 4-14 |
| 필기 FULL r03-64 — auditd: `auditctl -w … -p wa` 의미, 규칙은 재부팅 시 소멸 | 5-3, 5-4 |
| 필기 FULL r09-61 — `auditctl -w /etc/shadow -p wa -k shadow_watch` 해석 | 5-3 |
| 실기 r03-14, r06-14 — TCP Wrapper 검사 순서·우선 규칙 서술 + 설정 해석 | 6-1 (기출 해석 예 1·3) |
| 필기 FULL r05-91 — `/etc/hosts.allow` 의 `sshd : 192.168.1.` 의미(프리픽스) | 6-1 (기출 해석 예 2) |
| 필기 FULL r03-59, r05-92, r07-62, r08-61, r10-60 — allow → deny → 둘 다 없으면 **허용** | 6-1 순서표 |
| 필기 FULL r09-43 — hosts.allow `sshd: 192.168.10.` + hosts.deny `ALL: ALL` 최종 결과 | 6-1 (기출 해석 예 1) |
| 실기 r04-15 — PAM control 4종(`required`/`requisite`/`sufficient`/`optional`) 차이 서술 | 6-5 (Part 03 PAM 연계) |
| 필기 FULL r02-60, r03-58, r05-58, r05-59, r07-56, r09-38 — PAM 제어 플래그·`auth` 타입 해석 | 6-5 |
| 필기 FULL r06-57 — 5회 실패 시 10분 잠금 모듈 = `pam_faillock` | 6-5 |
| 필기 FULL r09-39 — `pam_faillock.so preauth deny=5 unlock_time=600` 해석 | 6-5 |
| 필기 FULL r09-27 — wheel 그룹만 `su` 허용 = `/etc/pam.d/su` 의 `pam_wheel.so` | 6-8 |
| 필기 FULL r09-30 — `/etc/securetty` 의 역할 (RHEL 9 에는 부재) | 6-8 |
| 필기 FULL r01-86, r07-100 — `sshd_config` `PermitRootLogin no` | 6-3 |
| 필기 FULL r02-65, r05-88, r06-83, r09-85, r10-64 — `sshd_config` 종합 효과 해석 | 6-3 |
| 필기 FULL r05-95, r08-95 — fail2ban `jail.local` 값·`fail2ban-client status sshd` 해석 | 6-6 |
| 필기 FULL r04-56, r05-63, r06-60, r07-63, r09-60 — `/var/log/btmp` 조회 = `lastb` | 6-4, 7-6 |
| 실기 r03-1, r05-1 — `grep -c 'Failed password' /var/log/secure` | 6-4 |
| 실기 r02-1, r05-3 — 루트 이하 SetUID 일반 파일 검색 `find / -perm -4000 -type f` | 7-1 |
| 필기 FULL r04-65, r05-65, r06-62, r09-31 — SetUID 검색 명령·`-perm -4000` 의미 | 7-1, 7-2 |
| 실기 r02-13, r03-6 — SetUID/SetGID 8진수 값과 효과 | 7-1 (Part 03 연계) |
| 실기 r04-8, r05-10 — `chattr +a` / `chattr +i` 빈칸 | 7-6 (Part 03 연계) |
| 필기 FULL r03-26, r06-61, r09-35 — `chattr +i` / `lsattr` 속성 | 7-6 |
| 필기 FULL r10-38 — `rpm -V` 출력 `S.5....T.  c` 해석 | 7-4 |
| 필기 FULL r10-63 — Tripwire·AIDE = 기준 DB 대비 해시·속성 비교 | 7-5 |
| 실기 r05-14 — SYN Flooding 원리와 방어 기법 2가지 | 7-9 (`tcp_syncookies`·백로그 증대) |
| 필기 FULL r01-91, r02-91, r02-93, r04-96, r05-100, r06-98, r07-95 — SYN Flooding·Smurf·Teardrop·Land·Ping of Death 구분 | 7-9 공격 유형표 |
| 실기 r06-15 — ARP 스푸핑 원리·MITM 연계·대응 1가지 | 7-10 |
| 필기 FULL r03-96, r04-97 — ARP 스푸핑 탐지(동일 MAC 다수) | 7-10 |
| 필기 FULL r03-95, r06-95, r07-96 — 스니핑·promiscuous 모드 점검 | 7-11 |
| 필기 FULL r01-92, r02-92 — IP/DNS 스푸핑·세션 하이재킹 구분 | 7-9 |
| 필기 FULL r08-92 — `tcpdump` 캡처 해석 | 7-10, 7-11 |
| 필기 FULL r01-99, r02-100, r03-65, r04-94, r07-98, r08-91 — nmap 스캔 유형(`-sS` 하프 오픈)·결과 해석 | 1-2, 10-2 |
| 필기 FULL r01-95, r02-97, r09-98, r10-97 — Snort 룰 해석·NIDS 특징 | 7-12 |
| 필기 FULL r03-94, r07-97 — 방화벽·IDS·IPS 차이 | 7-12 |
| 필기 FULL r01-96, r03-97, r04-98, r05-99, r07-99, r09-99, r10-98 — VPN·IPSec(AH/ESP)·L2TP | 7-12, 8-9 |
| 실기 r03-15, r05-15 — 대칭키 vs 비대칭키(키 사용·속도·키 분배) 서술 | 8-9 비교표 |
| 필기 FULL r01-98, r03-98, r04-99, r09-100 — AES/RSA/SHA-256/ECC 분류 | 8-9 |
| 필기 FULL r01-97, r02-98, r07-70 — TLS 핸드셰이크(비대칭으로 세션키 교환 → 대칭 암호화) | 8-9 |
| 필기 FULL r02-99 — 수신자 공개키로 파일 암호화 = `gpg -e -r 수신자 file` | 8-5 |
| 필기 FULL r04-100 — 전자서명 생성 키 = **송신자의 개인키** | 8-5, 8-6 |
| 필기 FULL r03-99 — PKI·CA·CRL (CA 개인키로 서명, 신뢰 저장소 루트로 검증) | 8-8, 8-9 |
| 필기 FULL r03-67, r05-97, r06-70, r07-71 — `SSLCertificateFile`/`SSLCertificateKeyFile`, CSR→CA 서명 절차 | 8-7, 8-8 |
| 필기 FULL r10-99 — Heartbleed(OpenSSL heartbeat 결함) | 8-7 (`openssl` 진단) |
| 실기 r04-4 — `sha256sum image.iso` | 8-1 |
| 필기 FULL r06-96 — 충돌 발견 해시(MD5) 대신 SHA-2 로 무결성 검증 | 8-1 |
| 필기 FULL r04-59, r09-25 — `/etc/shadow` 의 `$6$` = SHA-512 | 8-2 |
| 필기 FULL r03-31 — `rpm --import` 후 `rpm -K`(서명 검증) | 8-6 (서명 검증 개념) |
| 필기 FULL r02-64, r05-89, r06-63, r07-59 — SSH 공개키 인증 절차·`authorized_keys` | 6-3, 8-9 (Part 08 링크) |
| 실기 r01-10, r02-10, r04-10 / 필기 cron 필드 — `0 4 * * 1` 주간 스케줄 | 9-3 |
| 실기 r01-11 — `/etc/crontab` 7필드 해석 | 9-3 |
| 필기 FULL r09-24 — `pam_pwquality minlen=10 dcredit=-1` | 6-5 (Part 03 연계) |
| 필기 FULL r02-22 — `usermod -s /sbin/nologin` 으로 로그인 차단 | 6-8 |
| 필기 FULL r08-27 — `nosuid` 마운트로 SetUID 무력화 | 7-1 (Part 05 마운트 옵션 연계) |
| 필기 FULL r09-7, r09-9, r02-25, r08-25 — SetUID 동작(소유자 권한으로 실행)·8진수 표기 | 7-1, 7-2 |

---

## 이전 / 다음

[[09-network-services]] ← · → [[11-container-virtualization]]

- 허브: [[README]]
- 이론: [[../THEORY/selinux-security]] · [[../THEORY/system-security]] · [[../THEORY/network-security]]
- 명령어 볼트: [[../../SECURITY-SELINUX/getenforce]] · [[../../SECURITY-SELINUX/ausearch]] · [[../../SECURITY-SELINUX/firewall-cmd]] · [[../../SECURITY-SELINUX/iptables]]
