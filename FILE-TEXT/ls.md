---
command: ls
category: FILE-TEXT
aliases: [ll]
tags:
  - linux/text
  - task/inspect
  - topic/filesystem
  - task/verify
  - privilege/mixed
related: ["[[find]]", "[[stat]]", "[[du]]", "[[file]]", "[[lsblk]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# ls

- 디렉터리 내용·파일 속성 목록 출력 도구
- 어원: **l**i**s**t (목록)
- 조회는 일반 사용자 가능, 권한 제한 디렉터리는 root 필요

---

## ls

```bash
ls [옵션] [경로]

# Examples
ls -la /Users/sunwoo/Documents/Linux-Command            # 숨김 파일 포함 상세 목록
ls -la .env* 2>/dev/null                                # 글롭 + 존재 불확실 대응
ls -la .git/hooks/pre-commit 2>/dev/null || echo "설치 안 됨"   # 존재 여부 판정
ls -la docs/openapi.json 2>/dev/null                    # 단일 파일 속성 확인
ls /opt/homebrew/bin 2>/dev/null | head -5              # 목록 + 출력 제한
ls src/.../controller/ src/.../dto/req/ 2>/dev/null      # 다중 디렉터리 동시 조회
```

### 명령어 설명
- 사용 목적
	- 디렉터리 구성·파일 존재 여부 확인 시 사용
	- 권한·소유자·크기·수정시각 확인 시 사용 (`-l`)
	- 숨김 설정 파일 확인 시 사용 (`-a`)
- 특이사항
	- **`-a` 없으면 `.` 로 시작하는 파일 미표시** → `.env` `.git` 등 누락
	- 존재하지 않는 경로 지정 시 stderr 출력 → `2>/dev/null || echo` 로 존재 판정 관용
	- 재귀 탐색은 `-R` 이나 대량 출력 발생 → [[find]] 사용 권장
	- 파일명 정렬은 로케일 종속 → 재현성 필요 시 `LC_ALL=C`
	- 다중 디렉터리 지정 시 디렉터리명 헤더 자동 삽입

### 옵션
- `-l` : 상세 목록 (권한·소유자·크기·시각) (**l**ong)
- `-a` : 숨김 파일 포함 (**a**ll)
- `-la` : 상세 + 숨김 포함 — 최빈출 조합
- `-h` : 크기를 읽기 쉬운 단위로 (**h**uman-readable) ※ 미검증
- `-t` : 수정 시각 기준 정렬 (**t**ime) ※ 미검증
- `-R` : 하위 디렉터리 재귀 (**R**ecursive) ※ 미검증
- `-d` : 디렉터리 자체 정보만 (**d**irectory) ※ 미검증

---

## 연관 명령어
- [[find]] : 재귀 탐색·조건 검색 — `ls -R` 대체
- [[stat]] : 단일 파일 상세 메타데이터 — `ls -l` 보다 정밀
- [[du]] : 디렉터리 누적 용량 — `ls` 는 디렉터리 자체 크기만 표시
- [[file]] : 파일 유형·인코딩 판정
- [[lsblk]] : 블록 디바이스 목록 — 파일시스템이 아닌 장치 대상
