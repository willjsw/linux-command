---
title: 필기 모의고사 Round 05 — 문제
type: exam-written
round: 5
tags:
  - exam/linux-master
  - exam/written
  - exam/questions
updated: 2026-08-28
---

# 필기 모의고사 Round 05 — 문제

- 20문항 / 목표 20분 / 4지선다
- 채점·해설: [[answers]] (풀이 완료 후 열람)

---

## 리눅스 실무·시스템의 이해

**1.** 다음 오픈소스 라이선스 중 파생 저작물의 소스 비공개 상용 배포를 허용하는(카피레프트가 아닌) 것은?
- ① BSD
- ② GPL
- ③ AGPL
- ④ LGPL

**2.** systemd 환경에서 그래픽 없이 텍스트 기반 다중 사용자 모드에 해당하는 타깃은?
- ① multi-user.target
- ② graphical.target
- ③ rescue.target
- ④ emergency.target

**3.** 다음 중 저널링(journaling)을 지원하지 않는 파일시스템은?
- ① ext4
- ② xfs
- ③ btrfs
- ④ ext2

---

## 리눅스 시스템의 이해 — SHELL·프로세스·텍스트 처리

**4.** 현재 셸에 정의한 변수를 이후 실행되는 자식 프로세스에 상속시키는 명령은?
- ① `set`
- ② `env`
- ③ `alias`
- ④ `export`

**5.** 표준출력과 표준에러를 모두 `result.log` 파일로 보내는 표현으로 옳은 것은?
- ① `cmd > result.log`
- ② `cmd 2> result.log`
- ③ `cmd > result.log 2>&1`
- ④ `cmd &> /dev/null`

**6.** `grep` 정규표현식에서 문자열 `error` 로 **시작하는** 행만 출력하는 패턴은?
- ① `grep 'error$'`
- ② `grep '\<error'`
- ③ `grep 'error.'`
- ④ `grep '^error'`

**7.** `sed` 로 파일 내 모든 `apple` 을 `orange` 로 치환하는 명령은?
- ① `sed 's/apple/orange/'`
- ② `sed 's/apple/orange/g'`
- ③ `sed 'y/apple/orange/'`
- ④ `sed 's/orange/apple/g'`

**8.** 공백으로 구분된 입력에서 세 번째 필드만 출력하는 `awk` 표현은?
- ① `awk '{print 3}'`
- ② `awk '{print $3}'`
- ③ `awk -F3 '{print}'`
- ④ `awk '{$3}'`

**9.** 프로세스가 무시하거나 가로챌 수 없어 강제 종료에 사용되는 시그널은?
- ① SIGKILL (9)
- ② SIGTERM (15)
- ③ SIGHUP (1)
- ④ SIGINT (2)

---

## X 윈도

**10.** X 클라이언트 애플리케이션의 출력을 표시할 화면(디스플레이)을 지정하는 환경변수는?
- ① `$TERM`
- ② `$DISPLAY`
- ③ `$XAUTHORITY`
- ④ `$SHELL`

---

## 네트워크의 이해

**11.** 한 서브넷에 호스트 250대를 수용하기 위한 최소 서브넷 프리픽스는?
- ① /26
- ② /25
- ③ /24
- ④ /23

**12.** IMAP 서비스가 기본으로 사용하는 포트 번호는?
- ① 143
- ② 110
- ③ 993
- ④ 25

**13.** IPv6 주소의 총 비트 수로 옳은 것은?
- ① 32
- ② 64
- ③ 128
- ④ 256

---

## 시스템 관리·장치 관리

**14.** 매주 일요일 오전 3시에 작업을 실행하는 crontab 항목으로 옳은 것은? (필드: 분 시 일 월 요일)
- ① `3 0 * * 0`
- ② `0 3 * * 0`
- ③ `0 3 0 * *`
- ④ `* 3 * * 7`

**15.** 사용자 user1 의 기존 보조 그룹을 유지하면서 dev 그룹을 추가로 등록하는 명령은?
- ① `usermod -g dev user1`
- ② `usermod -G dev user1`
- ③ `usermod -A dev user1`
- ④ `usermod -aG dev user1`

**16.** LVM 구성에서 여러 물리 볼륨(PV)을 묶어 만든 저장 풀에 해당하는 것은?
- ① PE
- ② PV
- ③ VG
- ④ LV

---

## 시스템 보안·네트워크 서비스·네트워크 보안

**17.** 로그인 **실패** 이력을 저장하는 로그 파일과 조회 명령의 연결로 옳은 것은?
- ① `/var/log/btmp` — `lastb`
- ② `/var/log/wtmp` — `last`
- ③ `/var/run/utmp` — `who`
- ④ `/var/log/lastlog` — `lastlog`

**18.** Apache(httpd) 설정에서 웹 문서의 최상위 디렉터리를 지정하는 지시자는?
- ① `ServerRoot`
- ② `DirectoryIndex`
- ③ `DocumentRoot`
- ④ `UserDir`

**19.** iptables 에서 INPUT 체인의 기본 정책을 차단(DROP)으로 설정하는 명령은?
- ① `iptables -F INPUT`
- ② `iptables -A INPUT DROP`
- ③ `iptables -P INPUT DROP`
- ④ `iptables -D INPUT`

**20.** 다음 중 대칭키 암호 알고리즘이 **아닌** 것은?
- ① AES
- ② RSA
- ③ SEED
- ④ DES
