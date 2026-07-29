---
command: xargs
category: FILE-TEXT
aliases: []
tags:
  - linux/text
  - task/search
  - task/inspect
  - topic/filesystem
  - privilege/mixed
  - danger/data-loss
related: ["[[find]]", "[[basename]]", "[[lsof]]", "[[kill]]", "[[grep]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# xargs

- 표준입력을 명령 인자로 변환 실행하는 도구
- 어원: e**x**tended **args** (인자 확장)
- 실행 명령의 권한 요건에 종속

---

## xargs

```bash
<명령> | xargs [옵션] <실행할명령>

# Examples
find <dir> -type f | xargs -I{} basename {}          # 각 항목을 지정 위치에 치환
find . -name "*.jsonl" -print0 | xargs -0 cat        # NULL 구분자 (공백 안전)
lsof -ti:8080 | xargs -r kill 2>/dev/null            # 포트 점유 프로세스 종료
unzip -l "$j" | grep -o "[A-Za-z/]*Test.class" | xargs -n1 echo   # 항목별 개별 실행
```

### 명령어 설명
- 사용 목적
	- [[find]] 결과를 다른 명령 인자로 전달 시 사용
	- 파이프로 전달된 PID·파일명을 명령 인자로 변환 시 사용
	- 인자 위치가 말미가 아닌 명령 실행 시 사용 (`-I{}`)
- 특이사항
	- **공백·개행 포함 파일명에서 인자 분리 사고 발생** → `find -print0` + `xargs -0` 조합 필수
	- **입력이 비어도 명령 1회 실행** → 인자 없는 위험 명령 오작동 발생
		- 대응: `-r`(GNU) 로 빈 입력 시 미실행
		- macOS(BSD xargs)는 빈 입력 시 기본 미실행 → `-r` 불필요·미지원
	- `-I{}` 지정 시 **항목별 1회 실행** → 대량 입력 시 성능 저하, 일괄 전달은 `-I` 미사용
	- `rm`·`kill` 등 파괴적 명령 결합 시 **선행 `echo` 로 대상 확인 권장**
		- 예: `... | xargs echo rm` 로 대상 검증 후 `echo` 제거

### 옵션
- `-I {}` : 치환 문자열 지정, 항목별 1회 실행 (**I**nsert/replace)
- `-0` : NULL 문자 구분자 사용 (**0** = `\0`) — `find -print0` 대응
- `-r` : 입력 없으면 미실행 (**r**un-if-empty 부정, GNU 전용)
- `-n <n>` : 1회 실행당 인자 개수 제한 (**n**umber)
- `-t` : 실행 명령을 stderr 로 출력 (**t**race) ※ 미검증
- `-P <n>` : 병렬 실행 프로세스 수 (**P**arallel) ※ 미검증

---

## 연관 명령어
- [[find]] : 주요 입력 공급원 — `-print0` 조합 필수
- [[basename]] : `xargs -I{} basename {}` 파일명 추출 패턴
- [[lsof]] : `lsof -ti:포트 | xargs kill` 포트 점유 해제
- [[kill]] : `xargs` 로 PID 전달 대상 ⚠ 대상 확인 선행
- [[grep]] : `xargs grep` 다중 파일 검색
