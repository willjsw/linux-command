---
command: file
category: FILE-TEXT
aliases: []
tags:
  - linux/text
  - task/inspect
  - task/diagnose
  - topic/encoding
  - topic/filesystem
  - privilege/user
related: ["[[iconv]]", "[[cat]]", "[[ls]]", "[[grep]]", "[[uniq]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# file

- 파일 내용 기반 유형·인코딩 판정 도구
- 어원: **file** (파일 유형 식별)
- 일반 사용자 실행 가능

---

## file

```bash
file <file>

# Examples
file /Users/sunwoo/.../pbx/src/mmam/show_tenant.c        # 소스 파일 인코딩 판정
file script/ecli                                          # 실행 바이너리 유형 확인
file script/SU-CMD script/term 2>/dev/null                # 다중 파일 동시 판정
file COMMAND.xml                                          # 텍스트 유형 확인
file images/*.png 2>/dev/null | sed 's/.*: //' | sort | uniq -c   # 유형별 빈도 집계
```

### 명령어 설명
- 사용 목적
	- 확장자와 실제 내용 불일치 여부 확인 시 사용
	- **비UTF-8 인코딩 판정 시 사용** → [[iconv]] 변환 필요 여부 판단
	- 바이너리·텍스트 구분 시 사용 ([[cat]] 출력 전 안전성 확인)
	- 실행 파일 아키텍처 확인 시 사용
- 특이사항
	- **확장자 무시하고 내용(매직 넘버) 기반 판정** → 확장자 위장 파일 식별 가능
	- 출력 형식은 `<파일명>: <판정결과>` → 결과만 필요 시 `sed 's/.*: //'`
	- 인코딩 판정은 추정치 → **EUC-KR 은 `ISO-8859` 등으로 오판 가능**
		- 확정 필요 시 `iconv -f EUC-KR -t UTF-8` 변환 성공 여부로 교차 검증
	- NUL 바이트 포함 시 `data` 로 판정 → 실제로는 텍스트인 경우 존재 ([[tr]] 전처리 필요)

### 옵션
- `-i` : MIME 타입 형식 출력 (**i**nternet media type) ※ 미검증
- `-b` : 파일명 생략, 판정 결과만 (**b**rief) ※ 미검증
- `-L` : 심볼릭 링크 대상 판정 (**L**ink 추적) ※ 미검증

---

## 연관 명령어
- [[iconv]] : `file` 로 판정한 인코딩 변환 — 후속 작업
- [[cat]] : 텍스트 확정 후 조회 — 바이너리는 터미널 손상 위험
- [[ls]] : 파일 존재·속성 확인 — 내용 판정은 `file` 담당
- [[grep]] : 바이너리 판정 파일은 `-a` 필요 → `file` 로 사전 확인
- [[uniq]] : `file | sort | uniq -c` 유형별 집계 패턴
