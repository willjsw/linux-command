---
command: sshpass
category: NETWORK-MANAGEMENT
aliases: []
tags:
  - linux/network
  - linux/security
  - task/connect
  - task/configure
  - topic/remote-access
  - topic/password
  - topic/security
  - privilege/user
  - requires/network
  - danger/data-loss
related: ["[[ssh]]", "[[curl]]", "[[timeout]]", "[[tar]]", "[[pgrep]]"]
distro: 전체 (EPEL 저장소 필요)
verified: Rocky Linux 9.6 (원격), macOS (Darwin 25.5, 클라이언트)
updated: 2026-07-30
---

# sshpass

- SSH 비밀번호를 비대화형으로 전달하는 실행 래퍼
- 어원: **ssh** + **pass**word
- 일반 사용자 실행 가능, EPEL 저장소 설치 필요

---

## sshpass

```bash
sshpass -p '<비밀번호>' ssh [옵션] <계정>@<호스트> '<원격명령>'
sshpass -e ssh <계정>@<호스트>          # 환경변수 SSHPASS 경유 (권장)

# Examples
export SSHPASS='<비밀번호>'                                     # 히스토리 노출 회피
sshpass -e ssh -p 42222 -o StrictHostKeyChecking=no \
  -o ConnectTimeout=10 <계정>@<호스트> 'echo "=== connected ==="'   # 접속 확인

sshpass -e ssh -p 42222 -o StrictHostKeyChecking=no <계정>@<호스트> \
  'cd /home/<계정>/stat-job/jobs/stat-job-runner || exit 99'      # 원격 경로 검증

sshpass -e ssh -p 42222 -o StrictHostKeyChecking=no <계정>@<호스트> \
  'pgrep -af stat-scheduler || echo "none running"'               # 원격 프로세스 조회

sshpass -e ssh -p 42222 -o StrictHostKeyChecking=no <계정>@<호스트> \
  'tar tzf /home/<계정>/stat-job/jobs/stat-job-runner-5.0.0.tar.gz | head -20'   # 원격 아카이브 확인
```

### 명령어 설명
- 사용 목적
	- 키 인증 미설정 서버에 비대화형 접속 시 사용 (자동화 스크립트)
	- 원격 배포 상태·프로세스 일괄 점검 시 사용
	- 원격 명령 결과를 로컬 파이프로 수집 시 사용
- 특이사항
	- **`-p` 인자 방식은 비밀번호가 프로세스 목록·셸 히스토리에 평문 노출** ⚠
		- [[ps]] `aux` 로 타 사용자가 조회 가능 → 다중 사용자 환경 사용 금지
		- 대응: `export SSHPASS='...'` 후 `-e` 사용 → 프로세스 인자에서 제거
		- 근본 대응: **SSH 키 인증 전환** → `sshpass` 자체 불필요
	- **비밀번호를 스크립트·문서·저장소에 기록 금지** → 환경변수·비밀 관리 도구 경유
	- `-o StrictHostKeyChecking=no` 는 호스트 키 검증 생략 → **중간자 공격 탐지 불가** ⚠
		- 신뢰 가능한 사내망 자동화에 한정, 공용망 사용 부적합
	- `-o ConnectTimeout=<초>` 미지정 시 무응답 호스트에서 장기 대기 → [[timeout]] 병용 검토
	- 원격 명령은 단일 인용부호로 감쌈 → 로컬 셸 변수 확장 방지
	- 원격 명령 내 `|| exit <코드>` 로 실패 원인 구분 가능
	- 비표준 포트는 `ssh` 의 `-p` 옵션 → `sshpass -p` 와 **의미 상이, 혼동 주의**

### 옵션
- `-p <비밀번호>` : 비밀번호 직접 지정 (**p**assword) ⚠ 프로세스 목록 노출
- `-e` : 환경변수 `SSHPASS` 에서 읽음 (**e**nvironment) — 노출 완화, 권장
- `-f <파일>` : 파일 첫 행에서 읽음 (**f**ile) ※ 미검증

---

## 연관 명령어
- [[ssh]] : 기반 접속 도구 — **키 인증 설정 시 `sshpass` 불필요**
- [[curl]] : HTTP 계층 원격 검증 — SSH 불필요 시 대체
- [[timeout]] : 접속 무한 대기 차단
- [[tar]] : 원격 배포 아카이브 내용 확인 대상
- [[pgrep]] : 원격 프로세스 기동 여부 조회 대상
