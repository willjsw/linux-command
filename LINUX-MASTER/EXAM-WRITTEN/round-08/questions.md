---
title: 필기 모의고사 Round 08 — 문제
type: exam-written
round: 8
tags:
  - exam/linux-master
  - exam/written
  - exam/questions
updated: 2026-08-28
---

# 필기 모의고사 Round 08 — 문제

- 20문항 / 목표 20분 / 4지선다
- 채점·해설: [[answers]] (풀이 완료 후 열람)

---

## 리눅스 실무·시스템의 이해

**1.** 다음 라이선스 중 카피레프트(copyleft) 성격이 가장 강해 파생 저작물에도 동일 라이선스를 강제하는 것은?
- ① BSD
- ② MIT
- ③ GPL
- ④ Apache

**2.** 레드햇 계열에서 의존성을 자동으로 해결하며 패키지를 설치하는 명령으로 옳은 것은?
- ① `dnf install <패키지>`
- ② `rpm -ivh <패키지>`
- ③ `dpkg -i <패키지>`
- ④ `tar xvf <패키지>`

**3.** systemd 에서 X 윈도 없이 다중 사용자 텍스트 모드로 부팅되는 타깃(target)은?
- ① `rescue.target`
- ② `graphical.target`
- ③ `multi-user.target`
- ④ `poweroff.target`

**4.** BIOS 기반 시스템에서 1차 부트로더(GRUB stage1)가 저장되는 위치로 옳은 것은?
- ① 스왑 파티션
- ② MBR(디스크 첫 512바이트)
- ③ `/boot/grub2` 디렉터리
- ④ `/etc/default/grub`

**5.** 특정 파일이 어느 rpm 패키지에 의해 설치되었는지 조회하는 명령은?
- ① `rpm -qa`
- ② `rpm -ql`
- ③ `rpm -qf <파일>`
- ④ `rpm -V`

---

## 리눅스 시스템의 이해 — SHELL·프로세스·X 윈도

**6.** 명령 실행 파일을 탐색할 디렉터리 경로 목록을 저장하는 환경변수는?
- ① `$HOME`
- ② `$PATH`
- ③ `$SHELL`
- ④ `$PS1`

**7.** 프로세스에 정상 종료(gracefully)를 요청하며 `kill` 이 기본으로 전송하는 시그널 번호는?
- ① 1 (SIGHUP)
- ② 9 (SIGKILL)
- ③ 15 (SIGTERM)
- ④ 19 (SIGSTOP)

**8.** 자식 프로세스가 종료되었으나 부모가 `wait()` 로 회수하지 않아 프로세스 테이블에 항목만 남은 상태는?
- ① 좀비(Zombie) 프로세스
- ② 데몬(Daemon) 프로세스
- ③ 고아(Orphan) 프로세스
- ④ 세션 리더

**9.** 다음 중 X 윈도의 디스플레이 매니저(Display Manager)가 아닌 것은?
- ① GDM
- ② KDM
- ③ LightDM
- ④ GNOME

**10.** 포그라운드로 실행 중인 작업을 일시중지한 뒤 백그라운드로 전환하는 절차로 옳은 것은?
- ① `Ctrl+c` 후 `fg`
- ② `Ctrl+z` 후 `bg`
- ③ `Ctrl+d` 후 `jobs`
- ④ `Ctrl+z` 후 `kill`

---

## 네트워크의 이해

**11.** Apache(httpd)에서 실제 웹 문서가 위치하는 루트 디렉터리를 지정하는 지시자는?
- ① `ServerRoot`
- ② `ServerName`
- ③ `DocumentRoot`
- ④ `DirectoryIndex`

**12.** DNS 존 파일에서 메일 서버를 지정하며 숫자값이 낮을수록 우선하는 레코드는?
- ① `NS`
- ② `A`
- ③ `CNAME`
- ④ `MX`

**13.** NFS 서버에서 공유할 디렉터리와 접근 옵션을 정의하는 설정 파일은?
- ① `/etc/exports`
- ② `/etc/fstab`
- ③ `/etc/samba/smb.conf`
- ④ `/etc/nfs.conf`

**14.** SMTP, POP3, IMAP 의 기본 포트를 순서대로 바르게 나열한 것은?
- ① 21, 22, 25
- ② 25, 143, 110
- ③ 25, 110, 143
- ④ 110, 143, 25

**15.** 서브넷 마스크 `255.255.255.192` 를 CIDR 프리픽스로 표기하면?
- ① /24
- ② /25
- ③ /26
- ④ /27

---

## 시스템 관리·장치 관리

**16.** LVM 구성 요소 중 여러 물리 볼륨(PV)을 묶어 만든 저장 풀에 해당하는 것은?
- ① 볼륨 그룹(VG)
- ② 물리 볼륨(PV)
- ③ 논리 볼륨(LV)
- ④ 물리 익스텐트(PE)

**17.** 디스크 2개를 미러링하여 가용 용량은 절반이 되지만 내결함성을 제공하는 RAID 레벨은?
- ① RAID 0
- ② RAID 1
- ③ RAID 5
- ④ RAID 6

**18.** `/dev/sdb1` 파티션에 ext4 파일시스템을 생성하는 명령으로 옳은 것은?
- ① `tune2fs /dev/sdb1`
- ② `fsck.ext4 /dev/sdb1`
- ③ `mount -t ext4 /dev/sdb1`
- ④ `mkfs -t ext4 /dev/sdb1`

**19.** 2TB 를 초과하는 디스크에 GPT 파티션을 생성할 때 적합한 도구는?
- ① `fdisk`
- ② `parted`
- ③ `mkfs`
- ④ `df`

**20.** 파일시스템별 전체 용량과 남은 공간을 확인하는 명령은?
- ① `df`
- ② `du`
- ③ `fdisk`
- ④ `lsblk`
