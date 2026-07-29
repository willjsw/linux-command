---
command: sed
category: FILE-TEXT
aliases: [stream editor]
tags:
  - linux/text
  - task/search
  - task/configure
  - topic/regex
  - privilege/mixed
  - danger/data-loss
related: ["[[grep]]", "[[awk]]", "[[head]]", "[[tail]]", "[[diff]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# sed

- 스트림 단위 행 편집·치환·범위 추출 도구
- 어원: **s**tream **ed**itor
- 조회는 일반 사용자 가능, `-i` 파일 직접 수정 시 대상 파일 쓰기 권한 필요

---

## sed 범위 추출 (`-n ... p`)

```bash
sed -n '<시작>,<끝>p' <file>

# Examples
sed -n '4630,4750p' docs/openapi.json          # 대용량 파일 특정 구간만 조회
sed -n '1,8p' "$f"                              # 파일 선두 8행
sed -n '18,60p' 00_bootstrap.sql                # 중간 구간 조회
sed -n '60,120p' 00_bootstrap.sql | head -40    # 구간 조회 + 재제한
sed -n '/^DECLARE/,$p' norm_proc.sql            # 패턴 시작 → 파일 끝까지
sed -n '/^CREATE TABLE/,/^);/p' schema.sql      # 패턴 시작 → 패턴 끝까지
```

### 명령어 설명
- 사용 목적
	- 대용량 파일에서 특정 행 범위만 발췌 시 사용
	- 패턴 기준 블록 추출 시 사용 (`CREATE TABLE` ~ `);` 등)
	- [[head]]·[[tail]] 조합으로 도달 불가한 중간 구간 조회 시 사용
- 특이사항
	- **`-n` 없이 `p` 사용 시 해당 행이 2회 출력** → 기본 자동 출력과 중복
	- `$` 는 파일 마지막 행 의미
	- 행 번호는 1부터 시작

### 옵션
- `-n` : 자동 출력 억제 (**n**o auto-print) — `p` 와 필수 조합
- `p` : 해당 행 출력 (**p**rint)

---

## sed 치환 (`s///`)

```bash
sed 's/<찾을패턴>/<바꿀문자열>/<플래그>' <file>

# Examples
sed 's/cc_cti_admin\./cc_cti_admin2./g' "$BASE/$f"                      # 스키마명 일괄 변경
sed 's/cc_cti_stat_pipeline\([^_a-z0-9]\)/cc_cti_stat_pipeline_old\1/g' procs_before.sql > procs_after.sql
sed 's/[[:space:]]*$//' file.md                                          # 행말 공백 제거
sed 's/.*: //' <<< "$line"                                              # 콜론 앞부분 제거
sed -i '' 's/기존/신규/' SKILL.md                                        # macOS: 파일 직접 수정
```

### 명령어 설명
- 사용 목적
	- 스키마명·식별자 일괄 치환 시 사용
	- 행말 공백 등 포맷 정리 시 사용
	- 출력 문자열 가공 시 사용 (파이프 연계)
- 특이사항
	- **`-i` 옵션은 macOS(BSD sed)와 GNU sed 문법 상이**
		- macOS: `sed -i '' 's/a/b/'` — 빈 문자열 인자 필수
		- GNU(Linux): `sed -i 's/a/b/'` — 인자 없음
		- 혼용 시 첫 인자가 파일명으로 오인되어 오류 발생
	- `-i` 는 원본 파일 즉시 덮어씀 → **선행 백업 또는 리다이렉트 출력 검증 권장**
	- 기본 정규표현식(BRE) → 그룹 캡처는 `\( \)`, 후방참조는 `\1`
	- `g` 플래그 없으면 **각 행의 첫 일치만** 치환

### 옵션
- `s///` : 치환 (**s**ubstitute)
- `g` : 행 내 전체 일치 치환 (**g**lobal)
- `-i` : 파일 직접 수정 (**i**n-place) ⚠ 원본 덮어씀
- `-E` : 확장 정규표현식 사용 (**E**xtended) ※ 미검증

---

## 연관 명령어
- [[grep]] : 검색 전용 — `sed` 는 검색 + 편집
- [[awk]] : 필드 단위 처리 — `sed` 는 행 단위 치환
- [[head]] : 파일 선두 조회 — `sed -n '1,Np'` 와 동일 효과
- [[tail]] : 파일 말미 조회 — `sed -n '/pat/,$p'` 로 대체 가능
- [[diff]] : `sed` 로 정규화한 두 파일 비교
