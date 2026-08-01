---
command: crontab
category: SERVICE-SYSTEMD
aliases: [cron, cron.d]
tags:
  - linux/systemd
  - linux/service
  - task/inspect
  - task/diagnose
  - topic/security
  - topic/troubleshooting
  - privilege/mixed
related: ["[[systemctl]]", "[[journalctl]]", "[[ls]]", "[[redis-cli]]"]
distro: 전체 (cronie / cron)
verified: Rocky Linux 9.6 (사고 대응 세션)
updated: 2026-08-01
---

# crontab

- 사용자별 예약 작업(cron) 조회·편집 도구
- 어원: **cron** **tab**le (chronos, 시간 기반 작업표)
- 침해 시 지속성 등록 흔적 조사 대상 → 등록 작업·cron 디렉터리 전수 확인

---

## crontab -l

```bash
crontab -l
crontab -l -u <사용자>

# Examples
crontab -l                            # 현재 사용자 등록 작업
sudo crontab -l                       # root 등록 작업
crontab -l -u postgres 2>/dev/null    # 특정 사용자 (컨테이너 내 조사)
```

### 명령어 설명
- 사용 목적
	- 사용자·root cron 등록 작업 존재 여부 확인 시 사용
	- 침해 지속성(cron 페이로드) 흔적 조사 시 사용
- 특이사항
	- `no crontab for <user>` → 등록 작업 부재 (정상)
	- 등록 부재여도 cron 데몬만 기동된 사례 존재 → 프로세스 목록(`ps`) 병행 확인
	- Redis 등 경유 cron 페이로드는 crontab이 아닌 데이터 파일에 주입될 수 있음 → [[redis-cli]]

### 옵션
- `-l` : 등록 작업 목록 출력 (**l**ist)
- `-u` : 대상 사용자 지정 (**u**ser, root 권한 필요)

---

## cron 디렉터리·타이머 전수 점검

```bash
# Examples
ls -la /etc/cron.d /etc/cron.*/          # 시스템 cron 디렉터리 변조 여부
systemctl list-timers --all              # systemd 타이머 (악성 타이머 확인)
```

### 명령어 설명
- 사용 목적
	- `crontab` 외 경로(시스템 cron·systemd 타이머)의 지속성 등록 조사 시 사용
- 특이사항
	- OS 기본 파일(`0hourly`, `0anacron`)의 mtime이 배포 시점 유지 → 변조 없음 판정
	- systemd 타이머는 `list-timers --all`로 확인 → OS 기본 3종 외 존재 시 조사 → [[systemctl]]
	- 컨테이너 지속성은 systemd user unit·XDG autostart로도 등록 가능 (세션 부재 시 미작동)

---

## 연관 명령어
- [[systemctl]] : systemd 타이머(`list-timers`) 조회
- [[journalctl]] / `/var/log/cron` : cron 실행 로그 확인
- [[ls]] : cron 디렉터리 변조 여부 확인
- [[redis-cli]] : 노출 Redis 경유 cron 페이로드 조사
