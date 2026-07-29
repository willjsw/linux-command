---
command: basename
category: FILE-TEXT
aliases: [dirname, realpath, readlink]
tags:
  - linux/text
  - task/inspect
  - topic/filesystem
  - task/verify
  - privilege/user
related: ["[[find]]", "[[xargs]]", "[[ls]]", "[[stat]]", "[[file-ops]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# basename / dirname / realpath

- 경로 문자열에서 파일명·디렉터리명·절대경로 추출 도구군
- 문자열 처리 전용 → 실제 파일 존재 여부 무관 (`realpath` 제외)
- 일반 사용자 실행 가능

---

## basename

```bash
basename <경로>

# Examples
find <dir> -type f | xargs -I{} basename {}     # 파일 목록에서 파일명만 추출
basename /opt/app/config.yml                     # config.yml
basename /opt/app/config.yml .yml                # config (확장자 제거)
```

### 명령어 설명
- 사용 목적
	- 전체 경로에서 파일명만 추출 시 사용
	- [[find]] 결과를 파일명 목록으로 변환 시 사용 ([[xargs]] 연계)
	- 확장자 제거 후 이름 취득 시 사용
- 특이사항
	- **문자열 처리 전용 → 존재하지 않는 경로도 정상 처리**
	- 두 번째 인자로 제거할 접미사 지정 가능 → 확장자 제거
	- 대량 처리 시 `xargs -I{}` 는 항목별 프로세스 생성 → 성능 저하
		- 대응: `sed 's|.*/||'` 또는 `awk -F/ '{print $NF}'` 로 단일 프로세스 처리
	- 어원: **base** + **name** (기본 이름)

---

## dirname

```bash
dirname <경로>

# Examples
dirname /opt/app/config.yml         # /opt/app
DIR=$(dirname "$0")                  # 스크립트 자신의 디렉터리 취득
```

### 명령어 설명
- 사용 목적
	- 경로에서 상위 디렉터리 추출 시 사용
	- 스크립트 자신의 위치 기준 상대 경로 구성 시 사용
- 특이사항
	- 디렉터리 구분자 없는 입력은 `.` 반환
	- `$0` 조합은 심볼릭 링크 경유 실행 시 실제 위치와 상이 → `realpath` 병용 필요
	- 어원: **dir**ectory + **name**

---

## realpath / readlink

```bash
realpath <경로>
readlink -f <경로>

# Examples
realpath ./docs/../src           # 심볼릭 링크·상대경로 해소한 절대경로
readlink -f /usr/local/bin/app    # 링크 최종 대상 경로
```

### 명령어 설명
- 사용 목적
	- 상대경로·심볼릭 링크를 절대경로로 정규화 시 사용
	- 심볼릭 링크의 최종 대상 확인 시 사용
- 특이사항
	- **`realpath` 는 대상이 실제 존재해야 성공** → 부재 시 오류
	- **macOS 기본 `readlink` 는 `-f` 미지원** → GNU coreutils(`greadlink`) 필요
		- 이식성 필요 시 `cd "$(dirname "$0")" && pwd` 패턴 사용
	- 어원: **real** + **path** / **read** + **link**

### 옵션
- `-f` : 링크를 재귀적으로 해소한 절대경로 (**f**ollow) — macOS 기본 미지원

---

## 연관 명령어
- [[find]] : 경로 목록 공급 — `basename` 으로 파일명 변환
- [[xargs]] : `find | xargs -I{} basename {}` 조합 패턴
- [[ls]] : 디렉터리 내용 확인 — 경로 조작 전 대상 파악
- [[stat]] : 경로 실체의 메타데이터 확인
- [[file-ops]] : 추출한 경로로 복사·이동·삭제 수행
