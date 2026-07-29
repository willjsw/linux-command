---
command: uniq
category: FILE-TEXT
aliases: []
tags:
  - linux/text
  - task/inspect
  - task/verify
  - topic/capacity
  - privilege/user
related: ["[[sort]]", "[[comm]]", "[[wc]]", "[[file]]", "[[awk]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# uniq

- 인접 중복 행 제거·빈도 집계 도구
- 어원: **uniq**ue (고유 항목)
- 일반 사용자 실행 가능

---

## uniq

```bash
sort <file> | uniq [옵션]

# Examples
file images/*.png | sed 's/.*: //' | sort | uniq -c    # 파일 유형별 빈도 집계
sort types.txt | uniq                                   # 중복 제거
sort types.txt | uniq -d                                # 중복 항목만
sort types.txt | uniq -c | sort -rn | head              # 빈도 순위 상위 항목
```

### 명령어 설명
- 사용 목적
	- 목록에서 중복 항목 제거 시 사용
	- 값별 출현 빈도 집계 시 사용 (`-c`)
	- 중복 존재 여부 검출 시 사용 (`-d`)
- 특이사항
	- **인접 행만 비교 → 정렬되지 않은 입력은 중복 미제거**
		- [[sort]] 선행 필수 → `sort | uniq` 관용구
		- 단순 중복 제거만 필요 시 `sort -u` 로 축약 가능
	- `-c` 출력은 `<개수> <값>` 형태 → 빈도 순위는 `| sort -rn` 추가
	- macOS(BSD uniq)는 `-c` 출력에 공백 패딩 삽입 → 파싱 시 [[tr]] 또는 [[awk]] 정규화 필요

### 옵션
- `-c` : 각 행 앞에 출현 횟수 부여 (**c**ount)
- `-d` : 중복된 행만 출력 (**d**uplicate)
- `-u` : 중복 없는 행만 출력 (**u**nique only) ※ 미검증
- `-i` : 대소문자 무시 비교 (**i**gnore case) ※ 미검증

---

## 연관 명령어
- [[sort]] : `uniq` 입력 정렬 — 선행 필수, `sort -u` 로 대체 가능
- [[comm]] : 두 파일 간 중복·차이 — `uniq` 는 단일 입력 대상
- [[wc]] : 중복 제거 전후 개수 비교
- [[file]] : `file | sort | uniq -c` 유형별 집계 패턴
- [[awk]] : 정렬 불필요한 중복 제거·집계 시 대체 (`!seen[$0]++`)
