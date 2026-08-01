---
command: jq
category: FILE-TEXT
aliases: []
tags:
  - linux/text
  - task/inspect
  - task/verify
  - task/search
  - topic/encoding
  - privilege/user
related: ["[[curl]]", "[[grep]]", "[[diff]]", "[[sort]]", "[[cat]]", "[[aws-ec2]]"]
distro: 전체 (별도 설치 필요)
verified: macOS (Darwin 25.5)
updated: 2026-08-01
---

# jq

- JSON 전용 질의·변환 도구
- 어원: **j**son **q**uery
- 일반 사용자 실행 가능, 기본 미포함 → 별도 설치 필요

---

## jq

```bash
jq '<필터>' <file>

# Examples
jq '.valid' result.json                                          # 단일 필드 추출
jq '{valid, diagramType, liveEditUrl}' result.json               # 다중 필드 객체 재구성
jq -r '.nodes[] | select(.type=="file") | .id' graph.json | sort  # 배열 필터 + 원문 출력
jq '[.nodes[] | select(.id == null or .id == "")] | length'  graph.json   # 결측 항목 개수
jq '[.nodes[].id] as $ids | [.edges[] | select((.source as $s | $ids|index($s)|not))] | length' graph.json
jq '{nodeCount: (.nodes|length), edgeCount: (.edges|length)}' graph.json  # 집계
jq -r '.nodes[] | select(.id | startswith("function:src/")) | .id' graph.json   # 접두 필터
jq '[.nodes[] | select(.id | test("audit"))] | length' graph.json # 정규식 매칭 개수
```

### 명령어 설명
- 사용 목적
	- JSON 응답·산출물에서 특정 필드 추출 시 사용 ([[curl]] 연계)
	- 배열 요소 조건 필터·개수 집계 시 사용 (데이터 무결성 검증)
	- 참조 정합성 검증 시 사용 (`edges` 의 `source`·`target` 이 `nodes` 에 존재하는지)
- 특이사항
	- **`-r` 미지정 시 문자열이 이중인용부호 포함 출력** → 파이프 후속 처리 시 방해
		- 후속 [[sort]]·[[grep]] 연계 시 `-r` 필수
	- 필터 문법 오류 시 종료 코드 3 → 파싱 실패와 구분 가능
	- **JSON 이 아닌 입력 시 파싱 오류** → 응답 본문이 HTML 오류 페이지인 경우 발생
	- `select(...)` 는 조건 참인 요소만 통과 → `map`·`length` 와 조합
	- `as $var` 로 변수 바인딩 → 배열 간 상호 참조 검증에 사용
	- 키 순서 정렬 비교는 `-S` 또는 `python3 -m json.tool --sort-keys` → [[diff]] 연계
	- 대용량 JSON 은 처리 지연 발생 → 필요 필드만 조기 축소 권장

### 옵션
- `-r` : 문자열을 인용부호 없이 원문 출력 (**r**aw output)
- `-S` : 객체 키 정렬 출력 (**S**ort keys) ※ 미검증
- `-c` : 한 행 압축 출력 (**c**ompact) ※ 미검증
- `-e` : 결과가 `null`·`false` 면 비정상 종료 (**e**xit status) ※ 미검증

---

## 연관 명령어
- [[curl]] : JSON API 응답을 `jq` 로 정형화·필터
- [[grep]] : 단순 문자열 검색 — 구조 인식 불가, 소량 확인 시 대체
- [[diff]] : `jq -S` 정렬 후 두 JSON 비교
- [[sort]] : `jq -r` 출력 정렬
- [[cat]] : JSON 원문 확인 — 구조 파악 후 `jq` 필터 설계
- [[aws-ec2]] : `aws --output json` 결과 후처리 (JMESPath `--query` 보완)
