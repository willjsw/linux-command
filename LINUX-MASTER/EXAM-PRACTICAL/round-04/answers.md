---
title: 실기 모의고사 Round 04 — 정답·해설
type: exam-practical-answer
round: 4
tags:
  - exam/linux-master
  - exam/practical
  - exam/answers
updated: 2026-08-28
---

# 실기 모의고사 Round 04 — 정답·해설

- 문제: [[questions]]
- ⚠ 반드시 풀이를 마친 후 열람

---

## 1부. 명령어 작성

**1.**
```bash
find /home -size +100M -type f
```
- `-size +100M` : 100MB 초과(`+`), `-type f` 일반 파일. `100M`=정확히, `-100M`=미만 → [[find]]

**2.**
```bash
dd if=/dev/sda of=mbr.bak bs=512 count=1
```
- `bs=512 count=1` : 512바이트 블록 1개(=MBR) 복사. `if` 입력·`of` 출력 → [[dd]] ⚠ if/of 반대 지정 시 디스크 파괴

**3.**
```bash
chmod -R 750 /data
```
- `-R` 재귀. `750` = 소유자 rwx(7)·그룹 r-x(5)·기타 없음(0)

**4.**
```bash
sha256sum image.iso
```
- 무결성 검증은 `sha256sum -c image.iso.sha256` (기록된 해시와 대조) → [[sha256sum]] · [[network-security]] 4-1

**5.**
```bash
ssh -p 2222 user1@host
```
- `-p` 는 소문자(대문자 `-P` 는 scp/sftp 의 포트 옵션과 혼동 주의) → [[ssh]]

---

## 2부. 빈칸

**6.** `getenforce` — 현재 모드 문자열 출력. 변경은 `setenforce 0|1`, 영구 설정은 `/etc/selinux/config` → [[getenforce]] · [[system-security]] 2-8

**7.** `-a` — `/etc/fstab` 의 auto 대상 전체 마운트(**a**ll). 반대는 `umount -a` → [[mount]]

**8.** `+a` — append-only. 기존 내용 수정·삭제 불가, 추가만 허용(로그 변조 방지). 불변은 `+i` → [[system-security]] 2-3

**9.** `-D` — Delete. 번호로 지정 삭제. 전체 초기화는 `-F`(Flush) → [[iptables]] · [[network-security]] 2-1

**10.** `1` — 일 필드=1(매월 1일). 요일 `*` 이므로 요일 무관 → [[crontab]]

---

## 3부. 서술

**11.**
- `0` : 분 — 0분(정각)
- `*/2` : 시 — 2시간마다(0,2,4,…,22시)
- `*` : 일 — 매일
- `*` : 월 — 매월
- `*` : 요일 — 모든 요일
- 종합: **매일 2시간 간격(짝수 시 정각)마다 root 권한으로 sync.sh 실행** → [[crontab]]

**12.**
- 규칙은 위에서 아래로 순차 매칭되며, 기본 정책은 DROP
- ① 확립·연관 연결 응답 패킷 허용 → ② TCP 22(SSH) 신규 수신 허용 → 두 규칙에 매칭되지 않는 나머지 모든 수신 패킷은 기본 정책 DROP 으로 폐기
- 결과: **SSH 접속과 기존 연결의 응답만 허용, 그 외 인바운드는 전부 차단** → [[iptables]] · [[network-security]] 2-1

**13.**
- 파일: 기본 666 − 027 = **640** (rw- r-- ---)
- 디렉터리: 기본 777 − 027 = **750** (rwx r-x ---)
- umask 는 기본 권한에서 차감하는 마스크이며, 파일은 실행 비트를 기본 부여하지 않아 666 기준으로 계산 → [[system-security]] 2-2

**14.**
- `root_squash`(기본) : 클라이언트의 root 접근을 서버의 `nobody`(익명) 사용자로 매핑 — 원격 root 가 서버 파일을 root 권한으로 다루지 못하게 함
- `no_root_squash` : 클라이언트 root 를 서버 root 로 그대로 인정
- 위험: 클라이언트에서 root 를 얻은 공격자가 공유 디렉터리의 모든 파일을 root 권한으로 읽기·수정·소유권 변경할 수 있어 서버 침해로 이어짐 → [[network-service]] 4-1

**15.**
- `required` : 실패해도 즉시 중단하지 않고 나머지 모듈까지 검사한 뒤 **최종적으로 거부**(실패 사실 은닉)
- `requisite` : 실패 시 **즉시 거부**하고 이후 모듈 검사 중단
- `sufficient` : **성공하면 즉시 인증 허용**(앞선 required 실패가 없다면), 실패해도 다음으로 진행
- `optional` : 성공·실패가 최종 결과에 영향 없음(참고용, 다른 모듈이 없을 때만 결정적) → [[system-security]] 2-4

---

## 채점 기준
- 1부: 명령·옵션 정확 일치 (동등 대체 명령 인정)
- 2부: 정확한 값·키워드
- 3부: 핵심 개념 포함 시 정답 (표현 차이 무관)
