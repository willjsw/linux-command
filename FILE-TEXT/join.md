---
command: join
category: FILE-TEXT
aliases: []
tags:
  - linux/text
  - task/verify
  - task/inspect
  - topic/troubleshooting
  - privilege/user
related: ["[[sort]]", "[[comm]]", "[[awk]]", "[[diff]]", "[[stat]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# join

- 정렬된 두 파일을 공통 키 기준 결합 출력 도구
- 어원: **join** (관계형 조인)
- 일반 사용자 실행 가능

---

## join

```bash
join -t'<구분자>' -j<키필드> <file1> <file2>

# Examples
join -t'|' -j1 rt.txt dt.txt | awk -F'|' '{print $1, $2, $3}'   # 키 기준 결합 후 필드 가공
join -a1 -a2 -e '(없음)' -o 0,1.1,2.1 -t'|' rt.txt dt.txt        # 완전 외부 조인 + 결측 표기
join -t'|' -j1 <(sort repo.txt) <(sort db.txt)                   # 프로세스 치환으로 즉시 정렬
```

### 명령어 설명
- 사용 목적
	- 코드 정의 타입과 DB 실제 타입을 컬럼명 기준 대조 시 사용
	- 두 목록의 키별 값 병합 시 사용
	- 한쪽에만 존재하는 키까지 포함한 전체 대조 시 사용 (`-a1 -a2`)
- 특이사항
	- **양쪽 입력 모두 조인 키 기준 정렬 필수** → 미정렬 시 결과 누락 발생 (오류 미출력)
		- 정렬 기준이 `join` 의 키와 일치해야 함 → `sort -t'|' -k1` 등 명시 권장
	- **기본 동작은 내부 조인** → 한쪽에만 있는 키는 출력 누락
		- 전체 대조 필요 시 `-a1 -a2`(완전 외부 조인) + `-e`(결측 대체값) 조합
	- `-o` 필드 지정 표기 → `0`=조인키, `1.1`=file1 1번 필드, `2.1`=file2 1번 필드
	- 키에 구분자가 포함되면 필드 분해 오류 → 복합 키는 `~` 등으로 사전 결합 후 사용
	- 값 병합이 아닌 집합 차이만 필요 시 [[comm]] 이 간결

### 옵션
- `-t <구분자>` : 필드 구분자 지정 (**t**erminator)
- `-j <n>` : 양쪽 공통 조인 키 필드 번호 (**j**oin field)
- `-a1` / `-a2` : 해당 파일의 미일치 행도 출력 (**a**ll) — 외부 조인
- `-e <문자열>` : 결측 필드 대체값 (**e**mpty replacement)
- `-o <목록>` : 출력 필드 순서 지정 (**o**utput format)
- `-v1` / `-v2` : 해당 파일 미일치 행만 출력 (**v**s, 차집합) ※ 미검증

---

## 연관 명령어
- [[sort]] : `join` 입력 정렬 — 선행 필수 조건
- [[comm]] : 집합 차이만 필요 시 간결한 대체
- [[awk]] : `NR==FNR` 관용구로 동일 결합 가능 — 정렬 불필요
- [[diff]] : 행 단위 차이 비교 — 키 기준 결합 아님
- [[stat]] : `cut` 으로 `join` 입력 키 필드 사전 추출
