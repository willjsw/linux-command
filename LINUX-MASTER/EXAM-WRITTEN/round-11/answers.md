---
title: 필기 모의고사 Round 11 — 정답·해설
type: exam-written-answer
round: 11
tags:
  - exam/linux-master
  - exam/written
  - exam/answers
updated: 2026-08-28
---

# 필기 모의고사 Round 11 — 정답·해설

- 문제: [[questions]]
- ⚠ 반드시 풀이를 마친 후 열람

---

## 정답 요약

| 번호 | 정답 | 번호 | 정답 | 번호 | 정답 | 번호 | 정답 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | ④ | 6 | ③ | 11 | ② | 16 | ④ |
| 2 | ② | 7 | ① | 12 | ④ | 17 | ② |
| 3 | ③ | 8 | ④ | 13 | ① | 18 | ③ |
| 4 | ① | 9 | ② | 14 | ③ | 19 | ④ |
| 5 | ② | 10 | ③ | 15 | ② | 20 | ① |

---

## 해설

**1. ④** BSD 는 파생물의 소스 공개를 강제하지 않는 허용적(permissive) 라이선스. GPL·AGPL 은 강한 카피레프트, LGPL 은 약한 카피레프트.

**2. ②** `rpm -qa` (query all)로 설치 패키지 전체 조회. `-ivh` 설치, `-e` 삭제, `-Uvh` 업그레이드.

**3. ③** `systemctl set-default graphical.target` 로 그래픽 부팅 지정. `multi-user.target` 은 텍스트 다중 사용자(런레벨 3 상당) → [[systemctl]].

**4. ①** ext2 는 저널링 미지원. ext4·xfs·btrfs 는 저널링(또는 CoW) 지원으로 비정상 종료 후 복구가 빠름.

**5. ②** `/boot` 에 커널 이미지(vmlinuz)·initramfs·GRUB 설정이 위치. `/proc` 은 커널 가상 파일시스템.

**6. ③** 로그인 셸은 전역 `/etc/profile` 을 먼저 읽고 이어 `~/.bash_profile`(→ `~/.bashrc`) 순으로 읽음.

**7. ①** `export VAR` 로 셸 변수를 환경변수로 승격해 자식 프로세스에 상속. `set` 은 셸 변수 표시, `env` 는 환경 실행/표시.

**8. ④** SIGKILL = 9 (무조건 종료, 트랩 불가). 1=SIGHUP, 2=SIGINT, 15=SIGTERM(정상 종료 요청) → [[kill]].

**9. ②** `2>` 는 stderr(파일 디스크립터 2) 리다이렉트. `>`/`>>` 는 stdout, `<` 는 입력.

**10. ③** 디스플레이 매니저(gdm·kdm·xdm)가 그래픽 로그인 화면 제공. 윈도 매니저는 창 배치, 데스크톱 환경은 통합 GUI.

**11. ②** 라우터는 IP 주소 기반 경로 결정 → 3계층(네트워크). 2계층은 스위치, 7계층은 응용.

**12. ④** SYN Flooding 은 백로그 큐 고갈 공격 → SYN 쿠키·백로그 증대·rate limit 로 방어 → [[network-security]].

**13. ①** `-P` 는 체인 기본 정책(Policy) 설정. `-A` 규칙 추가, `-F` 초기화, `-D` 규칙 삭제 → [[iptables]] · [[network-security]].

**14. ③** SYN → SYN-ACK → ACK 순. 클라이언트 SYN, 서버 SYN-ACK, 클라이언트 ACK 로 연결 성립.

**15. ②** RSA 는 공개키/개인키 쌍의 비대칭키 알고리즘. AES·SEED·3DES 는 대칭키 → [[network-security]] 4-1.

**16. ④** 로그인 실패는 `/var/log/btmp`(바이너리) → `lastb` 로 조회. wtmp=성공/last, utmp=현재/who, lastlog=마지막 → [[system-security]] · [[last]].

**17. ②** `mail.=info` 는 `=` 로 info **레벨만** 지정. 접두 없는 `mail.info` 는 info 이상 전부, `mail.!err` 는 err 이상 제외 → [[system-security]] 1-2.

**18. ③** `chattr +i` (immutable)로 불변 설정 시 root 도 수정·삭제 불가. `+a` 는 추가만 허용(로그 변조 방지) → [[system-security]] 2-3.

**19. ④** `sufficient` 는 성공 시 즉시 인증 허용. `required`/`requisite` 는 실패 처리 방식 차이, `optional` 은 참고용 → [[system-security]] 2-4.

**20. ①** 증분 백업은 전체·증분 관계없이 **직전 백업 이후** 변경분만 저장 → 용량 최소·복원 복잡. ②는 차등(Differential) 설명 → [[system-security]] 3-1 · [[tar]].

---

## 오답 복습 링크
- 로그·PAM·chattr·백업 → [[system-security]]
- 침해 유형·iptables·암호화 → [[network-security]]
- 시그널·프로세스 → [[kill]]
- 서비스 타깃 → [[systemctl]]
