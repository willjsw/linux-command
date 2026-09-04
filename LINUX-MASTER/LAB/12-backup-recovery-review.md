---
title: LAB 12 — 백업·복구 검증과 전 범위 총정리
type: exam-lab
part: 12
tags:
  - exam/linux-master
  - exam/lab
  - linux/filesystem
  - linux/storage
  - linux/shell
  - task/configure
  - task/verify
related: ["[[README]]", "[[11-container-virtualization]]", "[[../THEORY/system-security]]", "[[../THEORY/disk-device]]", "[[../../FILE-TEXT/tar]]", "[[../../DISK-STORAGE/dd]]", "[[../../FILE-TEXT/sha256sum]]"]
distro: Rocky Linux 9 (aarch64, UTM)
updated: 2026-09-03
---

# LAB 12 — 백업·복구 검증과 전 범위 총정리

- 백업 전략(전체·증분·차등·스냅샷·3-2-1·RPO/RTO) 수립 → 대상 목록 정의 → RAID 볼륨 `/srv/raid/backup` 으로 백업
- `tar --listed-incremental`·`rsync --link-dest`·`dd`·`dump/restore`·`xfsdump`·LVM 스냅샷·Docker 볼륨 — 도구별로 **지우고 되살려서** 복구 증명
- Part 04 `backup.sh` 를 최종판(주간 전체 + 일일 증분, flock·logger·mail·보존·해시)으로 개정, 종료·재부팅 명령 총정리와 부팅 후 자동 복원 검증
- 두 번째 파트: 12 파트 전 범위 종합 점검표(약 110행) + 시험 직전 명령어 120선 + X 윈도 필기 요약

> **이 파트의 시나리오**: Part 01~11 로 `srv01.lab.local` 구축이 끝났다. 데이터(`/data`, `/srv/share`, `/srv/nfs/data`)·설정(`/etc`)·컨테이너(`redis-data`)·디스크 메타데이터가 모두 한 VM 에 있다. 디스크 장애·실수 삭제·재설치를 가정해 RAID 1 볼륨 `/srv/raid` 와 macOS 호스트(오프사이트)에 백업하고, 실제로 삭제한 뒤 복원해 백업이 "복구 가능한 백업" 임을 증명한다. 마지막으로 12 파트 전체를 되짚는 점검표로 시험 직전 상태를 확인한다.

- 선행 자원: `/data`(ext4, `/dev/vdb1`)·`/srv/raid`(xfs, `/dev/md0`)·`vg_lab/lv_share`(`/srv/share`) 는 Part 05, `backup.sh`·`/data/proj` 는 Part 04, cron `03:30` 은 Part 06, Docker `lab-redis`·`redis-data` 는 Part 11 에서 생성됨
- 모든 백업 산출물은 `/srv/raid/backup/` 하위. ⚠️ `of=`·`rm -rf`·`mkfs` 대상은 매번 `lsblk`/`findmnt` 로 재확인

---

## 1. 백업 전략 수립

### 1-1. 백업 방식·핵심 개념 정리

> **상황**: 무엇을 얼마나 자주 어디에 남길지 정하지 않은 백업은 복구 시점에 빠진 조각이 드러난다. 먼저 방식·복원 세트·목표 지표를 표로 고정한다.

```bash
# 현재 백업 대상지 상태 확인 (Part 05 RAID 1)
findmnt /srv/raid
df -hT /srv/raid
cat /proc/mdstat
mkdir -p /srv/raid/backup /srv/raid/mirror /srv/raid/snap
ls -ld /srv/raid/*
```

- `findmnt <경로>` : 마운트 소스·FS 유형·옵션 확인 (**find** **m**ou**nt**)
- `df -hT` : 사용량 + 파일시스템 유형 (**h**uman, **T**ype)
- `/proc/mdstat` : md 배열 상태 — `[UU]` = 두 멤버 정상

| 방식 | 저장 대상 | 복원 필요 세트 | 용량 | 백업 속도 | 복원 속도 |
| --- | --- | --- | --- | --- | --- |
| 전체 (Full) | 전체 데이터 | 전체 1개 | 최대 | 최저 | 최고 (1회) |
| 증분 (Incremental) | **직전 백업**(전체 또는 증분) 이후 변경분 | 전체 + **모든 증분 순서대로** | 최소 | 최고 | 최저 (n개) |
| 차등 (Differential) | **마지막 전체** 이후 변경분 | 전체 + **최신 차등 1개** | 중간(누적 증가) | 중간 | 중간 (2개) |
| 스냅샷 (Snapshot) | 특정 시점 블록 상태 (CoW) | 스냅샷 자체 | 변경분만 | 즉시 | 즉시 |
| 미러 (rsync --delete) | 원본과 동일 트리 | 미러 1개 | 원본과 동일 | 변경분 전송 | 최고 |

- 기출 계산 유형: 일 전체 → 월·화·수 차등 → 목 장애 = **전체 + 수요일 차등 (2개)**. 증분이면 **전체 + 월·화·수 전부 (4개)**
- `dump` 레벨 규칙: 레벨 n 은 **자신보다 낮은 레벨의 가장 최근 백업 이후** 변경분. 일0·월3·화2·수5 → 수요일까지 복원 = 0 + 2(화, 3보다 낮으므로 월 3 은 불필요) + 5(수)
- **3-2-1 원칙**: 복사본 **3**개 · 서로 다른 매체 **2**종 · 그중 **1**개는 오프사이트 → 이 LAB: 원본(vdb/vde) + RAID(`/srv/raid`) + macOS 호스트(`~/lab-backup`)
- **RPO**(Recovery Point Objective): 허용 가능한 데이터 손실 시간 = 백업 주기 (매일 03:30 → 최대 24h). **RTO**(Recovery Time Objective): 복구 완료까지 허용 시간 = 복원 세트 수·매체 속도에 좌우
- **RAID 는 백업이 아님**: 미러는 `rm -rf` 도 양쪽에 즉시 복제. 디스크 고장(가용성)에만 대응, 실수·랜섬웨어·논리 오류에는 무력 → 그래서 `/srv/raid` 위에 **시점 백업**을 따로 둠

**검증**

```bash
df -hT /srv/raid | tail -1
grep -c '\[UU\]' /proc/mdstat
```

```text
/dev/md0  xfs  5.0G  ...  /srv/raid
1
```

> 📝 **시험 포인트**: "마지막 백업 이후 변경분" = 증분, "마지막 **전체** 백업 이후 변경분" = 차등. 차등은 복원 2개·증분은 전부 필요. RAID 1 미러 ≠ 백업(시점 복원 불가).

### 1-2. 도구·매체 표와 대상 목록 정의

> **상황**: 대상마다 알맞은 도구가 다르다. 파일 단위(tar/rsync)·파일시스템 단위(dump/xfsdump)·블록 단위(dd)·메타데이터(sfdisk/vgcfgbackup/mdadm) 로 나눠 목록을 확정한다.

```bash
# 대상 존재 확인 (없는 항목은 해당 파트 미완료)
ls -ld /data /srv/share /srv/nfs/data /etc /home /var/spool/cron /var/named /etc/samba
docker volume ls | grep redis-data
ls /etc/lvm/backup /etc/mdadm.conf
rpm -qa | wc -l
```

| 도구 | 단위 | 증분 | 특징 | 이 LAB 사용처 |
| --- | --- | --- | --- | --- |
| `tar` | 파일 | `--listed-incremental`(`-g`) | 표준 아카이브, 압축 `-z/-j/-J`, ACL·SELinux 보존 옵션 | `/data`, `/srv/share`, `/etc` |
| `cpio` | 파일 | 없음 (find 로 선택) | 표준입력 파일 목록, 손상 아카이브 부분 복원 강함, `rpm2cpio` | 참고 실습 |
| `dd` | 블록 | 없음 | FS 무관 저수준 복제, MBR/GPT·파티션 이미지, ⚠️ `of=` 실수 = 파괴 | 파티션 테이블, `vdb1` 이미지 |
| `dump`/`restore` | FS (ext2/3/4) | 레벨 0~9, `/etc/dumpdates` | fstab 5번 필드 연동, 대화식 복원 | `/dev/vdb1` |
| `xfsdump`/`xfsrestore` | FS (xfs) | 레벨 0~9, 인벤토리 | xfs 전용, 세션 라벨·미디어 라벨 | `/srv/share` |
| `rsync` | 파일 | 델타 전송 | 변경 블록만, `--delete` 미러, `--link-dest` 하드링크 스냅샷, 원격 ssh | 미러, 오프사이트 |
| LVM 스냅샷 | 블록 (CoW) | — | 서비스 무중단 일관 백업 원본 | `lv_share` |
| `mdadm` 미러 | 블록 | — | **가용성** 수단, 시점 복원 불가 | 대상지 보호 |
| `docker save`/볼륨 tar | 이미지·볼륨 | — | 이미지는 레지스트리, 볼륨은 컨테이너 경유 tar | `redis-data` |

| 대상 | 내용 | 도구 | 산출물 (`/srv/raid/backup/`) |
| --- | --- | --- | --- |
| `/data` | 프로젝트 데이터 (ext4, 쿼터) | tar 증분 / dump / dd | `data-full.tar.gz`, `data-inc1.tar.gz`, `data.dump0`, `vdb1.img.gz` |
| `/srv/share` | Samba 공유 (xfs, LVM) | tar(스냅샷) / rsync / xfsdump | `share-snap-*.tar.gz`, `mirror/share`, `share.xfsdump` |
| `/srv/nfs/data` | NFS 내보내기 | rsync | `mirror/nfs-data` |
| `/etc` | 시스템 설정 전체 | tar (+gpg) | `etc-<날짜>.tar.gz` |
| `/home` | 사용자 홈 | tar | `home-<날짜>.tar.gz` |
| `/var/spool/cron` | 사용자 crontab | tar + `crontab -l` | `cron-<user>` |
| `/var/named`, `/etc/samba` | 존 파일·smb.conf | `/etc` 포함 + rsync | `etc-*.tar.gz`, `mirror/named` |
| `redis-data` | Docker 볼륨 | 컨테이너 경유 tar | `redis-data.tar.gz` |
| LVM 메타데이터 | VG 구성 | `vgcfgbackup` | `/etc/lvm/backup/vg_lab` 복사 |
| 파티션 테이블 | vda GPT, vdb | `dd`, `sfdisk -d`, `sgdisk --backup` | `vda-pt.bin`, `vdb-sfdisk.txt`, `vda.gpt` |
| RAID 구성 | md0 | `mdadm --detail --scan` | `mdadm.conf.bak` |
| 패키지 목록 | 설치 RPM | `rpm -qa` | `pkglist.txt`, `dnf-history.txt` |

**검증**

```bash
for d in /data /srv/share /srv/nfs/data /etc /home /var/spool/cron /var/named /etc/samba; do
  printf '%-18s %s\n' "$d" "$(du -sh $d 2>/dev/null | cut -f1)"
done
```

```text
/data              ...M
/srv/share         ...M
/etc               ...M
...
```

> 📝 **시험 포인트**: "파일시스템 단위 + 레벨 0~9 + fstab dump 필드" = `dump`. "블록 단위·MBR" = `dd`. "표준입력 목록·부분 복원" = `cpio`. "원격 동기화·변경분만" = `rsync`.

---

## 2. tar 전체·증분·차등 백업과 복구 리허설

### 2-1. 테스트 데이터 준비와 레벨 0 (전체) 백업

> **상황**: `/data/proj`(Part 04) 에 압축 가능한 텍스트와 압축 불가능한 바이너리를 섞어 두고, 스냅샷 파일(`.snar`) 을 지정한 첫 백업 = 레벨 0 을 만든다.

```bash
# 테스트 데이터 (Part 04 의 /data/proj 가 없으면 생성)
mkdir -p /data/proj/src /data/proj/docs
for i in $(seq 1 30); do seq 1 20000 > /data/proj/src/file$i.txt; done
dd if=/dev/urandom of=/data/proj/blob.bin bs=1M count=20 status=none
echo "share-file $(date)" > /srv/share/readme.txt
ls -Z /data/proj | head -3                      # 복원 후 대조용 SELinux 컨텍스트
sha256sum /data/proj/src/*.txt /data/proj/blob.bin > /srv/raid/backup/data-before.sha256

# 레벨 0 — snar 파일이 없으므로 전체 백업
cd /srv/raid/backup
time tar --create --gzip --verbose \
    --file=/srv/raid/backup/data-full.tar.gz \
    --listed-incremental=/srv/raid/backup/data.snar \
    /data /srv/share | tail -3
cp /srv/raid/backup/data.snar /srv/raid/backup/data-level0.snar   # 차등용 원본 보존 (2-3)
```

- `--create` (`-c`) : 아카이브 생성 (**c**reate)
- `--gzip` (`-z`) : gzip 압축 (**z**)
- `--verbose` (`-v`) : 처리 파일명 출력 (**v**erbose)
- `--file=` (`-f`) : 아카이브 파일 지정 (**f**ile) — 단축 옵션 묶음에서는 **마지막**에 위치
- `--listed-incremental=<snar>` (`-g`) : 스냅샷 파일 지정 (**g** = listed-incremental). 파일이 **없으면 레벨 0**, 있으면 그 시점 이후 변경분. 실행 후 snar 는 현재 상태로 갱신됨
- `time` : 실행 시간 측정 → 2-5 압축 비교용
- `tar` 가 앞의 `/` 를 제거 → 아카이브 내부 경로는 `data/proj/...` (`Removing leading '/'` 메시지)

**검증**

```bash
ls -lh /srv/raid/backup/data-full.tar.gz /srv/raid/backup/data.snar
tar --list --gzip --file=/srv/raid/backup/data-full.tar.gz | grep -c '^data/proj/src/'
file /srv/raid/backup/data.snar
```

```text
-rw-r--r-- 1 root root  ...M ... data-full.tar.gz
-rw-r--r-- 1 root root  ...K ... data.snar
30
data.snar: GNU tar incremental snapshot data, version 2
```

> 📝 **시험 포인트**: 실기 빈출 빈칸 — `tar ______ snap.snar -czf backup.tar.gz /home` → `--listed-incremental=` 또는 `-g`. `cvzf` 각 글자 의미(create·verbose·gzip·file) 서술형.

### 2-2. 변경 후 레벨 1 (증분) 백업과 변경분 검증

> **상황**: 파일을 추가·수정·삭제한 뒤 **같은 snar** 로 다시 백업하면 증분이 된다. 레벨 1 아카이브에 변경분만 담겼는지 목록으로 확인한다.

```bash
echo "new file" > /data/proj/src/file31.txt          # 추가
echo "appended" >> /data/proj/src/file1.txt          # 수정
rm -f /data/proj/src/file30.txt                      # 삭제
touch /data/proj/docs/spec.md

tar --create --gzip --verbose \
    --file=/srv/raid/backup/data-inc1.tar.gz \
    --listed-incremental=/srv/raid/backup/data.snar \
    /data /srv/share
```

- 같은 `data.snar` 재사용 → 직전 실행(레벨 0) 이후 변경분만 → **증분**. 실행 후 snar 는 다시 갱신되므로 다음 실행은 레벨 2
- 삭제된 파일은 아카이브에 데이터로 들어가지 않지만, **디렉터리 항목(dumpdir)** 에 "현재 존재하는 파일 목록" 이 기록되어 복원 시 삭제가 반영됨

**검증**

```bash
tar tvzf /srv/raid/backup/data-inc1.tar.gz
tar tvzf /srv/raid/backup/data-inc1.tar.gz | grep -c 'file[0-9]*\.txt'
ls -lh /srv/raid/backup/data-*.tar.gz
```

```text
drwxr-xr-x root/root  ... data/
drwxr-xr-x root/root  ... data/proj/
drwxr-xr-x root/root  ... data/proj/src/
-rw-r--r-- root/root  ... data/proj/src/file1.txt
-rw-r--r-- root/root  ... data/proj/src/file31.txt
-rw-r--r-- root/root  ... data/proj/docs/spec.md
...
2
-rw-r--r-- ... ...M ... data-full.tar.gz
-rw-r--r-- ... ...K ... data-inc1.tar.gz
```

- 디렉터리는 항상 목록에 나타나지만(메타데이터), 파일은 변경된 `file1.txt`·`file31.txt` 만 포함 → 크기가 KB 단위
- `tvzf` : **t**able of contents · **v**erbose · g**z**ip · **f**ile — 풀지 않고 목록만

> 📝 **시험 포인트**: `tar tzf` = 압축 해제 없이 내용 목록 확인. 증분은 "직전 백업 이후", 같은 snar 를 계속 쓰면 레벨이 1씩 증가.

### 2-3. 차등 백업 — 레벨 0 snar 복사본 사용

> **상황**: 차등은 항상 "마지막 전체 이후" 기준이어야 하므로, 레벨 0 직후 보존한 snar 를 매번 **복사해서** 쓴다(원본 snar 는 갱신되지 않게 유지).

```bash
echo "diff test" > /data/proj/src/file32.txt

# 차등: 레벨0 snar 의 복사본으로 실행 → 기준점이 항상 레벨0
cp /srv/raid/backup/data-level0.snar /srv/raid/backup/data-diff.snar
tar -czvf /srv/raid/backup/data-diff1.tar.gz \
    -g /srv/raid/backup/data-diff.snar /data /srv/share

# 다음 날 차등도 동일 — 다시 level0 snar 를 복사
cp /srv/raid/backup/data-level0.snar /srv/raid/backup/data-diff.snar
echo "diff test 2" > /data/proj/src/file33.txt
tar -czvf /srv/raid/backup/data-diff2.tar.gz -g /srv/raid/backup/data-diff.snar /data /srv/share
```

- `-g` : `--listed-incremental` 단축형
- `data-level0.snar` 원본은 절대 `-g` 에 직접 넣지 않음 → 넣으면 갱신되어 증분으로 변질
- 증분 체인: `data.snar`(계속 갱신) / 차등 체인: `data-level0.snar`(고정) → 복사 → 사용

**검증**

```bash
tar tzf /srv/raid/backup/data-diff2.tar.gz | grep 'file3[0-9]'
cmp /srv/raid/backup/data-level0.snar /srv/raid/backup/data-diff.snar; echo "cmp=$?"
```

```text
data/proj/src/file31.txt
data/proj/src/file32.txt
data/proj/src/file33.txt
cmp=1
```

- 차등 2회차에 `file31`·`file32`·`file33` 모두 포함(레벨 0 이후 누적) — 증분 `data-inc1` 에는 `file31` 만 있었던 것과 대비
- `cmp=1` : 사용된 snar 는 갱신됐지만 원본 level0 snar 는 그대로

> 📝 **시험 포인트**: 차등은 복원 세트가 "전체 + 최신 차등 1개" — 위 `data-full` + `data-diff2` 만으로 복원 가능. tar 자체에는 "차등 옵션" 이 없고 snar 운용 방식으로 구현.

### 2-4. 보존·선택 옵션 — exclude·권한·SELinux·시각·경로

> **상황**: 운영 백업은 제외 목록, 소유권·ACL·SELinux 컨텍스트 보존, 특정 시각 이후 파일만, 복원 경로 조정이 필수다. 옵션별로 한 번씩 실행해 결과를 눈으로 확인한다.

```bash
cd /srv/raid/backup
cat > exclude.lst <<'EOF'
*.bin
*/docs/*
EOF

# 제외 — 패턴 직접 / 파일에서
tar -czf data-excl.tar.gz --exclude='*.bin' /data/proj
tar -czf data-excl2.tar.gz --exclude-from=exclude.lst /data/proj

# 권한·소유자·ACL·SELinux·xattr 보존 생성
setfacl -m u:dev1:r /data/proj/src/file1.txt               # ACL 하나 부여 (Part 03 dev1)
tar --acls --selinux --xattrs -czpf data-attrs.tar.gz /data/proj

# 특정 시각 이후 변경 파일만
tar -czvf data-newer.tar.gz --newer-mtime='1 hour ago' /data/proj | head -5

# 통계 출력 · 무결성 검증(-W 는 비압축 아카이브만)
tar -cf data-plain.tar --totals /data/proj
tar -cWvf data-verify.tar /data/proj 2>&1 | grep -c '^Verify'

# 복원 경로 조정: -C 대상 디렉터리, --strip-components 상위 경로 제거, --wildcards 패턴
mkdir -p /tmp/restore-attr /tmp/restore-strip
tar --acls --selinux --xattrs -xzpf data-attrs.tar.gz -C /tmp/restore-attr
tar -xzf data-attrs.tar.gz -C /tmp/restore-strip --strip-components=2 --wildcards 'data/proj/src/file1*.txt'
```

- `--exclude=<패턴>` : 패턴 일치 파일 제외 (셸 글로브, 따옴표 필수)
- `--exclude-from=<파일>` (`-X`) : 파일의 각 줄을 제외 패턴으로
- `-p` (`--preserve-permissions`) : 추출 시 umask 무시하고 아카이브 권한 그대로 (**p**ermissions). root 는 기본 적용
- `--same-owner` : 소유자 복원 (root 추출 시 기본값, 일반 사용자는 불가)
- `--acls` / `--selinux` / `--xattrs` : POSIX **ACL** · **SELinux** 컨텍스트 · 확장 속성(**x**tended **attr**ibutes) 저장·복원 — 생성과 추출 **양쪽** 에 지정
- `--newer-mtime=<날짜>` (`-N`) : 지정 시각 이후 **수정(mtime)** 된 파일만 (GNU date 형식: `'2026-09-01'`, `'1 hour ago'`)
- `--totals` : 종료 시 총 기록 바이트 출력
- `-W` (`--verify`) : 기록 직후 아카이브를 다시 읽어 원본과 대조 (**W** = verify). 압축(`-z` 등)과 **병용 불가**
- `-C <디렉터리>` : 작업 디렉터리 변경 후 추출/생성 (**C**hange dir)
- `--strip-components=N` : 아카이브 경로 앞 N 단계 제거 (`data/proj/src/file1.txt` → N=2 → `src/file1.txt`)
- `--wildcards` : 추출 대상 이름에 `*`, `?` 패턴 허용

**검증**

```bash
tar tzf /srv/raid/backup/data-excl.tar.gz | grep -c '\.bin$'            # 0
tar tzf /srv/raid/backup/data-excl2.tar.gz | grep -c 'docs/'             # 0
getfacl /tmp/restore-attr/data/proj/src/file1.txt | grep dev1
ls -Z /data/proj/src/file1.txt /tmp/restore-attr/data/proj/src/file1.txt
ls -R /tmp/restore-strip | head
tar -tvf /srv/raid/backup/data-plain.tar --totals >/dev/null
```

```text
0
0
user:dev1:r--
unconfined_u:object_r:...  /data/proj/src/file1.txt
unconfined_u:object_r:...  /tmp/restore-attr/data/proj/src/file1.txt
/tmp/restore-strip:
src
/tmp/restore-strip/src:
file1.txt
Total bytes read: ...
```

- 두 `ls -Z` 컨텍스트가 같음 = `--selinux` 보존 성공. 컨텍스트 없이 복원하면 `/tmp` 기본 컨텍스트(`user_tmp_t`) 로 바뀜

> 📝 **시험 포인트**: `--exclude` 는 "제외", `--listed-incremental` 은 "증분" — 보기 혼동 유형. `-C` 는 "추출 위치 지정"(`tar xzvf backup.tar.gz -C /opt`). `-p` 는 권한 보존.

### 2-5. 압축 알고리즘 비교 — z / j / J

> **상황**: 같은 데이터를 gzip·bzip2·xz 로 각각 묶어 크기와 시간을 비교하고 확장자↔옵션 대응을 몸에 익힌다.

```bash
cd /srv/raid/backup
time tar -czf cmp.tar.gz  /data/proj    # gzip
time tar -cjf cmp.tar.bz2 /data/proj    # bzip2
time tar -cJf cmp.tar.xz  /data/proj    # xz
tar  -cf  cmp.tar         /data/proj    # 무압축 기준
ls -lh cmp.tar*
```

- `-z` : gzip (`.tar.gz`, `.tgz`) — 빠름·보통 압축률
- `-j` : bzip2 (`.tar.bz2`) — 중간
- `-J` : xz (`.tar.xz`) — 가장 느림·가장 높은 압축률(텍스트)
- 확장자와 옵션 불일치 시 `gzip: stdin: not in gzip format` → GNU tar 는 추출 시 `-a`(auto) 또는 옵션 생략으로 자동 판별 가능

**검증**

```bash
ls -l /srv/raid/backup/cmp.tar* | awk '{print $5, $9}' | sort -n
file /srv/raid/backup/cmp.tar.gz /srv/raid/backup/cmp.tar.bz2 /srv/raid/backup/cmp.tar.xz
```

```text
...  cmp.tar.xz
...  cmp.tar.bz2
...  cmp.tar.gz
...  cmp.tar
cmp.tar.gz:  gzip compressed data, ...
cmp.tar.bz2: bzip2 compressed data, block size = 900k
cmp.tar.xz:  XZ compressed data, ...
```

- 텍스트(`seq` 출력)는 xz < bz2 < gz 순으로 작아지고, `blob.bin`(난수 20 MB) 은 어떤 방식도 줄지 않음 → `real` 시간은 xz 가 가장 김

> 📝 **시험 포인트**: `tar cvJf data.tar.xz` 가 xz 생성(옳음), `tar cvzf data.tar.xz` 는 확장자 불일치(틀림). `tvjf` 는 bz2 목록.

### 2-6. 복구 리허설 — 삭제 후 레벨 0 → 레벨 1 순서 복원과 원본 대조

> **상황**: 백업이 진짜인지 확인하는 유일한 방법은 복원이다. `/data/proj` 를 지우고 레벨 0 → 레벨 1 순서로 복원해 추가·수정·삭제가 모두 반영되는지 해시로 대조한다.

⚠️ `rm -rf` 대상이 `/data/proj` 인지, 백업 `data-full.tar.gz` 와 `data-inc1.tar.gz` 가 존재하는지 먼저 확인

```bash
ls -ld /data/proj && ls -lh /srv/raid/backup/data-full.tar.gz /srv/raid/backup/data-inc1.tar.gz
sha256sum /data/proj/src/*.txt /data/proj/blob.bin > /srv/raid/backup/data-after-inc1.sha256   # 현재(=레벨1 시점 + 2-3 차등 파일) 기준

rm -rf /data/proj
ls /data

# ① 레벨 0 복원 — 증분 아카이브는 항상 -g 와 함께 추출 (/dev/null = 새 snar 기록 안 함)
tar -xzvf /srv/raid/backup/data-full.tar.gz -C / --listed-incremental=/dev/null | tail -2
ls /data/proj/src | grep -c txt                     # 30 (file30 있음, file31 없음)

# ② 레벨 1 복원 — 삭제(file30)·추가(file31)·수정(file1) 반영
tar -xzvf /srv/raid/backup/data-inc1.tar.gz -C / --listed-incremental=/dev/null
ls /data/proj/src | grep -c txt                     # 30 (file30 삭제 + file31 추가)
ls /data/proj/src/file30.txt /data/proj/src/file31.txt
```

- `-x` : 추출 (e**x**tract)
- `-C /` : 아카이브 내부 경로가 `data/...`(선행 `/` 제거됨) 이므로 루트에서 풀어야 원위치
- `--listed-incremental=/dev/null` : 추출 모드에서 증분 메타데이터(dumpdir) 를 해석해 **아카이브 시점에 없던 파일을 삭제**. `-g` 없이 풀면 삭제가 반영되지 않음
- 복원 순서: 레벨 0 → 1 → 2 … 순서 엄수. 역순이면 삭제·수정이 되돌아감

**검증**

```bash
ls /data/proj/src/file30.txt 2>&1 | grep -c 'No such'        # 1 = 삭제 반영
tail -1 /data/proj/src/file1.txt                              # appended
# 해시 대조 — 레벨1 시점 이후 만든 file32/33 은 아카이브에 없으므로 그 줄만 FAILED open 가 정상
grep -v 'file3[23]' /srv/raid/backup/data-after-inc1.sha256 | sha256sum -c --quiet && echo ALL-OK
# 대안: 스테이징 디렉터리에 풀어 디렉터리 단위 비교
mkdir -p /tmp/stage && tar -xzf /srv/raid/backup/data-full.tar.gz -C /tmp/stage -g /dev/null \
  && tar -xzf /srv/raid/backup/data-inc1.tar.gz -C /tmp/stage -g /dev/null
diff -rq /tmp/stage/data/proj /data/proj
```

```text
1
appended
ALL-OK
```

- `sha256sum -c` : 목록 파일의 해시와 실제 파일 비교 (**c**heck), `--quiet` 는 OK 줄 생략. 불일치 시 `FAILED` 와 종료코드 1
- `diff -rq` : 재귀(**r**ecursive)·차이 파일명만(**q**uiet) — 출력 없음 = 동일

> 📝 **시험 포인트**: `tar xzvf backup.tar.gz -C /restore` = 지정 디렉터리에 추출. 증분 복원은 전체 → 증분 순서. `sha256sum -c` 출력의 `OK`/`FAILED` 해석.

---

## 3. /etc 설정 백업과 선택 복구

### 3-1. /etc 아카이브, 특정 파일만 추출, 암호화 보관

> **상황**: 설정 파일 하나를 잘못 고쳤을 때 전체를 되돌리는 대신 그 파일만 꺼내 비교하는 것이 실무 복구의 90% 다. 날짜 이름 아카이브를 만들고 `sshd_config` 만 추출해 diff 한다.

```bash
cd /srv/raid/backup
tar czvf etc-$(date +%F).tar.gz /etc 2>/dev/null | wc -l
ln -sf etc-$(date +%F).tar.gz etc-latest.tar.gz

# 특정 파일만 추출 (아카이브 내부 경로 = 선행 / 없는 etc/…)
mkdir -p /tmp/restore
tar xzvf etc-latest.tar.gz etc/ssh/sshd_config -C /tmp/restore
diff /tmp/restore/etc/ssh/sshd_config /etc/ssh/sshd_config && echo SAME

# 설정을 건드린 뒤 차이 확인 → 백업본으로 되돌리기
echo '# test-change' >> /etc/ssh/sshd_config
diff /tmp/restore/etc/ssh/sshd_config /etc/ssh/sshd_config
cp -a /tmp/restore/etc/ssh/sshd_config /etc/ssh/sshd_config
sshd -t && echo "sshd config OK"

# 대칭키 암호화 백업 (기출) — 비밀번호는 대화식 입력
tar czf - /etc 2>/dev/null | gpg -c -o etc-$(date +%F).tar.gz.gpg
```

- `tar czvf <파일> /etc` : create·gzip·verbose·file — `$(date +%F)` = `YYYY-MM-DD`
- `tar xzvf <아카이브> <경로>` : 아카이브 내 해당 경로만 추출 — 경로는 목록(`tzf`) 에 보이는 형태(`etc/ssh/sshd_config`) 그대로
- `cp -a` : 권한·시각·소유자 보존 복사 (**a**rchive)
- `sshd -t` : 설정 문법 검사 (**t**est) — 복원 뒤 서비스 재시작 전에 필수
- `tar czf - /etc | gpg -c -o <out>` : `-`(표준출력) 로 스트리밍 → `gpg -c`(대칭 **c**onventional 암호화) → `-o` 출력. 복호화·복원: `gpg -d etc.tar.gz.gpg | tar xzf - -C /tmp/restore`

**검증**

```bash
ls -lh /srv/raid/backup/etc-*.tar.gz* /srv/raid/backup/etc-latest.tar.gz
tar tzf /srv/raid/backup/etc-latest.tar.gz | grep -E 'etc/(fstab|passwd|samba/smb.conf|named.conf|exports)$'
gpg -d /srv/raid/backup/etc-$(date +%F).tar.gz.gpg 2>/dev/null | tar tzf - | grep -c '^etc/ssh/'
```

```text
-rw-r--r-- ... etc-2026-09-03.tar.gz
-rw-r--r-- ... etc-2026-09-03.tar.gz.gpg
lrwxrwxrwx ... etc-latest.tar.gz -> etc-2026-09-03.tar.gz
etc/exports
etc/fstab
etc/named.conf
etc/passwd
etc/samba/smb.conf
...
```

> 📝 **시험 포인트**: `tar czf - /etc | gpg -c -o etc.tar.gz.gpg` = 대칭키 암호화 백업(기출). `chmod 777` 은 오답 보기. 특정 파일 추출 문법 `tar xzvf a.tar.gz 경로 -C 대상`.

### 3-2. 패키지 목록·dnf 이력 저장

> **상황**: 재설치 복구 시 "무엇이 설치돼 있었나" 를 알아야 동일 환경을 재현할 수 있다. 이름 목록과 트랜잭션 이력을 텍스트로 남긴다.

```bash
cd /srv/raid/backup
rpm -qa --qf '%{NAME}\n' | sort > pkglist.txt
rpm -qa --last | head -20 > pkglist-recent.txt
dnf history list > dnf-history.txt
dnf repolist --all > repolist.txt
wc -l pkglist.txt dnf-history.txt
```

- `rpm -qa` : 설치된 전체 패키지 질의 (**q**uery **a**ll)
- `--qf '%{NAME}\n'` : 출력 형식 지정 (**q**uery**f**ormat) — 버전 제외 이름만 → 재설치 시 최신 버전으로 설치 가능
- `--last` : 설치 시각 역순
- `dnf history list` : 트랜잭션 ID·명령·날짜·동작 목록 — 재설치 참고용 (ID 는 새 시스템에 이관 불가)
- 재설치 시 활용(참고, 실행 안 함): `dnf install -y $(cat pkglist.txt)` 또는 `dnf install -y $(comm -13 <(rpm -qa --qf '%{NAME}\n'|sort) pkglist.txt)` (부족한 것만)

**검증**

```bash
grep -cE '^(httpd|bind|samba|vsftpd|postfix|nfs-utils|docker-ce|mdadm|quota)$' /srv/raid/backup/pkglist.txt
head -3 /srv/raid/backup/dnf-history.txt
```

```text
9
ID     | Command line              | Date and time    | Action(s)      | Altered
-------------------------------------------------------------------------------
...
```

> 📝 **시험 포인트**: `rpm -qa` 전체 목록, `--qf` 형식 지정, `dnf history undo <ID>` 는 Part 02. 패키지 목록도 백업 대상.

### 3-3. 계정 DB·crontab 덤프

> **상황**: `/etc` 아카이브에 이미 들어 있지만, 계정·스케줄은 복구 시 가장 먼저 필요하므로 별도 텍스트로도 남긴다. `shadow` 는 해시가 들어 있어 권한을 반드시 조인다.

⚠️ `getent shadow` 결과는 비밀번호 해시 — 파일 권한 600, 오프사이트 전송 시 gpg 암호화 후 전송

```bash
mkdir -p /srv/raid/backup/accounts && cd /srv/raid/backup/accounts
umask 077
getent passwd > passwd.dump
getent group  > group.dump
getent shadow > shadow.dump
getent gshadow > gshadow.dump 2>/dev/null
umask 022

# 사용자 crontab 전부 (없는 사용자는 2>/dev/null 로 무시, 빈 파일 삭제)
for u in $(cut -d: -f1 /etc/passwd); do crontab -l -u $u > cron-$u 2>/dev/null || rm -f cron-$u; done
cp -a /etc/cron.d/lab-backup cron.d-lab-backup
ls -l
```

- `getent <db>` : NSS 를 통해 DB 조회 (**get** **ent**ry) — 로컬 파일 + LDAP 등 통합 결과. `shadow` 는 root 만 가능
- `umask 077` : 이후 생성 파일 권한 600 (파일 기본 666 − 077)
- `crontab -l -u <user>` : 해당 사용자의 crontab 출력 (**l**ist, **u**ser) — 없으면 `no crontab for …` 와 종료코드 1
- `cut -d: -f1` : `:` 구분 1번 필드 = 사용자명

**검증**

```bash
ls -l /srv/raid/backup/accounts/shadow.dump | awk '{print $1}'     # -rw-------
grep -E '^(dev1|dev2|ops1|admin1):' /srv/raid/backup/accounts/passwd.dump | cut -d: -f1,3,4
ls /srv/raid/backup/accounts/cron-*
cat /srv/raid/backup/accounts/cron-dev1
```

```text
-rw-------
admin1:1000:1000
dev1:2001:2000
dev2:2002:2000
ops1:2003:2001
/srv/raid/backup/accounts/cron-dev1
...
```

> 📝 **시험 포인트**: `/etc/shadow` 주기 백업은 침해 대비책(기출 정답 보기). `getent passwd` 는 `/etc/passwd` 직접 읽기와 달리 NSS 소스 전체를 조회.

---

## 4. rsync 미러·오프사이트 백업

### 4-1. 후행 슬래시(`/`) 하나의 차이 재현

> **상황**: rsync 사고의 대부분은 원본 경로 끝 `/` 하나에서 나온다. `/srv/share` 를 미러링하기 전에 슬래시 유무가 결과 트리를 어떻게 바꾸는지 먼저 눈으로 확인한다.

```bash
rpm -q rsync || dnf install -y rsync
rm -rf /tmp/rs-a /tmp/rs-b && mkdir -p /tmp/rs-a /tmp/rs-b

# ① 원본 끝 슬래시 없음 → 디렉터리 "자체" 가 대상 아래로 들어감
rsync -a /srv/share  /tmp/rs-a/
# ② 원본 끝 슬래시 있음 → 디렉터리 "내용" 만 대상으로 들어감
rsync -a /srv/share/ /tmp/rs-b/

find /tmp/rs-a /tmp/rs-b -maxdepth 2 | sort
```

- `-a` (`--archive`) : `-rlptgoD` 묶음 — **r**ecursive · 심볼릭**l**ink 보존 · **p**ermission · **t**ime · **g**roup · **o**wner · **D**evice/special 파일. **ACL·xattr·하드링크·압축은 포함되지 않음**(4-4 에서 별도 지정)
- 원본 `src` → 대상에 `dest/src/…` 생성 / 원본 `src/` → 대상에 `dest/…` 직접 전개
- 대상 경로의 슬래시는 결과에 영향 없음 — **원본 쪽만** 의미 있음

**검증**

```bash
ls /tmp/rs-a
ls /tmp/rs-b
```

```text
share
readme.txt
```

- `/tmp/rs-a` 에는 `share` 디렉터리가 한 겹 더 생겼고, `/tmp/rs-b` 에는 내용이 바로 풀림

> 📝 **시험 포인트**: `rsync -av /data /backup` → `/backup/data/…`, `rsync -av /data/ /backup/` → `/backup/…`. `--delete` 와 조합될 때 이 차이가 삭제 범위를 바꿔 실기 서술형 단골.

### 4-2. dry-run 선행 → 실제 미러 → `--delete` 동기화

> **상황**: 파괴 가능성이 있는 `--delete` 는 반드시 `-n` 으로 예행연습을 먼저 한다. `/srv/share` 를 `/srv/raid/mirror/share` 로 미러링하고, 원본에서 지운 파일이 미러에서도 사라지는지 확인한다.

```bash
mkdir -p /srv/raid/mirror
# ① 예행연습 — 무엇이 전송/삭제되는지만 출력
rsync -avn --delete /srv/share/ /srv/raid/mirror/share/
# ② 실제 실행
rsync -av --delete /srv/share/ /srv/raid/mirror/share/

# ③ 미러에만 있는 잉여 파일을 만들고 --delete 동작 확인
echo "orphan" > /srv/raid/mirror/share/orphan.txt
rsync -avn --delete /srv/share/ /srv/raid/mirror/share/ | grep orphan
rsync -av --delete --stats /srv/share/ /srv/raid/mirror/share/ | tail -12

# ④ 진행률 표시 (대용량 파일 전송 시)
rsync -av --progress /data/proj/blob.bin /srv/raid/mirror/
```

- `-n` (`--dry-run`) : 실제 전송·삭제 없이 목록만 출력 (**n**o-op) — `--delete` 와 항상 짝
- `-v` : 전송 파일명 출력. `-vv` 는 건너뛴 파일까지
- `--delete` : **원본에 없는 파일을 대상에서 삭제** → 미러 동기화. 원본 경로를 잘못 지정하면 대상이 통째로 비워질 수 있음
- `--stats` : 전송 파일 수·바이트·속도·가속비(speedup) 요약
- `--progress` : 파일별 진행률·전송률·ETA. `-P` = `--partial --progress`
- 삭제 시점 조정: `--delete-before`(기본, 전송 전) / `--delete-after`(전송 후) / `--delete-delay`

**검증**

```bash
ls /srv/raid/mirror/share/orphan.txt 2>&1 | grep -c 'No such'
diff -rq /srv/share /srv/raid/mirror/share; echo "diff=$?"
du -sh /srv/share /srv/raid/mirror/share
```

```text
1
diff=0
...M	/srv/share
...M	/srv/raid/mirror/share
```

- `diff=0` + 출력 없음 = 두 트리 완전 동일

> 📝 **시험 포인트**: 필기 FULL r03-61 · r06-42 — "원본에서 삭제된 파일을 대상에서도 제거해 동기화" = `--delete`. `rsync -avz --delete src/ dest/` 해석 문제.

### 4-3. 제외 목록·대역폭 제한·중단 재개

> **상황**: 운영 미러는 캐시·임시 파일을 빼고, 업무 시간 대역폭을 갉아먹지 않아야 하며, 끊긴 전송을 처음부터 다시 하지 않아야 한다.

```bash
cd /srv/raid/backup
cat > rsync-exclude.lst <<'EOF'
*.tmp
*.swp
lost+found/
cache/
EOF

# 패턴 직접 / 파일에서 제외
rsync -avn --exclude='*.bin' /data/proj/ /srv/raid/mirror/proj/
rsync -av  --exclude-from=/srv/raid/backup/rsync-exclude.lst /data/proj/ /srv/raid/mirror/proj/

# 대역폭 100 KB/s 제한 + 중단 재개 (blob.bin 20 MB)
time rsync -av --bwlimit=100 --partial --progress /data/proj/blob.bin /srv/raid/mirror/limited/
# 부분 파일을 별도 디렉터리에 모으기
rsync -av --partial-dir=.rsync-partial /data/proj/blob.bin /srv/raid/mirror/limited/
```

- `--exclude=<패턴>` : 전송 제외 패턴. `/` 로 시작하면 전송 루트 기준 절대 경로, 끝이 `/` 면 디렉터리만
- `--exclude-from=<파일>` : 파일의 각 줄을 제외 패턴으로. `--include` 와 조합 시 **먼저 매치된 규칙이 승리** → `--include` 를 앞에
- `--bwlimit=<KB/s>` : 전송 대역폭 상한 (**b**and**w**idth **limit**). `0` = 무제한
- `--partial` : 중단된 전송의 부분 파일을 지우지 않고 남겨 다음 실행에서 이어받음 (기본은 삭제)
- `--partial-dir=<디렉터리>` : 부분 파일 보관 위치 — 대상 트리에 불완전 파일이 섞이지 않게 함
- `-P` : `--partial --progress` 축약

**검증**

```bash
ls /srv/raid/mirror/proj/ | grep -c '\.bin$'
ls -lh /srv/raid/mirror/limited/blob.bin
rsync -avn /data/proj/blob.bin /srv/raid/mirror/limited/ | grep -c blob.bin
```

```text
0
-rw-r--r-- 1 root root 20M ... blob.bin
0
```

- 마지막 `0` = dry-run 전송 목록에 없음 → 이미 동일하므로 재전송 안 함

> 📝 **시험 포인트**: `--bwlimit` 단위는 **KB/s**. `--partial` 은 "재개", `--append` 는 "이어 붙이기"(크기만 비교, 위험). 제외는 tar 의 `--exclude` 와 동일 개념.

### 4-4. 하드링크·ACL·xattr 보존 검증 (`-H -A -X`)

> **상황**: `-a` 는 하드링크·ACL·확장 속성을 보존하지 않는다. Samba 공유(`/srv/share`)에는 ACL 이 걸려 있으므로 옵션을 추가하고 실제로 보존되는지 대조한다.

```bash
# 검증용 자원 준비 — 하드링크 1쌍, ACL 1개, xattr 1개
echo "hardlink-src" > /srv/share/hl1.txt
ln /srv/share/hl1.txt /srv/share/hl2.txt          # 하드링크 (inode 공유)
setfacl -m u:dev1:rw /srv/share/hl1.txt            # ACL (Part 03 dev1)
setfattr -n user.owner -v devteam /srv/share/hl1.txt
ls -li /srv/share/hl*.txt

# ① -a 만 → 하드링크가 별개 파일 2개로 복사됨
rm -rf /tmp/rs-noh && rsync -a /srv/share/ /tmp/rs-noh/
ls -li /tmp/rs-noh/hl*.txt

# ② -aHAX → 하드링크·ACL·xattr 보존
rm -rf /tmp/rs-hax && rsync -aHAX /srv/share/ /tmp/rs-hax/
ls -li /tmp/rs-hax/hl*.txt
```

- `-H` (`--hard-links`) : 원본의 **하드링크 관계**를 대상에서도 유지 (**H**ard link). 없으면 링크 수만큼 실제 복사본 발생 → 용량 증가
- `-A` (`--acls`) : POSIX **ACL** 전송. 대상 파일시스템이 ACL 을 지원해야 함(xfs·ext4 지원)
- `-X` (`--xattrs`) : 확장 속성 전송 — `user.*` 뿐 아니라 `security.selinux`(SELinux 컨텍스트) 도 포함. root 권한 필요
- SELinux 만 필요하면 `rsync -aX` 대신 대상에서 `restorecon -R` 로 정책 기본값 재적용도 가능(Part 10)

**검증**

```bash
stat -c '%i %h %n' /tmp/rs-noh/hl1.txt /tmp/rs-noh/hl2.txt
stat -c '%i %h %n' /tmp/rs-hax/hl1.txt /tmp/rs-hax/hl2.txt
getfacl -p /tmp/rs-hax/hl1.txt | grep dev1
getfattr -n user.owner /tmp/rs-hax/hl1.txt
getfacl -p /tmp/rs-noh/hl1.txt | grep -c dev1
```

```text
... 1 /tmp/rs-noh/hl1.txt
... 1 /tmp/rs-noh/hl2.txt
123456 2 /tmp/rs-hax/hl1.txt
123456 2 /tmp/rs-hax/hl2.txt
user:dev1:rw-
user.owner="devteam"
0
```

- `-aHAX` 쪽만 inode 가 같고 링크 수(`%h`)가 2 → 하드링크 보존 성공
- `-a` 만 쓴 쪽은 ACL 이 사라져 `grep -c dev1` = 0

> 📝 **시험 포인트**: `-a` 에 **포함되지 않는** 것 4가지 — `-H`(하드링크) `-A`(ACL) `-X`(xattr) `-z`(압축). "권한·소유자·시각 보존" 만 물으면 `-a` 로 충분.

### 4-5. 전송 판정 기준 — 크기·시각 비교 vs `--checksum`

> **상황**: rsync 는 기본적으로 **크기와 mtime** 만 보고 변경 여부를 판단한다. 크기·시각이 같은데 내용만 다른 파일은 놓친다. 그 상황을 인위적으로 만들어 `-c` 의 필요성을 확인한다.

```bash
echo "AAAA" > /srv/share/quick.txt
rsync -a /srv/share/ /srv/raid/mirror/share/

# 내용만 바꾸고 크기·mtime 은 대상과 동일하게 맞춤
printf 'BBBB\n' > /srv/share/quick.txt
touch -r /srv/raid/mirror/share/quick.txt /srv/share/quick.txt
stat -c '%s %Y %n' /srv/share/quick.txt /srv/raid/mirror/share/quick.txt

# ① 기본(크기+mtime) → 변경 감지 못 함
rsync -avn /srv/share/ /srv/raid/mirror/share/ | grep -c quick.txt
# ② --checksum → 내용 해시로 비교 → 전송 대상
rsync -avnc /srv/share/ /srv/raid/mirror/share/ | grep -c quick.txt
rsync -avc /srv/share/ /srv/raid/mirror/share/ | grep quick.txt
```

- `-c` (`--checksum`) : 파일 전체 체크섬으로 동일 여부 판정 — 정확하지만 **양쪽 전체를 읽으므로 느림**
- `--size-only` : 크기만 비교 (mtime 무시) — 시각이 신뢰 불가한 FAT/rsync 대상에 사용
- `-u` (`--update`) : 대상이 더 최신이면 건너뜀
- `--itemize-changes` (`-i`) : 파일별 변경 사유를 `>f.st....` 형태 플래그로 출력 — `s`=size, `t`=time, `c`=checksum

**검증**

```bash
cat /srv/raid/mirror/share/quick.txt
rsync -avni /srv/share/ /srv/raid/mirror/share/ | head -3
```

```text
BBBB
sending incremental file list
./
```

> 📝 **시험 포인트**: 필기 FULL r01-62 · r10-44 — "rsync 는 변경되지 않은 파일도 매번 전체 재전송한다" = **오답**. 델타 알고리즘으로 변경 블록만 전송하며, 판정 기본값은 크기+수정시각.

### 4-6. `--link-dest` 하드링크 스냅샷 백업 3세대

> **상황**: 매일 전체 백업처럼 보이면서 실제 용량은 변경분만 쓰는 방식이 하드링크 스냅샷이다. 3세대를 만들어 inode 공유와 용량 절감을 직접 확인한다.

```bash
BK=/srv/raid/backup/snapshots
mkdir -p $BK

# 1세대 — 기준 (link-dest 없음)
rsync -a --delete /data/proj/ $BK/gen1/

# 2세대 — 1세대와 비교해 변경된 파일만 새로 쓰고 나머지는 하드링크
echo "gen2 added" > /data/proj/src/gen2.txt
rsync -a --delete --link-dest=$BK/gen1 /data/proj/ $BK/gen2/

# 3세대
echo "gen3 added" > /data/proj/src/gen3.txt
echo "modified"  >> /data/proj/src/file1.txt
rsync -a --delete --link-dest=$BK/gen2 /data/proj/ $BK/gen3/

ls $BK
```

- `--link-dest=<디렉터리>` : 지정 디렉터리와 **내용이 같은 파일은 복사 대신 하드링크** 생성. 여러 개 지정 가능(최대 20). 상대 경로는 **대상 디렉터리 기준**이므로 절대 경로 권장
- 각 세대 디렉터리는 그 시점의 완전한 트리처럼 보이며(`ls` 로 전부 보임), 변경되지 않은 파일은 물리 블록 1벌만 차지
- 세대 삭제는 그냥 `rm -rf $BK/gen1` — 남은 세대의 하드링크가 데이터를 계속 붙잡으므로 안전
- 유사 옵션: `--copy-dest`(하드링크 대신 복사) / `--compare-dest`(같으면 아예 생성 안 함)

**검증**

```bash
# 변경 없는 파일 → 세대 간 inode 동일, 링크 수 3
ls -li $BK/gen1/src/file2.txt $BK/gen2/src/file2.txt $BK/gen3/src/file2.txt
# 변경된 파일 → gen3 만 다른 inode
ls -li $BK/gen1/src/file1.txt $BK/gen3/src/file1.txt
# 세대별 "논리" 용량 vs 전체 "실제" 용량
du -sh $BK/gen1 $BK/gen2 $BK/gen3
du -sh $BK
df -h /srv/raid | tail -1
```

```text
131074 3 -rw-r--r-- 3 root root ... $BK/gen1/src/file2.txt
131074 3 -rw-r--r-- 3 root root ... $BK/gen2/src/file2.txt
131074 3 -rw-r--r-- 3 root root ... $BK/gen3/src/file2.txt
131073 1 ... gen1/src/file1.txt
131099 1 ... gen3/src/file1.txt
21M	$BK/gen1
21M	$BK/gen2
21M	$BK/gen3
21M	$BK
```

- 세대별로는 각각 21M 로 보이지만 `$BK` 전체 합계도 21M → `du` 가 동일 inode 를 한 번만 세기 때문. **전체 백업 3벌의 외형 + 증분 1벌의 용량**

> 📝 **시험 포인트**: `--link-dest` 는 "하드링크 기반 스냅샷 백업" 서술형에 등장. 하드링크는 **같은 파일시스템 안에서만** 가능 → 세대 디렉터리를 다른 FS 로 옮기면 링크가 풀려 용량이 3배가 됨.

### 4-7. 오프사이트 전송 — macOS 호스트 (3-2-1 의 "1")

> **상황**: RAID 미러는 같은 VM 안에 있으므로 VM 이 통째로 날아가면 함께 사라진다. 3-2-1 원칙의 오프사이트 사본을 macOS 호스트(`~/lab-backup`)에 만든다.

- macOS 준비: **시스템 설정 → 일반 → 공유 → 원격 로그인(Remote Login) 켜기** → sshd 기동(기본 포트 **22**)
- macOS IP 는 UTM 공유 네트워크의 게이트웨이 = `192.168.64.1` (실제 값은 게스트에서 `ip route | grep default` 로 확인)
- Rocky 서버의 sshd 는 Part 08 에서 **2222** 로 바꿨지만, 이는 **서버로 들어오는** 포트다. 서버에서 macOS 로 **나갈 때**는 macOS 의 포트(22)를 씀 → 혼동 주의

```bash
ip route | grep default                       # → default via 192.168.64.1 dev enp0s1
MAC=192.168.64.1                              # macOS 호스트 (실습 환경 값으로 치환)
MACUSER=<mac-user>

# ① push 방식 — Rocky → macOS (macOS sshd 기본 22)
ssh $MACUSER@$MAC 'mkdir -p ~/lab-backup'
rsync -avz --delete /srv/raid/backup/ $MACUSER@$MAC:~/lab-backup/

# ② macOS sshd 포트가 22 가 아닐 때 — -e 로 원격 셸 지정
rsync -avz -e "ssh -p 2222" /srv/raid/backup/ $MACUSER@$MAC:~/lab-backup/

# ③ pull 방식 — macOS 터미널에서 실행 (서버 sshd 는 2222)
#   $ rsync -avz -e "ssh -p 2222" admin1@192.168.64.10:/srv/raid/backup/ ~/lab-backup/
```

- `-z` (`--compress`) : 전송 중 압축 (**z**) — 이미 `.gz`·`.xz` 인 파일에는 효과 없이 CPU 만 소모. `--skip-compress=gz/xz/bz2/img` 로 제외 가능
- `-e "<원격셸>"` : 원격 셸 명령 지정 (**e**xecute). 포트 변경·키 지정(`-e "ssh -p 2222 -i ~/.ssh/lab_ed25519"`) 시 필수
- 원격 경로 형식 `user@host:경로` — 콜론 1개. `::` 는 rsync 데몬(873/tcp) 모드
- 키 인증(Part 08 `ssh-keygen`·`ssh-copy-id`) 이 되어 있어야 cron 무인 실행 가능. 비밀번호 프롬프트가 뜨면 자동화 실패

**검증**

```bash
ssh $MACUSER@$MAC 'ls -1 ~/lab-backup | head; du -sh ~/lab-backup'
rsync -avzn --delete /srv/raid/backup/ $MACUSER@$MAC:~/lab-backup/ | tail -3
```

```text
accounts
data-full.tar.gz
data-inc1.tar.gz
etc-2026-09-03.tar.gz
...
sent ... bytes  received ... bytes  ... bytes/sec
total size is ...  speedup is ...
```

- 두 번째 dry-run 이 전송 목록 없이 요약만 출력 = 양쪽 동일

> 📝 **시험 포인트**: 실기 r02-5 — "권한·심볼릭링크 보존 및 압축 전송" = `rsync -avz /data user@host:/backup`. `-a` 안에 링크·권한 보존이 들어 있고 `-z` 가 압축. 포트 지정은 `-e "ssh -p <포트>"`.

### 4-8. `scp` 대안과 전송 무결성 검증

> **상황**: 단발성 전송에는 `scp` 로 충분하다. 다만 어느 쪽이든 "전송된 파일이 원본과 같은가" 는 체크섬으로 증명해야 한다. 해시 목록을 동봉해 원격에서 대조한다.

```bash
# 산출물 해시 목록 생성 (경로 없이 파일명만 담아야 원격에서 -c 가 편함)
cd /srv/raid/backup
sha256sum *.tar.gz *.img.gz 2>/dev/null > backup.sha256
cat backup.sha256 | head -3

# scp 대안 — 디렉터리 전체 전송
scp -P 22 -r /srv/raid/backup $MACUSER@$MAC:~/lab-backup-scp
# (macOS 에서 서버 것을 가져올 때는 서버 포트 2222)
#   $ scp -P 2222 -r admin1@192.168.64.10:/srv/raid/backup ~/lab-backup-scp

# rsync 로 해시 목록까지 동봉 후 원격 검증
rsync -avz /srv/raid/backup/backup.sha256 $MACUSER@$MAC:~/lab-backup/
```

- `scp -P <포트>` : 포트 지정 — **대문자 P** (소문자 `-p` 는 "시각·권한 보존", `ssh -p` 는 소문자) → 최빈출 혼동 지점
- `scp -r` : 디렉터리 재귀 (**r**ecursive)
- `scp -C` : 압축 전송. `scp -i <키파일>` : 키 지정
- RHEL 9 의 `scp` 는 내부적으로 **SFTP 프로토콜**을 사용(`-O` 로 구 SCP 프로토콜 강제 가능)
- `sha256sum -c <목록>` : 목록의 해시와 현재 디렉터리 파일 비교. **macOS 에는 GNU `sha256sum` 이 없음** → `shasum -a 256 -c backup.sha256` 또는 `shasum -a 256 <파일>` 사용

**검증**

```bash
# 서버 쪽 자체 검증
cd /srv/raid/backup && sha256sum -c backup.sha256
# macOS 쪽 검증 (macOS 터미널)
#   $ cd ~/lab-backup && shasum -a 256 -c backup.sha256
ssh $MACUSER@$MAC 'cd ~/lab-backup && shasum -a 256 -c backup.sha256'
```

```text
data-full.tar.gz: OK
data-inc1.tar.gz: OK
etc-2026-09-03.tar.gz: OK
...
```

- 한 줄이라도 `FAILED` 면 전송 중 손상 → 재전송. 종료 코드는 1

> 📝 **시험 포인트**: 필기 FULL r06-96 · r09-36 — "송수신 양쪽에서 `sha256sum` 값을 비교" 가 무결성 검증 정답. `sha256sum -c` 출력의 `OK`/`FAILED` 해석과 `scp -P`(포트) vs `ssh -p`(포트) 대문자·소문자 구분.

---

## 5. dd 저수준 백업·복구

### 5-1. 파티션 테이블·디스크 메타데이터 백업

> **상황**: 파일은 tar 로 살릴 수 있어도 파티션 테이블이 날아가면 어디부터 어디까지가 무슨 파일시스템이었는지 알 수 없다. 텍스트·바이너리 두 형태로 모두 남긴다.

```bash
lsblk                                          # ⚠️ 대상 장치 재확인
cd /srv/raid/backup

# ① GPT 선두 34섹터 = 보호MBR(LBA0) + GPT 헤더(LBA1) + 파티션 엔트리(LBA2~33)
dd if=/dev/vda of=vda-pt.bin bs=512 count=34 status=none
# ② MBR 디스크라면 첫 1섹터만 (기출 형태)
dd if=/dev/vdb of=vdb-mbr.bin bs=512 count=1 status=none
# ③ 부트코드(446B) 를 뺀 파티션 테이블 64바이트만
dd if=/dev/vdb of=vdb-pt64.bin bs=1 count=64 skip=446 status=none

# ④ 텍스트 덤프 — 사람이 읽고 편집 가능, 복원도 이 파일로
sfdisk -d /dev/vda > vda-sfdisk.txt
sfdisk -d /dev/vdb > vdb-sfdisk.txt
# ⑤ GPT 전용 백업 (gdisk 패키지)
rpm -q gdisk || dnf install -y gdisk
sgdisk --backup=vda.gpt /dev/vda

# ⑥ UUID·라벨·구성 스냅샷
blkid > blkid.txt
lsblk -f > lsblk.txt
lsblk -o NAME,SIZE,TYPE,FSTYPE,UUID,MOUNTPOINT > lsblk-full.txt
cat /etc/fstab > fstab.bak
ls -lh vda-pt.bin vdb-mbr.bin vdb-pt64.bin vda-sfdisk.txt vda.gpt blkid.txt
```

- `if=` : 입력 파일 (**i**nput **f**ile). 생략 시 표준입력
- `of=` : 출력 파일 (**o**utput **f**ile). ⚠️ 장치를 지정하면 그 장치를 덮어씀
- `bs=` : 블록 크기 (**b**lock **s**ize) — 한 번에 읽고 쓰는 단위
- `count=` : 복사할 블록 개수 → 총 바이트 = `bs × count`
- `skip=N` : **입력** 앞부분 N 블록 건너뜀
- `sfdisk -d <장치>` : 파티션 테이블을 텍스트로 덤프 (**d**ump). 복원은 `sfdisk /dev/vda < vda-sfdisk.txt`
- `sgdisk --backup=<파일> <장치>` : GPT 헤더+엔트리(주·백업 양쪽) 저장. 복원 `sgdisk --load-backup=<파일> <장치>`
- GPT 는 디스크 **끝**에도 백업 헤더를 두므로, 선두가 깨져도 `gdisk` 의 복구 메뉴(`r` → `b`)로 살릴 수 있음

**검증**

```bash
ls -l /srv/raid/backup/vda-pt.bin | awk '{print $5}'      # 34 × 512 = 17408
ls -l /srv/raid/backup/vdb-pt64.bin | awk '{print $5}'    # 64
file /srv/raid/backup/vda-pt.bin
head -6 /srv/raid/backup/vda-sfdisk.txt
grep vdb /srv/raid/backup/blkid.txt
```

```text
17408
64
vda-pt.bin: DOS/MBR boot sector; partition 1 : ID=0xee, ...
label: gpt
label-id: ...
device: /dev/vda
unit: sectors
first-lba: 34
last-lba: ...
/dev/vdb1: UUID="..." TYPE="ext4"
/dev/vdb2: UUID="..." TYPE="swap"
```

- `ID=0xee` = **보호 MBR** — GPT 디스크를 옛 MBR 도구가 "알 수 없는 꽉 찬 파티션"으로 보게 해 실수로 덮어쓰지 못하게 하는 장치

> 📝 **시험 포인트**: 필기 FULL r09-47 — 보호 MBR 의 목적은 "GPT 를 인식하지 못하는 도구가 디스크를 빈 것으로 오인해 덮어쓰는 것을 방지". GPT 는 파티션 테이블 백업본을 **디스크 끝**에 보관(r03-46).

### 5-2. dd 옵션 정리와 진행률 확인

> **상황**: dd 는 옵션 하나가 데이터 파괴와 직결된다. 실습 전에 옵션 의미를 표로 고정하고, 긴 작업의 진행률을 보는 두 가지 방법을 확인한다.

```bash
# ① 최신 coreutils — 실행 중 진행률 표시
dd if=/dev/zero of=/tmp/ddtest.img bs=1M count=300 status=progress

# ② 구형 호환 — 실행 중인 dd 에 USR1 시그널 → 그 시점 통계 출력
dd if=/dev/zero of=/tmp/ddtest2.img bs=1M count=2000 &
sleep 2; kill -USR1 $(pidof dd); sleep 2; kill -USR1 $(pidof dd)
wait

# ③ pv 로 파이프 진행률 (EPEL)
rpm -q pv || dnf install -y pv
dd if=/dev/zero bs=1M count=200 status=none | pv | dd of=/tmp/ddtest3.img bs=1M status=none
rm -f /tmp/ddtest*.img
```

| 옵션 | 의미 | 비고 |
| --- | --- | --- |
| `if=<경로>` | 입력 파일·장치 | 생략 시 stdin |
| `of=<경로>` | 출력 파일·장치 | ⚠️ 지정 실수 = 파괴 |
| `bs=<크기>` | 블록 크기 (`512`, `1M`, `4M`) | 클수록 빠름, 장치 백업은 `4M` 권장 |
| `ibs=`/`obs=` | 입력·출력 블록 크기 개별 지정 | `bs=` 는 둘 다 설정 |
| `count=<N>` | 복사할 블록 수 | 총량 = `bs × count` |
| `skip=<N>` | **입력** 시작 위치를 N 블록 건너뜀 | 소스 오프셋 |
| `seek=<N>` | **출력** 시작 위치를 N 블록 건너뜀 | 대상 오프셋 |
| `conv=noerror` | 읽기 오류가 나도 중단하지 않고 계속 | 불량 섹터 디스크 구조 |
| `conv=sync` | 읽은 블록이 짧으면 NUL 로 채워 크기 유지 | `noerror` 와 짝 (`conv=noerror,sync`) |
| `conv=notrunc` | 출력 파일을 자르지 않고 해당 부분만 덮어씀 | MBR 부분 복원에 필수 |
| `conv=fsync` | 종료 전 출력 버퍼를 디스크에 강제 기록 | 이미지 백업 신뢰성 |
| `conv=sparse` | 전부 0 인 블록은 기록 생략 (희소 파일) | 이미지 용량 절감 |
| `iflag=direct` / `oflag=direct` | 페이지 캐시 우회(O_DIRECT) | 실측 성능 측정·대용량 |
| `oflag=sync` | 매 블록마다 동기 기록 | 매우 느림 |
| `status=none\|noxfer\|progress` | 종료 통계 억제 / 전송률만 생략 / 실시간 진행률 | `progress` 권장 |

- `kill -USR1 <PID>` : 실행 중 dd 에 통계 출력 요청 — `status=progress` 가 없던 시절 방식이며 지금도 동작
- `pidof dd` : 실행 중인 dd 의 PID (여러 개면 전부) → `pgrep -x dd` 로도 가능

**검증**

```bash
dd if=/dev/zero of=/tmp/x.img bs=1M count=10 status=progress 2>&1 | tail -2
rm -f /tmp/x.img
```

```text
10+0 records in
10+0 records out
10485760 bytes (10 MB, 10 MiB) copied, ... s, ... MB/s
```

- `10+0` = 완전한 블록 10개 + 부분 블록 0개. `0+1` 형태가 보이면 블록이 잘렸다는 뜻

> 📝 **시험 포인트**: `bs × count` 계산 문제 빈출 — 필기 FULL r06-34 · r07-41 `dd if=/dev/zero of=/swapfile bs=1M count=2048` = **2 GB 스왑 파일**. `skip`=입력, `seek`=출력 방향 혼동 주의.

### 5-3. 파티션 이미지 백업 — `/dev/vdb1` (`/data`)

> **상황**: 파일 단위 백업(tar) 과 달리 이미지는 파일시스템 구조·UUID·inode 배치까지 그대로 보존한다. `/data` 를 언마운트하고 정지 상태에서 이미지를 뜬다.

⚠️ 마운트된 파일시스템을 그대로 dd 하면 일관성이 깨진다. 반드시 `umount` 후 진행하고, `if=`/`of=` 방향을 두 번 확인

```bash
# ① 대상 확인 → 언마운트
lsblk -f /dev/vdb
findmnt /data
fuser -vm /data 2>&1 | head            # 사용 중인 프로세스 확인
umount /data && findmnt /data; echo "umount rc=$?"

# ② 이미지 백업 (2 GB 파티션)
df -h /srv/raid | tail -1              # 여유 공간 확인
dd if=/dev/vdb1 of=/srv/raid/backup/vdb1.img bs=4M status=progress conv=fsync

# ③ 압축 보관 (빈 블록이 대부분이라 크게 줄어듦)
gzip -9 /srv/raid/backup/vdb1.img
ls -lh /srv/raid/backup/vdb1.img.gz

# ④ 파이프로 바로 압축하는 방식 (디스크 여유가 없을 때)
#   dd if=/dev/vdb1 bs=4M status=progress | gzip -c > /srv/raid/backup/vdb1.img.gz

# ⑤ 이미지 자체의 체크섬 기록
sha256sum /srv/raid/backup/vdb1.img.gz >> /srv/raid/backup/backup.sha256
mount /data && findmnt /data
```

- `fuser -vm <마운트지점>` : 해당 파일시스템을 쓰는 프로세스 목록 (**v**erbose, **m**ount) — `umount: target is busy` 원인 파악
- `gzip -9` : 최대 압축률 (**1**=최속 ~ **9**=최대). `-c` 는 표준출력으로
- `conv=fsync` : 마지막에 `fsync()` 호출 → 캐시에만 남은 채 "완료" 로 오인하는 것을 방지
- 이미지 크기는 **항상 파티션 전체 크기**(2 GB) — 사용량과 무관. 압축·`conv=sparse` 로만 줄어듦

**검증**

```bash
ls -lh /srv/raid/backup/vdb1.img.gz
gunzip -c /srv/raid/backup/vdb1.img.gz | file -            # ext4 인지 확인
gunzip -c /srv/raid/backup/vdb1.img.gz | head -c 2M > /tmp/probe.img
file /tmp/probe.img && rm -f /tmp/probe.img
findmnt /data
```

```text
-rw-r--r-- 1 root root ...M ... vdb1.img.gz
/dev/stdin: Linux rev 1.0 ext4 filesystem data, UUID=... (extents) (64bit) (large files) (huge files)
TARGET SOURCE    FSTYPE OPTIONS
/data  /dev/vdb1 ext4   rw,relatime,quota,usrquota,grpquota
```

> 📝 **시험 포인트**: `dd` 는 "파일시스템 구조와 무관한 블록 단위 복제"(필기 FULL r10-44 ①, r09-55 ①). 마운트 중 이미지는 불일치 위험 → 언마운트 또는 스냅샷 사용.

### 5-4. ⚠️ 고의 파괴 후 이미지 복구 리허설

> **상황**: 복구는 해봐야 복구다. `/dev/vdb1` 을 `mkfs` 로 완전히 밀어버린 뒤 이미지에서 되살리고, 파일 존재·해시·UUID 까지 대조한다. 이 리허설이 파트의 핵심 검증이다.

⚠️ 아래 `mkfs.ext4` 는 **`/data` 를 완전히 파괴**한다. 대상이 `/dev/vdb1` 이 맞는지, `vdb1.img.gz` 가 정상인지 반드시 먼저 확인. `/dev/vda*` 를 지정하면 시스템이 부팅 불능이 된다

```bash
# ⓪ 파괴 전 기준값 저장
mount /data 2>/dev/null
sha256sum /data/proj/src/*.txt > /srv/raid/backup/vdb1-before.sha256
blkid /dev/vdb1 | tee /srv/raid/backup/vdb1-uuid-before.txt
ls /data | head

# ① 대상 재확인 — 이 3줄을 반드시 눈으로 읽을 것
lsblk -f /dev/vdb
findmnt /data
ls -lh /srv/raid/backup/vdb1.img.gz

# ② 언마운트 후 고의 파괴
umount /data
mkfs.ext4 -F /dev/vdb1                 # ⚠️ 파괴 지점
blkid /dev/vdb1                        # UUID 가 바뀌었음을 확인
mount /data && ls /data                # lost+found 만 남음 → 데이터 소실 확인
umount /data

# ③ 이미지에서 복구
gunzip -c /srv/raid/backup/vdb1.img.gz | dd of=/dev/vdb1 bs=4M status=progress conv=fsync

# ④ 마운트 후 검증
mount /data
ls /data
```

- `mkfs.ext4 -F` : 확인 프롬프트 없이 강제 생성 (**F**orce) — 실습용이며 운영에서는 쓰지 않음
- 복원은 `gunzip -c … | dd of=<장치>` 파이프 — `if=` 없이 표준입력을 받음
- **UUID 도 블록에 들어 있으므로 이미지 복원과 함께 원래 값으로 돌아옴** → `/etc/fstab` 의 UUID 항목이 다시 맞아떨어짐. `mkfs` 만 하고 방치하면 부팅 시 fstab 불일치로 emergency 모드(Part 07 9절)

**검증**

```bash
blkid /dev/vdb1
diff <(blkid /dev/vdb1) /srv/raid/backup/vdb1-uuid-before.txt && echo "UUID 복원 OK"
cd / && sha256sum -c /srv/raid/backup/vdb1-before.sha256 | tail -3
ls /data/proj/src | wc -l
findmnt /data
repquota -a 2>/dev/null | head -5        # 쿼터도 이미지에 포함되어 복원됨 (Part 05)
```

```text
/dev/vdb1: UUID="..." TYPE="ext4"
UUID 복원 OK
/data/proj/src/file9.txt: OK
...
30
TARGET SOURCE    FSTYPE OPTIONS
/data  /dev/vdb1 ext4   rw,relatime,quota,usrquota,grpquota
```

- 전체 `OK` = 블록 단위 복원이 파일 내용까지 비트 단위로 되살렸다는 증명

> 📝 **시험 포인트**: "이미지 복원 후 UUID 는?" → 원본과 동일(블록째 복사). `mkfs` 는 새 UUID 생성 → fstab 갱신 필요. 실기 서술형: 백업·복구 절차를 umount → dd → mount → 검증 순으로 쓰기.

### 5-5. MBR 백업·복원의 바이트 수 구분 (기출 핵심)

> **상황**: MBR 512바이트는 부트코드 446 + 파티션 테이블 64 + 시그니처 2 로 나뉜다. 복원 시 어디까지 되돌릴지에 따라 명령이 달라지고, 그 구분이 그대로 출제된다.

```bash
# 백업 — 512바이트 전체 (부트코드 + 파티션 테이블 + 시그니처)
dd if=/dev/vdb of=/srv/raid/backup/vdb-mbr-512.bin bs=512 count=1
# 백업 — 부트코드만
dd if=/dev/vdb of=/srv/raid/backup/vdb-boot446.bin bs=446 count=1
# 백업 — 파티션 테이블 64바이트만
dd if=/dev/vdb of=/srv/raid/backup/vdb-pt64.bin bs=1 count=64 skip=446

# 복원 (⚠️ 개념 확인용 — 실행 시 대상 재확인)
#   ① 부트코드만 복원, 파티션 테이블은 현재 것 유지
#   dd if=vdb-boot446.bin of=/dev/vdb bs=446 count=1 conv=notrunc
#   ② 512바이트 전체 복원 = 파티션 테이블까지 백업 시점으로 되돌림
#   dd if=vdb-mbr-512.bin of=/dev/vdb bs=512 count=1 conv=notrunc
#   ③ 파티션 테이블 64바이트만 복원
#   dd if=vdb-pt64.bin of=/dev/vdb bs=1 count=64 seek=446 conv=notrunc
```

| 영역 | 오프셋 | 크기 | dd 표현 | 복원 시 영향 |
| --- | --- | --- | --- | --- |
| 부트코드 (bootstrap) | 0 | 446 B | `bs=446 count=1` | 부트로더만 교체, 파티션 정보 보존 |
| 파티션 테이블 | 446 | 64 B (16 B × 4) | `bs=1 count=64 skip=446` | 파티션 구성만 되돌림 |
| 시그니처 | 510 | 2 B (`0x55AA`) | — | 유효 MBR 표식 |
| **MBR 전체** | 0 | **512 B** | `bs=512 count=1` | 부트코드 + 파티션 테이블 모두 |

- 파티션 테이블 엔트리 16 B × 4 = 64 B → **MBR 주 파티션 최대 4개**의 물리적 근거
- `conv=notrunc` 를 빼면 장치가 아닌 **파일**로 복원할 때 뒤가 잘림. 장치 대상에는 영향이 적지만 습관적으로 붙임
- 파티션 테이블 복원 후에는 `partprobe /dev/vdb` 또는 `blockdev --rereadpt /dev/vdb` 로 커널에 재읽기 지시

**검증**

```bash
ls -l /srv/raid/backup/vdb-mbr-512.bin /srv/raid/backup/vdb-boot446.bin /srv/raid/backup/vdb-pt64.bin | awk '{print $5, $9}'
hexdump -C /srv/raid/backup/vdb-mbr-512.bin | tail -2
```

```text
512 /srv/raid/backup/vdb-mbr-512.bin
446 /srv/raid/backup/vdb-boot446.bin
64 /srv/raid/backup/vdb-pt64.bin
000001f0  ... 55 aa
00000200
```

- 끝 2바이트 `55 aa` = MBR 시그니처

> 📝 **시험 포인트**: 필기 FULL r01-55 · r02-63 · r04-61, 실기 r04-2 — `dd if=/dev/sda of=mbr.bak bs=512 count=1`. `if`/`of` 를 뒤집은 보기(`if=mbr.img of=/dev/sda`)는 **복원** 명령이며 백업 문제의 오답. `count=2` 도 오답(1024 B).

### 5-6. 디스크 소거·ISO 쓰기 (※ 개념 — 실행하지 않음)

> **상황**: 폐기·재활용 시 데이터 삭제와, 설치 USB 제작은 dd 의 대표 용도다. 실습 VM 에서 실행하면 복구가 불가능하므로 명령 형태와 차이만 정리한다.

```bash
# ※ 미실행 — 아래는 전부 파괴적 명령. 형태만 확인
# ① 0 으로 덮어쓰기 (1회) — 가장 흔한 소거
#   dd if=/dev/zero of=/dev/vdX bs=4M status=progress
# ② 난수로 덮어쓰기 — 복구 난도 상승, 매우 느림
#   dd if=/dev/urandom of=/dev/vdX bs=4M status=progress
# ③ shred — 다중 덮어쓰기 전용 도구
#   shred -v -n 3 -z /dev/vdX
# ④ wipefs — 파일시스템/파티션 시그니처만 제거 (데이터는 남음)
#   wipefs -a /dev/vdX
# ⑤ ISO → USB 쓰기 (파티션이 아닌 "디스크 전체" 에 씀)
#   dd if=Rocky-9-minimal.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

- `shred -n <횟수>` : 덮어쓰기 반복 횟수 (**n**umber, 기본 3). `-z` 는 마지막에 0 으로 한 번 더 (**z**ero) → 소거 흔적 은폐. `-v` 상세, `-u` 는 파일 삭제까지
- `wipefs -a <장치>` : 모든(**a**ll) 시그니처 삭제 — "포맷된 적 없는 디스크" 로 보이게 함. 파티션 재생성 전 잔여 시그니처 충돌 해결에 사용
- SSD 는 웨어 레벨링 때문에 덮어쓰기가 물리 셀 전체에 닿지 않음 → 제조사 **Secure Erase**(`hdparm --security-erase`) 또는 `blkdiscard` 사용
- ISO 는 하이브리드 이미지라 **파티션(`/dev/sdb1`) 이 아니라 디스크(`/dev/sdb`)** 에 써야 부팅 가능
- 대안 도구: `partclone`(사용 중인 블록만 복사 → 이미지 작음), `Clonezilla`(partclone·partimage 를 묶은 부팅 가능 배포판), `dd` + `conv=sparse`

**검증**

```bash
# 실행하지 않는 대신 도구 존재와 옵션만 확인
which shred wipefs
shred --help | head -8
wipefs /dev/vdb            # 읽기 전용 조회 — 현재 시그니처 목록
```

```text
/usr/bin/shred
/usr/sbin/wipefs
Usage: shred [OPTION]... FILE...
Overwrite the specified FILE(s) repeatedly, in order to make it harder
...
DEVICE OFFSET TYPE UUID LABEL
vdb    0x1fe  dos
```

- 인자 없는 `wipefs <장치>` 는 **조회만** 수행 (`-a`/`-o` 를 줘야 삭제)

> 📝 **시험 포인트**: 데이터 완전 삭제 도구 = `shred`(다중 덮어쓰기). `wipefs` 는 시그니처만 지움. `dd if=/dev/zero` 는 소거, `dd if=/dev/urandom` 은 난수 소거. 저널링 FS·SSD 에서는 덮어쓰기 소거가 불완전하다는 점이 서술형 포인트.

---

## 6. dump/restore 와 xfsdump/xfsrestore

### 6-1. dump 설치와 레벨 0 (전체) 백업 — ext4 `/data`

> **상황**: `dump` 는 파일이 아니라 **파일시스템(파티션) 자체**를 읽는다. `/etc/fstab` 5번 필드와 `/etc/dumpdates` 로 레벨 체계를 관리하는 고전 도구로, 필기 출제 비중이 높다.

```bash
# Rocky 9 기본 저장소에는 없음 → EPEL (Part 02 에서 epel-release 설치)
dnf list --available dump 2>/dev/null | tail -3
dnf install -y dump && rpm -q dump

findmnt /data                                   # ext4 여야 함 (dump 는 ext2/3/4 전용)
grep vdb1 /etc/fstab                            # 5번 필드(dump) = 1 이어야 대상

# 레벨 0 = 전체 백업
dump -0uf /srv/raid/backup/data.dump0 /dev/vdb1
ls -lh /srv/raid/backup/data.dump0
```

- `-0` ~ `-9` : 백업 **레벨**. `0` = 전체, `1~9` = **자신보다 낮은 레벨의 가장 최근 백업 이후** 변경분
- `-u` (`--update`) : 성공 시 `/etc/dumpdates` 에 파일시스템·레벨·시각 기록 (**u**pdate) — 이걸 빼면 다음 증분의 기준점이 생기지 않음
- `-f <파일>` : 출력 대상 (**f**ile). `-` 는 표준출력, 예전에는 테이프 장치 `/dev/st0`
- `-a` : 매체 크기 자동 판단 (**a**uto-size) — 파일로 받을 때 기본 동작
- `-j` / `-z` : bzip2 / zlib 압축 (`-z9` 처럼 레벨 지정 가능)
- `-b <KB>` : 블록 크기 (**b**locksize)
- `-L <라벨>` : 덤프 세션 라벨
- 대상은 **장치(`/dev/vdb1`) 또는 마운트 지점(`/data`)** — 디렉터리 하나만 골라 담는 용도가 아님
- 마운트된 채 덤프하면 경고가 나옴. 정확도를 원하면 언마운트 또는 LVM 스냅샷(7절) 사용

**검증**

```bash
file /srv/raid/backup/data.dump0
cat /etc/dumpdates
restore -tf /srv/raid/backup/data.dump0 | head -8
```

```text
/srv/raid/backup/data.dump0: new-fs dump file (little endian), ... Level 0, ...
/dev/vdb1  0  Wed Sep  3 ... 2026
Dump   date: ...
Dumped from: the epoch
Level  0 dump of /data on srv01.lab.local:/dev/vdb1
Label:  none
         2	.
         3	./proj
       ...
```

- `Dumped from: the epoch` = 기준 시점이 없음 → 전체 백업

> 📝 **시험 포인트**: 필기 FULL r01-61 · r08-63 — "0~9 레벨, `/etc/fstab` dump 필드와 연동, 파일시스템 단위" = `dump`. `/etc/fstab` 5번 필드(필기 FULL r01-33·r03-47·r04-30·r05-33·r06-30·r09-53·r10-29 반복 출제)는 **dump 백업 대상 여부**(0=제외, 1=대상)이며 "백업 주기" 가 아님.

### 6-2. `/etc/dumpdates` 와 레벨 1 증분

> **상황**: 파일을 바꾼 뒤 레벨 1 로 다시 떠서, 증분 크기가 전체보다 훨씬 작은지와 `/etc/dumpdates` 에 줄이 추가되는지 확인한다.

```bash
echo "dump-inc $(date)" > /data/proj/src/dumptest.txt
dd if=/dev/urandom of=/data/proj/inc.bin bs=1M count=3 status=none

dump -1uf /srv/raid/backup/data.dump1 /dev/vdb1
cat /etc/dumpdates
ls -lh /srv/raid/backup/data.dump0 /srv/raid/backup/data.dump1

# 레벨 2 (레벨 1 이후 변경분)
echo "level2" > /data/proj/src/lv2.txt
dump -2uf /srv/raid/backup/data.dump2 /dev/vdb1
cat /etc/dumpdates
```

- `/etc/dumpdates` 형식: `<장치> <레벨> <요일 월 일 시:분:초 연도>` — 레벨별로 한 줄씩 최신 시각 유지
- 레벨 n 은 `/etc/dumpdates` 에서 **n 보다 작은 레벨 중 가장 최근** 항목을 기준점으로 삼음
- 복원 세트 계산: 레벨 0 → 이후 레벨들 중 **단조 증가하는 최소 집합**을 순서대로

| 스케줄 예 | 수요일까지 완전 복원에 필요한 세트 |
| --- | --- |
| 일 0 · 월 3 · 화 2 · 수 5 | **0(일) + 2(화) + 5(수)** — 월(3) 은 화(2) 에 포함되어 불필요 |
| 일 0 · 월 1 · 화 2 · 수 3 | 0 + 1 + 2 + 3 (전부) |
| 일 0 · 월 2 · 화 3 · 수 2 · 목 4 (금 장애) | 0 + 2(수) + 4(목) — 월(2)·화(3) 은 수(2) 에 흡수 |
| 일 0 · 수 3 · 금 5 | 금 5 는 "수 3 이후 변경분" 만 저장 |

**검증**

```bash
cat /etc/dumpdates
restore -tf /srv/raid/backup/data.dump1 | grep -E 'dumptest|inc.bin'
ls -l /srv/raid/backup/data.dump* | awk '{print $5, $9}'
```

```text
/dev/vdb1  0  Wed Sep  3 ... 2026
/dev/vdb1  1  Wed Sep  3 ... 2026
/dev/vdb1  2  Wed Sep  3 ... 2026
    ...	./proj/src/dumptest.txt
    ...	./proj/inc.bin
```

- 레벨 1 파일 크기가 레벨 0 보다 훨씬 작음 = 증분 동작 확인

> 📝 **시험 포인트**: 필기 FULL r02-62 — "일 0, 수 3, 금 5" → 금요일 레벨 5 는 **수요일 레벨 3 백업 이후 변경분**. r07-27 · r10-45 — 레벨 조합에서 복원 세트 고르기. 규칙은 "자신보다 **낮은** 레벨의 최근 백업 이후".

### 6-3. restore — 목록·대화식·개별 파일 복원·원본 비교

> **상황**: 실무 복구의 대부분은 "그 파일 하나" 다. `restore` 의 4가지 모드를 순서대로 써 보고, 대화식 셸에서 파일을 골라 꺼낸다.

```bash
mkdir -p /tmp/dump-restore && cd /tmp/dump-restore

# ① -t : 목록 (아카이브 열지 않고 확인)
restore -tf /srv/raid/backup/data.dump0 | head -10

# ② -x : 특정 경로만 추출 (경로는 목록에 보이는 ./ 로 시작하는 형태)
restore -xf /srv/raid/backup/data.dump0 ./proj/src/file1.txt
find /tmp/dump-restore -type f

# ③ -i : 대화식 — ls / cd / add / delete / extract / quit
restore -if /srv/raid/backup/data.dump0
#   restore > ls
#   restore > cd proj/src
#   restore > add file2.txt file3.txt
#   restore > ls          (선택된 파일 앞에 * 표시)
#   restore > extract     ("specify next volume #: 1", "set owner/mode ...? [yn] n")
#   restore > quit

# ④ -C : 아카이브와 현재 파일시스템 비교 (변경 탐지)
cd / && restore -Cf /srv/raid/backup/data.dump0
```

- `-t` (`--list`) : 아카이브 목록 출력 — inode 번호 + 경로
- `-x` (`--extract`) : 지정 경로만 추출. 경로 생략 시 전체를 **현재 디렉터리** 기준으로 추출
- `-i` (`--interactive`) : 대화식 셸 진입. 서브명령 `ls`(목록) `cd`(이동) `pwd` `add`(복원 목록에 추가) `delete`(제외) `extract`(실행) `verbose` `help` `quit`
- `-C` (`--compare`) : 백업 시점과 현재 파일시스템을 비교해 달라진 파일 보고 — 변조·유실 탐지용
- `-f <파일>` : 입력 아카이브
- `-v` : 상세 출력. `-y` : 오류 시 자동 계속
- 추출은 항상 **현재 작업 디렉터리** 기준 → `cd` 를 먼저 하는 습관이 중요

**검증**

```bash
ls -R /tmp/dump-restore | head
diff /tmp/dump-restore/proj/src/file1.txt /data/proj/src/file1.txt && echo "동일"
cd / && restore -Cf /srv/raid/backup/data.dump0 2>&1 | tail -5
```

```text
/tmp/dump-restore:
proj
/tmp/dump-restore/proj:
src
...
동일
./proj/src/dumptest.txt: (inode ...) not found on tape
Some files were modified!  ... compare errors
```

- 레벨 0 이후 만든 파일이 "not found on tape" 로 잡힘 = `-C` 가 정상 동작

> 📝 **시험 포인트**: `restore -i` 의 `add`→`extract` 흐름은 서술형 단골. `-t` 목록, `-r` 전체 복원, `-x` 개별 추출, `-C` 비교 — 4개 모드 구분.

### 6-4. `restore -r` 전체 복원 리허설

> **상황**: `-r` 는 "빈 파일시스템에 통째로 되살리기" 전용이다. `/data` 를 비우고 레벨 0 → 1 → 2 순서로 복원해 증분 체인이 맞물리는지 확인한다.

⚠️ `/data` 내용을 전부 지운다. `data.dump0/1/2` 존재를 먼저 확인

```bash
ls -lh /srv/raid/backup/data.dump0 /srv/raid/backup/data.dump1 /srv/raid/backup/data.dump2
sha256sum /data/proj/src/*.txt > /srv/raid/backup/dump-before.sha256

# 파일시스템을 비움 (권장은 mkfs 후 mount — 여기서는 내용만 제거)
rm -rf /data/* /data/.??*
ls -a /data

# 반드시 대상 파일시스템의 루트에서 실행
cd /data
restore -rf /srv/raid/backup/data.dump0
restore -rf /srv/raid/backup/data.dump1
restore -rf /srv/raid/backup/data.dump2

ls /data/proj/src | head
ls -l /data/restoresymtable
rm -f /data/restoresymtable          # 체인 완료 후 반드시 삭제
```

- `-r` (`--rebuild`) : 파일시스템 전체 재구축 모드. **비어 있는(갓 mkfs 한) 파일시스템**에서 실행하는 것이 정석
- `restoresymtable` : 증분 체인을 이어가기 위해 restore 가 남기는 임시 심볼 테이블. 다음 레벨 복원에 필요하며 **모든 복원이 끝나면 삭제**
- 복원 순서는 레벨 오름차순 엄수 — 역순이면 옛 내용이 새 내용을 덮어씀
- 정석 절차: `mkfs.ext4 /dev/vdb1` → `mount /data` → `cd /data` → `restore -rf 레벨0` → `restore -rf 레벨1` → … → `rm restoresymtable`

**검증**

```bash
cd / && sha256sum -c /srv/raid/backup/dump-before.sha256 | grep -c ': OK$'
ls /data/proj/src/dumptest.txt /data/proj/src/lv2.txt
ls /data | grep -c restoresymtable
df -h /data | tail -1
```

```text
33
/data/proj/src/dumptest.txt
/data/proj/src/lv2.txt
0
/dev/vdb1  2.0G  ...  /data
```

- 레벨 1 의 `dumptest.txt` 와 레벨 2 의 `lv2.txt` 가 모두 존재 = 증분 체인 복원 성공

> 📝 **시험 포인트**: `restore -r` 는 "전체 복원", 대상은 빈 FS. 복원 후 `restoresymtable` 정리. 필기에서 `dump -0u -f /dev/st0 /dev/sda1` ↔ `restore -rf /dev/st0` 쌍으로 제시됨.

### 6-5. xfsdump — xfs `/srv/share` 레벨 0·증분·인벤토리

> **상황**: `/srv/share` 는 xfs 라 `dump` 가 통하지 않는다. xfs 전용 도구는 세션 라벨·미디어 라벨·자체 인벤토리를 쓰는 점이 다르다.

```bash
rpm -q xfsdump || dnf install -y xfsdump
findmnt /srv/share                              # FSTYPE 이 xfs 인지 확인

# 레벨 0 — -L(세션 라벨), -M(미디어 라벨) 을 주면 비대화식으로 진행
xfsdump -l 0 -L "share-full" -M "raid" -f /srv/raid/backup/share.xfsdump /srv/share

# 변경 후 레벨 1 증분
echo "xfs-inc $(date)" > /srv/share/inc.txt
xfsdump -l 1 -L "share-inc1" -M "raid" -f /srv/raid/backup/share.xfsdump1 /srv/share

# 인벤토리 조회
xfsdump -I
ls -lh /srv/raid/backup/share.xfsdump*
ls /var/lib/xfsdump/inventory
```

- `-l <레벨>` : 백업 레벨 0~9 (**l**evel). dump 와 동일 개념이나 기준 정보는 `/etc/dumpdates` 가 아닌 **자체 인벤토리**(`/var/lib/xfsdump/inventory`)
- `-L <라벨>` : 세션(**L**abel) 이름 — 인벤토리에서 이 백업을 식별
- `-M <라벨>` : 미디어(**M**edia) 라벨 — 매체 식별
- `-f <대상>` : 출력 파일·장치. 여러 개 주면 다중 매체 분할
- `-I` (`--inventory`) : 인벤토리 전체 출력 (파일시스템 UUID·세션·레벨·시각)
- `-e` : 백업 후 inode 의 dump 플래그 갱신, `-J` : 인벤토리에 기록하지 않음
- 대상은 **마운트된 xfs 파일시스템의 마운트 지점** — 하위 디렉터리 하나만 지정할 수 없음(`-s` 로 subtree 지정은 가능)

**검증**

```bash
xfsdump -I | head -20
file /srv/raid/backup/share.xfsdump
ls -l /srv/raid/backup/share.xfsdump /srv/raid/backup/share.xfsdump1 | awk '{print $5, $9}'
```

```text
file system 0:
	fs id:		...
	session 0:
		mount point:	srv01.lab.local:/srv/share
		device:		srv01.lab.local:/dev/mapper/vg_lab-lv_share
		time:		...
		session label:	"share-full"
		level:		0
	session 1:
		session label:	"share-inc1"
		level:		1
/srv/raid/backup/share.xfsdump: data
```

- 증분(`share.xfsdump1`) 이 전체보다 현저히 작음

> 📝 **시험 포인트**: ext 계열 = `dump`/`restore`, xfs = `xfsdump`/`xfsrestore` 로 **도구가 갈린다**는 점이 핵심. RHEL 7 이후 기본 FS 가 xfs 이므로 `dump` 는 기본 제공에서 빠짐.

### 6-6. xfsrestore 복원 검증과 도구 대응표

> **상황**: 덤프 파일을 별도 디렉터리에 풀어 원본과 대조한다. 원본을 건드리지 않는 "스테이징 복원" 이 안전한 검증법이다.

```bash
mkdir -p /srv/raid/restore-xfs

# ① 목록 확인
xfsrestore -t -f /srv/raid/backup/share.xfsdump 2>&1 | head -10

# ② 레벨 0 복원 → 레벨 1 증분 복원 (순서 엄수)
xfsrestore -f /srv/raid/backup/share.xfsdump  /srv/raid/restore-xfs
xfsrestore -f /srv/raid/backup/share.xfsdump1 /srv/raid/restore-xfs

# ③ 대화식 복원 (특정 파일만)
#   xfsrestore -i -f /srv/raid/backup/share.xfsdump /srv/raid/restore-xfs
ls -la /srv/raid/restore-xfs
```

- `-f <파일>` : 입력 덤프. 마지막 인자는 **복원 대상 디렉터리**(xfs 위가 권장 — ACL·xattr 온전 복원)
- `-t` : 내용 목록만 출력 (**t**able of contents)
- `-i` : 대화식 모드 — `dump`/`restore` 의 `-i` 와 유사한 `ls cd add extract quit`
- `-r` : 누적 복원 모드(여러 세션을 차례로 적용)
- `-L <라벨>` : 인벤토리에서 세션 라벨로 선택
- `-S <세션ID>` : 세션 ID 로 선택

| 구분 | ext2/3/4 | xfs |
| --- | --- | --- |
| 전체·증분 백업 | `dump -0uf` | `xfsdump -l 0 -f` |
| 복원 | `restore -rf` / `-xf` / `-if` / `-tf` | `xfsrestore -f` / `-i` / `-t` |
| 레벨 기준 기록 | `/etc/dumpdates` | `/var/lib/xfsdump/inventory` (`xfsdump -I`) |
| fstab 5번 필드 연동 | O (dump 대상 여부) | X (참조하지 않음) |
| 온라인 확장 | `resize2fs` | `xfs_growfs` (축소 불가) |
| 점검 | `fsck.ext4` / `e2fsck` | `xfs_repair` (마운트 해제 필요) |
| 라벨·UUID | `tune2fs -L` / `-U` | `xfs_admin -L` / `-U` |
| 왜 도구가 다른가 | 각 FS 의 **inode·익스텐트 구조를 직접 읽는** 저수준 도구이므로 FS 별 전용 구현 필요 | ← 동일 이유 |

**검증**

```bash
diff -rq /srv/share /srv/raid/restore-xfs; echo "diff=$?"
ls /srv/raid/restore-xfs/inc.txt
getfacl -p /srv/raid/restore-xfs/hl1.txt 2>/dev/null | grep dev1
du -sh /srv/share /srv/raid/restore-xfs
```

```text
diff=0
/srv/raid/restore-xfs/inc.txt
user:dev1:rw-
...M	/srv/share
...M	/srv/raid/restore-xfs
```

- 차이 없음 + ACL 까지 복원 = xfsdump 는 ACL·xattr 를 기본 보존

> 📝 **시험 포인트**: "xfs 파일시스템의 백업/복원 도구는?" → `xfsdump`/`xfsrestore`. `tar`·`rsync` 는 FS 무관 파일 단위, `dd` 는 FS 무관 블록 단위 — 이 3분류를 표로 외울 것.

---

## 7. LVM 스냅샷 기반 일관 백업과 메타데이터 백업

### 7-1. 스냅샷 생성 → 마운트 → 백업 → 해제 (수동 절차)

> **상황**: Samba 가 `/srv/share` 에 쓰고 있는 동안 tar 를 돌리면 "백업 중간에 바뀐 파일" 이 섞인다. CoW 스냅샷으로 시점을 고정한 뒤 그 사본을 백업하면 일관성이 보장된다.

```bash
# ① VG 여유 공간 확인 — 스냅샷 크기만큼 필요 (Part 05 vg_lab)
vgs vg_lab
lvs

# ② 스냅샷 생성
lvcreate -s -L 300M -n lv_share_bk /dev/vg_lab/lv_share
lvs -o lv_name,lv_size,origin,data_percent,lv_attr vg_lab

# ③ 읽기 전용 마운트 (xfs 는 UUID 중복 때문에 nouuid 필수)
mkdir -p /mnt/snap
mount -o ro,nouuid /dev/vg_lab/lv_share_bk /mnt/snap
findmnt /mnt/snap
ls /mnt/snap

# ④ 스냅샷을 원본 삼아 백업
tar czf /srv/raid/backup/share-snap-$(date +%F).tar.gz -C /mnt/snap .

# ⑤ 해제·제거
umount /mnt/snap
lvremove -f /dev/vg_lab/lv_share_bk
lvs
```

- `lvcreate -s` (`--snapshot`) : 스냅샷 LV 생성 (**s**napshot)
- `-L <크기>` : 스냅샷 영역 크기 — **변경분 저장용**이지 원본 전체 크기가 아님 (**L** = size)
- `-n <이름>` : LV 이름 (**n**ame). 마지막 인자는 **원본 LV 경로**
- `-o ro` : 읽기 전용 마운트 — 스냅샷을 실수로 변경하지 않음
- `-o nouuid` : xfs 는 UUID 가 같은 FS 를 동시에 마운트하지 못함 → 스냅샷은 원본과 UUID 가 동일하므로 필수. ext4 는 불필요
- `-C <디렉터리> .` : 아카이브 내부 경로를 `./…` 로 만들어 원위치 복원 시 유연 (2-4 참조)
- `lvremove -f` : 확인 없이 LV 제거 (**f**orce). 스냅샷은 백업 직후 지우는 것이 원칙 — 오래 두면 CoW 부담으로 원본 성능 저하
- `lv_attr` 첫 글자 `s` = 스냅샷, `o` = 원본(origin)

**검증**

```bash
tar tzf /srv/raid/backup/share-snap-$(date +%F).tar.gz | head -5
lvs vg_lab
ls /mnt/snap 2>&1 | head -1
findmnt /mnt/snap; echo "rc=$?"
```

```text
./
./readme.txt
./hl1.txt
./inc.txt
...
  LV       VG     Attr       LSize Pool Origin Data%
  lv_share vg_lab -wi-ao---- 1.00g
rc=1
```

- 스냅샷 LV 가 목록에서 사라지고 `/mnt/snap` 이 비어 있음 = 정리 완료

> 📝 **시험 포인트**: 필기 FULL r09-50 · r10-47 — "`lvcreate -s` 로 특정 시점 상태를 고정한 스냅샷 볼륨을 만들어 **서비스 중단 없이 일관된 백업**". "스냅샷은 생성 시점에 원본 전체를 복사한다" 는 오답(CoW = 변경분만).

### 7-2. 스냅샷 백업 자동화 스크립트

> **상황**: 위 5단계는 실수하기 쉽고 중간에 실패하면 스냅샷이 남아 원본을 갉아먹는다. `trap` 으로 정리를 보장하는 스크립트로 묶는다.

```bash
cat > /usr/local/bin/snap-backup.sh <<'EOF'
#!/bin/bash
# LVM 스냅샷 기반 일관 백업 — /srv/share (xfs on vg_lab/lv_share)
set -euo pipefail

VG=vg_lab
LV=lv_share
SNAP=${LV}_bk
SNAPSIZE=300M
MNT=/mnt/snap
DEST=/srv/raid/backup
STAMP=$(date +%F_%H%M)
OUT="$DEST/share-snap-$STAMP.tar.gz"

cleanup() {
  mountpoint -q "$MNT" && umount "$MNT" || true
  lvs "$VG/$SNAP" &>/dev/null && lvremove -f "$VG/$SNAP" || true
}
trap cleanup EXIT

mkdir -p "$MNT" "$DEST"
lvcreate -s -L "$SNAPSIZE" -n "$SNAP" "/dev/$VG/$LV"
mount -o ro,nouuid "/dev/$VG/$SNAP" "$MNT"
tar czf "$OUT" -C "$MNT" .
sha256sum "$OUT" > "$OUT.sha256"
logger -t snap-backup "OK $OUT ($(du -h "$OUT" | cut -f1))"
echo "created: $OUT"
EOF
chmod 750 /usr/local/bin/snap-backup.sh
bash -n /usr/local/bin/snap-backup.sh && echo "문법 OK"
/usr/local/bin/snap-backup.sh
```

- `set -e` : 명령 실패 시 즉시 종료 / `set -u` : 미정의 변수 사용 시 오류 / `set -o pipefail` : 파이프 중간 실패도 전체 실패로 (Part 04)
- `trap <함수> EXIT` : 정상·비정상 종료 모두에서 실행 → 스냅샷 잔존 방지
- `mountpoint -q <경로>` : 마운트 여부만 조용히 판정 (**q**uiet) — 종료 코드로 분기
- `|| true` : 정리 단계 실패가 `set -e` 로 스크립트를 다시 죽이지 않게 함
- `logger -t <태그>` : syslog 에 기록 (Part 07 rsyslog 연계)
- `bash -n <파일>` : 실행 없이 문법만 검사 (**n**o-exec)

**검증**

```bash
ls -lh /srv/raid/backup/share-snap-*.tar.gz*
lvs vg_lab                                   # 스냅샷이 남아 있지 않아야 함
journalctl -t snap-backup -n 3 --no-pager
cd /srv/raid/backup && sha256sum -c share-snap-*.sha256
tar tzf $(ls -t /srv/raid/backup/share-snap-*.tar.gz | head -1) | wc -l
```

```text
-rw-r--r-- 1 root root ...K ... share-snap-2026-09-03_1420.tar.gz
-rw-r--r-- 1 root root  ... ... share-snap-2026-09-03_1420.tar.gz.sha256
  LV       VG     Attr       LSize ...
  lv_share vg_lab -wi-ao---- 1.00g
... snap-backup[...]: OK /srv/raid/backup/share-snap-... (...K)
share-snap-2026-09-03_1420.tar.gz: OK
...
```

> 📝 **시험 포인트**: 스크립트 작성 서술형에서는 `set -euo pipefail` + `trap … EXIT` 조합이 감점 방지 포인트. 스냅샷은 "백업 직후 제거" 가 정답.

### 7-3. 스냅샷 용량 초과 위험 감시

> **상황**: 스냅샷 영역이 가득 차면 스냅샷은 **무효(invalid)** 가 되어 마운트조차 안 된다. Data% 를 감시하고 필요하면 확장하는 법을 확인한다.

```bash
# 감시용 스냅샷 생성 후 원본을 많이 변경해 Data% 상승 재현
lvcreate -s -L 100M -n lv_share_bk /dev/vg_lab/lv_share
lvs -o lv_name,origin,lv_size,data_percent,lv_attr vg_lab

# 원본에 데이터 기록 → 변경 전 블록이 스냅샷 영역으로 복사됨(CoW)
dd if=/dev/urandom of=/srv/share/churn.bin bs=1M count=60 status=none
sync
lvs -o lv_name,origin,data_percent,lv_attr vg_lab

# 부족하면 확장
lvextend -L +200M /dev/vg_lab/lv_share_bk
lvs vg_lab

rm -f /srv/share/churn.bin
lvremove -f /dev/vg_lab/lv_share_bk
```

- `lvs -o <컬럼>` : 출력 컬럼 지정. `data_percent` = 스냅샷 사용률, `lv_attr` 5번째 글자가 `I` 면 **Invalid**
- `lvextend -L +<크기>` : LV 확장 (`+` = 상대 증가). 파일시스템까지 늘리려면 `-r`(`--resizefs`) 추가 — 스냅샷은 FS 를 늘리지 않으므로 `-r` 불필요
- `dmeventd` / `lvm.conf` 의 `snapshot_autoextend_threshold`·`snapshot_autoextend_percent` 로 자동 확장 설정 가능
- 스냅샷이 무효화되면 데이터 복구 불가 → **원본 변경량 추정 후 여유 있게 잡고, 백업 즉시 제거**가 원칙

**검증**

```bash
lvs -o lv_name,origin,lv_size,data_percent,lv_attr vg_lab
vgs vg_lab
# 감시 한 줄 (임계 80% 경고)
lvs --noheadings -o lv_name,data_percent vg_lab | awk '$2+0>80{print "WARN snapshot", $1, $2"%"}'
```

```text
  LV           VG     Attr       LSize   Origin   Data%
  lv_share     vg_lab owi-aos--- 1.00g
  lv_share_bk  vg_lab swi-a-s--- 100.00m lv_share 61.20
  VG      #PV #LV #SN Attr   VSize  VFree
  vg_lab    1   2   1 wz--n- <2.00g ...
```

- `lv_attr` `s`(snapshot) / `o`(origin) / `S` 또는 `I` (무효) 로 상태 판별

> 📝 **시험 포인트**: 스냅샷 크기는 "원본 크기" 가 아니라 "백업 동안 예상되는 **변경량**". 가득 차면 무효화되어 사용 불가 — 서술형 위험 요소로 자주 물음.

### 7-4. LVM 메타데이터 백업과 복원 지점

> **상황**: LV 를 실수로 지웠을 때 되살리는 근거는 VG 메타데이터다. LVM 은 명령 실행마다 자동으로 이력을 남기지만, 백업 세트에도 명시적으로 포함시킨다.

```bash
# 명시적 메타데이터 백업
vgcfgbackup vg_lab
ls -l /etc/lvm/backup/ /etc/lvm/archive/ | head -20
head -30 /etc/lvm/backup/vg_lab

# 복원 지점(아카이브) 목록
vgcfgrestore -l vg_lab | head -20

# 백업 세트에 포함
cp -a /etc/lvm/backup /srv/raid/backup/lvm-backup
cp -a /etc/lvm/archive /srv/raid/backup/lvm-archive
tar czf /srv/raid/backup/lvm-meta-$(date +%F).tar.gz /etc/lvm/backup /etc/lvm/archive 2>/dev/null
ls -lh /srv/raid/backup/lvm-meta-*.tar.gz

# ⚠️ 참고 — 실제 롤백 (지금은 실행하지 않음)
#   vgcfgrestore -f /etc/lvm/archive/vg_lab_00003-*.vg vg_lab
#   vgchange -ay vg_lab
```

- `vgcfgbackup <VG>` : 현재 VG 메타데이터를 `/etc/lvm/backup/<VG>` 에 저장 (**c**on**f**i**g** **backup**)
- `/etc/lvm/archive/` : LVM 명령을 실행할 때마다 **변경 직전** 상태가 자동 누적 (`<VG>_NNNNN-XXXX.vg`)
- `vgcfgrestore -l <VG>` : 복원 가능한 시점 목록 (**l**ist) — 각 항목에 시각·실행 명령(`description`) 포함
- `vgcfgrestore -f <파일> <VG>` : 그 시점 메타데이터로 복원. ⚠️ **PV 구성이 물리적으로 그대로일 때만** 의미 있음. LV 삭제 직후 롤백에 사용
- `vgchange -ay <VG>` : VG 의 모든 LV 활성화 (**a**vailable **y**es)
- 메타데이터 복원은 "구성표" 복원이지 **데이터 복원이 아님** — 데이터 블록이 이미 덮어써졌으면 소용없음

**검증**

```bash
grep -E 'description|creation_time' /etc/lvm/backup/vg_lab | head -3
grep -c 'segment' /etc/lvm/backup/vg_lab
vgcfgrestore -l vg_lab | grep -c 'File:'
tar tzf /srv/raid/backup/lvm-meta-$(date +%F).tar.gz | head -4
```

```text
description = "Created *after* executing 'lvremove -f /dev/vg_lab/lv_share_bk'"
creation_time = ...
...
etc/lvm/backup/
etc/lvm/backup/vg_lab
etc/lvm/archive/
...
```

> 📝 **시험 포인트**: `vgcfgbackup`(저장) ↔ `vgcfgrestore`(복원), 자동 이력은 `/etc/lvm/archive/`. `pvs`·`vgs`·`lvs` 조회 3종과 함께 묶여 출제.

### 7-5. RAID 메타데이터 백업과 `md127` 대응

> **상황**: 재부팅 후 `/dev/md0` 이 `/dev/md127` 로 잡히면 `/etc/fstab` 의 장치명 항목이 깨진다. `mdadm.conf` 가 없거나 initramfs 에 반영되지 않은 것이 원인이다(Part 05 연계).

```bash
# 현재 배열 상태·구성 스캔
cat /proc/mdstat
mdadm --detail /dev/md0
mdadm --detail --scan

# mdadm.conf 갱신 (중복 방지: 기존 ARRAY 줄 제거 후 추가)
cp -a /etc/mdadm.conf /srv/raid/backup/mdadm.conf.bak 2>/dev/null || true
sed -i '/^ARRAY /d' /etc/mdadm.conf 2>/dev/null || true
mdadm --detail --scan >> /etc/mdadm.conf
cat /etc/mdadm.conf

# initramfs 에 반영 — 이 단계를 빼면 부팅 초기에 md127 로 조립됨
dracut -f
cp -a /etc/mdadm.conf /srv/raid/backup/mdadm.conf
mdadm --examine /dev/vdc /dev/vdd > /srv/raid/backup/mdadm-examine.txt

# ⚠️ 참고 — 배열이 안 잡힐 때 복구 (지금은 조회만)
#   mdadm --assemble --scan            # mdadm.conf/슈퍼블록 기준 자동 조립
#   mdadm --assemble /dev/md0 /dev/vdc /dev/vdd
#   mdadm --stop /dev/md0              # 잘못 잡힌 배열 정지 후 재조립
```

- `mdadm --detail <배열>` : 배열 상세 (레벨·크기·상태·멤버) — `State : clean`, `[UU]` 확인
- `mdadm --detail --scan` : `/etc/mdadm.conf` 에 넣을 `ARRAY` 줄 형식으로 출력 (`ARRAY /dev/md0 metadata=1.2 name=srv01:0 UUID=…`)
- `mdadm --examine <멤버장치>` : 멤버 디스크의 **슈퍼블록** 정보 — 배열 UUID·역할·이벤트 카운트
- `mdadm --assemble --scan` : 설정/슈퍼블록을 근거로 배열 조립. `--stop` 은 조립 해제
- `dracut -f` : initramfs 재생성 (**f**orce) — `mdadm.conf`·`/etc/crypttab`·드라이버 변경 후 필수
- **md127 이 생기는 이유**: 커널이 부팅 중 슈퍼블록만 보고 자동 조립하면 이름을 알 수 없어 **높은 번호부터 역순(127)** 배정. `mdadm.conf` 의 `ARRAY … name=` 또는 커널 파라미터로 이름을 알려주면 `md0` 유지
- fstab 은 장치명 대신 **UUID** 로 쓰는 것이 근본 대책(Part 05)

**검증**

```bash
grep '^ARRAY' /etc/mdadm.conf
lsinitrd | grep -c mdadm.conf
cat /proc/mdstat | head -4
findmnt /srv/raid
blkid /dev/md0
grep md0 /etc/fstab || grep "$(blkid -s UUID -o value /dev/md0)" /etc/fstab
```

```text
ARRAY /dev/md0 metadata=1.2 name=srv01.lab.local:0 UUID=...
1
Personalities : [raid1]
md0 : active raid1 vdd[1] vdc[0]
      ... blocks super 1.2 [2/2] [UU]
TARGET    SOURCE    FSTYPE OPTIONS
/srv/raid /dev/md0  xfs    rw,relatime,...
UUID=... /srv/raid xfs defaults 0 0
```

- `[2/2] [UU]` = 멤버 2개 모두 정상. 재부팅 후 이 값과 `/dev/md0` 이름이 유지되면 대응 성공

> 📝 **시험 포인트**: `mdadm --detail --scan >> /etc/mdadm.conf` 는 실기 단골. RAID 1 은 **가용성** 수단이지 백업이 아니며(1-1), 구성 정보(mdadm.conf)도 백업 대상이라는 점이 서술형 포인트.

---

## 8. Docker 자원 백업 (Part 11 연계)

### 8-1. 볼륨 백업 — 컨테이너를 경유한 tar

> **상황**: `redis-data` 볼륨은 호스트 경로(`/var/lib/docker/volumes/…`) 를 직접 건드리면 안 된다. 볼륨과 백업 대상지를 함께 붙인 임시 컨테이너로 tar 를 떠내는 것이 표준 방법이다.

```bash
docker ps -a
docker volume ls | grep redis-data
docker exec lab-redis redis-cli set labkey "backup-test-$(date +%s)"
docker exec lab-redis redis-cli get labkey
docker exec lab-redis redis-cli save            # RDB 를 디스크로 강제 저장

# 볼륨 → tar (임시 alpine 컨테이너 경유)
docker run --rm \
  -v redis-data:/data:ro \
  -v /srv/raid/backup:/backup \
  alpine tar czf /backup/redis-data.tar.gz -C /data .

ls -lh /srv/raid/backup/redis-data.tar.gz
sha256sum /srv/raid/backup/redis-data.tar.gz >> /srv/raid/backup/backup.sha256
```

- `docker run --rm` : 종료 즉시 컨테이너 삭제 (**rm**) — 백업용 일회성 컨테이너에 필수
- `-v <볼륨명>:<컨테이너경로>` : 명명 볼륨 마운트. `:ro` 를 붙여 읽기 전용으로 (**r**ead-**o**nly)
- `-v <호스트경로>:<컨테이너경로>` : 바인드 마운트 — 산출물을 호스트로 빼내는 통로
- `alpine` : 5 MB 대 경량 이미지 — tar 만 쓰면 충분
- `redis-cli save` : 메모리 상태를 RDB 파일로 즉시 저장 (동기). `bgsave` 는 백그라운드 — 볼륨 백업 직전에 실행해야 데이터 일관성 확보
- 컨테이너를 멈추고 뜨면(`docker stop`) 더 안전하지만, Redis 는 `save` 로 충분

**검증**

```bash
tar tzf /srv/raid/backup/redis-data.tar.gz
docker exec lab-redis redis-cli get labkey
docker volume inspect redis-data --format '{{.Mountpoint}}'
```

```text
./
./dump.rdb
"backup-test-..."
/var/lib/docker/volumes/redis-data/_data
```

> 📝 **시험 포인트**: 볼륨 백업은 "볼륨과 백업 경로를 동시에 마운트한 임시 컨테이너에서 tar" 가 정석. 호스트 경로 직접 조작은 오답 성향.

### 8-2. 볼륨 삭제 후 역방향 복원 검증

> **상황**: 백업 파일이 있어도 복원 절차를 모르면 소용없다. 컨테이너와 볼륨을 지우고 새로 만든 뒤 tar 를 되풀어 키가 되살아나는지 확인한다.

⚠️ `docker rm -f` 와 `docker volume rm` 은 데이터를 즉시 파괴한다. `redis-data.tar.gz` 가 정상인지 먼저 확인

```bash
ls -lh /srv/raid/backup/redis-data.tar.gz && tar tzf /srv/raid/backup/redis-data.tar.gz

# ① 파괴
docker rm -f lab-redis
docker volume rm redis-data
docker volume ls | grep -c redis-data          # 0

# ② 볼륨 재생성 후 역방향 복원
docker volume create redis-data
docker run --rm \
  -v redis-data:/data \
  -v /srv/raid/backup:/backup:ro \
  alpine sh -c 'cd /data && tar xzf /backup/redis-data.tar.gz'

# ③ 컨테이너 재기동 (Part 11 과 동일 옵션)
docker run -d --name lab-redis --restart=unless-stopped \
  -p 6379:6379 -v redis-data:/data redis:7-alpine \
  redis-server --appendonly no --save 60 1

sleep 3
docker ps --filter name=lab-redis --format '{{.Names}}\t{{.Status}}\t{{.Ports}}'
```

- `docker rm -f` : 실행 중이어도 강제 삭제 (**f**orce)
- `docker volume rm <볼륨>` : 볼륨 영구 삭제 — 참조하는 컨테이너가 있으면 거부됨
- `docker volume create <이름>` : 빈 명명 볼륨 생성
- 복원 시 `-C /data` 대신 `sh -c 'cd /data && tar xzf …'` 를 쓴 이유: alpine 의 BusyBox tar 도 `-C` 를 지원하지만, 대상 디렉터리를 명시적으로 이동해 두면 상대 경로(`./`) 아카이브가 확실히 제자리에 풀림
- `--restart=unless-stopped` : 재부팅·데몬 재시작 시 자동 기동, 사용자가 명시적으로 stop 한 경우는 제외 (10-4 검증 항목)

**검증**

```bash
docker exec lab-redis redis-cli get labkey
docker exec lab-redis redis-cli dbsize
docker volume inspect redis-data --format '{{.Name}} {{.Mountpoint}}'
ss -tlnp | grep 6379
docker inspect lab-redis --format '{{.HostConfig.RestartPolicy.Name}}'
```

```text
"backup-test-..."
(integer) 1
redis-data /var/lib/docker/volumes/redis-data/_data
LISTEN 0 4096 0.0.0.0:6379 0.0.0.0:* users:(("docker-proxy",...))
unless-stopped
```

- 삭제 전에 넣었던 `labkey` 값이 그대로 = 볼륨 백업·복원 성공

> 📝 **시험 포인트**: 컨테이너는 일회용, **데이터는 볼륨에** 라는 원칙. 컨테이너 삭제로 데이터가 사라지면 볼륨을 쓰지 않은 구성.

### 8-3. 이미지 `save`/`load` 와 `export` 차이

> **상황**: 레지스트리에 올릴 수 없는 사내 이미지는 파일로 떠서 옮긴다. `save` 와 `export` 는 결과물이 다르므로 구분해 둔다.

```bash
docker images | head
# 이미지 저장 — 레이어·태그·메타데이터 전부 포함
docker save lab/ubuntu-tools:1.0 -o /srv/raid/backup/ubuntu-tools.tar
ls -lh /srv/raid/backup/ubuntu-tools.tar
tar tf /srv/raid/backup/ubuntu-tools.tar | head -5

# 삭제 후 load 로 복원
docker rmi lab/ubuntu-tools:1.0
docker images | grep -c ubuntu-tools           # 0
docker load -i /srv/raid/backup/ubuntu-tools.tar
docker images | grep ubuntu-tools

# 컨테이너 파일시스템 스냅샷 — export (레이어·이력 없음)
docker export lab-redis -o /srv/raid/backup/lab-redis-fs.tar
tar tf /srv/raid/backup/lab-redis-fs.tar | head -5
# 되돌릴 때는 import (새 단일 레이어 이미지가 됨)
#   docker import /srv/raid/backup/lab-redis-fs.tar lab/redis-flat:1.0

# 압축 저장
docker save lab/ubuntu-tools:1.0 | gzip -c > /srv/raid/backup/ubuntu-tools.tar.gz
```

| 명령 | 대상 | 포함 내용 | 복원 명령 | 태그 유지 |
| --- | --- | --- | --- | --- |
| `docker save` | **이미지** | 모든 레이어 + 이력 + 태그 | `docker load` | O |
| `docker export` | **컨테이너** | 현재 파일시스템 1벌 (레이어·이력·볼륨 제외) | `docker import` | X (직접 지정) |
| 볼륨 tar (8-1) | **볼륨 데이터** | 볼륨 내용만 | 볼륨에 tar 풀기 | — |

- `-o <파일>` : 출력 파일 (**o**utput). 생략 시 표준출력 → `| gzip` 파이프 가능
- `-i <파일>` : 입력 파일 (**i**nput). `docker load < file.tar` 도 동일
- `docker export` 는 **볼륨 내용을 포함하지 않음** — 데이터는 8-1 방식으로 따로
- Dockerfile·`compose.yaml` 은 이미지 tar 대신 **git 저장소로 관리**하는 것이 정석 — 재현 가능한 코드가 이미지 파일보다 가치 있음. 이미지 tar 는 "빌드 환경이 없을 때의 이송 수단"

**검증**

```bash
docker images --format '{{.Repository}}:{{.Tag}}\t{{.Size}}' | grep ubuntu-tools
docker history lab/ubuntu-tools:1.0 | head -4
ls -lh /srv/raid/backup/ubuntu-tools.tar /srv/raid/backup/ubuntu-tools.tar.gz /srv/raid/backup/lab-redis-fs.tar
```

```text
lab/ubuntu-tools:1.0	...MB
IMAGE          CREATED        CREATED BY                    SIZE
...
-rw------- 1 root root ...M ... ubuntu-tools.tar
-rw-r--r-- 1 root root ...M ... ubuntu-tools.tar.gz
-rw------- 1 root root ...M ... lab-redis-fs.tar
```

- `docker history` 가 레이어 이력을 보여줌 = `load` 로 복원된 이미지가 원본과 동일 구조

> 📝 **시험 포인트**: `save`↔`load`(이미지), `export`↔`import`(컨테이너 FS) 짝 맞추기가 출제 형태. `commit` 은 컨테이너 → 새 이미지 생성(Part 11).

---

## 9. 백업 자동화 최종판 — `backup.sh` 개정

### 9-1. 스크립트 작성

> **상황**: Part 04 의 `backup.sh` 는 "tar 로 묶어 복사" 수준이었다. 운영에 쓰려면 주간 전체 + 일일 증분, 중복 실행 방지, 로그, 실패 통보, 보존 정책, 체크섬이 모두 필요하다.

```bash
cp -a /usr/local/bin/backup.sh /usr/local/bin/backup.sh.part04 2>/dev/null || true
cat > /usr/local/bin/backup.sh <<'EOF'
#!/bin/bash
#
# /usr/local/bin/backup.sh — LAB 최종판
#   일요일: 전체(레벨 0)  /  그 외: 증분  (tar --listed-incremental)
#   대상지: /srv/raid/backup   로그: /var/log/backup.log + journal(tag=backup)
#   실패 시 ops1 에게 메일, 30일 초과 산출물 정리, 산출물별 sha256
#
set -euo pipefail

SRC_DIRS=(/data /srv/share /etc)
DEST=/srv/raid/backup
SNAR="$DEST/lab.snar"
EXCLUDE="$DEST/rsync-exclude.lst"
LOG=/var/log/backup.log
LOCKFILE=/run/backup.lock
RETAIN_DAYS=30
MAILTO=ops1
STAMP=$(date +%F_%H%M)
DOW=$(date +%u)                 # 1=월 … 7=일
LABEL="init"
TMPD=$(mktemp -d /tmp/backup.XXXXXX)

log() { printf '%s [%s] %s\n' "$(date '+%F %T')" "$$" "$*" >> "$LOG"; logger -t backup -- "$*"; }

cleanup() {
  local rc=$?
  rm -rf "$TMPD"
  if [ "$rc" -eq 0 ]; then
    log "END   ok label=$LABEL rc=0"
  elif [ "$rc" -eq 3 ]; then
    log "SKIP  another instance is running"
  else
    log "FAIL  label=$LABEL rc=$rc"
    {
      echo "host   : $(hostname)"
      echo "time   : $(date)"
      echo "label  : $LABEL"
      echo "rc     : $rc"
      echo "--- last log ---"
      tail -20 "$LOG"
    } | mail -s "BACKUP FAIL $(hostname) $STAMP" "$MAILTO" || log "WARN  mail send failed"
  fi
  exit "$rc"
}
trap cleanup EXIT

# --- 배타 실행 (중복 기동 방지) ---
exec 9>"$LOCKFILE"
flock -n 9 || exit 3

log "START stamp=$STAMP dow=$DOW"

# --- 대상 점검 ---
mkdir -p "$DEST"
for d in "${SRC_DIRS[@]}"; do
  [ -d "$d" ] || { log "ERROR source missing: $d"; exit 4; }
done
[ -w "$DEST" ] || { log "ERROR dest not writable: $DEST"; exit 5; }

# --- 레벨 결정 ---
if [ "$DOW" -eq 7 ] || [ ! -f "$SNAR" ]; then
  LABEL=full
  rm -f "$SNAR"
else
  LABEL=inc
fi
OUT="$DEST/lab-$LABEL-$STAMP.tar.gz"

# --- 아카이브 생성 (tar 종료코드 1 = 읽는 중 변경 경고 → 허용) ---
set +e
tar --create --gzip --file="$OUT" \
    --listed-incremental="$SNAR" \
    ${EXCLUDE:+--exclude-from="$EXCLUDE"} \
    --exclude="$DEST" --exclude='/proc' --exclude='/sys' \
    --acls --selinux --xattrs \
    "${SRC_DIRS[@]}" 2> "$TMPD/tar.err"
trc=$?
set -e
if [ "$trc" -gt 1 ]; then
  log "ERROR tar rc=$trc : $(tail -3 "$TMPD/tar.err" | tr '\n' ' ')"
  exit 6
fi
[ "$trc" -eq 1 ] && log "WARN  tar rc=1 (file changed while reading)"

# --- 체크섬 ---
( cd "$DEST" && sha256sum "$(basename "$OUT")" > "$(basename "$OUT").sha256" )

# --- 보존 정책 ---
DELETED=$(find "$DEST" -maxdepth 1 -type f \( -name 'lab-*.tar.gz' -o -name 'lab-*.sha256' \) \
          -mtime +"$RETAIN_DAYS" -print -delete | wc -l)

log "OK    $OUT size=$(du -h "$OUT" | cut -f1) purged=$DELETED"
exit 0
EOF

chmod 750 /usr/local/bin/backup.sh
bash -n /usr/local/bin/backup.sh && echo "문법 OK"
touch /var/log/backup.log && chmod 600 /var/log/backup.log
```

- `exec 9>"$LOCKFILE"` : 파일 디스크립터 9번을 잠금 파일에 연결 (스크립트 종료 시 자동 해제)
- `flock -n 9` : FD 9 에 배타 잠금 시도, 이미 잠겨 있으면 즉시 실패 (**n**onblock). `-w <초>` 는 대기 후 포기
- `trap cleanup EXIT` : 정상·오류·`exit` 모두에서 실행 → 임시 디렉터리 정리 + 종료 코드별 분기
- `mktemp -d <템플릿>` : 임시 **d**irectory 생성. `XXXXXX` 가 난수로 치환
- `${EXCLUDE:+--exclude-from="$EXCLUDE"}` : 변수가 비어 있지 않을 때만 옵션 삽입 (파라미터 확장)
- `date +%u` : ISO 요일 번호 1(월)~7(일). `%w` 는 0(일)~6(토) — 혼동 주의
- `find … -mtime +30 -print -delete` : 30일 **초과** 파일 출력 후 삭제. `-mtime +N` = N일 이전
- `mail -s "<제목>" <수신자>` : 표준입력을 본문으로 메일 발송 (`s-nail` 제공, Part 09)
- 종료 코드 규약: `0` 성공 / `3` 중복 실행 / `4` 원본 없음 / `5` 대상지 쓰기 불가 / `6` tar 실패

**검증**

```bash
ls -l /usr/local/bin/backup.sh
bash -n /usr/local/bin/backup.sh; echo "syntax rc=$?"
grep -c 'flock\|trap\|sha256sum\|mtime\|logger\|mail -s' /usr/local/bin/backup.sh
shellcheck /usr/local/bin/backup.sh 2>/dev/null | head -5 || echo "(shellcheck 미설치 — 선택)"
```

```text
-rwxr-x--- 1 root root ... /usr/local/bin/backup.sh
syntax rc=0
6
```

> 📝 **시험 포인트**: 스크립트 서술형에서 `set -euo pipefail`·`trap`·`flock`·종료 코드·로그·통보는 감점 방지 6요소. cron 실행 스크립트는 **절대 경로**와 PATH 를 반드시 명시.

### 9-2. 수동 실행과 산출물 검증

> **상황**: cron 에 맡기기 전에 손으로 두 번 돌려 전체 → 증분 흐름과 산출물·로그·체크섬을 확인한다.

```bash
# ① 첫 실행 — snar 가 없으므로 full
/usr/local/bin/backup.sh; echo "rc=$?"
ls -lh /srv/raid/backup/lab-*.tar.gz*

# ② 파일 변경 후 두 번째 실행 — 평일이면 inc
echo "auto-test $(date)" > /data/proj/src/auto.txt
/usr/local/bin/backup.sh; echo "rc=$?"
ls -lh /srv/raid/backup/lab-*.tar.gz

# ③ 로그·저널
tail -6 /var/log/backup.log
journalctl -t backup -n 8 --no-pager

# ④ 체크섬 검증
cd /srv/raid/backup && sha256sum -c lab-*.sha256
```

**검증**

```bash
ls /srv/raid/backup/ | grep -E '^lab-(full|inc)-' | sort
tar tzf $(ls -t /srv/raid/backup/lab-inc-*.tar.gz | head -1) | grep auto.txt
file /srv/raid/backup/lab.snar
grep -E 'START|OK|END' /var/log/backup.log | tail -6
```

```text
lab-full-2026-09-03_1500.tar.gz
lab-full-2026-09-03_1500.tar.gz.sha256
lab-inc-2026-09-03_1503.tar.gz
lab-inc-2026-09-03_1503.tar.gz.sha256
data/proj/src/auto.txt
/srv/raid/backup/lab.snar: GNU tar incremental snapshot data, version 2
2026-09-03 15:00:11 [1234] START stamp=2026-09-03_1500 dow=4
2026-09-03 15:00:31 [1234] OK    /srv/raid/backup/lab-full-... size=...M purged=0
2026-09-03 15:00:31 [1234] END   ok label=full rc=0
2026-09-03 15:03:02 [1301] START stamp=2026-09-03_1503 dow=4
2026-09-03 15:03:05 [1301] OK    /srv/raid/backup/lab-inc-... size=...K purged=0
2026-09-03 15:03:05 [1301] END   ok label=inc rc=0
```

- 두 번째 산출물이 KB 단위 = 증분 동작. 새 파일 `auto.txt` 만 들어 있음

> 📝 **시험 포인트**: 같은 snar 재사용 = 증분. cron 자동화 검증은 "산출물 존재 → 내용 확인 → 로그 확인" 3단.

### 9-3. 배타 실행·실패 경로 검증

> **상황**: 이중 기동과 실패 통보는 "일부러 깨뜨려" 봐야 확인된다. 잠금이 실제로 두 번째 프로세스를 막는지, 실패 시 메일이 오는지 검사한다.

```bash
# ① 중복 실행 방지 — 첫 번째를 백그라운드로 돌린 직후 두 번째 시도
/usr/local/bin/backup.sh & sleep 0.3
/usr/local/bin/backup.sh; echo "second rc=$?"      # 3 이어야 정상
wait
grep 'SKIP' /var/log/backup.log | tail -2

# ② 실패 경로 — 원본 경로가 없는 상태 재현
mv /srv/raid/backup/rsync-exclude.lst /tmp/ 2>/dev/null || true
sed -i 's#^SRC_DIRS=.*#SRC_DIRS=(/data /srv/share /etc /nonexistent)#' /usr/local/bin/backup.sh
/usr/local/bin/backup.sh; echo "fail rc=$?"        # 4 이어야 정상
tail -3 /var/log/backup.log

# ③ 메일 수신 확인 (ops1)
su - ops1 -c 'mail -H' 2>/dev/null | head -5
ls -l /var/spool/mail/ops1
grep -c 'BACKUP FAIL' /var/spool/mail/ops1

# ④ 원상 복구
sed -i 's#^SRC_DIRS=.*#SRC_DIRS=(/data /srv/share /etc)#' /usr/local/bin/backup.sh
mv /tmp/rsync-exclude.lst /srv/raid/backup/ 2>/dev/null || true
/usr/local/bin/backup.sh; echo "restored rc=$?"
```

- `& sleep 0.3` : 첫 프로세스가 잠금을 잡을 시간을 준 뒤 두 번째 실행
- `wait` : 백그라운드 잡 종료까지 대기 (Part 06 잡 제어)
- `mail -H` : 메일함 헤더 목록만 표시 (**H**eader). `s-nail` 기준
- `/var/spool/mail/<user>` : mbox 형식 로컬 메일함 (Part 09 Postfix 로컬 배송)
- 종료 코드 확인은 `echo "rc=$?"` — cron 은 0 이 아닌 종료 코드를 MAILTO 로 알림

**검증**

```bash
grep -E 'SKIP|FAIL' /var/log/backup.log | tail -4
journalctl -t backup --since '10 min ago' --no-pager | grep -E 'SKIP|FAIL'
grep -A5 'Subject: BACKUP FAIL' /var/spool/mail/ops1 | head -10
ls -lh /srv/raid/backup/lab-*.tar.gz | tail -2
```

```text
2026-09-03 15:10:02 [1450] SKIP  another instance is running
2026-09-03 15:12:01 [1470] ERROR source missing: /nonexistent
2026-09-03 15:12:01 [1470] FAIL  label=init rc=4
Subject: BACKUP FAIL srv01.lab.local 2026-09-03_1512
host   : srv01.lab.local
label  : init
rc     : 4
```

> 📝 **시험 포인트**: `flock` 은 "cron 작업이 겹쳐 도는 것" 을 막는 표준 수단. 실패 통보는 cron 의 `MAILTO=` 변수 또는 스크립트 내부 `mail` 두 갈래 — 둘 다 답이 될 수 있음.

### 9-4. 스케줄·로그로테이트 연동 확인

> **상황**: 스크립트가 준비됐으면 스케줄과 로그 관리가 붙어야 완성이다. Part 06 의 cron 항목과 Part 07 의 logrotate 항목을 이 스크립트 기준으로 점검한다.

```bash
# ① cron.d 항목 (Part 06 에서 생성)
cat /etc/cron.d/lab-backup
ls -l /etc/cron.d/lab-backup                    # 권한 644, 소유 root:root
systemctl is-active crond

# 없거나 갱신이 필요하면
cat > /etc/cron.d/lab-backup <<'EOF'
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MAILTO=ops1
# 분 시 일 월 요일  사용자  명령
30 3 * * *  root  /usr/local/bin/backup.sh
EOF
chmod 644 /etc/cron.d/lab-backup

# ② systemd 타이머 표기로 스케줄 해석·검증
systemd-analyze calendar '*-*-* 03:30:00'
systemd-analyze calendar 'Sun *-*-* 03:30:00'
systemctl list-timers --all --no-pager | head -8

# ③ logrotate 항목 (Part 07)
cat > /etc/logrotate.d/backup <<'EOF'
/var/log/backup.log {
    weekly
    rotate 8
    compress
    delaycompress
    missingok
    notifempty
    create 0600 root root
}
EOF
logrotate -d /etc/logrotate.d/backup 2>&1 | head -12
```

- `/etc/cron.d/<파일>` : 시스템 cron 조각. **사용자 필드가 추가로 들어감**(6번째) — 사용자 crontab 과의 결정적 차이
- `MAILTO=` : 해당 파일의 작업 출력·실패를 받을 계정. 빈 값(`MAILTO=""`)이면 메일 안 보냄
- `PATH=` 명시 : cron 환경은 로그인 셸과 달라 PATH 가 짧음 → 스크립트 실패 최다 원인
- `systemd-analyze calendar '<식>'` : OnCalendar 표기의 정규화 결과와 **다음 실행 시각**을 계산 — 스케줄 표기 검증에 유용
- `systemctl list-timers --all` : 등록된 타이머의 NEXT·LEFT·LAST·UNIT
- logrotate 지시자: `weekly`(주기) `rotate 8`(보관 세대) `compress`(gzip) `delaycompress`(한 세대 늦춰 압축) `missingok`(없어도 오류 아님) `notifempty`(비었으면 회전 안 함) `create <권한> <소유자> <그룹>`(회전 후 새 파일 생성)
- `logrotate -d` : dry-run (**d**ebug) — 실제 회전 없이 계획만 출력

**검증**

```bash
grep -v '^#' /etc/cron.d/lab-backup | grep backup.sh
systemd-analyze calendar 'Sun *-*-* 03:30:00' | grep -E 'Normalized|Next elapse'
journalctl -u crond --since today --no-pager | tail -3
logrotate -d /etc/logrotate.d/backup 2>&1 | grep -E 'considering|log needs rotating|rotating'
run-parts --test /etc/cron.daily | head -3
```

```text
30 3 * * *  root  /usr/local/bin/backup.sh
  Normalized form: Sun *-*-* 03:30:00
    Next elapse: Sun 2026-09-06 03:30:00 KST
considering log /var/log/backup.log
  log does not need rotating (log has been already rotated)
```

> 📝 **시험 포인트**: cron 필드 순서 `분 시 일 월 요일`. `/etc/crontab`·`/etc/cron.d/*` 는 **사용자 필드 포함 6필드**, `crontab -e` 는 5필드. `30 3 * * *` = 매일 03:30.

### 9-5. 복구 런북 템플릿

> **상황**: 장애 순간에 절차를 떠올릴 수는 없다. 이 LAB 에서 실제로 검증한 복구 경로만 모아 런북 표로 남긴다. 각 행의 "복구 절차" 는 앞 절의 단계 번호와 대응한다.

```bash
mkdir -p /srv/raid/backup/runbook
cp -a /srv/raid/backup/{blkid.txt,lsblk.txt,fstab.bak,vda-sfdisk.txt,mdadm.conf} /srv/raid/backup/runbook/ 2>/dev/null || true
ls /srv/raid/backup/runbook
```

| 장애 유형 | 판단 근거 (진단 명령) | 복구 절차 | 검증 방법 | 예상 소요 |
| --- | --- | --- | --- | --- |
| 파일 실수 삭제 (`/data`) | 사용자 신고, `ls`, `find -newer` | 최신 `lab-full` → `lab-inc` 순서 추출 (2-6) 또는 `snapshots/genN` 에서 복사 (4-6) | `sha256sum -c`, `diff -rq` | 5~15분 |
| 설정 파일 오편집 (`/etc/*`) | 서비스 기동 실패, `sshd -t`·`httpd -t`·`named-checkconf` | `etc-latest.tar.gz` 에서 해당 파일만 추출 → `cp -a` → 문법 검사 → 재시작 (3-1) | `systemctl is-active`, 문법 검사 통과 | 5분 |
| 파일시스템 손상 (`/data`) | 마운트 실패, `dmesg`, `fsck -n` | `umount` → `fsck.ext4 -y` 시도 → 실패 시 `vdb1.img.gz` 로 dd 복원 (5-4) | `mount` 후 `sha256sum -c`, `repquota` | 20~40분 |
| 파티션 테이블 손실 | `lsblk` 에 파티션 없음, `fdisk -l` 오류 | `sfdisk /dev/vdX < vdX-sfdisk.txt` 또는 `sgdisk --load-backup` → `partprobe` (5-1) | `lsblk -f` 가 `blkid.txt` 와 일치 | 5분 |
| MBR/부트로더 손상 | 부팅 불가, GRUB 프롬프트 | 레스큐 부팅 → `chroot` → `grub2-install` + `grub2-mkconfig` (Part 07 10절), 또는 `dd if=mbr.bin of=/dev/vdX bs=512 count=1 conv=notrunc` (5-5) | 재부팅 후 정상 진입 | 20분 |
| RAID 멤버 1개 고장 | `/proc/mdstat` `[U_]`, `mdadm --detail` | 교체 후 `mdadm --manage /dev/md0 --add /dev/vdX` → 리빌드 대기 (Part 05) | `cat /proc/mdstat` `[UU]` | 리빌드 시간 의존 |
| RAID 가 `md127` 로 조립 | 부팅 후 `/srv/raid` 미마운트, `lsblk` | `mdadm --detail --scan >> /etc/mdadm.conf` → `dracut -f` → 재부팅 (7-5) | `cat /proc/mdstat` 에 `md0` | 10분 + 재부팅 |
| LV 실수 삭제 | `lvs` 에 없음, `vgcfgrestore -l` 이력 | `vgcfgrestore -f <archive> vg_lab` → `vgchange -ay` (7-4) | `lvs`, `mount`, 데이터 확인 | 10분 |
| Docker 볼륨 유실 | `docker volume ls`, 컨테이너 기동 실패 | `docker volume create` → tar 역방향 복원 → 컨테이너 재기동 (8-2) | `redis-cli get labkey` | 5분 |
| root 비밀번호 분실 | 로그인 거부 | GRUB `e` → `rd.break` → `chroot` → `passwd` → `/.autorelabel` (Part 07 8절) | 새 비밀번호 로그인, `getenforce` | 10분 + 재부팅 |
| `/etc/fstab` 오타로 부팅 실패 | emergency 모드 진입 | `mount -o remount,rw /` → fstab 수정 → `systemctl default` (Part 07 9절) | `systemctl --failed` 0건 | 10분 |
| 시스템 전손 (VM 유실) | VM 부팅 불가 | 새 VM 설치 → `pkglist.txt` 로 패키지 복원 → `etc-*.tar.gz`·`accounts/*` 복원 → 데이터 아카이브 복원 → `sfdisk`·`mdadm.conf`·LVM 메타 복원 | `final-check.sh` 전 항목 `[OK]` | 2~4시간 |

- 런북 갱신 규칙: 새 자원(서비스·볼륨·디스크) 추가 시 **백업 대상 목록(1-2)** 과 **이 표**를 동시에 갱신
- RTO 합의는 "예상 소요" 열이 근거 — 4시간을 넘길 수 없다면 이미지 백업 주기를 늘리거나 대기 VM 을 준비

**검증**

```bash
ls -l /srv/raid/backup/runbook/
du -sh /srv/raid/backup
find /srv/raid/backup -maxdepth 1 -name '*.sha256' | wc -l
df -h /srv/raid | tail -1
```

```text
-rw-r--r-- 1 root root ... blkid.txt
-rw-r--r-- 1 root root ... fstab.bak
-rw-r--r-- 1 root root ... lsblk.txt
-rw-r--r-- 1 root root ... mdadm.conf
-rw-r--r-- 1 root root ... vda-sfdisk.txt
...M	/srv/raid/backup
...
```

> 📝 **시험 포인트**: RPO(허용 손실 시간=백업 주기) vs RTO(허용 복구 시간). "장애 유형별 복구 절차 서술" 문항에서는 **진단 → 격리 → 복원 → 검증** 4단계 구조로 답하는 것이 안전.

---

## 10. 시스템 종료·재시작 총정리

### 10-1. `shutdown` 예약·통지·취소

> **상황**: 운영 중인 서버를 끄기 전에는 접속자에게 알리고 새 로그인을 막아야 한다. 예약 → 통지 확인 → 취소를 한 번에 실습한다(실제로 끄지는 않는다).

⚠️ `shutdown -h now` 는 즉시 종료된다. 아래 예약 명령은 반드시 `-c` 로 취소할 것

```bash
# 터미널 A (root)
shutdown -r +1 "정기 점검을 위한 재부팅 예정"
# 즉시 확인
ls -l /run/nologin && cat /run/nologin
systemctl status systemd-shutdownd 2>/dev/null | head -3
journalctl -n 5 --no-pager | grep -i shutdown

# 터미널 B (다른 세션 — SSH 또는 UTM 콘솔, 일반 사용자로 접속)
#   → 브로드캐스트 메시지가 화면에 뜸
#   Broadcast message from root@srv01.lab.local (Wed 2026-09-03 15:40:00 KST):
#   정기 점검을 위한 재부팅 예정
#   The system is going down for reboot at Wed 2026-09-03 15:41:00 KST!
#   → 이 상태에서 일반 사용자 로그인 시도 시 /run/nologin 내용이 출력되며 거부

# 터미널 A — 취소
shutdown -c
ls /run/nologin 2>&1 | grep -c 'No such'

# 임의 메시지 브로드캐스트
wall "테스트 공지 — 실습 중"
echo "파일에서 읽어 방송" | wall
```

- `shutdown [옵션] <시각> [메시지]` : 예약 종료. 시각은 `now`, `+<분>`, `hh:mm` 형식
- `-r` (`--reboot`) : 종료 후 재부팅
- `-h` : 종료(전원 차단). systemd 에서는 `--poweroff` 와 동일하게 동작(`-H` 를 함께 주면 halt)
- `-P` (`--poweroff`) : 전원 차단 명시
- `-H` (`--halt`) : CPU 정지만, 전원은 유지 (물리 서버에서 전원 버튼 대기 상태)
- `-c` (`--cancel`) : 예약된 종료 취소 — 취소 시에도 메시지 전달 가능
- `-k` : **실제로 끄지 않고 경고 메시지만** 브로드캐스트 (**k**idding) — 훈련·공지용
- `--no-wall` : 브로드캐스트 없이 진행
- `/run/nologin` : 종료 5분 전(또는 예약 즉시) systemd 가 생성. 존재하는 동안 **root 를 제외한 로그인 거부**, 파일 내용이 거부 사유로 출력됨. 취소하면 자동 삭제
- `wall <메시지>` : 로그인한 모든 터미널에 방송 (**w**rite **all**). 표준입력도 받음

**검증**

```bash
shutdown -k +5 "테스트 공지 (실제 종료 안 함)" ; sleep 2 ; shutdown -c
ls /run/nologin 2>&1 | tail -1
who
last -x | head -5
runlevel; who -r
```

```text
ls: cannot access '/run/nologin': No such file or directory
root     pts/0        ... (192.168.64.1)
admin1   pts/1        ... (192.168.64.1)
reboot   system boot  5.14.0-... ...   still running
runlevel (to lvl 3)   5.14.0-...       ...
N 3
         run-level 3  ...
```

- `last -x` 의 `runlevel`·`reboot`·`shutdown` 항목은 `/var/log/wtmp` 에 기록된 시스템 상태 전이 (**x** = 시스템 종료·런레벨 항목 포함)

> 📝 **시험 포인트**: `shutdown -r +1 "msg"` = 1분 뒤 재부팅 + 메시지. `shutdown -c` 취소. `-k` 는 "경고만". 필기 FULL r10-7 유형 — `shutdown -r +3` 는 "3분 후 재부팅"(3초 아님).

### 10-2. 종료·재부팅 명령 대응표

> **상황**: RHEL 9 에서 `shutdown`·`halt`·`poweroff`·`reboot`·`init`·`telinit` 은 모두 `systemctl` 로 이어지는 심볼릭 링크다. 어느 것을 써도 되지만 시험은 옵션과 런레벨 대응을 묻는다.

```bash
ls -l /usr/sbin/shutdown /usr/sbin/halt /usr/sbin/poweroff /usr/sbin/reboot /usr/sbin/init /usr/sbin/telinit
ls -l /usr/lib/systemd/system/runlevel[0-6].target
systemctl get-default
systemd-analyze critical-chain --no-pager | head -5
# sync 습관 — 버퍼를 디스크로 밀어냄 (systemd 는 종료 시 자동 수행하나 관례)
sync; sync
```

| 명령 | 동작 | 대응 타겟/런레벨 | 비고 |
| --- | --- | --- | --- |
| `shutdown -h now` / `-P now` | 즉시 전원 차단 | `poweroff.target` / 0 | 예약·메시지·취소 지원 |
| `shutdown -r now` | 즉시 재부팅 | `reboot.target` / 6 | |
| `shutdown -H now` | CPU 정지(전원 유지) | `halt.target` | |
| `shutdown -c` | 예약 취소 | — | 예약이 없으면 오류 |
| `halt` | 시스템 정지 | `halt.target` | `-p` 주면 전원까지 차단 |
| `poweroff` | 전원 차단 | `poweroff.target` | |
| `reboot` | 재부팅 | `reboot.target` | `--force` = 서비스 정상 종료 생략(1회), `-ff` = 즉시 강제 |
| `init 0` | 종료 | 런레벨 0 | systemd 하위 호환 |
| `init 6` | 재부팅 | 런레벨 6 | |
| `init 1` / `init s` | 단일 사용자 모드 | `rescue.target` / 1 | |
| `telinit <n>` | 런레벨 전환 요청 | 동일 | `init` 과 사실상 동일 링크 |
| `systemctl poweroff` | 전원 차단 | `poweroff.target` | 권장 표기 |
| `systemctl reboot` | 재부팅 | `reboot.target` | |
| `systemctl halt` | 정지 | `halt.target` | |
| `systemctl kexec` | 펌웨어 초기화를 건너뛴 **빠른 재부팅** | `kexec.target` | `kexec-tools` + 커널 로드 필요 |
| `systemctl suspend` / `hibernate` | 절전 / 최대 절전 | — | 서버에서는 거의 미사용 |
| `systemctl isolate multi-user.target` | 실행 중 타겟 전환 | 런레벨 3 | 종료 없이 상태 변경 (Part 07) |
| `sync` | 버퍼 캐시를 디스크로 기록 | — | 과거 `sync; sync; halt` 관례 |

- `reboot -f` (`--force`) : systemd 를 거치지 않고 곧바로 재부팅 — **파일시스템 손상 위험**, 응답 없는 시스템에만
- `-n` (`--no-sync`) : sync 생략. 정상 상황에서는 쓰지 않음
- `-w` (`--wtmp-only`) : 실제 동작 없이 `/var/log/wtmp` 에만 기록
- 런레벨 2·3·4 는 RHEL 계열에서 모두 `multi-user.target` 으로 수렴, 5 = `graphical.target`

**검증**

```bash
readlink -f /usr/sbin/shutdown /usr/sbin/init /usr/sbin/reboot
readlink -f /usr/lib/systemd/system/runlevel0.target /usr/lib/systemd/system/runlevel6.target
systemctl list-units --type=target --no-pager | head -8
```

```text
/usr/bin/systemctl
/usr/bin/systemctl
/usr/bin/systemctl
/usr/lib/systemd/system/poweroff.target
/usr/lib/systemd/system/reboot.target
```

> 📝 **시험 포인트**: 런레벨 0=종료, 6=재부팅, 1=단일 사용자 — `init 0`/`init 6` 는 반드시 암기. 필기 FULL r01-7·r02-6·r03-18·r04-7·r05-7·r07-15 반복 출제. `halt` 는 전원 차단이 아님(`-p` 필요)이 함정.

### 10-3. 전 자원 자동 복원 검증 스크립트 `final-check.sh`

> **상황**: 재부팅 후 12개 파트에서 만든 자원이 전부 자동으로 돌아왔는지 사람이 눈으로 확인하면 반드시 빠뜨린다. `[OK]`/`[NG]` 한 줄씩 찍는 점검 스크립트로 자동화한다.

```bash
cat > /usr/local/bin/final-check.sh <<'EOF'
#!/bin/bash
# 재부팅 후 전 자원 복원 점검 — LAB Part 01~12
NG=0
chk() {  # chk "<항목>" "<명령>"
  if eval "$2" &>/dev/null; then
    printf '[OK] %s\n' "$1"
  else
    printf '[NG] %s   ← %s\n' "$1" "$2"; NG=$((NG+1))
  fi
}

echo "===== 1. 마운트 (Part 05) ====="
chk "/data (ext4, vdb1)"        "findmnt -n /data"
chk "/srv/raid (xfs, md0)"      "findmnt -n /srv/raid"
chk "/srv/share (xfs, LVM)"     "findmnt -n /srv/share"
chk "fstab 문법"                "findmnt --verify --verbose | grep -q 'Success'"

echo "===== 2. 스왑 2종 (Part 05) ====="
chk "스왑 파티션 vdb2"          "swapon --show=NAME --noheadings | grep -q /dev/vdb2"
chk "스왑 파일 /swapfile"       "swapon --show=NAME --noheadings | grep -q /swapfile"
chk "총 스왑 > 0"               "[ \$(free -m | awk '/Swap/{print \$2}') -gt 0 ]"

echo "===== 3. RAID / LVM (Part 05) ====="
chk "md0 존재"                  "test -b /dev/md0"
chk "md0 [UU] 정상"             "grep -q '\[UU\]' /proc/mdstat"
chk "VG vg_lab"                 "vgs --noheadings -o vg_name | grep -q vg_lab"
chk "LV lv_share 활성"          "lvs --noheadings -o lv_name,lv_attr vg_lab | grep lv_share | grep -q 'a'"
chk "스냅샷 잔존 없음"          "! lvs --noheadings -o lv_attr vg_lab | grep -q '^  s'"

echo "===== 4. 쿼터 (Part 05) ====="
chk "/data 쿼터 옵션"           "findmnt -no OPTIONS /data | grep -q quota"
chk "쿼터 집계 동작"            "repquota /data"

echo "===== 5. 서비스 (Part 07~11) ====="
for s in sshd chronyd crond httpd named nfs-server smb vsftpd postfix docker; do
  chk "서비스 $s active"        "systemctl is-active --quiet $s"
done
chk "실패 유닛 0건"             "[ \$(systemctl --failed --no-legend | wc -l) -eq 0 ]"

echo "===== 6. 포트 (Part 08~11) ====="
chk "sshd 2222 LISTEN"          "ss -tln | grep -q ':2222 '"
chk "httpd 80 LISTEN"           "ss -tln | grep -q ':80 '"
chk "named 53 LISTEN"           "ss -tuln | grep -q ':53 '"
chk "smb 445 LISTEN"            "ss -tln | grep -q ':445 '"
chk "redis 6379 LISTEN"         "ss -tln | grep -q ':6379 '"

echo "===== 7. 컨테이너 (Part 11) ====="
chk "lab-redis running"         "docker ps --format '{{.Names}}' | grep -q lab-redis"
chk "restart 정책 복귀"         "docker inspect lab-redis --format '{{.HostConfig.RestartPolicy.Name}}' | grep -qE 'always|unless-stopped'"
chk "volume redis-data"         "docker volume ls -q | grep -q redis-data"
chk "redis 데이터 생존"         "docker exec lab-redis redis-cli ping | grep -q PONG"

echo "===== 8. 보안 (Part 10) ====="
chk "SELinux Enforcing"         "getenforce | grep -q Enforcing"
chk "firewalld active"          "systemctl is-active --quiet firewalld"
chk "존에 ssh 서비스"           "firewall-cmd --list-all | grep -qE 'ssh|2222'"
chk "sshd PermitRootLogin no"   "grep -qE '^\s*PermitRootLogin\s+no' /etc/ssh/sshd_config"

echo "===== 9. 백업·스케줄 (Part 06·12) ====="
chk "backup.sh 실행권한"        "test -x /usr/local/bin/backup.sh"
chk "cron.d/lab-backup"         "grep -q backup.sh /etc/cron.d/lab-backup"
chk "최근 백업 산출물"          "find /srv/raid/backup -maxdepth 1 -name 'lab-*.tar.gz' -mtime -2 | grep -q ."
chk "백업 체크섬 유효"          "cd /srv/raid/backup && sha256sum -c --quiet lab-*.sha256"
chk "backup.log 존재"           "test -s /var/log/backup.log"

echo
if [ "$NG" -eq 0 ]; then echo "===== 전체 정상 (NG=0) ====="; else echo "===== 문제 $NG 건 — 위 [NG] 항목 확인 ====="; fi
exit "$NG"
EOF

chmod 750 /usr/local/bin/final-check.sh
bash -n /usr/local/bin/final-check.sh && echo "문법 OK"
/usr/local/bin/final-check.sh; echo "NG=$?"
```

- `eval "$2"` : 문자열로 담은 점검 명령을 실행 — 표 형태로 항목을 나열할 수 있게 해 줌
- `&>/dev/null` : 표준출력·표준오류 모두 버림 (bash 전용 축약)
- `systemctl is-active --quiet <유닛>` : 출력 없이 종료 코드로만 판정 (0=active)
- `findmnt --verify --verbose` : `/etc/fstab` 항목의 유효성 검사 — 재부팅 전 필수 점검
- `swapon --show=NAME --noheadings` : 활성 스왑 장치명만 출력
- `repquota <경로>` : 쿼터 집계 보고 (Part 05)
- 종료 코드 = NG 건수 → cron·모니터링에서 그대로 활용 가능

**검증**

```bash
/usr/local/bin/final-check.sh | grep -c '^\[OK\]'
/usr/local/bin/final-check.sh | grep '^\[NG\]'
/usr/local/bin/final-check.sh > /srv/raid/backup/final-check-before-reboot.txt; echo "rc=$?"
tail -3 /srv/raid/backup/final-check-before-reboot.txt
```

```text
38
===== 전체 정상 (NG=0) =====
rc=0
```

- `[NG]` 가 나온 항목은 해당 파트로 돌아가 `enable` 누락·fstab 누락·정책 미저장을 점검

> 📝 **시험 포인트**: "재부팅 후에도 유지되는가" 는 필기·실기 전 영역의 공통 함정 — `systemctl enable`(서비스), `/etc/fstab`(마운트·스왑), `--permanent`(firewalld), `/etc/selinux/config`(SELinux), `/etc/sysctl.d/`(커널 파라미터), `mdadm.conf`+`dracut`(RAID), `--restart` 정책(Docker).

### 10-4. 실제 재부팅과 부팅 이력 확인

> **상황**: 마지막으로 실제 재부팅을 걸고, 올라온 뒤 점검 스크립트를 다시 돌려 두 결과를 비교한다. 부팅 이력 조회 명령도 함께 확인한다.

```bash
# 재부팅 전 — 상태 저장
/usr/local/bin/final-check.sh > /srv/raid/backup/final-check-before-reboot.txt
sync; sync
systemctl reboot        # 또는 shutdown -r now / reboot / init 6

# ── 재부팅 후 재접속 (포트 2222) ──
# $ ssh -p 2222 admin1@192.168.64.10

uptime
uptime -s                      # 부팅 시각
uptime -p                      # 사람이 읽는 가동 시간
last -x | head -8              # reboot / shutdown / runlevel 엔트리
last reboot | head -5
journalctl --list-boots | tail -5
journalctl -b -p err --no-pager | head
systemd-analyze
systemd-analyze blame --no-pager | head -5

# 점검 재실행 후 비교
/usr/local/bin/final-check.sh > /srv/raid/backup/final-check-after-reboot.txt; echo "NG=$?"
diff /srv/raid/backup/final-check-before-reboot.txt /srv/raid/backup/final-check-after-reboot.txt && echo "재부팅 전후 동일"
```

- `uptime` : 현재 시각·가동 시간·로그인 수·1/5/15분 **load average**. `-s` 부팅 시각(**s**ince), `-p` 보기 좋은 형식(**p**retty)
- `last -x` : `/var/log/wtmp` 에서 `reboot`·`shutdown`·`runlevel` 항목까지 표시. `last reboot` 은 재부팅 이력만
- `journalctl --list-boots` : 부팅 세션 목록 (`0`=현재, `-1`=직전). `journalctl -b -1` 로 직전 부팅 로그 조회 — **저널 영구화(Part 07 5-3) 가 되어 있어야 여러 건이 보임**
- `systemd-analyze` : 커널·initrd·userspace 부팅 소요 시간. `blame` 은 유닛별 지연 내림차순
- load average 는 "실행 중 + 실행 대기 + **I/O 대기(D 상태)**" 프로세스 수 평균 — CPU 코어 수와 비교해 해석(Part 06)

**검증**

```bash
uptime -p
last -x | grep -E 'reboot|shutdown' | head -4
journalctl --list-boots | wc -l
systemctl --failed --no-legend | wc -l
/usr/local/bin/final-check.sh | tail -1
findmnt -t xfs,ext4 | grep -E '/data|/srv'
swapon --show
cat /proc/mdstat | grep md0
docker ps --format '{{.Names}} {{.Status}}'
```

```text
up 3 minutes
reboot   system boot  5.14.0-...  ...   still running
shutdown system down  5.14.0-...  ...
reboot   system boot  5.14.0-...  ...
2
0
===== 전체 정상 (NG=0) =====
/data      /dev/vdb1                ext4
/srv/raid  /dev/md0                 xfs
/srv/share /dev/mapper/vg_lab-lv_share xfs
NAME       TYPE      SIZE USED PRIO
/dev/vdb2  partition   1G   0B   -2
/swapfile  file      512M   0B   -3
md0 : active raid1 vdd[1] vdc[0]
lab-redis Up 2 minutes
```

- 재부팅 전후 점검 결과가 동일하면 **모든 자원이 영구 설정으로 등록**되어 있다는 증명

> 📝 **시험 포인트**: 필기 FULL r01-57 · r02-58 — `journalctl --list-boots` 는 "부팅 세션 목록". `uptime` 의 load average 해석(r08-30)은 코어 수 대비 비교. `last`(wtmp) / `lastb`(btmp) / `lastlog`(lastlog) 3종 구분.

---

## 11. 전 범위 종합 체크리스트

> 12개 파트에서 만든 자원과 익힌 명령을 한 표에서 되짚는다. **검증 명령을 실제로 쳐 보고** 기대 결과가 나오는지 확인한 항목만 체크한다. `[NG]` 가 나오면 해당 파트 문서로 돌아가 그 단계만 다시 수행한다.

### 11-1. Part 01~03 — 시스템 점검 · 패키지 · 계정/권한

> **상황**: 서버 신원(호스트명·커널·FHS), 소프트웨어 공급(dnf/rpm), 그리고 사람(계정·권한) 세 축이 모두 서 있는지 확인한다.

| 파트 | 항목 | 검증 명령 | 기대 결과 | ☐ |
| --- | --- | --- | --- | --- |
| 01 | 호스트명 `srv01.lab.local` | `hostnamectl` | `Static hostname: srv01.lab.local` | ☐ |
| 01 | 커널·아키텍처 | `uname -r`, `uname -m`, `arch` | `5.14.0-…`, `aarch64` | ☐ |
| 01 | 배포판 정보 | `cat /etc/os-release`, `cat /etc/redhat-release` | `Rocky Linux 9.x` | ☐ |
| 01 | CPU·메모리 | `lscpu \| head`, `free -h`, `cat /proc/meminfo \| head -3` | 2 vCPU / 약 3.7Gi | ☐ |
| 01 | 블록 장치 목록 | `lsblk`, `lsblk -f`, `df -hT` | `vda~vde` 5개 | ☐ |
| 01 | FHS 주요 디렉터리 | `ls /`, `ls /etc /var /usr/bin /srv` | `/bin`→`usr/bin` 심볼릭 | ☐ |
| 01 | 셸 환경·PATH | `echo $SHELL $PATH`, `env \| head`, `set \| head` | `/bin/bash`, PATH 6~7개 경로 | ☐ |
| 01 | 별칭·이력 | `alias`, `history \| tail -5`, `echo $HISTSIZE` | 별칭 3종 이상, 이력 출력 | ☐ |
| 01 | 접속·세션 정보 | `who`, `w`, `id`, `last \| head`, `lastlog -u dev1` | 현재 세션 표시 | ☐ |
| 01 | 시각·로케일 | `timedatectl`, `localectl`, `date` | `Time zone: Asia/Seoul` | ☐ |
| 02 | 저장소 목록 | `dnf repolist`, `dnf repolist --all \| head` | baseos·appstream·extras·epel | ☐ |
| 02 | 패키지 조회 4종 | `rpm -qi httpd`, `rpm -ql httpd \| head`, `rpm -qf /etc/passwd`, `rpm -qc httpd` | 정보·파일목록·소속·설정파일 | ☐ |
| 02 | 패키지 무결성 검증 | `rpm -V httpd`, `rpm -Va \| head` | 변경 파일에 `S.5....T c` | ☐ |
| 02 | dnf 설치·삭제·이력 | `dnf history list \| head`, `dnf history info <ID>` | 트랜잭션 목록 | ☐ |
| 02 | EPEL 활성 | `rpm -q epel-release`, `dnf repolist \| grep epel` | 설치됨 | ☐ |
| 02 | 그룹·모듈 | `dnf group list \| head`, `dnf module list \| head` | 그룹·모듈 스트림 출력 | ☐ |
| 02 | 소스 컴파일 3단계 | `which hello`, `hello` | `./configure && make && make install` 산출물 | ☐ |
| 02 | 라이브러리 의존성 | `ldd /bin/ls`, `ldconfig -p \| wc -l`, `cat /etc/ld.so.conf.d/*` | 공유 라이브러리 목록 | ☐ |
| 02 | rpm2cpio 추출 | `rpm2cpio <pkg>.rpm \| cpio -idmv \| head` | 패키지 내부 파일 추출 | ☐ |
| 03 | 그룹 2종 | `getent group devteam opsteam` | GID 2000·2001 | ☐ |
| 03 | 사용자 4종 | `getent passwd dev1 dev2 ops1 guest1`, `id dev1` | UID 2001~2004 | ☐ |
| 03 | shadow 필드 | `getent shadow dev1 \| awk -F: '{print $3,$4,$5,$6,$7}'` | 최종변경일·min·max·warn·inactive | ☐ |
| 03 | 비밀번호 정책 | `chage -l dev1`, `grep PASS_MAX /etc/login.defs` | 최대 90일 등 정책 반영 | ☐ |
| 03 | 계정 잠금·만료 | `passwd -S guest1`, `usermod -L guest1` 후 `getent shadow guest1` | 해시 앞 `!` | ☐ |
| 03 | sudo 권한 | `visudo -c`, `sudo -lU ops1`, `grep wheel /etc/sudoers` | 문법 OK, ops1 허용 목록 | ☐ |
| 03 | 특수 권한 | `ls -l /usr/bin/passwd`, `ls -ld /tmp`, `find / -perm -4000 -type f 2>/dev/null \| head` | `rws`, `drwxrwxrwt` | ☐ |
| 03 | umask | `umask`, `umask -S`, 새 파일 `ls -l` | `0022` → 파일 644 / 디렉터리 755 | ☐ |
| 03 | ACL | `getfacl /srv/share`, `setfacl -m u:dev1:rwx <경로>` | `user:dev1:rwx`, `ls -l` 에 `+` | ☐ |
| 03 | 파일 속성 | `lsattr <파일>`, `chattr +i` 후 `rm` 시도 | `----i---------`, `Operation not permitted` | ☐ |
| 03 | PAM 구성 | `ls /etc/pam.d/`, `grep pam_pwquality /etc/pam.d/system-auth` | 모듈 타입·control flag 확인 | ☐ |

**일괄 확인**

```bash
hostnamectl; echo ---
getent group devteam opsteam; getent passwd dev1 dev2 ops1 guest1 | cut -d: -f1,3,4
echo ---; rpm -q httpd bind samba vsftpd postfix nfs-utils mdadm quota docker-ce 2>&1 | head
echo ---; find / -perm -4000 -type f 2>/dev/null | wc -l
```

```text
 Static hostname: srv01.lab.local
---
devteam:x:2000:
opsteam:x:2001:dev2
dev1:2001:2000
...
```

> 📝 **시험 포인트**: `/etc/passwd` 7필드 · `/etc/shadow` 9필드 · `/etc/group` 4필드는 매 회차 출제. `rpm -qf`(파일→패키지) vs `rpm -ql`(패키지→파일) 방향 혼동 주의.

### 11-2. Part 04~06 — 파일/텍스트/셸 · 디스크 · 프로세스

> **상황**: 데이터를 다루는 손(텍스트 도구·스크립트), 데이터를 담는 그릇(디스크·LVM·RAID), 데이터를 처리하는 일꾼(프로세스·스케줄러)을 점검한다.

| 파트 | 항목 | 검증 명령 | 기대 결과 | ☐ |
| --- | --- | --- | --- | --- |
| 04 | find 조건 검색 | `find /data -name '*.txt' -size +1k -mtime -7`, `find / -perm -4000 -type f` | 조건 일치 목록 | ☐ |
| 04 | find 실행·삭제 | `find /tmp -name '*.tmp' -exec rm -f {} \;`, `-delete` | 대상 제거 | ☐ |
| 04 | grep 계열 | `grep -rn 'Listen' /etc/httpd`, `grep -c`, `grep -E`, `grep -v` | 행번호·개수·정규식 | ☐ |
| 04 | sed 치환 | `sed -i 's/Listen 80/Listen 8080/' <파일>`, `sed -n '5,10p'` | 파일 직접 수정·범위 출력 | ☐ |
| 04 | awk 필드 처리 | `awk -F: '$3>=1000{print $1}' /etc/passwd`, `awk '{sum+=$1}END{print sum}'` | 조건 필드 출력 | ☐ |
| 04 | 텍스트 유틸 | `cut -d: -f1,3`, `sort -t: -k3 -n`, `uniq -c`, `tr`, `wc -l`, `head`/`tail` | 각 출력 | ☐ |
| 04 | 리다이렉션·파이프 | `cmd > f 2>&1`, `cmd 2>/dev/null`, `tee`, `xargs` | 스트림 분리 동작 | ☐ |
| 04 | 압축 계열 | `gzip`/`gunzip`, `bzip2`, `xz`, `zip`/`unzip`, `split` | 확장자별 산출물 | ☐ |
| 04 | 셸 변수·스크립트 | `bash -n <스크립트>`, `echo $?`, `export`, `set -e` | 문법 OK, 종료 코드 | ☐ |
| 04 | vi 편집 | `vi` → `:set nu`, `dd`, `yy`, `/검색`, `:%s/a/b/g`, `:wq` | 편집·저장 성공 | ☐ |
| 04 | 링크·inode | `ln f hard`, `ln -s f soft`, `ls -li`, `stat f` | 하드=inode 동일, 심볼릭=별도 | ☐ |
| 05 | 파티션 구성 | `fdisk -l /dev/vdb`, `parted /dev/vde print` | vdb1/vdb2/vdb3 존재 | ☐ |
| 05 | 파일시스템 생성 | `blkid`, `lsblk -f` | vdb1 ext4, md0·lv_share xfs | ☐ |
| 05 | 마운트·fstab | `findmnt --verify`, `mount -a`, `df -hT` | 오류 없음, 3개 마운트 | ☐ |
| 05 | UUID 등록 | `grep UUID /etc/fstab`, `blkid` 값 대조 | 장치명 아닌 UUID 사용 | ☐ |
| 05 | 스왑 2종 | `swapon --show`, `free -h`, `grep swap /etc/fstab` | 파티션 + 파일 모두 활성 | ☐ |
| 05 | RAID 1 | `cat /proc/mdstat`, `mdadm --detail /dev/md0` | `[2/2] [UU]`, `State: clean` | ☐ |
| 05 | LVM 계층 | `pvs`, `vgs`, `lvs`, `pvdisplay \| head` | PV→VG `vg_lab`→LV `lv_share` | ☐ |
| 05 | 온라인 확장 | `lvextend -r -L +200M /dev/vg_lab/lv_share`, `df -h /srv/share` | 마운트 상태로 용량 증가 | ☐ |
| 05 | 쿼터 | `repquota -a`, `quotaon -p /data`, `edquota -u dev1` | soft/hard 한계 표시 | ☐ |
| 05 | 점검·용량 | `df -h`, `df -i`, `du -sh /*`, `fsck -n /dev/vdb1` | 사용률·inode·점검 결과 | ☐ |
| 05 | 커널 모듈 | `lsmod \| head`, `modinfo raid1`, `modprobe -r <모듈>` | 모듈 목록·정보 | ☐ |
| 06 | 프로세스 조회 | `ps -ef \| head`, `ps aux --sort=-%cpu \| head -5`, `pgrep -a sshd` | PID·PPID·%CPU | ☐ |
| 06 | 프로세스 트리 | `pstree -p \| head`, `ps -ejH \| head` | systemd 루트 트리 | ☐ |
| 06 | 잡 제어 | `sleep 300 &` → `jobs` → `fg %1` → `Ctrl+Z` → `bg %1` | 상태 전이 확인 | ☐ |
| 06 | 백그라운드 유지 | `nohup <명령> &`, `disown`, `ls nohup.out` | 로그아웃 후 생존 | ☐ |
| 06 | 시그널 | `kill -l \| head`, `kill -15 <PID>`, `kill -9`, `pkill -u dev1`, `killall` | 종료 확인 | ☐ |
| 06 | 우선순위 | `nice -n 10 <명령>`, `renice -n -5 -p <PID>`, `ps -eo pid,ni,cmd \| head` | NI 값 변경 | ☐ |
| 06 | top 심화 | `top -b -n1 \| head -12`, 대화식 `P M k r 1 c` | 정렬·시그널·코어별 표시 | ☐ |
| 06 | 통계 도구 | `vmstat 1 3`, `iostat -x 1 2`, `sar -u 1 3`, `mpstat`, `lsof \| head` | 각 지표 출력 | ☐ |
| 06 | at / cron | `at now +5min`, `atq`, `crontab -l -u dev1`, `cat /etc/cron.d/lab-backup` | 예약 목록 표시 | ☐ |
| 06 | systemd 타이머 | `systemctl list-timers --no-pager \| head` | NEXT·LEFT 표시 | ☐ |

**일괄 확인**

```bash
lsblk -f; echo ---; swapon --show; echo ---; cat /proc/mdstat | head -4
echo ---; pvs; vgs; lvs; echo ---; repquota -a 2>/dev/null | head -6
echo ---; systemctl list-timers --no-pager | head -4
```

```text
NAME FSTYPE FSVER LABEL UUID MOUNTPOINTS
vdb1 ext4   1.0         ...  /data
...
```

> 📝 **시험 포인트**: `lvextend -r`(파일시스템까지 확장) · `xfs_growfs`(xfs 온라인 확장, 축소 불가) · `resize2fs`(ext) 3종 구분. `nice` 는 실행 시, `renice` 는 실행 중 변경.

### 11-3. Part 07~09 — 부팅/systemd/로그 · 네트워크 · 네트워크 서비스

> **상황**: 서버가 스스로 일어나고(부팅·systemd), 밖과 통하고(네트워크), 서비스를 내보내는지(웹·DNS·공유·메일) 확인한다.

| 파트 | 항목 | 검증 명령 | 기대 결과 | ☐ |
| --- | --- | --- | --- | --- |
| 07 | 부팅 단계·시간 | `systemd-analyze`, `systemd-analyze blame \| head`, `journalctl -b \| head` | kernel+userspace 소요 | ☐ |
| 07 | GRUB 설정 | `grep GRUB_TIMEOUT /etc/default/grub`, `grubby --info=DEFAULT` | 반영값 확인 | ☐ |
| 07 | GRUB 재생성 | `grub2-mkconfig -o /boot/grub2/grub.cfg` | 오류 없이 완료 | ☐ |
| 07 | 유닛 관리 | `systemctl list-units --type=service --state=running`, `--failed` | 실패 0건 | ☐ |
| 07 | enable/mask | `systemctl is-enabled httpd`, `systemctl mask/unmask <유닛>` | enabled / `/dev/null` 링크 | ☐ |
| 07 | 기본 타겟 | `systemctl get-default`, `runlevel`, `who -r` | `multi-user.target`, `N 3` | ☐ |
| 07 | 커스텀 서비스 | `systemctl status lab-monitor`, `systemctl cat lab-monitor` | 유닛 파일·상태 | ☐ |
| 07 | 저널 조회 | `journalctl -u sshd -n 20`, `-b -p err`, `--since today`, `-f` | 필터 동작 | ☐ |
| 07 | 저널 영구화 | `ls /var/log/journal`, `journalctl --list-boots \| wc -l` | 2건 이상 | ☐ |
| 07 | rsyslog 규칙 | `grep -v '^#' /etc/rsyslog.conf \| head`, `logger -p local5.info -t test` → `tail /var/log/lab.log` | 지정 파일에 기록 | ☐ |
| 07 | 바이너리 로그 | `last \| head`, `lastb \| head`, `lastlog \| head` | wtmp/btmp/lastlog | ☐ |
| 07 | logrotate | `logrotate -d /etc/logrotate.d/backup`, `ls /var/log/*.1 /var/log/*.gz` | 회전 계획·산출물 | ☐ |
| 07 | root 비번 복구 절차 | (수행 경험) `rd.break` → remount rw → chroot → passwd → `/.autorelabel` | 새 비번 로그인 성공 | ☐ |
| 08 | IP·라우팅 | `ip -br a`, `ip r`, `ip -br link` | `enp0s1 192.168.64.10/24` | ☐ |
| 08 | NetworkManager | `nmcli con show`, `nmcli dev status`, `nmcli con show <프로필> \| grep ipv4` | manual, 고정 IP | ☐ |
| 08 | DNS 클라이언트 | `cat /etc/resolv.conf`, `dig @127.0.0.1 srv01.lab.local`, `getent hosts srv01.lab.local` | A 레코드 응답 | ☐ |
| 08 | 소켓·포트 | `ss -tlnp \| head`, `ss -tulnp \| grep -E '22\|53\|80\|445'` | LISTEN 목록 | ☐ |
| 08 | 진단 도구 | `ping -c2 192.168.64.1`, `nc -zv 127.0.0.1 2222`, `traceroute`/`tracepath`, `tcpdump -c5 -i enp0s1` | 응답·연결 성공 | ☐ |
| 08 | SSH 키 인증 | `ls ~/.ssh/`, `ssh -p 2222 -i <키> admin1@localhost 'hostname'` | 비밀번호 없이 로그인 | ☐ |
| 08 | SSH 포트 변경 | `grep '^Port' /etc/ssh/sshd_config`, `ss -tlnp \| grep 2222`, `semanage port -l \| grep ssh` | 2222, SELinux 포트 라벨 | ☐ |
| 08 | 시간 동기화 | `chronyc sources`, `chronyc tracking`, `timedatectl` | `NTP service: active` | ☐ |
| 09 | Apache 기동·문법 | `httpd -t`, `apachectl configtest`, `systemctl is-active httpd`, `curl -I localhost` | `Syntax OK`, `HTTP/1.1 200` | ☐ |
| 09 | 가상 호스트 | `httpd -S`, `curl -H 'Host: intranet.lab.local' localhost` | vhost 목록·페이지 응답 | ☐ |
| 09 | BIND 존 | `named-checkconf`, `named-checkzone lab.local /var/named/lab.local.zone`, `dig @localhost lab.local SOA` | `OK`, SOA 응답 | ☐ |
| 09 | 역방향 존 | `dig @localhost -x 192.168.64.10 +short` | `srv01.lab.local.` | ☐ |
| 09 | NFS 내보내기 | `exportfs -v`, `showmount -e localhost`, `mount -t nfs localhost:/srv/nfs/data /mnt/nfs` | 공유 목록·마운트 성공 | ☐ |
| 09 | Samba | `testparm -s`, `smbclient -L localhost -U dev1`, `pdbedit -L` | 공유 `[share]` 표시 | ☐ |
| 09 | vsftpd | `systemctl is-active vsftpd`, `ftp localhost` 또는 `curl ftp://localhost/pub/` | 접속·목록 | ☐ |
| 09 | Postfix·별칭 | `postconf -n \| head`, `newaliases`, `mailq`, `echo t \| mail -s x ops1` → `mail -H` | 로컬 배송 확인 | ☐ |
| 09 | CUPS 프린터 | `lpstat -t`, `lpstat -p labprn`, `lpr -P labprn <파일>`, `lpq` | 프린터 등록·큐 | ☐ |

**일괄 확인**

```bash
systemctl is-active sshd httpd named nfs-server smb vsftpd postfix chronyd crond docker
echo ---; ss -tuln | grep -E ':(21|22|25|53|80|139|445|631|2049|2222|6379) '
echo ---; httpd -t; named-checkconf && echo "named OK"; testparm -s 2>/dev/null | head -3
```

```text
active
active
...
LISTEN 0 128 0.0.0.0:2222 ...
Syntax OK
named OK
```

> 📝 **시험 포인트**: 설정 검사 명령 3종 — Apache `httpd -t`/`apachectl configtest`, BIND `named-checkconf`/`named-checkzone`, Samba `testparm`. 서비스 재시작 **전에** 반드시 실행.

### 11-4. Part 10~12 — 보안 · 컨테이너/가상화 · 백업/복구

> **상황**: 방어(방화벽·SELinux), 실행 환경(컨테이너), 그리고 마지막 안전망(백업·복구)을 확인한다.

| 파트 | 항목 | 검증 명령 | 기대 결과 | ☐ |
| --- | --- | --- | --- | --- |
| 10 | firewalld 존 | `firewall-cmd --get-active-zones`, `--list-all` | 존·서비스·포트 목록 | ☐ |
| 10 | 영구 규칙 | `firewall-cmd --permanent --add-service=http` → `--reload` → `--list-services` | 재부팅 후 유지 | ☐ |
| 10 | 포트·리치룰 | `firewall-cmd --list-ports`, `--list-rich-rules` | `2222/tcp` 등 | ☐ |
| 10 | iptables 규칙 | `iptables -L -n -v --line-numbers`, `iptables -S` | 체인·정책·카운터 | ☐ |
| 10 | iptables 저장·복원 | `iptables-save > /etc/sysconfig/iptables`, `iptables-restore <` | 규칙 재적용 | ☐ |
| 10 | SELinux 모드 | `getenforce`, `sestatus`, `grep SELINUX= /etc/selinux/config` | `Enforcing` (일시: `setenforce 0`) | ☐ |
| 10 | 컨텍스트 | `ls -Z /srv/www/intranet`, `semanage fcontext -l \| grep srv`, `restorecon -Rv` | `httpd_sys_content_t` | ☐ |
| 10 | 불린 | `getsebool -a \| grep httpd \| head`, `setsebool -P <불린> on` | `-P` 로 영구 반영 | ☐ |
| 10 | 포트 라벨 | `semanage port -l \| grep -E 'ssh\|http'` | 2222 등록 확인 | ☐ |
| 10 | 감사 로그 분석 | `ausearch -m avc -ts recent \| head`, `sealert -a /var/log/audit/audit.log \| head` | AVC 거부 원인 | ☐ |
| 10 | SSH 강화 | `grep -E '^(Port\|PermitRootLogin\|PasswordAuthentication\|MaxAuthTries)' /etc/ssh/sshd_config` | root 로그인 차단 | ☐ |
| 10 | 침해 점검 | `find / -perm -4000 -type f 2>/dev/null`, `lastb \| head`, `grep 'Failed password' /var/log/secure \| wc -l` | 목록·실패 횟수 | ☐ |
| 10 | 암호화 | `gpg -c <파일>` → `gpg -d`, `openssl enc -aes-256-cbc`, `openssl dgst -sha256` | 복호화 성공 | ☐ |
| 11 | Docker 기동 | `systemctl is-active docker`, `docker version`, `docker info \| head` | active | ☐ |
| 11 | 컨테이너 운영 | `docker ps -a`, `docker logs lab-redis \| tail -3`, `docker exec -it lab-redis sh` | 실행 중·로그 | ☐ |
| 11 | 볼륨·네트워크 | `docker volume ls`, `docker network ls`, `docker inspect lab-redis \| head` | `redis-data`, bridge | ☐ |
| 11 | 이미지 빌드 | `docker images`, `docker history lab/ubuntu-tools:1.0` | 태그·레이어 | ☐ |
| 11 | 컨테이너 내 apt/dpkg | `docker exec lab-ubuntu apt list --installed \| head`, `dpkg -l \| head` | Debian 계열 명령 동작 | ☐ |
| 11 | 자동 재시작 | `docker inspect lab-redis --format '{{.HostConfig.RestartPolicy.Name}}'` | `unless-stopped` | ☐ |
| 11 | libvirt 조회 | `systemctl is-active libvirtd`, `virsh list --all`, `virsh net-list --all` | 조회 성공 (※ 중첩 가상화 불가) | ☐ |
| 12 | tar 전체·증분 | `ls /srv/raid/backup/data-full.tar.gz data-inc1.tar.gz`, `file *.snar` | snar = incremental snapshot | ☐ |
| 12 | tar 복구 리허설 | `rm -rf /data/proj` → 레벨0→1 복원 → `sha256sum -c` | 전부 `OK` | ☐ |
| 12 | rsync 미러 | `diff -rq /srv/share /srv/raid/mirror/share` | 차이 없음 | ☐ |
| 12 | rsync 스냅샷 | `ls -li snapshots/*/src/file2.txt`, `du -sh snapshots` | inode 공유, 합계=1세대 크기 | ☐ |
| 12 | 오프사이트 | `ssh <mac> 'ls ~/lab-backup'`, 원격 `shasum -a 256 -c` | 목록·해시 OK | ☐ |
| 12 | dd 메타데이터 | `ls -l vda-pt.bin`(17408) `vdb-pt64.bin`(64), `head -3 vda-sfdisk.txt` | 크기 일치 | ☐ |
| 12 | dd 이미지 복구 | `mkfs.ext4` 파괴 → `gunzip -c \| dd of=` → `mount` → `sha256sum -c` | UUID·데이터 복원 | ☐ |
| 12 | dump/restore | `cat /etc/dumpdates`, `restore -tf data.dump0 \| head`, `restore -rf` 체인 | 레벨 0/1/2 기록·복원 | ☐ |
| 12 | xfsdump/xfsrestore | `xfsdump -I`, `diff -rq /srv/share /srv/raid/restore-xfs` | 세션 목록·차이 없음 | ☐ |
| 12 | LVM 스냅샷 백업 | `/usr/local/bin/snap-backup.sh` → `lvs` | 산출물 생성·스냅샷 제거됨 | ☐ |
| 12 | 메타데이터 백업 | `ls /etc/lvm/backup/vg_lab`, `grep ARRAY /etc/mdadm.conf`, `lsinitrd \| grep mdadm.conf` | 3종 모두 존재 | ☐ |
| 12 | Docker 볼륨 복구 | 볼륨 삭제 → tar 복원 → `redis-cli get labkey` | 값 생존 | ☐ |
| 12 | backup.sh 자동화 | `/usr/local/bin/backup.sh; echo $?`, `tail /var/log/backup.log`, `sha256sum -c lab-*.sha256` | rc=0, `OK` | ☐ |
| 12 | 실패·중복 실행 | 이중 실행 rc=3, 원본 누락 rc=4 → `grep 'BACKUP FAIL' /var/spool/mail/ops1` | 메일 수신 | ☐ |
| 12 | 종료·재부팅 | `shutdown -r +1 "msg"` → `/run/nologin` → `shutdown -c` | 생성·삭제 확인 | ☐ |
| 12 | 재부팅 후 전 자원 | `/usr/local/bin/final-check.sh` | `NG=0` | ☐ |

**일괄 확인**

```bash
getenforce; firewall-cmd --list-all | head -8
echo ---; docker ps --format '{{.Names}} {{.Status}}'
echo ---; ls /srv/raid/backup | head -20
echo ---; /usr/local/bin/final-check.sh | tail -1
```

```text
Enforcing
public (active)
  services: ssh dhcpv6-client http https samba
lab-redis Up ...
data-full.tar.gz
...
===== 전체 정상 (NG=0) =====
```

> 📝 **시험 포인트**: "영구 적용" 키워드 매핑 — firewalld `--permanent`+`--reload`, SELinux 불린 `-P`, iptables `iptables-save`, 마운트 `/etc/fstab`, 서비스 `systemctl enable`, 커널 파라미터 `/etc/sysctl.d/`.

---

## 12. 시험 직전 명령어 정리표

> 기출 빈도 상위부터 배치. **핵심 옵션**은 시험에서 옵션 자체를 물었거나 오답 보기로 자주 등장한 것 위주. 한 줄 용도만 읽고 옵션이 떠오르지 않으면 그 파트로 돌아갈 것.

### 12-1. 최빈출 1~40 — 패키지 · 계정 · 권한 · 디스크 기초

| 명령 | 핵심 옵션 | 한 줄 용도 | 파트 |
| --- | --- | --- | --- |
| `rpm` | `-qa` `-qi` `-ql` `-qf` `-qc` `-qd` `-V` `-Va` `-ivh` `-Uvh` `-Fvh` `-e` `--nodeps` `--force` `-qp` `--qf` | RPM 패키지 질의·검증·설치·제거 (의존성 자동 해결 없음) | 02 |
| `dnf` | `install` `remove` `update` `search` `info` `provides` `list` `repolist` `history` `group` `module` `-y` `--nogpgcheck` `--enablerepo` | 저장소 기반 패키지 관리 (의존성 자동 해결) | 02 |
| `systemctl` | `start` `stop` `restart` `reload` `status` `enable` `disable` `--now` `mask` `is-active` `is-enabled` `list-units` `list-unit-files` `get-default` `set-default` `isolate` `daemon-reload` | systemd 유닛·타겟 제어 | 07 |
| `passwd` | `-l` `-u` `-d` `-e` `-S` `-n` `-x` `-w` `-i` `--stdin` | 비밀번호 설정·잠금·만료 상태 조회 | 03 |
| `useradd` | `-u` `-g` `-G` `-d` `-m` `-M` `-s` `-e` `-f` `-c` `-D` `-r` | 사용자 계정 생성 (`-D` 는 기본값 조회·변경) | 03 |
| `chmod` | `u/g/o/a` `+ - =` `r w x` `4755` `2755` `1777` `-R` | 파일 권한·특수 권한(SetUID/SetGID/Sticky) 변경 | 03 |
| `usermod` | `-u` `-g` `-G` `-aG` `-d -m` `-s` `-L` `-U` `-e` `-l` | 기존 계정 속성 변경 (`-aG` 없이 `-G` 는 보조그룹 교체) | 03 |
| `iptables` | `-L -n -v` `-A` `-I` `-D` `-P` `-t nat` `-j ACCEPT/DROP/REJECT` `-p` `-s` `-d` `--dport` `-m state --state` `-F` | 패킷 필터 규칙 직접 관리 | 10 |
| `find` | `-name` `-iname` `-type` `-size` `-perm` `-user` `-group` `-mtime` `-atime` `-newer` `-exec … {} \;` `-delete` `-maxdepth` `-print0` | 조건 기반 파일 검색·일괄 처리 | 04 |
| `mount` | `-t` `-o ro,rw,noexec,nosuid,remount,loop,nouuid` `-a` `-o remount,rw` | 파일시스템 마운트 (`-a` 는 fstab 전체) | 05 |
| `chage` | `-l` `-M` `-m` `-W` `-I` `-E` `-d 0` | 비밀번호 유효기간·계정 만료 정책 | 03 |
| `tar` | `-c` `-x` `-t` `-v` `-f` `-z` `-j` `-J` `-C` `-p` `-g/--listed-incremental` `--exclude` `--acls` `--selinux` `--xattrs` `-W` | 아카이브 생성·추출·목록·증분 백업 | 04, 12 |
| `su` | `-` `-c` `-l` `-s` | 사용자 전환 (`-` 는 로그인 셸 환경까지 전환) | 03 |
| `kill` | `-l` `-9(SIGKILL)` `-15(SIGTERM)` `-1(SIGHUP)` `-2(SIGINT)` `-19/-18(STOP/CONT)` `-USR1` | PID 에 시그널 전송 | 06 |
| `lastlog` | `-u` `-b` `-t` | 사용자별 **마지막 로그인** 시각 (`/var/log/lastlog`) | 07 |
| `renice` | `-n` `-p` `-u` `-g` | **실행 중** 프로세스 우선순위(NI) 변경 | 06 |
| `pvcreate` | `-f` `-v` | 물리 볼륨(PV) 생성 — LVM 1단계 | 05 |
| `mkfs` | `-t ext4/xfs` `-L` `-b` `-F` `mkfs.ext4` `mkfs.xfs` | 파일시스템 생성 (⚠️ 기존 데이터 파괴) | 05 |
| `fdisk` | `-l` `n p e w q d t m` | MBR 파티션 편집 (대화식, `w` 로 저장) | 05 |
| `modprobe` | `-r` `-v` `--show-depends` `-c` | 커널 모듈 적재·제거 (**의존성 자동 처리**) | 05 |
| `grub2-mkconfig` | `-o /boot/grub2/grub.cfg` | `/etc/default/grub` 수정 후 GRUB 설정 재생성 | 07 |
| `du` | `-s` `-h` `-a` `-c` `--max-depth=N` `-x` | 디렉터리·파일 **사용량** 집계 | 05 |
| `nice` | `-n <값>` (`-20`~`19`) | **실행 시** 우선순위 지정 (낮을수록 우선) | 06 |
| `lvcreate` | `-L` `-l` `-n` `-s`(스냅샷) `-l 100%FREE` | 논리 볼륨(LV)·스냅샷 생성 | 05, 12 |
| `exportfs` | `-v` `-a` `-r` `-u` `-o` | NFS 공유 목록 적용·재적용·해제 | 09 |
| `ls` | `-l` `-a` `-h` `-i` `-d` `-R` `-t` `-r` `-S` `-Z` `-li` | 파일 목록·속성·inode·SELinux 컨텍스트 | 01 |
| `journalctl` | `-u` `-b` `-b -1` `-p` `-f` `-n` `-r` `-k` `--since` `--until` `--list-boots` `--disk-usage` `--vacuum-size` `-o` | systemd 저널 조회 | 07 |
| `insmod` | (파일 경로 직접) | 모듈 파일 직접 적재 — **의존성 자동 해결 안 함** | 05 |
| `lsmod` | (인자 없음) | 적재된 커널 모듈 목록 (`/proc/modules`) | 05 |
| `fg` | `%<잡번호>` | 백그라운드 잡을 포그라운드로 전환 | 06 |
| `virsh` | `list --all` `start` `shutdown` `destroy` `dominfo` `net-list` `console` | libvirt 가상 머신 관리 (`shutdown`=정상 종료, `destroy`=강제) | 11 |
| `vgcreate` | `-s`(PE 크기) | 볼륨 그룹(VG) 생성 — LVM 2단계 | 05 |
| `setenforce` | `0`(Permissive) `1`(Enforcing) | SELinux 모드 **일시** 전환 (재부팅 시 config 값 복귀) | 10 |
| `ping` | `-c` `-i` `-s` `-W` `-4` `-6` | ICMP 로 도달성 확인 | 08 |
| `grep` | `-i` `-v` `-c` `-n` `-r` `-l` `-w` `-E` `-A/-B/-C` `--color` | 패턴 검색 (`-E` = egrep 확장 정규식) | 04 |
| `env` | `-i` `<VAR=값> <명령>` | 환경변수 목록 조회·임시 지정 실행 | 01 |
| `lsblk` | `-f` `-o` `-a` `-p` `-t` | 블록 장치 트리·FS·UUID·마운트 지점 | 05 |
| `dd` | `if=` `of=` `bs=` `count=` `skip=` `seek=` `conv=noerror,sync,notrunc,fsync` `status=progress` `iflag/oflag=direct` | 블록 단위 저수준 복제 (MBR·파티션 이미지) | 04, 12 |
| `ss` | `-t` `-u` `-l` `-n` `-p` `-a` `-tlnp` `-s` | 소켓·포트 상태 (`netstat` 대체) | 08 |
| `ps` | `-ef` `aux` `-eo` `--sort=-%cpu` `-u` `-C` `-ejH` `-L` | 프로세스 스냅샷 조회 | 06 |

### 12-2. 최빈출 41~80 — 네트워크 · 서비스 · LVM/스왑 · 셸

| 명령 | 핵심 옵션 | 한 줄 용도 | 파트 |
| --- | --- | --- | --- |
| `ip` | `a`/`addr` `addr add` `link set` `r`/`route` `route add default via` `-br` `neigh` | IP·라우팅·링크 설정 (`ifconfig`/`route` 대체) | 08 |
| `chattr` | `+i`(불변) `+a`(추가전용) `-i` `-R` | 파일 확장 속성 설정 (root 도 삭제 불가하게) | 03 |
| `bg` | `%<잡번호>` | 정지된 잡을 백그라운드에서 계속 실행 | 06 |
| `xfs_growfs` | `<마운트지점>` `-d` | xfs **온라인 확장** (축소 불가) | 05 |
| `who` | `-r` `-b` `-a` `-H` `-q` | 현재 로그인 사용자·런레벨·부팅 시각 | 01 |
| `nohup` | `<명령> &` | 로그아웃 후에도 실행 유지 (`nohup.out`) | 06 |
| `resize2fs` | `<장치>` `-p` `-M` | ext2/3/4 크기 조정 (축소는 언마운트 필요) | 05 |
| `export` | `VAR=값` `-p` `-n` | 셸 변수를 **환경변수**로 승격 | 01, 04 |
| `crontab` | `-e` `-l` `-r` `-u` `-i` | 사용자 cron 작업 편집·조회·삭제 | 06 |
| `testparm` | `-s` `-v` | Samba `smb.conf` 문법 검사·정규화 출력 | 09 |
| `lvextend` | `-L +크기` `-l +100%FREE` `-r`(FS 동시 확장) | LV 확장 (`-r` 없으면 FS 는 그대로) | 05 |
| `jobs` | `-l` `-p` `-r` `-s` | 현재 셸의 잡 목록·상태 | 06 |
| `sync` | (인자 없음) | 버퍼 캐시를 디스크로 강제 기록 | 12 |
| `ssh` | `-p` `-i` `-l` `-X` `-L/-R`(터널) `-v` `-o` | 원격 셸 접속 (**소문자 -p** 가 포트) | 08 |
| `smbpasswd` | `-a` `-x` `-d` `-e` | Samba 사용자 등록·삭제·비활성 (`pdbedit -L` 로 확인) | 09 |
| `showmount` | `-e` `-a` `-d` | NFS 서버의 공유 목록·클라이언트 조회 | 09 |
| `sed` | `-i` `-n` `s///g` `p` `d` `a` `i` `-e` `-f` `1,5p` | 스트림 편집기 — 치환·삭제·추출 | 04 |
| `mkfs.xfs` | `-f` `-L` `-b size=` | xfs 파일시스템 생성 | 05 |
| `httpd` | `-t`(문법검사) `-S`(vhost) `-M`(모듈) `-v` | Apache 데몬·설정 점검 | 09 |
| `getenforce` | (인자 없음) | SELinux 현재 모드 출력 (`sestatus` 는 상세) | 10 |
| `fsck` | `-y` `-n` `-f` `-t` `-A` `e2fsck` `xfs_repair` | 파일시스템 점검·복구 (**언마운트 상태**에서) | 05 |
| `firewall-cmd` | `--list-all` `--add-service` `--add-port` `--permanent` `--reload` `--zone=` `--get-active-zones` `--add-rich-rule` `--runtime-to-permanent` | firewalld 동적 방화벽 관리 | 10 |
| `dig` | `@서버` `-x`(역방향) `+short` `+trace` `ANY` `MX` `NS` `SOA` `AXFR` | DNS 질의 도구 | 08, 09 |
| `apt` | `update` `install` `remove` `purge` `search` `show` `list --installed` | Debian 계열 패키지 관리 (의존성 자동) | 11 |
| `dpkg` | `-i` `-r` `-P` `-l` `-L` `-S` `--info` | Debian 저수준 패키지 관리 (의존성 자동 해결 없음) | 11 |
| `xauth` | `list` `add` `remove` `extract` `merge` | X 인증 쿠키(MIT-MAGIC-COOKIE-1) 관리 — 사용자 단위 | 13 (※) |
| `wait` | `%<잡>` `<PID>` | 백그라운드 잡 종료까지 대기 | 06 |
| `visudo` | `-c`(문법검사) `-f` | `/etc/sudoers` 안전 편집 (잠금·문법 검사) | 03 |
| `swapon` | `--show` `-a` `-s` `-p`(우선순위) `swapoff -a` | 스왑 활성화·목록 조회 | 05 |
| `sudo` | `-l` `-u` `-i` `-s` `-k` `-lU` | 권한 위임 실행 (`/etc/sudoers` 기반) | 03 |
| `semanage` | `port -a -t -p` `fcontext -a -t` `port -l` `fcontext -l` `login` `boolean -l` | SELinux 정책 **영구** 관리 | 10 |
| `mkswap` | `-L` `-f` `-U` | 스왑 영역 생성 (파티션·파일 모두) | 05 |
| `gpasswd` | `-a` `-d` `-A` `-M` `-r` | 그룹 멤버·관리자 지정 | 03 |
| `awk` | `-F` `{print $1}` `$3>=1000` `BEGIN/END` `NR` `NF` `-v` | 필드 단위 텍스트 처리 | 04 |
| `apachectl` | `configtest` `graceful` `start/stop/restart` `-S` | Apache 제어 래퍼 (`httpd -t` 와 동일 검사) | 09 |
| `newaliases` | (인자 없음) | `/etc/aliases` 변경을 `aliases.db` 로 반영 | 09 |
| `mdadm` | `--create` `--level` `--raid-devices` `--detail` `--detail --scan` `--examine` `--assemble --scan` `--add` `--fail` `--remove` `--stop` | 소프트웨어 RAID 관리 | 05, 12 |
| `lpr` | `-P` `-#`(부수) `-o` | 프린터로 출력 요청 (CUPS) | 09 |
| `lpstat` | `-t` `-p` `-a` `-o` `-d` | 프린터·큐·작업 상태 조회 | 09 |
| `depmod` | `-a` `-n` | 모듈 의존성 데이터베이스(`modules.dep`) 갱신 | 05 |

### 12-3. 81~120 — 진단 · 백업 · 텍스트 · 파일 정보

| 명령 | 핵심 옵션 | 한 줄 용도 | 파트 |
| --- | --- | --- | --- |
| `cat` | `-n` `-A` `-v` `-b` `-s` | 파일 내용 출력·연결 (`tac` 은 역순) | 01 |
| `blkid` | `-s UUID -o value` `-p` `-L` | 장치 UUID·라벨·FS 유형 조회 | 05 |
| `docker` | `run -d --name -p -v --restart` `ps -a` `exec -it` `logs` `images` `build -t` `volume` `network` `save/load` `export/import` `rm -f` `system prune` | 컨테이너·이미지·볼륨 관리 | 11, 12 |
| `top` | `-b` `-n` `-d` `-u` `-p` / 대화식 `P M T k r 1 c H z W q` | 실시간 프로세스·부하 모니터 | 06 |
| `vmstat` | `<간격> <횟수>` `-s` `-d` `-a` | 메모리·스왑·I/O·CPU 요약 (`si`/`so` = 스왑 in/out) | 06 |
| `iostat` | `-x` `-d` `-c` `-k/-m` `<간격>` | 장치별 I/O 통계 (`%util`, `await`) | 06 |
| `sar` | `-u` `-r` `-b` `-n DEV` `-q` `-f` `-A` | 과거·현재 시스템 활동 기록 (`sysstat`) | 06 |
| `at` | `now +5min` `-l`(atq) `-d`(atrm) `-c` `-f` | **1회성** 예약 실행 | 06 |
| `nc` | `-z` `-v` `-l` `-u` `-w` `-p` | 포트 점검·간이 서버/클라이언트 | 08 |
| `tcpdump` | `-i` `-nn` `-c` `-w` `-r` `-A` `port` `host` `and/or` | 패킷 캡처·분석 | 08, 10 |
| `nmcli` | `con show` `con mod` `con up/down` `dev status` `ipv4.addresses` `ipv4.method manual` `general` | NetworkManager CLI (영구 설정) | 08 |
| `rsync` | `-a` `-v` `-z` `-n` `--delete` `--exclude` `--exclude-from` `--bwlimit` `--partial` `-H -A -X` `-c` `--link-dest` `-e "ssh -p"` `--progress` | 증분 동기화·미러·원격 백업 | 08, 12 |
| `dump` | `-0`~`-9` `-u` `-f` `-a` `-j/-z` `-b` `-L` | ext2/3/4 **파일시스템 단위** 레벨 백업 | 12 |
| `restore` | `-t` `-r` `-x` `-i` `-C` `-f` `-v` | dump 아카이브 목록·복원·비교 | 12 |
| `gpg` | `-c`(대칭) `-e -r`(공개키) `-d` `--gen-key` `--list-keys` `--export` `--import` `-o` | 암호화·복호화·서명 | 10, 12 |
| `openssl` | `enc -aes-256-cbc -salt` `dgst -sha256` `genrsa` `req -x509` `rsa` `s_client` `x509 -text` | 암호화·해시·인증서 도구 | 10 |
| `umask` | `-S` `0022` `0077` `0002` | 신규 파일·디렉터리 기본 권한 마스크 | 03 |
| `setfacl` | `-m` `-x` `-b` `-R` `-d`(기본 ACL) `-M` | POSIX ACL 설정 (`u:이름:권한`) | 03 |
| `getfacl` | `-p` `-R` `-d` | ACL 조회 (`ls -l` 의 `+` 표시와 대응) | 03 |
| `lsattr` | `-a` `-d` `-R` | 파일 확장 속성 조회 (`chattr` 결과 확인) | 03 |
| `last` | `-x` `-n` `-f` `-F` `reboot` | 로그인·재부팅 이력 (`/var/log/wtmp`) | 07 |
| `lastb` | `-n` `-a` | **로그인 실패** 이력 (`/var/log/btmp`, root 전용) | 07, 10 |
| `wc` | `-l` `-w` `-c` `-m` | 행·단어·바이트 수 세기 | 04 |
| `cut` | `-d` `-f` `-c` `-b` `--complement` | 구분자·문자 위치 기준 필드 추출 | 04 |
| `sort` | `-n` `-r` `-k` `-t` `-u` `-h` `-M` | 정렬 (`-t:` `-k3 -n` 조합 빈출) | 04 |
| `uniq` | `-c` `-d` `-u` `-i` | 인접 중복 행 처리 (**sort 선행 필수**) | 04 |
| `tr` | `-d` `-s` `-c` `'a-z' 'A-Z'` | 문자 치환·삭제·압축 | 04 |
| `head` | `-n` `-c` `-q` | 앞부분 출력 (기본 10행) | 04 |
| `tail` | `-n` `-f` `-F` `-c` `+N` | 뒷부분 출력·실시간 추적 (`-f`) | 04 |
| `ln` | `-s`(심볼릭) `-f` `-n` `-r` | 하드링크·심볼릭 링크 생성 | 04 |
| `stat` | `-c '%i %h %s %U %a %n'` `-f` | inode·링크수·권한·시각 상세 | 04 |
| `file` | `-b` `-i` `-L` `-s` | 파일 유형 판별 (매직 넘버 기반) | 04 |
| `which` | `-a` | PATH 상의 실행 파일 경로 | 01 |
| `whereis` | `-b` `-m` `-s` | 실행 파일·man·소스 위치 일괄 | 01 |
| `man` | `-k`(apropos) `-f`(whatis) `-a` `<섹션번호>` | 매뉴얼 조회 (1=명령, 5=파일형식, 8=관리자) | 01 |
| `alias` | `-p` `unalias` `unalias -a` | 명령 별칭 정의·해제 | 01 |
| `history` | `-c` `-d` `-w` `!!` `!N` `!문자열` `Ctrl+R` | 명령 이력 조회·재실행 | 01 |
| `echo` | `-n` `-e` `$?` `$$` `$PATH` | 문자열·변수 출력 | 04 |
| `printf` | `'%s\n'` `'%-10s'` `'%d'` | 서식 지정 출력 (스크립트용) | 04 |
| `date` | `+%F` `+%Y%m%d` `+%s` `+%u` `-d '1 hour ago'` `-s` | 시각 출력·형식 지정 | 04 |

### 12-4. 121~161 — 시스템 정보 · 압축 · 쿼터 · 서비스 점검

| 명령 | 핵심 옵션 | 한 줄 용도 | 파트 |
| --- | --- | --- | --- |
| `cal` | `-y` `-3` `<월> <연>` | 달력 출력 | 01 |
| `uname` | `-a` `-r` `-m` `-s` `-n` `-v` | 커널·아키텍처·호스트 정보 | 01 |
| `hostnamectl` | `status` `set-hostname` `--static` | 호스트명 조회·변경 (`/etc/hostname`) | 01 |
| `timedatectl` | `status` `set-timezone` `set-ntp` `list-timezones` | 시각·시간대·NTP 동기화 설정 | 01, 08 |
| `localectl` | `status` `set-locale` `list-locales` `set-keymap` | 로케일·키맵 설정 | 01 |
| `free` | `-h` `-m` `-g` `-s` `-t` | 메모리·스왑 사용량 (`available` 이 실사용 여유) | 01 |
| `df` | `-h` `-T` `-i` `-a` `--total` | 파일시스템 **여유 공간**·inode | 05 |
| `uptime` | `-p` `-s` | 가동 시간·로그인 수·load average | 06, 12 |
| `w` | `-h` `-s` `-u` | 로그인 사용자와 각자 실행 중인 작업 | 01 |
| `lsof` | `-i` `-i:포트` `-p` `-u` `+D` `-n` | 열린 파일·소켓 조회 (`umount busy` 원인 추적) | 06 |
| `pgrep` | `-a` `-u` `-f` `-l` `-x` `-n` | 이름·조건으로 PID 검색 | 06 |
| `pkill` | `-9` `-u` `-f` `-t` | 이름·조건으로 시그널 전송 | 06 |
| `killall` | `-9` `-u` `-i` `-w` | **프로세스 이름**으로 일괄 종료 | 06 |
| `pstree` | `-p` `-u` `-a` `-h` | 프로세스 부모-자식 트리 | 06 |
| `watch` | `-n` `-d` `-t` | 명령을 주기적으로 재실행해 변화 관찰 | 06 |
| `tee` | `-a` | 표준출력을 화면과 파일에 동시 기록 | 04 |
| `xargs` | `-n` `-I{}` `-0` `-P` `-r` | 표준입력을 인자로 변환해 명령 실행 | 04 |
| `diff` | `-r` `-q` `-u` `-y` `-i` `-N` | 파일·디렉터리 차이 비교 (`-u` 는 패치 형식) | 04, 12 |
| `patch` | `-p0` `-p1` `-R` `-b` | diff 결과를 원본에 적용·되돌리기 | 04 |
| `gzip` | `-d`(gunzip) `-c` `-9` `-l` `-k` `-r` | gzip 압축·해제 (`.gz`) | 04 |
| `bzip2` | `-d`(bunzip2) `-c` `-k` `-9` | bzip2 압축·해제 (`.bz2`) | 04 |
| `xz` | `-d`(unxz) `-c` `-k` `-9` `-T0` | xz 압축·해제 (`.xz`, 압축률 최고) | 04 |
| `zip` | `-r` `-e` `-9` `-x` | zip 아카이브 생성 (Windows 호환) | 04 |
| `unzip` | `-l` `-d` `-o` `-q` | zip 아카이브 목록·추출 | 04 |
| `cpio` | `-o`(생성) `-i`(추출) `-d` `-v` `-m` `-t` `--no-absolute-filenames` | 표준입력 파일 목록 기반 아카이브 (`rpm2cpio` 와 짝) | 04 |
| `split` | `-b` `-l` `-d` `-a` | 큰 파일을 여러 조각으로 분할 (`cat` 으로 결합) | 04 |
| `sha256sum` | `-c` `--quiet` `-b` | SHA-256 해시 생성·검증 (macOS 는 `shasum -a 256`) | 04, 12 |
| `md5sum` | `-c` `--quiet` | MD5 해시 생성·검증 (충돌 취약 — 무결성 확인용만) | 04 |
| `ldd` | `-v` `-u` | 실행 파일의 공유 라이브러리 의존성 | 02 |
| `ldconfig` | `-p` `-v` `-n` | 라이브러리 캐시(`/etc/ld.so.cache`) 갱신·조회 | 02 |
| `make` | `-j` `-f` `install` `clean` `-n` | Makefile 기반 빌드·설치 | 02 |
| `configure` | `--prefix=` `--enable-` `--disable-` `--with-` `--help` | 소스 빌드 환경 검사·Makefile 생성 (1단계) | 02 |
| `quotaon` | `-a` `-u` `-g` `-v` `-p` (`quotaoff`) | 파일시스템 쿼터 활성화·상태 확인 | 05 |
| `edquota` | `-u` `-g` `-t`(유예기간) `-p`(복제) | 사용자·그룹 쿼터 한계 편집 | 05 |
| `repquota` | `-a` `-u` `-g` `-s` `-v` | 쿼터 사용 현황 보고 | 05 |
| `named-checkzone` | `<존이름> <존파일>` | BIND 존 파일 문법 검사 (`named-checkconf` 는 설정) | 09 |
| `rndc` | `status` `reload` `reconfig` `flush` `querylog` | BIND 원격 제어 | 09 |
| `smbclient` | `-L` `-U` `//서버/공유` `-c` | Samba 클라이언트 — 공유 목록·접속 검증 | 09 |
| `postconf` | `-n` `-d` `-e` `-m` | Postfix 설정 조회·변경 (`-n` 은 기본값과 다른 것만) | 09 |
| `mailq` | (= `postqueue -p`) `postqueue -f` `postsuper -d` | 메일 큐 조회·강제 전송·삭제 | 09 |
| `shutdown` | `-h` `-r` `-P` `-H` `-c` `-k` `--no-wall` `now` `+N` `hh:mm` | 예약 종료·재부팅·취소 (+ 브로드캐스트) | 12 |

**검증 (자기 점검용 무작위 확인)**

```bash
# 표에서 임의로 고른 명령의 옵션이 실제로 존재하는지 즉석 확인
for c in tar rsync dd dump restore mdadm lvcreate firewall-cmd semanage docker; do
  printf '%-14s : ' "$c"; command -v "$c" >/dev/null && echo "설치됨" || echo "미설치 — 해당 파트 확인"
done
man -k backup 2>/dev/null | head -5
```

```text
tar            : 설치됨
rsync          : 설치됨
dd             : 설치됨
dump           : 설치됨
restore        : 설치됨
mdadm          : 설치됨
...
```

> 📝 **시험 포인트**: 옵션 대문자·소문자 혼동이 가장 큰 실점 요인 — `ssh -p` vs `scp -P`, `tar -C`(디렉터리) vs `-c`(생성), `rsync -a`(아카이브) vs `-A`(ACL), `find -perm -4000` vs `4000`, `rpm -qf`(파일→패키지) vs `-ql`(패키지→파일).

---

## 13. X 윈도 요약 (※ 미실행 — UTM minimal 설치라 GUI 없음)

> 이 LAB 의 VM 은 Rocky 9 **minimal** 로 설치해 X 서버·데스크톱 환경이 없다. 아래 명령은 **실행하지 않고** 필기 대비용으로 구조·파일·옵션만 정리한다. 설치를 원할 경우의 명령은 13-5 에 명시하되 역시 미실행.

### 13-1. 구조 — X 서버와 X 클라이언트가 직관과 반대

> **상황**: X 윈도는 "사용자 앞의 기계가 **서버**" 다. 일반적인 서버-클라이언트 감각과 반대여서 매 회차 함정 보기로 나온다.

```bash
# ※ 미실행 — GUI 설치 시 확인할 수 있는 명령 (현재 VM 에는 없음)
#   ps -ef | grep Xorg          # X 서버 프로세스
#   xdpyinfo | head             # 현재 디스플레이 정보
#   xrandr                      # 해상도·출력 장치
#   glxinfo | head              # OpenGL 렌더러
```

| 요소 | 역할 | 위치 |
| --- | --- | --- |
| **X 서버** | 디스플레이·키보드·마우스 **입출력 하드웨어 제어** | **사용자 앞 로컬 머신** |
| **X 클라이언트** | 응용 프로그램(터미널·브라우저 등) — 서버에 그리기를 요청 | 로컬 또는 **원격 서버** |
| X 프로토콜 | 서버-클라이언트 통신 규약, **네트워크 투명성** | TCP 6000+N 또는 유닉스 소켓 |
| Xlib / XCB | X 프로토콜 저수준 C 라이브러리 | 클라이언트 측 |
| XFree86 → X.org | 오픈소스 X 구현 (라이선스 문제로 분기, 현행은 **X.org**) | — |
| 윈도 매니저 | 창 배치·테두리·이동 관리 (Mutter, KWin, Openbox) | X 클라이언트의 일종 |
| 데스크톱 환경 | 윈도 매니저 + 파일관리자 + 패널 통합 (GNOME, KDE, Xfce, LXDE) | — |
| 디스플레이 매니저 | 그래픽 로그인 화면 (`gdm`, `kdm`, `xdm`, `lightdm`, `sddm`) | 부팅 시 자동 기동 |
| Wayland | X11 후속 디스플레이 프로토콜 — RHEL 9 GNOME 기본 세션 | `XWayland` 로 X 앱 호환 |

- 원격 실행 흐름: **원격 호스트의 X 클라이언트** → X 프로토콜 → **로컬 X 서버** → 로컬 화면에 표시
- 즉 "서버에서 GUI 프로그램을 띄워 내 PC 화면에 보이게 한다" 는 상황에서 **내 PC 가 X 서버**

**검증** (현재 VM 에서 확인 가능한 것만)

```bash
rpm -qa | grep -c xorg-x11-server          # minimal 설치 → 0
systemctl get-default                       # multi-user.target (GUI 아님)
ls /etc/X11 2>&1 | head -2
echo "${DISPLAY:-<미설정>}"
```

```text
0
multi-user.target
ls: cannot access '/etc/X11': No such file or directory
<미설정>
```

- `DISPLAY` 가 비어 있고 `/etc/X11` 이 없음 = X 미설치 상태 확인

> 📝 **시험 포인트**: 필기 FULL r01-17 · r02-13 · r07-8 — "X 서버는 사용자 측, X 클라이언트는 응용 프로그램" 이 정답 축. "X 윈도는 커널에 포함되어 분리 불가" 는 **오답**(사용자 영역 응용).

### 13-2. `DISPLAY` 환경변수와 원격 표시

> **상황**: X 클라이언트는 `DISPLAY` 를 보고 어느 X 서버에 그릴지 결정한다. 형식과 포트 대응이 그대로 출제된다.

```bash
# ※ 미실행 — 형식만 확인
#   export DISPLAY=:0.0                      # 로컬 첫 번째 디스플레이·첫 화면
#   export DISPLAY=192.168.5.10:0            # 원격 호스트의 X 서버로 출력
#   export DISPLAY=192.168.5.10:1.0
#   xterm &                                  # 위 DISPLAY 가 가리키는 화면에 뜸
#   echo $DISPLAY
```

- 형식: **`호스트:디스플레이번호.화면번호`** (`host:D.S`)
  - **호스트** — 생략 시 로컬(유닉스 소켓). IP·호스트명을 쓰면 TCP 로 원격 X 서버 접속
  - **디스플레이번호** — X 서버 인스턴스 번호. TCP 포트 = **6000 + 디스플레이번호** (`:0` → 6000)
  - **화면번호** — 멀티 모니터 중 화면 (보통 `.0`, 생략 가능)
- 원격 표시 2가지 경로
  - **직접 지정**: 원격에서 `export DISPLAY=<내PC>:0` + 내 PC 에서 `xhost +` → 평문 전송, **보안 취약**
  - **SSH X 포워딩**: `ssh -X <서버>` → 접속 후 `DISPLAY` 가 `localhost:10.0` 처럼 자동 설정되고 트래픽이 **SSH 로 암호화** → 권장. `-Y`(신뢰 모드), 서버 측 `sshd_config` 의 `X11Forwarding yes` 필요
- 최근 배포판은 X 서버가 기본적으로 **TCP 리스닝을 끔**(`-nolisten tcp`) → 직접 지정 방식은 별도 설정 필요

| DISPLAY 값 | 의미 | TCP 포트 |
| --- | --- | --- |
| `:0` / `:0.0` | 로컬 X 서버, 첫 디스플레이·첫 화면 | 6000 (유닉스 소켓 우선) |
| `:1` | 로컬 두 번째 X 서버 (`startx -- :1`) | 6001 |
| `192.168.5.10:0` | 해당 호스트의 X 서버 | 6000 |
| `localhost:10.0` | SSH X 포워딩 터널 | 6010 |

**검증**

```bash
grep -E '^\s*X11Forwarding' /etc/ssh/sshd_config
echo "DISPLAY=${DISPLAY:-<미설정>}"
ss -tln | grep -c ':600'          # X 서버 없음 → 0
```

```text
X11Forwarding yes
DISPLAY=<미설정>
0
```

> 📝 **시험 포인트**: 필기 FULL r04-13 — `export DISPLAY=192.168.5.10:0` 실행 후 X 클라이언트는 **192.168.5.10 의 화면**에 표시. 포트 = 6000 + 디스플레이 번호.

### 13-3. 접근 제어 — `xhost` vs `xauth`

> **상황**: 내 X 서버에 아무 호스트나 그리게 두면 화면 캡처·키 입력 가로채기가 가능하다. 호스트 단위(`xhost`) 와 사용자 단위(`xauth`) 두 방식의 차이가 출제 핵심.

```bash
# ※ 미실행 — 형식만
#   xhost                        # 현재 허용 목록 조회
#   xhost +192.168.1.10          # 특정 호스트 허용
#   xhost -192.168.1.10          # 허용 해제
#   xhost +                      # ⚠️ 모든 호스트 허용 (보안상 금지)
#   xhost -                      # 접근 제어 활성화(기본 목록만 허용)
#
#   xauth list                   # 쿠키 목록 (~/.Xauthority)
#   xauth add <호스트>:0 MIT-MAGIC-COOKIE-1 <쿠키값>
#   xauth remove <호스트>:0
#   xauth extract - <호스트>:0 | ssh <원격> xauth merge -
```

| 구분 | `xhost` | `xauth` |
| --- | --- | --- |
| 통제 단위 | **호스트(IP)** 단위 | **사용자(쿠키)** 단위 |
| 인증 방식 | 목록에 있으면 무조건 허용 | **MIT-MAGIC-COOKIE-1** 등 쿠키 일치 |
| 저장 위치 | X 서버 메모리 (세션 한정) | `~/.Xauthority` 파일 |
| 보안 | 낮음 — 그 호스트의 **모든 사용자**가 접근 | 높음 — 쿠키를 가진 사용자만 |
| 대표 위험 | `xhost +` = 전 세계 허용 | 쿠키 파일 유출 시 동일 위험 |
| 시험 표현 | "호스트 단위 접근 허용" | "MIT-MAGIC-COOKIE-1 기반 사용자 인증" |

- `ssh -X` 는 내부적으로 **xauth 쿠키를 원격에 전달**하므로 `xhost` 를 열 필요가 없음 → 실무 권장 경로
- `~/.Xauthority` 는 세션마다 갱신됨. `su` 로 사용자 전환 후 GUI 앱이 안 뜨는 전형적 원인이 이 파일 접근 권한

**검증**

```bash
ls -l ~/.Xauthority 2>&1 | tail -1
which xhost xauth 2>&1 | tail -2
rpm -q xorg-x11-xauth 2>&1 | tail -1
```

```text
ls: cannot access '/root/.Xauthority': No such file or directory
/usr/bin/which: no xhost in (...)
package xorg-x11-xauth is not installed
```

> 📝 **시험 포인트**: 필기 FULL r05-12 — "호스트 단위 허용" = `xhost +192.168.1.10`. r05-13 · r09-10 — "쿠키 기반 사용자 단위 인증" = `xauth`. `xhost +` 는 **보안 위험**으로 오답 유도에 자주 사용.

### 13-4. `/etc/X11/xorg.conf` 섹션과 관련 파일

> **상황**: 최신 배포판은 자동 감지로 `xorg.conf` 없이 동작하지만, 섹션 이름과 역할은 여전히 출제 대상이다.

```bash
# ※ 미실행 — GUI 설치 시 생성·확인 절차
#   X -configure                       # /root/xorg.conf.new 생성
#   Xorg -configure
#   cp /root/xorg.conf.new /etc/X11/xorg.conf
#   ls /etc/X11/xorg.conf.d/           # 조각 설정 디렉터리 (현행 방식)
#   cat /var/log/Xorg.0.log | head     # X 서버 기동 로그
```

| 섹션 | 역할 |
| --- | --- |
| `ServerLayout` | 전체 구성 묶음 — 어떤 Screen·InputDevice 를 쓸지 연결 |
| `Files` | 폰트 경로·모듈 경로 등 파일 위치 |
| `Module` | 동적으로 적재할 X 서버 모듈(확장 기능) |
| `InputDevice` | 키보드·마우스 등 입력 장치 정의 |
| `Monitor` | 모니터 사양 — 수평/수직 주파수(HorizSync·VertRefresh) |
| `Device` | 그래픽 카드(비디오 어댑터)와 드라이버 |
| `Screen` | **Monitor + Device 결합** — 해상도·색상 깊이(Display 서브섹션) |
| `ServerFlags` | 전역 옵션 (예: `DontZap`) |

- 관련 파일: `/etc/X11/xorg.conf`(단일) · `/etc/X11/xorg.conf.d/*.conf`(조각, 현행) · `/var/log/Xorg.0.log`(기동 로그) · `~/.xinitrc` · `~/.xsession` · `/etc/X11/xinit/xinitrc`
- 문제 발생 시 진단 순서: `/var/log/Xorg.0.log` 의 `(EE)` 행 확인 → 드라이버(`Device` 섹션) → 모니터 주파수(`Monitor`)

**검증**

```bash
ls /etc/X11 /etc/X11/xorg.conf.d 2>&1 | head -3
ls /var/log/Xorg.0.log 2>&1 | tail -1
rpm -qa | grep -c '^xorg-x11'
```

```text
ls: cannot access '/etc/X11': No such file or directory
ls: cannot access '/var/log/Xorg.0.log': No such file or directory
0
```

> 📝 **시험 포인트**: `Screen` 섹션이 **Monitor 와 Device 를 묶는다**는 관계가 자주 출제. `ServerLayout` 은 최상위 묶음.

### 13-5. GUI 설치·전환 절차와 원격 X (※ 미실행)

> **상황**: 시험에서는 "텍스트 모드에서 GUI 로 전환하는 방법" 을 묻는다. 이 VM 은 리소스·목적상 설치하지 않으므로 명령만 제시한다.

```bash
# ※ 미실행 — GUI 를 실제로 올릴 때의 절차
#   dnf group list | grep -i gui
#   dnf group install -y "Server with GUI"        # 또는 "Workstation"
#   systemctl set-default graphical.target        # 기본 타겟 변경 (런레벨 5)
#   systemctl isolate graphical.target            # 재부팅 없이 즉시 전환
#   systemctl get-default                         # graphical.target 확인
#   startx                                        # 텍스트 로그인 상태에서 X 세션 수동 시작
#   startx -- :1                                  # 두 번째 디스플레이로 시작
#   xinit                                         # startx 의 하위 계층 (기본 클라이언트만)
#   systemctl set-default multi-user.target       # 다시 텍스트 모드로
```

| 항목 | 명령·값 | 비고 |
| --- | --- | --- |
| GUI 패키지 그룹 | `dnf group install "Server with GUI"` | GNOME + 서버 도구 |
| 기본 부팅 모드 GUI | `systemctl set-default graphical.target` | 런레벨 5 대응 |
| 기본 부팅 모드 텍스트 | `systemctl set-default multi-user.target` | 런레벨 3 대응 |
| 즉시 전환 | `systemctl isolate graphical.target` | 재부팅 불필요 |
| 텍스트 로그인에서 X 실행 | `startx` | `~/.xinitrc` 참조 |
| 디스플레이 매니저 | `gdm`(GNOME) `kdm`/`sddm`(KDE) `lightdm` `xdm` | `systemctl status gdm` |
| 현재 런레벨 확인 | `runlevel`, `who -r`, `systemctl get-default` | `N 3` = 이전 없음·현재 3 |

| 원격 X 방식 | 명령·구성 | 특징 |
| --- | --- | --- |
| SSH X11 포워딩 | `ssh -X user@host` (서버 `X11Forwarding yes`, `xorg-x11-xauth` 필요) | 암호화, 앱 단위 전달, 설정 간단 |
| `ssh -Y` | 신뢰(trusted) X11 포워딩 | 일부 앱 호환 문제 해결, 보안은 `-X` 보다 느슨 |
| DISPLAY 직접 지정 | `export DISPLAY=<내PC>:0` + `xhost +<서버>` | 평문 전송, 방화벽 6000 개방 필요 — **비권장** |
| VNC | `tigervnc-server`, `vncserver :1`, 5900+N 포트 | **전체 데스크톱** 전달, 세션 유지 |
| RDP | `xrdp` (3389/tcp) | Windows 원격 데스크톱 클라이언트로 접속 |
| X2Go / NX | 압축·캐싱으로 저대역폭 최적화 | 참고 |

**검증** (현재 VM 에서 가능한 조회만)

```bash
systemctl get-default
dnf group list --available 2>/dev/null | grep -iE 'gui|workstation' | head -3
ls /usr/lib/systemd/system/graphical.target
systemctl list-unit-files | grep -E '^gdm|^lightdm' | head -2
runlevel; who -r
```

```text
multi-user.target
   Server with GUI
   Workstation
/usr/lib/systemd/system/graphical.target
N 3
         run-level 3  ...
```

- `graphical.target` 유닛 파일 자체는 존재하지만 GUI 패키지가 없어 전환해도 화면이 뜨지 않음 → **실행하지 않음**

> 📝 **시험 포인트**: 필기 FULL r04-7 — 런레벨 5 = "X 윈도 기반 다중 사용자". r04-12 — 런레벨 3 텍스트 모드에서 X 구동 = **`startx`**(`xhost`·`xauth`·`export DISPLAY` 는 오답). 기본 모드 변경은 `systemctl set-default`.

---

## 14. 마무리

### 14-1. 실습 환경 정리 — 스냅샷 복원 vs 보존

> **상황**: 12개 파트가 끝났다. VM 을 어떻게 둘지 먼저 정해야 이후 정리 범위가 달라진다.

```bash
# 판단 근거 — 현재 사용량과 자원 목록
df -hT | grep -vE 'tmpfs|devtmpfs'
du -sh /srv/raid/backup /data /srv/share /var/log /var/lib/docker 2>/dev/null
docker system df
systemctl list-units --type=service --state=running --no-pager | wc -l
```

| 선택 | 방법 | 장점 | 단점 |
| --- | --- | --- | --- |
| **보존(권장)** | 그대로 두고 14-2 의 임시 파일만 정리 | 실기 재현·오답 역추적에 그대로 사용 | 디스크 점유 유지 |
| 스냅샷 복원 | UTM 스냅샷(QEMU 백엔드) 으로 Part 01/05/10 시점 복귀 | 깨끗한 재시작 | 이후 파트 자원 전부 소실 |
| VM 복제 | UTM → VM 우클릭 → 복제 후 한쪽만 정리 | 원본 보존 + 실험 가능 | 디스크 2배 |
| 완전 삭제 | UTM 에서 VM 삭제 | 공간 회수 | **오프사이트 백업(macOS `~/lab-backup`) 확인 후에만** |

- 시험 직전까지는 **보존**을 권장 — 11절 체크리스트를 실제 명령으로 다시 돌려 보는 것이 최고의 복습
- 삭제를 택한다면 반드시 14-3 의 macOS 백업 확인을 **먼저** 수행

**검증**

```bash
df -h / /data /srv/raid /srv/share | tail -4
docker system df
ls ~/lab-backup 2>/dev/null || echo "(macOS 에서 확인)"
```

```text
/dev/mapper/rl-root  ...  ...% /
/dev/vdb1            2.0G ...% /data
/dev/md0             5.0G ...% /srv/raid
...
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          ...       ...       ...       ...
```

> 📝 **시험 포인트**: 정리 자체는 출제 대상이 아니지만 `docker system df`·`du -sh`·`df -i` 로 "무엇이 공간을 먹는지" 판단하는 절차는 실기 진단 문항과 동일한 사고 흐름.

### 14-2. 더미·부하 파일 정리

> **상황**: 실습 중 만든 난수 파일·부하 테스트 산출물·중복 아카이브가 수 GB 를 차지한다. **백업 산출물과 검증용 기준 파일은 남기고** 임시물만 지운다.

⚠️ 삭제 목록에 `/srv/raid/backup/lab-*.tar.gz`·`*.sha256`·`accounts/` 가 **들어가지 않도록** 경로를 확인

```bash
# ① 삭제 대상 미리보기 (실행 전 반드시 확인)
ls -lh /data/proj/blob.bin /data/proj/inc.bin /srv/share/churn.bin 2>/dev/null
ls -lh /srv/raid/backup/cmp.tar* /srv/raid/backup/data-excl*.tar.gz /srv/raid/backup/data-plain.tar /srv/raid/backup/data-verify.tar 2>/dev/null
du -sh /tmp/rs-a /tmp/rs-b /tmp/rs-noh /tmp/rs-hax /tmp/restore /tmp/restore-attr /tmp/restore-strip /tmp/stage /tmp/dump-restore 2>/dev/null

# ② 임시 산출물 제거
rm -f /data/proj/blob.bin /data/proj/inc.bin /srv/share/churn.bin
rm -f /srv/raid/backup/cmp.tar*
rm -f /srv/raid/backup/data-excl.tar.gz /srv/raid/backup/data-excl2.tar.gz
rm -f /srv/raid/backup/data-plain.tar /srv/raid/backup/data-verify.tar
rm -rf /tmp/rs-a /tmp/rs-b /tmp/rs-noh /tmp/rs-hax /tmp/restore /tmp/restore-attr /tmp/restore-strip /tmp/stage /tmp/dump-restore
rm -rf /srv/raid/mirror/limited /srv/raid/mirror/proj

# ③ Docker 정리 (⚠️ -a 는 미사용 이미지까지 전부 삭제)
docker system df
docker system prune -f                 # 중지된 컨테이너·미사용 네트워크·dangling 이미지
#   docker system prune -a -f          # ⚠️ 사용 중이 아닌 모든 이미지까지 (재빌드 필요)
#   docker volume prune -f             # ⚠️ redis-data 가 컨테이너에 연결돼 있어야 안전

# ④ 로그·저널·패키지 캐시
journalctl --vacuum-size=200M
dnf clean all
find /var/log -name '*.gz' -mtime +30 -delete

# ⑤ 오래된 백업 산출물 (backup.sh 의 보존 정책과 동일 기준)
find /srv/raid/backup -maxdepth 1 -name 'lab-*.tar.gz' -mtime +30 -print
```

- `docker system prune -f` : 확인 없이(**f**orce) 중지 컨테이너·미사용 네트워크·dangling 이미지·빌드 캐시 제거. `-a` 는 **참조되지 않는 모든 이미지**까지 → `lab/ubuntu-tools` 재빌드 필요
- `docker volume prune` : 어느 컨테이너도 참조하지 않는 볼륨 삭제 — `lab-redis` 가 실행 중이면 `redis-data` 는 보호됨
- `journalctl --vacuum-size=<크기>` : 저널 총량 상한 지정 정리. `--vacuum-time=30d` 도 가능
- `dnf clean all` : 다운로드 캐시·메타데이터 제거 (`/var/cache/dnf`)

**검증**

```bash
df -h /data /srv/raid /srv/share | tail -3
du -sh /srv/raid/backup
ls /srv/raid/backup | head -20
docker system df
journalctl --disk-usage
/usr/local/bin/final-check.sh | tail -1
```

```text
/dev/vdb1  2.0G  ...  /data
/dev/md0   5.0G  ...  /srv/raid
...
===== 전체 정상 (NG=0) =====
```

- 정리 후에도 `final-check.sh` 가 `NG=0` 이면 **필요한 자원은 그대로** 라는 뜻

> 📝 **시험 포인트**: 디스크 부족 대응 순서 — `df -h`(어느 FS) → `du -sh /* \| sort -h`(어느 디렉터리) → `find -size +100M`(어느 파일) → `lsof \| grep deleted`(삭제됐지만 열려 있는 파일). 마지막 항목이 "지웠는데 용량이 안 줄어드는" 전형적 함정.

### 14-3. macOS 백업 보관 확인과 다음 학습

> **상황**: 마지막으로 오프사이트 사본이 실제로 열리는지 확인하고, 남은 학습 경로를 정한다.

```bash
# ① 오프사이트 최종 동기화 + 목록·해시 검증
rsync -avz --delete /srv/raid/backup/ $MACUSER@$MAC:~/lab-backup/
ssh $MACUSER@$MAC 'cd ~/lab-backup && ls -1 | wc -l && du -sh . && shasum -a 256 -c backup.sha256 2>&1 | tail -3'

# ② macOS 에서 실제로 열리는지 (아카이브가 손상되지 않았는지)
#   $ tar tzf ~/lab-backup/lab-full-*.tar.gz | head
#   $ gunzip -t ~/lab-backup/*.gz && echo "gzip 무결성 OK"

# ③ 이 LAB 의 최종 상태 스냅샷을 문서로 남기기
/usr/local/bin/final-check.sh > /srv/raid/backup/final-check-$(date +%F).txt
rpm -qa --qf '%{NAME}\n' | sort > /srv/raid/backup/pkglist-final.txt
lsblk -f > /srv/raid/backup/lsblk-final.txt
```

- `gunzip -t <파일>` : 압축 파일 무결성만 검사 (**t**est) — 해제하지 않고 CRC 확인
- macOS 는 GNU coreutils 가 아니므로 `sha256sum`·`du --max-depth` 대신 `shasum -a 256`·`du -d` 사용

**다음 학습 경로**

| 순서 | 할 일 | 방법 |
| --- | --- | --- |
| 1 | 실기 재풀이 | `EXAM-PRACTICAL/round-01` ~ `round-06` 를 **답 보지 않고** 다시 풀기 |
| 2 | 오답 역추적 | 틀린 문항의 명령을 12절 표에서 찾아 **파트 번호** 확인 → 해당 LAB 문서의 그 단계만 재수행 |
| 3 | 필기 재풀이 | `EXAM-WRITTEN-FULL/round-01` ~ `round-10`, 특히 반복 출제 주제(fstab 5필드·런레벨·dump 레벨·MBR/GPT) |
| 4 | 이론 정리 | [[../THEORY/linux-basics]] 등 12종을 11절 체크리스트 순서로 훑기 |
| 5 | 커버리지 확인 | [[README]] **5절 기출 커버리지 요약** 표로 빠진 주제 점검 |
| 6 | 최종 리허설 | 시험 전날 12절 명령어 161선을 "용도 → 명령" 방향으로 역암기 |

- 오답 역추적 예: `tar --listed-incremental` 을 틀렸다 → 12절 표에서 `tar` 행의 파트 `04, 12` 확인 → 이 문서 **2-1 · 2-2 · 9-1** 재수행
- 시간이 부족하면 **11절 체크리스트의 검증 명령만** 위에서 아래로 실행 — 손이 기억하는지 확인하는 가장 빠른 방법

**검증**

```bash
ls -lh /srv/raid/backup/final-check-*.txt /srv/raid/backup/pkglist-final.txt
tail -1 /srv/raid/backup/final-check-$(date +%F).txt
ssh $MACUSER@$MAC 'ls ~/lab-backup | wc -l'
```

```text
-rw-r--r-- 1 root root ... final-check-2026-09-03.txt
-rw-r--r-- 1 root root ... pkglist-final.txt
===== 전체 정상 (NG=0) =====
...
```

> 📝 **시험 포인트**: "백업은 복원해 봐야 백업" — 이 파트에서 실제로 지우고 되살린 5가지(tar 증분·dd 이미지·dump 체인·xfsdump·Docker 볼륨)가 곧 실기 서술형의 답안 골격.

---

## 체크리스트

| 수행 항목 | 명령 | 확인 방법 | ☐ |
| --- | --- | --- | --- |
| 백업 대상지·전략 확정 | `findmnt /srv/raid`, `mkdir -p /srv/raid/{backup,mirror,snap}` | `df -hT /srv/raid`, `[UU]` | ☐ |
| tar 레벨 0 (전체) | `tar -czvf data-full.tar.gz -g data.snar /data /srv/share` | `file data.snar` = incremental snapshot | ☐ |
| tar 레벨 1 (증분) | 같은 snar 재사용 | 아카이브에 변경 파일만, 크기 KB | ☐ |
| tar 차등 | level0 snar 복사본 사용 | 2회차에 누적 변경분 전부 포함 | ☐ |
| tar 보존·제외 옵션 | `--exclude` `--exclude-from` `--acls --selinux --xattrs -p` `-C` `--strip-components` | `getfacl`·`ls -Z` 대조 | ☐ |
| 압축 3종 비교 | `-z` `-j` `-J` + `time` | xz < bz2 < gz 크기 순 | ☐ |
| tar 복구 리허설 | `rm -rf /data/proj` → 레벨0 → 레벨1 (`-g /dev/null`) | `sha256sum -c` 전부 OK, file30 삭제 반영 | ☐ |
| /etc 백업·선택 복원·암호화 | `tar czvf etc-$(date +%F).tar.gz /etc`, `gpg -c` | 특정 파일 추출 후 `diff`, `sshd -t` | ☐ |
| 계정·패키지·crontab 덤프 | `getent`, `rpm -qa`, `crontab -l -u` | `shadow.dump` 권한 600 | ☐ |
| rsync 슬래시 차이 재현 | `rsync -a /srv/share /tmp/rs-a/` vs `/srv/share/ /tmp/rs-b/` | 트리 한 겹 차이 확인 | ☐ |
| rsync 미러 + `--delete` | `-avn` 선행 → `-av --delete` | `diff -rq` 차이 없음, orphan 제거 | ☐ |
| rsync 제외·대역폭·재개 | `--exclude-from` `--bwlimit=100` `--partial` `-P` | 제외 파일 0건 | ☐ |
| rsync `-H -A -X` 보존 | 하드링크·ACL·xattr 생성 후 비교 | inode 공유·링크수 2·`user:dev1` | ☐ |
| rsync `--checksum` | 크기·mtime 동일 + 내용만 변경 재현 | 기본은 미감지, `-c` 는 전송 | ☐ |
| `--link-dest` 3세대 | gen1 → gen2 → gen3 | `ls -li` inode 공유, `du -sh` 합계 = 1세대 | ☐ |
| 오프사이트 전송 | `rsync -avz … <mac>:~/lab-backup/`, `-e "ssh -p"` | 원격 `ls`, dry-run 전송 0건 | ☐ |
| 전송 무결성 검증 | `sha256sum > backup.sha256` → 원격 `shasum -a 256 -c` | 전부 `OK` | ☐ |
| 파티션 테이블 백업 | `dd bs=512 count=34`, `sfdisk -d`, `sgdisk --backup`, `blkid`, `lsblk -f` | 17408 B / 64 B 크기 확인 | ☐ |
| dd 진행률 | `status=progress`, `kill -USR1 $(pidof dd)` | 실시간 통계 출력 | ☐ |
| 파티션 이미지 백업 | `umount /data` → `dd if=/dev/vdb1 … conv=fsync` → `gzip -9` | `gunzip -c \| file -` = ext4 | ☐ |
| ⚠️ 이미지 복구 리허설 | `mkfs.ext4 -F /dev/vdb1` → `gunzip -c \| dd of=` → `mount` | UUID 복원, `sha256sum -c` OK | ☐ |
| MBR 446/64/512 구분 | `bs=446` / `bs=1 count=64 skip=446` / `bs=512 count=1` | `hexdump` 끝 `55 aa` | ☐ |
| 소거·ISO 쓰기 개념 (※) | `shred -n 3 -z`, `wipefs -a`, `dd if=iso of=/dev/sdX` | 미실행 — 형태·차이 숙지 | ☐ |
| dump 레벨 0/1/2 | `dump -0uf` `-1uf` `-2uf /dev/vdb1` | `cat /etc/dumpdates` 3줄 | ☐ |
| restore 4모드 | `-tf` `-xf` `-if` `-Cf` | 대화식 `add`→`extract` 성공 | ☐ |
| restore -r 전체 복원 | `/data` 비우고 레벨 0→1→2 | 증분 파일 존재, `restoresymtable` 삭제 | ☐ |
| xfsdump 레벨 0·1 | `-l 0 -L -M -f /srv/share` | `xfsdump -I` 세션 2건 | ☐ |
| xfsrestore 검증 | `xfsrestore -f … /srv/raid/restore-xfs` | `diff -rq` 차이 없음 | ☐ |
| LVM 스냅샷 백업 | `lvcreate -s` → `mount -o ro,nouuid` → `tar` → `lvremove -f` | 산출물 생성, `lvs` 에 스냅샷 없음 | ☐ |
| 스냅샷 자동화 스크립트 | `/usr/local/bin/snap-backup.sh` (`trap` 포함) | `journalctl -t snap-backup` | ☐ |
| 스냅샷 용량 감시 | `lvs -o data_percent`, `lvextend -L +200M` | `lv_attr` 무효(`I`) 아님 | ☐ |
| LVM 메타데이터 | `vgcfgbackup`, `vgcfgrestore -l` | `/etc/lvm/backup/vg_lab` 존재 | ☐ |
| RAID 메타데이터·md127 대응 | `mdadm --detail --scan >> /etc/mdadm.conf` → `dracut -f` | `lsinitrd \| grep mdadm.conf`, 재부팅 후 `md0` | ☐ |
| Docker 볼륨 백업 | `docker run --rm -v redis-data:/data -v /srv/raid/backup:/backup alpine tar czf` | `tar tzf` 에 `dump.rdb` | ☐ |
| Docker 볼륨 복구 | 볼륨 삭제 → 재생성 → 역방향 tar → 컨테이너 재기동 | `redis-cli get labkey` 값 생존 | ☐ |
| 이미지 save/load | `docker save -o` → `docker rmi` → `docker load -i` | `docker images` 복귀, `history` 레이어 | ☐ |
| backup.sh 최종판 작성 | 주간 전체 + 일일 증분, flock·trap·logger·mail·보존·sha256 | `bash -n` 통과, 실행 rc=0 | ☐ |
| 산출물·로그·체크섬 검증 | `ls lab-*.tar.gz*`, `tail /var/log/backup.log`, `sha256sum -c` | 전부 OK | ☐ |
| 중복 실행 차단 | 이중 실행 | 두 번째 rc=3, 로그 `SKIP` | ☐ |
| 실패 경로·메일 | 원본 경로 제거 후 실행 | rc=4, `BACKUP FAIL` 메일 수신 | ☐ |
| 스케줄·logrotate 연동 | `/etc/cron.d/lab-backup`, `systemd-analyze calendar`, `logrotate -d` | 다음 실행 시각 출력 | ☐ |
| 복구 런북 작성 | 장애 12유형 표 + `runbook/` 메타데이터 사본 | 표 완성, 파일 존재 | ☐ |
| shutdown 예약·통지·취소 | `shutdown -r +1 "msg"` → 다른 세션 wall → `shutdown -c` | `/run/nologin` 생성·삭제 | ☐ |
| 종료 명령 대응 확인 | `readlink -f /usr/sbin/{shutdown,init,reboot}` | 모두 `/usr/bin/systemctl` | ☐ |
| final-check.sh 작성 | `/usr/local/bin/final-check.sh` | `[OK]` 38건, `NG=0` | ☐ |
| 실제 재부팅 후 검증 | `systemctl reboot` → 재접속 → 재실행 | 전후 `diff` 동일, `NG=0` | ☐ |
| 부팅 이력 조회 | `last -x`, `uptime -s/-p`, `journalctl --list-boots` | reboot 엔트리 증가 | ☐ |
| 전 범위 종합 체크리스트 수행 | 11절 표의 검증 명령 전부 | 미체크 항목 0 | ☐ |
| 명령어 161선 역암기 | 12절 표를 "용도 → 명령" 방향으로 | 막힘 없이 상기 | ☐ |
| X 윈도 필기 정리 (※) | 13절 — 서버/클라이언트·DISPLAY·xhost/xauth·xorg.conf | 미실행, 개념 암기 | ☐ |
| 임시 파일 정리 | `docker system prune -f`, blob/churn 삭제, `journalctl --vacuum-size` | 정리 후에도 `NG=0` | ☐ |
| macOS 오프사이트 최종 확인 | 원격 `shasum -a 256 -c`, `gunzip -t` | 전부 OK | ☐ |

---

## 기출 연결

| 기출 (문제집·회차·번호 또는 주제) | 이 파트의 단계 |
| --- | --- |
| 필기 FULL r01-33, r03-47, r04-30, r05-33, r06-30, r09-53, r10-29 — `/etc/fstab` **5번 필드 = dump 백업 대상 여부** (반복 최빈출) | 6-1 (fstab 연동), 1-2 도구표 |
| 필기 FULL r01-55 · r02-63 · r04-61, 실기 r04-2 — `dd if=/dev/sda of=mbr.img bs=512 count=1` (MBR 512바이트 백업) | 5-5 바이트 구분표, 5-1 |
| 필기 FULL r01-60 — `tar cvzf backup.tar.gz /home/kim` 동작 해석 | 2-1 |
| 실기 r01-13 — `/home` 을 `home.tar.gz` 로 압축 백업 + `c z v f` 각 의미 서술 | 2-1 옵션 설명 |
| 필기 FULL r01-61 — dump 레벨 0=전체, 1~9=자신보다 **낮은 레벨의 최근 백업 이후** 증분 | 6-1, 6-2 레벨 규칙표 |
| 필기 FULL r01-62 · r03-61 · r10-44 — rsync 델타 전송·`--delete`·"매번 전체 재전송" 오답 | 4-2, 4-5 |
| 필기 FULL r02-61 — `tar cvJf data.tar.xz /home/data` (xz 아카이브 생성) | 2-5 |
| 필기 FULL r02-62 — 일 0 · 수 3 → 금 5 는 **수요일 레벨 3 이후 변경분** | 6-2 레벨 조합표 |
| 필기 FULL r03-41 · r07-29 · r08-41 — `tar tzf` 는 압축 해제 없이 목록, `tar zxvf` 는 추출 | 2-2, 6-3 대비 |
| 필기 FULL r03-46 · r09-47 · r10-46 — MBR/GPT 비교, **보호 MBR 목적**, GPT 백업본은 디스크 끝 | 5-1 (34섹터 구조), 5-5 |
| 필기 FULL r03-48 · r09-50 · r10-47 — LVM 스냅샷 CoW, "생성 시점에 전체 복사" 는 오답 | 7-1, 7-3 |
| 필기 FULL r03-60 · r07-28 — 일요일 전체 + 월~수 차등 → 목요일 장애 = **전체 + 수요일 차등 (2개)** | 1-1 전략표, 2-3 |
| 필기 FULL r04-44 — `tar xzvf backup.tar.gz -C /opt` = 지정 디렉터리에 추출 | 2-4 (`-C`) |
| 실기 r02-3 — `backup.tar.gz` 를 `/restore` 에 추출하는 명령 | 2-4, 2-6 |
| 필기 FULL r04-58 · r06-42 — `rsync -avz --delete /data/ host:/backup/` 원격 미러 동기화 | 4-2, 4-7 |
| 실기 r02-5 — 권한·심볼릭링크 보존 + 압축 원격 전송 = `rsync -avz /data user@host:/backup` | 4-7 (`-a` 구성 요소) |
| 필기 FULL r04-63 — "마지막 백업 이후 변경분만" = **증분 백업** | 1-1 |
| 필기 FULL r05-64, 실기 r03-10 — tar 증분 스냅샷 파일 지정 옵션 = `-g` / `--listed-incremental` | 2-1, 2-2, 9-1 |
| 필기 FULL r06-34 · r07-41 — `dd if=/dev/zero of=/swapfile bs=1M count=2048` = 2 GB 스왑 파일 | 5-2 옵션표 (`bs × count`) |
| 필기 FULL r06-96 · r09-36 — 송수신 양쪽 `sha256sum` 비교, `sha256sum -c backup.sha256` 출력 해석 | 4-8, 9-2 |
| 필기 FULL r07-27 · r10-45 — dump 레벨 조합(0·3·2·5 / 0·2·3·2·4)에서 복원 세트 고르기 | 6-2 레벨 조합표 |
| 필기 FULL r07-24 — `rpm2cpio` 로 RPM 내부 추출 | 12-4 `cpio` 행 (Part 02 연계) |
| 필기 FULL r08-63 — "0~9 레벨 + fstab dump 필드 연동 + 파일시스템 단위" = `dump` | 6-1, 6-6 도구 대응표 |
| 필기 FULL r09-55 · r10-44 — `dd` 는 블록 단위 저수준 복제, `of=` 실수 시 파괴 / `tar` 는 레벨 체계 없음 | 5-2, 5-3, 1-2 도구표 |
| 필기 FULL r09-61 — 침해 대비책으로 `/etc/shadow` 주기적 백업 | 3-3 |
| 필기 FULL r09-62 — 백업 **3-2-1 원칙** (사본 3 · 매체 2 · 오프사이트 1) | 1-1, 4-7 |
| 필기 FULL r09-63 — `tar czf - /etc \| gpg -c -o etc.tar.gz.gpg` (대칭키 암호화 백업) | 3-1 |
| 필기 FULL r09-46 — `gpg --encrypt /dev/sdb1` 은 오답(장치 직접 암호화 아님) | 3-1, 12-3 `gpg` 행 |
| 필기 FULL r01-7 · r02-6 · r03-18 · r04-7 · r05-7 · r07-15 — 런레벨 ↔ systemd 타겟 대응 (0=poweroff, 1=rescue, 3=multi-user, 5=graphical, 6=reboot) | 10-2 대응표 |
| 필기 FULL r04-8 · r08-9 — 현재 런레벨 확인 (`runlevel`, `who -r`) | 10-1 검증, 10-2 |
| 필기 FULL r10-7 — `shutdown` 시각 인자 해석 ("3초 후" 오답 / "+N = N분 후") | 10-1 |
| 필기 FULL r01-57 · r02-58 — `journalctl --list-boots` = 부팅 세션 목록 | 10-4 |
| 필기 FULL r08-30 — `uptime` load average 를 코어 수와 비교해 해석 | 10-4 |
| 필기 FULL r02-88 · r03-88 — `virsh shutdown` = 정상 종료(강제 차단은 `destroy`) | 10-2 대응표 `virsh` 행 |
| 필기 FULL r01-17 · r02-13 · r07-8 — X 서버(사용자 측) / X 클라이언트(응용) 역할, 원격 표시 흐름 | 13-1 |
| 필기 FULL r04-7 — 런레벨 5 = X 윈도 기반 다중 사용자 모드 | 13-5, 10-2 |
| 필기 FULL r04-12 — 텍스트 모드에서 X 구동 = `startx` | 13-5 |
| 필기 FULL r04-13 — `export DISPLAY=192.168.5.10:0` 후 X 클라이언트 출력 위치 | 13-2 |
| 필기 FULL r05-12 — 호스트 단위 X 접근 허용 = `xhost +192.168.1.10` | 13-3 |
| 필기 FULL r05-13 · r09-10 — `xauth` = MIT-MAGIC-COOKIE-1 기반 **사용자 단위** 인증 | 13-3 비교표 |
| 주제 — 백업 유형 비교(전체·증분·차등) 및 복원 세트 수 계산 | 1-1, 2-2, 2-3 |
| 주제 — 장애 유형별 복구 절차 서술 (진단 → 격리 → 복원 → 검증) | 9-5 복구 런북 |
| 주제 — 백업 자동화 스크립트 + cron 등록 (`/etc/cron.d` 6필드) | 9-1, 9-4 |

---

## 이전 / 다음

[[11-container-virtualization]] ← · **(마지막 파트)**

[[README]] · [[../THEORY/system-security]] · [[../THEORY/disk-device]]
