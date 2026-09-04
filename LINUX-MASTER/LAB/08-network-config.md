---
title: LAB 08 — 네트워크 설정·진단·SSH
type: exam-lab
part: 08
tags:
  - exam/linux-master
  - exam/lab
  - linux/network
  - linux/security
  - topic/static-ip
  - topic/dns
  - topic/remote-access
  - task/configure
  - task/verify
  - task/diagnose
related: ["[[README]]", "[[07-boot-systemd-log]]", "[[09-network-services]]", "[[../THEORY/network-basics]]", "[[../THEORY/network-service]]", "[[../../NETWORK-MANAGEMENT/ip]]", "[[../../NETWORK-MANAGEMENT/nmcli]]", "[[../../NETWORK-MANAGEMENT/nmtui]]", "[[../../NETWORK-MANAGEMENT/ss]]", "[[../../NETWORK-MANAGEMENT/ping]]", "[[../../NETWORK-MANAGEMENT/nc]]", "[[../../NETWORK-MANAGEMENT/ssh]]", "[[../../NETWORK-MANAGEMENT/ssh-keygen]]", "[[../../NETWORK-MANAGEMENT/scp]]", "[[../../NETWORK-MANAGEMENT/sshpass]]", "[[../../NETWORK-MANAGEMENT/curl]]", "[[../../NETWORK-MANAGEMENT/hostnamectl]]", "[[../../NETWORK-MANAGEMENT/whois]]"]
distro: Rocky Linux 9 (aarch64, UTM)
updated: 2026-09-03
---

# LAB 08 — 네트워크 설정·진단·SSH

- DHCP 임시 주소 → **고정 IP `192.168.64.10/24`** 전환을 `nmcli` 로 수행하고 keyfile 프로파일·레거시 `ifcfg` 대응까지 확인
- `ip`·`ss`·`ping`·`dig`·`nc`·`tcpdump`·`nmap` 전 계열 진단 도구를 자기 자신·게이트웨이 대상으로 실습, **3-way handshake 를 직접 캡처**
- 이론(OSI·서브네팅·포트)을 `ipcalc`·`/etc/services` 로 손이 아닌 명령으로 검증 — 필기 계산 문제를 실습으로 환산
- SSH 를 **키 인증 + 포트 2222** 로 전환 (SELinux 포트 레이블 → 방화벽 → 재시작 순서 필수), `scp`·`sftp`·`rsync`·포트 포워딩까지 활용
- 마지막에 `chrony` 시간 동기화 정리 + `/usr/local/bin/check-part08.sh` 로 재부팅 영구성 검증

> **이 파트의 시나리오**: Part 07 까지 서버 내부(부팅·서비스·로그)를 정비했으나 IP 는 아직 DHCP 임시 주소다. Part 09 에서 Apache·BIND·NFS·Samba 를 올리려면 **주소가 고정**되어야 하고 `srv01.lab.local` 이름으로 접근 가능해야 한다. 이번 파트에서 고정 IP·이름 해석·라우팅·시간 동기화를 확정하고, macOS 호스트 ↔ VM 사이 운영 통로인 SSH 를 비밀번호에서 **키 인증**으로 바꾼 뒤 포트를 2222 로 옮긴다.

- 선행 자원: `admin1`(설치 시 생성, wheel), `dev1`·`devteam`(Part 03), `/srv/share`(Part 05), sshd 기동 상태(기본 활성)
- ⚠️ 2절(고정 IP 전환)·9절(포트 변경)은 **SSH 세션이 끊길 수 있음** → UTM 디스플레이 콘솔을 먼저 열어 두고 진행. 잠기면 9-6 복구 절차 참조
- 방화벽(`firewall-cmd`) 상세는 [[10-security-firewall-selinux]], 서버 데몬 구축(BIND·httpd)은 [[09-network-services]] 에서 다룸 — 이 파트는 **클라이언트·설정·진단** 범위

---

## 1. 현재 네트워크 상태 파악

### 1-1. 주소 조회 — ip addr 3종

> **상황**: 고정 IP 로 바꾸기 전에 DHCP 가 지금 무엇을 할당했는지, 인터페이스 이름이 정말 `enp0s1` 인지 확정한다. 이후 모든 절이 이 값을 기준으로 한다.

```bash
ip addr                       # 전체 인터페이스 상세 (별칭 ip a / ip address)
ip -4 a                       # IPv4 만
ip -br a                      # 한 줄 요약 (brief)
ip -br -c a                   # 색상 강조 요약
ip a show enp0s1              # 특정 인터페이스만
```

- `addr` : address 오브젝트 — `a`, `address` 로 축약 가능
- `-4` / `-6` : IPv4 / IPv6 주소만 표시
- `-br` : 인터페이스·상태·주소만 한 줄로 (**br**ief)
- `-c` : 상태별 색상 출력 (**c**olor)
- `show <dev>` : 대상 인터페이스 한정 (`show` 는 생략 가능)

**검증**

```bash
ip -br a | awk '{print $1, $2, $3}'          # 이름·상태·주소만 추출
ip -4 -o a show enp0s1 | awk '{print $4}'    # 프리픽스 포함 주소 1줄
```

```text
# ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp0s1           UP             192.168.64.x/24 fe80::.../64
# ip -4 -o a show enp0s1 | awk '{print $4}'
192.168.64.x/24
```

- `-o` : 한 레코드를 한 줄로 (**o**neline) — `awk`·`grep` 조합 시 필수
- 현재 값 `192.168.64.x` 는 DHCP 임시 주소 → 2절에서 `.10` 으로 고정

> 📝 **시험 포인트**: `ifconfig` 대체 명령 묻는 문제의 정답은 `ip addr`. `ip a` 축약형과 `-br`(요약)·`-o`(한 줄) 구분 출제. IP 미할당 상태에서는 `ping` 이 대상 무관하게 `unreachable` — 상대 장애로 오판하지 말 것.

### 1-2. 링크 계층 — ip link 와 인터페이스 통계

> **상황**: 3계층(IP) 이전에 2계층(링크)이 살아 있는지 확인한다. MAC 주소·MTU·플래그와 함께 RX/TX 누적 카운터를 봐 두면 뒤의 `tcpdump` 실습에서 증가량 비교가 가능하다.

```bash
ip link                          # 링크 목록 (별칭 ip l)
ip -br link                      # 요약 — 상태·MAC·플래그
ip -s link show enp0s1           # 송수신 통계 (statistics)
ip -s -s link show enp0s1        # 오류 세부 항목까지 (중복 지정)
cat /sys/class/net/enp0s1/address    # MAC 주소 파일 직접 조회
cat /sys/class/net/enp0s1/mtu        # MTU
```

- `link` : 링크(2계층) 오브젝트 — `l` 로 축약
- `-s` : 통계 출력 (**s**tatistics), 두 번 지정 시 오류 항목 세분화
- 플래그 의미
  - `UP` : 관리자에 의해 활성화됨(소프트웨어)
  - `LOWER_UP` : 물리 링크 정상(케이블·캐리어 있음)
  - `NO-CARRIER` : 물리 신호 없음 → **소프트웨어 설정으로 해결 불가**
  - `BROADCAST` / `MULTICAST` : 해당 전송 지원
  - `LOOPBACK` : `lo` 전용

**검증**

```bash
ip -s link show enp0s1 | sed -n '3,6p'       # RX/TX 카운터 4줄
ip -br link | grep -c LOWER_UP               # 링크 정상 인터페이스 수
```

```text
# ip -br link
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
enp0s1           UP             <MAC> <BROADCAST,MULTICAST,UP,LOWER_UP>
# ip -s link show enp0s1
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether <MAC> brd ff:ff:ff:ff:ff:ff
    RX: bytes  packets  errors  dropped  overrun  mcast
    ...        ...      0       0        0        ...
    TX: bytes  packets  errors  dropped  carrier  collsns
    ...        ...      0       0        0        0
```

- 카운터 해석
  - `errors` : CRC·프레임 오류 → 케이블·NIC 하드웨어 의심
  - `dropped` : 버퍼 부족·필터로 폐기 → 부하 또는 방화벽 정책
  - `overrun` : 커널이 처리 못해 NIC 버퍼 넘침
  - `carrier` : 캐리어 손실 횟수 (링크 끊김 반복)
  - `collsns` : 충돌 — 전이중(full duplex) 환경에서는 0 이 정상

> 📝 **시험 포인트**: MAC 주소는 **2계층(데이터링크)**, IP 는 3계층. `ip -s link` 의 `errors`(하드웨어) vs `dropped`(정책·버퍼) 구분이 장애 진단 서술형 소재.

### 1-3. 라우팅 테이블과 ARP 캐시

> **상황**: 외부로 나가는 경로(기본 게이트웨이)와 같은 대역 이웃의 MAC 캐시를 확인한다. 고정 IP 전환 후 이 두 값이 유지되는지가 검증 포인트다.

```bash
ip route                         # 라우팅 테이블 (별칭 ip r / ip ro)
ip -4 route show                 # IPv4 경로만
ip route show default            # 기본 경로만
ip neigh                         # ARP 캐시 (neighbour)
ip neigh show 192.168.64.1       # 게이트웨이 MAC 확인
ip route get 8.8.8.8             # 특정 목적지로 나갈 때 선택될 경로 계산
```

- `route` : 라우팅 오브젝트 — `r`, `ro` 로 축약
- `neigh` : **neigh**bour(이웃) 테이블 = ARP(IPv4) / NDP(IPv6) 캐시
- `get <IP>` : 실제 라우팅 결정 결과(출발 주소·인터페이스·게이트웨이)를 커널에 질의
- 라우팅 항목 필드
  - `default via <GW> dev <IF>` : 기본 게이트웨이
  - `192.168.64.0/24 dev enp0s1 proto kernel scope link src <IP>` : 직접 연결 대역
  - `proto` : 경로 생성 주체 (`kernel` 자동, `static` 수동, `dhcp` DHCP)
  - `metric` : 값이 **작을수록 우선** — default 가 둘이면 이 값으로 결정
- ARP 상태값: `REACHABLE`(유효) / `STALE`(만료 대기) / `DELAY` / `FAILED`(응답 없음) / `PERMANENT`(고정 등록)

**검증**

```bash
ip route show default | awk '{print $3, $5}'     # 게이트웨이 IP, 인터페이스
ip route get 8.8.8.8 | head -1
ip neigh show 192.168.64.1
```

```text
# ip route
default via 192.168.64.1 dev enp0s1 proto dhcp src 192.168.64.x metric 100
192.168.64.0/24 dev enp0s1 proto kernel scope link src 192.168.64.x metric 100
# ip route get 8.8.8.8
8.8.8.8 via 192.168.64.1 dev enp0s1 src 192.168.64.x uid 0
# ip neigh show 192.168.64.1
192.168.64.1 dev enp0s1 lladdr <GW-MAC> REACHABLE
```

- `proto dhcp` 표기 → 아직 DHCP 로 받은 경로. 2절 이후 `proto static` 으로 바뀜
- ICMP 를 차단한 장비도 ARP 에는 응답 → **`ping` 무응답 시 `ip neigh` 로 존재 확정 판정** 가능

> 📝 **시험 포인트**: "기본 게이트웨이 확인 명령"의 정답은 `ip route`(구 `route -n`, `netstat -rn`). ARP 캐시 조회는 `ip neigh`(구 `arp -a`/`arp -n`). ARP = **IP → MAC**, RARP = **MAC → IP** 방향 혼동이 최빈출 함정.

### 1-4. IPv6 주소 확인

> **상황**: 이 실습망은 IPv4 전용이지만 링크로컬 IPv6 는 자동 생성된다. 시험에 IPv6 표기·범위가 나오므로 실물로 확인해 둔다.

```bash
ip -6 a                                   # IPv6 주소 전체
ip -6 a show enp0s1 scope link            # 링크로컬만
ip -6 route                               # IPv6 라우팅
ping6 -c 2 ::1                            # 루프백 IPv6 도달 확인
cat /proc/sys/net/ipv6/conf/all/disable_ipv6   # 0 = IPv6 활성
```

- `scope link` : 링크로컬 범위(`fe80::/10`) — 라우터를 넘지 못함
- `scope global` : 전역 범위 — 라우팅 가능
- `scope host` : 호스트 내부(`::1`)
- `ping6` : IPv6 전용 ping — Rocky 9 에서는 `ping -6` 과 동일 (`iputils` 제공)

**검증**

```bash
ip -6 -br a
ping6 -c 2 ::1 | tail -2
```

```text
# ip -6 -br a
lo               UNKNOWN        ::1/128
enp0s1           UP             fe80::.../64
# ping6 -c 2 ::1
2 packets transmitted, 2 received, 0% packet loss, time ...
```

- `fe80::` 로 시작하는 주소만 있음 = 전역 IPv6 미할당 (라우터 광고 없음) → 정상
- IPv6 를 쓰지 않으면 2절에서 `ipv6.method disabled` 로 대기 시간 제거 가능

> 📝 **시험 포인트**: IPv6 = **128비트**, 16비트 8그룹 16진수 콜론 표기. 축약 규칙은 ① 각 그룹 앞자리 `0` 생략 ② 연속된 0 그룹을 `::` 로 **딱 한 번만** 압축. `2001:0db8:0000:0000:0000:ff00:0042:8329` → `2001:db8::ff00:42:8329`.

### 1-5. NetworkManager 관점 조회

> **상황**: `ip` 는 커널의 현재 상태를, `nmcli` 는 저장된 프로파일을 보여준다. 둘이 어긋나 있으면 "설정은 했는데 적용이 안 된" 상태다. 전환 전에 양쪽을 대조한다.

```bash
nmcli device status                        # 장치별 상태·연결된 프로파일
nmcli connection show                      # 저장된 프로파일 목록
nmcli connection show --active             # 활성 프로파일만
nmcli con show enp0s1 | grep ipv4          # 프로파일의 IPv4 설정 전체
nmcli con show enp0s1 | grep -E 'IP4\.'    # 실제 적용된 IPv4 (대문자 = 런타임 값)
nmcli device show enp0s1                   # 장치 관점 상세 (MAC·MTU·IP4·DNS)
```

- `device` : 물리·논리 장치 (`dev`, `d` 축약)
- `connection` : 연결 프로파일 (`con`, `c` 축약)
- 출력 필드 대소문자 구분이 핵심
  - `ipv4.method`, `ipv4.addresses` … **소문자** = 프로파일에 저장된 설정값
  - `IP4.ADDRESS[1]`, `IP4.GATEWAY` … **대문자** = 지금 실제 적용된 런타임 값
- device STATE 값: `connected` / `disconnected` / `unavailable`(링크 없음) / `connecting (getting IP configuration)`(DHCP 대기) / `unmanaged`

**검증**

```bash
nmcli -t -f DEVICE,STATE,CONNECTION device status
nmcli con show enp0s1 | grep -E 'ipv4.method|IP4.ADDRESS|IP4.GATEWAY|IP4.DNS'
```

```text
# nmcli device status
DEVICE  TYPE      STATE      CONNECTION
enp0s1  ethernet  connected  enp0s1
lo      loopback  unmanaged  --
# nmcli con show enp0s1 | grep -E 'ipv4.method|IP4.ADDRESS'
ipv4.method:                            auto
IP4.ADDRESS[1]:                         192.168.64.x/24
```

- `ipv4.method: auto` = DHCP. 2절에서 `manual` 로 전환
- `lo` 가 `unmanaged` 인 것은 정상 (NetworkManager 는 루프백을 관리 대상에서 제외)

> 📝 **시험 포인트**: RHEL 8 이상은 NetworkManager 가 표준. `nmcli con mod` 는 **저장만** 하고 즉시 반영되지 않음 → `nmcli con up`(또는 `down`+`up`) 필요. 이 "저장 ≠ 적용" 구분이 실기 단골.

### 1-6. 이름 해석 설정 파일

> **상황**: DNS 서버 주소가 어디서 오는지, 이름을 찾을 때 어떤 순서로 뒤지는지 확인한다. 2절에서 `ipv4.dns` 를 바꾸면 이 파일이 어떻게 재생성되는지 대조할 기준이 된다.

```bash
cat /etc/resolv.conf                  # DNS 서버·검색 도메인
ls -l /etc/resolv.conf                # 심볼릭 링크 여부 확인
grep '^hosts:' /etc/nsswitch.conf     # 이름 해석 소스 순서
cat /etc/hosts                        # 정적 매핑
```

- `/etc/resolv.conf` 지시자
  - `nameserver <IP>` : 질의할 DNS 서버 — **최대 3개**, 위에서부터 순차 시도
  - `search <도메인>` : 짧은 이름 입력 시 붙여 볼 도메인 목록
  - `domain <도메인>` : 단일 기본 도메인 (`search` 와 병용 시 뒤에 온 것이 이김)
  - `options timeout:2 attempts:2` : 질의 타임아웃·재시도 횟수

**검증**

```bash
head -3 /etc/resolv.conf
grep '^hosts:' /etc/nsswitch.conf
```

```text
# cat /etc/resolv.conf
# Generated by NetworkManager
search lan
nameserver 192.168.64.1
# grep '^hosts:' /etc/nsswitch.conf
hosts:      files dns myhostname
```

- 첫 줄 `# Generated by NetworkManager` = **직접 편집해도 재활성화 시 덮어써짐** → 반드시 `nmcli ipv4.dns` 로 설정
- `hosts: files dns myhostname` 순서 의미
  1. `files` : `/etc/hosts` 를 **먼저** 조회 → 여기서 찾으면 DNS 질의 자체를 하지 않음
  2. `dns` : `/etc/resolv.conf` 의 nameserver 에 질의
  3. `myhostname` : systemd 제공 모듈 — 자기 호스트명·`localhost` 를 항상 해석 보장
- 순서를 `dns files` 로 바꾸면 `/etc/hosts` 보다 DNS 가 우선 → 로컬 오버라이드가 무력화

> 📝 **시험 포인트**: `/etc/nsswitch.conf` 의 `hosts:` 행 순서 = 조회 우선순위. "hosts 파일이 DNS 보다 먼저 조회되게 하려면?" → `hosts: files dns`. `/etc/resolv.conf` 의 `nameserver` 최대 3개 제한도 출제.

### 1-7. 호스트명 확인

> **상황**: Part 01 에서 `srv01.lab.local` 로 설정한 호스트명이 유지되는지, 짧은 이름·FQDN·IP 가 각각 어떻게 나오는지 확인한다. Part 09 의 가상호스트·메일 도메인이 이 값을 쓴다.

```bash
hostname                       # 현재 호스트명 (짧은/설정된 이름)
hostname -s                    # 짧은 이름 (short)
hostname -f                    # FQDN (fully qualified domain name)
hostname -d                    # 도메인 부분만
hostname -I                    # 할당된 모든 IP (대문자 I)
hostnamectl                    # 호스트명 + OS·커널·아키텍처 종합
hostnamectl hostname           # 호스트명만
cat /etc/hostname              # 영구 저장 파일
nmcli general hostname         # NetworkManager 가 보는 호스트명
```

- `hostname` 옵션
  - `-s` : 첫 `.` 앞부분만 (**s**hort)
  - `-f` : FQDN — `/etc/hosts` 또는 DNS 로 역방향 해석해서 만듦
  - `-d` : 도메인만 (**d**omain)
  - `-I` : 인터페이스에 붙은 IP 전부 공백 구분 출력 (**I**P, 루프백 제외)
- `hostnamectl` 이 보여주는 3종 호스트명
  - `Static hostname` : `/etc/hostname` 에 저장된 값 (영구)
  - `Transient hostname` : DHCP·mDNS 가 런타임에 준 값 (휘발)
  - `Pretty hostname` : 사람이 읽는 설명용 (`"개발팀 서버"` 같은 자유 문자열)

**검증**

```bash
hostnamectl | grep -E 'hostname|Machine ID|Boot ID'
hostname -f; hostname -I
```

```text
# hostnamectl
 Static hostname: srv01.lab.local
       Icon name: computer-vm
         Chassis: vm
      Machine ID: ...
         Boot ID: ...
  Operating System: Rocky Linux 9.x (Blue Onyx)
          Kernel: Linux 5.14.0-...aarch64
    Architecture: arm64
# hostname -f
srv01.lab.local
# hostname -I
192.168.64.x
```

- 호스트명이 `localhost` 이면 Part 01 미완 → `hostnamectl set-hostname srv01.lab.local` 후 재로그인
- `hostname <이름>` 으로 바꾼 값은 **재부팅 시 소멸** (transient) → 영구 설정은 `hostnamectl set-hostname` 또는 `nmcli general hostname <이름>`

> 📝 **시험 포인트**: `hostname -f` 가 FQDN 을 못 뽑으면 `/etc/hosts` 에 `<IP> <FQDN> <짧은이름>` 순서로 등록되어 있는지 확인. **FQDN 을 먼저** 쓰는 순서가 정답(첫 번째 항목이 정규 이름).

### 1-8. 포트·프로토콜 사전 파일

> **상황**: 5절·6절에서 포트 번호를 다룰 때 근거가 되는 시스템 사전 파일을 먼저 열어 본다. 시험의 "포트 번호 암기"를 이 파일로 확인하는 습관을 들인다.

```bash
grep -w '22/tcp' /etc/services            # 포트 → 서비스명
grep -w '^ssh' /etc/services              # 서비스명 → 포트
grep -wE '^(ftp|ssh|telnet|smtp|domain|http|pop3|imap|https)' /etc/services
wc -l /etc/services                       # 등록된 항목 수
head -20 /etc/protocols                   # 프로토콜명 ↔ 번호
grep -wE '^(icmp|tcp|udp|igmp)' /etc/protocols
getent services 22/tcp                    # NSS 경유 조회 (동일 결과)
getent services ssh
```

- `grep -w` : 단어 경계 일치 (**w**ord) — `22` 가 `2222`·`122` 에 걸리지 않게 함
- `/etc/services` 형식: `<서비스명> <포트>/<프로토콜> [별칭] # 주석`
- `/etc/protocols` 형식: `<프로토콜명> <번호> [별칭] # 주석` — IP 헤더의 Protocol 필드 값
- `getent <db> <key>` : nsswitch 를 거친 조회 — `services`·`protocols`·`hosts`·`passwd` 등 DB 지정

**검증**

```bash
for p in 20 21 22 23 25 53 80 110 143 443; do
  printf '%-5s %s\n' "$p" "$(grep -w "^[a-z-]*[[:space:]]*$p/tcp" /etc/services | head -1)"
done
```

```text
# grep -w '22/tcp' /etc/services
ssh             22/tcp                  # The Secure Shell (SSH) Protocol
# grep -wE '^(icmp|tcp|udp|igmp)' /etc/protocols
icmp    1       ICMP            # internet control message protocol
igmp    2       IGMP            # Internet Group Management
tcp     6       TCP             # transmission control protocol
udp     17      UDP             # user datagram protocol
```

- 프로토콜 번호 암기값: **ICMP=1, IGMP=2, TCP=6, UDP=17** — IP 헤더 Protocol 필드에 들어감
- 포트 30개 전수 검증은 6-7 에서 수행

> 📝 **시험 포인트**: "포트↔서비스 매핑 파일" = `/etc/services`, "프로토콜↔번호 매핑 파일" = `/etc/protocols`. Well-known 0~1023 / Registered 1024~49151 / Dynamic(Private) 49152~65535 범위 구분 필수.

### 1-9. net-tools 설치와 구형↔신형 명령 대응

> **상황**: 시험 문제는 여전히 `ifconfig`·`netstat`·`route`·`arp` 를 물어보지만 Rocky 9 minimal 에는 없다. 설치해서 양쪽 출력을 나란히 보고 1:1 대응을 몸으로 익힌다.

```bash
rpm -q net-tools || dnf install -y net-tools     # 미설치 시 설치
ifconfig -a                       # 전체 인터페이스 (비활성 포함)
ifconfig enp0s1                   # 특정 인터페이스
netstat -rn                       # 라우팅 테이블 (numeric)
route -n                          # 라우팅 테이블 (동일 정보)
arp -n                            # ARP 캐시
arp -a                            # ARP 캐시 (BSD 형식)
netstat -tulnp                    # 리스닝 포트 + 프로세스
netstat -i                        # 인터페이스별 패킷 통계
netstat -s                        # 프로토콜별 누적 통계
```

- `net-tools` 제공 명령: `ifconfig` `netstat` `route` `arp` `netstat` `mii-tool` `nameif`
- `-n` : 이름 역해석 생략, 숫자로 표시 (**n**umeric) — DNS 지연 회피
- `-a` : 전체 표시 (**a**ll)
- `-i` : 인터페이스 통계 (**i**nterface)
- `-s` : 프로토콜별 통계 (**s**tatistics)

**구형 ↔ 신형 대응표 (1:1)**

| 작업 | 구형 (net-tools, deprecated) | 현행 (iproute2) |
| --- | --- | --- |
| 주소 조회 | `ifconfig` / `ifconfig -a` | `ip addr` / `ip -br a` |
| 주소 추가 | `ifconfig enp0s1 192.168.64.11 netmask 255.255.255.0` | `ip addr add 192.168.64.11/24 dev enp0s1` |
| 주소 삭제 | `ifconfig enp0s1 0.0.0.0` | `ip addr del 192.168.64.11/24 dev enp0s1` |
| 인터페이스 up/down | `ifconfig enp0s1 up` / `down` | `ip link set enp0s1 up` / `down` |
| MTU 변경 | `ifconfig enp0s1 mtu 1400` | `ip link set enp0s1 mtu 1400` |
| MAC 변경 | `ifconfig enp0s1 hw ether <MAC>` | `ip link set enp0s1 address <MAC>` |
| 라우팅 조회 | `route -n` / `netstat -rn` | `ip route` |
| 기본 GW 추가 | `route add default gw 192.168.64.1` | `ip route add default via 192.168.64.1` |
| 정적 경로 추가 | `route add -net 10.10.0.0/16 gw 192.168.64.1` | `ip route add 10.10.0.0/16 via 192.168.64.1` |
| 경로 삭제 | `route del -net 10.10.0.0/16` | `ip route del 10.10.0.0/16` |
| ARP 캐시 | `arp -n` / `arp -a` | `ip neigh` |
| ARP 정적 등록 | `arp -s <IP> <MAC>` | `ip neigh add <IP> lladdr <MAC> dev enp0s1` |
| 소켓·포트 | `netstat -tulnp` | `ss -tulnp` |
| 연결 상태 전체 | `netstat -an` | `ss -an` |
| 인터페이스 통계 | `netstat -i` | `ip -s link` |
| 프로토콜 통계 | `netstat -s` | `ss -s` (요약) / `nstat` |
| 멀티캐스트 | `netstat -g` | `ip maddr` |

**검증**

```bash
ifconfig enp0s1 | head -3
diff <(route -n | tail -n +3 | awk '{print $1,$2,$3}' | sort) \
     <(netstat -rn | tail -n +3 | awk '{print $1,$2,$3}' | sort) && echo "route -n == netstat -rn"
```

```text
# ifconfig enp0s1
enp0s1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.64.x  netmask 255.255.255.0  broadcast 192.168.64.255
        inet6 fe80::...  prefixlen 64  scopeid 0x20<link>
# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.64.1    0.0.0.0         UG    100    0        0 enp0s1
192.168.64.0    0.0.0.0         255.255.255.0   U     100    0        0 enp0s1
route -n == netstat -rn
```

- `route -n` Flags 해석: `U`=경로 활성(**U**p), `G`=게이트웨이 경유(**G**ateway), `H`=호스트 단위 경로(**H**ost), `!`=거부
- `ifconfig` 는 **넷마스크를 점 표기**(`255.255.255.0`), `ip` 는 **프리픽스**(`/24`) — 변환 감각 필요
- `ifconfig` 는 IP 별칭(`enp0s1:0`)만 보여주고 `ip addr` 로 추가한 **보조 주소는 표시하지 못함** → 구형 도구의 대표적 한계

> 📝 **시험 포인트**: 대체 관계 4쌍 `ifconfig→ip addr`, `route→ip route`, `arp→ip neigh`, `netstat→ss` 는 필기·실기 양쪽 최빈출. `route -n` 의 `UG` 플래그 = 기본 게이트웨이 행 판별 근거.

### 1-10. NIC 하드웨어 정보 — ethtool (virtio 제약)

> **상황**: 물리 서버라면 속도·듀플렉스 협상 결과를 `ethtool` 로 확인하지만, 이 VM 은 virtio 가상 NIC 다. 어디까지 나오고 무엇이 안 나오는지 정확히 확인해 둔다.

```bash
rpm -q ethtool || dnf install -y ethtool
ethtool enp0s1                    # 속도·듀플렉스·링크 상태
ethtool -i enp0s1                 # 드라이버·펌웨어·버스 정보
ethtool -S enp0s1                 # NIC 통계 카운터
ethtool -k enp0s1 | head          # 오프로드 기능 on/off
```

- `-i` : 드라이버 정보 (**i**nformation/driver)
- `-S` : NIC 하드웨어 통계 (**S**tatistics)
- `-k` : 오프로드 기능 조회 (소문자 k, 설정은 대문자 `-K`)
- `-s` : 속도·듀플렉스 강제 설정 (**s**et) — ⚠️ 가상 NIC 에서는 의미 없음, 물리 NIC 에서도 링크 순단 발생

**검증**

```bash
ethtool -i enp0s1 | grep -E 'driver|bus-info'
ethtool enp0s1 | grep -E 'Speed|Duplex|Link detected'
```

```text
# ethtool -i enp0s1
driver: virtio_net
version: 1.0.0
bus-info: ...
# ethtool enp0s1
Settings for enp0s1:
        ...
        Speed: Unknown!
        Duplex: Unknown!
        Link detected: yes
```

- ⚠️ **virtio 제약**: 가상 NIC 은 물리 PHY 가 없어 `Speed`/`Duplex`/`Auto-negotiation` 이 `Unknown!` 또는 미표시. `Link detected: yes` 만 유효
- 물리 서버에서는 `Speed: 1000Mb/s`, `Duplex: Full` 로 표시 → 100Mb/s·Half 로 협상됐으면 케이블·스위치 포트 문제
- 실속도 진단 대체 수단: `ip -s link` 카운터 증가, `iperf3`(별도 패키지·미실행)

> 📝 **시험 포인트**: `ethtool` 은 **2계층(물리·링크) 진단** 도구 — 속도·듀플렉스·오토네고. 3계층 문제(`ping` 실패)와 구분해서 순서대로(링크→주소→경로→DNS) 좁혀 가는 진단 흐름이 서술형 소재.

### 1-11. nmtui 화면 흐름 (참고)

> **상황**: 콘솔에서 오타 없이 설정해야 할 때 쓰는 TUI 도구다. 2절은 `nmcli` 로 진행하지만, 실기에서 GUI 없는 환경을 가정한 문제가 나오므로 화면 이동 경로를 알아 둔다.

```bash
nmtui                     # 메인 메뉴
nmtui edit enp0s1         # 편집 화면 바로 열기
nmtui connect             # 활성화 화면 바로 열기
nmtui hostname            # 호스트명 설정 화면 바로 열기
```

- 메인 메뉴 3항목
  - `Edit a connection` : 프로파일 편집 (IP·게이트웨이·DNS·자동연결)
  - `Activate a connection` : 활성/비활성 전환 (`nmcli con up/down` 대응)
  - `Set system hostname` : 호스트명 (`hostnamectl set-hostname` 대응)

**고정 IP 설정 화면 흐름**

```text
nmtui
 → Edit a connection
 → enp0s1 선택 → <Edit...>
 → IPv4 CONFIGURATION  <Automatic> → <Manual> 로 변경
 → Addresses   <Add...>  192.168.64.10/24
 → Gateway               192.168.64.1
 → DNS servers <Add...>  192.168.64.1
               <Add...>  8.8.8.8
 → Search domains <Add...> lab.local
 → [X] Automatically connect      ← 화면 하단, 스크롤 필요 · 누락 빈발
 → <OK> → <Back> → Activate a connection → enp0s1 → <Deactivate> → <Activate>
```

**검증**

```bash
nmcli con show enp0s1 | grep -E 'ipv4.method|ipv4.addresses|connection.autoconnect'
```

- ⚠️ `[X] Automatically connect` 미체크 시 재부팅 후 네트워크 미기동 → **원격 접속까지 차단**. `nmcli` 의 `connection.autoconnect yes` 와 동일 항목
- 설정 후 반드시 `Activate a connection` 에서 재활성화해야 반영 (`nmcli con up` 과 동일)

> 📝 **시험 포인트**: `nmtui` = **N**etwork**M**anager **T**ext **U**ser **I**nterface. OS 설치 화면(Anaconda)의 네트워크 편집 창과 동일 레이아웃. `nmcli` 와 동일 기능·동일 저장 위치라는 점이 출제 포인트.

---

## 2. 고정 IP 전환 (nmcli)

### 2-1. 전환 전 준비 — 콘솔 확보와 현재 값 백업

> **상황**: `nmcli con up` 순간 IP 가 바뀌면서 **현재 SSH 세션이 즉시 끊긴다**. 되돌릴 수 있도록 현재 설정을 파일로 남기고 UTM 콘솔을 확보한다.

⚠️ 아래 명령을 실행하기 전에 **UTM 디스플레이 창을 열어 root 로 로그인해 둘 것**. SSH 세션만으로 진행하면 전환 직후 접속이 끊기고 새 주소로 다시 붙어야 한다.

```bash
mkdir -p /root/net-backup
ip a > /root/net-backup/ip-a.before
ip r > /root/net-backup/ip-r.before
cp /etc/resolv.conf /root/net-backup/resolv.conf.before
nmcli con show enp0s1 > /root/net-backup/nmcli-enp0s1.before
nmcli -t -f NAME,UUID,DEVICE con show > /root/net-backup/con-list.before
ls -l /etc/NetworkManager/system-connections/
```

- `nmcli -t` : 콜론 구분 스크립트용 출력 (**t**erse) — 파싱 편의
- `-f NAME,UUID,DEVICE` : 출력 필드 지정 (**f**ields)

**검증**

```bash
ls -l /root/net-backup/
grep -c . /root/net-backup/ip-a.before
```

```text
# ls -l /root/net-backup/
-rw-r--r--. 1 root root  ... con-list.before
-rw-r--r--. 1 root root  ... ip-a.before
-rw-r--r--. 1 root root  ... ip-r.before
-rw-r--r--. 1 root root  ... nmcli-enp0s1.before
-rw-r--r--. 1 root root  ... resolv.conf.before
```

> 📝 **시험 포인트**: 원격에서 네트워크 설정을 바꿀 때는 "콘솔 확보 → 백업 → 변경 → 검증" 순서. 실기 서술형에서 "IP 변경 시 주의사항"을 물으면 **세션 단절 대비(콘솔·롤백 수단)** 가 정답 요소.

### 2-2. nmcli con mod 로 고정 IP 지정

> **상황**: 프로파일 `enp0s1` 의 IPv4 방식을 DHCP(`auto`) 에서 수동(`manual`) 로 바꾸고 주소·게이트웨이·DNS·검색 도메인을 한 번에 지정한다. 이 단계는 저장만 하므로 아직 통신은 끊기지 않는다.

```bash
nmcli con mod enp0s1 \
  ipv4.method manual \
  ipv4.addresses 192.168.64.10/24 \
  ipv4.gateway 192.168.64.1 \
  ipv4.dns "192.168.64.1 8.8.8.8" \
  ipv4.dns-search lab.local \
  connection.autoconnect yes
```

- `con mod` : connection modify — 프로파일 설정 저장 (`con` = connection 축약)
- `ipv4.method` : `auto`(DHCP) / `manual`(고정) / `link-local` / `shared`(NAT 공유) / `disabled`
- `ipv4.addresses` : `IP/프리픽스` — 여러 개는 쉼표 구분 (`192.168.64.10/24,192.168.64.20/24`)
- `ipv4.gateway` : 기본 게이트웨이 — **IP 와 같은 대역**이어야 함 (다른 대역이면 외부 통신 불가)
- `ipv4.dns` : DNS 서버, 다중 지정 시 **공백 구분 + 따옴표**
- `ipv4.dns-search` : `/etc/resolv.conf` 의 `search` 행에 들어갈 검색 도메인
- `connection.autoconnect yes` : 부팅 시 자동 활성화 — **누락 시 재부팅 후 네트워크 미기동**
- 접두 기호로 목록형 속성 조작 가능
  - `+ipv4.dns 1.1.1.1` : 기존 목록에 추가
  - `-ipv4.dns 8.8.8.8` : 목록에서 제거
  - `ipv4.dns ""` : 전체 비우기

**검증** — 아직 적용 전이므로 **저장값만** 바뀌어 있어야 정상

```bash
nmcli con show enp0s1 | grep -E '^ipv4\.(method|addresses|gateway|dns|dns-search)'
nmcli con show enp0s1 | grep -E '^IP4\.ADDRESS'    # 런타임은 아직 옛 주소
ip -4 -o a show enp0s1 | awk '{print $4}'          # 커널도 아직 옛 주소
```

```text
# nmcli con show enp0s1 | grep -E '^ipv4\.'
ipv4.method:                            manual
ipv4.dns:                               192.168.64.1,8.8.8.8
ipv4.dns-search:                        lab.local
ipv4.addresses:                         192.168.64.10/24
ipv4.gateway:                           192.168.64.1
# ip -4 -o a show enp0s1 | awk '{print $4}'
192.168.64.x/24        ← 아직 DHCP 주소 (적용 전)
```

> 📝 **시험 포인트**: `nmcli connection modify <프로파일> ipv4.method manual ipv4.addresses <IP/프리픽스> ipv4.gateway <GW> ipv4.dns "<DNS>"` 전체 문장을 쓰게 하는 실기 문제 출제. `ipv4.addresses` 는 **프리픽스 포함**, `ipv4.dns` 는 **따옴표 안 공백 구분** 이 채점 포인트.

### 2-3. 적용 — nmcli con up 과 검증

> **상황**: 저장한 설정을 실제로 반영한다. 이 순간 주소가 `.x` → `.10` 으로 바뀌며 기존 SSH 세션이 끊긴다.

⚠️ **UTM 콘솔에서 실행 권장**. SSH 에서 실행하면 명령 출력이 오기 전에 세션이 죽는다. 부득이 SSH 로 해야 하면 아래처럼 `nohup` 으로 분리 실행.

```bash
# 권장 — UTM 콘솔에서
nmcli con up enp0s1

# 참고 — SSH 세션에서 부득이 실행할 때 (세션 단절과 무관하게 완료)
nohup bash -c 'sleep 2; nmcli con down enp0s1; nmcli con up enp0s1' >/tmp/netup.log 2>&1 &
```

- `con up <이름>` : 프로파일 활성화 — 저장된 설정을 커널에 반영
- `con down <이름>` : 비활성화 — 주소·경로 제거
- `con up` 만으로 대부분 반영되지만, 장치가 다른 프로파일에 물려 있으면 `down` → `up` 필요

**검증**

```bash
ip -4 -o a show enp0s1 | awk '{print $4}'          # 192.168.64.10/24
ip route show default                              # proto static 확인
cat /etc/resolv.conf                               # nameserver 2개 + search
ping -c 2 192.168.64.1                             # 게이트웨이 도달
ping -c 2 8.8.8.8                                  # 외부 도달
getent hosts rockylinux.org | head -1              # 이름 해석 동작
nmcli device status
```

```text
# ip -4 -o a show enp0s1 | awk '{print $4}'
192.168.64.10/24
# ip route
default via 192.168.64.1 dev enp0s1 proto static metric 100
192.168.64.0/24 dev enp0s1 proto kernel scope link src 192.168.64.10 metric 100
# cat /etc/resolv.conf
# Generated by NetworkManager
search lab.local
nameserver 192.168.64.1
nameserver 8.8.8.8
# nmcli device status
DEVICE  TYPE      STATE      CONNECTION
enp0s1  ethernet  connected  enp0s1
```

- `proto dhcp` → **`proto static`** 으로 바뀐 것이 전환 성공의 결정적 증거
- `/etc/resolv.conf` 가 자동 재생성됨 → 직접 편집이 불필요함을 확인
- macOS 에서 재접속: `ssh admin1@192.168.64.10` (포트 변경 전)

> 📝 **시험 포인트**: `nmcli con mod`(저장) → `nmcli con up`(적용) 2단계. "설정했는데 반영이 안 된다"의 정답은 재활성화 누락. `ip route` 의 `proto static`/`proto dhcp` 로 설정 출처 판별.

### 2-4. 설정 파일 확인 — keyfile 형식과 레거시 ifcfg 대응

> **상황**: `nmcli` 가 실제로 어떤 파일을 썼는지 확인한다. 시험은 여전히 `ifcfg-*` 항목을 묻기 때문에 두 형식의 항목을 1:1 로 맞춰 둔다.

```bash
ls -l /etc/NetworkManager/system-connections/
cat /etc/NetworkManager/system-connections/enp0s1.nmconnection
stat -c '%a %U:%G %n' /etc/NetworkManager/system-connections/enp0s1.nmconnection
ls -l /etc/sysconfig/network-scripts/            # RHEL9: 비어 있거나 없음
```

- keyfile 형식: INI 스타일 `[connection]` `[ipv4]` `[ipv6]` `[ethernet]` 섹션
- 파일 권한 **600 (root:root)** 필수 — 그렇지 않으면 NetworkManager 가 무시하고 경고 기록
- RHEL 9 기본 저장 플러그인은 `keyfile`. `ifcfg` 읽기는 `NetworkManager-initscripts-ifcfg-rh` 패키지가 있어야 가능하며 **신규 생성은 keyfile 로만** 됨

**keyfile ↔ 레거시 ifcfg 항목 대응표**

| 의미 | keyfile (`*.nmconnection`) | 레거시 (`ifcfg-enp0s1`) |
| --- | --- | --- |
| 장치 이름 | `[connection] interface-name=enp0s1` | `DEVICE=enp0s1` |
| 프로파일 이름 | `[connection] id=enp0s1` | `NAME=enp0s1` |
| 고유 ID | `[connection] uuid=...` | `UUID=...` |
| 연결 종류 | `[connection] type=ethernet` | `TYPE=Ethernet` |
| 부팅 시 자동 연결 | `[connection] autoconnect=true` (기본값·생략 가능) | `ONBOOT=yes` |
| 주소 획득 방식 | `[ipv4] method=manual` | `BOOTPROTO=none` (또는 `static`) |
| 주소 획득 방식(DHCP) | `[ipv4] method=auto` | `BOOTPROTO=dhcp` |
| IP 주소 + GW | `[ipv4] address1=192.168.64.10/24,192.168.64.1` | `IPADDR=192.168.64.10` |
| 프리픽스 | 위 `/24` 부분 | `PREFIX=24` |
| 넷마스크(구식) | 프리픽스로 통일 | `NETMASK=255.255.255.0` |
| 게이트웨이 | 위 `,192.168.64.1` 부분 (또는 `gateway=`) | `GATEWAY=192.168.64.1` |
| DNS 1·2 | `[ipv4] dns=192.168.64.1;8.8.8.8;` | `DNS1=192.168.64.1` / `DNS2=8.8.8.8` |
| 검색 도메인 | `[ipv4] dns-search=lab.local;` | `DOMAIN=lab.local` |
| IPv6 비활성 | `[ipv6] method=disabled` | `IPV6INIT=no` |
| 사용자 제어 허용 | `[connection] permissions=` | `USERCTL=yes` |

- keyfile 은 목록 구분자가 **세미콜론 `;`**, ifcfg 는 항목마다 별도 변수(`DNS1`,`DNS2`)
- `PREFIX` 와 `NETMASK` 는 **동시 지정 금지** — 충돌 시 `PREFIX` 우선

**검증**

```bash
grep -E '^(method|address1|dns|dns-search)=' /etc/NetworkManager/system-connections/enp0s1.nmconnection
stat -c '%a' /etc/NetworkManager/system-connections/enp0s1.nmconnection
```

```text
# cat /etc/NetworkManager/system-connections/enp0s1.nmconnection
[connection]
id=enp0s1
uuid=...
type=ethernet
interface-name=enp0s1

[ipv4]
address1=192.168.64.10/24,192.168.64.1
dns=192.168.64.1;8.8.8.8;
dns-search=lab.local;
method=manual

[ipv6]
addr-gen-mode=eui64
method=auto
# stat -c '%a' ...
600
```

> 📝 **시험 포인트**: `ifcfg-*` 항목 의미를 묻는 문제 다발 — `BOOTPROTO`(static/none/dhcp), `ONBOOT=yes`(부팅 시 활성), `IPADDR`/`PREFIX`/`NETMASK`, `GATEWAY`, `DNS1`. RHEL 9 는 keyfile 이 기본이라는 사실도 최신 회차에서 출제.

### 2-5. 프로파일 2개 만들고 전환

> **상황**: 같은 장치에 여러 프로파일을 만들어 두고 상황별로 갈아 끼우는 방식은 실무의 표준이다. 임시 프로파일 `lab-static` 을 만들어 전환·복귀를 실습한 뒤 정리한다.

```bash
nmcli con add type ethernet con-name lab-static ifname enp0s1 \
  ipv4.method manual \
  ipv4.addresses 192.168.64.12/24 \
  ipv4.gateway 192.168.64.1 \
  ipv4.dns 192.168.64.1 \
  connection.autoconnect no

nmcli con show                       # 프로파일 2개 확인
nmcli con up lab-static              # 전환 (⚠️ 콘솔에서)
ip -4 -o a show enp0s1               # .12 로 바뀜
nmcli con up enp0s1                  # 원래 프로파일로 복귀
ip -4 -o a show enp0s1               # .10 으로 복귀
```

- `con add` : 새 프로파일 생성
- `type ethernet` : 연결 종류 (`ethernet` `wifi` `bond` `bridge` `vlan` `team` 등)
- `con-name <이름>` : 프로파일 이름 (설정 파일명·`nmcli con` 목록 표시명)
- `ifname <장치>` : 바인딩할 실제 장치명
- `connection.autoconnect no` : 부팅 시 자동 활성화 안 함 → **두 프로파일이 부팅 때 경합하는 사고 방지**

**검증**

```bash
nmcli -t -f NAME,UUID,TYPE,DEVICE con show
nmcli con show --active | awk 'NR>1{print $1, $4}'
ip -4 -o a show enp0s1 | awk '{print $4}'
```

```text
# nmcli con show
NAME        UUID                                  TYPE      DEVICE
enp0s1      ...                                   ethernet  enp0s1
lab-static  ...                                   ethernet  --
# nmcli con up lab-static → ip -4 -o a
192.168.64.12/24
# nmcli con up enp0s1 → ip -4 -o a
192.168.64.10/24
```

**정리** — 실습 후 반드시 삭제 (남겨 두면 부팅 시 혼선)

```bash
nmcli con delete lab-static
nmcli con show | grep -c lab-static          # 0 이어야 함
ls /etc/NetworkManager/system-connections/   # lab-static.nmconnection 없음
ip -4 -o a show enp0s1 | awk '{print $4}'    # 192.168.64.10/24 유지
```

> 📝 **시험 포인트**: `nmcli con add type ethernet con-name <이름> ifname <장치>` 문형 출제. 같은 장치의 여러 프로파일 중 **활성은 하나**뿐이고 `con up` 이 전환 수단. `con delete` 는 프로파일과 설정 파일을 함께 제거.

### 2-6. nmcli 나머지 서브명령

> **상황**: 시험과 스크립트에서 쓰이는 `nmcli` 부속 명령을 한 번씩 통과시킨다. 위험한 것은 실행하지 않고 설명만 남긴다.

```bash
nmcli general status                        # NetworkManager 전체 상태 요약
nmcli general hostname                      # 호스트명 조회
nmcli general permissions                   # 현재 사용자 권한
nmcli networking connectivity check         # 인터넷 연결성 판정
nmcli con reload                            # 디스크의 설정 파일 다시 읽기
nmcli device reapply enp0s1                 # 재활성화 없이 변경분만 장치에 재적용
nmcli -t -f NAME,DEVICE con show            # 스크립트용 콜론 구분 출력
nmcli -f NAME,AUTOCONNECT con show          # autoconnect 일괄 점검
nmcli con mod enp0s1 +ipv4.routes "10.10.0.0/16 192.168.64.1"   # 영구 정적 경로 추가
nmcli con up enp0s1                         # 경로 반영
ip route | grep 10.10                       # 확인
nmcli con mod enp0s1 -ipv4.routes "10.10.0.0/16 192.168.64.1"   # 되돌리기
nmcli con up enp0s1
nmcli device wifi list                      # ※ 미실행 — 무선 NIC 없음
nmcli monitor                               # 실시간 이벤트 감시 (Ctrl+C 종료)
```

- `general status` 필드: `STATE`(connected/disconnected), `CONNECTIVITY`(full/limited/portal/none), `WIFI-HW`, `WIFI`, `WWAN`
- `connectivity check` 결과: `full`(정상) / `limited`(게이트웨이는 되나 인터넷 불가) / `portal`(로그인 필요 망) / `none`
- `con reload` : 파일을 직접 편집했을 때 NetworkManager 에 재읽기 지시 (**적용은 아님** → `con up` 별도)
- `device reapply` : 활성 상태를 유지한 채 변경분만 반영 → **세션 단절 최소화** (주소 자체가 바뀌면 여전히 끊김)
- `+ipv4.routes "<대상망> <게이트웨이>"` : 프로파일에 영구 정적 경로 추가 (`ip route add` 는 임시)

⚠️ 아래는 **설명만** — 실행 시 즉시 모든 네트워크가 내려가며 SSH 세션이 끊긴다.

```bash
nmcli networking off        # ⚠️ NetworkManager 관리 네트워크 전체 비활성 (실행 금지)
nmcli networking on         # 복구 — 콘솔에서만 가능
nmcli radio all off         # ⚠️ 무선 전체 차단
```

**검증**

```bash
nmcli general status
nmcli -t -f NAME,DEVICE con show
nmcli networking connectivity
```

```text
# nmcli general status
STATE      CONNECTIVITY  WIFI-HW  WIFI     WWAN-HW  WWAN
connected  full          missing  enabled  missing  enabled
# nmcli -t -f NAME,DEVICE con show
enp0s1:enp0s1
# nmcli networking connectivity
full
```

> 📝 **시험 포인트**: `nmcli -t -f <필드>` 조합은 스크립트 파싱용 — `-t`(terse, 콜론 구분)와 `-f`(fields) 를 짝으로 외울 것. `nmcli con reload`(파일 재읽기) ≠ `nmcli con up`(적용) 구분 출제.

---

## 3. 임시 조작 vs 영구 설정 (ip 명령)

### 3-1. ip addr add / del — 임시 보조 주소

> **상황**: 유지보수 중 잠깐 다른 주소가 필요하거나 IP 충돌을 조사할 때 쓰는 방식이다. 재부팅으로 사라진다는 점을 3-4 에서 직접 확인한다.

```bash
ip addr add 192.168.64.11/24 dev enp0s1        # 보조 주소 추가
ip -4 -br a show enp0s1                         # 주소 2개 확인
ping -c 2 -I 192.168.64.11 192.168.64.1         # 보조 주소로 송신 테스트
nmcli con show enp0s1 | grep ipv4.addresses     # 프로파일에는 반영 안 됨
ip addr del 192.168.64.11/24 dev enp0s1         # 제거
```

- `addr add <IP/프리픽스> dev <장치>` : 보조 주소 추가 — 기존 주소를 대체하지 않고 **병존**
- `addr del` : 동일 형식으로 제거 (프리픽스까지 정확히 일치해야 함)
- `label enp0s1:0` 을 붙이면 `ifconfig` 에도 별칭으로 표시됨 (`ip addr add ... label enp0s1:0`)
- `ping -I <IP|장치>` : 송신 출발 주소·인터페이스 지정 (**I**nterface)
- 특이사항
  - **재부팅 시 소멸** — NetworkManager 프로파일과 무관
  - `nmcli` 설정값과 동시에 존재 가능 → 혼동 주의
  - 임시 IP 자체가 충돌을 일으킬 수 있음 → 확인 후 즉시 제거

**IP 충돌 조사 절차 (실무 패턴)**

```bash
ip addr add 192.168.64.11/24 dev enp0s1     # 대역 진입
ping -c 2 192.168.64.50                     # 대상 확인
ip neigh show 192.168.64.50                 # MAC 표시되면 사용 중 확정
ip addr del 192.168.64.11/24 dev enp0s1     # 즉시 제거
```

**검증**

```bash
ip -4 -br a show enp0s1                      # 추가 직후: 주소 2개
ip -4 -br a show enp0s1                      # 삭제 후: 주소 1개
ip -4 -o a show enp0s1 | wc -l
```

```text
# ip addr add 192.168.64.11/24 dev enp0s1 → ip -4 -br a show enp0s1
enp0s1           UP             192.168.64.10/24 192.168.64.11/24
# ip addr del 192.168.64.11/24 dev enp0s1 → ip -4 -br a show enp0s1
enp0s1           UP             192.168.64.10/24
```

> 📝 **시험 포인트**: `ip addr add <IP>/<프리픽스> dev <인터페이스>` 는 실기 단골 서술 문제. **프리픽스·`dev` 키워드 누락이 감점 요인**. 구형 대응은 `ifconfig enp0s1:0 192.168.64.11 netmask 255.255.255.0 up`.

### 3-2. ip link set — 인터페이스 up/down 과 MTU

> **상황**: 링크를 내렸다 올리는 것은 가장 원초적인 네트워크 재시작이다. 원격에서 하면 그대로 잠기므로 반드시 콘솔에서 확인한다.

⚠️ `ip link set enp0s1 down` 은 **즉시 모든 통신을 끊는다**. UTM 콘솔에서만 실행하고, `down` 과 `up` 을 한 줄에 묶어 실행할 것.

```bash
# UTM 콘솔에서
ip link set enp0s1 down && sleep 2 && ip link set enp0s1 up
ip -br link show enp0s1                     # 상태 확인
ip -br a show enp0s1                        # 주소가 사라졌는지 확인
nmcli con up enp0s1                         # 주소 재적용 (down 시 주소도 함께 소멸)
```

- `link set <장치> down|up` : 관리 상태 변경
- `link set <장치> mtu <값>` : MTU 변경 (임시)
- `link set <장치> address <MAC>` : MAC 변경 (down 상태에서만)
- `link set <장치> name <새이름>` : 인터페이스 이름 변경 (down 상태에서만)
- 주의: `down` 하면 **해당 장치의 IP 주소와 경로도 함께 제거**됨

**MTU 조작 (참고)**

```bash
ip link show enp0s1 | grep -o 'mtu [0-9]*'      # 현재 MTU (기본 1500)
ip link set enp0s1 mtu 1400                     # 임시 변경
ping -c 2 -M do -s 1372 192.168.64.1            # 단편화 금지 상태로 경계 확인
ip link set enp0s1 mtu 1500                     # 복구
nmcli con mod enp0s1 802-3-ethernet.mtu 1400    # 영구 설정 방법 (참고 · 실행 선택)
```

- `-M do` : Path MTU Discovery 를 `do` 로 — **단편화 금지(DF 비트)** 설정
- `-s <바이트>` : ICMP 페이로드 크기 (**s**ize) — 실제 패킷 = `-s` 값 + ICMP 8 + IP 20
  - MTU 1400 → 페이로드 최대 `1400 - 28 = 1372`
- MTU 초과 + DF 이면 `Frag needed and DF set (mtu = 1400)` 오류 → PMTU 문제 진단 근거

**검증**

```bash
ip -br link show enp0s1
ip link show enp0s1 | grep -o 'mtu [0-9]*'
ip -4 -o a show enp0s1 | awk '{print $4}'
```

```text
# ip -br link show enp0s1  (down 상태)
enp0s1           DOWN           <MAC> <BROADCAST,MULTICAST>
# ip -br link show enp0s1  (up 복구 후)
enp0s1           UP             <MAC> <BROADCAST,MULTICAST,UP,LOWER_UP>
# ip link show enp0s1 | grep -o 'mtu [0-9]*'
mtu 1500
```

> 📝 **시험 포인트**: 인터페이스 활성/비활성 = `ip link set <장치> up|down` (구 `ifup`/`ifdown`, `ifconfig <장치> up|down`). 이더넷 기본 MTU **1500**, 점보 프레임 9000 은 암기값.

### 3-3. ip route add / del — 임시 정적 경로

> **상황**: 특정 대역만 다른 경로로 보내야 하는 상황을 흉내 낸다. 임시 경로를 넣고 `ip route get` 으로 라우팅 결정이 바뀌는지 확인한 뒤 되돌린다.

```bash
ip route get 10.10.5.5                                 # 경로 추가 전 결정 확인
ip route add 10.10.0.0/16 via 192.168.64.1             # 정적 경로 추가
ip route                                                # 테이블에 추가됨
ip route get 10.10.5.5                                 # 결정이 바뀌었는지
ip route add 172.20.0.0/16 dev enp0s1                  # 게이트웨이 없이 링크 직결 경로
ip route del 172.20.0.0/16
ip route del 10.10.0.0/16                              # 원복
ip route get 8.8.8.8                                   # 기본 경로 정상 확인
```

- `route add <대상망> via <게이트웨이>` : 게이트웨이 경유 경로
- `route add <대상망> dev <장치>` : 같은 링크에 직접 붙어 있는 경로 (게이트웨이 불필요)
- `route add default via <GW>` : 기본 경로 지정
- `route replace` : 있으면 교체, 없으면 추가 (`add` 는 중복 시 `File exists` 오류)
- `route get <IP>` : 커널의 실제 라우팅 결정 결과 (테이블 나열이 아니라 **계산 결과**)
- `metric <값>` : 우선순위 — 작을수록 우선

**검증**

```bash
ip route | grep 10.10
ip route get 10.10.5.5 | head -1
ip route | grep -c '10.10'      # 삭제 후 0
```

```text
# ip route add 10.10.0.0/16 via 192.168.64.1 → ip route
default via 192.168.64.1 dev enp0s1 proto static metric 100
10.10.0.0/16 via 192.168.64.1 dev enp0s1
192.168.64.0/24 dev enp0s1 proto kernel scope link src 192.168.64.10 metric 100
# ip route get 10.10.5.5
10.10.5.5 via 192.168.64.1 dev enp0s1 src 192.168.64.10 uid 0
```

- 영구화가 필요하면 2-6 의 `nmcli con mod enp0s1 +ipv4.routes "10.10.0.0/16 192.168.64.1"` 사용

> 📝 **시험 포인트**: `ip route add <네트워크>/<프리픽스> via <게이트웨이>` = 구 `route add -net <네트워크> netmask <마스크> gw <GW>`. 기본 경로 삭제는 `ip route del default`. `route get` 은 "어떤 경로가 선택되는가"를 묻는 문제의 검증 수단.

### 3-4. 임시 vs 영구 — 재부팅 검증

> **상황**: "설정했는데 재부팅하니 사라졌다"는 사고의 원인을 직접 만들어 확인한다. 임시 주소·임시 경로를 넣고 재부팅한다.

```bash
ip addr add 192.168.64.11/24 dev enp0s1
ip route add 10.10.0.0/16 via 192.168.64.1
ip -4 -br a show enp0s1; ip route | grep 10.10      # 재부팅 전 존재 확인
sync; reboot
```

- 재부팅 후 root 로그인 → 아래 검증

**검증**

```bash
ip -4 -br a show enp0s1        # 192.168.64.10/24 만 (보조 주소 소멸)
ip route | grep -c 10.10       # 0 (임시 경로 소멸)
ip route show default          # proto static 유지 (nmcli 설정은 살아남음)
cat /etc/resolv.conf | grep -c nameserver     # 2 (DNS 유지)
nmcli device status
```

```text
# ip -4 -br a show enp0s1
enp0s1           UP             192.168.64.10/24
# ip route | grep -c 10.10
0
# ip route show default
default via 192.168.64.1 dev enp0s1 proto static metric 100
```

**임시 ↔ 영구 대응표**

| 항목 | 임시 (재부팅 시 소멸) | 영구 (프로파일 저장) |
| --- | --- | --- |
| IP 주소 | `ip addr add 192.168.64.11/24 dev enp0s1` | `nmcli con mod enp0s1 ipv4.addresses 192.168.64.10/24` |
| 게이트웨이 | `ip route add default via 192.168.64.1` | `nmcli con mod enp0s1 ipv4.gateway 192.168.64.1` |
| DNS | `/etc/resolv.conf` 직접 편집 (NM 이 덮어씀) | `nmcli con mod enp0s1 ipv4.dns "192.168.64.1 8.8.8.8"` |
| 검색 도메인 | 위와 동일 | `nmcli con mod enp0s1 ipv4.dns-search lab.local` |
| 정적 경로 | `ip route add 10.10.0.0/16 via 192.168.64.1` | `nmcli con mod enp0s1 +ipv4.routes "10.10.0.0/16 192.168.64.1"` |
| MTU | `ip link set enp0s1 mtu 1400` | `nmcli con mod enp0s1 802-3-ethernet.mtu 1400` |
| 인터페이스 활성 | `ip link set enp0s1 up` | `nmcli con mod enp0s1 connection.autoconnect yes` |
| 호스트명 | `hostname srv01` | `hostnamectl set-hostname srv01.lab.local` |
| 커널 파라미터 | `sysctl -w net.ipv4.ip_forward=1` | `/etc/sysctl.d/*.conf` 에 기록 |

- 적용 후 반영: 임시 명령은 **즉시**, `nmcli con mod` 는 **`con up` 필요**

> 📝 **시험 포인트**: "재부팅 후에도 유지되는 설정 방법"을 묻는 실기 문제의 핵심 — `ip` 명령은 임시, `nmcli`(또는 설정 파일)는 영구. 이 대응표가 그대로 답안이 된다.

### 3-5. 커널 파라미터 — IP 포워딩

> **상황**: Part 10 에서 NAT·라우터 실습을 하려면 커널이 패킷을 전달하도록 켜야 한다. 지금은 값 확인과 임시/영구 설정 방법만 익히고 원복한다.

```bash
sysctl net.ipv4.ip_forward                       # 현재 값 (0 = 비활성)
cat /proc/sys/net/ipv4/ip_forward                # 동일 값, 파일 직접 조회
sysctl -w net.ipv4.ip_forward=1                  # 임시 활성
sysctl net.ipv4.ip_forward
echo 'net.ipv4.ip_forward = 1' > /etc/sysctl.d/99-lab-forward.conf   # 영구 (참고)
sysctl --system | tail -5                        # 모든 sysctl.d 파일 재적용
sysctl -a | grep -E 'ip_forward|rp_filter|icmp_echo_ignore' | head
sysctl -w net.ipv4.ip_forward=0                  # 원복
rm -f /etc/sysctl.d/99-lab-forward.conf          # 영구 설정도 원복
```

- `sysctl <키>` : 값 조회
- `-w <키>=<값>` : 값 변경 (**w**rite) — 재부팅 시 소멸
- `-a` : 전체 파라미터 나열 (**a**ll)
- `-p [파일]` : 파일에서 읽어 적용 (기본 `/etc/sysctl.conf`)
- `--system` : `/etc/sysctl.d/`·`/run/sysctl.d/`·`/usr/lib/sysctl.d/` 전부 순서대로 적용
- 관련 파라미터
  - `net.ipv4.ip_forward` : 라우터 역할 활성 (NAT·게이트웨이 필수)
  - `net.ipv4.conf.all.rp_filter` : 역경로 필터 — 출발지 위조 패킷 차단
  - `net.ipv4.icmp_echo_ignore_all` : `1` 이면 ping 무응답 (보안 목적)
  - `net.ipv4.tcp_syncookies` : SYN Flooding 대응 → 상세는 [[10-security-firewall-selinux]]

**검증**

```bash
sysctl -n net.ipv4.ip_forward       # 0
ls /etc/sysctl.d/ | grep -c lab     # 0 (원복 확인)
```

```text
# sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 0
# sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1
# sysctl -n net.ipv4.ip_forward   (원복 후)
0
```

- NAT·마스커레이드 실제 구성은 [[10-security-firewall-selinux]] 에서 `firewall-cmd --add-masquerade` 와 함께 수행

> 📝 **시험 포인트**: 리눅스를 라우터로 쓰려면 `net.ipv4.ip_forward = 1`. 임시는 `sysctl -w` 또는 `echo 1 > /proc/sys/net/ipv4/ip_forward`, 영구는 `/etc/sysctl.conf`·`/etc/sysctl.d/*.conf` + `sysctl -p`. 이 3가지 표현이 모두 정답 후보로 출제.

---

## 4. 이름 해석 (hosts · DNS 클라이언트)

### 4-1. /etc/hosts 정적 매핑 등록

> **상황**: Part 09 에서 DNS 서버(BIND)를 세우기 전까지 `srv01.lab.local`·`intranet.lab.local` 이름으로 접근할 수 있게 정적 매핑을 넣는다. 이후 Apache 가상호스트 검증이 이 이름으로 이뤄진다.

```bash
cp /etc/hosts /root/net-backup/hosts.before          # 백업
cat >> /etc/hosts <<'EOF'
192.168.64.10   srv01.lab.local srv01 intranet.lab.local
EOF
cat /etc/hosts
```

- `/etc/hosts` 형식: `<IP주소>  <정규 이름(FQDN)>  [별칭...]`
  - **두 번째 필드가 정규 이름(canonical name)** — `hostname -f` 가 이 값을 돌려줌
  - 세 번째 이후는 모두 별칭 → 하나의 IP 에 여러 이름 매핑 가능
  - 한 IP 를 여러 줄에 나눠 쓰면 **먼저 나온 줄이 우선**
- 기본 항목의 의미
  - `127.0.0.1 localhost localhost.localdomain localhost4 ...` : IPv4 루프백
  - `::1 localhost localhost.localdomain localhost6 ...` : IPv6 루프백

**검증**

```bash
getent hosts srv01                     # nsswitch 경유 조회
getent hosts srv01.lab.local
getent hosts intranet.lab.local
ping -c 1 srv01                        # 짧은 이름으로 도달
ping -c 1 intranet.lab.local
hostname -f                            # FQDN 정상 해석
```

```text
# getent hosts srv01
192.168.64.10   srv01.lab.local srv01 intranet.lab.local
# ping -c 1 srv01
PING srv01.lab.local (192.168.64.10) 56(84) bytes of data.
64 bytes from srv01.lab.local (192.168.64.10): icmp_seq=1 ttl=64 time=0.0... ms
--- srv01.lab.local ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
```

- `ttl=64` = 로컬(자기 자신)로 해석되었다는 근거 — 5-1 의 TTL 해석 참조

> 📝 **시험 포인트**: `/etc/hosts` 는 **IP → 이름 정적 매핑 파일**이며 `hosts: files dns` 순서상 **DNS 보다 먼저** 조회. 필드 순서 `IP → FQDN → 별칭` 이 채점 포인트. 파밍(pharming) 공격이 이 파일 변조를 이용한다는 보안 문항과도 연결.

### 4-2. hosts 는 DNS 가 아니다 — dig NXDOMAIN 확인

> **상황**: `ping srv01` 은 되는데 `dig srv01.lab.local` 은 실패한다. 두 경로가 완전히 다르다는 사실을 실물로 확인해 둔다. 이 차이를 모르면 Part 09 의 DNS 구축 검증에서 헤맨다.

```bash
rpm -q bind-utils || dnf install -y bind-utils     # dig·nslookup·host 제공
ping -c 1 srv01.lab.local        # 성공 — /etc/hosts 경유 (glibc resolver)
getent hosts srv01.lab.local     # 성공 — nsswitch 경유
dig srv01.lab.local +short       # 무응답 — DNS 서버에는 이 이름이 없음
dig srv01.lab.local | grep -E 'status|ANSWER:'
host srv01.lab.local             # NXDOMAIN
```

- `bind-utils` 패키지가 제공하는 도구: `dig` `nslookup` `host` `nsupdate` `dnssec-*`
- **핵심 차이**
  - `ping`·`getent`·브라우저·대부분의 응용 프로그램 → **glibc resolver** 사용 → `nsswitch.conf` 를 따름 → `/etc/hosts` 를 봄
  - `dig`·`nslookup`·`host` → **DNS 프로토콜로 직접 질의** → `nsswitch.conf` 를 **무시**하고 `/etc/resolv.conf` 의 nameserver 에만 물음
- 따라서 `/etc/hosts` 에 넣은 이름은 `dig` 로 절대 조회되지 않음

**검증**

```bash
dig srv01.lab.local | grep -E '^;; ->>HEADER|ANSWER SECTION'
getent hosts srv01.lab.local | wc -l      # 1 (해석 성공)
```

```text
# dig srv01.lab.local
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: ...
;; QUESTION SECTION:
;srv01.lab.local.               IN      A
(ANSWER SECTION 없음)
# host srv01.lab.local
Host srv01.lab.local not found: 3(NXDOMAIN)
```

- `NXDOMAIN` = Non-eXistent DOMAIN, 해당 이름이 DNS 에 존재하지 않음
- 이 이름을 DNS 로도 해석되게 하려면 Part 09 에서 BIND 존 파일에 A 레코드를 등록해야 함 → [[09-network-services]]

> 📝 **시험 포인트**: "`/etc/hosts` 에 등록했는데 `nslookup` 이 안 된다" 는 상황 판단 문제의 정답은 **`nslookup`/`dig` 는 hosts 파일을 참조하지 않음**. `getent hosts` 는 참조함 — 이 3자 구분이 함정.

### 4-3. nslookup 과 host

> **상황**: `dig` 보다 간결한 두 도구를 통과시킨다. 시험에서 출력 해석 문제로 나오므로 `Non-authoritative answer` 같은 문구의 의미까지 확인한다.

```bash
nslookup rockylinux.org                     # 기본 A 조회
nslookup rockylinux.org 8.8.8.8             # DNS 서버 지정
nslookup -type=MX rockylinux.org            # 레코드 유형 지정
nslookup -type=NS rockylinux.org
nslookup 8.8.8.8                            # 역방향 조회 (PTR)
host rockylinux.org                         # 간단 조회 (A/AAAA/MX 요약)
host -t A rockylinux.org                    # 유형 지정
host -t MX rockylinux.org
host -t NS rockylinux.org
host -t TXT rockylinux.org
host -a rockylinux.org                      # 전체 레코드 (= -t ANY + verbose)
host 8.8.8.8                                # 역방향
```

- `nslookup` 옵션
  - `-type=<유형>` (= `-querytype=`, `-q=`) : 조회할 레코드 유형
  - 인자 두 번째에 서버 IP → 해당 서버에 직접 질의
  - 인자 없이 실행하면 대화형 모드 (`exit` 로 종료)
- `host` 옵션
  - `-t <유형>` : 레코드 유형 지정 (**t**ype)
  - `-a` : 전체 레코드 상세 (**a**ll)
  - `-v` : 상세 출력 (**v**erbose)
  - `-4` / `-6` : 질의에 IPv4/IPv6 전송만 사용

**검증**

```bash
host -t A rockylinux.org | tail -2
nslookup rockylinux.org | grep -A3 'Non-authoritative'
```

```text
# nslookup rockylinux.org
Server:         192.168.64.1
Address:        192.168.64.1#53

Non-authoritative answer:
Name:   rockylinux.org
Address: ...
# host -t MX rockylinux.org
rockylinux.org mail is handled by <우선순위> <메일서버>.
```

- `Server:`/`Address:` = **질의를 보낸 DNS 서버** (`/etc/resolv.conf` 첫 nameserver)
- `Non-authoritative answer` = 권한 있는 네임서버가 아니라 **캐시에서 온 응답** — 정상. 권한 서버가 직접 답하면 이 문구가 없음

> 📝 **시험 포인트**: `Non-authoritative answer` = 캐싱 DNS 서버의 응답(오류 아님). `host -t mx <도메인>` 출력의 "mail is handled by \<우선순위\> \<서버\>" 에서 **숫자가 작을수록 우선**인 MX preference 해석 출제.

### 4-4. dig 기본 사용과 출력 섹션 해석

> **상황**: `dig` 는 DNS 응답을 프로토콜 구조 그대로 보여준다. Part 09 에서 만든 존이 제대로 응답하는지 판정하려면 이 섹션 구조를 읽을 줄 알아야 한다.

```bash
dig rockylinux.org                          # 기본 (A 레코드)
dig @192.168.64.1 rockylinux.org            # 질의 서버 지정
dig @8.8.8.8 rockylinux.org                 # 외부 공개 DNS 로 질의
dig rockylinux.org +short                   # 결과 값만
dig rockylinux.org +noall +answer           # ANSWER 섹션만
dig rockylinux.org +nocmd +noall +answer +stats
dig rockylinux.org A +norecurse             # 재귀 질의 금지 (캐시만 확인)
```

- `@<서버>` : 질의를 보낼 DNS 서버 지정 (미지정 시 `/etc/resolv.conf` 첫 nameserver)
- `+short` : 값만 한 줄씩 출력 — 스크립트용
- `+noall` : 모든 섹션 출력 끄기 → 필요한 것만 `+answer` 등으로 다시 켜는 방식
- `+answer` / `+authority` / `+additional` / `+question` / `+comments` / `+stats` : 섹션별 on/off
- `+nocmd` : 첫 줄의 실행 명령 에코 제거
- `+trace` : 루트부터 단계적 위임 추적 (4-5)
- `+norecurse` (`+nord`) : RD 플래그 끔 — 캐시에 있는 것만 응답받음
- `+tcp` : UDP 대신 TCP 로 질의 (응답이 512바이트 초과·존 전송 시)
- `+time=<초>` / `+tries=<n>` : 타임아웃·재시도

**dig 출력 섹션 구조**

| 섹션 | 내용 | 읽는 법 |
| --- | --- | --- |
| `;; ->>HEADER<<-` | `opcode`, **`status`**, `id` | `status: NOERROR`(정상) / `NXDOMAIN`(이름 없음) / `SERVFAIL`(서버 오류) / `REFUSED`(질의 거부) |
| `;; flags:` | `qr aa rd ra ad cd` | `qr`=응답, **`aa`=권한 있는 응답**, `rd`=재귀 요청함, `ra`=재귀 가능, `ad`=DNSSEC 검증됨 |
| `;; QUESTION SECTION` | 무엇을 물었는가 | `<이름>. IN A` |
| `;; ANSWER SECTION` | **답** | `<이름> <TTL> IN <유형> <값>` |
| `;; AUTHORITY SECTION` | 이 존의 권한 네임서버(NS) | 답이 없을 때 위임 정보로 채워짐 |
| `;; ADDITIONAL SECTION` | 부가 정보 (NS 의 A 레코드 등) | 추가 질의를 아끼게 해 주는 글루(glue) 레코드 |
| `;; Query time:` | 응답 소요 시간(ms) | 0ms 에 가까우면 **캐시 적중** |
| `;; SERVER:` | 실제 질의한 서버·포트 | `192.168.64.1#53` |

- `TTL` : 이 레코드를 캐시에 보관해도 되는 초 수 — 반복 질의 시 값이 **줄어들면 캐시 응답**

**검증**

```bash
dig rockylinux.org | grep -E 'status:|flags:|ANSWER SECTION|Query time|SERVER:'
dig rockylinux.org +short | head -2
dig rockylinux.org | grep -A2 'ANSWER SECTION'; sleep 2
dig rockylinux.org | grep -A2 'ANSWER SECTION'    # TTL 감소 = 캐시 적중
```

```text
# dig rockylinux.org
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: ...
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;rockylinux.org.                        IN      A

;; ANSWER SECTION:
rockylinux.org.         ...     IN      A       ...

;; Query time: ... msec
;; SERVER: 192.168.64.1#53(192.168.64.1) (UDP)
```

> 📝 **시험 포인트**: `dig` 출력에서 **`status: NOERROR` + ANSWER SECTION 존재** = 정상 해석. `flags` 의 `aa`(authoritative answer) 유무로 권한 서버 응답/캐시 응답 구분. `dig +short` 는 값만, `dig +trace` 는 루트부터 추적 — 두 옵션 구분 출제.

### 4-5. dig 레코드 유형별 조회와 역방향·추적

> **상황**: Part 09 에서 만들 존 파일에 넣을 레코드 유형을 실제 도메인으로 미리 조회해 형태를 익힌다. 역방향 조회와 위임 추적도 함께 통과시킨다.

```bash
dig rockylinux.org A     +noall +answer          # IPv4 주소
dig rockylinux.org AAAA  +noall +answer          # IPv6 주소
dig rockylinux.org MX    +noall +answer          # 메일 서버
dig rockylinux.org NS    +noall +answer          # 네임서버
dig rockylinux.org SOA   +noall +answer          # 존 권한 정보
dig rockylinux.org TXT   +noall +answer          # 텍스트(SPF·검증용)
dig www.rockylinux.org CNAME +noall +answer      # 별칭
dig -x 8.8.8.8 +short                            # 역방향 조회 (PTR)
dig -x 1.1.1.1 +noall +answer
dig +trace rockylinux.org | tail -20             # 루트 → TLD → 권한 서버 추적
```

- `-x <IP>` : 역방향 조회 — `8.8.8.8` 을 `8.8.8.8.in-addr.arpa` 로 자동 변환해 PTR 질의
- `+trace` : 재귀 서버에 맡기지 않고 **루트 네임서버부터 직접 따라감** → 위임 체인 확인, DNS 장애 지점 특정
- IPv6 역방향은 `ip6.arpa` 도메인 사용

**DNS 레코드 유형표 (필수 암기)**

| 유형 | 이름 | 역할 | 존 파일 예 |
| --- | --- | --- | --- |
| `A` | Address | 도메인 → **IPv4** 주소 | `srv01  IN  A     192.168.64.10` |
| `AAAA` | Quad-A | 도메인 → **IPv6** 주소 | `srv01  IN  AAAA  2001:db8::10` |
| `CNAME` | Canonical Name | 별칭 → 정규 이름 (**IP 아님**) | `www    IN  CNAME srv01.lab.local.` |
| `MX` | Mail eXchanger | 메일 수신 서버 + **우선순위(작을수록 우선)** | `@      IN  MX 10 mail.lab.local.` |
| `NS` | Name Server | 이 존의 권한 네임서버 | `@      IN  NS    ns1.lab.local.` |
| `PTR` | Pointer | **IP → 도메인** (역방향 존 전용) | `10     IN  PTR   srv01.lab.local.` |
| `SOA` | Start Of Authority | 존 시작·**Serial**·Refresh·Retry·Expire·Minimum | 존 파일 맨 앞 1개 |
| `TXT` | Text | 임의 텍스트 — SPF·DKIM·소유권 검증 | `@      IN  TXT   "v=spf1 ..."` |
| `SRV` | Service | 서비스 위치(프로토콜·포트·가중치) | `_sip._tcp IN SRV 10 5 5060 sip.lab.local.` |

- `SOA` 의 **Serial** 은 존 변경 시 반드시 증가 → 세컨더리가 이 값을 보고 존 전송(AXFR) 수행 여부 결정
- `CNAME` 은 같은 이름에 다른 레코드와 공존 불가, 존 정점(`@`)에는 사용 불가
- 존 파일 작성·`named` 구성은 [[09-network-services]]

**검증**

```bash
dig rockylinux.org NS +short | wc -l         # NS 개수 (2개 이상이 정상)
dig -x 8.8.8.8 +short                        # dns.google.
dig rockylinux.org SOA +short                # 7개 필드 (MNAME RNAME Serial Refresh Retry Expire Minimum)
```

```text
# dig -x 8.8.8.8 +short
dns.google.
# dig rockylinux.org SOA +short
<주 네임서버> <관리자메일> <Serial> <Refresh> <Retry> <Expire> <Minimum>
# dig +trace rockylinux.org | tail -5
rockylinux.org.   ...  IN  NS  ...
rockylinux.org.   ...  IN  A   ...
;; Received ... bytes from ...#53(...) in ... ms
```

> 📝 **시험 포인트**: 레코드 유형 매칭이 매 회차 출제 — **역방향=PTR**, **메일=MX(숫자 작을수록 우선)**, **별칭=CNAME**, **IPv6=AAAA**, **존 정보=SOA(Serial 증가 필수)**. `dig -x` = 역방향 조회, `dig +trace` = 루트부터 위임 추적.

### 4-6. resolvectl / systemd-resolved 상태 확인 (정직 표기)

> **상황**: 우분투 계열 자료에는 `resolvectl` 이 자주 나오지만 RHEL 계열은 구성이 다르다. 이 서버가 어느 쪽인지 확정해 둔다.

```bash
systemctl is-enabled systemd-resolved 2>&1     # 활성화 여부
systemctl is-active systemd-resolved 2>&1
ls -l /etc/resolv.conf                          # 심볼릭 링크인지 일반 파일인지
resolvectl status 2>&1 | head                   # 조회 시도
```

- `resolvectl` : `systemd-resolved` 제어 도구 (`systemd-resolved` 패키지 제공)
- `resolvectl status` / `query <이름>` / `flush-caches` / `dns <if> <서버>` 서브명령 보유

**검증**

```bash
systemctl is-active systemd-resolved 2>&1
head -1 /etc/resolv.conf
file /etc/resolv.conf
```

```text
# systemctl is-active systemd-resolved
inactive
# head -1 /etc/resolv.conf
# Generated by NetworkManager
# file /etc/resolv.conf
/etc/resolv.conf: ASCII text          ← 일반 파일 (심볼릭 링크 아님)
```

- ※ **RHEL 9 / Rocky 9 는 `systemd-resolved` 가 기본 비활성**이며, `/etc/resolv.conf` 를 NetworkManager 가 직접 생성한다
- 따라서 `resolvectl status` 는 서비스가 내려가 있으면 조회 실패 메시지를 반환할 수 있음 → 이 환경의 DNS 확인 정본은 **`cat /etc/resolv.conf`** 와 **`nmcli con show enp0s1 | grep -i dns`**
- Ubuntu/Debian 계열은 `/etc/resolv.conf` 가 `../run/systemd/resolve/stub-resolv.conf` 심볼릭 링크이고 `nameserver 127.0.0.53` (스텁 리졸버) → **배포판 차이로 출제 가능**

> 📝 **시험 포인트**: DNS 서버 지정 파일은 `/etc/resolv.conf`. RHEL 계열에서 이 파일을 직접 고쳐도 NetworkManager 가 덮어씀 → 영구 설정은 `nmcli ipv4.dns`. 굳이 파일을 지키려면 `chattr +i /etc/resolv.conf` (Part 03 참조) — 기출에도 등장하는 우회법.

---

## 5. 연결 진단 도구

### 5-1. ping 옵션 전개와 TTL 해석

> **상황**: 고정 IP 전환 후 계층별 도달성을 순서대로 확인한다. `ping` 의 옵션을 전부 통과시키면서 응답의 `ttl` 값으로 상대 OS 를 추정하는 요령도 익힌다.

```bash
ping -c 4 192.168.64.1                                   # 기본 4회
ping -c 4 -i 0.5 192.168.64.1                            # 0.5초 간격
ping -c 3 -s 1000 192.168.64.1                           # 페이로드 1000바이트
ping -c 3 -W 1 192.168.64.1                              # 응답 대기 1초
ping -c 10 -w 5 192.168.64.1                             # 전체 5초 후 종료
ping -c 2 -n 192.168.64.1                                # 이름 역해석 생략
ping -c 2 -I enp0s1 192.168.64.1                         # 송신 인터페이스 지정
ping -c 4 -i 0.5 -s 1000 -W 1 -w 5 -n 192.168.64.1       # 전체 조합
ping -c 2 127.0.0.1                                      # 루프백 (TTL 64)
ping -c 2 8.8.8.8                                        # 외부 (TTL 감소 확인)
```

- `-c <n>` : 지정 횟수 전송 후 종료 (**c**ount) — 미지정 시 무한 반복(`Ctrl+C`)
- `-i <초>` : 전송 간격 (**i**nterval) — 0.2 미만은 root 전용
- `-s <바이트>` : ICMP 페이로드 크기 (**s**ize, 기본 56 → IP+ICMP 헤더 28 포함해 84바이트)
- `-W <초>` : **개별 응답** 대기 제한 (대문자 **W**ait)
- `-w <초>` : **전체 실행** 시간 제한 (소문자 **w**, deadline)
- `-n` : 응답 IP 를 이름으로 바꾸지 않음 (**n**umeric)
- `-I <if|IP>` : 송신 인터페이스·출발 주소 지정 (**I**nterface)
- `-t <ttl>` : 보낼 패킷의 TTL 설정 (traceroute 원리)
- `-q` : 요약만 출력 (**q**uiet)
- `-4` / `-6` : 프로토콜 강제

⚠️ `-f`(flood) 는 **응답을 기다리지 않고 최대 속도로 전송** — root 전용이며 대상 네트워크에 부하를 준다. 실습망 자기 자신 외에는 사용 금지.

```bash
ping -f -c 100 127.0.0.1        # ⚠️ 참고 — 루프백 한정, root 필요
```

**TTL 로 대상 OS 추정 (기본 TTL 값)**

| 응답 `ttl` 관측값 | 초기 TTL 추정 | 대상 OS·장비 |
| --- | --- | --- |
| 64 이하 (57~64) | 64 | **Linux / Unix / macOS / Android** |
| 128 이하 (121~128) | 128 | **Windows** |
| 255 이하 (248~255) | 255 | 네트워크 장비(Cisco IOS), 일부 Unix(Solaris) |

- 관측 TTL = 초기 TTL − 경유한 라우터 수(홉 수) → `64 - 관측값` 으로 홉 수 역산 가능
- 자기 자신·같은 대역 응답은 라우터를 안 거치므로 초기값 그대로 (`ttl=64`)

**검증**

```bash
ping -c 1 127.0.0.1  | grep -o 'ttl=[0-9]*'
ping -c 1 192.168.64.1 | grep -o 'ttl=[0-9]*'
ping -c 1 8.8.8.8    | grep -o 'ttl=[0-9]*'
ping -c 4 -q 192.168.64.1 | tail -2
```

```text
# ping -c 4 -i 0.5 -s 1000 -W 1 -w 5 -n 192.168.64.1
PING 192.168.64.1 (192.168.64.1) 1000(1028) bytes of data.
1008 bytes from 192.168.64.1: icmp_seq=1 ttl=64 time=0.4... ms
...
--- 192.168.64.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time ...ms
rtt min/avg/max/mdev = .../.../.../... ms
```

**단계별 진단 순서 (실패 지점 = 원인)**

```bash
ping -c 2 127.0.0.1        # ① 자기 자신(TCP/IP 스택)
ping -c 2 192.168.64.10    # ② 자기 IP (인터페이스 설정)
ping -c 2 192.168.64.1     # ③ 게이트웨이 (같은 대역·링크)
ping -c 2 8.8.8.8          # ④ 외부 IP (라우팅·NAT)
ping -c 2 rockylinux.org   # ⑤ 이름 해석 (DNS)
```

- ① 실패 → TCP/IP 스택 이상 (사실상 드묾)
- ② 실패 → IP 미할당 → `ip a` 확인. **IP 없으면 대상 무관하게 `unreachable`** (상대 장애 아님)
- ③ 실패 → 서브넷·넷마스크 오류 또는 물리 링크 (`ip -br link` 의 `NO-CARRIER`)
- ④ 실패 → 게이트웨이 값 오류 (`ip route show default`)
- ⑤ 만 실패 → **DNS 문제** (`cat /etc/resolv.conf`, `nmcli ipv4.dns`)

> 📝 **시험 포인트**: `-c`(횟수) vs `-i`(간격) vs `-W`(개별 대기) vs `-w`(전체 시간) 4개 구분이 최빈출. "`ping 8.8.8.8` 은 되는데 `ping google.com` 은 안 된다" → **DNS 설정(`/etc/resolv.conf`)** 점검이 정답. TTL 64=Linux / 128=Windows 도 단골.

### 5-2. 경로 추적 — traceroute · tracepath · mtr

> **상황**: 게이트웨이 너머에서 어디까지 가는지 확인한다. 홉마다 어떤 프로토콜을 쓰는지가 도구별 차이이자 출제 포인트다.

```bash
rpm -q traceroute || dnf install -y traceroute
rpm -q mtr        || dnf install -y mtr

traceroute -n 8.8.8.8                # 기본 UDP 방식, 이름 역해석 생략
traceroute -n -I 8.8.8.8             # ICMP Echo 방식 (root 필요)
traceroute -n -T -p 443 8.8.8.8      # TCP SYN 방식 (방화벽 우회에 유리, root)
traceroute -n -m 10 8.8.8.8          # 최대 홉 10
traceroute -n -q 1 8.8.8.8           # 홉당 프로브 1회
tracepath 8.8.8.8                    # root 불필요, PMTU 도 함께 표시
tracepath -n 8.8.8.8
mtr -r -c 5 8.8.8.8                  # 리포트 모드 5회 (ping+traceroute 결합)
mtr -rn -c 5 8.8.8.8
```

- `traceroute` 옵션
  - `-n` : 이름 역해석 생략 (**n**umeric) — 훨씬 빠름
  - `-I` : ICMP Echo 사용 (**I**CMP) — root 필요, `ping` 과 같은 방식
  - `-T` : TCP SYN 사용 (**T**CP) — root 필요, `-p` 로 포트 지정
  - `-U` : UDP 사용 (기본값)
  - `-m <n>` : 최대 홉 수 (**m**ax TTL, 기본 30)
  - `-q <n>` : 홉당 프로브 개수 (**q**ueries, 기본 3)
  - `-w <초>` : 응답 대기
- `tracepath` : `iputils` 제공, **root 불필요**, 경로상 MTU 변화(`pmtu`) 표시
- `mtr` : My TraceRoute — 각 홉에 반복 ping 을 보내 **손실률·지연 통계**를 실시간 갱신
  - `-r` : 리포트 모드(한 번 실행 후 결과 출력, **r**eport)
  - `-c <n>` : 사이클 수 (**c**ount)
  - `-n` : 이름 역해석 생략
  - `-T` / `-u` : TCP / UDP 사용

**동작 원리**: TTL 을 1부터 1씩 늘려 보내고, TTL 이 0 이 된 라우터가 돌려주는 **ICMP Time Exceeded (Type 11)** 로 홉을 식별. 마지막 대상은 UDP 방식이면 **ICMP Port Unreachable (Type 3 Code 3)**, ICMP 방식이면 Echo Reply 로 종료 판정.

**검증**

```bash
traceroute -n -m 5 192.168.64.1        # 1홉에서 끝나야 정상
traceroute -n -m 8 8.8.8.8 | head -5
mtr -rn -c 3 192.168.64.1
```

```text
# traceroute -n -m 5 192.168.64.1
traceroute to 192.168.64.1 (192.168.64.1), 5 hops max, 60 byte packets
 1  192.168.64.1  0.4... ms  0.3... ms  0.3... ms
# traceroute -n 8.8.8.8
 1  192.168.64.1  ... ms
 2  * * *
 3  ...
# mtr -rn -c 3 192.168.64.1
HOST: srv01                       Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- 192.168.64.1               0.0%     3    0.4   0.4   0.3   0.5   0.1
```

- `* * *` = 해당 홉이 ICMP Time Exceeded 를 반환하지 않음 → **그 라우터가 응답을 막을 뿐, 경로 단절이 아님** (뒤 홉이 정상이면 통과 중)
- 마지막 홉까지 계속 `* * *` 면 그 지점부터 실제 차단

> 📝 **시험 포인트**: `traceroute` 원리 = **TTL 을 1씩 증가시키며 ICMP Time Exceeded 응답으로 경로 파악**. 중간 홉의 `* * *` 는 "해당 라우터가 ICMP 응답 안 함" 이지 장애가 아님 — 출력 해석 문제로 자주 나옴. Windows 대응 명령은 `tracert`.

### 5-3. ss — 소켓·포트 상태 조회

> **상황**: 어떤 포트가 열려 있고 누가 붙어 있는지 확인한다. 9절에서 sshd 가 2222 로 옮겨졌는지 판정하는 것도 이 명령이다.

```bash
ss -tulnp                                   # TCP+UDP 리스닝 + 프로세스 (가장 많이 씀)
ss -tlnp                                    # TCP 리스닝만
ss -ulnp                                    # UDP 리스닝만
ss -tan                                     # TCP 전체 (상태 포함)
ss -tan state established                   # 수립된 연결만
ss -tan state listening
ss -tan state time-wait
ss -s                                       # 소켓 요약 통계
ss -o state established '( dport = :ssh or sport = :ssh )'   # 타이머 정보 포함
ss -lntp 'sport = :22'                      # 필터 문법 — 출발 포트 22
ss -tnp dst 192.168.64.1                    # 목적지 필터
ss -x                                       # 유닉스 도메인 소켓
ss -xl                                      # 유닉스 소켓 리스닝
ss -tp                                      # 프로세스 표시 (root 필요)
ss -i                                       # TCP 내부 정보(cwnd·rtt)
ss -m                                       # 소켓 메모리 사용량
ss -4 -tln                                  # IPv4 만
```

- `-t` : TCP (**t**cp)
- `-u` : UDP (**u**dp)
- `-l` : 리스닝 상태만 (**l**istening)
- `-n` : 포트를 숫자로 (**n**umeric) — 이름 변환 생략
- `-p` : 소켓 소유 프로세스 (**p**rocess) — **root 권한 필요**
- `-a` : 전체 소켓 (**a**ll) — 리스닝 + 연결
- `-s` : 요약 통계 (**s**ummary)
- `-o` : 타이머 정보 (**o**ptions) — 재전송·keepalive 남은 시간
- `-x` : 유닉스 도메인 소켓
- `-i` : TCP 내부 상태 정보 (**i**nternal)
- `-m` : 소켓 메모리 (**m**emory)
- `-4` / `-6` : 주소 계열 한정
- 필터 문법: `state <상태>`, `sport = :<포트>`, `dport = :<포트>`, `src <IP>`, `dst <IP>`, `and`/`or`/`not` 결합

**TCP 상태 표 (필수 암기 — 3·4-way handshake 와 연결)**

| 상태 | 시점 | 의미 |
| --- | --- | --- |
| `LISTEN` | 서버 대기 | 연결 요청 수신 대기 중 (`ss` 표시도 `LISTEN`) |
| `SYN-SENT` | 클라이언트 ① | SYN 보내고 응답 대기 |
| `SYN-RECV` | 서버 ② | SYN 받고 SYN+ACK 보낸 뒤 ACK 대기 — **급증 시 SYN Flooding 의심** |
| `ESTABLISHED` | 양측 ③ 이후 | 연결 성립·데이터 송수신 중 (`ss` 표시는 **`ESTAB`**) |
| `FIN-WAIT-1` | 능동 종료 측 ① | FIN 보내고 ACK 대기 |
| `FIN-WAIT-2` | 능동 종료 측 ② | ACK 받고 상대 FIN 대기 |
| `CLOSE-WAIT` | **수동 종료 측** | 상대 FIN 받고 ACK 보냄 — 자기 FIN 미전송. **누적되면 애플리케이션이 소켓을 안 닫는 버그** |
| `LAST-ACK` | 수동 종료 측 | FIN 보내고 마지막 ACK 대기 |
| `TIME-WAIT` | **능동 종료 측** | 마지막 ACK 후 2×MSL 대기 — 지연 패킷 처리용. 다수 누적은 정상 동작 |
| `CLOSED` | — | 소켓 없음 |

**검증**

```bash
ss -tlnp | grep -E 'sshd|:22'
ss -tan state established | head -5
ss -s
ss -lntp 'sport = :22'
```

```text
# ss -tulnp
Netid State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
tcp   LISTEN 0      128          0.0.0.0:22          0.0.0.0:*     users:(("sshd",pid=...,fd=...))
tcp   LISTEN 0      128             [::]:22             [::]:*     users:(("sshd",pid=...,fd=...))
# ss -s
Total: ...
TCP:   ... (estab ..., closed ..., orphaned ..., timewait ...)
Transport Total     IP        IPv6
TCP       ...       ...       ...
UDP       ...       ...       ...
```

- `0.0.0.0:22` = **모든 인터페이스**에서 수신 (정상)
- `127.0.0.1:22` = 로컬 전용 수신 → **외부 접속 불가** (sshd_config 의 `ListenAddress` 확인)
- `Recv-Q`/`Send-Q` : 리스닝 소켓에서는 각각 **현재 대기 큐 길이 / 백로그 최대치**
- ⚠️ 리스너 없음(무출력) ≠ 방화벽 차단 → 외부 관점 검증은 5-5 의 `nc` 필요

> 📝 **시험 포인트**: `ss -tulnp` 5개 옵션 조합이 실기 빈칸 문제로 반복 출제 (t=TCP, u=UDP, l=listening, n=numeric, p=process). `ss -tan state established` / `ss -tan state syn-recv | wc -l`(SYN Flooding 판정) 도 기출. `CLOSE_WAIT`= 수동 종료 측, `TIME_WAIT`= 능동 종료 측 구분이 함정.

### 5-4. netstat 대응 (net-tools)

> **상황**: 시험 지문은 여전히 `netstat` 을 쓴다. 1-9 에서 설치한 net-tools 로 `ss` 와 같은 정보를 뽑아 옵션 대응을 확정한다.

```bash
netstat -tulnp                # ss -tulnp 대응
netstat -an                   # ss -an 대응 (전체 소켓)
netstat -tan                  # TCP 전체
netstat -rn                   # ip route 대응
netstat -i                    # ip -s link 대응 (인터페이스 통계)
netstat -s                    # 프로토콜별 누적 통계
netstat -s | grep -A5 '^Tcp:'
netstat -c 1 -tn              # 1초마다 반복 (continuous, Ctrl+C 종료)
netstat -ap | grep -i ssh     # 프로세스명으로 찾기
```

- `-t` `-u` `-l` `-n` `-p` `-a` : `ss` 와 동일 의미
- `-r` : 라우팅 테이블 (**r**oute)
- `-i` : 인터페이스 통계 (**i**nterface)
- `-s` : 프로토콜별 통계 (**s**tatistics)
- `-c` : 반복 출력 (**c**ontinuous)
- `-g` : 멀티캐스트 그룹 (**g**roup)

**옵션 대응표**

| 목적 | netstat | ss |
| --- | --- | --- |
| TCP/UDP 리스닝 + 프로세스 | `netstat -tulnp` | `ss -tulnp` |
| 전체 소켓 숫자 표기 | `netstat -an` | `ss -an` |
| 수립된 연결만 | `netstat -tan \| grep ESTABLISHED` | `ss -tan state established` |
| 라우팅 테이블 | `netstat -rn` | `ip route` |
| 인터페이스 통계 | `netstat -i` | `ip -s link` |
| 프로토콜 통계 | `netstat -s` | `ss -s`, `nstat` |
| 유닉스 소켓 | `netstat -x` | `ss -x` |

**검증**

```bash
netstat -tlnp 2>/dev/null | grep ':22'
ss     -tlnp 2>/dev/null | grep ':22'
netstat -i | head -3
```

```text
# netstat -tulnp | grep ':22'
tcp        0      0 0.0.0.0:22    0.0.0.0:*    LISTEN      .../sshd: ...
tcp6       0      0 :::22         :::*         LISTEN      .../sshd: ...
# netstat -i
Kernel Interface table
Iface   MTU    RX-OK RX-ERR RX-DRP RX-OVR    TX-OK TX-ERR TX-DRP TX-OVR Flg
enp0s1  1500   ...       0      0      0     ...        0      0      0 BMRU
lo     65536   ...       0      0      0     ...        0      0      0 LRU
```

- `netstat` 은 상태를 `ESTABLISHED`·`TIME_WAIT` 처럼 **밑줄(`_`)** 로, `ss` 는 `ESTAB`·`TIME-WAIT` 처럼 **하이픈(`-`)** 으로 표기 — grep 할 때 걸리는 함정
- `netstat -i` 의 Flg: `B`=Broadcast, `M`=Multicast, `R`=Running, `U`=Up, `L`=Loopback

> 📝 **시험 포인트**: `netstat -r`(라우팅 테이블 출력) 옵션 단독 출제 이력. `netstat -tulnp` ↔ `ss -tulnp` 대응은 그대로 외울 것. RHEL 9 에서 `netstat` 은 `net-tools` 설치 후에만 사용 가능하다는 점도 최신 회차 소재.

### 5-5. nc — 포트 점검·양방향 통신·파일 전송

> **상황**: 포트가 정말 열려 있는지 "밖에서" 두드려 보고, 임의 포트로 실제 데이터가 오가는지 확인한다. 6절의 handshake 캡처도 이 도구로 트래픽을 만든다.

```bash
rpm -q nmap-ncat || dnf install -y nmap-ncat     # RHEL9 의 nc 는 Ncat

nc -zv 127.0.0.1 22                     # 로컬 SSH 포트 점검
nc -zv -w 3 192.168.64.1 53             # 게이트웨이 TCP 53 점검
nc -zvu -w 3 192.168.64.1 53            # UDP 53 점검
nc -zv -w 2 127.0.0.1 20-25             # 포트 범위 스캔
nc -zv -w 2 192.168.64.10 22 80 443     # 여러 포트 동시
```

- `-z` : 데이터 전송 없이 연결만 시도 (**z**ero-I/O) — 포트 스캔
- `-v` : 상세 출력 (**v**erbose), `-vv` 로 증가
- `-w <초>` : 연결 타임아웃 (**w**ait) — 미지정 시 무응답 포트에서 장시간 대기
- `-u` : UDP 사용 (**u**dp, 기본은 TCP)
- `-l` : 리스닝 모드 (**l**isten)
- `-k` : 리스너를 연결 종료 후에도 유지 (**k**eep open)
- `-p <포트>` : 출발지 포트 지정
- `-n` : 이름 역해석 생략

**응답별 판정**

| 응답 | 판정 | 의미 |
| --- | --- | --- |
| `Connected to ...` / `succeeded!` | **개방 + 서비스 응답 중** | 정상 리스닝 |
| `Connection refused` | 포트 도달은 되나 **리스너 없음** | 방화벽 통과, 서비스 정지 상태 |
| 타임아웃(무응답) | **방화벽·보안그룹 차단** | 패킷이 DROP 됨 |

- 이 3분류가 "방화벽 차단인가 서비스 정지인가" 를 가르는 결정적 근거 — 서버 내부 `ss` 만으로는 판정 불가

**양방향 통신 검증** — 셸 2개 필요 (`tmux`·SSH 2세션·콘솔+SSH)

```bash
# [셸 A] 수신 대기
nc -l 9000

# [셸 B] 접속 후 문자열 입력
nc 127.0.0.1 9000
hello from B          ← 입력하면 셸 A 에 그대로 출력됨
```

```bash
# 확인 중 [셸 C] 에서 소켓 상태 관찰
ss -tanp | grep 9000
```

**UDP 통신 비교**

```bash
# [셸 A]
nc -u -l 9000
# [셸 B]
nc -u 127.0.0.1 9000
```

- UDP 는 handshake 없이 **첫 데이터그램이 바로 나감** → `ss -uan` 에는 연결 상태 개념이 없음(`UNCONN`)

**파일 전송**

```bash
# [셸 A] 수신 측
nc -l 9001 > /tmp/received.txt

# [셸 B] 송신 측
echo "lab08 file transfer test" > /tmp/send.txt
nc -w 2 127.0.0.1 9001 < /tmp/send.txt

# 검증
diff /tmp/send.txt /tmp/received.txt && echo "전송 무결성 OK"
sha256sum /tmp/send.txt /tmp/received.txt
```

**검증**

```bash
nc -zv -w 2 127.0.0.1 22
ss -tlnp | grep 9000        # 리스너 동작 중일 때
diff /tmp/send.txt /tmp/received.txt && echo OK
```

```text
# nc -zv 127.0.0.1 22
Ncat: Version 7.9... ( https://nmap.org/ncat )
Ncat: Connected to 127.0.0.1:22.
Ncat: 0 bytes sent, 0 bytes received in ... seconds.
# nc -zv -w 2 127.0.0.1 9999      (리스너 없는 포트)
Ncat: Connection refused.
# diff /tmp/send.txt /tmp/received.txt && echo "전송 무결성 OK"
전송 무결성 OK
```

- ※ 게이트웨이(192.168.64.1)의 TCP 53 은 UTM 가상 네트워크 구성에 따라 응답하지 않을 수 있음 → `-u` 로 UDP 53 을 확인하거나 `dig @192.168.64.1` 로 실제 DNS 응답 여부 판정
- RHEL 9 의 `nc` 는 **Ncat**(nmap-ncat) — 전통 netcat 과 옵션이 대부분 호환되나 출력 문구가 `Ncat:` 접두어로 나옴

> 📝 **시험 포인트**: `nc -zv <호스트> <포트>` 는 실기 빈칸 최빈출 — `-z`(연결만) `-v`(상세). 방화벽 진단에서 **타임아웃=차단 / refused=서비스 정지** 구분이 서술형 정답 요소.

### 5-6. telnet — 배너 확인

> **상황**: 암호화 없는 원격 접속 프로토콜이지만, **임의 TCP 포트에 붙어 응답 배너를 읽는 진단 도구**로는 여전히 쓰인다. SSH 서버 버전 확인에 써 본다.

```bash
rpm -q telnet || dnf install -y telnet          # 클라이언트만 설치 (서버는 telnet-server)
telnet 127.0.0.1 22                             # SSH 배너 확인 → Ctrl+] 후 quit
telnet 192.168.64.10 22
telnet 127.0.0.1 9999                           # 닫힌 포트 → Connection refused
```

- `telnet <호스트> <포트>` : 포트 생략 시 기본 23
- 종료: `Ctrl+]` → `telnet>` 프롬프트에서 `quit`
- 배너에서 얻는 정보: SSH 프로토콜 버전·OpenSSH 버전 → **버전 노출은 보안 취약점**

**검증**

```bash
timeout 3 telnet 127.0.0.1 22 2>&1 | head -4
nc -w 2 127.0.0.1 22 </dev/null | head -1        # nc 로도 동일 배너 확인
```

```text
# telnet 127.0.0.1 22
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
SSH-2.0-OpenSSH_...
# telnet 127.0.0.1 9999
Trying 127.0.0.1...
telnet: connect to address 127.0.0.1: Connection refused
```

- ⚠️ **telnet 프로토콜(23번)은 ID·비밀번호를 평문 전송** → 원격 접속 용도로는 사용 금지, SSH 로 대체
- 마찬가지로 FTP(21) → SFTP/FTPS, HTTP(80) → HTTPS(443) 로 대체하는 것이 원칙

> 📝 **시험 포인트**: telnet(23) vs SSH(22) 비교는 매 회차 출제 — **평문 vs 암호화**. telnet 을 포트 점검·배너 확인 용도로 쓰는 것과 원격 접속 프로토콜로 쓰는 것을 구분. 대체 파일 전송 수단은 `scp`·`sftp`.

### 5-7. curl 과 wget

> **상황**: 응용 계층(HTTP) 도달성을 확인한다. Part 09 에서 Apache 를 올린 뒤 이 명령들로 검증하게 되므로 옵션을 미리 훑는다.

```bash
curl -I https://rockylinux.org                       # 헤더만 (HEAD 요청)
curl -v https://rockylinux.org -o /dev/null          # 상세 통신 과정
curl -s -o /tmp/rocky.html https://rockylinux.org    # 파일로 저장 (조용히)
curl -s -O https://rockylinux.org/index.html         # 원격 파일명 그대로 저장
curl -sL -o /dev/null -w '%{http_code} %{time_total}\n' https://rockylinux.org
curl -k https://192.168.64.10                        # 인증서 검증 생략 (자체서명)
curl -u admin1:'<비밀번호>' http://192.168.64.10/secret/    # 기본 인증
curl -X POST -H 'Content-Type: application/json' -d '{"k":"v"}' http://127.0.0.1:8080/api
curl --resolve intranet.lab.local:80:192.168.64.10 http://intranet.lab.local/
```

- `-I` : HEAD 요청 — 헤더만 받음 (대문자 I, **I**nclude headers only)
- `-v` : 통신 과정 상세 (**v**erbose) — `*` 연결 정보, `>` 요청, `<` 응답
- `-o <파일>` : 응답 본문을 지정 파일로 저장 (소문자 **o**utput)
- `-O` : **원격 파일명 그대로** 저장 (대문자 **O**)
- `-L` : 리다이렉트(3xx) 추적 (**L**ocation)
- `-k` : TLS 인증서 검증 생략 (**k**, `--insecure`) — ⚠️ 중간자 공격 탐지 불가, 자체서명 테스트 한정
- `-u <user>:<pass>` : HTTP 기본 인증 (**u**ser) — ⚠️ 비밀번호가 히스토리·프로세스 목록에 노출
- `-d <데이터>` : 요청 본문 (**d**ata) — 지정 시 메서드가 POST 로 자동 전환
- `-H <헤더>` : 요청 헤더 추가 (**H**eader)
- `-X <메서드>` : HTTP 메서드 명시 (PUT·PATCH·DELETE 는 필수)
- `-s` : 진행률 숨김 (**s**ilent)
- `-w <포맷>` : 완료 후 정보 출력 (**w**rite-out) — `%{http_code}`, `%{time_total}`
- `--resolve <호스트>:<포트>:<IP>` : DNS 를 거치지 않고 **강제 매핑** — 가상호스트 검증에 필수

```bash
rpm -q wget || dnf install -y wget
wget https://rockylinux.org -O /tmp/rocky2.html      # 파일명 지정 저장
wget -q https://rockylinux.org -O /dev/null          # 조용히
wget -c <URL>                                        # 중단 지점부터 이어받기
wget --spider -q <URL> && echo "URL 살아 있음"        # 다운로드 없이 존재만 확인
wget -r -l 1 -np <URL>                               # 재귀 수집 (참고 · 서버 부하 주의)
```

- `-O <파일>` : 저장 파일명 지정 (대문자 O)
- `-c` : 이어받기 (**c**ontinue) — 중단된 대용량 파일 재개
- `-q` : 조용히 (**q**uiet)
- `--spider` : 존재 여부만 확인 (다운로드 안 함)
- `-r` / `-l <깊이>` / `-np` : 재귀 / 깊이 제한 / 상위 디렉터리 미추적

**curl vs wget 비교**

| 항목 | curl | wget |
| --- | --- | --- |
| 기본 동작 | **표준 출력**으로 내보냄 | **파일로 저장** |
| 이어받기 | `-C -` | `-c` |
| 재귀 수집 | 불가 | `-r` 지원 |
| 프로토콜 | HTTP/FTP/SFTP/SMTP 등 다수 | HTTP/HTTPS/FTP |
| 용도 | API 호출·헤더 검증 | 파일 다운로드·미러링 |

**검증**

```bash
curl -sI https://rockylinux.org | head -1
curl -s -o /dev/null -w '%{http_code}\n' https://rockylinux.org
wget --spider -q https://rockylinux.org && echo "spider OK"
```

```text
# curl -I https://rockylinux.org
HTTP/2 200
content-type: text/html; charset=utf-8
...
# curl -s -o /dev/null -w '%{http_code}\n' https://rockylinux.org
200
```

- ⚠️ `curl` 은 **HTTP 4xx·5xx 도 종료 코드 0** → 성공 오판 주의. `-f`(fail) 또는 `-w '%{http_code}'` 로 명시 판정
- 응답 코드: `200` OK / `301` 영구 이동 / `302` 임시 이동 / `401` 인증 필요 / `403` 금지 / `404` 없음 / `500` 서버 오류 / `503` 서비스 불가

> 📝 **시험 포인트**: `curl -I` 출력의 `301 Moved Permanently` + `Location:` 헤더 해석 = "영구 리다이렉트, `-L` 로 따라가야 함". `-o`(파일명 지정) vs `-O`(원격 파일명 유지) 대소문자 구분 함정.

### 5-8. nmap — 포트 스캔

> **상황**: 서버 자신에게 스캔을 걸어 열린 포트 목록을 외부 관점으로 확인한다. 9절 포트 변경 후 22 가 닫히고 2222 가 열렸는지 판정하는 데도 쓴다.

⚠️ **자신이 관리하는 호스트에만** 사용할 것. 타인의 시스템 스캔은 법적 문제가 될 수 있다.

```bash
rpm -q nmap || dnf install -y nmap

nmap -sT -p 1-1024,2222 127.0.0.1        # TCP Connect 스캔 (일반 사용자 가능)
nmap -sS -p 1-1024 127.0.0.1             # SYN(하프오픈) 스캔 — root 필요
nmap -sU -p 53,67,123 127.0.0.1          # UDP 스캔 — root 필요, 느림
nmap -sV -p 22,2222 127.0.0.1            # 서비스·버전 탐지
nmap -sn 192.168.64.0/24                 # 호스트 발견만 (핑 스윕, 포트 스캔 안 함)
nmap -O 127.0.0.1                        # OS 추정 — root 필요
nmap -A -p 22 127.0.0.1                  # 종합 (버전+OS+스크립트+traceroute)
nmap -p- 127.0.0.1                       # 전체 65535 포트 (오래 걸림)
nmap -Pn -p 22 192.168.64.10             # ping 생략하고 바로 포트 스캔
```

- `-sT` : **TCP Connect 스캔** — 3-way handshake 완결. 권한 불필요, 대상 로그에 남음
- `-sS` : **SYN(하프오픈) 스캔** — SYN 만 보내고 SYN+ACK 받으면 RST 로 끊음. **연결을 완성하지 않아 로그에 덜 남음**, root 필요
- `-sU` : UDP 스캔 — 응답이 없으면 open|filtered 로 판정, 매우 느림
- `-sV` : 서비스 버전 탐지 (**V**ersion) — 배너·프로브로 판별
- `-sn` : 포트 스캔 없이 **호스트 존재만** 확인 (구 `-sP`, ping sweep)
- `-O` : OS 지문 추정 (**O**S detection)
- `-A` : 공격적 종합 스캔 (**A**ggressive)
- `-p <포트>` : 대상 포트 (`80`, `1-1024`, `22,80,443`, `-`=전체)
- `-Pn` : 호스트 발견 단계 생략 (ICMP 차단 대상용)
- `-n` : DNS 역해석 생략

**포트 상태 판정**

| 상태 | 의미 |
| --- | --- |
| `open` | 리스닝 중 — 연결 수락 |
| `closed` | 도달은 되나 리스너 없음 (RST 응답) |
| `filtered` | 응답 없음 → **방화벽이 패킷을 폐기** |
| `open\|filtered` | 구분 불가 (UDP 스캔에서 흔함) |
| `unfiltered` | 도달 가능하나 open/closed 판별 불가 (ACK 스캔) |

**검증**

```bash
nmap -sT -p 22,2222,80 127.0.0.1 | grep -E '^[0-9]+/tcp'
nmap -sn 192.168.64.0/24 | grep -c 'Nmap scan report'
nmap -sV -p 22 127.0.0.1 | grep -E '^22/tcp'
```

```text
# nmap -sT -p 1-1024,2222 127.0.0.1
Starting Nmap ... ( https://nmap.org )
Nmap scan report for localhost (127.0.0.1)
Host is up (0.0000... latency).
Not shown: ... closed tcp ports (conn-refused)
PORT     STATE SERVICE
22/tcp   open  ssh
# nmap -sn 192.168.64.0/24
Nmap scan report for 192.168.64.1
Host is up ...
Nmap scan report for 192.168.64.10
Host is up ...
Nmap done: 256 IP addresses (... hosts up) scanned in ... seconds
```

> 📝 **시험 포인트**: `-sS` = **SYN 스캔 / 하프오픈 / 스텔스 스캔** — 3-way handshake 를 완성하지 않아 로그 회피. `-sT` = Connect 스캔(완성). `-sn` = 호스트 발견(핑 스윕). `filtered` = 방화벽 차단 판정 — 출력 해석 문제로 자주 나옴.

### 5-9. tcpdump — 패킷 캡처

> **상황**: 실제 패킷을 눈으로 본다. 6절의 3-way handshake 캡처가 이 도구로 이뤄지므로 여기서 옵션과 필터 문법을 정리한다.

```bash
rpm -q tcpdump || dnf install -y tcpdump         # root 권한 필요

tcpdump -i enp0s1 -nn -c 10                      # 10개만 캡처
tcpdump -D                                       # 캡처 가능한 인터페이스 목록
tcpdump -i any -nn -c 5                          # 모든 인터페이스
tcpdump -i enp0s1 -nn port 22 -c 5               # 포트 필터
tcpdump -i enp0s1 -nn 'host 192.168.64.1' -c 5   # 호스트 필터
tcpdump -i enp0s1 -nn 'host 192.168.64.1 and not port 22' -c 10
tcpdump -i enp0s1 -nn 'tcp port 80 or tcp port 443' -c 5
tcpdump -i enp0s1 -nn icmp -c 4                  # ICMP 만 (다른 셸에서 ping)
tcpdump -i enp0s1 -nn -A -c 3 port 80            # 페이로드 ASCII 표시
tcpdump -i enp0s1 -nn -X -c 3 port 80            # 16진수 + ASCII
tcpdump -i enp0s1 -nn -e -c 3                    # 이더넷 헤더(MAC) 포함
tcpdump -i enp0s1 -w /tmp/cap.pcap -c 20         # 파일로 저장 (바이너리)
tcpdump -r /tmp/cap.pcap -nn | head              # 저장 파일 읽기
tcpdump -r /tmp/cap.pcap -nn 'port 22' | head    # 저장 파일에 필터 적용
```

- `-i <인터페이스>` : 캡처 대상 (**i**nterface) — `any` 는 전체
- `-D` : 인터페이스 목록 (**D**evices)
- `-n` : IP 를 이름으로 변환 안 함, `-nn` : **포트까지** 숫자로
- `-c <n>` : n 개 캡처 후 종료 (**c**ount)
- `-w <파일>` : pcap 형식으로 저장 (**w**rite) — Wireshark 로 열람 가능
- `-r <파일>` : 저장 파일 읽기 (**r**ead)
- `-A` : 페이로드를 ASCII 로 (**A**SCII)
- `-X` : 16진수 + ASCII 동시 (`-XX` 는 링크 헤더 포함)
- `-e` : 링크 계층 헤더(출발·목적 MAC) 표시 (**e**thernet)
- `-s <바이트>` : 캡처 길이 (**s**naplen), `0` = 전체
- `-v` / `-vv` / `-vvv` : 상세도
- `-q` : 간략 출력 (**q**uiet)
- `-t` / `-tttt` : 타임스탬프 생략 / 사람이 읽는 형식
- 필터(BPF) 문법: `host <IP>` `net <대역>` `port <번호>` `portrange 1-1024` `src`/`dst` 한정, `tcp`/`udp`/`icmp`/`arp`, `and` `or` `not` 결합 — 셸 특수문자 때문에 **작은따옴표로 감싸는 것이 안전**

**검증**

```bash
# [셸 A] 캡처 시작
tcpdump -i enp0s1 -nn icmp -c 4
# [셸 B] 트래픽 생성
ping -c 2 192.168.64.1
# 파일 저장·재생 확인
tcpdump -i lo -nn -w /tmp/cap.pcap -c 6 &
ping -c 3 127.0.0.1 >/dev/null; wait
ls -l /tmp/cap.pcap && tcpdump -r /tmp/cap.pcap -nn | wc -l
```

```text
# tcpdump -i enp0s1 -nn icmp -c 4
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
... IP 192.168.64.10 > 192.168.64.1: ICMP echo request, id ..., seq 1, length 64
... IP 192.168.64.1 > 192.168.64.10: ICMP echo reply,   id ..., seq 1, length 64
4 packets captured
```

- 출력 형식: `<시각> IP <출발IP>.<포트> > <목적IP>.<포트>: Flags [..], seq ..., ack ..., win ..., length ...`
- ⚠️ 캡처 파일에는 **평문 자격증명이 그대로 담길 수 있음** → `/tmp` 방치 금지, 실습 후 삭제

> 📝 **시험 포인트**: `tcpdump -i eth0 tcp port 80 -w web.pcap` 해석 = "eth0 에서 TCP 80 트래픽을 web.pcap 에 저장". `-nn`(이름·포트 모두 숫자), `-w`(저장) ↔ `-r`(읽기) 대응이 출력 해석 문제로 나옴.

### 5-10. arping · whois · 인터페이스 오류 카운터

> **상황**: ICMP 가 막힌 상대의 존재를 2계층에서 확인하고(arping), 외부 IP 의 소속을 조회하며(whois), 인터페이스 오류를 다시 본다.

```bash
arping -I enp0s1 -c 3 192.168.64.1           # ARP 요청으로 존재 확인 (root)
arping -I enp0s1 -c 2 -D 192.168.64.10       # DAD — 주소 중복 검사 모드
ip neigh show 192.168.64.1                   # 결과가 ARP 캐시에 반영됨

rpm -q whois || dnf install -y whois
whois 8.8.8.8 | grep -iE 'orgname|netname|country|abuse' | head
whois rockylinux.org | grep -iE 'registrar|creation|expiry' | head
dig +short -x 8.8.8.8                        # 역방향 DNS 로 보완

ip -s link show enp0s1                       # 오류·드롭 카운터 재확인
ip -s -s link show enp0s1                    # 세부 항목까지
```

- `arping` 옵션 (`iputils` 제공)
  - `-I <인터페이스>` : 송신 인터페이스 (**I**nterface) — **필수**
  - `-c <n>` : 횟수 (**c**ount)
  - `-D` : 중복 주소 감지 모드 (**D**uplicate address detection) — 출발지 0.0.0.0 으로 질의
  - `-f` : 첫 응답에서 종료 (**f**inish)
- `whois` 옵션
  - `-h <서버>` : 조회 서버 지정 (**h**ost) — 예: `-h whois.radb.net`
  - 핵심 필드: `OrgName`(소속 조직), `NetName`(할당 이름), `Country`, `abuse-mailbox`(신고 창구)

**검증**

```bash
arping -I enp0s1 -c 2 192.168.64.1 | tail -2
whois 8.8.8.8 | grep -i orgname | head -1
ip -s link show enp0s1 | awk '/RX:|TX:/{getline; print}'
```

```text
# arping -I enp0s1 -c 2 192.168.64.1
ARPING 192.168.64.1 from 192.168.64.10 enp0s1
Unicast reply from 192.168.64.1 [<GW-MAC>]  0.6...ms
Sent 2 probes (1 broadcast(s))
Received 2 response(s)
```

- `arping` 은 **2계층 ARP** 로 묻기 때문에 ICMP 를 차단한 장비도 응답 → `ping` 무응답 시 존재 확정 판정 수단
- 다만 **같은 브로드캐스트 도메인(같은 서브넷)** 안에서만 동작 — 라우터 너머에는 쓸 수 없음
- `whois` 는 침해 대응에서 공격 출발 IP 의 사업자·abuse 창구 확인에 사용 → [[10-security-firewall-selinux]]

> 📝 **시험 포인트**: ARP 는 **같은 네트워크 안에서만** 동작(브로드캐스트 기반). ARP 스푸핑은 위조 ARP 응답으로 MAC 테이블을 오염시켜 트래픽을 가로채는 중간자(MITM) 공격 — 대응은 정적 ARP 등록(`ip neigh add ... nud permanent`)·동적 ARP 검사(DAI)·암호화 통신.

### 5-11. 실시간 대역폭 모니터 (참고)

> **상황**: 트래픽 폭주 진단용 도구는 EPEL 저장소에 있다. 설치 방법과 용도만 정리한다.

```bash
rpm -q epel-release || dnf install -y epel-release      # Part 02 에서 이미 등록됨
dnf install -y iftop nload                              # ※ 선택 설치
iftop -i enp0s1 -n                                      # 연결별 실시간 대역폭 (Ctrl+C 종료)
nload enp0s1                                            # 인터페이스 인/아웃 그래프
```

- `iftop` : 연결(쌍) 단위 대역폭 — **누가 대역폭을 쓰는지** 식별
  - `-i <if>` : 인터페이스, `-n` : 이름 역해석 생략, `-P` : 포트 표시
- `nload` : 인터페이스 단위 인/아웃 실시간 그래프 — 총량 추이 확인
- 대체 수단(추가 설치 없음)
  - `ip -s link` 반복 실행으로 카운터 증가분 비교
  - `sar -n DEV 1 5` (`sysstat`, Part 06 에서 설치) — 인터페이스별 초당 패킷·바이트

```bash
sar -n DEV 1 3 | grep -E 'IFACE|enp0s1'        # 설치 없이 대역폭 측정 (Part 06 자원)
```

**검증**

```bash
sar -n DEV 1 2 | grep enp0s1 | tail -2
```

> 📝 **시험 포인트**: 네트워크 사용량 통계는 `sar -n DEV`(sysstat) — Part 06 의 `vmstat`·`iostat` 와 한 세트로 출제. `iftop`·`nload` 는 EPEL 필요.

---

## 6. TCP/IP 이론을 명령으로 검증

### 6-1. 3-way handshake 실제 캡처

> **상황**: 필기에서 "SYN → SYN+ACK → ACK" 를 외우기만 했다면, 여기서 실제 패킷으로 확인한다. 루프백에서 `nc` 로 연결을 만들고 `tcpdump` 로 잡는다. 셸 3개(또는 SSH 2 + 콘솔)를 준비한다.

```bash
# [셸 A] 캡처 시작 — 연결 수립 3패킷만
tcpdump -i lo -nn 'tcp port 9000' -c 3

# [셸 B] 서버 역할 리스너
nc -l 9000

# [셸 C] 클라이언트 접속 (연결만 만들고 대기)
nc 127.0.0.1 9000
```

- `-i lo` : 루프백 인터페이스 — 자기 자신끼리의 통신은 여기로 흐름
- `'tcp port 9000'` : BPF 필터 — 9000 번을 출발·목적 어느 쪽으로든 쓰는 TCP
- `-c 3` : 3패킷 = 정확히 handshake 만

**tcpdump 플래그 표기 해석**

| 표기 | TCP 플래그 | 의미 |
| --- | --- | --- |
| `[S]` | SYN | 연결 요청 (① 클라이언트 → 서버) |
| `[S.]` | SYN + ACK | 요청 수락 + 확인 (② 서버 → 클라이언트) |
| `[.]` | ACK 만 | 확인 응답 (③ 클라이언트 → 서버) → **ESTABLISHED** |
| `[P.]` | PSH + ACK | 실제 데이터 전송 |
| `[F.]` | FIN + ACK | 종료 요청 |
| `[R]` / `[R.]` | RST / RST+ACK | 강제 초기화 (닫힌 포트 접속 시) |

- `.` 하나가 **ACK 비트**를 뜻함 — `[S.]` = SYN+ACK 로 읽는 것이 핵심

**검증**

```bash
# [셸 D] 연결이 살아 있는 동안 상태 확인
ss -tanp | grep 9000
```

```text
# [셸 A] tcpdump -i lo -nn 'tcp port 9000' -c 3
... IP 127.0.0.1.<임시포트> > 127.0.0.1.9000: Flags [S],  seq ..., win ..., options [...], length 0
... IP 127.0.0.1.9000 > 127.0.0.1.<임시포트>: Flags [S.], seq ..., ack ..., win ..., options [...], length 0
... IP 127.0.0.1.<임시포트> > 127.0.0.1.9000: Flags [.],  ack ..., win ..., length 0
3 packets captured
# [셸 D] ss -tanp | grep 9000
LISTEN 0  ... 0.0.0.0:9000  0.0.0.0:*  users:(("nc",pid=...,fd=...))
ESTAB  0  ... 127.0.0.1:9000       127.0.0.1:<임시포트>  users:(("nc",...))
ESTAB  0  ... 127.0.0.1:<임시포트> 127.0.0.1:9000       users:(("nc",...))
```

- `ESTAB` 항목이 **2개** 보이는 이유: 같은 호스트 안에 클라이언트 소켓과 서버 소켓이 모두 있기 때문
- 클라이언트의 출발 포트는 **동적 포트 대역(49152~65535)** 에서 커널이 자동 할당
- `seq` 는 임의의 초기 순서번호(ISN), `ack` = 상대 `seq` + 1

> 📝 **시험 포인트**: 3-way handshake 순서 나열 = **SYN → SYN+ACK → ACK**. 각 단계 후 상태는 `SYN_SENT`(클라이언트) → `SYN_RECV`(서버) → `ESTABLISHED`(양쪽). SYN Flooding 은 ③ ACK 를 보내지 않아 서버를 `SYN_RECV` 로 묶어 백로그 큐를 고갈시키는 공격 — 대응은 SYN 쿠키(`net.ipv4.tcp_syncookies=1`)·백로그 확대·방화벽 rate limit.

### 6-2. 4-way 종료와 UDP 비교

> **상황**: 연결 종료가 왜 4단계인지, UDP 는 왜 이런 절차가 없는지 같은 방법으로 확인한다.

```bash
# [셸 A] 종료 패킷 캡처 (연결이 살아 있는 상태에서 시작)
tcpdump -i lo -nn 'tcp port 9000' -c 6

# [셸 C] 클라이언트에서 Ctrl+D (EOF) → 정상 종료 시작
```

```text
# [셸 A] 종료 시 캡처
... 127.0.0.1.<임시포트> > 127.0.0.1.9000: Flags [F.], seq ..., ack ..., length 0   ① FIN
... 127.0.0.1.9000 > 127.0.0.1.<임시포트>: Flags [.],  ack ...                       ② ACK
... 127.0.0.1.9000 > 127.0.0.1.<임시포트>: Flags [F.], seq ..., ack ..., length 0   ③ FIN
... 127.0.0.1.<임시포트> > 127.0.0.1.9000: Flags [.],  ack ...                       ④ ACK
```

- ※ 구현·타이밍에 따라 ②③ 이 하나의 `[F.]` 로 합쳐져 **3패킷**으로 보일 수 있음 (piggyback) — 개념상은 4단계
- 종료 직후 능동 종료 측은 `TIME-WAIT` 상태로 2×MSL 대기 → `ss -tan state time-wait` 로 확인

```bash
ss -tan state time-wait | head          # 종료 직후 TIME-WAIT 확인
ss -tan state close-wait | head         # 수동 종료 측 미종료 소켓 (누적 시 버그)
```

**UDP 비교 캡처**

```bash
# [셸 A]
tcpdump -i lo -nn 'udp port 9000' -c 4
# [셸 B]
nc -u -l 9000
# [셸 C]
nc -u 127.0.0.1 9000
hello udp        ← 입력
```

```text
# tcpdump -i lo -nn 'udp port 9000' -c 4
... IP 127.0.0.1.<임시포트> > 127.0.0.1.9000: UDP, length 10
... IP 127.0.0.1.9000 > 127.0.0.1.<임시포트>: UDP, length ...
```

- **handshake 패킷이 전혀 없음** — 첫 데이터그램이 곧바로 데이터
- `ss -uan` 에서 UDP 소켓 상태는 `UNCONN` — 연결 상태 개념 자체가 없음

**TCP vs UDP 비교표**

| 구분 | TCP | UDP |
| --- | --- | --- |
| 연결 방식 | 연결지향(connection-oriented) | 비연결(connectionless) |
| 연결 수립 | **3-way handshake** | 없음 |
| 연결 종료 | **4-way handshake** | 없음 |
| 신뢰성 | 재전송·순서 보장·흐름 제어·혼잡 제어 | 보장 없음 |
| 헤더 크기 | **20바이트**(옵션 시 최대 60) | **8바이트**(고정) |
| 헤더 고유 필드 | seq, ack, 플래그, 윈도, 긴급 포인터 | 없음 (출발·목적 포트, 길이, 체크섬 4개뿐) |
| PDU 명칭 | 세그먼트(Segment) | 데이터그램(Datagram) |
| 속도 | 느림 | 빠름 |
| 대표 서비스 | HTTP, FTP, SMTP, SSH, Telnet | DNS, DHCP, TFTP, SNMP, NTP, 스트리밍 |

- DNS 는 질의 **UDP 53**, 존 전송(AXFR)은 **TCP 53** → 양쪽 모두 사용하는 대표 예

**검증**

```bash
ss -tan state time-wait | wc -l
ss -uan | head -3
grep -w '^domain' /etc/services         # 53/tcp, 53/udp 둘 다 등록됨 확인
```

> 📝 **시험 포인트**: 4-way 종료 순서 = **FIN → ACK → FIN → ACK**, 종료 후 능동 측은 `TIME_WAIT`. "TCP 헤더에는 있고 UDP 헤더에는 없는 필드" 문제의 정답 후보 = **순서번호(Sequence)·확인응답번호(ACK)·윈도 크기·제어 플래그** (체크섬·포트는 양쪽 다 있음 → 함정).

### 6-3. OSI 7계층 ↔ TCP/IP 4계층 ↔ 캡슐화

> **상황**: 지금까지 만진 도구들이 각각 몇 계층을 보는 것인지 정리한다. 이 표가 계층 문제의 정답표가 된다.

| OSI 7계층 | TCP/IP 4계층 | PDU | 주소 | 대표 프로토콜 | 대표 장비 | 이 파트의 진단 도구 |
| --- | --- | --- | --- | --- | --- | --- |
| 7 응용(Application) | 응용(Application) | 데이터(Data) | — | HTTP, FTP, SMTP, DNS, Telnet, SSH, DHCP, SNMP | 게이트웨이 | `curl` `wget` `dig` `ssh` |
| 6 표현(Presentation) | 〃 | 데이터 | — | SSL/TLS, JPEG, ASCII, MIME | — | `openssl s_client`(Part 10) |
| 5 세션(Session) | 〃 | 데이터 | — | NetBIOS, RPC, SQL | — | — |
| 4 전송(Transport) | 전송(Transport) | **세그먼트**(TCP)·**데이터그램**(UDP) | **포트** | **TCP, UDP** | L4 스위치 | `ss` `netstat` `nc` `nmap` `telnet` |
| 3 네트워크(Network) | 인터넷(Internet) | **패킷**(Packet) | **IP 주소** | **IP**, ICMP, ARP, RARP, IGMP | 라우터, L3 스위치 | `ip route` `ping` `traceroute` `mtr` |
| 2 데이터링크(Data Link) | 네트워크 접근 | **프레임**(Frame) | **MAC 주소** | Ethernet, PPP, ARP | 스위치, 브리지 | `ip link` `ip neigh` `arping` `ethtool` |
| 1 물리(Physical) | 〃 | **비트**(Bit) | — | RS-232, 케이블·커넥터 규격 | 리피터, 허브 | `ethtool`(Link detected) |

- 암기: **A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing (7→1)
- TCP/IP 4계층의 **응용 계층이 OSI 의 5·6·7 을 통합** — 대응 문제의 핵심
- ARP 는 관례상 2·3계층 경계 (논리 IP ↔ 물리 MAC 변환)

**캡슐화 (송신 측, 위 → 아래)**

```text
[응용]   데이터
   ↓ 전송 계층 헤더(포트·순서번호) 부착
[전송]   세그먼트 =  TCP헤더 + 데이터
   ↓ IP 헤더(출발·목적 IP, TTL, 프로토콜) 부착
[네트워크] 패킷    =  IP헤더 + TCP헤더 + 데이터
   ↓ 이더넷 헤더(출발·목적 MAC) + 트레일러(FCS) 부착
[데이터링크] 프레임 =  MAC헤더 + IP헤더 + TCP헤더 + 데이터 + FCS
   ↓
[물리]   비트 스트림 (전기·광 신호)
```

- 수신 측은 역순으로 헤더를 벗김 = **역캡슐화(decapsulation)**
- `tcpdump -e` 로 **MAC 헤더까지** 보면 캡슐화 결과를 실물로 확인 가능

**검증** — 한 패킷 안에 각 계층 헤더가 모두 들어 있음을 확인

```bash
tcpdump -i enp0s1 -nn -e -c 2 icmp &         # [배경] MAC 헤더 포함 캡처
ping -c 2 192.168.64.1 >/dev/null; wait
```

```text
# tcpdump -i enp0s1 -nn -e -c 2 icmp
... <출발MAC> > <목적MAC>, ethertype IPv4 (0x0800), length 98: 192.168.64.10 > 192.168.64.1: ICMP echo request, ...
   └─2계층(MAC)──────────────┘ └─EtherType┘        └─3계층(IP)───────────────┘ └─ICMP─┘
```

- `ethertype IPv4 (0x0800)` : 이더넷 프레임이 담고 있는 상위 프로토콜 — ARP 는 `0x0806`

> 📝 **시험 포인트**: 계층별 PDU 명칭(데이터-세그먼트-패킷-프레임-비트)과 장비(게이트웨이-L4스위치-라우터-스위치-리피터/허브) 매칭이 매 회차 출제. **2계층=MAC(물리 주소)**, **3계층=IP(논리 주소)**, **4계층=포트** 3단 구분이 함정 방지의 핵심.

### 6-4. ipcalc 로 서브네팅 계산 검증

> **상황**: 손으로 계산한 서브넷 값을 명령으로 대조한다. 시험장에서는 손계산이지만, 연습 단계에서는 정답 확인 수단이 필요하다.

```bash
rpm -q ipcalc || dnf install -y ipcalc

ipcalc -n 192.168.64.10/24            # 네트워크 주소
ipcalc -b 192.168.64.10/24            # 브로드캐스트 주소
ipcalc -m 192.168.64.10/24            # 넷마스크
ipcalc -p 192.168.64.10/24            # 프리픽스
ipcalc -n -b -m -p 192.168.64.10/24   # 한 번에
ipcalc --minaddr --maxaddr --addresses 192.168.64.10/24   # 첫·끝 호스트, 총 주소 수
ipcalc 192.168.64.10/26               # /26 으로 계산
ipcalc 10.0.0.5/8                     # 사설 A 대역
ipcalc 172.16.5.1/12                  # 사설 B 대역
```

- `-n` : 네트워크 주소 (**n**etwork)
- `-b` : 브로드캐스트 주소 (**b**roadcast)
- `-m` : 넷마스크 (**m**ask)
- `-p` : 프리픽스 길이 (**p**refix)
- `--minaddr` / `--maxaddr` : 사용 가능한 첫 / 마지막 호스트 주소
- `--addresses` : 대역의 총 주소 수 (네트워크·브로드캐스트 포함)
- `-s <호스트수>...` / `--split` : 요구 호스트 수에 맞춰 분할 (※ 버전에 따라 지원 여부 상이)
- ※ 출력 형식은 배포판의 ipcalc 버전에 따라 `KEY=VALUE` 형태 또는 표 형태로 다름 — **값 자체**만 확인할 것

**서브네팅 계산 실습표** — 기준 `192.168.64.0/24` 를 분할

| 프리픽스 | 넷마스크 | 마지막 옥텟(2진) | 블록 크기 | 서브넷 개수 | 서브넷당 호스트 | 첫 서브넷 (네트워크 / 첫 호스트 ~ 끝 호스트 / 브로드캐스트) |
| --- | --- | --- | --- | --- | --- | --- |
| /24 | 255.255.255.0 | `00000000` | 256 | 1 | 2⁸−2 = **254** | .0 / .1 ~ .254 / .255 |
| /25 | 255.255.255.128 | `10000000` | 128 | 2 | 2⁷−2 = **126** | .0 / .1 ~ .126 / .127 |
| /26 | 255.255.255.192 | `11000000` | 64 | 4 | 2⁶−2 = **62** | .0 / .1 ~ .62 / .63 |
| /27 | 255.255.255.224 | `11100000` | 32 | 8 | 2⁵−2 = **30** | .0 / .1 ~ .30 / .31 |
| /28 | 255.255.255.240 | `11110000` | 16 | 16 | 2⁴−2 = **14** | .0 / .1 ~ .14 / .15 |
| /29 | 255.255.255.248 | `11111000` | 8 | 32 | 2³−2 = **6** | .0 / .1 ~ .6 / .7 |
| /30 | 255.255.255.252 | `11111100` | 8 → 4 | 64 | 2²−2 = **2** | .0 / .1 ~ .2 / .3 (라우터 간 연결용) |

- 계산 공식 3개
  - 사용 가능 호스트 수 = **2^(호스트 비트) − 2** (네트워크·브로드캐스트 제외)
  - 서브넷 개수 = **2^(차용한 비트 수)**
  - 블록 크기 = **256 − (넷마스크 마지막 옥텟 값)** → 네트워크 주소는 블록 크기의 배수
- 넷마스크 옥텟 값 순서 암기: **128 → 192 → 224 → 240 → 248 → 252 → 254 → 255**
- 역산 예: "호스트 100대 수용" → 2⁷−2=126 ≥ 100 → **/25** 필요 (/26 은 62 로 부족)
- `192.168.64.0/26` 의 4개 서브넷: `.0`(0~63), `.64`(64~127), `.128`(128~191), `.192`(192~255)

**검증** — 표의 값을 명령으로 확인

```bash
for p in 24 25 26 27 28; do
  echo "--- /$p"
  ipcalc -n -b -m 192.168.64.10/$p
done
ipcalc --minaddr --maxaddr 192.168.64.10/26
ipcalc -n 172.16.35.7/20        # 기출 유형: /20 의 네트워크 주소
```

```text
# ipcalc -n -b -m 192.168.64.10/26
NETWORK=192.168.64.0
NETMASK=255.255.255.192
BROADCAST=192.168.64.63
# ipcalc -n 172.16.35.7/20
NETWORK=172.16.32.0        ← /20 = 255.255.240.0, 블록 16 → 32의 배수
```

**CIDR 과 VLSM**

- **CIDR**(Classless Inter-Domain Routing) : 클래스 경계를 무시하고 `/n` 프리픽스로 임의 길이 분할 — 주소 낭비 감소, 라우팅 테이블 축약(supernetting)
- **VLSM**(Variable Length Subnet Mask) : 한 네트워크 안에서 **서브넷마다 다른 프리픽스** 사용
  - 예: `203.0.113.0/24` 를 100대·60대·20대용으로 → **/25(126)** + **/26(62)** + **/27(30)**
  - 배정 순서: **큰 것부터** 할당해야 주소가 겹치지 않음
    - 100대 → `203.0.113.0/25` (0~127)
    - 60대 → `203.0.113.128/26` (128~191)
    - 20대 → `203.0.113.192/27` (192~223)

> 📝 **시험 포인트**: `/26` → 호스트 62개, `/25` → 126개 계산이 매 회차 출제. "호스트 30대씩 균등 분할" → 30 ≤ 2⁵−2 → **/27 (255.255.255.224)**. "172.16.35.7/20 의 네트워크 주소" → 마스크 255.255.240.0, 세 번째 옥텟 블록 16 → **172.16.32.0**.

### 6-5. 주소 클래스·사설 대역·특수 주소

> **상황**: 시험의 암기 항목을 실제 주소로 대조한다. 이 서버의 `192.168.64.10` 이 어느 분류에 드는지 확인하면서 표를 굳힌다.

```bash
ip -4 -o a show enp0s1 | awk '{print $4}'        # 192.168.64.10/24 → C클래스 사설
ipcalc 192.168.64.10/24 | head
ipcalc 127.0.0.1/8 | head
ip a show lo | grep 'inet '                      # 127.0.0.1/8 루프백
ip route show default | awk '{print $1}'         # default = 0.0.0.0/0
grep -c . /etc/hosts
```

**클래스별 범위 (최우선 암기)**

| 클래스 | 첫 옥텟 범위 | 상위 비트 | 기본 넷마스크 | 프리픽스 | 네트워크 수 / 호스트 수 | 용도 |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 ~ 127 | `0` | 255.0.0.0 | /8 | 126 / 약 1,677만 | 대규모 |
| B | 128 ~ 191 | `10` | 255.255.0.0 | /16 | 16,384 / 65,534 | 중규모 |
| C | 192 ~ 223 | `110` | 255.255.255.0 | /24 | 약 209만 / **254** | 소규모 |
| D | 224 ~ 239 | `1110` | — | — | — | **멀티캐스트** |
| E | 240 ~ 255 | `1111` | — | — | — | 연구·예약 |

**사설 IP 대역 (RFC 1918) — 3개만**

| 대역 | 클래스 | 범위 |
| --- | --- | --- |
| `10.0.0.0/8` | A | 10.0.0.0 ~ 10.255.255.255 |
| `172.16.0.0/12` | B | 172.16.0.0 ~ **172.31**.255.255 |
| `192.168.0.0/16` | C | 192.168.0.0 ~ 192.168.255.255 |

**특수 주소 (함정 단골)**

| 주소·대역 | 이름 | 의미 |
| --- | --- | --- |
| `127.0.0.0/8` | 루프백(loopback) | 자기 자신 — `127.0.0.1` = localhost. **A클래스 범위지만 호스트 할당 불가** |
| `0.0.0.0` | 미지정(unspecified) | 출발지: "주소 없음"(DHCP 요청) / 목적지·라우팅: **전체·기본 경로** / 리스닝: **모든 인터페이스** |
| `0.0.0.0/0` | 기본 경로 | `ip route` 의 `default` 와 동일 |
| `169.254.0.0/16` | 링크로컬(APIPA) | DHCP 실패 시 자동 할당 — **사설 대역 아님** (오답 유도 최빈출) |
| `224.0.0.0/4` | 멀티캐스트(D클래스) | 그룹 전송. `224.0.0.1`=모든 호스트, `224.0.0.2`=모든 라우터 |
| `255.255.255.255` | 제한 브로드캐스트 | 같은 링크 전체 — 라우터를 넘지 못함 |
| `<네트워크>.255` (예 `192.168.64.255`) | 직접 브로드캐스트 | 해당 서브넷 전체 |
| `<네트워크>.0` (예 `192.168.64.0`) | 네트워크 주소 | 호스트에 할당 불가 |
| `100.64.0.0/10` | CGNAT (RFC 6598) | 통신사 대규모 NAT 전용 |

**검증**

```bash
ping -c 1 -b 192.168.64.255 2>&1 | head -3    # 브로드캐스트 (응답은 환경에 따라 다름)
ip a | grep -c '169.254'                       # 0 이면 DHCP 정상 (APIPA 미발생)
ss -tlnp | grep '0.0.0.0:22'                   # 0.0.0.0 = 모든 인터페이스 수신
ip route show default
```

```text
# ss -tlnp | grep '0.0.0.0:22'
tcp   LISTEN 0  128  0.0.0.0:22   0.0.0.0:*   users:(("sshd",...))
# ip route show default
default via 192.168.64.1 dev enp0s1 proto static metric 100
```

> 📝 **시험 포인트**: 사설 IP 3대역은 **10/8, 172.16/12, 192.168/16**. `172.16/12` 는 172.16 ~ **172.31** 까지 (172.32 는 공인) — 범위 함정. `169.254.0.0/16`(APIPA) 은 **사설 대역이 아님**. `127.0.0.0/8` 은 A클래스 숫자 범위에 들지만 루프백 예약.

### 6-6. IPv6 표기 축약 규칙

> **상황**: 1-4 에서 확인한 링크로컬 주소를 근거로 축약 규칙을 정리한다.

```bash
ip -6 -o a show enp0s1 | awk '{print $4}'      # 실제 축약된 형태로 출력됨
ping6 -c 2 ::1                                  # 루프백 (= 0:0:0:0:0:0:0:1)
ping -6 -c 2 ::1                                # 동일 (ping 의 -6 옵션)
ip -6 route
sysctl net.ipv6.conf.all.disable_ipv6           # 0 = 활성
```

**축약 규칙 3단계**

```text
원본:    2001:0db8:0000:0000:0000:ff00:0042:8329
① 각 그룹 선행 0 제거   → 2001:db8:0:0:0:ff00:42:8329
② 연속 0 그룹을 :: 로   → 2001:db8::ff00:42:8329
③ :: 는 주소당 한 번만  → 2001:0:0:1:0:0:0:1 은
                          2001::1:0:0:0:1  (앞을 압축) 또는
                          2001:0:0:1::1    (뒤를 압축) — 둘 중 하나만
```

- `::1` = 루프백 (IPv4 의 `127.0.0.1`)
- `::` = 미지정 주소 (IPv4 의 `0.0.0.0`)
- `fe80::/10` = 링크로컬 — 라우터를 넘지 못함, 인터페이스 지정 필요 (`ping6 fe80::1%enp0s1`)
- `ff00::/8` = 멀티캐스트 — **IPv6 에는 브로드캐스트가 없음**

**IPv4 ↔ IPv6 비교표**

| 항목 | IPv4 | IPv6 |
| --- | --- | --- |
| 주소 길이 | **32비트** | **128비트** |
| 표기 | 10진수 점 표기(4옥텟) | 16진수 콜론 표기(8그룹) |
| 주소 자동 설정 | DHCP 필요 | **SLAAC** 자동 구성 지원 |
| 헤더 | 가변(20~60바이트) | 고정 40바이트 |
| 보안 | IPsec 선택 | IPsec 기본 내장 |
| 브로드캐스트 | 있음 | **없음** (멀티캐스트·애니캐스트로 대체) |
| 단편화 | 라우터가 수행 가능 | 송신 호스트만 수행 |

**검증**

```bash
ping6 -c 2 ::1 | tail -2
ip -6 -br a | head
```

```text
# ping6 -c 2 ::1
2 packets transmitted, 2 received, 0% packet loss, time ...
# ip -6 -br a
lo               UNKNOWN        ::1/128
enp0s1           UP             fe80::.../64
```

> 📝 **시험 포인트**: 축약 표기 문제는 "**`::` 는 한 번만**" 이 핵심 함정. IPv6 는 브로드캐스트가 없다(멀티캐스트/애니캐스트로 대체)는 문항도 반복 출제.

### 6-7. 주요 포트 30종 — /etc/services 로 전수 검증

> **상황**: 암기표를 만들고 `grep` 으로 하나씩 대조한다. Part 09 에서 각 서비스를 실제로 띄울 때 이 표가 방화벽·`ss` 검증의 기준이 된다.

**주요 포트 표 (30종)**

| 포트 | 프로토콜 | 서비스 | `/etc/services` 이름 | 파트 |
| --- | --- | --- | --- | --- |
| 20 | TCP | FTP **데이터** (능동 모드) | `ftp-data` | 09 |
| 21 | TCP | FTP **제어** | `ftp` | 09 |
| 22 | TCP | **SSH / SCP / SFTP** | `ssh` | 08 |
| 23 | TCP | Telnet (평문) | `telnet` | 08 |
| 25 | TCP | SMTP (메일 **송신**) | `smtp` | 09 |
| 53 | **UDP/TCP** | DNS (질의 UDP, 존 전송 TCP) | `domain` | 09 |
| 67 | UDP | DHCP **서버** | `bootps` | — |
| 68 | UDP | DHCP **클라이언트** | `bootpc` | — |
| 69 | UDP | TFTP | `tftp` | — |
| 80 | TCP | HTTP | `http` | 09 |
| 110 | TCP | POP3 (수신 후 **삭제**) | `pop3` | 09 |
| 111 | TCP/UDP | RPC portmapper (NFS 필수) | `sunrpc` | 09 |
| 119 | TCP | NNTP (뉴스) | `nntp` | — |
| 123 | **UDP** | **NTP** (시간 동기화) | `ntp` | 08 |
| 137 | UDP | NetBIOS 이름 서비스 | `netbios-ns` | 09 |
| 138 | UDP | NetBIOS 데이터그램 | `netbios-dgm` | 09 |
| 139 | TCP | NetBIOS 세션 (구 SMB) | `netbios-ssn` | 09 |
| 143 | TCP | IMAP (수신 후 **보관**) | `imap` | 09 |
| 161 | UDP | SNMP | `snmp` | — |
| 162 | UDP | SNMP Trap | `snmptrap` | — |
| 389 | TCP | LDAP | `ldap` | — |
| 443 | TCP | HTTPS (HTTP over TLS) | `https` | 09 |
| 445 | TCP | **SMB / CIFS** (Samba 직접) | `microsoft-ds` | 09 |
| 465 | TCP | SMTPS (암시적 TLS) | `smtps`/`submissions` | 09 |
| 514 | **UDP** | syslog 원격 전송 | `syslog` | 07 |
| 587 | TCP | SMTP Submission (STARTTLS) | `submission` | 09 |
| 631 | TCP | **IPP / CUPS** 웹 관리 | `ipp` | 09 |
| 636 | TCP | LDAPS | `ldaps` | — |
| 993 | TCP | IMAPS | `imaps` | 09 |
| 995 | TCP | POP3S | `pop3s` | 09 |
| 2049 | TCP/UDP | **NFS** | `nfs` | 09 |
| 3306 | TCP | MySQL / MariaDB | `mysql` | — |
| 5432 | TCP | PostgreSQL | `postgresql` | — |
| 6379 | TCP | Redis | `redis` | 11 |
| 8080 | TCP | HTTP 대체 (프록시·앱 서버) | `webcache`/`http-alt` | 09 |
| **2222** | TCP | 이 실습의 **SSH 변경 포트** | 미등록 (사용자 지정) | 08 |

- 포트 범위: **Well-known 0~1023** / **Registered 1024~49151** / **Dynamic(Private) 49152~65535**
- ⚠️ `514` 는 **UDP=syslog, TCP=shell(rsh)** 로 서로 다름 — grep 시 프로토콜까지 확인
- 일부 포트(`redis` 등)는 배포판 `/etc/services` 에 미등록일 수 있음 → 출력 없음이 정상

**검증** — 전수 대조 스크립트

```bash
for p in 20 21 22 23 25 53 67 68 69 80 110 111 119 123 137 138 139 143 \
         161 162 389 443 445 465 514 587 631 636 993 995 2049 3306 5432 8080; do
  printf '%-6s tcp: %-14s udp: %s\n' "$p" \
    "$(awk -v P="$p/tcp" '$2==P{print $1; exit}' /etc/services)" \
    "$(awk -v P="$p/udp" '$2==P{print $1; exit}' /etc/services)"
done
```

```text
20     tcp: ftp-data       udp: ftp-data
21     tcp: ftp            udp: ftp
22     tcp: ssh            udp: ssh
23     tcp: telnet         udp: telnet
25     tcp: smtp           udp: smtp
53     tcp: domain         udp: domain
67     tcp: bootps         udp: bootps
68     tcp: bootpc         udp: bootpc
80     tcp: http           udp: http
110    tcp: pop3           udp: pop3
123    tcp: ntp            udp: ntp
143    tcp: imap           udp: imap
443    tcp: https          udp: https
631    tcp: ipp            udp: ipp
2049   tcp: nfs            udp: nfs
...
```

- ※ `/etc/services` 는 관례상 대부분의 항목을 TCP·UDP 양쪽에 등록해 둠 → **실제로 그 프로토콜을 쓴다는 뜻은 아님**. 실사용 프로토콜은 위 표의 굵은 표기를 따를 것 (예: NTP=UDP 123, DNS 질의=UDP 53)

> 📝 **시험 포인트**: 포트↔서비스 매칭은 매 회차 3~5문항. 특히 혼동 유발 조합 — **20/21 FTP 데이터·제어**, **110 POP3 vs 143 IMAP**, **67/68 DHCP 서버·클라이언트**, **465/587 SMTPS·Submission**, **631 CUPS**, **445 SMB**, **2049 NFS + 111 RPC**. Well-known 상한은 **1023**.

---

## 7. SSH 서버 설정 점검

### 7-1. sshd 상태·버전 확인

> **상황**: 키 인증·포트 변경에 들어가기 전에 현재 sshd 가 어떤 상태로 떠 있는지 기준점을 잡는다.

```bash
rpm -q openssh-server openssh-clients          # 설치 확인 (minimal 에도 기본 포함)
systemctl status sshd --no-pager
systemctl is-enabled sshd                      # enabled → 부팅 시 자동 시작
systemctl is-active sshd                       # active
ss -tlnp | grep sshd                           # 리스닝 포트·PID
ssh -V                                         # 클라이언트·OpenSSL 버전
sshd -V 2>&1 | head -2                         # 서버 버전 (일부 버전은 -V 미지원 → ssh -V 사용)
ls -l /etc/ssh/                                # 설정 파일·호스트키 목록
journalctl -u sshd -n 20 --no-pager            # 최근 로그
```

- `openssh-server` : `sshd` 데몬 — RHEL/Rocky minimal 설치에도 **기본 포함·자동 시작**
- `openssh-clients` : `ssh` `scp` `sftp` `ssh-copy-id` `ssh-agent` `ssh-add`
- `ssh -V` 출력: `OpenSSH_<버전>, OpenSSL <버전> <날짜>`

**검증**

```bash
systemctl is-enabled sshd; systemctl is-active sshd
ss -tlnp | awk '/sshd/{print $1, $5}'
ls /etc/ssh/ | grep -E 'sshd_config|ssh_host'
```

```text
# systemctl is-enabled sshd
enabled
# systemctl is-active sshd
active
# ss -tlnp | grep sshd
tcp LISTEN 0 128    0.0.0.0:22   0.0.0.0:*  users:(("sshd",pid=...,fd=...))
tcp LISTEN 0 128       [::]:22      [::]:*  users:(("sshd",pid=...,fd=...))
# ls /etc/ssh/
moduli  ssh_config  ssh_config.d  sshd_config  sshd_config.d
ssh_host_ecdsa_key      ssh_host_ecdsa_key.pub
ssh_host_ed25519_key    ssh_host_ed25519_key.pub
ssh_host_rsa_key        ssh_host_rsa_key.pub
```

- 호스트키 3쌍(`rsa`·`ecdsa`·`ed25519`)은 최초 기동 시 자동 생성 — 개인키는 **600**, 공개키는 644

> 📝 **시험 포인트**: SSH 데몬 이름은 **`sshd`**(서비스 유닛 `sshd.service`), 기본 포트 **22/TCP**. `systemctl enable sshd`(부팅 자동) ↔ `start`(즉시 시작) 구분. 접속 실패 시 `Connection refused` = 데몬 정지, `Connection timed out` = 방화벽·경로 문제.

### 7-2. /etc/ssh/sshd_config 주요 지시자

> **상황**: 8·9절에서 고칠 항목을 미리 표로 정리하고 현재 값을 확인한다. RHEL 9 는 `Include` 구조가 추가되어 "어느 파일이 이기는가"를 알아야 한다.

```bash
cp /etc/ssh/sshd_config /root/net-backup/sshd_config.orig     # 백업 필수
head -20 /etc/ssh/sshd_config                                  # Include 행 확인
ls -l /etc/ssh/sshd_config.d/                                  # 드롭인 디렉터리
grep -vE '^\s*#|^\s*$' /etc/ssh/sshd_config                    # 주석 제외 실제 설정만
grep -rvE '^\s*#|^\s*$' /etc/ssh/sshd_config.d/ 2>/dev/null    # 드롭인 실제 설정
```

- ⚠️ **RHEL 9 는 `/etc/ssh/sshd_config` 상단에 `Include /etc/ssh/sshd_config.d/*.conf` 가 있음**
  - sshd 는 **먼저 읽은 값이 이긴다**(first obtained wins) → `Include` 가 앞에 있으면 **드롭인 파일이 본문보다 우선**
  - 본문 수정이 안 먹으면 `/etc/ssh/sshd_config.d/*.conf` 를 먼저 확인할 것
  - 실효값 판정은 항상 `sshd -T` (7-4)

**주요 지시자 표**

| 지시자 | 의미 | 값 예 | 비고 |
| --- | --- | --- | --- |
| `Port` | 수신 포트 | `22` → `2222` | 여러 줄로 복수 포트 가능. **SELinux·방화벽 동반 필요** |
| `ListenAddress` | 수신 주소 | `0.0.0.0` / `192.168.64.10` | 특정 IP 로 한정 시 다른 인터페이스 접속 차단 |
| `AddressFamily` | 주소 계열 | `any` / `inet`(IPv4) / `inet6` | IPv6 리스닝 제거 시 `inet` |
| `PermitRootLogin` | root 직접 로그인 | `yes` / `no` / `prohibit-password` / `forced-commands-only` | `prohibit-password` = 키만 허용 |
| `PasswordAuthentication` | 비밀번호 인증 | `yes` / `no` | **`no` 로 바꾸기 전 키 인증 성공 확인 필수** |
| `PubkeyAuthentication` | 공개키 인증 | `yes`(기본) / `no` | |
| `AuthorizedKeysFile` | 공개키 등록 파일 | `.ssh/authorized_keys` | 홈 기준 상대 경로 |
| `PermitEmptyPasswords` | 빈 비밀번호 허용 | `no`(기본) | `yes` 는 심각한 취약점 |
| `MaxAuthTries` | 인증 시도 횟수 | `6`(기본) → `3` | 초과 시 연결 종료 (무차별 대입 완화) |
| `MaxSessions` | 연결당 최대 세션 | `10`(기본) | 다중화 세션 수 |
| `MaxStartups` | 미인증 동시 연결 | `10:30:100` | DoS 완화 |
| `AllowUsers` | 접속 허용 사용자 | `admin1 dev1` | 지정 시 **나머지 전원 거부** |
| `DenyUsers` | 접속 거부 사용자 | `guest1` | Deny 가 Allow 보다 우선 |
| `AllowGroups` | 허용 그룹 | `wheel devteam` | |
| `DenyGroups` | 거부 그룹 | `nologin` | 평가 순서: **DenyUsers → AllowUsers → DenyGroups → AllowGroups** |
| `ClientAliveInterval` | 무응답 점검 간격(초) | `300` | 0 = 비활성 |
| `ClientAliveCountMax` | 무응답 허용 횟수 | `3`(기본) | 300×3 = 15분 후 세션 종료 |
| `LoginGraceTime` | 로그인 완료 제한(초) | `120`(기본) → `60` | 미인증 연결 점유 방지 |
| `X11Forwarding` | X11 전달 | `yes`(RHEL 기본) / `no` | 사용 안 하면 `no` 권장 |
| `Banner` | 접속 전 경고문 | `/etc/issue.net` | 법적 고지 |
| `PrintMotd` | `/etc/motd` 출력 | `no`(RHEL) | PAM 이 대신 출력 |
| `LogLevel` | 로그 상세도 | `INFO`(기본) / `VERBOSE` / `DEBUG` | `VERBOSE` 는 키 지문까지 기록 |
| `SyslogFacility` | 로그 facility | `AUTHPRIV` | → `/var/log/secure` (Part 07) |
| `UseDNS` | 접속지 역방향 조회 | `no`(기본) | `yes` 면 DNS 지연 시 접속 느려짐 |
| `Subsystem sftp` | SFTP 하위 시스템 | `/usr/libexec/openssh/sftp-server` | 이 줄 없으면 `sftp` 불가 |
| `StrictModes` | 홈·키 권한 검사 | `yes`(기본) | 권한 과다 시 키 인증 무시 |
| `Protocol` | (구) 프로토콜 버전 | — | **OpenSSH 7.4+ 에서 제거** — SSH-2 전용 |

**검증**

```bash
grep -nE '^\s*(Port|PermitRootLogin|PasswordAuthentication|PubkeyAuthentication|Subsystem|Include)' \
  /etc/ssh/sshd_config
ls /etc/ssh/sshd_config.d/
```

```text
# grep -n 'Include' /etc/ssh/sshd_config
2:Include /etc/ssh/sshd_config.d/*.conf
# ls /etc/ssh/sshd_config.d/
50-redhat.conf
# grep -nE '^\s*(Port|PermitRootLogin|PasswordAuthentication)' /etc/ssh/sshd_config
...:#Port 22
...:#PermitRootLogin prohibit-password
...:PasswordAuthentication yes
```

- `#` 로 시작하는 줄은 **주석이자 기본값 표시** — 값을 바꾸려면 `#` 을 제거하고 수정
- 현재 실효값은 파일이 아니라 `sshd -T` 로 판정할 것 (7-4)

> 📝 **시험 포인트**: `Port 2222` / `PermitRootLogin no` / `PasswordAuthentication no` 3종 세트 해석 문제가 거의 매 회차 출제. `PermitRootLogin prohibit-password` = **비밀번호는 막고 키 인증은 허용**. `AuthorizedKeysFile` 기본값 `.ssh/authorized_keys`.

### 7-3. ssh_config vs ~/.ssh/config — 클라이언트 설정 우선순위

> **상황**: 서버 설정(`sshd_config`)과 클라이언트 설정(`ssh_config`)은 완전히 다른 파일이다. 매번 `-p 2222 -i ...` 를 치지 않도록 별칭을 만들어 둔다.

```bash
ls -l /etc/ssh/ssh_config /etc/ssh/ssh_config.d/            # 시스템 전역 클라이언트 설정
grep -vE '^\s*#|^\s*$' /etc/ssh/ssh_config
```

- macOS(클라이언트) 에서 개인 설정 작성

```bash
# [macOS 터미널]
mkdir -p ~/.ssh && chmod 700 ~/.ssh
cat >> ~/.ssh/config <<'EOF'
Host srv01
    HostName 192.168.64.10
    Port 2222
    User admin1
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 60
    ServerAliveCountMax 3
EOF
chmod 600 ~/.ssh/config
ssh srv01              # → ssh -p 2222 -i ~/.ssh/id_ed25519 admin1@192.168.64.10 과 동일
```

- `Host <별칭>` : 이 블록이 적용될 대상 패턴 (`*` 와일드카드 가능)
- `HostName` : 실제 접속 주소 (별칭과 분리)
- `Port` / `User` / `IdentityFile` : 포트·계정·개인키
- `ServerAliveInterval` / `ServerAliveCountMax` : **클라이언트가** 서버에 살아있음 신호를 보내는 주기·허용 실패 횟수 → NAT 타임아웃으로 끊기는 것 방지
- `ProxyJump` : 점프 호스트 (`ssh -J` 의 설정 파일 판)
- `StrictHostKeyChecking` : 호스트키 검증 정책 (`yes`/`accept-new`/`no`)

**클라이언트 설정 우선순위 (위가 이김)**

| 순위 | 위치 | 비고 |
| --- | --- | --- |
| 1 | `ssh` 명령행 옵션 (`-p`, `-i`, `-o`) | 최우선 |
| 2 | `~/.ssh/config` | 사용자 개인 설정 — **권한 600 필수** |
| 3 | `/etc/ssh/ssh_config` (+ `ssh_config.d/*`) | 시스템 전역 기본값 |

- 서버 측은 별개: `/etc/ssh/sshd_config` + `/etc/ssh/sshd_config.d/*.conf`
- 클라이언트 설정은 **먼저 얻은 값이 이김**(first obtained wins) → 구체적인 `Host` 블록을 파일 위쪽에 둘 것

**검증**

```bash
# [macOS]
ssh -G srv01 | grep -E '^(hostname|port|user|identityfile)'   # 실효 클라이언트 설정
stat -f '%A %N' ~/.ssh/config                                  # macOS: 권한 600 확인
```

```text
# ssh -G srv01
hostname 192.168.64.10
port 2222
user admin1
identityfile ~/.ssh/id_ed25519
```

- `ssh -G <호스트>` : 실제 적용될 클라이언트 설정 전체 출력 (**G**et config) — `sshd -T` 의 클라이언트 판

> 📝 **시험 포인트**: **서버 설정 = `/etc/ssh/sshd_config`**, **클라이언트 설정 = `/etc/ssh/ssh_config`·`~/.ssh/config`**. 파일명 `sshd_config` 와 `ssh_config` 한 글자 차이가 함정. `~/.ssh/config` 권한이 과다하면 무시됨(600 필요).

### 7-4. 문법 검사와 실효 설정 확인

> **상황**: `sshd` 재시작 전에 문법 오류를 잡는다. 오타 하나로 데몬이 안 뜨면 원격에서 복구할 방법이 없다.

```bash
sshd -t                                    # 문법 검사 (오류 없으면 무출력)
echo $?                                    # 0 = 정상
sshd -T | head -20                         # 실효 설정 전체 (Extended test mode)
sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication)'
sshd -T | grep -E '^(maxauthtries|logingracetime|clientaliveinterval|x11forwarding|usedns)'
sshd -T | wc -l                            # 실효 항목 수
sshd -t -f /etc/ssh/sshd_config            # 특정 파일 지정 검사
```

- `-t` : 설정 파일 문법 검사 (**t**est) — 오류 시 줄 번호와 함께 출력
- `-T` : 실효 설정 전부 덤프 (**T**est extended) — **Include·드롭인·기본값이 모두 반영된 최종값**
- `-f <파일>` : 검사할 설정 파일 지정 (**f**ile)
- `-d` : 디버그 모드로 포그라운드 실행 (**d**ebug) — 접속 문제 심층 진단용
- ⚠️ `-t`/`-T` 는 **root 권한 필요** (호스트키를 읽어야 하므로)

**검증**

```bash
sshd -t && echo "문법 OK"
sshd -T | grep -E '^port|^permitrootlogin|^passwordauthentication'
```

```text
# sshd -t && echo "문법 OK"
문법 OK
# sshd -T | grep -E '^port|^permitrootlogin|^passwordauthentication'
port 22
permitrootlogin ...
passwordauthentication yes
```

- 실효값은 배포판·드롭인 구성에 따라 다름 → **문서 값을 믿지 말고 이 명령의 출력을 기준**으로 삼을 것
- 문법 오류 예: `Bad configuration option: Prot` / `/etc/ssh/sshd_config line 17: Missing argument`

> 📝 **시험 포인트**: 설정 변경 후 **`sshd -t` 로 검사 → `systemctl reload sshd`** 순서가 정답. `reload`(설정만 다시 읽음, 기존 세션 유지) vs `restart`(데몬 재시작, 포트 변경 시 필요) 구분. Apache 의 `apachectl configtest` 와 같은 위치의 명령.

---

## 8. 키 인증 전환

### 8-1. macOS 에서 키 쌍 생성

> **상황**: 비밀번호 로그인을 끄려면 먼저 키 인증이 확실히 되어야 한다. 클라이언트(macOS)에서 키를 만든다.

```bash
# [macOS 터미널]
mkdir -p ~/.ssh && chmod 700 ~/.ssh
ssh-keygen -t ed25519 -C "mac->srv01"                        # 권장 (짧고 빠르고 안전)
# 대안 — 구형 서버 호환이 필요할 때
ssh-keygen -t rsa -b 4096 -C "mac->srv01-rsa" -f ~/.ssh/id_rsa_lab
ls -l ~/.ssh/
```

- `-t <유형>` : 키 알고리즘 (**t**ype) — `ed25519`(권장) / `rsa` / `ecdsa` / `dsa`(폐기)
- `-b <비트>` : 키 길이 (**b**its) — RSA 는 최소 2048, 권장 **4096**. ed25519 는 길이 고정이라 `-b` 무의미
- `-C <주석>` : 공개키 끝에 붙는 주석 (**C**omment) — 어느 PC 의 키인지 식별용
- `-f <파일>` : 저장 경로 (**f**ile) — 미지정 시 `~/.ssh/id_<유형>`
- `-N '<암호>'` : 패스프레이즈를 명령행에서 지정 (**N**ew passphrase) — `''` 는 무암호. ⚠️ 히스토리 노출
- `-p` : 기존 키의 **패스프레이즈만 변경** (**p**assphrase) — 키 자체는 그대로
- `-y` : 개인키에서 공개키 재생성 (`.pub` 분실 시)
- `-l -f <키>` : 지문(fingerprint) 출력 (**l**ist)
- 패스프레이즈는 **개인키 파일이 유출됐을 때의 마지막 방어선** → 설정 권장, 대신 `ssh-agent` 로 반복 입력 회피

```bash
# [macOS] 패스프레이즈 변경 (키 교체 없이)
ssh-keygen -p -f ~/.ssh/id_ed25519
```

**검증**

```bash
# [macOS]
ls -l ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub
ssh-keygen -lf ~/.ssh/id_ed25519.pub          # 지문 확인
cat ~/.ssh/id_ed25519.pub
```

```text
# ls -l ~/.ssh/
-rw-------  1 sunwoo  staff  ... id_ed25519       ← 개인키 600
-rw-r--r--  1 sunwoo  staff  ... id_ed25519.pub   ← 공개키 644
# ssh-keygen -lf ~/.ssh/id_ed25519.pub
256 SHA256:<지문> mac->srv01 (ED25519)
# cat ~/.ssh/id_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... mac->srv01
```

- ⚠️ **개인키(`id_ed25519`)는 절대 서버·저장소·메신저로 옮기지 않는다.** 전달·등록 대상은 **공개키(`.pub`)** 뿐
- 개인키 권한이 과다(예: 644)면 `WARNING: UNPROTECTED PRIVATE KEY FILE!` 로 **무시됨** → `chmod 600`(또는 400)

> 📝 **시험 포인트**: 공개키 인증 절차 순서 = **① 클라이언트에서 `ssh-keygen` 으로 키 쌍 생성 → ② 공개키를 서버의 `~/.ssh/authorized_keys` 에 등록 → ③ 접속 시 개인키로 서명, 서버가 공개키로 검증**. 서버에 등록되는 것은 **공개키**라는 점이 최빈출 함정.

### 8-2. ssh-copy-id 로 공개키 배포와 권한 확인

> **상황**: 공개키를 서버 계정에 등록한다. 수동 등록 방법도 함께 익혀 둔다(도구가 없는 환경 대비).

```bash
# [macOS] 자동 등록 — 이 시점에는 아직 비밀번호 인증이 살아 있어야 함
ssh-copy-id -i ~/.ssh/id_ed25519.pub admin1@192.168.64.10

# 포트 변경 후라면
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 2222 admin1@192.168.64.10
```

- `-i <공개키>` : 등록할 공개키 지정 (**i**dentity) — **`.pub` 를 명시**하지 않으면 기본 키를 찾음
- `-p <포트>` : 서버 SSH 포트 (소문자)
- `-f` : 이미 등록됐는지 확인하지 않고 강제 추가 (**f**orce)
- 동작: 서버에 접속해 `~/.ssh` 를 700 으로 만들고 `authorized_keys` 에 **추가(append)** 한 뒤 600 으로 설정

**수동 등록 방법 (ssh-copy-id 가 없을 때)**

```bash
# [macOS] 방법 A — 파이프로 한 번에
cat ~/.ssh/id_ed25519.pub | ssh admin1@192.168.64.10 \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

# 방법 B — scp 로 옮긴 뒤 서버에서 붙이기
scp ~/.ssh/id_ed25519.pub admin1@192.168.64.10:/tmp/
# [서버]
mkdir -p ~/.ssh && chmod 700 ~/.ssh
cat /tmp/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
rm -f /tmp/id_ed25519.pub
restorecon -Rv ~/.ssh                     # SELinux 컨텍스트 복구 (중요)
```

**권한 요구사항 (StrictModes 검사 대상)**

| 대상 | 요구 권한 | 소유자 |
| --- | --- | --- |
| 홈 디렉터리 `~` | **그룹·기타 쓰기 불가** (`755`·`700` 등, `77x` 금지) | 본인 |
| `~/.ssh` | **700** (`drwx------`) | 본인 |
| `~/.ssh/authorized_keys` | **600** (`-rw-------`) | 본인 |
| `~/.ssh/id_*` (개인키) | **600** 또는 400 | 본인 |
| `~/.ssh/*.pub` (공개키) | 644 | 본인 |

- 권한이 과다하면 sshd 가 **조용히 키를 무시**하고 비밀번호를 다시 물음 → 원인 파악이 어려운 대표 사고
- SELinux 컨텍스트도 필요: `~/.ssh` 는 `ssh_home_t` — `restorecon -Rv ~/.ssh` 로 복구

**검증**

```bash
# [서버]
ls -ld ~ ~/.ssh
ls -l ~/.ssh/authorized_keys
wc -l ~/.ssh/authorized_keys                    # 등록된 키 개수
ssh-keygen -lf ~/.ssh/authorized_keys           # 등록된 키 지문
ls -Zd ~/.ssh                                   # SELinux 컨텍스트
# [macOS] 키만으로 접속되는지 (비밀번호 안 물으면 성공)
ssh -o PreferredAuthentications=publickey -o PasswordAuthentication=no admin1@192.168.64.10 'hostname; whoami'
```

```text
# [서버] ls -ld ~/.ssh; ls -l ~/.ssh/authorized_keys
drwx------. 2 admin1 admin1 ... /home/admin1/.ssh
-rw-------. 1 admin1 admin1 ... /home/admin1/.ssh/authorized_keys
# ssh-keygen -lf ~/.ssh/authorized_keys
256 SHA256:<지문> mac->srv01 (ED25519)
# ls -Zd ~/.ssh
unconfined_u:object_r:ssh_home_t:s0 /home/admin1/.ssh
# [macOS] 키 전용 접속 테스트
srv01.lab.local
admin1
```

- macOS 의 `ssh-keygen -lf ~/.ssh/id_ed25519.pub` 지문과 서버의 `ssh-keygen -lf ~/.ssh/authorized_keys` 지문이 **일치**해야 정상

> 📝 **시험 포인트**: 서버에 공개키를 등록하는 파일은 **`~/.ssh/authorized_keys`**(복수형 `keys`). 권한은 **`.ssh`=700, `authorized_keys`=600**. `ssh-copy-id` 는 이 과정을 자동화하는 명령.

### 8-3. ssh-agent 로 패스프레이즈 반복 입력 회피

> **상황**: 개인키에 패스프레이즈를 걸면 접속마다 물어본다. 에이전트에 한 번만 올려 두고 세션 동안 재사용한다.

```bash
# [macOS 또는 서버 셸]
eval "$(ssh-agent -s)"                 # 에이전트 기동 + 환경변수 설정
ssh-agent bash                         # 대안 — 에이전트가 감싼 새 셸 시작
ssh-add ~/.ssh/id_ed25519              # 키 등록 (패스프레이즈 1회 입력)
ssh-add -l                             # 등록된 키 목록 (지문)
ssh-add -L                             # 등록된 키 공개키 전문
ssh-add -t 3600 ~/.ssh/id_ed25519      # 1시간 후 자동 삭제
ssh-add -d ~/.ssh/id_ed25519           # 특정 키 제거
ssh-add -D                             # 전체 제거
ssh-agent -k                           # 에이전트 종료
```

- `ssh-agent -s` : Bourne 셸용 환경변수 출력 (`SSH_AUTH_SOCK`, `SSH_AGENT_PID`) → `eval` 로 적용
- `ssh-add -l` : 등록 키 지문 나열 (**l**ist)
- `ssh-add -L` : 공개키 원문 나열 (대문자 **L**)
- `ssh-add -t <초>` : 수명 제한 (**t**ime)
- `ssh-add -d` / `-D` : 개별 삭제 / 전체 삭제 (**d**elete)
- `ssh -A` : 에이전트 전달(agent forwarding) — ⚠️ 중간 서버가 신뢰 가능할 때만. 침해 시 키 사용 권한이 넘어감
- macOS 는 `ssh-add --apple-use-keychain` 으로 키체인에 저장 가능 (`~/.ssh/config` 에 `UseKeychain yes`)

**검증**

```bash
ssh-add -l
echo $SSH_AUTH_SOCK
ssh admin1@192.168.64.10 'echo agent-ok'     # 패스프레이즈를 다시 묻지 않으면 성공
```

```text
# ssh-add -l
256 SHA256:<지문> mac->srv01 (ED25519)
# echo $SSH_AUTH_SOCK
/private/tmp/com.apple.launchd.../Listeners
```

- `ssh-add -l` 이 `Could not open a connection to your authentication agent` 면 에이전트 미기동 → `eval "$(ssh-agent -s)"` 먼저

> 📝 **시험 포인트**: `ssh-agent` = 개인키를 메모리에 보관해 패스프레이즈 재입력을 없애는 데몬, `ssh-add` = 그 에이전트에 키를 등록하는 명령. 이 한 쌍의 역할 구분이 출제 포인트.

### 8-4. known_hosts 와 호스트키 검증

> **상황**: 첫 접속 시 나오는 지문 확인 프롬프트의 의미와, IP 를 재사용했을 때 나오는 경고의 처리 방법을 정리한다.

```bash
# [서버] 이 서버의 호스트키 지문 (기준값)
for k in /etc/ssh/ssh_host_*_key.pub; do ssh-keygen -lf "$k"; done

# [macOS] 접속 시 표시되는 지문과 대조
ssh admin1@192.168.64.10
# The authenticity of host '192.168.64.10 (...)' can't be established.
# ED25519 key fingerprint is SHA256:<지문>.
# Are you sure you want to continue connecting (yes/no/[fingerprint])? yes

cat ~/.ssh/known_hosts | head -2
ssh-keygen -F 192.168.64.10                    # known_hosts 에서 항목 찾기
ssh-keygen -R 192.168.64.10                    # 항목 제거 (호스트키 변경 시)
ssh-keygen -R '[192.168.64.10]:2222'           # 비표준 포트 항목 제거
ssh-keyscan -t ed25519 192.168.64.10           # 서버 호스트키 미리 수집
```

- `ssh-keygen -F <호스트>` : known_hosts 에서 항목 검색 (**F**ind)
- `ssh-keygen -R <호스트>` : 항목 제거 (**R**emove) — 호스트키 변경 경고 해소
- `ssh-keyscan` : 접속 없이 서버의 공개 호스트키 수집 → `known_hosts` 사전 등록에 사용
- 비표준 포트는 known_hosts 에 **`[호스트]:포트`** 형식으로 기록됨

**호스트키 변경 경고**

```text
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
...
Offending ECDSA key in /Users/.../.ssh/known_hosts:<줄번호>
```

- 정당한 원인: **서버 재설치**, OS 재구축, 같은 IP 를 다른 장비가 재사용, 호스트키 재생성
- 부당한 원인: **중간자 공격(MITM)** — 지문이 왜 바뀌었는지 확인 없이 지우면 안 됨
- 처리 순서: ① 서버 콘솔에서 `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` 로 실제 지문 확인 → ② 화면에 뜬 지문과 대조 → ③ 일치하면 `ssh-keygen -R <호스트>` 후 재접속

**검증**

```bash
# [서버]
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
# [macOS]
ssh-keygen -F 192.168.64.10 | ssh-keygen -lf -      # known_hosts 항목의 지문
```

```text
# [서버] ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
256 SHA256:<지문> root@srv01.lab.local (ED25519)
# [macOS] 대조 결과 동일 지문 → 정상
```

- ⚠️ `-o StrictHostKeyChecking=no` 는 이 검증을 통째로 건너뜀 → **중간자 공격 탐지 불가**. 신뢰 가능한 사내 자동화에 한정
- 대안: `StrictHostKeyChecking=accept-new` — 처음 보는 호스트만 자동 수락, **변경된 키는 여전히 거부**

> 📝 **시험 포인트**: `~/.ssh/known_hosts` = 접속했던 **서버의 공개 호스트키** 저장소. 서버 재설치 후 경고가 나오는 이유와 `ssh-keygen -R <호스트>` 해소법이 서술형 소재. `authorized_keys`(서버가 보관하는 클라이언트 공개키)와 **방향이 반대**임을 구분.

### 8-5. 비밀번호 인증·root 로그인 차단

> **상황**: 키 인증이 확실히 되는 것을 확인했으니 비밀번호 로그인을 막는다. 순서를 어기면 스스로 잠긴다.

⚠️ **전제 조건 3가지를 모두 확인한 뒤에만 진행**
1. `ssh -o PasswordAuthentication=no admin1@192.168.64.10` 이 **비밀번호 없이 성공**
2. `admin1` 이 `wheel` 그룹에 있어 `sudo` 가능 (`id admin1`)
3. **UTM 콘솔 로그인 가능** (잠겼을 때의 유일한 복구 경로)

```bash
id admin1 | grep -o 'wheel'                         # ① wheel 확인
cp /etc/ssh/sshd_config /root/net-backup/sshd_config.beforekey

# ② 설정 변경 — 드롭인 파일 방식 (Include 우선순위 문제를 피함)
cat > /etc/ssh/sshd_config.d/60-lab-hardening.conf <<'EOF'
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
MaxAuthTries 3
LoginGraceTime 60
ClientAliveInterval 300
ClientAliveCountMax 3
X11Forwarding no
EOF
chmod 600 /etc/ssh/sshd_config.d/60-lab-hardening.conf

# ③ 문법 검사 → 실효값 확인 → 반영
sshd -t && echo "문법 OK"
sshd -T | grep -E '^(passwordauthentication|permitrootlogin|pubkeyauthentication|maxauthtries)'
systemctl reload sshd
```

- 드롭인(`/etc/ssh/sshd_config.d/*.conf`) 방식의 장점: 원본 파일 무손상, 되돌리려면 파일 하나만 삭제
- ※ 본문 `/etc/ssh/sshd_config` 를 직접 고쳐도 되지만, **`Include` 가 위에 있으면 드롭인이 이김** → 반드시 `sshd -T` 로 실효값 확인
- `reload` : 설정만 다시 읽음 — **기존 SSH 세션 유지**. `restart` 는 데몬 재시작(포트 변경 시 필요)

**검증**

```bash
sshd -T | grep -E '^passwordauthentication|^permitrootlogin'
# [macOS] 키로는 성공
ssh admin1@192.168.64.10 'echo key-login-ok'
# [macOS] 비밀번호 인증만 강제 → 거절되어야 정상
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password admin1@192.168.64.10
# [macOS] root 직접 로그인 → 거절되어야 정상
ssh root@192.168.64.10
# [서버] 로그로 확인
journalctl -u sshd -n 30 --no-pager | grep -Ei 'accepted|failed|denied|publickey'
grep -Ei 'sshd.*(Accepted|Failed)' /var/log/secure | tail -5
```

```text
# sshd -T | grep -E '^passwordauthentication|^permitrootlogin'
passwordauthentication no
permitrootlogin no
# [macOS] 비밀번호 강제 접속 시도
admin1@192.168.64.10: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
# [서버] journalctl -u sshd
... sshd[...]: Accepted publickey for admin1 from 192.168.64.1 port ... ssh2: ED25519 SHA256:<지문>
... sshd[...]: Connection closed by authenticating user root 192.168.64.1 port ... [preauth]
```

- `Accepted publickey ... SHA256:<지문>` 로그의 지문이 등록한 키의 지문과 일치 → 어떤 키로 들어왔는지 추적 가능 (`LogLevel VERBOSE` 시 더 상세)
- 되돌리기: `rm /etc/ssh/sshd_config.d/60-lab-hardening.conf && sshd -t && systemctl reload sshd`

> 📝 **시험 포인트**: `PasswordAuthentication no` + `PermitRootLogin no` 조합의 효과 = "**root 직접 로그인 차단 + 모든 계정 비밀번호 로그인 차단, 공개키 인증만 허용**". 적용 명령은 `systemctl reload sshd`(또는 restart). SSH 인증 로그는 `/var/log/secure`(facility `AUTHPRIV`) — Part 07 연계.

---

## 9. SSH 포트 2222 로 변경

### 9-1. 순서가 중요한 이유

> **상황**: 포트만 바꾸고 재시작하면 SELinux 가 sshd 의 2222 바인딩을 막고, 통과하더라도 방화벽이 접속을 막는다. 세 가지를 정해진 순서로 처리해야 한다.

**필수 순서**

| 단계 | 작업 | 명령 | 빠뜨렸을 때의 증상 |
| --- | --- | --- | --- |
| ① | 설정 파일에 `Port 2222` | `sshd_config` 편집 → `sshd -t` | — |
| ② | **SELinux 포트 레이블 추가** | `semanage port -a -t ssh_port_t -p tcp 2222` | `systemctl restart sshd` **실패** — `Permission denied` / `Cannot bind any address` |
| ③ | **방화벽 포트 개방** | `firewall-cmd --permanent --add-port=2222/tcp` + `--reload` | 데몬은 뜨지만 **외부 접속 타임아웃** (`ss` 에는 LISTEN 보임) |
| ④ | 데몬 재시작 | `systemctl restart sshd` | — |
| ⑤ | 검증 | `ss -tlnp`, macOS 에서 `ssh -p 2222` | — |

- ②③ 를 ④ **앞에** 해야 하는 이유: 재시작이 실패하면 기존 sshd 도 내려가 원격 복구 수단이 사라짐
- 22 번을 당분간 함께 열어 두는 "안전 이행" 방식도 가능 (`Port 22` + `Port 2222` 두 줄 → 검증 후 22 제거)

> 📝 **시험 포인트**: SSH 포트 변경 3종 세트 = **sshd_config + SELinux(semanage) + 방화벽(firewall-cmd)**. SELinux 를 빼먹으면 **서비스가 아예 안 뜬다**는 점이 실기 서술형 정답 요소.

### 9-2. sshd_config 에 Port 2222 지정

> **상황**: 설정을 바꾸되 아직 재시작하지 않는다. 안전을 위해 22 와 2222 를 잠시 함께 연다.

⚠️ 이 절 전체는 **UTM 콘솔을 열어 둔 상태**에서 진행할 것.

```bash
cp /etc/ssh/sshd_config /root/net-backup/sshd_config.beforeport

# 안전 이행 — 두 포트 동시 개방
cat >> /etc/ssh/sshd_config.d/60-lab-hardening.conf <<'EOF'
Port 22
Port 2222
EOF

sshd -t && echo "문법 OK"
sshd -T | grep '^port'          # port 22 / port 2222 두 줄
```

- `Port` 를 여러 줄 쓰면 **모든 포트에서 수신** — 이행 기간용
- 검증 완료 후 `Port 22` 줄을 제거하면 2222 단독이 됨

**검증**

```bash
sshd -T | grep '^port'
grep -n 'Port' /etc/ssh/sshd_config.d/60-lab-hardening.conf
```

```text
# sshd -T | grep '^port'
port 22
port 2222
```

> 📝 **시험 포인트**: `sshd_config` 의 `Port` 지시자는 여러 줄 지정 가능. 포트 변경 실기 문제에서 "기존 포트를 함께 열어 두고 검증 후 제거" 하는 절차를 쓰면 실무 감점을 피함.

### 9-3. SELinux 포트 레이블 추가 (반드시 먼저)

> **상황**: SELinux 는 `sshd` 프로세스(`sshd_t`)가 `ssh_port_t` 레이블이 붙은 포트에만 바인딩하도록 강제한다. 2222 에 그 레이블을 붙인다.

```bash
getenforce                                          # Enforcing 확인
rpm -q policycoreutils-python-utils || dnf install -y policycoreutils-python-utils

semanage port -l | grep -w ssh_port_t               # 현재 레이블된 포트 (22)
semanage port -a -t ssh_port_t -p tcp 2222          # 2222 추가
semanage port -l | grep -w ssh_port_t               # 재확인 (22, 2222)
```

- `semanage port -l` : 포트 레이블 전체 목록 (**l**ist)
- `-a` : 추가 (**a**dd)
- `-m` : 수정 (**m**odify) — 이미 다른 타입에 등록된 포트일 때 `-a` 대신 사용
- `-d` : 삭제 (**d**elete) — 원복 시 `semanage port -d -t ssh_port_t -p tcp 2222`
- `-t <타입>` : 부여할 SELinux 타입 (**t**ype)
- `-p <프로토콜>` : `tcp` / `udp` (**p**rotocol)
- `policycoreutils-python-utils` 패키지가 `semanage` 제공 — minimal 설치에는 없음

**검증**

```bash
semanage port -l | grep -w ssh_port_t
```

```text
# semanage port -l | grep -w ssh_port_t   (추가 전)
ssh_port_t                     tcp      22
# semanage port -l | grep -w ssh_port_t   (추가 후)
ssh_port_t                     tcp      2222, 22
```

- SELinux 를 빠뜨렸을 때의 증상과 확인법

```bash
systemctl status sshd --no-pager            # Failed to start / bind: Permission denied
journalctl -u sshd -n 20 --no-pager | grep -i 'bind\|permission'
ausearch -m avc -ts recent | tail -20       # AVC 거부 기록 (audit 패키지)
ausearch -m avc -ts recent | grep -i ssh
sealert -a /var/log/audit/audit.log | head  # setroubleshoot-server 설치 시 해설 제공
```

- `ausearch -m avc -ts recent` : 최근 AVC(Access Vector Cache) 거부 기록 조회
  - `-m avc` : 메시지 타입 (**m**essage)
  - `-ts recent` : 최근 10분 (**t**ime **s**tart) — `today`, `boot` 도 가능
- 전형적 AVC: `avc: denied { name_bind } for pid=... comm="sshd" src=2222 scontext=system_u:system_r:sshd_t:... tclass=tcp_socket`
- SELinux 상세는 [[10-security-firewall-selinux]]

> 📝 **시험 포인트**: 비표준 포트로 서비스를 옮길 때 SELinux 조치는 **`semanage port -a -t <타입>_port_t -p tcp <포트>`**. httpd 는 `http_port_t`, sshd 는 `ssh_port_t`. `setenforce 0` 으로 넘기는 것은 임시방편이며 감점 요소.

### 9-4. 방화벽 포트 개방

> **상황**: SELinux 를 통과해 데몬이 2222 에서 듣더라도, firewalld 가 막으면 밖에서는 타임아웃이 난다. 미리 열어 둔다.

```bash
systemctl is-active firewalld
firewall-cmd --state
firewall-cmd --list-all                                  # 현재 허용 목록

firewall-cmd --permanent --add-port=2222/tcp             # 영구 규칙 추가
firewall-cmd --reload                                    # 영구 규칙을 런타임에 반영
firewall-cmd --list-ports                                # 확인
firewall-cmd --list-services                             # ssh 서비스(22) 도 아직 열려 있음
```

- `--permanent` : 영구 규칙 — **재부팅 후에도 유지**되지만 **즉시 반영되지 않음**
- `--reload` : 영구 규칙을 런타임에 적용 (기존 연결은 유지)
- `--add-port=<포트>/<프로토콜>` : 포트 단위 허용
- `--add-service=<이름>` : 서비스 단위 허용 (`/usr/lib/firewalld/services/*.xml` 정의)
- `--list-all` / `--list-ports` / `--list-services` : 현재 존 설정 조회
- 상세(존·리치룰·마스커레이드·`--runtime-to-permanent`)는 [[10-security-firewall-selinux]]

**검증**

```bash
firewall-cmd --list-ports
firewall-cmd --permanent --list-ports        # 영구 규칙에도 반영됐는지
firewall-cmd --list-all | grep -E 'ports|services'
```

```text
# firewall-cmd --list-ports
2222/tcp
# firewall-cmd --list-all
public (active)
  ...
  services: cockpit dhcpv6-client ssh
  ports: 2222/tcp
```

- ⚠️ `--permanent` 만 하고 `--reload` 를 빼먹으면 **재부팅 전까지 적용 안 됨** — 가장 흔한 실수

> 📝 **시험 포인트**: `firewall-cmd --permanent --add-port=2222/tcp` **+ `firewall-cmd --reload`** 2단계가 정답. `--permanent` 없이 추가하면 재부팅 시 소실, `--reload` 없으면 즉시 미반영 — 이 두 함정이 실기 빈칸으로 반복 출제.

### 9-5. 재시작과 검증

> **상황**: 준비가 끝났으니 데몬을 재시작하고 새 포트로 접속되는지 확인한다.

⚠️ `reload` 가 아니라 **`restart`** 가 필요하다 (리스닝 포트 변경은 소켓 재바인딩을 요구). 기존 SSH 세션은 유지되지만 **새 연결은 재시작 성공 여부에 달려 있다.**

```bash
sshd -t && echo "문법 OK"                    # 최종 문법 확인
systemctl restart sshd
systemctl status sshd --no-pager | head -5
ss -tlnp | grep sshd                         # 22, 2222 둘 다 LISTEN
```

**검증** — 서버 측

```bash
ss -tlnp | grep -E ':22\b|:2222'
nmap -sT -p 22,2222 127.0.0.1 | grep -E '^(22|2222)/tcp'
journalctl -u sshd -n 10 --no-pager | grep -i 'server listening'
```

```text
# ss -tlnp | grep sshd
tcp LISTEN 0 128 0.0.0.0:22    0.0.0.0:* users:(("sshd",pid=...,fd=...))
tcp LISTEN 0 128 0.0.0.0:2222  0.0.0.0:* users:(("sshd",pid=...,fd=...))
# journalctl -u sshd | grep -i listening
... sshd[...]: Server listening on 0.0.0.0 port 22.
... sshd[...]: Server listening on 0.0.0.0 port 2222.
```

**검증** — macOS 클라이언트 측

```bash
# [macOS]
ssh -p 2222 admin1@192.168.64.10 'hostname; ss -tlnp | grep sshd | wc -l'
nc -zv -w 3 192.168.64.10 2222
nc -zv -w 3 192.168.64.10 22
```

**22 번 폐쇄 (이행 완료)**

```bash
# [서버] Port 22 줄 제거
sed -i '/^Port 22$/d' /etc/ssh/sshd_config.d/60-lab-hardening.conf
sshd -t && sshd -T | grep '^port'            # port 2222 만
systemctl restart sshd
ss -tlnp | grep sshd                          # 2222 만 LISTEN

firewall-cmd --permanent --remove-service=ssh    # 22 허용 제거
firewall-cmd --reload
firewall-cmd --list-all | grep -E 'services|ports'
```

**최종 검증** — 22 실패 / 2222 성공

```bash
# [macOS]
ssh -p 2222 admin1@192.168.64.10 'echo "2222 OK"'
timeout 5 ssh -p 22 admin1@192.168.64.10 'echo should-fail' ; echo "종료코드=$?"
nc -zv -w 3 192.168.64.10 22
nc -zv -w 3 192.168.64.10 2222
```

```text
# [macOS] ssh -p 2222 ...
2222 OK
# [macOS] ssh -p 22 ...
ssh: connect to host 192.168.64.10 port 22: Connection refused      ← 방화벽 제거 + 리스너 없음
종료코드=255
# [macOS] nc -zv -w 3 192.168.64.10 2222
Connection to 192.168.64.10 port 2222 [tcp/*] succeeded!
```

- `~/.ssh/config` 에 `Port 2222` 를 넣어 뒀다면(7-3) 이후로는 `ssh srv01` 만으로 접속

> 📝 **시험 포인트**: 포트 변경 후 반드시 **`restart`** (reload 로는 리스닝 포트가 안 바뀜). 검증은 `ss -tlnp | grep sshd` 로 서버 측, `ssh -p 2222` 로 클라이언트 측 양쪽. `scp` 는 대문자 **`-P`**, `ssh` 는 소문자 `-p` 로 포트를 지정하는 차이가 함정.

### 9-6. 실패 증상별 원인과 콘솔 복구 절차

> **상황**: 실패 지점을 증상으로 역추적할 수 있어야 한다. 잠겼을 때의 복구 경로도 정리한다.

**증상 → 원인 판정표**

| 증상 | 확인 명령 | 원인 | 조치 |
| --- | --- | --- | --- |
| `systemctl restart sshd` 실패, `bind: Permission denied` | `journalctl -u sshd -n 20`, `ausearch -m avc -ts recent` | **SELinux 포트 레이블 없음** | `semanage port -a -t ssh_port_t -p tcp 2222` |
| 데몬은 active 인데 외부 접속 **타임아웃** | 서버: `ss -tlnp \| grep 2222`(LISTEN 있음) / 밖: `nc -zv` 타임아웃 | **방화벽 미개방** | `firewall-cmd --permanent --add-port=2222/tcp; firewall-cmd --reload` |
| 외부에서 `Connection refused` | `ss -tlnp \| grep 2222` (출력 없음) | **리스너 없음** — 설정 미반영 또는 데몬 정지 | `sshd -T \| grep ^port` 확인 → `systemctl restart sshd` |
| `Permission denied (publickey)` | `journalctl -u sshd`, `ls -ld ~/.ssh` | 키 미등록 또는 **권한 과다** | `chmod 700 ~/.ssh; chmod 600 ~/.ssh/authorized_keys; restorecon -Rv ~/.ssh` |
| `sshd` 재시작 시 문법 오류 | `sshd -t` | 설정 오타 | 오류 줄 수정 후 재검사 |
| `No route to host` | `ip route`, `ping` | 라우팅·대역 문제 | `ip route show default` |
| 접속은 되는데 매우 느림 | `sshd -T \| grep usedns` | `UseDNS yes` + DNS 지연 | `UseDNS no` |

**잠겼을 때 UTM 콘솔 복구 절차**

```text
① UTM 디스플레이 창 클릭 → 콘솔 로그인 (root 또는 admin1 + su -)
② 원인 확인
     systemctl status sshd
     journalctl -u sshd -n 30 --no-pager
     sshd -t
     ss -tlnp | grep sshd
     ausearch -m avc -ts recent | tail
③ 즉시 복구 (원상 복귀)
     rm -f /etc/ssh/sshd_config.d/60-lab-hardening.conf
     cp /root/net-backup/sshd_config.orig /etc/ssh/sshd_config
     sshd -t && systemctl restart sshd
     firewall-cmd --permanent --add-service=ssh && firewall-cmd --reload
     ss -tlnp | grep ':22'
④ 원인을 고친 뒤 9-2 부터 다시 진행
```

- 네트워크까지 죽어 콘솔조차 답답하면 Part 07 의 `rd.break`·rescue 절차 사용 → [[07-boot-systemd-log]]
- 스냅샷(QEMU 백엔드)이 있으면 되돌리기가 가장 빠름

**검증** (복구 후)

```bash
systemctl is-active sshd
ss -tlnp | grep sshd
sshd -T | grep '^port'
firewall-cmd --list-all | grep -E 'services|ports'
```

> 📝 **시험 포인트**: "SSH 포트를 바꿨더니 서비스가 안 뜬다" → **SELinux**, "서비스는 떴는데 접속이 안 된다" → **방화벽**. 이 2단 구분이 장애 진단 서술형의 채점 기준. `ausearch -m avc -ts recent` 로 SELinux 거부 확인.

---

## 10. SSH 활용 — 원격 실행·포워딩·파일 전송

### 10-1. 원격 단일 명령과 의사 터미널

> **상황**: 접속해서 명령 하나만 치고 나오는 패턴은 자동화의 기본이다. 대화형이 필요한 명령은 `-t` 가 필요하다는 점도 확인한다.

```bash
# [macOS] — 이하 모든 예에서 포트는 2222
ssh -p 2222 admin1@192.168.64.10 'hostname; uptime'          # 단일 명령
ssh -p 2222 admin1@192.168.64.10 'df -hT | grep -v tmpfs'    # 파이프는 따옴표 안에
ssh -p 2222 admin1@192.168.64.10 'ss -tlnp' | grep sshd      # 따옴표 밖 파이프 = 로컬 처리
ssh -p 2222 -t admin1@192.168.64.10 'sudo journalctl -u sshd -n 5'   # sudo 비번 입력 필요 → -t
ssh -p 2222 -t admin1@192.168.64.10 'top -b -n 1 | head'
ssh -p 2222 admin1@192.168.64.10 'cat /etc/os-release' > /tmp/remote-os.txt   # 출력 로컬 저장
cat /etc/hosts | ssh -p 2222 admin1@192.168.64.10 'cat > /tmp/hosts.copy'      # 로컬 → 원격 스트림
```

- `-t` : **의사 터미널(pseudo-tty) 강제 할당** — `sudo` 비밀번호 입력, `vi`·`top` 등 화면 제어 명령에 필수
- `-T` : 터미널 할당 안 함 (대문자, 순수 파이프용)
- `-t -t` (`-tt`) : 표준 입력이 터미널이 아니어도 강제 할당
- 원격 명령은 **작은따옴표**로 감싸 로컬 셸의 변수·와일드카드 확장을 막을 것
- 파이프 위치의 차이: `ssh host 'cmd | grep x'` = 원격에서 필터, `ssh host 'cmd' | grep x` = 로컬에서 필터

**검증**

```bash
ssh -p 2222 admin1@192.168.64.10 'hostname -f'
ssh -p 2222 admin1@192.168.64.10 'whoami; id -nG'
diff <(cat /etc/hosts) <(ssh -p 2222 admin1@192.168.64.10 'cat /tmp/hosts.copy') && echo "스트림 전송 OK"
```

```text
# ssh -p 2222 admin1@192.168.64.10 'hostname -f'
srv01.lab.local
# ssh -t ... 'sudo journalctl -u sshd -n 5'   (-t 없이 실행 시)
sudo: a terminal is required to read the password; ...
```

> 📝 **시험 포인트**: `ssh <user>@<host> '<명령>'` 형식으로 원격 단일 명령 실행. `-t` 는 의사 터미널 할당 — `sudo`·`vi` 실행 시 필요. `ssh` 세션을 끊어도 계속 돌게 하려면 `nohup`·`screen`·`tmux`·`systemd-run --scope` 사용(기출 소재).

### 10-2. X11 포워딩 (※ 미실행 — X 미설치)

> **상황**: 원격 GUI 도구를 로컬 화면에 띄우는 기능이다. 이 서버는 X 윈도가 없어 실행할 수 없으므로 개념과 설정 위치만 정리한다.

```bash
ssh -X -p 2222 admin1@192.168.64.10          # X11 포워딩 활성 (신뢰 모드 아님)
ssh -Y -p 2222 admin1@192.168.64.10          # 신뢰(trusted) X11 포워딩 — 제약 적음, 보안 낮음
ssh -x -p 2222 admin1@192.168.64.10          # 포워딩 비활성 (소문자 x)
# [원격 접속 후] echo $DISPLAY; xauth list; xclock
```

- ※ **미실행**: 이 VM 은 X 서버·X 클라이언트(`xorg-x11-*`, `xauth`)가 설치되지 않아 실제 창이 뜨지 않음. `echo $DISPLAY` 도 비어 있음
- 동작 구조
  1. 서버의 `sshd_config` 에 `X11Forwarding yes` 필요 (RHEL 기본 yes, 8-5 에서 `no` 로 바꿈)
  2. 서버에 `xauth` 설치 필요 (`xorg-x11-xauth`) — 없으면 `DISPLAY` 가 설정되지 않음
  3. 접속하면 sshd 가 원격 셸에 **`DISPLAY=localhost:10.0`** 같은 값을 설정하고, X 트래픽을 SSH 터널로 로컬 X 서버에 전달
  4. `~/.Xauthority` 의 **MIT-MAGIC-COOKIE** 로 접근 인증
- 용어 방향 주의: **X 클라이언트**(원격에서 실행되는 GUI 프로그램) → **X 서버**(로컬의 화면·키보드·마우스)
- macOS 에서 쓰려면 **XQuartz** 설치 필요
- 서버에서 X 를 쓰지 않으면 `X11Forwarding no` 가 보안상 권장

> 📝 **시험 포인트**: X11 의 클라이언트/서버 방향이 직관과 반대 — **사용자 앞의 화면이 X 서버**. 원격 GUI 실행은 `ssh -X`, 환경변수는 `DISPLAY`, 인증 쿠키 관리는 `xauth`. 필기 순서 나열 문제로 출제.

### 10-3. 포트 포워딩 — 로컬·원격·동적

> **상황**: 방화벽 너머의 서비스에 SSH 터널로 접근하는 방법이다. Part 09 에서 httpd 를 올린 뒤 실제 검증할 수 있도록 명령 형식과 방향을 정리한다.

```bash
# ① 로컬 포워딩 (-L) — 로컬 포트 → SSH 서버가 대신 접속
ssh -p 2222 -L 8080:127.0.0.1:80 admin1@192.168.64.10
#   [macOS] localhost:8080  →  (SSH 터널)  →  [서버] 127.0.0.1:80
#   ※ Part 09 에서 httpd 기동 후 검증: 다른 macOS 터미널에서 curl -I http://127.0.0.1:8080

# ② 원격 포워딩 (-R) — 원격 포트 → 로컬(또는 로컬이 닿는 곳)으로
ssh -p 2222 -R 9090:127.0.0.1:8000 admin1@192.168.64.10
#   [서버] localhost:9090  →  (SSH 터널)  →  [macOS] 127.0.0.1:8000
#   ※ 서버 밖에서도 접근하려면 sshd_config 의 GatewayPorts yes 필요

# ③ 동적 포워딩 (-D) — SOCKS5 프록시
ssh -p 2222 -D 1080 admin1@192.168.64.10
#   [macOS] SOCKS5 프록시 127.0.0.1:1080 → 서버를 경유해 임의 목적지
curl --socks5-hostname 127.0.0.1:1080 https://rockylinux.org -o /dev/null -w '%{http_code}\n'

# ④ 터널만 (셸 없이 백그라운드)
ssh -p 2222 -N -f -L 8080:127.0.0.1:80 admin1@192.168.64.10
ps aux | grep '[s]sh -p 2222 -N'
pkill -f 'ssh -p 2222 -N'            # 종료
```

- `-L <로컬포트>:<목적호스트>:<목적포트>` : **L**ocal 포워딩 — 접속을 **내가 시작**, 트래픽은 SSH 서버가 대리 연결
- `-R <원격포트>:<목적호스트>:<목적포트>` : **R**emote 포워딩 — 접속을 **원격이 시작**, 트래픽은 내 쪽으로
- `-D <포트>` : **D**ynamic 포워딩 — SOCKS4/5 프록시, 목적지를 미리 정하지 않음
- `-N` : 원격 명령 실행 안 함 (**N**o command) — 터널 전용
- `-f` : 인증 후 백그라운드로 (**f**ork)
- `-g` : 다른 호스트도 이 포워딩 포트에 접속 허용 (**g**ateway) — ⚠️ 노출 주의
- 방향 암기: **`-L` 은 로컬로 끌어온다(pull), `-R` 은 원격에 밀어 넣는다(push)**

**검증**

```bash
# [macOS] 터널 기동 후
ss -tln 2>/dev/null | grep 8080 || netstat -an | grep 8080 | grep LISTEN   # macOS 는 netstat
lsof -nP -iTCP:8080 -sTCP:LISTEN                       # macOS 에서 리스너 확인
curl -sI http://127.0.0.1:8080 | head -1               # ※ Part 09 httpd 기동 후 200 확인
```

```text
# [macOS] lsof -nP -iTCP:8080 -sTCP:LISTEN
ssh   ...  TCP 127.0.0.1:8080 (LISTEN)
# ※ Part 09 에서 httpd 기동 후
HTTP/1.1 200 OK
```

- ※ 현재는 서버에 웹 서비스가 없어 `Connection refused` 가 정상 — 실검증은 [[09-network-services]] 진행 후

> 📝 **시험 포인트**: `ssh -L 8080:intranet.example.com:80 admin@gw` 해석 = "**내 8080 접속을 gw 를 거쳐 intranet.example.com:80 으로 전달**"(로컬 포워딩). `-L`/`-R`/`-D` 3종 방향 구분이 필기 최빈출. `-N -f` = 셸 없이 백그라운드 터널.

### 10-4. 점프 호스트·디버깅·호스트키 옵션

> **상황**: 중간 서버를 거쳐 접속하거나 접속 실패 원인을 파고들 때 쓰는 옵션들이다.

```bash
# 점프 호스트 (bastion 경유)
ssh -J admin1@192.168.64.10:2222 dev1@10.0.0.5                 # 신형 (OpenSSH 7.3+)
ssh -o ProxyJump=admin1@192.168.64.10:2222 dev1@10.0.0.5       # 동일
ssh -J bastion1,bastion2 target                                 # 다단 경유
# ※ 이 실습망에는 2단 서버가 없어 형식만 — 실제 실행은 미수행

# 접속 디버깅
ssh -v   -p 2222 admin1@192.168.64.10 exit          # 1단계 상세
ssh -vvv -p 2222 admin1@192.168.64.10 exit 2>&1 | grep -iE 'auth|key|offer|debug1: (Connecting|Authenticat)'

# 옵션 임시 지정
ssh -o ConnectTimeout=5 -p 2222 admin1@192.168.64.10 'echo ok'
ssh -o BatchMode=yes -p 2222 admin1@192.168.64.10 'echo ok'      # 대화 프롬프트 금지(스크립트용)
ssh -o StrictHostKeyChecking=accept-new -p 2222 admin1@192.168.64.10 'echo ok'
ssh -G srv01 | head                                              # 실효 클라이언트 설정
```

- `-J <점프호스트>` : 점프(bastion) 경유 (**J**ump) — 중간 서버에 파일을 남기지 않음
- `-v` / `-vv` / `-vvv` : 디버그 상세도 — `debug1:`/`debug2:`/`debug3:` 접두어
- `-o <옵션>=<값>` : `ssh_config` 지시자를 명령행에서 지정
  - `ConnectTimeout=<초>` : 연결 시도 제한
  - `BatchMode=yes` : 비밀번호·확인 프롬프트를 띄우지 않고 즉시 실패 → **자동화 필수**
  - `StrictHostKeyChecking=accept-new` : 새 호스트만 자동 수락, 변경된 키는 거부 (권장 타협점)
- ⚠️ `-o StrictHostKeyChecking=no` 의 위험
  - 호스트키 검증을 통째로 생략 → **중간자 공격(MITM) 탐지 불가**
  - `known_hosts` 에 무엇이 들어오든 무조건 수락 → 서버가 바뀌어도 경고 없음
  - 신뢰 가능한 사내 자동화에 한정, 공용망·인터넷 경유 접속에는 사용 금지
  - 대안: `accept-new` 사용, 또는 `ssh-keyscan` 으로 사전에 `known_hosts` 를 채워 둘 것

**검증**

```bash
ssh -v -p 2222 admin1@192.168.64.10 exit 2>&1 | grep -E 'Authenticated|Offering|Server accepts'
ssh -o BatchMode=yes -o ConnectTimeout=3 -p 2222 admin1@192.168.64.10 'echo batch-ok'
```

```text
# ssh -v ... | grep
debug1: Offering public key: /Users/.../.ssh/id_ed25519 ED25519 SHA256:<지문>
debug1: Server accepts key: /Users/.../.ssh/id_ed25519 ED25519 SHA256:<지문>
Authenticated to 192.168.64.10 ([192.168.64.10]:2222) using "publickey".
```

> 📝 **시험 포인트**: 접속 실패 원인 판정 — `Connection timed out`(방화벽·경로), `Connection refused`(데몬 정지), `Permission denied (publickey)`(키·권한), `No route to host`(라우팅). `ssh -v` 출력으로 어느 단계에서 멈췄는지 특정하는 것이 서술형 답안.

### 10-5. scp — SSH 채널 파일 복사

> **상황**: macOS ↔ VM 파일 전송을 양방향으로 실측한다. 포트 옵션의 대소문자 함정을 반드시 몸에 익힌다.

```bash
# [서버] 전송용 자료 준비 (/srv/share 는 Part 05 의 LV 마운트)
mkdir -p /srv/share/lab08
for i in 1 2 3; do dd if=/dev/urandom of=/srv/share/lab08/f$i.bin bs=1K count=64 status=none; done
echo "lab08 scp test $(date)" > /srv/share/lab08/note.txt
sha256sum /srv/share/lab08/* > /srv/share/lab08/SHA256SUMS
chown -R admin1:admin1 /srv/share/lab08
ls -l /srv/share/lab08/
```

```bash
# [macOS] 원격 → 로컬 (다운로드)
scp -P 2222 admin1@192.168.64.10:/srv/share/lab08/note.txt /tmp/
scp -P 2222 -r admin1@192.168.64.10:/srv/share/lab08 /tmp/lab08-down

# [macOS] 로컬 → 원격 (업로드)
echo "from macOS $(date)" > /tmp/from-mac.txt
scp -P 2222 /tmp/from-mac.txt admin1@192.168.64.10:/tmp/
scp -P 2222 -r /tmp/lab08-down admin1@192.168.64.10:/tmp/lab08-back

# 옵션 조합
scp -P 2222 -p -C -v /tmp/from-mac.txt admin1@192.168.64.10:/tmp/from-mac2.txt
scp -P 2222 -i ~/.ssh/id_ed25519 /tmp/from-mac.txt admin1@192.168.64.10:/tmp/
```

- `-P <포트>` : **대문자 P** — 원격 SSH 포트. ⚠️ `ssh` 는 소문자 `-p`, `scp` 는 **대문자 `-P`**
- `-r` : 디렉터리 재귀 복사 (**r**ecursive)
- `-p` : **소문자 p** — 원본의 수정 시각·권한 보존 (**p**reserve)
- `-C` : 전송 중 압축 (**C**ompression)
- `-i <키>` : 개인키 지정 (**i**dentity)
- `-v` : 상세 출력
- `-3` : 원격↔원격 복사를 로컬을 경유해 수행
- `-q` : 진행률 표시 안 함 (**q**uiet)
- 특이사항
  - `scp` 는 **접속 사용자 권한으로** 원격 파일을 읽음 → `/root` 등 권한 밖 경로는 실패
    - 우회: `ssh -p 2222 admin1@192.168.64.10 "sudo cat /root/파일" > 로컬파일`
  - **명령 실행 위치 확인 필수** — SSH 로 서버에 들어간 창에서 `scp` 를 치면 서버가 자기 자신에 접속을 시도해 실패. 반드시 **로컬(macOS) 셸**에서 실행

**검증**

```bash
# [macOS]
ls -l /tmp/note.txt /tmp/lab08-down/
shasum -a 256 /tmp/lab08-down/f1.bin           # macOS 는 shasum
# [서버]
sha256sum /srv/share/lab08/f1.bin
ls -l /tmp/from-mac.txt /tmp/lab08-back/
cd /tmp/lab08-back && sha256sum -c SHA256SUMS 2>/dev/null | tail -5
```

```text
# [서버] cd /tmp/lab08-back && sha256sum -c SHA256SUMS
f1.bin: OK
f2.bin: OK
f3.bin: OK
note.txt: OK
```

- 전송 후 **무결성 검증 필수** — `sha256sum`(Linux) ↔ `shasum -a 256`(macOS) 값 대조 (Part 04 참조)

> 📝 **시험 포인트**: `scp -P <포트>`(대문자) vs `ssh -p <포트>`(소문자) 대소문자 함정이 최빈출. `-r`(재귀), `-p`(속성 보존), `-C`(압축). 문법은 `scp [옵션] <출발> <도착>` 이며 원격은 `user@host:/경로` 형식.

### 10-6. sftp — 대화형 파일 전송

> **상황**: `scp` 와 달리 원격 디렉터리를 돌아다니며 골라 받을 수 있다. FTP 명령어와 같은 조작 체계라 시험에도 나온다.

```bash
# [macOS]
sftp -P 2222 admin1@192.168.64.10
sftp -P 2222 -i ~/.ssh/id_ed25519 admin1@192.168.64.10
sftp -P 2222 -b /tmp/sftp-batch.txt admin1@192.168.64.10    # 배치 모드 (비대화형)
```

- `-P <포트>` : **대문자 P** (scp 와 동일, ssh 와 반대)
- `-i <키>` : 개인키
- `-b <파일>` : 배치 파일의 명령을 순서대로 실행 (**b**atch) — 자동화용
- `-r` : 재귀 (내부 `get`/`put` 에도 적용)

**대화형 명령 (FTP 계열과 동일)**

| 명령 | 대상 | 의미 |
| --- | --- | --- |
| `ls` / `dir` | 원격 | 원격 디렉터리 목록 |
| `lls` | 로컬 | 로컬 디렉터리 목록 |
| `cd <경로>` | 원격 | 원격 디렉터리 이동 |
| `lcd <경로>` | 로컬 | 로컬 디렉터리 이동 |
| `pwd` / `lpwd` | 원격 / 로컬 | 현재 경로 |
| `get <파일>` | 원격 → 로컬 | 내려받기 |
| `mget <패턴>` | 원격 → 로컬 | 여러 개 내려받기 |
| `put <파일>` | 로컬 → 원격 | 올리기 |
| `mput <패턴>` | 로컬 → 원격 | 여러 개 올리기 |
| `get -r <디렉터리>` | 원격 → 로컬 | 디렉터리 재귀 |
| `mkdir` / `rmdir` | 원격 | 디렉터리 생성·삭제 |
| `rm <파일>` | 원격 | 파일 삭제 |
| `rename <old> <new>` | 원격 | 이름 변경 |
| `chmod` / `chown` / `chgrp` | 원격 | 권한·소유자 변경 |
| `df -h` | 원격 | 원격 디스크 여유 |
| `!<명령>` | 로컬 | 로컬 셸 명령 실행 |
| `progress` | — | 진행률 표시 on/off |
| `bye` / `quit` / `exit` | — | 종료 |

**배치 모드 예**

```bash
# [macOS]
cat > /tmp/sftp-batch.txt <<'EOF'
cd /srv/share/lab08
lcd /tmp
get note.txt
put /tmp/from-mac.txt
ls -l
bye
EOF
sftp -P 2222 -b /tmp/sftp-batch.txt admin1@192.168.64.10
```

**검증**

```bash
# [macOS] 대화형 세션 안에서
sftp> pwd            # 원격 현재 경로
sftp> lpwd           # 로컬 현재 경로
sftp> ls -l
sftp> bye
# 결과 확인
ls -l /tmp/note.txt
# [서버]
ls -l /tmp/from-mac.txt
grep -c 'sftp' /etc/ssh/sshd_config                     # Subsystem sftp 행 존재
sshd -T | grep -i subsystem
```

```text
# sshd -T | grep -i subsystem
subsystem sftp /usr/libexec/openssh/sftp-server
```

- `Subsystem sftp` 행이 없거나 주석 처리되면 `sftp` 접속 시 `subsystem request failed` 오류
- SFTP 는 **SSH(22 또는 2222) 위에서 동작** — FTP(21)·FTPS(990/암시적 TLS)와는 다른 프로토콜

> 📝 **시험 포인트**: **SFTP ≠ FTPS**. SFTP 는 SSH 채널 기반(포트 22), FTPS 는 FTP + TLS(21/990). `lcd`·`lls`(로컬) vs `cd`·`ls`(원격) 구분, `mget`/`mput` 복수 전송이 출제 포인트.

### 10-7. rsync — 증분 동기화

> **상황**: 백업·배포의 표준 도구다. 후행 슬래시 유무 차이는 반드시 손으로 확인해야 몸에 남는다.

```bash
rpm -q rsync || dnf install -y rsync           # 양쪽 모두 설치 필요

# [서버] 후행 슬래시 실습용 자료
mkdir -p /tmp/rsync-src/sub /tmp/dst-with /tmp/dst-without
echo a > /tmp/rsync-src/a.txt; echo b > /tmp/rsync-src/sub/b.txt

rsync -av /tmp/rsync-src/  /tmp/dst-with/        # 슬래시 있음 → 내용물만 복사
rsync -av /tmp/rsync-src   /tmp/dst-without/     # 슬래시 없음 → 디렉터리째 복사
find /tmp/dst-with /tmp/dst-without -maxdepth 2 | sort
```

- **후행 슬래시 규칙**: 출발 경로 끝의 `/` 는 "**이 디렉터리의 내용물**"을 뜻함
  - `src/` → `dst/` : `dst/a.txt`, `dst/sub/b.txt`
  - `src`  → `dst/` : `dst/rsync-src/a.txt`, `dst/rsync-src/sub/b.txt`

```bash
# SSH 를 통한 원격 동기화 (비표준 포트)
rsync -avz -e "ssh -p 2222" /srv/share/lab08/ admin1@192.168.64.10:/tmp/lab08-sync/

# 삭제분까지 반영 (미러링) — 먼저 -n 으로 예행
rsync -avzn --delete -e "ssh -p 2222" /srv/share/lab08/ admin1@192.168.64.10:/tmp/lab08-sync/
rsync -avz  --delete -e "ssh -p 2222" /srv/share/lab08/ admin1@192.168.64.10:/tmp/lab08-sync/

# 제외 패턴·진행률·중단 재개
rsync -avz --progress --partial --exclude='*.bin' --exclude='SHA256SUMS' \
      -e "ssh -p 2222" /srv/share/lab08/ admin1@192.168.64.10:/tmp/lab08-filtered/

# 원격 → 로컬 (내려받기)
rsync -avz -e "ssh -p 2222" admin1@192.168.64.10:/tmp/lab08-sync/ /tmp/lab08-pull/
```

- `-a` : **a**rchive — `-rlptgoD` 묶음 (재귀 + 심볼릭 링크 + 권한 + 시각 + 그룹 + 소유자 + 장치·특수 파일). **속성 보존의 핵심**
- `-v` : 상세 출력 (**v**erbose)
- `-z` : 전송 중 압축 (**z**ip) — 느린 회선에서 유리, 고속 LAN 에서는 CPU 낭비일 수 있음
- `-n` (= `--dry-run`) : **예행 연습** — 실제로 바꾸지 않고 무엇이 바뀔지만 출력. `--delete` 전에 필수
- `--delete` : **출발지에 없는 파일을 도착지에서 삭제** → 완전 미러링. ⚠️ 경로 오타 시 대량 삭제 위험
- `--progress` : 파일별 진행률
- `--partial` : 중단된 전송의 부분 파일 보존 → 재실행 시 이어서
- `-P` : `--partial --progress` 묶음
- `--exclude=<패턴>` / `--include=` : 제외·포함 패턴 (여러 번 지정 가능)
- `--exclude-from=<파일>` : 패턴 목록 파일
- `-e "<명령>"` : 원격 셸 지정 (**e**xecute) — **비표준 포트는 `-e "ssh -p 2222"`**
- `--bwlimit=<KB/s>` : 대역폭 제한
- `-u` : 도착지가 더 새로우면 건너뜀 (**u**pdate)
- `-c` : 체크섬으로 비교 (**c**hecksum, 기본은 크기+수정시각)
- 증분 원리: 파일 단위로 크기·수정시각을 비교하고, 바뀐 파일은 **rolling checksum 으로 달라진 블록만** 전송 → `scp` 보다 재전송량이 훨씬 적음

⚠️ macOS 15 이상은 기본 `rsync` 가 **openrsync** 로 대체되어 일부 옵션(`--progress` 등) 동작이 다를 수 있음 → macOS 에서 문제가 생기면 서버 측에서 실행하거나 별도 rsync 설치 후 사용.

**검증**

```bash
find /tmp/dst-with -maxdepth 1 | sort
find /tmp/dst-without -maxdepth 1 | sort
diff -r /srv/share/lab08 /tmp/lab08-sync && echo "동기화 일치"
ls /tmp/lab08-filtered/                       # *.bin 제외됐는지
rsync -avzn --delete -e "ssh -p 2222" /srv/share/lab08/ admin1@192.168.64.10:/tmp/lab08-sync/ | tail -3
```

```text
# find /tmp/dst-with -maxdepth 1 | sort        (슬래시 있음)
/tmp/dst-with
/tmp/dst-with/a.txt
/tmp/dst-with/sub
# find /tmp/dst-without -maxdepth 1 | sort     (슬래시 없음)
/tmp/dst-without
/tmp/dst-without/rsync-src
# diff -r /srv/share/lab08 /tmp/lab08-sync && echo "동기화 일치"
동기화 일치
# ls /tmp/lab08-filtered/
note.txt
```

- 백업 스크립트 연계(`/usr/local/bin/backup.sh`)와 증분 백업 전략은 [[12-backup-recovery-review]]

> 📝 **시험 포인트**: `rsync -avz --delete /data/ user@host:/backup/data/` 해석 = "**속성 보존·압축 전송하며, 출발지에 없는 파일은 도착지에서 삭제(미러링)**". `-a` 가 권한·심볼릭 링크·타임스탬프를 보존한다는 점, **후행 슬래시 유무**, `rsync` 는 변경분만 전송(전체 재전송 아님) 이 3대 출제 포인트.

### 10-8. sshpass — 비대화형 비밀번호 전달 (⚠️ 사용 지양)

> **상황**: 키 인증을 못 쓰는 레거시 장비 자동화에만 쓰는 도구다. 위험성을 정확히 알고 대안을 기본으로 삼는다.

```bash
rpm -q epel-release || dnf install -y epel-release
dnf install -y sshpass                                   # EPEL 필요

# ⚠️ 위험 — 비밀번호가 프로세스 목록·히스토리에 평문 노출
sshpass -p '<비밀번호>' ssh -p 2222 admin1@192.168.64.10 'hostname'

# 완화 — 환경변수 경유 (프로세스 인자에서 제거)
export SSHPASS='<비밀번호>'
sshpass -e ssh -p 2222 -o ConnectTimeout=10 admin1@192.168.64.10 'hostname'
unset SSHPASS

# 파일 경유
sshpass -f /root/.ssh-pass ssh -p 2222 admin1@192.168.64.10 'hostname'
```

- `-p <비밀번호>` : 명령행에서 직접 전달 (**p**assword) — ⚠️ **`ps aux` 로 다른 사용자가 조회 가능**. 다중 사용자 환경 사용 금지
- `-e` : 환경변수 `SSHPASS` 에서 읽음 (**e**nvironment) — 노출 완화, 셋 중 가장 나음
- `-f <파일>` : 파일 첫 줄에서 읽음 (**f**ile) — 파일 권한 600 필수
- ⚠️ 위험 요약
  - `-p` 방식은 프로세스 목록·셸 히스토리에 **평문 노출**
  - 비밀번호를 스크립트·문서·저장소에 기록하는 순간 자격증명 유출 사고
  - `-o StrictHostKeyChecking=no` 와 함께 쓰이는 관행 → **중간자 공격 탐지 불가**
  - **근본 대안 = SSH 키 인증**(8절). 키를 쓰면 `sshpass` 자체가 불필요
- 이 실습 서버는 8-5 에서 `PasswordAuthentication no` 로 바꿨으므로 **sshpass 로는 접속되지 않는 것이 정상**

**검증**

```bash
sshd -T | grep '^passwordauthentication'                # no
export SSHPASS='<비밀번호>'
sshpass -e ssh -p 2222 -o ConnectTimeout=5 admin1@192.168.64.10 'echo x' ; echo "종료코드=$?"
unset SSHPASS
```

```text
# sshd -T | grep '^passwordauthentication'
passwordauthentication no
# sshpass -e ssh ... → 비밀번호 인증이 꺼져 있으므로 실패
Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
종료코드=5
```

- 이 실패가 곧 **8-5 의 보안 강화가 실제로 작동한다는 증거**

> 📝 **시험 포인트**: 자동화에서 비밀번호를 명령행에 넣는 방식은 **프로세스 목록 노출**로 감점. 정답은 **SSH 키 인증 전환**. `sshpass` 는 키 인증이 불가능한 예외 상황에 한정.

---

## 11. 시간 동기화 (chrony)

### 11-1. timedatectl 로 현재 상태 확인

> **상황**: 로그 시각·인증서 유효기간·cron 스케줄이 모두 시스템 시각에 의존한다. Part 09 이전에 시간이 맞는지 확정한다.

```bash
timedatectl                                # 종합 상태
timedatectl status                         # 동일
timedatectl show                           # 기계 판독용 KEY=VALUE
timedatectl list-timezones | grep Seoul    # 시간대 목록 검색
timedatectl set-ntp true                   # NTP 동기화 활성 (chronyd 기동)
timedatectl set-ntp false                  # 비활성 (⚠️ 실습 후 다시 true)
date                                       # 현재 시각
date -u                                    # UTC
date '+%F %T %Z %z'                        # 형식 지정
```

- `timedatectl` 출력 항목
  - `Local time` / `Universal time` / `RTC time` : 로컬 / UTC / 하드웨어 시계
  - `Time zone` : 시간대 (Part 01 에서 `Asia/Seoul` 설정)
  - **`System clock synchronized: yes`** : NTP 로 동기화 완료
  - **`NTP service: active`** : 시간 동기화 데몬(chronyd) 동작 중
  - `RTC in local TZ: no` : 하드웨어 시계를 UTC 로 유지 (권장)
- `set-ntp true/false` : 내부적으로 `chronyd.service` 를 start/stop
- `set-timezone <지역/도시>` : 시간대 변경 — **Part 01 에서 완료됨**, 여기서는 확인만

**검증**

```bash
timedatectl | grep -E 'Time zone|System clock|NTP service'
timedatectl show -p Timezone -p NTPSynchronized
date '+%F %T %Z'
```

```text
# timedatectl
               Local time: ... KST
           Universal time: ... UTC
                 RTC time: ...
                Time zone: Asia/Seoul (KST, +0900)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

- `System clock synchronized: no` 이면 11-3 의 `chronyc sources` 로 원인 추적
- 시간대가 `Asia/Seoul` 이 아니면 `timedatectl set-timezone Asia/Seoul` (Part 01 참조)

> 📝 **시험 포인트**: `timedatectl` 로 확인할 항목 3종 = **시간대 / System clock synchronized / NTP service**. 시간대 변경은 `timedatectl set-timezone`, NTP on/off 는 `set-ntp`. 구식 `/etc/localtime` 심볼릭 링크 방식도 병행 출제.

### 11-2. /etc/chrony.conf 설정 항목

> **상황**: 동기화 소스와 동작 방식을 정하는 파일이다. 각 지시자의 의미를 확인하고 내부 NTP 서버를 쓰는 경우의 수정 방법도 정리한다.

```bash
rpm -q chrony
systemctl is-enabled chronyd; systemctl is-active chronyd
grep -vE '^\s*#|^\s*$' /etc/chrony.conf
cp /etc/chrony.conf /root/net-backup/chrony.conf.orig
```

**주요 지시자**

| 지시자 | 의미 | 예 |
| --- | --- | --- |
| `pool <호스트> iburst` | **여러 서버**를 DNS 라운드로빈으로 받아 사용 | `pool 2.rocky.pool.ntp.org iburst` |
| `server <호스트> iburst` | **단일 서버** 지정 (내부 NTP 서버용) | `server 192.168.64.1 iburst` |
| `iburst` | 기동 직후 짧은 간격으로 4~8개 요청 → **첫 동기화 가속** | — |
| `driftfile <경로>` | 시계 오차율(ppm) 저장 → 재부팅 후 즉시 보정 | `/var/lib/chrony/drift` |
| `makestep <초> <횟수>` | 오차가 `<초>` 이상이면 **점프(step)** 보정, 기동 후 `<횟수>` 번까지만 | `makestep 1.0 3` |
| `rtcsync` | 커널이 주기적으로 **RTC(하드웨어 시계)를 갱신** | — |
| `allow <대역>` | 이 서버를 **NTP 서버로** 쓸 클라이언트 허용 (기본 비허용) | `allow 192.168.64.0/24` |
| `local stratum <n>` | 상위 서버와 끊겨도 stratum n 으로 **자체 시각 제공** | `local stratum 10` |
| `logdir <경로>` | 로그 디렉터리 | `/var/log/chrony` |
| `keyfile <경로>` | 인증 키 파일 | `/etc/chrony.keys` |
| `leapsectz <TZ>` | 윤초 정보 획득용 시간대 | `right/UTC` |
| `bindaddress` / `port` | 수신 주소·포트 (기본 **123/UDP**) | — |

**내부 NTP 서버(게이트웨이)를 쓰도록 변경 (참고 · 선택 실행)**

⚠️ 변경 전 원본을 백업하고, 외부 pool 을 지우기 전에 대체 서버가 실제로 응답하는지 확인할 것.

```bash
nc -zvu -w 3 192.168.64.1 123 2>&1 | tail -1        # 게이트웨이 NTP 응답 확인 (환경에 따라 다름)
# 응답이 있을 때만 아래 진행
sed -i 's/^pool /#pool /' /etc/chrony.conf
sed -i '/^#pool /a server 192.168.64.1 iburst' /etc/chrony.conf
grep -nE '^(server|#?pool)' /etc/chrony.conf
systemctl restart chronyd
chronyc sources -v
# 원복
cp /root/net-backup/chrony.conf.orig /etc/chrony.conf && systemctl restart chronyd
```

**검증**

```bash
grep -vE '^\s*#|^\s*$' /etc/chrony.conf
systemctl is-active chronyd
ss -ulnp | grep -E 'chronyd|:323'          # 로컬 제어 소켓(323/UDP)
```

```text
# grep -vE '^\s*#|^\s*$' /etc/chrony.conf
pool 2.rocky.pool.ntp.org iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
keyfile /etc/chrony.keys
ntsdumpdir /var/lib/chrony
leapsectz right/UTC
logdir /var/log/chrony
```

> 📝 **시험 포인트**: NTP 포트 = **123/UDP**. `server`(단일) vs `pool`(복수) 구분, `iburst`(초기 동기화 가속), `makestep`(큰 오차를 점프로 보정) 의미가 설정 파일 해석 문제로 출제.

### 11-3. chronyc — 동기화 상태 조회

> **상황**: 실제로 어느 서버와 맞추고 있고 오차가 얼마인지 본다. 출력 열의 기호 의미가 핵심이다.

```bash
chronyc sources                    # 동기화 소스 목록
chronyc sources -v                 # 열 설명 포함
chronyc tracking                   # 현재 동기화 상태 상세
chronyc sourcestats                # 소스별 통계(오차·표류율)
chronyc activity                   # 온라인/오프라인 소스 개수
chronyc ntpdata                    # NTP 패킷 상세 (소스별)
chronyc makestep                   # ⚠️ 즉시 점프 보정 강제
chronyc -a 'burst 4/4'             # 강제 재질의 (-a = 인증)
systemctl status chronyd --no-pager | head -5
journalctl -u chronyd -n 20 --no-pager
```

**`chronyc sources -v` 열 해석**

```text
  .-- Source mode  '^' = server, '=' = peer, '#' = local clock.
 / .- Source state '*' = current best, '+' = combined, '-' = not combined,
| /               'x' = may be in error, '~' = too variable, '?' = unusable.
||                                                 .- xxxx [ yyyy ] +/- zzzz
||      Reachability register (octal) -.           |  xxxx = adjusted offset,
||      Log2(Polling interval) --.      |          |  yyyy = measured offset,
||                                \     |          |  zzzz = estimated error.
||                                 |    |           \
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* <서버>                        2   6   377    ...   +...us[ +...us] +/-  ...ms
^- <서버>                        2   6   377    ...   -...ms[ -...ms] +/-  ...ms
```

| 열 | 의미 |
| --- | --- |
| `M` (1번째 문자) | 소스 모드 — `^` 서버, `=` 피어, `#` 로컬 기준 시계 |
| `S` (2번째 문자) | 상태 — **`*` 현재 동기화 대상**, `+` 결합 사용, `-` 미사용, `x` 오류 의심(falseticker), `~` 변동 과다, `?` 도달 불가 |
| `Name/IP address` | 소스 주소 |
| `Stratum` | 기준 시계로부터의 거리 — **0=원자시계·GPS, 1=직접 연결 서버, 2=1에 동기화된 서버…** 최대 15, **16=미동기화** |
| `Poll` | 질의 주기 = 2^Poll 초 (6 → 64초) |
| `Reach` | 최근 8회 응답 이력을 8진수로 — **`377` = 8회 모두 성공(최상)**, `0` = 무응답 |
| `LastRx` | 마지막 응답 수신 경과 시간 |
| `Last sample` | 오차와 추정 오차 범위 |

**`chronyc tracking` 주요 항목**

| 항목 | 의미 |
| --- | --- |
| `Reference ID` / `Stratum` | 현재 동기화 중인 상위 소스와 계층 |
| `System time` | 시스템 시계의 오차 (fast/slow of NTP time) |
| `Last offset` | 마지막 보정량 |
| `RMS offset` | 오차의 장기 평균 |
| `Frequency` | 시계의 표류율(ppm) |
| `Skew` | 주파수 추정의 불확실도 |
| `Leap status` | `Normal` 이면 정상 |

**검증**

```bash
chronyc tracking | grep -E 'Reference ID|Stratum|System time|Leap status'
chronyc sources | grep '^\^\*'          # * 표시된 현재 동기화 소스가 있어야 정상
chronyc activity
timedatectl | grep 'System clock synchronized'
```

```text
# chronyc sources
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* <서버>                        2   6   377     ...   +...us[+...us] +/- ...ms
^+ <서버>                        2   6   377     ...   -...ms[-...ms] +/- ...ms
# chronyc tracking
Reference ID    : ... (<서버>)
Stratum         : 3
...
Leap status     : Normal
# chronyc activity
... sources online
0 sources offline
```

- `^*` 가 하나도 없고 전부 `?` 면 → 외부 NTP 도달 불가. `nc -zvu <서버> 123`·`firewall-cmd` 확인
- Stratum 이 **16** 이면 미동기화 상태

> 📝 **시험 포인트**: `chronyc sources` 의 **`^*` = 현재 동기화 중인 서버**, `Reach 377` = 최근 8회 모두 응답. Stratum 은 기준 시계로부터의 거리이며 **숫자가 작을수록 정확**, 16은 미동기화. `chronyc tracking` 은 현재 오차 확인.

### 11-4. 수동 시각 설정과 하드웨어 시계

> **상황**: NTP 가 있는 환경에서 `date -s` 를 쓰면 왜 문제가 되는지, 하드웨어 시계는 어떻게 다루는지 정리한다.

⚠️ `date -s` 로 시각을 크게 바꾸면 로그 시각 역전·cron 중복 실행·TLS 인증서 검증 실패·DB 트랜잭션 순서 오류가 발생할 수 있다. NTP 가 동작 중이면 곧 되돌려지므로 **실습은 아주 작은 폭으로만** 하고 즉시 복구한다.

```bash
date                                        # 현재
timedatectl set-ntp false                   # NTP 잠시 중지 (안 그러면 즉시 되돌아감)
date -s '+2 seconds'                        # 2초 앞으로 (⚠️ 작은 폭)
date
timedatectl set-ntp true                    # NTP 재개
sleep 5; chronyc tracking | grep 'System time'
chronyc makestep                            # 즉시 점프 보정
timedatectl | grep 'System clock synchronized'
```

```bash
# 하드웨어 시계(RTC)
hwclock -r                     # 읽기 (--show)
hwclock -w                     # 시스템 → 하드웨어 (= --systohc)
hwclock --systohc              # 동일
hwclock -s                     # 하드웨어 → 시스템 (= --hctosys)
hwclock --hctosys              # 동일
hwclock --verbose | head       # 상세
```

- `-r` / `--show` : RTC 값 읽기 (**r**ead)
- `-w` / `--systohc` : **sys**tem **to** **h**ardware **c**lock — 시스템 시각을 RTC 에 기록
- `-s` / `--hctosys` : **h**ardware **c**lock **to** **sys**tem — RTC 값을 시스템에 적용
- `--utc` / `--localtime` : RTC 를 UTC / 로컬 시각으로 해석
- ※ 가상 머신에서는 RTC 접근이 제한되어 `hwclock` 이 오류를 낼 수 있음 — `chronyd` 의 `rtcsync` 가 이미 담당하므로 수동 조작 불필요

**ntpdate / ntpq 는 chrony 로 대체됨**

| 구식 (ntp 패키지) | 현행 (chrony) | 비고 |
| --- | --- | --- |
| `ntpd` | `chronyd` | RHEL 8 부터 **기본이 chrony**, `ntp` 패키지는 저장소에서 제거 |
| `ntpdate <서버>` | `chronyd -q 'server <서버> iburst'` 또는 `chronyc makestep` | `ntpdate` 는 upstream 에서도 폐기(deprecated) |
| `ntpq -p` | `chronyc sources -v` | 소스 목록 조회 |
| `ntpstat` | `chronyc tracking` | 동기화 상태 |
| `/etc/ntp.conf` | `/etc/chrony.conf` | 설정 파일 |

- chrony 의 장점: 간헐적 네트워크·가상화 환경에서 빠른 수렴, 오프라인 후 재동기화가 빠름

**검증**

```bash
timedatectl | grep -E 'System clock|NTP service'
chronyc tracking | grep -E 'Leap status|System time'
hwclock -r 2>&1 | head -1
date '+%F %T %Z'
```

```text
# timedatectl | grep -E 'System clock|NTP service'
System clock synchronized: yes
              NTP service: active
# chronyc tracking | grep 'Leap status'
Leap status     : Normal
```

> 📝 **시험 포인트**: RHEL 8/9 의 시간 동기화 기본 데몬은 **`chronyd`**(구 `ntpd`), 설정 파일 **`/etc/chrony.conf`**, 상태 조회 **`chronyc sources`/`tracking`**, 포트 **123/UDP**. `hwclock --systohc`(시스템→RTC) 방향 구분이 함정.

---

## 12. 참고 항목 (※ 미실행 또는 개념)

### 12-1. 본딩·팀·브리지·VLAN

> **상황**: NIC 이 하나뿐이라 실제 구성은 불가능하다. 명령 형식과 모드 개념만 정리한다.

```bash
# 본딩(bonding) — 여러 NIC 을 하나로 (※ 미실행 — NIC 1개)
nmcli con add type bond con-name bond0 ifname bond0 bond.options "mode=active-backup,miimon=100"
nmcli con add type ethernet con-name bond0-p1 ifname enp0s1 master bond0
nmcli con add type ethernet con-name bond0-p2 ifname enp0s2 master bond0
cat /proc/net/bonding/bond0          # 본딩 상태 확인

# 브리지 — 가상 스위치 (※ 미실행)
nmcli con add type bridge con-name br0 ifname br0
nmcli con add type ethernet con-name br0-p1 ifname enp0s1 master br0
ip link add br0 type bridge          # ip 명령판
ip link set enp0s1 master br0
bridge link show                     # 브리지 포트 목록

# VLAN — 802.1Q 태그 (※ 미실행 — 스위치 미지원)
nmcli con add type vlan con-name vlan10 ifname vlan10 dev enp0s1 id 10
ip link add link enp0s1 name enp0s1.10 type vlan id 10
```

**본딩 모드 표**

| mode | 이름 | 특징 |
| --- | --- | --- |
| 0 | `balance-rr` | 라운드로빈 — 부하 분산 + 이중화, 스위치 설정 필요 |
| 1 | **`active-backup`** | 하나만 활성, 장애 시 전환 — **스위치 설정 불필요, 가장 안전** |
| 2 | `balance-xor` | 출발/목적 MAC 해시로 분산 |
| 3 | `broadcast` | 모든 슬레이브로 전송 |
| 4 | **`802.3ad`** | LACP 링크 애그리게이션 — **스위치가 LACP 지원해야 함** |
| 5 | `balance-tlb` | 송신만 부하 분산 |
| 6 | `balance-alb` | 송수신 모두 부하 분산 (ARP 협상 이용) |

- **팀(teaming)** : 본딩의 후속 기술(`teamd`) — RHEL 9 에서 **deprecated**, 본딩 사용 권장
- 브리지 : 컨테이너·가상머신 네트워크의 기반 (Part 11 Docker 의 `docker0` 이 브리지)

> 📝 **시험 포인트**: 본딩 mode 1 = `active-backup`(장애 조치), mode 4 = `802.3ad`(LACP). 팀은 RHEL 9 에서 폐기 예정이라는 점도 최신 회차 소재.

### 12-2. 네트워크 네임스페이스와 무선

> **상황**: 컨테이너 네트워크의 기반 기술과 무선 설정 명령을 형식만 확인한다.

```bash
# 네트워크 네임스페이스 (※ 개념 — Docker 의 기반)
ip netns list                            # 네임스페이스 목록 (기본 비어 있음)
ip netns add labns                       # 생성
ip netns exec labns ip a                 # 그 안에서 명령 실행 (lo 만 보임)
ip netns exec labns ip link set lo up
ip netns exec labns ping -c 1 127.0.0.1
ip netns delete labns                    # 정리
```

- `ip netns` : 격리된 네트워크 스택(인터페이스·라우팅·방화벽 규칙) 생성 — **컨테이너 네트워크 격리의 실체**
- Docker 는 컨테이너마다 netns 를 만들고 veth 쌍으로 `docker0` 브리지에 연결 → [[11-container-virtualization]]

```bash
# 무선 (※ 미실행 — 무선 NIC 없음)
nmcli device wifi list                   # 주변 AP 검색
nmcli device wifi connect <SSID> password '<암호>'
nmcli radio wifi on/off
iwconfig                                 # 구식 (wireless-tools)
iw dev; iw dev wlan0 scan                # 현행
rfkill list                              # 하드웨어 차단 스위치 상태
```

**검증**

```bash
ip netns list; echo "(비어 있으면 정상)"
nmcli device status | grep -c wifi        # 0 = 무선 장치 없음
```

> 📝 **시험 포인트**: `ip netns` = 네트워크 네임스페이스 — 컨테이너 격리의 기반. 무선 관리 현행 명령은 `nmcli device wifi`·`iw`, 구식은 `iwconfig`(wireless-tools).

### 12-3. 레거시 설정 파일과 ifup/ifdown

> **상황**: 시험 지문에 나오는 구식 파일·명령의 현재 상태를 정확히 짚는다.

```bash
ls -l /etc/sysconfig/network-scripts/ 2>/dev/null    # RHEL9: 비어 있거나 ifcfg 없음
cat /etc/sysconfig/network 2>/dev/null                # 전역 설정 (거의 비어 있음)
which ifup ifdown 2>/dev/null || echo "ifup/ifdown 없음"
dnf info NetworkManager-initscripts-updown 2>/dev/null | head -8
```

- `/etc/sysconfig/network-scripts/ifcfg-<장치>` : RHEL 7 까지의 표준 인터페이스 설정 → **RHEL 9 에서는 신규 생성되지 않음**(keyfile 사용). 기존 파일은 `NetworkManager-initscripts-ifcfg-rh` 플러그인이 있어야 읽힘
- `/etc/sysconfig/network` : `NETWORKING=yes`, `HOSTNAME=`, `GATEWAY=` 등 전역 설정 → 현재는 `hostnamectl`·NetworkManager 가 대체
- `ifup`/`ifdown` : RHEL 9 에서 기본 미제공 → `NetworkManager-initscripts-updown` 설치 시 **`nmcli con up/down` 의 얇은 래퍼**로 제공됨
- 현행 대응: `ifup enp0s1` → **`nmcli con up enp0s1`**, `ifdown enp0s1` → **`nmcli con down enp0s1`**
- 마찬가지로 RHEL 9 에서 사라진 것들: `network.service`(→ NetworkManager), `hosts.allow/deny`(TCP Wrapper 미지원 → 개념만, [[10-security-firewall-selinux]]), `xinetd`(기본 저장소 없음)

> 📝 **시험 포인트**: `ifcfg-*` 항목(`BOOTPROTO`·`ONBOOT`·`IPADDR`·`PREFIX`·`GATEWAY`·`DNS1`) 은 여전히 출제되지만, **RHEL 9 실기 환경에서는 `nmcli` 로 답해야 함**. 두 체계의 대응(2-4 표)을 함께 외울 것.

### 12-4. DHCP 클라이언트 동작과 프록시 환경변수

> **상황**: 고정 IP 로 바꾸기 전까지 무슨 일이 있었는지, 그리고 폐쇄망에서 필요한 프록시 설정을 정리한다.

```bash
# DHCP — RHEL9 의 기본 클라이언트는 NetworkManager 내장 (internal)
nmcli con show enp0s1 | grep -i dhcp
ls -l /var/lib/NetworkManager/ | grep -i lease           # 리스 파일
cat /var/lib/NetworkManager/*lease* 2>/dev/null | head    # 임대 내용 (DHCP 사용 시)
journalctl -u NetworkManager | grep -i dhcp | tail -10   # DHCP 협상 로그
rpm -q dhcp-client 2>/dev/null || echo "dhclient 미설치 (NM internal 사용)"
dhclient -v enp0s1                                        # ※ 실행 금지 — 고정 IP 와 충돌
```

- **DORA 절차** (DHCP 임대 4단계)
  1. **D**iscover : 클라이언트 → 브로드캐스트 (`0.0.0.0` → `255.255.255.255`, 68 → 67)
  2. **O**ffer : 서버 → 사용 가능한 IP 제안 (67 → 68)
  3. **R**equest : 클라이언트 → 그 IP 를 쓰겠다고 요청 (브로드캐스트)
  4. **A**ck : 서버 → 확정 + 임대 기간·게이트웨이·DNS 전달
- 포트: **서버 67/UDP, 클라이언트 68/UDP**
- 갱신: 임대 기간의 50%(T1) 에서 갱신 시도, 87.5%(T2) 에서 재바인딩, 실패 시 만료 → APIPA(`169.254.x.x`)
- ⚠️ `dhclient` 를 지금 실행하면 고정 IP 설정과 충돌 → **실행하지 말 것**

```bash
# 프록시 환경변수 (폐쇄망·사내 프록시 환경 — ※ 이 실습망은 불필요)
export http_proxy="http://proxy.lab.local:3128"
export https_proxy="http://proxy.lab.local:3128"
export no_proxy="localhost,127.0.0.1,192.168.64.0/24,.lab.local"
env | grep -i proxy
unset http_proxy https_proxy no_proxy
```

- 영구 설정 위치
  - 전역 셸: `/etc/profile.d/proxy.sh`
  - `dnf` 전용: `/etc/dnf/dnf.conf` 의 `proxy=http://proxy:3128`
  - `wget`: `~/.wgetrc` / `/etc/wgetrc`
  - `curl`: `~/.curlrc`
- `no_proxy` : 프록시를 거치지 않을 대상 — **로컬·사내 대역을 반드시 넣을 것** (누락 시 내부 통신까지 프록시 경유)

**검증**

```bash
nmcli con show enp0s1 | grep -E 'ipv4.method'      # manual (DHCP 사용 안 함)
env | grep -ci proxy || echo "프록시 미설정 (정상)"
grep -c 'proxy' /etc/dnf/dnf.conf 2>/dev/null
```

> 📝 **시험 포인트**: DHCP 절차 **DORA**(Discover-Offer-Request-Ack), 포트 **서버 67 / 클라이언트 68 (UDP)**. `dhcpd.conf` 의 `option routers`= 게이트웨이, `range`= 임대 범위, `fixed-address`= 고정 할당 — 서버 구성은 시험 범위이나 이 실습에서는 미구축.

---

## 13. 마무리 검증

### 13-1. check-part08.sh 작성

> **상황**: 이 파트에서 만든 설정이 전부 살아 있는지 한 번에 판정하는 스크립트를 만든다. 재부팅 후 다시 돌려 영구성을 확인한다.

```bash
cat > /usr/local/bin/check-part08.sh <<'EOF'
#!/bin/bash
# LAB 08 검증 — IP·GW·DNS·hosts·sshd 포트·키 인증·chrony
ok(){ printf '  [OK]  %s\n' "$1"; }
ng(){ printf '  [NG]  %s\n' "$1"; RC=1; }
RC=0
IF=enp0s1; IP=192.168.64.10; GW=192.168.64.1; SSHPORT=2222

echo "== 주소·라우팅"
ip -4 -br a show "$IF"
ip -4 -o a show "$IF" | grep -q "$IP/24" && ok "고정 IP $IP/24" || ng "IP 불일치: $(ip -4 -o a show "$IF" | awk '{print $4}')"
ip route show default | grep -q "via $GW" && ok "기본 게이트웨이 $GW" || ng "기본 게이트웨이 없음/불일치"
ip route show default | grep -q 'proto static' && ok "proto static (고정 설정)" || ng "proto static 아님 → DHCP 로 되돌아감"
nmcli -g ipv4.method con show "$IF" 2>/dev/null | grep -qx manual && ok "nmcli ipv4.method=manual" || ng "ipv4.method 가 manual 이 아님"
nmcli -g connection.autoconnect con show "$IF" 2>/dev/null | grep -qi yes && ok "autoconnect yes" || ng "autoconnect 미설정 → 재부팅 시 미기동"

echo "== DNS·이름 해석"
grep -q "nameserver $GW" /etc/resolv.conf && ok "DNS $GW 등록" || ng "resolv.conf 에 $GW 없음"
grep -q 'nameserver 8.8.8.8' /etc/resolv.conf && ok "보조 DNS 8.8.8.8" || ng "보조 DNS 없음"
grep -q '^search .*lab.local' /etc/resolv.conf && ok "search lab.local" || ng "검색 도메인 없음"
getent hosts srv01 >/dev/null 2>&1 && ok "/etc/hosts srv01 해석" || ng "/etc/hosts 항목 없음"
getent hosts intranet.lab.local >/dev/null 2>&1 && ok "/etc/hosts intranet.lab.local" || ng "intranet 항목 없음"
grep -Eq '^hosts:.*files.*dns' /etc/nsswitch.conf && ok "nsswitch hosts: files dns" || ng "nsswitch 순서 확인 필요"

echo "== 연결성"
ping -c 1 -W 2 "$GW" >/dev/null 2>&1 && ok "게이트웨이 도달" || ng "게이트웨이 무응답"
ping -c 1 -W 3 8.8.8.8 >/dev/null 2>&1 && ok "외부 IP 도달" || ng "외부 통신 불가"
getent hosts rockylinux.org >/dev/null 2>&1 && ok "외부 이름 해석" || ng "DNS 해석 실패"

echo "== 호스트명"
[ "$(hostname -f 2>/dev/null)" = "srv01.lab.local" ] && ok "FQDN srv01.lab.local" || ng "FQDN=$(hostname -f 2>/dev/null)"

echo "== SSH"
systemctl is-active --quiet sshd && ok "sshd active" || ng "sshd 정지"
systemctl is-enabled --quiet sshd && ok "sshd enabled" || ng "sshd 자동시작 아님"
ss -tln | grep -q ":$SSHPORT " && ok "sshd $SSHPORT 리스닝" || ng "$SSHPORT 리스닝 아님"
ss -tln | grep -q ':22 ' && ng "22 번이 아직 열려 있음" || ok "22 번 폐쇄됨"
sshd -T 2>/dev/null | grep -qx 'passwordauthentication no' && ok "비밀번호 인증 차단" || ng "PasswordAuthentication 이 no 아님"
sshd -T 2>/dev/null | grep -qx 'permitrootlogin no' && ok "root 로그인 차단" || ng "PermitRootLogin 이 no 아님"
sshd -T 2>/dev/null | grep -qx 'pubkeyauthentication yes' && ok "공개키 인증 활성" || ng "PubkeyAuthentication 확인 필요"
[ -s /home/admin1/.ssh/authorized_keys ] && ok "admin1 authorized_keys 존재" || ng "authorized_keys 없음/비어 있음"
[ "$(stat -c %a /home/admin1/.ssh 2>/dev/null)" = "700" ] && ok ".ssh 권한 700" || ng ".ssh 권한=$(stat -c %a /home/admin1/.ssh 2>/dev/null)"
[ "$(stat -c %a /home/admin1/.ssh/authorized_keys 2>/dev/null)" = "600" ] && ok "authorized_keys 권한 600" || ng "authorized_keys 권한 확인"
semanage port -l 2>/dev/null | grep -w ssh_port_t | grep -q "$SSHPORT" && ok "SELinux ssh_port_t $SSHPORT" || ng "SELinux 포트 레이블 없음"
firewall-cmd --list-ports 2>/dev/null | grep -q "$SSHPORT/tcp" && ok "방화벽 $SSHPORT/tcp 개방" || ng "방화벽 미개방"

echo "== 시간 동기화"
systemctl is-active --quiet chronyd && ok "chronyd active" || ng "chronyd 정지"
timedatectl show -p NTPSynchronized --value 2>/dev/null | grep -qi yes && ok "System clock synchronized" || ng "시각 미동기화"
chronyc sources 2>/dev/null | grep -q '^\^\*' && ok "동기화 소스(^*) 존재" || ng "동기화 소스 없음"
[ "$(timedatectl show -p Timezone --value 2>/dev/null)" = "Asia/Seoul" ] && ok "시간대 Asia/Seoul" || ng "시간대=$(timedatectl show -p Timezone --value 2>/dev/null)"

echo
[ $RC -eq 0 ] && echo "ALL OK" || echo "일부 실패 — 위 [NG] 확인"
exit $RC
EOF
chmod +x /usr/local/bin/check-part08.sh
/usr/local/bin/check-part08.sh
```

- `nmcli -g <필드>` : 값만 출력 (**g**et-values) — `-t -f` 보다 스크립트에 편함
- `timedatectl show -p <속성> --value` : 속성 값만 추출
- `grep -qx` : 줄 전체 일치 + 조용히 (**x**act line)
- 종료 코드로 성공/실패를 반환하므로 cron·다른 스크립트에서 재사용 가능

**검증**

```bash
/usr/local/bin/check-part08.sh | grep -E '^\s+\[(OK|NG)\]|ALL OK'
echo "종료코드=$?"
```

```text
  [OK]  고정 IP 192.168.64.10/24
  [OK]  기본 게이트웨이 192.168.64.1
  [OK]  proto static (고정 설정)
  [OK]  nmcli ipv4.method=manual
  [OK]  autoconnect yes
  [OK]  DNS 192.168.64.1 등록
  [OK]  보조 DNS 8.8.8.8
  [OK]  search lab.local
  [OK]  /etc/hosts srv01 해석
  [OK]  /etc/hosts intranet.lab.local
  [OK]  nsswitch hosts: files dns
  [OK]  게이트웨이 도달
  [OK]  외부 IP 도달
  [OK]  외부 이름 해석
  [OK]  FQDN srv01.lab.local
  [OK]  sshd active
  [OK]  sshd enabled
  [OK]  sshd 2222 리스닝
  [OK]  22 번 폐쇄됨
  [OK]  비밀번호 인증 차단
  [OK]  root 로그인 차단
  [OK]  공개키 인증 활성
  [OK]  admin1 authorized_keys 존재
  [OK]  .ssh 권한 700
  [OK]  authorized_keys 권한 600
  [OK]  SELinux ssh_port_t 2222
  [OK]  방화벽 2222/tcp 개방
  [OK]  chronyd active
  [OK]  System clock synchronized
  [OK]  동기화 소스(^*) 존재
  [OK]  시간대 Asia/Seoul
ALL OK
```

> 📝 **시험 포인트**: 실기에서 "설정 후 확인 명령을 쓰시오" 유형이 반복됨. IP=`ip a`, 게이트웨이=`ip route`, DNS=`cat /etc/resolv.conf`, 포트=`ss -tlnp`, 서비스=`systemctl is-active`, 시각=`timedatectl` — 대응을 통째로 외울 것.

### 13-2. 재부팅 후 영구성 확인

> **상황**: 임시 설정과 영구 설정의 차이는 재부팅이 갈라 준다. 마지막으로 한 번 더 확인한다.

```bash
# 재부팅 전 정리
rm -f /tmp/cap.pcap /tmp/send.txt /tmp/received.txt      # 캡처·테스트 파일 정리
ip -4 -br a show enp0s1                                   # 보조 주소 없음 확인
ip route | grep -c '10.10'                                # 0
sync; reboot
```

- 재부팅 후 `admin1` 로 SSH(2222) 접속 → `sudo /usr/local/bin/check-part08.sh`

**검증**

```bash
# [macOS]
ssh -p 2222 admin1@192.168.64.10 'sudo /usr/local/bin/check-part08.sh | tail -3'
ssh -p 2222 admin1@192.168.64.10 'ip -4 -br a show enp0s1; ip route show default; uptime'
# [서버]
journalctl -b -u NetworkManager -p err --no-pager | head    # 부팅 중 네트워크 오류 없는지
systemctl --failed --no-pager
```

```text
# [macOS] ssh -p 2222 ... check-part08.sh | tail -3
  [OK]  시간대 Asia/Seoul

ALL OK
# ip -4 -br a show enp0s1
enp0s1           UP             192.168.64.10/24
# systemctl --failed
0 loaded units listed.
```

- 재부팅 후에도 `ALL OK` → 이 파트의 모든 설정이 **영구 반영** 완료
- 이제 [[09-network-services]] 에서 이 고정 IP·이름 위에 Apache·BIND·NFS·Samba 를 올릴 수 있음

> 📝 **시험 포인트**: 네트워크 설정 문제의 마지막 채점 기준은 거의 항상 **"재부팅 후에도 유지되는가"** — `nmcli`(또는 설정 파일) + `connection.autoconnect yes` + 방화벽 `--permanent` 3종이 그 답.

---

## 체크리스트

| 수행 항목 | 명령 | 확인 방법 | ☐ |
| --- | --- | --- | --- |
| 현재 상태 백업 | `ip a`·`ip r`·`nmcli con show` → `/root/net-backup/` | `ls -l /root/net-backup/` 5개 파일 | ☐ |
| 주소·링크·통계 조회 | `ip -br a`, `ip -br link`, `ip -s link show enp0s1` | RX/TX·errors·dropped 카운터 출력 | ☐ |
| 라우팅·ARP 조회 | `ip route`, `ip neigh`, `ip route get 8.8.8.8` | `default via 192.168.64.1` | ☐ |
| IPv6 확인 | `ip -6 a`, `ping6 -c 2 ::1` | `fe80::/64` 링크로컬, 루프백 응답 | ☐ |
| NetworkManager 상태 | `nmcli device status`, `nmcli con show enp0s1 \| grep ipv4` | `connected`, `ipv4.method` | ☐ |
| 이름 해석 설정 확인 | `cat /etc/resolv.conf`, `grep '^hosts:' /etc/nsswitch.conf` | `# Generated by NetworkManager`, `files dns myhostname` | ☐ |
| 호스트명 확인 | `hostname -f`·`-I`, `hostnamectl` | `srv01.lab.local` | ☐ |
| 포트·프로토콜 사전 | `grep -w '22/tcp' /etc/services`, `grep -w '^tcp' /etc/protocols` | `ssh 22/tcp`, `tcp 6` | ☐ |
| net-tools 설치·대응 | `dnf install -y net-tools` → `ifconfig -a`·`netstat -rn`·`route -n`·`arp -n` | `route -n == netstat -rn` | ☐ |
| ethtool (virtio 제약) | `ethtool -i enp0s1`, `ethtool enp0s1` | `driver: virtio_net`, `Link detected: yes` | ☐ |
| **고정 IP 전환** | `nmcli con mod enp0s1 ipv4.method manual ipv4.addresses 192.168.64.10/24 …` → `nmcli con up enp0s1` | `ip -4 -o a` = `192.168.64.10/24`, `proto static` | ☐ |
| keyfile 확인 | `cat /etc/NetworkManager/system-connections/enp0s1.nmconnection` | `method=manual`, 권한 600 | ☐ |
| 프로파일 2개 전환 | `nmcli con add … con-name lab-static` → `con up` 전환 → `con delete` | 주소 `.12` ↔ `.10` 왕복, 삭제 후 목록 1개 | ☐ |
| nmcli 부속 명령 | `general status`, `-t -f NAME,DEVICE con show`, `+ipv4.routes`, `reload`, `reapply` | 각 출력 | ☐ |
| 임시 주소·경로 | `ip addr add 192.168.64.11/24 dev enp0s1` → `ip route add 10.10.0.0/16 via …` → 삭제 | 추가/삭제 전후 개수 변화 | ☐ |
| 링크·MTU 조작 (콘솔) | `ip link set enp0s1 down/up`, `ip link set mtu 1400` → 복구 | `ip -br link`, `mtu 1500` | ☐ |
| ip_forward | `sysctl -w net.ipv4.ip_forward=1` → 0 복구 | `sysctl -n net.ipv4.ip_forward` = 0 | ☐ |
| 재부팅으로 임시/영구 구분 | 임시 주소·경로 넣고 `reboot` | 보조 주소·경로 소멸, `.10` 유지 | ☐ |
| **/etc/hosts 등록** | `192.168.64.10 srv01.lab.local srv01 intranet.lab.local` | `getent hosts srv01`, `ping -c1 srv01` | ☐ |
| hosts ≠ DNS 확인 | `dig srv01.lab.local` | `status: NXDOMAIN` (getent 는 성공) | ☐ |
| nslookup·host | `nslookup rockylinux.org`, `host -t MX rockylinux.org` | `Non-authoritative answer`, MX 우선순위 | ☐ |
| dig 섹션 해석 | `dig @192.168.64.1 rockylinux.org`, `+short +noall +answer +trace` | `status: NOERROR`, ANSWER 존재 | ☐ |
| dig 유형·역방향 | `dig … A/AAAA/MX/NS/SOA/TXT`, `dig -x 8.8.8.8` | 각 레코드, `dns.google.` | ☐ |
| resolvectl 상태 확인 | `systemctl is-active systemd-resolved`, `file /etc/resolv.conf` | `inactive`, 일반 파일 (RHEL9 기본) | ☐ |
| ping 옵션 전개 | `ping -c 4 -i 0.5 -s 1000 -W 1 -w 5 -n 192.168.64.1` | 4 packets, 0% loss | ☐ |
| TTL 로 OS 추정 | `ping -c 1 8.8.8.8 \| grep -o 'ttl=[0-9]*'` | 로컬 64, 외부는 감소값 | ☐ |
| 경로 추적 | `traceroute -n/-I/-T 8.8.8.8`, `tracepath`, `mtr -rn -c 5` | 홉 목록, `* * *` 의미 이해 | ☐ |
| **ss 전개** | `ss -tulnp`·`-tan state established`·`-s`·`-o`·`-lntp 'sport = :22'`·`-x` | `0.0.0.0:22 LISTEN sshd` | ☐ |
| netstat 대응 | `netstat -tulnp -an -rn -i -s` | `ss` 출력과 동일 정보 | ☐ |
| nc 포트 점검 | `nc -zv 127.0.0.1 22`, `nc -zvu -w 3 192.168.64.1 53` | `Connected` / `refused` / 타임아웃 구분 | ☐ |
| nc 양방향·파일 전송 | `nc -l 9000` ↔ `nc 127.0.0.1 9000`, `nc -l 9001 > f` ↔ `nc … < f` | 문자열 도달, `diff` 일치 | ☐ |
| telnet 배너 | `telnet 127.0.0.1 22` | `SSH-2.0-OpenSSH_...` | ☐ |
| curl·wget | `curl -I -v -o -O -L -k -u -d -H -X --resolve`, `wget -c -O -q --spider` | `HTTP/2 200`, `%{http_code}` | ☐ |
| nmap 스캔 | `nmap -sT -p 1-1024,2222 127.0.0.1`, `-sU -p 53`, `-sV`, `-sn 192.168.64.0/24`, `-O` | `22/tcp open ssh`, 호스트 목록 | ☐ |
| tcpdump 캡처 | `-i enp0s1 -nn -c 10`, `port 22`, `-w cap.pcap` → `-r`, `-A`/`-X`, `'host x and not port 22'` | 패킷 출력, 파일 재생 | ☐ |
| arping·whois | `arping -I enp0s1 -c 3 192.168.64.1`, `whois 8.8.8.8` | `Unicast reply`, `OrgName` | ☐ |
| **3-way handshake 캡처** | `tcpdump -i lo -nn 'tcp port 9000' -c 3` + `nc -l 9000` / `nc 127.0.0.1 9000` | `[S]` → `[S.]` → `[.]` 3패킷 | ☐ |
| 4-way 종료·UDP 비교 | 종료 시 `[F.]`·`[.]` 캡처, `nc -u` 로 UDP | handshake 없음 확인 | ☐ |
| OSI·캡슐화 확인 | `tcpdump -e -c 2 icmp` | MAC + `ethertype IPv4` + IP 헤더 동시 | ☐ |
| 서브네팅 계산 | `ipcalc -n -b -m -p 192.168.64.10/24`(및 /25~/28) | 표의 네트워크·브로드캐스트 값과 일치 | ☐ |
| 포트 30종 검증 | `/etc/services` awk 루프 | `22 ssh`, `53 domain`, `123 ntp` 등 | ☐ |
| sshd 상태·설정 | `systemctl status sshd`, `sshd -t`, `sshd -T \| grep port` | `enabled`/`active`, 문법 OK | ☐ |
| 클라이언트 설정 | `~/.ssh/config` Host 별칭 → `ssh -G srv01` | `hostname/port/user/identityfile` | ☐ |
| **키 생성·배포** | `ssh-keygen -t ed25519 -C "mac->srv01"` → `ssh-copy-id -i …` | `ssh-keygen -lf` 지문 일치 | ☐ |
| 키 권한 확인 | `ls -ld ~/.ssh`, `ls -l ~/.ssh/authorized_keys`, `ls -Zd ~/.ssh` | 700 / 600 / `ssh_home_t` | ☐ |
| ssh-agent | `eval "$(ssh-agent -s)"` → `ssh-add` → `ssh-add -l` | 지문 표시, 패스프레이즈 재입력 없음 | ☐ |
| known_hosts 관리 | `ssh-keygen -F/-R <host>`, 서버 `ssh-keygen -lf /etc/ssh/ssh_host_*` | 지문 대조 일치 | ☐ |
| **비밀번호 인증 차단** | `PasswordAuthentication no` + `PermitRootLogin no` → `systemctl reload sshd` | `sshd -T` 확인, 비번 접속 `Permission denied` | ☐ |
| **SELinux 포트** | `semanage port -a -t ssh_port_t -p tcp 2222` | `semanage port -l \| grep ssh_port_t` → `2222, 22` | ☐ |
| **방화벽 포트** | `firewall-cmd --permanent --add-port=2222/tcp` + `--reload` | `firewall-cmd --list-ports` = `2222/tcp` | ☐ |
| **포트 2222 전환** | `Port 2222` → `systemctl restart sshd` → 22 제거 | `ss -tlnp \| grep sshd` = 2222 만, macOS `ssh -p 2222` 성공 | ☐ |
| 잠금 복구 절차 숙지 | UTM 콘솔 → `journalctl -u sshd` → `ausearch -m avc -ts recent` → 백업 복원 | 증상 3종 판정 가능 | ☐ |
| SSH 원격 실행 | `ssh -p 2222 host 'cmd'`, `ssh -t … sudo` | 출력 반환, `-t` 없이는 sudo 실패 | ☐ |
| 포트 포워딩 | `-L 8080:127.0.0.1:80`, `-R`, `-D 1080`, `-N -f` | `lsof -iTCP:8080` LISTEN (검증은 Part 09) | ☐ |
| **scp 양방향** | `scp -P 2222 -r -p -C` 업/다운로드 | `sha256sum -c SHA256SUMS` 전부 OK | ☐ |
| sftp | `sftp -P 2222` → `ls cd lcd put get mput mget bye` | 파일 도달, `sshd -T \| grep subsystem` | ☐ |
| **rsync 동기화** | `rsync -avz --delete -e "ssh -p 2222" …`, 후행 슬래시 비교, `-n` 예행 | `diff -r` 일치, 슬래시 유무 차이 확인 | ☐ |
| sshpass 위험 확인 | `sshpass -e ssh …` | 비밀번호 인증 차단으로 **실패**하는 것이 정상 | ☐ |
| **chrony 동기화** | `timedatectl set-ntp true`, `chronyc sources -v`, `tracking` | `^*` 소스 존재, `System clock synchronized: yes` | ☐ |
| hwclock·date | `hwclock -r`, `date -s '+2 seconds'` → `chronyc makestep` 복구 | 시각 복구, `Leap status: Normal` | ☐ |
| 참고 항목 정리 (※) | 본딩·브리지·VLAN·netns·무선·ifup/ifdown·DHCP·프록시 | 형식·개념 숙지 (미실행) | ☐ |
| **검증 스크립트** | `/usr/local/bin/check-part08.sh` 작성·실행 | `ALL OK` | ☐ |
| **재부팅 영구성** | `reboot` → `check-part08.sh` 재실행 | `ALL OK`, `systemctl --failed` 0 | ☐ |

---

## 기출 연결

| 기출 (문제집·회차·번호 또는 주제) | 이 파트의 단계 |
| --- | --- |
| 실기 r03-5, r05-5 — `eth0` 에 `192.168.x.x/24` 추가하는 `ip` 명령 작성 | 3-1 `ip addr add … dev …` |
| 실기 r01-5 — 8080 포트 LISTEN 프로세스 확인 `ss` 명령 | 5-3 `ss -tlnp` |
| 실기 r02-4 — ESTABLISHED TCP 연결을 숫자 표기로 조회하는 `ss` | 5-3 `ss -tan state established` |
| 실기 r03-8 — 53번 UDP LISTEN 소켓을 프로세스와 함께 조회 (`ss -ulnp`) | 5-3 옵션표, 6-7 포트 53 |
| 실기 r04-5 — 원격 host 에 user1 로 2222 포트 SSH 접속 명령 | 9-5, 10-1 `ssh -p 2222 <user>@<host>` |
| 실기 r05-9 — sshd 주 설정 파일 경로 (`/etc/ssh/sshd_config`) | 7-2 |
| 실기 r06-4 — 443 포트를 연결 없이 점검하는 `nc` 명령 (`nc -zv`) | 5-5 |
| 실기 r02-5 — `/data` 를 `user@host:/backup` 으로 권한·심링크 보존 압축 전송 (`rsync -avz`) | 10-7 |
| 실기 r02-9, r05-4 — firewalld `--permanent` + `--reload` 2단계 | 9-4 |
| 실기 r05-14 — SYN Flooding 원리와 방어 기법 2가지 | 6-1 handshake 캡처·시험 포인트, 5-3 `SYN-RECV` |
| 실기 r06-15 — ARP 스푸핑 원리·MITM·대응 | 5-10 arping·ARP 캐시, 1-3 `ip neigh` |
| 실기 r01-14, r06-7, r06-12 — DNS Zone 의 MX·A 레코드 차이, SOA Serial | 4-5 레코드 유형표 |
| 실기 r06-8 — IMAP 기본 포트 번호 | 6-7 포트 30종 표 |
| 필기 FULL r01-19, r05-14, r08-17 — TCP/UDP 가 동작하는 OSI 전송 계층 | 6-3 계층 대응표 |
| 필기 FULL r03-14, r07-20 — OSI 계층과 네트워크 장비 연결, 라우터의 계층 | 6-3 |
| 필기 FULL r07-12, r07-13 — 캡슐화 순서, PDU-계층 연결 | 6-3 캡슐화 도해 |
| 필기 FULL r01-20 — 255.255.255.192(/26) 할당 가능 호스트 수 | 6-4 서브네팅 표 |
| 필기 FULL r02-14 — 호스트 30대씩 균등 분할할 서브넷 마스크 (/27) | 6-4 |
| 필기 FULL r03-16 — VLSM 으로 100/60/20 호스트 분할 프리픽스 조합 | 6-4 VLSM |
| 필기 FULL r05-18 — `192.168.10.0/26` 호스트 수 | 6-4 |
| 필기 FULL r06-14, r06-15 — 6개 서브넷 25대 이상, `172.16.35.7/20` 네트워크 주소 | 6-4 (`ipcalc -n 172.16.35.7/20`) |
| 필기 FULL r10-18 — 호스트 500대 수용 최소 프리픽스·마스크 | 6-4 계산 공식 |
| 필기 FULL r05-16, r06-19, r07-19 — RFC 1918 사설 IP 대역 판별 | 6-5 사설·특수 주소표 |
| 필기 FULL r02-15, r03-20, r04-14 — TCP 헤더에만 있는 필드 / UDP 특징 | 6-2 TCP vs UDP 표 |
| 필기 FULL r07-9, r07-10 — 3-way handshake·4-way 종료 순서 나열 | 6-1, 6-2 실제 캡처 |
| 필기 FULL r03-15 — TCP 종료 시 수동 종료 측 상태(CLOSE_WAIT) | 5-3 TCP 상태표 |
| 필기 FULL r01-91, r08-96, r08-99 — SYN 대량 전송 공격·하프오픈 스캔 | 6-1 시험 포인트, 5-8 `-sS` |
| 필기 FULL r06-98 — `ss -tan state syn-recv \| wc -l` = 3842 로 의심되는 공격 | 5-3 `SYN-RECV`, 6-1 |
| 필기 FULL r02-16, r10-17 — IPv6 설명·주소 축약 표기 | 6-6 축약 규칙 |
| 필기 FULL r02-17, r04-18, r02-96 — ping 의 ICMP 타입, `-c` 옵션, ICMP 차단 | 5-1 |
| 필기 FULL r04-19, r08-12 — traceroute 동작 원리, `* * *` 해석 | 5-2 |
| 필기 FULL r08-11 — ping 통계(20% packet loss, rtt) 해석 | 5-1 기대 출력 |
| 필기 FULL r02-20, r08-10 — `ip addr show` 출력(mtu 1500, `/26`) 해석 | 1-1, 3-2 MTU |
| 필기 FULL r06-16, r08-13 — `ip route` 출력으로 기본 게이트웨이 누락 진단 | 1-3, 3-3 |
| 필기 FULL r04-20 — `netstat` 라우팅 테이블 출력 옵션 `-r` | 5-4 대응표 |
| 필기 FULL r08-14, r10-19, r03-96, r04-97, r09-91 — `arp -a` 해석, ARP 동작·스푸핑 | 1-3, 1-9, 5-10 |
| 필기 FULL r02-18, r03-95, r07-96, r09-13 — MAC 학습 장비(스위치)·스니핑 | 6-3 계층·장비표 |
| 필기 FULL r01-71, r06-17, r06-61 — `/etc/resolv.conf` 역할·DNS 실패 진단·`chattr +i` | 1-6, 4-6, 5-1 단계별 진단 |
| 필기 FULL r05-43, r07-11 — `/etc/nsswitch.conf` 의 `hosts: files dns` 의미·해석 흐름 | 1-6, 4-2 |
| 필기 FULL r04-16, r02-94, r08-5, r10-92 — `/etc/hosts` 역할·파밍 공격 | 4-1 |
| 필기 FULL r05-17, r08-15 — `/etc/services` 의 `ssh 22/tcp`·`smtp 25/tcp mail` 해석 | 1-8, 6-7 |
| 필기 FULL r01-74, r02-72, r04-70 — 역방향 조회 레코드(PTR) | 4-5 레코드표, `dig -x` |
| 필기 FULL r05-73, r07-74 — MX 레코드·메일 수신 서버 지정 | 4-5 |
| 필기 FULL r10-73 — CNAME 레코드 설명 | 4-5 |
| 필기 FULL r02-74, r08-71, r08-72 — `dig +short`·`dig -x`·`nslookup`·`host -t mx` 출력 해석 | 4-3, 4-4, 4-5 |
| 필기 FULL r10-75 — `dig +trace www.example.co.kr` 설명 | 4-5 `+trace` |
| 필기 FULL r03-70, r03-71, r07-73, r09-75 — DNS 질의 동작·존 전송·캐시 포이즈닝 | 4-4 섹션 해석 (구축은 Part 09) |
| 필기 FULL r01-75, r02-90, r03-17, r04-89, r05-77, r07-76, r10-20 — 서비스↔기본 포트 매칭 | 6-7 포트 30종 표 |
| 필기 FULL r04-49, r05-54, r06-48, r08-55 — CUPS 웹 관리 포트 631 | 6-7 |
| 필기 FULL r05-85, r05-87, r03-81, r07-84, r10-84 — FTP 능동 20번, DHCP 67/68·DORA | 6-7, 12-4 |
| 필기 FULL r01-84, r02-84, r05-86, r06-82 — `dhcpd.conf` 의 `option routers`·`range`·`fixed-address` | 12-4 (서버 구축은 시험 범위 참고) |
| 필기 FULL r04-17 — `DEVICE=eth0, BOOTPROTO=dhcp, ONBOOT=yes` 의미 | 2-4 keyfile↔ifcfg 대응표 |
| 필기 FULL r02-55, r06-93, r08-65 — `net.ipv4.ip_forward` 즉시/영구 설정 | 3-5 |
| 필기 FULL r09-65 — `icmp_echo_ignore_all`·`accept_redirects` 효과 | 3-5 관련 파라미터 |
| 필기 FULL r01-85, r06-99, r10-85 — SSH 설명·telnet 대비 우위·SFTP/SCP | 5-6, 10-5, 10-6 |
| 필기 FULL r09-15 — TCP 22 번을 쓰며 원격 접속을 암호화하는 프로토콜 | 7-1 |
| 필기 FULL r01-86, r08-100 — `sshd_config` 에서 root 원격 로그인 금지 | 7-2, 8-5 |
| 필기 FULL r02-65, r05-88, r06-83, r09-85, r10-64 — `Port 2222`/`PermitRootLogin no`/`PasswordAuthentication no` 해석 | 7-2 지시자표, 8-5, 9-2 |
| 필기 FULL r02-64, r07-59 — SSH 공개키 인증 절차 순서 (`ssh-keygen` → `authorized_keys`) | 8-1, 8-2 |
| 필기 FULL r05-89, r08-62, r09-86 — 서버에 공개키를 등록하는 파일(`authorized_keys`) | 8-2 권한표 |
| 필기 FULL r06-63 — 암호 인증 비활성화 전 키 인증 준비 (`ssh-keygen` → `ssh-copy-id`) | 8-1, 8-2, 8-5 |
| 필기 FULL r04-84 — `ssh` 접속 시 개인키 지정 옵션 `-i` | 8-1, 10-4 |
| 필기 FULL r04-85 — `ssh -L 8080:intranet.example.com:80 admin@gw` (로컬 포워딩) | 10-3 |
| 필기 FULL r06-12, r07-8, r10-15 — 원격 GUI 표시(`ssh -X`)·X 클라이언트 흐름 | 10-2 (※ 미실행) |
| 필기 FULL r06-11 — SSH 를 끊어도 스크립트가 계속 실행되게 하는 방법 | 10-1 시험 포인트 (nohup·tmux) |
| 필기 FULL r04-86 — `scp` 명령 설명 중 틀린 것 | 10-5 (`-P` 대문자 함정) |
| 필기 FULL r01-62, r03-61, r04-58, r06-42, r10-44 — `rsync -avz --delete` 해석·증분 전송 | 10-7 |
| 필기 FULL r01-57, r02-58, r06-43, r08-45 — `journalctl -u sshd` 로그 조회·`Failed password` | 7-1, 8-5 (Part 07 연계) |
| 필기 FULL r03-38 — `ps` 출력의 `/usr/sbin/sshd -D` 해석 | 7-1 |
| 필기 FULL r08-81, r07-65 — `ss -tnlp` 출력(`0.0.0.0:22 LISTEN sshd`) 해석·침해 초동 점검 | 5-3, 9-5 |
| 필기 FULL r08-94 — `firewall-cmd --list-all` 출력(services·ports) 해석 | 9-4 |
| 필기 FULL r01-99, r03-65, r03-100, r04-94, r06-97, r07-61, r07-98, r08-91 — `nmap -sS`/`-sT` 스캔 유형·`filtered` 해석 | 5-8 |
| 필기 FULL r02-100, r04-95, r08-92 — `tcpdump -i eth0 tcp port 80 -w web.pcap`·`Flags [S]` 해석 | 5-9, 6-1 플래그표 |
| 필기 FULL r08-66 — `curl -I` 응답(301 Moved Permanently, Location) 해석 | 5-7 |
| 필기 FULL r06-59 — 보기 중 `netcat` 의 용도 판별 | 5-5 |
| 필기 FULL r06-95 — NIC promiscuous 모드 점검(`ethtool`) | 1-10 |
| 필기 FULL r04-53 — 네트워크 카드 등 PCI 장치 목록(`lspci`) | 1-10 (Part 01 하드웨어 조회 연계) |
| 필기 FULL r09-44 — xinetd `telnet` 의 `only_from` 의미 (※ RHEL9 미제공) | 5-6, 12-3 |
| 실기 r01-12, r04-12, 필기 r01-93, r02-95, r04-92, r05-94, r07-92, r09-97, r10-94 — iptables 22 번 포트 규칙 해석 | 6-7 포트, 9-4 (상세는 Part 10) |
| 실기 r03-14, r06-14, 필기 r03-59, r05-91, r05-92, r07-62, r08-61, r09-43, r10-60 — TCP Wrapper `hosts.allow/deny` (※ RHEL9 미지원) | 12-3 (개념만, 상세는 Part 10) |
| 주제 — 고정 IP 설정 절차 서술 (`nmcli con mod` → `con up` → 검증) | 2-2, 2-3 |
| 주제 — SSH 포트 변경 시 SELinux·방화벽 동반 조치 서술 | 9-1 ~ 9-5 |
| 주제 — NTP 시간 동기화 데몬(chronyd)·`chronyc sources` 출력 해석 | 11-2, 11-3 |

---

## 이전 / 다음

[[07-boot-systemd-log]] ← · → [[09-network-services]]

[[README]]
