---
command: tr
category: FILE-TEXT
aliases: []
tags:
  - linux/text
  - task/search
  - topic/encoding
  - topic/regex
  - privilege/user
related: ["[[sed]]", "[[iconv]]", "[[cat]]", "[[grep]]", "[[wc]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# tr

- 문자 단위 치환·삭제 도구
- 어원: **tr**anslate (문자 변환)
- 일반 사용자 실행 가능

---

## tr

```bash
tr <집합1> <집합2> < <file>
tr -d <집합> < <file>

# Examples
tr ':' '\n' < /tmp/cp.txt | grep -i mybatis            # 클래스패스를 행 단위 분해
tr ':' '\n' < /tmp/cp.txt | grep -iE "h2|postgresql"   # 구분자 분해 후 필터
tr -d '\000' < src/mmam/cmd_list.dat | head -100       # NUL 바이트 제거 후 조회
tr -d '\r' < src/pbxm/pbxm.h | tr -cd '\11\12\15\40-\176' > /tmp/ascii.txt   # CR 제거 + ASCII 외 제거
wc -l < file | tr -d ' '                               # 공백 제거 (macOS wc 대응)
echo "$name" | tr 'A-Z' 'a-z'                          # 소문자 변환
tr "\0" " " < /proc/678992/cmdline; echo               # /proc cmdline NUL→공백 (argv 확인)
tr "\0" "\n" < /proc/678992/environ                    # /proc environ NUL→개행 (환경변수 나열)
```

### 명령어 설명
- 사용 목적
	- 콜론·쉼표 구분 문자열을 행 단위로 분해 시 사용 (클래스패스 분석 등)
	- NUL·CR 등 제어문자 제거 시 사용 (바이너리 혼재 파일 조회)
	- `/proc/<PID>/cmdline`·`environ`의 NUL 구분 필드를 공백·개행으로 변환 시 사용 (프로세스 조사)
	- 출력 문자열의 공백 제거 시 사용 (macOS `wc` 패딩 대응)
	- 대소문자 일괄 변환 시 사용
- 특이사항
	- **표준입력만 처리 → 파일 인자 미지원** → `tr ... < file` 리다이렉트 필수
	- 문자 집합 단위 처리 → 문자열 치환은 불가, [[sed]] 사용
	- `-c` 는 지정 집합의 **여집합** 대상 → `-cd '\40-\176'` = 출력 가능 ASCII 외 전부 삭제
	- 8진 이스케이프 표기 사용 → `\000`(NUL) `\11`(TAB) `\12`(LF) `\15`(CR) `\40-\176`(출력 가능 ASCII)
	- 비UTF-8 인코딩 변환 목적에는 부적합 → [[iconv]] 사용

### 옵션
- `-d <집합>` : 해당 문자 삭제 (**d**elete)
- `-c` : 집합의 여집합 대상 (**c**omplement)
- `-cd <집합>` : 집합 외 문자 전부 삭제 — 정상 문자만 잔존
- `-s` : 반복 문자 1개로 축약 (**s**queeze) ※ 미검증

---

## 연관 명령어
- [[sed]] : 문자열·정규표현식 치환 — `tr` 은 문자 단위 한정
- [[iconv]] : 문자 인코딩 변환 — 다국어 문자셋 변환 시 사용
- [[cat]] : `tr` 전처리 후 조회 대상
- [[grep]] : `tr` 분해 결과 필터링
- [[wc]] : macOS `wc` 출력 공백 제거에 `tr -d ' '` 사용
