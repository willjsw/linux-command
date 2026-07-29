---
command: grep
category: FILE-TEXT
aliases: []
tags:
  - linux/text
  - task/search
  - topic/regex
  - topic/encoding
  - privilege/user
related: ["[[journalctl]]", "[[ss]]", "[[rpm]]", "[[find]]", "[[sed]]", "[[awk]]", "[[iconv]]", "[[xargs]]", "[[ps]]"]
distro: 전체
verified: Rocky Linux 9.6
updated: 2026-07-30
---

# grep

- 파일·표준입력에서 패턴 일치 행 출력 도구
- 어원: **g**lobally search **r**egular **e**xpression and **p**rint (ed 편집기 명령 `g/re/p`)
- 일반 사용자 실행 가능

---

## grep

```bash
grep <pattern> <file>

# Examples
grep -i ssh /etc/ssh/sshd_config          # 대소문자 무시
grep -rn "함수명" ./src                    # 재귀 + 행번호
grep -a "패턴" file                        # 바이너리 취급 파일도 텍스트로 검색
grep -v "제외패턴" file                    # 일치하지 않는 행
grep -c "패턴" file                        # 일치 행 개수
```

### 명령어 설명
- 사용 목적
	- 파일 내 특정 문자열·패턴 검색 시 사용
	- 명령 출력 필터링 시 사용 (파이프 연계)
	- 설정 파일에서 특정 항목 확인 시 사용
- 특이사항
	- **EUC-KR 등 비UTF-8 인코딩 파일은 바이너리로 판정되어 검색 결과 누락**
		- 대응: `-a` 옵션으로 텍스트 강제 취급
		- 한글 주석이 EUC-KR로 작성된 소스 검색 시 `grep -a` 또는 `grep -arn` 필수
	- 정규표현식 사용 시 `-E`(확장) 또는 `-P`(Perl 호환) 지정

### 옵션
- `-i` : 대소문자 무시 (**i**gnore case)
- `-r` : 디렉터리 재귀 검색 (**r**ecursive)
- `-n` : 행 번호 표시 (**n**umber)
- `-a` : 바이너리 파일을 텍스트로 취급 (**a**ll text) — 비UTF-8 인코딩 대응
- `-v` : 일치하지 않는 행 출력 (in**v**ert)
- `-c` : 일치 행 개수만 (**c**ount)
- `-l` : 일치하는 파일명만 (**l**ist)
- `-E` : 확장 정규표현식 (**E**xtended)

---

## 파이프 연계 활용

```bash
ss -tlnp | grep :22                                # 특정 포트 리스닝 확인
firewall-cmd --list-all | grep -i ssh              # 방화벽 규칙 확인
rpm -qa | grep kernel                              # 설치 패키지 검색
journalctl -u NetworkManager -n 50 | grep -i dhcp  # 로그 필터링
nmcli connection show | grep eno1                  # 연결 프로파일 검색
```

---

## 연관 명령어
- [[journalctl]] : 로그 출력 필터링 대상
- [[ss]] : 포트 목록 필터링 대상
- [[rpm]] : 패키지 목록 필터링 대상
- [[find]] : 파일명 기준 탐색 — `grep` 은 내용 기준, 역할 분리
- [[sed]] : 검색 + 치환·범위 추출
- [[awk]] : 필드 단위 조건 처리
- [[iconv]] : **EUC-KR 파일 검색 시 변환 선행 필수** — `-a` 는 한글 깨짐
- [[xargs]] : `find | xargs grep` 다중 파일 검색
- [[ps]] : `ps aux | grep` 자기제외 처리 필요
