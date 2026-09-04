---
title: LAB — 리눅스마스터 1급 UTM 실습 절차서 (허브)
type: exam-lab-hub
tags:
  - exam/linux-master
  - exam/lab
  - moc/index
distro: Rocky Linux 9 (aarch64, UTM on Apple Silicon)
updated: 2026-09-03
---

# LAB — 리눅스마스터 1급 UTM 실습 절차서

- 목적: 필기·실기 기출과 이론 12종에 등장하는 **모든 명령·설정을 실제 VM에서 최소 1~2회 수행**
- 형식: 하나의 가상 시나리오를 12개 파트로 나눈 **작업 절차서** — 자원 생성 → 검증 → 진단 → 백업/복구까지 일련 흐름
- 원칙: **만들었으면 반드시 확인**. 모든 생성·변경 명령 뒤에 검증 명령과 기대 출력이 따라옴
- 이론 역참조: [[../THEORY/linux-basics|THEORY]] 12종, 기출 역참조: `EXAM-PRACTICAL`, `EXAM-WRITTEN-FULL`

---

## 0. 시나리오

> 소규모 개발팀(개발 2명, 운영 1명, 관리자 1명)이 사내 인트라넷용 리눅스 서버 1대를 새로 구축한다.
> 서버는 **팀 파일 공유(Samba·NFS·FTP), 사내 웹(Apache), 내부 DNS(BIND), 로컬 메일(Postfix), 컨테이너 기반 캐시(Redis on Docker)** 를 제공하고,
> 별도 디스크에 **LVM·RAID·스왑·쿼터**를 구성하며, **SSH 강화·firewalld/iptables·SELinux** 로 보호한다.
> 운영 중 **CPU 폭주·I/O 병목·좀비 프로세스** 장애를 `top`·`vmstat`·`iostat`·`sar` 로 진단하고, **cron 백업**과 **root 비밀번호 복구** 까지 검증한다.

- 서버 호스트명: `srv01.lab.local`  (도메인 `lab.local`)
- 하나의 VM 안에서 서버·클라이언트 역할 모두 수행 (루프백 `127.0.0.1` 또는 자기 IP로 접속 검증)

---

## 1. 실습 환경 (UTM)

| 항목                 | 값                                                                                   | 비고                                         |
| ------------------ | ----------------------------------------------------------------------------------- | ------------------------------------------ |
| 하이퍼바이저 | UTM 4.x (Apple Silicon) — **QEMU 백엔드** ('Use Apple Virtualization' 체크 해제) | 스냅샷 사용 가능. 중첩 가상화 미지원 → Part 11 `virsh` 는 조회 위주 |
| 게스트 OS | Rocky Linux 9.x **aarch64** minimal ISO — **직접 다운로드 필요** ([[01-vm-setup-and-inspection]] 1-0) | 시험 기준 RHEL 계열. x86_64 차이는 각 파트에 명시 |
| CPU / RAM          | 2 vCPU / 4 GB                                                                       | `top` 진단 시 부하 재현용                          |
| 시스템 디스크            | `/dev/vda` 40 GB (자동 파티션: `/boot/efi`, `/boot`, LVM VG `rlm`)                        | UTM virtio → **`vd*`** 표기. SATA 선택 시 `sd*` |
| 추가 디스크             | `/dev/vdb` 5 GB · `/dev/vdc` 5 GB · `/dev/vdd` 5 GB · `/dev/vde` 2 GB               | UTM 설정 → Drives → New… (VirtIO) 로 4개 추가    |
| 네트워크               | Shared Network (NAT) — 인터페이스 `enp0s1`, 서브넷 `192.168.64.0/24`, GW/DNS `192.168.64.1` | 실제 값은 `ip a` 로 확인 후 치환                     |
| 고정 IP (Part 08 이후) | `192.168.64.10/24`                                                                  | 그 전까지 DHCP                                 |
| 디스플레이 | 디스플레이 카드를 **`virtio-ramfb`** 로 변경 필수 | UTM 기본값 `virtio-gpu-pci` 는 화면이 검게 나옴 ([[01-vm-setup-and-inspection]] 1-2 ③) |
| 콘솔 | **평소 macOS 터미널 `ssh`** + UTM 창의 **직렬 포트 탭** | 직렬 포트 장치 추가 후 `console=ttyAMA0` 지정 (1-2 ④, 2-3). Part 07 복구·Part 08 네트워크 변경 시 유일한 통로 |

- 설치 시 root 비밀번호 설정, 일반 사용자 **`admin1`** 을 "관리자(wheel)" 로 생성
- 스냅샷 권장 지점: Part 01 완료 후, Part 05 진입 전, Part 10 진입 전 (UTM → 스냅샷 기능은 QEMU 백엔드에서만 가능)

---

## 2. 자원 명세 (전 파트 공통 — 이 값으로 통일)

### 2-1. 계정·그룹

| 구분 | 이름 | UID/GID | 소속 | 용도 |
| --- | --- | --- | --- | --- |
| 관리자 | `admin1` | 1000 | `wheel` | sudo 관리자 (설치 시 생성) |
| 그룹 | `devteam` | 2000 | — | 개발팀 |
| 그룹 | `opsteam` | 2001 | — | 운영팀 |
| 사용자 | `dev1` | 2001 | `devteam` (1차) | 개발자, Samba·FTP 사용 |
| 사용자 | `dev2` | 2002 | `devteam`, `opsteam`(보조) | 개발자 |
| 사용자 | `ops1` | 2003 | `opsteam`, `wheel`(보조) | 운영자, sudo 제한 부여 |
| 사용자 | `guest1` | 2004 | — | 임시 계정 → 잠금·만료·삭제 실습 |

- 비밀번호: 명령에 직접 쓰지 않고 `passwd <user>` 로 대화식 입력. 절차서 안의 `<비밀번호>` 는 자리표시자(실사용 값 아님)

### 2-2. 디스크 배치

| 장치 | 용량 | 용도 | 파일시스템 | 마운트 | 파트 |
| --- | --- | --- | --- | --- | --- |
| `/dev/vdb1` | 2 GB | 데이터 파티션 (+ 디스크 쿼터) | ext4 | `/data` | 05 |
| `/dev/vdb2` | 1 GB | 스왑 파티션 | swap | — | 05 |
| `/dev/vdb3` | 나머지 | LVM PV (VG 확장용) | — | — | 05 |
| `/dev/vdc` + `/dev/vdd` | 5 GB × 2 | RAID 1 `/dev/md0` | xfs | `/srv/raid` (백업 대상지) | 05 |
| `/dev/vde` | 2 GB | LVM PV → VG `vg_lab` → LV `lv_share` (1 GB → 확장) | xfs | `/srv/share` (Samba 공유) | 05 |
| 파일 | 512 MB | 스왑 파일 `/swapfile` | swap | — | 05 |

### 2-3. 서비스·경로·포트

| 서비스 | 패키지/데몬 | 주 설정 | 실습 경로 | 포트 | 파트 |
| --- | --- | --- | --- | --- | --- |
| SSH | `openssh-server` / `sshd` | `/etc/ssh/sshd_config` | 키 인증, 포트 **2222** | 2222 | 08 |
| 웹 | `httpd` | `/etc/httpd/conf/httpd.conf`, `conf.d/intranet.conf` | `/srv/www/intranet` (vhost `intranet.lab.local`) | 80, 8080, 443 | 09 |
| DNS | `bind` / `named` | `/etc/named.conf`, `/var/named/lab.local.zone` | 정방향·역방향 존 | 53 | 09 |
| NFS | `nfs-utils` / `nfs-server` | `/etc/exports` | `/srv/nfs/data` → 클라이언트 `/mnt/nfs` | 2049, 111 | 09 |
| Samba | `samba` / `smb`, `nmb` | `/etc/samba/smb.conf` | 공유 `[share]` = `/srv/share` → 클라이언트 `/mnt/smb` | 445, 139 | 09 |
| FTP | `vsftpd` | `/etc/vsftpd/vsftpd.conf` | `/var/ftp/pub`, 사용자 홈 | 21 | 09 |
| 메일 | `postfix`, `s-nail` | `/etc/postfix/main.cf`, `/etc/aliases` | 로컬 배송 `dev1` → `ops1` | 25 | 09 |
| 시간 | `chrony` / `chronyd` | `/etc/chrony.conf` | — | 123 | 08 |
| 인쇄 | `cups` | — | 더미 프린터 `labprn` | 631 | 09 |
| 컨테이너 | `docker-ce` / `dockerd` | `/etc/docker/daemon.json` | `lab-redis`, `lab-ubuntu`(apt·dpkg 실습) | 6379 | 11 |
| 가상화 | `libvirt` / `libvirtd` | — | 조회 명령 위주 | — | 11 |

### 2-4. 스크립트·백업·스케줄

| 항목 | 값 |
| --- | --- |
| 백업 스크립트 | `/usr/local/bin/backup.sh` — `/data`, `/srv/share`, `/etc` → `/srv/raid/backup/` |
| 진단 스크립트 | `/usr/local/bin/sysreport.sh` — `top -b -n 1`, `vmstat`, `df` 요약 |
| cron | 매일 03:30 백업 (`/etc/cron.d/lab-backup`), 사용자 crontab 은 `dev1` |
| 커스텀 서비스 | `lab-monitor.service` (Part 07) |

---

## 3. 절차서 표기 규칙

- 명령은 전부 코드 블록. 프롬프트 `#` = root, `$` = 일반 사용자
- 각 소단계 = **상황 → 명령 → 옵션 설명 → 검증 → 기대 출력 → 시험 포인트** 순
- 옵션은 `-` `옵션` : 의미 (원어 굵게, 예: **h**uman-readable) 형식
- 기대 출력은 ```text 블록, 환경마다 달라지는 값은 `...` 또는 `<값>` 으로 표기
- `※ 미실행` : 시험 범위지만 UTM 환경 제약으로 실행하지 않는 참고 항목 (예: 중첩 가상화·X 윈도)
- `⚠️` : 파괴적 또는 잠금 위험 명령 — 대상 장치·경로 재확인 후 실행

---

## 4. 진행 순서 (12 파트)

| 파트 | 문서 | 내용 | 주요 명령 |
| --- | --- | --- | --- |
| 01 | [[01-vm-setup-and-inspection]] | VM 준비, 첫 로그인, 시스템 정보·FHS·셸 환경 점검 | `hostnamectl` `uname` `lscpu` `free` `lsblk` `df` `who` `last` `man` `env` `history` |
| 02 | [[02-package-management]] | dnf/rpm 저장소·설치·검증, EPEL, 소스 컴파일, 라이브러리 | `dnf` `rpm` `configure/make` `ldd` `ldconfig` |
| 03 | [[03-user-group-permission]] | 계정·그룹·비밀번호 정책, sudo, 권한·특수권한·ACL·chattr, PAM | `useradd` `usermod` `chage` `visudo` `chmod` `setfacl` `chattr` |
| 04 | [[04-file-text-shell]] | 파일·텍스트 처리, 압축·아카이브, 셸 변수·리다이렉션, 스크립트 작성, vi | `find` `grep` `sed` `awk` `tar` `dd` `sha256sum` `bash` |
| 05 | [[05-disk-lvm-raid-swap-quota]] | 파티션·파일시스템·fstab·LVM·RAID·스왑·쿼터·커널 모듈 | `fdisk` `mkfs` `mount` `pvcreate…` `mdadm` `mkswap` `edquota` `modprobe` |
| 06 | [[06-process-scheduling-diagnosis]] | 프로세스·잡 제어·시그널·우선순위, **top 심화 진단**, vmstat/iostat/sar, cron/at | `ps` `top` `kill` `nice` `jobs` `crontab` `at` `vmstat` `sar` |
| 07 | [[07-boot-systemd-log]] | systemd 유닛·타겟, 커스텀 서비스, 저널·rsyslog·logrotate, GRUB, 레스큐·root 비번 복구 | `systemctl` `journalctl` `logger` `grub2-mkconfig` `grubby` `rd.break` |
| 08 | [[08-network-config]] | IP·라우팅·DNS 클라이언트·고정 IP, 진단 도구, SSH 키·포트 변경·scp/rsync, chrony | `ip` `nmcli` `ss` `ping` `dig` `nc` `tcpdump` `ssh-keygen` `chronyc` |
| 09 | [[09-network-services]] | Apache·BIND·NFS·Samba·vsftpd·Postfix·CUPS 구축과 클라이언트 검증 | `httpd` `named` `exportfs` `testparm` `smbclient` `vsftpd` `mail` `lpadmin` |
| 10 | [[10-security-firewall-selinux]] | firewalld·iptables·SELinux·sshd 강화·침해 점검·gpg/openssl | `firewall-cmd` `iptables` `semanage` `restorecon` `ausearch` `gpg` `openssl` |
| 11 | [[11-container-virtualization]] | Docker 설치·운영, Ubuntu 컨테이너에서 apt/dpkg, libvirt/virsh | `docker` `apt` `dpkg` `virsh` |
| 12 | [[12-backup-recovery-review]] | tar 증분·rsync·dd·dump/restore 백업과 복구 검증, 종료·재부팅, **전 범위 체크리스트** | `tar --listed-incremental` `rsync` `dd` `dump` `restore` `shutdown` |

- 순서 의존: 03(계정) → 05(디스크) → 09(서비스) 순서 필수. 06·07·10 은 05 이후 자유
- 하루 2파트 권장 (총 6일). 파트마다 마지막 **체크리스트** 로 수행 여부 자체 점검

---

## 5. 기출 커버리지 요약

| 실기 주제 (EXAM-PRACTICAL) | 파트 |
| --- | --- |
| find 조건 검색·삭제, SetUID 파일 점검 | 04, 10 |
| useradd/usermod/chage/umask/특수권한 | 03 |
| fdisk·mkfs·mount·fstab·LVM(`lvextend -r`)·RAID | 05 |
| ps/pgrep/ss/kill/nice/cron 필드 | 06 |
| systemctl enable/status, setenforce/getenforce | 07, 10 |
| tar cvzf/증분, rsync, dd MBR 백업, sha256sum | 04, 12 |
| iptables 정책·규칙 해석, firewall-cmd permanent | 10 |
| /etc/exports·root_squash, smb.conf·testparm, vsftpd anonymous | 09 |
| Apache vhost·configtest·.htaccess, DNS SOA/MX/A | 09 |
| /etc/shadow 필드, PAM control, rsyslog 규칙, hosts.allow | 03, 07, 10 |
| ip addr add, ssh -p, nc -zv, awk -F: | 08 |
| 암호화(대칭·비대칭), SYN Flooding·ARP 스푸핑 대응 | 10 |

---

## 관련 문서
- [[../README|LINUX-MASTER 허브]] · [[../../INDEX|명령어 볼트 색인]]
