---
title: 실기 모의고사 Round 04 — 문제
type: exam-practical
round: 4
tags:
  - exam/linux-master
  - exam/practical
  - exam/questions
updated: 2026-08-28
---

# 실기 모의고사 Round 04 — 문제

- 15문항 / 명령어 작성·빈칸·설정 서술형
- 채점·해설: [[answers]] (풀이 완료 후 열람)
- 답안은 명령어·옵션 정확 표기 기준 채점

---

## 1부. 명령어 작성 (직접 서술)

**1.** `/home` 디렉터리 이하에서 크기가 **100MB 를 초과**하는 **일반 파일**을 모두 찾는 `find` 명령을 작성하시오.

**2.** 디스크 `/dev/sda` 의 **첫 512바이트(MBR)** 를 파일 `mbr.bak` 으로 백업하는 `dd` 명령을 작성하시오.

**3.** `/data` 디렉터리와 그 하위 전체의 권한을 **재귀적으로 `750`** 으로 변경하는 명령을 작성하시오.

**4.** 파일 `image.iso` 의 **SHA-256** 해시값을 계산하는 명령을 작성하시오.

**5.** 원격 호스트 `host` 에 사용자 `user1` 로 **2222번 포트**를 사용해 SSH 접속하는 명령을 작성하시오.

---

## 2부. 빈칸 채우기

**6.** 현재 SELinux 의 **적용 모드(Enforcing/Permissive/Disabled)** 를 확인하는 명령:
`______`

**7.** `/etc/fstab` 에 등록된 항목을 **모두 한 번에 마운트**하는 명령:
`mount ______`

**8.** 로그 파일 `/var/log/app.log` 를 **추가만 허용(append-only)** 속성으로 지정하여 변조를 방지하는 명령:
`chattr ______ /var/log/app.log`

**9.** `INPUT` 체인의 **3번째 규칙을 삭제**하는 명령:
`iptables ______ INPUT 3`

**10.** cron 에서 **매월 1일 0시 0분**에 작업을 실행하려는 시간 필드 (분 시 일 월 요일):
`0 0 ______ * *`

---

## 3부. 설정·개념 서술

**11.** `/etc/crontab` 한 줄이 다음과 같을 때 각 시간 필드의 의미를 종합하여 서술하시오.
```
0 */2 * * * root /usr/local/bin/sync.sh
```

**12.** 다음 두 iptables 명령이 **함께** 설정되었을 때의 최종 동작을, 규칙 매칭 순서를 근거로 서술하시오.
```
iptables -P INPUT DROP
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

**13.** `umask` 값이 `027` 일 때 새로 생성되는 **파일**과 **디렉터리**의 권한(8진수)을 각각 계산 과정과 함께 서술하시오.

**14.** NFS 서버의 `/etc/exports` 옵션 중 `root_squash` 와 `no_root_squash` 의 차이를 서술하고, `no_root_squash` 의 보안 위험을 설명하시오.

**15.** PAM 의 제어 구문(control) 4종 `required` · `requisite` · `sufficient` · `optional` 의 동작 차이를 서술하시오.
