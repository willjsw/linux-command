---
title: 필기 모의고사 Round 03 — 정답·해설
type: exam-written-answer
round: 3
tags:
  - exam/linux-master
  - exam/written
  - exam/answers
updated: 2026-08-28
---

# 필기 모의고사 Round 03 — 정답·해설

- 문제: [[questions]]
- ⚠ 반드시 풀이를 마친 후 열람

---

## 정답 요약

| 번호 | 정답 | 번호 | 정답 | 번호 | 정답 | 번호 | 정답 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | ② | 6 | ④ | 11 | ② | 16 | ② |
| 2 | ① | 7 | ② | 12 | ② | 17 | ③ |
| 3 | ② | 8 | ② | 13 | ③ | 18 | ① |
| 4 | ② | 9 | ③ | 14 | ③ | 19 | ③ |
| 5 | ② | 10 | ② | 15 | ② | 20 | ① |

---

## 해설

**1. ②** 2.6 이전 커널은 두 번째 숫자가 짝수면 안정 버전, 홀수면 개발 버전. (예: 2.4 안정, 2.5 개발) 현행 커널은 이 관례를 쓰지 않음.

**2. ①** `rpm -qf`(query file)는 특정 파일의 소속 패키지를 역조회. `-ql` 은 패키지의 파일 목록, `-qa` 는 전체 목록, `-V` 는 무결성 검증.

**3. ②** `multi-user.target` 이 구 런레벨 3(텍스트 다중 사용자), `graphical.target` 은 런레벨 5 → [[systemctl]].

**4. ②** 표준 오류의 파일 디스크립터는 2번이므로 `2> file`. `> file`(=`1>`)은 표준 출력, `>>` 는 추가, `<` 는 입력.

**5. ②** `Ctrl+Z` 로 작업을 정지(suspend)시킨 뒤 `bg` 로 백그라운드 재개. `fg` 는 포그라운드 복귀 → [[ps]].

**6. ④** GNOME 은 데스크톱 환경(DE)이며 디스플레이 매니저가 아님. GDM·KDM·LightDM 이 대표적 디스플레이 매니저.

**7. ②** 스위치는 MAC 주소 기반으로 프레임을 전달하는 데이터링크 계층(2계층) 장비. 라우터가 3계층.

**8. ②** `/23` → 호스트 비트 32 - 23 = 9 → 2^9 - 2 = 510.

**9. ③** `ping` 은 ICMP(Echo Request/Reply)를 사용. TCP/UDP 포트를 쓰지 않음.

**10. ②** RAID 1+0 은 미러링(1/2 용량)이므로 4 × 1TB ÷ 2 = 2TB. (RAID 5 였다면 3TB) → [[network-service]] 7-2.

**11. ②** fstab 6개 필드 `장치 / 마운트포인트 / 파일시스템 / 옵션 / dump / pass` 중 6번째 pass 는 부팅 시 fsck 검사 순서(0=검사 안 함, 1=루트, 2=그 외) → [[mount]].

**12. ②** `df -h` 는 파일시스템(파티션)별 용량. `du` 는 디렉터리·파일 사용량 → [[mount]].

**13. ③** Sticky Bit(1000)를 디렉터리에 설정하면 소유자만 자기 파일 삭제 가능(예: `/tmp`) → [[system-security]] 2-1.

**14. ③** 차등(Differential) 백업은 마지막 전체 백업 이후 변경분을 매번 누적 저장 → 복원은 전체 + 최신 차등 1개. 증분은 직전 백업 이후 변경분이라 전체 + 증분 전부 필요 → [[system-security]] 3-1.

**15. ②** `.=info` 처럼 `=` 를 붙이면 해당 우선순위만 기록. `mail.info`(= 없음)는 info 이상 전부, `mail.!err` 는 err 이상 제외 → [[system-security]] 1-2.

**16. ②** `getenforce` 가 현재 모드를 출력. `setenforce 0` 은 모드 변경(Permissive), `semanage`·`chcon` 은 정책·컨텍스트 조작 → [[getenforce]].

**17. ③** MX 레코드가 메일 서버를 지정하며 우선순위 숫자가 낮을수록 우선. A 는 IPv4, CNAME 은 별칭, PTR 은 역방향 → [[network-service]] 2-2.

**18. ①** NFS 공유 대상·권한은 `/etc/exports` 에 정의하고 `exportfs -ra` 로 반영. smb.conf 는 Samba → [[network-service]] 4-1.

**19. ③** RSA 는 공개키/개인키 쌍을 쓰는 비대칭키 알고리즘. AES·DES·SEED 는 대칭키 → [[network-security]] 4-1 · [[sha256sum]].

**20. ①** `--permanent` 로 영구 규칙을 추가한 뒤에는 `firewall-cmd --reload` 로 런타임에 반영해야 함 → [[firewall-cmd]] · [[network-security]] 2-3.

---

## 오답 복습 링크
- 특수권한·백업·로그·SELinux → [[system-security]] · [[getenforce]]
- DNS·NFS·서비스 → [[network-service]]
- 암호화·방화벽 → [[network-security]] · [[firewall-cmd]]
- 파일시스템·RAID → [[mount]]
