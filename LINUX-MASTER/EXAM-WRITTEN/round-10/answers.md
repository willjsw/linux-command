---
title: 필기 모의고사 Round 10 — 정답·해설
type: exam-written-answer
round: 10
tags:
  - exam/linux-master
  - exam/written
  - exam/answers
updated: 2026-08-28
---

# 필기 모의고사 Round 10 — 정답·해설

- 문제: [[questions]]
- ⚠ 반드시 풀이를 마친 후 열람

---

## 정답 요약

| 번호 | 정답 | 번호 | 정답 | 번호 | 정답 | 번호 | 정답 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | ③ | 6 | ④ | 11 | ① | 16 | ② |
| 2 | ④ | 7 | ① | 12 | ② | 17 | ④ |
| 3 | ② | 8 | ③ | 13 | ④ | 18 | ① |
| 4 | ③ | 9 | ② | 14 | ③ | 19 | ③ |
| 5 | ① | 10 | ④ | 15 | ① | 20 | ② |

---

## 해설

**1. ③** 리눅스는 다중 사용자·다중 작업을 지원하는 오픈 소스 OS. '단일 사용자 전용'은 틀린 설명.

**2. ④** `rpm -V` 는 설치 시점 대비 파일의 크기·권한·MD5 등 변경 여부를 검증(무결성 점검). 설치는 `-i`, 삭제는 `-e`.

**3. ②** `systemctl isolate graphical.target` 은 즉시 GUI 모드로 전환. `set-default` 는 다음 부팅 기본값 지정 → [[systemctl]].

**4. ③** initramfs(initrd)는 커널이 실제 루트 파일시스템을 마운트하기 전 필요한 드라이버·도구를 담은 임시 루트. `dracut` 로 생성.

**5. ①** `ldd` 는 실행 파일이 링크한 공유 라이브러리 의존성을 표시. `ldconfig` 는 캐시 갱신, `nm` 은 심볼 목록 → [[ldd]].

**6. ④** stderr 는 파일 디스크립터 2 → `2> file`. `> file`·`1> file` 은 stdout 리다이렉트.

**7. ①** `alias` 로 별칭을 정의(`alias ll='ls -l'`). `export` 는 환경변수 내보내기, `env` 는 환경변수 조회.

**8. ③** `renice` 는 실행 중 프로세스의 nice 값을 변경. `nice` 는 실행 시점에 우선순위를 지정 → [[ps]].

**9. ②** `$DISPLAY`(예 `:0`, `host:0`)가 X 클라이언트의 출력 대상 X 서버·화면을 지정. `$XAUTHORITY` 는 인증 쿠키 경로.

**10. ④** `top` 은 CPU·메모리·프로세스를 실시간 갱신하는 대화형 모니터. `ps` 는 정적 스냅숏 → [[ps]].

**11. ①** `httpd -t`(= `apachectl configtest`)로 설정 문법을 검사. 오류가 없어야 재시작이 안전 → [[network-service]] 1-2.

**12. ②** 능동 모드는 서버 20번 포트가 데이터 연결을 개시(제어는 21). 방화벽 환경에서는 수동(Passive) 모드 사용 → [[network-service]] 4-3.

**13. ④** `PTR` 레코드가 IP→도메인 역방향 조회에 사용(역방향 존). `A` 는 정방향(도메인→IP) → [[network-service]] 2-2.

**14. ③** MTA(sendmail·postfix)가 메일 전송·라우팅 담당. MUA 는 사용자 프로그램, MDA 는 로컬 배달 → [[network-service]] 3.

**15. ①** `ss -tlnp` = TCP(-t) LISTEN(-l) 숫자표기(-n) 프로세스(-p) 조회. `ss -u` 는 UDP 만 → [[ss]].

**16. ②** 확장 파티션 내부에 여러 개 만들 수 있는 것은 논리(Logical) 파티션. MBR 은 주 4개(또는 주 3 + 확장 1) 제한 → [[fdisk]].

**17. ④** RAID 6 은 패리티를 두 벌 저장해 디스크 2개 동시 장애까지 견딤(가용 용량 n-2). RAID 5 는 1개 → [[network-service]] 7-2.

**18. ①** `pvcreate` 로 디스크·파티션을 PV 로 초기화 → `vgcreate` → `lvcreate` 순으로 구성 → [[lvm]].

**19. ③** `dd` 는 블록 단위 저수준 복제로 디스크 전체 이미지 생성(`dd if=/dev/sda of=disk.img`). ⚠ 파괴적 주의 → [[dd]].

**20. ②** `du` 는 디렉터리 하위를 재귀 합산한 실제 사용량 표시(`du -sh`). `df` 는 파일시스템 단위 → [[du]] · [[df]].

---

## 오답 복습 링크
- 서비스 설정·포트·메일·DNS → [[network-service]]
- 디스크·파티션·RAID·LVM → [[fdisk]] · [[lvm]] · [[dd]] · [[du]]
- 프로세스·서비스 → [[ps]] · [[systemctl]] · [[ss]]
- 라이브러리 의존성 → [[ldd]]
