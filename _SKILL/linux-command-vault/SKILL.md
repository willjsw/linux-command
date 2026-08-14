---
name: linux-command-vault
description: Linux-Command Obsidian 저장소에 리눅스 명령어 문서를 작성·갱신하는 규칙. 사용자가 리눅스/유닉스 명령어를 실행하거나 지시한 세션에서 그 명령어를 정리·기록·추가·업데이트할 때 사용. "명령어 정리", "vault 업데이트", "Linux-Command 추가" 등의 요청 시 적용.
type: SKILL
tags:
  - moc/skill
  - meta/convention
updated: 2026-07-30
---

# SKILL — Linux Command Vault 작성 규칙

- 대상 저장소: `/Users/sunwoo/Documents/Obsidian/Linux-Command`
- 형식: Obsidian vault (백링크·그래프뷰·태그페인 활성)
- 목적: Claude 세션에서 **실사용·검증한** 리눅스 명령어의 체계적 축적
- 색인: [[INDEX]]

---

## 1. 적용 시점

- 세션에서 리눅스 명령어를 **실행했거나 실행을 지시한 경우**
- 사용자가 명령어 정리·문서화를 요청한 경우
- 기존 문서의 내용이 실제 동작과 달랐음을 확인한 경우 (수정 필요)
- 문서화된 명령어에 **범용적으로 중요·빈출하는 미기재 옵션**이 확인된 경우 → 아래 `1-2` 자체 보강 규칙 적용

### 1-1. 미적용 대상
- **미사용·미검증 명령어의 신규 문서 생성** → 추측 기반 문서 양산 금지
- 일회성 조합 명령 (파이프 조합 등) → 개별 명령어 문서의 예시로 편입

### 1-2. 자체 보강 규칙 (미사용 옵션의 선제적 기재)

세션에서 사용하지 않았더라도 아래 **전 조건 충족 시 문서에 자체 추가**.

#### 편입 기준 (AND 조건)
- 해당 명령어 문서가 **이미 존재** (신규 문서 생성 근거로는 불가)
- 실무에서 **표준적으로 빈출**하는 옵션·서브명령
- 배포판·버전 무관하게 **동작이 안정적**인 항목
- man page·공식 문서로 **근거 확인 가능**

#### 배제 기준 (OR 조건— 하나라도 해당 시 미기재)
- 잘못 섞여 들어간 mac 전용, window 전용 명령어
- 특정 버전·배포판 한정 동작
	- 해당 경우 특이사항과 태그에 버전을 명시
- 파괴적 작업의 미검증 옵션 → `danger/*` 대상 명령어는 실검증 후 기재
- 사용 빈도 낮은 특수 목적 옵션
- 동작 결과를 근거로 단정할 수 없는 항목

#### 표기 의무
- 미검증 항목은 **`※ 미검증` 표시 필수** → 실사용 검증 항목과 시각적 구분
- 검증 후 표시 제거 + `updated` 갱신

```markdown
### 옵션
- `-h` : 읽기 쉬운 단위 (**h**uman-readable)          ← 실사용 검증
- `-i` : inode 정보 (**i**node) ※ 미검증               ← 자체 보강
```

- frontmatter `verified` 는 **실제 검증 환경 기준 유지** → 자체 보강으로 변경 금지
- 전 항목이 미검증인 참조용 문서는 `verified: 미검증 (참조용)` 표기

#### 판단 예시
| 항목 | 판정 | 근거 |
| --- | --- | --- |
| `df -i` (inode 조회) | ✅ 기재 | 표준 빈출, 비파괴, 근거 명확 |
| `journalctl -p err` (우선순위 필터) | ✅ 기재 | 진단 표준 옵션 |
| `grep -E` (확장 정규식) | ✅ 기재 | 범용 표준 |
| `dd conv=noerror,sync` | ❌ 배제 | 파괴적 명령의 미검증 옵션 |
| `parted resizepart` | ❌ 배제 | 파괴적, 실검증 필요 |
| `systemctl --user` | ❌ 배제 | 서버 운영 맥락에서 빈도 낮음 |

---

## 2. 작성 원칙 (필수 준수)

### 2-1. 문체 — 개조식·명사형 종결
- **모든 서술은 명사형으로 종결**
- 금지: `~한 때에 사용한다`, `~합니다`, `~해야 합니다`
- 준수: `~ 시 사용`, `~ 필요`, `~ 불가`, `~ 권장`, `~ 발생`

| 구분  | 예시                       |
| --- | ------------------------ |
| ❌   | dnf 명령어를 패키지 설치할 때에 사용한다 |
| ❌   | 외부 인터넷이 없으면 사용할 수 없습니다   |
| ✅   | 저장소에서 패키지 설치 시 사용        |
| ✅   | 외부 인터넷·사내 미러 접근 불가 시 실패  |

### 2-2. 간결성
- 1항목 1줄 원칙, 부연은 하위 불릿으로 분리
- 하위 불릿의 경우, 내용의 중요도에 따라 최대 5줄까지 작성 가능
- 중복 서술 배제 → 동일 내용은 위키링크로 연결
- 산문 단락 금지 → 불릿·표·코드블록만 사용

### 2-3. 검증 범위 명시
- 실제 확인한 환경만 `verified` 에 기재
- 버전·배포판 차이가 있으면 명시
- **미검증 항목은 `1-2` 자체 보강 규칙 충족 시에만 기재** + `※ 미검증` 표시 의무
- 추정·불확실 동작은 표시 여부와 무관하게 기재 금지

---

## 3. 파일 구조

### 3-1. 경로 규칙
```
Linux-Command/
├── README.md                   ← 저장소 소개 (구조화 시스템·현행화 방식)
├── INDEX.md                    ← 전체 색인 (MOC)
├── _SKILL/                     ← 스킬 정본 디렉터리
│   └── linux-command-vault/
│       └── SKILL.md            ← 본 문서 (작성 규칙 정본)
└── <CATEGORY>/<command>.md     ← 명령어 문서
```

- `_SKILL/`이 정본. `~/.claude/skills/linux-command-vault/SKILL.md` 는 이곳을 가리키는 심볼릭 링크

- 카테고리 디렉터리: **대문자 + 하이픈** (`PACKAGE-MANAGEMENT`)
- 파일명: **명령어 원문 소문자** (`dnf.md`, `nmcli.md`)
- 하위 명령이 많은 경우 단일 파일에 `## <명령> <서브명령>` 으로 분리 (예: `dnf.md` 내 `## dnf install`)
- 명령어가 아닌 절차는 케밥케이스 (`rescue-mode.md`)

### 3-2. 현재 카테고리
| 디렉터리 | 범위 |
| --- | --- |
| `PACKAGE-MANAGEMENT` | 패키지 설치·갱신·검증 |
| `NETWORK-MANAGEMENT` | IP·라우팅·원격접속·포트 |
| `DISK-STORAGE` | 디스크·파티션·LVM·마운트 |
| `USER-PERMISSION` | 계정·그룹·권한 |
| `SERVICE-SYSTEMD` | 서비스·타겟·로그·재시작 |
| `BOOT-RECOVERY` | 부트로더·레스큐·복구 |
| `SECURITY-SELINUX` | 방화벽·SELinux·감사로그 |
| `SYSTEM-INFO` | 로케일·시간대·시스템 정보·명령 가용성·환경변수 |
| `FILE-TEXT` | 파일·텍스트 처리 |
| `PROCESS-MANAGEMENT` | 프로세스 조회·종료·백그라운드 실행·포트 점유 |

### 3-3. 신규 카테고리 추가 기준
- 기존 카테고리에 3개 이상 문서가 누적되며 성격이 분리되는 경우
- 추가 시 [[INDEX]] 의 카테고리 목록·표 동시 갱신

---

## 4. 메타데이터 헤더 (필수)

```yaml
---
command: <명령어 원문>
category: <카테고리 디렉터리명>
aliases: [<동의어·관련 명령>]
tags:
  - <아래 태그 체계 참조>
related: ["\[\[관련명령\]\]"]
distro: <적용 배포판>
verified: <검증 환경>
updated: <YYYY-MM-DD>
---
```

### 필드 설명
- `command` : 명령어 원문 (파일명과 일치)
- `category` : 상위 디렉터리명
- `aliases` : 별칭·동일 계열 명령 (Obsidian 검색 연결)
- `tags` : **필수** — 아래 체계 준수
- `related` : 연관 문서 위키링크 배열
- `distro` : 적용 배포판·패키지 (예: `RHEL 계열 (Rocky, CentOS)`)
- `verified` : 실제 검증 환경 (예: `Rocky Linux 9.6`)
- `updated` : 최종 갱신일

---

## 5. 태그 체계 (필수)

**태그는 문서 간 횡단 연결의 핵심** → 최소 5개 이상 부여.

### 5-1. 계열 태그 (`linux/*`) — 1개 이상
- `linux/package` `linux/network` `linux/disk` `linux/user`
- `linux/systemd` `linux/service` `linux/boot` `linux/security`
- `linux/log` `linux/text` `linux/process` `linux/system`

### 5-2. 작업 태그 (`task/*`) — 1개 이상
- `task/install` `task/update` `task/configure` `task/inspect`
- `task/diagnose` `task/verify` `task/recovery` `task/expand`
- `task/connect` `task/search` `task/mount` `task/partition`
- `task/wipe` `task/restart` `task/privilege-escalation`

### 5-3. 주제 태그 (`topic/*`) — 1개 이상
- `topic/static-ip` `topic/dhcp` `topic/routing` `topic/port` `topic/socket`
- `topic/lvm` `topic/partition` `topic/mbr` `topic/filesystem` `topic/capacity`
- `topic/bootloader` `topic/grub` `topic/bls` `topic/rescue` `topic/kernel-parameter`
- `topic/selinux` `topic/firewall` `topic/security` `topic/sudo` `topic/group` `topic/password`
- `topic/remote-access` `topic/hostname` `topic/locale` `topic/encoding`
- `topic/desktop-environment` `topic/boot-target` `topic/troubleshooting` `topic/regex`

### 5-4. 권한 태그 (`privilege/*`) — 1개 필수
- `privilege/root` : root 전용
- `privilege/user` : 일반 사용자 가능
- `privilege/mixed` : 조회는 일반, 변경은 root

### 5-5. 위험 태그 (`danger/*`) — 해당 시 필수
- `danger/destructive` : 데이터 손실 가능
- `danger/irreversible` : 복구 불가
- `danger/data-loss` : 옵션 오용 시 손실

### 5-6. 배포판 태그 (`distro/*`) — 특정 계열 전용 시
- `distro/rhel` `distro/rocky` `distro/debian` `distro/ubuntu`

### 5-7. 기타
- `requires/network` : 네트워크 필수
- `replaces/<old>` : 구형 명령 대체 (`replaces/ifconfig`)
- `interface/tui` : TUI 제공

---

## 6. 본문 구조

````markdown
# <명령어>

- 한 줄 정의
- 어원: <축약어 원형>
- 특징·전제 (권한, 패키지 등)

---

## <명령어> <서브명령>

```bash
<기본 문법>

# Examples
<실사용 예시>          # 주석으로 목적 명시
```

### 명령어 설명
- 사용 목적
	- <용도 1> 시 사용
	- <용도 2> 시 사용
- 특이사항
	- <함정·주의사항>

### 옵션
- `-x` : 설명 (**x**의 원형 단어)

---

## 연관 명령어
- 위키링크 : 연관 맥락 명시
- 위키링크 : 연관 맥락 명시
````

> 템플릿 내 `위키링크` 표기는 실제 작성 시 `\[\[명령어명\]\]` 형식으로 대체.
> 본 문서에는 실제 링크를 두지 않음 → 그래프뷰에 존재하지 않는 노드 생성 방지 목적.

### 6-1. 필수 섹션
- `# 제목` 하단 한 줄 정의 + 어원
- 서브명령별 `## ` 섹션
- 각 섹션에 `### 명령어 설명`(사용 목적 / 특이사항)
- 옵션 존재 시 `### 옵션`
- 문서 말미 `## 연관 명령어`

### 6-2. 옵션 표기 규칙
- 축약 옵션은 **원형 단어를 굵게 표시**
- 예: `-h` : 읽기 쉬운 단위 (**h**uman-readable)
- 예: `-aG` : 기존 그룹 유지하며 추가 (**a**ppend / **G**roups)

### 6-3. 예시 코드 규칙
- 실제 세션에서 사용한 값 우선 기재 (가상값보다 실증값)
- 사내 환경값 기재 시 용도 주석 필수
- 위험 명령은 선행 확인 절차를 함께 배치

---

## 7. 연결 규칙 (Obsidian 그래프 형성)

### 7-1. 링크 3중 배치 — 필수
1. **frontmatter `related`** : 배열 형태
2. **본문 내 맥락 링크** : 서술 중 자연스럽게 삽입
3. **말미 `## 연관 명령어`** : 연관 이유 명시

### 7-2. 링크 방향 원칙
- **진단 흐름** : 상위 진단 → 하위 확인 (`ssh` → `ss` → `firewall-cmd`)
- **선행 조건** : 필수 선행 명령 링크 (`dd` → `lsblk`)
- **대체 수단** : 동일 목적 다른 도구 (`nmcli` ↔ `nmtui`)
- **후속 작업** : 이어지는 필수 작업 (`lvextend` → `xfs_growfs`)

### 7-3. 상호 링크 유지
- A 문서가 B를 링크하면 **B 문서도 A를 링크** (백링크 대칭)
- 미작성 문서 링크 허용 → 향후 작성 대상 표시 기능

---

## 8. 갱신 절차

### 8-1. 신규 문서 추가
1. 카테고리 판별 (없으면 신규 생성 + [[INDEX]] 표 갱신)
2. `<CATEGORY>/<command>.md` 생성
3. frontmatter 작성 (태그 5개 이상)
4. 본문 작성 (개조식·명사형)
5. `related` 대상 문서에 **역방향 링크 추가**
6. [[INDEX]] 카테고리 목록에 1줄 추가
7. 해당되는 작업 시나리오 경로에 편입

### 8-2. 기존 문서 갱신
1. 신규 서브명령·옵션 → 해당 섹션 추가
2. 실사용 예시 → `# Examples` 블록에 추가
3. 발견한 함정 → `특이사항` 에 추가
4. `updated` 날짜 갱신
5. 새 연관 관계 발생 시 양방향 링크 추가

### 8-3. 오류 정정
- 실제 동작과 다른 기술 발견 시 **즉시 수정**
- 잘못된 기대 출력·버전 차이는 `특이사항` 에 반영
- 예: `grub2-mkconfig` 의 `Found linux image` 미출력이 Rocky 9 정상 동작

---

## 9. 품질 체크리스트

문서 작성·갱신 후 확인.

- [ ] frontmatter 전 필드 작성 (`command` `category` `tags` `related` `distro` `verified` `updated`)
- [ ] 태그 5개 이상, `privilege/*` 1개 포함
- [ ] 위험 명령에 `danger/*` 태그 부여
- [ ] 전 서술이 명사형 종결
- [ ] 산문 단락 없음 (불릿·표·코드블록만)
- [ ] 어원 기재
- [ ] 옵션에 원형 단어 굵게 표시
- [ ] 링크 3중 배치 (frontmatter / 본문 / 말미)
- [ ] 역방향 링크 추가 완료
- [ ] [[INDEX]] 갱신 완료
- [ ] 미검증 항목에 `※ 미검증` 표시 (`1-2` 자체 보강 규칙)
- [ ] `danger/*` 명령어의 미검증 옵션 미포함
- [ ] 추정·불확실 동작 미포함

---

## 10. Git 반영 절차 (필수)

문서 추가·수정 후 **Git 커밋 및 `main` 푸시 필수**. 변경 이력이 곧 지식 축적 이력.

### 10-1. 커밋 단위
- **문서 1개 = 커밋 1개** 원칙 → 변경 추적·되돌리기 용이
- 신규 문서 다수 생성 시에도 개별 커밋 분리
- 예외 — 아래는 단일 커밋으로 묶음 허용
	- 역방향 링크 추가만 발생한 다수 문서 (`docs:` 로 일괄)
	- `INDEX.md` 갱신 (신규 문서 커밋에 포함 또는 별도 1커밋)

### 10-2. 커밋 메시지 컨벤션

```
<작업종류>: <명령어 이름 or 디렉터리 이름 or 문서 이름> - <설명 간단히>

- <상세 설명 — 필요한 경우에만>
```

**작업종류**

| 종류         | 용도                                   |
| ---------- | ------------------------------------ |
| `feat`     | 신규 문서 생성                             |
| `docs`     | 기존 문서 내용 추가·보강 (옵션·예시·특이사항)          |
| `fix`      | 오류 정정 (실제 동작과 다른 기술 수정)              |
| `refactor` | 구조 변경 (문서 분리·통합, 카테고리 이동)            |
| `chore`    | 저장소 설정 (`.gitignore`, README, 색인 정리) |

**예시**

```
feat: ss 문서 생성
```

```
refactor: SKILL.md 수정

- 2. 작성 규칙 업데이트 — 명사형 종결 예시 보강
- 5. 태그 체계에 linux/process 추가
```

```
fix: grub2-mkconfig - Rocky 9 출력 차이 정정

- Found linux image 미출력이 정상 동작임을 특이사항에 반영
```

```
docs: df - inode 조회 옵션 추가

- -i 옵션 자체 보강 (※ 미검증 표시)
```

### 10-3. 작업 절차

```bash
# 1. 변경 확인
git status
git diff

# 2. 문서 단위 스테이징 → 커밋 (문서마다 반복)
git add <CATEGORY>/<command>.md
git commit -m "feat: <command> 문서 생성"

# 3. 색인 갱신 커밋
git add INDEX.md
git commit -m "docs: INDEX - <command> 색인 등재"

# 4. main 푸시
git push origin main
```

### 10-4. 특이사항
- **푸시 전 [[SKILL]] 9장 품질 체크리스트 통과 필수** → 미완성 문서 푸시 금지
- 역방향 링크 누락 상태로 커밋 금지 → 그래프 비대칭 발생
- 대규모 구조 변경(카테고리 신설·문서 통합)은 **브랜치 분리 후 PR** 권장
	- 브랜치명: `chore/<작업명>` `refactor/<작업명>` 형식
- 자격증명·내부 IP·계정명은 **커밋 전 플레이스홀더 치환 확인 필수** ⚠
	- 사내 환경값 기재 시 용도 주석만 남기고 실제 값 제거
- `.obsidian/` 개인 설정은 `.gitignore` 처리 → 커밋 대상 아님

---

## 11. 확장 제안 (미적용, 검토 대상)

향후 문서 누적 시 도입 검토 항목.

### 11-1. 시나리오 문서 분리
- `SCENARIO/` 디렉터리 신설
- 예: `SCENARIO/os-install-postsetup.md`, `SCENARIO/boot-failure-recovery.md`
- 명령어 문서는 참조 사전, 시나리오 문서는 절차 흐름으로 역할 분리
- 도입 기준: 명령어 문서 30개 이상 누적 시

### 11-2. Obsidian Bases 활용
- `bases` 코어 플러그인 활성 상태 확인됨
- frontmatter 기반 동적 테이블 구성 가능
- 예: `privilege/root` 필터 목록, `danger/*` 전용 목록, `updated` 기준 정렬

### 11-3. 실패 사례 로그
- `TROUBLESHOOTING/` 디렉터리에 증상 기준 문서 축적
- 증상 문구 → 원인 → 해당 명령어 문서 링크 구조
- 예: `SQUASHFS error`, `Login incorrect`, `NO-CARRIER`, `Duplicate UUID`
- 명령어 기준 검색으로는 도달 불가한 진입점 확보 목적

### 11-4. 배포판 분기 표기
- Debian/Ubuntu 계열 병행 사용 시 `## <배포판> 대응` 섹션 추가
- 예: `dnf` ↔ `apt`, `firewall-cmd` ↔ `ufw`

---

## 관련 문서
- [[INDEX]] — 전체 색인 및 시나리오 경로
- `README.md` — 저장소 소개·구조화 시스템 설명 (외부 공개용, 위키링크 미사용)
