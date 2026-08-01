---
title: Linux Command Vault — INDEX
type: MOC
tags:
  - moc/index
  - linux/reference
updated: 2026-08-01
---

# Linux Command Vault

- Claude 세션에서 실사용·검증한 리눅스 명령어 정리 저장소
- 검증 환경 기준 표기 → 미검증 정보 배제
- 작성 규칙: [[SKILL]] 참조

---

## 카테고리별 목록

### PACKAGE-MANAGEMENT (패키지 관리)
- [[dnf]] — RHEL 계열 패키지 매니저 (install / update / check / reinstall)
- [[dnf-group]] — 환경·그룹 단위 설치 (GUI 추가 등)
- [[rpm]] — 저수준 패키지 조회·무결성 검증

### NETWORK-MANAGEMENT (네트워크)
- [[nmcli]] — NetworkManager CLI, 고정 IP 설정
- [[nmtui]] — NetworkManager 대화형 TUI
- [[ip]] — 주소·라우팅·물리 링크 조회 (`NO-CARRIER` 판정)
- [[ping]] — 단계별 연결성 검증
- [[ss]] — 포트 리스닝 상태 조회
- [[ssh]] — 원격 접속 및 실패 원인 진단
- [[sshpass]] — 비대화형 비밀번호 전달 ⚠ 평문 노출 위험
- [[curl]] — HTTP 요청·응답 검증 (헬스체크·API·IMDSv2)
- [[hostnamectl]] — 호스트 이름 설정
- [[scp]] — SSH 채널 파일 복사 (`/root`는 `sudo cat` 스트림 우회)
- [[ssh-keygen]] — SSH 키 지문 조회·식별 (`-lf`)
- [[nc]] — 외부 관점 포트 도달 실검증 (보안그룹 차단 판정)
- [[whois]] — 공격 IP 소속·abuse 연락처 조회

### DISK-STORAGE (디스크·스토리지)
- [[lsblk]] — 블록 디바이스 트리, **장치명 확인 (파괴적 명령 전 필수)**
- [[df]] — 파일시스템 용량·사용률
- [[lvm]] — LVM 조회 및 온라인 확장 (`lvextend` + `xfs_growfs`)
- [[fdisk]] — 파티션 테이블·부트 플래그 조회
- [[parted]] — 파티션 조작·플래그 설정
- [[dd]] — MBR·파티션 헤더 초기화 ⚠ 파괴적
- [[mount]] — 마운트·재마운트 (레스큐 작업 기반)
- [[du]] — 디렉터리 단위 사용량 집계

### USER-PERMISSION (사용자·권한)
- [[usermod]] — 계정 그룹 변경 (`-aG wheel` = sudo 부여)
- [[passwd]] — 비밀번호 설정·상태 확인·재설정
- [[sudo]] — 권한 상승 실행 (`sudo sh -c` 리다이렉션·글로브 함정)
- [[id]] — UID·GID·그룹 소속 확인
- [[last]] — 로그인 이력 조회 (침해 시각 전후 대화형 접속 판정)

### SERVICE-SYSTEMD (서비스·systemd)
- [[systemctl]] — 서비스 제어, 부팅 타겟 전환
- [[journalctl]] — systemd 통합 로그 조회
- [[reboot]] — 시스템 재시작
- [[dmesg]] — 커널 링 버퍼 메시지 (하드웨어 인식 진단)
- [[crontab]] — cron 등록 작업·지속성 흔적 조사

### BOOT-RECOVERY (부팅·복구)
- [[rescue-mode]] — 레스큐 / `rd.break` 진입 절차
- [[chroot]] — 설치 시스템으로 루트 전환
- [[grub2-install]] — 부트로더 설치·설정 생성

### SECURITY-SELINUX (보안·SELinux)
- [[firewall-cmd]] — firewalld 방화벽 제어
- [[getenforce]] — SELinux 모드 조회·변경, autorelabel
- [[ausearch]] — SELinux 차단(AVC) 로그 조회
- [[iptables]] — 패킷 필터링 규칙 (C2 차단 `DOCKER-USER`/`OUTPUT`)

### SYSTEM-INFO (시스템 정보)
- [[localectl]] — 로케일·키맵·시간대 설정
- [[which]] — 명령 설치 여부·실제 경로 확인 (`type` / `command -v` 포함)
- [[env]] — 환경변수·셸 옵션 관리 (`export` / `source` / `set -euo pipefail`)

### PROCESS-MANAGEMENT (프로세스)
- [[ps]] — 프로세스 스냅샷 조회 (대괄호 트릭으로 grep 자기제외)
- [[pgrep]] — 패턴 기준 PID 조회 (`-f` 필수)
- [[pkill]] — 패턴 기준 종료 ⚠ 광범위 종료 위험
- [[kill]] — PID 기준 시그널 전송
- [[lsof]] — 포트 점유 프로세스 조회 (`-nP -iTCP:포트`)
- [[nohup]] — 세션 독립 백그라운드 기동
- [[timeout]] — 실행 시간 상한 (macOS 는 `gtimeout`)

### CONTAINER (컨테이너)
- [[docker]] — 컨테이너 실행·조회·조작 (`exec`/`inspect`/`logs`/`cp`/`stats`)

### DATABASE (데이터베이스)
- [[psql]] — PostgreSQL 클라이언트 (`pg_dump`/`pg_restore` 포함, `COPY FROM PROGRAM` RCE)
- [[redis-cli]] — Redis 클라이언트 (노출 침해 조사, `backup1`~`4` 페이로드)

### CLOUD-CLI (클라우드 CLI)
- [[aws-ec2]] — AWS EC2 사고 대응 (스냅샷·보안그룹·IMDSv2)

### FILE-TEXT (파일·텍스트)
- [[grep]] — 패턴 검색 (`-a` = 비UTF-8 인코딩 대응)
- [[find]] — 파일명·조건 기준 재귀 탐색
- [[sed]] — 범위 추출·치환 (`-i` 는 macOS/GNU 문법 상이)
- [[awk]] — 필드 단위 처리·블록 추출 (`NR==FNR` 대조)
- [[head]] — 파일 선두 조회·출력량 제한
- [[tail]] — 파일 말미 조회 (`-f` 는 자동화 무한대기 주의)
- [[cat]] — 파일 전체 출력 및 히어독 생성
- [[ls]] — 디렉터리 목록·속성 조회
- [[sort]] — 정렬 (`comm`·`join`·`uniq` 선행 필수)
- [[uniq]] — 중복 제거·빈도 집계
- [[comm]] — 두 파일 차집합·교집합 (스키마 대조)
- [[join]] — 키 기준 결합 (외부 조인 `-a1 -a2`)
- [[diff]] — 두 파일 차이 (프로세스 치환 정규화 패턴)
- [[wc]] — 행·단어·바이트 집계 (macOS 패딩 주의)
- [[tr]] — 문자 치환·제어문자 제거
- [[xargs]] — 표준입력을 명령 인자로 변환 (`-print0` + `-0`)
- [[iconv]] — 문자 인코딩 변환 (EUC-KR 소스 조회)
- [[file]] — 파일 유형·인코딩 판정
- [[jq]] — JSON 질의·검증
- [[unzip]] — JAR·ZIP 조회 (`-p` 로 sources JAR 내부 확인)
- [[tar]] — tar.gz 아카이브 조회·생성 (`tzf`/`czf`, `--exclude`)
- [[stat]] — 파일 메타데이터 조회 (`cut` 포함, `Modify`/`Birth` 변조 판정)
- [[sha256sum]] — 파일 해시 산출·검증 (IOC, macOS는 `shasum -a 256`)
- [[zip]] — ZIP 암호화 압축 (악성 샘플 격리)
- [[basename]] — 경로 분해 (`dirname` / `realpath` 포함)
- [[test]] — 조건 판정 (`seq` / `sleep` / `date` 포함)
- [[file-ops]] — 파일 조작 통합 (`mkdir` / `cp` / `mv` / `rm` / `touch` / `ln` / `chmod`) ⚠

---

## 작업 시나리오별 경로

### OS 설치 직후 초기 설정
1. [[dnf]] `check` · [[rpm]] `-Va` — 설치 무결성 검증
2. [[lsblk]] · [[df]] — 파티션 확인
3. [[ip]] `-br link` — 물리 링크 확인
4. [[nmcli]] 또는 [[nmtui]] — 고정 IP 설정
5. [[hostnamectl]] — 호스트명 설정
6. [[ping]] — 단계별 연결성 검증
7. [[usermod]] `-aG wheel` → [[id]] → [[sudo]] — 권한 부여·검증
8. [[dnf]] `update` · `install` — 패키지 설치
9. [[ssh]] · [[ss]] · [[firewall-cmd]] — 원격 접속 확인
10. [[reboot]] — 재부팅 후 설정 유지 검증

### 부팅 실패 복구
1. [[rescue-mode]] — 레스큐 환경 진입
2. [[lsblk]] — 장치명 확인 (USB는 `RM=1`)
3. [[chroot]] — 설치 시스템 진입
4. [[grub2-install]] · `grub2-mkconfig` — 부트로더 복구
5. [[reboot]] — USB 제거 후 재부팅

### 로그인 실패 복구
1. [[rescue-mode]] — GRUB `rd.break enforcing=0`
2. [[mount]] `-o remount,rw /sysroot`
3. [[chroot]] `/sysroot`
4. [[passwd]] — 비밀번호 재설정
5. [[getenforce]] — `touch /.autorelabel` 필수

### 서비스 접근 불가 진단
1. [[systemctl]] `is-active` — 서비스 기동 여부
2. [[ss]] `-tlnp` — 포트 리스닝 여부
3. [[firewall-cmd]] `--list-all` — 방화벽 허용 여부
4. [[ping]] — 네트워크 도달 여부
5. [[getenforce]] · [[ausearch]] — SELinux 차단 여부
6. [[journalctl]] — 서비스 로그 상세

### 디스크 용량 부족 대응
1. [[df]] `-h` — 부족 파티션 식별
2. [[lvm]] `vgs` — 볼륨그룹 여유 공간 확인
3. [[du]] `-sh` → [[sort]] `-h` — 원인 디렉터리 추적
4. [[lvm]] `vgs` → `lvextend` → `xfs_growfs` — 온라인 확장

### 애플리케이션 기동·검증·종료 (개발 루프)
1. [[lsof]] `-ti:8080` — 포트 선점 여부 확인
2. [[pkill]] `-f` — 잔존 프로세스 정리 (사전 [[pgrep]] 확인 필수)
3. [[nohup]] `... > log 2>&1 &` — 백그라운드 기동
4. [[test]] `sleep` / [[timeout]] — 초기화 대기
5. [[tail]] — 기동 로그 확인
6. [[pgrep]] `-f` — 프로세스 생존 확인
7. [[curl]] `/actuator/health` — 응답 검증
8. [[pkill]] `-f` — 검증 후 종료

### 포트 점유 충돌 해소 (`Address already in use`)
1. [[lsof]] `-nP -iTCP:<포트> -sTCP:LISTEN` — 점유 프로세스 식별 (macOS)
2. [[ss]] `-tlnp` — 동일 확인 (Linux)
3. [[ps]] — 해당 PID 상세 확인
4. [[kill]] 또는 [[pkill]] — 종료 (`lsof -ti:포트 | xargs kill`)
5. [[lsof]] 재확인 — 해제 검증

### 코드 정의 ↔ DB 스키마 대조
1. [[grep]] / [[find]] — 코드 측 정의 파일 수집
2. [[sed]] / [[awk]] — DDL 블록·컬럼 목록 추출
3. [[sort]] — 양측 목록 정렬 (선행 필수)
4. [[comm]] `-23` / `-13` / `-12` — 누락·잉여·공통 항목 분리
5. [[join]] `-t'|' -a1 -a2` — 타입 값까지 대조
6. [[diff]] — 정규화 후 정의 본문 비교

### 레거시 소스 조사 (EUC-KR·바이너리 혼재)
1. [[file]] — 인코딩·유형 판정
2. [[iconv]] `-f EUC-KR -t UTF-8` — UTF-8 변환
3. [[tr]] `-d '\000'` — NUL·제어문자 제거
4. [[grep]] / [[sed]] — 변환 결과 검색·범위 추출

### 의존성 라이브러리 내부 확인
1. [[find]] — 로컬 저장소(`~/.m2`) JAR 탐색
2. [[unzip]] `-l` — 클래스 포함 여부 확인
3. [[unzip]] `-p` — sources JAR 내부 구현 조회 (해제 불필요)
4. [[grep]] `-n` — 시그니처·메서드 위치 확인

### 컨테이너 침해 사고 대응 (RCE·채굴)
1. [[docker]] `ps -a` / `stats` — 노출 포트·CPU 이상(채굴) 식별
2. [[stat]] / [[find]] `-newermt` — 변조 시각·침해 산출물 파일 탐색
3. [[last]] / [[journalctl]] — 대화형 로그인·인증 경로 배제
4. [[psql]] / [[redis-cli]] — 침해 DB 객체·페이로드 조사
5. [[ps]] `--sort=-%cpu` / `/proc/<PID>/exe` — 위장 프로세스·실행 경로 확정
6. [[file]] / [[sha256sum]] — 악성 샘플 형식 판정·IOC 해시
7. [[docker]] `stop` + [[iptables]] `-I` — 컨테이너 격리·C2 차단
8. [[psql]] `pg_dump` → [[tar]] `czf` → [[sha256sum]] → [[scp]] — 증거 보존·무결성 반출
9. [[nc]] / [[whois]] / [[aws-ec2]] — 외부 노출 실검증·공격 IP 소속·스냅샷

---

## 주요 태그

### 위험도
- `#danger/destructive` — 데이터 손실 가능 → [[dd]], [[fdisk]], [[parted]], [[file-ops]], [[pkill]], [[kill]]
- `#danger/irreversible` — 복구 불가 → [[dd]]
- `#danger/data-loss` — 옵션 오용 시 손실 → [[usermod]], [[sed]], [[xargs]], [[tar]], [[sshpass]]

### 작업 유형
- `#task/inspect` `#task/diagnose` — 조회·진단
- `#task/configure` — 설정 변경
- `#task/recovery` — 복구 작업
- `#task/install` `#task/expand` — 설치·확장
- `#task/search` — 파일·내용 검색 → [[grep]], [[find]], [[sed]], [[awk]]
- `#task/verify` — 결과 검증·대조 → [[diff]], [[comm]], [[join]], [[curl]]
- `#task/restart` — 프로세스 종료·재기동 → [[pkill]], [[kill]], [[nohup]]

### 권한
- `#privilege/root` — root 필요
- `#privilege/user` — 일반 사용자 가능
- `#privilege/mixed` — 조회는 일반, 변경은 root

### 주제
- `#topic/selinux` `#topic/firewall` `#topic/bootloader` `#topic/lvm`
- `#topic/static-ip` `#topic/sudo` `#topic/partition` `#topic/rescue`
- `#topic/encoding` — 문자 인코딩 → [[iconv]], [[tr]], [[file]], [[grep]]
- `#topic/regex` — 정규표현식 → [[grep]], [[sed]], [[awk]], [[find]]
- `#topic/filesystem` — 파일·경로 → [[find]], [[ls]], [[stat]], [[file-ops]]
- `#topic/port` `#topic/socket` — 포트·소켓 → [[lsof]], [[ss]], [[curl]], [[nc]]
- `#topic/security` — 침해·보안 → [[iptables]], [[whois]], [[last]], [[sha256sum]], [[psql]], [[redis-cli]]
- `#topic/troubleshooting` — 문제 진단 → 다수

### 플랫폼 차이 주의 문서
- **GNU(Linux) ↔ BSD(macOS) 문법·동작 상이** → 이식성 검토 필수
	- [[sed]] `-i` (빈 인자 유무) · [[stat]] `-c` ↔ `-f` · [[test]] `date` 연산 옵션
	- [[wc]] 출력 공백 패딩 · [[pgrep]] `-a` 미지원 · [[timeout]] ↔ `gtimeout`
	- [[basename]] `readlink -f` 미지원 · [[xargs]] `-r` 미지원

---

## 관련 문서
- [[SKILL]] — 문서 작성·갱신 규칙
