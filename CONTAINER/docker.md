---
command: docker
category: CONTAINER
aliases: [docker-cli]
tags:
  - linux/process
  - task/inspect
  - task/diagnose
  - topic/troubleshooting
  - topic/socket
  - privilege/mixed
related: ["[[psql]]", "[[redis-cli]]", "[[ps]]", "[[iptables]]", "[[stat]]", "[[find]]", "[[tar]]"]
distro: 전체 (docker-ce / Docker Engine)
verified: Rocky Linux 9.6 (사고 대응 세션)
updated: 2026-08-01
---

# docker

- 컨테이너 실행·조회·조작 CLI
- 어원: 부두 노동자(docker) — 컨테이너 적재 은유
- `docker` 그룹 소속 사용자는 실질 호스트 root 동등 → 접근 통제 재검토 대상

---

## docker ps

```bash
docker ps -a

# Examples
docker ps -a                          # 정지 포함 전체 컨테이너
docker ps                             # 실행 중만
```

### 명령어 설명
- 사용 목적
	- 전체 컨테이너 상태·포트 노출·이미지 확인 시 사용
	- 침해·장애 시 노출 포트(`0.0.0.0:5432`) 식별 시 사용
- 특이사항
	- `PORTS`에 `0.0.0.0:5432->5432/tcp` → 전체 대역 공개 (침입 경로)
	- `expose`만 설정 시 `PORTS`에 `0.0.0.0` 미표시 → 외부 비노출
	- `STATUS`의 `unhealthy` → 헬스체크 실패 고착 상태

### 옵션
- `-a` : 정지 컨테이너 포함 (**a**ll)

---

## docker exec

```bash
docker exec <컨테이너ID> <명령>

# Examples
docker exec aaf032465d9b stat /var/lib/postgresql/data/pg_hba.conf   # 컨테이너 내 파일 조회
docker exec aaf032465d9b sh -c "grep -vE '^\s*#|^\s*$' .../pg_hba.conf"  # 주석 제외 확인
docker exec aaf032465d9b psql -U developer -d postgres -c "\dt"      # 컨테이너 내 DB 조회
docker exec 4c8c56cc07f0 redis-cli --scan --pattern '*'              # Redis 키 조회
docker exec aaf032465d9b ps auxww                                    # 컨테이너 내부 프로세스
```

### 명령어 설명
- 사용 목적
	- 실행 중 컨테이너 내부에서 명령 실행 시 사용
	- 조사 대상 컨테이너 특정 후 내부 상태 확인 시 사용
- 특이사항
	- **컨테이너 ID 오지정 시 대상 불일치** → 사전에 `docker ps`로 ID 확인 필수
		- 예: 애플리케이션 컨테이너에 `stat pg_hba.conf` → `No such file or directory`
	- 복합 명령·글로브·파이프는 `sh -c '...'` 로 감싸 컨테이너 셸이 처리
	- 컨테이너 내부에서 `/proc/*/exe` 조회 제한 → 호스트측 조회로 전환 필요

---

## docker inspect

```bash
docker inspect <컨테이너ID> --format '<Go 템플릿>'

# Examples
docker inspect aaf032465d9b --format '{{json .Mounts}}' | python3 -m json.tool   # 마운트 구조
docker inspect aaf032465d9b --format 'Started: {{.State.StartedAt}} Restarts: {{.RestartCount}}'
docker inspect aaf032465d9b --format 'Priv={{.HostConfig.Privileged}} Caps={{.HostConfig.CapAdd}} PID={{.HostConfig.PidMode}}'
docker inspect aaf032465d9b 4c8c56cc07f0 > inspect.json                          # 다수 컨테이너 구성 덤프
```

### 명령어 설명
- 사용 목적
	- 마운트·기동 이력·권한·환경변수 등 컨테이너 구성 조회 시 사용
	- 침해 범위 판정 근거(Privileged 여부, `docker.sock` 마운트 여부) 확보 시 사용
- 특이사항
	- `Type: bind`, `RW: true` → 호스트 디렉터리 직접 쓰기 가능 구조 (악성 바이너리 기록 경로)
	- `Privileged=false` + `docker.sock` 미마운트 → 컨테이너 탈출 수단 부재 판정 근거
	- `.Config.Env`에 `POSTGRES_PASSWORD` 등 노출 가능 → 자격증명 교체 전제 대응
	- `--format` Go 템플릿에서 `{{json .필드}}` 로 하위 구조 JSON 출력

### 옵션
- `--format` : Go 템플릿으로 출력 형식 지정 (**format**)

---

## docker logs

```bash
docker logs <컨테이너ID>

# Examples
docker logs aaf032465d9b > pg-full.log 2>&1                                       # 전체 로그 파일화
docker logs --since 2026-07-30T00:00:00 --until 2026-07-30T20:30:00 aaf032465d9b  # 기간 지정
docker logs --since 2026-07-30T00:00:00 aaf032465d9b 2>&1 | grep -Ei 'FATAL'      # 실패 로그 필터
```

### 명령어 설명
- 사용 목적
	- 컨테이너 표준출력·표준에러 로그 조회 시 사용
	- 인증 실패·사전 공격 이력 등 애플리케이션 로그 분석 시 사용
- 특이사항
	- **`docker logs`는 컨테이너 삭제 시 소멸** → 재구축 전 파일 수집 필수
	- 로그 시각은 컨테이너 타임존 기준. 내부 `ls`는 UTC인 경우 다수 → 환산 주의
	- `--since`/`--until`는 RFC3339 또는 상대시간 지정
	- `sudo` 필요 시 리다이렉션은 `sudo sh -c '... > 파일'` 로 감쌈 → [[sudo]]

### 옵션
- `--since` : 시작 시각 이후 로그만 (**since**)
- `--until` : 종료 시각 이전 로그만 (**until**)

---

## docker cp

```bash
docker cp <컨테이너ID>:<경로> <호스트경로>

# Examples
docker cp aaf032465d9b:/var/tmp/zsTCX /tmp/zsTCX.sample     # 악성 샘플 반출
docker cp aaf032465d9b:/tmp/bandage.dump /home/rocky/       # DB 덤프 반출
```

### 명령어 설명
- 사용 목적
	- 컨테이너 ↔ 호스트 파일 복사 시 사용
	- 악성 샘플·백업 파일 반출 시 사용
- 특이사항
	- **목적지 기록은 실행 사용자 권한으로 처리** → `/root` 지정 시 `permission denied`
		- 대응: 홈 디렉터리 경유 후 `sudo mv` 로 격리 이동
	- 반출한 악성 샘플은 `chmod 000` 으로 실수 실행 방지 → [[stat]]

---

## docker stats

```bash
docker stats --no-stream

# Examples
docker stats --no-stream               # 전체 컨테이너 리소스 1회 스냅샷
```

### 명령어 설명
- 사용 목적
	- 컨테이너별 CPU·메모리·네트워크 사용량 확인 시 사용
	- 채굴 등 CPU 집약 감염 위치 특정 시 사용
- 특이사항
	- CPU 189% + 낮은 메모리 → CPU 집약 작업(채굴) 특징
	- 정지 후 재실행 시 대상 목록 소멸 → 채굴 중단 확인 근거
	- `--no-stream` 미지정 시 실시간 갱신 → 스크립트에서는 필수

### 옵션
- `--no-stream` : 1회 출력 후 종료 (**no-stream**)

---

## docker stop / rm / port

```bash
docker stop <컨테이너ID>...
docker rm <컨테이너ID>...
docker port <컨테이너ID>

# Examples
docker stop aaf032465d9b 4c8c56cc07f0     # 채굴·페이로드 컨테이너 동시 격리
docker rm aaf032465d9b 4c8c56cc07f0       # 재구축 시 삭제
docker port aaf032465d9b                  # 매핑된 포트 확인
```

### 명령어 설명
- 사용 목적
	- 감염 컨테이너 격리(`stop`) 시 사용
	- 재구축 시 삭제(`rm`) 시 사용
- 특이사항
	- `stop`은 데이터(bind mount·named volume) 유지 → 논리 백업 선확보 후 수행 권장
	- **RCE 발생 컨테이너는 정리 아닌 폐기 대상** → 삭제 후 이미지 재취득·볼륨 재구축
	- 데이터 디렉터리는 삭제 아닌 개명 보관 후 재사용 금지

---

## 연관 명령어
- [[psql]] : 컨테이너 내 PostgreSQL 조회 (`docker exec ... psql`)
- [[redis-cli]] : 컨테이너 내 Redis 조회
- [[ps]] : 호스트↔컨테이너 PID 대응 확인
- [[iptables]] : 컨테이너 트래픽 차단 (`DOCKER-USER` 체인)
- [[stat]] : 컨테이너 내 파일 메타데이터 조회
- [[find]] : 컨테이너 쓰기 레이어(`snapshots/*/fs`) 변경 파일 탐색
- [[tar]] : 증거 아카이브 생성
