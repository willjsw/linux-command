---
title: 리눅스 실무의 이해
type: exam-theory
category: LINUX-MASTER
tags:
  - exam/linux-master
  - exam/theory
  - linux/basics
  - linux/boot
  - topic/os
  - topic/license
  - task/study
related: ["[[system-structure]]", "[[systemd-service]]", "[[disk-device]]", "[[grub2-install]]", "[[systemctl]]", "[[localectl]]"]
updated: 2026-08-28
---

# 리눅스 실무의 이해

- 리눅스마스터 1급 필기 대비 이론 — 운영체제 개요·유닉스/리눅스 역사·라이선스·배포판·부팅 과정
- 역사(연도·인물), 라이선스 copyleft 여부, 런레벨↔systemd 타겟 대응이 최빈출 → 표 단위 암기 필요

---

## 1. 운영체제 개요

### 1-1. OS 역할

- 하드웨어와 사용자·응용프로그램 사이의 중재 계층
- 자원 관리: 프로세스(CPU)·메모리·파일시스템·입출력장치·네트워크 → [[process-management]]
- 사용자 인터페이스 제공(CLI·GUI), 보안·접근 제어 → [[user-permission]]

### 1-2. OS 구성 3요소

| 구성 | 역할 |
| --- | --- |
| 커널(Kernel) | OS 핵심 — 자원 관리·스케줄링·메모리 관리·장치 제어. 부팅 시 메모리 상주 |
| 셸(Shell) | 사용자 명령 해석기 — 커널과 사용자 사이 인터페이스 → [[system-structure]] |
| 유틸리티(Utility) | 응용 명령·도구 모음 (`ls`, `cp`, `grep` 등) |

### 1-3. 커널 유형

| 유형 | 구조 | 특징 | 예 |
| --- | --- | --- | --- |
| 모놀리식(Monolithic) | 모든 핵심 기능을 단일 커널 공간에 포함 | 성능 우수, 크기 큼, 모듈 결합도 높음 | 리눅스, 전통 유닉스 |
| 마이크로(Micro) | 최소 기능만 커널에, 나머지는 사용자 공간 서버로 분리 | 안정성·이식성 우수, 통신 오버헤드 | MINIX, Mach, QNX |

- 리눅스는 모놀리식 기반이나 **로드 가능 모듈(LKM)** 로 기능 동적 적재 → 하이브리드 성격

---

## 2. 유닉스·리눅스 역사

### 2-1. 연표 (최빈출)

| 연도 | 사건 | 인물 |
| --- | --- | --- |
| 1969 | 유닉스 최초 개발 (AT&T 벨 연구소) | 켄 톰슨(Ken Thompson) |
| 1973 | 유닉스를 C 언어로 재작성 (이식성 확보) | 데니스 리치(Dennis Ritchie) |
| 1983 | GNU 프로젝트 시작 (자유 소프트웨어 운동) | 리처드 스톨먼(Richard Stallman) |
| 1985 | FSF(자유 소프트웨어 재단) 설립 | 리처드 스톨먼 |
| 1987 | MINIX 발표 (교육용 유닉스 호환 OS) | 앤드루 타넨바움(A. Tanenbaum) |
| 1991 | 리눅스 커널 최초 공개 (v0.01) | 리누스 토르발스(Linus Torvalds) |
| 1994 | 리눅스 커널 1.0 발표 | 리누스 토르발스 |

- 유닉스 C 재작성이 이식성의 기반 → 하드웨어 종속 탈피가 출제 포인트
- 리눅스는 토르발스가 MINIX 에 자극받아 개발, GPL 채택으로 확산

### 2-2. GNU 프로젝트

- 목표: 완전한 자유 유닉스 호환 시스템 구축
- 컴파일러(GCC)·셸(bash)·핵심 유틸리티(coreutils) 등 확보, 커널(Hurd)만 미완성
- 토르발스의 커널 + GNU 도구 = **GNU/Linux** → 리눅스 배포판의 실체

---

## 3. 라이선스

### 3-1. 주요 라이선스 비교 (최빈출)

| 라이선스 | copyleft | 파생물 공개 의무 | 특징 |
| --- | --- | --- | --- |
| GPL | 강한 copyleft | 소스 공개 필수 | 링크·수정 시 동일 라이선스 전파 |
| LGPL | 약한 copyleft | 라이브러리 수정분만 공개 | 동적 링크한 독점 SW 허용 |
| BSD | 비 copyleft | 없음 | 저작권 표시만, 독점 SW 편입 자유 |
| Apache | 비 copyleft | 없음 | 특허권 명시 조항 포함 |
| MIT | 비 copyleft | 없음 | 가장 단순·관대, 표시만 요구 |

- **copyleft**: 파생 저작물도 동일 자유를 유지하도록 강제하는 개념 → GPL 이 대표
- BSD·Apache·MIT 는 **허용적(permissive)** — 독점 소프트웨어에 편입 가능

### 3-2. 자유 소프트웨어 vs 오픈소스

| 구분 | 자유 소프트웨어(FSF) | 오픈소스(OSI) |
| --- | --- | --- |
| 강조점 | 사용자의 **자유**(윤리·철학) | 개발 방법론·실용적 이점 |
| 대표 인물 | 리처드 스톨먼 | 에릭 레이먼드 등 |
| 4대 자유 | 실행·연구·재배포·개선의 자유 | — |

- "자유(free)" 는 무료가 아닌 **자유**를 의미 → 함정 포인트

---

## 4. 배포판 계열

### 4-1. RedHat 계열 vs Debian 계열 (최빈출)

| 구분          | RedHat 계열                              | Debian 계열            |
| ----------- | -------------------------------------- | -------------------- |
| 대표 배포판      | RHEL, Rocky, AlmaLinux, Fedora, CentOS | Debian, Ubuntu, Mint |
| 패키지 형식      | `.rpm`                                 | `.deb`               |
| 저수준 패키지 도구  | `rpm`                                  | `dpkg`               |
| 고수준(의존성) 도구 | `dnf` (구 `yum`)                        | `apt` (`apt-get`)    |
| 특징          | 기업·서버 중심, 안정성                          | 방대한 패키지, 데스크톱 강세     |

- Fedora = RHEL 의 상위(테스트) 배포판, CentOS Stream = RHEL 상류
- 패키지 관리 상세는 [[package-software]] 참조

---

## 5. 부팅 과정 (최빈출)

### 5-1. 부팅 단계

```
전원 ON
  → BIOS/UEFI (POST: 하드웨어 자가진단)
  → 부트로더 (GRUB2 / GRUB Legacy) — MBR 또는 EFI 파티션
  → 커널 로드 + initramfs(초기 램디스크) 마운트
  → systemd (PID 1) 시작
  → 기본 타겟(default.target) 도달 → 로그인
```

| 단계 | 역할 |
| --- | --- |
| BIOS/UEFI | 펌웨어 — POST 수행 후 부팅 장치 지정, 부트로더 호출 |
| POST | Power-On Self-Test — 하드웨어 이상 점검 |
| 부트로더 | 커널을 메모리에 적재·실행 (GRUB) → [[grub2-install]] |
| initramfs | 루트 파일시스템 마운트 전 필요한 드라이버·도구를 담은 임시 루트 |
| systemd | 최초 프로세스(PID 1) — 이후 모든 서비스의 부모 → [[systemd-service]] |

### 5-2. BIOS vs UEFI

| 구분 | BIOS | UEFI |
| --- | --- | --- |
| 디스크 파티션 | MBR (최대 2TB, 4 주 파티션) | GPT (2TB 초과, 128 파티션) |
| 부팅 방식 | MBR 부트섹터 실행 | EFI 시스템 파티션(ESP)의 부트로더 실행 |
| 부가 기능 | — | 보안 부팅(Secure Boot), GUI 설정 |

### 5-3. GRUB2 vs GRUB Legacy

| 구분 | GRUB Legacy (0.9x) | GRUB2 (1.9x~) |
| --- | --- | --- |
| 주 설정 파일 | `/boot/grub/menu.lst` (`grub.conf`) | `/boot/grub2/grub.cfg` (직접 편집 금지) |
| 설정 생성 | 직접 편집 | `grub2-mkconfig` 로 생성, `/etc/default/grub` + `/etc/grub.d/` 기반 |
| 메뉴 번호 | 0부터 시작 | 0부터 시작, 스크립트 기반 동적 생성 |

- 디스크·파티션 구조 상세는 [[disk-device]] 참조

---

## 6. 런레벨 ↔ systemd 타겟 (최빈출)

| 런레벨 | systemd 타겟 | 상태 |
| --- | --- | --- |
| 0 | `poweroff.target` | 시스템 종료 |
| 1 | `rescue.target` | 단일 사용자 모드(복구) |
| 2 | `multi-user.target` | 다중 사용자(NFS 미포함, 관례) |
| 3 | `multi-user.target` | 다중 사용자 + 네트워크 (텍스트 콘솔) |
| 4 | `multi-user.target` | 예약(미사용) |
| 5 | `graphical.target` | 다중 사용자 + GUI |
| 6 | `reboot.target` | 재부팅 |

- 기본 타겟 확인·변경: `systemctl get-default` / `systemctl set-default <타겟>` → [[systemctl]]
- 즉시 전환: `systemctl isolate multi-user.target`
- `runlevel` 명령은 이전/현재 런레벨 출력 (systemd 하위 호환)

---

## 7. 로그인 셸·기본 디렉터리 개요

- 로그인 시 사용자에게 부여되는 셸 = **로그인 셸** — `/etc/passwd` 7번째 필드에 지정
- 사용 가능 셸 목록: `/etc/shells`, 로그인 차단용 셸: `/sbin/nologin`, `/bin/false`
- 셸 종류·초기화 파일·환경변수 상세는 [[system-structure]] 의 SHELL 절 참조
- 기본 디렉터리 계층(FHS)·파일 유형·inode 는 [[system-structure]] 참조
- 로케일·키맵 설정: [[localectl]]

---

## 연관 문서
- [[system-structure]] — 파일시스템 계층·셸·X 윈도·하드웨어
- [[systemd-service]] — systemd 유닛·타겟 관리
- [[disk-device]] — 디스크 파티션·MBR/GPT
- [[package-software]] — rpm/dnf·dpkg/apt 패키지 관리
- [[process-management]] — 프로세스·스케줄링
- [[user-permission]] — 사용자·권한 관리
- [[grub2-install]] — GRUB2 부트로더 설치 실사용 문서
- [[systemctl]] — systemd 제어 실사용 문서
- [[localectl]] — 로케일·키맵 설정 실사용 문서
