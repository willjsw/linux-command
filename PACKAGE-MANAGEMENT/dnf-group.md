---
command: dnf group
category: PACKAGE-MANAGEMENT
aliases: [dnf groupinstall, groupinstall]
tags:
  - linux/package
  - distro/rhel
  - distro/rocky
  - task/install
  - topic/desktop-environment
  - privilege/root
  - requires/network
related: ["[[dnf]]", "[[systemctl]]", "[[rpm]]"]
distro: RHEL 계열 (Rocky, CentOS, Fedora)
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# dnf group

- 개별 패키지가 아닌 **환경·그룹 단위** 묶음 설치 도구
- OS 설치 화면의 [소프트웨어 선택] 항목과 동일한 단위
- 설치 시점에 선택하지 않은 항목을 사후 추가 시 사용

---

## dnf group list

```bash
dnf group list

# Examples
dnf group list                    # 사용 가능 환경·그룹 목록
dnf group list --hidden           # 숨김 그룹 포함
dnf group info "Server with GUI"  # 특정 그룹 구성 패키지 확인
```

### 명령어 설명
- 사용 목적
	- 설치 가능한 환경 그룹 및 추가 요소 목록 확인 시 사용
	- 최소 설치 후 필요한 그룹 탐색 시 사용

---

## dnf group install

```bash
dnf group install "<group name>"

# Examples
dnf group install -y "Development Tools"     # 개발용 도구 일괄
dnf group install -y "Server with GUI"       # GUI 환경 추가
dnf groupinstall -y "Server with GUI"        # 구형 표기 (동일 동작)
```

### 명령어 설명
- 사용 목적
	- 최소 설치(Minimal) 환경에 GUI·개발 도구 등 그룹 사후 추가 시 사용
	- OS 설치 시 [추가 요소] 미선택 항목 보충 시 사용
- 특이사항
	- 그룹명에 공백 포함 → **따옴표 필수**
	- GUI 추가 시 기본 부팅 타겟 변경 필요 → [[systemctl]] `set-default` 참고
	- 설치 시점에 미리 체크하지 않아도 사후 추가 가능 → 최소 설치 시 추가 요소 무선택 권장

### 옵션
- `-y` : 프롬프트 자동 승인 (**y**es)

### GUI 전환 전체 절차
```bash
dnf group install -y "Server with GUI"
systemctl set-default graphical.target
reboot
```

### 콘솔 복귀
```bash
systemctl set-default multi-user.target
reboot
```

---

## 연관 명령어
- [[dnf]] : 개별 패키지 설치·업데이트
- [[systemctl]] : 부팅 타겟(`graphical.target` / `multi-user.target`) 전환
- [[rpm]] : 설치 결과 검증
