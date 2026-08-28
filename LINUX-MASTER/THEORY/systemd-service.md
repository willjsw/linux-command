---
title: systemd·서비스 관리
type: exam-theory
category: LINUX-MASTER
tags:
  - exam/linux-master
  - exam/theory
  - linux/systemd
  - linux/boot
  - topic/service
  - task/manage
related: ["[[system-structure]]", "[[process-management]]", "[[network-service]]", "[[journalctl]]", "[[reboot]]"]
updated: 2026-08-28
---

# systemd·서비스 관리

- 리눅스마스터 1급 필기 대비 이론 — init 시스템 변천·systemd 유닛·서비스 관리·부팅 타겟
- 런레벨↔타겟 대응표와 `systemctl` 하위 명령이 최빈출 → 표 단위 암기 필요

---

## 1. init 시스템 변천

### 1-1. 세대별 비교

| 세대 | init 시스템 | 특징 | 한계 |
| --- | --- | --- | --- |
| 1세대 | **SysV init** | 런레벨 기반, `/etc/inittab` 설정, 스크립트 순차 실행 (`/etc/rc.d/`) | 직렬 부팅 → 느림, 의존성 수동 관리 |
| 2세대 | **Upstart** (Ubuntu) | 이벤트 기반, 비동기 처리 도입 | SysV 과도기, 표준화 실패 |
| 3세대 | **systemd** (현행 표준) | 유닛 기반, 병렬 부팅, 의존성 자동 해석 | 복잡도 증가·거대 단일체 비판 |

### 1-2. systemd 등장 배경

- **병렬 부팅**: 서비스 간 의존성만 만족하면 동시 기동 → 부팅 시간 단축
- **의존성 자동 해석**: 유닛 간 `After`/`Requires` 로 순서·요구 관계 명시
- **소켓 활성화**: 소켓을 먼저 열어두고 실제 요청 시점에 데몬 기동 (지연 로딩)
- **온디맨드 기동**: `.socket` `.path` `.timer` 트리거로 필요 시에만 서비스 시작
- SysV 의 직렬 스크립트 실행 병목·수동 의존성 관리 문제 해소가 핵심 동기

---

## 2. systemd 개념

- **PID 1**: 부팅 후 커널이 최초로 실행하는 프로세스 = systemd (모든 프로세스의 조상) → [[process-management]]
- **유닛(unit)** 기반: 관리 대상을 유닛이라는 단위로 추상화, 확장자로 종류 구분
- 명령체계: `systemctl` (서비스·유닛 제어), `journalctl` (로그 조회) → [[systemctl]] · [[journalctl]]
- 유닛 파일 위치
  - `/usr/lib/systemd/system/` : 패키지 설치 시 배포되는 기본 유닛 (배포판 제공)
  - `/etc/systemd/system/` : 관리자 커스텀·오버라이드 (우선순위 높음)
  - `/run/systemd/system/` : 런타임 생성 유닛 (휘발성)

---

## 3. 유닛 종류 (최빈출)

| 확장자 | 유닛 종류 | 용도 |
| --- | --- | --- |
| `.service` | 서비스 유닛 | 데몬·프로세스 관리 (가장 일반적) |
| `.target` | 타겟 유닛 | 유닛 그룹화 = 런레벨 대응 (부팅 상태 정의) |
| `.socket` | 소켓 유닛 | 소켓 기반 활성화 (요청 시 서비스 기동) |
| `.mount` | 마운트 유닛 | 파일시스템 마운트 지점 관리 |
| `.automount` | 자동마운트 유닛 | 접근 시점 자동 마운트 |
| `.timer` | 타이머 유닛 | 시간 기반 실행 (cron 대체) |
| `.device` | 디바이스 유닛 | udev 인식 하드웨어 장치 표현 |
| `.path` | 경로 유닛 | 파일·디렉터리 변화 감시 후 서비스 트리거 |
| `.swap` | 스왑 유닛 | 스왑 공간 관리 |
| `.slice` | 슬라이스 유닛 | cgroup 계층 리소스 그룹 |

- 시험 포인트: 확장자↔용도 매칭, `.service`(데몬)·`.target`(런레벨)·`.timer`(cron 대체)·`.mount`(마운트) 우선 암기

---

## 4. 서비스 관리 — `systemctl`

### 4-1. 상태 제어

| 명령 | 기능 |
| --- | --- |
| `systemctl start <유닛>` | 서비스 즉시 시작 |
| `systemctl stop <유닛>` | 서비스 즉시 중지 |
| `systemctl restart <유닛>` | 중지 후 재시작 |
| `systemctl reload <유닛>` | 설정만 재적용 (프로세스 재기동 없음) |
| `systemctl status <유닛>` | 현재 상태·최근 로그 조회 |

### 4-2. 부팅 연동 (자동 시작)

| 명령 | 기능 |
| --- | --- |
| `systemctl enable <유닛>` | 부팅 시 자동 시작 등록 (`.wants` 심볼릭 링크 생성) |
| `systemctl disable <유닛>` | 자동 시작 해제 |
| `systemctl enable --now <유닛>` | 등록과 동시에 즉시 시작 |
| `systemctl mask <유닛>` | 유닛 완전 차단 (`/dev/null` 링크 → start 조차 불가) |
| `systemctl unmask <유닛>` | mask 해제 |

- `disable` 과 `mask` 차이가 함정: `disable` 은 수동 start 가능, `mask` 는 수동 start 도 차단

### 4-3. 상태 질의 (스크립트용)

- `systemctl is-active <유닛>` : 실행 중 여부 (`active`/`inactive`)
- `systemctl is-enabled <유닛>` : 자동 시작 등록 여부 (`enabled`/`disabled`)
- `systemctl is-failed <유닛>` : 실패 상태 여부

### 4-4. 유닛 목록

- `systemctl list-units` : 현재 로드된(활성) 유닛 목록
- `systemctl list-unit-files` : 설치된 전체 유닛 파일과 enable 상태
- `systemctl --failed` : 실패한 유닛만 필터

---

## 5. 타겟(target) — 런레벨 대응

### 5-1. 런레벨↔타겟 대응표 (최빈출)

| 런레벨 | 타겟 | 상태 |
| --- | --- | --- |
| 0 | `poweroff.target` | 시스템 종료 |
| 1 / S | `rescue.target` | 단일 사용자 모드 (복구) |
| 2 | `multi-user.target` | 다중 사용자 (NFS 없음, 관례상 매핑) |
| 3 | `multi-user.target` | 다중 사용자 **텍스트 모드** (CLI) |
| 4 | `multi-user.target` | 미사용 (사용자 정의) |
| 5 | `graphical.target` | 다중 사용자 **GUI 모드** |
| 6 | `reboot.target` | 재부팅 → [[reboot]] |

- 런레벨 2·3·4 는 모두 `multi-user.target` 으로 수렴 (systemd 는 텍스트 다중 사용자를 단일 타겟으로 통합)
- `emergency.target` : rescue 보다 최소한의 복구 환경 (루트 파일시스템만 읽기 전용 마운트)

### 5-2. 기본 타겟 조회·변경

| 명령 | 기능 | SysV 대응 |
| --- | --- | --- |
| `systemctl get-default` | 현재 기본 타겟 조회 | `runlevel` |
| `systemctl set-default <타겟>` | 기본 타겟 변경 (부팅 시 진입) | `/etc/inittab` 편집 |
| `systemctl isolate <타겟>` | 현재 세션에서 타겟 즉시 전환 | `init <런레벨>` |

- `set-default` 는 `/etc/systemd/system/default.target` 심볼릭 링크를 갱신
- `isolate graphical.target` : 재부팅 없이 GUI 모드로 전환

---

## 6. 유닛 파일 구조

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application Daemon
After=network.target                  # network 이후 기동 (순서만)
Requires=postgresql.service           # 의존 유닛 (동반 기동·실패 시 함께 실패)

[Service]
Type=simple                           # simple/forking/oneshot/notify/dbus
ExecStart=/usr/local/bin/myapp        # 시작 명령 (필수)
ExecStop=/usr/local/bin/myapp --stop
Restart=on-failure                    # 비정상 종료 시 자동 재시작
User=myapp

[Install]
WantedBy=multi-user.target            # enable 시 연결될 타겟
```

- **[Unit]**: `Description`(설명), `After`(기동 순서), `Requires`(강한 의존), `Wants`(약한 의존)
  - `After` 는 순서만, `Requires` 는 의존 관계 → 순서 보장하려면 `After` 병행 필요
- **[Service]**: `Type`, `ExecStart`(필수), `ExecStop`, `ExecReload`, `Restart`
  - `Type=simple`(기본, 포그라운드), `forking`(데몬화), `oneshot`(1회 실행)
- **[Install]**: `WantedBy`(enable 대상 타겟) — 대부분 `multi-user.target`
- 유닛 파일 수정 후 `systemctl daemon-reload` 로 재로딩 필수

---

## 7. 저널 — `journalctl`

- systemd 통합 로그 시스템 (journald 데몬이 수집) → [[journalctl]]

| 옵션 | 기능 |
| --- | --- |
| `journalctl -u <유닛>` | 특정 유닛 로그만 조회 |
| `journalctl -f` | 실시간 추적 (`tail -f` 대응) |
| `journalctl -p err` | 우선순위 필터 (err 이상) |
| `journalctl -b` | 이번 부팅 로그만 (`-b -1` = 직전 부팅) |
| `journalctl --since "2026-08-28"` | 시간 범위 필터 (`--until` 병행) |

- **로그 영속화**: 기본은 휘발성(`/run/log/journal`) → `/var/log/journal/` 디렉터리 생성 시 영구 저장
  - `/etc/systemd/journald.conf` 의 `Storage=persistent` 설정으로 영속화

---

## 8. SysV 명령 대응

| SysV 명령 | systemd 대응 |
| --- | --- |
| `service <name> start` | `systemctl start <name>` |
| `service <name> status` | `systemctl status <name>` |
| `chkconfig <name> on` | `systemctl enable <name>` |
| `chkconfig <name> off` | `systemctl disable <name>` |
| `chkconfig --list` | `systemctl list-unit-files` |
| `init <런레벨>` | `systemctl isolate <타겟>` |
| `runlevel` | `systemctl get-default` |

- `service` · `chkconfig` 는 호환성 위해 일부 배포판에 잔존하나 내부적으로 `systemctl` 로 전달

---

## 9. 부팅 분석

- `systemd-analyze` : 부팅 총 소요 시간 (커널·initrd·userspace 분할)
- `systemd-analyze blame` : 유닛별 기동 소요 시간 내림차순 (부팅 지연 원인 추적)
- `systemd-analyze critical-chain` : 임계 경로(의존성 사슬) 시각화
- `systemctl list-dependencies <유닛>` : 유닛 의존성 트리 조회
- 부팅 지연 진단 → `blame` 으로 느린 서비스 식별이 실무 단골

---

## 연관 문서
- [[system-structure]] — 부팅 과정·런레벨 개념
- [[process-management]] — PID 1·프로세스 계층
- [[network-service]] — 서비스별 데몬·설정 파일
- [[systemctl]] — 서비스 관리 실사용 문서
- [[journalctl]] — systemd 저널 조회 실사용 문서
- [[reboot]] — 재부팅 실사용 문서
