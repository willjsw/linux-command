---
command: sha256sum
category: FILE-TEXT
aliases: [shasum, sha256]
tags:
  - linux/text
  - task/verify
  - topic/security
  - topic/filesystem
  - privilege/user
related: ["[[file]]", "[[tar]]", "[[scp]]", "[[stat]]"]
distro: 전체 (coreutils)
verified: Rocky Linux 9.6 / macOS (Darwin 25.5) (사고 대응 세션)
updated: 2026-08-01
---

# sha256sum

- 파일 SHA-256 해시 산출·검증 도구
- 어원: **SHA-256** + **sum** (checksum)
- 악성 샘플 지표(IOC) 산출·전송 무결성 검증에 사용

---

## sha256sum

```bash
sha256sum <파일>...

# Examples
sha256sum /root/ir-20260730/samples/*      # 샘플 해시 산출 (복제본 동일성 확인)
sha256sum ir-evidence-20260730.tar.gz      # 아카이브 해시 (전송 기준값)
```

### 명령어 설명
- 사용 목적
	- 악성 샘플 SHA256 지표 확보(VirusTotal 해시 검색용) 시 사용
	- 전송 전후 해시 대조로 비트 단위 동일성 검증 시 사용
- 특이사항
	- **`shasum -a 256`은 macOS 전용, Linux는 `sha256sum`** → 서버에서 `shasum` 실행 시 `command not found`
	- 동일 해시 다수 파일 → 단일 바이너리 복제본 판정
	- 글로브 대상이 `/root` 하위면 비root 셸 전개 실패 → `sudo sh -c 'sha256sum ...'` → [[sudo]]
	- VirusTotal은 **해시 검색만** 사용 (파일 업로드 시 내부 정보 외부 공개 위험)

---

## 연관 명령어
- [[file]] : 해시 대상 파일 형식(ELF·UPX 등) 확인
- [[tar]] : 아카이브 생성 후 해시 산출
- [[scp]] : 전송 후 해시 대조로 무결성 검증
- [[stat]] : 파일 크기와 함께 동일성 교차 확인
