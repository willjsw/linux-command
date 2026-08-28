---
title: 실기 모의고사 Round 06 — 문제
type: exam-practical
round: 6
tags:
  - exam/linux-master
  - exam/practical
  - exam/questions
updated: 2026-08-28
---

# 실기 모의고사 Round 06 — 문제

- 15문항 / 명령어 작성·빈칸·설정 서술형
- 종합 실전 난이도 — 네트워크 서비스 설정 파일·보안 서술 비중 상향
- 채점·해설: [[answers]] (풀이 완료 후 열람)
- 답안은 명령어·옵션 정확 표기 기준 채점

---

## 1부. 명령어 작성 (직접 서술)

**1.** `/etc/httpd/conf/httpd.conf` 파일에서 `Listen 80` 을 `Listen 8080` 으로 **파일에 직접(in-place) 치환** 저장하는 `sed` 명령을 작성하시오.

**2.** iptables 에서 `INPUT` 체인의 **기본 정책(policy)을 DROP** 으로 설정하는 명령을 작성하시오.

**3.** 물리 디스크 `/dev/sdc` 를 사용해 볼륨그룹 `vg01` 을 생성하는 LVM 명령을 작성하시오. (물리 볼륨은 이미 생성되어 있다고 가정)

**4.** 원격 호스트 `192.168.0.10` 의 `443` 포트가 열려 있는지 실제 연결은 맺지 않고 점검하는 `nc` 명령을 작성하시오.

**5.** `/etc/passwd` 에서 **UID(3번째 필드)가 1000 이상**인 계정의 **사용자명만** 출력하는 `awk` 명령을 작성하시오.

---

## 2부. 빈칸 채우기

**6.** Apache 설정 파일의 문법 오류를 반영 전에 검사하는 명령 (두 가지 중 하나):
`______ -t`

**7.** BIND 존(zone) 파일을 수정한 뒤, 슬레이브로의 존 전송을 트리거하기 위해 **반드시 증가**시켜야 하는 SOA 레코드 항목의 이름:
`______`

**8.** IMAP 프로토콜이 사용하는 기본 포트 번호:
`______`

**9.** iptables 에서 이미 성립된 연결(`ESTABLISHED,RELATED`)을 허용하기 위해 연결 상태 추적에 사용하는 매치 모듈:
`iptables -A INPUT -m ______ --state ESTABLISHED,RELATED -j ACCEPT`

**10.** vsftpd 에서 **익명 접속을 차단**하는 설정 항목:
`anonymous_enable=______`

---

## 3부. 설정·개념 서술

**11.** 다음 Apache 가상 호스트 블록이 어떤 동작을 하는지, `ServerName` 과 `DocumentRoot` 의 역할을 포함해 서술하시오. 또한 이 방식이 이름 기반인지 IP 기반인지 밝히시오.
```apache
<VirtualHost *:80>
    ServerName www.example.com
    DocumentRoot /var/www/example
</VirtualHost>
```

**12.** DNS Zone 파일의 `SOA` 레코드에서 `Serial`, `Refresh`, `Expire` 세 항목의 의미를 서술하고, 마스터 존 수정 시 `Serial` 을 왜 반드시 올려야 하는지 설명하시오.

**13.** 다음 iptables NAT 규칙이 수행하는 동작을 서술하시오. (어떤 테이블·체인이며 최종 결과가 무엇인지 포함)
```
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to 192.168.0.10:8080
```

**14.** TCP Wrapper 의 `/etc/hosts.allow` 와 `/etc/hosts.deny` 검사 순서·우선 규칙을 서술하고, 아래 두 설정의 최종 접근 통제 결과를 해석하시오.
```
# /etc/hosts.allow
sshd : 192.168.0.
# /etc/hosts.deny
ALL : ALL
```

**15.** `ARP 스푸핑` 공격의 원리와, 이를 통한 스니핑·중간자(MITM) 연계 방식을 서술하고 대응 방안 1가지를 제시하시오.
