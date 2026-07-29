---
command: curl
category: NETWORK-MANAGEMENT
aliases: [wget]
tags:
  - linux/network
  - task/verify
  - task/diagnose
  - topic/port
  - topic/remote-access
  - topic/troubleshooting
  - privilege/user
  - requires/network
related: ["[[ss]]", "[[lsof]]", "[[jq]]", "[[ping]]", "[[timeout]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# curl

- URL 기반 데이터 송수신 도구
- 어원: **c**lient **URL** (see **URL**)
- 일반 사용자 실행 가능

---

## curl 응답 조회 (GET)

```bash
curl -s <URL>

# Examples
curl -s http://localhost:8080/actuator/health 2>&1 | head -c 200         # 헬스체크 응답 확인
curl -s -o /dev/null -w "health: %{http_code}\n" http://localhost:8080/actuator/health   # 상태 코드만
curl -s -H "Authorization: Bearer $TOKEN" "$BASE" | python3 -m json.tool | head -20      # 인증 조회 + 정렬 출력
curl -s -H "Authorization: Bearer $TOKEN" "$BASE" | grep -E "boardId|name|pinned"        # 필드 필터
```

### 명령어 설명
- 사용 목적
	- 애플리케이션 기동·헬스체크 검증 시 사용 (`/actuator/health`)
	- API 응답 확인·회귀 검증 시 사용
	- 인증 토큰 기반 보호 엔드포인트 호출 시 사용
- 특이사항
	- **`-s` 미지정 시 진행률 표시가 stderr 로 출력** → 파이프·로그 오염 발생
	- **응답 본문 무시하고 상태 코드만 필요 시 `-o /dev/null -w '%{http_code}'`** 조합
		- `-w` 는 완료 후 포맷 문자열 출력 → `%{http_code}` `%{time_total}` 등
	- **기본적으로 HTTP 오류 상태(4xx·5xx)도 종료 코드 0** → 성공 오판 발생 ⚠
		- 대응: `-f`(실패 시 비정상 종료) 또는 `-w '%{http_code}'` 로 명시 판정
	- 응답이 JSON 이면 `python3 -m json.tool` 또는 [[jq]] 로 정형화
	- 연결 자체가 되지 않으면 종료 코드 7 → 포트 리스닝 여부는 [[lsof]]·[[ss]] 확인
	- 응답 지연 시 무한 대기 가능 → `--max-time` 또는 [[timeout]] 병용

### 옵션
- `-s` : 진행률·오류 메시지 억제 (**s**ilent)
- `-o <파일>` : 응답 본문을 파일로 저장 (**o**utput) — `/dev/null` 로 폐기
- `-w <포맷>` : 완료 후 지정 정보 출력 (**w**rite-out) — `%{http_code}` 등
- `-H <헤더>` : 요청 헤더 추가 (**H**eader) — 인증·Content-Type
- `-f` : HTTP 오류 시 비정상 종료 (**f**ail) ※ 미검증
- `-L` : 리다이렉트 추적 (**L**ocation) ※ 미검증
- `--max-time <초>` : 전체 소요 시간 상한 ※ 미검증

---

## curl 데이터 전송 (POST / PUT / PATCH)

```bash
curl -s -X <메서드> <URL> -H "Content-Type: application/json" -d '<JSON>'

# Examples
curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" -d '{"loginId":"...","password":"..."}'    # 토큰 취득
curl -s -X POST "$BASE" -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" -d '{"name":"..."}'                        # 리소스 생성
curl -s -X PATCH "$BASE/$BOARD" -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" -d '{"name":"..."}'                        # 부분 수정
curl -s -X PUT "$BASE/$BOARD/blocks/$BLOCK" -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" -d '{"startDate":"..."}'                   # 전체 교체
```

### 명령어 설명
- 사용 목적
	- API 엔드포인트 동작 검증 시 사용 (생성·수정 시나리오)
	- 인증 토큰 발급 후 후속 요청 연계 시 사용
	- 백엔드 변경 후 수동 회귀 확인 시 사용
- 특이사항
	- **`-d` 지정 시 메서드가 POST 로 자동 전환** → PUT·PATCH 는 `-X` 명시 필수
	- **`Content-Type: application/json` 미지정 시 서버가 본문 파싱 실패** → 415·400 발생
	- 셸 인용 주의 → JSON 내 `"` 보존을 위해 단일 인용부호로 감쌈
		- 본문에 셸 변수 주입 필요 시 이중 인용 + 이스케이프, 복잡하면 파일 사용(`-d @file.json`)
	- **인증 토큰·비밀번호가 셸 히스토리·프로세스 목록에 노출** ⚠
		- 환경변수 경유 권장 → `-H "Authorization: Bearer $TOKEN"`
		- 실제 자격증명은 문서·저장소에 기록 금지

### 옵션
- `-X <메서드>` : HTTP 메서드 지정 (**X** = request method)
- `-d <데이터>` : 요청 본문 전송 (**d**ata) — 지정 시 POST 자동 적용
- `-d @<파일>` : 파일 내용을 본문으로 전송 ※ 미검증

---

## 연관 명령어
- [[ss]] : 대상 포트 리스닝 여부 사전 확인 (Linux)
- [[lsof]] : 포트 점유 프로세스 확인 (macOS)
- [[jq]] : JSON 응답 정형화·필드 추출
- [[ping]] : 네트워크 계층 도달성 확인 — HTTP 실패 시 하위 계층 진단
- [[timeout]] : 응답 무한 대기 차단
