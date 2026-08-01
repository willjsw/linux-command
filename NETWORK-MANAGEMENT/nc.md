---
command: nc
category: NETWORK-MANAGEMENT
aliases: [netcat, ncat]
tags:
  - linux/network
  - task/diagnose
  - task/verify
  - topic/port
  - topic/socket
  - privilege/user
related: ["[[ss]]", "[[iptables]]", "[[firewall-cmd]]", "[[ping]]", "[[curl]]", "[[whois]]"]
distro: 전체 (nmap-ncat / netcat)
verified: 사고 대응 세션 (외부 노출 검증 기준)
updated: 2026-08-01
---

# nc

- TCP/UDP 임의 포트 연결·수신 도구
- 어원: **n**et**c**at (네트워크판 `cat`)
- 외부 관점 포트 노출 검증에 사용 → 서버 내부 `ss`로는 판정 불가한 방화벽 상태 확인

---

## nc -zv

```bash
nc -zv -w <초> <호스트> <포트>

# Examples
nc -zv -w 5 3.37.243.226 5432       # 외부에서 PostgreSQL 포트 도달 검증
nc -zv -w 5 3.37.243.226 6379       # 외부에서 Redis 포트 도달 검증
```

### 명령어 설명
- 사용 목적
	- 특정 호스트·포트의 외부 도달 가능 여부 검증 시 사용
	- 보안그룹·방화벽 차단 여부 실검증 시 사용 (**외부 네트워크에서 실행**)
- 특이사항
	- **컨테이너 정지 상태에서 서버 내부 `ss` 결과만으로 차단 여부 판정 불가** → 외부 `nc` 필수
	- 리스너 부재(`ss` 무출력) ≠ 방화벽 차단 → 응답 유형으로 구분
	- `-w` 미지정 시 응답 없는 포트에서 장시간 대기

### 응답별 판정
| 응답 | 판정 | 조치 |
| --- | --- | --- |
| `Operation timed out` | 보안그룹 차단 상태 | 정상 |
| `Connection refused` | 개방 + 서비스 정지 | 즉시 규칙 제거. 재기동 시 재노출 |
| `succeeded!` | 개방 및 응답 중 | 즉시 규칙 제거 |

### 옵션
- `-z` : 데이터 전송 없이 연결만 스캔 (**z**ero-I/O)
- `-v` : 상세 출력 (**v**erbose)
- `-w` : 연결 타임아웃(초) (**w**ait)

---

## 연관 명령어
- [[ss]] : 서버 내부 리스너 조회 (외부 관점과 상호보완)
- [[iptables]] : 차단 규칙 적용 후 `nc`로 효과 검증
- [[firewall-cmd]] : firewalld 허용·차단 상태 확인
- [[ping]] : 하위 계층(ICMP) 도달성 확인
- [[curl]] : HTTP 계층 응답 검증 (포트 개방 확인 후)
- [[whois]] : 스캔·공격 출발 IP 소속 확인
