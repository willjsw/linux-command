---
command: psql
category: DATABASE
aliases: [pg_dump, pg_restore, postgres]
tags:
  - linux/process
  - task/inspect
  - task/diagnose
  - task/verify
  - topic/troubleshooting
  - privilege/user
related: ["[[docker]]", "[[redis-cli]]", "[[grep]]", "[[stat]]"]
distro: 전체 (postgresql-client)
verified: PostgreSQL 15.18 (컨테이너, 사고 대응 세션)
updated: 2026-08-01
---

# psql / pg_dump / pg_restore

- PostgreSQL 대화형 클라이언트(`psql`) 및 논리 백업·복원 도구(`pg_dump`/`pg_restore`)
- 어원: **p**ost**g**re**s** + **q**uery **l**anguage
- 컨테이너 내부는 `docker exec <ID> psql ...` 형태로 실행 → [[docker]]

---

## psql

```bash
psql -U <롤> -d <DB> -c "<SQL>"

# Examples
psql -U developer -d postgres -c "SELECT pg_conf_load_time();"          # 설정 재로딩 시각
psql -U developer -d postgres -c "SELECT oid, datname FROM pg_database ORDER BY oid;"
psql -U developer -d bandage -c "\dt"                                   # 테이블 목록
psql -U developer -d bandage -c "\d demo"                               # 테이블 구조
psql -U developer -d postgres -c "SELECT * FROM demo LIMIT 50;"
```

### 명령어 설명
- 사용 목적
	- 단일 SQL 비대화형 실행(`-c`) 시 사용
	- 신규 롤·테이블·확장 등 DB 객체 조사 시 사용
- 특이사항
	- **슈퍼유저명 미확인 시 `-U postgres` 오지정** → `role "postgres" does not exist`
		- 대응: 실제 롤명(`developer` 등) 지정. `pg_hba.conf` `reject`가 TCP 관리자 접속까지 차단해도 `local all all trust` 존치 시 컨테이너 내부 소켓 접속 가능
	- `OID > 16383` 객체는 `initdb` 이후 생성분 → 백도어 계정·객체 판별 기준
	- `COPY <테이블> FROM PROGRAM '<명령>'` → 슈퍼유저 롤의 임의 명령 실행 경로 (RCE)
		- 슈퍼유저 아닌 롤은 `COPY FROM PROGRAM` 불가 → 최소권한 롤이 피해 원천 차단

### 조사용 조회 (사고 대응)
- `SELECT oid, rolname, rolsuper, rolcanlogin FROM pg_roles WHERE oid > 16383;` : 신규 롤
- `SELECT c.oid, c.relname, c.relkind FROM pg_class c WHERE c.oid > 16383;` : 신규 객체
- `SELECT * FROM pg_extension;` : 설치 확장 (신뢰되지 않은 언어 `plpython3u` 등 확인)
- `SELECT oid, pg_get_userbyid(lomowner) FROM pg_largeobject_metadata;` : 라지오브젝트 경유 파일 기록 여부

### 옵션
- `-U` : 접속 롤 지정 (**U**ser)
- `-d` : 대상 데이터베이스 (**d**atabase)
- `-c` : 단일 명령 실행 후 종료 (**c**ommand)

---

## pg_dump

```bash
pg_dump -U <롤> -d <DB> -Fc -f <출력파일>

# Examples
pg_dump -U developer -d bandage -Fc -f /tmp/bandage.dump    # 압축 커스텀 포맷 논리 백업
```

### 명령어 설명
- 사용 목적
	- 단일 DB 논리 백업(복원 원본) 확보 시 사용
- 특이사항
	- `-Fc` 커스텀 포맷 → `pg_restore` 선택 복원 가능
	- 개인정보 포함 가능 → 증거 아카이브에서 분리 보관·전송 대상 구분
	- 침해 산출물이 있는 DB(예: `postgres`의 `demo`)는 복원 대상 제외

### 옵션
- `-F` : 출력 포맷 (`c`=custom 압축, `p`=plain SQL) (**F**ormat)
- `-f` : 출력 파일 (**f**ile)

---

## pg_restore

```bash
pg_restore -U <롤> -d <DB> <덤프파일>

# Examples
pg_restore -U bandage_app -d bandage /tmp/bandage.dump      # 커스텀 포맷 복원
```

### 명령어 설명
- 사용 목적
	- `pg_dump -Fc` 산출물 복원 시 사용
- 특이사항
	- 재구축 시 신규 최소권한 롤로 복원 → 슈퍼유저 롤 재사용 금지

---

## 연관 명령어
- [[docker]] : 컨테이너 내 psql 실행 (`docker exec ... psql`)
- [[redis-cli]] : 동일 사고의 2차 침해 대상 조회
- [[grep]] : `pg_hba.conf` 유효 규칙·로그 필터
- [[stat]] : `pg_hba.conf` 변조 시각(mtime) 확인
