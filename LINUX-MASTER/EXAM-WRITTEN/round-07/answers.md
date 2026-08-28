---
title: 필기 모의고사 Round 07 — 정답·해설
type: exam-written-answer
round: 7
tags:
  - exam/linux-master
  - exam/written
  - exam/answers
updated: 2026-08-28
---

# 필기 모의고사 Round 07 — 정답·해설

- 문제: [[questions]]
- ⚠ 반드시 풀이를 마친 후 열람

---

## 정답 요약

| 번호 | 정답 | 번호 | 정답 | 번호 | 정답 | 번호 | 정답 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | ① | 6 | ② | 11 | ③ | 16 | ④ |
| 2 | ② | 7 | ③ | 12 | ④ | 17 | ① |
| 3 | ③ | 8 | ④ | 13 | ① | 18 | ② |
| 4 | ④ | 9 | ① | 14 | ② | 19 | ③ |
| 5 | ① | 10 | ② | 15 | ③ | 20 | ④ |

---

## 해설

**1. ①** 앤드루 타넨바움의 교육용 유닉스 클론 MINIX 가 리눅스 초기 설계의 참고 모델. MULTICS 는 유닉스의 전신 개념, BSD 는 버클리 유닉스 계열.

**2. ②** 사용자 명령어 해석은 **셸(shell)** 의 역할. 커널은 프로세스·메모리·장치·파일시스템 등 하드웨어 자원 관리를 담당.

**3. ③** `/boot` 에 커널 이미지(vmlinuz)·initramfs·GRUB 설정이 위치. `/proc`·`/sys` 는 가상 파일시스템 → [[grub2-install]].

**4. ④** `$(command)`(또는 백틱)이 명령 치환. `${var}` 는 변수 확장, `$[...]` 는 (구식) 산술 확장.

**5. ①** `A && B` 는 A 성공 시에만 B 실행. `A || B` 는 A 실패 시 B, `;` 는 결과 무관 순차 실행.

**6. ②** `>>` 는 추가(append), `>` 는 덮어쓰기(truncate). `<` 는 입력, `2>` 는 표준에러 재지정.

**7. ③** `-i`(ignore-case)가 대소문자 무시. `-v` 반전, `-c` 일치 행 수, `-n` 행 번호 → [[grep]].

**8. ④** 주소 없이 `3d` 처럼 행 번호 + `d` 로 특정 행 삭제. `/3/d` 는 3 을 포함한 행 삭제, `-n '3p'` 는 3행 출력 → [[sed]].

**9. ①** `Z`(Zombie)는 종료됐지만 부모가 회수(wait)하지 않아 종료 상태만 남은 프로세스. R=실행, S=대기, T=정지 → [[ps]].

**10. ②** 윈도 매니저가 창 이동·크기·테두리·포커스를 제어. 디스플레이 매니저는 로그인, 데스크톱 환경은 통합 GUI, 위젯 툴킷은 GTK·Qt 등.

**11. ③** `A` 레코드가 호스트 → IPv4, `AAAA` 는 IPv6. `MX` 메일 서버, `PTR` 역방향, `CNAME` 별칭 → [[network-service]] 2-2.

**12. ④** `ip addr`(iproute2)이 인터페이스 IP·상태 조회로 `ifconfig` 를 대체. `route` 는 라우팅, `ss` 는 소켓 → [[ip]] · [[ss]].

**13. ①** `traceroute`(TTL 증가로 홉별 응답 수집)가 경로 추적. `ping` 은 도달성만, `netstat` 은 연결·라우팅 → [[ping]].

**14. ②** `systemctl enable` 은 부팅 시 자동 시작 등록(심볼릭 링크 생성). `start` 는 즉시 기동으로 재부팅 후 유지 안 됨 → [[systemctl]].

**15. ③** `lsblk` 는 디스크·파티션을 트리로 표시. `fdisk -l` 은 파티션 테이블 상세, `df` 는 사용량 → [[lsblk]].

**16. ④** `mkswap` 으로 스왑 시그니처를 만든 뒤 `swapon` 으로 활성화. 영구 적용은 `/etc/fstab` 에 `swap` 항목 등록 → [[mount]].

**17. ①** `visudo` 는 잠금·문법 검증을 제공해 잘못된 문법으로 sudo 가 마비되는 것을 방지. 직접 `vi` 편집은 위험 → [[sudo]] · [[system-security]] 2-5.

**18. ②** `anonymous_enable=NO` 로 익명 접속 차단. `local_enable` 로컬 계정 허용, `write_enable` 업로드 허용, `chroot_local_user` 상위 이동 차단 → [[network-service]] 4-3.

**19. ③** firewalld 는 런타임/영구가 분리되어 `--permanent` 로 영구 규칙 추가 후 `--reload` 로 적용해야 함. `--permanent` 없는 규칙은 재부팅 시 소실 → [[firewall-cmd]] · [[network-security]] 2-3.

**20. ④** Tripwire(및 AIDE)는 파일 해시 기준선(DB)과 현재 상태를 비교해 변조를 탐지하는 HIDS. Nmap 포트 스캔, John 비밀번호 크래킹, tcpdump 패킷 캡처 → [[system-security]] 2-8 · [[network-security]] 3.

---

## 오답 복습 링크
- sudo·무결성 → [[system-security]]
- FTP·DNS → [[network-service]]
- firewalld·HIDS → [[network-security]]
- 텍스트 처리·프로세스 → [[grep]] · [[sed]] · [[ps]]
