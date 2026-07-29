---
command: stat
category: FILE-TEXT
aliases: [cut]
tags:
  - linux/text
  - task/inspect
  - task/verify
  - topic/filesystem
  - privilege/user
related: ["[[ls]]", "[[file]]", "[[du]]", "[[file-ops]]", "[[awk]]", "[[join]]"]
distro: 전체
verified: macOS (Darwin 25.5)
updated: 2026-07-30
---

# stat / cut

- 파일 메타데이터 조회(`stat`) 및 고정 위치 필드 추출(`cut`) 도구
- 단순 명령 → 단일 문서 통합
- 일반 사용자 실행 가능

---

## stat

```bash
stat <file>
stat -f "<포맷>" <file>        # BSD(macOS)
stat -c "<포맷>" <file>        # GNU(Linux)

# Examples
stat -f "%Sp %Su %Sg %z bytes" src/main/java/.../WebMvcConfig.java 2>&1   # macOS: 권한·소유자·크기
```

### 명령어 설명
- 사용 목적
	- 파일 권한·소유자·크기·시각 정밀 확인 시 사용
	- 스크립트에서 특정 속성만 추출 시 사용 (포맷 지정)
	- inode·링크 수 확인 시 사용
- 특이사항
	- **포맷 옵션이 GNU 와 BSD(macOS) 간 완전 상이** ⚠
		- GNU(Linux): `stat -c "%A %U %G %s" file` — `-c` 사용
		- BSD(macOS): `stat -f "%Sp %Su %Sg %z" file` — `-f` 사용
		- **이식성 필요 시 [[ls]] `-l` 출력 파싱 또는 분기 처리 필수**
	- 포맷 미지정 시 전체 정보 출력 → 형식도 양 계열 상이
	- 심볼릭 링크는 링크 자체 정보 → 대상 정보는 `-L` 필요 ※ 미검증
	- 어원: **stat** (file **stat**us — `stat(2)` 시스템 콜 유래)

### 옵션 (BSD / macOS)
- `-f "<포맷>"` : 출력 포맷 지정 (**f**ormat)
- `%Sp` : 권한 문자열 (**S**tring **p**ermission)
- `%Su` / `%Sg` : 소유자·그룹명 (**S**tring **u**ser / **g**roup)
- `%z` : 바이트 크기 (si**z**e)

### 옵션 (GNU / Linux)
- `-c "<포맷>"` : 출력 포맷 지정 (**c**ustom format) ※ 미검증
- `%A` : 권한 문자열 ※ 미검증
- `%U` / `%G` : 소유자·그룹명 ※ 미검증
- `%s` : 바이트 크기 ※ 미검증

---

## cut

```bash
cut -f<필드> [-d'<구분자>'] <file>

# Examples
du -sh "$d" 2>/dev/null | cut -f1        # du 출력에서 용량만 추출 (탭 구분)
cut -d':' -f1 /etc/passwd                 # 콜론 구분 첫 필드 (계정명)
cut -c1-10 file.txt                       # 문자 위치 기준 절단
```

### 명령어 설명
- 사용 목적
	- **탭 구분 출력에서 특정 필드 추출 시 사용** ([[du]] `-sh` 결과 등)
	- 콜론·쉼표 구분 설정 파일 필드 추출 시 사용
	- 고정 폭 데이터의 문자 범위 추출 시 사용
- 특이사항
	- **기본 구분자는 탭** → 공백 구분 데이터는 미동작, `-d' '` 명시 필요
	- **연속 공백을 단일 구분자로 처리 안 함** → 공백 정렬 출력에는 부적합
		- 대응: [[awk]] 사용 → 연속 공백 자동 축약 처리
	- 필드 순서 재배치 불가 → `-f2,1` 지정해도 원본 순서 유지, [[awk]] 필요
	- 어원: **cut** (절단)

### 옵션
- `-f <n>` : n번째 필드 추출 (**f**ield)
- `-d '<구분자>'` : 필드 구분자 지정 (**d**elimiter) — 기본 탭
- `-c <범위>` : 문자 위치 기준 추출 (**c**haracter)

---

## 연관 명령어
- [[ls]] : `-l` 로 권한·크기 개괄 확인 — 이식성 높은 대안
- [[file]] : 파일 유형·인코딩 판정 — `stat` 은 메타데이터 전용
- [[du]] : `du -sh | cut -f1` 용량만 추출 패턴
- [[file-ops]] : `chmod` 등 변경 전후 권한 확인
- [[awk]] : 연속 공백·필드 재배치 처리 — `cut` 한계 보완
- [[join]] : `cut` 으로 추출한 키 필드로 두 파일 결합
