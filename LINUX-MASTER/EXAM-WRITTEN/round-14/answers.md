---
title: 필기 모의고사 Round 14 — 정답·해설
type: exam-written-answer
round: 14
tags:
  - exam/linux-master
  - exam/written
  - exam/answers
updated: 2026-08-28
---

# 필기 모의고사 Round 14 — 정답·해설

- 문제: [[questions]]
- ⚠ 반드시 풀이를 마친 후 열람

---

## 정답 요약

| 번호 | 정답 | 번호 | 정답 | 번호 | 정답 | 번호 | 정답 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | ③ | 6 | ② | 11 | ④ | 16 | ① |
| 2 | ② | 7 | ① | 12 | ① | 17 | ④ |
| 3 | ④ | 8 | ③ | 13 | ③ | 18 | ③ |
| 4 | ② | 9 | ④ | 14 | ② | 19 | ② |
| 5 | ② | 10 | ② | 15 | ① | 20 | ① |

---

## 해설

**1. ③** Apache 2.0 은 허용적 라이선스로 명시적 특허 사용 허가 조항을 포함하며 파생물의 비공개(상용) 배포를 허용. 소스 공개 강제는 GPL 계열.

**2. ②** Ubuntu 만 데비안 계열(dpkg/apt). Fedora·openSUSE·Rocky 는 RPM 계열(dnf/zypper).

**3. ④** systemd `timer` 유닛이 특정 시각·주기 실행을 담당해 cron 을 대체 가능(`OnCalendar` 등) → [[systemctl]] · [[crontab]].

**4. ②** 하드 링크는 원본과 동일 inode 를 공유하며 동일 파일시스템 내에서만, 일반 파일에만 생성. 원본을 지워도 링크 수가 남아 접근 가능.

**5. ②** `mkswap` 로 스왑 영역을 초기화한 뒤 `swapon` 으로 활성화. `free` 는 메모리 확인. `/etc/fstab` 등록 시 부팅 시 자동 활성화.

**6. ②** `PATH` 가 명령 실행 파일 검색 경로. PS1 은 프롬프트, HOME 은 홈 디렉터리, TERM 은 터미널 종류.

**7. ①** SIGHUP(1)=터미널 연결 종료 시 발생하며, 많은 데몬이 이를 설정 재적재 트리거로 사용. 2=인터럽트, 15=종료요청, 9=강제종료 → [[kill]].

**8. ③** D=인터럽트 불가 대기(uninterruptible sleep, 주로 디스크 I/O). R=실행, S=중단 가능 대기, T=중지 → [[ps]].

**9. ④** `su - user` 는 대상 사용자의 로그인 환경(환경변수·작업 디렉터리)까지 완전히 전환. `-` 가 없으면 환경 일부 유지 → [[sudo]] · [[system-security]] 2-5.

**10. ②** 데스크톱 환경(GNOME·KDE)이 창 관리·패널·파일관리자 등을 통합 제공. 윈도 매니저는 창 배치만, 디스플레이 매니저는 로그인 화면.

**11. ④** 172.16.0.0/12(172.16~172.31)은 B 클래스 대역의 사설 IP. A=10/8, C=192.168/16, D=멀티캐스트(224~239).

**12. ①** Smurf 는 출발지를 피해자로 위조한 ICMP 요청을 브로드캐스트로 보내 다수 응답으로 증폭 → [[network-security]] 1-2.

**13. ③** 포트포워딩(목적지 변환)은 `nat` 테이블 PREROUTING 체인에서 DNAT 로 처리 → [[iptables]] · [[network-security]] 2-2.

**14. ②** 이상 탐지(anomaly)는 정상 행위 프로파일에서 벗어남을 탐지해 미지 공격에 유리(오탐 많음). 오용 탐지는 알려진 시그니처 기반 → [[network-security]] 3.

**15. ①** Kerberos 는 KDC·TGT 기반 티켓 인증 프로토콜. SSL/TLS·IPSec·PGP 와 구분 → [[network-security]] 4-2.

**16. ①** `setfacl` 로 확장 ACL(개별 사용자·그룹 권한) 부여, `getfacl` 로 조회. chmod 는 기본 3주체 권한, chattr 는 파일 속성.

**17. ④** `/var/log/secure` 는 ssh·su·sudo·login 등 인증·보안 이벤트 기록. 커널 부팅은 dmesg/messages, 메일은 maillog, cron 은 cron 로그 → [[system-security]] 1-1 · [[journalctl]].

**18. ③** fail2ban 이 로그의 반복 실패를 감지해 해당 IP 를 iptables 로 자동 차단. Tripwire=무결성, John=크래킹, Nessus=취약점 스캐너 → [[system-security]] 2-8 · [[iptables]].

**19. ②** permissive 는 정책 위반을 차단하지 않고 감사 로그만 남김(정책 튜닝용). enforcing=강제 차단, disabled=비활성 → [[getenforce]].

**20. ①** `chage -M 90 user` 로 최대 사용기간(Maxdays) 90일 설정. `-m` 은 최소기간, `usermod -L`·`passwd -l` 은 계정/비밀번호 잠금 → [[system-security]] 2-7 · [[usermod]].

---

## 오답 복습 링크
- 인증 로그·ACL·SELinux·계정 정책 → [[system-security]]
- 침해 유형·DNAT·IDS·Kerberos → [[network-security]]
- 시그널·프로세스 상태 → [[kill]] · [[ps]]
- 타이머 유닛 → [[systemctl]]
