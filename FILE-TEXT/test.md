---
command: test
category: FILE-TEXT
aliases: ["[", seq, sleep, date]
tags:
  - linux/text
  - task/verify
  - task/inspect
  - topic/filesystem
  - topic/troubleshooting
  - privilege/user
related: ["[[env]]", "[[ls]]", "[[find]]", "[[timeout]]", "[[nohup]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# test / seq / sleep / date

- 스크립트 조건 판정·반복·대기·시각 취득 보조 명령군
- 단순 명령 → 단일 문서 통합
- 일반 사용자 실행 가능

---

## test / `[ ]`

```bash
test <조건>
[ <조건> ]

# Examples
test -f "$FILE" && echo "존재"           # 파일 존재 확인
[ -d "$DIR" ] || mkdir -p "$DIR"          # 디렉터리 부재 시 생성
[ -z "$VAR" ] && echo "미설정"            # 변수 비어있음 확인
[ "$COUNT" -eq 0 ] && echo "결과 없음"    # 숫자 비교
```

### 명령어 설명
- 사용 목적
	- 파일·디렉터리 존재 여부 판정 시 사용
	- 변수 설정 여부·값 비교 시 사용
	- 스크립트 분기 조건 구성 시 사용
- 특이사항
	- **`[` 는 명령어 → 대괄호 양쪽 공백 필수** → `[$X]` 는 문법 오류
	- **변수 인용부호 누락 시 값이 비어있으면 인자 소실로 문법 오류** ⚠
		- 대응: `[ -z "$VAR" ]` 형태로 항상 인용
	- 문자열 비교는 `=`, 숫자 비교는 `-eq`·`-lt`·`-gt` → **혼용 시 오판정**
	- bash 전용 `[[ ]]` 는 인용 생략 허용·패턴 매칭 지원 → 이식성은 `[ ]` 우위
	- 어원: **test** (조건 검사)

### 옵션
- `-f` : 일반 파일 존재 (**f**ile)
- `-d` : 디렉터리 존재 (**d**irectory)
- `-z` : 문자열 길이 0 (**z**ero)
- `-n` : 문자열 길이 0 초과 (**n**on-zero)
- `-e` : 경로 존재 (유형 무관) (**e**xists) ※ 미검증
- `-eq` / `-ne` / `-lt` / `-gt` : 숫자 비교 (**eq**ual / **n**ot **e**qual / **l**ess **t**han / **g**reater **t**han)

---

## seq

```bash
seq <시작> <끝>

# Examples
for i in $(seq 1 60); do ...; done      # 반복 횟수 지정
for i in $(seq 1 20); do ...; done      # 폴링 루프 상한
```

### 명령어 설명
- 사용 목적
	- 지정 횟수 반복 루프 구성 시 사용
	- 폴링 대기 시 시도 횟수 제한 시 사용
- 특이사항
	- bash 전용 `{1..60}` 확장이 더 경량 → 프로세스 생성 없음
		- 단, **중괄호 확장은 변수 사용 불가** → `{1..$N}` 미동작, `seq 1 $N` 필요
	- 대량 값 생성 시 메모리 사용 → 큰 범위는 `while` 루프 권장
	- 어원: **seq**uence (수열)

---

## sleep

```bash
sleep <초>

# Examples
sleep 5                                  # 기동 대기
nohup ./gradlew bootRun > /tmp/a.log 2>&1 & sleep 30; tail -8 /tmp/a.log   # 기동 후 로그 확인
```

### 명령어 설명
- 사용 목적
	- 백그라운드 기동 후 초기화 완료 대기 시 사용 ([[nohup]] 연계)
	- 폴링 루프 간격 확보 시 사용
	- 재시도 전 대기 시 사용
- 특이사항
	- **고정 대기는 환경 성능차에 취약** → 조건 폴링 루프가 안정적
		- 권장: `for i in $(seq 1 30); do <조건확인> && break; sleep 2; done`
	- 접미사 지정 가능 → `5s` `2m` `1h` (미지정 시 초)
	- 어원: **sleep** (수면 = 대기)

---

## date

```bash
date [+<포맷>]

# Examples
date                        # 현재 시각 (로케일 형식)
date +%Y%m%d                # 20260730 (파일명·디렉터리명 접미)
date +%s                    # Unix epoch 초 (경과시간 계산)
date -u +%s                 # UTC 기준 epoch
```

### 명령어 설명
- 사용 목적
	- 로그·백업 파일명에 날짜 부여 시 사용
	- 작업 경과 시간 측정 시 사용 (`+%s` 차분)
	- 현재 시각·시간대 확인 시 사용
- 특이사항
	- **날짜 연산 옵션이 GNU 와 BSD(macOS) 간 완전 상이** ⚠
		- GNU: `date -d "yesterday" +%Y%m%d`
		- BSD(macOS): `date -v-1d +%Y%m%d`
		- 이식성 필요 시 `python3` 등 사용 권장
	- `+%s` (epoch 초)는 양쪽 공통 → 경과시간 계산에 안전
	- 시간대는 시스템 설정 종속 → 고정 필요 시 `TZ=UTC date` 또는 `-u`
	- 시스템 시간대 설정은 [[localectl]] 참조
	- 어원: **date** (날짜)

### 옵션
- `+%Y%m%d` : 연월일 형식 출력
- `+%s` : Unix epoch 초
- `-u` : UTC 기준 (**U**TC)

---

## 연관 명령어
- [[env]] : `set -u` 와 `test -z` 병용으로 변수 검증
- [[ls]] : 존재 확인 대체 수단 — `test -f` 가 스크립트에 적합
- [[find]] : 조건 기반 파일 탐색 — `test` 는 단일 경로 판정
- [[timeout]] : 대기 상한 강제 — `sleep` 은 고정 대기
- [[nohup]] : 기동 후 `sleep` 대기 → 로그 확인 패턴
