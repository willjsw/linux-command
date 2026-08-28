---
title: 네트워크 보안
type: exam-theory
category: LINUX-MASTER
tags:
  - exam/linux-master
  - exam/theory
  - linux/network
  - linux/security
  - topic/firewall
  - topic/security
  - topic/intrusion
  - task/diagnose
related: ["[[system-security]]", "[[network-service]]", "[[iptables]]", "[[firewall-cmd]]", "[[nc]]", "[[whois]]", "[[ss]]"]
updated: 2026-08-28
---

# 네트워크 보안

- 리눅스마스터 1급 필기 대비 이론 — 침해 유형·특징·대비·대처
- 공격 명칭 ↔ 계층 ↔ 대응 기법 매칭, iptables 규칙 해석이 최빈출

---

## 1. 침해 공격 유형

### 1-1. 스캐닝·정찰

| 공격    | 특징                                                               |
| ----- | ---------------------------------------------------------------- |
| 포트 스캔 | 열린 포트·서비스 탐지 (`nmap`) — SYN 스캔(Half-open), FIN·NULL·XMAS 스캔(스텔스) |
| 풋프린팅  | 대상 정보 사전 수집 (whois·DNS·구글 해킹) → [[whois]]                        |
| 스니핑   | 네트워크 패킷 도청 (수동 공격) — 스위치 환경은 ARP 스푸핑 선행                          |

### 1-2. 서비스 거부 (DoS / DDoS)

| 공격            | 원리                                 |
| ------------- | ---------------------------------- |
| SYN Flooding  | TCP 3-way 중 SYN 만 대량 전송 → 백로그 큐 고갈 |
| Smurf         | 브로드캐스트로 ICMP 증폭 (출발지 위조)           |
| Ping of Death | 규격 초과 ICMP 패킷으로 시스템 마비             |
| Teardrop      | 조작된 프래그먼트 오프셋으로 재조합 오류             |
| Land          | 출발지=목적지 동일 위조 패킷                   |
| DDoS          | 다수 좀비(봇넷) 동원 분산 공격 (핸들러·에이전트 구조)   |

- 방어: SYN 쿠키, 백로그 큐 증대, 방화벽 rate limit, 이상 트래픽 탐지

### 1-3. 스푸핑·중간자

| 공격 | 특징 |
| --- | --- |
| IP 스푸핑 | 출발지 IP 위조 → 신뢰 관계 악용 |
| ARP 스푸핑 | ARP 캐시 위조 → 스위치 환경 스니핑·MITM |
| DNS 스푸핑 | 위조 응답으로 악성 사이트 유도 (파밍) |
| 세션 하이재킹 | 인증된 세션 탈취 (시퀀스 번호 예측·TCP 재설정) |

### 1-4. 애플리케이션·웹 공격

| 공격 | 특징 |
| --- | --- |
| 버퍼 오버플로 | 경계 검사 미흡으로 스택·힙 덮어써 코드 실행 |
| 포맷 스트링 | `printf` 등 서식 문자열 취약점 |
| 레이스 컨디션 | 검사·사용 시점 차이(TOCTOU) 악용 — SetUID 임시파일 표적 |
| SQL Injection | 입력값에 SQL 삽입 → DB 조작 |
| XSS | 악성 스크립트 삽입 → 클라이언트 실행 |
| CSRF | 인증된 사용자 권한으로 위조 요청 |
| 백도어 | 인증 우회 은닉 통로 (rootkit 동반) |

---

## 2. 방화벽 — iptables / firewalld

### 2-1. iptables 구조

- 테이블: `filter` (기본, 필터링) / `nat` (주소 변환) / `mangle` (패킷 변조) / `raw`
- 체인: `INPUT` (수신) / `OUTPUT` (송신) / `FORWARD` (경유) / `PREROUTING` / `POSTROUTING`
- 정책(Policy): 기본 처리 — `ACCEPT` / `DROP` (응답 없이 폐기) / `REJECT` (거부 응답)

```bash
# 정책 설정
iptables -P INPUT DROP                                   # INPUT 기본 차단 (P = Policy)
# 규칙 추가 (A = Append)
iptables -A INPUT -p tcp --dport 22 -s 192.168.0.0/24 -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
# 규칙 조회 (L = List, n = 숫자표기, v = 상세)
iptables -L -n -v --line-numbers
# 규칙 삭제 (D = Delete)
iptables -D INPUT 3
```

| 옵션 | 의미 |
| --- | --- |
| `-A` / `-I` | 규칙 추가(끝) / 삽입(지정 위치) |
| `-D` / `-F` | 삭제 / 전체 초기화 (**F**lush) |
| `-P` | 체인 기본 정책 |
| `-p` | 프로토콜 (tcp/udp/icmp) |
| `-s` / `-d` | 출발지 / 목적지 IP |
| `--sport` / `--dport` | 출발지 / 목적지 포트 |
| `-j` | 처리 방식 (ACCEPT/DROP/REJECT/LOG/DNAT/SNAT) |
| `-i` / `-o` | 입력 / 출력 인터페이스 |
| `-m state` | 연결 상태 추적 모듈 |

- 규칙은 **위에서 아래 순차 매칭**, 먼저 매칭된 규칙 적용 → 순서가 결과 결정
- 실사용·침해 차단 사례는 [[iptables]]

### 2-2. NAT

- SNAT (출발지 변환, 내부→외부): `iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -j SNAT --to 1.1.1.1`
- MASQUERADE (동적 IP SNAT): `-j MASQUERADE`
- DNAT (목적지 변환, 포트포워딩): `iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to 192.168.0.10:8080`

### 2-3. firewalld

- iptables 상위 추상화 — **zone** 기반, 런타임/영구 분리 → [[firewall-cmd]]
- 대표 명령: `firewall-cmd --add-service=http --permanent` 후 `--reload`
- zone: `public` `trusted` `internal` `dmz` `drop` `block`

---

## 3. 침입 탐지·차단 (IDS / IPS)

| 구분 | 특징 |
| --- | --- |
| IDS | 침입 **탐지**·경보 (수동) — 오탐(False Positive)·미탐(False Negative) |
| IPS | 침입 **탐지 + 차단** (능동) |
| HIDS | 호스트 기반 (Tripwire·AIDE — 파일 무결성) |
| NIDS | 네트워크 기반 (Snort·Suricata — 패킷 시그니처) |

- 탐지 방식: 오용 탐지(시그니처 기반, 알려진 공격) / 이상 탐지(비정상 행위 프로파일, 미지 공격)
- Snort: 룰 기반 NIDS, `alert tcp any any -> 192.168.0.0/24 80 (msg:"..."; sid:1000001;)`
- fail2ban: 로그 감시 → 반복 실패 IP 를 iptables 로 자동 차단 → [[system-security]]

---

## 4. 암호화·보안 프로토콜

### 4-1. 암호 방식

| 구분 | 특징 | 알고리즘 |
| --- | --- | --- |
| 대칭키 | 동일 키 암복호, 빠름, 키 분배 문제 | DES, 3DES, AES, SEED, ARIA |
| 비대칭키 | 공개키/개인키 쌍, 키 분배 해결, 느림 | RSA, ECC, DSA, ElGamal |
| 해시 | 단방향, 무결성 검증 | MD5, SHA-1, SHA-256 → [[sha256sum]] |

- 공개키 암호 응용: 기밀성(수신자 공개키로 암호), 전자서명(송신자 개인키로 서명)

### 4-2. 보안 프로토콜

| 프로토콜 | 계층·용도 |
| --- | --- |
| SSL/TLS | 전송 계층 암호화 (HTTPS 443) |
| SSH | 원격 접속 암호화 (22) → [[ssh]] |
| IPSec | 네트워크 계층 VPN (AH 무결성 / ESP 기밀성) |
| Kerberos | 티켓 기반 인증 (KDC·TGT) |
| PGP / GPG | 메일 암호화·서명 |

---

## 5. 침해 대응·분석 절차

- 흐름: 탐지 → 격리(연결 차단) → 증거 보존 → 원인 분석 → 복구 → 재발 방지
- 로그 상관 분석: `/var/log/secure` (인증) + `wtmp`/`btmp` (접속) → [[last]] · [[system-security]]
- 외부 관점 검증: [[nc]] (포트 도달), [[whois]] (공격 IP 소속), [[ss]] (현재 연결)
- 프로세스·네트워크 실시간: `netstat -antp` / `ss -antp`, 이상 프로세스 → [[ps]]
- rootkit 점검 도구: `chkrootkit`, `rkhunter`
- 무결성 기준선 비교: Tripwire·AIDE (사전 DB 필수)

---

## 연관 문서
- [[system-security]] — 로그·PAM·보안 도구
- [[network-service]] — 서비스별 취약점 맥락
- [[iptables]] — 방화벽 규칙 실사용 문서
- [[firewall-cmd]] — firewalld 실사용 문서
- [[nc]] · [[whois]] — 외부 관점 점검·공격 IP 조회
- [[ss]] — 연결 상태 조회
- [[sha256sum]] — 해시·무결성
