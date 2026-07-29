---
command: dmesg
category: SERVICE-SYSTEMD
aliases: []
tags:
  - linux/log
  - linux/boot
  - task/diagnose
  - topic/troubleshooting
  - privilege/mixed
related: ["[[journalctl]]", "[[lsblk]]", "[[ip]]"]
distro: 전체 (util-linux 패키지)
verified: 미검증 (참조용)
updated: 2026-07-29
---

# dmesg

- 커널 링 버퍼 메시지 출력 도구
- 어원: **d**isplay **mes**sa**g**e
- 하드웨어 인식·드라이버 오류 진단의 1차 수단

---

## dmesg

```bash
dmesg

# Examples
dmesg -T | tail -50              # 사람이 읽을 수 있는 시각 + 최근 50줄
dmesg -l err,warn                # 오류·경고만
dmesg | grep -i sd               # 디스크 인식 메시지
dmesg | grep -i eth              # 네트워크 인터페이스 인식
dmesg -w                         # 실시간 추적
```

### 명령어 설명
- 사용 목적
	- 디스크·NIC 등 하드웨어 인식 결과 확인 시 사용
	- 커널 레벨 오류·드라이버 실패 진단 시 사용
	- 부팅 과정 메시지 확인 시 사용
- 특이사항
	- 기본 출력의 시각은 부팅 후 경과 초 → `-T` 로 실제 시각 변환
	- 링 버퍼 용량 초과 시 과거 메시지 소실 → 영구 로그는 [[journalctl]] `-k` 사용
	- 배포판·설정에 따라 root 권한 필요

### 옵션

> 전 항목 미검증 (참조용) — 실사용 검증 후 표시 제거 및 `updated` 갱신 대상.

- `-T` : 사람이 읽을 수 있는 시각 표시 (**T**ime)
- `-l <level>` : 우선순위 필터 (**l**evel), `err`·`warn` 등
- `-w` : 실시간 추적 (**w**ait/follow)
- `-C` : 링 버퍼 비우기 (**C**lear)

---

## 연관 명령어
- [[journalctl]] : `-k` 로 동일 커널 메시지 영구 조회
- [[lsblk]] : 디스크 인식 결과 대조
- [[ip]] : 네트워크 인터페이스 인식 결과 대조
