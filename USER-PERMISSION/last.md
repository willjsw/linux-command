---
command: last
category: USER-PERMISSION
aliases: []
tags:
  - linux/user
  - linux/log
  - task/inspect
  - task/diagnose
  - topic/security
  - topic/remote-access
  - privilege/user
related: ["[[ssh]]", "[[journalctl]]", "[[grep]]", "[[ss]]"]
distro: 전체 (util-linux)
verified: Rocky Linux 9.6 (사고 대응 세션)
updated: 2026-08-01
---

# last

- 로그인·로그아웃 이력 조회 도구
- 어원: **last** logins (최근 로그인 목록)
- `/var/log/wtmp` 기반 → 대화형 로그인 존재 여부 판정에 사용

---

## last

```bash
last

# Examples
last | head -20                       # 최근 로그인 20건
```

### 명령어 설명
- 사용 목적
	- 침해 시각 전후 대화형 로그인 존재 여부 확인 시 사용
	- 접속 소스 IP·세션 지속 시간 확인 시 사용
- 특이사항
	- 특정 시각에 활성 세션 부재 → SSH 대화형 로그인 외 경로 침입 정황
	- **pty 없는 명령 실행(초 단위 연속 세션)은 `last` 미기록** → `/var/log/secure`의 `Accepted publickey` 병행 확인
		- CI 파이프라인 등 자동화 접속 추정 근거 → [[journalctl]]
	- `still logged in` → 조회 시점 활성 세션 (조사자 세션 포함)
	- 소스 IP가 국내 ISP 대역·사람 접속 패턴과 일관한지 대조

---

## 연관 명령어
- [[ssh]] : 접속 경로·인증 방식 확인
- [[journalctl]] / `/var/log/secure` : `Accepted publickey` 성공 기록 대조
- [[grep]] : 인증 로그 필터
- [[ss]] : 현재 활성 연결(조사자 세션 등) 확인
