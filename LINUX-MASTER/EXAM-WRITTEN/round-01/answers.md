---
title: 필기 모의고사 Round 01 — 정답·해설
type: exam-written-answer
round: 1
tags:
  - exam/linux-master
  - exam/written
  - exam/answers
updated: 2026-08-28
---

# 필기 모의고사 Round 01 — 정답·해설

- 문제: [[questions]]
- ⚠ 반드시 풀이를 마친 후 열람

---

## 정답 요약

| 번호 | 정답 | 번호 | 정답 | 번호 | 정답 | 번호 | 정답 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | ② | 6 | ③ | 11 | ② | 16 | ③ |
| 2 | ④ | 7 | ① | 12 | ② | 17 | ② |
| 3 | ④ | 8 | ② | 13 | ③ | 18 | ② |
| 4 | ① | 9 | ② | 14 | ④ | 19 | ③ |
| 5 | ② | 10 | ③ | 15 | ④ | 20 | ① |

---

## 해설

**1. ②** 리누스 토르발스가 1991년 리눅스 커널을 최초 공개. 리처드 스톨먼은 GNU 프로젝트·GPL 창시자.

**2. ④** GPL 은 파생물에 동일 라이선스(copyleft)를 강제하므로 소스 비공개 상용 배포는 불가. 나머지는 GPL 특징.

**3. ④** CentOS 는 RHEL 계열로 rpm 사용. Debian·Ubuntu 는 dpkg/apt, Rocky 는 rpm/dnf.

**4. ①** 전원 → BIOS/UEFI(POST) → 부트로더(GRUB) → 커널 로드 → init/systemd(PID 1) 순.

**5. ②** RHEL 계열은 `grub2-mkconfig -o /boot/grub2/grub.cfg`. `update-grub` 는 Debian 계열 래퍼 → [[grub2-install]].

**6. ③** `/etc/passwd` 7번째 필드에 로그인 셸 기록. `/etc/shadow` 는 암호화 비밀번호 → [[system-security]].

**7. ①** `!!` 는 직전 명령 재실행. `!n` 은 히스토리 n번, `!string` 은 해당 문자열로 시작한 최근 명령.

**8. ②** 부모가 먼저 종료된 자식은 고아 프로세스가 되어 init/systemd(PID 1)에 입양. 좀비는 종료됐으나 부모가 회수(wait)하지 않은 상태.

**9. ②** X 윈도는 역구조 — X **서버**가 로컬 디스플레이·키보드·마우스를 관리하고, X **클라이언트**(응용)가 원격일 수 있음.

**10. ③** nice 범위 -20~19, 값이 낮을수록 우선순위 높음. 일반 사용자는 우선순위를 낮추는(양수 증가) 방향만 가능, root 만 음수 지정 가능. `renice` 는 실행 중 프로세스에 적용 가능.

**11. ②** 네트워크 계층(3계층)이 IP 주소·라우팅 담당. 데이터링크(2)는 MAC, 전송(4)은 포트·TCP/UDP.

**12. ②** `/26` = 호스트 비트 6개 → 2^6 - 2 = 62. (네트워크·브로드캐스트 주소 제외)

**13. ③** 3-way handshake 는 TCP 의 연결 수립 절차. UDP 는 비연결형으로 handshake 없음.

**14. ④** `arp -a` 는 ARP 캐시(IP↔MAC) 조회로 라우팅 테이블과 무관. 나머지는 라우팅 조회 → [[ip]].

**15. ④** `169.254.0.0/16` 은 APIPA(링크 로컬) 자동 할당 대역. 사설 대역은 10/8, 172.16/12, 192.168/16.

**16. ③** 디렉터리 기본 777 - umask 022 = 755. 파일은 666 - 022 = 644 → [[system-security]] 2-2.

**17. ②** `fsck` (file system check)로 슈퍼블록·inode 정합성 점검·복구. `mkfs` 는 파일시스템 생성.

**18. ②** fstab 필드: `장치(UUID) / 마운트포인트 / 파일시스템 유형 / 마운트옵션 / dump / fsck순서(pass)` → [[mount]].

**19. ③** RAID 5 가용 용량 = (n-1) × 디스크 용량 = (4-1) × 1TB = 3TB. 패리티에 디스크 1개분 소모 → [[network-service]] 7-2.

**20. ①** SetUID = 8진수 4000, `find / -perm -4000 -type f` 로 전수 검색. 2000=SetGID, 1000=Sticky → [[find]] · [[system-security]] 2-1.

---

## 오답 복습 링크
- 로그·특수권한·umask → [[system-security]]
- RAID·서비스 → [[network-service]]
- 라우팅·IP → [[ip]]
