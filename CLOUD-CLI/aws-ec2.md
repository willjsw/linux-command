---
command: aws
category: CLOUD-CLI
aliases: [aws-cli, aws-ec2]
tags:
  - linux/network
  - task/inspect
  - task/configure
  - topic/security
  - topic/troubleshooting
  - privilege/user
related: ["[[curl]]", "[[nc]]", "[[docker]]", "[[jq]]"]
distro: 전체 (awscli v2)
verified: 사고 대응 세션 (ap-northeast-2, 일부 명령 미완)
updated: 2026-08-01
---

# aws (EC2)

- AWS 리소스 조작 CLI (본 문서는 EC2 사고 대응 범위)
- 어원: **A**mazon **W**eb **S**ervices
- 인스턴스 메타데이터는 IMDSv2(토큰 기반)로 조회

---

## aws ec2 (조회·스냅샷)

```bash
aws ec2 <서브명령> --region <리전> [옵션]

# Examples
# 볼륨 ID 조회
aws ec2 describe-instances --instance-ids "$IID" --region ap-northeast-2 \
  --query 'Reservations[].Instances[].BlockDeviceMappings[].Ebs.VolumeId' --output text
# EBS 스냅샷 생성 (증거 보존)
aws ec2 create-snapshot --volume-id <실제vol-id> --region ap-northeast-2 \
  --description "pg_hba incident 2026-07-30"
# 보안그룹 인바운드 확인 (5432/6379 노출 여부)
aws ec2 describe-security-groups --region ap-northeast-2 \
  --query 'SecurityGroups[].{Name:GroupName,Ingress:IpPermissions[?FromPort==`5432`||FromPort==`6379`]}' \
  --output json
```

### 명령어 설명
- 사용 목적
	- 침해 인스턴스 EBS 스냅샷 생성(증거 보존) 시 사용
	- 보안그룹 인바운드 규칙(포트 노출) 확인 시 사용
- 특이사항
	- **문서 내 자리표시자 `<vol-id>`는 셸이 `<`를 리다이렉션으로 해석** → `No such file or directory`. 치환 후 실행
	- 예시 형식값(`vol-0abc123def456`) 그대로 실행 시 `InvalidVolumeID.Malformed` → 실제 ID 조회 필요
	- `--query`는 JMESPath, `--output`은 `text`/`json`/`table`
	- CLI 권한 부재 시 스냅샷은 EC2 콘솔 수행 (볼륨 → Actions → Create snapshot)

### 옵션
- `--region` : 대상 리전 (**region**)
- `--query` : JMESPath 출력 필터 (**query**)
- `--output` : 출력 형식 `text`/`json`/`table` (**output**)

---

## IMDSv2 메타데이터 조회

```bash
# Examples
TOKEN=$(curl -sX PUT http://169.254.169.254/latest/api/token \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
IID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
echo "$IID"
```

### 명령어 설명
- 사용 목적
	- 인스턴스 ID·리전 등 자기 메타데이터 조회 시 사용
- 특이사항
	- **변수 대입 명령은 출력 없음이 정상** → `echo "$IID"`로 값 확인
	- 빈 값이면 IMDS 접근 실패 (IMDSv2 필수화·홉 제한 등)
	- IMDSv2는 PUT으로 토큰 발급 후 GET 헤더에 토큰 전달 → [[curl]]

---

## 연관 명령어
- [[curl]] : IMDSv2 토큰 발급·메타데이터 조회
- [[nc]] : 보안그룹 차단 여부 외부 실검증
- [[docker]] : 인스턴스 내 컨테이너 포트 노출 확인
- [[jq]] : `--output json` 결과 후처리
