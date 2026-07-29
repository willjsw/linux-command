---
command: getenforce
category: SECURITY-SELINUX
aliases: [setenforce, sestatus, autorelabel]
tags:
  - linux/security
  - topic/selinux
  - task/inspect
  - task/configure
  - privilege/mixed
  - distro/rhel
  - distro/rocky
related: ["[[ausearch]]", "[[firewall-cmd]]", "[[rescue-mode]]", "[[passwd]]", "[[file-ops]]"]
distro: RHEL 계열
verified: Rocky Linux 9.6
updated: 2026-07-30
---

# getenforce / setenforce

- SELinux 동작 모드 조회·변경 도구
- SELinux: **S**ecurity-**E**nhanced **Linux** — 프로세스·파일 레이블 기반 강제 접근 제어
- RHEL 계열 기본값: **Enforcing**(차단 활성)

---

## getenforce

```bash
getenforce

# Examples
getenforce            # Enforcing / Permissive / Disabled
sestatus              # 상세 상태 (정책·마운트 정보 포함)
```

### 명령어 설명
- 사용 목적
	- SELinux 현재 동작 모드 확인 시 사용
	- 애플리케이션 차단 원인 조사 시 사용
- 특이사항
	- `Enforcing` : 정책 위반 차단 (기본값)
	- `Permissive` : 차단 없이 로그만 기록
	- `Disabled` : 완전 비활성
	- 비표준 포트·경로를 사용하는 애플리케이션에서 차단 원인이 되는 경우 존재
	- **비활성화 전 차단 내역 확인 우선** → [[ausearch]] 사용

---

## setenforce

```bash
setenforce 0|1

# Examples
setenforce 0          # Permissive 로 일시 전환
setenforce 1          # Enforcing 복귀
```

### 명령어 설명
- 사용 목적
	- 차단 원인 검증을 위해 일시적으로 정책 완화 시 사용
- 특이사항
	- **재부팅 시 원복** → 영구 변경은 `/etc/selinux/config` 편집 필요
	- 영구 비활성화는 보안 저하 → 원인 해소 후 `Enforcing` 복귀 권장

---

## autorelabel (파일 레이블 재생성)

```bash
touch /.autorelabel

# Examples
touch /.autorelabel      # 다음 부팅 시 전체 파일 레이블 재생성 예약
```

### 명령어 설명
- 사용 목적
	- **레스큐·`rd.break` 모드에서 파일 변경 후 레이블 정합성 복구** 시 사용
	- SELinux 레이블 불일치로 인한 로그인·서비스 실패 방지 목적으로 사용
- 특이사항
	- 빈 파일 생성만으로 다음 부팅 시 재레이블 트리거
	- **디스크 전체 순회** → 수분 소요, 완료 후 자동 재부팅 1회 추가 발생
	- 레스큐 셸 사용 시 시스템이 자동 경고 출력
	  ```
	  Warning: The rescue shell will trigger SELinux autorelabel on the subsequent boot.
	  Add "enforcing=0" on the kernel command line for autorelabel to work properly.
	  ```
	- **비밀번호 재설정 절차에서 누락 금지** → `/etc/shadow` 레이블 불일치로 로그인 재실패

---

## 관련 부팅 파라미터
- `enforcing=0` : 부팅 시 SELinux 일시 비활성 → [[rescue-mode]] 참고
- `selinux=0` : SELinux 완전 비활성 (권장하지 않음)

---

## 연관 명령어
- [[ausearch]] : SELinux 차단(AVC) 로그 조회
- [[firewall-cmd]] : 방화벽 차단과 구분 진단
- [[rescue-mode]] : `enforcing=0` 파라미터 사용 맥락
- [[passwd]] : 비밀번호 재설정 시 autorelabel 필요
- [[file-ops]] : `touch /.autorelabel` 재라벨링 예약
