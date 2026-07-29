---
command: rpm
category: PACKAGE-MANAGEMENT
aliases: []
tags:
  - linux/package
  - distro/rhel
  - distro/rocky
  - task/verify
  - task/query
  - topic/troubleshooting
  - privilege/user
related: ["[[dnf]]", "[[dnf-group]]", "[[grep]]", "[[which]]", "[[file]]"]
distro: RHEL 계열 (Rocky, CentOS, Fedora)
verified: Rocky Linux 9.6
updated: 2026-07-30
---

# rpm

- RHEL 계열 저수준 패키지 관리 도구
- 어원: **R**PM **P**ackage **M**anager (초기 Red Hat Package Manager)
- 의존성 자동 해결 없음 → 설치는 [[dnf]] 권장, 조회·검증 용도로 사용

---

## rpm -q (조회)

```bash
rpm -q <package>

# Examples
rpm -q openssh-server           # 설치 여부 및 버전 확인
rpm -qa | grep kernel           # 설치된 커널 패키지 전체 목록
rpm -ql openssh-server          # 해당 패키지가 설치한 파일 목록
```

### 명령어 설명
- 사용 목적
	- 특정 패키지 설치 여부·버전 확인 시 사용
	- 최소 설치 환경에서 기본 포함 패키지 존재 확인 시 사용
- 특이사항
	- 미설치 시 `package <name> is not installed` 출력
	- root 권한 불필요

### 옵션
- `-q` : 조회 모드 (**q**uery)
- `-a` : 전체 패키지 대상 (**a**ll)
- `-l` : 파일 목록 출력 (**l**ist)
- `-i` : 상세 정보 출력 (**i**nfo)

---

## rpm -V (무결성 검증)

```bash
rpm -Va

# Examples
rpm -Va                          # 전체 패키지 파일 무결성 검증
rpm -Va --nofiles --nodigest     # 요약 검증 (빠름)
rpm -V openssh-server            # 특정 패키지만 검증
```

### 명령어 설명
- 사용 목적
	- 설치된 파일이 패키지 원본과 일치하는지 검증 시 사용
	- 설치 매체 불량(USB `SQUASHFS error` 등) 이후 손상 여부 판정에 사용
- 특이사항
	- 출력 몇 줄은 정상 (설정 파일 변경 등)
	- 대량 출력 시 설치 손상 의심 → 재설치 검토
	- 출력 코드: `S`(크기) `M`(권한) `5`(체크섬) `T`(시각) `c`(설정 파일)

### 옵션
- `-V` : 검증 모드 (**V**erify)
- `-a` : 전체 패키지 대상 (**a**ll)
- `--nofiles` : 파일 존재 검사 생략
- `--nodigest` : 체크섬 검사 생략

### 연관 명령어
- [[dnf]] `check` : 의존성 레벨 무결성 검사

---

## 연관 명령어
- [[dnf]] : 의존성 해결 포함 설치·업데이트
- [[dnf-group]] : 그룹 단위 설치
- [[grep]] : 패키지 목록 필터링
- [[which]] : 명령 실제 경로 확인 → `rpm -qf` 로 제공 패키지 역추적
- [[file]] : 패키지 파일 유형 판정
