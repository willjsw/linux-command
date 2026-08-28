---
title: 필기 모의고사 Round 09 — 문제
type: exam-written
round: 9
tags:
  - exam/linux-master
  - exam/written
  - exam/questions
updated: 2026-08-28
---

# 필기 모의고사 Round 09 — 문제

- 20문항 / 목표 20분 / 4지선다
- 채점·해설: [[answers]] (풀이 완료 후 열람)

---

## 리눅스 실무·시스템의 이해

**1.** GNU 프로젝트를 시작하고 자유 소프트웨어 재단(FSF)을 설립한 인물은?
- ① 리누스 토르발스
- ② 데니스 리치
- ③ 리처드 스톨먼
- ④ 에릭 레이먼드

**2.** 데비안 계열에서 로컬 `.deb` 파일 하나를 직접 설치하는 명령은?
- ① `dpkg -i <파일>`
- ② `rpm -i <파일>`
- ③ `yum install <파일>`
- ④ `zypper in <파일>`

**3.** 시스템 부팅 시 서비스가 자동으로 시작되도록 등록하는 명령은?
- ① `systemctl start <유닛>`
- ② `systemctl status <유닛>`
- ③ `systemctl enable <유닛>`
- ④ `systemctl mask <유닛>`

**4.** GRUB2 의 기본 부팅 항목·타임아웃을 정의하며 `grub2-mkconfig` 의 입력이 되는 파일은?
- ① `/boot/grub2/grub.cfg`
- ② `/etc/default/grub`
- ③ `/etc/grub.conf`
- ④ `/etc/fstab`

**5.** 커널 모듈을 의존성까지 자동 해결하여 적재하는 명령은?
- ① `insmod`
- ② `lsmod`
- ③ `rmmod`
- ④ `modprobe`

---

## 리눅스 시스템의 이해 — SHELL·프로세스·X 윈도

**6.** 다수 리눅스 배포판에서 기본 로그인 셸로 널리 채택되는 것은?
- ① csh
- ② bash
- ③ ksh
- ④ tcsh

**7.** crontab 항목 `0 3 * * 1 /usr/bin/backup.sh` 의 실행 시점으로 옳은 것은?
- ① 매일 03:00
- ② 매월 1일 03:00
- ③ 매주 월요일 03:00
- ④ 매주 일요일 03:00

**8.** 데몬(daemon) 프로세스에 대한 설명으로 옳은 것은?
- ① 항상 포그라운드에서 실행된다
- ② 사용자가 로그인해야만 실행된다
- ③ 좀비 상태의 다른 표현이다
- ④ 백그라운드에서 서비스 요청을 대기하는 프로세스이다

**9.** 텍스트 모드로 부팅한 상태에서 X 윈도 세션을 수동으로 시작하는 명령은?
- ① `startx`
- ② `Xorg -configure`
- ③ `init 3`
- ④ `xhost +`

**10.** 터미널 로그아웃(SIGHUP) 이후에도 백그라운드 작업이 종료되지 않도록 실행하는 명령은?
- ① `bg`
- ② `jobs`
- ③ `nohup`
- ④ `fg`

---

## 네트워크의 이해

**11.** 윈도우와의 파일·프린터 공유(SMB/CIFS)를 제공하는 서비스의 주 설정 파일은?
- ① `/etc/exports`
- ② `/etc/samba/smb.conf`
- ③ `/etc/vsftpd/vsftpd.conf`
- ④ `/etc/dovecot/dovecot.conf`

**12.** SSH 서버의 기본 포트와 설정 파일이 바르게 짝지어진 것은?
- ① 22 / `/etc/ssh/sshd_config`
- ② 23 / `/etc/ssh/ssh_config`
- ③ 22 / `/etc/sshd.conf`
- ④ 443 / `/etc/ssh/sshd_config`

**13.** 다음 중 DNS 이름 질의 조회 도구가 아닌 것은?
- ① `dig`
- ② `nslookup`
- ③ `host`
- ④ `netstat`

**14.** DHCP 클라이언트가 IP 를 할당받는 DORA 절차의 순서로 옳은 것은?
- ① Discover → Offer → Request → Ack
- ② Offer → Discover → Ack → Request
- ③ Request → Offer → Discover → Ack
- ④ Discover → Request → Offer → Ack

**15.** IPv4 주소 `172.16.5.10` 이 속하는 주소 클래스는?
- ① A 클래스
- ② B 클래스
- ③ C 클래스
- ④ D 클래스

---

## 시스템 관리·장치 관리

**16.** 부팅 시 파일시스템을 자동 마운트하기 위해 항목을 등록하는 파일은?
- ① `/etc/mtab`
- ② `/proc/mounts`
- ③ `/etc/fstab`
- ④ `/etc/exports`

**17.** LVM 논리 볼륨(LV)의 크기를 온라인으로 확장하는 명령은?
- ① `pvcreate`
- ② `vgreduce`
- ③ `lvremove`
- ④ `lvextend`

**18.** 여러 디스크에 데이터를 분산 저장해 성능을 높이지만 내결함성이 전혀 없는 RAID 레벨은?
- ① RAID 0
- ② RAID 1
- ③ RAID 5
- ④ RAID 10

**19.** `mkswap` 으로 초기화한 스왑 공간을 실제로 활성화하는 명령은?
- ① `mkswap`
- ② `swapon`
- ③ `free`
- ④ `swapoff`

**20.** 시스템의 블록 장치와 파티션 구조를 트리 형태로 보여주는 명령은?
- ① `df`
- ② `du`
- ③ `mount`
- ④ `lsblk`
