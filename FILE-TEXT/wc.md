---
command: wc
category: FILE-TEXT
aliases: []
tags:
  - linux/text
  - task/inspect
  - task/verify
  - topic/capacity
  - privilege/user
related: ["[[find]]", "[[grep]]", "[[sort]]", "[[tr]]", "[[head]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# wc

- 행·단어·바이트 수 집계 도구
- 어원: **w**ord **c**ount
- 일반 사용자 실행 가능

---

## wc

```bash
wc -l <file>

# Examples
wc -l src/main/kotlin/.../ScheduleBlock.kt          # 파일 행수 (규모 파악)
wc -l procs_before.sql                              # SQL 파일 행수
wc -l grub2-install.md BOOT-RECOVERY/grub2-install.md   # 다중 파일 비교
find . -type f 2>/dev/null | wc -l                  # 파일 개수 집계
diff a.json b.json | wc -l                          # 차이 규모 수치화
wc -l < "$OUT"                                      # 파일명 없이 숫자만
wc -l < $W/tbl_common.txt | tr -d ' '               # 앞 공백 제거 (macOS 대응)
```

### 명령어 설명
- 사용 목적
	- 파일 규모(행수) 파악 시 사용
	- [[find]]·[[grep]] 결과 개수 집계 시 사용
	- [[diff]] 차이 규모 수치화 시 사용
	- 스크립트 내 변수 대입용 카운트 취득 시 사용
- 특이사항
	- **파일명 인자 지정 시 출력에 파일명 포함** → 변수 대입 시 방해
		- 대응: `wc -l < file` 리다이렉트 사용 → 숫자만 출력
	- **macOS(BSD wc)는 숫자 앞에 공백 패딩 삽입** → 문자열 비교·연산 시 오류 발생
		- 대응: `| tr -d ' '` 로 공백 제거
		- GNU wc(Linux)는 리다이렉트 시 패딩 없음
	- 마지막 행에 개행 없으면 행수 1 적게 집계
	- 다중 파일 지정 시 `total` 행 자동 추가

### 옵션
- `-l` : 행 수 (**l**ines)
- `-w` : 단어 수 (**w**ords)
- `-c` : 바이트 수 (**c**haracters/bytes)
- `-m` : 문자 수 (**m**ultibyte 인식) ※ 미검증

---

## 연관 명령어
- [[find]] : `find | wc -l` 파일 개수 집계
- [[grep]] : `grep -c` 로 동일 집계 가능 — 파이프 불필요
- [[sort]] : 정렬 결과 개수 확인
- [[tr]] : macOS `wc` 출력 공백 제거
- [[head]] : `wc -l` 로 규모 확인 후 조회 범위 결정
