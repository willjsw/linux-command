---
command: dnf
category: PACKAGE-MANAGEMENT
aliases: [yum]
tags:
  - linux/package
  - distro/rhel
  - distro/rocky
  - task/install
  - task/update
  - task/verify
  - topic/troubleshooting
  - privilege/root
  - requires/network
related: ["[[rpm]]", "[[dnf-group]]", "[[systemctl]]", "[[localectl]]", "[[which]]"]
distro: RHEL 계열 (Rocky, CentOS, Fedora)
verified: Rocky Linux 9.6
updated: 2026-07-30
---

# dnf

- RHEL 계열 표준 패키지 매니저
- 어원: **D**andified **YUM** (yum 후속)
- `yum` 은 `dnf` 심볼릭 링크로 유지 (하위 호환)

---

## dnf install

```bash
dnf install <package>

# Examples
dnf install -y vim                                  # 확인 프롬프트 생략
dnf install -y vim wget curl net-tools              # 다중 패키지 동시 설치
dnf install -y gcc make gcc-c++ mysql-devel         # 빌드 도구 일괄
dnf install -y glibc-langpack-ko                    # 한국어 로케일 데이터
```

### 명령어 설명
- 사용 목적
	- 저장소에서 패키지 및 의존성 자동 해결 후 설치 시 사용
	- 최소 설치(Minimal) 환경에서 누락 유틸리티 보충 시 사용
- 특이사항
	- 외부 인터넷 또는 사내 미러 접근 불가 시 실패
	- root 권한 필요
	- 최소 설치 환경에서 `Failed to set locale, defaulting to C.UTF-8` 경고 출력 → 동작 무관, `glibc-langpack-ko` 설치로 제거

### 옵션
- `-y` : 모든 프롬프트에 yes 자동 응답 (**y**es)
- `--nogpgcheck` : GPG 서명 검증 생략
- `--disablerepo=*` : 지정 저장소 비활성화
- `--enablerepo=<repo>` : 지정 저장소 활성화
- `--repofrompath=<name>,<path>` : 로컬 경로를 임시 저장소로 등록 (오프라인 설치)

### 오프라인 설치 (설치 USB를 저장소로 사용)
```bash
dnf --disablerepo=* --repofrompath=usb,/run/install/repo \
    install -y <package>
```

---

## dnf update

```bash
dnf update

# Examples
dnf update -y            # 전체 패키지 업데이트
dnf update -y kernel     # 특정 패키지만 업데이트
```

### 명령어 설명
- 사용 목적
	- 설치된 전체 패키지를 최신 버전으로 갱신 시 사용
	- OS 설치 직후 보안 패치 적용 시 사용
- 특이사항
	- 커널 갱신 시 재부팅 필요 → [[reboot]] 참고
	- `dnf upgrade` 와 동일 동작 (RHEL 8+ 부터 별칭)

---

## dnf check

```bash
dnf check

# Examples
dnf check                # 출력 없으면 정상
```

### 명령어 설명
- 사용 목적
	- 로컬 패키지 DB 의존성 무결성 검사 시 사용
	- 설치 매체 불량(USB 읽기 오류) 의심 시 설치 상태 검증에 사용
- 특이사항
	- 정상 시 아무 출력 없이 종료
	- 로케일 경고만 출력되고 그 아래 내용이 없으면 정상 판정
	- 파일 단위 검증은 [[rpm]] `-Va` 사용

### 연관 명령어
- [[rpm]] : 개별 패키지 무결성·설치 여부 확인
- [[dnf-group]] : 패키지 그룹 단위 설치

---

## dnf reinstall

```bash
dnf reinstall <package>

# Examples
dnf reinstall -y kernel-core kernel kernel-modules   # 커널 재설치
dnf reinstall -y grub2-efi-x64 shim-x64              # UEFI 부트로더 복구
```

### 명령어 설명
- 사용 목적
	- 파일 손상·삭제된 패키지를 동일 버전으로 재설치 시 사용
	- 부트로더 손상 복구 시 사용 → [[grub2-install]] 참고

---

## 연관 명령어
- [[rpm]] : 하위 레벨 패키지 관리, 설치 여부 조회
- [[dnf-group]] : 환경/그룹 단위 설치 (GUI 추가 등)
- [[systemctl]] : 설치한 서비스 기동·활성화
- [[reboot]] : 커널 업데이트 후 재시작
- [[localectl]] : 로케일 경고 해소 관련
- [[which]] : 설치 전후 명령 가용 여부 확인
