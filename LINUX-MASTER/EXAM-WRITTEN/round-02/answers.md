---
title: 필기 모의고사 Round 02 — 정답·해설
type: exam-written-answer
round: 2
tags:
  - exam/linux-master
  - exam/written
  - exam/answers
updated: 2026-08-28
---

# 필기 모의고사 Round 02 — 정답·해설

- 문제: [[questions]]
- ⚠ 반드시 풀이를 마친 후 열람

---

## 정답 요약

| 번호 | 정답 | 번호 | 정답 | 번호 | 정답 | 번호 | 정답 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | ② | 6 | ② | 11 | ① | 16 | ① |
| 2 | ③ | 7 | ② | 12 | ① | 17 | ④ |
| 3 | ② | 8 | ① | 13 | ② | 18 | ② |
| 4 | ④ | 9 | ③ | 14 | ① | 19 | ② |
| 5 | ③ | 10 | ① | 15 | ② | 20 | ② |

---

## 해설

**1. ②** 리처드 스톨먼이 1983년 GNU 프로젝트를 시작하고 FSF·GPL을 창시. 리누스 토르발스는 커널 개발자, 데니스 리치·브라이언 커니핸은 C 언어·유닉스 관련 인물.

**2. ③** BSD 는 허용적(permissive) 라이선스로 수정본의 소스 비공개·독점 배포를 허용. GPL·AGPL 은 강한 카피레프트, LGPL 도 라이브러리 수정분은 공개 의무.

**3. ②** 데비안 계열은 `dpkg -l` 로 설치 패키지를 조회. `rpm -qa` `dnf` `yum` 은 RHEL 계열.

**4. ④** `csh`(및 `tcsh`)는 C 셸 계열. `sh`·`bash`·`ksh` 는 본 셸 계열.

**5. ③** 9번 SIGKILL 은 프로세스가 잡거나 무시할 수 없는 강제 종료 시그널. 15번 SIGTERM 은 정상 종료 요청(무시 가능) → [[ps]].

**6. ②** `$DISPLAY` 는 X 클라이언트 출력을 표시할 `호스트:디스플레이.스크린` 을 지정. 원격 X 전달의 핵심 변수.

**7. ②** `224` = `11100000` → 상위 3비트 → 24 + 3 = `/27`. (참고: `/26`=192, `/28`=240)

**8. ①** well-known 포트 0~1023, registered 1024~49151, dynamic/private 49152~65535 → [[ss]].

**9. ③** IPv6 는 128비트(16진수 8그룹). IPv4 는 32비트.

**10. ①** crontab 필드 순서는 `분 시 일 월 요일`, 일요일은 0(또는 7). 따라서 `30 3 * * 0` → [[crontab]].

**11. ①** LVM 은 물리 볼륨(PV)을 묶어 볼륨 그룹(VG)을 만들고, VG 에서 논리 볼륨(LV)을 할당하는 순서.

**12. ①** `mkfs.ext4`(= `mkfs -t ext4`)로 파일시스템 생성. `fsck` 는 점검, `mount` 는 연결, `tune2fs` 는 속성 변경 → [[mount]].

**13. ②** `/var/log/btmp`(바이너리, 로그인 실패) → `lastb` 로 조회. wtmp 는 성공 이력(`last`), lastlog 는 마지막 로그인(`lastlog`) → [[system-security]] 1-1 · [[last]].

**14. ①** 파일 기본 666 - umask 077 = 600. (디렉터리는 777 - 077 = 700) → [[system-security]] 2-2.

**15. ②** `chattr +i` 는 불변 속성으로 root 도 삭제·수정 불가. `+a` 는 추가만 허용(로그 변조 방지) → [[system-security]] 2-3.

**16. ①** `pam_wheel.so` 는 `/etc/pam.d/su` 에서 wheel 그룹만 `su` 를 허용하도록 제한. securetty 는 root 터미널 제한, tally2 는 실패 횟수 제한 → [[system-security]] 2-4 · [[sudo]].

**17. ④** IMAP 은 143번, 110번은 POP3. 나머지 연결은 모두 정확 → [[network-service]] 0.

**18. ②** `DocumentRoot` 가 웹 문서 루트(기본 `/var/www/html`). `ServerRoot` 는 설정·로그 기준 디렉터리 → [[network-service]] 1-1.

**19. ②** `-P`(Policy)로 체인 기본 정책 지정. `-A` 추가, `-F` 초기화, `-D` 삭제 → [[iptables]] · [[network-security]] 2-1.

**20. ②** SYN Flooding 은 SYN 만 대량 전송해 백로그 큐를 고갈시키는 DoS. 방어는 SYN 쿠키·백로그 증대 → [[network-security]] 1-2.

---

## 오답 복습 링크
- 로그·특수권한·PAM·umask → [[system-security]]
- 서비스 포트·Apache → [[network-service]]
- 방화벽·공격 유형 → [[network-security]] · [[iptables]]
- 파일시스템·크론 → [[mount]] · [[crontab]]
