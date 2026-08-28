---
title: 소프트웨어·패키지 관리
type: exam-theory
category: LINUX-MASTER
tags:
  - exam/linux-master
  - exam/theory
  - linux/package
  - linux/software
  - topic/package
  - topic/library
  - task/install
related: ["[[rpm]]", "[[dnf]]", "[[dnf-group]]", "[[tar]]", "[[ldd]]", "[[system-structure]]", "[[system-security]]"]
updated: 2026-08-28
---

# 소프트웨어·패키지 관리

- 리눅스마스터 1급 필기 대비 이론 — 패키지 관리·소스 컴파일·라이브러리·아카이브 4개 영역
- 계열별 명령 대응(RedHat rpm/dnf ↔ Debian dpkg/apt)과 rpm 질의/검증 옵션이 최빈출 → 옵션 표 단위 암기 필요

---

## 1. 패키지 관리 개요

### 1-1. 패키지 관리의 필요성

- 소프트웨어의 설치·업그레이드·삭제·질의를 표준 절차로 관리
- 파일 목록·버전·의존성 정보를 데이터베이스로 유지 → 일관된 상태 추적

### 1-2. 의존성 문제

- **의존성(dependency)**: 한 패키지가 동작하려면 다른 패키지·라이브러리가 먼저 설치돼 있어야 하는 관계
- 저수준 도구(`rpm`, `dpkg`)는 의존성을 **자동 해결하지 못함** → 누락 시 설치 실패
- 고수준 도구(`dnf`/`yum`, `apt`)는 저장소를 참조해 의존 패키지를 **자동 다운로드·설치**
- 의존성 지옥(dependency hell): 상호 충돌·순환 의존으로 해결 불가에 빠지는 상황

### 1-3. 바이너리 설치 vs 소스 설치

| 구분 | 바이너리 패키지 | 소스 설치 |
| --- | --- | --- |
| 형태 | 미리 컴파일된 실행 파일 (`.rpm`, `.deb`) | 소스 코드 압축본 (`.tar.gz` 등) |
| 설치 속도 | 빠름 (컴파일 불필요) | 느림 (컴파일 필요) |
| 의존성 | 패키지 관리자가 추적 | 수동 확인 필요 |
| 최적화 | 배포판 기본 설정 | 아키텍처·옵션 커스터마이징 가능 |
| 관리 | DB 등록 → 삭제·업그레이드 용이 | DB 미등록 → 수동 관리 |

---

## 2. RedHat 계열 — rpm / dnf / yum

### 2-1. rpm 주요 옵션 (최빈출)

- 패키지 파일명 규칙: `<이름>-<버전>-<릴리스>.<아키텍처>.rpm` (예: `httpd-2.4.6-97.el7.x86_64.rpm`)

| 기능 | 옵션 | 설명 |
| --- | --- | --- |
| 설치 | `rpm -ivh <파일>.rpm` | **i**nstall + **v**erbose + **h**ash(진행바) |
| 업그레이드 | `rpm -Uvh <파일>.rpm` | **U**pgrade — 없으면 신규 설치, 있으면 갱신 |
| 갱신 | `rpm -Fvh <파일>.rpm` | **F**reshen — 이미 설치된 것만 갱신 (신규 설치 안 함) |
| 삭제 | `rpm -e <패키지>` | **e**rase (버전·아키텍처 없이 이름만) |
| 질의(단일) | `rpm -q <패키지>` | 설치 여부·버전 확인 |
| 질의(전체) | `rpm -qa` | 설치된 **a**ll 패키지 목록 |
| 질의(정보) | `rpm -qi <패키지>` | **i**nfo — 상세 정보 |
| 질의(파일목록) | `rpm -ql <패키지>` | **l**ist — 포함 파일 경로 목록 |
| 질의(파일소속) | `rpm -qf <파일경로>` | 특정 파일이 어느 패키지 소속인지 (**f**ile) |
| 질의(파일기준) | `rpm -qp <파일>.rpm` | 미설치 rpm **p**ackage 파일에 질의 |
| 검증 | `rpm -V <패키지>` | **V**erify — 설치 후 변경 여부 검사 |

- 삭제 시 다른 패키지가 의존하면 거부 → `--nodeps` 로 강제 가능(비권장)
- `rpm -V` 출력 문자: `S`(크기) `M`(권한) `5`(MD5) `L`(심볼릭링크) `T`(시각) `U`(소유자) `G`(그룹) — 변경 없으면 `.` 표기 → [[rpm]]

### 2-2. dnf / yum

- `yum` 의 후속이 `dnf` — 저장소 기반 의존성 자동 해결 도구
- 명령 체계는 사실상 동일 (`yum` → `dnf` 치환 가능)

| 명령 | 기능 |
| --- | --- |
| `dnf install <패키지>` | 저장소에서 설치 (의존성 자동) |
| `dnf update` / `dnf upgrade` | 전체·개별 패키지 업그레이드 |
| `dnf remove <패키지>` | 삭제 |
| `dnf search <키워드>` | 패키지 검색 |
| `dnf info <패키지>` | 상세 정보 |
| `dnf list [installed]` | 패키지 목록 |
| `dnf grouplist` | 패키지 그룹 목록 |
| `dnf groupinstall "<그룹>"` | 그룹 단위 설치 → [[dnf-group]] |
| `dnf repolist` | 등록된 저장소 목록 |
| `dnf provides <파일>` | 파일을 제공하는 패키지 검색 |

- 실사용 문서 → [[dnf]] · [[dnf-group]]

### 2-3. 저장소(repository) 설정

- 위치: `/etc/yum.repos.d/*.repo`
- 주요 지시자

```ini
# /etc/yum.repos.d/example.repo
[baseos]                     # 저장소 ID (대괄호)
name=Example BaseOS          # 표시 이름
baseurl=http://mirror/os/    # 저장소 URL
enabled=1                    # 1=활성, 0=비활성
gpgcheck=1                   # 서명 검증 여부
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY
```

- `mirrorlist=` : baseurl 대신 미러 목록 URL 지정 가능

---

## 3. Debian 계열 — dpkg / apt

### 3-1. dpkg (저수준)

- Debian 패키지(`.deb`) 저수준 관리 — 의존성 자동 해결 **안 함**

| 옵션 | 기능 |
| --- | --- |
| `dpkg -i <파일>.deb` | 설치 (**i**nstall) |
| `dpkg -r <패키지>` | 삭제 (설정 파일 유지, **r**emove) |
| `dpkg -P <패키지>` | 완전 삭제 (설정 포함, **P**urge) |
| `dpkg -l` | 설치된 패키지 목록 (**l**ist) |
| `dpkg -L <패키지>` | 패키지 포함 파일 목록 |
| `dpkg -S <파일경로>` | 파일 소속 패키지 검색 |

### 3-2. apt / apt-get (고수준)

- 저장소 기반 의존성 자동 해결 (RedHat 의 `dnf`/`yum` 대응)

| 명령 | 기능 |
| --- | --- |
| `apt update` | 저장소 패키지 목록 갱신 (설치 전 선행) |
| `apt upgrade` | 설치된 패키지 업그레이드 |
| `apt install <패키지>` | 설치 |
| `apt remove <패키지>` | 삭제 (설정 유지) |
| `apt purge <패키지>` | 완전 삭제 |
| `apt search <키워드>` | 검색 |

- `apt` 는 대화형 통합 명령, `apt-get`/`apt-cache` 는 스크립트용 저수준 명령
- 주의: RedHat `update` = 업그레이드, Debian `update` = **목록 갱신**(≠ 업그레이드) → 함정 포인트

### 3-3. 저장소 설정

- 위치: `/etc/apt/sources.list`, `/etc/apt/sources.list.d/*.list`
- 형식: `deb <URL> <배포판코드명> <컴포넌트>` (예: `deb http://archive.ubuntu.com/ubuntu focal main`)

### 3-4. 계열 대응 요약

| 기능 | RedHat (rpm/dnf) | Debian (dpkg/apt) |
| --- | --- | --- |
| 패키지 형식 | `.rpm` | `.deb` |
| 저수준 도구 | `rpm` | `dpkg` |
| 고수준 도구 | `dnf` / `yum` | `apt` / `apt-get` |
| 설치 | `dnf install` | `apt install` |
| 저장소 설정 | `/etc/yum.repos.d/` | `/etc/apt/sources.list` |

---

## 4. 소스 컴파일 설치

### 4-1. 3단계 절차 (최빈출)

```bash
# 0단계: 압축 해제 선행
tar -xvzf httpd-2.4.6.tar.gz
cd httpd-2.4.6

# 1단계: 환경 설정 — Makefile 생성, 의존성·설치 경로 검사
./configure --prefix=/usr/local/httpd

# 2단계: 컴파일 — 소스를 실행 바이너리로 빌드
make

# 3단계: 설치 — 지정 경로로 복사 (보통 root 권한)
make install
```

| 단계 | 명령 | 역할 |
| --- | --- | --- |
| ① 설정 | `./configure` | 시스템 검사 → `Makefile` 생성, `--prefix` 로 설치 경로 지정 |
| ② 컴파일 | `make` | `Makefile` 기반으로 소스 컴파일 |
| ③ 설치 | `make install` | 빌드 결과물을 시스템 경로로 복사 |

- 제거는 `make uninstall` (소스에 규칙이 있을 때만), 빌드 산출물 정리는 `make clean`
- 아카이브 해제가 `./configure` **선행 조건** → 순서 출제 → [[tar]]

### 4-2. 의존 라이브러리·헤더

- 컴파일에는 실행 라이브러리 외에 **개발 헤더**(`*-devel`/`*-dev` 패키지) 필요
- `./configure` 단계에서 누락 헤더·라이브러리 감지 시 오류 → 해당 devel 패키지 선설치

---

## 5. 라이브러리 관리

### 5-1. 정적 vs 공유 라이브러리

| 구분 | 정적 라이브러리 | 공유 라이브러리 |
| --- | --- | --- |
| 확장자 | `.a` (archive) | `.so` (shared object) |
| 링크 시점 | **컴파일 시** 실행 파일에 포함 | **실행 시** 동적 로드 |
| 실행 파일 크기 | 큼 (라이브러리 내장) | 작음 (외부 참조) |
| 갱신 | 재컴파일 필요 | 라이브러리만 교체 |
| 메모리 | 프로세스마다 복사 | 여러 프로세스가 공유 |

### 5-2. 공유 라이브러리 경로·설정

- 표준 경로: `/lib`, `/lib64`, `/usr/lib`, `/usr/lib64`
- 설정 파일: `/etc/ld.so.conf`, `/etc/ld.so.conf.d/*.conf` (추가 검색 경로)
- 캐시: `/etc/ld.so.cache` — `ldconfig` 실행으로 갱신
- `ldconfig` : 설정 파일을 읽어 캐시 재생성 (새 라이브러리 추가 후 실행)
- `LD_LIBRARY_PATH` : 표준 경로 외 라이브러리를 임시로 검색하게 하는 환경변수

### 5-3. 의존성 확인 — ldd

- `ldd <실행파일>` : 실행 파일이 요구하는 공유 라이브러리 목록·경로 출력
- `not found` 표시 = 의존 라이브러리 누락 → 실행 불가 원인 진단
- 실사용 문서 → [[ldd]]

---

## 6. 아카이브·압축

### 6-1. tar (아카이브)

- 여러 파일을 하나로 묶는 아카이브 도구 (묶기 + 압축 필터 결합)

| 옵션 | 기능 |
| --- | --- |
| `-c` | 생성 (**c**reate) |
| `-x` | 추출 (e**x**tract) |
| `-t` | 목록 조회 (**t**able) |
| `-v` | 상세 출력 (**v**erbose) |
| `-f` | 파일명 지정 (**f**ile) — 항상 마지막 |
| `-z` | gzip 압축 필터 (`.tar.gz`) |
| `-j` | bzip2 압축 필터 (`.tar.bz2`) |
| `-J` | xz 압축 필터 (`.tar.xz`) |
| `-C` | 대상 디렉터리 지정 |

```bash
tar -cvzf backup.tar.gz /home     # gzip 압축 아카이브 생성
tar -xvjf backup.tar.bz2          # bzip2 아카이브 추출
tar -tvf  backup.tar              # 내용 목록만 조회
```

- 실사용 문서 → [[tar]]

### 6-2. 압축 명령 비교

| 도구 | 확장자 | tar 옵션 | 특징 |
| --- | --- | --- | --- |
| `gzip` | `.gz` | `-z` | 속도 빠름, 압축률 보통 (최다 사용) |
| `bzip2` | `.bz2` | `-j` | gzip보다 압축률 높음·속도 느림 |
| `xz` | `.xz` | `-J` | 압축률 최고·속도 가장 느림 |
| `zip` / `unzip` | `.zip` | (별도) | 크로스플랫폼, 아카이브+압축 일체형 → [[zip]] · [[unzip]] |

- `gzip`/`bzip2`/`xz` 는 단일 파일 압축이 기본 → 여러 파일은 `tar` 로 먼저 묶음
- 압축 해제: 각 도구의 `-d` 옵션 (`gzip -d`, `bzip2 -d`, `xz -d`)
- 압축률 순서(높음): `xz` > `bzip2` > `gzip` — 속도는 반대

---

## 연관 문서
- [[rpm]] — RedHat 패키지 저수준 관리 실사용 문서
- [[dnf]] · [[dnf-group]] — 저장소 기반 패키지·그룹 설치 실사용 문서
- [[tar]] — 아카이브·백업 실사용 문서
- [[zip]] · [[unzip]] — zip 아카이브 실사용 문서
- [[ldd]] — 공유 라이브러리 의존성 확인
- [[system-structure]] — 파일시스템 계층·표준 경로
- [[system-security]] — 패키지 서명·무결성 검증 연계
