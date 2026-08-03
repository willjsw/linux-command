---
command: ldd
category: SYSTEM-INFO
aliases: [otool, objdump]
tags:
  - linux/system
  - task/inspect
  - task/diagnose
  - task/verify
  - topic/troubleshooting
  - topic/filesystem
  - privilege/user
related: ["[[file]]", "[[which]]", "[[rpm]]", "[[dnf]]", "[[env]]", "[[find]]"]
distro: 전체 (glibc 기반 — RHEL 계열, Debian 계열)
verified: 미검증 (참조용)
updated: 2026-08-03
---

# ldd

- 실행 파일·공유 라이브러리의 동적 링크 의존성 목록 출력 도구
- 어원: **l**ist **d**ynamic **d**ependencies
- 일반 사용자 실행 가능, glibc 제공 스크립트 (별도 패키지 설치 불요)
- macOS 미제공 → `otool -L` 이 대응 도구

---

## ldd

```bash
ldd <실행파일|공유라이브러리>

# Examples
ldd /usr/bin/ls                      # 바이너리 의존 라이브러리 열거
ldd /usr/sbin/httpd | grep -i ssl    # 특정 라이브러리 링크 여부 확인
ldd <바이너리> | grep "not found"     # 누락 의존성만 추출 → 실행 실패 원인 규명
```

### 명령어 설명
- 사용 목적
	- `error while loading shared libraries` 발생 시 누락 라이브러리 식별에 사용
	- 바이너리가 특정 라이브러리 버전에 링크되었는지 확인 시 사용
	- 정적·동적 링크 구분 판정 시 사용
- 특이사항
	- **`not found` 표시 항목이 실행 실패 직접 원인** → 해당 라이브러리 제공 패키지 설치 필요
		- 제공 패키지 역추적은 [[rpm]] `-qf` 또는 [[dnf]] `provides`
	- 정적 링크 바이너리는 `not a dynamic executable` 반환 → 의존성 없음 의미
		- [[file]] 출력의 `statically linked` 와 동일 판정 → 상호 교차 확인 가능
	- 탐색 경로는 `/etc/ld.so.conf` · `LD_LIBRARY_PATH` 종속 → 환경별 결과 상이 가능 → [[env]]
	- **신뢰할 수 없는 바이너리 대상 실행 금지** ⚠
		- 구현에 따라 대상을 실제 로더로 적재 → 임의 코드 실행 위험 존재
		- 악성 샘플 조사 시 `objdump -p` · `readelf -d` 등 비실행 방식 권장
	- macOS 미제공 → 동일 목적은 `otool -L`, 크로스 환경 스크립트에서 가용성 확인 선행 → [[which]]

### 옵션
- `-v` : 심볼 버전 정보 포함 상세 출력 (**v**erbose) ※ 미검증
- `-u` : 미사용 직접 의존성 표시 (**u**nused) ※ 미검증
- `-d` : 누락 객체 탐지 + 데이터 재배치 수행 (**d**ata relocation) ※ 미검증
- `-r` : 데이터·함수 재배치 모두 수행 (**r**elocation) ※ 미검증

---

## 연관 명령어
- [[file]] : 바이너리 형식·정적링크 여부 판정 — `ldd` 는 의존 대상 열거, 역할 보완
- [[which]] : 대상 바이너리 실제 경로 확보 — `ldd $(which <명령>)` 조합
- [[rpm]] : `-qf <라이브러리경로>` 로 제공 패키지 역추적
- [[dnf]] : `provides` 로 누락 라이브러리 제공 패키지 검색 후 설치
- [[env]] : `LD_LIBRARY_PATH` 등 탐색 경로 환경변수 확인
- [[find]] : 라이브러리 파일 실제 존재 위치 탐색
