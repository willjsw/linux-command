---
command: mkdir
category: FILE-TEXT
aliases: [cp, mv, rm, touch, ln, chmod]
tags:
  - linux/text
  - task/configure
  - topic/filesystem
  - privilege/mixed
  - danger/destructive
  - danger/data-loss
related: ["[[ls]]", "[[find]]", "[[stat]]", "[[du]]", "[[basename]]"]
distro: 전체
verified: macOS (Darwin 25.5) / Rocky Linux 9.6
updated: 2026-07-30
---

# 파일 조작 명령 (mkdir / cp / mv / rm / touch / ln / chmod)

- 파일·디렉터리 생성·복사·이동·삭제·권한 변경 기본 명령군
- 개별 문서 분리 불필요 수준의 단순 명령 → 단일 문서 통합
- 대상 경로 쓰기 권한 필요, 시스템 경로는 root 필요

---

## mkdir

```bash
mkdir -p <경로>

# Examples
mkdir -p /tmp/work/output          # 중간 디렉터리 포함 생성
mkdir -p "$OUT"                     # 존재해도 오류 없음 (스크립트 안전)
```

### 명령어 설명
- 사용 목적
	- 작업용 디렉터리 생성 시 사용
	- 스크립트 내 출력 경로 사전 확보 시 사용
- 특이사항
	- **`-p` 미지정 시 중간 디렉터리 부재하면 실패**
	- **`-p` 는 이미 존재해도 오류 미발생** → 스크립트에서 존재 확인 분기 불필요
	- 어원: **m**a**k**e **dir**ectory

### 옵션
- `-p` : 중간 경로 포함 생성, 기존 존재 시 무시 (**p**arents)

---

## cp

```bash
cp [-r] <원본> <대상>

# Examples
cp config.yml config.yml.bak       # 백업 생성
cp -r src/ /tmp/src-backup/         # 디렉터리 재귀 복사
```

### 명령어 설명
- 사용 목적
	- 파일 백업 생성 시 사용 (수정 전 원본 보존)
	- 디렉터리 구조 복제 시 사용
- 특이사항
	- **기존 대상 파일을 확인 없이 덮어씀** ⚠ → `-i` 로 확인 요청 가능
	- 디렉터리는 `-r` 필수 → 미지정 시 실패
	- 대상 경로 말미 `/` 유무에 따라 동작 상이 → 디렉터리 지정 시 `/` 명시 권장
	- 어원: **c**o**p**y

### 옵션
- `-r` : 디렉터리 재귀 복사 (**r**ecursive)
- `-p` : 권한·시각 속성 보존 (**p**reserve) ※ 미검증
- `-i` : 덮어쓰기 전 확인 (**i**nteractive) ※ 미검증

---

## mv

```bash
mv <원본> <대상>

# Examples
mv old-name.md new-name.md         # 이름 변경
mv build/report.json docs/          # 다른 디렉터리로 이동
```

### 명령어 설명
- 사용 목적
	- 파일·디렉터리 이름 변경 시 사용
	- 파일 위치 이동 시 사용
- 특이사항
	- **기존 대상을 확인 없이 덮어씀** ⚠ → 원본 소멸로 복구 불가
	- 동일 파일시스템 내에서는 메타데이터만 변경 → 대용량도 즉시 완료
	- 다른 파일시스템 간 이동은 복사 + 삭제 → 중단 시 불완전 상태 발생 가능
	- 어원: **m**o**v**e

### 옵션
- `-i` : 덮어쓰기 전 확인 (**i**nteractive) ※ 미검증
- `-n` : 기존 대상 존재 시 미실행 (**n**o-clobber) ※ 미검증

---

## rm

```bash
rm [-rf] <대상>

# Examples
rm /tmp/work/temp.txt              # 파일 삭제
rm -rf /tmp/work/                   # 디렉터리 재귀 강제 삭제 ⚠
```

### 명령어 설명
- 사용 목적
	- 임시 파일·작업 디렉터리 정리 시 사용
- 특이사항
	- **복구 수단 없음** ⚠ → 휴지통 개념 부재, 즉시 영구 삭제
	- **`rm -rf` 는 경로 오타 시 광범위 삭제 발생** → 최대 위험 명령
		- 대응: 삭제 전 동일 경로로 [[ls]] 실행하여 대상 확인 필수
		- 변수 사용 시 `rm -rf "$DIR"/` 는 `$DIR` 미설정 시 루트 대상화 위험 → 사전 검증 필수
	- 상대경로보다 절대경로 사용 시 오작동 위험 감소
	- 어원: **r**e**m**ove

### 옵션
- `-r` : 디렉터리 재귀 삭제 (**r**ecursive) ⚠
- `-f` : 확인 없이 강제, 부재 시 오류 무시 (**f**orce) ⚠
- `-i` : 삭제 전 개별 확인 (**i**nteractive) ※ 미검증

---

## touch

```bash
touch <파일>

# Examples
touch /.autorelabel                # SELinux 재라벨링 예약 (부팅 시 처리)
touch /tmp/lockfile                 # 빈 파일 생성
```

### 명령어 설명
- 사용 목적
	- 빈 파일 생성 시 사용
	- **SELinux 재라벨링 트리거 파일 생성 시 사용** (`/.autorelabel`)
	- 파일 수정 시각 갱신 시 사용
- 특이사항
	- 기존 파일에 실행 시 **내용 변경 없이 시각만 갱신** → 안전
	- `/.autorelabel` 은 레스큐 모드 비밀번호 재설정 후 필수 → [[getenforce]] 참조
	- 어원: **touch** (접촉 = 시각 갱신)

---

## ln

```bash
ln -sf <원본> <링크명>

# Examples
ln -sf /opt/app/current/bin/app /usr/local/bin/app    # 심볼릭 링크 갱신
```

### 명령어 설명
- 사용 목적
	- 버전 전환용 심볼릭 링크 구성 시 사용
	- 경로 단축 참조 생성 시 사용
- 특이사항
	- **`-s` 미지정 시 하드 링크 생성** → 디렉터리·파일시스템 간 불가
	- `-f` 는 기존 링크를 덮어씀 → 갱신 시 필수
	- **원본을 상대경로로 지정하면 링크 위치 기준 해석** → 절대경로 권장
	- 원본 삭제 시 끊어진 링크(dangling) 잔존 → [[ls]] `-l` 로 확인
	- 어원: **l**i**n**k

### 옵션
- `-s` : 심볼릭 링크 생성 (**s**ymbolic)
- `-f` : 기존 링크 덮어씀 (**f**orce)

---

## chmod

```bash
chmod +x <파일>

# Examples
chmod +x script/start-stat-scheduler.sh        # 실행 권한 부여
chmod +x "$SP/gen-tokens.sh"                    # 스크립트 실행 가능화
chmod +x script/oms/start-stat-collector.sh script/oms/start-stat-realtime-collector.sh
```

### 명령어 설명
- 사용 목적
	- **셸 스크립트 실행 권한 부여 시 사용** (`Permission denied` 해소)
	- 파일 접근 권한 조정 시 사용
- 특이사항
	- **`+x` 는 소유자·그룹·기타 전체에 실행 권한 부여** → 최소 권한 원칙상 `u+x` 권장
	- 숫자 표기 사용 가능 → `755`(소유자 전체 + 타인 읽기·실행), `644`(소유자 쓰기 + 타인 읽기)
	- **`777` 지정은 보안상 부적절** ⚠ → 모든 사용자 쓰기 허용
	- 디렉터리의 `x` 는 진입 권한 의미 → 파일의 실행 권한과 의미 상이
	- 소유자 아닌 파일은 root 필요 → [[sudo]] 병용
	- 어원: **ch**ange **mod**e

### 옵션
- `+x` : 실행 권한 부여 (e**x**ecute)
- `-x` : 실행 권한 제거
- `u+x` : 소유자만 실행 권한 (**u**ser) ※ 미검증
- `-R` : 디렉터리 재귀 적용 (**R**ecursive) ※ 미검증

---

## 연관 명령어
- [[ls]] : 조작 전후 대상·권한 확인 — **`rm` 전 확인 필수**
- [[find]] : 조작 대상 탐색
- [[stat]] : 권한·소유자 상세 확인
- [[du]] : 삭제 전 용량 확인
- [[basename]] : 경로에서 파일명 추출 후 조작
