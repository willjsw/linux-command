---
title: SELinux
type: exam-theory
category: LINUX-MASTER
tags:
  - exam/linux-master
  - exam/theory
  - linux/security
  - linux/selinux
  - topic/security
  - task/manage
related: ["[[system-security]]", "[[network-security]]", "[[getenforce]]", "[[ausearch]]", "[[dmesg]]"]
updated: 2026-08-28
---

# SELinux

- 리눅스마스터 1급 필기 대비 이론 — SELinux 강제 접근 제어(MAC)·동작 모드·보안 컨텍스트·불린
- 동작 모드 값(Enforcing 1 / Permissive 0)과 보안 컨텍스트 4요소가 최빈출 → 정확 암기 필요

---

## 1. SELinux 개요

- **SELinux** (Security-Enhanced Linux) — 커널 수준 강제 접근 제어 구현 (NSA 개발)
- **MAC vs DAC** (핵심 대비)

| 구분 | DAC (임의 접근 제어) | MAC (강제 접근 제어) |
| --- | --- | --- |
| 정의 | 소유자가 권한 임의 부여 | 시스템 정책이 강제 |
| 기준 | 소유자·그룹·기타 (rwx) | 보안 컨텍스트·정책 규칙 |
| 통제 주체 | 파일 소유자 | 관리자·정책 (사용자 변경 불가) |
| root 예외 | root 는 전권 | root 도 정책에 종속 |
| 예 | 전통적 `chmod`/`chown` 권한 | SELinux, AppArmor |

- **최소 권한(least privilege)**: 프로세스는 정책이 허용한 최소 자원에만 접근 → 침해 시 피해 국소화
- DAC 을 통과해도 MAC(SELinux) 정책에서 거부되면 최종 차단 (이중 검사)

---

## 2. 동작 모드

### 2-1. 3종 모드 (최빈출)

| 모드 | 값 | 동작 |
| --- | --- | --- |
| **Enforcing** | `1` | 정책 위반을 **차단하고 로그 기록** (운영 권장) |
| **Permissive** | `0` | 위반을 **차단하지 않고 로그만 기록** (정책 튜닝·디버깅) |
| **Disabled** | — | SELinux 완전 비활성 (라벨링도 중단) |

- Enforcing↔Permissive 는 재부팅 없이 전환 가능, Disabled↔활성 전환은 **재부팅 필요**
- 모드 값 함정: Enforcing = 1, Permissive = 0 (숫자 반대로 외우지 말 것)

### 2-2. 설정 파일 (영구)

```
# /etc/selinux/config
SELINUX=enforcing          # enforcing | permissive | disabled
SELINUXTYPE=targeted       # 정책 유형
```

- `/etc/selinux/config` 의 `SELINUX=` 값이 부팅 시 적용되는 영구 설정
- Disabled → Enforcing 변경 시 재부팅 후 전체 재라벨링 발생 (시간 소요)

### 2-3. 조회·일시 전환 명령

| 명령 | 기능 |
| --- | --- |
| [[getenforce]] | 현재 모드 조회 (`Enforcing`/`Permissive`/`Disabled`) |
| `sestatus` | 모드·정책·마운트 상태 상세 조회 |
| `setenforce 1` | Enforcing 으로 일시 전환 (재부팅 시 원복) → [[getenforce]] |
| `setenforce 0` | Permissive 로 일시 전환 |

- `setenforce` 는 **일시** 전환 (재부팅 시 `config` 값으로 복귀), 영구 변경은 `config` 편집
- `setenforce` 로는 Disabled 전환 불가 (Enforcing↔Permissive 만 가능)

---

## 3. 정책 유형

| 유형 | 설명 |
| --- | --- |
| **targeted** | 기본값 — 지정된 대상 데몬(httpd, sshd 등)만 제한, 나머지는 unconfined |
| **mls** | 다단계 보안(Multi-Level Security) — 군·정부용 엄격 등급 통제 |
| **minimum** | targeted 축소판 — 최소 프로세스만 보호 |

- 대부분의 배포판 기본은 `targeted` (`SELINUXTYPE=targeted`)

---

## 4. 보안 컨텍스트

### 4-1. 4요소 구조 (최빈출)

```
user : role : type : level
   |      |      |      |
system_u system_r httpd_t s0
```

| 요소 | 이름 | 역할 |
| --- | --- | --- |
| 1 | **user** (SELinux 사용자) | 리눅스 계정과 별개의 SELinux 사용자 (`system_u`, `unconfined_u`) |
| 2 | **role** (역할) | 사용자가 진입 가능한 도메인 집합 (`object_r`, `system_r`) |
| 3 | **type** (타입/도메인) | **접근 통제 핵심** — 파일=type, 프로세스=도메인 |
| 4 | **level** (레벨) | MLS/MCS 민감도·범주 (`s0`, `s0-s0:c0.c1023`) |

- **type(타입) 중심 제어**: targeted 정책은 Type Enforcement(TE) 기반 → type 이 접근 허용의 핵심
  - 프로세스의 type 을 특히 **도메인(domain)** 이라 부름 (예: `httpd_t`)
  - 규칙 예: 도메인 `httpd_t` 는 type `httpd_sys_content_t` 파일만 읽기 허용

### 4-2. 컨텍스트 확인 (`-Z` 옵션)

| 명령 | 대상 |
| --- | --- |
| `ls -Z <파일>` | 파일·디렉터리 컨텍스트 |
| `ps -Z` | 프로세스 도메인 |
| `id -Z` | 현재 사용자 컨텍스트 |

- 공통 옵션 `-Z` 로 컨텍스트 표시 (암기 포인트)

---

## 5. 컨텍스트 관리

| 명령 | 지속성 | 용도 |
| --- | --- | --- |
| `chcon` | 일시 | 컨텍스트 직접 변경 (재라벨링 시 원복됨) |
| `restorecon` | 기본값 복원 | 정책 기본 컨텍스트로 되돌림 |
| `semanage fcontext` | 영구 | 파일 컨텍스트 규칙을 정책 DB 에 영구 등록 |

- 영구 변경 표준 절차: `semanage fcontext -a -t <type> "<경로>"` → `restorecon -Rv <경로>` 적용
- `chcon` 만 쓰면 이후 `restorecon`·재라벨링 시 원복 → 영구 반영은 `semanage fcontext` 필수
- **전체 재라벨링**: `/.autorelabel` 파일 생성 후 재부팅 → 전체 파일시스템 컨텍스트 재적용 → [[getenforce]]
  - Disabled→Enforcing 복귀나 대규모 라벨 손상 시 사용

---

## 6. 불린(boolean)

- 정책을 재컴파일 없이 런타임에 토글하는 on/off 스위치

| 명령 | 기능 |
| --- | --- |
| `getsebool -a` | 전체 불린 상태 조회 |
| `getsebool <불린>` | 특정 불린 상태 조회 |
| `setsebool <불린> on` | 일시 토글 (재부팅 시 원복) |
| `setsebool -P <불린> on` | **영구** 토글 (`-P` = persistent) |

- `-P` 유무가 핵심: 미지정 시 재부팅으로 원복
- 대표 불린 예
  - `httpd_can_network_connect` : httpd 의 외부 네트워크 연결 허용
  - `httpd_enable_homedirs` : httpd 의 사용자 홈 디렉터리 접근 허용
  - `ftpd_anon_write` : FTP 익명 쓰기 허용

---

## 7. 포트 라벨

- 서비스가 비표준 포트를 쓰려면 해당 포트에 type 라벨 부여 필요

```
# httpd 를 8888 포트로 운용하려면 http_port_t 라벨 추가
semanage port -a -t http_port_t -p tcp 8888

semanage port -l              # 포트 라벨 목록 조회
```

- `semanage port -a`(추가) / `-m`(수정) / `-d`(삭제) / `-l`(목록)
- 포트 라벨 누락이 "서비스는 떠 있는데 바인딩 거부" 증상의 흔한 원인

---

## 8. 로그·문제 해결

- **AVC(Access Vector Cache) denied** 로그: SELinux 거부 이벤트 기록 → `/var/log/audit/audit.log`
- 조회·분석 도구

| 도구 | 기능 |
| --- | --- |
| [[ausearch]] | 감사 로그 검색 (`ausearch -m avc` = AVC 메시지) |
| `sealert` | setroubleshoot — AVC 거부 분석·해결책 제안 |
| `audit2allow` | 거부 로그로부터 커스텀 정책 모듈 생성 |
| [[dmesg]] | audit 미동작 시 커널 링버퍼에서 AVC 확인 |

- 진단 흐름: Permissive 로 전환 → 거부 로그 수집 → `sealert`/`audit2allow` 로 원인·해결 도출
- 로그·감사 상세와 다른 보안 점검 도구는 [[system-security]] 참조

---

## 연관 문서
- [[system-security]] — 로그·감사·특수 권한·PAM 등 시스템 보안 전반
- [[network-security]] — 네트워크 침해 대응 (방화벽·IDS)
- [[getenforce]] — SELinux 모드 조회 실사용 문서
- [[ausearch]] — 감사 로그 검색 실사용 문서
- [[dmesg]] — 커널 메시지 조회 실사용 문서
