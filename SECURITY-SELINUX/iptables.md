---
command: iptables
category: SECURITY-SELINUX
aliases: [iptables-save, iptables-services]
tags:
  - linux/security
  - linux/network
  - task/configure
  - topic/firewall
  - topic/security
  - topic/port
  - privilege/root
related: ["[[firewall-cmd]]", "[[docker]]", "[[nc]]", "[[ss]]", "[[dnf]]", "[[systemctl]]"]
distro: RHEL 계열 (iptables / iptables-services)
verified: Rocky Linux 9.6 (사고 대응 세션)
updated: 2026-08-01
---

# iptables

- 넷필터 기반 패킷 필터링 규칙 관리 도구
- 어원: **IP** + **tables** (netfilter 테이블)
- Docker 환경에서는 컨테이너 트래픽 전용 체인 `DOCKER-USER` 존재

---

## iptables -I

```bash
iptables -I <체인> <매칭> -j <타겟>

# Examples
iptables -I DOCKER-USER -d 34.70.205.211 -j DROP    # 컨테이너 아웃바운드 C2 차단
iptables -I OUTPUT -d 38.150.0.118 -j DROP          # 호스트 아웃바운드 C2 차단
```

### 명령어 설명
- 사용 목적
	- 특정 목적지 IP로의 트래픽 즉시 차단 시 사용
	- 침해 시 C2 아웃바운드 차단(재감염 경로 차단) 시 사용
- 특이사항
	- **`DOCKER-USER`는 컨테이너 트래픽, `OUTPUT`은 호스트 트래픽** 대상 → 양쪽 모두 삽입 필요
	- Docker가 관리하는 체인을 직접 수정하면 재적용으로 소실 → 사용자 규칙은 `DOCKER-USER`에 배치
	- `-I`는 체인 최상단 삽입(우선 평가), `-A`는 말단 추가 → 차단은 `-I` 권장
	- 런타임 규칙은 **재부팅으로 소실** → 영구화 필요

### 옵션
- `-I` : 규칙을 체인 상단에 삽입 (**I**nsert)
- `-A` : 규칙을 체인 말단에 추가 (**A**ppend)
- `-d` : 목적지 주소 매칭 (**d**estination)
- `-j` : 매칭 시 타겟 점프 (`DROP`/`ACCEPT`/`REJECT`) (**j**ump)

---

## iptables-save (영구화)

```bash
iptables-save > /etc/sysconfig/iptables

# Examples
dnf install -y iptables-services
sh -c 'iptables-save > /etc/sysconfig/iptables'
systemctl enable --now iptables
```

### 명령어 설명
- 사용 목적
	- 현재 규칙 상태를 파일로 덤프하여 영구 저장 시 사용
	- 증거 보존용 규칙 상태 캡처(`iptables-save > iptables-after.txt`) 시 사용
- 특이사항
	- 미수행 시 재부팅으로 차단 해제 → `iptables-services` + `systemctl enable`로 영속화
	- 검증: `test -f /etc/sysconfig/iptables && echo saved` → [[test]]
	- `sudo` 하에서 리다이렉션은 `sudo sh -c '...'` 로 감쌈 → [[sudo]]

---

## 연관 명령어
- [[firewall-cmd]] : firewalld 상위 추상화 (RHEL 기본 방화벽 관리)
- [[docker]] : 컨테이너 트래픽 대상 `DOCKER-USER` 체인
- [[nc]] : 차단 후 외부에서 포트 도달 여부 실검증
- [[ss]] : 리스너 상태 조회 (리스너 부재 ≠ 방화벽 차단)
- [[dnf]] : `iptables-services` 설치
- [[systemctl]] : 영구 규칙 서비스 활성화
