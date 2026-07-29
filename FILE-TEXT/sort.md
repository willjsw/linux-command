---
command: sort
category: FILE-TEXT
aliases: []
tags:
  - linux/text
  - task/inspect
  - task/verify
  - topic/capacity
  - privilege/user
related: ["[[uniq]]", "[[comm]]", "[[join]]", "[[wc]]", "[[du]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# sort

- 행 단위 정렬 도구
- 어원: **sort** (정렬)
- 일반 사용자 실행 가능

---

## sort

```bash
sort [옵션] <file>

# Examples
sort cols_repo.txt > a.txt                       # 정렬 결과 파일 저장
find <dir> -type f | sort                        # 파일 목록 정렬
du -sh */ | sort -h | tail -30                   # 사람이 읽는 단위 기준 정렬
file images/*.png | sed 's/.*: //' | sort | uniq -c   # 빈도 집계 전 정렬
comm -23 <(sort a.txt) <(sort b.txt)             # comm 입력 정렬 (필수)
```

### 명령어 설명
- 사용 목적
	- 파일 목록·컬럼 목록 정렬 시 사용
	- [[comm]]·[[join]] 입력 전처리 시 사용 (정렬 필수 요건)
	- [[uniq]] 로 빈도 집계 전 인접화 시 사용
	- `du -sh` 용량 순 정렬 시 사용 (`-h`)
- 특이사항
	- **[[comm]]·[[join]]·[[uniq]] 는 정렬된 입력을 전제** → 미정렬 시 결과 오류 발생
		- 오류가 아닌 잘못된 결과로 나타남 → 사전 정렬 습관화 필요
	- 기본 정렬은 사전순(문자열) → 숫자는 `-n`, 용량 단위는 `-h` 필수
	- `-h` 는 `1K` `2M` `3G` 단위 인식 → `du -sh` 출력 정렬 전용
	- 로케일에 따라 대소문자·특수문자 순서 상이 → 재현성 필요 시 `LC_ALL=C` 지정

### 옵션
- `-h` : 사람이 읽는 용량 단위 기준 정렬 (**h**uman-numeric)
- `-n` : 숫자값 기준 정렬 (**n**umeric)
- `-r` : 역순 정렬 (**r**everse)
- `-u` : 중복 제거 후 출력 (**u**nique) — `sort | uniq` 축약
- `-k <n>` : n번째 필드 기준 정렬 (**k**ey) ※ 미검증
- `-t <구분자>` : 필드 구분자 지정 (**t**erminator) ※ 미검증

---

## 연관 명령어
- [[uniq]] : 중복 제거·빈도 집계 — `sort` 선행 필수
- [[comm]] : 두 파일 차집합·교집합 — `sort` 선행 필수
- [[join]] : 키 기준 결합 — `sort` 선행 필수
- [[wc]] : 정렬 결과 개수 확인
- [[du]] : `du -sh | sort -h` 용량 순위 조회
