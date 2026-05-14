## 1. 개요

### 1.1 사고 배경

Codefinger는 AWS의 정상적인 네이티브 기능을 악용하여 클라우드 스토리지 데이터를 암호화하는 새로운 유형의 랜섬웨어 공격 캠페인입니다.

기존 랜섬웨어는 피해자 시스템에 악성코드를 설치하여 로컬 파일을 암호화하는 방식을 사용하였습니다. 그러나 Codefinger는 악성코드 배포 없이 **AWS의 서버 측 암호화 기능(SSE-C, Server-Side Encryption with Customer Provided Keys)** 을 통해 S3의 객체들을 암호화하는 기존과는 다른 방식을 사용한 사례입니다.

이 공격은 보안 전문 기업 Halcyon에 의해 최초 발견되었으며, 초기 조사 시점 기준으로 최소 두 곳 이상의 피해 조직이 확인되었습니다.

---

### 1.2 사고 요약

| 항목 | 내용 |
|------|------|
| **공격 유형** | 클라우드 스토리지 랜섬웨어 (Ransomware-as-a-Tactic) |
| **피해 대상** | AWS S3 버킷 내 저장된 데이터 전체 |
| **악용 기능** | AWS SSE-C (고객 제공 키 기반 서버 측 암호화) |
| **암호화 알고리즘** | AES-256 |
| **초기 침투 경로** | 노출된 AWS IAM 자격증명 (Access Key / Secret Key) |
| **필요 권한** | `s3:GetObject`, `s3:PutObject` |
| **데이터 복구 가능성** | **불가** (AWS도 키를 보유하지 않음) |
| **추가 피해 위협** | S3 Lifecycle 정책을 통한 7일 후 자동 삭제 설정 |
| **요구 사항** | Bitcoin 지불 후 AES-256 복호화 키 제공 |

본 공격의 핵심 위험 요소는 **AWS 플랫폼 차원에서도 복구가 불가능**하다는 점입니다. AWS는 SSE-C 방식에서 고객이 제공한 암호화 키를 저장하지 않으므로, 피해자가 AWS에 신고하더라도 암호화된 데이터를 복원하는 것은 기술적으로 불가능합니다.

---

## 2. 공격 분석

### 2.1 공격 흐름 (Attack Flow)

<p align="center"><img src="https://cdn.prod.website-files.com/64e50cbe2b6f932c04238c14/6798e29c7264e072bc79e8d8_6798e2001324ff8a272bd047_Codefinger-AWS-ransomware-attack.webp" width="100%"></p>

### 2.2 단계별 공격 프로세스

<p align="center"><img src="https://www.trendmicro.com/content/dam/trendmicro/global/en/research/25/k/s3-ransomware/S3Ransomware_Fig06.png" width="80%"></p>

#### AWS 자격증명 탈취 

공격자는 사전에 노출된 AWS IAM 자격증명(`Access Key ID` / `Secret Access Key`)을 획득합니다. 자격증명 노출의 주요 원인은 다음과 같습니다.

- 소스코드 저장소(GitHub, GitLab 등)에 하드코딩된 키 커밋
- 애플리케이션 설정 파일(`.env`, `config.yaml` 등)의 외부 노출
- 운영 환경의 환경 변수 유출
- 피싱 또는 기타 소셜 엔지니어링 기법

탈취된 자격증명은 다이어그램에서 IAM role credentials 아이콘으로 표현되어 있으며, 공격자는 이 자격증명을 이용해 AWS 환경 내 Permissions(권한 목록) 에 접근합니다. 이 시점에서 AWS 입장에서는 정상적인 인증된 요청으로 인식되기 때문에, 자격증명 탈취 자체를 실시간으로 차단하기가 매우 어렵습니다.

---

#### S3 권한 확인 및 공격 대상 버킷 탐색

공격자는 탈취한 자격증명으로 AWS 환경에 접근하여, 아래 두 가지 권한을 보유한 키를 탐색합니다.

| 권한 | 설명 |
|------|------|
| `s3:GetObject` | S3 버킷 내 객체를 읽고 다운로드하는 권한 |
| `s3:PutObject` | S3 버킷에 객체를 업로드(덮어쓰기 포함)하는 권한 |

이 두 가지 권한만으로도 SSE-C를 활용한 암호화 공격이 완전히 가능합니다. `s3:DeleteObject` 와 같은 명시적인 삭제 권한이 없어도, PutObject를 통한 덮어쓰기만으로 원본 데이터를 사실상 파괴할 수 있습니다.

추가로 아래와같은 기준들을 통해 선정합니다.

|선정 기준|이유|
|-|-|
|버전 관리(Versioning) 비활성화|이전 버전으로 복구 불가|
|Object Lock 미설정|덮어쓰기/삭제가 자유로움|
|MFA Delete 비활성화|인증 없이 버전 삭제 가능|
|과도한 쓰기 권한|`s3:PutObject` 등이 `*` 로 열려 있음|
|고가치 데이터 보유|`backup.sql`, `prod.env`, `db-backup` 등 파일명/버킷명으로 식별|

---

#### AES-256 암호화 키 로컬 생성 (Key Generation)

공격자는 자신의 로컬 환경에서 AES-256 암호화 키를 직접 생성합니다. 이 키는 **AWS 서버에 전송되거나 저장되지 않습니다.**

SSE-C의 작동 원리는 다음과 같습니다.

```
고객(공격자)이 키를 직접 생성
        ↓
PUT 요청 HTTP 헤더에 키를 포함하여 AWS에 전송
        ↓
AWS는 해당 키로 데이터를 암호화한 뒤, 키를 즉시 폐기
        ↓
이후 복호화 시에도 동일한 키를 헤더에 포함해야만 접근 가능
```
AWS는 SSE-C 방식에서 전달받은 암호화 키를 일시적으로만 사용하고 즉시 폐기합니다. 저장되는 것은 키 자체가 아닌 HMAC(Hash-based Message Authentication Code) 뿐이며 이는 동일한 키로 요청이 들어왔을 때 검증하는 용도로만 쓰입니다.

```
키 원본  →  AWS 사용 후 즉시 폐기  (복구 불가)
HMAC    →  CloudTrail에 로깅       (원본 키 역산 불가)
```

이 구조로 인해 피해자가 AWS 고객 지원팀에 신고하더라도, AWS 자체적으로도 데이터를 복호화할 수 없습니다. 암호화된 객체를 다시 읽으려면 반드시 최초에 사용된 AES-256 키 원본을 HTTP 헤더에 포함시켜야 하므로, 그 키를 가진 공격자만이 유일한 복호화 수단을 보유하게 됩니다.

---

#### S3 객체 전체 암호화

공격자는 `GetObject`로 파일을 읽은 뒤, SSE-C 헤더를 포함한 `PutObject` 요청으로 동일 경로에 암호화된 파일을 덮어씁니다. 이 과정에서 사용되는 HTTP 헤더는 다음과 같습니다.

```http
PUT /{object-key} HTTP/1.1
Host: {bucket-name}.s3.amazonaws.com
x-amz-server-side-encryption-customer-algorithm: AES256
x-amz-server-side-encryption-customer-key: {Base64 인코딩된 256비트 키}
x-amz-server-side-encryption-customer-key-MD5: {키의 MD5 해시}
```

이 요청은 AWS API 관점에서 정상적인 트래픽이므로, 일반적인 보안 솔루션이 이를 악성 행위로 탐지하기 매우 어렵습니다.

---

#### 자동 삭제 타이머 설정 

공격자는 S3 Object Lifecycle Management API를 통해 **7일 후 자동 삭제 정책**을 설정하여 피해자에게 심리적 압박을 가합니다.

```json
{
  "Rules": [
    {
      "ID": "DeleteAfter7Days",
      "Status": "Enabled",
      "Filter": { "Prefix": "" },
      "Expiration": { "Days": 7 }
    }
  ]
}
```

이 정책이 적용되면 피해자가 협박에 응하지 않을 경우 7일 이내에 버킷 내 모든 암호화 데이터가 영구 삭제됩니다.

---

#### 랜섬노트 배포 및 협박

공격자는 피해 버킷의 모든 디렉토리에 랜섬노트 파일을 생성합니다. 랜섬노트에는 다음 내용이 포함됩니다.

- 지정된 Bitcoin 주소로 몸값 송금 요청
- 송금 완료 시 AES-256 복호화 키 제공 약속
- 피해자가 계정 권한을 변경하거나 파일을 수정할 경우 협상을 일방적으로 종료하겠다는 경고
- 삭제 시한(7일) 명시를 통한 심리적 압박

---

## 3. 대응 방안

<p align="center"><img src="https://cdn.prod.website-files.com/64e50cbe2b6f932c04238c14/6798e29c7264e072bc79e8d5_6798e2338727e27c308ad533_how-to-detect-cloud-ransomware.webp" width="90%"></p>

### 3.1 즉각 대응 절차

####  S3 Lifecycle 정책 삭제

공격자가 설정한 자동 삭제 타이머를 제거하여 추가적인 데이터 손실을 방지합니다.

```bash
# 버킷의 Lifecycle 정책 확인
aws s3api get-bucket-lifecycle-configuration \
  --bucket {버킷명}

# 삭제 정책 제거
aws s3api delete-bucket-lifecycle \
  --bucket {버킷명}
```

---

#### CloudTrail 로그 분석 및 피해 범위 확인

AWS CloudTrail을 통해 비정상적인 API 호출 내역을 확인하고, 암호화된 객체의 범위를 파악합니다.

```bash
# 특정 기간 내 S3 PutObject 이벤트 조회
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=PutObject \
  --start-time {시작시간} \
  --end-time {종료시간}
```


### 3.2 사후 조치 및 재발 방지

#### SSE-C 사용 차단 버킷 정책 적용

SSE-C를 이용한 PutObject 요청을 버킷 수준에서 원천 차단합니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenySSEC",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::{버킷명}/*",
      "Condition": {
        "Null": {
          "s3:x-amz-server-side-encryption-customer-algorithm": "false"
        }
      }
    }
  ]
}
```

---

####  IAM Role 기반 인증으로 전환

하드코딩된 Access Key 방식 대신, IAM Role 기반의 단기 자격증명 방식으로 전환합니다.

| 환경 | 권장 인증 방식 |
|------|--------------|
| EC2 인스턴스 | EC2 Instance Profile (IAM Role) |
| Lambda 함수 | Lambda Execution Role |
| ECS / EKS | Task Role / Pod Identity |
| 온프레미스 서버 | IAM Roles Anywhere |
| 개발자 워크스테이션 | AWS IAM Identity Center (SSO) |

IAM Role은 AWS STS(Security Token Service)를 통해 단기 자격증명을 자동으로 발급·교체하므로, 장기 자격증명 노출 위험을 근본적으로 제거할 수 있습니다.

---

####  S3 버전 관리(Versioning) 및 MFA Delete 활성화

버전 관리를 활성화하면 PutObject로 덮어쓰인 객체의 이전 버전을 보존할 수 있습니다. MFA Delete를 함께 활성화하면, 버전 삭제 시 MFA 인증이 추가로 요구되므로 공격자가 이전 버전까지 파괴하는 것을 방지할 수 있습니다.

```bash
# 버전 관리 활성화
aws s3api put-bucket-versioning \
  --bucket {버킷명} \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "{시리얼번호} {MFA코드}"
```

---

#### AWS CloudTrail 및 GuardDuty 상시 활성화

**CloudTrail**: 모든 AWS API 호출을 로깅하여 침해 발생 시 사후 분석의 근거 자료로 활용합니다.
**GuardDuty**: 비정상적인 대량 PutObject 호출, 알 수 없는 IP에서의 API 접근 등을 자동으로 탐지하고 경보를 발생시킵니다.

구체적으로는 다음과같은 IAM 이벤트를 탐지하는 것이 필요합니다.
- AdministratorAccess 정책이 User/Role/Group에 새로 부착된 경우
- IAM 사용자의 MFA가 비활성화된 경우
- 과도한 신뢰 정책을 가진 Role이 생성된 경우
- IAM 사용자에게 새 Access Key가 발급된 경우

---

#### 자격증명 노출 모니터링 자동화

소스코드 저장소에 자격증명이 포함되지 않도록 사전에 차단하는 도구를 CI/CD 파이프라인에 통합합니다.

| 도구 | 역할 |
|------|------|
| `git-secrets` | Git 커밋 시 AWS 키 패턴 자동 탐지 및 차단 |
| `truffleHog` | 저장소 히스토리 전체에서 민감 정보 탐색 |
| AWS Secrets Manager | 데이터베이스 패스워드, API 키 등 비밀값 중앙 관리 |
| AWS Config | IAM 정책 변경 사항 지속적 모니터링 |

---

출처:

- [Abusing AWS Native Services: Ransomware Encrypting S3 Buckets with SSE-C](https://www.halcyon.ai/blog/abusing-aws-native-services-ransomware-encrypting-s3-buckets-with-sse-c)
- [Ransomware abuses Amazon AWS feature to encrypt S3 buckets](https://www.bleepingcomputer.com/news/security/ransomware-abuses-amazon-aws-feature-to-encrypt-s3-buckets/)
- [Codefinger Ransomware Campaign Targeting S3 Buckets](https://threats.wiz.io/all-incidents/codefinger-ransomware-campaign-targeting-s3-buckets)
- [Defending Against Codefinger Ransomware in AWS S3](https://www.vectra.ai/blog/defending-against-codefinger-ransomware-in-aws-s3)