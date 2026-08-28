---
title: 필기 모의고사 Round 06 — 문제
type: exam-written
round: 6
tags:
  - exam/linux-master
  - exam/written
  - exam/questions
updated: 2026-08-28
---

# 필기 모의고사 Round 06 — 문제

- 20문항 / 목표 20분 / 4지선다
- 채점·해설: [[answers]] (풀이 완료 후 열람)

---

## 리눅스 실무·시스템의 이해

**1.** 리눅스 배포판 간 디렉터리 구조·파일 위치의 표준을 규정하는 규격은?
- ① FHS
- ② POSIX
- ③ LSB
- ④ GNU

**2.** SysV init 런레벨 중 시스템 재시작(reboot)에 해당하는 것은?
- ① 0
- ② 1
- ③ 5
- ④ 6

**3.** 저장소에서 패키지를 내려받으며 의존성을 자동으로 해결해 설치하는 도구는?
- ① `dnf`
- ② `rpm`
- ③ `dpkg`
- ④ `tar`

---

## 리눅스 시스템의 이해 — SHELL·프로세스·텍스트 처리

**4.** 현재 셸에 설정된 **환경변수만** 출력하는 명령으로 옳은 것은?
- ① `alias`
- ② `jobs`
- ③ `env`
- ④ `which`

**5.** `wc -l < list.txt` 에서 기호 `<` 의 역할로 옳은 것은?
- ① 표준출력을 파일로 저장
- ② 표준입력을 파일 내용으로 대체
- ③ 표준에러를 재지정
- ④ 두 명령을 파이프로 연결

**6.** `cat access.log | grep 404 | wc -l` 명령의 실행 결과로 옳은 것은?
- ① 문자열 404 의 등장 횟수
- ② 404 가 포함된 행의 개수
- ③ 파일 전체 행의 개수
- ④ 파일의 바이트 크기

**7.** 문자열 `cat` 또는 `dog` 가 포함된 행을 찾는 명령으로 옳은 것은?
- ① `grep 'cat|dog' file`
- ② `grep -F 'cat|dog' file`
- ③ `grep -w 'cat|dog' file`
- ④ `grep -E 'cat|dog' file`

**8.** `/etc/passwd` 에서 콜론(`:`)으로 구분된 첫 번째 필드(사용자명)만 출력하는 명령은?
- ① `awk -F: '{print $1}' /etc/passwd`
- ② `awk '{print $1}' /etc/passwd`
- ③ `awk -F: '{print 1}' /etc/passwd`
- ④ `awk ':' '{print $1}' /etc/passwd`

**9.** 명령을 백그라운드로 실행하기 위해 명령 끝에 붙이는 기호는?
- ① `&`
- ② `;`
- ③ `&&`
- ④ `|`

---

## X 윈도

**10.** 그래픽 로그인 화면을 제공하고 사용자 인증 후 X 세션을 시작하는 구성요소는?
- ① 윈도 매니저
- ② 데스크톱 환경
- ③ 위젯 툴킷
- ④ 디스플레이 매니저

---

## 네트워크의 이해

**11.** 이더넷 MAC 주소의 총 비트 수는?
- ① 16
- ② 32
- ③ 48
- ④ 64

**12.** 다음 중 ICMP 프로토콜을 이용해 대상 호스트의 도달 여부를 진단하는 명령은?
- ① `dig`
- ② `ssh`
- ③ `ftp`
- ④ `ping`

**13.** TCP 연결 수립(3-way handshake)의 순서로 옳은 것은?
- ① SYN → ACK → SYN/ACK
- ② ACK → SYN → SYN/ACK
- ③ SYN → SYN/ACK → ACK
- ④ SYN/ACK → SYN → ACK

---

## 시스템 관리·장치 관리

**14.** 지정한 시각에 작업을 **한 번만** 실행하도록 예약하는 명령은?
- ① `cron`
- ② `anacron`
- ③ `at`
- ④ `batch`

**15.** 파일의 그룹(group)에 쓰기 권한을 추가하는 심볼릭 모드 명령은?
- ① `chmod u+w file`
- ② `chmod o+w file`
- ③ `chmod a-w file`
- ④ `chmod g+w file`

**16.** 마운트된 파일시스템별 디스크 사용량과 남은 용량을 확인하는 명령은?
- ① `du`
- ② `df`
- ③ `lsblk`
- ④ `mount`

---

## 시스템 보안·네트워크 서비스·네트워크 보안

**17.** `/tmp` 처럼 디렉터리 안에서 자기 소유 파일만 삭제할 수 있도록 제한하는 특수 권한은?
- ① Sticky Bit
- ② SetUID
- ③ SetGID
- ④ umask

**18.** NFS 서버에서 공유할 디렉터리와 접근 권한을 정의하는 설정 파일은?
- ① `/etc/fstab`
- ② `/etc/exports`
- ③ `/etc/samba/smb.conf`
- ④ `/etc/nfs.conf`

**19.** TCP 3-way handshake 중 SYN 패킷만 대량 전송해 백로그 큐를 고갈시키는 공격은?
- ① Smurf
- ② Land
- ③ SYN Flooding
- ④ Teardrop

**20.** SELinux 의 현재 동작 모드(Enforcing/Permissive/Disabled)를 확인하는 명령은?
- ① `setenforce`
- ② `getenforce`
- ③ `semanage`
- ④ `firewall-cmd`
