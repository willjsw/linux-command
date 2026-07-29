---
command: find
category: FILE-TEXT
aliases: [locate]
tags:
  - linux/text
  - task/search
  - task/inspect
  - topic/filesystem
  - topic/regex
  - privilege/mixed
related: ["[[grep]]", "[[xargs]]", "[[ls]]", "[[wc]]", "[[sort]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# find

- 디렉터리 트리를 재귀 순회하며 조건 일치 파일·디렉터리 탐색 도구
- 어원: **find** (탐색)
- 조회는 일반 사용자 가능, 권한 없는 디렉터리 접근 시 `Permission denied` 발생 → `2>/dev/null` 억제 관용

---

## find

```bash
find <경로> <조건> [동작]

# Examples
find . -name "sp_insert_cc_agent_dd_tot.sql" 2>/dev/null     # 파일명 정확 일치
find src/main/kotlin -iname "*SetlistTrackToJamConverter*"    # 대소문자 무시 부분 일치
find src/main/kotlin/com/bandage/domain/schedule -type f      # 일반 파일만
find src/main/kotlin/.../schedule_bak -type f | sort          # 정렬 조합
find src/main/kotlin/.../setlist -type f | head -50           # 출력 제한
find src/test/kotlin/.../schedule_bak -type f 2>/dev/null | wc -l   # 개수 집계
find . -path "*cc_cti_stat_pipeline*" -name "*st_agent_dd_tot*"     # 경로 + 이름 동시 조건
find <dir> -type f | xargs -I{} basename {}                   # 파일명만 추출
find . -name "*.jsonl" -print0 | xargs -0 cat                 # 공백 안전 전달
```

### 명령어 설명
- 사용 목적
	- 파일명·확장자 기준 파일 위치 탐색 시 사용
	- 특정 디렉터리 하위 파일 목록 열거 시 사용
	- 다른 명령으로 파이프 전달할 파일 목록 생성 시 사용
- 특이사항
	- **조건 표현식 순서가 결과 결정** → `-name` 앞에 경로 인자 필수
	- `-o`(OR) 사용 시 우선순위 혼동 발생 → 괄호 `\( \)` 로 명시 권장
	- 권한 오류 다발 시 `2>/dev/null` 로 stderr 분리
	- **공백 포함 파일명은 파이프 전달 시 분리 사고 발생** → `-print0` + `xargs -0` 조합 필수
	- `-name` 은 셸 글롭 패턴, 정규표현식 아님 → `*` `?` `[]` 만 유효

### 옵션
- `-name` : 파일명 글롭 패턴 일치 (**name**)
- `-iname` : 대소문자 무시 파일명 일치 (**i**gnore case + **name**)
- `-type f` : 일반 파일만 (**f**ile)
- `-type d` : 디렉터리만 (**d**irectory)
- `-path` : 전체 경로 기준 패턴 일치 (**path**)
- `-print0` : NULL 구분자 출력 (**print** + **0**) — 공백 포함 파일명 대응
- `-maxdepth <n>` : 탐색 깊이 제한 (**max** + **depth**) ※ 미검증
- `-mtime <n>` : 수정 시각 기준 필터 (**m**odify + **time**) ※ 미검증
- `-exec <cmd> {} \;` : 일치 항목마다 명령 실행 (**exec**ute) ※ 미검증

---

## 연관 명령어
- [[grep]] : 파일 내용 기준 검색 — `find` 는 파일명 기준, 역할 분리
- [[xargs]] : `find` 결과를 명령 인자로 전달
- [[ls]] : 단일 디렉터리 조회 — 재귀 불필요 시 대체
- [[wc]] : `find` 결과 개수 집계
- [[sort]] : `find` 출력 정렬
