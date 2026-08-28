---
title: 실기 모의고사 Round 01 — 정답·해설
type: exam-practical-answer
round: 1
tags:
  - exam/linux-master
  - exam/practical
  - exam/answers
updated: 2026-08-28
---

# 실기 모의고사 Round 01 — 정답·해설

- 문제: [[questions]]
- ⚠ 반드시 풀이를 마친 후 열람

---

## 1부. 명령어 작성

**1.**
```bash
find . -name "*.log" -mtime +7 -delete
```
- `-mtime +7` : 7일 초과 경과, `-delete` 대신 `-exec rm {} \;` 도 정답 → [[find]]

**2.**
```bash
usermod -aG wheel user1
```
- `-aG` : 기존 그룹 유지(**a**ppend)하며 추가(**G**roups). `-G` 단독은 기존 그룹 소속 상실 ⚠ → [[usermod]]

**3.**
```bash
mkfs.ext4 /dev/sdb1
mount /dev/sdb1 /data
```
- 영구 마운트는 `/etc/fstab` 등록 필요 → [[mount]]

**4.**
```bash
pgrep -f nginx
# 또는
ps -ef | grep [n]ginx
```
- 대괄호 트릭으로 grep 자기 자신 제외 → [[pgrep]] · [[ps]]

**5.**
```bash
ss -tlnp | grep :8080
# 또는
ss -tlnp sport = :8080
```
- `-tlnp` : TCP·LISTEN·숫자표기·프로세스 → [[ss]]

---

## 2부. 빈칸

**6.** `755` — 소유자 rwx(7), 그룹·기타 r-x(5)

**7.** `setenforce 0` — 0=Permissive, 1=Enforcing (재부팅 시 초기화) → [[getenforce]]

**8.** `enable` — `enable --now` 는 즉시 시작 겸용 → [[systemctl]]

**9.** `#!/bin/bash` — 셔뱅(shebang)

**10.** `30 3` — 분(30) 시(3) 순서 → [[crontab]]

---

## 3부. 서술

**11.**
- `0` : 분 (0분)
- `2` : 시 (2시)
- `*` : 일 (매일)
- `*` : 월 (매월)
- `1` : 요일 (월요일, 0·7=일요일)
- `root` : 실행 사용자 (`/etc/crontab` 에만 존재하는 필드)
- `/usr/local/bin/backup.sh` : 실행 명령
- 종합: **매주 월요일 새벽 2시 정각에 root 권한으로 backup.sh 실행** → [[crontab]]

**12.**
- 출발지 `192.168.0.0/24` 대역에서 오는 TCP 22번(SSH) 포트 수신 패킷을 허용(ACCEPT)하는 규칙을 INPUT 체인 끝에 추가 → [[iptables]] · [[network-security]]

**13.**
```bash
tar -czvf home.tar.gz /home
```
- `c` : 생성 (**c**reate)
- `z` : gzip 압축
- `v` : 진행 상세 출력 (**v**erbose)
- `f` : 파일명 지정 (**f**ile) — 반드시 파일명 바로 앞 → [[tar]]

**14.**
- `A` 레코드 : 호스트명 → IPv4 주소 매핑 (`www IN A 192.168.0.20`)
- `MX` 레코드 : 도메인의 메일 서버 지정, 앞의 숫자는 우선순위(낮을수록 우선) (`IN MX 10 mail.example.com.`) → [[network-service]] 2-2

**15.**
- `t` = Sticky Bit — 디렉터리 내 파일은 **소유자(또는 root)만 삭제·이름변경** 가능
- 목적: `/tmp` 처럼 모두 쓰기 가능한 공용 디렉터리에서 타인 파일 삭제 방지 → [[system-security]] 2-1

---

## 채점 기준
- 1부: 명령·옵션 정확 일치 (동등 대체 명령 인정)
- 2부: 정확한 값·키워드
- 3부: 핵심 개념 포함 시 정답 (표현 차이 무관)
