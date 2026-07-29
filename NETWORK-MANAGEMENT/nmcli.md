---
command: nmcli
category: NETWORK-MANAGEMENT
aliases: []
tags:
  - linux/network
  - topic/networkmanager
  - topic/static-ip
  - topic/dhcp
  - task/configure
  - task/diagnose
  - privilege/root
  - distro/rhel
  - distro/rocky
related: ["[[nmtui]]", "[[ip]]", "[[ping]]", "[[hostnamectl]]", "[[systemctl]]"]
distro: NetworkManager 사용 배포판 (RHEL/Rocky/Ubuntu 등)
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# nmcli

- NetworkManager 제어용 CLI 도구
- 어원: **N**etwork**M**anager **C**ommand **L**ine **I**nterface
- 최소 설치(Minimal) 환경에도 기본 포함
- 대화형 TUI 필요 시 [[nmtui]] 사용

---

## nmcli device status

```bash
nmcli device status

# Examples
nmcli device status
# DEVICE  TYPE      STATE                                  CONNECTION
# eno1    ethernet  connecting (getting IP configuration)   eno1
# eno2    ethernet  unavailable                             --
```

### 명령어 설명
- 사용 목적
	- 전체 네트워크 인터페이스의 연결 상태 일괄 확인 시 사용
	- 다중 NIC 서버에서 랜선 연결 포트 식별 시 사용
- 특이사항
	- `unavailable` : 랜선 미연결 (물리 링크 없음)
	- `connecting (getting IP configuration)` : DHCP 응답 대기 중
	- `disconnected` : DHCP 타임아웃 등으로 연결 실패
	- `connected` : 정상 연결
	- 물리 링크 확정 판정은 [[ip]] `-br link` 의 `NO-CARRIER` 플래그로 수행

---

## nmcli connection modify

```bash
nmcli connection modify <name> <설정> <값>

# Examples — 고정 IP 설정
nmcli connection modify eno1 \
  ipv4.method manual \
  ipv4.addresses 10.1.18.85/24 \
  ipv4.gateway 10.1.18.254 \
  ipv4.dns "154.10.6.11 8.8.8.8" \
  ipv6.method disabled \
  connection.autoconnect yes

# Examples — 재부팅 후 IP 유지만 설정
nmcli connection modify eno1 connection.autoconnect yes
```

### 명령어 설명
- 사용 목적
	- 인터페이스 연결 프로파일의 IP·게이트웨이·DNS 영구 설정 시 사용
	- 재부팅 후 네트워크 자동 활성화 설정 시 사용
- 특이사항
	- **`modify` 는 설정 저장만 수행, 실행 중 인터페이스에 즉시 반영되지 않음**
	- 적용에는 `down` → `up` 재활성화 필수
	- `connection.autoconnect yes` 누락 시 재부팅 후 네트워크 미기동 → GUI 없는 환경에서 원격 접속까지 차단
	- 게이트웨이는 **IP와 동일 대역** 값 입력 필수 (다른 대역 입력 시 외부 통신 불가)

### 주요 설정 키
- `ipv4.method` : `manual`(고정) / `auto`(DHCP) / `disabled`
- `ipv4.addresses` : `IP/프리픽스` 형식 (예: `10.1.18.85/24`)
- `ipv4.gateway` : 기본 게이트웨이
- `ipv4.dns` : DNS 서버, 다중 지정 시 공백 구분 + 따옴표
- `ipv6.method disabled` : IPv6 미사용 환경에서 불필요한 대기 제거
- `connection.autoconnect` : 부팅 시 자동 연결 여부

---

## nmcli connection up / down

```bash
nmcli connection up <name>
nmcli connection down <name>

# Examples
nmcli connection down eno1 && nmcli connection up eno1   # 설정 즉시 반영
```

### 명령어 설명
- 사용 목적
	- `modify` 로 저장한 설정을 실행 중 인터페이스에 실제 적용 시 사용
	- 연결 재시작으로 네트워크 문제 해소 시도 시 사용
- 특이사항
	- DHCP 서버 부재 시 아래 오류 발생 → 고정 IP 방식으로 전환 필요
	  ```
	  Error: Connection activation failed: IP configuration could not be reserved
	         (no available address, timeout, etc.)
	  ```

---

## nmcli connection show

```bash
nmcli connection show

# Examples
nmcli connection show                                          # 프로파일 목록
nmcli -f NAME,AUTOCONNECT,AUTOCONNECT-PRIORITY connection show  # autoconnect 확인
nmcli connection show eno1                                     # 특정 프로파일 전체 설정
```

### 명령어 설명
- 사용 목적
	- 저장된 연결 프로파일 목록·설정값 확인 시 사용
	- 재부팅 후 IP 유지 설정(`AUTOCONNECT`) 검증 시 사용
- 특이사항
	- `AUTOCONNECT` 가 `아니요` 인 항목은 재부팅 시 미기동

### 옵션
- `-f <필드>` : 출력 필드 지정 (**f**ields)

---

## 연관 명령어
- [[nmtui]] : 동일 작업의 대화형 TUI 버전 (오타 위험 없음)
- [[ip]] : 실제 적용된 주소·라우팅·물리 링크 확인
- [[ping]] : 게이트웨이 → DNS → 외부 순 연결성 검증
- [[hostnamectl]] : 호스트 이름 설정
- [[journalctl]] : NetworkManager 로그로 DHCP 실패 원인 확인
