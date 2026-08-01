---
command: zip
category: FILE-TEXT
aliases: []
tags:
  - linux/text
  - task/configure
  - topic/filesystem
  - topic/security
  - privilege/user
related: ["[[tar]]", "[[unzip]]", "[[sha256sum]]", "[[dnf]]"]
distro: 전체 (zip 패키지)
verified: Rocky Linux 9.6 (사고 대응 세션)
updated: 2026-08-01
---

# zip

- ZIP 아카이브 생성·암호화 압축 도구
- 어원: 빠른 압축의 의성어(zip) → PKZIP 유래
- 악성 샘플 격리 시 실수 실행 방지 목적의 암호화 압축에 사용 (기밀 보호용 아님)

---

## zip

```bash
zip [-e] [-j] <출력.zip> <파일>...

# Examples
zip -e -j /root/ir-samples.zip /root/ir-20260730/samples/zsTCX.sample   # 암호화, 경로 제거
```

### 명령어 설명
- 사용 목적
	- 파일을 ZIP 아카이브로 압축 시 사용
	- 악성 샘플을 암호화 압축하여 자동 삭제·실수 실행 방지 시 사용
- 특이사항
	- `-e` 사용 시 비밀번호 프롬프트 → **입력 후 `Verify` 재입력 필요**. 중단 시 `Interrupted (aborting)`으로 파일 미생성
	- **zip 기본 암호화는 강도 낮음** → 기밀 보호 아닌 실수 실행 방지 목적에 한정
	- 실무 지표는 파일이 아닌 SHA256 → 압축 생략하고 해시만 확보해도 진행 가능 → [[sha256sum]]
	- 로그·설정 등 다수 파일 아카이브는 `tar czf`가 일반적 → [[tar]]

### 옵션
- `-e` : 암호화(비밀번호 프롬프트) (**e**ncrypt)
- `-j` : 디렉터리 경로 제거하고 파일만 저장 (**j**unk paths)

---

## 연관 명령어
- [[tar]] : 로그·설정 다수 파일 아카이브 (증거 묶음 표준)
- [[unzip]] : ZIP 해제·목록 확인
- [[sha256sum]] : 아카이브·샘플 해시 산출
- [[dnf]] : `zip` 패키지 설치 여부 확인
