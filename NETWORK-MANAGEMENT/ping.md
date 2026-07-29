---
command: ping
category: NETWORK-MANAGEMENT
aliases: []
tags:
  - linux/network
  - task/diagnose
  - topic/icmp
  - privilege/user
  - task/verify
  - topic/routing
related: ["[[ip]]", "[[nmcli]]", "[[ss]]", "[[ssh]]", "[[curl]]", "[[timeout]]"]
distro: 전체
verified: Rocky Linux 9.6
updated: 2026-07-30
---

# ping

- ICMP Echo 패킷 기반 네트워크 연결성 검사 도구
- 어원: 잠수함 음향탐지(sonar ping) 비유
- 일반 사용자 실행 가능

---

## ping

```bash
ping <host>

# Examples
ping -c 3 10.1.18.254        # 게이트웨이 (사내망)
ping -c 3 154.10.6.11        # 사내 DNS
ping -c 3 8.8.8.8            # 외부 인터넷
ping -c 3 google.com         # DNS 해석 포함 검증
```

### 명령어 설명
- 사용 목적
	- 대상 호스트 도달 가능 여부 확인 시 사용
	- 네트워크 설정 후 단계별 연결성 검증 시 사용
- 특이사항
	- `-c` 미지정 시 무한 반복 → `Ctrl+C` 중단 필요
	- **자신에게 IP 미할당 상태에서는 대상 무관하게 `unreachable`** → 경로 부재가 원인, 상대 장애 아님
	- ICMP 차단 정책 장비는 정상 동작 중에도 무응답 → 확정 판정은 [[ip]] `neigh`(ARP) 사용

### 옵션
- `-c <n>` : 지정 횟수만 전송 후 종료 (**c**ount)
- `-W <sec>` : 응답 대기 시간 제한 (**W**ait)
- `-I <if>` : 송신 인터페이스 지정 (**I**nterface)

---

## 단계별 진단 순서

```bash
ping -c 3 10.1.18.254     # ① 게이트웨이
ping -c 3 154.10.6.11     # ② 사내 DNS
ping -c 3 8.8.8.8         # ③ 외부 인터넷
ping -c 3 google.com      # ④ 이름 해석
```

### 실패 지점별 원인
- ① 실패 → IP·서브넷 설정 오류 또는 물리 링크 문제 ([[ip]] `-br link` 로 `NO-CARRIER` 확인)
- ① 성공 / ②③ 실패 → 게이트웨이 값 오류 (다른 대역 주소 입력 여부 확인)
- ①② 성공 / ③ 실패 → 외부 인터넷 차단 정책 (사내망 전용 환경이면 정상)
- ③ 성공 / ④ 실패 → DNS 설정 오류 ([[nmcli]] `ipv4.dns` 확인)

---

## 연관 명령어
- [[ip]] : 주소·라우팅·물리 링크 확인, ARP 기반 확정 판정
- [[nmcli]] : IP·게이트웨이·DNS 설정
- [[ss]] : 포트 단위 서비스 접근성 확인
- [[ssh]] : 원격 접속 단계 진단
- [[curl]] : 응용 계층(HTTP) 도달성 확인 — `ping` 성공 후 수행
- [[timeout]] : `timeout 5 ping` 무응답 호스트 대기 차단
