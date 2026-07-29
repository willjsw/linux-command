---
command: awk
category: FILE-TEXT
aliases: [gawk]
tags:
  - linux/text
  - task/search
  - task/inspect
  - topic/regex
  - privilege/user
related: ["[[sed]]", "[[grep]]", "[[stat]]", "[[join]]", "[[sort]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# awk

- 행을 필드로 분해하여 패턴·조건별 처리하는 텍스트 처리 언어
- 어원: 개발자 3인 이름 머리글자 — **A**ho, **W**einberger, **K**ernighan
- 일반 사용자 실행 가능

---

## awk 필드 추출

```bash
awk '{print $<필드번호>}' <file>
awk -F'<구분자>' '{print $1, $2}' <file>

# Examples
lsof -nP -iTCP:5432 -sTCP:LISTEN | awk '{print $1, $2, $9}'    # 프로세스·PID·주소만 추출
awk -F'|' '{print $1"|"$2"|"$3}' types_repo.txt                 # 파이프 구분 필드 재조합
awk -F'|' 'NR==FNR{ok[$1]=1; next} ok[$1]' a.txt b.txt          # 두 파일 키 교집합
```

### 명령어 설명
- 사용 목적
	- 명령 출력에서 특정 컬럼만 추출 시 사용 ([[ps]]·[[lsof]] 연계)
	- 구분자 기준 필드 재조합 시 사용
	- 두 파일 간 키 기준 대조 시 사용 (`NR==FNR` 관용구)
- 특이사항
	- `$0` 은 전체 행, `$1` 부터 필드 — **0번은 필드 아님**
	- 기본 구분자는 연속 공백·탭 → 고정 구분자는 `-F` 명시 필수
	- `NR` 은 전체 누적 행번호, `FNR` 은 현재 파일 행번호 → **다중 파일 처리 시 구분 필수**
	- `NR==FNR{...; next}` 은 첫 파일만 배열에 적재하는 표준 관용구

### 옵션
- `-F <구분자>` : 입력 필드 구분자 지정 (**F**ield separator)
- `-v <var>=<val>` : 셸 값을 awk 변수로 전달 (**v**ariable) ※ 미검증

---

## awk 범위·플래그 블록 추출

```bash
awk '/<시작패턴>/,/<끝패턴>/' <file>
awk '/<시작패턴>/{f=1} f{print} /<끝패턴>/{f=0}' <file>

# Examples
awk '/CREATE TABLE public.p_schedule_board /,/\);/' schema.sql   # 범위 패턴 직접 지정
awk '/^CREATE TABLE/{f=1} f{print} /^\);/{f=0}' schema.sql       # 플래그 변수 방식
```

### 명령어 설명
- 사용 목적
	- DDL 등 블록 구조 텍스트에서 특정 정의만 발췌 시 사용
	- 시작·종료 패턴이 명확한 구간 추출 시 사용
- 특이사항
	- **범위 패턴(`/a/,/b/`)은 시작 패턴 재출현 시 블록 재개** → 다중 매칭 발생
		- 단일 블록만 필요 시 플래그 변수 방식이 제어 용이
	- 정규표현식 내 `/` 는 이스케이프 필요
	- 동일 목적으로 `sed -n '/a/,/b/p'` 사용 가능 → [[sed]] 참조

---

## 연관 명령어
- [[sed]] : 행 단위 치환 — `awk` 는 필드 단위 연산
- [[grep]] : 단순 패턴 필터 — 조건 로직 불필요 시 대체
- [[stat]] : `cut` 고정 위치 필드 추출 — 단순 절단 시 경량 대체
- [[join]] : 정렬된 두 파일 키 결합 — `awk NR==FNR` 의 전용 도구 버전
- [[sort]] : `awk` 전처리·후처리 정렬
