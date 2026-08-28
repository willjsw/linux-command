---
title: 필기 모의고사 Round 04 — 정답·해설
type: exam-written-answer
round: 4
tags:
  - exam/linux-master
  - exam/written
  - exam/answers
updated: 2026-08-28
---

# 필기 모의고사 Round 04 — 정답·해설

- 문제: [[questions]]
- ⚠ 반드시 풀이를 마친 후 열람

---

## 정답 요약

| 번호 | 정답 | 번호 | 정답 | 번호 | 정답 | 번호 | 정답 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | ② | 6 | ① | 11 | ② | 16 | ① |
| 2 | ② | 7 | ① | 12 | ③ | 17 | ② |
| 3 | ② | 8 | ③ | 13 | ② | 18 | ② |
| 4 | ② | 9 | ② | 14 | ② | 19 | ② |
| 5 | ② | 10 | ① | 15 | ① | 20 | ② |

---

## 해설

**1. ②** 켄 톰슨과 데니스 리치가 1969년 벨 연구소에서 유닉스를 개발. 스톨먼은 GNU, 토르발스는 리눅스 커널, 타넨바움은 MINIX.

**2. ②** RHEL 8 이상은 `dnf`(구 yum 후속)가 기본. `apt`·`dpkg` 는 데비안, `zypper` 는 SUSE.

**3. ②** GRUB2 는 `/etc/default/grub`(및 `/etc/grub.d/`)를 편집한 뒤 `grub2-mkconfig -o /boot/grub2/grub.cfg` 로 `grub.cfg` 를 재생성. grub.cfg 는 직접 편집 대상이 아님 → [[grub2-install]].

**4. ②** `~/.bashrc` 는 사용자별 대화형 셸 설정으로 alias·함수 정의에 사용. `/etc/profile`·`/etc/bashrc` 는 전역, `~/.bash_history` 는 명령 이력.

**5. ②** `nice` 는 새 프로세스를 지정한 우선순위로 시작. `renice` 는 실행 중 프로세스의 우선순위 변경 → [[ps]].

**6. ①** `startx` 는 콘솔에서 X 세션을 수동 기동. `xhost` 는 접근 제어, `xauth` 는 인증 쿠키 관리.

**7. ①** ARP 는 IP → MAC 변환. RARP 는 반대(MAC → IP), ICMP 는 진단, DHCP 는 IP 자동 할당.

**8. ③** `/24` 에서 `/28` 로 4비트를 빌리면 2^4 = 16개 서브넷 생성. (서브넷당 호스트는 2^4 - 2 = 14개)

**9. ②** 연결 해제는 FIN 플래그로 시작하는 4-way handshake. SYN 은 연결 수립, RST 는 강제 초기화.

**10. ①** `usermod -s` 로 로그인 셸 변경. `-d` 는 홈 디렉터리, `-g` 는 기본 그룹, `useradd` 는 신규 생성 → [[usermod]].

**11. ②** `/dev/tty`(터미널)는 문자 단위 입출력의 문자 장치. 디스크·SSD·USB 저장장치는 블록 장치.

**12. ③** Quota 는 사용자·그룹별 디스크 사용량(블록·inode)을 제한. RAID·LVM 은 볼륨 구성, Swap 은 가상 메모리.

**13. ②** SetGID = 8진수 2000 → `find / -perm -2000 -type f`. 4000=SetUID, 1000=Sticky → [[find]] · [[system-security]] 2-1.

**14. ②** TCP Wrapper 는 `/etc/hosts.allow` 를 먼저 검사(allow 우선)한 뒤 `/etc/hosts.deny` 검사, 양쪽에 없으면 허용 → [[system-security]] 2-6.

**15. ①** `chage -l` 로 만료·최대 사용 기간 등 정책 조회. `passwd -l`·`usermod -L` 은 계정 잠금, `id` 는 UID/GID 확인 → [[system-security]] 2-7 · [[usermod]].

**16. ①** `-p`(priority)로 우선순위 필터. `err` 이상만 조회하려면 `journalctl -p err`. `-u` 는 유닛, `-b` 는 부팅, `-f` 는 실시간 추적 → [[journalctl]].

**17. ②** Samba 의 주 설정 파일은 `/etc/samba/smb.conf`(데몬 smbd·nmbd). `/etc/exports` 는 NFS → [[network-service]] 4-2.

**18. ②** 방화벽 환경에서는 클라이언트가 데이터 연결을 여는 수동(Passive) 모드가 권장. 능동 모드는 서버가 20번 포트로 역접속 → [[network-service]] 4-3.

**19. ②** 목적지 주소·포트를 바꾸는 DNAT 가 포트 포워딩에 사용(`-t nat -A PREROUTING ... -j DNAT --to`). SNAT/MASQUERADE 는 출발지 변환 → [[iptables]] · [[network-security]] 2-2.

**20. ②** 정상 트래픽을 공격으로 오판하는 것은 오탐(False Positive). 실제 공격을 놓치는 것은 미탐(False Negative) → [[network-security]] 3.

---

## 오답 복습 링크
- 특수권한·TCP Wrapper·계정 정책·저널 → [[system-security]] · [[journalctl]]
- Samba·FTP 서비스 → [[network-service]]
- NAT·IDS → [[network-security]] · [[iptables]]
- 사용자·부트로더 관리 → [[usermod]] · [[grub2-install]]
