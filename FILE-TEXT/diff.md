---
command: diff
category: FILE-TEXT
aliases: []
tags:
  - linux/text
  - task/verify
  - task/diagnose
  - topic/troubleshooting
  - privilege/user
related: ["[[comm]]", "[[sed]]", "[[sort]]", "[[wc]]", "[[cat]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# diff

- 두 파일의 행 단위 차이 출력 도구
- 어원: **diff**erence (차이)
- 일반 사용자 실행 가능

---

## diff

```bash
diff <file1> <file2>

# Examples
diff cols_db_cc_agent_mi_tot.txt cols_local_st_agent_mi_tot.txt      # DB vs 로컬 컬럼 대조
diff /tmp/committed.json /tmp/gen.json | head -60                     # 생성물 일치 검증
diff /tmp/committed.json /tmp/gen.json | wc -l                        # 차이 규모 수치화
diff <(sed 's/[[:space:]]*$//' a.md) <(sed 's/[[:space:]]*$//' b.md) >/dev/null   # 공백 정규화 후 비교
diff <(python3 -m json.tool --sort-keys a.json) <(python3 -m json.tool --sort-keys b.json)   # 키 정렬 후 비교
diff <(sed -n '/^DECLARE/,$p' remote.sql) <(sed -n '/^DECLARE/,$p' local.sql)     # 본문 구간만 비교
```

### 명령어 설명
- 사용 목적
	- 원격 DB 정의와 로컬 소스 파일 간 동일성 검증 시 사용
	- 커밋된 산출물과 재생성 산출물 일치 확인 시 사용 (CI 검증 패턴)
	- 마이그레이션 전후 스키마 차이 확인 시 사용
- 특이사항
	- **프로세스 치환 `<(...)` 조합이 핵심 활용 패턴** → 정규화 후 비교로 무의미한 차이 제거
		- 행말 공백 : `sed 's/[[:space:]]*$//'`
		- JSON 키 순서 : `python3 -m json.tool --sort-keys`
		- 헤더 등 가변 구간 : `sed -n '/^패턴/,$p'` 로 본문만 추출
	- **종료 코드가 판정 수단** → 0 = 동일, 1 = 차이 존재, 2 = 오류
		- `diff a b >/dev/null && echo SAME` 형태 스크립트 활용
	- 순서 무관 집합 비교 목적에는 부적합 → [[comm]] 사용
	- 출력량 과다 시 `| head` 또는 `| wc -l` 로 규모만 우선 확인

### 옵션
- `-u` : 통합(unified) 형식 출력 (**u**nified) ※ 미검증
- `-r` : 디렉터리 재귀 비교 (**r**ecursive) ※ 미검증
- `-q` : 차이 유무만 보고 (**q**uiet) ※ 미검증
- `-w` : 공백 차이 무시 (**w**hitespace) ※ 미검증
- `-i` : 대소문자 차이 무시 (**i**gnore case) ※ 미검증

---

## 연관 명령어
- [[comm]] : 집합 연산 비교 — 행 순서 무관 대조 시 사용
- [[sed]] : 비교 전 정규화 전처리
- [[sort]] : 순서 차이 제거 후 비교
- [[wc]] : 차이 행수 집계로 규모 판정
- [[cat]] : 차이 확인 후 원본 조회
