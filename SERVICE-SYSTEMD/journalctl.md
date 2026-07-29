---
command: journalctl
category: SERVICE-SYSTEMD
aliases: []
tags:
  - linux/systemd
  - linux/log
  - task/diagnose
  - topic/troubleshooting
  - privilege/mixed
related: ["[[systemctl]]", "[[nmcli]]", "[[ausearch]]", "[[dmesg]]", "[[grep]]", "[[reboot]]", "[[tail]]", "[[ps]]", "[[lsof]]"]
distro: systemd 사용 배포판
verified: Rocky Linux 9.6
updated: 2026-07-30
---

# journalctl

- systemd 통합 로그(journal) 조회 도구
- 어원: **journal** **ctl**(control)
- root 권한 시 전체 로그, 일반 사용자는 자신의 로그만 조회

---

## journalctl -u (서비스별)

```bash
journalctl -u <service>

# Examples
journalctl -u sshd -n 50 --no-pager             # 최근 50줄
journalctl -u NetworkManager -n 50 --no-pager | grep -i dhcp   # DHCP 실패 원인
journalctl -u sshd -f                           # 실시간 추적
journalctl -u sshd --since "10 min ago"         # 시간 범위 지정
```

### 명령어 설명
- 사용 목적
	- 특정 서비스 로그 조회로 기동 실패·오류 원인 파악 시 사용
	- **DHCP 주소 획득 실패 원인 확인** 시 사용 → `DHCPv4 lease timeout` 등
	- 서비스 이상 동작 진단 시 사용
- 특이사항
	- 페이저 자동 실행 → `--no-pager` 로 즉시 전체 출력
	- 커널 부팅 메시지는 [[dmesg]] 또는 `-k` 옵션 사용
	- SELinux 차단 로그는 [[ausearch]] 사용

### 옵션
- `-u <unit>` : 특정 유닛(서비스) 로그 (**u**nit)
- `-n <n>` : 최근 n줄 (**n**umber)
- `-f` : 실시간 추적 (**f**ollow)
- `-b` : 현재 부팅 세션만 (**b**oot), `-b -1` = 이전 부팅
- `-p <level>` : 우선순위 필터 (**p**riority), `err`·`warning` 등
- `-k` : 커널 메시지만 (**k**ernel)
- `--since` / `--until` : 시간 범위
- `--no-pager` : 페이저 미사용
- `-xe` : 설명 포함 + 마지막으로 이동 (**x**plain + **e**nd)

---

## journalctl -xe (최근 오류 확인)

```bash
journalctl -xe

# Examples
journalctl -xe                                   # 최근 로그 + 설명
journalctl -xe NM_DEVICE=eno1                    # 특정 장치 관련 로그
```

### 명령어 설명
- 사용 목적
	- 직전 발생 오류 원인 즉시 확인 시 사용
	- [[nmcli]] 등이 오류 메시지에서 안내하는 후속 진단 명령으로 사용

---

## 연관 명령어
- [[systemctl]] : 서비스 상태 확인·제어
- [[nmcli]] : 네트워크 설정, 실패 시 로그 확인 대상
- [[ausearch]] : SELinux 차단(AVC) 로그 전용 조회
- [[dmesg]] : 커널 링 버퍼 메시지
- [[grep]] : 로그 출력 필터링
- [[reboot]] : `-b -1` 로 이전 부팅 로그 조회
- [[tail]] : 파일 기반 애플리케이션 로그 조회 — systemd 외부 로그
- [[ps]] : 프로세스 기동 여부 확인
- [[lsof]] : 포트 점유·리스닝 확인
