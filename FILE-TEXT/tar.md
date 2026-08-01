---
command: tar
category: FILE-TEXT
aliases: [unzip]
tags:
  - linux/text
  - task/inspect
  - task/verify
  - topic/filesystem
  - privilege/mixed
  - danger/data-loss
related: ["[[unzip]]", "[[sshpass]]", "[[grep]]", "[[file]]", "[[head]]", "[[sha256sum]]", "[[zip]]"]
distro: 전체
verified: Rocky Linux 9.6 (원격 조회 / 증거 아카이브 생성)
updated: 2026-08-01
---

# tar

- 다중 파일을 단일 아카이브로 묶고 해제하는 도구
- 어원: **t**ape **ar**chive (테이프 백업 유래)
- 조회는 일반 사용자 가능, 해제 대상 경로 쓰기 권한 필요

---

## tar 목록 조회 (`tzf`)

```bash
tar tzf <아카이브>

# Examples
tar tzf /home/<계정>/stat-job/jobs/stat-job-runner-5.0.0.tar.gz | head -20   # 배포 아카이브 구성 확인
```

### 명령어 설명
- 사용 목적
	- 배포 아카이브 구성·경로 구조 확인 시 사용 (해제 전 검증)
	- 아카이브 내 특정 파일 포함 여부 확인 시 사용
	- 해제 시 생성될 디렉터리 구조 사전 확인 시 사용
- 특이사항
	- **해제 전 목록 조회 필수** → 절대경로·상위경로(`../`) 포함 아카이브는 해제 시 시스템 파일 덮어쓰기 위험 ⚠
	- 최상위 디렉터리 없는 아카이브는 해제 시 현재 디렉터리 오염 → `-C` 로 대상 지정 권장
	- **압축 형식과 옵션 일치 필요**
		- `.tar.gz` `.tgz` : `z` (gzip)
		- `.tar.bz2` : `j` (bzip2)
		- `.tar.xz` : `J` (xz)
		- `.tar` : 압축 옵션 없음
	- GNU tar 는 형식 자동 판별 지원 → `-a` 또는 옵션 생략 가능 ※ 미검증
	- 출력량 과다 시 [[head]] 제한, 특정 파일 확인은 [[grep]] 조합

### 옵션
- `t` : 내용 목록 출력 (**t**able of contents) — 조회 전용, 안전
- `z` : gzip 압축·해제 (**z**ip/gzip)
- `f <파일>` : 아카이브 파일 지정 (**f**ile) — 마지막 위치 필수
- `tzf` : gzip 아카이브 목록 조회 — 최빈출 조합
- `x` : 해제 (e**x**tract) ⚠ 기존 파일 덮어씀 ※ 미검증
- `c` : 아카이브 생성 (**c**reate)
- `v` : 처리 파일명 표시 (**v**erbose) ※ 미검증
- `-C <경로>` : 대상 디렉터리 지정 (**C**hange directory)

---

## tar 아카이브 생성 (`czf`)

```bash
tar czf <출력.tar.gz> -C <기준경로> <대상>

# Examples
tar czf /root/ir-evidence-20260730.tar.gz -C /root \
  --exclude='ir-20260730/samples' --exclude='ir-20260730/bandage.dump' ir-20260730
tar tzf /root/ir-evidence-20260730.tar.gz          # 생성 후 내용 검증
```

### 명령어 설명
- 사용 목적
	- 로그·설정 등 증거 다수 파일을 단일 아카이브로 묶을 시 사용
- 특이사항
	- **악성 샘플·개인정보 포함 덤프는 `--exclude`로 분리** → 백신 격리·실수 실행·전송 위험 회피
		- 악성 샘플은 별도 암호화 압축([[zip]]), DB 덤프는 전송 대상 분리
	- `-C <경로>`로 기준 디렉터리 지정 → 아카이브 내 상대경로 정규화
	- 생성 직후 `tar tzf`로 목록·제외 대상 미포함 검증 권장
	- 전송 무결성은 생성 시점 해시로 검증 → [[sha256sum]]

---

## 연관 명령어
- [[unzip]] : ZIP·JAR 아카이브 조회 — Java 산출물은 ZIP 형식
- [[sshpass]] : 원격 서버 배포 아카이브 조회 시 연계
- [[grep]] : 아카이브 목록 내 특정 파일 검색
- [[file]] : 아카이브 압축 형식 판정 — 옵션 선택 근거
- [[head]] : 목록 출력량 제한
- [[sha256sum]] : 아카이브 무결성 해시 산출·대조
- [[zip]] : 악성 샘플 등 분리 대상의 암호화 압축
