---
command: tail
category: FILE-TEXT
aliases: []
tags:
  - linux/text
  - linux/log
  - task/inspect
  - task/diagnose
  - topic/troubleshooting
  - privilege/mixed
related: ["[[head]]", "[[journalctl]]", "[[grep]]", "[[sed]]", "[[nohup]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# tail

- 파일·표준입력 말미 일부 출력 도구
- 어원: **tail** (꼬리 부분)
- 조회는 일반 사용자 가능, 시스템 로그(`/var/log/*`) 접근 시 root 필요

---

## tail

```bash
tail [-n <행수>] <file>

# Examples
tail -5 /tmp/bootrun.log 2>/dev/null                # 최근 5행 (기동 결과 확인)
tail -8 /tmp/bootrun.log                            # 최근 8행
tail -20 db/changelog/db.changelog-master.yaml      # 파일 말미 구조 확인
tail -40 /tmp/boot_verify3.log | grep -vE "^\s+at |jar:na" | head -30   # 스택트레이스 제거 후 조회
timeout 5 ping -c 2 10.1.17.62 2>&1 | tail -3       # 명령 출력 요약부만
du -sh */ | sort -h | tail -30                      # 정렬 후 상위 항목 추출
```

### 명령어 설명
- 사용 목적
	- 로그 파일 최신 메시지 확인 시 사용 (오류 원인 진단)
	- 백그라운드 기동 프로세스 상태 확인 시 사용 ([[nohup]] 연계)
	- `sort` 결과 상위·하위 항목 추출 시 사용
	- 명령 출력의 요약·결과부만 확인 시 사용
- 특이사항
	- 기본값은 10행
	- **`-f` 는 파일 추가 기록을 실시간 추적 → 종료 대기 상태 진입**
		- 자동화·스크립트 내 사용 시 무한 대기 발생 → `timeout` 병용 또는 미사용 권장
	- **`sort -h | tail` 조합은 `sort -hr | head` 보다 직관 낮음** → 목적에 따라 선택
	- 회전(rotate)된 로그는 `-f` 로 추적 불가 → `-F` 필요 ※ 미검증

### 옵션
- `-n <n>` : 출력 행수 지정 (**n**umber of lines) — `-5` 형태 축약 가능
- `-c <n>` : 출력 바이트수 지정 (**c**haracters/bytes) ※ 미검증
- `-f` : 추가 기록 실시간 추적 (**f**ollow) — 자동화 환경 주의
- `-n +<n>` : n번째 행부터 끝까지 (**n**umber, `+` = 선두 기준) ※ 미검증

---

## 연관 명령어
- [[head]] : 파일 선두 출력 — 반대 방향
- [[journalctl]] : systemd 통합 로그 — 서비스 로그는 `journalctl -n` 우선
- [[grep]] : `tail` 출력 필터링
- [[sed]] : 중간 구간 조회 — `tail` 로 도달 불가한 범위
- [[nohup]] : 백그라운드 실행 로그를 `tail` 로 추적
