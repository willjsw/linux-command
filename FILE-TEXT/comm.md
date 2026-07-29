---
command: comm
category: FILE-TEXT
aliases: []
tags:
  - linux/text
  - task/verify
  - task/diagnose
  - topic/troubleshooting
  - privilege/user
related: ["[[sort]]", "[[diff]]", "[[join]]", "[[uniq]]", "[[grep]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# comm

- 정렬된 두 파일을 행 단위 비교하여 차집합·교집합 출력 도구
- 어원: **comm**on (공통 항목 비교)
- 일반 사용자 실행 가능

---

## comm

```bash
comm [-123] <file1> <file2>

# Examples
comm -23 a.txt b.txt                                   # file1 에만 있는 행
comm -13 views_src.txt views_old.txt                   # file2 에만 있는 행
comm -12 views_src.txt views_old.txt                   # 양쪽 공통 행
comm -12 $W/tbl_repo.txt $W/tbl_db.txt > $W/tbl_common.txt   # 교집합 저장
comm -23 <(sort cols_repo.txt) <(sort cols_db.txt)     # 프로세스 치환으로 즉시 정렬
comm -13 <(sort cols_repo.txt) <(sort cols_db.txt)     # 누락 컬럼 추출
```

### 명령어 설명
- 사용 목적
	- 코드 정의와 DB 실제 스키마 간 테이블·컬럼 차이 검출 시 사용
	- 두 목록의 누락·잉여 항목 식별 시 사용
	- 마이그레이션 전후 객체 목록 대조 시 사용
- 특이사항
	- **양쪽 입력 모두 정렬 필수** → 미정렬 시 오류 없이 잘못된 결과 출력
		- `comm -23 <(sort a) <(sort b)` 프로세스 치환 패턴 권장
	- 기본 출력은 3열 구조 → 억제 옵션 조합으로 원하는 집합만 추출
		- 1열: file1 전용 / 2열: file2 전용 / 3열: 공통
	- **옵션 번호는 "억제할 열"** → `-23` 은 2·3열 억제 = 1열(file1 전용)만 출력
	- 대소문자·후행 공백 차이도 불일치로 판정 → 필요 시 [[sed]] 정규화 선행

### 옵션
- `-1` : 1열(file1 전용) 억제
- `-2` : 2열(file2 전용) 억제
- `-3` : 3열(공통) 억제
- `-23` : file1 에만 있는 행만 출력 (좌차집합)
- `-13` : file2 에만 있는 행만 출력 (우차집합)
- `-12` : 공통 행만 출력 (교집합)

---

## 연관 명령어
- [[sort]] : `comm` 입력 정렬 — 선행 필수 조건
- [[diff]] : 행 순서·내용 차이 비교 — 집합 연산이 아닌 편집 거리 관점
- [[join]] : 키 기준 결합 — 차집합이 아닌 값 병합 목적
- [[uniq]] : 단일 파일 내 중복 처리
- [[grep]] : 단순 포함 여부 확인 — 소량 항목 시 대체
