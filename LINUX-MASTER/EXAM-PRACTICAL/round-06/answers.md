---
title: 실기 모의고사 Round 06 — 정답·해설
type: exam-practical-answer
round: 6
tags:
  - exam/linux-master
  - exam/practical
  - exam/answers
updated: 2026-08-28
---

# 실기 모의고사 Round 06 — 정답·해설

- 문제: [[questions]]
- ⚠ 반드시 풀이를 마친 후 열람

---

## 1부. 명령어 작성

**1.**
```bash
sed -i 's/Listen 80/Listen 8080/' /etc/httpd/conf/httpd.conf
```
- `-i` : 파일 직접(in-place) 수정. `s/찾을것/바꿀것/` 치환. 원본 백업은 `-i.bak` → [[sed]] · [[network-service]] 1-1

**2.**
```bash
iptables -P INPUT DROP
```
- `-P` : 체인 **기본 정책**(**P**olicy) 설정. 어떤 규칙에도 매칭되지 않은 패킷의 최종 처리. 실무에서는 SSH 허용 규칙을 먼저 넣고 DROP 하지 않으면 원격 접속이 끊길 수 있음 ⚠ → [[iptables]] · [[network-security]] 2-1

**3.**
```bash
vgcreate vg01 /dev/sdc
```
- LVM 순서: `pvcreate`(물리 볼륨) → `vgcreate`(볼륨 그룹) → `lvcreate`(논리 볼륨). 문제는 PV 완료 가정이므로 VG 생성 단계 → [[lvm]]

**4.**
```bash
nc -zv 192.168.0.10 443
```
- `-z` : 데이터 전송 없이 포트 스캔(zero-I/O), `-v` : 결과 상세 출력. 도달 점검용 → [[nc]] · [[network-security]] 5

**5.**
```bash
awk -F: '$3>=1000 {print $1}' /etc/passwd
```
- `-F:` : 필드 구분자를 `:` 로 지정. `$3` 은 UID, `$1` 은 계정명. 일반 사용자 UID 는 보통 1000 이상 → [[awk]] · [[system-security]] 2-7

---

## 2부. 빈칸

**6.** `httpd` — `httpd -t` 또는 `apachectl -t`(=`apachectl configtest`) 로 문법 검사. `apachectl` 도 정답 → [[network-service]] 1-2

**7.** `Serial` — 시리얼 번호. 마스터에서 증가해야 슬레이브가 변경을 감지해 존 전송(AXFR/IXFR)을 수행. 통상 `YYYYMMDDNN` 형식 → [[network-service]] 2-2

**8.** `143` — IMAP 143(서버 보관·동기화). POP3 는 110, SMTP 는 25 → [[network-service]] 3

**9.** `state` — 연결 상태 추적 모듈. `-m state --state ESTABLISHED,RELATED` 로 응답 트래픽 허용 → [[iptables]] · [[network-security]] 2-1

**10.** `NO` — `anonymous_enable=NO` 로 익명 접속 차단. 로컬 계정 허용은 `local_enable=YES`, 업로드 허용은 `write_enable=YES` → [[network-service]] 4-3

---

## 3부. 서술

**11.**
- 동작: 80 포트로 들어온 요청 중 호스트 헤더가 `www.example.com` 인 요청을 이 가상 호스트가 처리하며, 문서를 `/var/www/example` 에서 제공
- `ServerName` : 이 가상 호스트가 응답할 도메인명(요청 Host 헤더와 매칭)
- `DocumentRoot` : 해당 도메인의 웹 문서가 위치한 최상위 디렉터리
- 방식: `*:80` 하나의 IP/포트에서 도메인명으로 구분 → **이름 기반(name-based) 가상 호스트** → [[network-service]] 1-2

**12.**
- `Serial` : 존 데이터의 버전 번호. 슬레이브는 마스터 Serial 이 자신보다 크면 존을 다시 전송받음
- `Refresh` : 슬레이브가 마스터에 갱신 여부를 확인하는 주기(초)
- `Expire` : 슬레이브가 마스터와 통신 불가일 때 보유 존 데이터를 유효로 유지하는 최대 시간(초) — 초과 시 응답 중단
- Serial 을 올려야 하는 이유: 슬레이브는 **Serial 값 비교**로만 변경을 인지하므로, 존을 수정하고도 Serial 을 올리지 않으면 슬레이브가 낡은 데이터를 그대로 유지 → [[network-service]] 2-2

**13.**
- `nat` 테이블의 `PREROUTING` 체인(라우팅 판단 전)에 규칙 추가
- 목적지 TCP 80 포트로 들어온 패킷의 **목적지 주소·포트를 `192.168.0.10:8080` 으로 변경**(DNAT)
- 결과: 외부에서 80 포트로 온 요청을 내부 서버 `192.168.0.10` 의 8080 으로 전달하는 **포트 포워딩** → [[iptables]] · [[network-security]] 2-2

**14.**
- 검사 순서: `/etc/hosts.allow` **먼저** 검사 → 매칭되면 즉시 허용, 없으면 `/etc/hosts.deny` 검사 → 매칭되면 거부. 양쪽 모두 없으면 허용. **allow 우선**
- 해석: `192.168.0.` 대역(192.168.0.0/24)에서의 `sshd` 접속은 allow 에서 매칭되어 **허용**, 그 외 모든 호스트·서비스는 deny 의 `ALL : ALL` 에 걸려 **거부** → [[system-security]] 2-6

**15.**
- 원리: 공격자가 위조된 ARP 응답을 뿌려 피해자의 ARP 캐시에서 게이트웨이(또는 대상) MAC 을 **공격자 MAC 으로 오염**시킴
- 연계: 스위치 환경에서도 피해자 트래픽이 공격자를 경유하게 되어 패킷 도청(스니핑)·변조가 가능해지고, 양방향을 가로채면 중간자(MITM) 성립
- 대응(1가지 이상): 중요 게이트웨이 ARP **정적(static) 등록**, ARP 스푸핑 탐지 도구(arpwatch) 운용, 스위치 포트 보안(DAI) → [[network-security]] 1-3

---

## 채점 기준
- 1부: 명령·옵션 정확 일치 (동등 대체 명령 인정)
- 2부: 정확한 값·키워드
- 3부: 핵심 개념 포함 시 정답 (표현 차이 무관)
