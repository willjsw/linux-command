---
command: head
category: FILE-TEXT
aliases: []
tags:
  - linux/text
  - task/inspect
  - topic/filesystem
  - task/search
  - privilege/user
related: ["[[tail]]", "[[sed]]", "[[cat]]", "[[wc]]", "[[grep]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# head

- 파일·표준입력 선두 일부 출력 도구
- 어원: **head** (머리 부분)
- 일반 사용자 실행 가능

---

## head

```bash
head [-n <행수>] <file>

# Examples
head -15 db/changelog/024-band-application-is-latest.sql   # 선두 15행
head -1 $W/tbl_common.txt                                   # 첫 행만
head -50 docs/schedule-analysis.md 2>/dev/null              # 존재 불확실 파일 조회
head -c 300 docs/openapi.json.new                           # 바이트 단위 제한
head -50 /tmp/bootrun.log | grep -i "jdbc\|Connection\|refused"   # 로그 선두 필터
find . -type f | head -50                                   # 파이프 출력 제한
```

### 명령어 설명
- 사용 목적
	- 대용량 파일 구조·헤더 확인 시 사용
	- 파이프 출력량 제한 시 사용 (토큰·화면 절약)
	- 로그 파일 초기 기동 메시지 확인 시 사용
- 특이사항
	- 기본값은 10행 → `-n` 생략 시 10행
	- **`-c` 는 행 경계 무시하고 바이트 절단** → JSON 등 구조 데이터는 중간에서 잘림
	- 파이프 상류가 대용량이어도 `head` 종료 시 상류에 `SIGPIPE` 전달 → 조기 중단으로 효율적
	- 다중 파일 지정 시 `==> file <==` 헤더 자동 삽입

### 옵션
- `-n <n>` : 출력 행수 지정 (**n**umber of lines) — `-15` 형태 축약 가능
- `-c <n>` : 출력 바이트수 지정 (**c**haracters/bytes)
- `-q` : 다중 파일 시 파일명 헤더 억제 (**q**uiet) ※ 미검증

---

## 연관 명령어
- [[tail]] : 파일 말미 출력 — 반대 방향
- [[sed]] : `sed -n '1,Np'` 로 동일 효과, 중간 구간은 `sed` 만 가능
- [[cat]] : 전체 출력 — 소용량 파일 시 대체
- [[wc]] : 행수 확인 후 `head` 범위 결정
- [[grep]] : `head` 출력 필터링
