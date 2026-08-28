---
title: 실기 모의고사 Round 05 — 정답·해설
type: exam-practical-answer
round: 5
tags:
  - exam/linux-master
  - exam/practical
  - exam/answers
updated: 2026-08-28
---

# 실기 모의고사 Round 05 — 정답·해설

- 문제: [[questions]]
- ⚠ 반드시 풀이를 마친 후 열람

---

## 1부. 명령어 작성

**1.**
```bash
grep -c "Failed password" /var/log/secure
```
- `-c` : 매칭 행 **개수**만 출력 (count). SSH 인증 실패는 `/var/log/secure` 에 기록 → [[grep]] · [[system-security]] 1-1

**2.**
```bash
usermod -L user2
```
- `-L` : 계정 잠금(**L**ock) — `/etc/shadow` 비밀번호 필드 앞에 `!` 부가. 해제는 `-U`. 셸 자체 차단은 `usermod -s /sbin/nologin` → [[usermod]] · [[system-security]] 2-7

**3.**
```bash
find / -perm -4000 -type f
```
- `-perm -4000` : SetUID 비트가 켜진 파일(4000 이상 포함). `-type f` 로 일반 파일 한정. 침해 점검 단골 → [[find]] · [[system-security]] 2-1

**4.**
```bash
firewall-cmd --add-service=http --permanent
firewall-cmd --reload
```
- `--permanent` 는 영구 설정만 바꾸고 런타임에 미반영 → `--reload` 로 적용해야 함. 즉시+영구 동시 적용은 두 번 실행 또는 `--runtime-to-permanent` → [[firewall-cmd]] · [[network-security]] 2-3

**5.**
```bash
ip addr add 192.168.10.5/24 dev eth0
```
- `ip addr add <IP/prefix> dev <장치>`. 구형 `ifconfig eth0 192.168.10.5/24` 도 동작하나 `ip` 가 표준 → [[ip]]

---

## 2부. 빈칸

**6.** `022` — 파일 666−022=644, 디렉터리 777−022=755 → [[system-security]] 2-2

**7.** `-M 90` — `-M` = 최대 사용기간(**M**axdays). 최소기간은 `-m`, 만료 조회는 `chage -l` → [[system-security]] 2-7

**8.** `-ra` — `-r` 재적용(**r**e-export), `-a` 전체(**a**ll). 조회는 `exportfs -v` → [[network-service]] 4-1 · [[mount]]

**9.** `/etc/ssh/sshd_config` — 클라이언트 설정 `/etc/ssh/ssh_config` 와 혼동 주의 → [[ssh]] · [[network-service]] 0

**10.** `+i` — 불변(**i**mmutable) 속성. 조회는 `lsattr`, 해제는 `chattr -i`. 추가만 허용은 `+a` → [[system-security]] 2-3

---

## 3부. 서술

**11.**
- `18800` : 비밀번호 **최종 변경일** (1970-01-01 기준 경과 일수)
- `7` : 비밀번호 변경 후 재변경까지의 **최소 사용기간(일)**
- `90` : 비밀번호 **최대 사용기간(일)** — 초과 시 변경 요구
- `14` : 만료 전 **경고 시작일(일)** — 만료 14일 전부터 경고
- `/etc/shadow` 필드 순서: 계정:암호:최종변경:최소:최대:경고:비활성:만료:예약 → [[system-security]] 2-7

**12.**
- `authpriv.*  /var/log/secure` : authpriv 퍼실리티의 **모든 우선순위** 메시지(ssh·su·sudo·login 등 인증·보안)를 `/var/log/secure` 에 기록
- `*.info;mail.none;authpriv.none  /var/log/messages` : 전체 퍼실리티의 **info 이상** 메시지를 `/var/log/messages` 에 기록하되, `mail` 과 `authpriv` 는 **제외**(`.none`) → [[system-security]] 1-2

**13.**
- `rw` : 클라이언트에 **읽기·쓰기** 허용 (읽기전용은 `ro`)
- `sync` : 디스크에 실제 기록 완료 후 응답하는 **동기** 방식 (안전, `async` 는 빠르나 유실 위험)
- `no_root_squash` : 클라이언트의 **root 를 그대로 root 로 유지** (기본 `root_squash` 는 nobody 로 매핑)
- 위험: 클라이언트 측 root 가 공유 디렉터리에서 서버 root 권한으로 파일을 생성·수정할 수 있어, 공유를 탈취당하면 서버 파일시스템까지 위협 → [[network-service]] 4-1 · [[mount]]

**14.**
- 원리: TCP 3-way 핸드셰이크에서 `SYN` 만 대량 전송하고 `ACK` 를 보내지 않아, 서버의 **백로그(연결 대기) 큐**를 half-open 연결로 고갈시켜 정상 접속을 막는 DoS 공격
- 방어: ① **SYN 쿠키** 활성화, ② 백로그 큐 크기 증대, ③ 방화벽에서 SYN rate limit / 이상 트래픽 탐지 (2가지 이상) → [[network-security]] 1-2

**15.**
- 대칭키 : 암호화·복호화에 **같은 키** 사용 → 연산이 **빠름**, 그러나 키를 안전하게 전달하는 **키 분배 문제** 존재 (DES, AES, SEED)
- 비대칭키 : **공개키/개인키 쌍** 사용(한쪽으로 암호, 다른 쪽으로 복호) → 키 분배 문제 해결, 그러나 연산이 **느림** (RSA, ECC)
- 실무는 키 교환은 비대칭키, 대량 데이터 암호는 대칭키로 결합(하이브리드) → [[network-security]] 4-1

---

## 채점 기준
- 1부: 명령·옵션 정확 일치 (동등 대체 명령 인정)
- 2부: 정확한 값·키워드
- 3부: 핵심 개념 포함 시 정답 (표현 차이 무관)
