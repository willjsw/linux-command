---
command: iconv
category: FILE-TEXT
aliases: []
tags:
  - linux/text
  - task/search
  - task/inspect
  - topic/encoding
  - privilege/user
related: ["[[grep]]", "[[tr]]", "[[file]]", "[[cat]]", "[[sed]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# iconv

- 문자 인코딩 변환 도구
- 어원: **i**nternationalization **conv**ersion (문자셋 변환)
- 일반 사용자 실행 가능

---

## iconv

```bash
iconv -f <원본인코딩> -t <목표인코딩> <file>

# Examples
iconv -f EUC-KR -t UTF-8 add_agent.c 2>/dev/null | cat -n            # 레거시 C 소스 조회
iconv -f EUC-KR -t UTF-8 show_tenant.c 2>&1 | grep -n "TENANT|brief" # 변환 후 검색
iconv -f EUC-KR -t UTF-8 show_tenant.c 2>&1 | sed -n '210p'          # 변환 후 특정 행
iconv -f EUC-KR -t UTF-8 "$F" 2>/dev/null | sed -n '385,410p'        # 변환 후 범위 추출
iconv -f EUC-KR -t UTF-8 libmmi.h 2>/dev/null | grep -n -A 20 "mmiArg_t"   # 구조체 정의 조회
iconv -f UTF-8 -t UTF-8 "보고서.md" 2>/dev/null | head -80 || head -80 "보고서.md"   # 인코딩 검증 후 폴백
```

### 명령어 설명
- 사용 목적
	- EUC-KR 등 레거시 인코딩 소스 파일 조회 시 사용 (한글 주석 포함 C 소스)
	- 인코딩 변환 후 [[grep]]·[[sed]] 검색 파이프 구성 시 사용
	- 인코딩 유효성 검증 시 사용 (`-f UTF-8 -t UTF-8` 자기 변환)
- 특이사항
	- **EUC-KR 파일은 [[grep]] 이 바이너리로 판정 → 검색 결과 누락**
		- 대응 1: `iconv -f EUC-KR -t UTF-8 file | grep` 파이프 (권장)
		- 대응 2: `grep -a` 로 텍스트 강제 취급 → 한글은 깨져 출력
	- 변환 불가 바이트 발생 시 즉시 중단 → `2>/dev/null` 로 억제 관용
		- 부분 출력만 얻어지므로 **완전성 필요 시 오류 확인 필수**
	- `-f UTF-8 -t UTF-8` 성공 여부로 UTF-8 유효성 판정 가능 → 실패 시 다른 인코딩
	- 파일 직접 수정 옵션 없음 → 리다이렉트로 새 파일 생성 필요
	- 지원 인코딩 목록은 `iconv -l` 로 확인

### 옵션
- `-f <enc>` : 원본 인코딩 지정 (**f**rom)
- `-t <enc>` : 목표 인코딩 지정 (**t**o)
- `-l` : 지원 인코딩 목록 출력 (**l**ist)
- `-c` : 변환 불가 문자 무시하고 계속 (**c**lear/omit) ※ 미검증

---

## 연관 명령어
- [[grep]] : EUC-KR 파일 직접 검색 시 누락 발생 → `iconv` 선행 필수
- [[tr]] : 제어문자·NUL 제거 — 인코딩 변환과 목적 상이
- [[file]] : 파일 인코딩·유형 사전 판정
- [[cat]] : 변환 후 조회 대상
- [[sed]] : 변환 후 범위 추출·치환
