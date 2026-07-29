---
command: cat
category: FILE-TEXT
aliases: [heredoc]
tags:
  - linux/text
  - task/inspect
  - topic/filesystem
  - topic/encoding
  - privilege/mixed
related: ["[[head]]", "[[tail]]", "[[grep]]", "[[tr]]", "[[iconv]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# cat

- 파일 내용을 표준출력으로 연결 출력하는 도구
- 어원: con**cat**enate (연결)
- 조회는 일반 사용자 가능, 권한 제한 파일은 root 필요

---

## cat

```bash
cat <file>

# Examples
cat /tmp/proc_cols.txt                                   # 파일 전체 출력
cat db/changelog/010-schedule-board-performance.sql      # SQL 정의 확인
cat scripts/pre-commit 2>/dev/null | head -30            # 존재 불확실 파일 + 출력 제한
cat -n add_agent.c                                       # 행 번호 부여
tr -d '\000' < src/mmam/cmd_list.dat | head -100         # cat 대신 tr 로 NUL 제거 후 조회
```

### 명령어 설명
- 사용 목적
	- 소용량 파일 전체 내용 확인 시 사용
	- 다중 파일 연결 출력 시 사용
	- 행 번호 부여 조회 시 사용 (`-n`)
- 특이사항
	- **대용량 파일에 무제한 사용 금지** → [[head]]·[[tail]]·[[sed]] 범위 지정 우선
	- 바이너리 파일 출력 시 터미널 제어문자 손상 발생 → [[file]] 로 사전 판정 권장
	- NUL 바이트 포함 파일은 [[tr]] `-d '\000'` 전처리 필요
	- 비UTF-8 인코딩 파일은 [[iconv]] 변환 후 조회

### 옵션
- `-n` : 전 행에 행 번호 부여 (**n**umber)
- `-A` : 비출력 문자 가시화 (**A**ll, GNU 전용) ※ 미검증
- `-b` : 비어있지 않은 행만 번호 부여 (**b**lank 제외) ※ 미검증

---

## cat 히어독 (heredoc)

```bash
cat <<'EOF' > <file>
<내용>
EOF

# Examples
cat <<'EOF' > /tmp/query.sql
SELECT * FROM cc_agent_dd_tot;
EOF

cat <<EOF > /tmp/conf.txt
HOST=$DB_HOST
EOF
```

### 명령어 설명
- 사용 목적
	- 여러 행 텍스트를 파일로 생성 시 사용
	- 스크립트 내 설정 파일·SQL 생성 시 사용
- 특이사항
	- **구분자 인용 여부가 변수 확장 결정**
		- `<<'EOF'` (인용) : 변수·백틱 확장 없음 → 원문 그대로 기록
		- `<<EOF` (비인용) : `$변수` 확장 발생 → 셸 값 주입 시 사용
	- 종료 구분자는 **행 선두에 위치 필수** → 앞 공백 시 미인식
	- `<<-EOF` 사용 시 선행 탭 제거 허용 ※ 미검증

---

## 연관 명령어
- [[head]] : 선두 일부만 조회 — 대용량 파일 시 `cat` 대체
- [[tail]] : 말미 일부만 조회
- [[grep]] : 내용 필터링 — `cat file | grep` 은 불필요, `grep pat file` 권장
- [[tr]] : NUL·제어문자 제거 후 조회
- [[iconv]] : 비UTF-8 인코딩 파일 변환 후 조회
