---
title: 필기 모의고사 Round 03 — 문제
type: exam-written
round: 3
tags:
  - exam/linux-master
  - exam/written
  - exam/questions
updated: 2026-08-28
---

# 필기 모의고사 Round 03 — 문제

- 20문항 / 목표 20분 / 4지선다
- 채점·해설: [[answers]] (풀이 완료 후 열람)

---

## 리눅스 실무·시스템의 이해

**1.** 구버전(2.x대) 리눅스 커널의 버전 번호 체계에서 안정(stable) 버전을 의미하는 것은?
- ① 두 번째 숫자가 홀수
- ② 두 번째 숫자가 짝수
- ③ 첫 번째 숫자가 짝수
- ④ 세 번째 숫자가 0

**2.** 특정 파일이 어느 rpm 패키지에 속해 있는지 역으로 조회하는 명령으로 옳은 것은?
- ① `rpm -qf /경로/파일`
- ② `rpm -ql 패키지`
- ③ `rpm -qa`
- ④ `rpm -V 패키지`

**3.** systemd 환경에서 그래픽 없이 다중 사용자 텍스트 모드(구 런레벨 3)에 해당하는 target 은?
- ① `graphical.target`
- ② `multi-user.target`
- ③ `rescue.target`
- ④ `poweroff.target`

---

## 리눅스 시스템의 이해 — SHELL·프로세스·X 윈도

**4.** 명령의 표준 오류(stderr)만을 파일로 리다이렉션하는 표현으로 옳은 것은?
- ① `명령 > file`
- ② `명령 2> file`
- ③ `명령 < file`
- ④ `명령 >> file`

**5.** 포그라운드에서 실행 중인 작업을 백그라운드로 전환하는 절차로 옳은 것은?
- ① `Ctrl+C` 로 종료 후 `bg`
- ② `Ctrl+Z` 로 정지 후 `bg`
- ③ `Ctrl+D` 로 종료 후 `fg`
- ④ `kill -9` 후 `bg`

**6.** 다음 중 X 윈도의 디스플레이 매니저(로그인 관리자)가 아닌 것은?
- ① GDM
- ② KDM
- ③ LightDM
- ④ GNOME

---

## 네트워크의 이해

**7.** OSI 참조 모델에서 스위치(L2 switch)가 주로 동작하는 계층은?
- ① 물리 계층
- ② 데이터링크 계층
- ③ 네트워크 계층
- ④ 전송 계층

**8.** `192.168.10.0/23` 네트워크에서 사용 가능한 호스트 수는?
- ① 254
- ② 510
- ③ 512
- ④ 1022

**9.** `ping` 명령이 사용하는 프로토콜로 옳은 것은?
- ① TCP
- ② UDP
- ③ ICMP
- ④ ARP

---

## 시스템 관리·장치 관리

**10.** 각 1TB 디스크 4개로 RAID 1+0(RAID 10)을 구성할 때 사용 가능한 총 용량은?
- ① 1TB
- ② 2TB
- ③ 3TB
- ④ 4TB

**11.** `/etc/fstab` 의 6번째 필드(pass)가 의미하는 것은?
- ① dump 백업 대상 여부
- ② 부팅 시 fsck 검사 순서
- ③ 마운트 옵션
- ④ 파일시스템 유형

**12.** 마운트된 파일시스템별 전체·사용·잔여 용량을 사람이 읽기 쉬운 단위로 확인하는 명령은?
- ① `du -sh`
- ② `df -h`
- ③ `ls -l`
- ④ `fdisk -l`

---

## 시스템 보안 및 관리

**13.** 여러 사용자가 공유하는 디렉터리에서 자신이 생성한 파일만 삭제할 수 있도록 제한하는 특수 권한은?
- ① SetUID
- ② SetGID
- ③ Sticky Bit
- ④ umask

**14.** 마지막 전체 백업 이후 변경된 데이터만 백업하며, 복원 시 전체 백업 1개와 최신 백업 1개만 있으면 되는 백업 방식은?
- ① 전체 백업
- ② 증분 백업
- ③ 차등 백업
- ④ 실시간 백업

**15.** rsyslog 설정에서 `mail.=info` 규칙이 의미하는 것으로 옳은 것은?
- ① info 이상 우선순위 전부 기록
- ② info 우선순위만 기록
- ③ info 를 제외하고 기록
- ④ info 이하 우선순위 기록

**16.** SELinux 의 현재 동작 모드(Enforcing/Permissive/Disabled)를 확인하는 명령은?
- ① `setenforce 0`
- ② `getenforce`
- ③ `chcon`
- ④ `semanage`

---

## 네트워크 서비스·네트워크 보안

**17.** 메일 서버를 지정하며 우선순위 숫자가 낮을수록 먼저 선택되는 DNS 레코드는?
- ① A
- ② CNAME
- ③ MX
- ④ PTR

**18.** NFS 서버에서 공유할 디렉터리와 접근 권한을 설정하는 파일은?
- ① `/etc/exports`
- ② `/etc/fstab`
- ③ `/etc/samba/smb.conf`
- ④ `/etc/hosts.allow`

**19.** 다음 중 비대칭키(공개키) 암호 알고리즘에 해당하는 것은?
- ① AES
- ② DES
- ③ RSA
- ④ SEED

**20.** firewalld 에서 http 서비스를 영구(permanent)로 허용한 뒤, 변경 사항을 적용하기 위해 실행해야 하는 명령은?
- ① `firewall-cmd --reload`
- ② `systemctl restart network`
- ③ `iptables -F`
- ④ `firewall-cmd --list-all`
