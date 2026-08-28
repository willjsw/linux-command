---
title: 실기 모의고사 Round 05 — 문제
type: exam-practical
round: 5
tags:
  - exam/linux-master
  - exam/practical
  - exam/questions
updated: 2026-08-28
---

# 실기 모의고사 Round 05 — 문제

- 15문항 / 명령어 작성·빈칸·설정 서술형
- 채점·해설: [[answers]] (풀이 완료 후 열람)
- 답안은 명령어·옵션 정확 표기 기준 채점

---

## 1부. 명령어 작성 (직접 서술)

**1.** `/var/log/secure` 파일에서 `Failed password` 문자열이 포함된 행의 **개수**를 출력하는 명령을 작성하시오.

**2.** 사용자 `user2` 의 계정을 **잠가**(로그인 차단) 비밀번호 인증을 막는 `usermod` 명령을 작성하시오.

**3.** 루트(`/`) 이하 전체에서 **SetUID** 가 설정된 일반 파일을 모두 검색하는 `find` 명령을 작성하시오.

**4.** firewalld 에서 `http` 서비스를 **영구(permanent)** 로 허용하고 규칙을 즉시 반영하는 명령 2개를 순서대로 작성하시오.

**5.** 인터페이스 `eth0` 에 IP 주소 `192.168.10.5/24` 를 추가하는 `ip` 명령을 작성하시오.

---

## 2부. 빈칸 채우기

**6.** 새로 만드는 파일이 `644`, 디렉터리가 `755` 권한을 갖게 하려면 umask 값은:
`umask ______`

**7.** 사용자 `user1` 의 비밀번호 **최대 사용기간을 90일**로 설정하는 명령:
`chage ______ user1`

**8.** NFS 서버에서 `/etc/exports` 수정 내용을 다시 읽어 전체 재적용하는 명령:
`exportfs ______`

**9.** OpenSSH 서버 데몬(`sshd`)의 주 설정 파일 경로:
`______`

**10.** `/etc/passwd` 파일에 **불변(immutable) 속성**을 부여해 root 도 수정·삭제하지 못하게 하는 명령:
`chattr ______ /etc/passwd`

---

## 3부. 설정·개념 서술

**11.** `/etc/shadow` 의 한 줄이 다음과 같을 때, 표시된 필드 중 `18800`, `7`, `90`, `14` 각각의 의미를 서술하시오.
```
user1:$6$abcd...:18800:7:90:14:::
```

**12.** 다음 `rsyslog` 규칙 두 줄이 각각 어떤 로그를 어디에 기록하는지 서술하시오.
```
authpriv.*                       /var/log/secure
*.info;mail.none;authpriv.none   /var/log/messages
```

**13.** `/etc/exports` 의 다음 한 줄에서 각 옵션(`rw`, `sync`, `no_root_squash`)의 의미를 서술하고, `no_root_squash` 의 보안상 위험을 설명하시오.
```
/share  192.168.0.0/24(rw,sync,no_root_squash)
```

**14.** `SYN Flooding` 공격의 동작 원리와 대표적인 방어 기법 2가지를 서술하시오.

**15.** 대칭키 암호화와 비대칭키(공개키) 암호화 방식의 차이를 키 사용·속도·키 분배 관점에서 서술하시오.
