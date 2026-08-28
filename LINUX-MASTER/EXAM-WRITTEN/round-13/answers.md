---
title: 필기 모의고사 Round 13 — 정답·해설
type: exam-written-answer
round: 13
tags:
  - exam/linux-master
  - exam/written
  - exam/answers
updated: 2026-08-28
---

# 필기 모의고사 Round 13 — 정답·해설

- 문제: [[questions]]
- ⚠ 반드시 풀이를 마친 후 열람

---

## 정답 요약

| 번호 | 정답 | 번호 | 정답 | 번호 | 정답 | 번호 | 정답 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | ③ | 6 | ① | 11 | ③ | 16 | ② |
| 2 | ① | 7 | ② | 12 | ① | 17 | ④ |
| 3 | ② | 8 | ③ | 13 | ② | 18 | ① |
| 4 | ④ | 9 | ① | 14 | ④ | 19 | ③ |
| 5 | ② | 10 | ② | 15 | ③ | 20 | ② |

---

## 해설

**1. ③** `lsmod` 로 적재된 모듈 목록 확인(`/proc/modules` 요약). `modprobe`/`insmod` 는 적재, `depmod` 는 의존성 DB 생성.

**2. ①** RHEL 8+ 는 `dnf` 가 기본 패키지 관리자(yum 후속). `rpm` 은 의존성 미해결, `dpkg`/`apt` 는 데비안 계열.

**3. ②** 부트로더(GRUB2) 메뉴 단계에서 `e` 로 커널 파라미터 편집·항목 선택 가능. POST 는 하드웨어 점검 단계 → [[grub2-install]].

**4. ④** RHEL 7 이상 기본 파일시스템은 xfs(저널링). ext2 는 비저널링, vfat 은 FAT, swap 은 가상 메모리용.

**5. ②** `/proc` 은 프로세스·커널 정보를 담은 가상 파일시스템. `/sys` 는 장치·커널 객체, `/dev` 는 장치 파일.

**6. ①** 명령 치환은 `$(command)` 또는 백틱. `${var}` 는 변수 확장, `$[...]` 는 (구식) 산술, `$'...'` 는 이스케이프 문자열.

**7. ②** `> out` 으로 stdout 을 out 에 연결한 뒤 `2>&1` 로 stderr 를 stdout(=out)에 합류. ③은 순서가 반대라 stderr 가 화면으로 감.

**8. ③** 일반 사용자는 자신의 프로세스 nice 값을 **높이는(우선순위 낮추는)** 방향만 가능. 음수(우선순위 상향)·타 사용자 변경은 root 권한.

**9. ①** crontab 필드: 분 시 일(월중) 월 요일. 예 `0 3 * * 1` = 매주 월요일 03:00 → [[crontab]].

**10. ②** `DISPLAY` (예 `:0`, `host:0`)로 X 클라이언트가 출력할 X 서버 지정. TERM 은 터미널 종류, LANG 은 로캘.

**11. ③** `/27` 은 호스트 비트 5개 → 2^5 - 2 = 30. (네트워크·브로드캐스트 제외)

**12. ①** FIN·NULL·XMAS 스캔은 플래그 조합으로 방화벽·IDS 로깅을 우회하는 스텔스 스캔 → [[network-security]] 1-1.

**13. ②** `-m state --state ESTABLISHED,RELATED -j ACCEPT` 는 이미 성립·관련된 연결을 허용해 응답 트래픽을 통과시킴 → [[iptables]] · [[network-security]] 2-1.

**14. ④** ESP 가 기밀성(암호화)+선택적 무결성 제공. AH 는 무결성·인증만, IKE 는 키 교환, SA 는 보안 연관 → [[network-security]] 4-2.

**15. ③** 개인키 서명 → 공개키로 검증 = 송신자 인증·부인방지(전자서명). 기밀성은 수신자 공개키로 암호화할 때 확보 → [[network-security]] 4-1.

**16. ②** 8진수 2000 = SetGID. 디렉터리에 설정 시 내부 생성 파일이 디렉터리 그룹을 상속. 4000=SetUID, 1000=Sticky → [[system-security]] 2-1.

**17. ④** shadow 비밀번호 필드의 `!`·`*` 는 계정 잠금(로그인 불가). `usermod -L` 이 `!` 를 붙임 → [[system-security]] 2-7 · [[usermod]].

**18. ①** `pam_wheel.so` 로 wheel 그룹만 `su` 허용(`/etc/pam.d/su`). securetty=root 터미널 제한, tally2=실패 횟수 제한 → [[system-security]] 2-4 · [[sudo]].

**19. ③** RAID 6 가용 용량 = (n-2) × 디스크 = (6-2) × 2TB = 8TB. 패리티 2개분 소모 → [[network-service]] 7-2.

**20. ②** TCP Wrapper 는 `hosts.allow` → `hosts.deny` 순서로 검사, allow 가 우선. 양쪽 모두 미등록이면 허용 → [[system-security]] 2-6.

---

## 오답 복습 링크
- 특수권한·shadow·PAM·TCP Wrapper → [[system-security]]
- 스캔·iptables·IPSec·암호화 → [[network-security]]
- RAID 용량 → [[network-service]]
- 부트로더 → [[grub2-install]]
