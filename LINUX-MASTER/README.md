---
title: LINUX-MASTER — 리눅스마스터 1급 시험 대비
type: exam-hub
tags:
  - exam/linux-master
  - moc/index
updated: 2026-08-28
---

# LINUX-MASTER 1급 시험 대비

- 리눅스마스터 1급 필기(1차) 대비 이론·문제집 모음
- 시험 목표일 기준 2주 집중 대비용 (2026-08-28 기준)

---

## 시험 개요 (KAIT 공식)

- 형식: 객관식 4지선다, **100문항 / 100분**
- 합격: 전체 평균 **60점 이상**, 과목별 **40% 미만 과락**
- 8개 과목: 리눅스 실무의 이해 · 리눅스 시스템의 이해 · 네트워크의 이해 · 리눅스 시스템 관리 · 장치 관리 · 시스템 보안 및 관리 · 네트워크 서비스 · 네트워크 보안

---

## 이론 문서 (THEORY)

- 전 시험범위 커버 — 8과목 이론 12종

### 리눅스 실무·시스템의 이해
- [[linux-basics]] — 리눅스 실무의 이해 (OS 개요·역사·라이선스·배포판·부팅과정·런레벨)
- [[system-structure]] — 리눅스 시스템의 이해 (FHS·파일유형·inode·SHELL·X 윈도)

### 네트워크
- [[network-basics]] — 네트워크의 이해 (OSI·TCP/IP·IP주소·서브네팅·포트·프로토콜)
- [[network-service]] — 네트워크 서비스 (웹·DNS·메일·파일공유·가상화)
- [[network-security]] — 네트워크 보안 (침해 유형·iptables·IDS·암호화)

### 시스템 관리·장치
- [[process-management]] — 프로세스 관리 (상태·시그널·잡제어·cron·우선순위)
- [[user-permission]] — 사용자·권한 관리 (계정파일·권한·특수권한·ACL·쿼터)
- [[package-software]] — 소프트웨어·패키지 관리 (rpm·dnf·dpkg·소스컴파일·라이브러리)
- [[disk-device]] — 디스크·장치 관리 (파티션·파일시스템·RAID·LVM·스왑·모듈)
- [[systemd-service]] — systemd·서비스 관리 (유닛·타겟·저널·런레벨 대응)
- [[selinux-security]] — SELinux (MAC·모드·보안컨텍스트·불린·라벨링)

### 보안
- [[system-security]] — 시스템 보안 및 관리 (로그·PAM·백업·특수권한)

> 실습 명령어 상세는 볼트 본편 [[INDEX]] 참조

---

## 문제집 사용법

- `questions.md` 를 먼저 풀고 → `answers.md` 로 채점·해설 확인
- 하루 1회차씩, 14일간 필기 → 실기 순 권장
- 오답은 이론 문서·볼트 명령어 문서로 역추적

### 필기 문제집 (EXAM-WRITTEN) — 14회차
- 회차당 20문항, 8과목 고루 배분
- `EXAM-WRITTEN/round-01/` ~ `round-14/`

### 실기 문제집 (EXAM-PRACTICAL) — 6회차
- 회차당 15문항, 명령어 작성·빈칸·설정 파일 서술형
- `EXAM-PRACTICAL/round-01/` ~ `round-06/`

---

## 2주 학습 로드맵

| 일차       | 학습                                    |
| -------- | ------------------------------------- |
| Day 1    | 이론 [[system-security]] + 필기 round-01  |
| Day 2    | 이론 [[network-service]] + 필기 round-02  |
| Day 3    | 이론 [[network-security]] + 필기 round-03 |
| Day 4~7  | 필기 round-04 ~ round-07 (오답 이론 복습 병행)  |
| Day 8~11 | 필기 round-08 ~ round-11                |
| Day 12   | 필기 round-12 + 실기 round-01             |
| Day 13   | 필기 round-13~14 + 실기 round-02~04       |
| Day 14   | 실기 round-05~06 + 전 회차 오답 재점검          |

---

## 관련 문서
- [[INDEX]] — 명령어 볼트 전체 색인
- [[SKILL]] — 볼트 작성 규칙
