---
title: 필기 모의고사 Round 11 — 문제
type: exam-written
round: 11
tags:
  - exam/linux-master
  - exam/written
  - exam/questions
updated: 2026-08-28
---

# 필기 모의고사 Round 11 — 문제

- 20문항 / 목표 20분 / 4지선다
- 채점·해설: [[answers]] (풀이 완료 후 열람)

---

## 리눅스 실무·시스템의 이해

**1.** 오픈소스 라이선스 중 파생물의 소스 공개를 강제하지 않는(비카피레프트) 계열은?
- ① GPL
- ② AGPL
- ③ LGPL
- ④ BSD

**2.** rpm 명령에서 시스템에 설치된 패키지 전체 목록을 조회하는 옵션은?
- ① `rpm -ivh`
- ② `rpm -qa`
- ③ `rpm -e`
- ④ `rpm -Uvh`

**3.** systemd 환경에서 기본 부팅 타깃을 그래픽 모드로 지정하는 명령은?
- ① `init 3`
- ② `systemctl set-default multi-user.target`
- ③ `systemctl set-default graphical.target`
- ④ `systemctl isolate rescue.target`

**4.** 다음 파일시스템 중 저널링을 지원하지 않는 것은?
- ① ext2
- ② ext4
- ③ xfs
- ④ btrfs

**5.** 리눅스 디렉터리 계층 표준(FHS)에서 커널 이미지·부트로더 파일이 위치하는 디렉터리는?
- ① `/proc`
- ② `/boot`
- ③ `/var`
- ④ `/opt`

---

## 리눅스 시스템의 이해 — SHELL·프로세스·X 윈도

**6.** bash 로그인 셸이 시작될 때 가장 먼저 읽는 전역 설정 파일은?
- ① `~/.bashrc`
- ② `~/.bash_logout`
- ③ `/etc/profile`
- ④ `~/.bash_profile`

**7.** 현재 셸의 변수를 자식 프로세스에도 상속되도록 환경변수로 등록하는 명령은?
- ① `export`
- ② `set`
- ③ `env`
- ④ `unset`

**8.** 프로세스를 강제로 즉시 종료(SIGKILL)할 때 사용하는 시그널 번호는?
- ① 1
- ② 2
- ③ 15
- ④ 9

**9.** 표준 오류(stderr)만 파일 `err.log` 로 리다이렉트하는 표현으로 옳은 것은?
- ① `cmd > err.log`
- ② `cmd 2> err.log`
- ③ `cmd < err.log`
- ④ `cmd >> err.log`

**10.** X 윈도에서 사용자에게 그래픽 로그인 화면을 제공하는 프로그램 분류는?
- ① 윈도 매니저
- ② 위젯 툴킷
- ③ 디스플레이 매니저
- ④ 데스크톱 환경

---

## 네트워크의 이해

**11.** OSI 참조 모델에서 라우터가 동작하는 계층은?
- ① 2계층
- ② 3계층
- ③ 4계층
- ④ 7계층

**12.** SYN Flooding 공격의 방어 기법으로 가장 적절한 것은?
- ① ARP 테이블 정적 등록
- ② DNS 캐시 삭제
- ③ 파일 무결성 검사
- ④ SYN 쿠키 적용

**13.** iptables 에서 INPUT 체인의 기본 정책 자체를 DROP 으로 설정하는 명령은?
- ① `iptables -P INPUT DROP`
- ② `iptables -A INPUT -j DROP`
- ③ `iptables -F INPUT`
- ④ `iptables -D INPUT DROP`

**14.** TCP 연결 수립을 위한 3-way handshake 의 순서로 옳은 것은?
- ① SYN → ACK → SYN-ACK
- ② ACK → SYN → SYN-ACK
- ③ SYN → SYN-ACK → ACK
- ④ SYN-ACK → SYN → ACK

**15.** 다음 중 대칭키 암호 알고리즘이 아닌 것은?
- ① AES
- ② RSA
- ③ SEED
- ④ 3DES

---

## 시스템 관리·장치 관리

**16.** 로그인 실패 이력이 기록되는 바이너리 로그 파일과 조회 명령의 연결로 옳은 것은?
- ① `/var/log/wtmp` — `last`
- ② `/var/run/utmp` — `who`
- ③ `/var/log/lastlog` — `lastlog`
- ④ `/var/log/btmp` — `lastb`

**17.** rsyslog 설정에서 `mail.=info` 규칙의 의미로 옳은 것은?
- ① info 이상 모든 레벨 기록
- ② info 레벨만 기록
- ③ info 이상 제외
- ④ mail facility 전체 제외

**18.** 파일에 설정하면 root 도 수정·삭제할 수 없게 만드는 chattr 속성은?
- ① `+a`
- ② `+u`
- ③ `+i`
- ④ `+s`

**19.** PAM 제어 플래그 중 "검사에 성공하면 즉시 인증을 허용"하는 것은?
- ① required
- ② requisite
- ③ optional
- ④ sufficient

**20.** 증분 백업(Incremental)에 대한 설명으로 옳은 것은?
- ① 직전 백업(전체·증분 무관) 이후 변경분만 백업
- ② 마지막 전체 백업 이후 변경분을 매번 백업
- ③ 항상 전체 데이터를 백업
- ④ 복원 시 최신 백업 1개만 필요
