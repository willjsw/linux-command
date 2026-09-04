---
title: LAB 02 — 패키지 관리와 소스 컴파일
type: exam-lab
part: 02
tags:
  - exam/linux-master
  - exam/lab
  - linux/package
  - linux/software
  - task/configure
  - task/install
  - task/verify
related: ["[[README]]", "[[01-vm-setup-and-inspection]]", "[[03-user-group-permission]]", "[[../THEORY/package-software]]", "[[../../PACKAGE-MANAGEMENT/dnf]]", "[[../../PACKAGE-MANAGEMENT/rpm]]", "[[../../PACKAGE-MANAGEMENT/dnf-group]]", "[[../../SYSTEM-INFO/ldd]]"]
distro: Rocky Linux 9 (aarch64, UTM)
updated: 2026-09-03
---

# LAB 02 — 패키지 관리와 소스 컴파일

- 저장소(`/etc/yum.repos.d`, `dnf.conf`) 점검 → `dnf` 조회·설치·이력 → `rpm` 저수준 질의·검증 → 그룹·모듈 → EPEL → 이후 파트용 유틸 일괄 설치
- GNU hello 소스 컴파일 3단계(`configure` → `make` → `make install`) 와 공유 라이브러리(`ldd`, `ldconfig`, `ld.so.conf`) 실습
- Debian 계열(`dpkg`/`apt`) 은 비교표만 — 실습은 [[11-container-virtualization]] 의 Ubuntu 컨테이너에서
- 마지막에 `/etc/hosts` 를 고의 변경해 `rpm -V` 변경 코드를 읽고 원본 복원

> **이 파트의 시나리오**: Part 01 에서 `srv01.lab.local` VM 을 minimal 로 설치해 시스템 정보와 FHS 를 점검했다. 이제 Part 03~12 에서 쓸 도구(net-tools, bind-utils, sysstat, mdadm, quota, stress-ng …)를 한꺼번에 설치해야 하는데, 그 전에 저장소가 정상인지 확인하고 `dnf`/`rpm` 사용법을 손에 익힌다. 개발팀이 요구한 "소스 빌드 환경" 준비를 위해 Development Tools 그룹도 설치하고 GNU hello 로 빌드 절차를 검증한다.

- 전 과정 root(`#`) 로 진행. 일반 사용자 `admin1` 은 `sudo dnf …` 형태로 동일 수행 가능
- 인터넷 연결 필수(NAT). 실패 시 [[01-vm-setup-and-inspection]] 의 네트워크 확인 단계로 복귀

---

## 1. 저장소 점검

### 1-1. 활성 저장소 목록

> **상황**: 패키지를 설치하려면 dnf 가 어느 저장소를 바라보는지 알아야 한다. 설치 직후 기본 활성 저장소와 비활성 저장소를 구분해 확인한다.

```bash
dnf repolist            # 활성(enabled) 저장소만
dnf repolist --all      # 비활성 포함 전체
dnf repolist -v         # 상세 (URL, 패키지 수, 메타데이터 갱신 시각)
```

- `repolist` : 등록된 저장소 목록 출력 (**repo**sitory **list**)
- `--all` : `enabled=0` 저장소까지 표시 (**all**)
- `-v` : 저장소별 baseurl·mirrorlist·패키지 수 상세 (**v**erbose)

**검증**

```bash
dnf repolist --all | grep -iE 'crb|epel|extras'
```

```text
repo id            repo name                                  status
appstream          Rocky Linux 9 - AppStream                  enabled
baseos             Rocky Linux 9 - BaseOS                     enabled
crb                Rocky Linux 9 - CRB                        disabled
extras             Rocky Linux 9 - Extras                     enabled
...
```

> 📝 **시험 포인트**: `dnf repolist` = 저장소 목록, `dnf provides` = 파일 제공 패키지 — 두 서브명령의 역할을 뒤바꾼 오답 보기가 반복 출제(R04-43).

### 1-2. `.repo` 파일 구조

> **상황**: 저장소 정의가 실제로 어디에 어떤 형식으로 있는지 확인한다. 나중에 사내 미러나 EPEL 을 추가할 때 같은 형식을 쓴다.

```bash
ls -l /etc/yum.repos.d/
grep -vE '^\s*(#|$)' /etc/yum.repos.d/rocky.repo | head -30   # 주석·빈 줄 제외
```

- `/etc/yum.repos.d/*.repo` : 저장소 정의 파일 디렉터리 (yum 시절 이름 유지)
- `[id]` : 저장소 ID — `dnf repolist` 의 repo id
- `name=` : 표시 이름
- `mirrorlist=` : 미러 서버 목록을 돌려주는 URL (Rocky 기본). `$releasever`, `$basearch` 변수 치환
- `baseurl=` : 저장소 직접 URL (미러 대신 특정 서버 고정 시 사용, Rocky 기본은 주석 처리)
- `enabled=` : `1` 활성 / `0` 비활성
- `gpgcheck=` : `1` 이면 패키지 GPG 서명 검증
- `gpgkey=` : 검증용 공개키 경로 (`file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-9`)
- `countme=` : Rocky 미러 통계용 (시험 범위 외)

**검증**

```bash
grep -E '^\[|enabled|gpgcheck|gpgkey' /etc/yum.repos.d/rocky.repo
ls -l /etc/pki/rpm-gpg/
```

```text
[baseos]
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-9
[appstream]
enabled=1
...
[crb]
enabled=0
...
-rw-r--r--. 1 root root ... RPM-GPG-KEY-Rocky-9
```

> 📝 **시험 포인트**: 저장소 설정 위치 `/etc/yum.repos.d/` ↔ Debian `/etc/apt/sources.list`. 키 이름 `baseurl`·`enabled`·`gpgcheck` 빈칸 채우기 출제.

### 1-3. CRB 저장소 활성화

> **상황**: EPEL 의 일부 패키지와 개발용 `-devel` 패키지가 CRB(CodeReady Builder) 에 있다. 파일을 직접 편집하는 대신 `dnf config-manager` 로 `enabled=1` 을 켠다.

```bash
dnf install -y dnf-plugins-core            # config-manager, download 등 플러그인 (대개 기설치)
dnf config-manager --set-enabled crb
```

- `install -y` : 확인 프롬프트 자동 승인 (**y**es)
- `dnf-plugins-core` : `config-manager`, `download`, `repoquery` 확장 등을 담은 플러그인 묶음
- `config-manager` : `.repo` 파일·`dnf.conf` 를 명령으로 수정하는 플러그인
- `--set-enabled <id>` : 해당 저장소 `enabled=1` 로 변경 (`--set-disabled` 는 반대)
- `--add-repo <URL>` : `.repo` 파일 신규 생성 (참고)

**검증**

```bash
dnf repolist | grep crb
grep -A6 '^\[crb\]' /etc/yum.repos.d/rocky.repo | grep enabled
```

```text
crb                Rocky Linux 9 - CRB
enabled=1
```

> 📝 **시험 포인트**: 파일 편집 없이 저장소 on/off 하는 명령이 `dnf config-manager --set-enabled/--set-disabled`. RHEL 은 CRB 대신 `codeready-builder-for-rhel-9-*` 라는 긴 ID 사용.

### 1-4. dnf 전역 설정 파일

> **상황**: 모든 저장소에 공통으로 적용되는 dnf 동작(GPG 검사, 커널 보관 수, 제외 패키지)을 정하는 파일을 읽는다.

```bash
cat /etc/dnf/dnf.conf
```

- `[main]` : 전역 섹션
- `gpgcheck=1` : 저장소별 설정이 없을 때 기본 서명 검사
- `installonly_limit=3` : 커널처럼 "업그레이드 대신 나란히 설치" 되는 패키지의 보관 개수
- `clean_requirements_on_remove=True` : 삭제 시 함께 끌려온 의존 패키지도 제거 (autoremove 동작 기반)
- `best=True` : 최신 후보 설치 불가 시 오류 (낮은 버전으로 조용히 대체 안 함)
- `skip_if_unavailable=False` : 저장소 접근 실패를 오류로 처리
- `exclude=` : 전역 제외 패키지 (예: `exclude=kernel*`) — 기본 파일에는 없음, 필요 시 추가

**검증**

```bash
grep -c '=' /etc/dnf/dnf.conf
dnf config-manager --dump | grep -E '^(gpgcheck|installonly_limit|exclude)'   # 실제 적용값
```

```text
5
exclude =
gpgcheck = 1
installonly_limit = 3
```

> 📝 **시험 포인트**: `gpgcheck=0` 은 검증 **비활성화**(R03-31 오답 보기). `installonly_limit` = 보관 커널 개수.

### 1-5. 메타데이터 갱신·캐시 정리·업데이트 유무

> **상황**: 저장소 메타데이터를 강제로 받아 연결을 확인하고, 설치 전에 대기 중인 업데이트가 있는지 본다.

```bash
dnf clean all           # /var/cache/dnf 의 메타데이터·패키지 캐시 삭제
dnf makecache           # 메타데이터 새로 내려받아 캐시 생성
dnf check-update        # 업데이트 가능 패키지 목록 (설치는 안 함)
echo $?                 # 100 = 업데이트 있음, 0 = 없음, 1 = 오류
```

- `clean all` : 캐시 전체 삭제 (`clean metadata`, `clean packages` 를 합친 것)
- `makecache` : 저장소 메타데이터 다운로드·캐시 생성
- `check-update` : 업데이트 가능 목록 조회 — **종료 코드 100** 이 "있음" (스크립트 분기용)

**검증**

```bash
ls /var/cache/dnf/
du -sh /var/cache/dnf
```

```text
appstream-...  baseos-...  crb-...  extras-...  expired_repos.json  ...
... /var/cache/dnf
```

> 📝 **시험 포인트**: RedHat `dnf update` 는 실제 업그레이드, Debian `apt update` 는 목록 갱신만 — dnf 에서 "목록 갱신" 에 대응하는 것은 `makecache`.

---

## 2. 패키지 조회

### 2-1. search · info · list

> **상황**: 설치 전에 패키지 이름·버전·저장소·설치 여부를 조회한다. `dnf info` 의 `Installed Packages` / `Available Packages` 구분을 확인한다.

```bash
dnf search net-tools                # 이름·요약에서 검색
dnf search all ifconfig             # 설명(description)까지 검색
dnf info net-tools                  # 상세 정보 (설치 여부 포함)
dnf list installed | head -5        # 설치된 패키지
dnf list available | wc -l          # 설치 가능 패키지 수
dnf list updates                    # 업데이트 대기 패키지 (= --upgrades)
dnf list kernel*                    # 와일드카드 (installonly 패키지 확인)
```

- `search <키워드>` : 이름·요약(summary) 검색. `search all` 은 설명까지
- `info <패키지>` : Name/Version/Release/Architecture/Repository/Summary 등
- `list installed` : 설치된 패키지만 / `list available` : 미설치 중 설치 가능 / `list updates` : 업데이트 후보
- `@System` : `Repository` 열에서 "이미 설치됨" 을 뜻하는 표기

**검증**

```bash
dnf info net-tools | grep -E '^(Name|Version|Repository|Available|Installed)'
rpm -q net-tools
```

```text
Available Packages
Name         : net-tools
Version      : 2.0
Repository   : baseos
package net-tools is not installed
```

> 📝 **시험 포인트**: `dnf list installed` = 설치된 목록(R01-43, R02-40, R04-43 의 오답 보기가 이걸 "설치 가능 전체" 또는 "즉시 설치" 로 왜곡). `dnf info` 출력의 `Available Packages` 헤더 = 미설치 상태(R08-37).

### 2-2. provides · repoquery · 의존성

> **상황**: `ifconfig` 를 치면 "command not found" 다. 어느 패키지가 그 파일을 제공하는지 찾고, 설치 전에 파일 목록과 의존성을 본다.

```bash
dnf provides '*/ifconfig'                  # 파일 경로로 제공 패키지 역추적
dnf provides /usr/bin/ss                   # 절대경로도 가능
dnf repoquery -l net-tools                 # 미설치 패키지의 파일 목록 (저장소 메타데이터 기준)
dnf repoquery --requires net-tools         # 요구 의존성
dnf repoquery --whatrequires net-tools     # 이 패키지를 필요로 하는 패키지 (역방향)
dnf deplist net-tools | head -20           # 의존성 + 제공 패키지 (구형 표기, 동일 정보)
```

- `provides <파일/기능>` : 지정 파일·capability 를 제공하는 패키지 검색. `*/이름` 으로 경로 미상 파일 검색
- `repoquery` : 저장소 메타데이터에 질의 (설치 여부 무관)
- `-l` : 파일 목록 (**l**ist) — `rpm -ql` 의 미설치 대응
- `--requires` : 요구 의존성 / `--whatrequires` : 역의존성
- `deplist` : 의존성과 그 제공자를 함께 출력 (dnf4 에서 `repoquery --deplist` 별칭, 비권장 표기지만 동작)

**검증**

```bash
dnf provides '*/ifconfig' | grep -E '^net-tools'
```

```text
net-tools-2.0-0.64.20160912git.el9.aarch64 : Basic networking tools
```

> 📝 **시험 포인트**: "`/usr/bin/ss` 를 제공하는 패키지 찾기" = `dnf provides`(R07-34). 미설치 패키지 파일 목록은 `rpm -ql` 불가 → `dnf repoquery -l`.

### 2-3. 트랜잭션 이력

> **상황**: dnf 는 모든 설치·삭제를 트랜잭션 ID 로 기록한다. 지금까지의 이력을 보고 되돌리기 문법을 익힌다 (실제 undo 는 3-1 에서 수행).

```bash
dnf history                    # = history list : ID, 명령, 일시, 동작, 변경 수
dnf history info 1             # 1번(설치 시점) 트랜잭션 상세
dnf history info last          # 가장 최근 트랜잭션
```

- `history` / `history list` : 트랜잭션 목록
- `history info <ID|last>` : 트랜잭션 내 패키지 단위 동작 (Install/Upgrade/Erase)
- `history undo <ID>` : 해당 트랜잭션 반대 수행 (설치→삭제, 삭제→재설치)
- `history redo <ID>` : 동일 트랜잭션 재수행
- `history rollback <ID>` : 그 ID 시점 상태까지 모두 되돌림 (참고)

**검증**

```bash
dnf history | head -5
```

```text
ID     | Command line              | Date and time    | Action(s)      | Altered
-----------------------------------------------------------------------------
     2 | install -y dnf-plugins-core | 2026-09-03 ... | Install        |    ...
     1 |                           | 2026-09-0. ...   | Install        |  ... EE
```

> 📝 **시험 포인트**: `dnf history undo <ID>` 는 설치·삭제 모두 되돌림, `history info` 는 존재하는 하위 명령(R03-32, R10-39 의 오답 보기 반박). 이력은 `/var/lib/dnf/history.sqlite` 에 저장돼 재부팅 후 유지.

---

## 3. dnf 설치·삭제·업그레이드

### 3-1. install · remove · autoremove · reinstall · history undo

> **상황**: Part 06 잡 제어 실습에서 쓸 `tmux` 를 설치하고, 삭제·자동정리·재설치·되돌리기를 한 번씩 수행해 dnf 트랜잭션 흐름을 몸에 익힌다.

```bash
dnf install -y tmux
tmux -V
dnf remove -y tmux                 # 삭제 (의존성으로 끌려온 패키지는 clean_requirements_on_remove 에 따라 함께 제거)
dnf autoremove                     # 어떤 패키지도 필요로 하지 않는 "leaf" 의존 패키지 정리
dnf history | head -4              # tmux remove 트랜잭션 ID 확인 → 아래 <ID> 에 대입
dnf history undo -y <ID>           # remove 를 되돌림 = tmux 재설치
dnf reinstall -y tmux              # 동일 버전 덮어쓰기 (파일 손상 복구용)
```

- `remove` : 삭제 (`erase` 별칭). 다른 패키지가 의존하면 그 패키지까지 삭제 목록에 오르므로 요약 확인 필요
- `autoremove` : 의존성으로 설치됐으나 지금은 아무도 요구하지 않는 패키지 일괄 제거
- `reinstall` : 같은 버전 재설치 — 삭제된 바이너리·손상 파일 복구
- `history undo <ID>` : 2-3 참조

**검증**

```bash
rpm -q tmux
dnf history | head -6           # install → remove → undo(Install) → reinstall 4 건 확인
```

```text
tmux-3.2a-...el9.aarch64
ID | Command line            | ...  | Action(s)  | Altered
 6 | reinstall -y tmux       | ...  | Reinstall  |  1
 5 | history undo -y 4       | ...  | Install    |  1
 4 | remove -y tmux          | ...  | Removed    |  1
 3 | install -y tmux         | ...  | Install    |  1
```

> 📝 **시험 포인트**: `dnf remove` = 제거(R01-43·R02-40 정답 보기). `rpm -e` 는 의존성 있으면 거부, `dnf remove` 는 의존 패키지까지 제안 — 반대 방향 차이.

### 3-2. update vs upgrade · --exclude · downgrade

> **상황**: 서비스 구축 전에 시스템을 최신 상태로 맞춘다. 단, 커널은 이 파트에서 재부팅을 피하기 위해 제외한다.

```bash
dnf check-update | head -20
dnf upgrade -y --exclude='kernel*'         # 커널 제외 전체 업그레이드
dnf update  -y --exclude='kernel*'         # 동일 동작 (update 는 upgrade 의 구형 별칭)
dnf upgrade -y tmux                        # 개별 패키지만
# dnf downgrade <패키지>                   # ※ 참고 — 저장소에 이전 버전이 남아 있을 때만 가능
```

- `upgrade` : 설치된 패키지를 저장소 최신으로 갱신. 인자 없으면 전체
- `update` : `upgrade` 와 동일 (dnf 에서 deprecated 별칭, yum 호환)
- `--exclude=<글롭>` : 이번 트랜잭션에서 제외. 영구 제외는 `/etc/dnf/dnf.conf` 의 `exclude=`
- `downgrade <패키지>` : 저장소에 존재하는 바로 이전 버전으로 내림 — 보안 패치 회귀 테스트용 (참고)

**검증**

```bash
dnf check-update ; echo "exit=$?"
dnf list installed kernel*
uname -r                                   # 현재 실행 커널 = 재부팅 전이므로 변화 없음
```

```text
exit=0            # 커널 제외 후 남은 업데이트 없음 (커널 업데이트가 있으면 100 과 kernel 줄 표시)
kernel.aarch64          5.14.0-...el9       @anaconda
kernel-core.aarch64     5.14.0-...el9       @anaconda
...
5.14.0-...el9.aarch64
```

> 📝 **시험 포인트**: RHEL 계열 `update`=`upgrade`(실제 갱신). Debian 은 `update`(목록) ≠ `upgrade`(갱신) — 계열 간 의미 차이 함정. 커널 업그레이드 후 반영은 재부팅 필요.

---

## 4. rpm 저수준 관리

### 4-1. rpm 파일 받기 → 파일 대상 질의(-qp) → 서명 검증(-K)

> **상황**: 저장소 없는 폐쇄망 서버에 넘길 rpm 파일을 미리 받아, 설치 전에 내용과 서명을 점검하는 절차다. 의존성이 glibc 뿐인 `tree` 로 연습한다.

```bash
mkdir -p /root/rpms && cd /root/rpms
dnf download tree                          # 현재 디렉터리에 .rpm 저장 (dnf-plugins-core)
dnf download --resolve tree                # 의존 패키지까지 함께 받기 (참고, 이미 설치된 의존성은 생략)
ls -l
rpm -qpi tree-*.rpm                        # 파일 대상 정보
rpm -qpl tree-*.rpm                        # 파일 대상 목록
rpm -qpR tree-*.rpm                        # 파일 대상 의존성
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-9   # 공개키 등록 (설치 시 이미 등록됨 — 재실행 무해)
rpm -K tree-*.rpm                          # 서명·다이제스트 검증 (= --checksig)
rpm -Kv tree-*.rpm                         # 상세
```

- `download <패키지>` : 저장소에서 rpm 파일만 내려받음 (설치 안 함). `--resolve` 로 의존성 포함, `--destdir=<경로>` 로 저장 위치 지정
- `-q` : 질의 모드 (**q**uery)
- `-p` : 설치 DB 대신 **p**ackage 파일에 질의 — `-qpi`, `-qpl`, `-qpR` 처럼 다른 질의 옵션과 결합
- `-i` : 상세 정보 (**i**nfo) / `-l` : 파일 목록 (**l**ist) / `-R` : 요구 의존성 (**R**equires)
- `--import <키파일>` : GPG 공개키를 rpm DB 에 등록 — `-K` 검증의 선행 조건
- `-K` / `--checksig` : 패키지 파일의 GPG 서명·다이제스트 검증 (설치된 파일 변경 검사인 `-V` 와 다름)
- `-v` : 검증 항목별 결과 (**v**erbose)

**검증**

```bash
rpm -qa 'gpg-pubkey*'                      # 등록된 공개키 목록 (gpg-pubkey 가상 패키지)
rpm -qi gpg-pubkey-$(rpm -qa 'gpg-pubkey*' | head -1 | cut -d- -f3-) | grep -E '^(Summary|Packager)'
rpm -K tree-*.rpm
```

```text
gpg-pubkey-350d275d-...
Summary     : Rocky Enterprise Software Foundation - Release key 2022 <releng@rockylinux.org> public key
tree-1.8.0-10.el9.aarch64.rpm: digests signatures OK
```

> 📝 **시험 포인트**: 서명 검증 절차 = `rpm --import <키>` → `rpm -K`(R03-31). `-qp` 는 "미설치 파일에 질의" — `-qpl`·`-qpi` 형태로 결합 출제. 파일명 규칙 `이름-버전-릴리스.아키.rpm` 분해 문제(R08-36 의 `Release: 8.el9` = RHEL9 용 8번째 빌드).

### 4-2. rpm -ivh / -Uvh / -Fvh / -e · 의존성 거부와 --nodeps

> **상황**: 받아둔 rpm 파일을 저수준 명령으로 설치·갱신·삭제한다. 의존성 자동 해결이 없다는 점을 `--test` 로 안전하게 확인한다.

```bash
rpm -ivh tree-*.rpm                # 설치 (진행바)
rpm -Uvh tree-*.rpm                # 업그레이드 — 같은 버전이면 "already installed" 로 거부 (정상)
rpm -Fvh tree-*.rpm                # 갱신 — 설치된 것만 대상 (이미 최신이면 조용히 종료)
rpm -e tree                        # 삭제 — 패키지 이름만 (버전·아키·.rpm 붙이지 않음)
rpm -e --test glibc-common         # ※ 삭제 예행연습 — 의존성 거부 메시지만 확인, 실제 삭제 안 됨
```

- `-i` : 설치 (**i**nstall) — 이미 설치돼 있으면 오류
- `-U` : 업그레이드 (**U**pgrade) — 없으면 신규 설치, 있으면 갱신 (가장 안전한 기본값)
- `-F` : 갱신 (**F**reshen) — 설치된 패키지만 갱신, 미설치면 건너뜀
- `-v` : 상세 (**v**erbose) / `-h` : 진행바 `#` (**h**ash)
- `-e` : 삭제 (**e**rase)
- `--test` : 실제 변경 없이 의존성·충돌 검사만
- `--nodeps` : 의존성 검사 생략 / `--force` : 파일 충돌·재설치 강제 (`--replacepkgs --replacefiles --oldpackage` 합)

⚠️ `rpm -e --nodeps` · `rpm -ivh --force --nodeps` 는 의존성 DB 를 깨뜨려 이후 `dnf` 트랜잭션 실패의 원인이 됨. 이 실습에서는 **실행하지 않음**. 의존성 오류의 정답은 저장소 기반 `dnf install` (R06-40).

**검증**

```bash
rpm -q tree                                 # -e 뒤 → not installed
rpm -ivh tree-*.rpm && rpm -q tree          # 다시 설치해 두기 (7절 일괄 설치 목록에도 포함)
which tree
```

```text
package tree is not installed
Verifying...                          ################################# [100%]
Preparing...                          ################################# [100%]
Updating / installing...
   1:tree-1.8.0-10.el9                ################################# [100%]
tree-1.8.0-10.el9.aarch64
/usr/bin/tree
```

```text
# rpm -e --test glibc-common 의 기대 출력
error: Failed dependencies:
        glibc-common = 2.34-...el9 is needed by (installed) glibc-2.34-...el9.aarch64
        ...
```

> 📝 **시험 포인트**: `-ivh`(신규) · `-Uvh`(있으면 갱신, 없으면 설치) · `-Fvh`(있는 것만 갱신) 구분(R04-41). `rpm -e` 인자는 **패키지 이름** — `.rpm` 파일명을 넣은 보기는 오답(R01-44·R02-41·R07-35 의 `dpkg -r pkg.deb` 도 같은 함정).

### 4-3. 설치된 패키지 질의 (-qa, -qi, -ql, -qc, -qd, -qf, -qR, --changelog)

> **상황**: 운영 중 "이 파일은 어느 패키지 것인가", "이 패키지의 설정 파일은 어디인가" 를 즉시 답해야 한다. 질의 옵션을 한 바퀴 돈다.

```bash
rpm -qa | wc -l                     # 설치 패키지 수
rpm -qa | grep -i ssh               # 이름 필터
rpm -qa --last | head -5            # 설치 시각 역순 (최근 설치 추적)
rpm -q openssh-server               # 설치 여부·버전
rpm -qi openssh-server              # 상세 정보
rpm -ql openssh-server              # 포함 파일 전체
rpm -qc openssh-server              # 설정 파일만 (%config)
rpm -qd openssh-server | head       # 문서 파일만 (%doc — man, README)
rpm -qf /usr/bin/ls                 # 파일 → 소속 패키지
rpm -qf /etc/ssh/sshd_config
rpm -qR openssh-server | head       # 요구 의존성 (= --requires)
rpm -q --whatrequires openssh       # 역의존성 (참고)
rpm -q --changelog openssh-server | head -8    # 변경 이력 (CVE 패치 확인용)
rpm -q --scripts openssh-server | head         # 설치/삭제 스크립틀릿 (참고)
rpm --querytags | head              # --queryformat 에 쓸 수 있는 태그 이름 목록 (참고)
rpm -qa --queryformat '%{NAME}-%{VERSION} %{INSTALLTIME:date}\n' | head -3   # 출력 형식 지정 (참고)
```

- `-a` : 전체 설치 패키지 (**a**ll)
- `--last` : 설치 시각 내림차순 정렬
- `-c` : 설정 파일 목록 (**c**onfig) — `-ql` 의 부분집합
- `-d` : 문서 파일 목록 (**d**oc)
- `-f <경로>` : 파일 소속 패키지 (**f**ile) — 인자는 **절대경로 파일**, 패키지 이름 아님
- `-R` : 의존성 목록 (**R**equires)
- `--whatrequires <cap>` : 해당 capability 를 요구하는 설치 패키지
- `--changelog` : 패키지 변경 이력
- `--scripts` : %pre/%post/%preun/%postun 스크립틀릿
- `--querytags` : 질의 태그 목록 / `--queryformat` : 태그로 출력 형식 지정

**검증**

```bash
rpm -qf /usr/bin/ls /etc/hosts /etc/passwd /usr/bin/tree
rpm -qc openssh-server | grep sshd_config
```

```text
coreutils-8.32-...el9.aarch64
setup-2.13.7-...el9.noarch
setup-2.13.7-...el9.noarch
tree-1.8.0-10.el9.aarch64
/etc/ssh/sshd_config
```

> 📝 **시험 포인트**: `rpm -qf <파일경로>` 가 최다 출제(R01-42, R02-39, R04-42, R06-39). `-ql <패키지>` = 파일 목록(R07-33). `-qi` 출력의 `Install Date` 존재 = 설치된 패키지(R08-36). `-qa`, `-ql`, `-qf` 를 서로 바꾼 오답 보기에 주의.

### 4-4. rpm -V 무결성 검증과 코드 표

> **상황**: 침해 점검 초동 단계에서 설치 파일이 원본과 같은지 검사한다. 정상 시스템의 기준 출력을 먼저 확보해 두고, 11절에서 고의 변경 후 비교한다.

```bash
rpm -V coreutils                    # 출력 없음 = 모든 파일 원본과 일치
rpm -V openssh-server               # 설정 파일 변경 시 c 표시 줄 등장
rpm -Va | head -20                  # 전체 패키지 검증 (수 분 소요)
rpm -Va --nofiles --nodigest        # 요약 검증 (빠름, 참고)
rpm -Vp /root/rpms/tree-*.rpm       # 파일 기준으로 설치본 검증 (참고)
```

- `-V` : 검증 (**V**erify) — 설치 DB 에 기록된 크기·다이제스트·권한·소유자·시각과 현재 파일 비교
- `-a` : 전체 패키지 (**a**ll)
- `--nofiles` : 파일 존재·속성 검사 생략 / `--nodigest` : 다이제스트 검사 생략
- `-Vp` : rpm 파일을 기준으로 검증

| 위치 | 코드 | 의미 (변경 없음 = `.`) |
| --- | --- | --- |
| 1 | `S` | 파일 **S**ize 다름 |
| 2 | `M` | **M**ode (권한·파일 유형) 다름 |
| 3 | `5` | MD**5**/SHA256 다이제스트 다름 (내용 변경) |
| 4 | `D` | **D**evice 번호 다름 (장치 파일) |
| 5 | `L` | 심볼릭 **L**ink 대상 다름 |
| 6 | `U` | 소유 **U**ser 다름 |
| 7 | `G` | 소유 **G**roup 다름 |
| 8 | `T` | 수정 **T**ime(mtime) 다름 |
| 9 | `P` | ca**P**abilities 다름 |
| — | `?` | 검사 불가 (권한 부족 등) |
| — | `missing` | 파일 삭제됨 |

- 코드 뒤 속성 문자: `c`(설정 **c**onfig) · `d`(**d**oc) · `g`(**g**host — 패키지에 내용 없이 경로만 등록) · `l`(**l**icense) · `r`(**r**eadme)

**검증**

```bash
rpm -V coreutils ; echo "exit=$?"          # 0 = 이상 없음
rpm -Va 2>/dev/null | grep -c '^' 
```

```text
exit=0
...            # 정상 minimal 설치에서도 수 건(설정 파일 c, .pyc 등) 출력 가능 — 대량이면 손상 의심
```

> 📝 **시험 포인트**: `S.5....T.  c /etc/httpd/conf/httpd.conf` = 설정 파일의 크기·해시·시각 변경(R10-38). `-V` 는 "설치된 파일 변경 검사", `-K` 는 "rpm 파일 서명 검증" — 혼동 유도(R03-31 ③). `rpm -Va` 는 침해 사고 초동 점검 항목(R07-65).

---

## 5. 그룹·모듈

### 5-1. Development Tools 그룹 설치

> **상황**: 8절 소스 컴파일에 `gcc`·`make` 가 필요하다. 개별 설치 대신 그룹 단위로 빌드 도구를 한 번에 넣는다.

```bash
dnf group list                             # 환경 그룹 / 설치된 그룹 / 사용 가능 그룹
dnf group list --hidden | head -30         # 숨김 그룹 포함
dnf group info "Development Tools"         # 구성 패키지 (Mandatory/Default/Optional)
dnf group install -y "Development Tools"   # 공백 포함 그룹명 → 따옴표 필수
dnf groupinstall -y "Development Tools"    # 구형 표기 (동일)
```

- `group list` : 그룹 목록. `--hidden` 으로 비표시 그룹 포함, `--installed`/`--available` 필터
- `group info "<그룹>"` : 필수(Mandatory)·기본(Default)·선택(Optional) 패키지 구분 출력
- `group install "<그룹>"` : Mandatory+Default 설치. `--with-optional` 로 Optional 까지
- `groupinstall` / `grouplist` / `groupinfo` : yum 호환 한 단어 표기 — 시험 지문에 이 형태 다수

**검증**

```bash
dnf group list --installed
rpm -q gcc make autoconf automake binutils gdb
gcc --version | head -1 ; make --version | head -1
```

```text
Installed Groups:
   Development Tools
gcc-11.x.x-...el9.aarch64
make-4.3-...el9.aarch64
autoconf-...  automake-...  binutils-...  gdb-...
gcc (GCC) 11.x.x ...
GNU Make 4.3
```

> 📝 **시험 포인트**: `dnf group install` 은 패키지 묶음 설치 — "저장소 추가" 로 왜곡한 오답(R10-39 ③). 그룹명 공백 → 따옴표. Development Tools = 소스 컴파일 선행 조건.

### 5-2. 모듈(AppStream 스트림) — 조회 위주

> **상황**: AppStream 은 nodejs·php 처럼 여러 버전(스트림)을 모듈로 제공한다. 개발팀이 특정 Node.js 버전을 요구할 때 스트림 선택 문법을 확인한다. 설치는 하지 않는다.

```bash
dnf module list                            # 모듈·스트림·프로파일 목록. [d]=default [e]=enabled [x]=disabled [i]=installed
dnf module list nodejs
dnf module info nodejs                     # 모든 스트림 상세
dnf module info nodejs:20                  # 특정 스트림 (스트림 번호는 module list 출력 기준)
# dnf module enable  -y nodejs:20          # ※ 참고 — 스트림 활성화만 (패키지 설치 안 함)
# dnf module install -y nodejs:20/common   # ※ 참고 — 스트림 활성화 + 프로파일 패키지 설치
# dnf module reset   -y nodejs             # ※ 참고 — 스트림 선택 해제 (패키지 삭제는 아님)
```

- `module list [이름]` : 모듈 스트림 목록과 상태 플래그
- `module info <모듈>[:스트림]` : 스트림 포함 패키지·프로파일
- `module enable <모듈>:<스트림>` : 해당 스트림만 보이게 활성화 — 한 모듈에 **한 스트림만** 활성 가능
- `module install <모듈>:<스트림>[/프로파일]` : 활성화 + 설치
- `module reset <모듈>` : 스트림 선택 초기화 (설치 파일은 남음)

**검증**

```bash
dnf module list nodejs | grep -E 'nodejs'
dnf module list --enabled 2>/dev/null | grep -c nodejs    # 0 = 아직 활성 스트림 없음
```

```text
nodejs   18   common [d], development, minimal, s2i   Javascript runtime
nodejs   20   common [d], development, minimal, s2i   Javascript runtime
...
0
```

> 📝 **시험 포인트**: `dnf module enable nodejs:18` 은 활성화만·설치 없음(R03-33 ③ 정답). 두 스트림 동시 활성 불가, `reset` 은 파일 삭제 아님 — 오답 보기 근거.

---

## 6. EPEL 저장소

### 6-1. epel-release 설치와 EPEL 패키지 설치

> **상황**: `htop`·`stress-ng` 는 기본 저장소에 없다. Part 06 진단 실습에서 CPU 부하를 걸려면 EPEL 이 필요하다. `extras` 저장소의 `epel-release` 로 저장소 정의를 추가한다.

```bash
dnf info epel-release | grep -E '^(Repository|Summary)'   # extras 저장소에 존재
dnf install -y epel-release
dnf repolist | grep -i epel
ls /etc/yum.repos.d/ | grep -i epel
grep -E '^\[|enabled|gpgkey' /etc/yum.repos.d/epel.repo | head -8
dnf makecache
dnf install -y htop stress-ng
```

- `epel-release` : EPEL(**E**xtra **P**ackages for **E**nterprise **L**inux) 의 `.repo` 파일과 GPG 키를 담은 패키지 — 설치 자체가 저장소 등록
- `/etc/yum.repos.d/epel.repo` : `[epel]` 활성, `[epel-debuginfo]`·`[epel-source]` 비활성
- `htop` : 대화식 프로세스 뷰어 (Part 06) / `stress-ng` : CPU·메모리·I/O 부하 생성기 (Part 06)

**검증**

```bash
dnf repolist | grep epel
rpm -q epel-release htop stress-ng
rpm -qi htop | grep -E '^(Vendor|Packager)'       # Fedora Project = EPEL 빌드
dnf info htop | grep -E '^(Repository|From repo)'
```

```text
epel                Extra Packages for Enterprise Linux 9 - aarch64
epel-release-9-...el9.noarch
htop-3.x.x-...el9.aarch64
stress-ng-0.x.x-...el9.aarch64
Vendor      : Fedora Project
From repo    : epel
```

> 📝 **시험 포인트**: EPEL = RHEL 계열용 추가 저장소(Fedora 프로젝트 운영). `epel-release` 는 `extras` 저장소에서 제공되며 설치 = 저장소 등록. EPEL 은 CRB 활성화를 권장(1-3 에서 선행 완료).

---

## 7. 이후 파트용 필수 유틸 일괄 설치

### 7-1. 패키지 일괄 설치

> **상황**: Part 03~12 에서 필요한 도구를 지금 한꺼번에 설치한다. 이미 설치된 것(`lvm2`, `cronie` 등)은 "already installed" 로 건너뛰므로 목록을 그대로 실행해도 안전하다.

```bash
dnf install -y \
  vim-enhanced bash-completion \
  net-tools bind-utils tcpdump nmap traceroute mtr telnet lftp wget curl \
  tar bzip2 xz zip unzip \
  sysstat lsof psmisc procps-ng util-linux-user \
  policycoreutils-python-utils setroubleshoot-server \
  quota mdadm lvm2 \
  at cronie s-nail \
  man-pages tree rsync pciutils usbutils dmidecode
```

| 묶음 | 패키지 | 쓰는 파트 |
| --- | --- | --- |
| 편집·셸 | `vim-enhanced`(vim), `bash-completion`(탭 완성) | 04 |
| 네트워크 진단 | `net-tools`(ifconfig·netstat·route), `bind-utils`(dig·nslookup·host), `tcpdump`, `nmap`, `traceroute`, `mtr`, `telnet`, `lftp`, `wget`, `curl` | 08, 09 |
| 압축 | `tar`, `bzip2`, `xz`, `zip`, `unzip` | 04, 12 |
| 프로세스·성능 | `sysstat`(sar·iostat·mpstat), `lsof`, `psmisc`(pstree·killall·fuser), `procps-ng`(ps·top·vmstat), `util-linux-user`(chsh·chfn) | 03, 06 |
| SELinux | `policycoreutils-python-utils`(semanage), `setroubleshoot-server`(sealert) | 10 |
| 디스크 | `quota`, `mdadm`, `lvm2` | 05 |
| 스케줄·메일 | `at`, `cronie`(crontab), `s-nail`(mail 명령, mailx 대체) | 06, 09 |
| 기타 | `man-pages`, `tree`, `rsync`, `pciutils`(lspci), `usbutils`(lsusb), `dmidecode` | 01, 12 |

- `\` : 줄 계속 — 한 트랜잭션으로 묶어 의존성 계산 1회
- `-y` : 설치 요약 확인 자동 승인

**검증**

```bash
dnf history info last | grep -E '^(Command Line|Packages Altered)' -A2 | head
which ifconfig dig sar semanage stress-ng mail
```

```text
Command Line   : install -y vim-enhanced bash-completion net-tools ...
Packages Altered:
    Install ...
/usr/sbin/ifconfig
/usr/bin/dig
/usr/bin/sar
/usr/sbin/semanage
/usr/bin/stress-ng
/usr/bin/mail
```

> 📝 **시험 포인트**: 명령 ↔ 패키지 대응이 출제 — `ifconfig`·`netstat` → `net-tools`, `dig`·`nslookup` → `bind-utils`, `sar`·`iostat` → `sysstat`, `semanage` → `policycoreutils-python-utils`, `mail` → `s-nail`(RHEL 9, 구 `mailx`).

### 7-2. 설치 결과 일괄 확인 스크립트

> **상황**: 수십 개 패키지 중 하나라도 빠졌으면 뒤 파트에서 "command not found" 로 발목을 잡는다. `rpm -q` 종료 코드로 일괄 검사한다.

```bash
cat > /root/check-pkgs.sh <<'EOF'
#!/bin/bash
# 인자로 받은 패키지들의 설치 여부를 rpm -q 종료 코드로 판정
missing=0
for p in "$@"; do
  if rpm -q "$p" >/dev/null 2>&1; then
    printf 'OK       %s\n' "$p"
  else
    printf 'MISSING  %s\n' "$p"; missing=$((missing+1))
  fi
done
echo "missing: $missing"
exit $missing
EOF
chmod +x /root/check-pkgs.sh
/root/check-pkgs.sh vim-enhanced bash-completion net-tools bind-utils tcpdump nmap traceroute mtr \
  telnet lftp wget curl tar bzip2 xz zip unzip sysstat lsof psmisc procps-ng util-linux-user \
  policycoreutils-python-utils setroubleshoot-server quota mdadm lvm2 at cronie s-nail \
  man-pages tree rsync pciutils usbutils dmidecode epel-release htop stress-ng tmux
dnf list installed | wc -l
```

- `rpm -q <이름>` : 설치 시 종료 코드 0, 미설치 시 1 — 출력은 `/dev/null` 로 버리고 코드만 사용
- `"$@"` : 스크립트 인자 전체 (Part 04 셸 스크립트에서 상세)
- `exit $missing` : 누락 수를 종료 코드로 반환 → 자동화 분기 가능

**검증**

```bash
/root/check-pkgs.sh net-tools no-such-pkg ; echo "exit=$?"
```

```text
OK       net-tools
MISSING  no-such-pkg
missing: 1
exit=1
```

> 📝 **시험 포인트**: `rpm -q` 는 root 불필요·종료 코드로 설치 여부 판정 가능. `dnf list installed | wc -l` 은 헤더 줄 1개 포함 — 정확 수는 `rpm -qa | wc -l`.

---

## 8. 소스 컴파일 3단계 — GNU hello

### 8-1. 소스 아카이브 받기와 해제

> **상황**: 개발팀이 "저장소에 없는 도구를 소스로 빌드해 `/usr/local` 에 넣어 달라" 고 요청했다. 가장 단순한 GNU hello 로 절차를 검증한다. 0단계는 항상 아카이브 해제다.

```bash
mkdir -p /usr/local/src && cd /usr/local/src
wget https://ftp.gnu.org/gnu/hello/hello-2.12.1.tar.gz
tar tzvf hello-2.12.1.tar.gz | head -5        # 풀기 전 목록 확인
tar xzvf hello-2.12.1.tar.gz
cd hello-2.12.1
ls
```

- `/usr/local/src` : 로컬 빌드 소스의 FHS 관례 위치
- `wget <URL>` : 파일 다운로드 (7절 설치)
- `tar t` : 목록 (**t**able) / `x` : 추출 (e**x**tract) / `z` : g**z**ip 필터 / `v` : 상세 (**v**erbose) / `f` : 파일명 (**f**ile, 항상 마지막)

**검증**

```bash
ls -l /usr/local/src/hello-2.12.1/configure /usr/local/src/hello-2.12.1/Makefile.in
ls /usr/local/src/hello-2.12.1/Makefile 2>&1        # 아직 없음 — configure 가 생성
```

```text
-rwxr-xr-x. 1 ... configure
-rw-r--r--. 1 ... Makefile.in
ls: cannot access '/usr/local/src/hello-2.12.1/Makefile': No such file or directory
```

> 📝 **시험 포인트**: 순서 문제의 첫 항목은 항상 `tar xvzf`(R02-42, R07-23). 압축 해제 전에는 `configure` 스크립트가 존재하지 않으므로 `./configure` 가 선행할 수 없음.

### 8-2. ① configure — 환경 점검과 Makefile 생성

> **상황**: 컴파일러·헤더·라이브러리가 있는지 점검하고 설치 경로를 `/usr/local` 로 지정해 Makefile 을 만든다.

```bash
./configure --help | head -40                 # 사용 가능한 옵션 (--prefix, --with-*, --enable-*)
./configure --prefix=/usr/local
echo "exit=$?"
```

- `./configure` : autoconf 가 만든 셸 스크립트 — 현재 디렉터리 실행이므로 `./` 필수
- `--help` : 옵션 목록 (**help**)
- `--prefix=<경로>` : 설치 루트. 기본값도 `/usr/local` 이지만 명시가 관례 (`bin/`, `share/man/`, `share/info/` 하위 생성)
- `--with-<lib>=<경로>` / `--enable-<기능>` : 외부 라이브러리 위치·선택 기능 지정 (참고)
- 종료 코드 0 ≠ 0 이면 누락 헤더·라이브러리 → `config.log` 에서 원인 확인, `-devel` 패키지 설치 후 재실행

**검증**

```bash
ls -l Makefile config.log config.status config.h
grep -E '^prefix|^CC =' Makefile
grep -m1 -E '^configure:.*gcc' config.log
```

```text
-rw-r--r--. 1 root root ... Makefile
-rw-r--r--. 1 root root ... config.log
-rwxr-xr-x. 1 root root ... config.status
-rw-r--r--. 1 root root ... config.h
prefix = /usr/local
CC = gcc
configure:...: checking for gcc
```

> 📝 **시험 포인트**: `configure` 의 역할 = "시스템 환경·의존성 점검 + Makefile 생성"(R03-42, R08-42, R10-40). `--prefix` = 설치 대상 디렉터리. 의존 패키지 자동 다운로드는 하지 않음(R08-42 ④ 오답).

### 8-3. ② make · ③ make install — 컴파일과 설치

> **상황**: Makefile 규칙대로 컴파일하고, 산출 바이너리를 `/usr/local` 로 복사한다. `make install` 은 시스템 경로에 쓰므로 root.

```bash
make                      # Makefile 의 기본 타깃(all) — 소스 → 목적 파일(.o) → 실행 파일
ls -l hello src/*.o 2>/dev/null | head
./hello                   # 설치 전 빌드 디렉터리에서 실행 테스트
make install              # bin/hello, share/man/man1/hello.1, share/info/hello.info 복사
```

- `make` : `Makefile` 을 읽어 의존 관계에 따라 컴파일. `-j<N>` 으로 병렬 (참고)
- `make install` : `install` 타깃 — 빌드 결과물을 `prefix` 하위로 복사 (root 필요)
- `make -n install` : 실제 복사 없이 수행할 명령만 표시 (참고, dry-run)

**검증**

```bash
which hello
hello --version | head -1
hello
ls -l /usr/local/bin/hello /usr/local/share/man/man1/hello.1
man -w hello                       # man 페이지 경로 (bash-completion·man-db 캐시 갱신 전에도 -w 는 동작)
file /usr/local/bin/hello
rpm -qf /usr/local/bin/hello       # 패키지 DB 미등록 확인
```

```text
/usr/local/bin/hello
hello (GNU Hello) 2.12.1
Hello, world!
-rwxr-xr-x. 1 root root ... /usr/local/bin/hello
-rw-r--r--. 1 root root ... /usr/local/share/man/man1/hello.1
/usr/local/share/man/man1/hello.1
/usr/local/bin/hello: ELF 64-bit LSB pie executable, ARM aarch64, ... dynamically linked, ...
file /usr/local/bin/hello is not owned by any package
```

> 📝 **시험 포인트**: `make` = 컴파일, `make install` = 복사 — 둘의 역할을 바꾼 보기(R10-40 ③) 오답. 소스 설치는 rpm DB 에 없음 → `rpm -qf` 가 "not owned" (이론 1-3 표). `which` 결과가 `/usr/local/bin` 이면 소스 설치본, `/usr/bin` 이면 패키지본.

### 8-4. make clean · make uninstall · 대안 도구

> **상황**: 빌드 중간 산출물을 정리하고, 제거 방법을 확인한다. hello 는 `uninstall` 타깃을 제공하므로 실제 제거 후 다시 설치해 둔다 (9절 `ldd` 대상).

```bash
make clean                          # .o, 빌드 바이너리 삭제 (Makefile 은 유지)
ls hello 2>&1
make uninstall                      # prefix 에 복사된 파일 제거 (소스에 uninstall 규칙이 있을 때만)
which hello 2>&1
make && make install                # 9절 실습용으로 재설치
make distclean                      # configure 산출물(Makefile, config.*) 까지 삭제 → 재 configure 필요 (참고)
./configure --prefix=/usr/local >/dev/null && make >/dev/null   # distclean 뒤 원상복구 (참고)
```

- `clean` : 컴파일 산출물 삭제 (재빌드용)
- `uninstall` : `install` 의 역 — 소스 Makefile 에 규칙이 없으면 실패. 이 경우 `make -n install` 로 복사 목록을 확인해 수동 삭제
- `distclean` : `clean` + configure 산출물 삭제 (배포 상태로 복원)

| 도구 | 역할 | 이 파트 포함 여부 |
| --- | --- | --- |
| `checkinstall` | `make install` 을 가로채 rpm/deb 패키지 생성 | **미포함** — Rocky 9 기본·EPEL 9 저장소에 없고 개발 중단 상태로 실습 불가 |
| `rpmbuild` | `.spec`/`.src.rpm` 으로 바이너리 rpm 빌드 (`rpmbuild --rebuild pkg.src.rpm`) | 참고 — R07-24 |
| `cmake` | `configure` 대신 `CMakeLists.txt` 로 Makefile 생성 (`cmake . && make`) | 참고 — 최신 C/C++ 프로젝트 표준 |
| `meson`/`ninja` | autotools 대체 빌드 시스템 | 참고 |

**검증**

```bash
which hello && hello --version | head -1
ls /usr/local/src/hello-2.12.1/Makefile
```

```text
/usr/local/bin/hello
hello (GNU Hello) 2.12.1
/usr/local/src/hello-2.12.1/Makefile
```

> 📝 **시험 포인트**: 제거는 `make uninstall`(소스 규칙 필요), 정리는 `make clean`. 소스 RPM → 바이너리 RPM 은 `rpmbuild --rebuild`(R07-24). httpd 소스 빌드 의존 순서 apr → apr-util → pcre → httpd(R07-68) — 의존 라이브러리 먼저.

---

## 9. 공유 라이브러리

### 9-1. ldd — 실행 파일의 공유 라이브러리 의존성

> **상황**: 방금 빌드한 hello 와 시스템 `ls` 가 어떤 `.so` 를 실행 시 로드하는지 확인한다. `not found` 가 있으면 실행 불가 원인이다.

```bash
ldd /usr/local/bin/hello
ldd /bin/ls
ldd /usr/bin/ssh | grep -E 'crypto|libc\.so'
ldd -v /bin/ls | head -20            # 버전 심볼 정보까지 (참고)
```

- `ldd <실행파일>` : 동적 링커가 해석한 공유 라이브러리 목록과 적재 주소 출력 (**l**ist **d**ynamic **d**ependencies)
- `linux-vdso.so.1` : 커널이 제공하는 가상 라이브러리 (파일 없음)
- `=> /lib64/xxx.so` : 실제 해석된 경로. `not found` = 누락
- `/lib/ld-linux-aarch64.so.1` : 동적 링커(로더) 자체 — x86_64 는 `/lib64/ld-linux-x86-64.so.2`
- `-v` : 상세 (**v**erbose)

⚠️ `ldd` 는 신뢰할 수 없는 바이너리에 쓰지 않음 — 일부 구현은 프로그램을 실제 실행함. 출처 불명 파일은 `objdump -p <파일> | grep NEEDED` 로 대체.

**검증**

```bash
ldd /usr/local/bin/hello | grep -c 'not found'      # 0 이어야 정상
ldd /bin/ls | awk '/=>/{print $3}' | xargs rpm -qf | sort -u
```

```text
0
glibc-2.34-...el9.aarch64
libcap-2.48-...el9.aarch64
libselinux-3.x-...el9.aarch64
pcre2-...el9.aarch64
```

```text
# ldd /usr/local/bin/hello 기대 출력
        linux-vdso.so.1 (0x...)
        libc.so.6 => /lib64/libc.so.6 (0x...)
        /lib/ld-linux-aarch64.so.1 (0x...)
```

> 📝 **시험 포인트**: `ldd /bin/ls` 출력 해석 = "의존하는 공유 라이브러리 목록"(R02-43, R08-38). 정적 포함 목록·캐시 재생성으로 왜곡한 오답. `not found` 줄 → `dnf provides '*/libxxx.so.N'` 으로 패키지 찾기.

### 9-2. ld.so.conf · ldconfig — 검색 경로와 캐시

> **상황**: 동적 링커가 라이브러리를 어디서 찾는지, 그 결과를 캐시한 `/etc/ld.so.cache` 를 어떻게 갱신하는지 확인한다.

```bash
cat /etc/ld.so.conf
ls -l /etc/ld.so.conf.d/
cat /etc/ld.so.conf.d/*.conf
ldconfig -p | head -5                 # 캐시 내용 (첫 줄에 개수)
ldconfig -p | grep -E 'libc\.so|libz\.so'
ldconfig                              # 캐시 재생성 (새 .so 추가·경로 추가 후 필수)
ldconfig -v 2>/dev/null | grep -vE '^\s' | head    # 검색 디렉터리 목록만 (참고)
```

- `/etc/ld.so.conf` : 추가 검색 경로 설정. Rocky 9 는 `include ld.so.conf.d/*.conf` 한 줄
- `/etc/ld.so.conf.d/*.conf` : 패키지별 경로 조각 (예: `dyninst-aarch64.conf`, `libiscsi-aarch64.conf`)
- 기본 경로 `/lib64`, `/usr/lib64` 는 설정 없이도 검색 (신뢰 경로)
- `/etc/ld.so.cache` : 경로→라이브러리 매핑 캐시 (바이너리 파일)
- `ldconfig` : 설정 파일과 신뢰 경로를 훑어 `/etc/ld.so.cache` 재생성 + soname 심볼릭 링크 정리
- `-p` : 캐시 내용 출력 (**p**rint)
- `-v` : 스캔 디렉터리·링크 생성 과정 표시 (**v**erbose)

**검증**

```bash
ls -l /etc/ld.so.cache
ldconfig -p | head -1
file /etc/ld.so.cache
```

```text
-rw-r--r--. 1 root root ... /etc/ld.so.cache        # ldconfig 실행 시각으로 갱신됨
... libs found in cache `/etc/ld.so.cache'
/etc/ld.so.cache: data
```

> 📝 **시험 포인트**: "`/etc/ld.so.conf` 에 경로 추가 후 캐시 갱신" = `ldconfig`(R05-44, R10-41 ③). `ldd` 는 조회, `ldconfig` 는 갱신 — 둘을 바꾼 오답.

### 9-3. LD_LIBRARY_PATH — 임시 검색 경로

> **상황**: 시스템 경로 밖(예: `/opt/app/lib`) 에 놓인 라이브러리를 캐시 갱신 없이 임시로 찾게 하는 환경 변수를 실험한다. 라이브러리를 복사해 우선순위가 바뀌는 것을 `ldd` 로 확인한다.

```bash
mkdir -p /opt/labtest/lib
cp /lib64/libz.so.1 /opt/labtest/lib/                     # 실험용 복사 (원본 유지)
ldd /usr/bin/gzip | grep libz                             # 기본: /lib64/libz.so.1
LD_LIBRARY_PATH=/opt/labtest/lib ldd /usr/bin/gzip | grep libz    # 변수 지정 시: /opt/labtest/lib 우선
export LD_LIBRARY_PATH=/opt/labtest/lib                   # 현재 셸에 지속 적용
ldd /usr/bin/gzip | grep libz
unset LD_LIBRARY_PATH                                     # 실험 종료 — 반드시 해제
```

- `LD_LIBRARY_PATH` : 콜론 구분 디렉터리 목록. 동적 링커가 캐시·기본 경로 **보다 먼저** 검색
- `VAR=값 명령` : 그 명령 한 번에만 적용 / `export` : 이후 자식 프로세스 전체
- 영구 등록은 `/etc/ld.so.conf.d/labtest.conf` 에 경로를 쓰고 `ldconfig` — 환경 변수 방식은 임시·디버깅용

⚠️ `LD_LIBRARY_PATH` 를 전역 프로필(`/etc/profile`)에 두면 SetUID 프로그램·보안 취약점 원인 — 실습 후 `unset` 확인.

**검증**

```bash
echo "LD_LIBRARY_PATH=[$LD_LIBRARY_PATH]"                # 비어 있어야 함
ldd /usr/bin/gzip | grep libz
rm -rf /opt/labtest
```

```text
LD_LIBRARY_PATH=[]
        libz.so.1 => /lib64/libz.so.1 (0x...)
```

> 📝 **시험 포인트**: `LD_LIBRARY_PATH` = 표준 경로 외 라이브러리를 임시 검색하게 하는 환경 변수. 영구 방식은 `ld.so.conf(.d)` + `ldconfig`.

### 9-4. .so 버전 링크 구조와 정적 vs 동적

> **상황**: `libz.so.1` 이 실제 파일이 아니라 심볼릭 링크임을 보고, soname·real name·linker name 세 층의 관계를 이해한다.

```bash
ls -l /lib64/libz.so*
ls -l /lib64/libc.so*                  # glibc 2.34+ 는 libc.so.6 자체가 실제 파일 (링크 아님)
ls -l /lib64/libselinux.so*
readelf -d /lib64/libz.so.1 | grep SONAME
ls /usr/lib64/*.a 2>/dev/null | head   # 정적 라이브러리 (glibc-static 등 미설치면 거의 없음)
file /lib64/libz.so.1.*
```

- `libz.so.1.2.11` : **real name** — 실제 파일
- `libz.so.1` : **soname** — 실행 파일이 참조하는 이름 (`ldconfig` 가 real name 을 가리키는 링크 생성)
- `libz.so` : **linker name** — 컴파일(`gcc -lz`) 시 사용, `zlib-devel` 패키지가 제공하는 링크
- `readelf -d` : ELF 동적 섹션 (**d**ynamic) — `SONAME` 태그로 라이브러리가 선언한 soname 확인
- `.a` : 정적 라이브러리 (**a**rchive) — 링크 시 실행 파일에 포함

| 구분 | 정적 링크 (`.a`) | 동적 링크 (`.so`) |
| --- | --- | --- |
| 결합 시점 | 컴파일(링크) 시 실행 파일에 포함 | 실행 시 로더가 적재 |
| 실행 파일 크기 | 큼 | 작음 |
| 라이브러리 패치 | **재컴파일 필요** | `.so` 교체만으로 반영 |
| 메모리 | 프로세스마다 복사 | 여러 프로세스 공유 |
| 배포 | 단일 파일로 이식 용이 | 대상 시스템에 `.so` 필요 (`not found` 위험) |
| 확인 | `file` → `statically linked` | `file` → `dynamically linked`, `ldd` 로 목록 |

**검증**

```bash
readlink -f /lib64/libz.so.1
file /usr/local/bin/hello | grep -o 'dynamically linked'
ls -l /lib64/libz.so.1 | grep -c -- '->'
```

```text
/usr/lib64/libz.so.1.2.11
dynamically linked
1
```

> 📝 **시험 포인트**: 정적 링크 파일은 라이브러리만 교체해도 패치 미적용 → 재컴파일(R10-41 ④ 오답). `.a`=정적, `.so`=공유. LGPL 라이브러리를 **동적** 링크하면 응용 소스 공개 의무 없음(R03 라이선스 문항 연계).

---

## 10. Debian 계열 비교 · 기타 패키지 도구

### 10-1. rpm/dnf ↔ dpkg/apt 대응표

> **상황**: 시험은 두 계열을 나란히 묻는다. 실습은 [[11-container-virtualization]] 의 `lab-ubuntu` 컨테이너에서 하고, 여기서는 이번 파트에서 수행한 명령을 Debian 명령으로 1:1 치환해 둔다.

| 기능 | RedHat (이 파트에서 수행) | Debian 저수준 `dpkg` | Debian 고수준 `apt` / `apt-get` · `apt-cache` |
| --- | --- | --- | --- |
| 패키지 파일 | `.rpm` | `.deb` | — |
| 로컬 파일 설치 | `rpm -ivh pkg.rpm` | `dpkg -i pkg.deb` | `apt install ./pkg.deb` |
| 삭제(설정 유지) | `rpm -e pkg` / `dnf remove` | `dpkg -r pkg` | `apt remove pkg` / `apt-get remove` |
| 완전 삭제(설정 포함) | — (`rpm -e` 는 설정을 `.rpmsave` 로 보존) | `dpkg -P pkg` | `apt purge pkg` / `apt-get purge` |
| 설치 목록 | `rpm -qa` | `dpkg -l` | `apt list --installed` |
| 포함 파일 | `rpm -ql pkg` | `dpkg -L pkg` | — |
| 파일→패키지 | `rpm -qf /path` | `dpkg -S /path` | `apt-file search`(별도 패키지) |
| 저장소 설치 | `dnf install pkg` | — | `apt install pkg` / `apt-get install` |
| 목록 갱신 | `dnf makecache` | — | `apt update` / `apt-get update` |
| 전체 업그레이드 | `dnf upgrade` (= `update`) | — | `apt upgrade` / `apt-get upgrade` (`full-upgrade` = 의존성 변경 허용) |
| 검색 | `dnf search kw` | — | `apt search kw` / `apt-cache search kw` |
| 상세 정보 | `dnf info pkg` | `dpkg -s pkg` | `apt show pkg` / `apt-cache show pkg` |
| 의존성 | `dnf repoquery --requires` | — | `apt-cache depends pkg` |
| 불필요 정리 | `dnf autoremove` | — | `apt autoremove` |
| 저장소 설정 | `/etc/yum.repos.d/*.repo` | — | `/etc/apt/sources.list`, `/etc/apt/sources.list.d/*.list` (`deb http://archive.ubuntu.com/ubuntu jammy main`) |
| GPG 키 | `rpm --import` | — | `/etc/apt/trusted.gpg.d/`, `apt-key`(구형) |
| 설정 재구성 | — | `dpkg-reconfigure pkg` | — |

- `apt` : 대화형 통합 명령(진행바·색상). `apt-get`/`apt-cache` : 스크립트용 안정 인터페이스 — 기능은 같음
- `dpkg -r` vs `-P` : **r**emove 는 `/etc` 설정 유지, **P**urge 는 설정까지 삭제
- `apt update`(목록 갱신) ≠ `apt upgrade`(패키지 갱신) — RedHat `dnf update` 와 의미 다름

| 배포판 | 패키지 형식 | 저수준 | 고수준 |
| --- | --- | --- | --- |
| RHEL · Rocky · CentOS · Fedora | `.rpm` | `rpm` | `dnf` / `yum` |
| Debian · Ubuntu · Linux Mint | `.deb` | `dpkg` | `apt` / `apt-get` |
| openSUSE · SLES | `.rpm` | `rpm` | `zypper` |
| Arch Linux · Manjaro | `.pkg.tar.zst` | — | `pacman` |
| Gentoo | 소스(ebuild) | — | `emerge`(Portage) |
| 배포판 독립 | 샌드박스 번들 | — | `snap`(Canonical), `flatpak`(Fedora 계열 GUI 앱), `AppImage` |

**검증**

```bash
which dpkg apt 2>&1              # Rocky 에는 없음 — Part 11 컨테이너에서 실행
rpm -q rpm dnf | sed 's/-[0-9].*//'
```

```text
/usr/bin/which: no dpkg in (...)
/usr/bin/which: no apt in (...)
rpm
dnf
```

> 📝 **시험 포인트**: `.deb` 로컬 설치 = `dpkg -i`(R01-44, R02-41, R07-35) — `dpkg -r pkg.deb`·`apt-cache install` 은 오답. openSUSE = `zypper`(R05-04 ③ 오답), RPM 계열 짝 = CentOS·openSUSE(R02-04). Rocky 는 RHEL 호환 `dnf`/`rpm`(R08-05, R10-04).

---

## 11. 패키지 검증 시나리오 — 설정 파일 변경 탐지와 복원

### 11-1. 소속 확인 → 기준 검증

> **상황**: 운영자가 "누가 시스템 파일을 건드린 것 같다" 고 신고했다. 의심 파일의 소속 패키지를 찾고, 변경 전 기준 상태를 검증한다.

```bash
rpm -qf /bin/ls                     # coreutils
rpm -V coreutils ; echo "exit=$?"   # 출력 없음 + 0 = 원본 그대로
rpm -qf /etc/hosts                  # setup
rpm -qc setup                       # setup 이 관리하는 설정 파일 목록 (/etc/hosts, /etc/passwd, /etc/group, /etc/shells …)
rpm -V setup ; echo "exit=$?"       # 기준 상태 (passwd/group 등은 %verify 예외로 보통 미출력)
```

- `/bin/ls` : Rocky 9 는 `/bin → usr/bin` 링크이므로 `/usr/bin/ls` 와 동일 파일
- `setup` : `/etc/hosts`, `/etc/passwd`, `/etc/group`, `/etc/shells`, `/etc/protocols`, `/etc/services` 등 기본 설정 골격을 담은 noarch 패키지
- `%config(noreplace)` : rpm 이 설정 파일에 붙이는 표식 — 업그레이드 시 사용자 변경본을 덮어쓰지 않음 (11-3)

**검증**

```bash
cp -p /etc/hosts /root/hosts.orig          # 11-3 복원용 백업 (-p 로 시각·권한 보존)
sha256sum /etc/hosts /root/hosts.orig
```

```text
<해시>  /etc/hosts
<해시>  /root/hosts.orig                   # 두 줄 해시 동일
```

> 📝 **시험 포인트**: `rpm -qf /etc/hosts` → `setup`, `rpm -qf /etc/httpd/conf/httpd.conf` → `httpd`(R06-39). `-V` 종료 코드 0 = 이상 없음.

### 11-2. 고의 변경 후 rpm -V 코드 읽기

> **상황**: Part 09 DNS 구축 전까지 이름 풀이가 필요해 `/etc/hosts` 에 서버 자기 항목을 추가한다 (실제 운영에서도 하는 정상 변경). 이 변경이 `rpm -V` 에 어떻게 잡히는지 확인한다.

```bash
echo '192.168.64.10   srv01.lab.local srv01' >> /etc/hosts
cat /etc/hosts
rpm -V setup
rpm -Va 2>/dev/null | grep hosts
```

- `>>` : 파일 끝에 추가 (덧붙이기) — `>` 는 덮어쓰기이므로 주의
- `S.5....T.  c /etc/hosts` : 크기(S)·다이제스트(5)·수정 시각(T) 변경, 속성 `c` = 설정 파일 → "설정 파일이 편집됨" 으로 해석. 권한(M)·소유자(U)·그룹(G) 은 `.` 로 불변

**검증**

```bash
rpm -V setup ; echo "exit=$?"
getent hosts srv01.lab.local            # 추가한 항목이 실제 이름 풀이에 사용됨
```

```text
S.5....T.  c /etc/hosts
exit=1
192.168.64.10   srv01.lab.local srv01
```

> 📝 **시험 포인트**: R10-38 과 동일 패턴 — `S.5....T.  c` 를 "소유자·그룹 변경"(U·G 는 `.`) 이나 "파일 삭제"(`missing`) 로 읽은 보기는 오답. `-V` 는 파일이 **바뀌었다** 만 알려주고 정상 변경인지 침해인지는 판단하지 않음.

### 11-3. 복원 — reinstall 의 한계와 .rpmnew / .rpmsave

> **상황**: 변경이 침해로 판명되면 원본으로 되돌려야 한다. `dnf reinstall setup` 으로 복구되는지 시험해 보고, 안 되는 이유(`%config(noreplace)`)와 실제 복원 방법을 익힌다.

```bash
dnf reinstall -y setup
rpm -V setup                           # 여전히 S.5....T. — 편집된 설정 파일은 재설치로 덮어쓰지 않음
ls /etc/hosts*                         # .rpmnew 도 생성되지 않음 (같은 버전 = 패키지 내 원본 내용 불변)

# 방법 1) 사전 백업본 복원
cp -p /root/hosts.orig /etc/hosts
rpm -V setup ; echo "exit=$?"          # 크기·해시 원복, T 만 남을 수 있음(-p 로 시각까지 복원되면 0)

# 방법 2) 백업이 없을 때 — rpm 파일에서 원본 추출
mkdir -p /root/extract && cd /root/extract
dnf download setup
rpm2cpio setup-*.rpm | cpio -idmv ./etc/hosts
diff /root/extract/etc/hosts /etc/hosts && echo SAME
# cp -p /root/extract/etc/hosts /etc/hosts     # 필요 시 이 명령으로 원본 복사
```

- `dnf reinstall` : 동일 버전 재설치 — **바이너리·비설정 파일**은 복구되지만 `%config(noreplace)` 파일은 그대로 둠
- `rpm2cpio <rpm>` : rpm 의 페이로드를 cpio 아카이브 스트림으로 출력
- `cpio -i` : 추출 (**i**nput) / `-d` : 디렉터리 생성 (**d**irectories) / `-m` : 수정 시각 보존 (**m**odification) / `-v` : 상세 (**v**erbose). 뒤의 `./etc/hosts` 는 추출 대상 패턴
- `diff` : 두 파일 차이 (출력 없음 = 동일)

| 상황 (config 파일) | `%config` | `%config(noreplace)` |
| --- | --- | --- |
| 디스크 파일 미변경 | 새 파일로 교체 | 새 파일로 교체 |
| 디스크 파일 변경 + 새 패키지의 파일이 원본과 **동일** (재설치) | 디스크 파일 유지 (조용히) | 디스크 파일 유지 (조용히) |
| 디스크 파일 변경 + 새 패키지의 파일이 원본과 **다름** (업그레이드) | 디스크 파일을 `.rpmsave` 로 백업 후 새 파일 설치 | 디스크 파일 유지, 새 파일을 `.rpmnew` 로 저장 |

- `.rpmnew` : 업그레이드 시 "새 기본 설정" 이 옆에 놓인 것 → 관리자가 `diff` 후 병합
- `.rpmsave` : 업그레이드가 내 설정을 밀어내고 백업해 둔 것 → 필요한 항목을 새 파일로 옮김
- 업그레이드 후 점검: `find /etc -name '*.rpmnew' -o -name '*.rpmsave'`

**검증**

```bash
rpm -V setup ; echo "exit=$?"
sha256sum /etc/hosts /root/hosts.orig
find /etc -name '*.rpmnew' -o -name '*.rpmsave'
```

```text
exit=0
<해시>  /etc/hosts
<해시>  /root/hosts.orig                   # 동일
                                           # 출력 없음 — 이번 실습에서는 .rpmnew/.rpmsave 미생성
```

- 복원 확인 뒤, Part 08 까지 이름 풀이가 필요하면 `echo '192.168.64.10   srv01.lab.local srv01' >> /etc/hosts` 를 다시 적용 (정상 운영 변경으로 `rpm -V` 에 계속 표시됨 — 기준선으로 기록)

> 📝 **시험 포인트**: `rpm -V` 결과가 정상 변경인지 판단하려면 `-qc` 로 설정 파일인지 확인. `.rpmnew`(새 기본값 보류) vs `.rpmsave`(내 설정 백업) 구분. 원본 추출은 `rpm2cpio | cpio -idmv`(R07-24 ④ 의 rpm2cpio 역할 = 페이로드 추출, 바이너리 rpm 생성 아님).

---

## 체크리스트

| 수행 항목 | 명령 | 확인 방법 | ☐ |
| --- | --- | --- | --- |
| 활성·비활성 저장소 확인 | `dnf repolist --all` | baseos/appstream/extras enabled, crb 상태 | ☐ |
| `.repo` 키 5종 확인 | `grep -E '^\[\|enabled\|gpgcheck\|gpgkey' /etc/yum.repos.d/rocky.repo` | name/baseurl(mirrorlist)/enabled/gpgcheck/gpgkey | ☐ |
| CRB 활성화 | `dnf config-manager --set-enabled crb` | `dnf repolist \| grep crb` | ☐ |
| dnf.conf 키 확인 | `cat /etc/dnf/dnf.conf` | gpgcheck·installonly_limit·best | ☐ |
| 캐시 정리·갱신·업데이트 유무 | `dnf clean all; dnf makecache; dnf check-update` | `echo $?` = 0/100 | ☐ |
| search/info/list | `dnf info net-tools` | Available/Installed 헤더 | ☐ |
| 파일 제공 패키지 | `dnf provides '*/ifconfig'` | net-tools | ☐ |
| 미설치 파일 목록·의존성 | `dnf repoquery -l net-tools`, `--requires` | 목록 출력 | ☐ |
| 이력 조회 | `dnf history`, `history info last` | ID 증가 | ☐ |
| install/remove/autoremove/undo/reinstall | `dnf … tmux` | `rpm -q tmux`, history 4건 | ☐ |
| 커널 제외 업그레이드 | `dnf upgrade -y --exclude='kernel*'` | `dnf check-update; echo $?` | ☐ |
| rpm 파일 다운로드·-qp·-K | `dnf download tree`, `rpm -qpi`, `rpm -K` | `digests signatures OK` | ☐ |
| rpm -ivh/-Uvh/-Fvh/-e | `rpm -ivh tree-*.rpm` … `rpm -e tree` | `rpm -q tree` | ☐ |
| 의존성 거부 확인 (안전) | `rpm -e --test glibc-common` | `Failed dependencies` | ☐ |
| rpm 질의 9종 | `-qa --last`, `-qi`, `-ql`, `-qc`, `-qd`, `-qf`, `-qR`, `--changelog` | 출력 확인 | ☐ |
| rpm -V 기준선 | `rpm -V coreutils; rpm -Va \| head` | exit 0 | ☐ |
| Development Tools | `dnf group install -y "Development Tools"` | `gcc --version`, `make --version` | ☐ |
| 모듈 조회 | `dnf module list nodejs`, `module info` | 스트림 목록 | ☐ |
| EPEL | `dnf install -y epel-release; dnf install -y htop stress-ng` | `dnf repolist \| grep epel`, `rpm -q htop` | ☐ |
| 필수 유틸 일괄 설치 | 7-1 목록 | `/root/check-pkgs.sh …` missing 0 | ☐ |
| 소스 해제·configure | `tar xzvf`, `./configure --prefix=/usr/local` | `ls Makefile config.log` | ☐ |
| make·make install | `make; make install` | `which hello`, `hello --version` | ☐ |
| make clean·uninstall | `make clean; make uninstall; make && make install` | `which hello` | ☐ |
| ldd | `ldd /usr/local/bin/hello`, `ldd /bin/ls` | `not found` 0건 | ☐ |
| ld.so.conf·ldconfig | `cat /etc/ld.so.conf; ldconfig; ldconfig -p \| head -1` | 캐시 시각 갱신 | ☐ |
| LD_LIBRARY_PATH | `LD_LIBRARY_PATH=/opt/labtest/lib ldd /usr/bin/gzip` | 경로 변화 → `unset` | ☐ |
| .so 링크 구조 | `ls -l /lib64/libz.so*`, `readelf -d … \| grep SONAME` | soname → real name | ☐ |
| /etc/hosts 변경 탐지 | `echo … >> /etc/hosts; rpm -V setup` | `S.5....T.  c /etc/hosts` | ☐ |
| 원본 복원 | `cp -p /root/hosts.orig /etc/hosts` 또는 `rpm2cpio \| cpio -idmv` | `rpm -V setup` exit 0 | ☐ |

---

## 기출 연결

| 기출 (문제집·회차·번호 또는 주제) | 이 파트의 단계 |
| --- | --- |
| 필기 R01-42 `rpm -qf /usr/bin/vim` 용도 | 4-3, 11-1 |
| 필기 R01-43 `dnf` 설명 중 틀린 것 (`list installed`) | 2-1, 3-1 |
| 필기 R01-44 `.deb` 로컬 설치 `dpkg -i` | 10-1 |
| 필기 R01-45 `./configure` → `make` → `make install` | 8-2 ~ 8-3 |
| 필기 R02-04 RPM 계열 배포판 (CentOS, openSUSE) | 10-1 배포판 표 |
| 필기 R02-39 `/etc/vsftpd/vsftpd.conf` 소속 → `rpm -qf` | 4-3 |
| 필기 R02-40 `dnf history` / `list installed` / `remove` | 2-3, 3-1 |
| 필기 R02-41 `dpkg -i pkg.deb` | 10-1 |
| 필기 R02-42 `tar xvzf` → `./configure` → `make` → `make install` 순서 | 8-1 ~ 8-3 |
| 필기 R02-43 `ldd /bin/ls` 출력 해석 | 9-1 |
| 필기 R03-31 `rpm --import` → `rpm -K`(`--checksig`), `gpgcheck` | 1-4, 4-1 |
| 필기 R03-32 `dnf history info` / `undo <ID>` | 2-3, 3-1 |
| 필기 R03-33 `dnf module enable nodejs:18` = 활성화만 | 5-2 |
| 필기 R03-42 configure 역할 (환경 점검 + Makefile 생성) | 8-2 |
| 필기 R03-54 · R07-50 커널 컴파일 순서 (`make menuconfig` → `make` → `modules_install` → `install`) | 8-3 참고 (커널은 Part 05 모듈 절에서 개념 연계) |
| 필기 R04-41 `-Uvh` (있으면 갱신·없으면 설치) | 4-2 |
| 필기 R04-42 `rpm -qf` | 4-3 |
| 필기 R04-43 `dnf provides` ≠ 저장소 목록 | 1-1, 2-2 |
| 필기 R05-04 openSUSE = `zypper` (apt 아님) | 10-1 배포판 표 |
| 필기 R05-44 `/etc/ld.so.conf` 추가 후 `ldconfig` | 9-2 |
| 필기 R06-39 `/etc/httpd/conf/httpd.conf` 소속 → `rpm -qf` | 4-3, 11-1 |
| 필기 R06-40 `rpm -ivh` 의존성 오류 → `dnf install` (`--nodeps` 비권장) | 4-2 |
| 필기 R07-23 `app-1.0.tar.gz` 컴파일 절차 (`--prefix`) | 8-1 ~ 8-3 |
| 필기 R07-24 `.src.rpm` → `rpmbuild --rebuild`, `rpm2cpio` 역할 | 8-4 표, 11-3 |
| 필기 R07-33 `rpm -ql httpd` 파일 목록 | 4-3 |
| 필기 R07-34 `/usr/bin/ss` 제공 패키지 → `dnf provides` | 2-2 |
| 필기 R07-35 `dpkg -i pkg.deb` | 10-1 |
| 필기 R07-65 침해 초동 점검 `rpm -Va` | 4-4, 11-2 |
| 필기 R07-68 httpd 소스 빌드 의존 순서 apr → apr-util → pcre → httpd | 8-4 시험 포인트 |
| 필기 R08-05 · R10-04 Rocky = RHEL 호환 `dnf`/`rpm` | 10-1 배포판 표 |
| 필기 R08-36 `rpm -qi httpd` 출력 해석 (Release `8.el9`, Install Date) | 4-1, 4-3 |
| 필기 R08-37 `dnf info nginx` `Available Packages` = 미설치 | 2-1 |
| 필기 R08-38 `ldd /usr/bin/ssh` 해석 | 9-1 |
| 필기 R08-42 configure 역할 (의존 패키지 자동 다운로드 아님) | 8-2 |
| 필기 R10-38 `rpm -V` 출력 `S.5....T.  c` 해석 | 4-4 표, 11-2 |
| 필기 R10-39 `dnf history undo`, `group install` ≠ 저장소 추가, 의존성 자동 해결 | 2-3, 3-1, 5-1 |
| 필기 R10-40 `./configure --prefix=/opt/nginx` → `make` → `make install` 해석 | 8-2 ~ 8-3 |
| 필기 R10-41 정적(.a)·동적(.so) 라이브러리, `ldconfig` | 9-2, 9-4 표 |
| 실기 R01-13 · R02-03 · R03 `tar` 옵션 (c/x/z/v/f, `-C`) | 8-1 (상세 실습은 [[04-file-text-shell]]) |

---

## 이전 / 다음

[[01-vm-setup-and-inspection]] ← · → [[03-user-group-permission]]

[[README]] · 이론: [[../THEORY/package-software]] · 실사용 문서: [[../../PACKAGE-MANAGEMENT/dnf]] · [[../../PACKAGE-MANAGEMENT/rpm]] · [[../../PACKAGE-MANAGEMENT/dnf-group]] · [[../../SYSTEM-INFO/ldd]]
