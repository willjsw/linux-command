---
title: 실기 모의고사 Round 02 — 정답·해설
type: exam-practical-answer
round: 2
tags:
  - exam/linux-master
  - exam/practical
  - exam/answers
updated: 2026-08-28
---

# 실기 모의고사 Round 02 — 정답·해설

- 문제: [[questions]]
- ⚠ 반드시 풀이를 마친 후 열람

---

## 1부. 명령어 작성

**1.**
```bash
find / -perm -4000 -type f
```
- `-perm -4000` : SetUID 비트 포함(`-` 접두는 해당 비트 포함 매칭), `-type f` 일반 파일 한정 → [[find]] · [[system-security]] 2-1

**2.**
```bash
chown -R apache:apache /var/www/html
```
- `-R` : 재귀, `소유자:그룹` 표기. 그룹만이면 `:apache`, 소유자만이면 `apache`

**3.**
```bash
tar -xzvf backup.tar.gz -C /restore
```
- `x` 추출 / `z` gzip / `v` 상세 / `f` 파일명, `-C` 대상 디렉터리 지정 → [[tar]]

**4.**
```bash
ss -tan state established
# 또는
ss -tnp state established
```
- `-t` TCP · `-a` 전체 · `-n` 숫자표기, `state established` 상태 필터 → [[ss]]

**5.**
```bash
rsync -avz /data user@host:/backup
```
- `-a` 아카이브(권한·소유·심볼릭링크·타임스탬프 보존) · `-v` 상세 · `-z` 전송 압축 → [[system-security]] 3-2

---

## 2부. 빈칸

**6.** `022` — 파일 666−022=644, 디렉터리 777−022=755 → [[system-security]] 2-2

**7.** `-M 90` — `-M` = Maxdays(최대 사용기간). `-m` 최소기간, `-l` 조회 → [[system-security]] 2-7

**8.** `DROP` — `-P` 는 Policy(기본 정책). `DROP` 은 응답 없이 폐기 → [[iptables]] · [[network-security]] 2-1

**9.** `--permanent` — 영구 규칙. 적용은 이후 `firewall-cmd --reload` 필요 → [[firewall-cmd]] · [[network-service]] 0

**10.** `0` — 요일 필드에서 일요일=0 (7 도 일요일로 인정) → [[crontab]]

---

## 3부. 서술

**11.**
- `18900` : 최종 비밀번호 변경일 (1970-01-01 기준 경과 일수)
- `0` : 비밀번호 변경 후 재변경까지 **최소 사용기간**(일)
- `90` : 비밀번호 **최대 사용기간**(일) — 초과 시 변경 강제
- 참고: 이어지는 `7`=만료 전 경고일, 이후 비활성일·만료일 필드는 비어 있음 → [[system-security]] 2-7

**12.**
- 연결 상태 추적(`-m state`)을 사용해, 이미 **확립된(ESTABLISHED)** 연결과 그와 **연관된(RELATED)** 패킷의 수신을 허용하는 규칙을 INPUT 체인 끝에 추가
- 기본 정책이 DROP 인 환경에서 나가는 연결의 응답 패킷을 되받기 위한 필수 규칙 → [[iptables]] · [[network-security]] 2-1

**13.**
- `SetUID` = 8진수 **4000**, 표기 `-rwsr-xr-x`(소유자 x 자리 `s`) — 실행 시 **파일 소유자 권한**으로 동작 (예: `/usr/bin/passwd`)
- `SetGID` = 8진수 **2000**, 표기 `-rwxr-sr-x`(그룹 x 자리 `s`) — 실행 시 **파일 그룹 권한**으로 동작 / 디렉터리에 걸면 하위 생성 파일이 그 디렉터리 그룹을 상속
- 실행 권한(x)이 없으면 소문자 `s` 대신 대문자 `S` 표기 → [[system-security]] 2-1

**14.**
- `mail.info` : mail facility 의 info **이상(포함) 우선순위 전부** 기록 (info·notice·warning·err … )
- `mail.=info` : `=` 는 정확히 일치 — mail 의 **info 레벨만** 기록, 상위 레벨 제외
- 참고: `mail.!err` 는 err 이상 제외, `*.none` 은 해당 facility 완전 제외 → [[system-security]] 1-2

**15.**
- RAID 1(미러링) : 동일 데이터를 2개 이상 디스크에 복제. **최소 2개**, 가용 용량 = **D**(전체 용량의 절반), 1개 고장 감내
- RAID 5(패리티 분산) : 데이터+패리티를 분산 저장. **최소 3개**, 가용 용량 = **(n−1)×D**, 디스크 1개 고장 감내
- RAID 1 은 용량 손실이 크지만 단순·고신뢰, RAID 5 는 용량 효율이 높고 1개 장애까지 복구 → [[network-service]] 7-2

---

## 채점 기준
- 1부: 명령·옵션 정확 일치 (동등 대체 명령 인정)
- 2부: 정확한 값·키워드
- 3부: 핵심 개념 포함 시 정답 (표현 차이 무관)
