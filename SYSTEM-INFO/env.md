---
command: env
category: SYSTEM-INFO
aliases: [export, source, set, alias]
tags:
  - linux/system
  - task/configure
  - task/inspect
  - topic/troubleshooting
  - topic/security
  - privilege/user
related: ["[[which]]", "[[sshpass]]", "[[curl]]", "[[localectl]]", "[[test]]", "[[ldd]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# env / export / source / set

- 환경변수·셸 옵션 관리 명령군
- `export`·`source`·`set` 은 셸 내장 → [[which]] 로 미탐지
- 일반 사용자 실행 가능

---

## export

```bash
export <변수>=<값>

# Examples
export SSHPASS='<비밀번호>'                 # 자격증명을 프로세스 인자에서 분리
export PGPASSWORD='<비밀번호>'               # psql 비대화형 인증
export TOKEN=$(curl -s -X POST .../login | jq -r .accessToken)   # 토큰 취득 후 재사용
export PATH="/opt/homebrew/bin:$PATH"        # 탐색 경로 추가
```

### 명령어 설명
- 사용 목적
	- **자격증명을 명령 인자에서 분리 시 사용** → 프로세스 목록 노출 방지
	- 하위 프로세스에 값 전달 시 사용 (자식 프로세스 상속)
	- 실행 경로(`PATH`) 확장 시 사용
- 특이사항
	- **`export` 없는 변수 대입은 현재 셸 한정** → 하위 프로세스 미상속
	- **환경변수 값은 `/proc/<PID>/environ` 등으로 조회 가능** ⚠ → 완전한 은닉 아님
		- 인자 노출([[ps]] 조회 가능)보다는 안전, 근본 대응은 키 인증·비밀 관리 도구
	- **자격증명을 스크립트·저장소에 하드코딩 금지** → 별도 주입 필요
	- 값에 공백·특수문자 포함 시 인용부호 필수
	- 셸 세션 종료 시 소멸 → 영구 적용은 프로파일 파일 기재 필요
	- 어원: **export** (외부로 내보냄 = 하위 프로세스 전달)

---

## source

```bash
source <파일>
. <파일>

# Examples
source .env                      # 환경변수 파일 적용
source ~/.zshrc                   # 셸 설정 재적용
```

### 명령어 설명
- 사용 목적
	- 환경변수 정의 파일을 현재 셸에 적용 시 사용
	- 셸 설정 변경 후 재로그인 없이 반영 시 사용
- 특이사항
	- **현재 셸에서 실행 → 변수·함수 정의가 현재 셸에 잔존**
		- `bash script.sh` 실행은 하위 셸 → 정의 소멸, 목적 상이
	- `.`(점) 은 동일 기능의 POSIX 표기
	- **신뢰할 수 없는 파일 `source` 금지** ⚠ → 임의 명령 실행 위험
	- 어원: **source** (원본 파일 내용을 현재 셸에 주입)

---

## env

```bash
env
env <변수>=<값> <명령>

# Examples
env | grep -i proxy                     # 특정 환경변수 확인
env LC_ALL=C sort file.txt              # 해당 명령에만 로케일 적용
```

### 명령어 설명
- 사용 목적
	- 현재 환경변수 전체 조회 시 사용
	- **단일 명령에만 환경변수 적용 시 사용** → 셸 환경 오염 방지
	- 로케일 고정으로 정렬·출력 재현성 확보 시 사용 ([[localectl]] 참조)
- 특이사항
	- `env <변수>=<값> <명령>` 은 해당 명령 실행 동안만 유효 → `export` 보다 부작용 적음
	- 스크립트 shebang 에 `#!/usr/bin/env bash` 사용 시 `PATH` 기반 인터프리터 탐색 → 이식성 향상
	- 어원: **env**ironment (환경)

---

## set

```bash
set -euo pipefail

# Examples
set -e                    # 명령 실패 시 즉시 중단
set -u                    # 미정의 변수 참조 시 오류
set -o pipefail           # 파이프 중간 실패도 전체 실패로 판정
set -x                    # 실행 명령 추적 출력 (디버깅)
```

### 명령어 설명
- 사용 목적
	- **스크립트 오류 조기 검출 시 사용** → 실패 무시로 인한 오작동 방지
	- 미정의 변수로 인한 의도 외 동작 차단 시 사용
	- 실행 흐름 디버깅 시 사용 (`-x`)
- 특이사항
	- **`set -e` 미지정 시 명령 실패에도 스크립트 계속 진행** → 파괴적 작업에서 위험 ⚠
		- 예: `cd /wrong/path` 실패 후 `rm -rf *` 실행 → 의도 외 디렉터리 삭제
	- **`set -u` 는 `rm -rf "$DIR"` 의 `$DIR` 미설정 사고 예방**
	- **`set -e` 만으로는 파이프 중간 실패 미검출** → `set -o pipefail` 병용 필수
		- 예: `curl ... | jq ...` 에서 `curl` 실패 시 `jq` 종료 코드만 반영
	- 표준 조합: `set -euo pipefail` → 스크립트 선두 배치
	- `set -e` 는 조건문·`||` 문맥에서 미적용 → 의도적 실패 허용 가능
	- 어원: **set** (셸 옵션 설정)

### 옵션
- `-e` : 명령 실패 시 즉시 종료 (**e**rror exit)
- `-u` : 미정의 변수 참조 시 오류 (**u**nset)
- `-o pipefail` : 파이프 내 실패를 전체 실패로 판정 (**o**ption)
- `-x` : 실행 명령 추적 출력 (e**x**ecution trace)

---

## 연관 명령어
- [[which]] : 셸 내장 명령은 미탐지 → `type` 사용 필요
- [[sshpass]] : `export SSHPASS` 로 비밀번호 노출 완화
- [[curl]] : `export TOKEN` 으로 인증 토큰 재사용
- [[localectl]] : 시스템 로케일 설정 — `env LC_ALL` 은 명령 단위 재정의
- [[test]] : 스크립트 조건 분기 — `set -u` 와 병용
- [[ldd]] : `LD_LIBRARY_PATH` 가 의존성 탐색 경로 결정
