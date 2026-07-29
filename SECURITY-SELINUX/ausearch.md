---
command: ausearch
category: SECURITY-SELINUX
aliases: [audit2why, audit2allow]
tags:
  - linux/security
  - linux/log
  - topic/selinux
  - topic/troubleshooting
  - task/diagnose
  - privilege/root
  - distro/rhel
  - distro/rocky
related: ["[[getenforce]]", "[[journalctl]]", "[[firewall-cmd]]"]
distro: RHEL 계열 (audit 패키지)
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# ausearch

- audit 로그 검색 도구
- 어원: **au**dit **search**
- SELinux 차단(AVC) 내역 확인의 표준 수단

---

## ausearch -m avc

```bash
ausearch -m avc -ts recent

# Examples
ausearch -m avc -ts recent              # 최근 차단 내역
ausearch -m avc -ts today               # 당일 전체
ausearch -m avc -ts recent | audit2why  # 차단 원인 해설
```

### 명령어 설명
- 사용 목적
	- **SELinux가 차단한 동작 확인** 시 사용
	- 애플리케이션 동작 실패가 SELinux 원인인지 판별 시 사용
	- SELinux 비활성화 이전 원인 진단 목적으로 사용
- 특이사항
	- `avc` = **A**ccess **V**ector **C**ache → SELinux 접근 거부 기록
	- 출력 없으면 SELinux 차단 아님 → 방화벽([[firewall-cmd]]) 등 다른 원인 조사 필요
	- 차단 해소 정책 생성은 `audit2allow` 사용

### 옵션
- `-m <type>` : 메시지 타입 (**m**essage), `avc`·`USER_LOGIN` 등
- `-ts <time>` : 시작 시각 (**t**ime **s**tart), `recent`·`today`·`boot` 등
- `-i` : 숫자를 이름으로 변환 (**i**nterpret)

---

## audit2why / audit2allow

```bash
ausearch -m avc -ts recent | audit2why      # 차단 원인 해설
ausearch -m avc -ts recent | audit2allow    # 허용 정책 규칙 생성
```

### 명령어 설명
- 사용 목적
	- `audit2why` : 차단 사유를 사람이 읽을 수 있는 형태로 확인 시 사용
	- `audit2allow` : 차단 해소용 SELinux 정책 모듈 생성 시 사용
- 특이사항
	- `audit2allow` 결과를 검토 없이 적용 시 보안 정책 과대 완화 위험 → 내용 확인 필수

---

## 진단 순서 (서비스 동작 실패 시)

```bash
systemctl status <service>              # ① 서비스 자체 오류
journalctl -u <service> -n 50           # ② 서비스 로그
firewall-cmd --list-all                 # ③ 방화벽 차단
getenforce                              # ④ SELinux 모드
ausearch -m avc -ts recent              # ⑤ SELinux 차단 내역
```

---

## 연관 명령어
- [[getenforce]] : SELinux 모드 확인·변경
- [[journalctl]] : 서비스 일반 로그 조회
- [[firewall-cmd]] : 방화벽 차단과 구분
