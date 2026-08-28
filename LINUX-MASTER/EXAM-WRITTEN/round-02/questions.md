---
title: 필기 모의고사 Round 02 — 문제
type: exam-written
round: 2
tags:
  - exam/linux-master
  - exam/written
  - exam/questions
updated: 2026-08-28
---

# 필기 모의고사 Round 02 — 문제

- 20문항 / 목표 20분 / 4지선다
- 채점·해설: [[answers]] (풀이 완료 후 열람)

---

## 리눅스 실무·시스템의 이해

**1.** 자유 소프트웨어 재단(FSF)과 GNU 프로젝트를 창시한 인물은?
- ① 리누스 토르발스
- ② 리처드 스톨먼
- ③ 데니스 리치
- ④ 브라이언 커니핸

**2.** 다음 중 소스 코드를 수정한 뒤 비공개(독점) 형태로 재배포하는 것이 허용되는 라이선스는?
- ① GPL
- ② AGPL
- ③ BSD
- ④ LGPL

**3.** 데비안 계열 시스템에서 현재 설치된 패키지 목록을 조회하는 명령으로 옳은 것은?
- ① `rpm -qa`
- ② `dpkg -l`
- ③ `dnf list`
- ④ `yum search`

---

## 리눅스 시스템의 이해 — SHELL·프로세스·X 윈도

**4.** 다음 셸 중 본 셸(Bourne shell) 계열이 아닌 것은?
- ① `sh`
- ② `bash`
- ③ `ksh`
- ④ `csh`

**5.** 실행 중인 프로세스를 무시할 수 없이 강제 종료하는 시그널의 번호는?
- ① 1 (SIGHUP)
- ② 2 (SIGINT)
- ③ 9 (SIGKILL)
- ④ 15 (SIGTERM)

**6.** X 윈도 응용 프로그램의 출력을 표시할 디스플레이 위치를 지정하는 환경변수는?
- ① `$TERM`
- ② `$DISPLAY`
- ③ `$PATH`
- ④ `$SHELL`

---

## 네트워크의 이해

**7.** 서브넷 마스크 `255.255.255.224` 를 CIDR 프리픽스로 표기하면?
- ① `/26`
- ② `/27`
- ③ `/28`
- ④ `/29`

**8.** TCP/IP 에서 well-known(잘 알려진) 포트의 범위로 옳은 것은?
- ① 0 ~ 1023
- ② 1024 ~ 49151
- ③ 49152 ~ 65535
- ④ 0 ~ 65535

**9.** IPv6 주소의 전체 비트 수로 옳은 것은?
- ① 32비트
- ② 64비트
- ③ 128비트
- ④ 256비트

---

## 시스템 관리·장치 관리

**10.** 매주 일요일 03시 30분에 작업을 실행하도록 하는 crontab 항목으로 옳은 것은? (필드: 분 시 일 월 요일)
- ① `30 3 * * 0`
- ② `3 30 * * 7`
- ③ `30 3 0 * *`
- ④ `* * 30 3 0`

**11.** LVM(논리 볼륨 관리) 구성 요소의 생성 순서로 옳은 것은?
- ① 물리 볼륨(PV) → 볼륨 그룹(VG) → 논리 볼륨(LV)
- ② 볼륨 그룹(VG) → 물리 볼륨(PV) → 논리 볼륨(LV)
- ③ 논리 볼륨(LV) → 볼륨 그룹(VG) → 물리 볼륨(PV)
- ④ 물리 볼륨(PV) → 논리 볼륨(LV) → 볼륨 그룹(VG)

**12.** 파티션 `/dev/sdb1` 을 ext4 파일시스템으로 생성하는 명령으로 옳은 것은?
- ① `mkfs.ext4 /dev/sdb1`
- ② `fsck.ext4 /dev/sdb1`
- ③ `mount -t ext4 /dev/sdb1`
- ④ `tune2fs /dev/sdb1`

---

## 시스템 보안 및 관리

**13.** 로그인 실패 이력이 기록되는 바이너리 로그 파일과 이를 조회하는 명령의 연결로 옳은 것은?
- ① `/var/log/wtmp` — `last`
- ② `/var/log/btmp` — `lastb`
- ③ `/var/log/secure` — `dmesg`
- ④ `/var/log/lastlog` — `lastb`

**14.** umask 값이 `077` 일 때 새로 생성되는 **파일**의 기본 권한은?
- ① 600
- ② 640
- ③ 700
- ④ 644

**15.** root 사용자조차 파일을 삭제·수정할 수 없도록 불변(immutable) 속성을 부여하는 명령은?
- ① `chattr +a 파일`
- ② `chattr +i 파일`
- ③ `chmod 000 파일`
- ④ `chown root 파일`

**16.** wheel 그룹에 속한 사용자만 `su` 명령을 사용하도록 제한할 때 관여하는 PAM 모듈은?
- ① `pam_wheel.so`
- ② `pam_securetty.so`
- ③ `pam_tally2.so`
- ④ `pam_cracklib.so`

---

## 네트워크 서비스·네트워크 보안

**17.** 다음 서비스와 기본 포트 번호의 연결이 옳지 않은 것은?
- ① SSH — 22
- ② SMTP — 25
- ③ DNS — 53
- ④ IMAP — 110

**18.** Apache(httpd) 웹 서버에서 웹 문서 루트 디렉터리를 지정하는 지시자는?
- ① `ServerRoot`
- ② `DocumentRoot`
- ③ `DirectoryIndex`
- ④ `UserDir`

**19.** iptables 에서 INPUT 체인의 기본 정책(Policy)을 DROP 으로 설정하는 명령으로 옳은 것은?
- ① `iptables -A INPUT DROP`
- ② `iptables -P INPUT DROP`
- ③ `iptables -F INPUT DROP`
- ④ `iptables -D INPUT DROP`

**20.** TCP 3-way handshake 과정 중 SYN 패킷만 대량으로 전송하여 서버의 백로그 큐를 고갈시키는 공격은?
- ① Smurf
- ② SYN Flooding
- ③ Ping of Death
- ④ Land Attack
