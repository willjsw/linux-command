---
title: 필기 모의고사 Round 07 — 문제
type: exam-written
round: 7
tags:
  - exam/linux-master
  - exam/written
  - exam/questions
updated: 2026-08-28
---

# 필기 모의고사 Round 07 — 문제

- 20문항 / 목표 20분 / 4지선다
- 채점·해설: [[answers]] (풀이 완료 후 열람)

---

## 리눅스 실무·시스템의 이해

**1.** 리누스 토르발스가 리눅스 커널을 개발할 때 참고한, 교육용으로 만들어진 유닉스 계열 운영체제는?
- ① MINIX
- ② MULTICS
- ③ BSD
- ④ Solaris

**2.** 다음 중 커널(kernel)의 역할로 보기 어려운 것은?
- ① 프로세스 관리
- ② 사용자 명령어 해석
- ③ 메모리 관리
- ④ 장치 제어

**3.** 부팅에 필요한 커널 이미지(vmlinuz)와 initramfs 가 저장되는 디렉터리는?
- ① `/sys`
- ② `/proc`
- ③ `/boot`
- ④ `/lib`

---

## 리눅스 시스템의 이해 — SHELL·프로세스·텍스트 처리

**4.** 명령의 실행 결과(출력)를 다른 명령의 인자로 삽입하는 명령 치환(command substitution) 문법은?
- ① `${cmd}`
- ② `$[cmd]`
- ③ `$<cmd>`
- ④ `$(cmd)`

**5.** 앞 명령이 **성공(종료 코드 0)했을 때만** 뒤 명령을 실행하는 연산자는?
- ① `&&`
- ② `;`
- ③ `||`
- ④ `|`

**6.** 기존 파일 내용을 유지한 채 출력을 파일 끝에 덧붙이는 리다이렉션 기호는?
- ① `>`
- ② `>>`
- ③ `<`
- ④ `2>`

**7.** `grep` 에서 대소문자를 구분하지 않고 검색하는 옵션은?
- ① `-v`
- ② `-c`
- ③ `-i`
- ④ `-n`

**8.** `sed` 로 입력의 세 번째 행을 삭제하는 명령은?
- ① `sed 'd3'`
- ② `sed '/3/d'`
- ③ `sed -n '3p'`
- ④ `sed '3d'`

**9.** `ps` 출력의 STAT 항목이 `Z` 로 표시되는 프로세스의 상태는?
- ① 좀비(Zombie)
- ② 실행 대기(Running)
- ③ 인터럽트 가능 대기(Sleep)
- ④ 정지(Stopped)

---

## X 윈도

**10.** X 윈도에서 창의 이동·크기 조절·테두리 등 창 배치를 관리하는 구성요소는?
- ① 디스플레이 매니저
- ② 윈도 매니저
- ③ 데스크톱 환경
- ④ 위젯 툴킷

---

## 네트워크의 이해

**11.** DNS 에서 호스트 이름을 IPv4 주소로 매핑하는 자원 레코드는?
- ① MX
- ② PTR
- ③ A
- ④ CNAME

**12.** 네트워크 인터페이스의 IP 주소와 상태를 확인하는 명령(ifconfig 대체)은?
- ① `route`
- ② `arp`
- ③ `ss`
- ④ `ip addr`

**13.** 목적지까지 거치는 경로상의 라우터(홉)를 순서대로 추적하는 명령은?
- ① `traceroute`
- ② `ping`
- ③ `netstat`
- ④ `nslookup`

---

## 시스템 관리·장치 관리

**14.** 서비스가 부팅 시 자동으로 시작되도록 등록하는 systemctl 하위 명령은?
- ① `start`
- ② `enable`
- ③ `status`
- ④ `restart`

**15.** 블록 장치와 파티션을 트리 형태로 나열해 계층 구조를 보여 주는 명령은?
- ① `fdisk -l`
- ② `df`
- ③ `lsblk`
- ④ `mount`

**16.** 이미 생성된 스왑 영역을 활성화(사용 시작)하는 명령은?
- ① `mkswap`
- ② `mount`
- ③ `free`
- ④ `swapon`

---

## 시스템 보안·네트워크 서비스·네트워크 보안

**17.** `/etc/sudoers` 파일을 문법 검사 기능과 함께 안전하게 편집하는 전용 명령은?
- ① `visudo`
- ② `vi /etc/sudoers`
- ③ `sudoedit`
- ④ `usermod`

**18.** vsftpd 에서 익명(anonymous) 사용자의 접속을 차단하는 설정 항목은?
- ① `local_enable=NO`
- ② `anonymous_enable=NO`
- ③ `write_enable=NO`
- ④ `chroot_local_user=NO`

**19.** firewalld 에서 http 서비스를 영구적으로 개방하고 즉시 반영하는 절차로 옳은 것은?
- ① `firewall-cmd --add-port=80`
- ② `iptables -A INPUT -p tcp --dport 80 -j ACCEPT`
- ③ `firewall-cmd --add-service=http --permanent` 실행 후 `firewall-cmd --reload`
- ④ `systemctl reload httpd`

**20.** 파일의 무결성 기준선을 저장해 두고 변조 여부를 탐지하는 HIDS 도구는?
- ① Nmap
- ② John the Ripper
- ③ tcpdump
- ④ Tripwire
