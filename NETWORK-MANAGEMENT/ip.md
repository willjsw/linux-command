---
command: ip
category: NETWORK-MANAGEMENT
aliases: [iproute2]
tags:
  - linux/network
  - task/diagnose
  - task/configure
  - topic/routing
  - topic/static-ip
  - privilege/mixed
  - replaces/ifconfig
related: ["[[nmcli]]", "[[ping]]", "[[ss]]", "[[nmtui]]", "[[ssh]]", "[[dmesg]]"]
distro: 전체 (iproute2 패키지)
verified: Rocky Linux 9.6
updated: 2026-07-29
---

# ip

- 네트워크 인터페이스·주소·라우팅 조회 및 조작 도구
- `ifconfig`, `route`, `arp` 대체 (iproute2 패키지)
- 조회는 일반 사용자 가능, 변경은 root 필요

---

## ip addr

```bash
ip addr
ip -br addr

# Examples
ip -br addr              # 인터페이스별 IP 요약 (brief)
ip addr show eno1        # 특정 인터페이스 상세
```

### 명령어 설명
- 사용 목적
	- 인터페이스에 실제 할당된 IP 주소 확인 시 사용
	- [[nmcli]] `modify` 설정이 실제 반영되었는지 검증 시 사용
- 특이사항
	- IP 미할당 상태에서는 `lo` 의 `127.0.0.1` 만 표시
	- **IP 없는 상태의 [[ping]] 은 대상 무관하게 `unreachable`** → 경로 부재가 원인, 상대 장애 아님

### 옵션
- `-br` : 요약 출력 (**br**ief)
- `-4` / `-6` : IPv4 / IPv6 만 표시

---

## ip link

```bash
ip link
ip -br link

# Examples
ip -br link
# eno1  UP    20:04:0f:ea:1d:08  <BROADCAST,MULTICAST,UP,LOWER_UP>       ← 링크 정상
# eno2  DOWN  20:04:0f:ea:1d:09  <NO-CARRIER,BROADCAST,MULTICAST,UP>     ← 랜선 없음
```

### 명령어 설명
- 사용 목적
	- **물리 링크(랜선) 연결 여부 확정 판정** 시 사용
	- MAC 주소 확인 시 사용
	- 다중 NIC 서버에서 실제 케이블 연결 포트 식별 시 사용
- 특이사항
	- `NO-CARRIER` : 물리 신호 없음 → 랜선 미연결·스위치 포트 장애·케이블 불량
	- `LOWER_UP` : 물리 링크 정상
	- **`NO-CARRIER` 는 소프트웨어 설정으로 해결 불가** → 서버 뒷면 케이블·포트 LED 확인 필요
	- 인터페이스가 `UP` 이면서 `NO-CARRIER` 동시 표시 가능 (활성화됐으나 신호 없음)

---

## ip route

```bash
ip route

# Examples
ip route                 # 라우팅 테이블 확인
# default via 10.1.18.254 dev eno1    ← 기본 게이트웨이
```

### 명령어 설명
- 사용 목적
	- 기본 게이트웨이 설정 확인 시 사용
	- 외부 통신 불가 원인 진단 시 사용
- 특이사항
	- `default via` 항목 부재 또는 잘못된 대역 → 외부 통신 불가
	- 다중 인터페이스 활성화 시 `default` 중복 → 라우팅 불안정

---

## ip addr add / del (임시 주소)

```bash
ip addr add <IP/prefix> dev <interface>
ip addr del <IP/prefix> dev <interface>

# Examples
ip addr add 10.1.18.200/24 dev eno1     # 임시 IP 부여
ip addr del 10.1.18.200/24 dev eno1     # 제거
```

### 명령어 설명
- 사용 목적
	- 설정 파일 변경 없이 임시 IP 부여 시 사용
	- IP 충돌 조사를 위해 대상 대역에 일시 진입 시 사용
- 특이사항
	- **재부팅 시 소멸** (NetworkManager 프로파일과 무관)
	- [[nmcli]] 로 설정한 값과 동시 존재 가능 → 혼동 주의
	- 임시 IP 자체가 충돌 유발 가능 → 확인 후 즉시 제거 필수
	- 영구 설정은 [[nmcli]] 또는 [[nmtui]] 사용

### IP 충돌 확인 절차
```bash
ip addr add 10.1.18.200/24 dev eno1      # 대역 진입
ping -c 2 10.1.18.85
ip neigh show 10.1.18.85                 # MAC 표시 → 사용 중
ip addr del 10.1.18.200/24 dev eno1      # 즉시 제거
```

---

## ip neigh

```bash
ip neigh

# Examples
ip neigh                        # ARP 캐시 전체
ip neigh show 10.1.18.85        # 특정 IP 사용 여부
```

### 명령어 설명
- 사용 목적
	- ARP 캐시 조회로 특정 IP 사용 중 여부 판정 시 사용
	- IP 충돌 사전 확인 시 사용
- 특이사항
	- 어원: **neigh**bour (이웃 = 동일 네트워크 노드)
	- **ICMP 차단 장비에도 응답** → [[ping]] 보다 신뢰도 높음
	- MAC 주소 표시 시 해당 IP 사용 장비 존재 확정

---

## 연관 명령어
- [[nmcli]] : 영구 네트워크 설정
- [[nmtui]] : 대화형 네트워크 설정
- [[ping]] : 연결성 단계별 검증
- [[ss]] : 포트 리스닝 상태 확인
- [[ssh]] : 원격 접속 경로 확인
- [[dmesg]] : NIC 인식 커널 메시지 확인
