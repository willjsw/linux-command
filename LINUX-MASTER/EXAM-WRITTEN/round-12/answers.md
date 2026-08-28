---
title: 필기 모의고사 Round 12 — 정답·해설
type: exam-written-answer
round: 12
tags:
  - exam/linux-master
  - exam/written
  - exam/answers
updated: 2026-08-28
---

# 필기 모의고사 Round 12 — 정답·해설

- 문제: [[questions]]
- ⚠ 반드시 풀이를 마친 후 열람

---

## 정답 요약

| 번호 | 정답 | 번호 | 정답 | 번호 | 정답 | 번호 | 정답 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | ③ | 6 | ① | 11 | ③ | 16 | ① |
| 2 | ④ | 7 | ④ | 12 | ④ | 17 | ② |
| 3 | ② | 8 | ③ | 13 | ① | 18 | ② |
| 4 | ③ | 9 | ② | 14 | ③ | 19 | ③ |
| 5 | ① | 10 | ② | 15 | ④ | 20 | ④ |

---

## 해설

**1. ③** 리처드 스톨먼이 GNU 프로젝트·FSF·GPL 창시. 리누스 토르발스는 커널 개발자, 데니스 리치는 C·UNIX, 귀도 반 로섬은 파이썬.

**2. ④** `dpkg -i pkg.deb` 는 데비안 저수준 설치(의존성 자동 해결 없음). `apt` 는 상위 래퍼, `rpm`·`dnf` 는 RHEL 계열.

**3. ②** PV(물리 볼륨) → VG(볼륨 그룹, 묶음) → LV(논리 볼륨, 할당). PE 는 VG 를 나눈 최소 할당 단위.

**4. ③** 심볼릭 링크는 경로를 가리키므로 다른 파일시스템·디렉터리에도 생성 가능. 하드 링크가 동일 inode 공유·동일 fs 제한. 원본 삭제 시 심볼릭 링크는 깨짐.

**5. ①** `systemctl status <유닛>` 으로 활성·PID·최근 로그 확인. `service`·`chkconfig` 는 구형 SysV 도구 → [[systemctl]].

**6. ①** `chsh` (change shell)로 `/etc/passwd` 7번째 필드(로그인 셸) 변경. `chage` 는 비밀번호 만료 정책.

**7. ④** `jobs` 로 현재 셸의 작업 목록 확인. `bg`/`fg` 는 백그라운드/포그라운드 전환, `nohup` 은 HUP 무시 실행.

**8. ③** Z(zombie) = 종료됐으나 부모가 wait 로 회수하지 않은 상태. R=실행, S=대기(interruptible), D=불가중단 대기.

**9. ②** `nohup cmd &` 로 SIGHUP 을 무시해 로그아웃 후에도 지속 실행. 출력은 기본 `nohup.out` 에 기록.

**10. ②** `xhost +/-` 로 X 서버 접근 제어(접근 목록 관리). `startx`·`xinit` 는 X 세션 시작, DISPLAY 는 출력 대상 지정.

**11. ③** TCP/IP 4계층에서 IP·ICMP 는 인터넷 계층. 응용(HTTP 등), 전송(TCP/UDP), 네트워크 접근(이더넷)과 구분.

**12. ④** 143 은 IMAP, POP3 는 110. 나머지 SSH 22·SMTP 25·DNS 53 은 정상 → [[network-service]] 0 · [[ss]].

**13. ①** 출발지 IP 위조로 신뢰 기반 접근을 악용하는 것이 IP 스푸핑 → [[network-security]] 1-3.

**14. ③** 공인 IP 가 동적으로 바뀌는 환경은 `MASQUERADE`(동적 SNAT). 고정 IP 는 `SNAT --to` → [[iptables]] · [[network-security]] 2-2.

**15. ④** Tripwire 는 파일 무결성 기반 HIDS. Snort·Suricata 는 NIDS, Nmap 은 포트 스캐너 → [[network-security]] 3.

**16. ①** `/etc/skel` 의 파일들이 새 계정 홈으로 복사됨. `/etc/login.defs` 는 UID·비밀번호 정책, `/etc/default/useradd` 는 기본값.

**17. ②** 현재 로그인 상태는 `/var/run/utmp`(=`/run/utmp`), `w`·`who`·`users` 로 조회. 이 파일만 `/var/log` 밖에 위치 → [[system-security]] 1-1.

**18. ②** `visudo` 는 편집 후 문법 검증·잠금을 제공해 sudoers 오류로 인한 권한 사고를 예방 → [[sudo]] · [[system-security]] 2-5.

**19. ③** `compress` 로 순환된 로그를 gzip 압축. `rotate N` 은 보관 세대수, `notifempty` 는 빈 파일 미순환 → [[system-security]] 1-3.

**20. ④** dump 레벨 0 = 전체 백업, 1~9 = 이전 하위 레벨 이후 증분. 일 0 → 월 1 … 스케줄 계산 출제 → [[system-security]] 3-2.

---

## 오답 복습 링크
- 로그 파일·sudo·logrotate·백업 → [[system-security]]
- 포트·서비스 → [[network-service]]
- 스푸핑·NAT·IDS → [[network-security]]
- 서비스 상태 → [[systemctl]]
