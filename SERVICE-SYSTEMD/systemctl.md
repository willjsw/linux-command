---
command: systemctl
category: SERVICE-SYSTEMD
aliases: []
tags:
  - linux/systemd
  - linux/service
  - task/configure
  - task/inspect
  - topic/boot-target
  - topic/desktop-environment
  - privilege/mixed
related: ["[[journalctl]]", "[[ss]]", "[[ssh]]", "[[dnf-group]]", "[[reboot]]", "[[nmcli]]", "[[hostnamectl]]", "[[dnf]]", "[[firewall-cmd]]", "[[localectl]]"]
distro: systemd 사용 배포판
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# systemctl

- systemd 서비스·타겟 제어 도구
- 어원: **system** **ctl**(control)
- 조회는 일반 사용자 가능, 제어는 root 필요

---

## systemctl status / is-active / is-enabled

```bash
systemctl status <service>
systemctl is-active <service>
systemctl is-enabled <service>

# Examples
systemctl status sshd        # 상세 상태 + 최근 로그
systemctl is-active sshd     # active / inactive (스크립트 친화적)
systemctl is-enabled sshd    # enabled / disabled (부팅 자동시작 여부)
```

### 명령어 설명
- 사용 목적
	- 서비스 실행 상태 확인 시 사용
	- **부팅 시 자동 시작 설정 여부 확인** 시 사용
	- 서비스 접근 불가 원인 1차 진단 시 사용
- 특이사항
	- `is-active` : 현재 실행 여부
	- `is-enabled` : **재부팅 후 자동 시작 여부** (별개 개념)
	- `status` 출력이 길 경우 `q` 로 종료
	- 포트 리스닝 확인은 [[ss]] 병행

---

## systemctl start / stop / restart / enable / disable

```bash
systemctl start <service>
systemctl enable <service>
systemctl enable --now <service>

# Examples
systemctl start sshd             # 즉시 기동
systemctl enable sshd            # 부팅 시 자동시작 설정
systemctl enable --now sshd      # 자동시작 설정 + 즉시 기동
systemctl restart NetworkManager # 재시작
```

### 명령어 설명
- 사용 목적
	- 서비스 기동·중지·재시작 시 사용
	- 부팅 시 자동 시작 등록 시 사용
- 특이사항
	- `start` 는 일시적 → 재부팅 시 미기동, 영구 설정은 `enable` 필요
	- `enable --now` 로 두 작업 동시 수행
	- 설정 파일 변경 후 반영에는 `restart` 또는 `reload` 필요

---

## systemctl get-default / set-default (부팅 타겟)

```bash
systemctl get-default
systemctl set-default <target>

# Examples
systemctl get-default                        # 현재 부팅 타겟 확인
systemctl set-default multi-user.target      # 콘솔(CLI) 모드
systemctl set-default graphical.target       # GUI 모드
```

### 명령어 설명
- 사용 목적
	- 부팅 시 GUI / 콘솔 모드 전환 시 사용
	- 최소 설치 서버에 GUI 추가 후 전환 시 사용
- 특이사항
	- **최소 설치 기본값 = `multi-user.target`** (콘솔) → 검은 화면에 로그인 프롬프트만 표시되는 것이 정상
	- GUI 설치 기본값 = `graphical.target`
	- 변경 후 재부팅 필요
	- GUI 패키지 미설치 상태에서 `graphical.target` 설정 시 부팅 실패 → [[dnf-group]] 로 GUI 선행 설치 필요

### GUI 전환 전체 절차
```bash
dnf group install -y "Server with GUI"
systemctl set-default graphical.target
reboot
```

---

## 연관 명령어
- [[journalctl]] : 서비스 로그 상세 조회
- [[ss]] : 서비스 포트 리스닝 확인
- [[ssh]] : sshd 서비스 관리 대상
- [[dnf-group]] : GUI 환경 그룹 설치
- [[reboot]] : 타겟 변경 후 재시작
- [[nmcli]] : NetworkManager 서비스 제어 대상
- [[hostnamectl]] : systemd 계열 호스트명 설정
- [[dnf]] : 설치한 서비스 기동
- [[firewall-cmd]] : firewalld 서비스 제어
- [[localectl]] : systemd 계열 로케일 설정
