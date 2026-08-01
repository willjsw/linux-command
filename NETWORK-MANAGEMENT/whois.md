---
command: whois
category: NETWORK-MANAGEMENT
aliases: [dig]
tags:
  - linux/network
  - task/inspect
  - task/diagnose
  - topic/security
  - topic/remote-access
  - privilege/user
related: ["[[nc]]", "[[ss]]", "[[ping]]", "[[grep]]"]
distro: 전체 (whois 패키지)
verified: 사고 대응 세션 (공격 IP 소속 확인)
updated: 2026-08-01
---

# whois

- IP·도메인 등록 정보 조회 도구
- 어원: **who is** (소유 주체 질의)
- 공격·스캔 출발 IP의 소속 사업자·abuse 연락처 확인에 사용

---

## whois

```bash
whois <IP 또는 도메인>

# Examples
whois 34.70.205.211 | grep -iE 'orgname|netname|country|abuse'   # 소속·abuse 연락처
whois 38.150.0.118  | grep -iE 'orgname|netname|country|abuse'
```

### 명령어 설명
- 사용 목적
	- 공격 인프라 IP의 소유 사업자·국가·abuse 창구 확인 시 사용
	- abuse 신고 대상·전달 여부 판단 시 사용
- 특이사항
	- `OrgName`/`NetName`/`Country`/`abuse` 필드가 신고 핵심 정보
	- **트랜짓 사업자 최상위 할당만 등록되고 하위 재할당 미등록 시 실사용자 특정 불가**
		- 예: `38.0.0.0/8` 최상위(`COGENT-A`)만 등록, `Parent` 비어 있음 → 사업자 단독 신고 후 전달 요청
	- 보완 조회: `dig +short -x <IP>`(역방향 DNS), `whois -h whois.radb.net <IP>`(route 객체 `descr`·`origin` AS)

### 옵션
- `-h` : 조회 서버 지정 (**h**ost) — 예: `-h whois.radb.net`

---

## 연관 명령어
- [[nc]] : 공격 출발 IP의 포트 도달성 확인
- [[ss]] : 진행 중 연결의 상대 IP 확인
- [[ping]] : 대상 IP 도달성 확인
- [[grep]] : whois 출력에서 핵심 필드 필터
