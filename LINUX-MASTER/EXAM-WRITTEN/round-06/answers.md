---
title: 필기 모의고사 Round 06 — 정답·해설
type: exam-written-answer
round: 6
tags:
  - exam/linux-master
  - exam/written
  - exam/answers
updated: 2026-08-28
---

# 필기 모의고사 Round 06 — 정답·해설

- 문제: [[questions]]
- ⚠ 반드시 풀이를 마친 후 열람

---

## 정답 요약

| 번호 | 정답 | 번호 | 정답 | 번호 | 정답 | 번호 | 정답 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | ① | 6 | ② | 11 | ③ | 16 | ② |
| 2 | ④ | 7 | ④ | 12 | ④ | 17 | ① |
| 3 | ① | 8 | ① | 13 | ③ | 18 | ② |
| 4 | ③ | 9 | ① | 14 | ③ | 19 | ③ |
| 5 | ② | 10 | ④ | 15 | ④ | 20 | ② |

---

## 해설

**1. ①** FHS(Filesystem Hierarchy Standard)가 `/etc` `/var` `/usr` 등 디렉터리 구조를 규정. POSIX 는 API·셸 표준, LSB 는 바이너리 호환 표준, GNU 는 프로젝트명.

**2. ④** 런레벨 0=정지(halt), 1=단일 사용자, 3=텍스트 다중, 5=GUI, 6=재부팅. `init 6` = reboot → [[reboot]].

**3. ①** `dnf`(구 yum)는 저장소 메타데이터로 의존성을 자동 해결. `rpm`·`dpkg` 는 단일 패키지 설치로 의존성 수동 처리 → [[dnf]] · [[rpm]].

**4. ③** `env`(또는 `printenv`)는 환경변수만 출력. `set` 은 셸 변수까지 모두 표시, `alias` 는 별칭 목록, `jobs` 는 작업 목록 → [[env]].

**5. ②** `<` 는 입력 리다이렉션으로 표준입력을 파일 내용으로 대체. `wc` 가 파일명을 인자로 받지 않고 stdin 을 읽으므로 출력에 파일명이 붙지 않음 → [[wc]].

**6. ②** `grep 404` 로 404 를 포함한 행만 걸러 `wc -l` 이 그 **행 수**를 셈. 파이프는 앞 명령의 표준출력을 뒤 명령의 표준입력으로 연결 → [[grep]] · [[wc]].

**7. ④** 교대(alternation) `|` 는 확장 정규표현식이므로 `-E`(또는 `egrep`) 필요. 기본 grep 은 `|` 를 일반 문자로 처리, `-F` 는 고정 문자열, `-w` 는 단어 단위 → [[grep]].

**8. ①** 구분자가 콜론이므로 `-F:` 로 지정해야 `$1` 이 사용자명이 됨. `-F` 없으면 공백 기준이라 전체가 한 필드. `{print 1}` 은 숫자 1 만 출력 → [[awk]].

**9. ①** 단일 `&` 는 백그라운드 실행. `;` 는 순차 실행, `&&` 는 앞 명령 성공 시 실행, `|` 는 파이프.

**10. ④** 디스플레이 매니저(gdm·kdm·xdm 등)가 그래픽 로그인·세션 시작을 담당. 윈도 매니저는 창 제어, 데스크톱 환경은 통합 GUI 묶음.

**11. ③** MAC 주소는 48비트(6바이트) — 앞 24비트 OUI(제조사) + 뒤 24비트. 데이터링크 계층 주소.

**12. ④** `ping` 은 ICMP Echo Request/Reply 로 도달성·지연을 진단. `dig` 는 DNS, `ssh` 는 원격 접속, `ftp` 는 파일 전송 → [[ping]].

**13. ③** 연결 수립은 SYN → SYN/ACK → ACK 3단계. 종료는 FIN/ACK 4단계 → [[network-security]] 1-2.

**14. ③** `at` 은 일회성 예약 실행. `cron`/`crontab` 은 반복 스케줄, `anacron` 은 놓친 주기 보완, `batch` 는 부하 낮을 때 실행 → [[crontab]].

**15. ④** `g+w` 는 그룹에 쓰기 추가. `u`=소유자, `o`=기타, `a`=전체. `+` 부여, `-` 제거.

**16. ②** `df`(disk free)는 파일시스템 단위 사용량·여유 공간. `du` 는 디렉터리·파일별 사용량, `lsblk` 는 블록 장치 트리 → [[df]] · [[du]].

**17. ①** Sticky Bit(1000)를 디렉터리에 설정하면 소유자·root 만 자기 파일 삭제 가능(`drwxrwxrwt`). `/tmp` 가 대표 예 → [[system-security]] 2-1.

**18. ②** NFS 공유 정의는 `/etc/exports`(`rw`,`sync`,`root_squash` 등). 반영은 `exportfs -ra`. `smb.conf` 는 Samba 용 → [[network-service]] 4-1.

**19. ③** SYN Flooding 은 SYN 만 대량 전송해 반개방 연결로 백로그 큐 고갈. 방어는 SYN 쿠키·큐 증대. Smurf 는 ICMP 증폭 → [[network-security]] 1-2.

**20. ②** `getenforce` 로 현재 모드 조회, `setenforce 0|1` 로 일시 전환, 영구 설정은 `/etc/selinux/config` → [[getenforce]].

---

## 오답 복습 링크
- 특수권한 → [[system-security]]
- 서비스 설정·NFS → [[network-service]]
- DoS·handshake → [[network-security]]
- 텍스트 처리 → [[grep]] · [[awk]] · [[wc]]
