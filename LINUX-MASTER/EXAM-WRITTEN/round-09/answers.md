---
title: 필기 모의고사 Round 09 — 정답·해설
type: exam-written-answer
round: 9
tags:
  - exam/linux-master
  - exam/written
  - exam/answers
updated: 2026-08-28
---

# 필기 모의고사 Round 09 — 정답·해설

- 문제: [[questions]]
- ⚠ 반드시 풀이를 마친 후 열람

---

## 정답 요약

| 번호 | 정답 | 번호 | 정답 | 번호 | 정답 | 번호 | 정답 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | ③ | 6 | ② | 11 | ② | 16 | ③ |
| 2 | ① | 7 | ③ | 12 | ① | 17 | ④ |
| 3 | ③ | 8 | ④ | 13 | ④ | 18 | ① |
| 4 | ② | 9 | ① | 14 | ① | 19 | ② |
| 5 | ④ | 10 | ③ | 15 | ② | 20 | ④ |

---

## 해설

**1. ③** 리처드 스톨먼이 1983년 GNU 프로젝트를 시작하고 FSF·GPL 을 창시. 리누스 토르발스는 커널 개발자(Round 01 대비).

**2. ①** 데비안 계열은 `dpkg -i` 로 로컬 `.deb` 설치(의존성 수동). `rpm` 은 RHEL 계열, `zypper` 는 SUSE 계열.

**3. ③** `systemctl enable` 은 부팅 시 자동 시작 등록(심볼릭 링크 생성). `start` 는 즉시 실행, `mask` 는 시작 자체를 차단 → [[systemctl]].

**4. ②** `/etc/default/grub` 가 설정 원본이며 `grub2-mkconfig` 가 이를 읽어 `grub.cfg` 를 생성. `grub.cfg` 는 직접 편집 대상이 아님 → [[grub2-install]].

**5. ④** `modprobe` 는 `modules.dep` 를 참조해 의존 모듈까지 적재. `insmod` 는 의존성 처리 없이 단일 적재, `lsmod` 조회, `rmmod` 제거.

**6. ②** 대부분 배포판 기본 로그인 셸은 bash. csh·tcsh·ksh 는 대체 셸.

**7. ③** cron 필드는 `분 시 일 월 요일`, 요일 1=월요일. `0 3 * * 1` = 매주 월요일 03:00 → [[crontab]].

**8. ④** 데몬은 백그라운드에서 상주하며 서비스 요청을 대기하는 프로세스(httpd·sshd 등). 로그인·포그라운드와 무관 → [[ps]].

**9. ①** `startx` 는 텍스트 모드에서 X 세션(xinit 래퍼)을 시작. `xhost +` 는 접근 제어 설정.

**10. ③** `nohup` 은 SIGHUP 을 무시하게 하여 로그아웃 후에도 작업 유지. `&` 와 함께 사용.

**11. ②** SMB/CIFS 공유는 Samba 의 `/etc/samba/smb.conf` 로 설정하고 `smbd`·`nmbd` 데몬이 담당 → [[network-service]] 4-2.

**12. ①** SSH 서버 포트 22, 서버 설정은 `/etc/ssh/sshd_config`(클라이언트는 `ssh_config`) → [[ssh]].

**13. ④** `netstat` 은 소켓·라우팅 조회 도구로 DNS 질의와 무관. `dig`·`nslookup`·`host` 가 DNS 조회 → [[network-service]] 2-2.

**14. ①** DHCP 할당 절차 DORA = Discover(탐색) → Offer(제안) → Request(요청) → Ack(확인) → [[network-service]] 6-3.

**15. ②** B 클래스 범위는 첫 옥텟 128~191. 172 는 B 클래스(사설 대역 172.16~172.31 포함).

**16. ③** `/etc/fstab` 이 부팅 시 자동 마운트 항목을 정의. `/etc/mtab`·`/proc/mounts` 는 현재 마운트 상태 조회용 → [[mount]].

**17. ④** `lvextend` 로 LV 크기를 확장(이후 `resize2fs`/`xfs_growfs` 로 파일시스템 확장). `pvcreate` 는 PV 초기화 → [[lvm]].

**18. ①** RAID 0(스트라이핑)은 성능·용량은 좋으나 패리티·미러가 없어 디스크 1개만 고장나도 전체 손실 → [[network-service]] 7-2.

**19. ②** `mkswap` 은 스왑 영역 포맷, `swapon` 이 실제 활성화. `swapoff` 는 비활성화, `free` 는 사용량 조회.

**20. ④** `lsblk` 는 블록 장치·파티션을 트리로 표시. `df` 는 파일시스템 용량, `du` 는 디렉터리 사용량 → [[lsblk]].

---

## 오답 복습 링크
- 서비스 설정 파일·포트 → [[network-service]] · [[ssh]]
- 부팅·서비스·모듈 → [[systemctl]] · [[grub2-install]]
- 디스크·LVM·마운트 → [[lvm]] · [[mount]] · [[lsblk]]
- 예약 작업 → [[crontab]]
