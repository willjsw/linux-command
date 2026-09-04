---
title: LAB 04 — 파일·텍스트 처리와 셸 스크립트
type: exam-lab
part: 04
tags:
  - exam/linux-master
  - exam/lab
  - linux/file
  - linux/text
  - linux/shell
  - task/configure
  - task/verify
related: ["[[README]]", "[[03-user-group-permission]]", "[[05-disk-lvm-raid-swap-quota]]", "[[../THEORY/system-structure]]", "[[../THEORY/linux-basics]]"]
distro: Rocky Linux 9 (aarch64, UTM)
updated: 2026-09-03
---

# LAB 04 — 파일·텍스트 처리와 셸 스크립트

- 목표: 팀 공유 디렉터리에 샘플 프로젝트·로그를 만들고 **파일 조작 → 검색(find/grep) → 텍스트 처리(sed/awk/sort) → 압축·아카이브(tar/gzip/증분) → 리다이렉션·변수 → 스크립트 작성 → vi** 까지 전 범위 수행
- 산출물: `/usr/local/bin/backup.sh`, `/usr/local/bin/sysreport.sh` — [[06-process-scheduling-diagnosis|Part 06]](cron)·[[07-boot-systemd-log|Part 07]](서비스)·[[12-backup-recovery-review|Part 12]](백업) 에서 그대로 재사용
- 원칙: 명령마다 **다른 명령으로 결과 검증** (`ls -li`, `stat`, `tar tvf`, `sha256sum -c`, `echo $?`)
- 실기 최빈출(find 조건 삭제, grep -c, awk -F:, sed -i, tar czf/증분, sha256sum, `#!/bin/bash`) 전부 포함

> **이 파트의 시나리오**: Part 03 에서 `devteam` 그룹과 공유 디렉터리 `/srv/devteam`(setgid 2775) 이 만들어졌다. 개발팀이 여기에 샘플 프로젝트를 올리기 시작했고, 운영자 `ops1` 은 로그를 분석·정리·아카이브해야 한다. 매번 손으로 하기 싫으니 마지막에 백업·진단 스크립트로 자동화한다.

- 선행 자원: 사용자 `dev1`·`dev2`·`ops1`, 그룹 `devteam`, 디렉터리 `/srv/devteam` — Part 03 에서 생성됨. 없으면 `mkdir -p /srv/devteam && chgrp devteam /srv/devteam && chmod 2775 /srv/devteam`
- 작업 계정: 별도 표기 없으면 root(`#`)

---

## 1. 실습 데이터 생성

### 1-1. 프로젝트 디렉터리 골격과 텍스트 파일

> **상황**: 개발팀 샘플 프로젝트 구조(`src/logs/docs`)를 한 번에 만들고, 이후 실습에 쓸 텍스트 파일을 `echo`·heredoc·`seq` 로 채운다.

```bash
mkdir -p /srv/devteam/proj/{src,logs,docs}          # 중괄호 확장으로 3개 하위 디렉터리 동시 생성
cd /srv/devteam/proj
echo "Sample Project v1.0" > docs/README.txt         # 한 줄 파일
cat > docs/notes.txt <<'EOF'                          # heredoc 으로 여러 줄 파일
# 프로젝트 메모
apple banana cherry
Apple pie
banana split

grape apple
EOF
seq 1 100 > src/numbers.txt                           # 1~100 숫자 100행
seq 10 -2 0 > src/even.txt                            # 10 8 6 4 2 0 (역순, 간격 2)
for i in $(seq 1 5); do echo "module $i" > src/mod$i.c; done   # 반복으로 파일 5개
printf 'id,name,score\n1,kim,90\n2,lee,85\n3,park,77\n' > docs/score.csv
```

- `mkdir -p` : 중간 경로 포함 생성, 존재해도 오류 없음 (**p**arents)
- `{src,logs,docs}` : 셸 **중괄호 확장** — `proj/src proj/logs proj/docs` 3개로 전개
- `<<'EOF'` : heredoc — `EOF` 줄까지 표준입력으로 전달. 종결자를 `'EOF'` 로 인용하면 내부 `$` 확장 억제
- `seq 시작 [증분] 끝` : 수열 출력 (**seq**uence)
- `$(seq 1 5)` : 명령 치환 — 출력 결과를 `for` 목록으로 사용
- `printf` : 서식 출력 — `\n` 개행 해석 (echo -e 보다 이식성 우수)

**검증**
```bash
tree /srv/devteam/proj              # tree 없으면 dnf install -y tree
wc -l src/numbers.txt docs/notes.txt
cat src/even.txt | tr '\n' ' '; echo
```

```text
/srv/devteam/proj
├── docs
│   ├── README.txt
│   ├── notes.txt
│   └── score.csv
├── logs
└── src
    ├── even.txt
    ├── mod1.c
    ...
    └── numbers.txt
3 directories, 9 files
100 src/numbers.txt
  7 docs/notes.txt
10 8 6 4 2 0
```

> 📝 **시험 포인트**: `mkdir -p a/b/c` 와 중괄호 확장 `{a,b}` 는 필기 단답 빈출. heredoc `<<` 와 here-string `<<<` 구분.

### 1-2. 샘플 로그·더미 대용량 파일·오래된 파일

> **상황**: 로그 분석 실습용으로 실제 시스템 로그를 복사하고, `find -size`·`du` 실습용 대용량 파일과 `find -mtime` 실습용 오래된 파일을 만든다.

```bash
cp /var/log/secure /var/log/messages /srv/devteam/proj/logs/     # 실제 로그 사본
cd /srv/devteam/proj/logs
dd if=/dev/urandom of=big.bin bs=1M count=50 status=progress     # 50 MB 난수 더미
fallocate -l 120M huge.img                                       # 120 MB 즉시 할당 (내용 0)
touch -d "10 days ago" old-app.log                               # 수정 시각을 10일 전으로
touch -d "3 days ago" mid-app.log
touch -t 202601011200 newyear.log                                # [[CC]YY]MMDDhhmm 형식
: > empty.log                                                    # 빈 파일 (null 명령 + 리다이렉션)
```

- `dd if= of=` : 입력(**i**nput **f**ile)·출력(**o**utput **f**ile) 장치/파일 지정
- `bs=1M count=50` : 블록 크기(**b**lock **s**ize) 1 MiB × 50 회 = 50 MiB
- `status=progress` : 진행률 표시
- `/dev/urandom` : 난수 문자 장치 — 블로킹 없는 의사난수 (`/dev/random` 은 엔트로피 부족 시 대기 가능, RHEL 9 커널에선 실질 동일)
- `fallocate -l` : 실제 쓰기 없이 블록만 예약 → 즉시 완료 (**l**ength)
- `touch -d "<날짜 문자열>"` : 접근·수정 시각을 지정 문자열로 설정 (**d**ate) — `"10 days ago"`, `"2026-01-01 12:00"` 등 GNU date 형식
- `touch -t [[CC]YY]MMDDhhmm[.ss]` : 숫자 형식 시각 지정 (**t**ime)
- `: > 파일` : `:` 은 항상 참인 null 명령 → 출력 없이 파일만 생성/비움

**검증**
```bash
ls -lh /srv/devteam/proj/logs
ls -l --time-style=long-iso old-app.log newyear.log
du -sh /srv/devteam/proj/logs
```

```text
-rw-r--r--. 1 root root  50M ... big.bin
-rw-r--r--. 1 root root 120M ... huge.img
-rw-r--r--. 1 root root    0 ... empty.log
-rw-r--r--. 1 root root    0 2026-08-24 ... old-app.log
-rw-r--r--. 1 root root    0 2026-01-01 12:00 newyear.log
171M    /srv/devteam/proj/logs
```

> 📝 **시험 포인트**: `dd if=/dev/zero of=/swapfile bs=1M count=2048` (스왑 파일, Part 05) 과 `dd if=/dev/vda of=mbr.img bs=512 count=1` (MBR 백업, Part 12) 은 `if/of/bs/count` 의미를 묻는 최빈출. `touch` 는 빈 파일 생성 + 시각 갱신 두 용도.

---

## 2. 기본 파일 조작

### 2-1. ls 옵션 총정리

> **상황**: 방금 만든 파일들을 여러 관점(숨김·크기순·시간순·inode·SELinux 컨텍스트)으로 나열한다.

```bash
cd /srv/devteam/proj
ls -l            # 긴 형식
ls -a            # 숨김 파일(.으로 시작) 포함
ls -lh logs      # 사람이 읽는 단위
ls -R            # 재귀
ls -lt logs      # 수정 시각 최신순
ls -lS logs      # 크기 큰 순
ls -li docs      # inode 번호
ls -ld /srv/devteam        # 디렉터리 자체 정보 (내용 아님)
ls -Z docs/README.txt      # SELinux 컨텍스트
ls -lrt logs               # 시간순 역정렬 → 최신이 맨 아래
```

- `-l` : 긴 형식 — 유형·권한·링크수·소유자·그룹·크기·시각·이름 (**l**ong)
- `-a` : 숨김 포함 (**a**ll), `-A` 는 `.`/`..` 제외
- `-h` : K/M/G 단위 (**h**uman-readable)
- `-R` : 하위 디렉터리 재귀 (**R**ecursive)
- `-t` : 수정 시각순 (**t**ime), `-r` 로 역순 (**r**everse)
- `-S` : 크기순 (**S**ize)
- `-i` : inode 번호 (**i**node)
- `-d` : 디렉터리 자체 (**d**irectory)
- `-Z` : SELinux 보안 컨텍스트 (SELinu**Z**)

**검증**
```bash
ls -ld /srv/devteam        # Part 03 의 setgid(2775) 확인
ls -lS logs | head -3
```

```text
drwxrwsr-x. 3 root devteam ... /srv/devteam
total ...
-rw-r--r--. 1 root root 125829120 ... huge.img
-rw-r--r--. 1 root root  52428800 ... big.bin
```

> 📝 **시험 포인트**: `ls -l` 첫 문자(`-` `d` `l` `b` `c` `s` `p`)와 두 번째 필드 **링크 수** 해석 문제. `-d` 없이 디렉터리를 주면 내용이 나옴.

### 2-2. cp · mv · rm · rmdir · mkdir -m

> **상황**: 프로젝트를 백업 사본으로 복제하고, 파일 이름 변경·이동·삭제 동작 차이를 확인한다.

```bash
cd /srv/devteam
cp -a proj proj.bak                    # 속성·링크·시각 보존 재귀 복사 (= -dR --preserve=all)
cp -r proj/docs /tmp/docs-copy         # 단순 재귀 복사 (소유자·시각은 현재값)
cp -p proj/docs/README.txt /tmp/       # 권한·시각 보존
cp -i proj/docs/README.txt /tmp/       # 덮어쓰기 전 확인 (n 입력)
cp -u proj/docs/*.txt /tmp/docs-copy/  # 원본이 더 새로울 때만 복사
mv proj/docs/README.txt proj/docs/README.md   # 이름 변경
mv proj/src/mod5.c proj/               # 이동
mv -i proj/mod5.c proj/src/            # 되돌리기 (확인 프롬프트)
mkdir -m 750 proj/private              # 권한 지정 생성 (umask 무시)
rmdir proj/private                     # 빈 디렉터리만 삭제 가능
mkdir -p proj/tmp/a/b && rmdir -p proj/tmp/a/b   # 빈 상위까지 연쇄 삭제
```

- `cp -a` : 아카이브 모드 — 재귀 + 심볼릭 링크 유지 + 모든 속성 보존 (**a**rchive)
- `cp -r`/`-R` : 디렉터리 재귀 복사 (**r**ecursive)
- `cp -p` : 소유자·권한·시각 보존 (**p**reserve)
- `cp -i` : 덮어쓰기 전 확인 (**i**nteractive)
- `cp -u` : 대상이 없거나 원본이 더 새로울 때만 (**u**pdate)
- `mkdir -m` : 생성 시 권한 모드 지정 (**m**ode)
- `rmdir` : **빈** 디렉터리 삭제, `-p` 는 빈 부모까지 (**p**arents)

**검증**
```bash
ls -l proj/docs/README.md proj.bak/docs/README.txt   # 사본은 원본 이름 유지
stat -c '%y %n' proj/docs/notes.txt proj.bak/docs/notes.txt   # -a 는 시각 동일
rmdir proj/docs 2>&1 | head -1                         # 비어 있지 않아 실패해야 정상
```

```text
-rw-r--r--. 1 root root 20 ... proj/docs/README.md
-rw-r--r--. 1 root root 20 ... proj.bak/docs/README.txt
2026-09-03 ... proj/docs/notes.txt
2026-09-03 ... proj.bak/docs/notes.txt
rmdir: failed to remove 'proj/docs': Directory not empty
```

> 📝 **시험 포인트**: `cp -a` = `-dpR`, `rmdir` 은 빈 디렉터리 전용(내용 있으면 `rm -r`). `mkdir -m 750` 은 umask 와 무관하게 지정.

### 2-3. rm -rf (⚠️) 와 안전 습관

> **상황**: `/tmp/docs-copy` 를 정리한다. `rm -rf` 는 복구 불가 — 반드시 대상을 먼저 `ls` 로 확인한다.

⚠️ `rm -rf` 는 확인 없이 재귀 강제 삭제. 변수 사용 시 `rm -rf "$DIR"/` 에서 `$DIR` 이 비면 `/` 대상이 됨. 아래처럼 **삭제 전 같은 경로를 `ls -ld` 로 재확인**.

```bash
ls -ld /tmp/docs-copy                  # 대상 재확인
rm -rf /tmp/docs-copy                  # 재귀 + 강제
rm -i /tmp/README.txt                  # 개별 확인 (y)
rm -f /tmp/nonexistent                 # 없어도 오류 없음, 종료 코드 0
echo $?
```

- `-r` : 디렉터리 재귀 삭제 (**r**ecursive)
- `-f` : 확인 없음·부재 무시 (**f**orce)
- `-i` : 삭제 전 확인 (**i**nteractive) — root 의 alias 로 흔히 기본 적용

**검증**
```bash
ls -ld /tmp/docs-copy /tmp/README.txt 2>&1
```

```text
ls: cannot access '/tmp/docs-copy': No such file or directory
ls: cannot access '/tmp/README.txt': No such file or directory
```

> 📝 **시험 포인트**: `rm -rf /` 방지용 `--preserve-root` 가 GNU rm 기본값. `rm -f` 는 없는 파일에도 0 반환 → 스크립트에서 `&&` 체인 주의.

### 2-4. 하드 링크 vs 심볼릭 링크

> **상황**: `docs/notes.txt` 에 하드 링크와 심볼릭 링크를 각각 만들어 inode·링크 수·원본 삭제 후 동작 차이를 실증한다.

```bash
cd /srv/devteam/proj/docs
ln notes.txt notes.hard            # 하드 링크
ln -s notes.txt notes.sym          # 심볼릭 링크 (상대 경로)
ln -s /srv/devteam/proj/docs/notes.txt /tmp/notes.abs   # 절대 경로 심볼릭 링크
ln -s docs /srv/devteam/proj/docs-link                 # 디렉터리 심볼릭 링크 (하드는 불가)
ln notes.txt /tmp/notes.hard2 2>&1 | head -1           # 같은 파일시스템이면 성공, 다르면 오류
```

- `ln 원본 링크` : 하드 링크 — 같은 inode 를 가리키는 새 이름, 링크 수 +1
- `ln -s 원본 링크` : 심볼릭 링크 — 경로 문자열을 담은 **새 inode** (**s**ymbolic)
- `ln -sf` : 기존 링크 덮어쓰기 (**f**orce), `-n` : 링크가 디렉터리를 가리킬 때 그 안으로 들어가지 않음

**검증**
```bash
ls -li notes.txt notes.hard notes.sym      # inode·링크 수 비교
stat -c '%i %h %n' notes.txt notes.hard
readlink notes.sym; readlink -f notes.sym  # 링크가 가리키는 경로 / 최종 절대 경로
mv notes.txt notes.orig                    # 원본 이름 변경 → 심볼릭은 깨짐, 하드는 정상
cat notes.sym 2>&1 | head -1
cat notes.hard | head -2
mv notes.orig notes.txt                    # 복구
```

```text
1234567 -rw-r--r--. 2 root root 63 ... notes.hard
1234567 -rw-r--r--. 2 root root 63 ... notes.txt
1234570 lrwxrwxrwx. 1 root root  9 ... notes.sym -> notes.txt
1234567 2 notes.txt
1234567 2 notes.hard
notes.txt
/srv/devteam/proj/docs/notes.txt
cat: notes.sym: No such file or directory      ← dangling
# 프로젝트 메모
apple banana cherry
```

> 📝 **시험 포인트**: 하드 링크 = 동일 inode·링크 수 증가·파일시스템 경계 불가·디렉터리 불가. 심볼릭 = 새 inode·원본 삭제 시 dangling·경계 넘음·디렉터리 가능. `ls -l` 의 `l` 과 `-> 원본` 표기.

### 2-5. stat · file · basename · dirname · pwd · cd -

> **상황**: 파일의 메타데이터(inode·3종 시각·권한 8진수)와 내용 기반 유형을 확인하고 경로 문자열을 분해한다.

```bash
stat notes.txt
stat -c '%a %A %U %G %s %n' notes.txt /usr/bin/passwd   # 8진 권한·문자 권한·소유자·그룹·크기·이름
file notes.txt ../logs/big.bin ../logs/huge.img notes.sym /bin/ls /etc
file -b notes.txt          # 파일명 없이 판정만
file -i notes.txt          # MIME 타입
file -L notes.sym          # 링크 대상 판정
basename /srv/devteam/proj/docs/notes.txt          # notes.txt
basename /srv/devteam/proj/docs/notes.txt .txt     # 확장자 제거 → notes
dirname  /srv/devteam/proj/docs/notes.txt          # /srv/devteam/proj/docs
pwd; cd /etc; pwd; cd -; pwd                       # cd - 는 직전 디렉터리로
```

- `stat -c` : 출력 형식 지정 (**c**ustom format) — `%a` 8진 권한, `%A` 문자 권한, `%U`/`%G` 소유자/그룹, `%s` 크기, `%i` inode, `%h` 링크 수, `%y` 수정시각, `%n` 이름
- `file -b` : 이름 생략 (**b**rief), `-i` : MIME (**i**nternet media type), `-L` : 심볼릭 링크 추적 (**L**ink)
- `basename 경로 [접미사]` : 마지막 요소, 접미사 제거 가능
- `dirname 경로` : 마지막 요소 제외한 디렉터리 부분
- `cd -` : `$OLDPWD` 로 이동 후 경로 출력

**검증**
```bash
stat notes.txt | grep -E 'Access: \(|Modify|Change|Birth'
```

```text
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Modify: 2026-09-03 ...
Change: 2026-09-03 ...
Birth: 2026-09-03 ...
```

> 📝 **시험 포인트**: `stat -c '%a %U %G' /usr/bin/passwd` → `4755 root root` (SetUID) 해석 문제. atime(읽기)·mtime(내용 변경)·ctime(inode 변경 — chmod·chown 도 갱신) 구분.

### 2-6. du · df · wc

> **상황**: 프로젝트가 차지하는 공간과 파일시스템 여유·inode 를 확인하고, 텍스트 파일 규모를 센다.

```bash
du -sh /srv/devteam/proj                   # 합계만
du -sh /srv/devteam/proj/* | sort -h       # 하위 항목별, 단위 인식 정렬
du -h --max-depth=1 /srv/devteam | sort -h # 깊이 1 까지
df -h                                      # 파일시스템 사용량
df -hT /srv                                # 유형 포함
df -i                                      # inode 사용량
wc -l /etc/passwd                          # 행 수
wc -w -c /srv/devteam/proj/docs/notes.txt  # 단어·바이트
wc -l < /etc/passwd                        # 파일명 없이 숫자만
```

- `du -s` : 합계만 (**s**ummarize), `-h` 단위, `--max-depth=N` : 표시 깊이 제한
- `sort -h` : `K/M/G` 단위 인식 정렬 (**h**uman-numeric)
- `df -T` : 파일시스템 **T**ype 열 추가, `-i` : **i**node 사용률
- `wc -l`/`-w`/`-c`/`-m` : 행(**l**ines)·단어(**w**ords)·바이트(**c**hars=bytes)·문자(**m**ultibyte chars)

**검증**
```bash
df -hT /srv | tail -1
```

```text
/dev/mapper/rl-root xfs   ...G  ...G  ...G  ..% /
```

> 📝 **시험 포인트**: 여유 공간이 있어도 `touch` 실패 → `df -i` 로 inode 고갈 확인(필기 기출). `du` 는 실제 사용 블록, `ls -l` 은 논리 크기(sparse 파일 차이).

### 2-7. cat · tac · nl · rev · head · tail · more · less

> **상황**: 로그·텍스트를 여러 방식으로 조회한다. 특히 `tail -f` 로 실시간 로그 추적을 확인한다.

```bash
cd /srv/devteam/proj
cat -n docs/notes.txt          # 행 번호
cat -A docs/notes.txt          # 탭(^I)·행끝($) 등 비출력 문자 표시
tac docs/notes.txt             # 역순 출력
nl docs/notes.txt              # 비어있지 않은 행에만 번호
rev docs/README.md             # 각 행 문자 뒤집기
head -n 3 src/numbers.txt      # 처음 3행
head -c 10 src/numbers.txt; echo   # 처음 10바이트
head -n -95 src/numbers.txt    # 마지막 95행 제외 → 앞 5행
tail -n 3 src/numbers.txt      # 마지막 3행
tail -n +98 src/numbers.txt    # 98행부터 끝까지
tail -f /var/log/messages      # 추가되는 내용 실시간 (Ctrl+C 종료). 다른 터미널에서 logger test 실행
tail -F /var/log/messages      # 파일이 회전(logrotate)되어도 이름 기준으로 계속 추적
more /etc/services             # 페이지 단위 (Space 다음, q 종료)
less /etc/services             # 양방향 페이저
```

- `cat -n` : 번호 (**n**umber), `-A` : 모두 표시 (**A**ll = `-vET`)
- `tac` : `cat` 역순 (이름도 거꿀)
- `nl` : 번호 붙이기 (**n**umber **l**ines) — 빈 행 제외가 `cat -n` 과 차이
- `head -n N`/`-c N` : 앞 N 행/바이트, `-n -N` : 마지막 N 행 제외
- `tail -n N`, `-n +N` : N 행부터 끝까지, `-f` : 추가분 추적 (**f**ollow), `-F` : 이름 기준 재열기 (= `--follow=name --retry`)
- `less` 조작키: `Space`/`f` 다음 페이지, `b` 이전, `/패턴` 검색, `n`/`N` 다음/이전 일치, `g`/`G` 처음/끝, `q` 종료, `-N` 옵션으로 행 번호

**검증**
```bash
# 터미널 1
tail -f /var/log/messages
# 터미널 2
logger "LAB04 tail test"
# 터미널 1 에 즉시 출력되는지 확인 후 Ctrl+C
```

```text
Sep  3 ... srv01 root[...]: LAB04 tail test
```

> 📝 **시험 포인트**: `tail -f` 실시간 로그 감시(`history` 재실행 `!n` 과 묶여 출제), `head -n 5` = `head -5`. `-f` 와 `-F` 차이(로그 회전).

### 2-8. split 과 cat 병합 검증 · cmp · diff · patch

> **상황**: 대용량 더미를 조각내어 옮긴 뒤 다시 합쳐 원본과 같은지 확인하고, 텍스트 두 버전의 차이를 패치로 만들어 적용한다.

```bash
cd /srv/devteam/proj/logs
split -b 20M big.bin big.part.          # 20 MB 단위 → big.part.aa ab ac
split -l 30 ../src/numbers.txt num.     # 30행 단위 → num.aa ab ac ad
cat big.part.* > big.merged             # 병합
cmp big.bin big.merged && echo SAME     # 바이트 비교 (동일하면 출력 없음)
sha256sum big.bin big.merged            # 해시 비교
rm -f big.part.* num.* big.merged

cd ../docs
cp notes.txt notes.v2.txt
sed -i 's/cherry/citrus/; $a new last line' notes.v2.txt
diff notes.txt notes.v2.txt             # 기본(normal) 형식
diff -u notes.txt notes.v2.txt > notes.patch   # unified 형식 → patch 입력
dnf install -y patch
patch notes.txt < notes.patch           # notes.txt 에 변경 적용
diff notes.txt notes.v2.txt && echo IDENTICAL
patch -R notes.txt < notes.patch        # 되돌리기
```

- `split -b 크기` : 바이트 단위 분할 (**b**ytes), `-l 행수` : 행 단위 (**l**ines). 접두사 뒤 `aa ab …` 자동 부여
- `cmp` : 바이트 단위 비교 — 첫 차이 위치 출력, 동일하면 무출력·종료 0
- `diff -u` : unified 형식 (**u**nified) — `---`/`+++` 헤더, `-`/`+` 행. `-r` 디렉터리 재귀, `-q` 차이 유무만
- `patch 대상 < 패치파일` : diff 출력을 적용. `-R` 역적용 (**R**everse), `-p1` 경로 앞부분 제거 (git diff 적용 시)

**검증**
```bash
cat notes.patch
grep -c citrus notes.txt    # patch 전 0, 후 1, -R 후 0
```

```text
--- notes.txt   2026-09-03 ...
+++ notes.v2.txt        2026-09-03 ...
@@ -1,7 +1,8 @@
 # 프로젝트 메모
-apple banana cherry
+apple banana citrus
 ...
+new last line
```

> 📝 **시험 포인트**: `diff -u` → `patch` 흐름은 소스 패치 적용 문제(`patch -p1 < x.patch`)로 출제. `cmp` 는 바이너리, `diff` 는 텍스트.

### 2-9. od · hexdump · strings — 바이너리 들여다보기

> **상황**: 텍스트가 아닌 파일의 실제 바이트를 확인하고, 바이너리 안의 문자열만 추출한다.

```bash
dnf install -y binutils            # strings 포함
cd /srv/devteam/proj
od -c docs/README.md | head -3     # 문자 표기 (개행은 \n)
od -An -tx1 docs/README.md | head -2   # 주소 없이 16진 1바이트
hexdump -C docs/README.md | head -3    # 16진 + ASCII 병기 (canonical)
head -c 64 logs/big.bin | hexdump -C
strings /bin/ls | head -5
strings -n 8 /bin/ls | grep -i 'gnu' | head -3
```

- `od -c` : 문자로 덤프 (**o**ctal **d**ump, **c**haracter), `-An` : 주소 열 생략 (**A**ddress **n**one), `-tx1` : 16진 1바이트 단위 (**t**ype)
- `hexdump -C` : 16진 + ASCII 정규 형식 (**C**anonical)
- `strings -n N` : 최소 길이 N(기본 4) 이상 출력 가능 문자열만 (**n**umber)

**검증**
```bash
od -c docs/README.md | head -1
```

```text
0000000   S   a   m   p   l   e       P   r   o   j   e   c   t       v
```

> 📝 **시험 포인트**: `od -c` 로 탭·개행 등 제어문자 확인, `strings` 는 침해 분석에서 바이너리 IOC 추출(Part 10). `dd` 로 MBR 을 백업한 뒤 `hexdump -C mbr.img | tail -2` 로 `55 aa` 시그니처 확인은 Part 12 에서 수행.

---

## 3. 검색 — find · locate · grep

### 3-1. find 기본 — 경로·이름·유형·깊이

> **상황**: 프로젝트 트리에서 소스 파일과 로그 파일을 이름·유형으로 골라낸다. `find` 는 "경로 → 조건 → 동작" 순서를 지켜야 하며, 조건 앞에 경로가 반드시 온다.

```bash
cd /srv/devteam/proj
find . -name "*.c"                    # 이름 글롭 일치 (대소문자 구분)
find . -iname "*.C"                   # 대소문자 무시
find /srv/devteam -type f             # 일반 파일만
find /srv/devteam -type d             # 디렉터리만
find /srv/devteam -type l             # 심볼릭 링크만
find . -path "*/src/*.c"              # 전체 경로 패턴
find . -regex '.*/mod[0-9]\.c'        # 정규표현식(BRE) 경로 일치
find /etc -maxdepth 1 -name "*.conf"  # 깊이 1 까지만 (하위 디렉터리 미탐색)
find /etc -mindepth 2 -name "*.conf" | head -5   # 깊이 2 이상만
find / -xdev -name "core" 2>/dev/null            # 마운트 경계 넘지 않음
find /proc -prune -o -name "*.log" -print 2>/dev/null | head -3   # /proc 만 건너뛰고 나머지 탐색
```

- `-name <글롭>` : 파일명 패턴 — 셸 글롭(`* ? []`)이며 **정규표현식 아님**. 셸이 먼저 전개하지 않도록 `" "` 인용 필수
- `-iname` : 대소문자 무시 (**i**gnore case)
- `-type` : 파일 유형 — `f` 일반, `d` 디렉터리, `l` 심볼릭 링크, `b` 블록, `c` 문자, `s` 소켓, `p` 파이프
- `-path <패턴>` : 파일명이 아닌 **전체 경로** 기준 패턴
- `-regex <정규식>` : 경로 전체를 정규표현식으로 (기본 BRE, `-regextype posix-extended` 로 ERE 전환)
- `-maxdepth N` / `-mindepth N` : 탐색 깊이 상한/하한 — **다른 조건보다 앞에** 두어야 경고 없음
- `-xdev` : 다른 파일시스템으로 내려가지 않음 (**x** = cross **dev**ice 금지)
- `-prune` : 일치 항목의 하위로 내려가지 않음 — 보통 `-o -print` 와 짝
- `2>/dev/null` : 권한 없는 디렉터리의 `Permission denied` 억제 (관용)

**검증**
```bash
find . -name "*.c" | wc -l          # mod1~mod5 = 5
find /srv/devteam -type d | sort
find . -maxdepth 1 -type f | wc -l  # 최상위에는 일반 파일 없음 → 0
```

```text
5
/srv/devteam
/srv/devteam/proj
/srv/devteam/proj/docs
/srv/devteam/proj/logs
/srv/devteam/proj/src
...
0
```

> 📝 **시험 포인트**: `find <경로> <조건> <동작>` 순서 고정. `-name` 은 글롭이고 `grep` 은 정규식 — 둘의 패턴 문법 혼동이 단골. `find / -name "*.conf" 2>/dev/null | wc -l` 해석(필기 R10-11).

### 3-2. find 크기·시간 조건

> **상황**: `/srv/devteam/proj/logs` 에 만들어 둔 50 MB·120 MB 더미와 10일 전·3일 전 로그로 `-size`·`-mtime` 의 부호(`+`/`-`/무부호) 의미를 실증한다.

```bash
cd /srv/devteam/proj/logs
find . -size +100M                 # 100 MiB 초과 (huge.img)
find . -size -100k                 # 100 KiB 미만 (빈 파일·작은 파일)
find . -size +40M -size -60M       # 40~60 MiB (조건 나열 = AND)
find . -size 0                     # 정확히 0 블록
find . -size +1000c                # 1000 바이트 초과
find . -mtime +7                   # 수정된 지 7일 초과 (old-app.log)
find . -mtime -1                   # 최근 24시간 이내 수정
find . -mtime 3                    # 3일 전 ~ 4일 전 구간 (mid-app.log)
find . -mmin -60                   # 최근 60분 이내 수정
find . -mmin +1440                 # 1440분(24시간) 초과
find . -atime +7                   # 마지막 접근 후 7일 초과
find . -ctime -1                   # inode 변경(권한·소유자 포함) 최근 1일
touch /tmp/ref -d "5 days ago"
find . -newer /tmp/ref             # 기준 파일보다 최근에 수정된 것
find . ! -newer /tmp/ref           # 그 반대 (! = NOT)
find . -newermt "2026-01-02" ! -newermt "2026-09-01"   # 시각 구간 지정
```

- `-size N[cwbkMG]` : 크기 조건 — `c` 바이트, `w` 2바이트, `b` 512바이트 블록(**기본값**), `k` KiB, `M` MiB, `G` GiB
- 부호 규칙: `+N` = N 초과, `-N` = N 미만, `N` = 정확히 N (단위 절상 후 비교)
- `-mtime N` : **수정**(**m**odify) 시각 기준 N×24시간 — `+N` N일 초과, `-N` N일 이내, `N` N~N+1일 구간
- `-mmin N` : 분 단위 수정 시각 (**m**odify **min**ute)
- `-atime` / `-amin` : 접근(**a**ccess) 시각, `-ctime` / `-cmin` : inode 변경(**c**hange) 시각
- `-newer <파일>` : 기준 파일의 mtime 보다 최근, `-newermt "<시각>"` : 지정 시각 이후 (**newer** **m**time **t**ime)
- `!` 또는 `-not` : 조건 부정, 조건 나열은 AND, `-o` 는 OR (우선순위 명시는 `\( \)`)

**검증**
```bash
find . -size +100M -printf '%s %p\n'
find . -mtime +7 -printf '%TY-%Tm-%Td %p\n'
```

```text
125829120 ./huge.img
2026-08-25 ./old-app.log
```

> 📝 **시험 포인트**: `-size +100M` 의 `+` 는 "초과", `-mtime +7` 은 "7일 **지난**" — 부호 없는 `7` 은 "정확히 7일째"라는 점이 함정. 단위 생략 시 512바이트 블록이라는 것도 출제.

### 3-3. find 소유자·그룹·권한·빈 파일

> **상황**: 침해 점검·정리 작업의 기본인 소유자 없는 파일, 특수 권한 파일, 빈 파일을 조건으로 뽑는다. (권한 이론은 [[03-user-group-permission|Part 03]])

```bash
find /srv/devteam -user root                 # 소유자 UID 기준
find /srv/devteam -group devteam             # 그룹 기준
find /home -uid 2001 2>/dev/null             # 숫자 UID 로 지정
find / -nouser  2>/dev/null                  # /etc/passwd 에 없는 UID 소유 (삭제된 계정 잔여물)
find / -nogroup 2>/dev/null                  # /etc/group 에 없는 GID 소유
find /usr/bin -perm 755   | head -3          # 권한이 정확히 755
find / -perm -4000 -type f 2>/dev/null       # SetUID 비트가 "포함된" 파일
find / -perm -2000 -type f 2>/dev/null | head -5   # SetGID 포함
find / -perm /6000 -type f 2>/dev/null | head -5   # SetUID 또는 SetGID (OR)
find /srv -perm -o+w -type f                 # 기타 사용자 쓰기 가능 (위험 파일 점검)
find /srv/devteam -empty                     # 크기 0 인 파일 + 빈 디렉터리
find /srv/devteam -empty -type f             # 빈 파일만 (empty.log)
find /srv/devteam -empty -type d             # 빈 디렉터리만
```

- `-user <이름|UID>` / `-group <이름|GID>` : 소유자·소유 그룹 일치, `-uid` / `-gid` : 숫자 지정
- `-nouser` / `-nogroup` : 대응하는 계정·그룹이 없는 파일 — 계정 삭제 후 남은 파일 추적
- `-perm <모드>` : **정확히 일치** (예: `-perm 755`)
- `-perm -<모드>` : 지정 비트를 **모두 포함**(AND) — `-4000` = SetUID 보유
- `-perm /<모드>` : 지정 비트 중 **하나라도 포함**(OR) — `/6000` = SetUID 또는 SetGID (구형 `+6000` 는 폐지)
- `-empty` : 크기 0 인 일반 파일 또는 항목 없는 디렉터리

**검증**
```bash
find / -perm -4000 -type f 2>/dev/null | head -5
stat -c '%a %n' /usr/bin/passwd /usr/bin/su
find /srv/devteam -empty -type f
```

```text
/usr/bin/su
/usr/bin/mount
/usr/bin/passwd
...
4755 /usr/bin/passwd
4755 /usr/bin/su
/srv/devteam/proj/logs/empty.log
```

> 📝 **시험 포인트**: `-perm 4000`(정확히) · `-perm -4000`(포함) · `-perm /4000`(OR) 3형태 구분이 최빈출. SetUID 점검 정답은 `find / -perm -4000 -type f`(필기 R04-65·R05-65·R06-62·R09-31, 실기 R02-1·R05-3).

### 3-4. find 동작 — -exec · -ok · -delete · -print0 | xargs · -printf

> **상황**: 조건으로 찾은 결과에 실제 동작을 건다. `\;` 와 `+` 의 성능 차이, 공백 포함 파일명 대응(`-print0`)을 실측한다.

⚠️ `-delete` 와 `-exec rm` 은 복구 불가. **먼저 동작 없이 목록만 출력**해 대상을 확인한 뒤 실행한다.

```bash
cd /srv/devteam/proj
find . -name "*.c" -exec ls -l {} \;         # 일치 항목마다 명령 1회 실행
find . -name "*.c" -exec ls -l {} +          # 인자를 모아 명령 실행 횟수 최소화
find . -name "*.c" -exec grep -H module {} + # 여러 파일을 한 번에 grep
find . -name "*.txt" -ok rm {} \;            # 실행 전 항목마다 y/n 확인 (n 입력)
find . -name "*.c" -printf '%p %s %u:%g %m %TY-%Tm-%Td\n'   # 사용자 정의 출력
find . -name "*.c" -ls                        # ls -dils 형식
touch "src/my file.c"                         # 공백 포함 파일명 생성
find . -name "*.c" | xargs ls -l 2>&1 | tail -3     # 공백에서 분리 사고 발생
find . -name "*.c" -print0 | xargs -0 ls -l | tail -3   # NULL 구분자로 안전 처리
rm -f "src/my file.c"
```

- `-exec <명령> {} \;` : 일치 항목마다 명령 실행. `{}` 는 항목 자리, `\;` 는 명령 끝(셸이 `;` 를 먹지 않도록 이스케이프)
- `-exec <명령> {} +` : 인자를 모아 한 번에 실행 → **프로세스 생성 횟수 감소**, 대량 처리 시 유리
- `-ok <명령> {} \;` : `-exec` + 항목별 확인 프롬프트 (**ok**?)
- `-delete` : 일치 항목 삭제 (내부적으로 `-depth` 적용 → 디렉터리는 안부터). **조건 뒤에 배치** 필수
- `-print` : 경로 출력(기본 동작), `-print0` : NULL 구분자 출력 → `xargs -0` 와 짝
- `-printf '<형식>'` : `%p` 경로, `%f` 파일명, `%s` 바이트, `%u`/`%g` 소유자/그룹, `%m` 8진 권한, `%T@`·`%TY-%Tm-%Td` 수정 시각, `%d` 깊이 — **개행 `\n` 을 직접 넣어야 함**
- `-ls` : `ls -dils` 형식 상세 출력

**검증**
```bash
find . -name "*.c" -exec basename {} \; | sort | tr '\n' ' '; echo
find . -name "*.c" -printf '%f\n' | wc -l
```

```text
mod1.c mod2.c mod3.c mod4.c mod5.c
5
```

> 📝 **시험 포인트**: `-exec … {} \;` 의 `{}` 와 `\;` 는 빠뜨리면 오답(필기 R03-27 의 보기 ③ 이 `\;` 누락). `\;` = 항목마다, `+` = 묶어서. 공백 파일명은 `-print0 | xargs -0`.

### 3-5. 기출 단골 조합 3종 — 실행과 검증

> **상황**: 실기에서 반복 출제되는 세 가지 `find` 조합을 실습 데이터로 직접 돌려 본다. 시스템 파일이 지워지지 않도록 삭제 계열은 **실습 디렉터리 안에서만** 수행한다.

⚠️ 아래 ① 은 삭제 명령. 반드시 `-delete` 를 뺀 형태로 목록을 먼저 확인하고, 대상 경로가 `/srv/devteam/proj/logs` 인지 다시 본다.

```bash
# ▸ ① .log 이면서 7일 경과 → 삭제 (실기 R01-1)
cd /srv/devteam/proj/logs
find . -name "*.log" -type f -mtime +7                 # 1) 대상 확인 (동작 없음)
find . -name "*.log" -type f -mtime +7 -delete         # 2) 삭제 (GNU find 전용)
# 동일 결과의 이식성 있는 형태
# find . -name "*.log" -type f -mtime +7 -exec rm -f {} \;
# find . -name "*.log" -type f -mtime +7 -print0 | xargs -0 rm -f

# ▸ ② 100MB 초과 일반 파일 (실기 R04-1)
find /srv/devteam -type f -size +100M
find /home -type f -size +100M 2>/dev/null             # 기출 원문 형태

# ▸ ③ SetUID 가 설정된 일반 파일 (실기 R02-1·R05-3)
find / -perm -4000 -type f 2>/dev/null > /tmp/suid-$(date +%F).txt
wc -l /tmp/suid-$(date +%F).txt
```

- `-delete` : `find` 내장 삭제 — `rm` 프로세스를 만들지 않아 빠르나 **되돌릴 수 없음**
- `2>/dev/null > 파일` : 오류는 버리고 표준출력만 파일로 (리다이렉션 순서는 6절)
- SetUID 목록 파일화 → 주기적으로 `diff` 하면 신규 SetUID 파일 탐지(침해 점검, [[10-security-firewall-selinux|Part 10]])

**검증**
```bash
ls -l /srv/devteam/proj/logs/*.log 2>&1 | head -3     # old-app.log 는 사라져야 정상
find /srv/devteam -type f -size +100M -printf '%s %p\n'
head -3 /tmp/suid-$(date +%F).txt
```

```text
-rw-r--r--. 1 root root 0 ... /srv/devteam/proj/logs/mid-app.log
-rw-r--r--. 1 root root 0 ... /srv/devteam/proj/logs/newyear.log
125829120 /srv/devteam/proj/logs/huge.img
/usr/bin/su
/usr/bin/mount
/usr/bin/umount
```

> 📝 **시험 포인트**: 세 조합은 실기 단답으로 그대로 출제된다. ①은 `-name "*.log" -mtime +7 -exec rm {} \;` 또는 `-delete`, ②는 `-type f -size +100M`, ③은 `-perm -4000 -type f`. `-type f` 를 빠뜨리면 디렉터리까지 포함되어 감점.

### 3-6. locate · updatedb · /etc/updatedb.conf

> **상황**: `find` 는 매번 트리를 훑어 느리다. 파일명 색인 DB 기반의 `locate` 를 설치해 즉시 검색을 비교한다. RHEL 9 는 `mlocate` 대신 **`plocate`** 를 제공한다.

```bash
dnf install -y plocate
updatedb                                  # 색인 DB 생성/갱신 (최초 1회 필수)
locate numbers.txt                        # 색인에서 즉시 검색
locate -i README                          # 대소문자 무시
locate -b '\numbers.txt'                  # 경로 전체가 아닌 기본 이름(basename) 일치
locate -c conf                            # 일치 개수만
locate -l 5 passwd                        # 결과 5건으로 제한
locate -e /etc/passwd                     # 실제로 존재하는 항목만 (삭제된 잔여 색인 제외)
grep -vE '^\s*#|^\s*$' /etc/updatedb.conf # 색인 제외 규칙 확인
systemctl list-timers plocate-updatedb.timer   # 자동 갱신 타이머 (systemd)
```

- `updatedb` : 파일명 색인 DB 갱신 — plocate 의 DB 는 `/var/lib/plocate/plocate.db`
- `locate -i` : 대소문자 무시, `-b` : basename 일치, `-c` : 개수(**c**ount), `-l N` : 결과 제한(**l**imit), `-e` : 존재 확인(**e**xisting), `-r` : 정규표현식
- `/etc/updatedb.conf` 주요 키
  - `PRUNE_BIND_MOUNTS` : 바인드 마운트 중복 색인 제외 (`yes`)
  - `PRUNEFS` : 색인 제외 파일시스템 유형 (`nfs`, `tmpfs`, `proc` 등)
  - `PRUNENAMES` : 색인 제외 디렉터리 **이름** (`.git`, `.snapshot` 등)
  - `PRUNEPATHS` : 색인 제외 **경로** (`/tmp`, `/var/spool`, `/media` 등)
- 한계: **DB 갱신 시점 기준** → 방금 만든 파일은 `updatedb` 전까지 안 나옴 (`find` 는 실시간)

**검증**
```bash
touch /srv/devteam/proj/docs/justnow.txt
locate justnow.txt ; echo "exit=$?"       # 갱신 전 → 결과 없음, exit 1
updatedb && locate justnow.txt            # 갱신 후 → 경로 출력
locate /tmp/ | head -3                    # PRUNEPATHS 에 /tmp 가 있으면 결과 없음
```

```text
exit=1
/srv/devteam/proj/docs/justnow.txt
```

> 📝 **시험 포인트**: `locate` 는 DB 검색(빠름·최신성 없음), `find` 는 실시간 탐색(느림·정확). DB 갱신 명령은 `updatedb`, 설정 파일은 `/etc/updatedb.conf`. RHEL 9 패키지명이 `plocate` 로 바뀐 점도 확인.

### 3-7. which · whereis · type · command

> **상황**: 방금 설치한 명령의 실제 실행 파일 위치와, 셸 내장 명령인지 외부 명령인지를 구분한다.

```bash
which locate                  # PATH 상 실행 파일 경로
which -a python3 2>/dev/null  # PATH 상 모든 후보
whereis passwd                # 실행 파일 + 소스 + man 페이지
whereis -b passwd             # 바이너리만
whereis -m passwd             # man 페이지만
type cd                       # 셸 내장 여부 판별
type -a echo                  # 내장 + 외부 모두
type -t ls                    # 종류만 (alias/builtin/file/function/keyword)
command -v grep               # 이식성 있는 경로 조회 (POSIX)
hash -r                       # 명령 경로 캐시 초기화 (설치 직후 인식 안 될 때)
```

- `which` : `$PATH` 를 순서대로 훑어 첫 실행 파일 반환 (**which**), `-a` 는 전부
- `whereis` : 표준 경로에서 바이너리·소스·man 검색 — `-b` binary, `-s` source, `-m` manual
- `type` : **셸 관점** 판별 — alias·함수·내장·키워드·외부 파일 구분. `-a` 전부, `-t` 종류만
- `command -v` : POSIX 표준 경로 조회 — 스크립트 내 존재 확인에 권장
- `hash -r` : 셸의 명령 경로 해시 테이블 초기화

**검증**
```bash
type -t cd; type -t ls; type -t if
whereis -b locate
```

```text
builtin
file
keyword
locate: /usr/bin/locate
```

> 📝 **시험 포인트**: `which` 는 PATH 상 실행 파일만, `whereis` 는 man·소스까지, `type` 은 내장/별칭까지 구분. `cd` 는 `which cd` 로 안 나오고 `type cd` 로만 나온다는 점이 함정.

### 3-8. grep 옵션 총정리

> **상황**: 로그·설정 파일에서 원하는 행을 뽑는다. `grep` 은 실기·필기 모두 최빈출이므로 옵션을 한 번에 정리한다.

```bash
cd /srv/devteam/proj
grep apple docs/notes.txt          # 기본 — 일치 행 출력
grep -i apple docs/notes.txt       # 대소문자 무시 → Apple pie 포함
grep -v apple docs/notes.txt       # 일치하지 "않는" 행
grep -n apple docs/notes.txt       # 행 번호 표시
grep -c apple docs/notes.txt       # 일치 행 개수 (일치 "횟수" 아님)
grep -l apple docs/*               # 일치하는 파일명만
grep -L apple docs/*               # 일치하지 않는 파일명만
grep -r apple /srv/devteam         # 디렉터리 재귀
grep -rn --include="*.txt" apple /srv/devteam    # 확장자 한정 재귀
grep -rn --exclude-dir=logs apple /srv/devteam   # 특정 디렉터리 제외
grep -w apple docs/notes.txt       # 단어 단위 일치 (pineapple 제외)
grep -x "banana split" docs/notes.txt   # 행 전체 일치
grep -o 'ap[a-z]*' docs/notes.txt  # 일치한 부분만 출력
grep -E 'apple|grape' docs/notes.txt    # 확장 정규식 (OR)
grep -F 'a.b' docs/notes.txt       # 고정 문자열 — 메타문자 해석 안 함
grep -e apple -e grape docs/notes.txt   # 패턴 여러 개
printf 'apple\ngrape\n' > /tmp/pats.txt
grep -f /tmp/pats.txt docs/notes.txt    # 패턴을 파일에서 읽기
grep -A2 cherry docs/notes.txt     # 일치 행 + 뒤 2행
grep -B1 cherry docs/notes.txt     # 앞 1행 + 일치 행
grep -C1 cherry docs/notes.txt     # 앞뒤 1행씩
grep -q apple docs/notes.txt; echo "exit=$?"   # 출력 없이 종료 코드만
grep --color=auto apple docs/notes.txt          # 일치 부분 색상 (RHEL 기본 alias)
grep -a Failed /var/log/secure | head -2        # 바이너리 판정 파일도 텍스트 취급
grep -s apple /nonexistent; echo "exit=$?"      # 오류 메시지 억제
```

- `-i` 대소문자 무시(**i**gnore) · `-v` 반전(in**v**ert) · `-n` 행 번호(**n**umber) · `-c` 개수(**c**ount)
- `-l` 일치 파일명(**l**ist) · `-L` 불일치 파일명 · `-r` 재귀(**r**ecursive) · `-R` 재귀 + 심볼릭 링크 추적
- `-w` 단어 경계 일치(**w**ord) · `-x` 행 전체 일치(e**x**act line) · `-o` 일치 부분만(**o**nly)
- `-E` 확장 정규식(**E**xtended, = `egrep`) · `-F` 고정 문자열(**F**ixed, = `fgrep`) · `-G` 기본 정규식(기본값) · `-P` PCRE
- `-e <패턴>` 패턴 지정(여러 번 가능, `-` 로 시작하는 패턴에도 필수) · `-f <파일>` 패턴 목록 파일
- `-A N` 뒤 문맥(**A**fter) · `-B N` 앞 문맥(**B**efore) · `-C N` 양쪽(**C**ontext)
- `-q` 무출력(**q**uiet) · `-s` 오류 억제(**s**uppress) · `-a` 텍스트 강제(**a**ll text) · `--color=auto` 색상
- `--include=<글롭>` / `--exclude=<글롭>` / `--exclude-dir=<이름>` : 재귀 시 대상 한정
- **종료 코드**: `0` 일치 있음, `1` 일치 없음, `2` 오류 — `if grep -q …` 조건문의 근거

**검증**
```bash
grep -c apple docs/notes.txt        # 대소문자 구분 → 2
grep -ic apple docs/notes.txt       # 무시 → 3
grep -w apple docs/notes.txt | wc -l
grep -q zzz docs/notes.txt; echo "not found exit=$?"
```

```text
2
3
2
not found exit=1
```

> 📝 **시험 포인트**: `-c` 는 "일치한 **행 수**"(단어 출현 횟수 아님) — `cat access.log | grep 404 | wc -l` 과 같은 결과(필기 단답 R06-6). `grep 패턴 파일; echo $?` 가 `1` 이면 "오류 없이 못 찾음"(필기 R08-5). `-i`(R07-7), `-v`, `-n`, `-r` 는 단답 최빈출.

### 3-9. 정규표현식 — BRE 와 ERE

> **상황**: `grep`·`sed`(BRE) 와 `grep -E`·`awk`(ERE) 는 메타문자 이스케이프 규칙이 다르다. 같은 목적의 패턴을 두 문법으로 각각 확인한다.

```bash
cd /srv/devteam/proj
grep '^apple'  docs/notes.txt      # 행 시작
grep 'apple$'  docs/notes.txt      # 행 끝
grep '^$'      docs/notes.txt      # 빈 행
grep 'a.ple'   docs/notes.txt      # 임의의 1문자
grep '[Aa]pple' docs/notes.txt     # 문자 클래스
grep '[^a-z]'  docs/notes.txt      # 부정 클래스 (소문자 아닌 문자 포함 행)
grep '[[:digit:]]' /etc/passwd | head -2      # POSIX 클래스
grep '\bapple\b' docs/notes.txt    # 단어 경계 (GNU 확장)
grep '\<apple\>' docs/notes.txt    # 동일 (전통 표기)
grep 'ba*n' docs/notes.txt         # 0회 이상 반복
grep -E 'ba+n'  docs/notes.txt     # ERE: 1회 이상
grep    'ba\+n' docs/notes.txt     # BRE: 같은 뜻, 역슬래시 필요
grep -E 'colou?r' /etc/services    # ERE: 0 또는 1회
grep -E '^[a-z]{3,8}:' /etc/passwd | head -3   # ERE: 3~8자 반복
grep    '^[a-z]\{3,8\}:' /etc/passwd | head -3 # BRE: 같은 뜻
grep -E '(apple|grape)' docs/notes.txt         # ERE: 그룹 + 대체
grep    '\(apple\|grape\)' docs/notes.txt      # BRE: 같은 뜻
grep -E '^([0-9]{1,3}\.){3}[0-9]{1,3}$' <<< "192.168.64.10"   # 그룹 반복 (IPv4 형태)
```

| 요소 | BRE (`grep`, `sed`) | ERE (`grep -E`, `awk`, `sed -E`) | 의미 |
| --- | --- | --- | --- |
| 앵커 | `^` `$` | `^` `$` | 행 시작 / 행 끝 |
| 단어 경계 | `\b` `\<` `\>` | `\b` `\<` `\>` | 단어의 경계 (GNU 확장) |
| 임의 1문자 | `.` | `.` | 개행 제외 아무 문자 |
| 문자 클래스 | `[abc]` `[a-z]` | 동일 | 나열 중 1문자 |
| 부정 클래스 | `[^abc]` | 동일 | 나열 외 1문자 |
| POSIX 클래스 | `[[:digit:]]` `[[:alpha:]]` `[[:alnum:]]` `[[:space:]]` `[[:upper:]]` `[[:punct:]]` | 동일 | 문자 종류 집합 |
| 0회 이상 | `*` | `*` | 앞 요소 0회 이상 |
| 1회 이상 | `\+` | `+` | 앞 요소 1회 이상 |
| 0 또는 1회 | `\?` | `?` | 있어도 없어도 됨 |
| n~m회 | `\{n,m\}` | `{n,m}` | `\{3\}` 정확히 3회, `\{2,\}` 2회 이상 |
| 그룹 | `\( \)` | `( )` | 묶기 + 후방참조 대상 |
| 대체(OR) | `\|` | `\|` → `\|` | 좌우 중 하나 |
| 후방참조 | `\1` `\2` | `\1` `\2` | 앞 그룹이 일치한 문자열 재사용 |
| 이스케이프 | `\.` `\*` `\/` | 동일 | 메타문자를 문자 그대로 |

- **핵심 차이**: `+ ? { } ( ) |` 는 BRE 에서 `\` 를 붙여야 메타문자, ERE 에서는 그대로 메타문자
- 셸이 먼저 해석하지 않도록 패턴은 항상 **작은따옴표**로 감쌀 것
- 셸 글롭(`*` = 임의 문자열)과 정규식(`*` = 앞 문자 반복)의 의미는 **완전히 다름** — 최대 함정

**검증**
```bash
grep -c '^apple' docs/notes.txt        # apple 로 시작하는 행
echo "banana" | grep -E 'ba(na){2}'    # 그룹 2회 반복
echo "192.168.64.10" | grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}'
```

```text
1
banana
192.168.64.10
```

> 📝 **시험 포인트**: "`error` 로 시작하는 행" → `grep '^error'`(단답 R05-6), "`cat` 또는 `dog`" → `grep -E 'cat|dog'`(단답 R06-7). `^` 는 시작, `$` 는 끝, `[^…]` 는 클래스 내부에서만 부정.

### 3-10. 실전 — /var/log/secure 실패 로그 집계

> **상황**: 무차별 대입 시도를 점검한다. SSH 로 일부러 실패 로그인을 만든 뒤, 실패 행 수를 세고 출발지 IP·계정별로 집계한다. (`lastb` 는 [[01-vm-setup-and-inspection|Part 01]], 방화벽 차단은 [[10-security-firewall-selinux|Part 10]])

```bash
# 1) 실패 로그 생성 — 비밀번호를 3회 틀리게 입력 후 Ctrl+C
ssh -o PubkeyAuthentication=no baduser@127.0.0.1
ssh -o PubkeyAuthentication=no dev1@127.0.0.1

# 2) 실패 행 수 (실기 R03-1·R05-1 정답 형태)
grep -c 'Failed password' /var/log/secure

# 3) 출발지 IP 별 집계
grep 'Failed password' /var/log/secure \
  | grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' \
  | sort | uniq -c | sort -rn | head

# 4) 시도된 계정명 별 집계 (awk 로 "for" 다음 토큰 추출)
grep 'Failed password' /var/log/secure \
  | awk '{for(i=1;i<=NF;i++) if($i=="for") print $(i+1)}' \
  | sed 's/^invalid$//; /^$/d' | sort | uniq -c | sort -rn

# 5) 존재하지 않는 계정 시도만
grep -c 'Invalid user\|invalid user' /var/log/secure

# 6) 최근 실패 5건을 문맥과 함께
grep -n 'Failed password' /var/log/secure | tail -5
```

- `grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}'` : 일치 부분만(`-o`) ERE(`-E`)로 — "숫자 1~3자리 + 점" 을 3회 반복 후 마지막 옥텟
- `sort | uniq -c` : `uniq` 는 **인접한** 중복만 처리하므로 `sort` 선행 필수
- `sort -rn` : 숫자 역순 → 시도 횟수 많은 순 (**r**everse + **n**umeric)
- `awk '{for(i=1;i<=NF;i++) …}'` : 로그 형식이 `invalid user` 유무로 필드 위치가 달라지므로 토큰 위치를 고정하지 않고 탐색

**검증**
```bash
grep -c 'Failed password' /var/log/secure
grep 'Failed password' /var/log/secure | grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' | sort -u
lastb -n 5                     # /var/log/btmp 의 실패 이력과 교차 확인
```

```text
6
127.0.0.1
baduser  ssh:notty    127.0.0.1        ... - ...  (00:00)
dev1     ssh:notty    127.0.0.1        ... - ...  (00:00)
...
```

> 📝 **시험 포인트**: 실기 단답 "`Failed password` 포함 행의 **개수**만 출력" → `grep -c 'Failed password' /var/log/secure`. `-c` 대신 `| wc -l` 도 정답이나 옵션을 묻는 문제에서는 `-c`. 로그인 실패 **원장부**는 `/var/log/btmp`(`lastb`) 라는 짝도 함께 출제.

---

## 4. 텍스트 처리 — cut · awk · sed · sort · uniq · tr

### 4-1. cut — 구분자·필드·문자 위치

> **상황**: `/etc/passwd` 처럼 구분자가 명확한 파일에서 특정 열만 뽑는다. `cut` 은 가장 가벼운 필드 추출 도구다.

```bash
cut -d: -f1 /etc/passwd | head -5            # 1번 필드(계정명)
cut -d: -f1,3 /etc/passwd | head -5          # 1·3번 필드
cut -d: -f1-3 /etc/passwd | head -3          # 1~3번 필드 범위
cut -d: -f6- /etc/passwd | head -3           # 6번부터 끝까지
cut -d: -f1,7 --output-delimiter=' -> ' /etc/passwd | head -3   # 출력 구분자 변경
cut -d: --complement -f2 /etc/passwd | head -2    # 2번 필드만 제외
cut -c1-10 /etc/passwd | head -3             # 문자 위치 1~10
cut -c1,5,9 /etc/passwd | head -2            # 개별 문자 위치
cut -b1-4 /srv/devteam/proj/docs/README.md   # 바이트 위치
grep -v '^#' /etc/fstab | cut -f1 -s         # 기본 구분자(TAB), 구분자 없는 행 제외
```

- `-d <문자>` : 필드 구분자 (**d**elimiter) — **한 글자만**, 기본값은 TAB
- `-f <목록>` : 필드 번호 (**f**ield) — `1,3` 나열, `1-3` 범위, `6-` 이후 전부
- `-c <목록>` : 문자 위치 (**c**haracter), `-b` : 바이트 위치 (**b**yte)
- `--complement` : 지정한 것을 **제외**한 나머지
- `--output-delimiter=<문자열>` : 출력 시 구분자 교체
- `-s` : 구분자가 없는 행은 출력하지 않음 (**s**uppress)
- 한계: **연속 공백을 하나로 묶지 못함** → `ps`·`df` 같은 가변 공백 출력은 `awk` 사용

**검증**
```bash
cut -d: -f1,3 /etc/passwd | grep -E ':(0|100[0-9])$'
cut -d: -f1 /etc/passwd | wc -l
wc -l < /etc/passwd                       # 두 값이 같아야 정상
```

```text
root:0
admin1:1000
...
```

> 📝 **시험 포인트**: `cut -d: -f1` 이 계정명, `-f3` 이 UID, `-f7` 이 로그인 셸. `-d` 는 한 글자 구분자만 가능하며 공백 여러 칸은 처리 못 한다는 점이 `awk` 와의 차이.

### 4-2. awk 기본 — 필드·NR·NF·구분자

> **상황**: `/etc/passwd` 와 명령 출력에서 필드를 골라 낸다. `awk` 는 "행을 필드로 쪼개 조건별 처리" 하는 미니 언어다.

```bash
awk -F: '{print $1}' /etc/passwd | head -3           # 1번 필드
awk -F: '{print $1, $3}' /etc/passwd | head -3       # 쉼표 = OFS(기본 공백)
awk -F: '{print $1 ":" $7}' /etc/passwd | head -3    # 문자열 연결(공백 없이 나열)
awk -F: '{print NF}' /etc/passwd | head -2           # 필드 개수 (passwd = 7)
awk -F: '{print NR, $1}' /etc/passwd | head -3       # 행 번호
awk -F: 'NR==3 {print $0}' /etc/passwd               # 3번째 행 전체
awk -F: 'NR>=2 && NR<=4' /etc/passwd                 # 액션 생략 시 기본 {print $0}
awk -F: '{print $NF}' /etc/passwd | head -3          # 마지막 필드
awk -F: '{print $(NF-1)}' /etc/passwd | head -3      # 끝에서 두 번째
awk '{print $3}' /srv/devteam/proj/docs/notes.txt    # 기본 구분자 = 연속 공백·탭
df -hT | awk 'NR>1 {print $1, $6}'                   # 가변 공백 출력 처리 (cut 불가)
awk -F: -v OFS=' | ' '{print $1, $3, $7}' /etc/passwd | head -3   # 출력 구분자 지정
awk -F'[:,]' '{print $1}' /etc/group | head -3       # 구분자에 정규식 사용
```

- `-F <구분자>` : 입력 필드 구분자 (**F**ield separator) — `-F:` `-F'\t'` `-F'[:,]'`(정규식) 가능
- `$0` 전체 행, `$1`~`$n` n번째 필드, `$NF` 마지막 필드, `$(NF-1)` 끝에서 둘째
- `NF` : 현재 행의 **필드 개수** (**N**umber of **F**ields)
- `NR` : 지금까지 읽은 **누적 행 번호** (**N**umber of **R**ecords), `FNR` : 현재 파일 내 행 번호
- `FS`/`OFS`/`RS`/`ORS` : 입력·출력 필드 구분자, 입력·출력 레코드(행) 구분자
- `-v <변수>=<값>` : 셸 값을 awk 변수로 전달 (**v**ariable) — `-v OFS=…` 처럼 내장 변수도 지정 가능
- `print $1, $2` (쉼표) → OFS 삽입 / `print $1 $2` (나열) → 붙여서 출력

**검증**
```bash
awk -F: 'END{print NR}' /etc/passwd     # 전체 행 수
wc -l < /etc/passwd                     # 같은 값
awk -F: '{print NF}' /etc/passwd | sort -u   # 전 행이 7 필드여야 정상
```

```text
26
26
7
```

> 📝 **시험 포인트**: `awk '{print $3}'` = 공백 기준 3번째 필드(단답 R05-8). `$0` 은 전체 행, `NF` 는 필드 수, `NR` 은 행 번호. `-F:` 로 콜론 구분 지정은 실기 단골.

### 4-3. awk 심화 — 패턴·조건·BEGIN/END·printf·내장 함수

> **상황**: 단순 추출을 넘어 조건 필터·서식 출력·집계까지 한다. 실기 최빈출인 "UID 1000 이상 계정명" 을 여러 형태로 작성한다.

```bash
# ▸ 조건 (실기 R03-2·R06-5 정답 형태)
awk -F: '$3>=1000 {print $1}' /etc/passwd
awk -F: '$3>=1000 && $3<65534 {print $1, $3}' /etc/passwd
awk -F: '$7=="/sbin/nologin" {print $1}' /etc/passwd | head -5
awk -F: '$1 ~ /^dev/ {print $1, $3}' /etc/passwd        # 정규식 일치
awk -F: '$1 !~ /^(root|bin|daemon)$/ {c++} END{print c}' /etc/passwd   # 불일치
awk '/apple/{print NR": "$0}' /srv/devteam/proj/docs/notes.txt   # 패턴 { 액션 }
awk '!/^#/ && NF>0' /etc/fstab                          # 주석·빈 행 제외

# ▸ BEGIN / END 와 서식 출력
awk -F: 'BEGIN{printf "%-12s %6s %s\n","USER","UID","SHELL"}
         $3>=1000 {printf "%-12s %6d %s\n",$1,$3,$7}
         END{printf "총 %d 행 처리\n", NR}' /etc/passwd

# ▸ 집계 (연관 배열)
awk -F: '{cnt[$7]++} END{for(s in cnt) print cnt[s], s}' /etc/passwd | sort -rn
awk -F: '{sum+=$3} END{print "UID 합계:", sum, "평균:", sum/NR}' /etc/passwd

# ▸ -v 로 셸 값 전달
MIN=1000
awk -F: -v min="$MIN" '$3>=min {print $1}' /etc/passwd

# ▸ 내장 함수
awk -F: '{print $1, length($1)}' /etc/passwd | head -3          # 문자열 길이
awk -F: '{print substr($1,1,3)}' /etc/passwd | head -3          # 부분 문자열
awk -F: '{n=split($7,a,"/"); print a[n]}' /etc/passwd | head -3 # 분할 후 마지막 조각
awk '{print toupper($1)}' /srv/devteam/proj/docs/notes.txt | head -3
awk -F: '{gsub(/bin/,"BIN",$7); print $7}' /etc/passwd | head -3
```

- `패턴 { 액션 }` : 패턴 생략 시 전 행, 액션 생략 시 `{print $0}`
- `BEGIN{}` : 입력 읽기 **전** 1회 — 헤더 출력·`FS` 설정용
- `END{}` : 입력을 다 읽은 **후** 1회 — 합계·개수 출력용
- `printf "<서식>", 값…` : `%s` 문자열, `%d` 정수, `%f` 실수, `%-12s` 좌측 정렬 12칸, `%6d` 우측 정렬 6칸. **개행은 `\n` 직접 지정**
- 비교 연산: `== != < <= > >=`, 논리: `&& || !`, 정규식 일치: `~`, 불일치: `!~`
- 연관 배열 `cnt[키]++` + `for(k in cnt)` : awk 만으로 집계 가능
- 내장 함수: `length(s)` 길이, `substr(s,m,n)` m번째부터 n자, `split(s,arr,sep)` 분할(반환값=조각 수), `toupper`/`tolower`, `index(s,t)` 위치, `sub`/`gsub(정규식,대체[,대상])` 치환, `sprintf`, `int`

**검증**
```bash
awk -F: '$3>=1000 {print $1}' /etc/passwd
getent passwd | awk -F: '$3>=1000 {c++} END{print c" 명"}'
```

```text
admin1
dev1
dev2
ops1
guest1
nobody
6 명
```

> 📝 **시험 포인트**: `awk -F: '$3>=1000 {print $1}' /etc/passwd` 는 실기 단답으로 두 회차(R03-2, R06-5) 반복 출제. `BEGIN`/`END` 실행 시점, `NR` vs `NF` 혼동이 필기 함정.

### 4-4. sed — 출력·삭제·치환

> **상황**: 파일을 열지 않고 행 단위로 조회·삭제·치환한다. 먼저 **원본을 건드리지 않는** 형태로 결과를 확인한다.

```bash
cd /srv/devteam/proj
sed -n '3p' docs/notes.txt              # 3행만 출력
sed -n '3,5p' docs/notes.txt            # 3~5행
sed -n '$p' docs/notes.txt              # 마지막 행
sed -n '2,$p' docs/notes.txt            # 2행부터 끝까지
sed -n '/banana/p' docs/notes.txt       # 패턴 일치 행
sed -n '/^#/,/^$/p' docs/notes.txt      # 패턴 시작 ~ 패턴 끝 범위
sed -n '0~2p' src/numbers.txt | head -3 # 2행마다 (GNU 확장, 짝수 행)
sed -n '1~3p' src/numbers.txt | head -3 # 1행부터 3행 간격

sed '3d' docs/notes.txt                 # 3행 삭제 (단답 R07-8)
sed '2,4d' docs/notes.txt               # 범위 삭제
sed '$d' docs/notes.txt                 # 마지막 행 삭제
sed '/^#/d' docs/notes.txt              # 주석 행 삭제
sed '/^#/d;/^$/d' docs/notes.txt        # 주석 + 빈 행 삭제 (설정 파일 정리 관용구)
sed -n '/apple/!p' docs/notes.txt       # 일치하지 "않는" 행만 (! = 부정)

sed 's/apple/orange/'  docs/notes.txt   # 각 행의 첫 일치만 치환
sed 's/apple/orange/g' docs/notes.txt   # 행 내 모든 일치 (단답 R05-7)
sed 's/apple/orange/2' docs/notes.txt   # 행 내 2번째 일치만
sed 's/apple/orange/gi' docs/notes.txt  # 대소문자 무시 + 전역
sed 's/^/> /' docs/notes.txt            # 각 행 앞에 문자열 삽입
sed 's/[[:space:]]*$//' docs/notes.txt  # 행말 공백 제거
sed -E 's/(apple) (banana)/\2 \1/' docs/notes.txt   # ERE 그룹 + 후방참조로 순서 교환
sed 's|/usr/local|/opt|g' <<< "/usr/local/bin"      # 구분자를 | 로 교체 (경로 치환)
```

- `-n` : 자동 출력 억제 (**n**o auto-print) — `p` 와 짝. `-n` 없이 `p` 를 쓰면 **2회 출력**
- 주소: `N` 행 번호, `N,M` 범위, `$` 마지막 행, `/패턴/` 정규식, `/p1/,/p2/` 패턴 범위, `N~M` N행부터 M간격(GNU), `주소!` 부정
- `p` 출력(**p**rint), `d` 삭제(**d**elete), `s///` 치환(**s**ubstitute), `=` 행 번호 출력, `q` 종료
- `s` 플래그: `g` 행 전체(**g**lobal), `N` N번째 일치만, `i`/`I` 대소문자 무시, `p` 출력, `w 파일` 치환된 행을 파일로
- `\1`~`\9` : 그룹(`\( \)` 또는 `-E` 의 `( )`)이 잡은 문자열 재사용, `&` : 일치한 전체 문자열
- 구분자는 `/` 대신 아무 문자나 가능 — `s|a|b|`, `s#a#b#` (경로 치환 시 가독성 향상)
- `-E`(= `-r`) : 확장 정규식 사용

**검증**
```bash
sed -n '2p' docs/notes.txt
sed '/^#/d;/^$/d' docs/notes.txt | wc -l     # 원본 7행 → 주석 1 + 빈 행 1 제거 = 5
sed 's/apple/orange/g' docs/notes.txt | grep -c orange
cat docs/notes.txt | grep -c apple           # 원본은 그대로 (파일 미변경 확인)
```

```text
apple banana cherry
5
2
2
```

> 📝 **시험 포인트**: `sed '3d'`(3행 삭제, 단답 R07-8), `sed 's/apple/orange/g'`(전역 치환, 단답 R05-7), `sed -n '3,5p'`(범위 출력). `g` 플래그가 없으면 **행마다 첫 일치만** 치환된다는 점이 최대 함정.

### 4-5. sed -i — 원본 직접 수정과 백업

> **상황**: 실기 최빈출인 "설정 파일 in-place 치환" 을 수행한다. 되돌릴 수 없으므로 **백업 확장자**를 함께 익힌다.

⚠️ `sed -i` 는 원본을 즉시 덮어쓴다. 시스템 설정 파일에는 반드시 `-i.bak` 을 붙이거나 `cp` 로 먼저 백업한다.

```bash
cd /srv/devteam/proj/docs
cp notes.txt notes.work.txt
sed -i 's/apple/orange/g' notes.work.txt          # 백업 없이 덮어쓰기
sed -i.bak 's/orange/apple/g' notes.work.txt      # notes.work.txt.bak 자동 생성
sed -i -e 's/banana/melon/' -e '/^$/d' notes.work.txt   # -e 로 스크립트 여러 개
sed -i '2i\새로 삽입한 행' notes.work.txt          # 2행 "앞"에 삽입 (insert)
sed -i '2a\뒤에 붙인 행'   notes.work.txt          # 2행 "뒤"에 추가 (append)
sed -i '3c\3행을 통째로 교체' notes.work.txt       # 행 교체 (change)
sed -i 'y/abc/ABC/' notes.work.txt                # 문자 1:1 변환 (tr 과 동일 개념)

# ▸ 실기 기출 형태 (R06-1) — 대상 파일이 없으면 사본으로 실습
cp /etc/httpd/conf/httpd.conf /tmp/httpd.conf 2>/dev/null || printf 'Listen 80\n' > /tmp/httpd.conf
sed -i 's/Listen 80/Listen 8080/' /tmp/httpd.conf
sed -i.bak 's/^#\(ServerName\)/\1/' /tmp/httpd.conf 2>/dev/null   # 주석 해제 관용구
```

- `-i[SUFFIX]` : 파일 직접 수정 (**i**n-place). `-i.bak` 처럼 붙여 쓰면 원본을 `파일.bak` 으로 보존. **`-i` 와 접미사 사이에 공백을 두면 안 됨**
- `-e <스크립트>` : 스크립트를 여러 개 지정 (**e**xpression). `;` 로 이어 쓰는 것과 동등
- `-f <파일>` : 스크립트를 파일에서 읽기
- `a\<텍스트>` 뒤에 추가(**a**ppend) · `i\<텍스트>` 앞에 삽입(**i**nsert) · `c\<텍스트>` 행 교체(**c**hange)
- `y/원본집합/대상집합/` : 문자 단위 1:1 변환 (길이가 같아야 함)
- GNU sed 는 `sed -i '…'`, macOS(BSD) sed 는 `sed -i '' '…'` — 문법이 달라 스크립트 이식 시 주의

**검증**
```bash
ls -l notes.work.txt notes.work.txt.bak
diff notes.work.txt.bak notes.work.txt | head -5
grep -n Listen /tmp/httpd.conf
```

```text
-rw-r--r--. 1 root root ... notes.work.txt
-rw-r--r--. 1 root root ... notes.work.txt.bak
2a3
> 뒤에 붙인 행
Listen 8080
```

> 📝 **시험 포인트**: "파일에 직접 치환 저장" = `sed -i 's/찾을것/바꿀것/' 파일`(실기 R06-1, R03-7). `-i` 없이는 화면 출력만 되고 파일은 그대로. 백업본 생성은 `-i.bak`.

### 4-6. sort — 정렬 전 옵션

> **상황**: 집계 파이프라인의 필수 전처리. 숫자·필드·용량 단위·역순 정렬을 각각 확인한다.

```bash
cd /srv/devteam/proj
sort docs/notes.txt                        # 사전순 (기본)
sort -r docs/notes.txt                     # 역순
sort src/numbers.txt | head -3             # 문자열 정렬 → 1, 10, 100 순
sort -n src/numbers.txt | head -3          # 숫자 정렬 → 1, 2, 3
sort -nr src/numbers.txt | head -3         # 숫자 역순
sort -u src/even.txt                       # 중복 제거 후 정렬
sort -f docs/notes.txt                     # 대소문자 무시 (fold)
sort -t: -k3 -n /etc/passwd | head -3      # 구분자 : + 3번 필드 숫자 정렬
sort -t: -k3,3nr /etc/passwd | head -3     # 3번 필드만 키로, 숫자 역순
sort -t, -k3,3nr -k2,2 docs/score.csv      # 다중 키 (3번 숫자역순 → 동점은 2번 사전순)
du -sh /srv/devteam/proj/* | sort -h       # K/M/G 단위 인식 정렬
ls -l /var/log | awk 'NR>1{print $6, $7, $9}' | sort -M | head -3   # 월 이름 정렬
sort -c src/numbers.txt 2>&1 | head -1     # 정렬 여부 검사 (미정렬이면 오류)
sort -n src/numbers.txt -o /tmp/sorted.txt # 결과를 파일로 (입력=출력 가능)
sort -V <<< $'v1.10\nv1.9\nv1.2'           # 버전 문자열 정렬
```

- `-n` 숫자 값 기준(**n**umeric) · `-r` 역순(**r**everse) · `-u` 중복 제거(**u**nique) · `-f` 대소문자 무시(**f**old)
- `-t <구분자>` 필드 구분자(**t**erminator) · `-k <시작[,끝]>` 정렬 키 필드(**k**ey) — `-k3,3` 처럼 끝을 지정해야 그 필드만 사용
- `-h` 사람이 읽는 단위(K/M/G) 인식(**h**uman-numeric) · `-M` 월 이름(JAN…DEC, **M**onth) · `-V` 버전 문자열(**V**ersion)
- `-b` 선행 공백 무시(**b**lank) · `-s` 안정 정렬(**s**table) · `-c` 정렬 여부 검사(**c**heck) · `-o <파일>` 출력 파일(**o**utput)
- 키에 옵션 부착 가능: `-k3,3nr` = "3번 필드를 숫자 역순으로"

**검증**
```bash
sort src/numbers.txt | head -3 | tr '\n' ' '; echo    # 문자열 정렬
sort -n src/numbers.txt | head -3 | tr '\n' ' '; echo # 숫자 정렬
sort -t: -k3 -nr /etc/passwd | head -1 | cut -d: -f1,3
```

```text
1 10 100 
1 2 3 
nobody:65534
```

> 📝 **시험 포인트**: `-n` 없이는 `10` 이 `9` 보다 앞에 오는 사전순 정렬. `sort | uniq -c | sort -nr` 은 "빈도 상위" 관용구로 필기 지문에 그대로 등장(R02-11).

### 4-7. uniq — 중복 처리

> **상황**: 정렬된 입력에서 중복 행을 세거나 걸러낸다. `uniq` 는 **인접한** 중복만 본다는 제약을 실증한다.

```bash
printf 'a\nb\na\nb\nb\nC\nc\n' > /tmp/u.txt
uniq /tmp/u.txt                 # 정렬 안 된 입력 → 인접 중복만 제거
sort /tmp/u.txt | uniq          # 올바른 사용
sort /tmp/u.txt | uniq -c       # 각 행의 출현 횟수
sort /tmp/u.txt | uniq -d       # 2회 이상 나온 행만
sort /tmp/u.txt | uniq -D       # 중복 행 전부 (그룹째)
sort /tmp/u.txt | uniq -u       # 딱 1회만 나온 행
sort -f /tmp/u.txt | uniq -ci   # 대소문자 무시 비교 + 개수
sort /tmp/u.txt | uniq -c | sort -rn      # 빈도 상위 정렬
cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn   # 셸별 계정 수
```

- `-c` : 각 행 앞에 출현 횟수 (**c**ount)
- `-d` : 중복된 행을 **한 번씩만** 출력 (**d**uplicated), `-D` : 중복 행 전부 출력
- `-u` : 중복이 없는 행만 (**u**nique)
- `-i` : 대소문자 무시 비교 (**i**gnore case)
- `-f N` : 앞 N개 필드 건너뛰고 비교 (**f**ields), `-s N` : 앞 N개 문자 건너뜀 (**s**kip), `-w N` : 앞 N개 문자만 비교 (**w**idth)
- **전제**: 입력이 정렬되어 있어야 함 → 관용적으로 `sort | uniq` 로 사용 (`sort -u` 는 개수 집계 불가)

**검증**
```bash
sort /tmp/u.txt | uniq -c
```

```text
      2 a
      3 b
      1 C
      1 c
```

> 📝 **시험 포인트**: `uniq` 단독 사용 시 정렬되지 않은 입력에서는 중복이 남는다. `sort -u` 는 "중복 제거"만, 횟수를 세려면 `sort | uniq -c`.

### 4-8. tr — 문자 변환·삭제·압축

> **상황**: 대소문자 변환, 개행 제거, 제어문자 정리 등 문자 단위 가공을 한다. `tr` 은 **표준입력만** 받는다(파일 인자 없음).

```bash
echo "hello linux" | tr 'a-z' 'A-Z'            # 소문자 → 대문자
echo "HELLO" | tr '[:upper:]' '[:lower:]'      # POSIX 클래스 사용
tr 'a-z' 'A-Z' < /srv/devteam/proj/docs/README.md   # 파일은 리다이렉션으로 입력
echo "2026-09-04" | tr '-' '/'                 # 문자 치환
echo "a1b2c3" | tr -d '0-9'                    # 숫자 삭제
echo "hello    world" | tr -s ' '              # 연속 공백 1개로 압축
cat /srv/devteam/proj/src/even.txt | tr '\n' ' '; echo   # 개행 → 공백
echo "hello, world!" | tr -cd '[:alnum:]\n'    # 영숫자·개행 외 전부 삭제
echo "hello" | tr -c 'a-z' '*'                 # 지정 집합의 여집합을 * 로
cut -d: -f1 /etc/passwd | tr '\n' ':' | head -c 60; echo
tr -s '\n' < /etc/fstab | head -3              # 연속 빈 행 압축
```

- `tr <집합1> <집합2>` : 집합1의 각 문자를 같은 위치의 집합2 문자로 치환
- `-d <집합>` : 해당 문자 삭제 (**d**elete)
- `-s <집합>` : 연속 반복 문자를 1개로 압축 (**s**queeze)
- `-c` : 집합의 **여집합** 대상 (**c**omplement) — `-cd` 조합이 "허용 문자 외 전부 삭제"
- `-t` : 집합1이 길면 잘라 맞춤 (**t**runcate)
- 문자 클래스: `[:alpha:]` `[:digit:]` `[:alnum:]` `[:space:]` `[:upper:]` `[:lower:]` `[:punct:]`
- 이스케이프: `\n` 개행, `\t` 탭, `\\` 역슬래시

**검증**
```bash
echo "Rocky Linux 9" | tr 'a-z' 'A-Z'
echo "aaabbbccc" | tr -s 'abc'
printf '1\n2\n3\n' | tr -d '\n'; echo
```

```text
ROCKY LINUX 9
abc
123
```

> 📝 **시험 포인트**: `tr 'a-z' 'A-Z'` 대소문자 변환은 단답 최빈출. `tr` 은 파일명을 인자로 받지 않으므로 `tr … file` 은 오답 — `tr … < file` 또는 파이프.

### 4-9. paste · join · comm — 파일 합치기·대조

> **상황**: 두 개의 목록 파일을 열로 붙이고, 공통 키로 결합하고, 차집합·교집합을 구한다.

```bash
cd /tmp
printf 'dev1\ndev2\nops1\n' > users.txt
printf '2001\n2002\n2003\n' > uids.txt
paste users.txt uids.txt                 # 탭으로 나란히 붙이기
paste -d: users.txt uids.txt             # 구분자 지정
paste -s -d, users.txt                   # 한 행으로 이어 붙이기 (serial)

printf 'dev1:개발팀\ndev2:개발팀\nops1:운영팀\n' | sort > role.txt
printf 'dev1:2001\ndev2:2002\nguest1:2004\n' | sort > uid.txt
join -t: role.txt uid.txt                # 1번 필드 공통 키로 결합 (내부 조인)
join -t: -1 1 -2 1 role.txt uid.txt      # 조인 필드 명시
join -t: -a1 -e '(없음)' -o '0,1.2,2.2' role.txt uid.txt   # 왼쪽 외부 조인
join -t: -v2 role.txt uid.txt            # uid.txt 에만 있는 행

cut -d: -f1 role.txt | sort > a.txt
cut -d: -f1 uid.txt  | sort > b.txt
comm a.txt b.txt                         # 3열: a만 / b만 / 공통
comm -12 a.txt b.txt                     # 공통(교집합)만
comm -23 a.txt b.txt                     # a 에만 있는 것 (좌차집합)
comm -13 a.txt b.txt                     # b 에만 있는 것 (우차집합)
comm -3  a.txt b.txt                     # 공통 제외 (대칭 차집합)
```

- `paste [-d 구분자] 파일…` : 행 번호가 같은 줄끼리 옆으로 결합, `-s` 는 파일 단위로 한 행에 이어 붙임 (**s**erial)
- `join -t <구분자> -1 <필드> -2 <필드> 파일1 파일2` : 공통 키 기준 결합 — **양쪽 파일이 키로 정렬되어 있어야 함**
  - `-a1`/`-a2` : 해당 파일의 미일치 행도 출력(외부 조인, **a**ll), `-v1`/`-v2` : 미일치 행만
  - `-e <문자열>` : 결측 필드 대체값(**e**mpty), `-o <목록>` : 출력 필드 순서 (`0`=키, `1.2`=파일1의 2번 필드)
- `comm 파일1 파일2` : **정렬된** 두 파일을 비교해 3개 열로 출력 — `-1`/`-2`/`-3` 은 해당 열 **억제**
  - `-12` 공통만, `-23` 파일1 전용, `-13` 파일2 전용

**검증**
```bash
paste -d: users.txt uids.txt
comm -12 a.txt b.txt
```

```text
dev1:2001
dev2:2002
ops1:2003
dev1
dev2
```

> 📝 **시험 포인트**: `comm` 의 옵션 숫자는 "출력할 열"이 아니라 "**억제할** 열" — `-12` 가 교집합이 되는 이유. `join`·`comm` 모두 정렬 전제.

### 4-10. 서식 정리 — expand · unexpand · fmt · fold · column · nl

> **상황**: 리포트로 붙일 텍스트의 탭·행 길이·열 정렬을 다듬는다.

```bash
cd /tmp
printf 'id\tname\tscore\n1\tkim\t90\n2\tlee\t85\n' > tabbed.txt
cat -A tabbed.txt | head -2          # ^I 로 탭 확인
expand -t 8 tabbed.txt | cat -A | head -2     # 탭 → 공백 8칸
expand -t 4 tabbed.txt                        # 탭 폭 4로
unexpand -a -t 4 <(expand -t 4 tabbed.txt) | cat -A | head -2   # 공백 → 탭 복원

fold -w 20 /srv/devteam/proj/docs/notes.txt | head -5   # 20칸에서 강제 줄바꿈
fold -w 20 -s /srv/devteam/proj/docs/notes.txt | head -5 # 단어 경계에서 줄바꿈
fmt -w 30 /srv/devteam/proj/docs/notes.txt | head -5     # 문단 재배치

column -t -s, /srv/devteam/proj/docs/score.csv   # CSV 를 열 맞춤 표로
mount | column -t | head -3                      # 공백 구분 출력 정렬
df -hT | column -t | head -4

nl /srv/devteam/proj/docs/notes.txt          # 비어있지 않은 행에만 번호
nl -b a /srv/devteam/proj/docs/notes.txt     # 모든 행에 번호
nl -b a -n rz -w 3 -s ': ' /srv/devteam/proj/docs/notes.txt   # 0채움 3자리 + 구분자
```

- `expand -t N` : 탭을 공백 N칸으로 (**t**abs), `unexpand -a -t N` : 공백을 탭으로 되돌림 (`-a` = 선행 공백 외에도 전부)
- `fold -w N` : N열마다 강제 개행 (**w**idth), `-s` : 공백(단어 경계)에서 끊음 (**s**paces), `-b` : 바이트 기준
- `fmt -w N` : 단어 단위로 문단을 N열 폭에 맞춰 재배치 (**w**idth) — 문장형 텍스트용
- `column -t` : 입력을 열 맞춤 표로 (**t**able), `-s <구분자>` : 입력 구분자, `-o <구분자>` : 출력 구분자, `-N <이름들>` : 열 이름 부여
- `nl -b <스타일>` : 번호 부여 대상 — `a` 전 행, `t` 비어있지 않은 행(기본), `n` 없음, `pREGEX` 패턴 일치 행
- `nl -n <형식>` : `ln` 좌측정렬, `rn` 우측정렬, `rz` 우측정렬 0채움 / `-w N` 번호 폭 / `-s <문자열>` 번호와 본문 사이 구분자

**검증**
```bash
column -t -s, /srv/devteam/proj/docs/score.csv
nl -b a -n rz -w 3 /srv/devteam/proj/docs/README.md
```

```text
id  name  score
1   kim   90
2   lee   85
3   park  77
001	Sample Project v1.0
```

> 📝 **시험 포인트**: `nl` 은 기본적으로 **빈 행에 번호를 안 붙임** → `cat -n` 과의 차이가 출제 포인트. `column -t` 는 보고서 가독성 향상 용도.

### 4-11. xargs — 표준입력을 명령 인자로

> **상황**: `find`·`grep` 결과를 다른 명령의 인자로 넘긴다. 공백 파일명·인자 개수·병렬 실행을 다룬다.

```bash
cd /srv/devteam/proj
find . -name "*.c" | xargs ls -l                  # 목록을 ls 인자로
find . -name "*.c" | xargs -n 2 echo "[batch]"    # 한 번에 인자 2개씩
find . -name "*.c" | xargs -I{} basename {} .c    # {} 자리에 항목 1개씩 대입
find . -name "*.c" | xargs -I{} cp {} /tmp/       # 항목별 명령 실행
find . -name "*.log" -print0 | xargs -0 ls -l 2>/dev/null   # NULL 구분 (공백 안전)
find . -name "*.c" | xargs -t wc -l               # 실행할 명령을 화면에 표시
find . -name "*.nonexistent" | xargs -r rm        # 입력 없으면 아예 실행 안 함
seq 1 8 | xargs -n 1 -P 4 -I{} sh -c 'sleep 0.2; echo done {}'   # 4개 병렬
cut -d: -f1 /etc/passwd | head -3 | xargs -p id   # 실행 전 확인 프롬프트
echo "1 2 3" | xargs -d ' ' -n 1 echo             # 구분자 직접 지정
```

- `-n N` : 명령 1회당 인자 개수 (**n**umber)
- `-I <치환문자>` : 치환 문자열 지정 — 항목마다 1회 실행, 인자 위치를 자유롭게 배치 (`-I{}`)
- `-0` : NULL 구분자 입력 (**0** = `\0`) — `find -print0` 과 짝, 공백·개행 포함 파일명 대응
- `-r` / `--no-run-if-empty` : 입력이 비면 명령을 실행하지 않음 (GNU)
- `-p` : 실행 전 확인 프롬프트 (**p**rompt), `-t` : 실행할 명령을 stderr 에 표시 (**t**race)
- `-P N` : 동시 실행 프로세스 수 (**P**arallel), `-d <문자>` : 입력 구분자 지정
- `-a <파일>` : 표준입력 대신 파일에서 인자 읽기
- `find … -exec … +` 와 기능이 겹침 — `xargs` 는 파이프 중간에서 유연, `-exec` 는 파일명 안전성이 기본 보장

**검증**
```bash
find . -name "*.c" | xargs -I{} basename {} .c | sort | tr '\n' ' '; echo
ls /tmp/mod*.c | wc -l
rm -f /tmp/mod*.c
```

```text
mod1 mod2 mod3 mod4 mod5 
5
```

> 📝 **시험 포인트**: `find … -print0 | xargs -0` 조합은 공백 포함 파일명 처리의 표준 답. `-I{}` 는 항목마다 1회 실행이라 대량 처리 시 느리다는 점도 비교 대상.

### 4-12. tee · printf · wc 조합

> **상황**: 파이프 중간 결과를 파일로도 남기고, 집계 결과를 서식화해 출력한다.

```bash
cd /srv/devteam/proj
grep -c apple docs/notes.txt | tee /tmp/apple.cnt        # 화면 + 파일 동시
grep apple docs/notes.txt | tee -a /tmp/apple.log | wc -l  # 이어쓰기 + 다음 파이프로
ls /etc | tee /tmp/etc1.txt /tmp/etc2.txt > /dev/null    # 여러 파일에 동시 기록
du -sh /srv/devteam/proj/* | tee /tmp/du.txt | sort -h | tail -2

printf '%-10s %5s\n' USER UID                            # 좌/우 정렬 서식
printf '%-10s %5d\n' dev1 2001
printf '%s=%s\n' host srv01 domain lab.local             # 인자가 남으면 서식 재사용
printf '%.2f\n' 3.14159                                  # 소수 2자리
printf '%05d\n' 42                                       # 0 채움
printf 'A\tB\n1\t2\n' | column -t

wc -l docs/notes.txt        # 행 수
wc -w docs/notes.txt        # 단어 수
wc -c docs/notes.txt        # 바이트 수
wc -m docs/notes.txt        # 문자 수 (한글은 -c 와 다름)
wc -L docs/notes.txt        # 가장 긴 행의 길이
wc -l docs/*                # 여러 파일 + total
find /etc -name "*.conf" 2>/dev/null | wc -l             # 개수 집계 (필기 R10-11)
```

- `tee <파일>` : 표준입력을 화면과 파일 **양쪽**으로 (T자 분기), `-a` : 이어쓰기(**a**ppend), `-i` : SIGINT 무시
- `printf '<서식>' 인자…` : `%s` 문자열, `%d` 정수, `%f` 실수, `%-Ns` 좌측정렬, `%Ns` 우측정렬, `%0Nd` 0채움, `%.Nf` 소수 자릿수. 인자가 서식보다 많으면 **서식을 반복 적용**
- `wc -l` 행(**l**ines) · `-w` 단어(**w**ords) · `-c` 바이트(**c**hars) · `-m` 멀티바이트 문자 · `-L` 최장 행 길이

**검증**
```bash
cat /tmp/apple.cnt
printf '%-10s %5d\n' dev1 2001 dev2 2002
```

```text
2
dev1        2001
dev2        2002
```

> 📝 **시험 포인트**: `tee` 는 "화면 출력과 파일 저장을 동시에" — 파이프 중간 로깅 문제로 출제. `wc -l < 파일` 은 파일명 없이 숫자만 나온다는 차이(단답 R06-5 의 `<` 역할과 연결).

### 4-13. 종합 실습 — 셸별 집계 · 시간대별 로그 · 사용률 임계 추출

> **상황**: 지금까지의 도구를 하나의 파이프라인으로 엮는다. 세 과제 모두 실기·필기 지문에 등장하는 형태다.

```bash
# ▸ ① /etc/passwd 셸별 사용자 집계 (필기 R02-11 지문 그대로)
cat /etc/passwd | cut -d: -f7 | sort | uniq -c | sort -nr
# awk 단독 버전 (프로세스 1개)
awk -F: '{cnt[$7]++} END{for(s in cnt) printf "%5d  %s\n", cnt[s], s}' /etc/passwd | sort -rn

# ▸ ② 로그 시간대별 발생 건수
grep -c . /var/log/secure                                # 전체 행 수
awk '{print $3}' /var/log/secure | cut -d: -f1 | sort | uniq -c | sort -k2n
# rsyslog 기본 포맷: "Sep  4 10:23:45 srv01 sshd[...]" → $3 = 시:분:초
# journald 기반 ISO 포맷이면: awk '{print substr($1,12,2)}'

# ▸ ③ df 출력에서 사용률 80% 초과 행만
df -hP | awk 'NR>1 && int($5) >= 80 {print $5, $6}'
df -hP | sed '1d' | tr -d '%' | awk '$5>=80 {printf "%-20s %s%%\n", $6, $5}'
# 임계 초과 시 종료 코드 1 로 (스크립트 연동용)
df -hP | awk 'NR>1 && int($5)>=80 {print; f=1} END{exit f?1:0}'; echo "exit=$?"
```

- `int($5)` : `"45%"` 문자열을 숫자 45 로 해석 (awk 는 선행 숫자만 취함)
- `df -P` : POSIX 출력 형식 — 장치명이 길어도 **한 행으로** 유지되어 필드 위치가 안정
- `sort -k2n` : 2번째 필드(시각) 숫자 기준 정렬 → 시간 순 정렬
- `END{exit f?1:0}` : awk 의 종료 코드를 셸에 전달 → `if`·`&&` 조건으로 활용

**검증**
```bash
cat /etc/passwd | cut -d: -f7 | sort | uniq -c | sort -nr
df -hP | awk 'NR>1 && int($5) >= 80 {print $5, $6}' ; echo "임계 초과 행 수: $(df -hP | awk 'NR>1 && int($5)>=80' | wc -l)"
```

```text
     19 /sbin/nologin
      4 /bin/bash
      1 /bin/sync
      1 /sbin/shutdown
      1 /sbin/halt
임계 초과 행 수: 0
```

> 📝 **시험 포인트**: `cut -d: -f7 | sort | uniq -c | sort -nr` 파이프의 각 단계 역할을 묻는 문항(필기 R02-11)이 그대로 출제. 결과는 "셸별 사용자 수를 많은 순으로".

---

## 5. 압축과 아카이브

### 5-1. gzip · gunzip · zcat 계열

> **상황**: 로그 사본을 gzip 으로 압축해 보고, 압축 파일을 **풀지 않고** 그대로 조회·검색하는 방법을 익힌다.

```bash
dnf install -y bzip2 xz zip unzip
mkdir -p /tmp/comp && cd /tmp/comp
cp /srv/devteam/proj/logs/messages sample.log
ls -l sample.log

gzip -k sample.log            # 원본 유지하며 sample.log.gz 생성
ls -l sample.log sample.log.gz
gzip -l sample.log.gz         # 압축률·원본 크기 조회
gzip -t sample.log.gz; echo "test exit=$?"   # 무결성 검사
gzip -c sample.log > copy.gz  # 표준출력으로 → 리다이렉션
gzip -9 -c sample.log > best.gz   # 최고 압축률
gzip -1 -c sample.log > fast.gz   # 최고 속도
gunzip -k copy.gz             # 압축 해제 (원본 .gz 유지)
gzip -d best.gz               # gunzip 과 동일

zcat sample.log.gz | head -3      # 해제하지 않고 내용 출력
zgrep -c 'sshd' sample.log.gz     # 압축 파일 내 검색
zless sample.log.gz               # 페이저로 조회 (q 종료)
zdiff sample.log.gz copy.gz 2>/dev/null; echo "zdiff exit=$?"
```

- `gzip` 은 **파일 1개**를 압축하고 원본을 대체 (디렉터리·다중 파일은 `tar` 로 묶은 뒤 압축)
- `-k` : 원본 유지 (**k**eep) — 없으면 원본이 사라짐
- `-d` : 압축 해제 (**d**ecompress) = `gunzip`
- `-1`~`-9` : 압축 수준 — `-1` 최속·최저압축, `-9` 최저속·최고압축, 기본 `-6`
- `-l` : 압축 정보 목록 (**l**ist) — compressed/uncompressed/ratio/name
- `-c` : 결과를 표준출력으로 (**c**at) — 원본 보존 + 리다이렉션
- `-t` : 무결성 검사 (**t**est), `-v` : 상세, `-r` : 디렉터리 재귀
- `zcat`/`zgrep`/`zless`/`zmore`/`zdiff` : gzip 파일을 **해제 없이** 다루는 래퍼 (`bzip2` 는 `bz*`, `xz` 는 `xz*` 접두)

**검증**
```bash
gzip -l sample.log.gz
zcat sample.log.gz | wc -l
wc -l < sample.log            # 두 값이 같아야 정상
```

```text
         compressed        uncompressed  ratio uncompressed_name
              ...                 ...   ..%   sample.log
1234
1234
```

> 📝 **시험 포인트**: `gzip` 은 단일 파일 전용 → `.tar.gz` 는 "묶기(tar) + 압축(gzip)" 2단계. 원본 유지 옵션 `-k`, 해제 `gunzip`/`-d`, 조회 `zcat`.

### 5-2. bzip2 · xz 와 압축률·속도 실측 비교

> **상황**: 같은 파일을 3가지 알고리즘으로 압축해 **크기와 소요 시간**을 실제로 비교한다. 필기에서 "압축률이 가장 좋은 것" 을 묻는다.

```bash
cd /tmp/comp
cat sample.log sample.log sample.log > big.log     # 비교용으로 크기 확보
ls -l big.log

echo "== gzip =="  ; time gzip  -k -c big.log > big.log.gz
echo "== bzip2 ==" ; time bzip2 -k -c big.log > big.log.bz2
echo "== xz =="    ; time xz    -k -c big.log > big.log.xz

ls -lh big.log big.log.gz big.log.bz2 big.log.xz
du -b big.log*  | sort -n

bunzip2 -k -c big.log.bz2 | wc -l      # bzip2 해제 (= bzip2 -d)
unxz     -k -c big.log.xz  | wc -l      # xz 해제 (= xz -d)
bzcat big.log.bz2 | head -2
xzcat big.log.xz  | head -2
xz -l big.log.xz                        # xz 압축 정보
```

- `bzip2` : 블록 정렬(BWT) 방식 — gzip 보다 압축률 높고 느림. `-k` 유지, `-d` 해제(= `bunzip2`), `-c` 표준출력, `-1`~`-9` 블록 크기, `-t` 검사
- `xz` : LZMA2 방식 — 셋 중 **압축률 최고·속도 최저**. `-k -d -c -1`~`-9` `-l` 동일, `-T0` 는 CPU 코어 수만큼 병렬
- `time <명령>` : 실행 시간 측정 — `real`(체감), `user`(사용자 CPU), `sys`(커널 CPU)
- 일반적 경향: **압축률** `xz` > `bzip2` > `gzip`, **압축 속도** `gzip` > `bzip2` > `xz`
- 확장자 ↔ tar 옵션: `.gz`→`z`, `.bz2`→`j`, `.xz`→`J`

**검증**
```bash
ls -l big.log.gz big.log.bz2 big.log.xz | awk '{printf "%-16s %10d\n", $9, $5}'
```

```text
big.log.bz2          ...
big.log.gz           ...
big.log.xz           ...     ← 가장 작음
```

> 📝 **시험 포인트**: 압축률 순서(xz > bzip2 > gzip)와 tar 옵션 문자 매핑(`z`/`j`/`J`)이 짝으로 출제. `tar cvJf data.tar.xz /home/data`(필기 R02-61) 형태 확인.

### 5-3. zip · unzip — 크로스 플랫폼 아카이브

> **상황**: 윈도우 담당자에게 전달할 문서 묶음을 `zip` 으로 만든다. `zip` 은 tar 와 달리 **묶기와 압축을 동시에** 한다.

```bash
cd /tmp/comp
zip -r docs.zip /srv/devteam/proj/docs          # 디렉터리 재귀 압축
zip -r -9 docs9.zip /srv/devteam/proj/docs      # 최고 압축률
zip -j flat.zip /srv/devteam/proj/docs/*.txt    # 경로 없이 파일만
zip -e secret.zip /srv/devteam/proj/docs/notes.txt   # 비밀번호 암호화 (대화식 입력)

unzip -l docs.zip                  # 내용 목록 (해제 안 함)
unzip -t docs.zip                  # 무결성 검사
unzip -p docs.zip '*/README.md'    # 특정 파일 내용을 표준출력으로
mkdir -p /tmp/unz
unzip -o docs.zip -d /tmp/unz      # 확인 없이 덮어쓰며 지정 디렉터리로 해제
unzip -q docs.zip -d /tmp/unz      # 진행 메시지 억제
unzip -l secret.zip                # 목록은 보이나 내용은 비밀번호 필요
```

- `zip -r` : 디렉터리 재귀 (**r**ecursive) — 없으면 디렉터리가 빈 항목으로만 들어감
- `zip -e` : 비밀번호 암호화 (**e**ncrypt, 프롬프트 입력). `-P <암호>` 는 명령행에 노출되므로 **사용 금지**
- `zip -j` : 경로 제거하고 파일만 저장 (**j**unk paths), `-9` 최고 압축, `-q` 조용히, `-x <패턴>` 제외
- `unzip -l` : 목록(**l**ist), `-t` 검사(**t**est), `-p` 표준출력(**p**ipe), `-o` 덮어쓰기(**o**verwrite), `-n` 덮어쓰지 않음, `-d <경로>` 대상 디렉터리(**d**estination), `-q` 조용히
- tar 와 달리 zip 은 **파일별 개별 압축** → 일부만 꺼내 쓰기 유리, 압축률은 tar+gzip 보다 낮은 편
- 유닉스 권한·소유자 보존은 tar 가 우수 → 리눅스 내부 백업은 tar 를 사용

**검증**
```bash
unzip -l docs.zip | tail -3
ls -l /tmp/unz/srv/devteam/proj/docs | head -3
```

```text
 ---------                     -------
      ...                      5 files
-rw-r--r--. 1 root root ... README.md
-rw-r--r--. 1 root root ... notes.txt
```

> 📝 **시험 포인트**: `unzip -l` 목록 조회, `-d` 대상 디렉터리. `zip -r` 없이는 디렉터리가 담기지 않는다는 점, 권한 보존은 tar 가 낫다는 비교가 출제.

### 5-4. tar — 생성 · 목록 · 추출 (기출 최빈출)

> **상황**: 실기·필기에서 가장 많이 출제되는 `tar` 3동작을 전부 수행한다. 기출 원문 형태(`tar czvf home.tar.gz /home`)를 그대로 실행해 **선행 `/` 제거 경고**를 눈으로 확인한다.

```bash
cd /tmp/comp
# ▸ 생성 (c)
tar cvf docs.tar /srv/devteam/proj/docs             # 압축 없는 아카이브
tar czvf docs.tar.gz /srv/devteam/proj/docs         # gzip
tar cjvf docs.tar.bz2 /srv/devteam/proj/docs        # bzip2
tar cJvf docs.tar.xz /srv/devteam/proj/docs         # xz
tar czvf home.tar.gz /home                          # 기출 원문 (실기 R01-13)
#  → tar: Removing leading `/' from member names

# ▸ 목록 (t)
tar tvf docs.tar        | head -5     # 권한·소유자·크기·시각 포함 목록
tar tzf docs.tar.gz     | head -5     # gzip 아카이브 목록
tar tf docs.tar.gz                    # GNU tar 는 형식 자동 판별
tar tzf docs.tar.gz | grep notes      # 특정 파일 포함 여부

# ▸ 추출 (x)
mkdir -p /restore
tar xzvf docs.tar.gz -C /restore                    # 지정 디렉터리로 해제
tar xzvf docs.tar.gz -C /restore srv/devteam/proj/docs/notes.txt   # 특정 파일만
tar xzvf docs.tar.gz -C /restore --wildcards '*/*.txt'             # 패턴으로 선택 추출
tar xzvf docs.tar.gz -C /restore --strip-components=4              # 앞 4단계 경로 제거
```

- **주 동작(택 1, 필수)**: `c` 생성(**c**reate) · `t` 목록(lis**t**) · `x` 추출(e**x**tract) · `r` 아카이브에 추가(app**e**nd) · `u` 갱신된 것만 추가(**u**pdate) · `A` 아카이브 병합
- `-f <파일>` : 아카이브 파일 지정 (**f**ile) — **옵션 묶음의 맨 뒤**에 두고 바로 파일명이 와야 함
- `-v` : 처리 파일명 표시 (**v**erbose)
- `-z` gzip · `-j` bzip2 · `-J` xz · `--zstd` zstd — 생성 시 형식 지정, 추출 시 GNU tar 는 자동 판별
- `-C <경로>` : 해당 디렉터리로 이동 후 작업 (**C**hange directory) — 생성 시엔 기준 경로, 추출 시엔 대상 경로
- `--wildcards` : 멤버 이름에 글롭 패턴 허용, `--strip-components=N` : 추출 시 경로 앞 N단계 제거
- **선행 `/` 제거**: 절대 경로로 아카이브하면 tar 가 `/home` → `home` 으로 바꿔 저장하고 경고를 출력. 목적은 **추출 시 시스템 파일 덮어쓰기 사고 방지** — 아카이브는 항상 "현재 디렉터리 기준 상대 경로"로 풀린다. 절대 경로 그대로 저장하려면 `-P`/`--absolute-names`(⚠️ 위험)

**검증**
```bash
tar tzf home.tar.gz | head -3          # 앞에 / 가 없어야 정상
ls -l /restore/srv/devteam/proj/docs/ | head -3
tar tvf docs.tar | awk '{print $1, $NF}' | head -3
```

```text
home/
home/admin1/
home/admin1/.bash_logout
-rw-r--r--. 1 root root ... README.md
drwxr-xr-x srv/devteam/proj/docs/
-rw-r--r-- srv/devteam/proj/docs/README.md
```

> 📝 **시험 포인트**: `c`=생성, `x`=추출, `t`=목록(**해제 없이 확인**, 필기 R03-41·R08-41), `z`=gzip(R01-60), `v`=진행 표시, `f`=파일 지정. `tar xzvf backup.tar.gz -C /restore`(실기 R02-3, 필기 R04-44), `tar cvJf data.tar.xz`(R02-61), `tar zxvf app-1.0.tar.gz`(소스 설치 첫 단계, R07-23·R07-29).

### 5-5. tar 선택·보존 옵션 — exclude · 권한 · 검증

> **상황**: 백업 대상에서 캐시·임시 파일을 빼고, 복원 시 소유자·권한이 그대로 살아나도록 옵션을 지정한다.

```bash
cd /tmp/comp
tar czvf docs-ex.tar.gz --exclude='*.bak' --exclude='*/logs/*' /srv/devteam/proj
printf '*.bak\n*.tmp\n*/logs/*\n' > /tmp/ex.list
tar czvf docs-ex2.tar.gz --exclude-from=/tmp/ex.list /srv/devteam/proj
tar czvf docs-tot.tar.gz --totals /srv/devteam/proj/docs      # 처리 바이트·속도 요약
tar cvf docs-verify.tar -W /srv/devteam/proj/docs             # 기록 직후 재읽기 검증 (압축과 병용 불가)

# 권한·소유자 보존 (root 로 복원할 때 의미)
tar czpvf docs-perm.tar.gz /srv/devteam/proj/docs
tar xzpvf docs-perm.tar.gz -C /restore --same-owner
tar xzvf docs-perm.tar.gz -C /restore --no-same-owner         # 일반 사용자로 복원 시
tar xzvf docs.tar.gz -C /restore --keep-old-files             # 기존 파일 유지 (덮어쓰지 않음)
tar xzvf docs.tar.gz -C /restore --overwrite                  # 명시적 덮어쓰기
tar xzvf docs.tar.gz -C /restore --touch                      # 시각을 현재로 (mtime 미복원)
tar czvf docs-sel.tar.gz --selinux --acls --xattrs /srv/devteam/proj/docs   # 확장 속성 보존
```

- `--exclude=<패턴>` : 해당 패턴 제외 (여러 번 지정 가능, **대상 경로보다 앞에** 두는 것이 안전)
- `--exclude-from=<파일>` : 제외 패턴을 파일에서 읽기
- `-p` / `--preserve-permissions` : 권한 비트 보존 (**p**ermissions) — 생성 시엔 기본, 추출 시 root 는 기본 적용
- `--same-owner` : 원래 소유자로 복원(root 기본), `--no-same-owner` : 복원자 소유로 (일반 사용자 기본)
- `--keep-old-files` : 이미 있는 파일은 건드리지 않음, `--overwrite` : 덮어쓰기, `--touch` : mtime 을 현재 시각으로
- `--totals` : 처리 바이트·소요 시간·속도 요약 출력
- `-W` / `--verify` : 기록 직후 아카이브를 다시 읽어 원본과 대조 — **압축 옵션(`-z`/`-j`/`-J`)과 함께 쓸 수 없음**
- `--selinux --acls --xattrs` : SELinux 컨텍스트·ACL·확장 속성 보존 ([[03-user-group-permission|Part 03]], [[10-security-firewall-selinux|Part 10]])

**검증**
```bash
tar tzf docs-ex.tar.gz | grep -c 'logs/'      # 0 이어야 제외 성공
tar tzf docs-ex.tar.gz | grep -c 'docs/'      # 1 이상
tar tvf docs-verify.tar | head -2
```

```text
0
5
drwxr-xr-x root/root  0 ... srv/devteam/proj/docs/
-rw-r--r-- root/root 20 ... srv/devteam/proj/docs/README.md
```

> 📝 **시험 포인트**: `--exclude` 는 대상 경로 앞에 배치. `-p`(권한 보존)와 `--same-owner`(소유자 보존) 구분, `-W` 는 압축과 병용 불가.

### 5-6. tar 증분 백업 — --listed-incremental

> **상황**: 매일 전체 백업은 비효율적이다. 스냅샷 파일로 **레벨 0(전체) → 변경 → 레벨 1(증분)** 을 만들어 실제로 변경분만 담기는지 확인한다. (전략·복구 리허설 전체는 [[12-backup-recovery-review|Part 12]])

```bash
mkdir -p /tmp/inc/src /tmp/inc/bk && cd /tmp/inc
printf 'file A v1\n' > src/a.txt
printf 'file B v1\n' > src/b.txt

# ▸ 레벨 0 — snar 파일이 없는 상태에서 최초 실행 = 전체 백업
tar czf bk/lv0.tar.gz --listed-incremental=bk/snap.snar -C /tmp/inc src
tar tzf bk/lv0.tar.gz

# ▸ 변경 발생
printf 'file B v2 (수정)\n' > src/b.txt
printf 'file C new\n'      > src/c.txt

# ▸ 레벨 1 — 같은 snar 를 사용하면 변경분만
tar czf bk/lv1.tar.gz --listed-incremental=bk/snap.snar -C /tmp/inc src
tar tzf bk/lv1.tar.gz

# ▸ 차등(differential) 로 만들려면 레벨 0 시점의 snar 사본을 사용
cp bk/snap.snar bk/snap-lv0.snar    # 레벨 0 직후에 복사해 두는 것이 원칙
```

- `--listed-incremental=<snar파일>` : 증분 백업 스냅샷 파일 지정 — 파일 목록·inode·mtime 상태를 기록
  - snar 파일이 **없으면 레벨 0(전체)**, 있으면 그 시점 이후 변경분만 담고 snar 를 갱신
  - `--listed-incremental=/dev/null` : snar 를 남기지 않는 강제 전체 백업
- `-g <파일>` : `--listed-incremental` 의 축약형
- 증분 아카이브에는 **삭제·변경 정보**도 담기므로 복원은 반드시 **레벨 0 → 레벨 1 → … 순서**로
- 증분과 차등의 차이: 증분은 "직전 백업 이후", 차등은 "마지막 전체 백업 이후" → 차등은 레벨 0 시점 snar 사본을 매번 재사용

**검증**
```bash
echo "--- lv0 ---"; tar tzf bk/lv0.tar.gz
echo "--- lv1 ---"; tar tzf bk/lv1.tar.gz
ls -l bk/lv0.tar.gz bk/lv1.tar.gz    # lv1 이 훨씬 작아야 정상
```

```text
--- lv0 ---
src/
src/a.txt
src/b.txt
--- lv1 ---
src/
src/b.txt
src/c.txt
-rw-r--r--. 1 root root  ... bk/lv0.tar.gz
-rw-r--r--. 1 root root  ... bk/lv1.tar.gz
```

> 📝 **시험 포인트**: 증분 백업 스냅샷 옵션은 `--listed-incremental=<파일>`(= `-g`) — 실기 R03-10, 필기 R05-64 에 그대로 출제. `tar` 자체에는 `dump` 같은 0~9 레벨 체계가 없다는 것도 함정(필기 R10-44).

### 5-7. cpio · rpm2cpio

> **상황**: `find` 결과를 그대로 아카이브하는 전통 도구를 확인한다. RPM 패키지 내부 추출에도 쓰인다.

```bash
mkdir -p /tmp/cpio && cd /tmp/cpio
cp -a /srv/devteam/proj/docs .
find docs -depth -print | cpio -ov > docs.cpio          # copy-out (생성)
cpio -itv < docs.cpio                                    # 목록 조회
mkdir -p out && cd out
cpio -idmv < ../docs.cpio                                # copy-in (추출)
cd /tmp/cpio
find docs -print | cpio -pdmv /tmp/cpio/passthru         # pass-through (복사)

# ▸ 절대 경로 아카이브를 안전하게 풀기
cpio -idv --no-absolute-filenames < docs.cpio

# ▸ RPM 내부 파일만 꺼내기
cd /tmp/cpio
dnf download --downloaddir=/tmp/cpio tree 2>/dev/null || dnf install -y 'dnf-command(download)'
rpm2cpio /tmp/cpio/tree-*.rpm | cpio -itv | head -5
rpm2cpio /tmp/cpio/tree-*.rpm | cpio -idmv --no-absolute-filenames
```

- `-o` : copy-**o**ut — 표준입력의 파일 목록을 읽어 아카이브를 표준출력으로
- `-i` : copy-**i**n — 표준입력의 아카이브에서 파일 추출
- `-p` : **p**ass-through — 아카이브 없이 목록의 파일을 다른 디렉터리로 복사
- `-t` : 목록 출력(lis**t**), `-v` : 상세(**v**erbose), `-d` : 필요한 디렉터리 자동 생성(**d**irectory)
- `-m` : mtime 보존 (**m**time), `-u` : 무조건 덮어쓰기 (**u**nconditional)
- `-H <형식>` : 아카이브 형식 지정 — RPM 내부는 `newc`
- `--no-absolute-filenames` : 절대 경로를 상대 경로로 바꿔 추출 (**시스템 덮어쓰기 방지**)
- `find … -depth | cpio -o` : 디렉터리 권한 문제를 피하려면 `-depth`(안쪽 먼저) 사용
- `rpm2cpio <패키지.rpm> | cpio -idmv` : RPM 을 설치하지 않고 내용만 추출 ([[02-package-management|Part 02]])

**검증**
```bash
cpio -itv < docs.cpio | head -3
ls -l /tmp/cpio/out/docs | head -3
```

```text
-rw-r--r--   1 root     root           20 ... docs/README.md
-rw-r--r--   1 root     root           63 ... docs/notes.txt
...
-rw-r--r--. 1 root root 20 ... README.md
```

> 📝 **시험 포인트**: `find | cpio -ov > a.cpio` (생성) ↔ `cpio -idv < a.cpio` (추출) 방향이 헷갈리기 쉬움. `-o` = out(만들기), `-i` = in(풀기). `rpm2cpio | cpio -idmv` 는 패키지 내부 파일만 꺼내는 표준 관용구.

### 5-8. 체크섬 — md5sum · sha1/256/512sum · cksum

> **상황**: 백업 아카이브의 무결성 기준값을 남기고, 파일을 일부러 변조해 **FAILED** 가 뜨는 것까지 확인한다.

```bash
cd /tmp/comp
md5sum    docs.tar.gz
sha1sum   docs.tar.gz
sha256sum docs.tar.gz
sha512sum docs.tar.gz
cksum     docs.tar.gz            # CRC 체크섬 + 바이트 수 (POSIX)

# ▸ 기준값 파일 생성 → 검증
sha256sum docs.tar.gz > docs.tar.gz.sha256
cat docs.tar.gz.sha256
sha256sum -c docs.tar.gz.sha256              # OK
sha256sum -c --quiet docs.tar.gz.sha256      # 실패한 것만 출력
echo "체크 exit=$?"

# ▸ 여러 파일 한꺼번에
sha256sum *.tar.gz > all.sha256
sha256sum -c all.sha256

# ▸ ⚠️ 변조 재현 — 사본에만 수행 (원본 docs.tar.gz 는 건드리지 않음)
cp docs.tar.gz tampered.tar.gz
sha256sum tampered.tar.gz > tampered.sha256  # 변조 전 기준값 확보
printf '\x00' >> tampered.tar.gz             # 1바이트 추가로 내용 변조
sha256sum -c tampered.sha256; echo "변조 후 exit=$?"
```

- `md5sum`(128bit) · `sha1sum`(160bit) · `sha256sum`(256bit) · `sha512sum`(512bit) : 해시 산출. **MD5·SHA-1 은 충돌이 발견되어 무결성 용도로 부적합** → SHA-256 이상 권장
- `cksum` : CRC32 체크섬 + 바이트 수 — 전송 오류 검출용, 보안 용도 아님
- `-c <기준값파일>` : 파일에 적힌 해시와 실제 값을 대조 (**c**heck) — 결과는 `파일명: OK` / `파일명: FAILED`
- `--quiet` : OK 항목은 출력하지 않음, `--status` : 아무 출력 없이 **종료 코드로만** 결과 전달 (스크립트용)
- `-b` 바이너리 모드 · `-t` 텍스트 모드 (리눅스에선 동일)
- 기준값 파일 형식: `<해시값>␣␣<파일명>` — 공백 2개(바이너리 모드는 ` *`)

**검증**
```bash
sha256sum -c docs.tar.gz.sha256        # 원본 — OK, 종료 코드 0
sha256sum -c tampered.sha256           # 변조본 — FAILED, 종료 코드 1
sha256sum --status -c tampered.sha256; echo "status 모드 exit=$?"
```

```text
docs.tar.gz: OK
tampered.tar.gz: FAILED
sha256sum: WARNING: 1 computed checksum did NOT match
status 모드 exit=1
```

> 📝 **시험 포인트**: `sha256sum image.iso`(실기 R04-4), `sha256sum -c backup.sha256` → `OK`(필기 R09-36), "충돌 발견된 해시 대신 사용" → SHA-256 이상(필기 R06-96). 송·수신 양쪽 해시 비교가 무결성 검증의 정답.

---

## 6. 리다이렉션과 파이프

### 6-1. 표준 스트림 3종과 기본 기호

> **상황**: 정상 출력과 오류 출력을 분리해 로그로 남긴다. 존재하는 파일과 없는 파일을 동시에 `ls` 하여 stdout·stderr 가 나뉘는 것을 실증한다.

| fd | 이름 | 기본 연결 | 대표 표기 |
| --- | --- | --- | --- |
| `0` | 표준입력 (**std**ard **in**put) | 키보드 | `<`, `<<`, `<<<`, `0<` |
| `1` | 표준출력 (**std**ard **out**put) | 터미널 | `>`, `>>`, `1>`, `1>>` |
| `2` | 표준오류 (**std**ard **err**or) | 터미널 | `2>`, `2>>` |

```bash
cd /tmp
ls /etc/passwd /nofile                    # 정상·오류가 화면에 섞여 나옴
ls /etc/passwd /nofile > out.txt          # stdout 만 파일로, 오류는 화면
ls /etc/passwd /nofile 2> err.txt         # stderr 만 파일로, 정상은 화면
ls /etc/passwd /nofile > out.txt 2> err.txt        # 각각 다른 파일로
ls /etc/passwd /nofile > all.txt 2>&1     # 둘 다 all.txt 로 (전통 표기)
ls /etc/passwd /nofile &> all2.txt        # 둘 다 (bash 축약)
ls /etc/passwd /nofile >& all3.txt        # &> 와 동일 (csh 유래 표기)
ls /etc/passwd /nofile >> all.txt 2>&1    # 이어쓰기
ls /etc/passwd /nofile &>> all.txt        # 이어쓰기 축약
ls /nofile |& grep -c cannot              # |& = 2>&1 | (stderr 도 파이프로)

wc -l < /etc/passwd                       # 표준입력을 파일로 대체
tr 'a-z' 'A-Z' < /srv/devteam/proj/docs/README.md
cat <<'EOF' > /tmp/here.txt               # heredoc — 종결자까지를 입력으로
line 1
line 2
EOF
cat <<-EOF                                # <<- 는 선행 탭 제거
	들여쓴 heredoc
	EOF
grep apple <<< "apple banana"             # here-string — 문자열 1개를 입력으로
sort /etc/passwd | head -3 | cut -d: -f1  # 파이프 — 앞 출력이 뒤 입력
```

- `>` 덮어쓰기 · `>>` 이어쓰기 · `<` 입력 대체
- `2>` / `2>>` : stderr 만 리다이렉션
- `2>&1` : "fd 2 를 **현재 fd 1 이 가리키는 곳**으로 복제" — 순서에 의존 (6-2 참조)
- `&>` `>&` : stdout + stderr 동시 (bash 확장), `&>>` : 동시 이어쓰기
- `|` : 앞 명령의 stdout 을 뒤 명령의 stdin 으로, `|&` : stderr 까지 함께 전달 (= `2>&1 |`)
- `<<종결자` heredoc(여러 행 입력), `<<-종결자` 선행 **탭** 제거, `<<<문자열` here-string
- heredoc 종결자를 `'EOF'` 로 인용하면 내부 `$변수`·`` `명령` `` 확장이 **억제**됨

**검증**
```bash
cat out.txt; echo "--- err ---"; cat err.txt
wc -l < /etc/passwd; wc -l /etc/passwd    # 앞은 숫자만, 뒤는 파일명 포함
```

```text
/etc/passwd
--- err ---
ls: cannot access '/nofile': No such file or directory
26
26 /etc/passwd
```

> 📝 **시험 포인트**: stderr 만 파일로 = `명령 2> file`(단답 R03-4·R10-6·R11-9), 이어쓰기 = `>>`(단답 R07-6), `<` 는 "표준입력을 파일 내용으로 대체"(단답 R06-5). `ls /etc/passwd /nofile > out.txt 2>&1` 해석은 필기 R02-9.

### 6-2. 순서 함정 — `> f 2>&1` vs `2>&1 > f`

> **상황**: 같은 두 기호를 순서만 바꿔 쓰면 결과가 정반대다. 실제로 실행해 차이를 눈으로 확인한다.

```bash
cd /tmp
echo "=== A: > f 2>&1 (둘 다 파일로) ==="
ls /etc/passwd /nofile > A.txt 2>&1
echo "[화면 출력 없음이 정상]"
cat A.txt

echo "=== B: 2>&1 > f (stdout 만 파일, stderr 는 화면) ==="
ls /etc/passwd /nofile 2>&1 > B.txt
echo "[위에 오류가 화면으로 나왔으면 정상]"
cat B.txt

echo "=== C: 오류만 버리고 정상만 파일로 ==="
ls /etc/passwd /nofile > C.txt 2>/dev/null
cat C.txt
```

- 셸은 리다이렉션을 **왼쪽에서 오른쪽 순서대로** 처리한다
  - `> f 2>&1` : ① fd1 → 파일 f ② fd2 → **fd1 이 지금 가리키는 f** ⇒ 둘 다 파일
  - `2>&1 > f` : ① fd2 → **fd1 이 지금 가리키는 터미널** ② fd1 → 파일 f ⇒ stderr 는 터미널에 남음
- `2>&1` 의 `&` 는 "뒤의 1 이 파일명이 아니라 **fd 번호**" 라는 표시 — `2>1` 은 `1` 이라는 **파일**을 만든다
- `&>` 는 순서 문제가 없는 축약형 → 실무에서는 `&>` 또는 `> f 2>&1` 을 사용

**검증**
```bash
wc -l A.txt B.txt          # A 는 2행(정상+오류), B 는 1행(정상만)
grep -c 'cannot access' A.txt B.txt
ls -l 1 2>/dev/null && echo "2>1 을 쓰면 이런 파일이 생김"
```

```text
2 A.txt
1 B.txt
A.txt:1
B.txt:0
```

> 📝 **시험 포인트**: `cmd > file 2>&1` 만이 "둘 다 파일" — 순서를 바꾼 `cmd 2>&1 > file` 은 오답 보기로 자주 등장. `2>&1` 의 `&` 누락(`2>1`)도 오답.

### 6-3. /dev/null 과 tee

> **상황**: 필요 없는 출력을 버리고, 필요한 출력은 화면과 파일 양쪽에 남긴다.

```bash
find / -name "*.conf" > /tmp/conf.txt 2>/dev/null      # 오류만 버리기
find / -name "*.conf" 2>/dev/null | wc -l              # 개수만 (필기 R10-11)
command_not_exist > /dev/null 2>&1; echo "exit=$?"     # 출력 전부 버리고 코드만
cat /dev/null > /var/log/lab-test.log                  # 파일 내용 비우기 (inode 유지)
: > /var/log/lab-test.log                              # 같은 목적, 더 짧은 표기

dnf list installed | tee /tmp/pkg.txt | wc -l          # 화면 대신 다음 파이프로 넘기며 저장
df -hT | tee /tmp/df.txt | grep -v tmpfs               # 저장 + 필터
echo "1차 기록" | tee /tmp/multi.log > /dev/null
echo "2차 기록" | tee -a /tmp/multi.log > /dev/null    # 이어쓰기
ls /etc | tee /tmp/e1.txt /tmp/e2.txt > /dev/null      # 여러 파일 동시
sudo -k; echo "내용" | sudo tee /root/owned.txt > /dev/null   # 권한 있는 경로에 쓰기 관용구
```

- `/dev/null` : 쓰면 버려지고 읽으면 즉시 EOF 인 **비트 버킷** 문자 장치
- `> /dev/null` stdout 버리기 · `2>/dev/null` stderr 버리기 · `> /dev/null 2>&1` 전부 버리기
- `cat /dev/null > 파일` / `: > 파일` : 파일 크기를 0 으로 (삭제가 아니므로 **inode·열린 fd 유지** → 로그 회전에 안전)
- `tee` : 표준입력을 화면과 파일 양쪽으로 분기, `-a` 이어쓰기
- `… | sudo tee 파일` : `sudo cmd > 파일` 은 리다이렉션이 **원래 셸 권한**으로 수행되어 실패 → `tee` 로 우회

**검증**
```bash
ls -li /var/log/lab-test.log     # inode 번호는 그대로
wc -c /var/log/lab-test.log      # 0
cat /tmp/multi.log
```

```text
1234567 -rw-r--r--. 1 root root 0 ... /var/log/lab-test.log
0 /var/log/lab-test.log
1차 기록
2차 기록
```

> 📝 **시험 포인트**: `2>/dev/null` 은 오류만 억제. `> /dev/null 2>&1` 이 "모든 출력 억제" 의 표준형(cron 항목에서 자주 등장, [[06-process-scheduling-diagnosis|Part 06]]).

### 6-4. 명령 치환과 리스트 연산자

> **상황**: 명령의 출력을 변수·인자로 쓰고, 성공·실패에 따라 다음 명령 실행 여부를 제어한다.

```bash
NOW=$(date +%F_%H%M%S)          # 명령 치환 (권장)
OLD=`date +%F`                  # 백틱 — 중첩·이스케이프가 불편, 비권장
echo "$NOW / $OLD"
echo "커널: $(uname -r), 계정 수: $(getent passwd | wc -l)"
echo "중첩: $(dirname $(which bash))"        # $( ) 는 중첩이 자연스러움
FILES=$(find /etc -maxdepth 1 -name "*.conf" | wc -l); echo "$FILES 개"

mkdir -p /tmp/chain && cd /tmp/chain
true;  echo "; 는 앞 결과와 무관하게 실행"
true  && echo "&& 는 앞이 성공(0)일 때만"
false && echo "이 줄은 안 나옴"
false || echo "|| 는 앞이 실패(0 아님)일 때만"
grep -q root /etc/passwd && echo "root 있음" || echo "root 없음"
mkdir -p /tmp/chain/a && cd /tmp/chain/a && pwd
tar czf ok.tar.gz /etc/hostname 2>/dev/null && sha256sum ok.tar.gz > ok.sha256 || echo "백업 실패"
```

- `$(명령)` : 명령 치환 — 표준출력을 문자열로 대체. **후행 개행은 제거**됨
- `` `명령` `` : 구형 표기. 중첩 시 `` \` `` 이스케이프가 필요해 가독성 저하 → `$( )` 권장
- `;` : 앞 결과와 무관하게 순차 실행
- `&&` : 앞 명령의 종료 코드가 **0(성공)** 일 때만 다음 실행
- `||` : 앞 명령이 **0 이 아닐 때(실패)** 만 다음 실행
- `A && B || C` : "A 성공이면 B, 아니면 C" 처럼 보이지만 **B 가 실패해도 C 가 실행**됨 → 엄밀한 분기는 `if` 사용
- 변수에 담을 때는 반드시 `"$VAR"` 로 인용 (공백·개행 분리 방지)

**검증**
```bash
echo "$NOW" | grep -qE '^[0-9]{4}-[0-9]{2}-[0-9]{2}_[0-9]{6}$' && echo "형식 OK"
false || echo "exit 코드 확인: $?"
```

```text
형식 OK
exit 코드 확인: 0
```

> 📝 **시험 포인트**: `$(…)` 와 백틱은 동일 기능. `&&`/`||` 는 **종료 코드 0 = 성공** 규칙에 기반 — `$?` 문항과 함께 출제.

### 6-5. { } 와 ( ) — 그룹과 서브셸

> **상황**: 여러 명령의 출력을 한 번에 리다이렉션한다. 중괄호와 소괄호는 겉보기에 비슷하지만 **변수 유효 범위**가 다르다.

```bash
cd /tmp
{ echo "=== 헤더 ==="; date; uptime; } > /tmp/group.txt      # 현재 셸에서 실행
( echo "=== 헤더 ==="; date; uptime; ) > /tmp/subsh.txt      # 서브셸에서 실행
cat /tmp/group.txt

# ▸ 변수 유효 범위 실습
V=원본
{ V=중괄호에서변경; }
echo "중괄호 후: $V"          # 현재 셸이므로 변경됨
V=원본
( V=소괄호에서변경 )
echo "소괄호 후: $V"          # 서브셸이므로 원본 유지

# ▸ 디렉터리 이동 범위
pwd
( cd /etc && pwd )            # 서브셸 안에서만 이동
pwd                           # 원래 위치 그대로
{ cd /etc; }; pwd             # 현재 셸이 이동해 버림
cd /tmp

# ▸ 파이프도 서브셸을 만든다 (bash 기본)
CNT=0
seq 1 3 | while read n; do CNT=$((CNT+1)); done
echo "파이프 while 후: $CNT"                 # 0 — 서브셸에서 증가했기 때문
CNT=0
while read n; do CNT=$((CNT+1)); done < <(seq 1 3)
echo "프로세스 치환 후: $CNT"                # 3 — 현재 셸에서 실행
```

- `{ 명령; 명령; }` : **현재 셸**에서 실행하는 명령 그룹 — 앞뒤 공백과 **마지막 `;`(또는 개행) 필수**
- `( 명령; 명령 )` : **서브셸**에서 실행 — 변수·`cd`·`umask` 변경이 밖으로 새지 않음
- 파이프의 각 구간, `$( )`, `&` 백그라운드도 서브셸에서 실행됨 → 루프 안에서 변수를 누적하려면 주의
- `< <(명령)` : 프로세스 치환 — 명령 출력을 파일처럼 입력에 연결해 **본 셸에서 while 실행**
- 서브셸 판별: `echo $BASH_SUBSHELL` (0 이면 현재 셸)

**검증**
```bash
V=원본; { V=changed; }; echo "{}: $V"
V=원본; ( V=changed ); echo "(): $V"
echo "subshell level: $BASH_SUBSHELL / $( echo $BASH_SUBSHELL )"
```

```text
{}: changed
(): 원본
subshell level: 0 / 1
```

> 📝 **시험 포인트**: 소괄호는 서브셸 → "명령 종료 후 원래 셸로 복귀, 변수 변경 미반영"(필기 R03-9 계열). 중괄호는 현재 셸이므로 `cd` 가 유지된다는 차이.

### 6-6. 파일 디스크립터 직접 조작과 noclobber

> **상황**: 로그 파일을 fd 3 에 열어 두고 반복 기록하고, 실수로 파일을 덮어쓰는 사고를 셸 옵션으로 막는다.

```bash
cd /tmp
# ▸ fd 3 을 읽기·쓰기로 열기
exec 3<> /tmp/fd3.log            # fd 3 을 읽기·쓰기 모드로 개방
echo "첫 줄" >&3                  # fd 3 으로 기록
echo "둘째 줄" >&3
exec 3>&-                        # fd 3 닫기
cat /tmp/fd3.log

# ▸ 읽기 전용 fd
exec 4< /etc/hostname
read -r line <&4
echo "fd4 에서 읽음: $line"
exec 4<&-

ls -l /proc/self/fd | head -5    # 현재 프로세스의 열린 fd 확인

# ▸ noclobber — > 로 기존 파일 덮어쓰기 금지
echo "원본 유지되어야 함" > /tmp/nc.txt
set -o noclobber                  # = set -C
echo "덮어쓰기 시도" > /tmp/nc.txt 2>&1; echo "exit=$?"
echo "강제 덮어쓰기" >| /tmp/nc.txt; echo "exit=$?"     # >| 는 noclobber 무시
echo "이어쓰기는 허용" >> /tmp/nc.txt
set +o noclobber                  # 해제
cat /tmp/nc.txt
```

- `exec N<> 파일` : fd N 을 읽기·쓰기로 개방, `exec N< 파일` 읽기 전용, `exec N> 파일` 쓰기 전용
- `>&N` / `<&N` : 해당 fd 로 출력/입력
- `exec N>&-` / `exec N<&-` : fd N 닫기
- `/proc/self/fd/` : 현재 프로세스가 연 fd 를 심볼릭 링크로 확인
- `set -o noclobber`(= `set -C`) : `>` 로 **기존 파일 덮어쓰기 차단** → `cannot overwrite existing file`
- `>|` : noclobber 를 무시하고 강제로 덮어쓰기, `>>` 는 noclobber 와 무관하게 허용
- `exec` 를 **명령과 함께** 쓰면(`exec ls`) 현재 셸 프로세스를 그 명령으로 **교체** (fork 없이 exec) → 명령 종료 시 셸도 종료

**검증**
```bash
cat /tmp/fd3.log
cat /tmp/nc.txt
```

```text
첫 줄
둘째 줄
강제 덮어쓰기
이어쓰기는 허용
```

> 📝 **시험 포인트**: `exec 명령` 은 현재 셸을 대체(fork 없음) → 실행 후 셸이 종료된다는 것이 필기 함정(R03-9). `noclobber` 는 `>` 만 막고 `>>` 는 허용.

---

## 7. 셸 변수와 확장

### 7-1. 지역 변수 vs 환경 변수 — 자식 프로세스 상속 실습

> **상황**: 변수가 자식 프로세스로 넘어가는지 직접 확인한다. 이론은 [[../THEORY/system-structure|THEORY 4-3]], 여기서는 상속 여부를 실측한다.

```bash
LOCALVAR="셸 변수"                       # 현재 셸에만 존재
export ENVVAR="환경 변수"                 # 자식에게 상속
echo "부모: $LOCALVAR / $ENVVAR"
bash -c 'echo "자식: [${LOCALVAR:-미상속}] [${ENVVAR:-미상속}]"'

env | grep -E '^(LOCALVAR|ENVVAR)='      # env 는 환경 변수만
set | grep -E '^(LOCALVAR|ENVVAR)='      # set 은 셸 변수 + 함수까지
printenv ENVVAR                          # 환경 변수 1개 조회
printenv LOCALVAR; echo "exit=$?"        # 셸 변수는 안 나옴 → exit 1

export LOCALVAR                          # 기존 셸 변수를 환경 변수로 승격
bash -c 'echo "승격 후 자식: $LOCALVAR"'
export -n LOCALVAR                       # 환경 변수 → 셸 변수로 강등 (값은 유지)
bash -c 'echo "강등 후 자식: [${LOCALVAR:-미상속}]"'
unset ENVVAR LOCALVAR                    # 변수 제거

# ▸ 주요 환경 변수 확인
echo "PATH=$PATH"
echo "HOME=$HOME  USER=$USER  SHELL=$SHELL  PWD=$PWD  LANG=$LANG"
echo "PS1=$PS1"; echo "PS2=$PS2"
echo "HISTSIZE=$HISTSIZE HISTFILESIZE=$HISTFILESIZE"
export PATH="$PATH:/usr/local/bin"       # PATH 추가 (영구 적용은 ~/.bashrc)
export DISPLAY=192.168.64.1:0            # X 클라이언트 출력 위치 (X 미설치 → ※ 미실행)
```

- `VAR=값` : **셸(지역) 변수** — 현재 셸에서만 유효, 자식 프로세스에 상속되지 않음. `=` 앞뒤에 **공백 금지**
- `export VAR=값` / `export VAR` : **환경 변수**로 등록 → `fork` 되는 자식 프로세스에 상속
- `export -n VAR` : 환경 변수 표시만 해제(값 유지), `unset VAR` : 변수 자체 제거
- `env` / `printenv` : 환경 변수만 조회, `set` : 셸 변수·함수까지 전부 조회
- `env VAR=값 명령` : 해당 명령에만 일시적으로 환경 변수 부여
- 자식 → 부모 방향 상속은 **불가** → 스크립트로 변수를 남기려면 `source`(8-1) 사용

**검증**
```bash
export T1=abc; T2=def
bash -c 'echo "T1=[$T1] T2=[$T2]"'
env | grep -c '^T1='; env | grep -c '^T2='
unset T1 T2
```

```text
T1=[abc] T2=[]
1
0
```

> 📝 **시험 포인트**: "자식 프로세스에 상속되도록" = `export`(단답 R05-4·R11-7, 필기 R05-11·R07-36). `env`=환경변수만 / `set`=셸 변수 포함(필기 R08-6, 단답 R06-4). `PATH`(단답 R08-6·R14-6), `DISPLAY`(단답 R02-6·R05-10·R10-9·R13-10).

### 7-2. declare · readonly · unset — 변수 속성

> **상황**: 정수 전용·읽기 전용·배열 변수를 만들어 속성이 실제로 동작하는지 확인한다.

```bash
declare -i NUM=10          # 정수 속성 — 산술 자동 평가
NUM=NUM+5; echo "NUM=$NUM" # 15 (문자열이면 "NUM+5" 가 됨)
NUM="abc"; echo "NUM=$NUM" # 정수 변환 실패 → 0

declare -r CONST="변경 불가"
CONST="시도" 2>&1 | head -1        # readonly variable 오류
readonly RO="이것도 상수"
unset RO 2>&1 | head -1            # readonly 는 unset 도 불가

declare -a ARR=(alpha beta gamma)  # 인덱스 배열
echo "${ARR[0]} / ${ARR[@]} / ${#ARR[@]}"
ARR+=(delta); echo "${ARR[@]}"
declare -A MAP                     # 연관 배열 (bash 4+)
MAP[dev1]=2001; MAP[dev2]=2002
echo "${MAP[dev1]} / keys=${!MAP[@]}"

declare -x XV="내보내는 변수"       # = export
bash -c 'echo "자식: $XV"'
declare -p NUM ARR XV | head -3    # 속성 포함 선언문 출력
declare -f 2>/dev/null | head -3   # 정의된 함수 목록
unset NUM ARR MAP XV
```

- `declare` / `typeset` : 변수 속성 지정 — `-i` 정수(**i**nteger), `-r` 읽기 전용(**r**eadonly), `-x` 내보내기(e**x**port), `-a` 인덱스 배열(**a**rray), `-A` 연관 배열(**A**ssociative), `-l`/`-u` 소문자/대문자 강제, `-p` 선언문 출력(**p**rint), `-f` 함수
- `readonly VAR=값` : 상수화 — 재할당·`unset` 모두 거부 (셸 종료까지 유지)
- `unset VAR` : 변수 삭제, `unset -f 함수명` : 함수 삭제
- 배열: `${ARR[0]}` 요소, `${ARR[@]}` 전체, `${#ARR[@]}` 개수, `${!MAP[@]}` 키 목록, `ARR+=(값)` 추가

**검증**
```bash
declare -i N=7; N=N*3; echo "$N"
declare -r C=1; C=2 2>&1 | grep -o 'readonly variable'
```

```text
21
readonly variable
```

> 📝 **시험 포인트**: `declare -i` 정수, `-r`/`readonly` 상수, `-x`/`export` 환경변수. 배열 개수는 `${#ARR[@]}`, 문자열 길이는 `${#VAR}` — 표기가 비슷해 혼동 유발.

### 7-3. 특수 변수

> **상황**: 스크립트 인자·종료 코드·PID 를 다루는 특수 변수를 한 파일에서 전부 확인한다.

```bash
cat > /tmp/special.sh <<'EOF'
#!/bin/bash
echo "\$0 (스크립트명)  = $0"
echo "\$# (인자 개수)   = $#"
echo "\$1 \$2 \$3        = $1 $2 $3"
echo "\$@ (인자 전체)   = $@"
echo "\$* (인자 전체)   = $*"
echo "\$\$ (현재 PID)   = $$"
echo "\$PPID (부모 PID) = $PPID"
sleep 5 &
echo "\$! (직전 백그라운드 PID) = $!"
grep -q root /etc/passwd
echo "\$? (직전 종료 코드) = $?"
grep -q zzzz /etc/passwd
echo "\$? (실패했을 때)   = $?"
echo "\$_ (직전 명령 마지막 인자) = $_"
echo "IFS 길이 = ${#IFS}"
echo "--- \"\$@\" 는 인자별로, \"\$*\" 는 하나로 ---"
for a in "$@"; do echo "  @: [$a]"; done
for a in "$*"; do echo "  *: [$a]"; done
wait
EOF
chmod +x /tmp/special.sh
/tmp/special.sh alpha "beta gamma" delta
```

| 변수 | 의미 |
| --- | --- |
| `$0` | 실행된 스크립트(또는 셸) 이름 |
| `$1` `$2` … `${10}` | 위치 매개변수 — **10 이상은 중괄호 필수** |
| `$#` | 전달된 인자의 **개수** ( `$0` 미포함 ) |
| `$@` | 인자 전체 — `"$@"` 는 **인자마다 별개 문자열**로 전개 |
| `$*` | 인자 전체 — `"$*"` 는 IFS 첫 문자로 이어붙인 **하나의 문자열** |
| `$?` | 직전 명령의 **종료 상태** (0=성공, 1~255=실패) |
| `$$` | **현재 셸/스크립트의 PID** |
| `$!` | 직전에 **백그라운드로 보낸** 프로세스의 PID |
| `$_` | 직전 명령의 마지막 인자 |
| `$PPID` | 부모 프로세스 PID |
| `$IFS` | 내부 필드 구분자 (기본: 공백·탭·개행) |
| `$RANDOM` `$SECONDS` `$LINENO` | 난수 / 셸 시작 후 경과 초 / 현재 행 번호 |

- `shift` : 위치 매개변수를 왼쪽으로 밀기 (`$2`→`$1`), `set -- a b c` : 위치 매개변수 직접 설정
- 인자 처리 루프에서는 **반드시 `"$@"`** 사용 — `$*` 는 공백 포함 인자를 쪼갠다

**검증**
```bash
/tmp/special.sh alpha "beta gamma" delta | grep -E '^\$#|@:|\*:'
```

```text
$# (인자 개수)   = 3
  @: [alpha]
  @: [beta gamma]
  @: [delta]
  *: [alpha beta gamma delta]
```

> 📝 **시험 포인트**: `$?` 직전 종료 상태 · `$#` 인자 개수 · `$@` 인자 목록 · `$$` **현재** PID (필기 R03-35 는 `$$` 를 "직전 백그라운드 PID" 로 적은 보기가 오답 — 그것은 `$!`). `echo $?` 가 1 이면 "정상 실행됐으나 못 찾음"(필기 R08-5).

### 7-4. 매개변수 확장 — 기본값·길이·부분 문자열·치환

> **상황**: 스크립트에서 인자 누락 처리, 경로 분해, 확장자 교체를 외부 명령 없이 셸 기능만으로 처리한다.

```bash
unset UNDEF; EMPTY=""; NAME="backup"; FILE="/srv/raid/backup/data-2026-09-04.tar.gz"

echo "${UNDEF:-기본값}"      # 미설정/빈 값이면 기본값 사용 (변수는 그대로)
echo "UNDEF=[${UNDEF-}]"     # 여전히 미설정
echo "${UNDEF:=할당됨}"      # 미설정이면 기본값을 "대입"까지
echo "UNDEF=[$UNDEF]"        # 할당됨
echo "${EMPTY:+값있음}"      # 값이 있을 때만 대체 문자열 사용 → 빈 출력
echo "${NAME:+값있음}"       # 값이 있으므로 "값있음"
: "${MUST:?필수 변수가 없습니다}" 2>&1 | tail -1    # 없으면 오류 메시지 + 종료

echo "${#FILE}"              # 문자열 길이
echo "${FILE##*/}"           # 가장 긴 앞쪽 일치 제거 → basename
echo "${FILE#*/}"            # 가장 짧은 앞쪽 일치 제거
echo "${FILE%/*}"            # 가장 짧은 뒤쪽 일치 제거 → dirname
echo "${FILE%%-*}"           # 가장 긴 뒤쪽 일치 제거
BASE="${FILE##*/}"
echo "${BASE%.tar.gz}"       # 확장자 제거
echo "${BASE%.*}"            # 마지막 확장자만 제거
echo "${FILE/backup/BK}"     # 첫 일치 치환
echo "${FILE//backup/BK}"    # 모든 일치 치환
echo "${FILE/#\/srv/\/mnt}"  # 앞부분(prefix)이 일치할 때만 치환
echo "${FILE/%gz/GZ}"        # 뒷부분(suffix)이 일치할 때만 치환
echo "${FILE:1:10}"          # 1번째 문자부터 10자 (0부터 시작)
echo "${FILE: -6}"           # 뒤에서 6자 (콜론 뒤 공백 필요)
echo "${NAME^^} / ${NAME^} / ${FILE,,}"   # 대문자화 / 첫 글자만 / 소문자화
TARGETS=(/data /srv/share /etc)
echo "${TARGETS[@]#/}"       # 배열 전체에 확장 적용 → data srv/share etc
```

| 표기 | 의미 |
| --- | --- |
| `${var:-기본값}` | var 이 비었으면 **기본값을 사용** (대입 안 함) |
| `${var:=기본값}` | var 이 비었으면 **기본값을 대입하고 사용** |
| `${var:?메시지}` | var 이 비었으면 메시지 출력 후 **스크립트 종료** |
| `${var:+대체값}` | var 에 값이 **있을 때만** 대체값 사용 |
| `${#var}` | 문자열 길이 |
| `${var#패턴}` / `${var##패턴}` | **앞**에서 최소/최대 일치 제거 (`##*/` = basename) |
| `${var%패턴}` / `${var%%패턴}` | **뒤**에서 최소/최대 일치 제거 (`%/*` = dirname, `%.*` = 확장자 제거) |
| `${var/찾을것/바꿀것}` | 첫 일치 치환 |
| `${var//찾을것/바꿀것}` | 전체 치환 |
| `${var:시작:길이}` | 부분 문자열 (0부터) |
| `${var^^}` `${var,,}` `${var^}` | 전체 대문자 / 전체 소문자 / 첫 글자 대문자 |

- `:` 를 뺀 `${var-기본값}` 은 "**미설정**일 때만" 동작 (빈 문자열은 값으로 인정)
- `#` 은 키보드에서 `$` 왼쪽 → **앞쪽** 제거, `%` 는 오른쪽 → **뒤쪽** 제거 로 외우면 쉬움
- `basename`/`dirname` 외부 명령을 대체 → 스크립트 성능·이식성 향상

**검증**
```bash
F="/srv/raid/backup/data-2026-09-04.tar.gz"
echo "base=${F##*/} dir=${F%/*} noext=$(b=${F##*/}; echo ${b%.tar.gz})"
echo "len=${#F}"
```

```text
base=data-2026-09-04.tar.gz dir=/srv/raid/backup noext=data-2026-09-04
len=39
```

> 📝 **시험 포인트**: `${var:-기본값}`(사용만) vs `${var:=기본값}`(대입까지) 구분. `${var##*/}` = basename, `${var%/*}` = dirname 은 스크립트 작성 문항에서 감점 포인트.

### 7-5. 산술 연산 — $(( )) · let · expr

> **상황**: 카운터·용량 계산 등 스크립트의 수치 처리를 세 가지 방법으로 비교한다.

```bash
A=17; B=5
echo "$(( A + B )) $(( A - B )) $(( A * B )) $(( A / B )) $(( A % B )) $(( A ** 2 ))"
echo "$(( A > B )) $(( A == B ))"        # 참=1, 거짓=0
i=0; (( i++ )); (( i += 10 )); echo "i=$i"
(( A > B )) && echo "(( )) 는 조건식으로도 사용"   # 값이 0 이면 종료 코드 1

let C=A+B; let C++; echo "C=$C"
let "D = A * B"; echo "D=$D"

expr $A + $B                 # 외부 명령 — 연산자 앞뒤 공백 필수
expr $A \* $B                # * 는 이스케이프 필요
expr length "abcdef"
expr index "abcdef" c
expr substr "abcdef" 2 3
echo "5.5 + 2.3" | bc 2>/dev/null || echo "실수 연산은 bc 필요 (dnf install -y bc)"
awk 'BEGIN{printf "%.2f\n", 5.5/2.3}'    # bc 없이 실수 계산

# ▸ 진법 변환
echo $(( 16#FF ))            # 16진수 → 10진수
echo $(( 8#755 ))            # 8진수 → 10진수
printf '%o\n' 493            # 10진수 → 8진수
printf '%x\n' 255            # 10진수 → 16진수
```

- `$(( 식 ))` : 산술 확장 — **셸 내장, 가장 빠르고 권장**. 내부에서는 `$` 없이 변수명 사용 가능
- `(( 식 ))` : 산술 평가 명령 — 결과가 0 이면 종료 코드 1(거짓), 0 이 아니면 0(참)
- `let 식` : 산술 평가 내장 명령 — 공백이 있으면 인용 필요
- `expr 식` : **외부 명령** — 연산자 앞뒤 공백 필수, `*` 는 `\*` 로 이스케이프. 문자열 함수(`length`·`index`·`substr`)도 제공
- 연산자: `+ - * / % **`, 비교 `== != < <= > >=`, 논리 `&& || !`, 증감 `++ --`, 복합대입 `+= -= *= /=`
- **bash 산술은 정수만** → 실수는 `bc` 또는 `awk`
- 진법: `기수#숫자` 표기 (`16#FF`, `2#1010`)

**검증**
```bash
echo $(( 100 / 7 )) $(( 100 % 7 ))
awk 'BEGIN{printf "%.4f\n", 100/7}'
echo $(( 16#FF )) $(( 8#644 ))
```

```text
14 2
14.2857
255 420
```

> 📝 **시험 포인트**: bash 의 `$(( ))` 는 **정수 연산만** — 소수점이 잘린다는 점이 함정. `expr` 은 외부 명령이라 공백과 이스케이프 규칙이 다름.

### 7-6. 인용 3종 — 작은따옴표·큰따옴표·백슬래시

> **상황**: 변수·명령 치환이 언제 일어나는지 한 화면에서 비교한다. 스크립트 버그의 절반이 여기서 나온다.

```bash
NAME="Rocky"; FILES="*.txt"
echo '작은따옴표: $NAME `date` *  → 전부 문자 그대로'
echo "큰따옴표: $NAME $(date +%F) *  → 변수·명령 치환은 됨, 글로빙은 안 됨"
echo 백슬래시: \$NAME \`date\` \*  → 다음 한 글자만 무효화
echo "$FILES"        # *.txt 문자열 그대로
echo $FILES          # 인용 없음 → 글로빙 전개 (파일 있으면 파일명들)
cd /srv/devteam/proj/docs; echo $FILES; echo "$FILES"

# ▸ 공백 포함 파일명에서의 차이
touch "my report.txt"
ls -l my report.txt 2>&1 | head -2      # 인용 없음 → 두 파일로 해석 (오류)
ls -l "my report.txt"                   # 정상
F="my report.txt"
ls -l $F  2>&1 | head -1                # 변수도 인용 안 하면 분리
ls -l "$F"
rm -f "my report.txt"

# ▸ 큰따옴표 안에서 살아 있는 특수문자
echo "달러=\$ 백틱=\` 역슬래시=\\ 큰따옴표=\" 느낌표는 대화형에서 특별"
printf '%s\n' "$HOME" '$HOME'
```

| 인용 | 변수 `$` | 명령 치환 `` ` `` `$( )` | 글로빙 `*` | 용도 |
| --- | --- | --- | --- | --- |
| `'작은따옴표'` | ✗ | ✗ | ✗ | 정규식·패턴·리터럴 전달 |
| `"큰따옴표"` | ✓ | ✓ | ✗ | 변수 값 전달 (공백 보호) |
| `\문자` | 해당 1문자만 무효화 | — | — | 개별 메타문자 이스케이프 |

- 큰따옴표 안에서도 특별한 문자: `$` `` ` `` `\` `"` (그리고 대화형에서의 `!`)
- **원칙**: 변수는 항상 `"$VAR"`, 배열 전개는 `"${ARR[@]}"`, 명령 치환은 `"$(cmd)"`
- `grep '패턴'` — 정규식은 작은따옴표로 감싸 셸이 먼저 건드리지 못하게 함

**검증**
```bash
X="a b"; printf '[%s]\n' $X;  echo "---";  printf '[%s]\n' "$X"
```

```text
[a]
[b]
---
[a b]
```

> 📝 **시험 포인트**: 작은따옴표 = 모든 확장 차단, 큰따옴표 = 변수·명령 치환만 허용. `echo '$HOME'` 은 `$HOME` 문자열, `echo "$HOME"` 은 경로가 출력된다는 대비가 출제.

### 7-7. 글로빙과 중괄호 확장

> **상황**: 파일명 패턴으로 여러 파일을 한 번에 지정한다. 정규식과 다른 문법임을 다시 확인한다.

```bash
cd /srv/devteam/proj/src
ls *.c                    # 임의 길이 문자열
ls mod?.c                 # 정확히 1문자
ls mod[123].c             # 나열 중 1문자
ls mod[1-3].c             # 범위 중 1문자
ls mod[!12].c             # 나열 제외 (= [^12])
ls [[:digit:]]*  2>/dev/null; echo "exit=$?"
echo {1..5}               # 수열 확장
echo {01..05}             # 0 채움
echo {1..10..3}           # 간격 지정
echo {a..e}               # 문자 범위
echo file{1,2,3}.txt      # 목록 확장
mkdir -p /tmp/glob/{src,bin,etc}/{a,b}   # 조합 확장 → 6개 디렉터리
find /tmp/glob -type d | sort

# ▸ shopt 확장 옵션
shopt -s nullglob
ls *.nonexistent; echo "nullglob: 일치 없으면 빈 목록 → exit=$?"
shopt -u nullglob
ls *.nonexistent 2>&1 | head -1     # 기본: 패턴 문자열이 그대로 전달됨
shopt -s extglob
ls !(mod1).c                        # mod1.c 를 제외한 .c
ls +(mod)[0-9].c                    # 1회 이상 반복
shopt -u extglob
shopt -s globstar
ls /srv/devteam/**/*.txt 2>/dev/null | head -3   # ** 은 하위 전체 재귀
shopt -u globstar
shopt -s dotglob; ls /root/* 2>/dev/null | head -2; shopt -u dotglob
shopt | grep -E 'extglob|nullglob|globstar|dotglob'
```

- 글로빙(파일명 확장): `*` 임의 길이(0자 포함) · `?` 정확히 1문자 · `[abc]` 나열 중 1문자 · `[a-z]` 범위 · `[!abc]`/`[^abc]` 제외
- 중괄호 확장: `{1..5}` 수열 · `{01..05}` 0채움 · `{1..10..3}` 간격 · `{a,b,c}` 목록 — **파일 존재 여부와 무관하게** 문자열을 만든다
- 처리 순서: 중괄호 확장 → 물결(`~`) 확장 → 변수·명령 치환 → 단어 분리 → **글로빙** → 인용 제거
- `shopt -s <옵션>` 켜기 / `-u` 끄기 / `shopt` 로 목록 조회
  - `nullglob` : 일치 파일이 없으면 **빈 목록**으로 (기본은 패턴 문자열 그대로 전달)
  - `failglob` : 일치가 없으면 오류
  - `extglob` : 확장 패턴 `?(…)` 0~1회 · `*(…)` 0회 이상 · `+(…)` 1회 이상 · `@(…)` 정확히 1개 · `!(…)` 제외
  - `globstar` : `**` 가 하위 디렉터리 전체를 재귀 매치
  - `dotglob` : `*` 가 숨김 파일(`.` 시작)도 포함
- 글로빙 `*` 는 "임의 문자열", 정규식 `*` 는 "앞 문자 반복" — **의미가 완전히 다름**

**검증**
```bash
echo {1..5}; echo file{a,b}.txt
ls /tmp/glob | tr '\n' ' '; echo
ls /tmp/glob/src | tr '\n' ' '; echo
```

```text
1 2 3 4 5
filea.txt fileb.txt
bin etc src 
a b 
```

> 📝 **시험 포인트**: `*` `?` `[]` 는 파일명 확장(와일드카드)으로 필기 메타문자 문항에 등장. `[!a]` 와 정규식 `[^a]` 의 표기 차이도 확인.

### 7-8. set 옵션 · read · bash -x

> **상황**: 스크립트 디버깅과 안전장치를 미리 익힌다. 8절의 스크립트 작성에서 그대로 사용한다.

```bash
# ▸ set 옵션 (스크립트 안전장치)
bash -c 'set -e; false; echo "이 줄은 안 나옴"'; echo "set -e exit=$?"
bash -c 'set -u; echo "[$UNDEFINED_VAR]"' 2>&1 | tail -1
bash -c 'set -x; A=1; echo "$A"' 2>&1 | head -4
bash -c 'false | true; echo "pipefail 없음: $?"'
bash -c 'set -o pipefail; false | true; echo "pipefail 있음: $?"'
bash -c 'set -euo pipefail; echo "3종 세트 적용"'
set -o | grep -E 'errexit|nounset|xtrace|pipefail|noclobber'

# ▸ read — 사용자 입력
read -p "이름을 입력: " UNAME; echo "입력값=[$UNAME]"
read -s -p "비밀번호(화면 미표시): " PW; echo; echo "길이=${#PW}"
read -a WORDS -p "공백으로 구분해 여러 개: "; echo "개수=${#WORDS[@]} 첫째=${WORDS[0]}"
read -r -p "역슬래시 그대로 읽기: " RAW; echo "[$RAW]"
read -t 5 -p "5초 내 입력(초과 시 넘어감): " TO; echo "exit=$?"
read -n 1 -p "1글자만 받고 즉시 진행: " ONE; echo; echo "[$ONE]"
printf 'a:b:c\n' | IFS=: read -r f1 f2 f3; echo "$f1 / $f2 / $f3"
while IFS=: read -r user _ uid _ _ home shell; do
    [ "$uid" -ge 1000 ] && echo "$user $uid $shell"
done < /etc/passwd

# ▸ 디버깅
bash -n /tmp/special.sh; echo "문법 검사 exit=$?"
bash -x /tmp/special.sh one two 2>&1 | head -8
```

- `set -e` (`errexit`) : 명령이 실패(0 아님)하면 **즉시 종료** — 단, `if`/`&&`/`||` 조건부는 예외
- `set -u` (`nounset`) : **미정의 변수 참조 시 오류** — 오타로 인한 빈 값 사고 방지
- `set -x` (`xtrace`) : 실행 직전의 명령을 `+` 접두로 출력 (`set +x` 로 해제)
- `set -o pipefail` : 파이프의 **어느 한 구간이라도 실패하면** 전체를 실패로 — 기본은 마지막 명령의 코드만 반영
- `set -euo pipefail` : 운영 스크립트의 관용적 안전 3종
- `set -o` / `set +o` : 옵션 목록 조회, `set -o <이름>` = `set -<문자>`
- `read` : `-p <문구>` 프롬프트(**p**rompt), `-s` 화면 미표시(**s**ilent, 비밀번호), `-a <배열>` 배열로(**a**rray), `-r` 역슬래시 이스케이프 해석 안 함(**r**aw, **권장**), `-n N` N글자만, `-t N` 타임아웃(초), `-d <문자>` 종료 문자
- `IFS=: read -r a b c` : 구분자를 지정해 한 줄을 필드로 분해 — `while IFS= read -r line` 이 행 단위 처리의 표준형
- `bash -n <스크립트>` : 실행 없이 **문법만 검사**, `bash -x` : 추적 실행

**검증**
```bash
bash -c 'set -o pipefail; grep -q zzz /etc/passwd | cat; echo "pipefail exit=$?"'
bash -n /tmp/special.sh && echo "문법 OK"
```

```text
pipefail exit=1
문법 OK
```

> 📝 **시험 포인트**: `set -e` 즉시 종료, `set -u` 미정의 변수 오류, `set -x` 추적. `read -p`(프롬프트), `-s`(비밀번호 숨김) 는 스크립트 작성 문항 단골.

---

## 8. 셸 스크립트

### 8-1. shebang · 실행 권한 · 실행 방식 3종

> **상황**: 같은 스크립트를 세 가지 방법으로 실행하고, **변수가 현재 셸에 남는지**로 차이를 검증한다.

```bash
mkdir -p /tmp/sh && cd /tmp/sh
cat > run.sh <<'EOF'
#!/bin/bash
echo "\$0     = $0"
echo "PID    = $$   (부모 PID = $PPID)"
echo "SHLVL  = $SHLVL"
MYVAR="스크립트에서 설정"
EOF

ls -l run.sh                     # 아직 실행 권한 없음
./run.sh 2>&1 | head -1          # Permission denied
bash run.sh                      # 실행 권한 없어도 인터프리터에 넘기면 실행됨
chmod +x run.sh                  # = chmod u+x / chmod 755
ls -l run.sh

echo "=== ① ./run.sh (새 프로세스) ==="
unset MYVAR; ./run.sh;      echo "실행 후 MYVAR=[${MYVAR:-없음}]"
echo "=== ② bash run.sh (새 프로세스) ==="
unset MYVAR; bash run.sh;   echo "실행 후 MYVAR=[${MYVAR:-없음}]"
echo "=== ③ source run.sh (현재 셸) ==="
unset MYVAR; source run.sh; echo "실행 후 MYVAR=[${MYVAR:-없음}]"
echo "=== ③' . run.sh (source 와 동일) ==="
unset MYVAR; . run.sh;      echo "실행 후 MYVAR=[${MYVAR:-없음}]"
```

| 실행 방식 | 프로세스 | 실행 권한 | shebang | 변수 잔존 |
| --- | --- | --- | --- | --- |
| `./run.sh` | **자식 셸** 새로 생성 | **필요** | shebang 이 인터프리터 결정 | ✗ |
| `bash run.sh` | **자식 셸** 새로 생성 | 불필요 | 무시(bash 로 실행) | ✗ |
| `source run.sh` / `. run.sh` | **현재 셸**에서 실행 | 불필요 | 무시 | **✓ 남음** |

- `#!/bin/bash` (**she**-**bang**) : 파일 첫 줄에 인터프리터 절대 경로 지정 — `#!/bin/sh`, `#!/usr/bin/env python3` 등
- `chmod +x <파일>` : 실행 권한 부여 — `./` 로 실행하려면 필수 ([[03-user-group-permission|Part 03]])
- `./` 를 붙이는 이유: 현재 디렉터리(`.`)는 보안상 `PATH` 에 없기 때문
- `source`/`.` : 파일 내용을 **현재 셸이 직접 읽어 실행** → `~/.bashrc` 재적용, 환경 설정 스크립트에 사용
- `SHLVL` : 셸 중첩 깊이 — `source` 는 증가하지 않고 `./`·`bash` 는 증가

**검증**
```bash
unset MYVAR; ./run.sh   > /dev/null; echo "① [${MYVAR:-없음}]"
unset MYVAR; source run.sh > /dev/null; echo "③ [${MYVAR:-없음}]"
```

```text
① [없음]
③ [스크립트에서 설정]
```

> 📝 **시험 포인트**: shebang 표기 `#!/bin/bash`(실기 R01-9). `source`/`.` 는 현재 셸에서 실행되어 변수·함수가 남고, `./`·`bash` 는 자식 셸이라 남지 않음 — 최빈출 비교.

### 8-2. test · [ ] · [[ ]] — 조건 판정

> **상황**: 파일 존재·권한·문자열·숫자 비교를 각각 확인한다. `[` 는 명령이므로 **공백이 없으면 동작하지 않는다**.

```bash
cd /tmp/sh; touch empty.txt; echo "내용" > data.txt; mkdir -p dir1; ln -sf data.txt link1

# ▸ 파일 검사
test -f data.txt && echo "-f: 일반 파일"
[ -d dir1 ]      && echo "-d: 디렉터리"
[ -e link1 ]     && echo "-e: 존재 (링크는 대상 기준)"
[ -L link1 ]     && echo "-L: 심볼릭 링크 자체"
[ -s data.txt ]  && echo "-s: 크기 0 초과"
[ -s empty.txt ] || echo "-s: empty.txt 는 크기 0"
[ -r data.txt ] && [ -w data.txt ] && echo "-r -w: 읽기·쓰기 가능"
[ -x /bin/ls ]   && echo "-x: 실행 가능"
[ data.txt -nt empty.txt ] && echo "-nt: 더 최신"

# ▸ 문자열 검사
S="hello"; E=""
[ -z "$E" ] && echo "-z: 빈 문자열"
[ -n "$S" ] && echo "-n: 비어있지 않음"
[ "$S" = "hello" ]  && echo "= : 문자열 같음"
[ "$S" != "world" ] && echo "!= : 다름"
[[ "$S" == hel* ]]  && echo "[[ ]] 안의 == 는 패턴 매칭"
[[ "$S" =~ ^h.*o$ ]] && echo "=~ : 정규식 일치 (BASH_REMATCH=${BASH_REMATCH[0]})"

# ▸ 숫자 비교
N=10
[ "$N" -eq 10 ] && echo "-eq: 같음"
[ "$N" -ne 5 ]  && echo "-ne: 다름"
[ "$N" -gt 5 ]  && echo "-gt: 초과"
[ "$N" -ge 10 ] && echo "-ge: 이상"
[ "$N" -lt 20 ] && echo "-lt: 미만"
[ "$N" -le 10 ] && echo "-le: 이하"
(( N > 5 )) && echo "(( )) 안에서는 > < 를 그대로"

# ▸ 결합과 함정
[ -f data.txt -a -d dir1 ] && echo "-a: AND (구식)"
[ -f data.txt -o -f none ] && echo "-o: OR (구식)"
[ -f data.txt ] && [ -d dir1 ] && echo "&& 로 결합 (권장)"
[[ -f data.txt && -d dir1 ]] && echo "[[ ]] 안에서는 && || 사용"
[ -f data.txt ]; echo "종료 코드=$?"
[$N -eq 10] 2>&1 | head -1        # 공백 없음 → command not found
UNSET_VAR=
[ $UNSET_VAR = "x" ] 2>&1 | head -1   # 인용 누락 → 인자 개수 오류
[ "$UNSET_VAR" = "x" ] || echo "인용하면 안전"
```

| 구분 | 연산자 | 의미 |
| --- | --- | --- |
| 파일 | `-e` `-f` `-d` `-L`(`-h`) `-s` | 존재 / 일반 파일 / 디렉터리 / 심볼릭 링크 / 크기 0 초과 |
| 파일 | `-r` `-w` `-x` | 읽기 / 쓰기 / 실행 권한 보유 |
| 파일 | `-u` `-g` `-k` | SetUID / SetGID / Sticky 비트 |
| 파일 | `-nt` `-ot` `-ef` | 더 최신 / 더 오래됨 / 같은 inode |
| 문자열 | `-z` `-n` `=`(`==`) `!=` | 빈 문자열 / 비어있지 않음 / 같음 / 다름 |
| 숫자 | `-eq` `-ne` `-gt` `-ge` `-lt` `-le` | = ≠ > ≥ < ≤ |
| 결합 | `-a` `-o` `!` / `&&` `\|\|` | AND / OR / NOT |

- `test 식` = `[ 식 ]` — `[` 는 **명령**이므로 대괄호 안쪽에 **공백 필수**, 닫는 `]` 도 인자
- `[[ 식 ]]` : bash 확장 키워드 — 단어 분리·글로빙이 일어나지 않아 인용 누락에 안전, `&&`/`||`/`<`/`>`/`==`(패턴)/`=~`(정규식) 사용 가능
- `(( 식 ))` : 산술 조건 — 숫자 비교에는 `>` `<` 를 그대로 사용
- **문자열은 `=`·`!=`, 숫자는 `-eq`·`-ne`** — 뒤바꿔 쓰는 것이 최다 오답
- `=~` 의 정규식은 **인용하지 않아야** 정규식으로 해석됨, 결과는 `${BASH_REMATCH[@]}` 에 저장

**검증**
```bash
[ -f /etc/passwd ]; echo "파일 존재 exit=$?"
[ -f /nofile ];     echo "파일 없음 exit=$?"
[ "10" -gt "9" ] && echo "숫자 비교 OK"
[ "10" \> "9" ] || echo "문자열 비교로는 10 < 9"
```

```text
파일 존재 exit=0
파일 없음 exit=1
숫자 비교 OK
문자열 비교로는 10 < 9
```

> 📝 **시험 포인트**: `-f`(일반 파일) `-d`(디렉터리) `-e`(존재) `-z`(빈 문자열) `-n`(비어있지 않음) 과 숫자 비교 `-eq/-ne/-gt/-lt/-ge/-le` 는 스크립트 문항의 핵심. `[ ]` 안의 공백 누락은 즉시 오류.

### 8-3. 조건문 — if / elif / else / fi, case / esac

> **상황**: 백업 대상 존재 여부에 따라 분기하고, 사용자 입력을 `case` 로 처리하는 스크립트를 작성한다.

```bash
cd /tmp/sh
cat > cond.sh <<'EOF'
#!/bin/bash
TARGET="${1:-/etc}"

if [ ! -e "$TARGET" ]; then
    echo "[ERROR] $TARGET 없음"; exit 1
elif [ -d "$TARGET" ]; then
    echo "[INFO] $TARGET 은 디렉터리 (항목 $(ls -A "$TARGET" | wc -l) 개)"
elif [ -f "$TARGET" ]; then
    echo "[INFO] $TARGET 은 일반 파일 ($(stat -c %s "$TARGET") 바이트)"
else
    echo "[INFO] $TARGET 은 그 밖의 유형: $(stat -c %F "$TARGET")"
fi

# 종료 코드로 분기
if grep -q '^root:' /etc/passwd; then echo "[OK] root 계정 존재"; fi

# 한 줄 if
[ -w /tmp ] && echo "[OK] /tmp 쓰기 가능"

read -rp "동작 선택 [start|stop|restart|status|q]: " ACT
case "$ACT" in
    start)          echo "→ 시작" ;;
    stop)           echo "→ 중지" ;;
    restart|reload) echo "→ 재시작 (여러 패턴은 | 로)" ;;
    stat*)          echo "→ 상태 (글롭 패턴 사용 가능)" ;;
    [Qq]|quit)      echo "→ 종료"; exit 0 ;;
    "")             echo "→ 입력 없음"; exit 2 ;;
    *)              echo "→ 알 수 없는 동작: $ACT"; exit 2 ;;
esac
EOF
chmod +x cond.sh
./cond.sh /etc  <<< "restart"
./cond.sh /etc/hostname <<< "status"
./cond.sh /nofile; echo "exit=$?"
```

- `if 조건; then … elif 조건; then … else … fi` — **`fi` 로 닫음**(if 역순). `then` 은 같은 줄이면 앞에 `;` 필요
- 조건 자리에는 **명령이 온다** → `[ ]`·`[[ ]]`·`(( ))` 뿐 아니라 `grep -q`·`systemctl is-active` 등 어떤 명령이든 가능 (종료 코드 0 = 참)
- `case 값 in 패턴) 명령 ;; … *) 기본 ;; esac` — **`esac` 로 닫음**(case 역순)
  - 패턴에 글로빙(`stat*`)·문자 클래스(`[Qq]`)·다중 패턴(`a|b`) 사용 가능
  - `;;` 종료 / `;&` 다음 절로 통과(fall-through) / `;;&` 다음 패턴도 계속 검사
  - `*)` 는 기본 절 — **마지막에** 두어야 함

**검증**
```bash
./cond.sh /etc <<< "q"; echo "exit=$?"
./cond.sh /nofile <<< ""; echo "exit=$?"
```

```text
[INFO] /etc 은 디렉터리 (항목 ... 개)
[OK] root 계정 존재
[OK] /tmp 쓰기 가능
→ 종료
exit=0
[ERROR] /nofile 없음
exit=1
```

> 📝 **시험 포인트**: `if … fi`, `case … esac` 의 닫는 키워드가 역순 철자. `case` 의 각 절은 `;;` 로 끝나야 하며 `*)` 가 기본값.

### 8-4. 반복문 — for · while · until · break · continue

> **상황**: 파일 목록 순회, 숫자 반복, 파일 행 단위 처리, 조건 반복을 각각 작성한다.

```bash
cd /tmp/sh
cat > loop.sh <<'EOF'
#!/bin/bash
echo "--- ① for in 목록 ---"
for S in start stop restart; do echo "  액션: $S"; done

echo "--- ② for in 글로빙 (파일 순회) ---"
for F in /srv/devteam/proj/src/*.c; do
    [ -e "$F" ] || continue
    echo "  $(basename "$F") : $(wc -l < "$F") 행"
done

echo "--- ③ for in 명령 치환 ---"
for U in $(awk -F: '$3>=1000 && $3<65534 {print $1}' /etc/passwd); do
    echo "  사용자: $U (uid $(id -u "$U"))"
done

echo "--- ④ C 스타일 for ---"
for ((i=1; i<=5; i++)); do
    (( i == 3 )) && continue          # 3 건너뛰기
    (( i == 5 )) && break             # 5 에서 중단
    printf "  i=%d\n" "$i"
done

echo "--- ⑤ while read (행 단위 처리) ---"
while IFS=: read -r user _ uid _ _ home shell; do
    [ "$uid" -ge 1000 ] && [ "$uid" -lt 65534 ] && echo "  $user -> $shell"
done < /etc/passwd

echo "--- ⑥ while 조건 반복 ---"
n=0
while [ "$n" -lt 3 ]; do echo "  n=$n"; n=$((n+1)); done

echo "--- ⑦ until (조건이 참이 될 때까지) ---"
m=0
until [ "$m" -ge 3 ]; do echo "  m=$m"; m=$((m+1)); done

echo "--- ⑧ 무한 루프 + break ---"
c=0
while true; do
    c=$((c+1))
    [ "$c" -ge 3 ] && { echo "  $c 회에서 break"; break; }
done

echo "--- ⑨ seq / 범위 확장 ---"
for i in {1..3}; do echo -n "$i "; done; echo
for i in $(seq 1 2 7); do echo -n "$i "; done; echo
EOF
chmod +x loop.sh && ./loop.sh
```

- `for 변수 in 목록; do … done` : 목록의 각 요소를 변수에 대입하며 반복 — 목록은 글로빙·명령 치환·중괄호 확장 모두 가능
- `for ((초기; 조건; 증감)); do … done` : C 스타일 — 산술 문맥이라 `$` 없이 변수 사용
- `while 조건; do … done` : 조건이 **참인 동안** 반복
- `until 조건; do … done` : 조건이 **거짓인 동안** 반복 (참이 되면 종료) — while 의 반대
- `while IFS= read -r line; do … done < 파일` : **행 단위 처리의 표준형** (`for` + `cat` 은 공백에서 쪼개져 부적합)
- `break [n]` : 루프 탈출(n중 루프 지정 가능), `continue [n]` : 다음 반복으로
- 파이프로 while 에 입력하면 서브셸이 되어 **루프 안 변수가 밖으로 안 나감**(6-5) → `< 파일` 또는 `< <(명령)` 사용

**검증**
```bash
./loop.sh | grep -A3 '④'
./loop.sh | grep -c '^  '
```

```text
--- ④ C 스타일 for ---
  i=1
  i=2
  i=4
```

> 📝 **시험 포인트**: `for … in … do … done`, `while … do … done`, `until` 의 조건 방향 반대. 파일 행 단위 처리는 `while read line` 이 정답 형태.

### 8-5. 함수 · 종료 코드

> **상황**: 반복되는 로그 기록·검증 로직을 함수로 빼고, 반환값을 `return`(상태)과 `echo`(값)로 나눠 쓰는 차이를 확인한다.

```bash
cd /tmp/sh
cat > func.sh <<'EOF'
#!/bin/bash

# 상태를 돌려주는 함수 — return 은 0~255 종료 코드
is_dir() {
    local path="$1"                     # local 로 함수 밖 오염 방지
    [ -d "$path" ] && return 0
    return 1
}

# 값을 돌려주는 함수 — 표준출력 + 명령 치환으로 회수
file_size() {
    local f="$1"
    [ -f "$f" ] || { echo 0; return 1; }
    stat -c %s "$f"
}

# 여러 인자 처리
log() {
    local level="$1"; shift
    printf '%s [%-5s] %s\n' "$(date '+%F %T')" "$level" "$*"
}

usage() {
    echo "Usage: ${0##*/} <경로>" >&2
    return 2
}

[ $# -ge 1 ] || { usage; exit $?; }

TARGET="$1"
if is_dir "$TARGET"; then
    log INFO "$TARGET 은 디렉터리"
else
    SIZE="$(file_size "$TARGET")"       # 값 회수
    RC=$?                               # 상태 회수
    log WARN "$TARGET 크기=$SIZE rc=$RC"
fi

VAR_OUTSIDE="밖의 값"
scope_test() { local VAR_OUTSIDE="안의 값"; echo "  함수 안: $VAR_OUTSIDE"; }
scope_test
echo "  함수 밖: $VAR_OUTSIDE"

declare -f log | head -3                # 함수 정의 확인
exit 0
EOF
chmod +x func.sh
./func.sh /etc;            echo "exit=$?"
./func.sh /etc/hostname;   echo "exit=$?"
./func.sh;                 echo "exit=$?"
```

- 정의: `함수명() { 명령; }` 또는 `function 함수명 { 명령; }` — **정의가 호출보다 먼저** 나와야 함
- 인자: 함수 안에서도 `$1 $2 $# $@` 사용 (스크립트 인자와 별개), `shift` 로 밀기
- `local <변수>` : 함수 지역 변수 — 없으면 **전역 변수**가 되어 밖을 오염시킴
- `return N` : **종료 코드**만 반환 (0~255). 값 반환이 아님
- 값 반환은 `echo`(표준출력) + `RESULT=$(함수명)` 조합
- `exit N` : 스크립트 전체 종료. 함수 안의 `exit` 는 스크립트를 끝내므로 주의
- 종료 코드 관례: `0` 성공, `1` 일반 오류, `2` 사용법 오류, `126` 실행 불가, `127` 명령 없음, `128+N` 시그널 N 로 종료 (`130` = Ctrl+C)
- `declare -f [함수명]` : 정의 확인, `unset -f 함수명` : 해제, `export -f 함수명` : 자식 셸로 전달

**검증**
```bash
./func.sh /etc >/dev/null; echo "디렉터리 exit=$?"
./func.sh      >/dev/null 2>&1; echo "인자 없음 exit=$?"
bash -c 'exit 3'; echo "임의 코드=$?"
bash -c 'nonexist_cmd' 2>/dev/null; echo "명령 없음=$?"
```

```text
디렉터리 exit=0
인자 없음 exit=2
임의 코드=3
명령 없음=127
```

> 📝 **시험 포인트**: `return` 은 종료 코드, 값 전달은 `echo` + 명령 치환. `local` 없이 선언하면 전역이 된다는 점, `$?` 로 함수 결과를 받는 흐름이 출제.

### 8-6. getopts — 옵션 파싱

> **상황**: 백업 스크립트에 `-d 대상지 -k 보존일수 -v` 같은 옵션을 받도록 표준 방식으로 파싱한다.

```bash
cd /tmp/sh
cat > opts.sh <<'EOF'
#!/bin/bash
DEST=/tmp/backup; KEEP=7; VERBOSE=0

usage() {
    cat >&2 <<USAGE
Usage: ${0##*/} [-d DEST] [-k DAYS] [-v] [-h] [파일...]
  -d DEST   대상 디렉터리 (기본 $DEST)
  -k DAYS   보존 일수     (기본 $KEEP)
  -v        상세 출력
USAGE
    exit 2
}

while getopts ":d:k:vh" opt; do
    case "$opt" in
        d) DEST="$OPTARG" ;;
        k) KEEP="$OPTARG" ;;
        v) VERBOSE=1 ;;
        h) usage ;;
        :) echo "옵션 -$OPTARG 에 인자가 필요합니다" >&2; exit 2 ;;
        \?) echo "알 수 없는 옵션: -$OPTARG" >&2; usage ;;
    esac
done
shift $((OPTIND - 1))            # 옵션을 걷어내고 나머지를 $1.. 로

echo "DEST=$DEST KEEP=$KEEP VERBOSE=$VERBOSE"
echo "남은 인자 ($#개): $*"
EOF
chmod +x opts.sh
./opts.sh -d /srv/raid/backup -k 14 -v file1 file2
./opts.sh -vk 30 file3            # 묶어 쓰기도 가능
./opts.sh -d;      echo "exit=$?" # 인자 누락
./opts.sh -z 2>&1; echo "exit=$?" # 미지의 옵션
```

- `getopts <옵션문자열> <변수>` : 셸 내장 옵션 파서 — 한 번 호출마다 옵션 1개를 처리, 더 없으면 거짓 반환 → `while` 과 조합
- 옵션 문자열: 문자 뒤 `:` 는 **인자를 받는 옵션** (`d:` = `-d 값`)
- 맨 앞의 `:` : **오류를 셸이 아닌 스크립트가 처리** (silent mode) → `:` 케이스(인자 누락), `\?` 케이스(미지의 옵션) 사용 가능
- `$OPTARG` : 현재 옵션의 인자, `$OPTIND` : 다음에 처리할 인자 위치
- `shift $((OPTIND - 1))` : 옵션을 걷어내 나머지를 `$1`부터로 재정렬 — **필수 관용구**
- 한계: **긴 옵션(`--dest`)은 지원하지 않음** → 필요하면 `while case` 로 직접 파싱하거나 `getopt`(외부 명령) 사용

**검증**
```bash
./opts.sh -d /srv/raid/backup -k 14 -v a b
./opts.sh
```

```text
DEST=/srv/raid/backup KEEP=14 VERBOSE=1
남은 인자 (2개): a b
DEST=/tmp/backup KEEP=7 VERBOSE=0
남은 인자 (0개): 
```

> 📝 **시험 포인트**: `getopts` 는 셸 **내장**, `getopt` 는 외부 명령(긴 옵션 지원). `$OPTARG`(인자값)·`$OPTIND`(위치)와 `shift $((OPTIND-1))` 관용구가 출제 대상.

### 8-7. trap · date · logger

> **상황**: 스크립트가 중간에 죽어도 임시 파일이 남지 않도록 정리 훅을 걸고, 실행 결과를 시스템 로그에 남긴다.

```bash
cd /tmp/sh
cat > trap.sh <<'EOF'
#!/bin/bash
TMP=$(mktemp /tmp/trap.XXXXXX)
echo "임시 파일: $TMP"

cleanup() {
    local rc=$?
    rm -f "$TMP"
    echo "[cleanup] 임시 파일 삭제, 종료 코드=$rc"
    logger -t labtrap -p user.info "script finished rc=$rc"
}
trap cleanup EXIT                 # 정상·비정상 모두에서 실행
trap 'echo "[INT] Ctrl+C 감지"; exit 130' INT
trap 'echo "[TERM] 종료 요청"; exit 143' TERM
trap '' HUP                       # HUP 무시
trap -p | head -4                 # 등록된 트랩 조회

echo "작업 중... (5초 안에 Ctrl+C 를 눌러 보기)"
sleep 5
echo "정상 완료"
EOF
chmod +x trap.sh && ./trap.sh
ls -l /tmp/trap.* 2>&1 | head -1   # 임시 파일이 남지 않아야 정상

# ▸ date 서식
date                     # 기본
date +%F                 # 2026-09-04 (= %Y-%m-%d)
date +%T                 # 12:34:56 (= %H:%M:%S)
date +%F_%H%M%S          # 파일명용 타임스탬프
date '+%Y-%m-%d %H:%M:%S'
date +%s                 # 유닉스 epoch 초
date -d "@1757000000" +%F
date -d "yesterday" +%F
date -d "7 days ago" +%F
date -d "2026-09-04 +1 month" +%F
date +%j                 # 연중 일수, %U 주차, %a/%A 요일, %b/%B 월

# ▸ logger — rsyslog/journald 로 기록
logger "LAB04 기본 메시지"
logger -t backup "태그를 붙인 메시지"
logger -t backup -p user.err "오류 레벨 메시지"
logger -t backup -i "PID 도 함께 기록"
echo "표준입력에서" | logger -t backup
journalctl -t backup -n 5 --no-pager
grep 'backup' /var/log/messages | tail -3
```

- `trap '<명령>' <시그널…>` : 시그널 수신 시 실행할 명령 등록
  - `EXIT`(0) : 스크립트 종료 시 **항상** 실행 → 임시 파일 정리의 표준 위치
  - `INT`(2, Ctrl+C) · `TERM`(15, `kill` 기본) · `HUP`(1, 터미널 종료) · `ERR`(명령 실패 시, bash 확장)
  - `trap '' <시그널>` : 해당 시그널 **무시**, `trap - <시그널>` : 기본 동작으로 복원, `trap -p` : 등록 목록
  - `SIGKILL`(9)·`SIGSTOP`(19) 은 **가로챌 수 없음** ([[06-process-scheduling-diagnosis|Part 06]])
- `mktemp` : 충돌 없는 임시 파일 생성 (`-d` 는 디렉터리)
- `date +<서식>` : `%Y` 연 `%m` 월 `%d` 일 `%H` 시 `%M` 분 `%S` 초 `%F`=`%Y-%m-%d` `%T`=`%H:%M:%S` `%s` epoch `%j` 연중일 `%a` 요일
- `date -d "<문자열>"` : 상대·절대 시각 계산 (`yesterday`, `7 days ago`, `@epoch`)
- `logger` : syslog 에 메시지 기록 — `-t <태그>` 태그(**t**ag), `-p <facility.level>` 우선순위(**p**riority), `-i` PID 포함, `-s` 화면에도 출력 ([[07-boot-systemd-log|Part 07]])

**검증**
```bash
./trap.sh >/dev/null; ls /tmp/trap.* 2>&1 | head -1
journalctl -t labtrap -n 2 --no-pager
date +%F_%H%M%S | grep -cE '^[0-9]{4}-[0-9]{2}-[0-9]{2}_[0-9]{6}$'
```

```text
ls: cannot access '/tmp/trap.*': No such file or directory
Sep 04 ... srv01 labtrap[...]: script finished rc=0
1
```

> 📝 **시험 포인트**: `trap '명령' EXIT` 은 어떤 경로로 끝나도 실행 — 임시 파일 정리·잠금 해제의 정석. `SIGKILL(9)` 은 trap 불가라는 점이 시그널 문항과 연결.

### 8-8. `/usr/local/bin/backup.sh` 작성 — 백업 자동화

> **상황**: 지금까지의 요소(`tar`·`sha256sum`·`find -mtime`·`logger`·`trap`·`getopts`·`set -euo pipefail`)를 한 스크립트로 묶는다. README 2-4 사양의 정본이며 [[06-process-scheduling-diagnosis|Part 06]] cron, [[12-backup-recovery-review|Part 12]] 최종판에서 그대로 재사용된다.

⚠️ 대상지 기본값은 **`/tmp/backup`**. RAID 볼륨 `/srv/raid` 는 [[05-disk-lvm-raid-swap-quota|Part 05]] 에서 생성되므로, Part 05 완료 후 스크립트의 `DEST` 기본값을 `/srv/raid/backup` 으로 바꾸거나 `-d /srv/raid/backup` 으로 호출한다.

```bash
cat > /usr/local/bin/backup.sh <<'EOF'
#!/bin/bash
#
# backup.sh — /data /srv/share /etc 를 tar.gz 로 백업, SHA-256 기록, 오래된 백업 정리
# 사용: backup.sh [-d 대상지] [-k 보존일수] [-v]
# 로그: /var/log/backup.log 및 syslog(tag=backup)
#
set -euo pipefail

DEST="${DEST:-/tmp/backup}"          # ⚠️ Part 05 완료 후 /srv/raid/backup 으로 변경
KEEP=7
VERBOSE=0
TAG=backup
LOG=/var/log/backup.log
TARGETS=(/data /srv/share /etc)

usage() {
    echo "Usage: ${0##*/} [-d DEST] [-k DAYS] [-v]" >&2
    exit 2
}

log() {
    printf '%s [%-5s] %s\n' "$(date '+%F %T')" "$1" "${*:2}" >> "$LOG"
    logger -t "$TAG" -p "user.${1,,}" "${*:2}"
}

while getopts ":d:k:vh" opt; do
    case "$opt" in
        d) DEST="$OPTARG" ;;
        k) KEEP="$OPTARG" ;;
        v) VERBOSE=1 ;;
        h) usage ;;
        :) echo "옵션 -$OPTARG 에 인자 필요" >&2; exit 2 ;;
        \?) usage ;;
    esac
done
shift $((OPTIND - 1))

STAMP="$(date +%F)"
ARCHIVE="$DEST/backup-${STAMP}.tar.gz"
TARFLAGS="czf"
[ "$VERBOSE" -eq 1 ] && TARFLAGS="czvf"

cleanup() {
    local rc=$?
    if [ "$rc" -ne 0 ]; then
        log ERR "backup FAILED rc=$rc archive=$ARCHIVE"
        rm -f "$ARCHIVE" "$ARCHIVE.sha256"     # 불완전 산출물 제거
    fi
    exit "$rc"
}
trap cleanup EXIT
trap 'exit 130' INT
trap 'exit 143' TERM

mkdir -p "$DEST"
[ -w "$DEST" ] || { echo "대상지에 쓸 수 없음: $DEST" >&2; exit 1; }

# 1) 존재하는 대상만 수집 (선행 / 제거 → tar -C / 로 상대 경로 저장)
SRC=()
for t in "${TARGETS[@]}"; do
    if [ -e "$t" ]; then SRC+=("${t#/}"); else log WARN "대상 없음, 건너뜀: $t"; fi
done
[ "${#SRC[@]}" -gt 0 ] || { echo "백업 대상이 하나도 없음" >&2; exit 1; }

log INFO "START dest=$DEST targets=${SRC[*]} keep=${KEEP}d"

# 2) 아카이브 생성 — tar 종료 코드 1(읽는 중 변경)은 경고로 취급
if ! tar "$TARFLAGS" "$ARCHIVE" -C / --warning=no-file-changed \
        --exclude='*/tmp/*' --exclude='*.sock' "${SRC[@]}"; then
    rc=$?
    [ "$rc" -le 1 ] || exit "$rc"
    log WARN "tar 경고 rc=$rc (읽는 중 변경된 파일 있음)"
fi

# 3) 무결성 해시 기록 + 즉시 검증
( cd "$DEST" && sha256sum "$(basename "$ARCHIVE")" > "$(basename "$ARCHIVE").sha256" )
( cd "$DEST" && sha256sum -c --quiet "$(basename "$ARCHIVE").sha256" )

# 4) 보존 기간 초과분 삭제
DELETED=$(find "$DEST" -maxdepth 1 -type f -name 'backup-*.tar.gz*' -mtime "+$KEEP" -print -delete | wc -l)

SIZE=$(du -h "$ARCHIVE" | cut -f1)
log INFO "DONE archive=$ARCHIVE size=$SIZE purged=${DELETED}"
echo "$ARCHIVE"
EOF

chmod 755 /usr/local/bin/backup.sh
ls -l /usr/local/bin/backup.sh
bash -n /usr/local/bin/backup.sh && echo "문법 검사 OK"
```

- `set -euo pipefail` : 실패 즉시 종료 + 미정의 변수 오류 + 파이프 실패 감지 (7-8)
- `${*:2}` : 위치 매개변수 2번째부터 끝까지 — `log LEVEL 메시지…` 형태를 지원
- `${1,,}` : 소문자화 → `user.info` / `user.err` facility.level 구성
- `${t#/}` : 앞의 `/` 제거 (7-4) → `tar -C /` 와 조합해 **선행 `/` 제거 경고 없이** 상대 경로 아카이브
- `--warning=no-file-changed` : `/etc` 처럼 활성 디렉터리를 백업할 때 나오는 경고 억제
- `trap cleanup EXIT` : 실패 시 불완전한 아카이브를 남기지 않음
- `find … -mtime +$KEEP -print -delete` : 삭제한 항목을 출력하면서 제거 → 개수 집계

**검증**
```bash
# ▸ 실행
/usr/local/bin/backup.sh -v -d /tmp/backup -k 7

# ▸ ① 아카이브 내용 확인
ls -lh /tmp/backup/
tar tzvf /tmp/backup/backup-$(date +%F).tar.gz | head -5
tar tzf  /tmp/backup/backup-$(date +%F).tar.gz | wc -l

# ▸ ② 해시 검증
cd /tmp/backup && sha256sum -c backup-$(date +%F).tar.gz.sha256

# ▸ ③ 로그 확인
tail -5 /var/log/backup.log
journalctl -t backup -n 5 --no-pager

# ▸ ④ 실패 경로 확인 — 쓸 수 없는 대상지
/usr/local/bin/backup.sh -d /proc/nowhere; echo "실패 exit=$?"

# ▸ ⑤ 추적 실행
bash -x /usr/local/bin/backup.sh -d /tmp/backup 2>&1 | head -20
```

```text
-rw-r--r--. 1 root root  ... backup-2026-09-04.tar.gz
-rw-r--r--. 1 root root   85 ... backup-2026-09-04.tar.gz.sha256
drwxr-xr-x root/root ... etc/
-rw-r--r-- root/root ... etc/fstab
...
backup-2026-09-04.tar.gz: OK
2026-09-04 03:30:01 [INFO ] START dest=/tmp/backup targets=etc keep=7d
2026-09-04 03:30:14 [INFO ] DONE archive=/tmp/backup/backup-2026-09-04.tar.gz size=... purged=0
Sep 04 ... srv01 backup[...]: DONE archive=...
실패 exit=1
```

> 📝 **시험 포인트**: 실기 "백업 스크립트 작성" 문항은 ① shebang ② `tar czf` ③ 날짜 파일명 `$(date +%F)` ④ 로그 기록 ⑤ 오래된 파일 정리(`find -mtime +7 -delete`) 요소를 채점한다. cron 등록은 [[06-process-scheduling-diagnosis|Part 06]] 의 `/etc/cron.d/lab-backup`.

### 8-9. `/usr/local/bin/sysreport.sh` 작성 — 진단 리포트

> **상황**: 장애 신고가 들어왔을 때 한 번에 상태를 뜨는 진단 스크립트를 만든다. [[06-process-scheduling-diagnosis|Part 06]] 의 `at`·cron·timer 와 [[07-boot-systemd-log|Part 07]] 의 `lab-monitor.service` 에서 이 스크립트를 그대로 호출한다.

```bash
cat > /usr/local/bin/sysreport.sh <<'EOF'
#!/bin/bash
#
# sysreport.sh — 시스템 상태 요약을 /var/log/sysreport-<날짜>.txt 로 저장
# 사용: sysreport.sh [-o 출력파일] [-s]   (-s: 화면에도 출력)
#
set -u                     # ⚠️ pipefail 미사용: `top | head` 의 SIGPIPE(141) 를 실패로 보지 않기 위함

OUT="/var/log/sysreport-$(date +%F).txt"
SHOW=0

while getopts ":o:sh" opt; do
    case "$opt" in
        o) OUT="$OPTARG" ;;
        s) SHOW=1 ;;
        h|\?) echo "Usage: ${0##*/} [-o FILE] [-s]" >&2; exit 2 ;;
    esac
done

section() { printf '\n===== %s =====\n' "$1"; }

{
    printf '########## SYSTEM REPORT  %s  (%s) ##########\n' "$(date '+%F %T %Z')" "$(hostname -f)"
    section "date / uptime";  date; uptime
    section "top (1회 스냅샷)"; top -b -n 1 | head -15
    section "vmstat 1 3";      vmstat 1 3
    section "df -hT";          df -hT -x tmpfs -x devtmpfs
    section "free -h";         free -h
    section "ss -tuln";        ss -tuln
} > "$OUT" 2>&1

chmod 640 "$OUT"
logger -t sysreport "report written to $OUT ($(wc -l < "$OUT") lines)"
[ "$SHOW" -eq 1 ] && cat "$OUT"
echo "$OUT"
EOF

chmod 755 /usr/local/bin/sysreport.sh
bash -n /usr/local/bin/sysreport.sh && echo "문법 검사 OK"
```

- `set -u` 만 사용: `top -b -n 1 | head -15` 는 `head` 가 먼저 끝나면서 `top` 이 SIGPIPE(종료 코드 141)로 죽는다. `set -o pipefail` 을 켜면 정상 동작이 실패로 잡히므로 **의도적으로 제외**
- `{ … } > "$OUT" 2>&1` : 명령 그룹의 출력을 **한 번에** 리다이렉션 (6-5)
- `top -b -n 1` : 배치 모드(**b**atch) 1회 반복(**n**umber) — 대화형이 아니므로 스크립트에서 사용
- `vmstat 1 3` : 1초 간격 3회 — 첫 행은 부팅 후 평균이라 2~3행을 읽음
- `df -hT -x tmpfs -x devtmpfs` : 유형 포함, 가상 파일시스템 제외 (e**x**clude)
- `ss -tuln` : TCP(**t**)·UDP(**u**)·리스닝(**l**)·숫자(**n**) 소켓 목록
- `chmod 640` : 리포트에 소켓·프로세스 정보가 담기므로 일반 사용자 열람 차단

**검증**
```bash
/usr/local/bin/sysreport.sh
ls -l /var/log/sysreport-$(date +%F).txt
grep -c '^===== ' /var/log/sysreport-$(date +%F).txt      # 섹션 6개
head -3 /var/log/sysreport-$(date +%F).txt
grep -A3 '===== free -h' /var/log/sysreport-$(date +%F).txt
journalctl -t sysreport -n 2 --no-pager
/usr/local/bin/sysreport.sh -o /tmp/report.txt -s | tail -1
```

```text
/var/log/sysreport-2026-09-04.txt
-rw-r-----. 1 root root ... /var/log/sysreport-2026-09-04.txt
6
########## SYSTEM REPORT  2026-09-04 10:12:33 KST  (srv01.lab.local) ##########

===== date / uptime =====
===== free -h =====
               total        used        free      shared  buff/cache   available
Mem:           3.7Gi       ...         ...         ...        ...         ...
Swap:             0B          0B          0B
Sep 04 ... srv01 sysreport[...]: report written to /var/log/sysreport-2026-09-04.txt (... lines)
```

> 📝 **시험 포인트**: `top -b -n 1` 은 스크립트·리포트용 배치 모드. `vmstat 간격 횟수`, `df -hT`, `free -h`, `ss -tuln` 은 각각 단독으로도 필기 출제되는 진단 명령이며 상세 해석은 [[06-process-scheduling-diagnosis|Part 06]].

---

## 9. vi / vim 편집기

### 9-1. 3가지 모드와 전환

> **상황**: 서버에 GUI 가 없으므로 설정 파일 편집은 전부 `vi` 로 한다. 먼저 모드 개념과 전환 경로를 정리한다.

```bash
dnf install -y vim-enhanced       # vi(vim-minimal) 는 기본 설치, 전체 기능은 vim-enhanced
vim --version | head -2
cp /etc/skel/.bashrc /tmp/bashrc.lab      # 실습용 사본 (원본은 건드리지 않음)
vi /tmp/bashrc.lab
```

```text
                 i a o O I A R s S      :  또는  /  ?
   ┌────────────┐ ────────────→ ┌────────────┐ ────────────→ ┌────────────┐
   │  명령 모드  │               │  입력 모드  │               │  실행 모드  │
   │ (command)  │ ←──────────── │  (insert)  │               │ (ex/last)  │
   └────────────┘     ESC        └────────────┘               └────────────┘
         ↑                                                          │
         └──────────────────────  명령 실행 후 자동 복귀 / ESC ────────┘
```

| 모드 | 다른 이름 | 진입 | 복귀 | 역할 |
| --- | --- | --- | --- | --- |
| 명령 모드 | command / normal | **시작 시 기본** | — | 이동·삭제·복사·붙여넣기 |
| 입력 모드 | insert / edit | `i a o O I A R s S` | `ESC` | 실제 문자 입력 |
| 실행 모드 | ex / last-line / colon | `:` `/` `?` | `Enter` 또는 `ESC` | 저장·종료·치환·설정 |

- **모든 전환의 허브는 명령 모드** — 입력 모드에서 실행 모드로 바로 갈 수 없음(`ESC` 경유)
- 현재 모드 확인: 하단에 `-- INSERT --` 표시 (`:set showmode`, vim 기본 on)
- `vi 파일` / `vi +N 파일`(N행에서 시작) / `vi +/패턴 파일`(패턴 위치에서 시작) / `vi -R 파일`(읽기 전용)
- `view 파일` = `vi -R`, `vim -d 파일1 파일2` = `vimdiff`

> 📝 **시험 포인트**: "3가지 모드와 전환 키" 는 필기 서술형 단골. 입력 모드 → 실행 모드는 **ESC 를 거쳐야** 한다는 것이 핵심.

### 9-2. 입력 모드 진입 키

> **상황**: 커서 기준 어디에 문자가 들어가는지가 키마다 다르다. 실습 사본에서 하나씩 눌러 본다.

| 키 | 동작 |
| --- | --- |
| `i` | 커서 **앞**부터 입력 (**i**nsert) |
| `a` | 커서 **뒤**부터 입력 (**a**ppend) |
| `I` | 행의 **첫 문자 앞**부터 입력 (대문자 = 행 단위) |
| `A` | 행의 **끝**부터 입력 |
| `o` | 커서 행 **아래**에 새 행 열고 입력 (**o**pen) |
| `O` | 커서 행 **위**에 새 행 열고 입력 |
| `s` | 커서 문자 1개 지우고 입력 (**s**ubstitute) |
| `S` | 행 전체를 지우고 입력 |
| `cw` | 커서 위치 단어를 지우고 입력 (**c**hange **w**ord) |
| `C` | 커서부터 행 끝까지 지우고 입력 (= `c$`) |
| `R` | 덮어쓰기(replace) 모드로 입력 |

- 소문자 = 커서 기준, **대문자 = 행 기준**(`I`/`A`) 또는 위쪽(`O`) 이라는 규칙으로 외운다
- 입력 종료는 항상 `ESC`

> 📝 **시험 포인트**: `i`/`a`(앞/뒤), `I`/`A`(행 처음/행 끝), `o`/`O`(아래/위 새 행) 짝이 그대로 출제.

### 9-3. 저장·종료

> **상황**: 편집 결과를 저장하거나 버린다. 잘못 저장하면 되돌릴 수 없으므로 실습 사본에서 연습한다.

| 명령 | 동작 |
| --- | --- |
| `:w` | 저장 (**w**rite) |
| `:w <파일명>` | 다른 이름으로 저장 |
| `:w!` | 강제 저장 (읽기 전용 속성 무시, 권한은 필요) |
| `:q` | 종료 (**q**uit) — 변경분이 있으면 거부 |
| `:q!` | 저장하지 않고 강제 종료 |
| `:wq` | 저장 후 종료 |
| `:wq!` | 강제 저장 후 종료 |
| `:x` | **변경이 있을 때만** 저장 후 종료 (mtime 불필요 갱신 방지) |
| `ZZ` | `:x` 와 동일 (명령 모드, 콜론 없음) |
| `ZQ` | `:q!` 와 동일 |
| `:e!` | 저장하지 않고 마지막 저장 상태로 되돌려 다시 읽기 |
| `:w >> <파일>` | 현재 내용을 다른 파일 뒤에 덧붙임 |
| `:N,Mw <파일>` | N~M 행만 다른 파일로 저장 |

- 권한이 없어 저장이 안 될 때: `:w !sudo tee %` (현재 파일명 `%` 를 sudo tee 로 전달)
- 비정상 종료 시 스왑 파일(`.파일명.swp`) 생성 → 재편집 시 복구 여부 질문. `vi -r 파일` 로 복구, 필요 없으면 `.swp` 삭제

> 📝 **시험 포인트**: `:wq` 와 `:x`(= `ZZ`) 의 차이 — `:x` 는 변경이 없으면 파일을 다시 쓰지 않아 타임스탬프가 유지됨. `:q!` 는 변경 파기.

### 9-4. 이동 · 편집 · 되돌리기

> **상황**: 대용량 설정 파일에서 원하는 위치로 빠르게 이동하고, 행 단위 삭제·복사·붙여넣기를 수행한다.

| 분류 | 키 | 동작 |
| --- | --- | --- |
| 이동 | `h` `j` `k` `l` | 좌 · 하 · 상 · 우 (1칸) |
| 이동 | `w` `b` `e` | 다음 단어 앞 · 이전 단어 앞 · 단어 끝 |
| 이동 | `0` `^` `$` | 행 맨 앞 · 행 첫 문자 · 행 끝 |
| 이동 | `gg` `G` `:N` `NG` | 첫 행 · 마지막 행 · N행으로 |
| 이동 | `Ctrl+f` `Ctrl+b` | 한 화면 아래 · 위 (**f**orward / **b**ackward) |
| 이동 | `Ctrl+d` `Ctrl+u` | 반 화면 아래 · 위 (**d**own / **u**p) |
| 이동 | `H` `M` `L` | 화면 맨 위 · 중간 · 맨 아래 행 |
| 이동 | `{` `}` `%` | 이전 문단 · 다음 문단 · 짝 괄호로 |
| 삭제 | `x` `X` | 커서 문자 삭제 · 앞 문자 삭제 |
| 삭제 | `dd` `3dd` `dw` `d$`(`D`) `dG` | 행 삭제 · 3행 삭제 · 단어 삭제 · 행 끝까지 · 끝 행까지 |
| 복사 | `yy` `3yy` `yw` | 행 복사(**y**ank) · 3행 복사 · 단어 복사 |
| 붙여넣기 | `p` `P` | 커서 **뒤/아래**에 · **앞/위**에 |
| 되돌리기 | `u` `U` `Ctrl+r` | 직전 취소 · 행 전체 복원 · 다시 실행(redo) |
| 반복 | `.` | 직전 편집 명령 반복 |
| 기타 | `J` `~` `>>` `<<` | 다음 행 이어붙이기 · 대소문자 토글 · 들여쓰기 · 내어쓰기 |

- **숫자 접두**가 반복 횟수: `3dd`(3행 삭제) `5yy`(5행 복사) `10j`(10행 아래)
- 삭제(`d`·`x`)는 **잘라내기**여서 버퍼에 남음 → `p` 로 붙여넣으면 이동 효과
- 이름 있는 레지스터: `"ayy`(a 에 복사) → `"ap`(a 에서 붙여넣기)
- 마크: `ma`(현재 위치를 a 로 표시) → `'a`(그 행으로 이동)

> 📝 **시험 포인트**: `dd`(행 삭제) `3dd`(3행) `yy`+`p`(복사·붙여넣기) `u`(undo) `Ctrl+r`(redo) `.`(반복) 은 실기·필기 모두 단답으로 출제.

### 9-5. 검색과 치환

> **상황**: 설정 파일에서 특정 지시자를 찾아 일괄 교체한다. 실기 `sed -i` 문항과 같은 작업을 편집기에서 수행한다.

| 명령 | 동작 |
| --- | --- |
| `/패턴` | **아래 방향** 검색 |
| `?패턴` | **위 방향** 검색 |
| `n` / `N` | 다음 일치 / 반대 방향 다음 일치 |
| `*` / `#` | 커서 위치 단어를 아래/위로 검색 |
| `:s/찾을것/바꿀것/` | **현재 행**의 첫 일치 치환 |
| `:s/찾을것/바꿀것/g` | 현재 행의 모든 일치 치환 |
| `:%s/찾을것/바꿀것/g` | **파일 전체**의 모든 일치 치환 |
| `:%s/찾을것/바꿀것/gc` | 파일 전체 치환 + **건건이 확인**(**c**onfirm) |
| `:%s/찾을것/바꿀것/gi` | 대소문자 무시 |
| `:N,Ms/a/b/g` | N~M 행 범위만 치환 |
| `:.,$s/a/b/g` | 현재 행부터 끝까지 |
| `:g/패턴/d` | 패턴 일치 행 **전부 삭제** |
| `:v/패턴/d` (= `:g!`) | 패턴에 **일치하지 않는** 행 삭제 |

- `%` = 전체 행 범위(= `1,$`), `.` = 현재 행, `$` = 마지막 행
- 치환 확인 모드(`c`)에서: `y` 치환, `n` 건너뜀, `a` 이후 전부, `q` 중단, `l` 이번 것만 치환 후 종료
- 구분자는 `/` 대신 `#`·`|` 등 사용 가능 → 경로 치환 시 `:%s#/usr/local#/opt#g`
- 패턴은 vim 정규식(BRE 계열) — `\d`·`\+` 등 vim 고유 표기 존재

> 📝 **시험 포인트**: 전체 치환은 `:%s/a/b/g`, 확인하며 치환은 뒤에 `c` 추가. `g` 플래그가 없으면 **행마다 첫 일치만** — `sed` 와 동일한 함정.

### 9-6. 설정 · 파일 삽입 · 외부 명령 · 비주얼 모드

> **상황**: 행 번호를 켜고, 다른 파일 내용을 끌어오고, 편집 중에 셸 명령 결과를 삽입한다.

| 명령 | 동작 |
| --- | --- |
| `:set nu` / `:set nonu` | 행 번호 표시 / 해제 (**nu**mber) |
| `:set rnu` | 상대 행 번호 (**r**elative **nu**mber) |
| `:set ai` / `:set noai` | 자동 들여쓰기 (**a**uto**i**ndent) |
| `:set ts=4` `:set sw=4` `:set et` | 탭 폭 4 (**t**ab**s**top) · 들여쓰기 폭 (**s**hift**w**idth) · 탭 대신 공백 (**e**xpand**t**ab) |
| `:set hlsearch` / `:set nohlsearch` | 검색 결과 강조 / 해제 |
| `:set ic` / `:set noic` | 검색 시 대소문자 무시 (**i**gnore**c**ase) |
| `:set list` / `:set nolist` | 탭·행끝 등 비출력 문자 표시 |
| `:set paste` | 붙여넣기 시 자동 들여쓰기 방지 |
| `:set` | 기본값과 다른 설정 전체 조회 |
| `:r <파일>` | 커서 아래에 파일 내용 삽입 (**r**ead) |
| `:r !<명령>` | 명령 실행 결과를 삽입 |
| `:!<명령>` | 편집기를 벗어나지 않고 셸 명령 실행 |
| `:sh` | 임시 셸로 나갔다가 `exit` 로 복귀 |
| `:e <파일>` / `:e!` | 다른 파일 열기 / 현재 파일 다시 읽기(변경 파기) |
| `:sp` `:vs` `Ctrl+w w` | 가로 분할 · 세로 분할 · 창 전환 |
| `:noh` | 검색 강조 일시 해제 |

- 비주얼 모드: `v` 문자 단위 · `V` **행 단위** · `Ctrl+v` **블록(열) 단위** → 선택 후 `d` 삭제 · `y` 복사 · `>` 들여쓰기
  - 블록 모드 + `I` + 문자 + `ESC` : 여러 행 앞에 **동시 삽입**(주석 일괄 처리에 유용)
- 영구 설정은 `~/.vimrc` (사용자) 또는 `/etc/vimrc`(전역)

```bash
cat > ~/.vimrc <<'EOF'
set nu                 " 행 번호
set ai                 " 자동 들여쓰기
set ts=4 sw=4 et       " 탭 폭 4, 공백으로
set hlsearch incsearch " 검색 강조 + 점진 검색
set showmode ruler     " 모드·커서 위치 표시
syntax on              " 문법 강조 (vim-enhanced 필요)
EOF
cat ~/.vimrc
```

> 📝 **시험 포인트**: `:set nu`(행 번호)는 필기 단답 최빈출. `:r 파일`(파일 삽입) 과 `:!명령`(셸 실행) 구분, 영구 설정 파일은 `~/.vimrc`.

### 9-7. 실습 — /etc/skel/.bashrc 사본 편집과 diff 검증

> **상황**: 앞의 키들을 실제로 눌러 사본을 편집하고, 원본과 `diff` 로 대조해 편집 결과가 의도대로인지 검증한다.

```bash
cp /etc/skel/.bashrc /tmp/bashrc.lab
cp /tmp/bashrc.lab /tmp/bashrc.orig       # 비교용 원본 보관
vi /tmp/bashrc.lab
```

vi 안에서 순서대로 수행:

```text
:set nu                     " 행 번호 켜기
gg                          " 첫 행으로
G                           " 마지막 행으로
:1                          " 1행으로
o                           " 아래에 새 행 열기 → 입력 모드
# LAB04 편집 실습          " 입력
ESC                         " 명령 모드 복귀
yy                          " 방금 행 복사
p                           " 아래에 붙여넣기
dd                          " 붙여넣은 행 삭제 (되돌림)
/alias                      " alias 검색
n                           " 다음 일치
:%s/alias/ALIAS/gc          " 전체 치환, 건건이 y/n 확인
u                           " 방금 치환 취소
Ctrl+r                      " 다시 실행
:%s/ALIAS/alias/g           " 원복
:g/^#/d                     " 주석 행 전부 삭제
:r /etc/hostname            " 호스트명 파일 내용 삽입
:r !date +%F                " 명령 결과 삽입
:!wc -l %                   " 셸 명령 실행 (% = 현재 파일)
:wq                         " 저장 후 종료
```

**검증**
```bash
diff /tmp/bashrc.orig /tmp/bashrc.lab            # 변경 요약
diff -u /tmp/bashrc.orig /tmp/bashrc.lab | head -20
grep -c '^#' /tmp/bashrc.orig /tmp/bashrc.lab    # 주석 행이 0 이 되어야 정상
tail -2 /tmp/bashrc.lab                          # 삽입한 호스트명·날짜
bash -n /tmp/bashrc.lab && echo "문법 OK"
diff /etc/skel/.bashrc /tmp/bashrc.orig && echo "원본은 무손상"
```

```text
1a2
> # LAB04 편집 실습
...
/tmp/bashrc.orig:8
/tmp/bashrc.lab:0
srv01.lab.local
2026-09-04
문법 OK
원본은 무손상
```

> 📝 **시험 포인트**: 실기에서 "vi 로 설정 파일의 X 를 Y 로 바꾸시오" 는 `:%s/X/Y/g` 로 답한다. 편집 후 `diff` 로 검증하는 습관이 그대로 채점 근거가 된다.

### 9-8. vimtutor · nano · emacs

> **상황**: 학습·대체 편집기 선택지를 확인한다.

```bash
vimtutor                     # vim 대화형 학습 (vim-enhanced 필요, 약 30분)
vimtutor ko 2>/dev/null      # 한국어판 (설치되어 있으면)
dnf install -y nano
nano /tmp/bashrc.lab         # Ctrl+O 저장, Ctrl+X 종료, Ctrl+W 검색, Ctrl+K 행 삭제
dnf list --available emacs | head -3    # ※ 미설치 — 필요 시 dnf install -y emacs
echo $EDITOR; echo $VISUAL              # 기본 편집기 환경 변수
export EDITOR=vim                        # visudo·crontab -e 가 사용하는 편집기
```

- `vimtutor` : vim 공식 튜토리얼 — 사본을 열어 실습하므로 안전
- `nano` : 모드 없는 단순 편집기, 하단에 단축키가 표시됨 (`^` = Ctrl). 초보자용 대체
- `emacs` : Lisp 기반 확장형 편집기 — 필기에서 "vi/emacs 계열 구분" 정도로 출제, RHEL 9 기본 미설치
- `$EDITOR` / `$VISUAL` : `visudo`·`crontab -e`·`systemctl edit` 이 호출하는 편집기 지정
- Rocky 9 minimal 은 `vim-minimal`(= `vi`) 만 설치 → `vim` 명령·문법 강조는 `vim-enhanced` 필요

> 📝 **시험 포인트**: 리눅스 대표 편집기 계열은 **vi/vim** 과 **emacs**, 초보자용은 `nano`/`pico`. `visudo` 가 `$EDITOR` 를 따른다는 점은 [[03-user-group-permission|Part 03]] 과 연결.

---

## 10. 특수 파일과 기타 도구

### 10-1. /dev 특수 파일

> **상황**: 스크립트·테스트에서 자주 쓰는 의사 장치의 동작을 하나씩 확인한다.

| 장치 | 유형 | 읽기 | 쓰기 | 대표 용도 |
| --- | --- | --- | --- | --- |
| `/dev/null` | 문자 | 즉시 EOF | 버림 | 출력 억제 (`> /dev/null 2>&1`) |
| `/dev/zero` | 문자 | 무한한 NUL(`\0`) | 버림 | 0 채운 파일 생성 (`dd`, 스왑 파일) |
| `/dev/random` | 문자 | 난수 | 엔트로피 추가 | 암호키 — 엔트로피 부족 시 대기 가능 |
| `/dev/urandom` | 문자 | 난수 (비차단) | 엔트로피 추가 | 일반 난수 — 실무 기본 선택 |
| `/dev/full` | 문자 | 무한한 NUL | **항상 ENOSPC 오류** | "디스크 꽉 참" 오류 처리 테스트 |
| `/dev/tty` | 문자 | 현재 터미널 | 현재 터미널 | 리다이렉션 중에도 화면에 직접 출력 |
| `/dev/stdin` `/dev/stdout` `/dev/stderr` | 링크 | fd 0/1/2 | 동일 | 파일명을 요구하는 명령에 스트림 전달 |

```bash
ls -l /dev/null /dev/zero /dev/random /dev/urandom /dev/full /dev/tty
head -c 16 /dev/zero | od -An -tx1              # 전부 00
head -c 16 /dev/urandom | od -An -tx1           # 매번 다른 값
echo "버려짐" > /dev/null; echo "exit=$?"
echo "가득 참 테스트" > /dev/full; echo "exit=$?"   # No space left on device
cat /dev/null | wc -c                            # 0
dd if=/dev/zero of=/tmp/zero64k bs=1K count=64 status=none; ls -l /tmp/zero64k
echo "리다이렉션 중에도 화면에" > /dev/tty
grep root /etc/passwd > /tmp/out.txt 2>/dev/tty
sort /dev/stdin <<< $'c\na\nb'                   # 파일명 자리에 표준입력
```

- `ls -l /dev/*` 의 크기 자리에 나오는 두 숫자 = **주 번호(major, 드라이버)·부 번호(minor, 개별 장치)**
- `/dev/random` vs `/dev/urandom` : RHEL 9 커널에서는 초기화 이후 품질 차이가 없어 **`urandom` 사용 권장**
- `/dev/full` 은 "쓰기 실패" 경로를 테스트할 때 사용 (스크립트 오류 처리 검증)

**검증**
```bash
head -c 8 /dev/zero | od -An -tx1
echo x > /dev/full; echo "full exit=$?"
ls -l /dev/null | awk '{print $1, $5, $6}'
```

```text
 00 00 00 00 00 00 00 00
bash: echo: write error: No space left on device
full exit=1
crw-rw-rw-. 1, 3
```

> 📝 **시험 포인트**: `/dev/null` = 버리기, `/dev/zero` = 0 채우기(`dd if=/dev/zero of=/swapfile`, 필기 R06-34·R07-41). 장치 파일의 크기 자리는 major/minor 번호.

### 10-2. 명명 파이프(FIFO) — mkfifo 로 프로세스 간 통신

> **상황**: 두 셸을 띄워 한쪽에서 읽고 한쪽에서 쓰는 단방향 통신을 실제로 성립시킨다.

```bash
mkfifo /tmp/lab.pipe
ls -l /tmp/lab.pipe                # 첫 문자 p, 크기 0
file /tmp/lab.pipe

# ▸ 방법 ① 한 터미널에서 백그라운드 읽기 + 전경 쓰기
cat < /tmp/lab.pipe &              # 읽기 대기 (백그라운드)
READER=$!
echo "LAB04 FIFO 메시지 1" > /tmp/lab.pipe
wait "$READER"

# ▸ 방법 ② 터미널 2개 사용
# [터미널 1] 읽기 — 쓰는 쪽이 열릴 때까지 블로킹
cat /tmp/lab.pipe
# [터미널 2] 쓰기
echo "터미널 2 에서 보냄" > /tmp/lab.pipe
date                    > /tmp/lab.pipe

# ▸ 파이프를 통한 명령 결과 전달
( ps aux | head -5 > /tmp/lab.pipe ) &
wc -l < /tmp/lab.pipe

# ▸ 사용 후 정리 (일반 파일과 동일하게 삭제)
rm -f /tmp/lab.pipe
```

- `mkfifo [-m 모드] <경로>` : 명명 파이프(FIFO) 생성 — 디스크에 **이름만** 존재하고 데이터는 커널 버퍼를 통해 흐름
- 익명 파이프(`|`)와 달리 **부모-자식 관계가 없는 프로세스끼리도** 통신 가능
- 읽는 쪽과 쓰는 쪽이 **둘 다 열려야** 진행 — 한쪽만 열면 블로킹(`cat` 이 멈춘 것처럼 보임)
- 단방향 — 양방향이 필요하면 FIFO 2개 또는 유닉스 도메인 소켓 사용
- `ls -l` 첫 문자 `p`, 크기는 항상 0 (데이터가 파일에 저장되지 않음)

**검증**
```bash
mkfifo /tmp/lab.pipe
ls -l /tmp/lab.pipe | cut -c1-11
( sleep 0.3; echo "검증 메시지" > /tmp/lab.pipe ) &
cat /tmp/lab.pipe
stat -c '%F %s' /tmp/lab.pipe
rm -f /tmp/lab.pipe
```

```text
prw-r--r--.
검증 메시지
fifo 0
```

> 📝 **시험 포인트**: `ls -l` 첫 문자 `p` = 명명 파이프(FIFO), 생성 명령은 `mkfifo`. "프로세스 간 단방향 통신용 특수 파일"(필기 R04-9 계열 파일 유형 문항).

### 10-3. 소켓 파일 · mknod · file 로 유형 판별

> **상황**: 7가지 파일 유형을 한 화면에서 확인하고, `ls -l` 첫 문자와 `file`·`stat` 출력이 어떻게 대응되는지 맞춰 본다.

```bash
# ▸ 소켓 파일 — 데몬이 만들어 둔 것을 조회 (직접 생성하지 않음)
find /run -type s 2>/dev/null | head -5
ls -l /run/dbus/system_bus_socket 2>/dev/null
ls -l /run/systemd/private 2>/dev/null
ss -xl | head -5                      # 유닉스 도메인 소켓 목록

# ▸ mknod — 장치 파일 수동 생성 (※ udev 가 자동 생성하므로 참고용)
mknod /tmp/mynull c 1 3               # 문자 장치, major 1 minor 3 (= /dev/null)
ls -l /tmp/mynull
echo "버려짐" > /tmp/mynull; echo "exit=$?"
mknod /tmp/myfifo p                   # 명명 파이프 (= mkfifo)
ls -l /tmp/myfifo
rm -f /tmp/mynull /tmp/myfifo

# ▸ 7가지 유형 한 번에 판별
mkfifo /tmp/t.pipe
ln -sf /etc/passwd /tmp/t.link
for f in /etc/passwd /etc /tmp/t.link /dev/vda /dev/null /tmp/t.pipe $(find /run -type s 2>/dev/null | head -1); do
    printf '%-32s %s | %s\n' "$f" "$(stat -Lc %F "$f" 2>/dev/null || stat -c %F "$f")" "$(ls -ld "$f" | cut -c1)"
done
file /etc/passwd /etc /tmp/t.link /dev/vda /dev/null /tmp/t.pipe
rm -f /tmp/t.pipe /tmp/t.link
```

| `ls -l` 첫 문자 | 유형 | `stat -c %F` | 생성/예시 |
| --- | --- | --- | --- |
| `-` | 일반 파일 | `regular file` | `touch`, `/etc/passwd` |
| `d` | 디렉터리 | `directory` | `mkdir`, `/etc` |
| `l` | 심볼릭 링크 | `symbolic link` | `ln -s` |
| `b` | 블록 장치 | `block special file` | `/dev/vda` (디스크) |
| `c` | 문자 장치 | `character special file` | `/dev/null`, `/dev/tty` |
| `p` | 명명 파이프 | `fifo` | `mkfifo`, `mknod … p` |
| `s` | 소켓 | `socket` | 데몬이 생성, `/run/*.sock` |

- `mknod <경로> <b|c|p> [major] [minor]` : 장치 파일 생성 — 현대 리눅스는 **udev** 가 `/dev` 를 자동 관리하므로 수동 생성은 예외적 ([[05-disk-lvm-raid-swap-quota|Part 05]])
- `stat -c %F` 는 유형 문자열, `stat -L` 은 심볼릭 링크를 따라감
- `file` 은 **내용 기반** 판별(magic number) → 확장자와 무관, `file -b` 이름 생략, `file -i` MIME

**검증**
```bash
mkfifo /tmp/t.pipe
ls -ld /etc /etc/passwd /dev/vda /dev/null /tmp/t.pipe | cut -c1
rm -f /tmp/t.pipe
```

```text
d
-
b
c
p
```

> 📝 **시험 포인트**: `ls -l` 첫 문자 7종(`- d l b c p s`)은 필기 최빈출(R04-9, R08-3). 블록 장치=디스크, 문자 장치=터미널, `p`=FIFO, `s`=소켓.

### 10-4. truncate · shred · install · rename

> **상황**: 파일 크기 조작, 안전 삭제, 권한 지정 설치, 일괄 이름 변경을 확인한다.

```bash
cd /tmp
# ▸ truncate — 크기 조정 (sparse 파일 생성 포함)
truncate -s 100M sparse.img          # 100MB 로 확장 (실제 블록은 거의 0)
ls -lh sparse.img; du -sh sparse.img # ls 는 100M, du 는 0 에 가까움
truncate -s 10M sparse.img           # 축소 — ⚠️ 잘린 부분은 소실
truncate -s 0 /tmp/multi.log         # 내용 비우기 (inode 유지)
truncate -s +5M sparse.img           # 상대 증가
ls -l sparse.img
rm -f sparse.img

# ▸ shred — 덮어쓴 뒤 삭제
dd if=/dev/urandom of=/tmp/secret.dat bs=1K count=64 status=none
shred -v -n 3 -z /tmp/secret.dat 2>&1 | tail -3   # 3회 덮어쓰고 마지막에 0 채움
shred -u -n 1 -z /tmp/secret.dat                  # 덮어쓴 뒤 파일까지 삭제
ls -l /tmp/secret.dat 2>&1 | head -1

# ▸ install — 복사 + 권한·소유자·디렉터리 한 번에
install -m 755 -o root -g root /tmp/sh/run.sh /usr/local/bin/labrun.sh
ls -l /usr/local/bin/labrun.sh
install -d -m 750 /srv/lab/{conf,data}    # 디렉터리 생성 + 권한
ls -ld /srv/lab/conf /srv/lab/data
install -m 644 /dev/null /tmp/newempty.txt   # 빈 파일을 지정 권한으로
rm -f /usr/local/bin/labrun.sh /tmp/newempty.txt; rm -rf /srv/lab

# ▸ rename (util-linux) — 일괄 이름 변경
mkdir -p /tmp/ren && cd /tmp/ren
touch a.txt b.txt c.txt
rename .txt .bak *.txt        # 첫 번째 일치 문자열만 교체
ls
rename '' 'old-' *.bak        # 접두어 붙이기
ls
cd /tmp && rm -rf /tmp/ren
```

- `truncate -s <크기>` : 파일 크기를 지정 값으로 (**s**ize) — `+N`/`-N` 상대 조정, `-r <참조파일>` 참조 크기, `-o` 블록 단위
  - 확장 시 **sparse(희소) 파일** 생성 → `ls -l`(논리 크기) 과 `du`(실제 블록) 가 달라짐
  - ⚠️ 축소하면 잘린 데이터는 복구 불가
- `shred` : 파일을 난수로 여러 번 덮어쓴 뒤 삭제 — `-n N` 덮어쓰기 횟수(기본 3), `-z` 마지막에 0 으로, `-u` 덮어쓴 뒤 **삭제**(**u**nlink), `-v` 진행 표시, `-f` 권한 강제 변경
  - ⚠️ **한계**: `ext4`(journal)·`xfs`·`btrfs` 같은 저널링/CoW 파일시스템, SSD 의 wear-leveling, RAID·스냅샷 환경에서는 **원래 블록이 덮어써진다는 보장이 없음**. 실무에서는 파일 단위 `shred` 대신 **디스크 전체 암호화(LUKS)** 또는 물리 파기가 정답
- `install [-m 모드] [-o 소유자] [-g 그룹] <원본> <대상>` : 복사 + 속성 지정을 한 번에 — `-d` 디렉터리 생성, `-D` 대상의 상위 디렉터리까지 생성, `-b` 기존 파일 백업. Makefile 의 `make install` 이 사용하는 명령
- `rename <바꿀문자열> <새문자열> <파일…>` : util-linux 판 — **정규식이 아니라 단순 문자열 치환**. (Perl 판 `rename 's/a/b/' *` 과 문법이 다름)

**검증**
```bash
truncate -s 50M /tmp/sp.img; ls -l /tmp/sp.img | awk '{print $5}'; du -sh /tmp/sp.img; rm -f /tmp/sp.img
install -m 700 /dev/null /tmp/inst.txt; stat -c '%a %n' /tmp/inst.txt; rm -f /tmp/inst.txt
```

```text
52428800
0	/tmp/sp.img
700 /tmp/inst.txt
```

> 📝 **시험 포인트**: `truncate -s 0` 은 로그 파일 비우기(`: >` 와 동일 효과). `shred` 는 "안전 삭제" 개념 문항으로 나오되, 저널링 FS 에서 보장되지 않는다는 한계까지 묻는 경우가 있다.

---

## 체크리스트

| 수행 항목 | 명령 | 확인 방법 | ☐ |
| --- | --- | --- | --- |
| 실습 트리·텍스트 파일 생성 | `mkdir -p …/{src,logs,docs}`, heredoc, `seq`, `printf` | `tree`, `wc -l` 로 9개 파일 | ☐ |
| 더미 대용량·오래된 파일 | `dd`, `fallocate -l`, `touch -d/-t`, `: >` | `ls -lh`, `ls -l --time-style=long-iso` | ☐ |
| ls 옵션 전 범위 | `ls -l -a -h -R -t -S -i -d -Z -lrt` | `ls -lS \| head -3` 크기순 | ☐ |
| cp / mv / rm / rmdir / mkdir -m | `cp -a -r -p -i -u`, `mkdir -m 750`, `rmdir -p` | `stat -c '%y %n'`, rmdir 실패 메시지 | ☐ |
| 하드·심볼릭 링크 차이 | `ln`, `ln -s`, `readlink -f` | `ls -li` inode·링크 수, 원본 이동 후 dangling | ☐ |
| stat·file·basename·dirname | `stat -c '%a %A %U %G %s'`, `file -b -i -L` | `4755 root root` (passwd) | ☐ |
| du·df·wc | `du -sh`, `df -hT`, `df -i`, `wc -l/-w/-c` | `df -hT /srv` 한 행 | ☐ |
| cat·tac·nl·head·tail -f | `cat -n -A`, `nl`, `head -c/-n -N`, `tail -f/-F` | 다른 터미널 `logger` 후 즉시 표시 | ☐ |
| split·cmp·diff·patch | `split -b/-l`, `cmp`, `diff -u`, `patch -R` | `cmp` 무출력, `grep -c citrus` 0→1→0 | ☐ |
| od·hexdump·strings | `od -c/-An -tx1`, `hexdump -C`, `strings -n 8` | 첫 행 문자 덤프 | ☐ |
| find 이름·유형·깊이 | `-name -iname -type -path -regex -maxdepth -mindepth -prune -xdev` | `find . -name "*.c" \| wc -l` = 5 | ☐ |
| find 크기·시간 | `-size +100M/-100k`, `-mtime +7`, `-mmin -60`, `-atime`, `-ctime`, `-newer` | `-printf '%s %p\n'` 로 대상 확인 | ☐ |
| find 소유자·권한·빈 파일 | `-user -group -nouser -nogroup -perm -4000 -perm /6000 -empty` | SetUID 목록에 `/usr/bin/passwd` | ☐ |
| find 동작 | `-exec {} \;` `-exec {} +` `-ok` `-delete` `-print0 \| xargs -0` `-printf` `-ls` | 삭제 전후 `ls` 대조 | ☐ |
| 기출 조합 3종 실행 | `.log -mtime +7 -delete` / `-size +100M -type f` / `-perm -4000 -type f` | `/tmp/suid-<날짜>.txt` 생성 | ☐ |
| locate·updatedb | `dnf install -y plocate`, `updatedb`, `locate -i -b -c -l -e` | 갱신 전 exit 1 → 갱신 후 경로 출력 | ☐ |
| which·whereis·type | `which -a`, `whereis -b -m`, `type -a -t`, `command -v` | `type -t cd` = builtin | ☐ |
| grep 전 옵션 | `-i -v -n -c -l -L -r -R -w -x -E -F -o -q -e -f -A -B -C --include --exclude-dir` | `grep -c` 2 vs `grep -ic` 3 | ☐ |
| 정규식 BRE/ERE | `^ $ . [] [^] [[:digit:]] * \+ \? \{n,m\} \( \) \|` ↔ ERE | `grep -oE '([0-9]{1,3}\.){3}…'` | ☐ |
| /var/log/secure 집계 | `grep -c 'Failed password'`, `grep -oE` + `sort \| uniq -c \| sort -rn` | `lastb` 와 교차 확인 | ☐ |
| cut | `-d -f -c -b --complement -s --output-delimiter` | `cut -d: -f1` 행 수 = `wc -l` | ☐ |
| awk 기본·심화 | `-F -v $0 $1 NF NR FNR BEGIN END printf` 조건·배열·`length/substr/split` | `$3>=1000 {print $1}` 출력 | ☐ |
| sed 조회·삭제·치환 | `-n p`, `Nd`, `s///g`, 주소 `$` `/pat/` `N~M`, `y///` | `sed '/^#/d;/^$/d'` 결과 행 수 | ☐ |
| sed -i 와 백업 | `-i`, `-i.bak`, `-e`, `a\ i\ c\` | `.bak` 생성 + `diff` | ☐ |
| sort | `-n -r -u -f -h -M -V -t -k -c -o` | `sort` vs `sort -n` 첫 3행 비교 | ☐ |
| uniq | `-c -d -D -u -i -f -s -w` | `sort \| uniq -c` 빈도표 | ☐ |
| tr | `'a-z' 'A-Z'`, `-d -s -c -cd` | 대문자 변환 결과 | ☐ |
| paste·join·comm | `paste -d -s`, `join -t -1 -2 -a -v -o`, `comm -12 -23 -13 -3` | 교집합·차집합 출력 | ☐ |
| expand·fold·fmt·column·nl | `expand -t`, `fold -w -s`, `fmt -w`, `column -t -s`, `nl -b -n -w -s` | CSV 가 열 맞춤 표로 | ☐ |
| xargs | `-n -I{} -0 -r -p -t -P -d` | `-print0 \| xargs -0` 로 공백 파일명 처리 | ☐ |
| tee·printf·wc | `tee`, `tee -a`, `printf '%-10s %5d\n'` | 화면 + 파일 동시 기록 | ☐ |
| 종합 파이프라인 3종 | 셸별 집계 / 시간대별 카운트 / `df` 80% 초과 | 각 출력 존재 | ☐ |
| gzip 계열 | `-k -d -9 -1 -l -c -t`, `zcat zgrep zless zdiff` | `gzip -l` 압축률, `zcat \| wc -l` 원본과 동일 | ☐ |
| bzip2·xz·압축률 비교 | `bzip2 -k -d -c`, `xz -k -d -l`, `time`, `ls -lh` | xz < bzip2 < gzip 크기 순 | ☐ |
| zip·unzip | `zip -r -e -j -9`, `unzip -l -t -p -o -d -q` | `unzip -l` 목록, 해제 결과 | ☐ |
| tar 생성·목록·추출 | `c t x`, `-v -f -z -j -J -C -p --wildcards --strip-components` | `tar tzf` 에 선행 `/` 없음, `/restore` 에 복원 | ☐ |
| tar 선행 `/` 제거 경고 확인 | `tar czvf home.tar.gz /home` | `Removing leading '/'` 메시지 | ☐ |
| tar 선택·보존 옵션 | `--exclude --exclude-from --totals -W -p --same-owner --keep-old-files` | 제외 대상 0건 | ☐ |
| tar 증분 (레벨 0 → 1) | `--listed-incremental=snap.snar` | `tar tzf` 로 lv1 에 변경분만 | ☐ |
| cpio·rpm2cpio | `find \| cpio -ov`, `cpio -idmv`, `-t`, `--no-absolute-filenames` | 추출 결과 파일 존재 | ☐ |
| 체크섬 산출·검증·변조 | `md5sum sha1sum sha256sum sha512sum cksum`, `-c --quiet --status` | 정상 `OK` / 변조 `FAILED` + exit 1 | ☐ |
| 리다이렉션 기본 | `> >> < 2> 2>&1 &> >& \|& << <<- <<<` | `out.txt`/`err.txt` 분리 확인 | ☐ |
| 순서 함정 재현 | `cmd > f 2>&1` vs `cmd 2>&1 > f` | A 2행 / B 1행 | ☐ |
| /dev/null · tee | `> /dev/null 2>&1`, `: > 파일`, `\| sudo tee` | inode 유지 + 크기 0 | ☐ |
| 명령 치환·리스트 연산자 | `$( )`, 백틱, `;` `&&` `\|\|` | `$NOW` 형식 검사 통과 | ☐ |
| { } vs ( ) 서브셸 | 변수·`cd` 유효범위, `$BASH_SUBSHELL` | `{}: changed` / `(): 원본` | ☐ |
| fd 조작·noclobber | `exec 3<>`, `>&3`, `exec 3>&-`, `set -o noclobber`, `>\|` | `cannot overwrite` 후 `>\|` 성공 | ☐ |
| 지역 vs 환경 변수 | `VAR=`, `export`, `export -n`, `env`, `set`, `printenv`, `unset` | 자식 셸에서 `T1` 만 상속 | ☐ |
| declare·readonly | `-i -r -x -a -A -p -f` | `-i` 산술 자동, `-r` 재할당 거부 | ☐ |
| 특수 변수 | `$? $$ $! $# $@ $* $0 $1 $_ $IFS $PPID` | `"$@"` 3개 / `"$*"` 1개 | ☐ |
| 매개변수 확장 | `${:-} ${:=} ${:?} ${:+} ${#} ${%} ${%%} ${#*/} ${##*/} ${/} ${//} ${::}` | `base=` `dir=` 결과 | ☐ |
| 산술 연산 | `$(( ))`, `(( ))`, `let`, `expr`, `16#FF` | `100/7`=14, awk 실수 14.2857 | ☐ |
| 인용 3종 | `'…'` `"…"` `\` | `printf '[%s]\n' $X` vs `"$X"` | ☐ |
| 글로빙·중괄호·shopt | `* ? [] [!]`, `{1..5}` `{a,b}`, `extglob nullglob globstar dotglob` | `/tmp/glob` 6개 디렉터리 | ☐ |
| set 옵션·read·디버깅 | `set -e -u -x -o pipefail`, `read -p -s -a -r -n -t`, `bash -n/-x` | pipefail exit 1, 문법 OK | ☐ |
| 스크립트 실행 방식 3종 | `./s` vs `bash s` vs `source s`/`. s` | `source` 만 변수 잔존 | ☐ |
| test / [ ] / [[ ]] | `-f -d -e -r -w -x -s -z -n -L`, `= != -eq -ne -gt -lt -ge -le`, `=~` | 각 조건 참·거짓 exit 코드 | ☐ |
| if / case | `if elif else fi`, `case … esac`, `;;` `*)` | `./cond.sh` 분기별 출력 | ☐ |
| 반복문 | `for in`, `for (( ))`, `while read`, `until`, `break`, `continue` | `./loop.sh` 9개 블록 출력 | ☐ |
| 함수·종료 코드 | `name() { local …; return N; }`, `$(함수)`, `exit` | 0 / 2 / 127 코드 확인 | ☐ |
| getopts | `while getopts ":d:k:vh"`, `$OPTARG`, `shift $((OPTIND-1))` | 옵션·남은 인자 분리 | ☐ |
| trap·date·logger | `trap … EXIT INT TERM`, `date +%F_%H%M%S`, `logger -t -p` | 임시 파일 미잔존, journal 기록 | ☐ |
| **`/usr/local/bin/backup.sh` 작성·실행** | `set -euo pipefail`, `tar czf`, `sha256sum`, `find -mtime +7 -delete`, `logger` | `tar tzvf` 내용 · `sha256sum -c` OK · `/var/log/backup.log` | ☐ |
| backup.sh 실패 경로·추적 | 잘못된 `-d`, `bash -x` | exit 1 + `logger` err 기록 | ☐ |
| **`/usr/local/bin/sysreport.sh` 작성·실행** | `top -b -n 1`, `vmstat 1 3`, `df -hT`, `free -h`, `ss -tuln` | `/var/log/sysreport-<날짜>.txt` 섹션 6개 | ☐ |
| vi 3모드·입력 진입 | `i a o O I A R s S cw C`, `ESC` | `-- INSERT --` 표시 | ☐ |
| vi 저장·종료 | `:w :q :wq :q! :x ZZ :w! :e!` | 저장 후 `diff` 로 반영 확인 | ☐ |
| vi 편집·이동 | `dd 3dd yy p P u Ctrl+r x D C`, `h j k l w b 0 $ gg G :N Ctrl+f/b` | 편집 결과 diff | ☐ |
| vi 검색·치환 | `/pat n N ?pat`, `:s/a/b/ :%s/a/b/g :%s/a/b/gc :g/pat/d` | 주석 행 0개 | ☐ |
| vi 설정·삽입·외부 명령 | `:set nu ai ts=4 hlsearch`, `:r 파일`, `:r !명령`, `:!명령`, `v V Ctrl+v`, `~/.vimrc` | `~/.vimrc` 생성 | ☐ |
| vi 실습 검증 | `/etc/skel/.bashrc` 사본 편집 후 `diff` | 원본 무손상 + 변경분 확인 | ☐ |
| 특수 파일 표 | `/dev/null zero random urandom full tty stdin` | `/dev/full` 쓰기 exit 1 | ☐ |
| mkfifo 통신 | `mkfifo`, `cat < pipe &`, `echo > pipe` | 첫 문자 `p`, 메시지 수신 | ☐ |
| 소켓·mknod·유형 판별 | `find /run -type s`, `mknod c 1 3`, `file`, `stat -c %F` | `- d l b c p s` 7종 대조 | ☐ |
| truncate·shred·install·rename | `truncate -s`, `shred -u -n -z`, `install -m -o -g -d`, `rename` | sparse `ls` vs `du`, 권한 700 | ☐ |

---

## 기출 연결

| 기출 (문제집·회차·번호 또는 주제) | 이 파트의 단계 |
| --- | --- |
| EXAM-PRACTICAL r01-1 (`.log` 이면서 7일 경과 파일 삭제 `find`) | 3-5 ① `-name "*.log" -mtime +7 -delete` |
| EXAM-PRACTICAL r04-1 (`/home` 에서 100MB 초과 일반 파일) · EXAM-WRITTEN-FULL r06-27 (500MB 초과 파일 목록) | 3-2 `-size`, 3-5 ② |
| EXAM-PRACTICAL r02-1·r05-3 · EXAM-WRITTEN-FULL r04-65·r05-65·r06-62·r09-31 (SetUID 일반 파일 검색) | 3-3 `-perm -4000`, 3-5 ③ |
| EXAM-WRITTEN-FULL r03-27 (`-mtime +7 -size +10M -exec rm {} \;` 보기 비교) | 3-4 `-exec … \;` vs `+`, `-ok`, `-delete` |
| EXAM-WRITTEN-FULL r10-27 (`find /home -user kim -size +100M -name "*.log" -exec rm {} \;` 해석) | 3-3 소유자 조건, 3-4 동작 |
| EXAM-WRITTEN-FULL r10-11 (`find / -name "*.conf" 2>/dev/null \| wc -l` 해석) | 3-1 `-name`, 6-3 `2>/dev/null`, 4-12 `wc -l` |
| EXAM-PRACTICAL r03-1·r05-1 (`/var/log/secure` 의 `Failed password` **행 개수**) | 3-8 `grep -c`, 3-10 실전 집계 |
| EXAM-WRITTEN r07-7 (`grep` 대소문자 무시 옵션 `-i`) | 3-8 grep 옵션 총정리 |
| EXAM-WRITTEN r06-6 (`cat access.log \| grep 404 \| wc -l` 결과 해석) | 3-8 `-c` 의미, 4-12 `wc -l` |
| EXAM-WRITTEN r05-6 (`error` 로 **시작**하는 행 → `^error`) | 3-9 앵커 `^` `$` |
| EXAM-WRITTEN r06-7 (`cat` 또는 `dog` → `grep -E 'cat\|dog'`) | 3-9 ERE 대체 `\|`, BRE `\\\|` |
| EXAM-WRITTEN-FULL r08-5 (`grep root /etc/hosts; echo $?` 가 1 인 이유) | 3-8 grep 종료 코드, 7-3 `$?` |
| EXAM-PRACTICAL r03-2·r06-5 · EXAM-WRITTEN r05-8 (`awk -F:` UID ≥ 1000 사용자명) | 4-2 필드·`-F`, 4-3 조건 `$3>=1000` |
| EXAM-WRITTEN-FULL r02-11 (`cut -d: -f7 \| sort \| uniq -c \| sort -nr` 결과) | 4-1 `cut`, 4-6 `sort`, 4-7 `uniq -c`, 4-13 ① |
| EXAM-PRACTICAL r06-1 (`httpd.conf` 의 `Listen 80` → `8080` in-place 치환) | 4-5 `sed -i` |
| EXAM-PRACTICAL r03-7 (`file.txt` 의 `apple` → `banana` 원본 저장) | 4-5 `sed -i 's///g'` |
| EXAM-WRITTEN r05-7 (`sed 's/apple/orange/g'`), r07-8 (`sed '3d'`) | 4-4 치환 플래그 `g`, 주소 삭제 `Nd` |
| EXAM-PRACTICAL r01-13 (`tar` 로 `/home` → `home.tar.gz`, `c z v f` 의미 서술) · EXAM-WRITTEN-FULL r01-60 (`z` 역할) | 5-4 tar 생성 + 선행 `/` 제거 경고 |
| EXAM-PRACTICAL r02-3 (`backup.tar.gz` 를 `/restore` 로 추출) · EXAM-WRITTEN-FULL r04-44 (`tar xzvf … -C /opt` 해석) · r07-29 | 5-4 `x` + `-C` |
| EXAM-WRITTEN-FULL r02-61 (`/home/data` → `data.tar.xz` 생성 = `tar cvJf`) | 5-2 xz, 5-4 옵션 문자 매핑 |
| EXAM-WRITTEN-FULL r03-41·r08-41 (아카이브를 **풀지 않고** 목록만 확인 = `t`) | 5-4 `tar tvf` / `tar tzf` |
| EXAM-WRITTEN-FULL r07-23 (소스 설치 절차 첫 단계 `tar zxvf app-1.0.tar.gz`) | 5-4 추출 (컴파일 절차는 [[02-package-management\|Part 02]]) |
| EXAM-PRACTICAL r03-10 · EXAM-WRITTEN-FULL r05-64 (증분 백업 스냅샷 옵션) | 5-6 `--listed-incremental=snap.snar` |
| EXAM-WRITTEN-FULL r10-44 (백업 도구 비교 — `tar` 에 0~9 레벨 체계 없음) | 5-6 증분·차등 개념 ([[12-backup-recovery-review\|Part 12]] 심화) |
| EXAM-PRACTICAL r04-4 (`image.iso` 의 SHA-256 해시) · EXAM-WRITTEN-FULL r09-36 (`sha256sum -c` → `OK`) · r06-96 (충돌 발견 해시 대체) | 5-8 체크섬 산출·검증·변조 재현 |
| EXAM-WRITTEN-FULL r09-63 (`tar czf - /etc \| gpg -c -o etc.tar.gz.gpg`) | 5-4 아카이브를 stdout(`-`)으로, 6-1 파이프 (gpg 는 [[10-security-firewall-selinux\|Part 10]]) |
| EXAM-WRITTEN r03-4·r10-6·r11-9 (표준오류만 파일로 = `2> file`) | 6-1 fd 표와 기본 기호 |
| EXAM-WRITTEN r07-6 (내용 유지하며 덧붙이기 = `>>`) | 6-1 `>` vs `>>` |
| EXAM-WRITTEN r06-5 (`wc -l < list.txt` 의 `<` 역할) | 6-1 표준입력 대체, 4-12 `wc` |
| EXAM-WRITTEN-FULL r02-9 (`ls /etc/passwd /nofile > out.txt 2>&1` 해석) | 6-1 `2>&1`, 6-2 순서 함정 |
| EXAM-WRITTEN-FULL r03-9 (`exec ls` 로 명령 실행 시 동작) | 6-6 `exec` 는 현재 셸을 대체 |
| EXAM-WRITTEN-FULL r03-35 (`$?` `$#` `$@` `$$` 짝짓기 — `$$` 를 백그라운드 PID 로 쓴 보기가 오답) | 7-3 특수 변수 표 |
| EXAM-WRITTEN-FULL r03-34 (스크립트 조건이 참이 되어 `usage` 출력되는 경우) | 7-3 `$#`, 8-2 `[ ]`, 8-6 `usage`/`getopts` |
| EXAM-WRITTEN r05-4·r11-7 · EXAM-WRITTEN-FULL r05-11·r07-36 (자식 프로세스 상속 = `export`) | 7-1 지역 vs 환경 변수 상속 실습 |
| EXAM-WRITTEN-FULL r08-6 · EXAM-WRITTEN r06-4 (`env` 와 `set` 차이 / 환경변수만 출력) | 7-1 `env` `printenv` `set` |
| EXAM-WRITTEN r08-6·r14-6 (`PATH` 환경변수) · EXAM-WRITTEN-FULL r09-8 (`PATH` 에 `.` 포함 위험) | 7-1 주요 환경 변수, 8-1 `./` 로 실행하는 이유 |
| EXAM-WRITTEN r02-6·r05-10·r10-9·r13-10 · EXAM-WRITTEN-FULL r04-13 (`export DISPLAY=…`) | 7-1 `export` (X 윈도는 ※ 미실행) |
| EXAM-PRACTICAL r01-9 (셸 스크립트 첫 줄 인터프리터 표기 = `#!/bin/bash`) | 8-1 shebang·실행 권한·실행 방식 3종 |
| EXAM-WRITTEN-FULL r03-19 (셸 설명 중 틀린 것 — dash/csh 계열) · r05-10·r07-22 (초기화 파일 순서) | 8-1 `source` 와 초기화 파일 ([[01-vm-setup-and-inspection\|Part 01]] 7절) |
| EXAM-PRACTICAL r04-2 · EXAM-WRITTEN-FULL r01-55·r02-63·r04-61·r09-55 (`dd if=/dev/sda of=mbr.img bs=512 count=1`) | 1-2 `dd` 옵션 (MBR 백업 실습은 [[12-backup-recovery-review\|Part 12]]) |
| EXAM-WRITTEN-FULL r06-34·r07-41 (`dd if=/dev/zero of=/swapfile bs=1M count=2048`) | 10-1 `/dev/zero` (스왑 파일 생성은 [[05-disk-lvm-raid-swap-quota\|Part 05]]) |
| EXAM-WRITTEN-FULL r04-9·r08-3 (`ls -l` 출력의 파일 유형 해석), r04-8 보기 "파이프(FIFO) 파일" | 10-2 `mkfifo`, 10-3 유형 7종 대조표 |
| EXAM-WRITTEN-FULL r08-34 (`jobs` 에서 `vim memo.txt` 를 포그라운드로) | 9-1 vi 실행 (잡 제어는 [[06-process-scheduling-diagnosis\|Part 06]]) |
| EXAM-PRACTICAL r01-11·r03-11·r04-11 (`/etc/crontab` 필드 해석 — `backup.sh` 등록) | 8-8 `backup.sh` (cron 등록은 [[06-process-scheduling-diagnosis\|Part 06]] 7절) |
| EXAM-WRITTEN-FULL r06-9 (`!tar` 로 직전 tar 명령 재실행) | 5-4 tar (history 확장은 [[01-vm-setup-and-inspection\|Part 01]] 7-4) |

---

## 이전 / 다음

[[03-user-group-permission]] ← · → [[05-disk-lvm-raid-swap-quota]]

- [[README]] — LAB 허브(시나리오·자원 명세)
- 참조 이론: [[../THEORY/system-structure]] (셸·메타문자·리다이렉션·파일 유형·inode), [[../THEORY/linux-basics]] (편집기·기본 명령)
- 이 파트 산출물의 후속 사용: [[06-process-scheduling-diagnosis]] (cron·at·timer 로 `backup.sh`·`sysreport.sh` 등록), [[07-boot-systemd-log]] (`lab-monitor.service` 에서 `sysreport.sh` 호출), [[12-backup-recovery-review]] (`backup.sh` 최종판·증분 백업 심화)
