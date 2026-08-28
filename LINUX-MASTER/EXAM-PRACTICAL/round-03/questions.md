---
title: 실기 모의고사 Round 03 — 문제
type: exam-practical
round: 3
tags:
  - exam/linux-master
  - exam/practical
  - exam/questions
updated: 2026-08-28
---

# 실기 모의고사 Round 03 — 문제

- 15문항 / 명령어 작성·빈칸·설정 서술형
- 채점·해설: [[answers]] (풀이 완료 후 열람)
- 답안은 명령어·옵션 정확 표기 기준 채점

---

## 1부. 명령어 작성 (직접 서술)

**1.** `/var/log/secure` 파일에서 `Failed password` 문자열이 포함된 **행의 개수**만 출력하는 `grep` 명령을 작성하시오.

**2.** `/etc/passwd` 에서 UID(3번째 필드)가 **1000 이상**인 계정의 **이름(1번째 필드)** 만 출력하는 `awk` 명령을 작성하시오. (필드 구분자는 `:`)

**3.** 논리 볼륨 `/dev/vg0/lv0` 의 크기를 **5GB 증설**하면서 파일시스템까지 함께 확장하는 명령 한 줄을 작성하시오. (`-r` 옵션 활용)

**4.** `firewalld` 서비스의 **현재 실행 상태**를 확인하는 systemd 명령을 작성하시오.

**5.** 인터페이스 `eth0` 에 IP 주소 `192.168.0.10/24` 를 추가하는 `ip` 명령을 작성하시오.

---

## 2부. 빈칸 채우기

**6.** 기존 권한이 `755` 인 실행 파일에 **SetUID** 를 추가하여 권한을 한 번에 지정하는 8진수 값:
`chmod ______ /usr/local/bin/tool`

**7.** 파일 `file.txt` 안의 모든 `apple` 을 `banana` 로 치환하고 **원본 파일에 바로 저장**하는 명령:
`sed ______ 's/apple/banana/g' file.txt`

**8.** 53번 포트를 **LISTEN** 중인 **UDP** 소켓을 프로세스와 함께 조회하려는 `ss` 옵션:
`ss ______ | grep :53`

**9.** cron 에서 **10분마다** 작업을 실행하려는 분 필드:
`______ * * * *`

**10.** `tar` 증분 백업에서 스냅샷(변경 목록) 파일을 지정하는 옵션:
`tar ______ snap.snar -czf backup.tar.gz /home`

---

## 3부. 설정·개념 서술

**11.** `/etc/crontab` 한 줄이 다음과 같을 때 각 시간 필드의 의미를 종합하여 서술하시오.
```
*/5 9-18 * * 1-5 root /usr/local/bin/check.sh
```

**12.** 다음 iptables 명령이 수행하는 동작을 서술하시오.
```
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to 192.168.0.10:8080
```

**13.** `/etc/fstab` 한 줄이 다음과 같을 때 6개 필드 각각의 의미를 서술하시오.
```
/dev/sdb1   /data   ext4   defaults   0   2
```

**14.** TCP Wrapper 에서 `/etc/hosts.allow` 와 `/etc/hosts.deny` 의 **검사 순서**와 **우선 적용 규칙**, 두 파일 모두에 규칙이 없을 때의 동작을 서술하시오.

**15.** 대칭키 암호와 비대칭키(공개키) 암호의 차이를 **키 사용 방식·속도·키 분배 관점**에서 서술하고, 각 방식의 대표 알고리즘을 하나씩 제시하시오.
