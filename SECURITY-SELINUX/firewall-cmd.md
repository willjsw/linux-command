---
command: firewall-cmd
category: SECURITY-SELINUX
aliases: [firewalld]
tags:
  - linux/security
  - linux/network
  - topic/firewall
  - topic/port
  - task/configure
  - task/diagnose
  - privilege/root
  - distro/rhel
  - distro/rocky
related: ["[[ss]]", "[[ssh]]", "[[systemctl]]", "[[getenforce]]", "[[ausearch]]", "[[iptables]]", "[[nc]]"]
distro: RHEL 계열 (firewalld 패키지)
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# firewall-cmd

- firewalld 방화벽 제어 도구
- RHEL 계열 기본 방화벽 (`iptables` 상위 추상화)
- 기본 상태: **실행 중(running)**

---

## firewall-cmd --list-all (조회)

```bash
firewall-cmd --state
firewall-cmd --list-all

# Examples
firewall-cmd --state                       # running / not running
firewall-cmd --list-all                    # 현재 존 전체 설정
firewall-cmd --list-all | grep -i ssh      # ssh 허용 여부
firewall-cmd --list-ports                  # 허용 포트만
firewall-cmd --list-services               # 허용 서비스만
```

### 명령어 설명
- 사용 목적
	- 방화벽 허용 서비스·포트 확인 시 사용
	- **서비스 접근 불가 원인 진단** 시 사용 → 리스닝 정상이나 외부 접속 실패 시 1순위 확인 대상
- 특이사항
	- `services:` 항목에 `ssh` 포함 시 SSH 허용 상태 (Rocky 기본값)
	- 포트 리스닝 여부는 [[ss]] 로 별도 확인

---

## firewall-cmd --add-service / --add-port

```bash
firewall-cmd --permanent --add-service=<service>
firewall-cmd --permanent --add-port=<port>/<protocol>
firewall-cmd --reload

# Examples
firewall-cmd --permanent --add-service=ssh       # SSH 허용
firewall-cmd --permanent --add-port=8080/tcp     # 애플리케이션 포트
firewall-cmd --permanent --add-port=5060/udp     # SIP (PBX 등)
firewall-cmd --reload                            # 적용
```

### 명령어 설명
- 사용 목적
	- 서비스·포트 방화벽 허용 규칙 추가 시 사용
	- 애플리케이션 서비스 포트 개방 시 사용
- 특이사항
	- **`--permanent` 는 설정 저장만 수행** → `--reload` 없이는 즉시 미적용
	- `--permanent` 없이 실행 시 재부팅 후 소멸
	- 서비스명 방식(`--add-service`)이 포트 직접 지정보다 관리 용이

### 옵션
- `--permanent` : 영구 설정 (재부팅 후 유지)
- `--reload` : 영구 설정을 현재 실행 상태에 적용
- `--add-service=<name>` : 사전 정의 서비스 허용
- `--add-port=<port>/<proto>` : 포트 직접 허용
- `--remove-service` / `--remove-port` : 규칙 제거
- `--zone=<zone>` : 대상 존 지정

---

## 진단 순서 (서비스 접근 불가 시)

```bash
systemctl is-active sshd                   # ① 서비스 기동
ss -tlnp | grep :22                        # ② 포트 리스닝
firewall-cmd --list-all | grep -i ssh      # ③ 방화벽 허용
ping -c 3 <서버IP>                         # ④ 네트워크 도달
```

---

## 연관 명령어
- [[ss]] : 포트 리스닝 상태 확인 (방화벽 확인 선행 단계)
- [[ssh]] : 원격 접속 차단 원인 진단
- [[systemctl]] : firewalld 서비스 제어
- [[getenforce]] : SELinux 차단 여부 별도 확인
- [[ausearch]] : SELinux 차단 로그 조회
- [[iptables]] : 저수준 패킷 필터링 (컨테이너 `DOCKER-USER` 등 firewalld 미포괄 영역)
- [[nc]] : 방화벽 차단 여부 외부 관점 실검증
