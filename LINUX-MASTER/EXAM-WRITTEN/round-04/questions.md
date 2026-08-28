---
title: 필기 모의고사 Round 04 — 문제
type: exam-written
round: 4
tags:
  - exam/linux-master
  - exam/written
  - exam/questions
updated: 2026-08-28
---

# 필기 모의고사 Round 04 — 문제

- 20문항 / 목표 20분 / 4지선다
- 채점·해설: [[answers]] (풀이 완료 후 열람)

---

## 리눅스 실무·시스템의 이해

**1.** 벨 연구소에서 최초의 유닉스(UNIX)를 개발한 인물로 옳은 것은?
- ① 리누스 토르발스
- ② 켄 톰슨·데니스 리치
- ③ 리처드 스톨먼
- ④ 앤드루 타넨바움

**2.** RHEL 8 이상에서 rpm 패키지의 의존성을 자동 해결하며 설치하는 기본 패키지 관리 명령은?
- ① `apt`
- ② `dnf`
- ③ `dpkg`
- ④ `zypper`

**3.** GRUB2 에서 기본 부팅 항목·타임아웃 등 사용자 설정을 편집한 뒤 `grub2-mkconfig` 로 반영하는 원본 설정 파일은?
- ① `/boot/grub2/grub.cfg`
- ② `/etc/default/grub`
- ③ `/etc/grub.conf`
- ④ `/etc/fstab`

---

## 리눅스 시스템의 이해 — SHELL·프로세스·X 윈도

**4.** bash 사용자가 개인용 alias 와 함수를 정의할 때 주로 사용하는 사용자별 설정 파일은?
- ① `/etc/profile`
- ② `~/.bashrc`
- ③ `/etc/bashrc`
- ④ `~/.bash_history`

**5.** 명령을 실행할 때 우선순위(nice 값)를 지정하여 새 프로세스를 시작하는 명령은?
- ① `renice`
- ② `nice`
- ③ `nohup`
- ④ `top`

**6.** 콘솔에서 X 윈도 세션을 수동으로 시작하는 명령으로 옳은 것은?
- ① `startx`
- ② `xhost`
- ③ `xauth`
- ④ `Xorg -configure`

---

## 네트워크의 이해

**7.** IP 주소를 물리 주소(MAC)로 변환하는 데 사용하는 프로토콜은?
- ① ARP
- ② RARP
- ③ ICMP
- ④ DHCP

**8.** C 클래스 네트워크(`/24`)를 `/28` 로 서브네팅할 때 생성되는 서브넷의 개수는?
- ① 4개
- ② 8개
- ③ 16개
- ④ 32개

**9.** TCP 연결 해제(4-way handshake) 과정에서 사용되는 플래그로 옳은 것은?
- ① SYN
- ② FIN
- ③ RST
- ④ PSH

---

## 시스템 관리·장치 관리

**10.** 기존 사용자 `user1` 의 로그인 셸을 `/bin/bash` 로 변경하는 명령으로 옳은 것은?
- ① `usermod -s /bin/bash user1`
- ② `usermod -d /bin/bash user1`
- ③ `useradd -s /bin/bash user1`
- ④ `usermod -g /bin/bash user1`

**11.** 다음 중 문자 장치(character device)에 해당하는 것은?
- ① 하드디스크 (`/dev/sda`)
- ② 터미널 (`/dev/tty`)
- ③ USB 저장장치 (`/dev/sdb`)
- ④ SSD (`/dev/nvme0n1`)

**12.** 사용자·그룹별로 디스크 사용량을 제한하는 기능으로 옳은 것은?
- ① RAID
- ② LVM
- ③ Quota
- ④ Swap

---

## 시스템 보안 및 관리

**13.** SetGID 비트(8진수 2000)가 설정된 파일을 시스템 전체에서 검색하는 명령으로 옳은 것은?
- ① `find / -perm -4000 -type f`
- ② `find / -perm -2000 -type f`
- ③ `find / -perm -1000 -type d`
- ④ `find / -perm -0002 -type f`

**14.** TCP Wrapper 의 호스트 접근 제어에서 두 파일을 검사하는 순서로 옳은 것은?
- ① `/etc/hosts.deny` → `/etc/hosts.allow`
- ② `/etc/hosts.allow` → `/etc/hosts.deny`
- ③ `/etc/hosts.allow` 만 검사
- ④ `iptables` → `/etc/hosts.allow`

**15.** 특정 사용자의 비밀번호 만료·최대 사용 기간 등 계정 정책을 조회하는 명령으로 옳은 것은?
- ① `chage -l user1`
- ② `passwd -l user1`
- ③ `usermod -L user1`
- ④ `id user1`

**16.** systemd 저널에서 오류(err) 이상 우선순위의 메시지만 필터링하여 조회하는 명령은?
- ① `journalctl -p err`
- ② `journalctl -u err`
- ③ `journalctl -b err`
- ④ `journalctl -f err`

---

## 네트워크 서비스·네트워크 보안

**17.** 윈도 클라이언트와의 파일·프린터 공유를 제공하는 Samba 서비스의 주 설정 파일은?
- ① `/etc/exports`
- ② `/etc/samba/smb.conf`
- ③ `/etc/vsftpd/vsftpd.conf`
- ④ `/etc/nsswitch.conf`

**18.** 방화벽 뒤에 있는 클라이언트가 FTP 서버에 접속할 때 일반적으로 권장되는 데이터 전송 모드는?
- ① 능동(Active) 모드
- ② 수동(Passive) 모드
- ③ 브로드캐스트 모드
- ④ 멀티캐스트 모드

**19.** 외부 80번 포트로 들어온 요청을 내부 서버 `192.168.0.10:8080` 으로 전달하는 포트 포워딩에 사용하는 iptables NAT 유형은?
- ① SNAT
- ② DNAT
- ③ MASQUERADE
- ④ REJECT

**20.** 침입 탐지 시스템(IDS)에서 정상 트래픽을 공격으로 잘못 판단하는 오류를 가리키는 용어는?
- ① 미탐 (False Negative)
- ② 오탐 (False Positive)
- ③ 오용 탐지
- ④ 이상 탐지
