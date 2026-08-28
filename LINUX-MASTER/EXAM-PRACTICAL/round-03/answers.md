---
title: 실기 모의고사 Round 03 — 정답·해설
type: exam-practical-answer
round: 3
tags:
  - exam/linux-master
  - exam/practical
  - exam/answers
updated: 2026-08-28
---

# 실기 모의고사 Round 03 — 정답·해설

- 문제: [[questions]]
- ⚠ 반드시 풀이를 마친 후 열람

---

## 1부. 명령어 작성

**1.**
```bash
grep -c "Failed password" /var/log/secure
```
- `-c` : 매칭 행 **개수** 출력. 단순 검색은 옵션 없이, 대소문자 무시는 `-i` → [[grep]]

**2.**
```bash
awk -F: '$3 >= 1000 {print $1}' /etc/passwd
```
- `-F:` 필드 구분자 콜론, `$3` UID · `$1` 계정명. 일반 사용자 UID 는 보통 1000 이상 → [[awk]]

**3.**
```bash
lvextend -r -L +5G /dev/vg0/lv0
```
- `-L +5G` : 현재 크기에서 5GB 증설(`+` 없이 `-L 5G` 는 절대 크기 지정), `-r` : 파일시스템 동시 확장(`resize2fs`/`xfs_growfs` 자동 호출) → [[lvm]]

**4.**
```bash
systemctl status firewalld
```
- `status` 는 활성 여부·최근 로그 표시. 부팅 자동시작 여부는 `is-enabled` → [[systemctl]]

**5.**
```bash
ip addr add 192.168.0.10/24 dev eth0
```
- `addr add ... dev` 형식. 삭제는 `ip addr del`, 조회는 `ip addr show` → [[ip]]

---

## 2부. 빈칸

**6.** `4755` — 앞자리 `4`=SetUID, 뒤 `755`=기존 rwx r-x r-x. 심볼릭은 `u+s` → [[system-security]] 2-1

**7.** `-i` — in-place(원본 직접 수정). `g` 플래그는 행 내 전체 치환 → [[sed]]

**8.** `-ulnp` — `-u` UDP · `-l` LISTEN · `-n` 숫자 · `-p` 프로세스 → [[ss]]

**9.** `*/10` — 0,10,20,…,50 분마다 실행. `10` 단독은 매시 10분에 1회뿐 → [[crontab]]

**10.** `-g` — listed-incremental(스냅샷 기반 증분). 최초 실행 시 전체, 이후 변경분만 → [[tar]] · [[system-security]] 3-2

---

## 3부. 서술

**11.**
- `*/5` : 분 — 5분마다
- `9-18` : 시 — 9시부터 18시까지
- `*` : 일 — 매일
- `*` : 월 — 매월
- `1-5` : 요일 — 월요일~금요일
- 종합: **평일(월~금) 09시~18시 사이 5분마다 root 권한으로 check.sh 실행** → [[crontab]]

**12.**
- `nat` 테이블 `PREROUTING` 체인에서, 목적지 TCP 80번 포트로 들어온 패킷의 **목적지 주소를 `192.168.0.10:8080` 으로 변환**(DNAT)하는 규칙 = **포트 포워딩**
- 외부 80 요청을 내부 서버 8080 으로 전달하는 전형적 구성 → [[iptables]] · [[network-security]] 2-2

**13.**
- `/dev/sdb1` : 장치명(마운트 대상)
- `/data` : 마운트 지점
- `ext4` : 파일시스템 종류
- `defaults` : 마운트 옵션(rw, suid, exec, auto 등 기본값)
- `0` : dump 백업 대상 여부(0=제외)
- `2` : fsck 부팅 검사 순서(root=1, 그 외=2, 검사 안 함=0) → [[mount]]

**14.**
- 검사 순서: `/etc/hosts.allow` → `/etc/hosts.deny` 순
- 우선 적용: **allow 가 우선** — allow 에서 먼저 매칭되면 허용하고 deny 는 보지 않음
- 두 파일 모두 매칭 규칙이 없으면 **기본 허용** → [[system-security]] 2-6

**15.**
- 대칭키 : 암호화·복호화에 **동일한 키** 사용, 연산이 **빠름**, 통신 상대에게 키를 안전히 전달해야 하는 **키 분배 문제**가 있음 (예: AES, DES, SEED)
- 비대칭키 : **공개키/개인키 쌍** 사용(공개키로 암호→개인키로 복호), 상대적으로 **느림**, 공개키를 공개해도 되어 **키 분배 문제 해결** (예: RSA, ECC)
- 실무는 키 교환은 비대칭, 대량 데이터 암호는 대칭으로 결합 → [[network-security]] 4-1

---

## 채점 기준
- 1부: 명령·옵션 정확 일치 (동등 대체 명령 인정)
- 2부: 정확한 값·키워드
- 3부: 핵심 개념 포함 시 정답 (표현 차이 무관)
