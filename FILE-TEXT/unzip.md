---
command: unzip
category: FILE-TEXT
aliases: [tar]
tags:
  - linux/text
  - task/inspect
  - task/search
  - topic/filesystem
  - privilege/mixed
related: ["[[tar]]", "[[grep]]", "[[file]]", "[[find]]", "[[cat]]"]
distro: 전체
verified: macOS (Darwin 25.5)
updated: 2026-07-30
---

# unzip

- ZIP 형식 아카이브 조회·해제 도구
- 어원: **un**+**zip** (압축 해제)
- 조회는 일반 사용자 가능, 해제 대상 경로 쓰기 권한 필요

---

## unzip 목록 조회 (`-l`)

```bash
unzip -l <아카이브>

# Examples
unzip -l spring-webflux-6.2.19.jar | grep -i "DispatcherHandler|NoResourceFound"   # JAR 내 클래스 존재 확인
unzip -l spring-cloud-gateway-server-webflux-4.3.0.jar | grep -i "handler/.*Handler" | head -20
unzip -l "$JAR" | grep -i "DataJpaTest|jpa/.*Test" | head        # 테스트 클래스 포함 여부
unzip -l "$j" 2>/dev/null | grep -o "[A-Za-z/]*DataJpaTest.class"  # 클래스 경로만 추출
```

### 명령어 설명
- 사용 목적
	- JAR 의존성에 특정 클래스 포함 여부 확인 시 사용 (해제 없이 조회)
	- 아카이브 구성 파일 목록 확인 시 사용
	- 라이브러리 버전별 클래스 존재 차이 확인 시 사용
- 특이사항
	- **JAR·WAR 은 ZIP 형식 → `unzip` 으로 직접 조회 가능** (해제 불필요)
	- 조회 전용이므로 디스크 변경 없음 → 안전
	- 출력에 크기·시각 컬럼 포함 → 경로만 필요 시 [[grep]] `-o` 또는 [[awk]] 추출

### 옵션
- `-l` : 내용 목록만 출력 (**l**ist)

---

## unzip 표준출력 (`-p`) / 선택 해제 (`-o -q`)

```bash
unzip -p <아카이브> '<내부경로>'            # 해제 없이 단일 파일 내용 출력
unzip -o -q <아카이브> '<내부경로>'         # 지정 파일만 해제 (조용히·덮어씀)

# Examples
unzip -p querydsl-core-5.1.0-sources.jar 'com/querydsl/core/types/dsl/DateTimeExpression.java' \
  | grep -n "public|plus|minus" | head -40                       # 소스 JAR 내 API 시그니처 확인
unzip -p querydsl-core-5.1.0-sources.jar 'com/querydsl/core/types/dsl/Expressions.java' \
  | grep -n "dateTimeOperation|dateTimeTemplate"                 # 메서드 존재 확인
unzip -o -q spring-webflux-6.2.19-sources.jar \
  'org/springframework/web/reactive/DispatcherHandler.java'      # 필요 소스만 해제
unzip -o -q spring-cloud-gateway-server-webflux-4.3.0-sources.jar 2>/dev/null   # 전체 조용히 해제
```

### 명령어 설명
- 사용 목적
	- **sources JAR 에서 라이브러리 실제 구현 확인 시 사용** (디컴파일 불필요)
	- 프레임워크 내부 동작 원인 추적 시 사용
	- 특정 파일만 필요 시 전체 해제 회피 목적
- 특이사항
	- **`-p` 는 디스크에 쓰지 않고 표준출력** → 파이프 검색에 최적, 작업 디렉터리 오염 없음
	- **`-o` 는 확인 없이 기존 파일 덮어씀** ⚠ → 작업 디렉터리에서 사용 시 주의
		- 미지정 시 대화형 프롬프트 발생 → 자동화 환경 정지
	- 내부 경로는 정확 일치 필요 → 사전 `-l` + [[grep]] 로 경로 확인 권장
	- 내부 경로에 글롭 사용 시 셸 확장 방지를 위해 단일 인용부호 필수
	- 클래스 파일(`.class`)은 바이너리 → 소스는 `-sources.jar` 별도 조회 필요

### 옵션
- `-p` : 내용을 표준출력으로 (**p**ipe) — 디스크 미변경
- `-o` : 확인 없이 덮어쓰기 (**o**verwrite) ⚠
- `-q` : 진행 메시지 억제 (**q**uiet)
- `-d <경로>` : 해제 대상 디렉터리 지정 (**d**estination) ※ 미검증

---

## 연관 명령어
- [[tar]] : `tar.gz` 아카이브 조회·해제 — 배포 산출물은 주로 tar 형식
- [[grep]] : 아카이브 목록·내용 필터링
- [[file]] : 아카이브 형식 사전 판정
- [[find]] : 해제 후 파일 탐색
- [[cat]] : 해제한 소스 파일 조회
