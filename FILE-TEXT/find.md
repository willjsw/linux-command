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
related: ["[[grep]]", "[[xargs]]", "[[ls]]", "[[wc]]", "[[sort]]", "[[stat]]", "[[sudo]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6 (사고 대응 세션)
updated: 2026-08-03
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
# 침해 조사: 특정 시각 구간 변경 파일 탐색 (-ls로 상세)
sudo find /home/rocky /etc /var/spool/cron -xdev \
  -newermt "2026-07-30 20:20" ! -newermt "2026-07-30 20:40" -ls 2>/dev/null
sudo find / -xdev -newermt "2026-07-30 20:00" ! -newermt "2026-07-30 21:00" -type f -ls 2>/dev/null | head -60
# 언인스톨러 탐색: 실행 권한·확장자 기준 후보 열거
find <설치경로> -type f \( -perm -u+x -o -name "*.sh" \) 2>/dev/null | head -50
find <설치경로> -name "*.app" -maxdepth 4 2>/dev/null    # 깊이 제한 병용
```

### 명령어 설명
- 사용 목적
	- 파일명·확장자 기준 파일 위치 탐색 시 사용
	- 특정 디렉터리 하위 파일 목록 열거 시 사용
	- 다른 명령으로 파이프 전달할 파일 목록 생성 시 사용
- 특이사항
	- **조건 표현식 순서가 결과 결정** → `-name` 앞에 경로 인자 필수
	- `-o`(OR) 사용 시 우선순위 혼동 발생 → 괄호 `\( \)` 로 명시 권장
	- **패턴 기반 탐색은 축약형 파일명 미검출** → 탐색 실패를 "부재"로 오판 위험
		- 실증: `-iname "*uninstall*" -o -iname "*remove*"` 무결과였으나 실제 파일명은 축약형 `nosuninst`
		- 대응: 후보 부재 판정 전 `-perm -u+x`·`-name "*.app"` 등 **속성 기준 열거**로 재확인
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
- `-newermt <시각>` : 지정 시각 이후 수정 파일 (**newer** + **m**time + **t**ime) — 구간은 `! -newermt <종료>` 조합
- `-xdev` : 다른 파일시스템 미교차 (**x** + **dev**ice) — 마운트 경계에서 탐색 종료
- `-ls` : 일치 항목을 `ls -l` 형식 상세 출력 (**ls**)
- `-maxdepth <n>` : 탐색 깊이 제한 (**max** + **depth**)
- `-perm -u+x` : 소유자 실행 권한 보유 파일 (**perm**ission) — `-` 접두는 "해당 비트 포함" 의미
- `-o` : 조건 OR 결합 (**o**r) — 괄호 `\( \)` 병용 권장
- `-mtime <n>` : 수정 시각 기준 필터 (**m**odify + **time**) ※ 미검증
- `-exec <cmd> {} \;` : 일치 항목마다 명령 실행 (**exec**ute) ※ 미검증

---

## 연관 명령어
- [[grep]] : 파일 내용 기준 검색 — `find` 는 파일명 기준, 역할 분리
- [[xargs]] : `find` 결과를 명령 인자로 전달
- [[ls]] : 단일 디렉터리 조회 — 재귀 불필요 시 대체
- [[wc]] : `find` 결과 개수 집계
- [[sort]] : `find` 출력 정렬
- [[stat]] : 탐색 대상 파일의 정밀 시각(`Modify`/`Birth`) 확인
- [[sudo]] : `/root`·시스템 경로 탐색 시 권한 확보
