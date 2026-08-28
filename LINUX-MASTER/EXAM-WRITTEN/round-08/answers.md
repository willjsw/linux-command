---
title: 필기 모의고사 Round 08 — 정답·해설
type: exam-written-answer
round: 8
tags:
  - exam/linux-master
  - exam/written
  - exam/answers
updated: 2026-08-28
---

# 필기 모의고사 Round 08 — 정답·해설

- 문제: [[questions]]
- ⚠ 반드시 풀이를 마친 후 열람

---

## 정답 요약

| 번호 | 정답 | 번호 | 정답 | 번호 | 정답 | 번호 | 정답 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | ③ | 6 | ② | 11 | ③ | 16 | ① |
| 2 | ① | 7 | ③ | 12 | ④ | 17 | ② |
| 3 | ③ | 8 | ① | 13 | ① | 18 | ④ |
| 4 | ② | 9 | ④ | 14 | ③ | 19 | ② |
| 5 | ③ | 10 | ② | 15 | ③ | 20 | ① |

---

## 해설

**1. ③** GPL 은 파생물에도 동일 라이선스를 강제하는 copyleft 라이선스. BSD·MIT·Apache 는 소스 비공개 재배포를 허용하는 관대(permissive) 라이선스.

**2. ①** `dnf install`(구 `yum install`)은 저장소에서 의존성을 자동 해결·설치. `rpm -ivh` 는 단일 패키지 설치로 의존성 수동 처리, `dpkg` 는 데비안 계열.

**3. ③** `multi-user.target` = 텍스트 다중 사용자 모드(구 런레벨 3). `graphical.target` 은 GUI(런레벨 5), `rescue.target` 은 단일 사용자 → [[systemctl]].

**4. ②** BIOS/MBR 방식은 디스크 첫 512바이트(MBR)에 stage1 이 위치. `/etc/default/grub` 는 설정 원본 → [[grub2-install]].

**5. ③** `rpm -qf <파일>` 은 해당 파일의 소유 패키지를 역조회. `-qa` 전체 목록, `-ql` 패키지 내 파일 목록, `-V` 무결성 검증.

**6. ②** `$PATH` 에 등록된 디렉터리 순으로 실행 파일을 탐색. `$SHELL` 은 로그인 셸, `$PS1` 은 프롬프트 문자열.

**7. ③** `kill` 기본 시그널은 15(SIGTERM)로 정상 종료 요청. 9(SIGKILL)는 강제 종료, 1(SIGHUP)은 재기동/재읽기 → [[ps]].

**8. ①** 좀비는 종료됐으나 부모가 회수하지 않은 상태. 부모가 먼저 죽어 init 에 입양되는 것은 고아 프로세스(Round 01 과 대비).

**9. ④** GNOME 은 데스크톱 환경(DE)이며 디스플레이 매니저가 아님. GDM·KDM·LightDM 은 로그인 화면을 제공하는 디스플레이 매니저.

**10. ②** `Ctrl+z` 로 정지(SIGTSTP) 후 `bg` 로 백그라운드 재개. `fg` 는 다시 포그라운드로 복귀.

**11. ③** `DocumentRoot` 가 웹 문서 루트(`/var/www/html`)를 지정. `ServerRoot` 는 설정·로그 기준 디렉터리 → [[network-service]] 1-1.

**12. ④** `MX` 는 메일 서버 지정 레코드로 우선순위 숫자가 낮을수록 우선. `NS` 는 네임서버, `CNAME` 은 별칭 → [[network-service]] 2-2.

**13. ①** NFS 공유는 `/etc/exports` 에 정의하고 `exportfs -ra` 로 반영. Samba 는 `smb.conf` → [[network-service]] 4-1 · [[mount]].

**14. ③** SMTP 25 / POP3 110 / IMAP 143. IMAP 은 서버 보관·동기화, POP3 는 수신 후 삭제 방식 → [[network-service]] 3.

**15. ③** 마지막 옥텟 192 = `11000000` → 상위 2비트 사용, 24 + 2 = 26 → /26. 사용 호스트 62개.

**16. ①** 볼륨 그룹(VG)이 여러 PV 를 묶은 저장 풀. VG 에서 LV 를 잘라 사용하며 할당 단위는 PE → [[lvm]].

**17. ②** RAID 1(미러링)은 동일 데이터를 두 디스크에 복제 → 가용 용량 1/2, 디스크 1개 장애 허용. RAID 0 은 내결함성 없음 → [[network-service]] 7-2.

**18. ④** `mkfs -t ext4 /dev/sdb1` 로 파일시스템 생성. `fsck` 는 점검·복구, `tune2fs` 는 속성 변경, `mount` 는 연결.

**19. ②** `parted` 는 GPT·2TB 초과 대용량 디스크를 지원. 구형 `fdisk` 는 본래 MBR 중심(2TB 한계) → [[parted]] · [[fdisk]].

**20. ①** `df` 는 파일시스템별 전체·사용·잔여 용량 표시. `du` 는 디렉터리별 사용량, `lsblk` 는 블록 장치 구조 → [[df]].

---

## 오답 복습 링크
- 서비스 설정 파일·포트 → [[network-service]]
- 디스크·RAID·LVM → [[lvm]] · [[parted]] · [[df]]
- 부팅·서비스 관리 → [[grub2-install]] · [[systemctl]]
