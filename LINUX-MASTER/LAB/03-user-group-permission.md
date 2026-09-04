---
title: LAB 03 — 사용자·그룹·권한 관리
type: exam-lab
part: 03
tags:
  - exam/linux-master
  - exam/lab
  - linux/account
  - linux/permission
  - linux/security
  - task/configure
  - task/verify
related: ["[[README]]", "[[02-package-management]]", "[[04-file-text-shell]]", "[[../THEORY/user-permission]]", "[[../THEORY/system-security]]"]
distro: Rocky Linux 9 (aarch64, UTM)
updated: 2026-09-03
---

# LAB 03 — 사용자·그룹·권한 관리

- 팀 계정 4개(`dev1` `dev2` `ops1` `guest1`)·그룹 2개(`devteam` 2000, `opsteam` 2001) 생성 → `/etc/passwd`·`/etc/shadow`·`/etc/group` 필드 단위 검증
- 비밀번호 정책(`chage`·`pwquality`·PAM)·`pam_wheel` 로 `su` 제한·`limits.conf` 자원 제한
- 운영자 `ops1` 에게 `/etc/sudoers.d/` 로 **제한된 sudo** 부여 → `/var/log/secure` 로그 확인
- 팀 공유 디렉터리 `/srv/devteam`(SetGID 2770)·`/srv/dropbox`(Sticky 1777)·ACL·`chattr` 속성 실습, 임시 계정 `guest1` 은 잠금 → 만료 → 삭제 수명주기

> **이 파트의 시나리오**: [[02-package-management]] 에서 EPEL·개발 도구까지 갖춘 `srv01.lab.local` 에 팀원이 합류한다. 개발 2명·운영 1명·외부 임시 1명의 계정과 그룹을 만들고, 회사 보안 규정(비밀번호 90일·최소 8자·wheel 만 su)을 적용하며, 운영자에게는 서비스 재시작·로그 조회만 sudo 로 허용한다. 여기서 만든 `dev1`·`ops1`·`devteam` 은 Part 05(쿼터)·Part 09(Samba·FTP·메일)에서 그대로 사용된다.

---

## 1. 계정 생성 기본값 파악

### 1-1. `useradd -D` 와 `/etc/default/useradd`

> **상황**: 계정을 만들기 전에 `useradd` 가 옵션 생략 시 무엇을 기본값으로 쓰는지 확인한다. 기본 셸·홈·만료 정책이 팀 규정과 맞는지 먼저 본다.

```bash
useradd -D                      # 기본값 조회 (= /etc/default/useradd 내용)
cat /etc/default/useradd        # 실제 파일
```

- `-D` : 기본값 조회·변경 (**D**efaults). `-D` 뒤에 `-s /bin/zsh` 처럼 옵션을 붙이면 기본값 **변경**
- `GROUP=100` : `-g` 생략 시 기본 그룹(단, `USERGROUPS_ENAB yes` 이면 사용자명과 같은 개인 그룹이 대신 생성됨)
- `HOME=/home` : 홈 디렉터리 상위 경로
- `INACTIVE=-1` : 비밀번호 만료 후 계정 비활성까지 유예일 (`-1` = 비활성화 기능 미사용)
- `EXPIRE=` : 계정 만료일 (빈값 = 만료 없음)
- `SHELL=/bin/bash` : 로그인 셸
- `SKEL=/etc/skel` : 홈에 복사되는 템플릿 디렉터리
- `CREATE_MAIL_SPOOL=yes` : `/var/spool/mail/<user>` 생성 여부

**검증**

```bash
diff <(useradd -D) <(grep -v '^#' /etc/default/useradd) && echo SAME
```

```text
GROUP=100
HOME=/home
INACTIVE=-1
EXPIRE=
SHELL=/bin/bash
SKEL=/etc/skel
CREATE_MAIL_SPOOL=yes
...
SAME
```

> 📝 **시험 포인트**: `useradd -D` 출력 = `/etc/default/useradd`. `INACTIVE=-1` 의 의미(비활성 미사용), `useradd -D -s /bin/zsh` 가 기본 셸을 바꾸는 명령이라는 점 출제.

### 1-2. `/etc/login.defs` — UID 범위·비밀번호 기본 정책

> **상황**: UID 가 어디서부터 일반 사용자인지, 비밀번호 만료·최소 길이 기본값, 홈 디렉터리 권한을 확인한다. RHEL 9 는 `HOME_MODE 0700` 이 추가되어 홈이 700 으로 만들어진다.

```bash
grep -E '^(UID_MIN|UID_MAX|SYS_UID_MIN|SYS_UID_MAX|GID_MIN|PASS_MAX_DAYS|PASS_MIN_DAYS|PASS_MIN_LEN|PASS_WARN_AGE|UMASK|HOME_MODE|CREATE_HOME|USERGROUPS_ENAB|ENCRYPT_METHOD)' /etc/login.defs
```

- `UID_MIN`/`UID_MAX` : `useradd` 가 자동 할당하는 일반 사용자 UID 범위 (1000~60000)
- `SYS_UID_MIN`/`SYS_UID_MAX` : `-r` 시스템 계정 UID 범위 (201~999)
- `PASS_MAX_DAYS` : 비밀번호 최대 사용일 (기본 99999 = 무제한) → `/etc/shadow` 5번 필드 초기값
- `PASS_MIN_DAYS` : 변경 후 재변경까지 최소 일수 → 4번 필드
- `PASS_MIN_LEN` : 최소 길이 (PAM `pwquality` 가 실제로는 우선)
- `PASS_WARN_AGE` : 만료 전 경고일 → 6번 필드
- `UMASK` : 홈 디렉터리 생성 시 사용하던 마스크 (RHEL 9 는 `HOME_MODE` 가 있으면 그것을 우선)
- `HOME_MODE` : 홈 디렉터리 권한 (RHEL 9 기본 `0700`)
- `CREATE_HOME` : `-m` 생략 시에도 홈 생성 여부 (`yes`)
- `USERGROUPS_ENAB` : 사용자명과 같은 개인 그룹(UPG) 자동 생성 (`yes`)
- `ENCRYPT_METHOD` : 비밀번호 해시 알고리즘 (`SHA512` → shadow 의 `$6$`)

**검증**

```bash
grep -E '^(UID_MIN|PASS_MAX_DAYS|HOME_MODE|ENCRYPT_METHOD)' /etc/login.defs
```

```text
PASS_MAX_DAYS   99999
UID_MIN                  1000
HOME_MODE       0700
ENCRYPT_METHOD SHA512
```

> 📝 **시험 포인트**: `UID_MIN 1000` 의 의미, `PASS_WARN_AGE 7` 의 의미, `$6$` = SHA-512 가 단골. `login.defs` 는 **전역 기본값**, `/etc/default/useradd` 는 **useradd 전용 기본값**으로 구분.

### 1-3. `/etc/skel` — 팀 공통 alias 전파

> **상황**: 신규 사용자 모두에게 `ll` alias 를 기본으로 주기 위해 템플릿 `.bashrc` 를 수정한다. 이후 생성되는 dev1 홈에 복사되는지 3-1 에서 확인한다.

```bash
ls -la /etc/skel                          # 템플릿 파일 목록
cat >> /etc/skel/.bashrc <<'EOF'

# lab.local 팀 공통 alias
alias ll='ls -alF'
alias grep='grep --color=auto'
EOF
```

- `ls -la` : 숨김 파일(`.bashrc` 등) 포함 목록 (**a**ll, **l**ong)
- `/etc/skel/.bash_profile` : 로그인 셸 시작 시 1회 실행, 내부에서 `.bashrc` 호출
- `/etc/skel/.bashrc` : 대화형 셸마다 실행 — alias·함수 위치
- `/etc/skel/.bash_logout` : 로그아웃 시 실행

**검증**

```bash
ls -la /etc/skel
tail -3 /etc/skel/.bashrc
```

```text
total ...
drwxr-xr-x.  2 root root  62 ... .
drwxr-xr-x. ... root root ... ..
-rw-r--r--.  1 root root  18 ... .bash_logout
-rw-r--r--.  1 root root 141 ... .bash_profile
-rw-r--r--.  1 root root ... ... .bashrc
# lab.local 팀 공통 alias
alias ll='ls -alF'
alias grep='grep --color=auto'
```

> 📝 **시험 포인트**: `/etc/skel` = 새 홈으로 **복사되는 템플릿**. 이미 존재하는 사용자에게는 영향 없음. `useradd -k <dir>` 로 다른 skel 지정 가능.

---

## 2. 그룹 관리

### 2-1. 그룹 생성 — `groupadd -g`

> **상황**: README 2-1 자원 명세대로 개발팀 GID 2000, 운영팀 GID 2001 을 먼저 만든다. 사용자보다 그룹을 먼저 만들어야 `useradd -g devteam` 이 가능하다.

```bash
groupadd -g 2000 devteam
groupadd -g 2001 opsteam
```

- `-g <GID>` : GID 명시 (**g**id). 생략 시 `GID_MIN` 이상 미사용 값 자동 할당
- `-r` : 시스템 그룹 (`SYS_GID_MIN`~`SYS_GID_MAX` 범위) — 여기서는 미사용
- `-o` : 중복 GID 허용 (**o**verride, non-unique)

**검증**

```bash
getent group devteam opsteam
grep -E 'devteam|opsteam' /etc/group /etc/gshadow
```

```text
devteam:x:2000:
opsteam:x:2001:
/etc/group:devteam:x:2000:
/etc/group:opsteam:x:2001:
/etc/gshadow:devteam:!::
/etc/gshadow:opsteam:!::
```

> 📝 **시험 포인트**: `groupadd -g 2000 dev` 해석 문제 빈출. `/etc/group` 4필드 `그룹명:x:GID:보조구성원`, `/etc/gshadow` 4필드 `그룹명:암호:관리자:구성원` — 3번째가 **그룹 관리자**.

### 2-2. `/etc/group`·`/etc/gshadow` 필드와 `getent`

> **상황**: 그룹 파일 두 개의 필드 구조를 표로 정리하고, 파일을 직접 읽는 대신 NSS 를 거치는 `getent` 로 조회하는 습관을 만든다.

```bash
getent group wheel                # NSS(로컬+LDAP 등) 통합 조회
awk -F: '$3>=1000 {print $1, $3, $4}' /etc/group
```

- `getent group <이름|GID>` : `/etc/nsswitch.conf` 의 `group:` 소스(files, sss …)를 순서대로 조회 (**get** **ent**ry)
- `awk -F:` : 콜론 구분자 (**F**ield separator)

| 파일 | # | 필드 | 값 예 | 의미 |
| --- | --- | --- | --- | --- |
| `/etc/group` | 1 | 그룹명 | `wheel` | 그룹 이름 |
| | 2 | 비밀번호 | `x` | 실제 값은 `/etc/gshadow` |
| | 3 | GID | `10` | 그룹 ID |
| | 4 | 구성원 | `admin1` | 이 그룹을 **보조 그룹**으로 갖는 사용자 (1차 그룹 구성원은 여기 없음) |
| `/etc/gshadow` | 1 | 그룹명 | `wheel` | |
| | 2 | 암호 | `!` | `!` = 그룹 비밀번호 없음 (`newgrp` 시 비밀번호 요구 불가) |
| | 3 | 관리자 | `dev1` | `gpasswd -A` 로 지정, 그룹 구성원 추가·삭제 가능 |
| | 4 | 구성원 | `admin1` | `/etc/group` 4번과 동일 |

**검증**

```bash
ls -l /etc/group /etc/gshadow
```

```text
-rw-r--r--. 1 root root ... /etc/group
----------. 1 root root ... /etc/gshadow
```

> 📝 **시험 포인트**: 1차 그룹은 `/etc/passwd` 4번 필드, 보조 그룹은 `/etc/group` 4번 필드 — "kim,lee 의 의미" 문제. `/etc/gshadow` 는 `000` 권한(root 만 CAP_DAC_OVERRIDE 로 읽음).

### 2-3. `groupmod -n` 이름 변경·`groupdel` 실패 실습

> **상황**: 임시 그룹을 만들어 이름 변경·삭제를 연습하고, 사용 중인 1차 그룹은 삭제되지 않음을 확인한다 (3절에서 dev1 생성 후 다시 시도).

```bash
groupadd tmpgrp
groupmod -n labtmp tmpgrp            # tmpgrp → labtmp
groupmod -g 2999 labtmp              # GID 변경
groupdel labtmp
```

- `groupmod -n <새이름> <기존이름>` : 그룹 **이름** 변경 (**n**ew name). 인자 순서 주의 — 새 이름이 먼저
- `groupmod -g <GID>` : GID 변경. 기존 파일의 GID 는 자동 변경되지 않음
- `groupdel <그룹>` : 삭제. 누군가의 **1차 그룹**이면 거부

**검증**

```bash
getent group labtmp || echo "labtmp removed"
# (3-1 이후) 사용 중 그룹 삭제 시도
groupdel devteam
```

```text
labtmp removed
groupdel: cannot remove the primary group of user 'dev1'
```

> 📝 **시험 포인트**: `groupmod -n devteam backend` = backend 를 devteam 으로 **변경**(새이름 먼저). `groupdel` 은 1차 그룹 사용자 존재 시 불가, 보조 그룹은 삭제 가능.

### 2-4. 구성원 관리 — `gpasswd -a/-d/-A/-M`, `groups`

> **상황**: 3절에서 사용자를 만든 뒤 수행. dev1 을 devteam 관리자로 지정하고, 관리자가 root 없이 구성원을 넣고 빼는지 확인한다. (3-1~3-4 완료 후 실행)

```bash
gpasswd -A dev1 devteam               # dev1 을 devteam 그룹 관리자로
gpasswd -a guest1 devteam             # guest1 을 보조 구성원으로 추가
gpasswd -d guest1 devteam             # 제거
gpasswd -M dev1,dev2 devteam          # 구성원 목록을 통째로 지정 (기존 목록 대체)
groups dev2                           # dev2 소속 그룹
su - dev1 -c 'gpasswd -a guest1 devteam'   # 관리자 권한으로 추가 가능
gpasswd -d guest1 devteam
```

- `-A <user,...>` : 그룹 **관리자** 지정 (**A**dministrators) → `/etc/gshadow` 3번 필드
- `-a <user>` : 구성원 추가 (**a**dd)
- `-d <user>` : 구성원 삭제 (**d**elete)
- `-M <user,...>` : 구성원 목록 일괄 설정 (**M**embers) — 기존 목록 **대체**
- `-r` : 그룹 비밀번호 제거 (**r**emove) — 참고
- `groups [user]` : 사용자의 1차+보조 그룹 출력

**검증**

```bash
grep devteam /etc/group /etc/gshadow
id guest1
```

```text
/etc/group:devteam:x:2000:dev1,dev2
/etc/gshadow:devteam:!:dev1:dev1,dev2
uid=2004(guest1) gid=2004(guest1) groups=2004(guest1)
```

> 📝 **시험 포인트**: `gpasswd -A user1 devteam` = 그룹 관리자 지정, `-a` 와 `-A` 구분. `gpasswd -M` 은 `usermod -G` 처럼 **대체** 동작.

---

## 3. 사용자 생성 — `useradd`

### 3-1. dev1 — 옵션 전부 명시

> **상황**: 개발자 1호 계정. UID 2001, 1차 그룹 devteam, 설명·셸·홈 생성까지 명시해 각 옵션이 `/etc/passwd` 어느 필드로 가는지 눈으로 맞춘다.

```bash
useradd -u 2001 -g devteam -c "Developer 1" -s /bin/bash -m dev1
```

- `-u <UID>` : UID 지정 (**u**id)
- `-g <그룹>` : 1차(기본) 그룹 (**g**roup) → `/etc/passwd` 4번 필드. 지정 시 개인 그룹(UPG) 생성 안 함
- `-c "<설명>"` : GECOS 주석 (**c**omment) → 5번 필드
- `-s <셸>` : 로그인 셸 (**s**hell) → 7번 필드
- `-m` : 홈 디렉터리 생성 + `/etc/skel` 복사 (**m**ake home). RHEL 은 `CREATE_HOME yes` 라 생략해도 생성되나 명시 권장
- `-d <경로>` : 홈 경로 지정 (**d**irectory), 기본 `/home/<user>`
- `-k <skel>` : 다른 템플릿 디렉터리 (**k**eleton) — `-m` 과 함께
- `-M` : 홈을 만들지 **않음** (`-m` 반대, 서비스 계정에 사용)
- `-N` : 개인 그룹(UPG) 생성 안 함 (**N**o user group) — `-g` 없이 `GROUP=100`(users) 사용
- `-o` : 중복 UID 허용 (**o**verride) — `-u` 와 함께, 보안상 지양
- `-r` : 시스템 계정 (**r**eserved UID 201~999, 홈·만료 없음)

**검증**

```bash
id dev1
getent passwd dev1
grep dev1 /etc/passwd /etc/shadow /etc/group
ls -ld /home/dev1
ls -la /home/dev1
tail -3 /home/dev1/.bashrc              # 1-3 의 skel alias 전파 확인
```

```text
uid=2001(dev1) gid=2000(devteam) groups=2000(devteam)
dev1:x:2001:2000:Developer 1:/home/dev1:/bin/bash
/etc/passwd:dev1:x:2001:2000:Developer 1:/home/dev1:/bin/bash
/etc/shadow:dev1:!!:20699:0:99999:7:::
drwx------. 3 dev1 devteam 78 ... /home/dev1
...
-rw-r--r--. 1 dev1 devteam  18 ... .bash_logout
-rw-r--r--. 1 dev1 devteam 141 ... .bash_profile
-rw-r--r--. 1 dev1 devteam ... ... .bashrc
# lab.local 팀 공통 alias
alias ll='ls -alF'
alias grep='grep --color=auto'
```

- `/etc/group` 에 dev1 이 없음 = 1차 그룹 구성원은 `/etc/group` 4번 필드에 기록되지 않음
- 홈 `700` = RHEL 9 `HOME_MODE 0700` (RHEL 8 이하 755 와 차이)
- shadow 2번 필드 `!!` = 비밀번호 **미설정**(로그인 불가), 3번 `20699` = 오늘(2026-09-03)의 1970 기준 일수

> 📝 **시험 포인트**: `useradd -s` 가 로그인 셸 옵션, `-g` 는 1차·`-G` 는 보조, `-m` 은 홈 생성. `-M -s /sbin/nologin` 조합 = 서비스용 계정.

### 3-2. dev2·ops1 — 보조 그룹 `-G`

> **상황**: dev2 는 개발팀이면서 운영 업무를 겸해 opsteam 보조, ops1 은 운영팀이면서 wheel(관리자) 보조. 이 wheel 소속이 5-4(pam_wheel)와 7절(sudo)의 전제가 된다.

```bash
useradd -u 2002 -g devteam -G opsteam -c "Developer 2" -m dev2
useradd -u 2003 -g opsteam -G wheel   -c "Operator 1"  -m ops1
```

- `-G <그룹,...>` : 보조 그룹 목록 (**G**roups) → `/etc/group` 4번 필드에 사용자명 추가

**검증**

```bash
id dev2; id ops1
getent group opsteam wheel
```

```text
uid=2002(dev2) gid=2000(devteam) groups=2000(devteam),2001(opsteam)
uid=2003(ops1) gid=2001(opsteam) groups=2001(opsteam),10(wheel)
opsteam:x:2001:dev2
wheel:x:10:admin1,ops1
```

> 📝 **시험 포인트**: `useradd -m -s /bin/bash -e 2026-12-31 -G wheel kim` 같은 복합 명령 해석. `id` 출력의 `gid=` 가 1차, `groups=` 가 전체.

### 3-3. guest1 — 만료일 `-e`·비활성 `-f`

> **상황**: 외부 협력사 임시 계정. 연말 만료·만료 후 7일 유예를 생성 시점에 박아 두고, 6-5·6-6 에서 잠금 → 만료 → 삭제까지 수명주기를 돈다.

```bash
useradd -u 2004 -e 2026-12-31 -f 7 -c "Temp Guest" -m guest1
```

- `-e <YYYY-MM-DD>` : 계정 만료일 (**e**xpire) → `/etc/shadow` 8번 필드 (1970 기준 일수로 저장)
- `-f <일>` : 비밀번호 만료 후 계정 비활성까지 유예일 (ina**f**tive) → 7번 필드. `-1` = 미사용
- `-g` 생략 → `USERGROUPS_ENAB yes` 에 따라 개인 그룹 `guest1` 자동 생성 (GID 는 UID 와 같은 2004 가 비어 있으면 2004)

**검증**

```bash
id guest1
grep guest1 /etc/shadow
chage -l guest1 | grep -E 'Account expires|inactive'
date -d @$((20818*86400)) +%F          # 8번 필드 일수 → 날짜 환산
```

```text
uid=2004(guest1) gid=2004(guest1) groups=2004(guest1)
guest1:!!:20699:0:99999:7:7:20818:
Password inactive                                       : never
Account expires                                         : Dec 31, 2026
2026-12-31
```

> 📝 **시험 포인트**: `-e` 는 **계정** 만료, `-f` 는 **비밀번호 만료 후** 비활성 유예 — 둘의 shadow 필드 위치(8, 7) 구분.

### 3-4. `-r` 시스템 계정·`-M`·`-N`·`-o` 확인

> **상황**: 서비스 데몬용 계정은 어떻게 다른지 비교해 본다. 실습용으로 `labsvc` 를 만들어 UID 범위·홈·셸 차이를 보고 바로 삭제한다.

```bash
useradd -r -M -s /sbin/nologin -c "Lab service" labsvc
getent passwd labsvc
grep labsvc /etc/shadow
userdel labsvc
```

- `-r` : 시스템 계정 — UID 를 `SYS_UID_MIN`~`SYS_UID_MAX`(201~999) 에서 할당, 홈 미생성, 비밀번호 만료 정책 미적용
- `-M` : 홈 미생성 (**M** = no home)
- `-s /sbin/nologin` : 대화식 로그인 차단 셸 (6-4 참고)

**검증**

```bash
getent passwd labsvc || echo "labsvc removed"
awk -F: '$3>=1000 && $3<65534 {print $1":"$3":"$4":"$7}' /etc/passwd
```

```text
labsvc:x:9..:9..:Lab service:/home/labsvc:/sbin/nologin      # (userdel 전) UID 999 이하, 홈은 경로만 기록
labsvc:!!:20699::::::                                        # (userdel 전) 만료 필드 전부 빈값
labsvc removed
admin1:1000:1000:/bin/bash
dev1:2001:2000:/bin/bash
dev2:2002:2000:/bin/bash
ops1:2003:2001:/bin/bash
guest1:2004:2004:/bin/bash
```

> 📝 **시험 포인트**: `useradd -r` = 시스템 계정(UID < 1000, 홈 없음). `awk -F: '$3>=1000 {print $1}' /etc/passwd` 는 실기 단골(UID 1000 이상 사용자명 출력).

### 3-5. `/etc/passwd` 7필드·`/etc/shadow` 9필드 — 날짜 환산 실습

> **상황**: 필기·실기 최빈출. dev1 의 실제 줄을 놓고 필드마다 의미를 대응시키고, shadow 의 일수 필드를 `date` 로 환산해 본다.

```bash
getent passwd dev1 | tr ':' '\n' | nl          # 7필드 번호 매기기
grep '^dev1' /etc/shadow | tr ':' '\n' | nl    # 9필드
date -d @$((18900*86400)) +%F                  # 기출값 18900 → 날짜
date -d @$((18800*86400)) +%F
echo $(( $(date +%s) / 86400 ))                # 오늘의 일수
```

- `tr ':' '\n'` : 콜론을 줄바꿈으로 치환해 필드를 세로로
- `nl` : 줄 번호 부여 (**n**umber **l**ines)
- `date -d @<초>` : epoch 초를 날짜로 → 일수 × 86400

| # | `/etc/passwd` 필드 | dev1 값 | 의미 |
| --- | --- | --- | --- |
| 1 | 로그인명 | `dev1` | 계정 이름 |
| 2 | 비밀번호 | `x` | shadow 사용 표시 |
| 3 | UID | `2001` | 사용자 ID |
| 4 | GID | `2000` | **1차 그룹** ID |
| 5 | GECOS | `Developer 1` | 설명(실명·연락처 등, `chfn`) |
| 6 | 홈 | `/home/dev1` | 로그인 후 위치 |
| 7 | 셸 | `/bin/bash` | 로그인 셸 (`/sbin/nologin` = 차단) |

| # | `/etc/shadow` 필드 | dev1 값 (초기) | 의미 |
| --- | --- | --- | --- |
| 1 | 로그인명 | `dev1` | |
| 2 | 암호 해시 | `!!` | `$6$salt$hash` = SHA-512. `!!` = 미설정, `!`/`!!` 접두 = 잠금, `*` = 비밀번호 로그인 불가(시스템 계정) |
| 3 | 최종 변경일 | `20699` | 1970-01-01 기준 일수. `0` = 다음 로그인 시 변경 강제 |
| 4 | 최소 사용일 | `0` | 이 일수 전에는 재변경 불가 (`chage -m`) |
| 5 | 최대 사용일 | `99999` | 이 일수 후 만료 (`chage -M`) |
| 6 | 경고일 | `7` | 만료 전 경고 시작 (`chage -W`) |
| 7 | 비활성 유예일 | 빈값 | 만료 후 이 일수 지나면 계정 잠금 (`chage -I`) |
| 8 | 계정 만료일 | 빈값 | 1970 기준 일수 (`chage -E`) |
| 9 | 예약 | 빈값 | 미사용 |

**검증**

```bash
date -d @$((18900*86400)) +%F; date -d @$((18800*86400)) +%F
```

```text
2021-09-30
2021-06-22
```

> 📝 **시험 포인트**: 실기 "shadow 3~5번 필드 `18900`,`0`,`90` 의 의미" — 최종 변경일(일수)·최소·최대 사용일. `18800`,`7`,`90`,`14` = 변경일·최소·최대·경고. 잠금 접두 `!` 와 미설정 `!!`, `*` 구분.

---

## 4. 비밀번호 관리

### 4-1. `passwd` 대화식 설정과 `passwd -S`

> **상황**: 3절에서 만든 계정은 shadow 2번 필드가 `!!` 라 로그인이 불가능하다. 팀원 4명에게 비밀번호를 부여하고 상태를 한 줄 요약으로 확인한다.

```bash
passwd dev1                     # 대화식 — 입력은 화면에 표시되지 않음
passwd dev2
passwd ops1
passwd guest1
passwd -S dev1                  # 한 줄 상태 요약
passwd -Sa | awk '$2!="LK"' | head
```

- `passwd <user>` : root 는 기존 값 확인 없이 변경. 일반 사용자는 자기 계정만·현재 비밀번호 필요
- `-S` : 상태 한 줄 요약 (**S**tatus) — `계정 상태 최종변경일 최소 최대 경고 비활성`
- `-a` : 전 계정 대상 (**a**ll) — `-S` 와 함께만 사용
- 상태 코드 : `PS` = 비밀번호 설정됨(**P**assword **S**et) · `LK` = 잠김(**L**oc**K**ed) · `NP` = 비밀번호 없음(**N**o **P**assword)
- `--stdin` : 표준입력에서 수신 (RHEL 계열 전용) — 명령 이력에 평문이 남아 이 절차서에서는 미사용

**검증**

```bash
passwd -S dev1
grep '^dev1:' /etc/shadow | cut -c1-22
```

```text
dev1 PS 2026-09-03 0 99999 7 -1 (Password set, SHA512 crypt.)
dev1:$6$...
```

> 📝 **시험 포인트**: `$6$` = SHA-512 (필기 R04-59 · R09-25 · R10-61). `$1$` MD5 · `$5$` SHA-256 · `$2b$` bcrypt. `$6$` 뒤 문자열은 **솔트** — 같은 비밀번호라도 해시가 달라지게 함.

### 4-2. 잠금·해제·비밀번호 삭제 — `passwd -l` `-u` `-d`

> **상황**: 임시 계정 guest1 을 잠시 정지시켰다가 되살린다. 이 과정에서 "잠금"이 막는 것이 무엇이고 막지 못하는 것이 무엇인지 확인한다.

```bash
passwd -l guest1                        # 잠금
passwd -S guest1
grep '^guest1:' /etc/shadow | cut -d: -f2 | cut -c1-3
su - guest1 -c 'id -un'                 # root 의 su 는 비밀번호를 묻지 않으므로 성공
passwd -u guest1                        # 해제
passwd -S guest1
```

⚠️ 아래 `-d` 는 **비밀번호 없는 계정**을 만든다. 확인 즉시 재설정할 것.

```bash
passwd -d guest1
passwd -S guest1
grep '^guest1:' /etc/shadow | cut -d: -f1,2
passwd guest1                           # 곧바로 재설정
```

- `-l` : 잠금 (**l**ock) — 해시 앞에 `!` 접두 부착(RHEL `passwd` 는 `!!`), 해시 자체는 보존되어 해제 시 그대로 복구
- `-u` : 해제 (**u**nlock) — 접두 제거. 접두를 떼면 빈 비밀번호가 되는 경우 거부됨
- `-d` : 비밀번호 삭제 (**d**elete) — 2번 필드를 빈값으로 → `NP`
- `usermod -L`/`-U` 도 같은 효과이나 접두가 `!` 1개 (4-6 비교)

**검증**

```bash
passwd -S guest1
```

```text
guest1 LK 2026-09-03 0 99999 7 7 (Password locked.)
!!$
uid=2004(guest1) gid=2004(guest1) groups=2004(guest1)
guest1 PS 2026-09-03 0 99999 7 7 (Password set, SHA512 crypt.)
guest1 NP 2026-09-03 0 99999 7 7 (Empty password.)
guest1:
```

> 📝 **시험 포인트**: 잠금은 **비밀번호 인증만** 차단 — SSH 공개키 로그인·root 의 `su` 는 그대로 통과. 완전 차단은 `chage -E 0` + `usermod -s /sbin/nologin` 병행 (필기 R03-23 · R09-26 · 실기 R05-02).

### 4-3. 다음 로그인 시 변경 강제 — `passwd -e` / `chage -d 0`

> **상황**: 관리자가 임시 비밀번호를 발급했으므로 dev1 이 처음 로그인할 때 반드시 자기 비밀번호로 바꾸게 만든다.

```bash
chage -l dev1 | head -1
chage -d 0 dev1                     # 최종 변경일을 0 으로 → 즉시 만료 상태
chage -l dev1 | head -1
grep '^dev1:' /etc/shadow | cut -d: -f3
su - dev1                           # 실제 강제 변경 확인
```

- `chage -d 0 <user>` : shadow 3번 필드(최종 변경일)를 `0` 으로 → "한 번도 바꾼 적 없음" 취급
- `passwd -e <user>` : 동일 효과 (**e**xpire)
- 강제 변경은 **비밀번호 만료**이지 계정 만료가 아님 — 계정 만료는 `chage -E`

**검증**

```bash
chage -l dev1 | head -1
```

```text
Last password change					: password must be changed
0
You are required to change your password immediately (administrator enforced)
Changing password for user dev1.
New password:
Retype new password:
...
```

- root 가 `su` 로 진입하면 현재 비밀번호를 묻지 않고 새 비밀번호만 요구. 실제 SSH·콘솔 로그인에서는 `Current password:` 가 먼저 나옴
- 실습을 이어가려면 원복 : `chage -d $(date +%F) dev1`

> 📝 **시험 포인트**: shadow 3번 필드 `0` = "다음 로그인 시 변경 강제". 필드값이 `18900` 같은 숫자면 1970-01-01 기준 경과 일수 (실기 R02-11 · R05-11).

### 4-4. `passwd -n/-x/-w/-i` — shadow 4~7번 필드 직접 설정

> **상황**: 회사 규정(최소 7일·최대 90일·경고 14일·만료 후 30일 비활성)을 dev2 에 `passwd` 로 먼저 적용해 본다. 같은 값을 `chage` 로도 줄 수 있음을 4-5 에서 비교한다.

```bash
passwd -n 7 -x 90 -w 14 -i 30 dev2
passwd -S dev2
grep '^dev2:' /etc/shadow | awk -F: '{print $4, $5, $6, $7}'
```

- `-n <일>` : 최소 사용일 (mi**n**imum) → 4번 필드. 변경 후 이 기간에는 재변경 불가
- `-x <일>` : 최대 사용일 (ma**x**imum) → 5번 필드
- `-w <일>` : 만료 전 경고 시작일 (**w**arning) → 6번 필드
- `-i <일>` : 비밀번호 만료 후 계정 비활성까지 유예 (**i**nactive) → 7번 필드

**검증**

```bash
grep '^dev2:' /etc/shadow | awk -F: '{print "min="$4, "max="$5, "warn="$6, "inact="$7}'
```

```text
dev2 PS 2026-09-03 7 90 14 30 (Password set, SHA512 crypt.)
min=7 max=90 warn=14 inact=30
```

> 📝 **시험 포인트**: `passwd -n/-x/-w/-i` 와 `chage -m/-M/-W/-I` 의 대응 관계가 혼동 포인트. **소문자 `x` = 최대**, `chage` 는 **대문자 `M` = 최대**.

### 4-5. `chage` — 조회와 정책 일괄 적용

> **상황**: 같은 규정을 dev1 에 `chage` 한 줄로 적용하고, 계정 만료일까지 함께 넣는다. `chage -l` 출력은 필기·실기 해석 문제로 그대로 나온다.

```bash
chage -M 90 -m 7 -W 14 -I 30 -E 2026-12-31 dev1
chage -l dev1
chage -l dev1 | grep -E 'Maximum|Account expires'
```

- `-l` : 조회 (**l**ist) — 일반 사용자도 **자기 계정**에는 사용 가능
- `-M <일>` : 최대 사용일 (**M**axdays) → 5번 필드
- `-m <일>` : 최소 사용일 (**m**indays) → 4번 필드
- `-W <일>` : 경고일 (**W**arndays) → 6번 필드
- `-I <일>` : 비활성 유예일 (**I**nactive) → 7번 필드
- `-E <YYYY-MM-DD|일수|-1>` : **계정** 만료일 (**E**xpiredate) → 8번 필드. `-1` = 만료 없음
- `-d <YYYY-MM-DD|0>` : 최종 변경일 (**d**ate) → 3번 필드
- 옵션 없이 `chage dev1` : 항목별 대화식 질의

**검증**

```bash
chage -l dev1
```

```text
Last password change					: Sep 03, 2026
Password expires					: Dec 02, 2026
Password inactive					: Jan 01, 2027
Account expires						: Dec 31, 2026
Minimum number of days between password change		: 7
Maximum number of days between password change		: 90
Number of days of warning before password expires	: 14
```

> 📝 **시험 포인트**: `chage -l` 출력에서 `Password expires`(비밀번호 만료)와 `Account expires`(계정 만료)를 바꿔 묻는 함정이 반복 출제 (필기 R02-45 · R08-23 · R09-22). `chage -M 90 user1` 은 실기 단답 고정 문항 (R02-07 · R05-07).

### 4-6. `usermod -L/-U/-e/-f` 와의 관계

> **상황**: 같은 일을 하는 명령이 여럿이라 시험에서 서로 바꿔 출제된다. dev2 로 `usermod` 계열을 한 번씩 눌러 shadow 필드가 어떻게 바뀌는지 대응시킨다.

```bash
usermod -L dev2 ; passwd -S dev2 ; grep '^dev2:' /etc/shadow | cut -d: -f2 | cut -c1-2
usermod -U dev2 ; passwd -S dev2
usermod -e 2026-12-31 dev2               # = chage -E
usermod -f 30 dev2                       # = chage -I
chage -l dev2 | grep -E 'Account expires|Password inactive'
usermod -e '' dev2 ; usermod -f -1 dev2  # 원복
chage -l dev2 | grep -E 'Account expires|Password inactive'
```

- `-L` : 잠금 (**L**ock) — 해시 앞 `!` **1개**
- `-U` : 해제 (**U**nlock)
- `-e <YYYY-MM-DD>` : 계정 만료일. `-e ''` (빈 문자열) = 만료 해제
- `-f <일>` : 비밀번호 만료 후 비활성 유예. `-1` = 미사용

| 목적 | `passwd` | `chage` | `usermod` | shadow 필드 |
| --- | --- | --- | --- | --- |
| 잠금 / 해제 | `-l` / `-u` | — | `-L` / `-U` | 2 (`!` 접두) |
| 즉시 변경 강제 | `-e` | `-d 0` | — | 3 |
| 최소 사용일 | `-n` | `-m` | — | 4 |
| 최대 사용일 | `-x` | `-M` | — | 5 |
| 경고일 | `-w` | `-W` | — | 6 |
| 비활성 유예 | `-i` | `-I` | `-f` | 7 |
| 계정 만료일 | — | `-E` | `-e` | 8 |

**검증**

```bash
chage -l dev2 | grep -E 'Account expires|Password inactive'
```

```text
dev2 LK 2026-09-03 7 90 14 30 (Password locked.)
!$
dev2 PS 2026-09-03 7 90 14 30 (Password set, SHA512 crypt.)
Password inactive					: never
Account expires						: never
```

> 📝 **시험 포인트**: `usermod -L`(`!` 1개)과 RHEL `passwd -l`(`!!`)의 접두 개수 차이. `-f`(usermod)는 **비활성 유예**, `-e`(usermod)는 **계정 만료** — `useradd` 때와 같은 대응 (3-3 참조).

### 4-7. `pwconv` · `grpconv` — 섀도 동기화

> **상황**: shadow 방식이 이미 켜져 있는지, 파일 권한이 규정대로인지 확인한다. 수동으로 `/etc/passwd` 를 편집한 뒤 shadow 항목이 없을 때 복구하는 명령이기도 하다.

```bash
pwconv                       # /etc/passwd ↔ /etc/shadow 동기화 (이미 shadow 사용 중이면 무해)
grpconv                      # /etc/group ↔ /etc/gshadow
ls -l /etc/passwd /etc/shadow /etc/group /etc/gshadow
pwck -r                      # 읽기 전용 정합성 검사
grpck -r
```

- `pwconv` : `/etc/passwd` 의 비밀번호 필드를 `x` 로 바꾸고 해시를 `/etc/shadow` 로 이동. 누락된 shadow 항목 생성
- `grpconv` : `/etc/group` → `/etc/gshadow` 동일 작업
- `pwck` / `grpck` : 계정·그룹 파일 정합성 검사. `-r` = 읽기 전용(**r**ead-only), `-s` = 정렬
- ※ 미실행 — `pwunconv` / `grpunconv` : 섀도를 **해제**해 해시를 644 권한의 `/etc/passwd` 로 되돌림. 전 계정 해시가 모든 사용자에게 노출되므로 개념만 확인

**검증**

```bash
ls -l /etc/passwd /etc/shadow /etc/group /etc/gshadow
```

```text
-rw-r--r--. 1 root root ... /etc/passwd
-rw-r--r--. 1 root root ... /etc/group
----------. 1 root root ... /etc/shadow
----------. 1 root root ... /etc/gshadow
```

> 📝 **시험 포인트**: RHEL 9 의 `/etc/shadow`·`/etc/gshadow` 권한은 **`000`** — root 는 파일 권한을 무시(`CAP_DAC_OVERRIDE`)하므로 읽힌다. "shadow 는 600" 이라는 보기가 나오면 배포판·버전 전제를 확인.

### 4-8. `/etc/security/pwquality.conf` — 복잡도 강제

> **상황**: 회사 규정 "최소 10자, 숫자·대문자·소문자·특수문자 각 1개 이상, 이전 비밀번호와 5자 이상 상이"를 시스템에 강제하고, 약한 비밀번호가 실제로 거절되는지 일반 사용자로 확인한다.

```bash
cp -p /etc/security/pwquality.conf /root/pwquality.conf.orig
grep -vE '^\s*#|^\s*$' /etc/security/pwquality.conf     # 기본은 전부 주석 처리 상태
cat >> /etc/security/pwquality.conf <<'EOF'

# lab.local 비밀번호 규정
minlen = 10
dcredit = -1
ucredit = -1
lcredit = -1
ocredit = -1
difok = 5
maxrepeat = 3
EOF
grep -vE '^\s*#|^\s*$' /etc/security/pwquality.conf
grep -n pam_pwquality /etc/pam.d/system-auth
```

- `minlen` : 최소 길이 — credit 가산점을 반영한 **점수** 기준
- `dcredit`/`ucredit`/`lcredit`/`ocredit` : 숫자(**d**igit)·대문자(**u**pper)·소문자(**l**ower)·기타 특수문자(**o**ther). **양수 N** = 해당 종류 문자 최대 N개까지 길이 가산, **음수 -N** = 최소 N개 **필수**
- `difok` : 이전 비밀번호와 달라야 하는 문자 수
- `maxrepeat` : 같은 문자 연속 허용 횟수 (`aaaa` 차단)
- `minclass` : 필요한 문자 **종류** 수
- `usercheck`/`gecoscheck`/`dictcheck` : 계정명·GECOS·사전 단어 포함 검사
- `enforce_for_root` : root 에게도 강제 — **기본 미적용**이라 root 는 경고만 뜨고 통과
- `retry` : 실패 시 재입력 횟수 (PAM 줄의 인자로도 지정 가능)
- 모듈 호출 위치 : `/etc/pam.d/system-auth` 의 `password requisite pam_pwquality.so ...`

**검증** — 실제 거절은 일반 사용자로 확인해야 함

```bash
su - dev1
```

```text
$ passwd
Changing password for user dev1.
Current password:
New password:                                   ← abc123 입력
BAD PASSWORD: The password is shorter than 10 characters
New password:                                   ← password 입력
BAD PASSWORD: The password contains less than 1 digit
New password:                                   ← Lab#Rocky9$srv01 입력
Retype new password:
passwd: all authentication tokens updated successfully.
```

> 📝 **시험 포인트**: `minlen=10 dcredit=-1` = 최소 10자·숫자 1개 이상 (필기 R09-24). 음수 = 필수 개수, 양수 = 길이 가산점. `pam_pwquality` 는 구형 `pam_cracklib` 의 대체 모듈이며 **password** 타입 `requisite` 로 호출.

### 4-9. `pwscore` — 비밀번호 강도 점수

> **상황**: 정책에 걸리는지만이 아니라 얼마나 강한지 수치로 확인해 팀에 안내한다.

```bash
rpm -qf $(command -v pwscore)
echo 'abc123' | pwscore
echo 'Lab#Rocky9$srv01' | pwscore
pwmake 128
```

- `pwscore` : 표준입력의 비밀번호를 `pwquality.conf` 기준으로 채점 (0~100). 규칙 위반 시 점수 대신 사유 출력, 종료코드 ≠ 0
- `pwmake <비트수>` : 지정한 엔트로피의 무작위 비밀번호 생성 (예 `pwmake 128`)
- 두 명령 모두 `libpwquality` 패키지 제공

**검증**

```bash
echo 'abc123' | pwscore ; echo "rc=$?"
echo 'Lab#Rocky9$srv01' | pwscore ; echo "rc=$?"
```

```text
libpwquality-...
Password quality check failed:
 The password is shorter than 10 characters
rc=1
100
rc=0
...
```

> 📝 **시험 포인트**: `pwscore`·`pwmake` 자체는 출제 빈도가 낮으나, "복잡도 정책 설정 파일"을 묻는 문제에서 `/etc/security/pwquality.conf` 를 고르게 하는 근거로 함께 등장.

---

## 5. PAM — 인증 모듈 스택

### 5-1. `/etc/pam.d/` 구조와 설정 줄 4필드

> **상황**: 4-8 에서 고친 복잡도 정책이 어디를 거쳐 적용되는지 확인하려면 PAM 구조부터 봐야 한다. 서비스별 설정 파일과 모듈 실체의 위치를 잡는다.

```bash
ls /etc/pam.d | head -20
ls -l /etc/pam.d/system-auth /etc/pam.d/password-auth
grep -vE '^\s*#|^\s*$' /etc/pam.d/su
ls /usr/lib64/security | head -8
cat /etc/pam.d/other
```

- 설정 줄 형식 : `<타입> <컨트롤> <모듈경로> [모듈인자]` — 공백 구분 4필드
- 서비스명 = **파일명** — PAM 을 사용하는 프로그램이 자기 이름의 파일을 찾음 (`sshd`, `su`, `su-l`, `login`, `passwd`, `sudo`, `crond`)
- `/etc/pam.d/other` : 전용 파일이 없는 서비스의 기본값 — RHEL 은 전부 `pam_deny.so`(거부)
- 모듈 실체 : `/usr/lib64/security/pam_*.so` (32비트 환경은 `/lib/security`)
- 구형 단일 파일 `/etc/pam.conf` 는 `/etc/pam.d/` 가 있으면 무시됨 — 필드가 5개(서비스명 추가)

**검증**

```bash
grep -vE '^\s*#|^\s*$' /etc/pam.d/su
```

```text
auth		sufficient	pam_rootok.so
auth		substack	system-auth
auth		include		postlogin
account		sufficient	pam_succeed_if.so uid = 0 use_uid quiet
account		include		system-auth
password	include		system-auth
session		include		system-auth
session		include		postlogin
session		optional	pam_xauth.so
```

> 📝 **시험 포인트**: "PAM 설정 파일 위치" = `/etc/pam.d/` (필기 R01-59 · R07-56 · R08-60). 모듈은 `/usr/lib64/security/`. 첫 필드 `auth` 는 **인증 단계**를 뜻하며 모듈 경로가 아님 (필기 R05-58).

### 5-2. 4 타입 · 4 컨트롤과 스택 동작 순서

> **상황**: 실기 서술형(R04-15)과 필기 다수 문항이 컨트롤 플래그의 동작 차이를 묻는다. 표로 굳히고 실제 파일에서 각 플래그를 찾아 대응시킨다.

```bash
awk '$1=="auth"'     /etc/pam.d/system-auth
awk '$1=="account"'  /etc/pam.d/system-auth
awk '$1=="password"' /etc/pam.d/system-auth
awk '$1=="session"'  /etc/pam.d/system-auth
```

| 타입 | 담당 | 대표 모듈 |
| --- | --- | --- |
| `auth` | 신원 확인 — 비밀번호·키·토큰 검증, 자격증명 부여 | `pam_unix` `pam_wheel` `pam_rootok` `pam_faillock` `pam_sss` |
| `account` | 계정 **유효성** — 만료·시간대·접속 위치 제한 | `pam_unix` `pam_time` `pam_access` `pam_succeed_if` `pam_nologin` |
| `password` | 비밀번호 **변경** 규칙 | `pam_pwquality` `pam_unix` `pam_pwhistory` |
| `session` | 로그인 전후 환경 구성·정리 | `pam_limits` `pam_systemd` `pam_mkhomedir` `pam_lastlog` `pam_env` |

| 컨트롤 | 모듈 실패 시 | 모듈 성공 시 | 요약 |
| --- | --- | --- | --- |
| `required` | 실패를 기록하고 **스택 끝까지 계속** 실행 → 최종 실패 | 계속 진행 | 필수, 실패 지점을 감춤 |
| `requisite` | **즉시 중단**하고 실패 반환 | 계속 진행 | 필수, 즉시 거부 |
| `sufficient` | 무시하고 계속 진행 | 앞선 `required` 실패가 없으면 **즉시 성공** 반환 | 하나만 통과하면 됨 |
| `optional` | 최종 결과에 영향 없음 (스택에 다른 모듈이 있을 때) | 동일 | 참고용 |

- 그 밖의 컨트롤 : `include`(다른 파일의 **같은 타입** 줄을 그 자리에 삽입) · `substack`(하위 스택으로 분리 — `requisite` 의 중단 효과가 하위 스택 안으로 한정) · `[success=1 default=ignore]` 형식의 상세 제어
- 스택은 **파일에 적힌 순서대로** 위에서 아래로 실행 — 순서가 곧 정책

**검증**

```bash
grep -nE 'required|requisite|sufficient|optional' /etc/pam.d/system-auth | head -8
```

```text
2:auth        required                                     pam_env.so
4:auth        sufficient                                   pam_unix.so nullok
5:auth        required                                     pam_deny.so
...
:password    requisite                                    pam_pwquality.so local_users_only
:password    sufficient                                   pam_unix.so sha512 shadow nullok use_authtok
:password    required                                     pam_deny.so
:session     required                                     pam_limits.so
```

> 📝 **시험 포인트**: `required` 는 실패해도 나머지를 실행하고 최종 실패 — 어느 모듈에서 막혔는지 공격자에게 숨기기 위함. `requisite` 는 즉시 실패 반환 (필기 R05-59 · R09-38). `sufficient` 성공이 곧 최종 성공은 아님 — **앞선 `required` 실패가 있으면 실패** (필기 R02-60 · R10-57). 스택 끝의 `pam_deny.so` 는 "아무것도 통과 못 하면 거부"용 마감 줄.

### 5-3. `system-auth` · `password-auth` 와 `authselect`

> **상황**: RHEL 9 는 공통 인증 스택을 `authselect` 가 관리한다. 직접 편집하면 다음 `authselect apply-changes` 때 덮어써지므로 구조를 먼저 파악한다.

```bash
authselect current
readlink -f /etc/pam.d/system-auth
readlink -f /etc/pam.d/password-auth
authselect list
authselect check
grep -vE '^\s*#|^\s*$' /etc/pam.d/password-auth | head -8
```

- `system-auth` : 콘솔 로그인·`su`·`passwd` 등 **로컬** 인증의 공통 스택
- `password-auth` : `sshd` 등 **원격(비콘솔)** 인증의 공통 스택 — 두 파일을 나눈 이유는 콘솔/원격에 다른 정책을 걸기 위함
- 두 파일 모두 `/etc/authselect/` 로의 **심볼릭 링크** → 직접 편집 금지
- `authselect current` : 적용된 프로필과 활성 기능 / `list` : 사용 가능한 프로필(`sssd`, `winbind`, `nis`, `minimal`, `local`) / `check` : 설정 무결성 / `apply-changes` : 재적용
- 커스터마이즈 절차 : `authselect create-profile <이름> -b sssd` → `/etc/authselect/custom/<이름>/` 편집 → `authselect select custom/<이름>`
- `authselect enable-feature with-faillock` : 로그인 실패 잠금 기능 활성화 (Part 10)

**검증**

```bash
authselect current ; readlink -f /etc/pam.d/system-auth
```

```text
Profile ID: ...
Enabled features:
...
/etc/authselect/system-auth
```

> 📝 **시험 포인트**: 개별 서비스 파일(`/etc/pam.d/sshd` 등)이 `auth include password-auth` 로 공통 스택을 끌어 쓴다는 구조가 핵심. RHEL 9 에서 `system-auth` 를 직접 고치면 `authselect` 가 되돌릴 수 있다는 점이 실무 함정.

### 5-4. `pam_wheel.so` — `su` 를 wheel 그룹으로 제한

> **상황**: 회사 보안 규정 "wheel 구성원만 root 로 전환 가능"을 적용한다. ops1 은 3-2 에서 wheel 보조 그룹을 받았고 dev1 은 아니므로, 규정이 실제로 갈라지는지 두 계정으로 확인한다.

⚠️ PAM 편집 실수는 로그인 차단으로 이어진다. **root 세션을 하나 더 열어 둔 채** 진행하고, 원본을 먼저 백업한다.

```bash
cp -p /etc/pam.d/su /root/pam.d-su.orig
grep -n pam_wheel /etc/pam.d/su
sed -i -E 's/^#(auth[[:space:]]+required[[:space:]]+pam_wheel\.so use_uid)/\1/' /etc/pam.d/su
grep -nE '^auth' /etc/pam.d/su
cat /etc/pam.d/su-l
```

- `pam_wheel.so` : 호출자가 `wheel` 그룹(또는 `group=` 로 지정한 그룹) 구성원인지 검사
- `use_uid` : 로그인 이력(utmp)의 사용자 대신 **실제 UID** 로 판정 — `su` 를 여러 번 거친 세션에서도 정확
- `trust` : wheel 구성원이면 **비밀번호 없이** 통과 — ⚠️ 위험, 이 실습에서는 사용하지 않음
- `deny` : 판정을 반전 (지정 그룹을 오히려 차단)
- `group=<그룹>` : `wheel` 대신 다른 그룹으로 판정
- 앞줄의 `auth sufficient pam_rootok.so` 덕분에 **root 의 `su` 는 계속 허용** (root 는 이미 UID 0 이므로 즉시 성공)
- `/etc/pam.d/su-l` 이 `/etc/pam.d/su` 를 `include` → `su -`(로그인 형태)에도 동일하게 적용

**검증**

```bash
id -nG dev1 ; id -nG ops1
su - dev1        # → 세션 안에서 su - 시도
su - ops1        # → 세션 안에서 su - 시도
tail -6 /var/log/secure
```

```text
devteam
opsteam wheel

[dev1@srv01 ~]$ su -
Password:
su: Permission denied

[ops1@srv01 ~]$ su -
Password:
[root@srv01 ~]# whoami
root

... srv01 su[....]: pam_wheel(su-l:auth): Access denied to ... for ...
... srv01 su[....]: FAILED SU (to root) dev1 on pts/...
... srv01 su[....]: pam_unix(su-l:session): session opened for user root(uid=0) by ops1(uid=2003)
```

- dev1 이 **비밀번호를 먼저 입력한 뒤** 거부되는 이유 = `required` 는 스택을 끝까지 진행하기 때문 (5-2). `requisite` 였다면 비밀번호를 묻지 않고 즉시 거부
- 원복이 필요하면 `cp -p /root/pam.d-su.orig /etc/pam.d/su`

> 📝 **시험 포인트**: "wheel 그룹만 `su` 허용" → `/etc/pam.d/su` 의 `auth required pam_wheel.so use_uid` **주석 해제** (필기 R05-61 · R06-64 · R09-27). `/etc/login.defs` 의 `SU_WHEEL_ONLY` 나 `/etc/securetty` 는 오답 보기.

### 5-5. `pam_limits.so` 와 `/etc/security/limits.conf`

> **상황**: 개발자 계정이 fork 폭탄이나 파일 디스크립터 고갈로 서버를 마비시키지 못하도록 자원 상한을 건다. Part 06 의 프로세스 폭주 진단과 짝이 되는 예방 조치다.

```bash
grep -n pam_limits /etc/pam.d/system-auth /etc/pam.d/password-auth
cat /etc/security/limits.d/20-nproc.conf
cp -p /etc/security/limits.conf /root/limits.conf.orig
cat >> /etc/security/limits.conf <<'EOF'

# lab.local 자원 제한
dev1     soft    nproc      50
dev1     hard    nproc      80
dev1     soft    nofile     1024
dev1     hard    nofile     2048
@devteam hard    maxlogins  3
EOF
tail -8 /etc/security/limits.conf
```

- 형식 : `<대상> <종류> <항목> <값>` — 공백 구분 4필드
- 대상 : 사용자명 · `@그룹` · `*`(기본값) · `%그룹`(maxlogins 계열)
- 종류 : `soft`(현재 적용값 — 사용자가 hard 이하로 스스로 조정 가능) · `hard`(상한 — root 만 상향) · `-`(둘 다 동시 지정)
- 주요 항목 : `nproc`(프로세스 수) · `nofile`(열 수 있는 파일 수) · `fsize`(생성 파일 크기 KB) · `core`(코어 덤프 KB) · `as`(가상 메모리 KB) · `cpu`(CPU 시간 분) · `maxlogins`(동시 로그인 수) · `memlock` · `stack` · `nice` · `priority` · `locks`
- `/etc/security/limits.d/*.conf` : 조각 파일 — RHEL 9 기본 `20-nproc.conf` 가 `* soft nproc 4096` 를 미리 지정
- 적용 경로 : `session required pam_limits.so` (system-auth·password-auth) → **로그인 세션에만** 적용
- systemd 로 기동되는 서비스는 이 파일을 거치지 않음 → 유닛의 `LimitNOFILE=`·`LimitNPROC=` 사용 (Part 07)

**검증**

```bash
su - dev1 -c 'ulimit -a' | grep -E 'open files|max user processes'
su - dev1 -c 'ulimit -Sn; ulimit -Hn; ulimit -u'
ulimit -n                          # root 세션은 영향 없음
```

```text
open files                          (-n) 1024
max user processes                  (-u) 50
1024
2048
50
...
```

- `ulimit -a` : 전체 조회 (**a**ll)
- `-n` : nofile · `-u` : nproc · `-f` : fsize · `-c` : core · `-v` : as · `-t` : cpu
- `-H` / `-S` : hard / soft 지정 — 생략 시 **soft** 조회
- 값 변경은 현재 셸과 그 자식에만 유효 (셸 내장 명령)

> 📝 **시험 포인트**: "특정 계정의 프로세스 수 제한 설정 파일" = `/etc/security/limits.conf` (필기 R06-65 · R07-64 보기). `ulimit -n` = **열 수 있는 파일 수**(필기 R09-41), `-u` = 프로세스 수. `soft` 는 사용자가 올릴 수 있고 `hard` 는 root 만 (필기 R05-60 · R09-40).

### 5-6. 대표 모듈 목록과 RHEL 9 변경점

> **상황**: 필기 보기로 나오는 모듈 이름을 한 번에 정리하고, RHEL 9 에서 사라진 것을 정직하게 구분해 둔다.

```bash
ls /usr/lib64/security/pam_faillock.so
ls /usr/lib64/security/pam_tally2.so 2>/dev/null || echo "pam_tally2 : RHEL 9 에서 제거됨"
ls /etc/securetty 2>/dev/null       || echo "/etc/securetty : RHEL 9 에서 제거됨"
ls /usr/lib64/security/pam_cracklib.so 2>/dev/null || echo "pam_cracklib : pam_pwquality 로 대체"
ls /usr/lib64/security/pam_{unix,wheel,rootok,limits,access,time,nologin,env,succeed_if,deny,permit,pwquality,pwhistory,mkhomedir,lastlog}.so 2>/dev/null | wc -l
```

| 모듈 | 타입 | 역할 | 연동 파일 |
| --- | --- | --- | --- |
| `pam_unix.so` | auth·account·password·session | `/etc/shadow` 기반 표준 인증 | `/etc/shadow` |
| `pam_wheel.so` | auth | wheel 그룹만 `su` 허용 | `/etc/group` |
| `pam_rootok.so` | auth | 호출자가 root 면 무조건 성공 | — |
| `pam_pwquality.so` | password | 비밀번호 복잡도 강제 | `/etc/security/pwquality.conf` |
| `pam_pwhistory.so` | password | 최근 사용 비밀번호 재사용 금지 | `/etc/security/opasswd` |
| `pam_faillock.so` | auth·account | 로그인 실패 횟수 초과 시 잠금 | `/etc/security/faillock.conf`, `/var/run/faillock/` |
| `pam_limits.so` | session | 자원 상한 적용 | `/etc/security/limits.conf`, `limits.d/` |
| `pam_access.so` | account | 사용자·출발지 조합 접근 제어 | `/etc/security/access.conf` |
| `pam_time.so` | account | 시간대별 접근 제어 | `/etc/security/time.conf` |
| `pam_nologin.so` | auth·account | `/etc/nologin` 존재 시 일반 사용자 로그인 차단 | `/etc/nologin` |
| `pam_env.so` | auth·session | 환경변수 주입 | `/etc/security/pam_env.conf`, `/etc/environment` |
| `pam_mkhomedir.so` | session | 첫 로그인 시 홈 자동 생성 (LDAP 계정 등) | `/etc/skel` |
| `pam_succeed_if.so` | 전 타입 | 조건식 판정 (`uid = 0` 등) | — |
| `pam_deny.so` / `pam_permit.so` | 전 타입 | 무조건 실패 / 무조건 성공 (스택 마감·테스트용) | — |

- `pam_faillock` 상세 실습은 Part 10 참조 → [[10-security-firewall-selinux]]
- `pam_tally2.so` : RHEL 9 에서 **제거** — 동일 기능은 `pam_faillock` 사용. 필기 보기에 남아 있으면 "구형" 으로 판단
- `pam_securetty.so` / `/etc/securetty` : RHEL 9 에서 **제거** — root 콘솔 제한은 `sshd_config` 의 `PermitRootLogin`(Part 08)으로 대체
- `pam_cracklib.so` : `pam_pwquality.so` 로 대체됨

**검증**

```bash
ls /usr/lib64/security/pam_tally2.so 2>/dev/null || echo "pam_tally2 : RHEL 9 에서 제거됨"
ls /etc/securetty 2>/dev/null || echo "/etc/securetty : RHEL 9 에서 제거됨"
```

```text
pam_tally2 : RHEL 9 에서 제거됨
/etc/securetty : RHEL 9 에서 제거됨
```

> 📝 **시험 포인트**: "로그인 5회 실패 시 10분 잠금" = `pam_faillock` (필기 R06-57 · R09-39 `deny=5 unlock_time=600`). 기출에 `pam_tally2`·`pam_securetty`·`pam_cracklib` 가 정답으로 나오는 회차가 있으나 **RHEL 9 실물에는 없음** — 시험은 교재 기준, 실습은 대체 모듈 기준으로 이중 정리.

---

## 6. 계정 수정·삭제 — 수명주기

### 6-1. `usermod -aG` vs `-G` — `-a` 누락 사고 재현

> **상황**: 실기·필기 최빈출 함정. dev2 를 새 보조 그룹에 넣으면서 `-a` 를 빠뜨렸을 때 기존 소속이 실제로 사라지는지 눈으로 확인하고 복구한다.

```bash
id dev2                                  # 현재: 1차 devteam, 보조 opsteam
groupadd -g 2010 labqa                   # 실습용 임시 그룹
usermod -aG labqa dev2                   # 안전: 기존 보조 그룹 유지하며 추가
id dev2
```

⚠️ 아래는 **기존 보조 그룹을 잃는** 명령이다. 복구 절차까지 한 번에 수행한다.

```bash
usermod -G labqa dev2                    # -a 누락 → 보조 그룹 목록을 labqa 로 '대체'
id dev2                                  # opsteam 사라짐
getent group opsteam                     # 구성원 목록에서도 제거됨
usermod -G opsteam,labqa dev2            # 복구 (콤마로 전체 목록 재지정)
id dev2
usermod -G opsteam dev2 ; groupdel labqa # 임시 그룹 정리 → README 명세 상태로 원복
id dev2
```

- `-G <그룹,...>` : 보조 그룹 목록을 **통째로 대체** (**G**roups)
- `-a` : `-G` 와 함께만 유효 — 대체가 아닌 **추가** (**a**ppend)
- `gpasswd -a <user> <group>` : 그룹 쪽에서 한 명을 추가 — `-a` 누락 사고가 구조적으로 불가능 (2-4 참조)
- 그룹 변경은 **새 로그인 세션부터** 반영 — 실행 중인 셸에는 즉시 적용되지 않음 (`newgrp <그룹>` 으로 현재 세션의 1차 그룹만 임시 전환 가능)

**검증**

```bash
id dev2 ; getent group opsteam devteam
```

```text
uid=2002(dev2) gid=2000(devteam) groups=2000(devteam),2001(opsteam)      ← 시작 상태
uid=2002(dev2) gid=2000(devteam) groups=2000(devteam),2001(opsteam),2010(labqa)   ← -aG
uid=2002(dev2) gid=2000(devteam) groups=2000(devteam),2010(labqa)        ← -G (사고)
uid=2002(dev2) gid=2000(devteam) groups=2000(devteam),2001(opsteam)      ← 최종 원복
opsteam:x:2001:dev2
devteam:x:2000:dev1,dev2
```

> 📝 **시험 포인트**: "기존 그룹 유지하며 추가" = `usermod -aG <group> <user>` — 실기 R01-02, 필기 R01-25 · R04-23 · R07-32 에서 `-G`(대체) · `-g`(1차 그룹 변경) 보기와 함께 반복 출제.

### 6-2. 속성 변경 — `-c` `-d -m` `-s` `-l` `-u` `-g` `-p`

> **상황**: 부서 이동·홈 경로 정리·계정명 변경 등 운영 중 발생하는 수정 작업을 dev2 로 한 번씩 수행하고 즉시 원복한다.

```bash
usermod -c "Developer 2 (QA)" dev2 ; getent passwd dev2
usermod -s /bin/sh dev2            ; getent passwd dev2 ; usermod -s /bin/bash dev2
usermod -d /home/dev2new -m dev2   ; ls -ld /home/dev2new ; getent passwd dev2
usermod -d /home/dev2   -m dev2    ; ls -ld /home/dev2
usermod -l dev2x dev2              ; getent passwd dev2x ; usermod -l dev2 dev2x
usermod -u 2012 dev2               ; ls -ld /home/dev2 ; usermod -u 2002 dev2 ; ls -ld /home/dev2
usermod -g opsteam dev2            ; id dev2 ; usermod -g devteam dev2 ; id dev2
```

- `-c "<설명>"` : GECOS 주석 (**c**omment) → `/etc/passwd` 5번 필드
- `-d <경로>` : 홈 경로 **기록**만 변경. `-m` 과 함께 써야 실제 파일이 이동 (**m**ove)
- `-s <셸>` : 로그인 셸 (**s**hell)
- `-l <새이름>` : 로그인명 변경 (**l**ogin). 홈 디렉터리 **이름은 바뀌지 않음** → `-d -m` 별도 수행 필요
- `-u <UID>` : UID 변경 — **홈 디렉터리 안**의 파일 소유자는 자동 갱신, 홈 **밖**의 파일은 옛 UID 로 남아 고아가 됨 (6-6 에서 탐색)
- `-g <그룹>` : 1차 그룹 변경 — 기존 파일의 그룹 소유권은 자동 변경되지 않음
- `-o` : `-u` 와 함께 중복 UID 허용 (**o**verride)
- `-p '<해시>'` : 암호화된 해시를 직접 기록 — ⚠️ 평문이 아니며, 명령 이력·`ps` 에 노출되므로 **미사용**. 해시가 필요하면 `openssl passwd -6` 로 생성
- 대상 사용자가 **로그인 중이면** `-l`·`-u`·`-d` 는 거부됨 → 먼저 세션 종료

**검증**

```bash
getent passwd dev2 ; ls -ld /home/dev2 ; id dev2
```

```text
dev2:x:2002:2000:Developer 2 (QA):/home/dev2:/bin/bash
drwx------. 3 dev2 devteam ... /home/dev2
uid=2002(dev2) gid=2000(devteam) groups=2000(devteam),2001(opsteam)
```

> 📝 **시험 포인트**: `usermod -s /sbin/nologin user` = 대화식 로그인 차단(필기 R02-22 · R03-40). `-d` 만 쓰면 파일이 따라가지 않는다는 점, `-l` 이 홈 이름을 바꾸지 않는다는 점이 서술형 소재.

### 6-3. `chsh` · `chfn` · `getent`

> **상황**: 셸과 GECOS 는 전용 명령으로도 바꿀 수 있다. 일반 사용자가 스스로 바꿀 수 있는 범위를 확인한다.

```bash
cat /etc/shells
chsh -l                                     # = /etc/shells
chsh -s /bin/sh dev1     ; getent passwd dev1
chsh -s /bin/bash dev1   ; getent passwd dev1
chfn -f "Developer One" -o "R301" -p "02-000-0001" dev1
getent passwd dev1
grep -E '^CHFN_RESTRICT|^CHSH' /etc/login.defs
getent group devteam ; getent shadow dev1 | cut -c1-20 ; getent passwd 2001
getent passwd nosuchuser ; echo "rc=$?"
```

- `chsh -s <셸>` : 로그인 셸 변경 (**s**hell). 일반 사용자는 `/etc/shells` 등재 셸로만 변경 가능
- `chsh -l` : 선택 가능한 셸 목록 (**l**ist) = `/etc/shells` 내용
- `chfn -f/-o/-p/-h` : GECOS 의 실명(**f**ull name)·사무실(**o**ffice)·사무실 전화(office **p**hone)·집 전화(**h**ome phone)
- GECOS 는 **콤마 4구분** 문자열로 5번 필드에 저장 → `finger` 계열 명령이 해석
- `CHFN_RESTRICT` (`/etc/login.defs`) : 일반 사용자가 바꿀 수 있는 GECOS 항목 제한
- `getent <DB> [키]` : NSS 를 거친 조회 — `passwd` `group` `shadow` `hosts` `services` `networks` `protocols` `aliases` 등. 키 생략 시 전체, 없는 키면 **종료코드 2**
- 파일을 직접 `grep` 하는 것과 달리 LDAP·SSSD 등 원격 소스까지 포함 → 운영 환경 표준 조회 방법

**검증**

```bash
getent passwd dev1
getent passwd nosuchuser ; echo "rc=$?"
```

```text
dev1:x:2001:2000:Developer One,R301,02-000-0001,:/home/dev1:/bin/bash
rc=2
```

> 📝 **시험 포인트**: `awk -F: '$3>=1000 {print $1}' /etc/passwd` 는 실기 단골(R03-02 · R06-05)이지만, 운영에서는 `getent passwd` 로 조회하는 것이 정석. `chsh` 로 바뀌는 필드는 7번, `chfn` 은 5번.

### 6-4. 로그인 차단 셸 — `/sbin/nologin` vs `/bin/false`

> **상황**: 서비스 계정에 흔히 쓰는 두 셸의 차이를 직접 실행해 확인한다. Part 09 의 FTP·Samba 계정 설계에서 다시 쓰인다.

```bash
useradd -M -s /sbin/nologin nolog1
useradd -M -s /bin/false    false1
getent passwd nolog1 false1
su - nolog1 ; echo "rc=$?"
su - false1 ; echo "rc=$?"
ls -l /etc/nologin.txt 2>/dev/null || echo "/etc/nologin.txt 없음 → 기본 메시지 사용"
grep -c . /etc/shells ; grep -E 'nologin|false' /etc/shells || echo "두 셸 모두 /etc/shells 미등재"
userdel nolog1 ; userdel false1
```

- `/sbin/nologin` : 안내 메시지를 출력하고 **종료코드 1**. 메시지는 `/etc/nologin.txt` 가 있으면 그 내용, 없으면 기본 문구
- `/bin/false` : 아무 출력 없이 종료코드 1 — 로그 없이 조용히 끊김
- 둘 다 `/etc/shells` 에 없음 → `pam_shells.so` 나 vsftpd 의 셸 검사에 걸려 FTP 로그인도 차단 (Part 09)
- SFTP·포트포워딩은 셸 차단만으로 완전히 막히지 않는 경우가 있어 `sshd_config` 의 `Match`·`ForceCommand` 병행 필요 (Part 08)
- `/etc/nologin` (파일명이 다름) : 존재하면 `pam_nologin.so` 가 **모든 일반 사용자**의 로그인을 차단 — 점검 시간대에 사용

**검증**

```bash
su - nolog1 ; echo "rc=$?"
su - false1 ; echo "rc=$?"
```

```text
This account is currently not available.
rc=1
rc=1
```

> 📝 **시험 포인트**: "로그인 차단 셸" 보기에서 `/sbin/nologin` 과 `/bin/false` 는 둘 다 정답 후보 — 차이는 **안내 메시지 유무**. `/etc/nologin`(전체 차단)과 `/sbin/nologin`(계정별 차단)을 이름만 보고 헷갈리게 하는 문제 출제.

### 6-5. guest1 잠금 → 만료 → 세션 종료

> **상황**: 협력사 작업이 끝났다. 즉시 삭제하지 않고 **잠금 → 만료 → 세션 정리 → 삭제** 순으로 안전하게 회수한다. 실무에서 곧바로 `userdel` 하지 않는 이유는 인수인계와 감사 때문이다.

```bash
usermod -L guest1                        # 1) 비밀번호 인증 차단
passwd -S guest1
usermod -s /sbin/nologin guest1          # 2) 대화식 로그인 차단
chage -E 2026-09-01 guest1               # 3) 계정 자체 만료 (과거 날짜)
chage -l guest1 | grep -E 'Account expires'
grep '^guest1:' /etc/shadow | awk -F: '{print "만료일수="$8}'
who | grep guest1 || echo "guest1 로그인 세션 없음"   # 4) 세션 확인
ps -u guest1 2>/dev/null || echo "guest1 소유 프로세스 없음"
pkill -u guest1 ; echo "rc=$?"           # 5) 남은 프로세스 정리
```

- `usermod -L` : 비밀번호 인증만 차단 (4-6)
- `usermod -s /sbin/nologin` : 대화식 셸 차단 (6-4)
- `chage -E <과거날짜>` : 계정 자체를 만료 — 공개키 인증까지 포함해 로그인 거부. `chage -E 0` 도 만료 처리
- `pkill -u <user>` : 해당 사용자 소유 프로세스 전체에 SIGTERM. 대상이 없으면 **종료코드 1** (Part 06)
- `ps -u <user>` : 사용자 기준 프로세스 조회
- 삭제 전 세션이 남아 있으면 `userdel` 이 거부 → `-f` 강제는 파일 잠금·데이터 손상 위험

**검증**

```bash
chage -l guest1 | grep -E 'Account expires' ; passwd -S guest1
su - guest1 -c 'id -un'
```

```text
Account expires						: Sep 01, 2026
guest1 LK 2026-09-03 0 99999 7 7 (Password locked.)
guest1
```

- root 의 `su` 가 만료를 무시하고 성공하는 이유 = `/etc/pam.d/su` 의 `account sufficient pam_succeed_if.so uid = 0 use_uid quiet` 가 **호출자 UID 0** 을 즉시 통과시키기 때문
- 실제 만료 동작은 일반 로그인 경로(`ssh guest1@localhost`)에서 `Your account has expired ...` 로 확인 — SSH 실습은 [[08-network-config]]

> 📝 **시험 포인트**: `chage -E 2026-12-31 alice` = **계정 만료** (필기 R09-23). `-M` 은 비밀번호 최대 사용일이지 만료일이 아님. `usermod -L` 만으로는 공개키 로그인이 살아 있다는 점이 서술형 감점 포인트.

### 6-6. `userdel` vs `userdel -r` — 고아 파일 탐색과 정리

> **상황**: `-r` 없이 지웠을 때 무엇이 남는지 직접 확인한 뒤, 남은 것을 `find -nouser` 로 찾아 정리한다. 침해 점검에서도 쓰는 탐색 방법이다.

```bash
ls -ld /home/guest1
ls -l /var/spool/mail/guest1
crontab -l -u guest1 2>/dev/null || echo "guest1 crontab 없음"
last guest1 | head -3
userdel guest1                                     # -r 없이 삭제
getent passwd guest1 || echo "passwd 항목 제거됨"
getent group  guest1 || echo "개인 그룹(UPG)도 함께 제거됨"
ls -ld /home/guest1
ls -l /var/spool/mail/guest1
find / -xdev \( -nouser -o -nogroup \) 2>/dev/null | head
```

- `userdel <user>` : `/etc/passwd` · `/etc/shadow` · `/etc/group` · `/etc/gshadow` 항목만 제거. 홈·메일 스풀·기타 파일은 **잔존**
- `userdel -r <user>` : 홈 디렉터리와 `/var/spool/mail/<user>` 까지 삭제 (**r**emove) — 홈 **밖**의 파일은 여전히 남음
- `userdel -f <user>` : ⚠️ 로그인 중이어도 강제 삭제, 다른 사용자의 1차 그룹이어도 그룹 제거 (**f**orce)
- `userdel -Z` : SELinux 사용자 매핑도 제거
- 개인 그룹(UPG)은 다른 구성원이 없으면 자동 삭제
- `find / -nouser` : `/etc/passwd` 에 없는 UID 소유 파일 / `-nogroup` : 대응 그룹이 없는 파일
- `-xdev` : 다른 파일시스템으로 내려가지 않음 — `/proc`·`/sys`·`/run` 오탐 방지
- `\( … -o … \)` : `-o`(OR)의 결합 범위를 괄호로 고정 — 이스케이프 필수

**검증**

```bash
find / -xdev \( -nouser -o -nogroup \) 2>/dev/null | head
```

```text
drwx------. 3 2004 2004 ... /home/guest1
-rw-------. 1 2004 mail 0 ... /var/spool/mail/guest1
/home/guest1
/home/guest1/.bash_logout
/home/guest1/.bash_profile
/home/guest1/.bashrc
/var/spool/mail/guest1
```

정리 후 재확인:

```bash
rm -rf /home/guest1 /var/spool/mail/guest1
find / -xdev \( -nouser -o -nogroup \) 2>/dev/null | head || true
last guest1 | head -3
lastlog -u guest1 2>/dev/null || echo "lastlog 항목 없음"
```

```text
guest1   pts/1        ...   still logged in
guest1   pts/1        ...
wtmp begins ...
```

- 계정을 지워도 `/var/log/wtmp` 의 로그인 이력은 남아 `last <이름>` 으로 조회 가능 — 감사 추적의 근거 (Part 07·10)
- 삭제 전 반드시 확인할 잔여물 : 홈 · 메일 스풀 · `crontab -u` · `at` 작업 · `/tmp` 산출물 · cron/systemd 유닛의 `User=` 참조

> 📝 **시험 포인트**: `userdel -r` = 홈·메일 스풀까지 삭제. `-r` 없이 지운 뒤 `find / -nouser` 로 고아 파일을 찾는 흐름이 실기 서술 소재. `/var/log/wtmp` 는 계정 삭제와 무관하게 보존.

---

## 7. su / sudo — 권한 상승

### 7-1. `su` · `su -` · `su -c` — 환경 차이

> **상황**: 5-4 에서 `su` 제한을 걸었으니 이제 `su` 자체의 동작을 정리한다. `-` 하나의 유무가 환경변수와 작업 디렉터리를 어떻게 바꾸는지 직접 비교한다.

```bash
cd /etc
su   dev1 -c 'echo "PWD=$PWD"; echo "HOME=$HOME"; echo "USER=$USER"; echo "PATH=$PATH"'
su - dev1 -c 'echo "PWD=$PWD"; echo "HOME=$HOME"; echo "USER=$USER"; echo "PATH=$PATH"'
su - dev1 -c 'umask; id -un'
grep -E '^ENV_PATH|^ENV_SUPATH' /etc/login.defs
cd /root
```

- `su [<user>]` : **비로그인 셸** — 현재 디렉터리 유지, 대상의 `~/.bash_profile` 미실행. 사용자 생략 시 root
- `su - [<user>]` (= `su -l`, `su --login`) : **로그인 셸** — 대상 홈으로 이동하고 `/etc/profile` → `/etc/profile.d/*.sh` → `~/.bash_profile` → `~/.bashrc` → `/etc/bashrc` 순으로 실행, 환경 초기화
- `-c '<명령>'` : 명령 1회 실행 후 복귀 (**c**ommand). `su - dev1 -c '…'` 처럼 `-` 와 병행 가능
- `-s <셸>` : 사용할 셸 지정 (**s**hell) — `/sbin/nologin` 계정 점검에 사용
- `-m` / `-p` : 환경변수 보존 (**p**reserve-environment)
- PATH 는 `/etc/login.defs` 의 `ENV_PATH`(일반 사용자)·`ENV_SUPATH`(root) 값으로 재설정됨
- 인증 주체 : `su` 는 **전환 대상 계정**의 비밀번호, `sudo` 는 **실행자 본인**의 비밀번호

**검증**

```bash
cd /etc ; su dev1 -c 'echo $PWD' ; su - dev1 -c 'echo $PWD' ; cd /root
```

```text
PWD=/etc
HOME=/home/dev1
USER=dev1
PATH=/usr/local/bin:/usr/bin:...

PWD=/home/dev1
HOME=/home/dev1
USER=dev1
PATH=/usr/local/bin:/usr/bin:...:/home/dev1/.local/bin:/home/dev1/bin
```

- 로그인 셸에서만 `~/.bash_profile` 의 `PATH=$PATH:$HOME/.local/bin:$HOME/bin` 이 실행되어 끝에 두 경로가 추가됨

> 📝 **시험 포인트**: `su -` 는 대상 계정의 **환경변수까지 전환**, `su` 는 현재 환경 유지 — `-` 유무가 반복 출제 (필기 R03-21 · R07-60). "`su` 는 대상 비밀번호, `sudo` 는 본인 비밀번호" 도 세트로 암기.

### 7-2. `sudo` 주요 옵션

> **상황**: 규칙을 만들기 전에 조회·실행·타임스탬프 옵션을 먼저 익힌다. `sudo -l` 은 이후 모든 검증의 기준 명령이 된다.

```bash
sudo -l                                  # 현재 사용자(root)의 허용 규칙
sudo -l -U ops1                          # 다른 사용자의 규칙 조회
sudo -u dev1 id                          # dev1 권한으로 실행
sudo -u dev1 -g devteam id
su - ops1 -c 'sudo -l'                   # ops1 본인 관점
ls -l /run/sudo/ts/ 2>/dev/null || echo "타임스탬프 디렉터리 미생성"
sudo -k ; echo "타임스탬프 무효화"
sudo -v ; echo "타임스탬프 갱신"
```

- `-l` : 허용 규칙 나열 (**l**ist) — `-ll` 은 규칙을 줄 단위 상세 형식으로 출력
- `-U <user>` : 다른 사용자의 규칙 조회 (**U**ser) — root 권한 필요, `-l` 과 함께만 사용
- `-u <user>` : 지정 사용자 권한으로 실행 (**u**ser). 생략 시 root
- `-g <group>` : 지정 그룹 권한으로 실행
- `-i` : 대상 사용자의 **로그인 셸** 실행 (`su -` 에 대응, 환경 초기화)
- `-s` : 대상 사용자의 셸 실행 (비로그인, `su` 에 대응)
- `-k` : 인증 타임스탬프 무효화 (**k**ill) → 다음 `sudo` 에서 비밀번호 재입력
- `-v` : 타임스탬프 갱신 (**v**alidate) — 명령 실행 없이 유효기간만 연장
- `-n` : 비대화식 (**n**on-interactive) — 비밀번호가 필요하면 즉시 실패. 스크립트·cron 용
- `-b` : 백그라운드 실행 / `-E` : 현재 환경변수 유지 (정책이 허용할 때만)
- `-e` (= `sudoedit`) : 파일을 사용자 권한 편집기로 열고 결과만 교체 — 편집기 셸 탈출 차단
- 타임스탬프 저장 위치 : `/run/sudo/ts/<user>` (기본 유효 5분, `timestamp_timeout`)

**검증**

```bash
sudo -u dev1 id ; sudo -l -U ops1 | tail -3
```

```text
uid=2001(dev1) gid=2000(devteam) groups=2000(devteam)
User ops1 may run the following commands on srv01:
    (ALL) ALL
```

- 이 시점 ops1 은 `wheel` 소속이므로 `%wheel ALL=(ALL) ALL` 규칙에 걸려 **전권** — 7-5 에서 제한한다

> 📝 **시험 포인트**: `sudo` 실행 이력은 `/var/log/secure` 에 남아 감사 추적이 **가능**하다 — "로그가 남지 않는다"는 보기는 오답 (필기 R03-21 · R07-60).

### 7-3. `visudo` 와 `/etc/sudoers` 형식

> **상황**: sudoers 는 문법 오류 하나로 전 계정의 sudo 가 막힌다. 반드시 `visudo` 를 거치는 이유와 규칙 5요소를 확인한다.

```bash
visudo -c                                       # 현재 문법 검사
grep -vE '^\s*#|^\s*$' /etc/sudoers
tail -3 /etc/sudoers                            # #includedir 지시자 확인
ls -l /etc/sudoers /etc/sudoers.d
EDITOR=vi visudo                                # 실제 편집 (편집기 지정)
```

- `visudo` : 임시 사본을 편집 → 저장 시 문법 검사 → **통과해야 반영**. 동시 편집 잠금 제공
- `-c` : 검사만 수행 (**c**heck) — `#includedir` 로 포함된 파일까지 함께 검사
- `-f <파일>` : 지정 파일을 검사와 함께 편집 (조각 파일 편집용)
- `-s` : 엄격 모드 / `-q` : 조용히
- 편집기 우선순위 : `SUDO_EDITOR` → `VISUAL` → `EDITOR` → sudoers 의 `editor` 지시자
- `/etc/sudoers` 권한은 `0440`, 소유 `root:root` — 다르면 sudo 가 실행을 거부
- 파일 끝의 `#includedir /etc/sudoers.d` 는 주석이 아니라 **지시자**

| 위치 | 예 | 의미 |
| --- | --- | --- |
| 사용자 | `ops1` · `%wheel` · `#2003` · `%#2001` | 규칙 적용 대상 (`%` = 그룹, `#` = UID/GID) |
| 호스트 | `ALL` · `srv01` | 이 규칙이 유효한 호스트 — 하나의 sudoers 를 여러 서버에 배포할 때 사용 |
| `(실행대상)` | `(ALL)` · `(root)` · `(ALL:ALL)` | `-u`/`-g` 로 지정 가능한 대상 사용자[:그룹] |
| 태그 | `NOPASSWD:` `PASSWD:` `NOEXEC:` `SETENV:` `LOG_INPUT:` | 해당 명령 이후에 적용 |
| 명령 | `ALL` · `/usr/bin/systemctl restart httpd` · `!/usr/bin/su` | **절대경로** 필수, `!` = 부정 |

**검증**

```bash
visudo -c ; grep -E '^(root|%wheel)' /etc/sudoers
```

```text
/etc/sudoers: parsed OK
/etc/sudoers.d/... : parsed OK
root	ALL=(ALL) 	ALL
%wheel	ALL=(ALL)	ALL
```

> 📝 **시험 포인트**: "문법 검사와 동시 편집 잠금을 제공하는 명령" = `visudo` (필기 R04-60 · R09-29). `%wheel ALL=(ALL) ALL` 해석(그룹 전체·모든 호스트·모든 대상·모든 명령)은 필기 R08-59 · R09-28.

### 7-4. Alias 와 `Defaults` 옵션

> **상황**: 규칙이 늘어나면 별칭으로 묶고 동작 옵션을 조정한다. 조각 파일 하나로 실습용 `Defaults` 를 넣는다.

```bash
cat > /etc/sudoers.d/99-lab-defaults <<'EOF'
# lab.local sudo 동작 옵션
Defaults        logfile=/var/log/sudo-lab.log
Defaults:ops1   timestamp_timeout=5
Defaults        env_keep += "LANG LC_ALL"
EOF
chmod 0440 /etc/sudoers.d/99-lab-defaults
chown root:root /etc/sudoers.d/99-lab-defaults
visudo -c
grep -E '^Defaults' /etc/sudoers | head
```

- `User_Alias` / `Runas_Alias` / `Host_Alias` / `Cmnd_Alias` : 별칭 정의. 이름은 **대문자·숫자·밑줄**만 사용
	- `User_Alias  OPSADMIN = ops1, dev2, %opsteam`
	- `Cmnd_Alias  LAB_SVC  = /usr/bin/systemctl restart httpd, /usr/bin/journalctl`
	- `Host_Alias  LOCAL    = srv01, 192.168.64.10`
	- `Runas_Alias DBA      = postgres, mysql`
- `Defaults` 범위 한정 : `Defaults:<사용자>` · `Defaults@<호스트>` · `Defaults>(<실행대상>)` · `Defaults!<명령>`
- `logfile=<경로>` : sudo 전용 로그 파일 추가 (기본은 syslog → `/var/log/secure`)
- `timestamp_timeout=<분>` : 비밀번호 재확인 주기 (기본 5, `0` = 매번, `-1` = 무제한 ⚠️)
- `requiretty` : tty 없는 실행 차단 — RHEL 9 기본 **미설정**이라 스크립트·cron 에서도 sudo 사용 가능
- `env_reset` : 환경변수 초기화 (기본 켜짐) / `env_keep` : 예외 목록 (`+=` 로 추가) / `env_delete`
- `secure_path` : sudo 실행 시 강제 적용되는 PATH — 사용자 PATH 조작을 통한 권한 상승 차단
- `targetpw` : 실행 대상 계정의 비밀번호를 요구 (기본은 실행자 본인)
- `!visiblepw` : 비밀번호가 화면에 보이는 상황(파이프 입력 등)에서 실행 거부

**검증**

```bash
visudo -c ; ls -l /etc/sudoers.d/
```

```text
/etc/sudoers: parsed OK
/etc/sudoers.d/99-lab-defaults: parsed OK
-r--r-----. 1 root root ... 99-lab-defaults
```

> 📝 **시험 포인트**: `Defaults secure_path=...` 의 목적(PATH 조작 방어)과 `timestamp_timeout` 의 의미가 보기 지문으로 등장. 별칭 이름을 소문자로 쓰면 문법 오류라는 점도 함정.

### 7-5. `/etc/sudoers.d/ops1` — 운영자에게 제한된 sudo 부여

> **상황**: 운영자 ops1 에게 서비스 재시작과 로그 조회만 허용한다. ops1 은 3-2 에서 `wheel` 을 받았으므로 `%wheel ALL=(ALL) ALL` 에 이미 걸려 있다 — sudoers 의 **마지막 매칭 규칙 우선** 성질을 이용해 먼저 전부 차단하고 필요한 것만 되살린다.

```bash
visudo -f /etc/sudoers.d/ops1
```

작성 내용:

```
# /etc/sudoers.d/ops1 — 운영자 ops1 제한 위임
Cmnd_Alias LAB_SVC = /usr/bin/systemctl restart httpd, \
                     /usr/bin/systemctl reload  httpd, \
                     /usr/bin/systemctl status  httpd
Cmnd_Alias LAB_LOG = /usr/bin/journalctl

ops1    ALL=(root)  !ALL                 # 1) 먼저 전부 차단 (%wheel 규칙 무력화)
ops1    ALL=(root)  LAB_SVC, LAB_LOG     # 2) 필요한 것만 재허용 (뒤 규칙이 우선)
```

```bash
ls -l /etc/sudoers.d/ops1
visudo -c
sudo -l -U ops1
```

- `visudo -f <파일>` : 조각 파일을 문법 검사와 함께 편집 — `0440` · `root:root` 로 저장됨
- 파일명 규칙 : 이름에 `.` 이나 `~` 가 들어가면 sudo 가 **무시** (`ops1.conf` 는 읽히지 않음)
- `\` : 줄 이어쓰기
- `!ALL` : 모든 명령 부정 — 이후 줄에서 다시 허용하는 **화이트리스트** 패턴
- 규칙 평가는 **마지막으로 일치한 규칙**이 승리 — `#includedir` 가 `/etc/sudoers` 끝에 있으므로 조각 파일이 나중에 읽힘
- 대안 : `gpasswd -d ops1 wheel` 로 wheel 에서 제외 — 다만 README 2-1 명세(ops1 = wheel 보조)를 유지하기 위해 여기서는 규칙 우선순위로 해결
- 명령은 반드시 **절대경로** — 와일드카드(`/usr/bin/systemctl *`)는 인자 조작 우회 위험

**검증** — 허용·불허를 각각 실행

```bash
su - ops1
```

```text
[ops1@srv01 ~]$ sudo -l | tail -3
    (root) !ALL
    (root) /usr/bin/systemctl restart httpd, /usr/bin/systemctl reload httpd,
        /usr/bin/systemctl status httpd, /usr/bin/journalctl

[ops1@srv01 ~]$ sudo journalctl -n 3 --no-pager        ← 허용 → 실행됨
[sudo] password for ops1:
... srv01 systemd[1]: ...

[ops1@srv01 ~]$ sudo systemctl restart httpd           ← 허용됨. 유닛이 아직 없어 systemctl 이 실패
Failed to restart httpd.service: Unit httpd.service not found.

[ops1@srv01 ~]$ sudo systemctl restart sshd            ← 불허 → sudo 가 거부
Sorry, user ops1 is not allowed to execute '/usr/bin/systemctl restart sshd' as root on srv01.
```

- `httpd` 는 [[09-network-services]] 에서 설치 — 지금은 **sudo 인가는 통과하고 systemctl 이 실패**하는 것이므로 거부 메시지와 구분해서 읽을 것
- 거부 메시지의 경로는 `secure_path` 탐색 결과에 따라 `/bin/systemctl` 로 표시될 수 있음 (usrmerge 로 `/bin` → `/usr/bin`, sudo 는 inode 비교로 매칭)

> 📝 **시험 포인트**: `webadm ALL=(root) /usr/bin/systemctl restart httpd` 형태가 필기 R06-56 정답 형식. `lee ALL=(ALL) NOPASSWD: /usr/sbin/useradd` 해석은 R05-32, `%wheel ALL=(ALL) NOPASSWD: /usr/bin/systemctl` 은 R10-62 — **`%` = 그룹**, `NOPASSWD:` = 해당 명령에 한해 비밀번호 생략.

### 7-6. `/var/log/secure` 의 sudo 로그와 NOPASSWD 위험성

> **상황**: 위임했으면 추적할 수 있어야 한다. 성공·거부·비밀번호 실패가 각각 어떤 형태로 남는지 확인한다.

```bash
grep 'sudo' /var/log/secure | tail -6
grep 'COMMAND=' /var/log/secure | tail -3
grep 'command not allowed' /var/log/secure | tail -2
tail -3 /var/log/sudo-lab.log 2>/dev/null
journalctl -t sudo -n 5 --no-pager
sudo -ll -U ops1
```

- `/var/log/secure` : `authpriv.*` facility 의 목적지 — `su`·`sudo`·`sshd`·`login` 인증 이력 (Part 07 rsyslog)
- 성공 로그 필드 : `<사용자> : TTY=... ; PWD=... ; USER=<실행대상> ; COMMAND=<절대경로와 인자>`
- 거부 로그 : `command not allowed` 문구 포함
- 비밀번호 실패 : `N incorrect password attempts`
- `sudo -ll` : 규칙을 줄 단위 상세 형식으로 — 어느 파일에서 온 규칙인지 확인 가능
- `journalctl -t sudo` : 저널에서 syslog 태그 기준 조회 (Part 07)

**검증**

```bash
grep 'COMMAND=' /var/log/secure | tail -3
```

```text
... srv01 sudo[....]:     ops1 : TTY=pts/1 ; PWD=/home/ops1 ; USER=root ; COMMAND=/usr/bin/journalctl -n 3 --no-pager
... srv01 sudo[....]:     ops1 : command not allowed ; TTY=pts/1 ; PWD=/home/ops1 ; USER=root ; COMMAND=/usr/bin/systemctl restart sshd
... srv01 sudo[....]: pam_unix(sudo:session): session opened for user root(uid=0) by ops1(uid=2003)
```

**NOPASSWD 의 위험**

- `%wheel ALL=(ALL) NOPASSWD: ALL` = 잠기지 않은 세션을 탈취당하면 **즉시 root**
- 셸 탈출이 가능한 명령을 NOPASSWD 로 주면 사실상 전권 — `vi` · `less` · `awk` · `find -exec` · `tar --to-command` · `systemctl`(페이저 경유) 등
- 완화책 : `NOEXEC:` 태그(자식 프로세스 실행 차단) · `sudoedit` 사용 · 명령 인자를 고정 · `Defaults logfile` 과 `log_output` 으로 기록 강화

> 📝 **시험 포인트**: `NOPASSWD:` = "그 명령을 sudo 로 실행할 때 비밀번호 입력을 요구하지 않음" (필기 R05-32 · R10-62). 계정 비밀번호를 없애거나 대상 계정의 비밀번호를 쓰는 뜻이 **아님**.

---

## 8. 권한 기초 — 문자열 해석 · umask · chmod · chown

### 8-1. 권한 문자열 10자리 해석

> **상황**: 이후 모든 실습의 판독 기준. `ls -l` 한 줄에서 파일 유형·3주체 권한·확장 표시를 분해한다.

```bash
ls -l  /etc/passwd /etc/shadow
ls -ld /tmp /home/dev1 /srv/devteam
ls -l  /dev/vda /dev/null /dev/log 2>/dev/null
stat -c '%A %a %U %G %F %n' /etc/passwd /tmp /dev/null
```

| 자리 | 예 (`-rwxr-xr--`) | 의미 |
| --- | --- | --- |
| 1 | `-` | 파일 유형 |
| 2~4 | `rwx` | 소유자 (user) |
| 5~7 | `r-x` | 그룹 (group) |
| 8~10 | `r--` | 기타 (other) |
| 11 | `.` 또는 `+` | `.` = SELinux 컨텍스트 있음, `+` = ACL 있음, 공백 = 둘 다 없음 |

| 유형 문자 | 대상 | 예 |
| --- | --- | --- |
| `-` | 일반 파일 | `/etc/passwd` |
| `d` | 디렉터리 | `/tmp` |
| `l` | 심볼릭 링크 | `/bin` |
| `b` | 블록 장치 | `/dev/vda` |
| `c` | 문자 장치 | `/dev/null` |
| `p` | 명명 파이프 (FIFO) | `mkfifo` 생성물 |
| `s` | 소켓 | `/dev/log` |

- 권한값 : `r`=4 · `w`=2 · `x`=1 · `-`=0 → 주체별 합산 (`rwx`=7, `r-x`=5, `r--`=4)
- 디렉터리의 `x` = **진입·경로 통과**, `r` = 이름 목록 조회, `w` = 파일 생성·삭제·이름변경 — 삭제 가능 여부는 **파일 권한이 아니라 디렉터리 권한**이 결정
- `stat -c` 포맷 : `%A`(기호 권한) `%a`(8진수) `%U`/`%G`(소유자·그룹) `%F`(유형) `%n`(이름)

**검증**

```bash
stat -c '%A %a %U %G %F %n' /etc/passwd /tmp
```

```text
-rw-r--r--. 1 root root ... /etc/passwd
----------. 1 root root ... /etc/shadow
drwxrwxrwt. ... /tmp
drwx------. 3 dev1 devteam ... /home/dev1
-rw-r--r-- 644 root root regular file /etc/passwd
drwxrwxrwt 1777 root root directory /tmp
```

> 📝 **시험 포인트**: `-rwxr-sr-x` 에서 **그룹 위치 `s` = SetGID** (필기 R08-03). `-rwsr-xr-x` = 4755 (필기 R02-25 · R10-24). 첫 문자 `-` 는 일반 파일이며 심볼릭 링크는 `l`.

### 8-2. `umask` — 조회 · 계산 · 실측

> **상황**: 새로 만드는 파일 권한이 왜 644 로 나오는지 확인하고, 값을 바꿔가며 실제 결과와 계산이 일치하는지 검증한다. 9절의 공유 디렉터리 설계 전제가 된다.

```bash
umask                                   # 8진수
umask -S                                # 기호 표기 (남는 권한)
umask -p                                # 재사용 가능한 형태
su - dev1   -c 'umask'                  # 1차 그룹명(devteam) ≠ 사용자명 → 022
su - admin1 -c 'umask'                  # UPG(1차 그룹명 = 사용자명) & UID>199 → 002
mkdir -p /tmp/umasklab && cd /tmp/umasklab
umask 022 ; touch f022 ; mkdir d022
umask 027 ; touch f027 ; mkdir d027
umask 077 ; touch f077 ; mkdir d077
umask 023 ; touch f023 ; mkdir d023
umask 022
ls -l  f0*
ls -ld d0*
```

| umask | 파일 (666 기준) | 디렉터리 (777 기준) | 비고 |
| --- | --- | --- | --- |
| `002` | `664` | `775` | UPG 환경 기본 (그룹 공동작업) |
| `022` | `644` | `755` | 시스템 기본 |
| `027` | `640` | `750` | 그룹 읽기만·기타 차단 |
| `077` | `600` | `700` | 개인 전용 |
| `023` | **`644`** | `754` | 단순 뺄셈(643)이 **아님** |

- 파일 기준값은 `777` 이 아니라 **`666`** — 실행 권한은 umask 와 무관하게 기본 미부여
- 계산식은 뺄셈이 아니라 비트 마스킹 : `기준값 AND (NOT umask)` — `666 & ~023` = `644`
- `umask -S` : `u=rwx,g=rx,o=rx` 형태 (**S**ymbolic) — **남는 권한**을 표기 (마스크가 아님)
- `umask -p` : `umask 0022` 형태로 출력 (**p**rint, 스크립트 재사용용)
- 값은 현재 셸과 그 자식 프로세스에만 유효 (셸 내장 명령)

**검증**

```bash
ls -l f0* ; ls -ld d0*
```

```text
0022
u=rwx,g=rx,o=rx
umask 0022
0022
0002
-rw-r--r--. 1 root root 0 ... f022
-rw-r-----. 1 root root 0 ... f027
-rw-------. 1 root root 0 ... f077
-rw-r--r--. 1 root root 0 ... f023
drwxr-xr-x. 2 root root 6 ... d022
drwxr-x---. 2 root root 6 ... d027
drwx------. 2 root root 6 ... d077
drwxr-xr--. 2 root root 6 ... d023
```

> 📝 **시험 포인트**: umask `022` → 파일 `644`, umask `027` → 파일 `640`·디렉터리 `750` (실기 R02-06 · R04-13 · R05-06, 필기 R01-30 · R02-44 · R03-24 · R04-28 · R05-29 · R07-43 · R08-39 · R09-34 · R10-65). "파일이 640, 디렉터리가 750" → umask **027** (필기 R06-45). 파일 기준 **666** 을 777 로 착각하면 전부 오답.

### 8-3. umask 영구 설정 — `/etc/profile` · `/etc/bashrc` · `~/.bashrc`

> **상황**: 로그인할 때마다 유지되도록 설정한다. RHEL 이 UID 와 UPG 여부로 umask 를 나누는 코드도 함께 읽어 8-2 의 실측 결과(dev1=022, admin1=002)를 설명한다.

```bash
grep -n -A6 'umask' /etc/bashrc | head -14
grep -n -A6 'umask' /etc/profile | head -14
grep -nE '^UMASK|^HOME_MODE' /etc/login.defs
```

RHEL 9 의 조건부 umask 코드:

```bash
if [ $UID -gt 199 ] && [ "`id -gn`" = "`id -un`" ]; then
    umask 002          # UID 200 이상 & 1차 그룹명 = 사용자명(UPG) → 그룹 쓰기 허용
else
    umask 022          # 시스템 계정, 또는 공용 1차 그룹을 쓰는 계정
fi
```

전역·개인 설정 적용:

```bash
cat > /etc/profile.d/lab-umask.sh <<'EOF'
# lab.local : devteam 구성원은 공유 디렉터리 협업을 위해 002
if id -nG 2>/dev/null | grep -qw devteam; then
    umask 002
fi
EOF
chmod 644 /etc/profile.d/lab-umask.sh
su - dev1 -c 'umask'                      # 전역 설정 적용 → 0002

echo 'umask 027' >> /home/dev1/.bashrc    # 개인 설정 (나중에 실행되므로 우선)
su - dev1 -c 'umask; touch ~/u027.txt; ls -l ~/u027.txt'

sed -i '/^umask 027$/d' /home/dev1/.bashrc   # 원복 — 9절 SetGID 실습은 002 전제
su - dev1 -c 'umask'
```

- 로그인 셸 실행 순서 : `/etc/profile` → `/etc/profile.d/*.sh` → `~/.bash_profile` → `~/.bashrc` → `/etc/bashrc` — **나중에 실행된 `umask` 가 이긴다**
- `/etc/profile.d/*.sh` : 전역 설정의 권장 위치 — `/etc/profile` 원본을 건드리지 않아 패키지 갱신에 안전
- `/etc/bashrc` : 대화형 **비로그인** 셸에도 적용 (새 터미널 탭 등)
- `~/.bashrc` : 사용자 개인 설정 — 전역보다 뒤에 실행되어 우선
- `/etc/login.defs` 의 `UMASK` 는 `useradd` 가 홈을 만들 때 쓰던 값. RHEL 9 는 `HOME_MODE 0700` 이 우선 (1-2 참조)

**검증**

```bash
su - dev1 -c 'umask; touch /tmp/dev1-check.txt; ls -l /tmp/dev1-check.txt'
```

```text
0002
-rw-rw-r--. 1 dev1 devteam 0 ... /tmp/dev1-check.txt
```

> 📝 **시험 포인트**: umask 영구 적용 위치로 `/etc/profile`·`/etc/bashrc`·`~/.bashrc` 를 묻는 문제. "UPG 환경에서 일반 사용자 기본 umask 가 002 인 이유"는 서술형 소재 — 1차 그룹이 자기 전용이라 그룹 쓰기를 열어도 안전하기 때문.

### 8-4. `chmod` — 8진수와 기호 모드

> **상황**: 두 표기법을 모두 쓸 수 있어야 한다. 같은 파일에 번갈아 적용하며 결과를 확인하고, `-R` 과 `X` 의 차이까지 본다.

```bash
cd /tmp/umasklab
touch demo.sh ; mkdir -p demodir
chmod 755 demo.sh                  ; ls -l  demo.sh
chmod u+s demo.sh                  ; ls -l  demo.sh
chmod u-s,g+s demo.sh              ; ls -l  demo.sh
chmod g-s demo.sh
chmod a=rx demo.sh                 ; ls -l  demo.sh
chmod ugo+x,u+w demo.sh            ; ls -l  demo.sh
chmod o+t demodir                  ; ls -ld demodir
chmod 0750 demodir                 ; ls -ld demodir     # 3자리 → 특수비트 초기화
chmod --reference=demo.sh demodir  ; ls -ld demodir
mkdir -p tree/sub ; touch tree/a tree/sub/b
chmod -R 750 tree                  ; ls -l tree tree/sub
chmod -R u+rwX,go-rwx tree         ; ls -ld tree tree/sub ; ls -l tree/a
```

- 8진수 4자리 : `<특수><소유자><그룹><기타>` — **3자리만 쓰면 특수 비트가 0 으로 초기화**됨
- 기호 모드 : 대상 `u`(user) `g`(group) `o`(other) `a`(all) + 연산 `+` `-` `=` + 권한 `r` `w` `x` `s` `t` `X`
- `=` : 지정한 권한만 남기고 나머지 제거 (`chmod a=r file` → `444`)
- `X` : **디렉터리이거나 이미 실행 권한이 있는 파일**에만 `x` 부여 — `-R` 와 함께 쓰면 일반 파일에 불필요한 실행권한이 붙는 것을 방지
- `-R` : 하위 재귀 (**R**ecursive)
- `--reference=<파일>` : 다른 파일의 권한을 그대로 복사
- `-v` / `-c` : 변경 내역 출력 (**v**erbose / 변경된 것만 **c**hanges)
- 권한 변경은 **파일 소유자와 root** 만 가능 (그룹 구성원이어도 불가)

**검증**

```bash
ls -l demo.sh ; ls -ld demodir tree tree/sub ; ls -l tree/a
```

```text
-rwxr-xr-x. 1 root root 0 ... demo.sh
-rwsr-xr-x. 1 root root 0 ... demo.sh
-rwxr-sr-x. 1 root root 0 ... demo.sh
-r-xr-xr-x. 1 root root 0 ... demo.sh
-rwxr-xr-x. 1 root root 0 ... demo.sh
drwxr-x--t. 2 root root 6 ... demodir
drwxr-x---. 2 root root 6 ... demodir
drwx------. 3 root root ... tree
drwx------. 2 root root ... tree/sub
-rw-------. 1 root root 0 ... tree/a
```

> 📝 **시험 포인트**: `chmod 4755` = SetUID + 755 (실기 R03-06). `chmod -R 750 /data` (실기 R04-03). `chmod 755 report.txt` 를 기호로 쓰면 `u=rwx,go=rx` (실기 R01-06). `chmod 755` 처럼 3자리를 쓰면 기존 SetUID 가 **사라진다**는 점이 함정.

### 8-5. `chown` · `chgrp` — 소유권 이전

> **상황**: 공유 디렉터리를 만들기 전에 소유자·그룹 변경 옵션을 정리한다. 특히 소유권 변경이 특수 권한을 지운다는 점을 확인한다.

```bash
cd /tmp/umasklab
chown dev1 demo.sh                      ; ls -l demo.sh
chown dev1:devteam demo.sh              ; ls -l demo.sh
chown :opsteam demo.sh                  ; ls -l demo.sh      # 그룹만 = chgrp
chgrp devteam demo.sh                   ; ls -l demo.sh
chown --reference=demo.sh demodir       ; ls -ld demodir
chown -R dev1:devteam tree              ; ls -ld tree ; ls -l tree/a
chown --from=dev1:devteam dev2 tree/a   ; ls -l tree/a
ln -sfn demo.sh demo.link
chown -h dev2 demo.link                 ; ls -l demo.link
# SetUID 파일의 소유권 변경 → 특수 비트 자동 제거 확인
chmod 4755 demo.sh ; ls -l demo.sh
chown dev2 demo.sh ; ls -l demo.sh
```

- `chown <user>[:<group>] <대상>` : 소유자(와 그룹) 변경 — **root 만** 가능 (일반 사용자는 자기 파일도 남에게 못 넘김)
- `chown :<group>` (또는 `chown .<group>`) : 그룹만 변경 = `chgrp`
- `-R` : 하위 재귀 / `-h` : 심볼릭 링크 **자체**의 소유자 변경 (기본은 링크가 가리키는 파일)
- `--reference=<파일>` : 다른 파일의 소유자·그룹을 복사
- `--from=<현재소유자>[:<현재그룹>]` : 현재 소유자가 일치하는 대상만 변경 — 대량 변경 시 사고 방지
- `chgrp <group> <대상>` : 그룹만 변경. 일반 사용자는 **자기가 속한 그룹으로만** 변경 가능
- 소유자·그룹을 바꾸면 **SetUID·SetGID 비트가 자동 제거**됨 (권한 상승 악용 방지) → 변경 후 `chmod` 재적용 필요

**검증**

```bash
ls -l demo.sh
```

```text
-rwxr-xr-x. 1 dev1 root     0 ... demo.sh
-rwxr-xr-x. 1 dev1 devteam  0 ... demo.sh
-rwxr-xr-x. 1 dev1 opsteam  0 ... demo.sh
-rwsr-xr-x. 1 dev1 devteam  0 ... demo.sh     ← chmod 4755 직후
-rwxr-xr-x. 1 dev2 devteam  0 ... demo.sh     ← chown 후 SetUID 소실
```

> 📝 **시험 포인트**: "`/var/www/html` 이하 소유자·그룹을 apache 로 재귀 변경" = `chown -R apache:apache /var/www/html` (실기 R02-02). `chown user1:group1 file` 의 콜론 표기와 `chgrp` 의 역할 구분이 기본 문항.

---

## 9. 특수 권한 — SetUID · SetGID · Sticky Bit

### 9-1. 3종 요약과 실물 확인

> **상황**: 시스템에 이미 설치된 실물로 세 비트를 확인한다. 이론표만 외우면 대문자 `S`·`T` 문제에서 흔들리므로 실제 `ls -l` 출력과 맞춰 둔다.

```bash
ls -l  /usr/bin/passwd /usr/bin/su /usr/bin/mount /usr/bin/sudo
find /usr/bin /usr/sbin -perm -4000 -type f 2>/dev/null | head -8
find /usr/bin /usr/sbin -perm -2000 -type f 2>/dev/null | head -5
ls -l  $(find /usr/bin -perm -2000 -type f 2>/dev/null | head -2)
ls -ld /tmp /var/tmp /dev/shm
stat -c '%a %A %U %G %n' /usr/bin/passwd /tmp
```

| 구분 | 8진수 | 표기 위치 | **파일**에서의 효과 | **디렉터리**에서의 효과 |
| --- | --- | --- | --- | --- |
| SetUID | `4000` | 소유자 `x` 자리 → `s` | 실행 시 **파일 소유자** 권한(EUID)으로 동작 | 리눅스에서는 효과 없음 |
| SetGID | `2000` | 그룹 `x` 자리 → `s` | 실행 시 **파일 그룹** 권한(EGID)으로 동작 | 새로 생기는 파일·하위 디렉터리가 **디렉터리의 그룹을 상속**, 하위 디렉터리는 SetGID 도 상속 |
| Sticky Bit | `1000` | 기타 `x` 자리 → `t` | 현대 리눅스에서는 무시 | 디렉터리 안의 파일 삭제·이름변경을 **파일 소유자·디렉터리 소유자·root** 로 제한 |

- `/usr/bin/passwd` = `4755` — 일반 사용자가 권한 `000` 인 `/etc/shadow` 를 고칠 수 있는 이유
- 실행 중에는 실제 UID(RUID)와 유효 UID(EUID)가 분리됨 — RUID 는 호출자, EUID 는 파일 소유자
- **스크립트(`#!` 파일)에는 SetUID 가 적용되지 않음** — 커널이 무시 (경쟁 조건 악용 방지)
- `mount -o nosuid` 로 마운트한 파일시스템에서는 SetUID/SetGID 가 무효 → Part 05 의 `/etc/fstab` 실습
- 지정 방법 : `chmod 4755 <파일>` · `chmod u+s <파일>` / `chmod 2775 <디렉터리>` · `chmod g+s` / `chmod 1777 <디렉터리>` · `chmod +t`

**검증**

```bash
ls -l /usr/bin/passwd ; ls -ld /tmp ; stat -c '%a %A %n' /usr/bin/passwd /tmp
```

```text
-rwsr-xr-x. 1 root root ... /usr/bin/passwd
-rwsr-xr-x. 1 root root ... /usr/bin/su
-rwsr-xr-x. 1 root root ... /usr/bin/mount
/usr/bin/passwd
/usr/bin/su
...
-rwxr-sr-x. 1 root tty  ... /usr/bin/write
drwxrwxrwt. ... /tmp
drwxrwxrwt. ... /var/tmp
drwxrwxrwt. ... /dev/shm
4755 -rwsr-xr-x /usr/bin/passwd
1777 drwxrwxrwt /tmp
```

- SetUID/SetGID 목록은 설치 패키지에 따라 달라짐 — 위 목록은 예시이며 `find` 결과를 기준으로 볼 것

> 📝 **시험 포인트**: SetUID `4000`·SetGID `2000`·Sticky `1000` 의 8진수와 효과 서술이 실기 R02-13 · R01-15. `/usr/bin/passwd` 가 SetUID 인 이유(= `/etc/shadow` 접근)는 필기 R09-07 · R09-09 단골.

### 9-2. 대문자 `S` · `T` — 실행 권한 없는 특수 권한

> **상황**: 8진수만 보고 답하면 틀리는 유형. 실행 비트 유무에 따라 표기가 대소문자로 갈리는 것을 직접 만들어 확인한다.

```bash
touch /tmp/lab-s
chmod 4644 /tmp/lab-s ; ls -l /tmp/lab-s        # 소유자 x 없음 → S
chmod 4744 /tmp/lab-s ; ls -l /tmp/lab-s        # 소유자 x 있음 → s
chmod 2644 /tmp/lab-s ; ls -l /tmp/lab-s        # 그룹 x 없음 → S
chmod 2654 /tmp/lab-s ; ls -l /tmp/lab-s        # 그룹 x 있음 → s
chmod 6755 /tmp/lab-s ; ls -l /tmp/lab-s        # SetUID+SetGID 동시
mkdir -p /tmp/lab-t
chmod 1666 /tmp/lab-t ; ls -ld /tmp/lab-t       # 기타 x 없음 → T
chmod 1777 /tmp/lab-t ; ls -ld /tmp/lab-t       # 기타 x 있음 → t
chmod 7777 /tmp/lab-t ; ls -ld /tmp/lab-t       # 3종 동시
rm -f /tmp/lab-s ; rmdir /tmp/lab-t
```

| 표기 | 위치 | 의미 |
| --- | --- | --- |
| `s` | 소유자 `x` 자리 | SetUID + 소유자 실행 권한 있음 → 정상 동작 |
| `S` | 소유자 `x` 자리 | SetUID 는 설정됐으나 **실행 권한 없음** → 실질적으로 무의미 |
| `s` | 그룹 `x` 자리 | SetGID + 그룹 실행 권한 있음 |
| `S` | 그룹 `x` 자리 | SetGID 만 있음 — 디렉터리에서는 **그룹 상속은 여전히 동작** (그룹 진입만 불가) |
| `t` | 기타 `x` 자리 | Sticky + 기타 실행 권한 있음 → `/tmp` 형태 |
| `T` | 기타 `x` 자리 | Sticky 만 있고 기타 진입 불가 |

- 8진수 → 문자 변환 연습 : `4644` = `-rwSr--r--`, `2654` = `-rw-r-sr--`, `6755` = `-rwsr-sr-x`, `7777` = `drwsrwsrwt`

**검증**

```bash
for m in 4644 4744 2644 2654 6755; do touch /tmp/lab-s; chmod $m /tmp/lab-s; \
  printf '%s -> %s\n' "$m" "$(stat -c %A /tmp/lab-s)"; done; rm -f /tmp/lab-s
```

```text
4644 -> -rwSr--r--
4744 -> -rwsr--r--
2644 -> -rw-r-Sr--
2654 -> -rw-r-sr--
6755 -> -rwsr-sr-x
```

> 📝 **시험 포인트**: "실행 권한 없이 특수 권한만 있으면 **대문자**" — 필기 보기에서 `-rwSr--r--` 를 주고 8진수를 묻거나 그 반대로 출제. `S`/`T` 는 설정 실수 신호로 침해 점검에서도 확인 대상.

### 9-3. 팀 공유 디렉터리 `/srv/devteam` — SetGID 그룹 상속

> **상황**: 개발팀이 함께 쓰는 디렉터리를 만든다. 누가 만들든 그룹이 `devteam` 으로 고정되어야 서로 읽고 쓸 수 있다. 이 디렉터리는 [[04-file-text-shell]] 의 텍스트 처리 실습장으로 그대로 쓰인다.

```bash
mkdir -p /srv/devteam
chgrp devteam /srv/devteam
chmod 2770 /srv/devteam
ls -ld /srv/devteam
su - dev1 -c 'umask; touch /srv/devteam/from-dev1.txt; mkdir -p /srv/devteam/sub1'
su - dev2 -c 'touch /srv/devteam/from-dev2.txt; echo "dev2 wrote" >> /srv/devteam/from-dev1.txt'
ls -l  /srv/devteam
ls -ld /srv/devteam/sub1
su - ops1 -c 'ls /srv/devteam' ; echo "rc=$?"       # opsteam 은 기타(other) → 차단
```

- `chgrp devteam /srv/devteam` : 디렉터리 그룹을 팀 그룹으로
- `chmod 2770` : SetGID(2) + 소유자 `rwx` + 그룹 `rwx` + 기타 차단
- SetGID 가 없으면 새 파일의 그룹은 **생성자의 1차 그룹** — ops1(1차 opsteam)이 참여하는 순간 어긋남
- 하위 디렉터리는 그룹뿐 아니라 **SetGID 비트까지 상속** → 몇 단계를 내려가도 유지
- 그룹 쓰기가 실제로 되려면 파일 권한에도 `g+w` 가 필요 → 8-3 에서 devteam 구성원 umask 를 `002` 로 맞춘 이유. umask 가 `022` 였다면 파일이 `644` 로 생겨 동료가 수정 불가 → 10절의 **기본 ACL** 로도 해결 가능

**검증**

```bash
ls -ld /srv/devteam ; ls -l /srv/devteam ; ls -ld /srv/devteam/sub1
```

```text
drwxrws---. 2 root devteam  6 ... /srv/devteam
0002
total 4
-rw-rw-r--. 1 dev1 devteam 11 ... from-dev1.txt
-rw-rw-r--. 1 dev2 devteam  0 ... from-dev2.txt
drwxrwsr-x. 2 dev1 devteam  6 ... sub1
ls: cannot open directory '/srv/devteam': Permission denied
rc=2
```

- `sub1` 의 그룹이 `dev1` 이 아니라 **`devteam`**, 그리고 그룹 자리에 `s` 가 다시 붙은 것이 상속의 증거

> 📝 **시험 포인트**: "공유 디렉터리에 새로 생기는 파일이 상위 디렉터리 그룹을 상속" = **SetGID** (필기 R02-27 · R07-42 · R09-32). 구성 명령 조합은 `chgrp devteam /srv/project` 후 `chmod 2770` (필기 R06-24).

### 9-4. `/srv/dropbox` — Sticky Bit 로 남의 파일 삭제 차단

> **상황**: 부서 간 파일 전달용으로 누구나 쓸 수 있는 디렉터리를 만들되, 남이 올린 파일을 지우지 못하게 한다. `/tmp` 가 `1777` 인 이유를 재현한다.

```bash
mkdir -p /srv/dropbox
chmod 1777 /srv/dropbox
ls -ld /srv/dropbox
su - dev1 -c 'echo "dev1 file" > /srv/dropbox/dev1.txt'
ls -l /srv/dropbox
su - dev2 -c 'cat /srv/dropbox/dev1.txt'                  # 읽기는 가능
su - dev2 -c 'rm -f /srv/dropbox/dev1.txt' ; echo "rc=$?" # 삭제는 거부
su - dev1 -c 'rm -f /srv/dropbox/dev1.txt' ; echo "rc=$?" # 소유자는 가능
```

Sticky 를 뗀 상태와 비교:

⚠️ 아래는 남의 파일이 실제로 삭제되는 상태를 만든다. 확인 후 즉시 `1777` 로 되돌린다.

```bash
chmod 0777 /srv/dropbox
su - dev1 -c 'echo x > /srv/dropbox/dev1.txt'
su - dev2 -c 'rm -f /srv/dropbox/dev1.txt' ; echo "rc=$?"   # 삭제됨
chmod 1777 /srv/dropbox ; ls -ld /srv/dropbox
```

- 파일 **삭제 권한**은 파일 자체가 아니라 **디렉터리의 `w`** 가 결정 — Sticky 는 그 예외를 만드는 장치
- 삭제·이름변경이 가능한 주체 : 파일 소유자 · 디렉터리 소유자 · root
- `chmod +t` · `chmod o+t` · `chmod 1777` 모두 동일

**검증**

```bash
ls -ld /srv/dropbox
```

```text
drwxrwxrwt. 2 root root 6 ... /srv/dropbox
-rw-rw-r--. 1 dev1 devteam 10 ... dev1.txt
dev1 file
rm: cannot remove '/srv/dropbox/dev1.txt': Operation not permitted
rc=1
rc=0
```

> 📝 **시험 포인트**: `/tmp` 의 `drwxrwxrwt` 마지막 `t` 의 의미·목적 서술이 실기 R01-15. "모두 파일 생성 가능하되 자기 파일만 삭제" = Sticky Bit (필기 R04-64 · R09-33).

### 9-5. `find -perm` — 정확히 · 이상 · 하나라도

> **상황**: 침해 점검 표준 명령을 익히고, 헷갈리는 세 가지 `-perm` 표기를 같은 디렉터리에서 비교한다.

```bash
find / -perm -4000 -type f 2>/dev/null | head
find / -perm -4000 -type f 2>/dev/null | wc -l
find / -perm -2000 -type f 2>/dev/null | wc -l
find / -perm /6000 -type f 2>/dev/null | wc -l
find / -perm -1000 -type d 2>/dev/null | head -3
mkdir -p /tmp/permlab && cd /tmp/permlab
touch f600 f644 f664 f755
chmod 600 f600 ; chmod 644 f644 ; chmod 664 f664 ; chmod 755 f755
echo "== -perm 644 (정확히)" ; find . -maxdepth 1 -type f -perm 644
echo "== -perm -644 (모두 포함)" ; find . -maxdepth 1 -type f -perm -644
echo "== -perm /644 (하나라도)" ; find . -maxdepth 1 -type f -perm /644
```

| 표기 | 판정식 | 의미 | 위 예에서 매칭 |
| --- | --- | --- | --- |
| `-perm 644` | `mode == 644` | 권한이 **정확히** 644 | `f644` |
| `-perm -644` | `mode & 644 == 644` | 644 의 비트를 **모두 포함** | `f644` `f664` `f755` |
| `-perm /644` | `mode & 644 != 0` | 644 의 비트 중 **하나라도** 포함 | 4개 전부 |
| `-perm -4000` | SetUID 비트 보유 | 권한 상승 점검 표준 | — |
| `-perm /6000` | SetUID **또는** SetGID | 특수 권한 전수 조사 | — |

- 구형 `+644` 표기는 `/644` 로 대체됨 (GNU find 4.5.12 이후 제거)
- `-type f` : 일반 파일만 (**f**ile) — 디렉터리·장치 제외
- `2>/dev/null` : `/proc`·`/run` 등 접근 불가 경로의 오류 메시지 억제
- 기준선 저장 : `find / -perm -4000 -type f 2>/dev/null | sort > /root/suid-baseline.txt` → 이후 `diff` 로 신규 SetUID 탐지 (Part 10)

**검증**

```bash
cd /tmp/permlab
find . -maxdepth 1 -type f -perm 644 ; echo '---' ; find . -maxdepth 1 -type f -perm -644
```

```text
== -perm 644 (정확히)
./f644
== -perm -644 (모두 포함)
./f644
./f664
./f755
== -perm /644 (하나라도)
./f600
./f644
./f664
./f755
```

> 📝 **시험 포인트**: `find / -perm -4000 -type f 2>/dev/null` 는 통째 암기 대상 (실기 R02-01 · R05-03, 필기 R04-65 · R09-31). `-`(모두 포함)와 `/`(하나라도)의 차이가 오답 유도 포인트 — "권한이 정확히 4000 인 파일"은 `-perm 4000`.

---

## 10. ACL — 세밀한 접근 제어

### 10-1. ACL 지원 확인과 `getfacl` 출력 해석

> **상황**: 9-3 에서 만든 `/srv/devteam` 은 opsteam 이 전혀 볼 수 없다. 운영자에게 읽기만 열어 주려면 그룹을 하나 더 붙이는 대신 ACL 을 쓴다. 먼저 파일시스템이 ACL 을 지원하는지 확인한다.

```bash
rpm -q acl
findmnt -no SOURCE,FSTYPE,OPTIONS /
findmnt -no TARGET,FSTYPE / /srv 2>/dev/null
getfacl /srv/devteam
getfacl -c /srv/devteam            # 헤더 3줄 생략
```

- RHEL 9 의 `xfs`·`ext4` 는 ACL 을 **기본 지원** — `mount -o acl` 을 따로 줄 필요 없음
- ext4 는 `tune2fs -l <장치> | grep 'Default mount options'` 에 `user_xattr acl` 로 표시됨. xfs 는 항상 활성
- `acl` 패키지가 `getfacl`·`setfacl` 제공 (RHEL 9 기본 설치)
- ACL 은 **확장 속성(xattr)** 에 저장 → `getfattr -m . -d <파일>` 로도 확인 가능
- `getfacl -c` : 헤더(`# file:` `# owner:` `# group:`) 생략 (**c** = omit header)
- `getfacl -R` : 재귀 / `-p` : 절대경로 유지 / `-t` : 표 형식 출력

**getfacl 출력 행 해석**

| 행 | 의미 |
| --- | --- |
| `# file:` `# owner:` `# group:` | 대상 경로·소유자·소유 그룹 |
| `# flags: -s-` | 특수 권한 — 순서는 `setuid`·`setgid`·`sticky` (`-s-` = SetGID) |
| `user::rwx` | **소유자** 권한 (`ls -l` 첫 3자리) |
| `user:<이름>:rwx` | **명명 사용자** ACL 항목 |
| `group::rwx` | **소유 그룹** 권한 |
| `group:<이름>:rwx` | **명명 그룹** ACL 항목 |
| `mask::rwx` | 소유자·other 를 제외한 모든 항목의 **상한** |
| `other::---` | 기타 권한 |
| `default:` 접두 | **기본 ACL** — 하위에 상속될 초기값 |

**검증**

```bash
getfacl /srv/devteam
```

```text
getfacl: Removing leading '/' from absolute path names
# file: srv/devteam
# owner: root
# group: devteam
# flags: -s-
user::rwx
group::rwx
other::---
```

> 📝 **시험 포인트**: `ls -l` 권한열 끝의 `+` = ACL 존재 표시. `getfacl` 출력에서 `# flags: -s-` 는 SetGID — 9-3 에서 건 `2770` 이 그대로 보인다.

### 10-2. `setfacl -m` — 사용자·그룹 항목 추가

> **상황**: 운영자 ops1 에게 `/srv/devteam` **읽기·진입만** 허용하고, opsteam 그룹 전체에는 쓰기까지 준다. 명명 사용자 항목과 그룹 항목이 충돌할 때 무엇이 이기는지도 확인한다.

```bash
setfacl -m u:ops1:rx /srv/devteam
setfacl -m g:opsteam:rwx /srv/devteam
getfacl /srv/devteam
ls -ld /srv/devteam
su - ops1 -c 'ls /srv/devteam'                       # 진입·목록 가능
su - ops1 -c 'touch /srv/devteam/ops1.txt' ; echo "rc=$?"   # 생성은 거부
su - dev2 -c 'touch /srv/devteam/dev2-again.txt' ; echo "rc=$?"
```

- `-m` : 항목 추가·수정 (**m**odify)
- 항목 형식 : `u[ser]:<이름|UID>:<권한>` · `g[roup]:<이름|GID>:<권한>` · `m[ask]::<권한>` · `o[ther]::<권한>`
- 이름을 비우면 소유자/소유그룹 항목 (`u::rwx`, `g::r-x`)
- 여러 항목은 콤마로 연결 : `setfacl -m u:ops1:rx,g:opsteam:rwx <경로>`
- ACL 이 붙으면 `ls -l` 권한열 끝에 **`+`**
- 권한 판정 순서 : 소유자 → **명명 사용자** → 소유 그룹·명명 그룹(가장 유리한 것) → other. **명명 사용자 항목이 걸리면 그룹 항목은 보지 않음**

**검증**

```bash
getfacl -c /srv/devteam ; ls -ld /srv/devteam
```

```text
user::rwx
user:ops1:r-x
group::rwx
group:opsteam:rwx
mask::rwx
other::---

drwxrws---+ 2 root devteam ... /srv/devteam
from-dev1.txt  from-dev2.txt  sub1
touch: cannot touch '/srv/devteam/ops1.txt': Permission denied
rc=1
rc=0
```

- ops1 은 opsteam(rwx) 소속이지만 **`user:ops1:r-x` 항목이 먼저 걸려** 쓰기 불가 — 개인 항목이 그룹 항목보다 우선

> 📝 **시험 포인트**: `setfacl -m u:lee:rw data.txt` 형태가 필기 R01-31 · R02-29 정답 형식. `getfacl -m …` 은 존재하지 않는 조합(오답 보기), `setfacl -x` 는 삭제.

### 10-3. `mask` 와 실효 권한 — `chmod g=` 의 함정

> **상황**: ACL 을 걸어 둔 파일에 `chmod` 를 쓰면 예상과 다른 곳이 바뀐다. 필기에서 `#effective:` 해석 문제로 매 회차 나오는 지점을 직접 재현한다.

```bash
cd /srv/devteam
getfacl -c from-dev1.txt
setfacl -m u:ops1:rwx from-dev1.txt
getfacl -c from-dev1.txt
chmod g=r from-dev1.txt                  # ← ACL 이 있으면 group:: 이 아니라 mask 를 바꾼다
getfacl -c from-dev1.txt
ls -l from-dev1.txt
setfacl -m m::rwx from-dev1.txt          # mask 복구
getfacl -c from-dev1.txt
```

- `mask` : 소유자(`user::`)와 `other::` 를 **제외한** 모든 항목의 **상한**. 실효 권한 = 항목 권한 `AND` mask
- `#effective:` 주석 = mask 로 깎인 뒤의 실제 권한
- ACL 이 설정된 파일에서 `ls -l` 의 **그룹 자리는 `group::` 이 아니라 `mask`** 를 표시
- `chmod g=<권한>` 은 ACL 이 있으면 **mask** 를 변경 — 그래서 ACL 이 조용히 무력화되는 사고가 발생
- `setfacl -m u:…` 실행 시 mask 는 필요한 만큼 **자동 확장**됨. 자동 재계산을 막으려면 `-n`(`--no-mask`)
- mask 직접 지정 : `setfacl -m m::rwx <대상>`

**검증**

```bash
getfacl -c from-dev1.txt ; ls -l from-dev1.txt
```

```text
user::rw-
user:ops1:rwx
group::rw-
mask::rwx
other::r--

user::rw-
user:ops1:rwx			#effective:r--
group::rw-			#effective:r--
mask::r--
other::r--

-rw-r--r--+ 1 dev1 devteam ... from-dev1.txt
```

> 📝 **시험 포인트**: `user:kim:rwx  #effective:r--` → **mask 가 `r--` 이므로 실제 권한은 `r--`** (필기 R10-26). `mask` 는 소유자 권한에는 영향을 주지 않는다는 점(필기 R10-26 오답 보기)과, `getfacl` 출력에서 `other::---` 이면 기타는 읽기 불가(필기 R02-28 · R08-24)를 함께 확인.

### 10-4. 기본(default) ACL — 하위 상속

> **상황**: 앞으로 `/srv/devteam` 에 만들어질 파일에도 opsteam 읽기 권한이 자동으로 붙게 한다. SetGID 가 **그룹**을 상속시킨다면 기본 ACL 은 **권한**을 상속시킨다.

```bash
setfacl -d -m u::rwx,g::rwx,o::---,g:opsteam:r-x /srv/devteam
getfacl /srv/devteam
su - dev1 -c 'touch /srv/devteam/inherit.txt; mkdir -p /srv/devteam/sub2'
getfacl -c /srv/devteam/inherit.txt
getfacl -c /srv/devteam/sub2 | head -12
ls -l /srv/devteam/inherit.txt ; ls -ld /srv/devteam/sub2
```

- `-d` : 기본 ACL 지정 (**d**efault) — **디렉터리에만** 설정 가능
- 기존 파일에는 영향 없음. **이후 생성되는** 항목의 초기 ACL 이 됨
- 하위 **디렉터리**는 접근 ACL + 기본 ACL 을 모두 상속, 하위 **파일**은 접근 ACL 만 상속
- 기본 ACL 이 있으면 그 디렉터리에서는 **umask 가 무시**됨 (기본 ACL 이 우선)
- `u::`·`g::`·`o::` 세 항목은 기본 ACL 의 필수 구성 — 빠뜨리면 자동 생성됨
- `-k` (`--remove-default`) : 기본 ACL 만 제거
- 기존 파일에도 소급하려면 `setfacl -R -m …` 로 접근 ACL 을 별도 적용

**검증**

```bash
getfacl -c /srv/devteam | grep '^default:' ; getfacl -c /srv/devteam/inherit.txt
```

```text
default:user::rwx
default:group::rwx
default:group:opsteam:r-x
default:mask::rwx
default:other::---

# (inherit.txt)
user::rw-
group::rwx			#effective:rw-
group:opsteam:r-x		#effective:r--
mask::rw-
other::---
```

- 파일은 생성 요청 권한(`0666`)과 기본 ACL 의 교집합으로 결정되어 `x` 가 붙지 않음 → `group:opsteam` 의 실효 권한이 `r--`

> 📝 **시험 포인트**: "디렉터리에 `-d` 로 지정하면 하위 생성 파일에 상속" 이 기본 ACL 의 정의. SetGID(그룹 상속)와 기본 ACL(권한 상속)은 **다른 장치** — 서술형에서 둘을 섞어 쓰면 감점.

### 10-5. 항목 삭제 — `-x` · `-b` · `-k` · `-R` · `-M`

> **상황**: 권한 회수와 대량 적용 방법을 정리한다. 파일에 규칙을 적어 두고 일괄 적용하는 방식은 서버가 늘어날 때 유용하다.

```bash
cd /srv/devteam
setfacl -x u:ops1 from-dev1.txt
getfacl -c from-dev1.txt
setfacl -b from-dev1.txt                       # 확장 ACL 전체 제거
ls -l from-dev1.txt                            # '+' 사라짐
cat > /root/acl-rules.txt <<'EOF'
u:ops1:r-x
g:opsteam:r-x
EOF
setfacl -R -M /root/acl-rules.txt /srv/devteam/sub1
getfacl -c /srv/devteam/sub1
setfacl -R -b /srv/devteam/sub1
ls -ld /srv/devteam/sub1
```

- `-x <항목>` : 지정 항목 삭제 — 권한 부분은 **생략**해서 적음 (`u:ops1`, `g:opsteam`)
- `-b` (`--remove-all`) : 확장 ACL 전체 삭제 — 기본 3항목(`user::` `group::` `other::`)만 남고 `+` 표시가 사라짐
- `-k` (`--remove-default`) : 기본 ACL 만 삭제
- `-R` : 하위 재귀 (**R**ecursive)
- `-M <파일>` : 파일에 적힌 항목들을 일괄 `-m` (**M**odify from file) / `-X <파일>` : 일괄 삭제
- `--set` / `--set-file` : 기존 ACL 을 **대체** (`-m` 은 병합)
- `-n` : mask 자동 재계산 안 함 / `--test` : 실제 적용 없이 결과만 출력
- `-` 를 파일 인자로 주면 표준입력에서 규칙을 읽음

**검증**

```bash
ls -l /srv/devteam/from-dev1.txt ; ls -ld /srv/devteam/sub1
```

```text
user::rw-
group::rw-
mask::rw-
other::r--

-rw-r--r--. 1 dev1 devteam ... from-dev1.txt
drwxr-sr-x. 2 dev1 devteam ... sub1
```

> 📝 **시험 포인트**: `setfacl -x u:user2 file`(항목 삭제)와 `setfacl -b file`(전체 삭제) 구분. `-b` 뒤에 항목을 적는 `setfacl -b u:tom:rw data.txt` 는 오답 보기 (필기 R02-29).

### 10-6. ACL 백업·복원

> **상황**: 권한 구조를 통째로 보관해 두고, 실수로 지웠을 때 되살린다. Part 12 백업 실습에서 `tar --acls` 와 연결된다.

```bash
cd /
getfacl -R /srv/devteam > /root/acl-devteam.bak
head -14 /root/acl-devteam.bak
wc -l /root/acl-devteam.bak
setfacl -R -b /srv/devteam                     # 일부러 전부 제거
getfacl -c /srv/devteam ; ls -ld /srv/devteam
setfacl --restore=/root/acl-devteam.bak        # 복원
getfacl -c /srv/devteam ; ls -ld /srv/devteam
```

- `getfacl -R <경로> > <파일>` : 하위 전체 ACL 을 텍스트로 덤프. 선행 `/` 가 제거되므로 **`/` 디렉터리에서 복원**해야 경로가 맞음
- `--restore=<파일>` : 덤프 파일 그대로 복원 — 소유자·그룹·권한 비트까지 함께 되돌림
- `-p` (`--absolute-names`) : 선행 `/` 를 제거하지 않음 → 절대경로 그대로 저장
- 아카이브에서 ACL 보존 : `tar --acls -cvf …` · `rsync -A` · `cp -a`(`--preserve=all`) → Part 12
- `cp` 는 기본적으로 ACL 을 보존하지 않음 — `cp -p` 로도 부족하고 `--preserve=mode,ownership,timestamps,xattr` 필요

**검증**

```bash
getfacl -c /srv/devteam | grep -E 'ops1|opsteam|default' ; ls -ld /srv/devteam
```

```text
# file: srv/devteam
# owner: root
# group: devteam
# flags: -s-
user::rwx
user:ops1:r-x
group::rwx
group:opsteam:rwx
mask::rwx
other::---
default:user::rwx
...

drwxrws---+ 2 root devteam ... /srv/devteam
```

> 📝 **시험 포인트**: ACL 백업·복원 자체는 출제 빈도가 낮으나, "`cp`·`tar` 로 옮기면 ACL 이 사라지는 이유" 는 실무형 서술 소재. `getfacl -R` → `setfacl --restore` 쌍으로 기억.

---

## 11. 파일 속성 — chattr / lsattr

### 11-1. `lsattr` — 현재 속성 조회

> **상황**: 권한(`rwx`)·ACL 과는 **별개 계층**인 파일시스템 속성을 본다. `ls -l` 로는 보이지 않아 침해 점검에서 놓치기 쉬운 지점이다.

```bash
findmnt -no FSTYPE /
lsattr /etc/passwd /etc/shadow
lsattr -d /etc /root /srv/devteam
lsattr /srv/devteam/from-dev2.txt
lsattr -R /srv/devteam 2>/dev/null | head -5
```

- `lsattr` : 파일 속성 조회 (**l**i**s**t **attr**ibutes)
- `-d` : 디렉터리 **자체**의 속성 (내용 나열 대신) (**d**irectory)
- `-a` : 숨김 파일 포함 (**a**ll) / `-R` : 하위 재귀 / `-l` : 속성 이름을 길게 표기
- 표시 자릿수·문자는 파일시스템과 `e2fsprogs` 버전에 따라 다름
- `e`(extent) 는 **ext4 전용** — Rocky 9 기본 루트인 xfs 에서는 나타나지 않음
- 속성은 파일시스템 메타데이터에 저장되므로 `cp` 로 복사하면 **따라가지 않음**

**검증**

```bash
lsattr /etc/passwd
```

```text
-------------------- /etc/passwd          # xfs (Rocky 9 기본 루트)
--------------e----- /etc/passwd          # ext4 인 경우 e = extent
```

> 📝 **시험 포인트**: "설정된 속성 확인 명령" = `lsattr` (필기 R03-26 · R09-35 보기). 권한·소유자와 무관한 별도 계층이라는 점이 핵심.

### 11-2. `chattr +i` — 불변(immutable) 속성

> **상황**: 설정 파일이 알 수 없는 원인으로 계속 바뀔 때 원인 분석 동안 완전히 고정한다. root 조차 막힌다는 것을 직접 확인한다.

⚠️ `/etc/passwd` 를 잠그면 `useradd`·`usermod`·`passwd` 가 전부 실패한다. **해제까지 한 번에** 수행하고, 다른 작업과 겹치지 않을 때 실행할 것.

```bash
lsattr /etc/passwd                       # 변경 전 상태 확인
cp -p /etc/passwd /root/passwd.orig      # 안전망
chattr +i /etc/passwd
lsattr /etc/passwd
echo "x" >> /etc/passwd    ; echo "rc=$?"
useradd -M labtest         ; echo "rc=$?"
rm -f /etc/passwd          ; echo "rc=$?"
chmod 600 /etc/passwd      ; echo "rc=$?"
chattr -i /etc/passwd                    # 반드시 해제
lsattr /etc/passwd
useradd -M labtest && userdel labtest && echo "해제 후 정상 동작"
```

- `+i` : immutable — 수정·삭제·이름변경·하드링크 생성·`chmod`/`chown` 전부 차단. **root 도 불가**, 먼저 `chattr -i` 필요
- 설정·해제에 필요한 권한 : `CAP_LINUX_IMMUTABLE` (실질적으로 root)
- 파일 권한·SELinux 와 **독립적인 계층** — `ls -l` 로는 보이지 않음
- 침해 대응(설정 파일 고정)과 백도어 은닉(공격자가 자기 파일 고정) 양쪽에 쓰이는 양날 속성 → 점검 시 `lsattr -R` 필수
- `chattr -R` : 하위 재귀 / `chattr -V` : 상세 출력

**검증**

```bash
lsattr /etc/passwd ; echo "x" >> /etc/passwd ; echo "rc=$?"
```

```text
----i--------------- /etc/passwd
-bash: /etc/passwd: Operation not permitted
rc=1
useradd: failure while writing changes to /etc/passwd
rc=1
rm: cannot remove '/etc/passwd': Operation not permitted
rc=1
chmod: changing permissions of '/etc/passwd': Operation not permitted
rc=1
-------------------- /etc/passwd
해제 후 정상 동작
```

- `useradd` 의 정확한 문구는 `shadow-utils` 버전에 따라 `cannot lock /etc/passwd` 등으로 다를 수 있음 — 확실한 증거는 `echo >>` 의 `Operation not permitted`

> 📝 **시험 포인트**: `chattr +i /etc/passwd` 는 실기 R05-10 단답. "root 라도 속성을 해제하기 전에는 수정·삭제·이름 변경이 불가능" 이 정답 서술 (필기 R09-35 · R06-61). `chmod 000` 은 root 에게 무력하므로 오답.

### 11-3. `chattr +a` — 추가 전용(append-only) 로그

> **상황**: 애플리케이션 로그를 변조로부터 보호한다. 이어쓰기는 되고 덮어쓰기·삭제는 안 되는 상태를 만든다.

```bash
touch /var/log/app.log
chattr +a /var/log/app.log
lsattr /var/log/app.log
echo "2026-09-04 app started" >> /var/log/app.log ; echo "rc=$?"
echo "overwrite"               > /var/log/app.log ; echo "rc=$?"
truncate -s 0 /var/log/app.log                    ; echo "rc=$?"
mv /var/log/app.log /tmp/                         ; echo "rc=$?"
rm -f /var/log/app.log                            ; echo "rc=$?"
cat /var/log/app.log
```

- `+a` : append-only — `O_APPEND` 플래그로만 열림. **이어쓰기(`>>`)만 허용**, 덮어쓰기(`>`)·잘라내기·삭제·이름변경 차단
- 로그 변조 방지의 기본 수단 — 공격자가 흔적을 지우지 못하게 함
- `logrotate` 는 회전 시 이름변경·재생성이 필요하므로 `copytruncate` 를 쓰거나 회전 전후로 `chattr -a`/`+a` 를 실행해야 함 (Part 07)
- 해제 : `chattr -a <파일>`

**검증**

```bash
lsattr /var/log/app.log ; cat /var/log/app.log
```

```text
-----a-------------- /var/log/app.log
rc=0
-bash: /var/log/app.log: Operation not permitted
rc=1
truncate: cannot open '/var/log/app.log' for writing: Operation not permitted
rc=1
mv: cannot move '/var/log/app.log' to '/tmp/app.log': Operation not permitted
rc=1
rm: cannot remove '/var/log/app.log': Operation not permitted
rc=1
2026-09-04 app started
```

> 📝 **시험 포인트**: "로그 파일을 추가만 허용하여 변조 방지" = `chattr +a /var/log/app.log` (실기 R04-08). `+i` 와의 차이 — `+i` 는 **이어쓰기도 차단**하므로 활성 로그에는 부적합.

### 11-4. 그 밖의 속성 (참고)

> **상황**: 필기 보기로 등장하는 속성 문자를 정리하고, RHEL 9 에서 실제로 동작하는 것과 이름만 남은 것을 구분한다.

```bash
chattr +A /srv/devteam/from-dev2.txt ; lsattr /srv/devteam/from-dev2.txt
chattr -A /srv/devteam/from-dev2.txt
chattr +d /srv/devteam/from-dev2.txt ; lsattr /srv/devteam/from-dev2.txt
chattr -d /srv/devteam/from-dev2.txt
chattr +c /srv/devteam/from-dev2.txt ; echo "rc=$?"
chattr +S /srv/devteam/from-dev2.txt ; lsattr /srv/devteam/from-dev2.txt
chattr -S /srv/devteam/from-dev2.txt
lsattr /srv/devteam/from-dev2.txt
```

| 속성 | 의미 | 비고 |
| --- | --- | --- |
| `i` | immutable — 수정·삭제·이름변경 전면 차단 | ext4 · xfs |
| `a` | append-only — 이어쓰기만 허용 | ext4 · xfs |
| `A` | 접근 시각(atime) 갱신 안 함 — 디스크 I/O 절감 | ext4 · xfs |
| `d` | `dump` 백업 대상에서 제외 (**d**ump) | ext4 · xfs (Part 12 `dump` 와 연계) |
| `S` | 변경 사항을 즉시 디스크에 기록 (`sync` 마운트와 동일 효과) | ext4 · xfs |
| `e` | extent 사용 — ext4 가 자동 설정, **해제 불가** | ext4 전용 |
| `j` | 데이터까지 저널에 기록 (**j**ournal) | ext3/ext4 |
| `c` | 투명 압축 | ⚠️ ext4 **미구현** → `Operation not supported` |
| `s` | 삭제 시 블록을 0 으로 덮어씀 (secure deletion) | ⚠️ 미구현 |
| `u` | 삭제해도 내용 보존 (undelete) | ⚠️ 미구현 |
| `t` | 테일 병합 안 함 | 일부 파일시스템 |
| `C` | Copy-on-Write 비활성 | btrfs 전용 |

- 이론서에는 `c`·`s`·`u` 가 정상 속성으로 소개되지만 **리눅스 커널에 구현되어 있지 않음** — 시험은 교재 기준으로 답하고, 실습에서는 오류를 확인
- `chattr` 은 **파일시스템 종속** — xfs 는 `i`·`a`·`A`·`d`·`S` 중심으로 지원

**검증**

```bash
chattr +c /srv/devteam/from-dev2.txt ; echo "rc=$?" ; lsattr /srv/devteam/from-dev2.txt
```

```text
-------A------------ /srv/devteam/from-dev2.txt
------d------------- /srv/devteam/from-dev2.txt
chattr: Operation not supported while setting flags on /srv/devteam/from-dev2.txt
rc=1
--S----------------- /srv/devteam/from-dev2.txt
-------------------- /srv/devteam/from-dev2.txt
```

> 📝 **시험 포인트**: `chattr`/`lsattr` 설명 중 **틀린 것** 고르기가 필기 R03-26 유형 — "속성은 `ls -l` 로 확인한다" 같은 보기가 오답. 조회는 반드시 `lsattr`.

---

## 12. 최종 상태 점검

### 12-1. 계정·그룹 일괄 확인

> **상황**: README 2-1 자원 명세대로 만들어졌는지 한 화면에서 대조한다. 이 상태가 Part 05(쿼터)·Part 09(Samba·FTP·메일)의 전제가 된다.

```bash
getent passwd | awk -F: '$3>=1000 && $3<65534 {printf "%-8s uid=%-5s gid=%-5s shell=%s\n", $1,$3,$4,$7}'
getent group | grep -E '^(devteam|opsteam|wheel):'
for u in dev1 dev2 ops1; do echo "== $u"; id $u; chage -l $u | head -3; done
getent passwd guest1 || echo "guest1 : 삭제 완료"
```

- `awk -F: '$3>=1000 && $3<65534'` : 일반 사용자만 (65534 = `nobody` 제외)
- `printf` 로 정렬 출력 — 필드 폭 지정
- `chage -l … | head -3` : 최종 변경일·비밀번호 만료·비활성만 요약

**검증**

```bash
getent passwd | awk -F: '$3>=1000 && $3<65534 {print $1, $3, $4, $7}'
```

```text
admin1   uid=1000  gid=1000  shell=/bin/bash
dev1     uid=2001  gid=2000  shell=/bin/bash
dev2     uid=2002  gid=2000  shell=/bin/bash
ops1     uid=2003  gid=2001  shell=/bin/bash
devteam:x:2000:dev1,dev2
opsteam:x:2001:dev2
wheel:x:10:admin1,ops1
== dev1
uid=2001(dev1) gid=2000(devteam) groups=2000(devteam)
Last password change					: Sep 03, 2026
Password expires					: Dec 02, 2026
...
guest1 : 삭제 완료
```

> 📝 **시험 포인트**: `awk -F: '$3>=1000 {print $1}' /etc/passwd` 는 실기 고정 문항 (R03-02 · R06-05). `id` 출력에서 `gid=` 가 1차, `groups=` 가 전체 (필기 R08-21).

### 12-2. 정책·권한·속성 확인

> **상황**: 4~11절에서 건 설정이 파일에 실제로 남아 있는지 한 번에 확인한다.

```bash
grep -vE '^\s*#|^\s*$' /etc/security/pwquality.conf
grep -E '^auth\s+required\s+pam_wheel' /etc/pam.d/su
tail -6 /etc/security/limits.conf
sudo -l -U ops1 | tail -4
ls -ld /srv/devteam /srv/dropbox
getfacl -c /srv/devteam | head -8
lsattr /var/log/app.log
lsattr /etc/passwd
```

- `grep -vE '^\s*#|^\s*$'` : 주석·빈 줄 제외 = **유효 설정만** 보기
- 설정 파일 확인과 **동작 확인**은 별개 — 12-4 스크립트가 동작까지 검사

**검증**

```bash
ls -ld /srv/devteam /srv/dropbox ; lsattr /var/log/app.log
```

```text
minlen = 10
dcredit = -1
...
auth		required	pam_wheel.so use_uid
dev1     soft    nproc      50
...
    (root) !ALL
    (root) /usr/bin/systemctl restart httpd, ..., /usr/bin/journalctl
drwxrws---+ 2 root devteam ... /srv/devteam
drwxrwxrwt. 2 root root    ... /srv/dropbox
user::rwx
user:ops1:r-x
group::rwx
group:opsteam:rwx
mask::rwx
other::---
-----a-------------- /var/log/app.log
-------------------- /etc/passwd
```

> 📝 **시험 포인트**: `/srv/devteam` 의 `drwxrws---+` 한 줄에 이 파트의 핵심 3개(SetGID `s`, 그룹 `devteam`, ACL `+`)가 모두 들어 있다 — 출력 해석 문제의 압축본.

### 12-3. 자동 점검 스크립트 `/usr/local/bin/check-part03.sh`

> **상황**: 다음에 이 VM 을 다시 열었을 때, 또는 스냅샷을 되돌린 뒤 상태가 유지되는지 한 번에 확인할 수 있게 스크립트로 굳힌다.

```bash
cat > /usr/local/bin/check-part03.sh <<'EOF'
#!/bin/bash
# LAB Part 03 자체 점검 — 계정·그룹·정책·권한 상태를 [OK]/[NG] 로 출력
export LC_ALL=C LANG=C
ok=0; ng=0

chk() {                                  # chk "설명" '판정 명령'
    if eval "$2" >/dev/null 2>&1; then
        printf '[OK] %s\n' "$1"; ok=$((ok+1))
    else
        printf '[NG] %s\n' "$1"; ng=$((ng+1))
    fi
}

echo "== 계정·그룹 =="
chk "그룹 devteam GID 2000"        'test "$(getent group devteam | cut -d: -f3)" = 2000'
chk "그룹 opsteam GID 2001"        'test "$(getent group opsteam | cut -d: -f3)" = 2001'
chk "dev1 UID 2001"                'test "$(id -u dev1)" = 2001'
chk "dev1 1차 그룹 devteam"         'test "$(id -gn dev1)" = devteam'
chk "dev2 보조 그룹 opsteam"        'id -nG dev2 | grep -qw opsteam'
chk "ops1 보조 그룹 wheel"          'id -nG ops1 | grep -qw wheel'
chk "guest1 삭제 완료"              'test -z "$(getent passwd guest1)"'
chk "고아 파일 없음"                 'test -z "$(find /home /srv /var/spool/mail -xdev -nouser 2>/dev/null)"'

echo "== 비밀번호 정책 =="
chk "dev1 비밀번호 설정(PS)"        'test "$(passwd -S dev1 | cut -d" " -f2)" = PS'
chk "dev1 최대 사용일 90"           'chage -l dev1 | grep -q "Maximum number of days between password change.*: 90"'
chk "dev1 계정 만료 2026-12-31"     'chage -l dev1 | grep -q "Account expires.*Dec 31, 2026"'
chk "pwquality minlen=10"          'grep -Eq "^[[:space:]]*minlen[[:space:]]*=[[:space:]]*10" /etc/security/pwquality.conf'

echo "== PAM·자원 제한 =="
chk "pam_wheel 활성(su 제한)"       'grep -Eq "^auth[[:space:]]+required[[:space:]]+pam_wheel" /etc/pam.d/su'
chk "limits.conf dev1 nproc"       'grep -Eq "^dev1[[:space:]]+soft[[:space:]]+nproc" /etc/security/limits.conf'
chk "dev1 nofile soft=1024"        'test "$(su - dev1 -c "ulimit -Sn")" = 1024'

echo "== sudo =="
chk "sudoers 전체 문법 정상"         'visudo -c'
chk "sudoers.d/ops1 권한 0440"      'test "$(stat -c %a /etc/sudoers.d/ops1)" = 440'
chk "ops1 journalctl 허용"          'sudo -l -U ops1 | grep -q journalctl'

echo "== 권한·ACL·속성 =="
chk "/srv/devteam 2770 devteam"    'test "$(stat -c "%a %G" /srv/devteam)" = "2770 devteam"'
chk "/srv/devteam ACL(ops1)"       'getfacl -c /srv/devteam 2>/dev/null | grep -q "^user:ops1:"'
chk "/srv/devteam 기본 ACL"         'getfacl -c /srv/devteam 2>/dev/null | grep -q "^default:"'
chk "/srv/dropbox 1777"            'test "$(stat -c %a /srv/dropbox)" = 1777'
chk "/var/log/app.log 추가전용(a)"  'lsattr /var/log/app.log | cut -d" " -f1 | grep -q a'
chk "/etc/passwd 불변속성 해제"      'lsattr -d /etc/passwd | cut -d" " -f1 | grep -qv i'

printf '\n합계: OK %d / NG %d\n' "$ok" "$ng"
[ "$ng" -eq 0 ]
EOF
chmod 755 /usr/local/bin/check-part03.sh
/usr/local/bin/check-part03.sh ; echo "종료코드=$?"
```

- `eval "$2"` : 문자열로 받은 판정 명령을 실행 — 각 검사를 한 줄로 유지하기 위함
- `test "$(…)" = <값>` : 명령 출력과 기대값 비교
- `getfacl -c` : 헤더 없이 ACL 항목만 → `grep` 판정이 단순해짐
- `lsattr … | cut -d" " -f1` : 속성 문자열만 잘라내 경로에 포함된 글자와의 오탐 방지
- 마지막 `[ "$ng" -eq 0 ]` : NG 가 하나라도 있으면 **종료코드 1** → cron·다른 스크립트에서 성공 여부 판정 가능

**검증**

```bash
/usr/local/bin/check-part03.sh ; echo "종료코드=$?"
```

```text
== 계정·그룹 ==
[OK] 그룹 devteam GID 2000
[OK] 그룹 opsteam GID 2001
[OK] dev1 UID 2001
[OK] dev1 1차 그룹 devteam
[OK] dev2 보조 그룹 opsteam
[OK] ops1 보조 그룹 wheel
[OK] guest1 삭제 완료
[OK] 고아 파일 없음
== 비밀번호 정책 ==
[OK] dev1 비밀번호 설정(PS)
[OK] dev1 최대 사용일 90
[OK] dev1 계정 만료 2026-12-31
[OK] pwquality minlen=10
== PAM·자원 제한 ==
[OK] pam_wheel 활성(su 제한)
[OK] limits.conf dev1 nproc
[OK] dev1 nofile soft=1024
== sudo ==
[OK] sudoers 전체 문법 정상
[OK] sudoers.d/ops1 권한 0440
[OK] ops1 journalctl 허용
== 권한·ACL·속성 ==
[OK] /srv/devteam 2770 devteam
[OK] /srv/devteam ACL(ops1)
[OK] /srv/devteam 기본 ACL
[OK] /srv/dropbox 1777
[OK] /var/log/app.log 추가전용(a)
[OK] /etc/passwd 불변속성 해제

합계: OK 23 / NG 0
종료코드=0
```

> 📝 **시험 포인트**: 실기에서 "설정 후 확인 명령까지 함께 쓰시오" 형태가 나오면 `stat -c %a` · `getent` · `id` · `getfacl` · `lsattr` · `sudo -l -U` 조합이 표준 답안. 스크립트 자체보다 **각 판정 명령 한 줄**을 외워 둘 것.

---

## 체크리스트

| 수행 항목 | 명령 | 확인 방법 | ☐ |
| --- | --- | --- | --- |
| 계정 생성 기본값 파악 | `useradd -D`, `cat /etc/default/useradd` | `diff <(useradd -D) <(grep -v '^#' /etc/default/useradd)` | ☐ |
| UID 범위·비밀번호 기본 정책 | `grep -E 'UID_MIN\|PASS_MAX_DAYS\|HOME_MODE' /etc/login.defs` | `UID_MIN 1000`, `HOME_MODE 0700` 확인 | ☐ |
| skel 템플릿에 팀 alias | `cat >> /etc/skel/.bashrc` | `tail -3 /etc/skel/.bashrc` → dev1 홈에 전파 확인 | ☐ |
| 그룹 생성 (2000/2001) | `groupadd -g 2000 devteam`, `groupadd -g 2001 opsteam` | `getent group devteam opsteam` | ☐ |
| 그룹 이름·GID 변경·삭제 | `groupmod -n`, `groupmod -g`, `groupdel` | `getent group labtmp \|\| echo removed` | ☐ |
| 그룹 구성원·관리자 | `gpasswd -A/-a/-d/-M`, `groups` | `grep devteam /etc/group /etc/gshadow` | ☐ |
| dev1 생성 (옵션 전부 명시) | `useradd -u 2001 -g devteam -c … -s /bin/bash -m dev1` | `id dev1`, `ls -ld /home/dev1`(700) | ☐ |
| dev2·ops1 보조 그룹 | `useradd … -G opsteam dev2`, `useradd … -G wheel ops1` | `id dev2`, `id ops1`, `getent group wheel` | ☐ |
| guest1 만료·비활성 | `useradd -u 2004 -e 2026-12-31 -f 7 -m guest1` | `chage -l guest1`, `grep guest1 /etc/shadow` | ☐ |
| 시스템 계정 비교 | `useradd -r -M -s /sbin/nologin labsvc` → `userdel` | `getent passwd labsvc` UID<1000 | ☐ |
| passwd/shadow 필드 환산 | `getent passwd dev1 \| tr ':' '\n' \| nl`, `date -d @$((18900*86400)) +%F` | 9필드 대응표와 일치 | ☐ |
| 비밀번호 설정·상태 | `passwd dev1`, `passwd -S dev1` | `PS` 상태, `/etc/shadow` 2번 필드 `$6$` | ☐ |
| 잠금·해제·삭제 | `passwd -l/-u/-d guest1` | `passwd -S` → `LK`/`PS`/`NP` | ☐ |
| 변경 강제 | `chage -d 0 dev1` → `su - dev1` | `chage -l dev1` = `password must be changed` | ☐ |
| shadow 4~7 필드 직접 설정 | `passwd -n 7 -x 90 -w 14 -i 30 dev2` | `grep '^dev2:' /etc/shadow \| awk -F: '{print $4,$5,$6,$7}'` | ☐ |
| chage 정책 일괄 적용 | `chage -M 90 -m 7 -W 14 -I 30 -E 2026-12-31 dev1` | `chage -l dev1` 7행 확인 | ☐ |
| usermod 잠금·만료 | `usermod -L/-U/-e/-f dev2` | `passwd -S dev2`, `chage -l dev2` | ☐ |
| 섀도 동기화·정합성 | `pwconv`, `grpconv`, `pwck -r`, `grpck -r` | `ls -l /etc/shadow /etc/gshadow` = `000` | ☐ |
| 비밀번호 복잡도 강제 | `/etc/security/pwquality.conf` 에 `minlen=10 dcredit=-1 …` | `su - dev1` → `passwd` 로 약한 비번 거절 확인 | ☐ |
| 비밀번호 강도 점수 | `echo '…' \| pwscore`, `pwmake 128` | 약한 값은 사유 출력·종료코드 1 | ☐ |
| PAM 구조 파악 | `ls /etc/pam.d`, `grep -vE '^#\|^$' /etc/pam.d/su` | 4필드(타입·컨트롤·모듈·인자) 확인 | ☐ |
| 공통 스택·authselect | `authselect current`, `readlink -f /etc/pam.d/system-auth` | `/etc/authselect/system-auth` 링크 확인 | ☐ |
| su 를 wheel 로 제한 | `/etc/pam.d/su` 의 `pam_wheel.so use_uid` 주석 해제 | dev1 `su -` 실패 / ops1 성공 + `/var/log/secure` | ☐ |
| 자원 제한 | `/etc/security/limits.conf` 에 `dev1 soft nproc 50` 등 | `su - dev1 -c 'ulimit -a'` → `-u 50`, `-n 1024` | ☐ |
| RHEL 9 제거 모듈 확인 | `ls /usr/lib64/security/pam_tally2.so`, `ls /etc/securetty` | 둘 다 없음 → `pam_faillock` 사용 (Part 10) | ☐ |
| `-aG` vs `-G` 사고 재현 | `usermod -G labqa dev2` → 복구 | `id dev2` 로 opsteam 소실·복구 확인 | ☐ |
| 계정 속성 수정 | `usermod -c/-d -m/-s/-l/-u/-g` | `getent passwd dev2`, `ls -ld /home/dev2` | ☐ |
| 셸·GECOS 전용 명령 | `chsh -s`, `chsh -l`, `chfn -f/-o/-p` | `getent passwd dev1` 5·7번 필드 | ☐ |
| nologin vs false | `useradd -M -s /sbin/nologin nolog1` / `/bin/false false1` | `su - nolog1`(메시지) vs `su - false1`(무출력), rc=1 | ☐ |
| guest1 회수 절차 | `usermod -L` → `-s /sbin/nologin` → `chage -E` → `pkill -u` | `passwd -S`, `chage -l`, `ps -u guest1` | ☐ |
| userdel vs userdel -r | `userdel guest1` (`-r` 없이) | `ls -ld /home/guest1`, `ls -l /var/spool/mail/guest1` 잔존 | ☐ |
| 고아 파일 탐색·정리 | `find / -xdev \( -nouser -o -nogroup \) 2>/dev/null` | 정리 후 재실행 시 출력 없음, `last guest1` 이력 보존 | ☐ |
| su 환경 차이 | `su dev1 -c 'echo $PWD'` vs `su - dev1 -c 'echo $PWD'` | `PWD`·`PATH` 차이 확인 | ☐ |
| sudo 조회·실행 옵션 | `sudo -l`, `-l -U ops1`, `-u dev1 id`, `-k`, `-v` | `uid=2001(dev1)` 출력 | ☐ |
| sudoers 문법 검사 | `visudo -c`, `EDITOR=vi visudo` | `parsed OK` | ☐ |
| Defaults 옵션 | `/etc/sudoers.d/99-lab-defaults` (`logfile`, `timestamp_timeout`, `env_keep`) | `visudo -c`, `ls -l` = `0440` | ☐ |
| ops1 제한 sudo 부여 | `visudo -f /etc/sudoers.d/ops1` (`!ALL` → `LAB_SVC, LAB_LOG`) | `sudo -l -U ops1` / 허용·불허 명령 각각 실행 | ☐ |
| sudo 로그 확인 | `grep 'COMMAND=' /var/log/secure` | 성공 `COMMAND=`, 거부 `command not allowed` 각 1건 | ☐ |
| 권한 문자열 해석 | `ls -l`, `stat -c '%A %a %F %n'` | 유형 문자·11번째 `.`/`+` 구분 | ☐ |
| umask 계산·실측 | `umask 022/027/077/023` 후 `touch`/`mkdir` | `ls -l` = 644/640/600/644, `ls -ld` = 755/750/700/754 | ☐ |
| umask 영구 설정 | `/etc/profile.d/lab-umask.sh`, `~/.bashrc` | `su - dev1 -c 'umask'` = `0002` | ☐ |
| chmod 8진수·기호 | `chmod 755/4755`, `u+s`, `a=rx`, `-R u+rwX`, `--reference` | `ls -l` 로 각 단계 확인 | ☐ |
| chown·chgrp | `chown dev1:devteam`, `chown :opsteam`, `-h`, `--from`, `--reference` | SetUID 파일 `chown` 후 비트 소실 확인 | ☐ |
| 특수 권한 실물 확인 | `ls -l /usr/bin/passwd`, `find /usr/bin -perm -2000`, `ls -ld /tmp` | `-rwsr-xr-x`, `-rwxr-sr-x`, `drwxrwxrwt` | ☐ |
| 대문자 S/T 재현 | `chmod 4644/2644/1666` | `-rwSr--r--`, `-rw-r-Sr--`, `drw-rw-rwT` | ☐ |
| 공유 디렉터리 SetGID | `mkdir /srv/devteam`, `chgrp devteam`, `chmod 2770` | dev1·dev2 생성 파일 그룹이 `devteam`, `sub1` 도 `s` 상속 | ☐ |
| Sticky 디렉터리 | `mkdir /srv/dropbox`, `chmod 1777` | dev2 가 dev1 파일 삭제 시도 → `Operation not permitted` | ☐ |
| find -perm 3종 비교 | `-perm 644` / `-perm -644` / `-perm /644` | 매칭 파일 수 1 / 3 / 4 | ☐ |
| SetUID 전수 검색 | `find / -perm -4000 -type f 2>/dev/null` | 목록 저장 → 기준선(Part 10) | ☐ |
| ACL 지원·조회 | `rpm -q acl`, `getfacl /srv/devteam` | `# flags: -s-`, 기본 3항목 | ☐ |
| ACL 항목 추가 | `setfacl -m u:ops1:rx,g:opsteam:rwx /srv/devteam` | `ls -ld` 에 `+`, ops1 목록 가능·생성 불가 | ☐ |
| mask 와 실효 권한 | `chmod g=r <ACL 파일>` → `getfacl` | `mask::r--`, `#effective:r--` 표시 | ☐ |
| 기본 ACL 상속 | `setfacl -d -m g:opsteam:r-x /srv/devteam` | 새로 만든 파일에 `default:` 상속 확인 | ☐ |
| ACL 삭제·일괄 적용 | `setfacl -x`, `-b`, `-R -M <파일>` | `ls -l` 에서 `+` 소실 | ☐ |
| ACL 백업·복원 | `getfacl -R … > acl.bak`, `setfacl --restore=acl.bak` | 제거 후 복원해 원상 확인 (`cd /` 에서 실행) | ☐ |
| lsattr 조회 | `lsattr /etc/passwd`, `lsattr -d /etc` | 속성 문자열 확인 (xfs 는 `e` 없음) | ☐ |
| chattr +i ⚠️ | `chattr +i /etc/passwd` → `chattr -i` | `echo >> ` 실패·`useradd` 실패 → 해제 후 정상 | ☐ |
| chattr +a | `chattr +a /var/log/app.log` | `>>` 성공 / `>`·`mv`·`rm` 실패 | ☐ |
| 그 밖의 속성 | `chattr +A/+d/+S`, `chattr +c`(미지원) | `lsattr` 확인, `+c` 는 `Operation not supported` | ☐ |
| 최종 계정·그룹 대조 | `getent passwd \| awk -F: '$3>=1000'`, `getent group` | README 2-1 명세와 일치 | ☐ |
| 최종 정책·권한 대조 | `sudo -l -U ops1`, `ls -ld /srv/*`, `getfacl`, `lsattr` | 12-2 기대 출력과 일치 | ☐ |
| 자체 점검 스크립트 | `/usr/local/bin/check-part03.sh` | `합계: OK 23 / NG 0`, 종료코드 0 | ☐ |

---

## 기출 연결

| 기출 (문제집·회차·번호 또는 주제) | 이 파트의 단계 |
| --- | --- |
| 실기 R01-02 `user1` 을 기존 그룹 유지하며 wheel 추가 → `usermod -aG wheel user1` | 6-1 |
| 실기 R01-06 소유자 rwx·그룹/기타 r-x → `chmod 755 report.txt` | 8-4 |
| 실기 R01-15 `/tmp` 의 `drwxrwxrwt` 마지막 `t` 의 의미·목적 서술 | 9-1 표, 9-4 |
| 실기 R02-01 · R05-03 `find / -perm -4000 -type f 2>/dev/null` (SetUID 전수 검색) | 9-5 |
| 실기 R02-02 `/var/www/html` 소유자·그룹 재귀 변경 → `chown -R apache:apache` | 8-5 |
| 실기 R02-06 · R05-06 파일 644·디렉터리 755 → `umask 022` | 8-2 계산표 |
| 실기 R02-07 · R05-07 비밀번호 최대 사용기간 90일 → `chage -M 90 user1` | 4-5 |
| 실기 R02-11 `/etc/shadow` 3~5번 필드 `18900`,`0`,`90` 의미 서술 | 3-5 표, 4-3, 4-4 |
| 실기 R02-13 SetUID·SetGID 8진수와 효과 서술 | 9-1 표 |
| 실기 R03-06 기존 755 실행 파일에 SetUID → `chmod 4755` | 8-4, 9-1 |
| 실기 R03-02 · R06-05 UID 1000 이상 계정명만 출력 → `awk -F: '$3>=1000 {print $1}'` | 3-4, 12-1 |
| 실기 R04-03 `/data` 이하 재귀 750 → `chmod -R 750 /data` | 8-4 |
| 실기 R04-08 로그 추가만 허용 → `chattr +a /var/log/app.log` | 11-3 |
| 실기 R04-13 umask 027 일 때 파일·디렉터리 권한 계산 서술 (640·750) | 8-2 계산표 |
| 실기 R04-15 PAM 제어 구문 4종(required·requisite·sufficient·optional) 동작 차이 서술 | 5-2 표 |
| 실기 R05-02 사용자 계정 잠금 → `usermod -L user2` | 4-2, 4-6 |
| 실기 R05-10 `/etc/passwd` 불변 속성 → `chattr +i /etc/passwd` | 11-2 |
| 실기 R05-11 `/etc/shadow` 필드 `18800`,`7`,`90`,`14` 의미 서술 | 3-5 표, 4-4, 4-5 |
| 필기 R01-21 `useradd` 로그인 셸 지정 옵션 → `-s` | 3-1 |
| 필기 R01-23 `/etc/shadow` 두 번째 필드(암호화 비밀번호·잠금 표시) | 3-5 표, 4-1, 4-2 |
| 필기 R01-24 · R07-64 `/etc/login.defs` 설정 항목(UID 범위·`PASS_MAX_DAYS`) | 1-2 |
| 필기 R01-25 · R04-23 · R07-32 보조 그룹 유지 추가 → `usermod -aG` | 6-1 |
| 필기 R01-26 · R03-44 · R04-24 그룹 관리 명령(`gpasswd -a/-A`, `newgrp`) | 2-3, 2-4 |
| 필기 R01-30 · R02-44 · R03-24 · R04-28 · R07-43 · R08-39 · R09-34 · R10-65 umask 0027 → 파일 640 | 8-2 계산표 |
| 필기 R01-31 · R02-29 ACL 부여 → `setfacl -m u:lee:rw data.txt` | 10-2 |
| 필기 R01-59 · R07-56 · R08-60 PAM 개요·설정 파일 위치 `/etc/pam.d/` | 5-1 |
| 필기 R01-63 · R05-31 · R09-28 `/etc/sudoers` 규칙 해석(`%wheel`, 명령 한정) | 7-3 표, 7-5 |
| 필기 R01-64 · R05-62 비밀번호 만료 정보 조회 → `chage -l kim` | 4-5 |
| 필기 R02-21 · R04-22 · R05-26/28 `useradd -D` 출력 = `/etc/default/useradd` | 1-1 |
| 필기 R02-22 · R03-40 로그인 셸 `/sbin/nologin` 변경 → `usermod -s`(+`-c`) | 6-2, 6-4 |
| 필기 R02-23 그룹 이름 변경 → `groupmod -n backend devteam` | 2-3 |
| 필기 R02-25 · R10-24 `-rwsr-xr-x` = `4755` | 8-1, 9-1 |
| 필기 R02-27 · R07-42 · R09-32 공유 디렉터리 그룹 상속 = SetGID | 9-3 |
| 필기 R02-28 · R08-24 · R10-26 `getfacl` 출력 해석·`#effective` 와 mask | 10-1 표, 10-3 |
| 필기 R02-45 · R08-23 · R09-22 `chage -l` 출력 해석(비밀번호 만료 vs 계정 만료) | 4-5 |
| 필기 R02-60 · R03-58 · R05-59 · R09-38 · R10-57 PAM 컨트롤 플래그 동작 | 5-2 표 |
| 필기 R03-21 · R07-60 `su` 와 `sudo` 차이(비밀번호 주체·로그 기록·선별 위임) | 7-1, 7-2, 7-6 |
| 필기 R03-22 · R05-27 · R07-21 · R08-40 `useradd` 참조 파일(`/etc/skel`·`login.defs`·`default/useradd`) | 1-1 ~ 1-3 |
| 필기 R03-23 · R09-26 계정 잠금·만료(`usermod -L` 은 `!` 접두, `chage -E 0` 은 계정 만료) | 4-2, 4-6, 6-5 |
| 필기 R03-26 · R09-35 · R06-61 `chattr`/`lsattr` 속성 설명·`+i` 효과 | 11-1, 11-2, 11-4 |
| 필기 R04-21 · R06-21 · R10-21 `useradd -m -s /bin/bash -e 2026-12-31 -G wheel kim` 해석 | 3-1 ~ 3-3 |
| 필기 R04-57 `chage` 옵션 설명 중 틀린 것 (`-M`/`-m`/`-W`/`-I`/`-E`) | 4-5, 4-6 대응표 |
| 필기 R04-59 · R09-25 · R10-61 `/etc/shadow` 의 `$6$` = SHA-512·솔트 | 4-1 |
| 필기 R04-60 · R09-29 sudoers 안전 편집 → `visudo` | 7-3 |
| 필기 R04-64 · R09-33 `/tmp` 처럼 자기 파일만 삭제 = Sticky Bit | 9-4 |
| 필기 R04-65 · R09-31 SetUID 파일 검색 → `find / -perm -4000 -type f` | 9-5 |
| 필기 R05-22 `/etc/shadow` 여섯 번째 필드 `7` = 경고일 | 3-5 표, 4-4 |
| 필기 R05-24 `/etc/gshadow` 세 번째 필드 = 그룹 관리자 | 2-2 표, 2-4 |
| 필기 R05-25 · R09-21 `/etc/login.defs` 의 `UID_MIN 1000` · `PASS_WARN_AGE 7` | 1-2 |
| 필기 R05-32 · R10-62 sudoers `NOPASSWD:` 의 의미 | 7-5, 7-6 |
| 필기 R05-58 PAM 첫 필드 `auth` = 인증 단계 | 5-1, 5-2 표 |
| 필기 R05-60 · R09-40/41 `/etc/security/limits.conf` 와 `ulimit -n`·`-u` | 5-5 |
| 필기 R05-61 · R06-64 · R09-27 wheel 만 `su` 허용 → `/etc/pam.d/su` 의 `pam_wheel.so use_uid` | 5-4 |
| 필기 R06-22 기본 로그인 셸 변경 → `useradd -D -s /bin/zsh` | 1-1 |
| 필기 R06-23 홈·메일 보존하며 로그인만 차단 → `usermod -L -s /sbin/nologin` | 6-4, 6-5 |
| 필기 R06-24 팀 공유 디렉터리 → `chgrp devteam` 후 `chmod 2770` | 9-3 |
| 필기 R06-44 암호 만료 정보 저장 파일 = `/etc/shadow` | 3-5 표, 4-5 |
| 필기 R06-45 파일 640·디렉터리 750 → umask `027` | 8-2 계산표 |
| 필기 R06-56 httpd 재시작만 sudo 위임 → `webadm ALL=(root) /usr/bin/systemctl restart httpd` | 7-5 |
| 필기 R06-57 · R09-39 로그인 실패 잠금 → `pam_faillock` (`deny=5 unlock_time=600`) | 5-6 (실습은 Part 10) |
| 필기 R06-65 · R07-64 프로세스 수 제한 설정 파일 = `/etc/security/limits.conf` | 5-5 |
| 필기 R07-57 `sudo systemctl restart httpd` 내부 처리 흐름(sudoers 확인 → 인증 → 실행 → 로그) | 7-2, 7-5, 7-6 |
| 필기 R08-03 `-rwxr-sr-x` 그룹 위치 `s` = SetGID | 8-1, 9-1 |
| 필기 R08-21 `id` 출력에서 1차/2차 그룹 구분 | 3-2, 12-1 |
| 필기 R08-22 · R10-22 `/etc/shadow` 항목 해석(90일 만료·14일 전 경고) | 3-5 표, 4-5 |
| 필기 R08-59 wheel 그룹 전체 sudo 허용 → `%wheel ALL=(ALL) ALL` | 7-3 |
| 필기 R09-07 · R09-09 SetUID `passwd` 실행 시 EUID 가 root 로 전환 | 9-1 |
| 필기 R09-23 계정 자체 만료 → `chage -E 2026-12-31 alice` | 4-5, 6-5 |
| 필기 R09-24 `pam_pwquality` 의 `minlen=10 dcredit=-1` 해석 | 4-8 |
| 필기 R09-30 `/etc/securetty` 역할 (RHEL 9 에서 제거) | 5-6 |
| 필기 R09-52 · R08-27 `nosuid` 마운트 옵션 = SetUID 무효화 | 9-1 (파일시스템 실습은 Part 05) |
| 필기 R10-23 `groups` 와 `newgrp` 동작 | 2-4, 6-1 |

---

## 이전 / 다음

[[02-package-management]] ← · → [[04-file-text-shell]]

[[README]] · 이론: [[../THEORY/user-permission]] · [[../THEORY/system-security]] · 실사용 문서: [[../../USER-PERMISSION/usermod]] · [[../../USER-PERMISSION/passwd]] · [[../../USER-PERMISSION/id]] · [[../../USER-PERMISSION/sudo]] · [[../../USER-PERMISSION/last]]
