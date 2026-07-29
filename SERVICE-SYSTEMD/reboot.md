---
command: reboot
category: SERVICE-SYSTEMD
aliases: [shutdown, poweroff, halt]
tags:
  - linux/systemd
  - task/restart
  - privilege/root
  - linux/service
  - task/verify
  - topic/boot-target
related: ["[[systemctl]]", "[[dnf]]", "[[journalctl]]"]
distro: 전체
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# reboot

- 시스템 재시작 명령
- systemd 환경에서는 `systemctl reboot` 의 별칭
- root 권한 필요

---

## reboot

```bash
reboot

# Examples
reboot                       # 즉시 재시작
systemctl reboot             # 동일 동작
shutdown -r now              # 즉시 재시작 (구형 표기)
shutdown -r +10 "메시지"      # 10분 후 재시작 예고
```

### 명령어 설명
- 사용 목적
	- 커널 업데이트 후 신규 커널 적용 시 사용
	- 부팅 타겟(GUI/콘솔) 변경 반영 시 사용
	- **네트워크 `autoconnect` 설정 검증 시** 사용 → 재부팅 후 IP 유지 여부가 실제 검증 지점
- 특이사항
	- 레스큐·라이브 환경에서 실행 시 설치 매체(USB) 읽기 오류로 중단 가능
		- `SQUASHFS error: Failed to read block ... -5` 무한 반복 → USB 읽기 실패
		- 대응: 전원 버튼 5초 이상 강제 종료 → USB 제거 → 재부팅
	- 레스큐 셸 사용 후 첫 부팅은 SELinux 재레이블로 수분 소요, 완료 후 자동 재부팅 1회 추가

---

## shutdown / poweroff

```bash
poweroff                     # 전원 종료
shutdown -h now              # 즉시 종료 (halt)
shutdown -c                  # 예약된 종료 취소 (cancel)
```

### 옵션
- `-r` : 재시작 (**r**eboot)
- `-h` : 종료 (**h**alt)
- `-c` : 예약 취소 (**c**ancel)
- `now` / `+<분>` : 실행 시점

---

## 재부팅 후 검증 항목

```bash
ip -br addr                  # IP 유지 확인 (autoconnect 검증)
ip route                     # 기본 게이트웨이 유지 확인
systemctl is-active sshd     # 서비스 자동 기동 확인
df -h                        # 파티션 정상 마운트 확인
```

---

## 연관 명령어
- [[systemctl]] : 부팅 타겟 변경 후 재시작 필요
- [[dnf]] : 커널 업데이트 후 재시작 필요
- [[journalctl]] : `-b -1` 로 이전 부팅 로그 확인
