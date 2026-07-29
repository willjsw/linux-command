---
command: du
category: DISK-STORAGE
aliases: []
tags:
  - linux/disk
  - task/inspect
  - topic/capacity
  - privilege/mixed
  - topic/filesystem
related: ["[[df]]", "[[lsblk]]", "[[lvm]]", "[[sort]]", "[[stat]]", "[[ls]]", "[[find]]"]
distro: 전체 (coreutils 패키지)
verified: 미검증 (참조용)
updated: 2026-07-30
---

# du

- 디렉터리·파일 단위 디스크 사용량 집계 도구
- 어원: **d**isk **u**sage
- [[df]] 는 파티션 단위, `du` 는 디렉터리 단위 → 용도 구분

---

## du -sh

```bash
du -sh <path>

# Examples
du -sh /var/log                  # 특정 디렉터리 총 사용량
du -sh /data/*                   # 하위 항목별 사용량
du -h --max-depth=1 /            # 1단계 깊이까지
du -ah /log | sort -rh | head -20    # 용량 상위 20개
```

### 명령어 설명
- 사용 목적
	- 파티션 용량 부족 시 원인 디렉터리 추적에 사용
	- 로그 폭주 등 비정상 용량 증가 지점 식별에 사용
- 특이사항
	- 하위 전체 순회 → 대용량 디렉터리에서 시간 소요
	- 권한 없는 디렉터리는 오류 출력 → `2>/dev/null` 로 억제 가능
	- [[df]] 와 값이 불일치 가능 (삭제됐으나 열린 파일, 예약 블록 등)

### 옵션

> 전 항목 미검증 (참조용) — 실사용 검증 후 표시 제거 및 `updated` 갱신 대상.

- `-s` : 총합만 출력 (**s**ummarize)
- `-h` : 읽기 쉬운 단위 (**h**uman-readable)
- `-a` : 파일 단위까지 표시 (**a**ll)
- `--max-depth=<n>` : 표시 깊이 제한

---

## 연관 명령어
- [[df]] : 파티션 단위 사용량 (선행 확인)
- [[lsblk]] : 블록 디바이스 구조
- [[lvm]] : 용량 부족 시 논리볼륨 확장
- [[sort]] : `du -sh | sort -h` 용량 순위 조회
- [[stat]] : 단일 파일 정밀 메타데이터
- [[ls]] : 디렉터리 내용 목록 — 누적 용량은 `du` 담당
- [[find]] : 대용량 파일 조건 탐색
