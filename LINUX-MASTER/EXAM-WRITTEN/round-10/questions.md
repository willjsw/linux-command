---
title: 필기 모의고사 Round 10 — 문제
type: exam-written
round: 10
tags:
  - exam/linux-master
  - exam/written
  - exam/questions
updated: 2026-08-28
---

# 필기 모의고사 Round 10 — 문제

- 20문항 / 목표 20분 / 4지선다
- 채점·해설: [[answers]] (풀이 완료 후 열람)

---

## 리눅스 실무·시스템의 이해

**1.** 다음 중 리눅스 운영체제의 특징으로 옳지 않은 것은?
- ① 다중 사용자·다중 작업(multi-user, multi-tasking)을 지원한다
- ② 소스가 공개된 오픈 소스 운영체제이다
- ③ 한 번에 한 명만 사용 가능한 단일 사용자 전용이다
- ④ 다양한 하드웨어로의 이식성이 높다

**2.** `rpm -V <패키지>` 명령의 용도로 옳은 것은?
- ① 패키지를 새로 설치한다
- ② 패키지를 삭제한다
- ③ 패키지의 의존성 목록을 표시한다
- ④ 설치된 파일의 변경(무결성) 여부를 검증한다

**3.** 현재 실행 중인 시스템을 그래픽 모드로 즉시 전환하는 명령은?
- ① `systemctl isolate multi-user.target`
- ② `systemctl isolate graphical.target`
- ③ `systemctl set-default rescue.target`
- ④ `init 1`

**4.** 부팅 초기 단계에서 루트 파일시스템 마운트에 필요한 드라이버를 담아 메모리에 적재되는 임시 파일시스템은?
- ① MBR
- ② 스왑(swap)
- ③ initramfs(initrd)
- ④ GRUB

**5.** 실행 파일이 의존하는 공유 라이브러리 목록을 확인하는 명령은?
- ① `ldd`
- ② `ldconfig`
- ③ `nm`
- ④ `strace`

---

## 리눅스 시스템의 이해 — SHELL·프로세스·X 윈도

**6.** 명령의 표준 에러(stderr)만 파일로 리다이렉트하는 표현으로 옳은 것은?
- ① `명령 > file`
- ② `명령 < file`
- ③ `명령 1> file`
- ④ `명령 2> file`

**7.** 긴 명령이나 명령 조합에 짧은 별칭을 부여하는 셸 내장 명령은?
- ① `alias`
- ② `export`
- ③ `set`
- ④ `env`

**8.** 이미 실행 중인 프로세스의 우선순위(nice 값)를 변경하는 명령은?
- ① `nice`
- ② `kill`
- ③ `renice`
- ④ `top`

**9.** 원격에서 실행되는 X 클라이언트가 출력을 보낼 X 서버(화면)를 지정하는 환경변수는?
- ① `$TERM`
- ② `$DISPLAY`
- ③ `$XAUTHORITY`
- ④ `$PATH`

**10.** CPU·메모리 사용률과 프로세스 목록을 실시간으로 갱신하며 보여주는 대화형 명령은?
- ① `ps`
- ② `jobs`
- ③ `nice`
- ④ `top`

---

## 네트워크의 이해

**11.** Apache 설정 파일의 문법 오류를 점검하는 명령으로 옳은 것은?
- ① `httpd -t`
- ② `httpd -k restart`
- ③ `apachectl start`
- ④ `systemctl reload httpd`

**12.** FTP 능동(Active) 모드에서 데이터 전송에 사용되는 서버 측 포트는?
- ① 21
- ② 20
- ③ 22
- ④ 80

**13.** IP 주소를 도메인명으로 변환하는 역방향 조회에 사용되는 DNS 레코드는?
- ① `A`
- ② `MX`
- ③ `CNAME`
- ④ `PTR`

**14.** 메일 시스템 구성 요소 중 메일을 실제로 전송·라우팅하는 역할은?
- ① MUA
- ② MDA
- ③ MTA
- ④ MRA

**15.** 현재 시스템에서 LISTEN 상태인 TCP 포트를 확인하는 명령으로 옳은 것은?
- ① `ss -tlnp`
- ② `ping`
- ③ `traceroute`
- ④ `ss -u` 만 사용

---

## 시스템 관리·장치 관리

**16.** MBR 파티션 구조에서 확장(Extended) 파티션 안에 여러 개 생성할 수 있는 파티션 유형은?
- ① 주(Primary) 파티션
- ② 논리(Logical) 파티션
- ③ 확장(Extended) 파티션
- ④ 스왑 파티션

**17.** 패리티를 이중으로 저장해 디스크 2개가 동시에 고장나도 데이터를 유지하는 RAID 레벨은?
- ① RAID 0
- ② RAID 5
- ③ RAID 10
- ④ RAID 6

**18.** LVM 에서 물리 디스크(파티션)를 LVM 이 사용할 수 있도록 초기화하는 명령은?
- ① `pvcreate`
- ② `vgcreate`
- ③ `lvcreate`
- ④ `mkfs`

**19.** 디스크를 블록 단위로 통째로 복제하여 이미지를 만드는 명령은?
- ① `cp`
- ② `tar`
- ③ `dd`
- ④ `rsync`

**20.** 특정 디렉터리가 차지하는 실제 디스크 사용량을 하위까지 재귀적으로 합산해 보여주는 명령은?
- ① `df`
- ② `du`
- ③ `lsblk`
- ④ `free`
