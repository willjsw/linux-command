---
title: 필기 모의고사 Round 12 — 문제
type: exam-written
round: 12
tags:
  - exam/linux-master
  - exam/written
  - exam/questions
updated: 2026-08-28
---

# 필기 모의고사 Round 12 — 문제

- 20문항 / 목표 20분 / 4지선다
- 채점·해설: [[answers]] (풀이 완료 후 열람)

---

## 리눅스 실무·시스템의 이해

**1.** GNU 프로젝트를 시작하고 자유소프트웨어재단(FSF)을 설립한 인물은?
- ① 데니스 리치
- ② 귀도 반 로섬
- ③ 리처드 스톨먼
- ④ 리누스 토르발스

**2.** 데비안 계열에서 `.deb` 패키지를 직접 설치하는 저수준 명령은?
- ① `yum`
- ② `dnf`
- ③ `rpm -i`
- ④ `dpkg -i`

**3.** LVM 구성에서 여러 물리 볼륨(PV)을 묶어 만든 논리적 저장 공간 단위는?
- ① PV
- ② VG
- ③ LV
- ④ PE

**4.** 심볼릭 링크에 대한 설명으로 옳은 것은?
- ① 원본과 동일한 inode 를 공유한다
- ② 디렉터리에는 만들 수 없다
- ③ 다른 파일시스템으로도 생성할 수 있다
- ④ 원본을 삭제해도 항상 접근할 수 있다

**5.** systemd 에서 실행 중인 서비스 유닛의 활성 상태를 확인하는 명령은?
- ① `systemctl status sshd`
- ② `service list`
- ③ `chkconfig --list`
- ④ `init status`

---

## 리눅스 시스템의 이해 — SHELL·프로세스·X 윈도

**6.** 사용자의 로그인 기본 셸을 변경하는 명령은?
- ① `chsh`
- ② `chmod`
- ③ `chage`
- ④ `chown`

**7.** 현재 백그라운드에서 실행 중이거나 중지된 작업(job) 목록을 확인하는 명령은?
- ① `bg`
- ② `fg`
- ③ `nohup`
- ④ `jobs`

**8.** 프로세스 상태 코드 중 종료되었으나 부모가 회수하지 않은 좀비(zombie) 상태는?
- ① R
- ② S
- ③ Z
- ④ D

**9.** 로그아웃 후에도 프로세스가 계속 실행되도록 HUP 시그널을 무시시키며 실행하는 명령은?
- ① `kill`
- ② `nohup`
- ③ `wait`
- ④ `exec`

**10.** 원격 호스트가 로컬 X 서버에 접근하도록 허용/차단을 제어하는 명령은?
- ① `export DISPLAY`
- ② `xhost`
- ③ `startx`
- ④ `xinit`

---

## 네트워크의 이해

**11.** TCP/IP 4계층 모델에서 IP 프로토콜이 속하는 계층은?
- ① 응용 계층
- ② 전송 계층
- ③ 인터넷 계층
- ④ 네트워크 접근 계층

**12.** 다음 포트 번호와 서비스의 연결로 옳지 않은 것은?
- ① 22 — SSH
- ② 25 — SMTP
- ③ 53 — DNS
- ④ 143 — POP3

**13.** 출발지 IP 를 위조하여 신뢰 관계를 악용하는 공격 유형은?
- ① IP 스푸핑
- ② 포트 스캔
- ③ 버퍼 오버플로
- ④ SQL 인젝션

**14.** 내부 사설망을 외부로 내보낼 때 동적 공인 IP 환경에서 사용하는 iptables NAT 타깃은?
- ① SNAT
- ② DNAT
- ③ MASQUERADE
- ④ REDIRECT

**15.** 호스트 기반(HIDS)으로 파일 무결성을 검사하는 침입 탐지 도구는?
- ① Snort
- ② Suricata
- ③ Nmap
- ④ Tripwire

---

## 시스템 관리·장치 관리

**16.** 사용자 계정 생성 시 홈 디렉터리로 복사될 기본 파일들의 원본이 위치한 디렉터리는?
- ① `/etc/skel`
- ② `/etc/default/useradd`
- ③ `/etc/login.defs`
- ④ `/home/default`

**17.** 현재 시스템에 로그인해 있는 사용자 상태 정보를 담은 로그 파일은?
- ① `/var/log/wtmp`
- ② `/var/run/utmp`
- ③ `/var/log/btmp`
- ④ `/var/log/lastlog`

**18.** `/etc/sudoers` 파일을 문법 검증과 함께 편집하도록 권장되는 명령은?
- ① `vi /etc/sudoers`
- ② `visudo`
- ③ `sudoedit`
- ④ `usermod`

**19.** logrotate 설정 지시자 중 순환된 로그 파일을 압축하는 것은?
- ① `rotate`
- ② `notifempty`
- ③ `compress`
- ④ `create`

**20.** dump 백업에서 레벨(level) 0 이 의미하는 것은?
- ① 증분 백업
- ② 차등 백업
- ③ 백업 안 함
- ④ 전체(Full) 백업
